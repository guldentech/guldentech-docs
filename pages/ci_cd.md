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
      serviceAccountName: <service-account-name>  # optional — see Advanced section
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
          tags: harbor.guldentech.com/<project>/<image>:latest
```

---

## Repo Secrets

Most workflows will need secrets for Harbor and Rancher. Add them under your repo's **Settings → Secrets and variables → Actions**:

| Secret Name | Description |
|---|---|
| `HARBOR_USERNAME` | Harbor robot user name — used to authenticate image push/pull |
| `HARBOR_PASSWORD` | Harbor robot user password |
| `RANCHER_TOKEN` | Rancher API bearer token — used to trigger deployments |

> Ask a GuldenTech admin or email **guldentechjobs@gmail.com** if you need help retrieving these.

---

## Advanced — In-Cluster kubectl Access via RBAC

?> This approach is experimental. It skips the need for a `RANCHER_TOKEN` secret by leveraging the runner pod's mounted service account token for in-cluster `kubectl` access.

Since ARC runners run as pods inside the cluster, they can authenticate against the Kubernetes API directly using a Kubernetes ServiceAccount — no external token required. You grant that ServiceAccount namespace-scoped permissions via `Role` and `RoleBinding`, so runners can only touch the namespaces you explicitly allow.

### How it works

1. A dedicated `ServiceAccount` is created in your runner namespace
2. A `Role` is created in your **app's namespace** defining what actions are allowed
3. A `RoleBinding` in the app namespace binds the role to the runner ServiceAccount (cross-namespace references are supported)
4. The `RunnerDeployment` is updated to use the ServiceAccount
5. Inside the runner pod, `kubectl` automatically uses the mounted token — no kubeconfig or secret needed

### RBAC resources

Apply these to set up access. Repeat the `Role` + `RoleBinding` pair for each app namespace you want the runner to be able to deploy to.

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
    verbs: ["get", "list", "patch", "update"]
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

### Update your RunnerDeployment

If you already have a `RunnerDeployment` applied, patch it in place:

```bash
kubectl patch runnerdeployment <repo>-runner \
  -n <org-or-account>-runners \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"runner-deployer"}}}}'
```

For new deployments, include `serviceAccountName` directly in your YAML — see the template in the [Self-Hosted Runners](#self-hosted-runners) section above.

### Adjust permissions as needed

The example above grants access to `deployments` only. Expand `rules` based on what your pipelines actually do:

| Resource | apiGroup | Verbs | Use case |
|---|---|---|---|
| `deployments` | `apps` | `get`, `patch`, `update` | Rolling out a new image |
| `configmaps` | core | `get`, `create`, `update` | Managing app config |
| `secrets` | core | `get` | Reading secrets in-pipeline (use sparingly) |
| `pods` | core | `get`, `list` | Checking rollout status |
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
