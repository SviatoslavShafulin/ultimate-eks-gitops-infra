# Multi-Environment Amazon EKS Configuration Pipeline via Flux GitOps

This repository provides an enterprise-grade configuration blueprint for managing dual-cluster topologies (`test` and `production`) utilizing declarative GitOps workflows managed by Flux v2. The pipeline handles core cluster bootstrapping, centralized policy enforcement, fine-grained network segmentation, end-to-end monitoring stacks, and progressive traffic routing mechanisms.

---

## Architecture Design & Topology

The implementation leverages a **Repository-per-Team** separation model, isolating the foundational infrastructure state from application-level execution spaces. 

* **Platform Control Plane:** Managed within this repository to define baseline cluster add-ons, tenant onboarding namespaces, and localized `GitRepository` definitions.
* **Developer Workspaces:** Decoupled out to the tenant infrastructure to allow sovereign delivery cycles inside secure sandboxes.

              ┌───────────────────────────────┐
              │     Central Platform Team     │
              └───────────────┬───────────────┘
                              │
              ┌───────────────▼───────────────┐
              │  ultimate-eks-gitops-infra    │
              └───────┬───────────────┬───────┘
                      │               │
        ┌─────────────▼─────┐   ┌─────▼─────────────┐
        │  Clusters (Test)  │   │ Clusters (Prod)   │
        └───────────────────┘   └───────────────────┘


### Multi-Tenancy Security Model
* **Lockdown Mode:** Both environments explicitly activate `multi-tenancy-lockdown` to limit tenant capabilities and dynamically block cross-namespace vector access. Review configs in `clusters/<environment>/flux-system/kustomization.yaml`.
* **Impersonation Handshakes:** Tenant onboarding initializes a localized `ServiceAccount` bounded to a `cluster-admin` `ClusterRoleRoleBinding` confined strictly to that namespace space. This establishes a boundaries layout allowing the controller to safely impersonate tenant execution profiles. Refer to the Controller Permissions schema for granular custom boundary tuning.

---

## Core Infrastructure Add-on Matrix

This configuration layer orchestrates a foundational platform stack across target clusters:

| Component Name | System Responsibility | Reference Upstream |
| :--- | :--- | :--- |
| **AWS Load Balancer Controller** | Provisions and configures AWS Elastic Load Balancers dynamically. | [Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/) |
| **Metrics Server** | Aggregates resource metrics for Horizontal Pod Autoscaling (HPA). | [Source](https://github.com/kubernetes-sigs/metrics-server) |
| **Calico** | Network policy engine enforcing microsegmentation and strict isolation boundaries. | [Amazon EKS Reference](https://docs.aws.amazon.com/eks/latest/userguide/calico.html) |
| **Kyverno** | Policy framework scanning, mutating, and validating resource requests against structural baselines. | [Policies Guide](https://kyverno.io/) |
| **Prometheus Stacking** | Installs `kube-prometheus-stack` to provision systems metric scraping clusters. | [Operator Details](https://prometheus.io/) |
| **Flagger** | Automation controller routing and executing advanced Canary and Blue/Green application deployments. | [Execution Engine](https://flagger.app/) |
| **Nginx Ingress Controller** | Manages external traffic routing while acting as an edge driver for Flagger splitting loops. | [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/) |

> [!WARNING]
> Bundled manifests are tuned for execution exploration. Production-ready deployments require persistent telemetry configurations (e.g., long-term metric systems) and localized HPA scalability loops.

For deeper architectural breakdowns, inspect the [Repository Structure documentation](docs/repository-structure.md). Review the [Flux Docs](https://fluxcd.io/docs/) for prerequisite alignment.

### Ecosystem Inspiration
* [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example/)
* [flux2-multi-tenancy](https://github.com/fluxcd/flux2-multi-tenancy)
* [Kyverno Built-in Policies Catalog](https://kyverno.io/policies/)

---

## Deployment Playbook

### Core Prerequisites
1. **Amazon EKS Cluster (v1.21+):** Associated with an active IAM OIDC Identity Provider mapping.
2. **AWS LBC IAM Role:** Configured explicitly within the `kube-system` namespace context.

*To spin up these layers via `eksctl`, execute the exact step guides mapped inside [EKS Cluster Provisioning Docs](docs/deploy_cluster.md).*

*If utilizing HashiCorp Terraform pipelines, deploy via the [terraform-aws-eks-blueprints](https://github.com/aws-ia/terraform-aws-eks-blueprints) infrastructure modules. Turn off standard add-on installations within the [Blueprints Module Definition](https://github.com/aws-ia/terraform-aws-eks-blueprints/blob/main/examples/eks-cluster-with-new-vpc/main.tf#L108-L112) to hand state over to Flux. Configure explicit [LBC IRSA parameters](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/#setup-iam-role-for-service-accounts) manually.*

> [!IMPORTANT]
> **State Note for Existing EKS Clusters:** Custom Resource Definitions (CRDs) provisioned via Helm require manual lifecycle removals. Additionally, orphan Calico `iptables` parameters may persist on underlying worker topologies. It is highly recommended to provision clean EKS worker environments for this framework.

### Active Tooling Checklist
* **Flux CLI Engine:** v0.30+ installed. Framework explicitly validated using version `0.30.2`.
* **GitHub Integration Profile:** Valid token credentials authorizing repository lifecycle generation.

---

### Structural Deployment Steps

#### Step 1: Initialize Local Environmental Context
Fork this infrastructure engine code base onto your personal profile workspace and export the state parameters:
```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>