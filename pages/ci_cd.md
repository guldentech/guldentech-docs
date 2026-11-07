# CI/CD

GuldenTech uses [GitHub Actions](https://docs.github.com/en/actions) for CI/CD, with self-hosted runners running directly on the GuldenTech cluster.

Deployments use **in-cluster `kubectl`**: your runner pods authenticate to Kubernetes with their own ServiceAccount,
scoped by RBAC to the namespaces you allow. There's no Rancher token or kubeconfig stored in GitHub.

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

Apply the deploy access (ServiceAccount + RBAC) from the next section first, then create a `RunnerDeployment` and
`HorizontalRunnerAutoscaler` for each repo you want runners on.

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
      serviceAccountName: runner-deployer   # required for deploying, see below
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

## Deploy Access (RBAC)

Since ARC runners run as pods inside the cluster, they authenticate against the Kubernetes API directly with a
ServiceAccount. You grant that ServiceAccount namespace-scoped permissions with a `Role` and `RoleBinding`, so
runners can only touch the namespaces you explicitly allow.

### How it works

1. A `runner-deployer` ServiceAccount lives in your runner namespace
2. A `Role` in your **app's namespace** defines what the runner may do there
3. A `RoleBinding` in the app namespace grants that role to the runner ServiceAccount (cross-namespace references are supported)
4. Your `RunnerDeployment` uses the ServiceAccount (`serviceAccountName: runner-deployer`)
5. Inside the runner pod, `kubectl` automatically uses the mounted token — no kubeconfig or secret needed

### RBAC resources

Repeat the `Role` + `RoleBinding` pair for each app namespace you want the runner to deploy to.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: runner-deployer
  namespace: <org-or-account>-runners
---
# Create one Role + RoleBinding per app namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: runner-deployer
  namespace: <app-namespace>
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: runner-deployer
  namespace: <app-namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: runner-deployer
subjects:
  - kind: ServiceAccount
    name: runner-deployer
    namespace: <org-or-account>-runners
```

That role is enough to roll out a new image and wait for it (`kubectl set image` + `kubectl rollout status`).

### Existing RunnerDeployments

If you already have a `RunnerDeployment` applied without the ServiceAccount, patch it in place:

```bash
kubectl patch runnerdeployment <repo>-runner \
  -n <org-or-account>-runners \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"runner-deployer"}}}}'
```

### Adjust permissions as needed

Expand `rules` based on what your pipelines actually do:

| Resource | apiGroup | Verbs | Use case |
|---|---|---|---|
| `deployments` | `apps` | `get`, `list`, `watch`, `patch`, `update` | Rolling out a new image and waiting for it |
| `configmaps` | core | `get`, `create`, `update` | Managing app config |
| `secrets` | core | `get` | Reading secrets in-pipeline (use sparingly) |
| `pods` | core | `get`, `list` | Checking pods and logs |
| `services` | core | `get`, `create`, `patch`, `update` | Exposing app internally |
| `ingresses` | `networking.k8s.io` | `get`, `create`, `patch`, `update` | Managing external access / TLS |

### Full access (wildcard)

If you want the runner to have unrestricted access within a namespace, you can use a wildcard rule:

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

!> **Avoid this in production.** Wildcard permissions mean a compromised runner or a buggy pipeline step could delete, overwrite, or exfiltrate anything in that namespace — including secrets. Use specific rules whenever possible and only fall back to `*` for quick experimentation or non-sensitive environments.

---

## Using Runners in Your Workflows

Set `runs-on: [self-hosted]` in your workflow jobs. A typical pipeline builds and pushes an image to Harbor, then
rolls it out with in-cluster `kubectl`:

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: [self-hosted]
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Harbor
        uses: docker/login-action@v4
        with:
          registry: harbor.guldentech.com
          username: ${{ secrets.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: harbor.guldentech.com/<project>/<image>:${{ github.sha }}

      - name: Install kubectl
        uses: azure/setup-kubectl@v4

      - name: Deploy
        run: |
          kubectl -n <app-namespace> set image deployment/<deployment> \
            <container>=harbor.guldentech.com/<project>/<image>:${{ github.sha }}
          kubectl -n <app-namespace> rollout status deployment/<deployment> --timeout=5m
```

?> Tag images with the commit SHA rather than `latest`, so each deploy is a real change Kubernetes rolls out and you
can tell which commit is running.

---

## Repo Secrets

Only Harbor credentials are needed. Add them under your repo's **Settings → Secrets and variables → Actions**:

| Secret Name | Description |
|---|---|
| `HARBOR_USERNAME` | Harbor robot user name — used to authenticate image push/pull |
| `HARBOR_PASSWORD` | Harbor robot user password |

> Ask a GuldenTech admin or email **guldentechjobs@gmail.com** if you need help retrieving these.
