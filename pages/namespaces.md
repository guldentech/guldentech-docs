# Namespaces and limits

Each app lives in a Kubernetes **namespace** inside your Rancher project. Every namespace starts from the same
safe defaults. You can't create namespaces or change these rules in Rancher. They live in the `guldentech-k8s` repository, and you
change them with a pull request that a GuldenTech admin merges. Only these settings go through a pull request. Your
Deployments, Services and app code are yours: you apply them yourself, and nobody reviews or merges them.

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

Because everything is denied by default, a connection your app needs may be blocked until it's allowed. These are the
exceptions a namespace can have (each is one line in its `config.yaml`, see below):

| You need | Add to `config.yaml` |
|---|---|
| Your app calls the internet (an API, a package download) | `allowInternet: true` (public addresses only; never the home network) |
| Your app talks to another namespace (a database, Harbor, RustFS) | `allowTo: [that-namespace]` |
| Another namespace calls your app | `allowFrom: [that-namespace]` |
| Your app uses the Kubernetes API (an operator, a controller) | `allowApiServer: true` |
| Visitors reach your site | `allowFrom: [cloudflare-tunnel-ingress]`, see [Hostnames and domains](/pages/ingress) |

?> A pod that has just started can be refused for up to a minute, until the cluster learns its address. If a
connection fails right after a deploy, wait a minute and try again before changing anything.

## Making a change (pull request)

You need a GitHub account in the **guldentech** organization (we invite you when you're onboarded). The repository,
[guldentech/guldentech-k8s](https://github.com/guldentech/guldentech-k8s), is private and you can't push to it
directly, so you propose changes from a fork.

1. **Fork** `guldentech/guldentech-k8s` on GitHub.
2. In your fork, edit your namespace's file, `clusters/gt-production-1/<your-project>/<namespace>/config.yaml`. Only
   list what differs from the defaults. For a **new namespace**, create the folder and the file:
   ```yaml
   networkPolicies: {}
   ```
   Then add what the app needs, for example:
   ```yaml
   allowInternet: true            # my app calls api.example.com
   allowTo: [rustfs]              # and stores files in RustFS
   quota: { limits.memory: 4Gi }  # more memory than the default quota
   ```
3. **Open a pull request** against `guldentech/guldentech-k8s`. Say what you changed and why.
4. A GuldenTech admin checks that it only changes your namespace's settings, and merges it. When it's merged,
   the change is applied to the cluster within a minute or two.

`<your-project>` is the folder named after your Rancher project, and `<namespace>` your namespace's name; look at a
neighbouring folder for an example. Only change files under your own project folder: a pull request that touches another
project or the shared files at the top (`defaults.yaml`, `main.tf`, the workflows) won't be merged.
