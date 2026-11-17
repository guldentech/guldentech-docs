# Rancher

**Open Rancher: [https://console.guldentech.com](https://console.guldentech.com)** and sign in with GitHub.

?> Rancher moved here from `rancher.dropbear-eel.ts.net` in November 2026. If you set up the CLI or downloaded a
kubeconfig before that, log in again with the new address below, or download a fresh kubeconfig.

!> Login only works once you've been onboarded: an admin adds your GitHub account to your own Rancher project. Until
then, Rancher refuses the login. Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com) with your GitHub
username to get set up.

Rancher is the web console for the GuldenTech Kubernetes clusters. Each user gets their own Rancher project, and your
Kubernetes namespaces live inside it. You can use the console in your browser, or work from your terminal
with the Rancher CLI.

## Rancher CLI (recommended)

The [Rancher CLI](https://ranchermanager.docs.rancher.com/reference-guides/cli-with-rancher/rancher-cli) logs in
with an API key and runs `kubectl` through Rancher, so there's no kubeconfig file to download or edit.

1. Install it: `brew install rancher-cli` on macOS, or download it from the
   [Rancher CLI releases](https://github.com/rancher/cli/releases).
2. In the Rancher console, click your avatar (top right) → **Account & API Keys** → **Create API Key**. Give it a
   description and an expiry, and copy the **Bearer Token**. It's shown only once.
3. Log in, and pick your project when asked:
   ```bash
   rancher login https://console.guldentech.com --token <bearer-token>
   ```
4. Use it like `kubectl`:
   ```bash
   rancher context current                  # which cluster and project you're on
   rancher context switch                   # pick a different one
   rancher kubectl get pods -n <namespace>
   rancher kubectl logs deploy/<name> -n <namespace>
   rancher kubectl apply -f app.yaml -n <namespace>
   ```

?> Treat the token like a password. If it leaks, delete it under **Account & API Keys**.

## kubectl with a kubeconfig file

If a tool needs a real kubeconfig (for example `helm`), log into Rancher, open the cluster, and click the
Kubeconfig file button shown below.

![dashboard](../_media/dash.png)

The file works as downloaded, with no edits needed. Save it somewhere private (for example `~/.kube/guldentech.yaml`)
and point tools at it with `--kubeconfig` or `KUBECONFIG`. It contains a token that expires after 30 days, so treat it
like a password and download a new one when it stops working.

## Namespaces

You can't create namespaces in Rancher: they're set up by GuldenTech together with their limits and network rules.
Add one with a pull request, see [Making a change](/pages/namespaces?id=making-a-change-pull-request). Once it's merged
it shows up under your project in Rancher.

## What your role allows

You have the **Tenant** role in your project. You can run apps, read logs, open a shell in a pod and manage your own
Secrets, ConfigMaps, volumes and Roles. You can only look at Ingresses, network policies, quotas and events. You can't
create namespaces, change network rules or hostnames (those are pull requests, see
[Namespaces and limits](/pages/namespaces?id=making-a-change-pull-request)), or add people to your project.
