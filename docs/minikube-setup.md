# Running tyk-operator-demo on Minikube

A complete, self-contained walkthrough: two local Kubernetes clusters (staging and
production), each running the Tyk stack with Tyk Operator and ArgoCD, with this
repository driving APIs and security policies through GitOps.

This guide replaces the environment-setup half of the blog post
[*A practical guide: using Tyk Operator, ArgoCD and Kustomize*](https://tyk.io/blog/a-practical-guide-using-tyk-operator-argocd-and-kustomize/),
which is built on the archived `tyk-k8s-demo` repository and its `./up.sh` script.
The Tyk install assets now live in
[**TykTechnologies/tyk-install**](https://github.com/TykTechnologies/tyk-install),
and the steps below are the `kubernetes/helm-self-managed` path from that repo
adapted for minikube. See [`blog-update.md`](./blog-update.md) for a
step-by-step map of what changed and why.

---

## What replaced `./up.sh`

The archived demo wrapped everything in one script. There is no equivalent script
in `tyk-install`; installation is explicit Helm. Here is the mapping:

| `tyk-k8s-demo` did this | You now do this |
| --- | --- |
| `cp .env.example .env`, set `LICENSE` | `cp .env.example .env` in `tyk-install/kubernetes/helm-self-managed`, set `TYK_LICENSE_KEY` **and** `TYK_OPERATOR_LICENSE` |
| `./up.sh --deployments operator tyk-stack` | `helm install` for Redis, PostgreSQL and cert-manager, then `helm install tyk tyk-helm/tyk-stack` |
| Wired the Operator to the Dashboard for you | The `tyk-stack` chart's bootstrap job creates the `tyk-operator-conf` secret automatically |
| Exposed services over minikube ingress on HTTPS | `kubectl port-forward` over HTTP (see [Port map](#port-map)) |
| `--port-offset 1` for the second cluster | A second minikube profile plus different local port-forward ports |

Two consequences worth reading before you start:

- **URLs are `http://`, not `https://`.** The blog's `curl https://localhost:8080/...`
  becomes `curl http://localhost:8080/...`. TLS is off in the `tyk-install` values
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

### Apple Silicon / arm64

The demo deploys `kennethreitz/httpbin:latest`, which is published for `linux/amd64`
only. On an arm64 machine the pod will crash with `exec format error`. See
[Troubleshooting → httpbin pod crashes on Apple Silicon](#httpbin-pod-crashes-on-apple-silicon)
for a two-line fix.

---

## Port map

Each cluster gets its own set of host ports. This is what replaces the old
`--port-offset 1` flag.

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
- **Mode B — published NodePorts.** Fixed addresses served by the Minikube node
  container itself. No tunnel process, survives pod restarts, and can be bound to an
  address other than loopback. This is the one to pick if you want your own load
  balancer in front. See
  [Exposing both clusters to the host](#exposing-both-clusters-to-the-host).

The guide below uses Mode A because it needs no decisions up front. Mode B changes
only *how* the same host ports are served, so every URL in this guide stays valid
either way.

> **If you already know you want Mode B, read that section before Step 3.** The
> `minikube start --ports` mapping it relies on is applied only when the profile's
> container is first created, so switching later means deleting and recreating the
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

> Planning to front this with your own load balancer? Use the `minikube start` line
> from [B1](#b1-start-each-profile-with-the-port-mapping) instead — the port mapping it
> adds can only be set when the profile's container is first created.

The minikube profile name becomes the kubectl context name, so `staging` and
`production` are the context names the rest of this guide (and the blog) uses.

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

### 3.4 Install the Tyk stack (including Tyk Operator)

This is the step that replaces `./up.sh --deployments operator tyk-stack`. It layers
this repository's minikube overlay on top of the `tyk-install` values, so the
`tyk-install` repo stays the source of truth for the Tyk configuration:

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

> Using published NodePorts instead? Skip this — apply
> [`docs/nodeport-services.yaml`](./nodeport-services.yaml) once per cluster and the
> same URLs work without a forward. See
> [B2](#b2-apply-the-nodeport-services).

Log in to <http://localhost:3000> with `default@example.com` / `topsecret123`.

### 3.6 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available deployment --all -n argocd --timeout=600s

kubectl port-forward svc/argocd-server -n argocd 8000:443 &

kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
```

ArgoCD UI: <https://localhost:8000>, user `admin`, password as printed above. Your
browser will warn about the self-signed certificate — that is expected.

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

Then repeat sections **3.2**, **3.3** and **3.4** verbatim, and use these ports for
**3.5** and **3.6**:

```bash
kubectl port-forward -n tyk svc/dashboard-svc-tyk-tyk-dashboard 3001:3000 &
kubectl port-forward -n tyk svc/gateway-svc-tyk-tyk-gateway     8081:8080 &
kubectl port-forward svc/argocd-server -n argocd                8001:443  &
```

> On the NodePort path these forwards are unnecessary: the production profile's
> `--ports` mapping already puts the same services on 3001, 8081 and 8001.

> Note the mapping: the local port changes, the in-cluster port does not. The
> Dashboard still listens on 3000 and the Gateway on 8080 inside every cluster.

---

## Step 5 — Deploy the demo through ArgoCD

Order matters: the Tyk stack must already be installed, because it is what provides
the `ApiDefinition` and `SecurityPolicy` CRDs that these manifests use.

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
workload, the `ApiDefinition` and the `SecurityPolicy` resources.

### Verify

```bash
kubectl config use-context staging

kubectl get applications -n argocd
kubectl get pods            -n httpbin-staging
kubectl get apidefinition   -n httpbin-staging
kubectl get securitypolicy  -n policies-staging
```

The `ApiDefinition` printer columns include a `SyncStatus`; it should read
`Successful` once the Operator has pushed the API to the Dashboard:

```bash
kubectl get apidefinition httpbin-api -n httpbin-staging \
  -o jsonpath='{.status.latestTransaction.status}{"\n"}'
```

The same API should now be visible in the Dashboard at <http://localhost:3000> under
**APIs**, and both policies under **Policies**.

For production the namespaces are `httpbin` and `policies` (not `-staging`), because
the production Kustomize overlay keeps the base namespace:

```bash
kubectl config use-context production
kubectl get apidefinition  -n httpbin
kubectl get securitypolicy -n policies
```

> If `policies-staging` or `policies-prod` shows a sync error on the first pass, it is
> almost always because the policy synced before the API it references. It resolves on
> the next reconcile; you can force it with
> `kubectl -n policies-staging annotate securitypolicy --all reconcile="$(date +%s)" --overwrite`.

---

## Step 6 — Call the API

Both overlays set `use_standard_auth: true`, so a keyless request must be rejected:

```bash
curl -i http://localhost:8080/httpbin/get        # staging    -> 401
curl -i http://localhost:8081/httpbin/get        # production -> 401
```

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

This is the point of the demo: change Git, not the cluster.

Enable CORS on the staging API by editing
`apps/httpbin/overlays/staging/api_config.yaml`:

```yaml
apiVersion: tyk.tyk.io/v1alpha1
kind: ApiDefinition
metadata:
  name: not-important
spec:
  tags:
    - staging
  detailed_tracing: true
  CORS:
    enable: true
    allowed_origins:
      - "*"
    allowed_methods:
      - GET
      - POST
      - OPTIONS
    allowed_headers:
      - Origin
      - Accept
      - Content-Type
      - Authorization
    allow_credentials: false
    max_age: 24
    options_passthrough: false
    debug: false
```

Commit and push:

```bash
git add apps/httpbin/overlays/staging/api_config.yaml
git commit -m "Enable CORS on the staging httpbin API"
git push origin main
```

ArgoCD polls Git roughly every three minutes; click **Refresh** in the UI if you do
not want to wait. Once it syncs, the Operator pushes the change to the Gateway:

```bash
curl -i -H "Origin: https://example.com" -H "Authorization: Bearer $API_KEY" \
  http://localhost:8080/httpbin/get | grep -i access-control
```

You should see `Access-Control-Allow-Origin` in the response headers. Production is
untouched — that is Kustomize overlays doing their job:

```bash
curl -i -H "Origin: https://example.com" -H "Authorization: Bearer $PROD_API_KEY" \
  http://localhost:8081/httpbin/get | grep -i access-control   # no CORS headers
```

---

## Exposing both clusters to the host

Everything above reaches the clusters through `kubectl port-forward`. That is fine for
following the guide, but it is a bad foundation for your own load balancer: each
forward binds `127.0.0.1` only, is a foreground process you have to keep alive, and
drops as soon as the pod behind it restarts.

This section replaces those forwards with published NodePorts — fixed addresses served
by the Minikube node container, with no tunnel process in the middle.

### Why not just set `service.type: NodePort` in the Helm values?

Because you cannot pin the port that way. The tyk-charts service templates render only
`type`, `port`, `targetPort`, `protocol` and `name` — there is no `nodePort` field. Ask
for NodePort and Kubernetes assigns a random port in 30000-32767, which is useless when
the host-side mapping has to know the number in advance.

So this repo ships [`docs/nodeport-services.yaml`](./nodeport-services.yaml): three
extra Services with pinned NodePorts that select the same pods as the chart's own
Services. The chart-managed Services are left alone, so `helm upgrade` cannot clobber
these.

| Service | NodePort (both clusters) | Host port, staging | Host port, production |
| --- | --- | --- | --- |
| Tyk Gateway | `30080` | `8080` | `8081` |
| Tyk Dashboard | `30300` | `3000` | `3001` |
| ArgoCD server | `30443` | `8000` | `8001` |

The NodePort numbers are the same in both clusters — they are separate clusters, so
there is no collision. The clusters are told apart on the host by the mapping below.

### B1. Start each profile with the port mapping

This **replaces** the `minikube start` commands in Steps 3.1 and 4.

```bash
minikube start -p staging --cpus=4 --memory=6144 \
  --listen-address=127.0.0.1 \
  --ports=8080:30080 --ports=3000:30300 --ports=8000:30443

minikube start -p production --cpus=4 --memory=6144 \
  --listen-address=127.0.0.1 \
  --ports=8081:30080 --ports=3001:30300 --ports=8001:30443
```

Three things to know about `--ports`:

- **Docker and Podman drivers only.** Other drivers have no node container to publish
  ports from.
- **It applies only when the profile's container is first created.** Re-running
  `minikube start` with different `--ports` on an existing profile silently ignores
  them. To change the mapping: `minikube delete -p staging` and start again.
- **`--listen-address` chooses the bind address.** `127.0.0.1` keeps the ports on the
  loopback interface, which is right when your load balancer runs on the same machine.
  Use `--listen-address=0.0.0.0` to reach them from other machines on your network —
  and read [Before you bind to 0.0.0.0](#before-you-bind-to-0000) first.

### B2. Apply the NodePort Services

Do this **after** the Tyk stack (Step 3.4) and ArgoCD (Step 3.6) are installed, in each
cluster:

```bash
kubectl config use-context staging
kubectl apply -f docs/nodeport-services.yaml

kubectl config use-context production
kubectl apply -f docs/nodeport-services.yaml
```

Skip the `kubectl port-forward` commands in Steps 3.5, 3.6 and 4 entirely — these
Services do that job now.

### B3. Verify

```bash
# The node container is publishing the ports
docker port staging
docker port production

# The Services picked up endpoints (an empty ENDPOINTS column means the selector
# matched no pods -- check the release name is `tyk`)
kubectl get svc,endpoints -n tyk      -l app.kubernetes.io/part-of=tyk-operator-demo
kubectl get svc,endpoints -n argocd   -l app.kubernetes.io/part-of=tyk-operator-demo

# End to end
curl http://localhost:8080/hello      # staging gateway
curl http://localhost:8081/hello      # production gateway
curl -k https://localhost:8000        # staging ArgoCD
```

### The staging IP allowlist will now reject you

This is the one thing that genuinely breaks when you move off `port-forward`, and it is
easy to misdiagnose as a broken deployment.

`apps/httpbin/overlays/staging/api_auth.yaml` sets:

```yaml
  enable_ip_whitelisting: true
  allowed_ips:
    - 127.0.0.1
```

`kubectl port-forward` satisfies that by accident — the connection reaches the Gateway
from inside its own network namespace, so the source really is `127.0.0.1`. Through a
NodePort it is not: the Gateway sees the address traffic arrived from on the Minikube
node's network, typically the Docker bridge gateway. Every request gets rejected.

Find the CIDR Minikube gave the profile (the Docker network is named after the profile):

```bash
docker network inspect staging --format '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
# e.g. 192.168.49.0/24   -- if the name does not resolve, find it with `docker network ls`
```

Then widen the allowlist in `apps/httpbin/overlays/staging/api_auth.yaml` — `allowed_ips`
accepts CIDR notation:

```yaml
  allowed_ips:
    - 127.0.0.1
    - 192.168.49.0/24    # substitute your own subnet
```

Commit and push; ArgoCD and the Operator will roll it out. To take the allowlist out of
the picture entirely while you are wiring up a load balancer, set
`enable_ip_whitelisting: false` instead.

**With a load balancer in front**, the Gateway sees *your load balancer's* address, not
the original client's — unless the load balancer sets `X-Forwarded-For`. The
`tyk-install` values set `httpsServerOptions.xffDepth: 1`, so Tyk will honour XFF when
it is present. Configure your load balancer to send it (`option forwardfor` in HAProxy,
`proxy_set_header X-Forwarded-For` in nginx) and allowlist the real client addresses;
otherwise allowlist the load balancer itself.

### Optional: plain HTTP from ArgoCD

`argocd-server` serves TLS on its container port, so the NodePort is HTTPS with a
self-signed certificate. If your load balancer terminates TLS and wants to talk plain
HTTP to the backend, switch ArgoCD to insecure mode:

```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
```

Do this in each cluster you want it in. Only do it when something in front is
terminating TLS — it disables it, it does not move it.

### Example: one load balancer in front of both clusters

With the mapping above, both clusters are ordinary TCP backends on the host. An HAProxy
fragment routing by hostname:

```haproxy
frontend tyk_gateways
    bind *:80
    option forwardfor                       # Tyk reads X-Forwarded-For (xffDepth: 1)
    acl is_staging hdr(host) -i gw.staging.example.com
    acl is_prod    hdr(host) -i gw.example.com
    use_backend gw_staging if is_staging
    use_backend gw_prod    if is_prod
    default_backend gw_staging

backend gw_staging
    option httpchk GET /hello
    server staging 127.0.0.1:8080 check

backend gw_prod
    option httpchk GET /hello
    server production 127.0.0.1:8081 check
```

The equivalent in nginx:

```nginx
upstream tyk_staging { server 127.0.0.1:8080; }
upstream tyk_prod    { server 127.0.0.1:8081; }

server {
    listen 80;
    server_name gw.staging.example.com;
    location / {
        proxy_pass http://tyk_staging;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}

server {
    listen 80;
    server_name gw.example.com;
    location / {
        proxy_pass http://tyk_prod;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}
```

`/hello` is the Gateway's health endpoint and is the right target for a health check.

### Before you bind to 0.0.0.0

`--listen-address=0.0.0.0` puts the Tyk Dashboard and the ArgoCD UI on your network.
Both are administrative interfaces, and this demo ships them with a known default
password (`default@example.com` / `topsecret123`) over plain HTTP. On a trusted local
network for a demo that is a reasonable trade; on anything shared it is not. Keep
`--listen-address=127.0.0.1` and let a load balancer on the same machine be the only
thing exposed, or at minimum change `ADMIN_PASSWORD` in `.env` before the clusters are
built.

### Why not `minikube tunnel`?

It looks like the natural fit, since it keeps the stock `tyk-install` values with
`service.type: LoadBalancer`. Two problems with two clusters: both profiles default to
the same service CIDR, so two concurrent tunnels fight over host routes; and the tunnel
is another long-lived foreground process, which is what this section exists to get rid
of. NodePorts published at profile creation have neither problem.

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

### `no matches for kind "ApiDefinition"`

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

### 403 from the staging Gateway

The staging overlay sets `enable_ip_whitelisting: true` with `allowed_ips: [127.0.0.1]`.
`kubectl port-forward` satisfies this because the connection reaches the Gateway from
inside its own network namespace, so it appears to come from `127.0.0.1`. Every other
access path — published NodePorts, minikube ingress, `minikube tunnel`, or your own load
balancer — presents a different source IP, and every request is rejected.

Fix: widen `allowed_ips` in `apps/httpbin/overlays/staging/api_auth.yaml`, or set
`enable_ip_whitelisting: false`. Full walkthrough, including how to find the right CIDR
and how `X-Forwarded-For` interacts with it, is in
[The staging IP allowlist will now reject you](#the-staging-ip-allowlist-will-now-reject-you).

### httpbin pod crashes on Apple Silicon

`kennethreitz/httpbin:latest` is `linux/amd64` only. Swap in the arm64-capable
`go-httpbin` by editing `apps/httpbin/base/deployment.yaml`:

```yaml
        image: mccutchen/go-httpbin:v2.15.0
        ports:
        - containerPort: 8080
```

and pointing the Service at the new port in `apps/httpbin/base/service.yaml`:

```yaml
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

Commit, push, and let ArgoCD sync. The `ApiDefinition` needs no change — it targets
the Service on port 80.

### Bootstrap job fails waiting for the Developer Portal

`global.components.devPortal` and `tyk-bootstrap.bootstrap.devPortal` must agree. Both
are set to `false` in the overlay; if you re-enable the Portal, set both to `true`, add
a `TYK_PORTAL_LICENSE`, and create the `secrets-tyk-tyk-dev-portal` secret described in
the tyk-install README.

### Published ports are not reachable

Work outwards from the cluster:

```bash
# 1. Is the node container publishing them? Empty or missing lines mean the profile
#    was created without --ports (it is ignored on an existing container -- you have to
#    `minikube delete -p <profile>` and start again).
docker port staging

# 2. Do the NodePort Services have endpoints? An empty ENDPOINTS column means the
#    selector matched no pods -- most often because the Helm release is not named `tyk`,
#    which changes the pod labels the manifest selects on.
kubectl get svc,endpoints -n tyk -l app.kubernetes.io/part-of=tyk-operator-demo

# 3. Does it work from inside the node, bypassing the host mapping?
minikube ssh -p staging -- curl -s localhost:30080/hello
```

If step 3 works but step 1's mapping is present and the host still cannot connect,
check `--listen-address`: bound to `127.0.0.1`, the ports are unreachable from other
machines by design.

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
kubectl delete namespace tyk cert-manager argocd
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
