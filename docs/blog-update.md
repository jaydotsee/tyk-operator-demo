# How the blog post needs to be updated

Reference material for revising
[*A practical guide: using Tyk Operator, ArgoCD and Kustomize*](https://tyk.io/blog/a-practical-guide-using-tyk-operator-argocd-and-kustomize/).

The post's GitOps content — Kustomize overlays, app-of-apps, the change-and-push loop —
is still accurate. What has broken is the **environment setup**: it is built entirely on
[`TykTechnologies/tyk-k8s-demo`](https://github.com/TykTechnologies/tyk-k8s-demo), which
is archived. Its replacement,
[`TykTechnologies/tyk-install`](https://github.com/TykTechnologies/tyk-install), covers
both Docker and Kubernetes but deliberately does **not** ship an equivalent of
`up.sh`; Kubernetes installation is explicit Helm against
`kubernetes/helm-self-managed`.

The rewritten instructions are in [`minikube-setup.md`](./minikube-setup.md).

---

## Summary of required changes

| # | Blog section | Status | Action |
| --- | --- | --- | --- |
| 1 | Prerequisites | Incomplete | Add Helm 3.12+, add the Operator licence, add machine-resource guidance |
| 2 | "Install Tyk Stack" (`git clone tyk-k8s-demo` → `./up.sh`) | **Broken** | Replace wholesale with the Helm sequence from `tyk-install` |
| 3 | Production install (`./up.sh --port-offset 1`) | **Broken** | Replace with a second minikube profile + different port-forward ports |
| 4 | `minikube addons enable ingress` | Misleading | Drop, or demote to an optional alternative |
| 5 | ArgoCD install | Works | Add a readiness wait |
| 6 | Fork / `repoURL` / `kubectl apply -f argocd/*-apps.yaml` | Works | Keep; add a "push before applying" reminder |
| 7 | Test commands (`curl https://localhost:8080/...`) | **Broken** | `https://` → `http://` |
| 8 | Dashboard/Portal URLs | Partly wrong | Dashboard 3000/3001 still correct; Portal is not deployed in the new path |
| 9 | CORS example | Underspecified | Give the full `CORS:` block for the `ApiDefinition` CRD |
| 10 | — | Missing | Add a troubleshooting section |

---

## 1. Prerequisites

The post lists Kubernetes/Kustomize familiarity, Git basics, and two clusters. Add:

- **Helm 3.12+** — the new path is Helm-driven; the old `up.sh` hid this.
- **A Tyk Operator licence in addition to the Dashboard licence.** `tyk-k8s-demo` had a
  single `LICENSE` field in `.env`. `tyk-install` has `TYK_LICENSE_KEY` **and**
  `TYK_OPERATOR_LICENSE`, and the stack is installed with `global.components.operator: true`,
  so a missing Operator licence produces an Operator pod that starts and then fails its
  licence check. This is the single most likely thing to derail a reader.
- **Resource guidance.** Two full stacks plus two ArgoCD installs need roughly 8 CPUs and
  14 GB across both profiles. Offer running them sequentially as a fallback.
- **An arm64 caveat, stated up front rather than in troubleshooting.**
  `apps/httpbin/base/deployment.yaml` uses `kennethreitz/httpbin:latest` — last
  published in 2018, `linux/amd64` only — so on Apple Silicon the pod never starts.
  Since a large share of readers are now on Apple Silicon, and since the fix is a Git
  change that ArgoCD deploys (so making it late means an extra push and resync), the
  post should put the swap to `mccutchen/go-httpbin` in the fork-and-clone step where
  the reader is already editing and committing, not in a footnote. Worth noting the
  Service needs a `targetPort: 8080` because go-httpbin runs non-root and cannot bind
  80, and that go-httpbin's tags dropped their `v` prefix partway through the project's
  history, so `v2.25.0` does not exist while `2.25.0` does.

## 2. Installing the Tyk stack — the main rewrite

Delete this:

```bash
git clone https://github.com/TykTechnologies/tyk-k8s-demo.git
cd tyk-k8s-demo
cp .env.example .env
./up.sh --deployments operator tyk-stack
```

Replace it with the `tyk-install` sequence (full version in
[`minikube-setup.md` §3](./minikube-setup.md#step-3--build-the-staging-cluster)):

1. `git clone https://github.com/TykTechnologies/tyk-install.git`,
   `cd tyk-install/kubernetes/helm-self-managed`, `cp .env.example .env`, set both licences.
2. `kubectl create namespace tyk` and create the `tyk-conf` secret from `.env`.
3. `helm install` Bitnami PostgreSQL and Redis.
4. `helm install` cert-manager — **required**, the Operator's admission webhook depends
   on it. The old `up.sh` handled this invisibly; the post never mentions cert-manager
   and must now say so explicitly.
5. `helm install tyk tyk-helm/tyk-stack --values values.yaml --values <minikube overlay>`.

Two points worth calling out in the prose, because they are genuinely easier than before:

- The `tyk-stack` chart's bootstrap job creates the `tyk-operator-conf` secret
  (Dashboard URL, org ID, Dashboard API key, Operator licence) by itself. There is no
  manual Operator-to-Dashboard wiring step.
- The Tyk CRDs ship with the `tyk-operator` subchart, so `helm install` installs them.
  Worth noting the standard Helm caveat that `helm upgrade` does not update CRDs.

### The minikube overlay

`tyk-install/kubernetes/helm-self-managed/values.yaml` targets EKS/GKE/AKS and sets
`service.type: LoadBalancer` for the Gateway, Dashboard and Portal. On minikube those
services sit at `<pending>` forever unless `minikube tunnel` is running. The post needs
one of:

- **(recommended)** layer a small overlay that sets the services to `ClusterIP` and turns
  the Developer Portal off, then use `kubectl port-forward` — this is
  "Option 2: Port-Forward" in the tyk-install README. This repo ships that overlay as
  [`docs/tyk-minikube.values.yaml`](./tyk-minikube.values.yaml).
- for readers who want stable host endpoints — to put their own load balancer in front,
  say — install ingress-nginx per cluster and turn on the charts' own `ingress:` blocks
  (this repo ships the values and ArgoCD manifests in [`docs/ingress/`](./ingress)),
  published with `minikube start --ports`. Worth a short subsection: `port-forward`
  binds loopback only and dies with the pod, so it is a poor load-balancer backend.
- tell readers to run `minikube tunnel` per profile — but see item 7 below, this breaks
  the staging IP allowlist, and two concurrent tunnels fight over host routes because
  both profiles default to the same service CIDR.

A detail worth stating outright, because it cuts the other way from what you would
guess: ingress *is* expressible in the charts' values (tyk-install ships the blocks,
switched off), whereas a pinned NodePort is not — the service templates render no
`nodePort` field, so `type: NodePort` only ever gets you a random high port. Ingress is
the supported route, not the elaborate one.

## 3. The second cluster

`--port-offset 1` does not exist in `tyk-install`. Separation now comes from running a
second minikube profile and choosing different **local** port-forward ports; the
in-cluster ports are unchanged (Dashboard 3000, Gateway 8080 in both clusters).

| Service | staging | production |
| --- | --- | --- |
| Dashboard | `localhost:3000` | `localhost:3001` |
| Gateway | `localhost:8080` | `localhost:8081` |
| ArgoCD | `localhost:8000` | `localhost:8001` |

Everything else about the production install is byte-identical to staging — same `.env`,
same secret, same Helm values. The post should say so rather than repeating the block.

## 4. `minikube addons enable ingress`

Only needed because `tyk-k8s-demo` exposed Tyk over ingress. With port-forwarding it is
dead weight, and it actively breaks the staging demo (item 7). Remove it from the main
flow.

If the post wants an answer for "how do I put my own load balancer in front?", it is
worth noting that the addon is not the only option and arguably not the best one:
installing ingress-nginx by Helm lets you pin the controller's NodePorts and set
`use-forwarded-headers` at install time, and is the same command you would run on a
real cluster. Either way `minikube start --ports=8080:30080` publishes it, with no
tunnel process, and the two profiles are told apart by their host-side mapping rather
than by a port offset. The `--ports` flag is docker/podman only and applies only at
profile creation, which is worth saying in the same breath.

## 5. ArgoCD install

Still correct. Add a wait before port-forwarding, so readers do not hit a connection
refused on a half-started `argocd-server`:

```bash
kubectl wait --for=condition=available deployment --all -n argocd --timeout=600s
```

## 6. Fork, `repoURL`, and applying the app-of-apps

Unchanged and still correct. Two additions:

- Emphasise that the `repoURL` edits must be **committed and pushed** — ArgoCD reads the
  remote repository, not the local working copy. This trips people up.
- Note the ordering constraint: the Tyk stack must be installed **before**
  `kubectl apply -f argocd/staging-apps.yaml`, because it provides the `ApiDefinition`
  and `SecurityPolicy` CRDs.

A ready-made `sed` for the `repoURL` edit is in
[`minikube-setup.md` §1](./minikube-setup.md#step-1--fork-and-clone-this-repository).

## 7. Test commands

The post uses `https://localhost:8080/httpbin/get`. The `tyk-install` values set
`global.tls.gateway: false`, and port-forward adds no TLS, so every test URL becomes
`http://`. Same for the Dashboard.

Also worth a sentence: the staging overlay
(`apps/httpbin/overlays/staging/api_auth.yaml`) sets `enable_ip_whitelisting: true` with
`allowed_ips: [127.0.0.1]`. `kubectl port-forward` satisfies this — the connection
reaches the Gateway from inside its own network namespace, so the source address really
is `127.0.0.1`. Nothing else does: ingress, `minikube tunnel`, a published NodePort or
a load balancer all present a different source IP and get a 403. This is a concrete
reason the post should prefer port-forward for the walkthrough, and must tell readers
to widen `allowed_ips` the moment they move to anything else. `xffDepth: 1` is set in
the tyk-install values, so a load balancer that sends `X-Forwarded-For` will have the
real client IP evaluated.

## 8. Dashboard, Portal and credentials

- Dashboard on 3000/3001 is still right, over `http` not `https`.
- The post's Portal references should go: the recommended overlay disables the Developer
  Portal (it is unused by the demo, and it costs a third licence plus ~500Mi per cluster).
- Add the login, which `tyk-install`'s `.env.example` fixes at `default@example.com` /
  `topsecret123`. Also warn readers **not** to change `ADMIN_EMAIL` — the bootstrap job
  depends on it.
- Nice addition for an Operator-focused post: the Operator writes Tyk IDs back to the CR
  status, so a reader can go straight from Kubernetes to the policy ID:

  ```bash
  kubectl get securitypolicy standard-policy -n policies-staging -o jsonpath='{.status.pol_id}'
  kubectl get apidefinition  httpbin-api     -n httpbin-staging  -o jsonpath='{.status.latestTransaction.status}'
  ```

## 9. The CORS example

The post says "add CORS settings" without showing them. Give the block, since the CRD
field is `CORS` (capitalised) and readers will not guess it:

```yaml
spec:
  CORS:
    enable: true
    allowed_origins: ["*"]
    allowed_methods: [GET, POST, OPTIONS]
    allowed_headers: [Origin, Accept, Content-Type, Authorization]
    allow_credentials: false
    max_age: 24
    options_passthrough: false
    debug: false
```

And show the verification, including that production stays unchanged — that contrast is
the payoff of the overlay structure:

```bash
curl -i -H "Origin: https://example.com" -H "Authorization: Bearer $API_KEY" \
  http://localhost:8080/httpbin/get | grep -i access-control
```

## 10. Add a troubleshooting section

The post has none, and the Helm path surfaces failures the script used to swallow. The
cases readers actually hit are collected in
[`minikube-setup.md` → Troubleshooting](./minikube-setup.md#troubleshooting): missing
Operator licence, cert-manager not ready, missing CRDs, `<pending>` LoadBalancer IPs,
403 from the IP allowlist, arm64 httpbin, and dropped port-forwards.

---

## Things that did *not* change

Useful to state, so the revision does not look larger than it is:

- The repository layout (`apps/`, `policies/`, `argocd/`) and every manifest in it.
- The app-of-apps pattern and both root applications.
- The Kustomize base/overlay split, and the namespace asymmetry it produces:
  staging lands in `httpbin-staging` / `policies-staging` (the staging
  `kustomization.yaml` sets `namespace: httpbin-staging`), production lands in
  `httpbin` / `policies` (the production overlay keeps the base namespace).
- The GitOps loop: edit a manifest, push, ArgoCD syncs, the Operator reconciles Tyk.
- The Tyk Operator watches all namespaces by default — the chart grants it a
  ClusterRole — which is what lets a `SecurityPolicy` in `policies-staging` reference an
  `ApiDefinition` in `httpbin-staging`.
