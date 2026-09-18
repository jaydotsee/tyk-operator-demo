# Running tyk-operator-demo on Minikube

A complete, self-contained walkthrough: two local Kubernetes clusters (staging and
production), each running the Tyk stack with Tyk Operator and ArgoCD, with this
repository driving APIs and security policies through GitOps.

The Tyk install assets live in
[**TykTechnologies/tyk-install**](https://github.com/TykTechnologies/tyk-install),
and the steps below are the `kubernetes/helm-self-managed` path from that repo
adapted for minikube: two minikube profiles, explicit Helm installs (Redis,
PostgreSQL, cert-manager, then the `tyk-stack` chart), and `kubectl port-forward`
for local access.

Two things worth reading before you start:

- **URLs are `http://`, not `https://`.** TLS is off in the `tyk-install` values
  (`global.tls.gateway: false`).
- **You need two licence keys**, not one: a Dashboard licence and an Operator
  licence. Both come from the same free trial at <https://tyk.io/sign-up/>, but the
  Operator licence is issued separately — ask for it when you sign up, or request it
  from the Tyk team. Without it the Operator pod starts and then fails its licence check.

---

## Prerequisites

### Tools

| Tool | Version | Notes |
| --- | --- | --- |
| [minikube](https://minikube.sigs.k8s.io/docs/start/) | 1.32+ | Docker driver assumed |
| `kubectl` | 1.28+ | |
| [Helm](https://helm.sh/docs/intro/install/) | 3.12+ | required by the Tyk charts |
| Docker | any recent | must be running before `minikube start` |
| `git`, `curl` | | |

### Machine resources

You are running **two** full Tyk stacks plus two ArgoCD installs. Each profile is
started with 4 CPUs and 6 GB below, so give your container runtime at least
**8 CPUs and 14 GB** total (Docker Desktop: Settings → Resources).

If that is more than you have, run the two clusters **one at a time**: complete the
staging half, `minikube stop -p staging`, then do production. The demo still works;
you just cannot compare the two side by side.

### Licences

Get a free trial at <https://tyk.io/sign-up/>. You need:

- **Dashboard licence** → `TYK_LICENSE_KEY`
- **Tyk Operator licence** → `TYK_OPERATOR_LICENSE`

### Apple Silicon and other arm64 machines

Nothing to do — this repository's manifests use
[`mccutchen/go-httpbin`](https://github.com/mccutchen/go-httpbin), which publishes both
`linux/amd64` and `linux/arm64`, so the demo runs as-is on Apple Silicon.

---

## Port map

Each cluster gets its own set of host ports.

| Service | staging | production |
| --- | --- | --- |
| Tyk Dashboard | `http://localhost:3000` | `http://localhost:3001` |
| Tyk Gateway | `http://localhost:8080` | `http://localhost:8081` |
| ArgoCD UI | `https://localhost:8000` | `https://localhost:8001` |

There are two ways to make those ports real, and the choice matters if you intend to
put your own load balancer in front:

- **Mode A — `kubectl port-forward`.** Nothing to install, but each forward is a
  foreground process that binds `127.0.0.1` only and dies when the pod restarts.
  Fine for working through this guide; a poor backend for a load balancer.
- **Mode B — an NGINX ingress controller per cluster.** One stable published port per
  cluster, with the Gateway, Dashboard and ArgoCD behind it and routed by hostname. No
  tunnel process, survives pod restarts, and uses the Tyk charts' own `ingress:` blocks
  rather than anything bolted on. This is the one to pick if you want your own load
  balancer in front. See
  [Exposing both clusters to the host](#exposing-both-clusters-to-the-host).

The guide below uses Mode A because it needs no decisions up front. Mode B keeps every
step intact but changes the addresses: services move behind hostnames on a shared port
(`http://dash.staging.test:8080` rather than `http://localhost:3000`), and its section
gives the mapping.

> **If you already know you want Mode B, read the [Hostnames](#hostnames) section and
> set up `/etc/hosts` before Step 3.** The per-step command variants (the `minikube
> start` line, the ingress-nginx install, and so on) appear inline as you go, behind
> "Using Mode B?" `<details>` blocks — you don't need to pre-read them. The `minikube
> start --ports` mapping they rely on is applied only when the profile's container is
> first created, so switching to Mode B later means deleting and recreating the
> profile.

---

## Step 1 — Fork and clone this repository

ArgoCD pulls manifests from Git, so it must point at a repository **you can push to**.

```bash
# Fork https://github.com/TykTechnologies/tyk-operator-demo in the GitHub UI, then:
git clone https://github.com/YOUR_GITHUB_USERNAME/tyk-operator-demo.git
cd tyk-operator-demo
```

Repoint every ArgoCD `Application` at your fork:

```bash
grep -rl 'TykTechnologies/tyk-operator-demo' argocd/ \
  | xargs sed -i.bak "s#https://github.com/TykTechnologies/tyk-operator-demo#https://github.com/YOUR_GITHUB_USERNAME/tyk-operator-demo#g"
find argocd -name '*.bak' -delete

# Confirm the six manifests now point at your fork
grep -rn repoURL argocd/
```

Commit and push — ArgoCD reads Git, not your working copy:

```bash
git add argocd/
git commit -m "Point ArgoCD applications at my fork"
git push origin main
```

## Step 2 — Get the Tyk install assets

```bash
cd ..
git clone https://github.com/TykTechnologies/tyk-install.git
cd tyk-install/kubernetes/helm-self-managed
```

Create your `.env`:

```bash
cp .env.example .env
```

Edit `.env` and set both licences. Quote the values — licence keys are long:

```bash
TYK_LICENSE_KEY="<your dashboard licence>"
TYK_OPERATOR_LICENSE="<your operator licence>"
```

Leave everything else at its defaults. In particular, do **not** change
`ADMIN_EMAIL=default@example.com` — the bootstrap job expects it. The defaults also
give you the Dashboard login you will use later:

- email `default@example.com`
- password `topsecret123`

`TYK_PORTAL_LICENSE` can stay empty; this guide turns the Developer Portal off.

---

## Step 3 — Build the staging cluster

Run every command in this step from `tyk-install/kubernetes/helm-self-managed`.

### 3.1 Start the profile

```bash
minikube start -p staging --cpus=4 --memory=6144
kubectl config use-context staging
kubectl get nodes
```

<details>
<summary>Using Mode B (ingress)? Use this <code>minikube start</code> line instead.</summary>
The port mapping it adds can only be set when the profile's container is first
created, so getting it wrong here costs you a rebuild.

```bash
minikube start -p staging --cpus=4 --memory=6144 \
  --listen-address=127.0.0.1 --ports=8080:30080 --ports=8443:30443
```

Then continue with `kubectl config use-context staging` and `kubectl get nodes` above.

Three things to know about `--ports`:

- **Docker and Podman drivers only.** Other drivers have no node container to publish
  ports from.
- **It applies only at container creation.** Re-running `minikube start` with different
  `--ports` on an existing profile silently ignores them; you have to
  `minikube delete -p staging` and start again. This is why the ingress sits behind it
  and not the individual services.
- **`--listen-address` chooses the bind address.** `127.0.0.1` is right when your load
  balancer runs on the same machine. For other machines use `0.0.0.0` — and read
  [Before you bind to 0.0.0.0](#before-you-bind-to-0000) first.

</details>

The minikube profile name becomes the kubectl context name, so `staging` and
`production` are the context names the rest of this guide uses.

### 3.2 Create the namespace and secrets

```bash
source .env

kubectl create namespace tyk

kubectl create secret generic tyk-conf \
  --namespace tyk \
  --from-literal=APISecret=$TYK_API_SECRET \
  --from-literal=AdminSecret=$TYK_ADMIN_SECRET \
  --from-literal=DashLicense=$TYK_LICENSE_KEY \
  --from-literal=OperatorLicense=$TYK_OPERATOR_LICENSE \
  --from-literal=DevPortalLicense=$TYK_PORTAL_LICENSE \
  --from-literal=adminUserFirstName=$ADMIN_FIRST_NAME \
  --from-literal=adminUserLastName=$ADMIN_LAST_NAME \
  --from-literal=adminUserEmail=$ADMIN_EMAIL \
  --from-literal=adminUserPassword=$ADMIN_PASSWORD \
  --from-literal=DashDatabaseConnectionString="$DashDatabaseConnectionString" \
  --from-literal=DevPortalDatabaseConnectionString="$DevPortalDatabaseConnectionString"
```

Sanity-check that the two licence keys actually landed — an empty `OperatorLicense`
is the single most common cause of a broken demo:

```bash
kubectl get secret tyk-conf -n tyk -o jsonpath='{.data.DashLicense}'     | base64 -d | head -c 20; echo
kubectl get secret tyk-conf -n tyk -o jsonpath='{.data.OperatorLicense}' | base64 -d | head -c 20; echo
```

> The `secrets-tyk-tyk-dev-portal` secret from the tyk-install README is not needed
> here, because the Developer Portal is disabled.

### 3.3 Install PostgreSQL, Redis and cert-manager

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install tyk-postgres bitnami/postgresql \
  --set image.repository=bitnamilegacy/postgresql \
  --namespace tyk \
  --set auth.username=$POSTGRES_USER \
  --set auth.password=$POSTGRES_PASSWORD \
  --set auth.database=$POSTGRES_DB \
  --set primary.initdb.scripts."init\.sql"="CREATE DATABASE portal;" \
  --set primary.persistence.size=20Gi \
  --version 12.12.10

helm install tyk-redis oci://registry-1.docker.io/bitnamicharts/redis \
  --set image.repository=bitnamilegacy/redis \
  --namespace tyk \
  --set auth.enabled=false \
  --version 19.0.2

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n tyk --timeout=600s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=redis      -n tyk --timeout=600s
```

cert-manager is **required** — Tyk Operator's admission webhook gets its TLS
certificate from it, and the Operator will not become ready without it:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.17.4 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

<details>
<summary>Using Mode B (ingress)? Install ingress-nginx before the Tyk stack.</summary>

Do this before installing the Tyk stack, so the admission webhook is up when the
chart's Ingress resources are created.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443 \
  --set controller.config.use-forwarded-headers=true

kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/component=controller -n ingress-nginx --timeout=300s
```

Two of those flags carry weight:

- `controller.service.type=NodePort` with pinned `nodePorts` — the chart defaults to
  `LoadBalancer`, which sits at `<pending>` on Minikube exactly like the Tyk services
  do. Unlike tyk-charts, this chart *does* let you pin the numbers, which is what makes
  the `--ports` mapping above possible.
- `controller.config.use-forwarded-headers=true` — tells nginx to trust an
  `X-Forwarded-For` header from your load balancer instead of overwriting it. Without
  it, everything upstream sees your load balancer as the client.

> The `minikube addons enable ingress` addon would also work, and needs no Helm. It
> runs the same controller with `hostNetwork` and `hostPort: 80/443`, so you would
> publish `--ports=8080:80` instead. The Helm install is preferred here because it lets
> you set `use-forwarded-headers` at install time, pins the version, and is the same
> command you would run on a real cluster.

</details>

### 3.4 Install the Tyk stack (including Tyk Operator)

This layers this repository's minikube overlay on top of the `tyk-install` values,
so the `tyk-install` repo stays the source of truth for the Tyk configuration:

```bash
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo update

helm install tyk tyk-helm/tyk-stack \
  --namespace tyk \
  --values values.yaml \
  --values ../../../tyk-operator-demo/docs/tyk-minikube.values.yaml
```

> Adjust the second `--values` path if you cloned the two repositories somewhere
> other than side by side. What it changes is documented at the top of
> [`tyk-minikube.values.yaml`](./tyk-minikube.values.yaml): ClusterIP services,
> Developer Portal off, smaller resource requests.

<details>
<summary>Using Mode B (ingress)? Add a third <code>--values</code> layer for ingress.</summary>

The shared overlay already sets both services to `ClusterIP`, which is what the
tyk-install README asks for before enabling ingress, so the layers compose without
contradiction.

```bash
helm install tyk tyk-helm/tyk-stack --namespace tyk \
  --values values.yaml \
  --values ../../../tyk-operator-demo/docs/tyk-minikube.values.yaml \
  --values ../../../tyk-operator-demo/docs/ingress/tyk-ingress.staging.values.yaml
```

</details>

Wait for everything to settle (first run pulls several images — allow a few minutes):

```bash
kubectl get pods -n tyk -w   # Ctrl+C once all pods are Running or Completed
```

You should end up with `gateway-*`, `dashboard-*`, `tyk-pump-*`, `tyk-postgres-*`,
`tyk-redis-*` and `tyk-tyk-operator-*`, plus completed bootstrap jobs.

Confirm the Operator is up and that its CRDs and credentials exist:

```bash
kubectl get pods -n tyk -l control-plane=tyk-operator-controller-manager
kubectl get crd | grep tyk.tyk.io
kubectl get secret tyk-operator-conf -n tyk
```

`tyk-operator-conf` is created by the chart's bootstrap job from the `OperatorLicense`
key in `tyk-conf`, together with the Dashboard URL, org ID and a Dashboard API key.
There is nothing to wire up by hand.

### 3.5 Port-forward the Dashboard and Gateway

```bash
kubectl port-forward -n tyk svc/dashboard-svc-tyk-tyk-dashboard 3000:3000 &
kubectl port-forward -n tyk svc/gateway-svc-tyk-tyk-gateway     8080:8080 &

curl http://localhost:8080/hello
```

> Using Mode B (ingress)? Skip these port-forwards — the Gateway and Dashboard are
> reached through the ingress controller instead, at `http://gw.staging.test:8080` and
> `http://dash.staging.test:8080`.

Log in to <http://localhost:3000> (Mode B: <http://dash.staging.test:8080>) with
`default@example.com` / `topsecret123`.

### 3.6 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts

kubectl wait --for=condition=available deployment --all -n argocd --timeout=600s

kubectl port-forward svc/argocd-server -n argocd 8000:443 &

kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
```

ArgoCD UI: <https://localhost:8000>, user `admin`, password as printed above. Your
browser will warn about the self-signed certificate — that is expected.

<details>
<summary>Using Mode B (ingress)? Put ArgoCD behind the ingress instead of port-forwarding it.</summary>

Skip only the `kubectl port-forward svc/argocd-server` command above — still run the
`kubectl get secret argocd-initial-admin-secret` command from that same block, since
you need its output for the password below. ArgoCD serves its UI over TLS on its own
port by default. Behind an ingress that is a redirect loop, so switch it to plain HTTP
first and let the ingress own TLS:

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd rollout status deployment argocd-server

kubectl apply -f docs/ingress/argocd-ingress.staging.yaml     # or .production.yaml
```

> `server.insecure` means ArgoCD stops terminating TLS itself, not that TLS is
> abolished — the ingress or your load balancer terminates it instead. On this local
> setup nothing does, so the UI is plain HTTP over loopback. Terminate TLS at your load
> balancer before this leaves the machine.

ArgoCD UI: <http://argocd.staging.test:8080>, user `admin`, password from the
`kubectl get secret argocd-initial-admin-secret` command above.

If you use the `argocd` CLI against this, pass `--grpc-web`; its gRPC transport does
not survive an HTTP/1.1 ingress hop otherwise:

```bash
argocd login argocd.staging.test --grpc-web
```

</details>

---

## Step 4 — Build the production cluster

Identical to Step 3 with three substitutions:

- profile and context `production` instead of `staging`
- Dashboard `3001`, Gateway `8081`, ArgoCD `8001`
- nothing else changes — same `.env`, same secret, same Helm values

```bash
minikube start -p production --cpus=4 --memory=6144
kubectl config use-context production
```

<details>
<summary>Using Mode B (ingress)? Use this <code>minikube start</code> line instead.</summary>

```bash
minikube start -p production --cpus=4 --memory=6144 \
  --listen-address=127.0.0.1 --ports=8081:30080 --ports=8444:30443
```

Then continue with `kubectl config use-context production` above. See
[3.1](#31-start-the-profile) for what `--ports` and `--listen-address` do.

</details>

Then repeat sections **3.2** and **3.3** verbatim.

<details>
<summary>Using Mode B (ingress)? Install ingress-nginx in this cluster too.</summary>

Same command as staging's [3.3 Mode B block](#33-install-postgresql-redis-and-cert-manager),
run once per cluster, against the `production` context:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443 \
  --set controller.config.use-forwarded-headers=true

kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/component=controller -n ingress-nginx --timeout=300s
```

</details>

Then repeat section **3.4** verbatim.

<details>
<summary>Using Mode B (ingress)? Add a third <code>--values</code> layer for ingress.</summary>

```bash
helm install tyk tyk-helm/tyk-stack --namespace tyk \
  --values values.yaml \
  --values ../../../tyk-operator-demo/docs/tyk-minikube.values.yaml \
  --values ../../../tyk-operator-demo/docs/ingress/tyk-ingress.production.values.yaml
```

</details>

Then repeat sections **3.5** and **3.6** verbatim — including installing ArgoCD itself
into this cluster — using these ports instead:

```bash
kubectl port-forward -n tyk svc/dashboard-svc-tyk-tyk-dashboard 3001:3000 &
kubectl port-forward -n tyk svc/gateway-svc-tyk-tyk-gateway     8081:8080 &
kubectl port-forward svc/argocd-server -n argocd                8001:443  &
```

> Note the mapping: the local port changes, the in-cluster port does not. The
> Dashboard still listens on 3000 and the Gateway on 8080 inside every cluster.

<details>
<summary>Using Mode B (ingress)? Skip the port-forwards above and put ArgoCD behind the ingress.</summary>

The Gateway and Dashboard are already reached through the ingress controller, at
`http://gw.prod.test:8081` and `http://dash.prod.test:8081` — no port-forward needed
for either. For ArgoCD, switch it to plain HTTP and apply the production ingress:

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd rollout status deployment argocd-server

kubectl apply -f docs/ingress/argocd-ingress.production.yaml
```

ArgoCD UI: <http://argocd.prod.test:8081>, same credentials as staging's.

```bash
argocd login argocd.prod.test --grpc-web
```

</details>

---

<details>
<summary>Using Mode B (ingress)? Verify both clusters before moving on.</summary>

```bash
# The node container is publishing the ports
docker port staging
docker port production

# Ingress resources exist and have an address
kubectl --context staging get ingress -A
kubectl --context production get ingress -A

# End to end, without a load balancer in front
curl -H 'Host: gw.staging.test' http://localhost:8080/hello
curl -H 'Host: gw.prod.test'    http://localhost:8081/hello
```

Then open `http://dash.staging.test:8080` (Dashboard) and
`http://argocd.staging.test:8080` (ArgoCD), and the `:8081` equivalents for production.

Every URL elsewhere in this guide still works; only the host and port change. The
Dashboard is `http://dash.staging.test:8080` rather than `http://localhost:3000`, and
the Gateway `http://gw.staging.test:8080/httpbin/get` rather than
`http://localhost:8080/httpbin/get`.

</details>

---

## Step 5 — Deploy the demo through ArgoCD

Order matters: the Tyk stack must already be installed, because it is what provides
the `TykOasApiDefinition` and `SecurityPolicy` CRDs that these manifests use.

From your `tyk-operator-demo` checkout:

```bash
cd ../../../tyk-operator-demo   # or wherever your clone lives

kubectl config use-context staging
kubectl apply -f argocd/staging-apps.yaml

kubectl config use-context production
kubectl apply -f argocd/prod-apps.yaml
```

Each root application is an app-of-apps: it syncs `argocd/staging` (or `argocd/prod`),
which in turn creates the `httpbin-*` and `policies-*` applications that deploy the
workload, the `TykOasApiDefinition` and the `SecurityPolicy` resources.

### Verify

```bash
kubectl config use-context staging

kubectl get applications -n argocd
kubectl get pods              -n httpbin-staging
kubectl get tykoasapidefinition -n httpbin-staging
kubectl get securitypolicy    -n policies-staging
```

The `TykOasApiDefinition` printer columns include a `SyncStatus`; it should read
`Successful` once the Operator has pushed the API to the Dashboard:

```bash
kubectl get tykoasapidefinition httpbin-api -n httpbin-staging \
  -o jsonpath='{.status.latestTransaction.status}{"\n"}'
```

The same API should now be visible in the Dashboard at <http://localhost:3000> under
**APIs**, and both policies under **Policies**.

For production the namespaces are `httpbin` and `policies` (not `-staging`), because
the production Kustomize overlay keeps the base namespace:

```bash
kubectl config use-context production
kubectl get tykoasapidefinition -n httpbin
kubectl get securitypolicy      -n policies
```

> If `policies-staging` or `policies-prod` shows a sync error on the first pass, it is
> almost always because the policy synced before the API it references. It resolves on
> the next reconcile; you can force it with
> `kubectl -n policies-staging annotate securitypolicy --all reconcile="$(date +%s)" --overwrite`.

<details>
<summary>Already deployed an earlier, Classic-based version of this demo?
Upgrading in place needs a manual cutover, not just a re-sync.</summary>

If `httpbin-*` already exists from before this change (a Classic `ApiDefinition`
named `httpbin-api`), simply pulling this update will not migrate it. Tyk derives
an API's internal ID from `namespace/name`, so the new `TykOasApiDefinition` — same
name, same namespace — collides with the still-live Classic one
(`409 Found API with the same id`), and its `SyncStatus` gets stuck on `Failed`.
ArgoCD's `prune: false` here means Git alone never deletes the old object, and Tyk
Operator refuses to delete a Classic API while any `SecurityPolicy` still names it
— **even after** you change that policy's `access_rights_array[].kind` to
`TykOasApiDefinition`; the dependency check matches by name and namespace only; it
does not look at `kind`.

Per environment, once you've committed and pushed this change:

```bash
# 1. Temporarily unlink the policies (fully clears the entry — a kind change alone
#    will not satisfy the check below)
kubectl patch securitypolicy standard-policy -n policies-staging \
  --type=json -p='[{"op":"replace","path":"/spec/access_rights_array","value":[]}]'
kubectl patch securitypolicy trial-policy -n policies-staging \
  --type=json -p='[{"op":"replace","path":"/spec/access_rights_array","value":[]}]'

# 2. Delete the live Classic API. If it hangs waiting on its finalizer, the
#    Operator's own retry backoff is just slow to notice the dependency cleared —
#    restarting its pod forces an immediate reconcile.
kubectl delete apidefinition httpbin-api -n httpbin-staging
kubectl delete pod -n tyk -l control-plane=tyk-operator-controller-manager

# 3. Confirm the OAS definition is now live
kubectl get tykoasapidefinition httpbin-api -n httpbin-staging

# 4. Re-apply the policy manifests from Git to restore the real access rights
kubectl apply -f policies/staging/standard-policy.yaml -f policies/staging/trial-policy.yaml
```

Repeat against production (`policies`/`httpbin` namespaces). Existing API keys do
not survive this cutover either way — see the note below.

</details>

> **Existing API keys stop working once the Classic API is replaced by the OAS
> one.** The OAS `TykOasApiDefinition` is a distinct API with its own internal Tyk
> API ID — it is not the same API the old key was issued against. If you are
> picking this demo back up after an earlier session, issue a fresh key in
> **Step 6** rather than reusing one you already have.

---

## Step 6 — Call the API

Both overlays' OpenAPI documents declare an `apiKey` security scheme on the
`Authorization` header, so every request to `/httpbin/*` requires it — a keyless
request is expected to fail, not a sign that something is broken:

```bash
curl -i http://localhost:8080/httpbin/get        # staging    -> 401
curl -i http://localhost:8081/httpbin/get        # production -> 401
```

```json
{"error": "Authorization field missing"}
```

Get the key from **Create a key** below, then pass it as `-H "Authorization: Bearer
$API_KEY"` on every request — including through the ingress hostnames
(`gw.staging.test`, `gw.prod.test`) in Mode B.

### Create a key

In the Dashboard (<http://localhost:3000> for staging):

1. **Keys** → **Add Key**
2. Choose the policy **Standard policy (Staging)**
3. **Create** — copy the generated key

Then:

```bash
export API_KEY=<paste the key>
curl -H "Authorization: Bearer $API_KEY" http://localhost:8080/httpbin/get
```

Mode B (ingress) — same header, ingress hostname instead of `localhost`:

```bash
curl -H "Authorization: Bearer $API_KEY" http://gw.staging.test:8080/httpbin/get
```

You should get an httpbin JSON response, and the request should appear under
**Monitoring → Activity** in the Dashboard.

Repeat against <http://localhost:3001> and port `8081` for production, using
**Standard policy (Production)**.

If you would rather find the policy from Kubernetes, the Operator writes the Tyk
policy ID back into the CR's status:

```bash
kubectl get securitypolicy standard-policy -n policies-staging \
  -o jsonpath='{.status.pol_id}{"\n"}'
```

---

## Step 7 — Change something, GitOps style

This is the point of the demo: change Git, not the cluster. This time the change
makes the Gateway itself start **rejecting** requests it previously accepted —
production will require callers to identify themselves with an `X-Client-Id`
header, while staging keeps working exactly as before.

Tyk OAS can enforce this natively, with no plugin: a required header `parameter`
on the OpenAPI operation, plus the `validateRequest` middleware that rejects a
request failing that schema. Edit the `/get` operation inside the OpenAPI document
in `apps/httpbin/overlays/prod/httpbin-oas.yaml`'s ConfigMap — add the parameter:

```json
    "paths": {
      "/get": {
        "get": {
          "operationId": "httpbinGet",
          "parameters": [
            {
              "in": "header",
              "name": "X-Client-Id",
              "required": true,
              "schema": {
                "type": "string"
              }
            }
          ],
          "responses": {
            "200": {
              "description": "Success"
            }
          }
        }
      }
    },
```

and turn on `validateRequest` for that operation under `x-tyk-api-gateway`:

```json
      "middleware": {
        "operations": {
          "httpbinGet": {
            "validateRequest": {
              "enabled": true,
              "errorResponseCode": 400
            }
          }
        }
      }
```

Any non-empty `X-Client-Id` value satisfies the check — this is about presence, not
a specific value. Commit and push:

```bash
git add apps/httpbin/overlays/prod/httpbin-oas.yaml
git commit -m "Require X-Client-Id on the production httpbin API"
git push origin main
```

ArgoCD polls Git roughly every three minutes; click **Refresh** in the UI if you do
not want to wait. Once it syncs, the Operator pushes the change to the Gateway.
Production now rejects an authenticated request missing the header:

```bash
curl -i -H "Authorization: Bearer $PROD_API_KEY" http://localhost:8081/httpbin/get
```

```
HTTP/1.1 400 Bad Request
```

Add the header and it succeeds again:

```bash
curl -i -H "Authorization: Bearer $PROD_API_KEY" -H "X-Client-Id: my-client" \
  http://localhost:8081/httpbin/get
```

```
HTTP/1.1 200 OK
```

Staging's `httpbin-oas.yaml` was never touched, so the equivalent staging request —
still with no `X-Client-Id` — keeps working unaffected:

```bash
curl -i -H "Authorization: Bearer $API_KEY" http://localhost:8080/httpbin/get
```

```
HTTP/1.1 200 OK
```

That's the point made concrete: a Git-only change to production's ConfigMap changed
what the Gateway accepts, and staging — sharing nothing but the same Kustomize base
— was never at risk of being affected by it.

---

## Exposing both clusters to the host

Everything above reaches the clusters through `kubectl port-forward`. That is fine for
following the guide, but it is a bad foundation for your own load balancer: each
forward binds `127.0.0.1` only, is a foreground process you have to keep alive, and
drops as soon as the pod behind it restarts.

This section replaces those forwards with an NGINX ingress controller per cluster, so
each cluster has **one** stable entry point that your load balancer can treat as an
ordinary backend.

### Why this shape

- **The Tyk charts already support it.** `tyk-install`'s `values.yaml` ships `ingress:`
  blocks for the Gateway and Dashboard, switched off, with commented cloud examples.
  Turning them on is the documented "NGINX Ingress" path from the tyk-install README,
  not a workaround. (Contrast `service.type: NodePort`, which the charts *cannot*
  express usefully — their service templates render no `nodePort` field, so you get a
  random port in 30000-32767.)
- **One published host port per cluster, set once.** `minikube start --ports` applies
  only when the profile's container is first created, so anything published that way is
  frozen for the life of the profile. Publish the ingress controller and you can expose
  any number of services afterwards by adding Ingress resources — no profile rebuild.
- **Your load balancer gets two backends**, not six.

```
your LB :80  ──host: gw.staging.test──▶  127.0.0.1:8080 ─▶ ingress-nginx (staging)  ─▶ Gateway / Dashboard / ArgoCD
             ──host: gw.prod.test─────▶  127.0.0.1:8081 ─▶ ingress-nginx (production) ─▶ Gateway / Dashboard / ArgoCD
```

### Hostnames

Routing is by `Host` header, so each service needs a name. This guide uses the `.test`
TLD, which [RFC 6761](https://www.rfc-editor.org/rfc/rfc6761#section-6.2) reserves for
exactly this. (Avoid `.local` — it is claimed by mDNS and causes resolution stalls on
macOS.)

| | staging | production |
| --- | --- | --- |
| Tyk Gateway | `gw.staging.test` | `gw.prod.test` |
| Tyk Dashboard | `dash.staging.test` | `dash.prod.test` |
| ArgoCD | `argocd.staging.test` | `argocd.prod.test` |

Point them at whatever address your load balancer listens on. If it runs on this
machine, that is loopback:

```bash
sudo tee -a /etc/hosts <<'EOF'
127.0.0.1 gw.staging.test dash.staging.test argocd.staging.test
127.0.0.1 gw.prod.test    dash.prod.test    argocd.prod.test
EOF
```

If the load balancer is on another host, put **its** address here instead, and start
the Minikube profiles with `--listen-address=0.0.0.0` so it can reach them — see
[Before you bind to 0.0.0.0](#before-you-bind-to-0000). The names must resolve for
whoever is *originating* the request; the load balancer itself talks to the clusters by
IP and port, and never resolves them.

> Prefer not to touch `/etc/hosts`? Swap every `*.test` name for the equivalent
> `*.127.0.0.1.nip.io` — those resolve to loopback without local configuration, at the
> cost of a DNS lookup that fails offline.

### How a request finds the right component

Nothing in this chain routes on IP or port past the first hop. Everything after the
load balancer is decided by the `Host` header, which is why it has to survive intact.

```
  http://dash.staging.test/
      │
      │  1. DNS / /etc/hosts:  dash.staging.test -> 127.0.0.1
      ▼
  your load balancer, :80
      │  2. routes on Host: *.staging.test -> 127.0.0.1:8080
      │     AND passes Host through unchanged
      ▼
  staging ingress-nginx  (node :30080, published to host :8080)
      │  3. matches Host against its Ingress rules
      │       gw.staging.test     -> gateway-svc-tyk-tyk-gateway:8080
      │       dash.staging.test   -> dashboard-svc-tyk-tyk-dashboard:3000
      │       argocd.staging.test -> argocd-server:80
      ▼
  Tyk Dashboard
         4. serves it, because dashboard.hostName says this is its own name
```

**Every component's name has to be spelled the same in each place it appears** — and
for the Dashboard that is three places, not one. All of these live in
[`docs/ingress/`](./ingress), and each must also match your load balancer's routing
rule and `/etc/hosts`:

| | staging | production | Set in |
| --- | --- | --- | --- |
| Gateway ingress rule | `gw.staging.test` | `gw.prod.test` | `tyk-gateway.gateway.ingress.hosts[].host` |
| Gateway URL shown in the Dashboard UI | `gw.staging.test` | `gw.prod.test` | `tyk-dashboard.dashboard.hostConfig.overrideHostname` |
| Dashboard ingress rule | `dash.staging.test` | `dash.prod.test` | `tyk-dashboard.dashboard.ingress.hosts[].host` |
| Dashboard's own hostname | `dash.staging.test` | `dash.prod.test` | `tyk-dashboard.dashboard.hostName` |
| ArgoCD ingress rule | `argocd.staging.test` | `argocd.prod.test` | `docs/ingress/argocd-ingress.<env>.yaml` |

The two Dashboard settings are easy to miss, because `tyk-install` ships values for a
different topology and they are not obviously part of "ingress config":

- **`dashboard.hostName`** (`TYK_DB_HOSTCONFIG_HOSTNAME`) defaults to
  `tyk-dashboard.local`, and `tyk-install` also sets `hostConfig.enableHostNames: true`
  — so the Dashboard is doing hostname-based routing against a name you are not using.
  It has to be the name you actually reach it on.
- **`hostConfig.overrideHostname`** (`TYK_DB_HOSTCONFIG_GATEWAYHOSTNAME`) defaults to
  `tyk-gw.local`. This is the Gateway URL the Dashboard prints next to each API. Leave
  it and the UI will tell you your API is at `http://tyk-gw.local/httpbin`, which
  resolves to nothing.

The values files in this repo already set all of these consistently. If you rename a
host, change it in the values file, `/etc/hosts`, and your load balancer's rules
together.

> `tyk-gateway.gateway.hostName` looks like it belongs in this list. It does not — no
> template in the chart reads it, so its value has no effect.

### Host port map

Both clusters run ingress-nginx on the same NodePorts; the host mapping tells them
apart.

| | NodePort (both clusters) | Host port, staging | Host port, production |
| --- | --- | --- | --- |
| ingress HTTP | `30080` | `8080` | `8081` |
| ingress HTTPS | `30443` | `8443` | `8444` |

Without a load balancer you reach things at `http://dash.staging.test:8080`. Once your
load balancer listens on `:80` and forwards by hostname, the port disappears and
`http://dash.staging.test` works.

### Example: one load balancer in front of both clusters

Both clusters are now ordinary HTTP backends on the host, told apart by hostname. An
HAProxy fragment:

```haproxy
frontend clusters
    bind *:80
    option forwardfor                       # preserved by use-forwarded-headers
    acl is_staging hdr_end(host) -i .staging.test
    acl is_prod    hdr_end(host) -i .prod.test
    use_backend staging if is_staging
    use_backend prod    if is_prod

backend staging
    server s1 127.0.0.1:8080 check

backend prod
    server p1 127.0.0.1:8081 check
```

The equivalent in nginx:

```nginx
upstream staging { server 127.0.0.1:8080; }
upstream prod    { server 127.0.0.1:8081; }

server {
    listen 80;
    server_name ~^.+\.staging\.test$;
    location / {
        proxy_pass http://staging;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 80;
    server_name ~^.+\.prod\.test$;
    location / {
        proxy_pass http://prod;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Passing `Host` through unchanged is what lets the in-cluster ingress route; drop it and
every request lands on nginx's default backend as a 404. HAProxy preserves the original
`Host` by default, which is why the fragment above does not mention it; nginx does
**not** — `proxy_pass` rewrites `Host` to the upstream name unless you set
`proxy_set_header Host $host`, so that line is load-bearing.

For health checks, the Gateway serves `/hello` — but note it only answers under a
matching `Host`, so a check must send one:

```haproxy
backend staging
    option httpchk
    http-check send meth GET uri /hello hdr Host gw.staging.test
    server s1 127.0.0.1:8080 check
```

### Before you bind to 0.0.0.0

`--listen-address=0.0.0.0` puts the Tyk Dashboard and the ArgoCD UI on your network.
Both are administrative interfaces, and this demo ships them with a known default
password (`default@example.com` / `topsecret123`) over plain HTTP — and ArgoCD is now
running with `server.insecure` on top of that. On a trusted local network for a demo
that is a reasonable trade; on anything shared it is not. Keep
`--listen-address=127.0.0.1` and let a TLS-terminating load balancer on the same machine
be the only thing exposed, or at minimum change `ADMIN_PASSWORD` in `.env` before the
clusters are built.

### Why not `minikube tunnel`?

It looks like the natural fit, since it keeps the stock `tyk-install` values with
`service.type: LoadBalancer`. Two problems with two clusters: both profiles default to
the same service CIDR, so two concurrent tunnels fight over host routes; and the tunnel
is another long-lived foreground process, which is what this section exists to get rid
of. A published NodePort, set once at profile creation, has neither problem.

---

## Troubleshooting

### Operator pod is not ready

```bash
kubectl logs -n tyk -l control-plane=tyk-operator-controller-manager --tail=100
```

- `license` errors → `OperatorLicense` in `tyk-conf` was empty or wrong. Recreate the
  secret (Step 3.2) and `kubectl rollout restart deployment -n tyk`.
- webhook / certificate errors → cert-manager was not ready when the stack was
  installed. Install cert-manager, wait for it, then
  `helm upgrade tyk tyk-helm/tyk-stack -n tyk --values values.yaml --values <overlay>`.

### `no matches for kind "TykOasApiDefinition"`

The Tyk CRDs are missing. They ship with the `tyk-operator` subchart and are installed
by `helm install`, but Helm does **not** install or update CRDs on `helm upgrade`. Check:

```bash
kubectl get crd | grep tyk.tyk.io
```

If they are absent, apply them directly (match the version to the Operator image tag):

```bash
kubectl apply -f https://raw.githubusercontent.com/TykTechnologies/tyk-charts/refs/heads/main/tyk-operator-crds/crd-v1.4.2.yaml
```

### Services stuck on `<pending>` EXTERNAL-IP

You installed with the stock `tyk-install` values only. minikube has no cloud load
balancer. Add the overlay from Step 3.4, or run `minikube tunnel -p staging` in a
separate terminal.

### Bootstrap job fails waiting for the Developer Portal

`global.components.devPortal` and `tyk-bootstrap.bootstrap.devPortal` must agree. Both
are set to `false` in the overlay; if you re-enable the Portal, set both to `true`, add
a `TYK_PORTAL_LICENSE`, and create the `secrets-tyk-tyk-dev-portal` secret described in
the tyk-install README.

### Ingress returns 404, or the port is not reachable

A 404 from nginx with no other symptom almost always means the `Host` header did not
match any Ingress rule. Test the routing directly, bypassing DNS and your load balancer:

```bash
curl -H 'Host: gw.staging.test' http://localhost:8080/hello
```

If that works but the browser 404s, the `Host` header is being rewritten — a load
balancer that sets `Host` to the backend address rather than passing the original
through is the usual cause.

If nothing responds at all, work outwards from the cluster:

```bash
# 1. Is the node container publishing the port? A missing line means the profile was
#    created without --ports; it is ignored on an existing container, so you have to
#    `minikube delete -p <profile>` and start again.
docker port staging

# 2. Is the controller running, and did the Ingress resources register?
kubectl get pods -n ingress-nginx
kubectl get ingress -A

# 3. Does it work from inside the node, bypassing the host mapping?
minikube ssh -p staging -- curl -s -H 'Host: gw.staging.test' localhost:30080/hello
```

If step 3 works but the host cannot connect, check `--listen-address`: bound to
`127.0.0.1`, the ports are unreachable from other machines by design.

### Dashboard reachable, but the API URLs it shows are wrong

The UI prints something like `http://tyk-gw.local/httpbin` that resolves to nothing.
That is `hostConfig.overrideHostname` still carrying the `tyk-install` default. It is a
display value fed by `TYK_DB_HOSTCONFIG_GATEWAYHOSTNAME`, so nothing is broken in the
data path — but it means the values layer was not applied. Check what the pod actually
got:

```bash
kubectl -n tyk get deployment dashboard-tyk-tyk-dashboard \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="TYK_DB_HOSTCONFIG_GATEWAYHOSTNAME")].value}{"\n"}'
kubectl -n tyk get deployment dashboard-tyk-tyk-dashboard \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="TYK_DB_HOSTCONFIG_HOSTNAME")].value}{"\n"}'
```

Both should be your ingress hostnames (`gw.<env>.test` and `dash.<env>.test`). If they
are the `.local` defaults, the third `--values` layer was missing from `helm install` —
re-run it as `helm upgrade` with all three files.

### ArgoCD redirect loop behind the ingress

`server.insecure` was not set, so nginx is speaking HTTP to a backend still terminating
TLS. Apply the ConfigMap patch from the "Using Mode B?" block under
[3.6 Install ArgoCD](#36-install-argocd) (or the equivalent block in Step 4) and restart
`argocd-server`. Confirm it took:

```bash
kubectl -n argocd get configmap argocd-cmd-params-cm -o jsonpath='{.data.server\.insecure}'; echo
```

### Port-forward drops

`kubectl port-forward` dies when a pod restarts. Re-run it; nothing in the cluster is
lost. `curl http://localhost:8080/hello` is the quickest way to tell whether a forward
is still alive.

### Everything is slow or pods are `Pending`

Not enough CPU or memory. `kubectl describe node` will say `Insufficient cpu/memory`.
Give the profile more (`minikube stop -p staging` then start it with higher
`--cpus`/`--memory`), or run the clusters one at a time.

---

## Cleanup

```bash
minikube delete -p staging
minikube delete -p production
```

To tear down Tyk but keep a cluster:

```bash
helm uninstall tyk tyk-postgres tyk-redis -n tyk
helm uninstall ingress-nginx -n ingress-nginx --ignore-not-found
kubectl delete namespace tyk cert-manager argocd ingress-nginx
kubectl delete namespace httpbin-staging policies-staging httpbin policies --ignore-not-found
```

---

## Reference

- [tyk-install](https://github.com/TykTechnologies/tyk-install) — the install assets this guide is built on
- [tyk-install: Kubernetes self-managed README](https://github.com/TykTechnologies/tyk-install/blob/main/kubernetes/helm-self-managed/README.md)
- [tyk-install: standalone Tyk Operator README](https://github.com/TykTechnologies/tyk-install/blob/main/kubernetes/standalone-operator/README.md) — use this instead if you have an existing Dashboard
- [Tyk Operator documentation](https://tyk.io/docs/api-management/automations/operator)
- [Tyk Helm charts](https://github.com/TykTechnologies/tyk-charts)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/)
