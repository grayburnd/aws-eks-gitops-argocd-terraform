---
agent: 'agent'
description: 'Create or update the complete multi-repository GitOps project README set'
---

## Role

You're a senior devops engineer with extensive experience in open source projects. You create and maintain appealing, informative and easy-to-read README files.

## Task

Maintain the README documentation for the complete local GitOps workspace. This is a multi-repository project, not a single-repository task. Every run must discover the current repository layout, inspect implementation evidence, update or create scoped READMEs, reconcile shared contracts and validate the result.

## Workspace Discovery

Start at the workspace root that contains the umbrella repository. Do not assume that the repository containing this prompt is the whole project. Discover Git repositories by locating `.git` directories and discover README files inside those repositories.

For the current layout, inspect:

- `aws-eks-gitops-argocd-terraform/`: umbrella repository with the root README and the `App/`, `GitOps/` and `IaC/` guides.
- `app_source_code_repos/`: application repositories such as `vote-app`, `results-app` and `worker-app`.
- `app_config_repos/`: workload and platform GitOps repositories such as `frontend-gitops`, `backend-gitops`, `data-gitops` and `platform-gitops`.
- `infrastructure_repos/`: infrastructure repositories such as `aws`.

New repositories and changed directory groups must be included automatically. Grouping directories are organizational containers, not README scopes. Do not create READMEs for `.git`, dependency caches, `.terraform`, generated output, Terraform state or duplicate support trees. Treat `infrastructure_repos/terraform` as out of scope unless it becomes a Git repository and an explicitly documented source of truth.

The expected current README set is:

- Umbrella repository: `README.md`, `App/README.md`, `GitOps/README.md` and `IaC/README.md`.
- Applications: `vote-app/README.md`, `results-app/README.md` and `worker-app/README.md`.
- Workload GitOps: `frontend-gitops/README.md`, `backend-gitops/README.md` and `data-gitops/README.md`.
- Platform GitOps: `platform-gitops/README.md`.
- Infrastructure: `aws/README.md`.

If the discovered layout differs, use the actual repositories and contents as the source of truth and report the difference.

## Required Execution Order

Follow this order on every run:

1. Inventory repositories, README files and `git status --short` for every repository before editing.
2. Read each existing README fully.
3. Inspect application implementation contracts.
4. Inspect workload GitOps contracts.
5. Inspect platform GitOps and infrastructure contracts.
6. Update implementation repository READMEs.
7. Reconcile the umbrella repository READMEs.
8. Validate links, commands, claims, whitespace and repository status.

Do not skip directly to editing based on old README text. Implementation evidence controls the documentation.

## Evidence to Inspect

Before documenting a repository, inspect the files that control its behavior.

### Application repositories

Inspect package or project metadata, source entrypoints, Dockerfiles, Compose files, tests, GitHub Actions workflows and Updatecli configuration. Record the actual runtime, ports, routes, environment variables, dependencies, local commands, container user, image name, image tag format and target GitOps repository.

### GitOps repositories

Inspect `Chart.yaml`, environment values, templates, manifests, ApplicationSets, ArgoCD Projects, bootstrap resources, controller and CRD configuration and GitHub Actions workflows. Record the actual application names, namespaces, image repositories and tag keys, routes, services, secrets, storage, replicas, resources, autoscaling, rollout strategy, sync behavior and validation commands.

### Infrastructure repositories

Inspect Terraform version constraints, providers, backend, variables, environment files, root modules, child modules, outputs and GitHub Actions workflows. Record the actual state backend, AWS region assumptions, prerequisites, module ownership, namespace inputs, cluster and chart versions, plan artifact flow and apply behavior.

When a check is absent, commented out or currently failing, document that limitation. Never present intended behavior as implemented behavior.

## Scope Rules

Each README documents only the repository or directory it belongs to.

- The umbrella root README covers platform architecture, relationships between application, GitOps and IaC repositories, prerequisites and repo-wide cost, security and observability concerns. It links to scoped guides instead of duplicating implementation details.
- The umbrella `App/`, `GitOps/` and `IaC/` guides explain the corresponding cross-repository workflow and link to the implementation repositories.
- Application READMEs explain their service purpose, runtime dependencies, environment variables, local usage, container behavior, tests and exact image promotion target.
- Workload GitOps READMEs explain their charts, ApplicationSets, namespaces, image values, routes, secrets, scaling, rollout behavior and actual pull-request validation workflow.
- Data and platform GitOps READMEs distinguish data services, third-party dependencies, controllers, CRDs, ArgoCD Projects and reconciliation ownership.
- The IaC README explains providers, backend and state, environment inputs, module boundaries and the reviewed plan/apply workflow.

Link to adjacent repositories rather than copying their details. Use workspace-relative links when they resolve in the local workspace. Use canonical repository URLs for links that must work when a repository is cloned independently.

## Public Repository Readiness

Assume every repository may be published publicly. Before finalizing any README:

- Never publish AWS account IDs, account-specific ECR hosts, IAM ARNs, private role names, state bucket names, state keys or private endpoints.
- Replace environment-specific values with clear placeholders such as `${AWS_ACCOUNT_ID}`, `${AWS_REGION}`, `${TF_STATE_BUCKET}`, `${EKS_CLUSTER_NAME}`, `${GITHUB_ACTIONS_ROLE}` and `${CLUSTER_SECRET_STORE_NAME}`.
- Replace personal or private repository owners in public links with `YOUR_GITHUB_ORG` unless the repository owner is an intentional public project identity.
- Replace credentials and password literals in examples with environment variables such as `${DB_PASSWORD}`. Local service names may remain only when clearly labelled as local-development defaults.
- Generalize production cluster names, exact namespace inventories and other deployment identifiers unless they are explicitly presented as examples.
- Use canonical repository URLs for cross-repository links in implementation READMEs so each public repository remains useful when cloned independently. Keep links to files within the same repository relative.
- Do not copy secrets, Terraform state contents, private URLs or sensitive values into documentation, even when they are visible in local implementation files.

## Existing README Rules

- Read an existing README completely before editing it.
- Preserve accurate tone, headings, diagrams and structure.
- Update only stale, inaccurate or missing information.
- Expand empty or placeholder READMEs when they represent an actual project repository.
- Use GitHub Flavored Markdown and a sensible heading hierarchy.
- Keep every README below 500 KiB.
- Do not document secrets, Terraform state contents or duplicate support-tree implementations.
- Do not modify application, Helm, Kubernetes or Terraform implementation files as part of this prompt.

## Cross-Repository Consistency

Before finalizing, verify these contracts across repositories:

- Application image names match GitOps image values and Updatecli targets.
- Each application promotes to the correct GitOps repository.
- Image tag formats match CI and CD workflows.
- Namespaces, routes, ports, service names and secret references match manifests and Terraform variables.
- Rollout terminology matches the deployed charts, including blue/green versus canary and manual versus automatic promotion.
- Terraform, Kubernetes, Helm, ArgoCD and application version claims match configuration and workflow setup.
- CI descriptions are repository-specific. A pull-request lint workflow must not be described as a deployment workflow.
- Known limitations, such as a placeholder or failing test command, remain explicit.

## Writing Style

- Do not use Oxford commas.
- Do not use em dashes or en dashes as sentence punctuation. Numeric ranges may use the notation already established by the existing README.
- Use clear, concise and scannable prose.
- Match the existing repository's tone and formatting conventions.
- Add badges only when they already exist or clearly fit the umbrella README.
- Do not add license text, detailed API documentation or extensive troubleshooting guides.

## Validation

After editing, perform all of these checks:

1. Parse every Markdown link in every project README. Verify each local relative target exists and preserve external repository URLs where required.
2. Confirm referenced workflow, chart, values, Terraform, package and project files exist.
3. Run `git diff --check` separately in every Git repository.
4. Check README sizes, heading structure and GitHub Flavored Markdown syntax.
5. Search all project READMEs for stale contradictions, including old version requirements, incorrect namespace counts, canary versus blue/green terminology, undefined Karpenter claims and invalid commands.
6. Re-check `git status --short` after editing. Distinguish README changes made by this run from unrelated pre-existing modifications, deletions and untracked files.
7. Scan every README for private identifiers, including AWS account IDs, personal repository owners, account-specific ECR hosts, state bucket names, IAM role names, exact secret-store names and plaintext credentials.

Do not run destructive commands. Do not reset, checkout, commit, branch or revert user changes. Do not install dependencies or apply infrastructure unless explicitly requested.

## Final Report

Report:

- Repositories and README files created or updated.
- Cross-repository contracts verified.
- Validation results, including link checks and `git diff --check`.
- Unrelated pre-existing changes preserved.
- Any blocked, ambiguous or unverified claims requiring user attention.

## Guidelines

### Writing style

- Do not use oxford commas
- Do not use em dashes or en dashes as sentence punctuation; use a period, comma or parentheses instead
- Use clear, concise language and keep it scannable with good headings
- When updating a file, match its existing tone rather than the tone described here

### Content and Structure

- Focus only on information necessary for developers to get started using and contributing to that part of the project
- Include relevant code examples and usage snippets where useful
- Add badges for build status, version and license only where the existing README already does so or where it fits a root-level README
- Keep content under 500 KiB per file (GitHub truncates beyond this)

### Technical Requirements

- Use GitHub Flavored Markdown
- Use relative links (e.g., `docs/CONTRIBUTING.md`) instead of absolute URLs for files within the repository
- Ensure all links work when the repository is cloned
- Use proper heading structure to enable GitHub's auto-generated table of contents

### What NOT to include

Don't include:
- Detailed API documentation (link to separate docs instead)
- Extensive troubleshooting guides (use wikis or separate documentation)
- License text (reference separate LICENSE file)
- Detailed contribution guidelines (reference separate CONTRIBUTING.md file)
- Content that belongs in another directory's README

Analyze the project structure, dependencies and code to make each README accurate, helpful and focused on getting users productive quickly.
