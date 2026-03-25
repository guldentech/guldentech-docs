# CI/CD

GuldenTech uses [GitHub Actions](https://docs.github.com/en/actions) for CI/CD, with self-hosted runners running directly on the GuldenTech cluster.

---

## Self-Hosted Runners

Runners are managed centrally via [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller) — you don't need to manage any infrastructure yourself. Once onboarded, your workflows run on the cluster automatically.

### Getting Onboarded

**Step 1 — Install the GuldenTech GitHub App**

Install the app on your org or personal account:

👉 [https://github.com/apps/guldentech](https://github.com/apps/guldentech)

**Step 2 — Email us your details**

Send an email to **guldentechjobs@gmail.com** with:
- Your GitHub org name or personal account name
- Your **Installation ID**, found in the URL after installing the app:
  - Personal: `https://github.com/settings/installations/<INSTALLATION_ID>`
  - Org: `https://github.com/organizations/<org>/settings/installations/<INSTALLATION_ID>`

**Step 3 — We onboard you**

We'll set up your controller and confirm when you're ready. Your runner namespace will be `<org-or-account>-runners`.

**Step 4 — Deploy your runner resources**

Create a `RunnerDeployment` and `HorizontalRunnerAutoscaler` for each repo you want runners on, and apply them to your runner namespace.

!> Resources **must** be applied to your runner namespace (`<org-or-account>-runners`). The controller only watches that namespace — resources applied elsewhere will be ignored.

```bash
kubectl apply -f runners.yaml -n <org-or-account>-runners
```

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: <repo>-runner
spec:
  replicas: 1
  template:
    spec:
      repository: <org-or-account>/<repo>
      labels:
        - <your-label>
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: <repo>-runner-autoscaler
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: <repo>-runner
  minReplicas: 0
  maxReplicas: 2
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - <repo>
```

---

## Using Runners in Your Workflows

Once onboarded, set `runs-on: [self-hosted]` in your workflow jobs. That's it.

```yaml
name: Docker Build and Push

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: [self-hosted]
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
```

---

## Repo Secrets

Most workflows will need secrets for Harbor and Rancher. Add them under your repo's **Settings → Secrets and variables → Actions**:

| Secret | Description |
|---|---|
| Harbor robot user credentials | Used to push/pull images |
| Rancher API bearer token | Used to trigger deployments |

> Ask a GuldenTech admin or email **guldentechjobs@gmail.com** if you need help retrieving these.
