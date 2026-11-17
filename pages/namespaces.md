# Namespaces and limits

Each app lives in a Kubernetes **namespace** inside your Rancher project. Every namespace starts from the same
safe defaults. You can't create namespaces or change these rules yourself: you ask us, we change them in the
`guldentech-k8s` repository, and the change is reviewed before it goes live.

## Defaults

| What | Default |
|---|---|
| Pod security | `baseline`: no privileged containers, host networking or ports, or `hostPath` volumes |
| Default limits | A container that sets no resources gets 50m CPU / 64Mi memory requested and a 512Mi memory limit |
| Quota (per namespace) | 1 CPU and 1Gi memory requested, 2Gi memory limits, 10 pods, 10 volumes, 50Gi storage |
| Network | Deny everything, then allow DNS and traffic between pods in the same namespace |

Set `resources` on your containers. A pod that asks for more than the quota allows is refused with a clear message in
`kubectl describe`.

## Network rules

Because everything is denied by default, a connection your app needs may be blocked until we allow it. These are the
exceptions we can add to a namespace:

| You need | Ask for |
|---|---|
| Your app calls the internet (an API, a package download) | internet access (public addresses only; never the home network) |
| Your app talks to another namespace (a database, Harbor, RustFS) | access to that namespace |
| Another namespace calls your app | access from that namespace |
| Your app uses the Kubernetes API (an operator, a controller) | API server access |
| Visitors reach your site | nothing: publishing a hostname opens it |

?> A pod that has just started can be refused for up to a minute, until the cluster learns its address. If a
connection fails right after a deploy, wait a minute and try again before asking us.

## Asking for a change

Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com) with the namespace and what you need, for example:
"`myapp` needs to call `api.example.com` over the internet", or "I need 3 CPU of quota for a build job".
We'll tell you when it's live.

## What you can do in your namespaces

You have the **Tenant** role in your Rancher project. You can run apps (Deployments, StatefulSets, Jobs, CronJobs,
Services, ConfigMaps, Secrets, volumes), read logs, open a shell in a pod, and create Roles and RoleBindings (for a CI
service account, for example) limited to permissions you already hold.

You can only read: Ingresses, network policies, quotas, certificates and events. You can't create namespaces, write
network policies or Ingresses, add project members, or create clusters.
