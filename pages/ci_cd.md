# Continuous integration and continuous delivery (CICD)

We have switched to using Github actions for CICD. Setup for this is easy, you will need Helm and a GH access token. 

If you plan to use a private repo on guldentech infra, you will need to deploy self hosted runners to the cluster.

## Self hosted runners

We leverage self hosted runners using [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)

To deploy, its very simple, create a namespace on the rancher UI, create a fine grained github access token with the permissions [defined here](https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md) and run the following commands:

```bash
# Create secret for GH PAT
export gh_name=rgulden; \
export gh_pat=XXX; \
kubectl create secret generic controller-manager \
    -n $gh_name-actions-runner-system \
    --from-literal=github_token=$gh_pat

# Add helm chart
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

# Install
export gh_name=rgulden; \
helm upgrade \
    --install \
    --namespace $gh_name-actions-runner-system \
    --wait \
    "$gh_name-actions-runner-system" \
    actions-runner-controller/actions-runner-controller \
    --set syncPeriod=1m
```

## Creating deployment

Below is an example on creating a scale to zero runner for the `guldentech.com` repo. Make sure to update accordingly to your configuration. If you plan to get more detailed, please visit the `ARC` docs on their github repo.

!> Please using scaling to zero for your runners.

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: guldentech-k8s-action-runner
spec:
  replicas: 1
  template:
    spec:
      repository: guldentech/guldentech.com
      labels:
        - "guldentech"
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: guldentech-k8s-action-runner-autoscaler
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: guldentech-k8s-action-runner
  minReplicas: 0
  maxReplicas: 2
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - guldentech.com
```


## Repo secrets

You most likely will need to add your harbor robot user secret and rancher API token secret to the repo where you want actions to run. The rancher secret to add is the bearer token in most cases.

Ask guldentech admins for help on this if needed.