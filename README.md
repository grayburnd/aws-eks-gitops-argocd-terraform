# AWS EKS GitOps Platform - Demonstration-Grade Kubernetes on AWS

> An automated Kubernetes platform on AWS EKS built with Terraform, ArgoCD and GitHub Actions, with manual approval for infrastructure changes and demonstrations of DevSecOps, GitOps, observability and advanced deployment patterns

![Terraform](https://img.shields.io/badge/Terraform-1.6%2B-%237B42BC?style=plastic&logo=terraform)
![K8s](https://img.shields.io/badge/Kubernetes-1.36%2B-%23326CE5?style=plastic&logo=kubernetes)
![ArgoCD](https://img.shields.io/badge/ArgoCD-3.5%2B-%23EF754F?style=plastic&logo=argo)
![License](https://img.shields.io/badge/License-MIT-green?style=plastic)

---

## Architecture Overview

```mermaid
---
config:
  layout: dagre
---
flowchart TB
 subgraph GH["GitHub - Source of Truth"]
          CODE
          IAC
          GITOPS
  end
 subgraph CODE["App Code"]
          Dockerfile
 end
 subgraph IAC["IaC"]
          Terraform
 end
 subgraph GITOPS["GitOps"]
          Helm
 end
 subgraph CICD_APP["GitHub Actions - App Code"]
        APP_B1["Lint and checks"]
        APP_B2["Test (Unit/Integration)"]
        APP_B3["Build, Scan and Push → ECR\ngit SHA tag"]
        APP_B4["Commit image tag\nback to GitOps repo"]
  end
 subgraph CICD_GITOPS["GitHub Actions - GitOps"]
        GITOPS_B1["Helm lint"]
        GITOPS_B2["Kubeconform"]
  end
 subgraph CICD_IAC["GitHub Actions - IaC"]
        IAC_B1["Terraform fmt + init + validate"]
        IAC_B2["GitLeaks Scan"]
        IAC_B3["Checkov Terraform Security Scan"]
        IAC_B4["Terraform Plan"]
        IAC_B5["Manual Approval"]
        IAC_B6["Terraform apply"]
  end
 subgraph GITHUB["GitHub"]
        GH
        CICD_APP
        CICD_GITOPS
        CICD_IAC
  end
 subgraph SYS["Per Team Namespaces (Karpenter-controlled)"]
        ARGO["ArgoCD\nApp-of-Apps"]
        ESO["External Secrets Operator\n→ AWS Secrets Manager"]
        HPA["Horizontal Pod Autoscaler"]
        VPA["Vertical Pod Autoscaler"]
        SM["Service Monitor"]
  end
 subgraph APP["Kube-System Namespace (Fargate-controlled)"]
        SOCK["AWS EBS, CloudWatch,\nVPC CNI Add-Ons"]
        DEMO["ArgoCD, Argo Rollouts"]
        OBS["Prometheus, Grafana, Alertmanager"]
        DB["Redis & Postgresql Operators"]
        VP["Vertical Pod Autoscaler Controller"]
        LB["AWS Load Balancer Controller,\nKarptenter Controller"]
  end
 subgraph CLSTR["Cluster-Wide Resources"]
        PRJ["ArgoCD Projects"]
        GS["GitHub App Secret"]
        CRDs
  end
 subgraph AWSOBS["AWS-Observability Namespace"]
        FB["Fluent-Bit Config"]
  end
 subgraph EKS["AWS EKS Cluster"]
        SYS
        APP
        CLSTR
        AWSOBS
  end
 subgraph AWS["AWS"]
        ECR[("Amazon ECR", Secrets Manager)]
        EKS
  end
    APP_B1 --> APP_B2
    APP_B2 --> APP_B3
    APP_B3 --> APP_B4
    GITOPS_B1 --> GITOPS_B2
    IAC_B1 --> IAC_B2
    IAC_B2 --> IAC_B3
    IAC_B3 --> IAC_B4
    IAC_B4 --> IAC_B5
    IAC_B5 --> IAC_B6
```

## Repository Layout

| Directory | Purpose |
|-----------|---------|
| [`App/`](App/README.md) | Application development workflow, image delivery and promotion into GitOps |
| [`GitOps/`](GitOps/README.md) | ArgoCD ApplicationSets, Helm charts and Kubernetes deployment workflow |
| [`IaC/`](IaC/README.md) | Terraform modules and the infrastructure provisioning workflow |

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Parameter files over branches for deploying to different GitOps environments** | Reduces chance of environment configuration drift, resulting in an increase in deployment velocity and reliability, following Kuberenetes GitOps best practices, using tools such as Helm and Terraform at their best |
| **OIDC for GitHub Actions** | No long-lived AWS Credentials stored in GitHub. GitHub Actions assumes a scoped AWS IAM role through the GitHub OIDC identity provider |
| **EKS Access policies** | Controls human and workload access to the EKS cluster through managed EKS access entries and policies, supporting centralized permissions and reducing reliance on broad node-level access |
| **GitHub App for cross-repository access** | Provides ArgoCD with scoped access to private GitHub repositories through an App ID, installation ID and private key rather than using a personal access token |
| **App-of-Apps ArgoCD Pattern** | Follows the Dont Repeat Yourself (DRY) method for managing ArgoCD Applications at scale |
| **IRSA over node-level IAM** | Pod-scoped AWS permissions for least privilege per workload capabilities |
| **Fargate to Manage Third-Party Controllers** | Prioritizes operational simplicity/efficiency whilst trading off Cost Optimization benefits |
| **Karpenter over Cluster Autoscaler** | Provision diverse, optimal node configurations at scale whilst reaping cost optimization benefits through Kubernetes APIs |
| **External Secrets Operator** | Centralized, Encrypted Secrets Management with AWS Secrets Manager |
| **Redis Sentinel** | High Availability of the Redis Cluster to reduce downtime |
| **Argo Rollouts (Blue/Green)** | Zero-downtime deployments, allowing for manual testing before promotion, with rollback capability |
| **Fluent-Bit forwarded App Logs** | Log aggregation in CloudWatch, allowing for long-term observability investigations and custom alarming based on logs. |
| **Databases managed by Operators** | Kubernetes native deployment, managed by Custom Operators to promote operational excellence |
| **Namespace per Team** | Each of the four teams deploys Kubernetes resources into its own namespace, following multi-tenant best practices |
| **ArgoCD Project per Team** | Ensures secure multi-tenancy by isolating developer blast radiuses at both the GitOps deployment layer and the Kubernetes cluster runtime layer |
| **Automated VPC IP Address management with VPC IPAM service** | Improves both operational efficiency and scalability of the Network, allowing for expansions into multiple VPC's in the future with full automation of the private Network Address Management layer |


---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Terraform | ~> 1.16.1 | Infrastructure provisioning |
| AWS CLI | >= 2.x | AWS authentication |
| kubectl | >= 1.36+ | Cluster management |
| Helm | CI-installed version | Chart deployments and validation |

### Parameter Store

Create the following AWS Systems Manager Parameter Store entries before deploying workloads. External Secrets Operator reads these values through IRSA and materializes them as Kubernetes Secrets for the application pods.

`redis-connection`:

```json
{
        "REDIS_SENTINEL_HOST": "${REDIS_SENTINEL_HOST}",
        "REDIS_MASTER_NAME": "mymaster",
        "REDIS_SENTINEL_PORT": "26379",
        "REDIS_USER_NAME": "default"
}
```

`postgres-connection`:

```json
{
        "DB_HOST": "${POSTGRES_HOST}",
        "DB_SSL_MODE": "require",
        "DB": "postgres"
}
```

`${REDIS_SENTINEL_HOST}` and `${POSTGRES_HOST}` represent the service DNS names for the deployed Redis and Postgres services. Notably, `${POSTGRES_HOST}` is made up of the `<postgres-service-name>.<namespace>` i.e `postgres.data-prod`. `${REDIS_SENTINEL_HOST}` is the name of the Sentinel headless service (as this fronts the Redis Cluster) followed by the namespace i.e `redis-s-hl.data-prod`. See the following links for more info:
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://redis-operator.opstree.dev/docs/

Do not commit connection values or credentials to Git!

### AWS and GitHub Access

The following access configuration is required before running the platform workflows:

- A GitHub OIDC identity provider in AWS.
- A scoped IAM trust policy and IAM role, represented by `${GITHUB_ACTIONS_ROLE}`, for GitHub Actions to obtain temporary AWS credentials. See the [AWS GitHub Actions OIDC guidance](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/).
- EKS access entries and policies for the identities that need cluster access.
- A GitHub App installed for the repositories ArgoCD must read, with at least read-only Contents permission. ArgoCD uses this credential to authenticate to private GitHub repositories. See the [ArgoCD private repository documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/).

The OIDC role authenticates CI/CD workflows to AWS, EKS access policies authorize cluster access and the GitHub App allows ArgoCD to pull private GitOps repositories.

---
## Quick Start

### 1. Provision the infrastructure

Navigate to the [IaC repository](https://github.com/grayburnd/aws) and configure the Terraform S3 backend and production environment values required by its workflow. The pull-request workflow validates the configuration and creates a reviewed Terraform plan. After the pull request is merged into `main`, the CD workflow applies that exact plan to provision the AWS infrastructure and EKS cluster.

The infrastructure workflow uses GitHub OIDC to assume the configured AWS IAM role. Confirm that the AWS account, GitHub environment variables, remote state and EKS access configuration are ready before opening the pull request.

### 2. Connect to the cluster

Terraform updates kubeconfig during provisioning. You can also run the command locally to select the cluster context:

```bash
aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
kubectl get nodes
```

ArgoCD is installed in the `kube-system` namespace. Verify that you can reach the cluster before continuing:

```bash
kubectl cluster-info
kubectl get pods -n kube-system
```

### 3. Access ArgoCD

Retrieve the initial ArgoCD administrator password and keep it available for the login step:

```bash
kubectl get secret -n kube-system argocd-initial-admin-secret \
        -o jsonpath='{.data.password}' | base64 --decode
```

In a separate terminal, forward the ArgoCD server service to your machine:

```bash
kubectl port-forward -n kube-system svc/argocd-server 8080:443
```

Open <https://localhost:8080> and sign in with username `admin` and the password retrieved above. The local port-forward uses the ArgoCD server's TLS service port; the public application Gateway described below is intentionally HTTP-only for this demonstration.

### 4. Bootstrap the GitOps repositories

Each GitOps repository contains its own ApplicationSets. Clone each repository locally and apply its ApplicationSet manifests manually. These ApplicationSets are parent resources, not the final workload Applications: once an ApplicationSet exists, ArgoCD processes its Git generator, creates the downstream Applications it describes, and then syncs and reconciles the referenced repository.

Apply the repositories in this order because each stage provides dependencies for the next one:

#### 4.1 Platform: shared cluster resources

The [platform GitOps repository](https://github.com/grayburnd/platform-gitops) bootstraps shared cluster services, including CustomResourceDefinitions, controllers, operators, ArgoCD Projects, networking and observability. Its generated resources target `kube-system` and `platform-prod`.

From a local clone of `platform-gitops`, apply the Projects and locally managed GitHub App repository credential first, followed by the platform ApplicationSets:

```bash
kubectl apply -f projects/
kubectl apply -f github_app_secret.yml

# CRDs, then controllers/operators, then platform bootstrap and applications.
kubectl apply -f appsets/prod-appset-crds.yml
kubectl apply -f appsets/prod-appset-controllers.yml
kubectl apply -f appsets/prod-appset-bootstrap.yml
kubectl apply -f appsets/prod-appset.yml
```

The platform ApplicationSets are classified as follows:

| ApplicationSet | Type and purpose | Destination |
|----------------|------------------|-------------|
| `prod-appset-crds.yml` | CRD ApplicationSet: installs third-party CustomResourceDefinitions | `kube-system` |
| `prod-appset-controllers.yml` | Controller/operator ApplicationSet: installs shared controllers and operators | `kube-system` |
| `prod-appset-bootstrap.yml` | Bootstrap ApplicationSet: installs platform bootstrap services | `platform-prod` |
| `prod-appset.yml` | Platform application ApplicationSet: installs platform applications such as networking | `platform-prod` |

The GitHub App Secret is intentionally ignored by Git in the platform repository. Create it from your secure local secret source, and do not commit private keys or credentials.

#### 4.2 Data: PostgreSQL and Redis

After the platform dependencies are available, clone the [data GitOps repository](https://github.com/grayburnd/data-gitops). Its ApplicationSets deploy PostgreSQL and the Redis dependencies into `data-prod`. Apply the bootstrap resources first, then third-party applications such as Redis, and finally the data application charts:

```bash
kubectl apply -f appsets/prod-appset-bootstrap.yml
kubectl apply -f appsets/prod-appset-thirdparty.yml
kubectl apply -f appsets/prod-appset.yml
```

The data repository uses three ApplicationSet types: a bootstrap ApplicationSet for data-specific supporting resources, a third-party ApplicationSet for external Helm dependencies, and an application ApplicationSet for charts under `apps/*`.

#### 4.3 Backend: voting worker

After PostgreSQL and Redis are available, clone the [backend GitOps repository](https://github.com/grayburnd/backend-gitops). Its bootstrap ApplicationSet creates backend supporting resources, and its application ApplicationSet deploys the internal voting worker into `backend-prod`:

```bash
kubectl apply -f appsets/prod-appset-bootstrap.yml
kubectl apply -f appsets/prod-appset.yml
```

#### 4.4 Frontend: vote and results applications

After the worker and its data dependencies are available, clone the [frontend GitOps repository](https://github.com/grayburnd/frontend-gitops). Its bootstrap ApplicationSet creates frontend supporting resources, and its application ApplicationSet deploys the voting and results applications into `frontend-prod`:

```bash
kubectl apply -f appsets/prod-appset-bootstrap.yml
kubectl apply -f appsets/prod-appset.yml
```

The generated frontend Applications also create the HTTPRoutes used by the public Gateway.

#### Multi-source values

The team ApplicationSets use ArgoCD multi-source Applications. Chart or configuration content is read from the relevant team repository, while Helm values are referenced from the private `private-gitops-values` repository through `$values/...` paths. Example values files may be visible alongside the charts for documentation and validation, but production values remain in the private repository. When all repositories are private, storing the chart and environment values in the same repository can be operationally simpler; multi-source Applications remain useful when public chart configuration and private environment values need to be separated.

Wait for each dependency to become available before troubleshooting the next generated Application. For example, a ServiceMonitor may remain unhealthy until the kube-prometheus-stack CRDs and controller are ready. In that situation, wait for the dependency, then retry or reconcile the failed ArgoCD Application rather than deleting resources immediately.

### 5. Deploy the team workloads

After the repository ApplicationSets have been applied manually in the order above, their downstream Applications generate from the team repositories and ArgoCD watches and reconciles their Helm charts. The team workloads are not deployed by manually applying workload manifests from this repository. The production workload namespaces are:

| Repository | Namespace | Workloads |
|------------|-----------|-----------|
| [backend-gitops](https://github.com/grayburnd/backend-gitops) | `backend-prod` | Voting worker |
| [data-gitops](https://github.com/grayburnd/data-gitops) | `data-prod` | PostgreSQL and Redis dependencies |
| [frontend-gitops](https://github.com/grayburnd/frontend-gitops) | `frontend-prod` | Voting and results applications |

Before the workloads can become healthy, create the `redis-connection` and `postgres-connection` AWS Systems Manager Parameter Store values described in [Parameter Store](#parameter-store). Set their service DNS values after the data services have been deployed: typically `redis-s-hl.data-prod` for Redis Sentinel and `postgres.data-prod` for PostgreSQL. External Secrets Operator reads these values through IRSA and materializes Kubernetes Secrets for the application pods.

After the platform bootstrap is ready, merge the required workload changes into the respective GitOps repositories and allow ArgoCD to reconcile them. You can monitor the generated Applications with:

```bash
kubectl get applications -A
kubectl get pods -A
```

### 6. Access the applications

The platform networking application creates the internet-facing `pub-gateway` Gateway in the `platform-prod` namespace. Wait for the AWS Load Balancer Controller to provision its address, then retrieve it with:

```bash
GATEWAY_ADDRESS=$(kubectl get gateway pub-gateway -n platform-prod \
        -o jsonpath='{.status.addresses[0].value}')
echo "http://${GATEWAY_ADDRESS}"
```

Open the following routes in a browser:

- Voting: `http://${GATEWAY_ADDRESS}/vote`
- Results: `http://${GATEWAY_ADDRESS}/results`

The public Gateway intentionally uses HTTP without TLS termination or a configured domain name because this is a demonstration-grade deployment. A production deployment should add an appropriate Gateway listener, certificate and AWS networking configuration.

### 7. Logs and progressive delivery

To inspect a container's logs, first identify the pod and container, then run:

```bash
kubectl get pods -n <namespace>
kubectl logs -n <namespace> <pod-name> -c <container-name>
```

The frontend voting and results applications use Argo Rollouts blue/green deployments. New ReplicaSets are sent to preview services and are not promoted automatically because `autoPromotionEnabled` is `false`. Test the preview version, then promote it through the ArgoCD Rollouts extension or the Argo Rollouts CLI. Using the CLI is outside the scope of this demonstration; see the [ArgoCD Rollouts extension documentation](https://argo-cd.readthedocs.io/en/stable/proposals/002-ui-extensions/#argo-rollout-extension-poc) for the UI workflow.

The worker, voting and results applications may therefore remain paused until their new versions are promoted. If an Application fails because a required CRD or controller is not ready, wait for the dependency to become healthy and then retry or reconcile the affected Application.





---
## DevSecOps

### Secret Management

```
AWS Secrets Manager / SSM Parameter Store (source of truth)
          │
          ▼
  ExternalSecret CRD
          │
          ▼
  ESO Controller (authenticates to AWS via IRSA)
          │
          ▼
  Kubernetes Secret (controlled by ESO Controller)
          │
          ▼
  Pod environment variable / mounted volume
```
### Analysis
```
  Static application security testing (Bandit)
          │
          ▼
  Software composition analysis (Trivy)
          │
          ▼
  IaC security scanning (Checkov)
          │
          ▼
  Dockerfile scanning (hadolint)

```
---

## Observability Stack

**kube-prometheus-stack** (Prometheus + Grafana + Alertmanager):

- **Metrics**: Cluster, node, pod, and application-level metrics
- **Dashboards**: Pre-built + custom Grafana dashboards (Kubernetes cluster, Sock-Shop SLOs)
- **Alerts**: Configurable for a move to Production

---

## Blue/Green Deployments (Argo Rollouts)

The frontend applications use Argo Rollouts for zero-downtime blue/green deployments. Their preview services support validation before promotion and automatic promotion is disabled, so promotion happens after manual testing.

---

## Infrastructure Costs (Estimated)

> Example region: `${AWS_REGION}`. Figures marked **Grounded** are derived directly from the Terraform in the infrastructure repository. Figures marked **Assumption** cover usage or runtime consumption that is not fixed by repository configuration. Karpenter capacity settings and per-app Helm resource values are defined in the GitOps repositories, while actual monthly usage depends on workload demand.

| Item | Basis | Est. $/month |
|------|-------|---------------|
| EKS control plane | Grounded - fixed AWS price ($0.10/hr) | ~$73 |
| NAT Gateway (single AZ) | Grounded - one `aws_nat_gateway`, cost-optimized deliberately | ~$38 |
| Fargate (kube-system controllers: LBC, redis-operator, postgres-operator, ESO, kube-prometheus-stack, metrics-server, VPA, Karpenter, CoreDNS, VPC-CNI, EBS-CSI, CloudWatch add-on) | Assumption - ~10 pods avg 0.25 vCPU/0.5GB, 24/7 | ~$90–120 |
| Karpenter-managed EC2 nodes (5 team namespaces) | Assumption - Spot capacity selected by the platform Karpenter configuration at ~60–70% off on-demand | ~$45–90 |
| EBS volumes (Postgres/Redis via operators) | Assumption - 2–3 × 20GB gp3 | ~$5–8 |
| CloudWatch Logs (control-plane logging, all 5 log types + Fluent Bit app logs + Container Insights) | Assumption - 5–10GB ingested/month | ~$5–10 |
| KMS CMK (K8s secrets encryption) | Grounded - 1 dedicated key | ~$1 |
| S3 (Postgres log archive, 30-day expiry) | Grounded - 1 bucket, small footprint | <$1 |
| Secrets Manager (via ESO) | Assumption - ~8 secrets × $0.40 | ~$3 |
| ALB (via AWS Load Balancer Controller) | Assumption - 1 ALB | ~$16 |
| **Total (rough range)** | | **~$275–360/month** |

> For cost savings during development, use a single NAT gateway (already the default here) and consider scaling down Karpenter-managed nodes and Fargate profile replicas outside of demo hours.

---

## Main Technologies Used

| Category | Technology |
|----------|-----------|
| Cloud | AWS (EKS, ECR, VPC, EC2, IAM, Secrets Manager, SSM) |
| IaC | Terraform ~> 1.16.1, CloudFormation |
| Containers | Docker, AWS ECR |
| Orchestration | Kubernetes 1.36+, Helm 4 |
| GitOps | ArgoCD 3.5+ (App-of-Apps), Helm 4+, Argo Rollouts (ArgoCD Extension) |
| CI/CD | GitHub Actions (OIDC auth) |
| Security Scanning | Trivy (Docker Image), GitLeaks, Hadolint (Dockerfile), Checkov (IaC)  |
| Network Security | AWS Security Groups |
| Secret Management | External Secrets Operator + AWS Secrets Manager |
| Observability | Prometheus, Grafana, Alertmanager |
| Autoscaling | Karpenter, HPA (CPU/memory) |
| Gateway API | AWS Load Balancer Controller |
| DNS | Route53 (optional) |

---
## Possible Improvements if Moving to Production
> IMPORTANT: Due to the scope of this repo being a demo, the improvements below are for reference only.

| Category | Improvement | Rationale |
|----------|-------------|-----------|
| Security | Scope all IAM Roles to least-privellege | Strengthens principle of least privellege positioning thus improving security
| Security | Implement a WAF, with extensive layer 7 attack protections such as AWS WAF in combination with AWS Shield | Promotes a defence-in-depth approach by protecting the outer perimiter of the AWS Network
| Performance efficiency / Security / Reliability | Front any cached content with CloudFront | Reduces load on backend, utilizing caches closer to the end user for layered well architected benefits.
| Operational excellence | Deploy Kyverno into Cluster | Facilitates Governance at scale across the entire Cluster through Guardrails
| Security | Remove public endpoint access to the Cluster | Promotes defence in depth by ensuring the Cluster can only be accessed when connected to the hosting VPC privately
| Security | Implement network policies | Enforces network segregation, following network layer zero trust best practices
| Operational excellence | Migrate Redis and Postgres to AWS-Managed Services | Reduces the operational footprint of maintaing DB Services
| Reliability | Scale out all infra to multiple Availability Zones | Improves the availability of the application, making it resistent to disasters
| Operational excellence | Implement alerting for key app SLOs | Helps align resource focus to key SLI's, such as P99 Latency and detect issues before they occur
| Security | Configure RBAC in ArgoCD for each team member | Follows the principle of least privellege access within ArgoCD

## Author: Daniel Grayburn
- Cloud-native infrastructure design (AWS EKS)
- GitOps methodology at scale (ArgoCD App-of-Apps)
- DevSecOps pipeline with shift-left security (Trivy, Checkov, Hadolint)
- Zero-downtime deployment strategies (Argo Rollouts)