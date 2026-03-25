# Continuous Integration and Continuous Delivery (CI/CD)

We use GitHub Actions for CI/CD. If you plan to run workflows on GuldenTech infrastructure, you'll need self-hosted runners set up for your org or account.

---

## Self-Hosted Runners

We run self-hosted runners on the GuldenTech cluster using [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller). Runners are managed centrally — you don't need to deploy anything yourself.

### Getting Set Up

1. Install the **GuldenTech GitHub App** on your org or personal account:
   `https://github.com/apps/guldentech`

2. After installing, email **guldentechjobs@gmail.com** with:
   - Your **GitHub org name or personal account name**
   - The **Installation ID** (found in the URL after installing the app):
     - Personal account: `https://github.com/settings/installations/<INSTALLATION_ID>`
     - Org: `https://github.com/organizations/<org>/settings/installations/<INSTALLATION_ID>`

3. We'll onboard you and confirm when your runners are ready. Your runner namespace will be `<org-or-account>-runners`.

4. Once onboarded, apply a `RunnerDeployment` and `HorizontalRunnerAutoscaler` to your runner namespace for each repo you want to run jobs on:

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

Once onboarded, add `runs-on: [self-hosted]` to your workflow jobs. No other configuration needed.

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

You will likely need to add your Harbor robot user secret and Rancher API token to your repo for workflows to deploy. The Rancher secret to add is the bearer token in most cases.

Email **guldentechjobs@gmail.com** or ask a GuldenTech admin for help.
