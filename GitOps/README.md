To go directly to each of the Teams GitOps repositories, click any of the links below:
- [backend-gitops](https://github.com/grayburnd/backend-gitops)
- [frontend-gitops](https://github.com/grayburnd/frontend-gitops)
- [platform-gitops](https://github.com/grayburnd/platform-gitops)
- [data-gitops](https://github.com/grayburnd/data-gitops)

## Architecture Overview
```mermaid
---
config:
  layout: elk
---
graph TD
    GitOpsRepo[GitOps Repo]
    
    GitOpsRepo --> AppSetBootstrap[AppSet: Bootstrap]
    GitOpsRepo --> AppSetControllers[AppSet: Controllers]
    GitOpsRepo --> AppSetCRDs[AppSet: CRDs]
    GitOpsRepo --> AppSetApps[AppSet: Apps]
    
    AppSetBootstrap --> BootstrapChart[Self-Maintained<br/>Helm Chart]
    AppSetApps --> AppsChart[Self-Maintained<br/>Helm Chart]
    AppSetControllers --> ThirdPartyCharts[Third-Party<br/>Helm Charts]
    AppSetCRDs --> CRDManifests[Third-Party<br/>CRD Manifests]
    
    BootstrapChart --> BootstrapDeploy[Bootstrap Components<br/>Deployed]
    AppsChart --> AppsDeploy[Applications<br/>Deployed]
    ThirdPartyCharts --> ControllersDeploy[Controllers<br/>Deployed]
    CRDManifests --> CRDsDeploy[CRDs<br/>Deployed]
    
    classDef gitOpsRepo stroke:#818cf8,fill:#eef2ff
    classDef appSet stroke:#a78bfa,fill:#f5f3ff
    classDef selfMaintained stroke:#4ade80,fill:#f0fdf4
    classDef thirdParty stroke:#fb923c,fill:#fff7ed
    classDef deployment stroke:#22d3ee,fill:#ecfeff
    
    class GitOpsRepo gitOpsRepo
    class AppSetBootstrap,AppSetControllers,AppSetCRDs,AppSetApps appSet
    class BootstrapChart,AppsChart selfMaintained
    class ThirdPartyCharts,CRDManifests thirdParty
    class BootstrapDeploy,AppsDeploy,ControllersDeploy,CRDsDeploy deployment

```


## GitHub Actions Workflow Overview
```mermaid
---
config:
  layout: dagre
---
flowchart TB
    Developer["Developer"] -- Push changes --> FeatureBranch["Feature Branch<br>GitOps Repo"]
    FeatureBranch --> PRExists{"PR Exists?"}
    PRExists -- No --> CreatePR["PR Created"]
    CreatePR --> PRWorkflow["PR Workflow<br>GitHub Actions"]
    PRExists -- yes --> UsePR["Use existing PR"]
    UsePR --> PRWorkflow
    PRWorkflow -- Step 1 --> HelmLint["Helm Lint<br>Check Chart Config"]
    HelmLint -- Pass --> KubeConform["Kubeconform Validation<br>Validate Against Schemas"]
    HelmLint -- Fail --> PRFailed1["PR Validation Failed"]
    KubeConform -- Pass --> ReviewReady["Ready for Review"]
    KubeConform -- Fail --> PRFailed2["PR Validation Failed"]
    ReviewReady -- Team Review --> Approved{"PR Approved?"}
    Approved -- Yes --> Merged["PR Merged<br>to Main"]
    Approved -- No --> Developer
    Merged -- Detect Changes --> ArgoCDController["ArgoCD Controller"]
    ArgoCDController -- Apply --> CanaryDeploy["Blue/Green Deployment<br>New ReplicaSets"]
    CanaryDeploy -- Manual Testing --> Testing["Testing Phase"]
    Testing -- Promote --> Production["Production Rollout"]
    AppSource["App Source Code Repo"] -- Push Release --> AppFeatureBranch["Feature Branch<br>App Repo"]
    AppFeatureBranch -- Update Docker Tags --> DockerUpdate["Docker Image Tag Update"]
    DockerUpdate -- Create PR --> AppPRCreated["PR Created"]
    AppPRCreated -- Trigger --> PRWorkflow

     Developer:::developer
     FeatureBranch:::developer
     PRWorkflow:::workflow
     HelmLint:::validation
     KubeConform:::validation
     Merged:::success
     ArgoCDController:::deployment
     CanaryDeploy:::deployment
     Production:::deployment
     AppSource:::developer
     AppFeatureBranch:::developer
     DockerUpdate:::registry
    classDef developer fill:#f0fdf4,stroke:#4ade80
    classDef workflow fill:#f0f9ff,stroke:#38bdf8
    classDef validation fill:#fff7ed,stroke:#fb923c
    classDef success fill:#f5f3ff,stroke:#a78bfa
    classDef deployment fill:#fdf4ff,stroke:#e879f9
    classDef registry fill:#fef2f2,stroke:#f87171
```


