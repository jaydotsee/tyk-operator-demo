# tyk-operator-demo

A practical example demonstrating how to implement a GitOps workflow using **Tyk Operator**, **ArgoCD**, and **Kustomize** to manage API deployments across multiple Kubernetes environments.

## About
In this repo, we walk through setting up a GitOps workflow that automates the deployment of both applications and API configurations using Tyk Operator, ArgoCD, and Kustomize. By following the guide, you'll learn how to manage deployments efficiently in an organization using GitOps principles.
  
## Repository Structure
The repository is organized into four main directories:

```
tyk-operator-demo/
├── apps/
│   └── httpbin/
│       ├── base/
│       └── overlays/
│           ├── prod/
│           └── staging/
├── policies/
│   ├── prod/
│   └── staging/
├── argocd/
│   ├── prod/
│   └── staging/
└── docs/
    ├── minikube-setup.md          # full Minikube walkthrough
    ├── blog-update.md             # what changed vs. the blog post
    ├── tyk-minikube.values.yaml   # Helm overlay for Minikube
    └── ingress/                    # per-cluster ingress for host access
```

### 1. `apps/` Directory
Contains Kubernetes manifests for deploying applications. In this example, we're focusing on the `httpbin` application.

- `apps/httpbin/base/`: Base configuration common to all environments.
- `apps/httpbin/overlays/prod/`: Production-specific customizations.
- `apps/httpbin/overlays/staging/`: Staging-specific customizations.

### 2. `policies/` Directory
Stores security policy manifests, governing security options and access rights.

- `policies/prod/`: Security policies for the production environment.
- `policies/staging/`: Security policies for the staging environment.

### 3. `argocd/` Directory
Contains ArgoCD application manifests that automate the deployment process.

- `argocd/prod/`: ArgoCD applications for the production environment.
- `argocd/staging/`: ArgoCD applications for the staging environment.

### 4. `docs/` Directory
Setup and migration documentation.

- [`docs/minikube-setup.md`](./docs/minikube-setup.md): end-to-end Minikube walkthrough built on [tyk-install](https://github.com/TykTechnologies/tyk-install).
- [`docs/blog-update.md`](./docs/blog-update.md): how the original blog post's instructions need to be updated.
- [`docs/tyk-minikube.values.yaml`](./docs/tyk-minikube.values.yaml): Helm values overlay applied on top of the `tyk-install` values for Minikube.
- [`docs/ingress/`](./docs/ingress): per-cluster ingress values and ArgoCD `Ingress` manifests, for exposing both clusters on the host behind your own load balancer.

## Getting started

> **Setting up from scratch?** Follow [docs/minikube-setup.md](./docs/minikube-setup.md).
> It is a complete, current walkthrough that builds both clusters on Minikube using
> [tyk-install](https://github.com/TykTechnologies/tyk-install).
>
> The [original blog post](https://tyk.io/blog/a-practical-guide-using-tyk-operator-argocd-and-kustomize/)
> still describes the GitOps design accurately, but its environment setup relies on the
> archived `tyk-k8s-demo` repository and no longer works. See
> [docs/blog-update.md](./docs/blog-update.md) for what changed.

### Prerequisites

- Familiarity with Kubernetes and Kustomize.
- Basic understanding of Git.
- Two Kubernetes clusters (e.g. two Minikube profiles), each running the Tyk stack
  with Tyk Operator, plus ArgoCD.
- Helm 3.12+.
- A Tyk Dashboard licence **and** a Tyk Operator licence ([free trial](https://tyk.io/sign-up/)).
- On Apple Silicon or any other arm64 host, the demo's `httpbin` image needs a one-time swap — [see the guide](./docs/minikube-setup.md#if-you-are-on-arm64-swap-the-httpbin-image).

### Steps

1. **Fork and Clone the Repository**
    
    Fork this repository to your own GitHub account and then clone it locally:
    
    ```bash
    git clone https://github.com/your-username/tyk-operator-demo.git
    
    cd tyk-operator-demo
    ```
    
2. **Set Up the Environment**

    Follow [docs/minikube-setup.md](./docs/minikube-setup.md) to build the `staging` and
    `production` Minikube clusters, install the Tyk stack (including Tyk Operator) with
    the Helm charts from [tyk-install](https://github.com/TykTechnologies/tyk-install),
    and install ArgoCD in each cluster.

3. **Update ArgoCD Applications**
    
    Modify the `repoURL` in the ArgoCD application manifests under the `argocd/` directory to point to your forked repository.
    
    For example, in `argocd/staging/httpbin.yaml`:
    
    ```yaml
    spec:
      source:
        repoURL: 'https://github.com/your-username/tyk-operator-demo'
    ```
    
4. **Deploy with ArgoCD**
    
    Apply the ArgoCD application manifests to your clusters to deploy the applications and policies.
    
    **For Staging:**
    
    ```
    kubectl config use-context staging
    
    kubectl apply -f argocd/staging-apps.yaml
    ```
    
    **For Production:**
    
    ```
    kubectl config use-context production
    
    kubectl apply -f argocd/prod-apps.yaml
    ```
    
5. **Experiment with Changes**
    
    Modify the `ApiDefinition` or `SecurityPolicy` manifests in the `apps/` or `policies/` directories, then commit and push the changes to your repository.
    
    ```bash
    git add . 
    git commit -m "Your commit message" 
    git push origin main
    ```
    
    ArgoCD will automatically detect and synchronize the changes from Git to Kubernetes. Tyk Operator automatically detect changes of the custom resources and update the live configurations in Tyk accordingly.

## What You've Accomplished
By following this guide and using this repository, you've:

- Set up a comprehensive GitOps workflow.
- Automated deployment of applications and API configurations across multiple environments.
- Utilized Tyk Operator to manage API definitions and security policies as Kubernetes resources.
- Employed ArgoCD for continuous deployment and observed live updates.
- Leveraged Kustomize for environment-specific configurations without duplicating code.
- Gained insights into organizing repositories for scalability and maintainability.

This setup enables efficient management of deployments in an organization, ensuring consistency, reducing manual errors, and accelerating the development workflow.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request if you have suggestions for improvements.

## License

[Mozilla Public License Version 2.0](./LICENSE)

## Questions
For questions on products, please use [Tyk Community forum](https://community.tyk.io/).
  <br>
Clients can also use support@tyk.io.
   <br>
Prospects and evaluators, please use info@tyk.io.
