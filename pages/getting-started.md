# Getting started

How an app gets from your laptop to a public website on GuldenTech.

```
visitor  →  Cloudflare  →  tunnel  →  your Service  →  your pods
```

Cloudflare answers the visitor (HTTPS, caching), a tunnel carries the request into the cluster, and Kubernetes sends it
to your app. You never deal with IP addresses or certificates.

## What you do, and what we do

| You | We (GuldenTech) |
|---|---|
| Build your image and push it to Harbor | Create your Rancher project and Harbor project, and invite you to GitHub |
| Write the Deployment and Service for your app | Review and merge your pull requests |
| Open pull requests for your namespace, its network rules and your hostname | Run the cluster, TLS, DNS and backups |
| Deploy it (by hand or from GitHub Actions), and watch it | Keep the status page up |

Namespaces, network rules and hostnames are kept in one place, the `guldentech-k8s` repository. You change them with a
pull request, we review and merge it, and the cluster follows. Nobody edits them by hand in Rancher.

## 1. Get onboarded

Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com) as described on the [home page](/). You get:

- a Rancher project, and a login with GitHub ([Rancher](/pages/rancher))
- a Harbor project and a `robot` user to push and pull images ([Container storage](/pages/container_storage))
- an invitation to the **guldentech** GitHub organization, so you can open pull requests
- your project folder in `guldentech-k8s`, where your namespaces are defined ([Namespaces and limits](/pages/namespaces))

## 2. Build and push your image

```bash
docker build -t harbor.guldentech.com/<project>/<app>:<tag> .
docker login harbor.guldentech.com        # robot user and password
docker push harbor.guldentech.com/<project>/<app>:<tag>
```

Harbor scans every image you push and lists known vulnerabilities in its web page.

## 3. Create your namespace and deploy

Open a pull request that adds your namespace ([Namespaces and limits](/pages/namespaces?id=making-a-change-pull-request)). Once it's merged, create the image pull secret once per namespace ([Container storage](/pages/container_storage)), then apply a
Deployment and a Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: <your-namespace>
spec:
  replicas: 1
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      imagePullSecrets:
        - name: guldentech-harbor-registry
      containers:
        - name: myapp
          image: harbor.guldentech.com/<project>/<app>:<tag>
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits: { memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: <your-namespace>
spec:
  selector: { app: myapp }
  ports:
    - port: 80
      targetPort: 8080
```

```bash
rancher kubectl apply -f app.yaml
rancher kubectl -n <your-namespace> get pods
```

!> Your namespace has [limits](/pages/namespaces): pods can't run privileged or use host paths, the namespace has a
CPU, memory and pod quota, and it can only talk to the outside world through what its `config.yaml` allows.

## 4. Get a hostname

Add it to your namespace's `config.yaml` in a pull request, with the Service name and port from step 3. A minute or
two after it's merged, `https://<hostname>` works. Details, and what to do for your own domain, are in
[Hostnames and domains](/pages/ingress).

!> Don't create an Ingress yourself, and don't keep one in your app's repository. You don't have permission to write
Ingresses (it would be refused), and a deploy that tries will fail.

## 5. Deploy automatically

[CI/CD](/pages/ci_cd) shows how to build, push and roll out from GitHub Actions on our self-hosted runners.

## When something looks wrong

1. [status.guldentech.com](https://status.guldentech.com/) shows whether the platform itself has a problem.
2. `rancher kubectl -n <your-namespace> describe pod <pod>` and `logs` show why your pod isn't running.
3. A connection that is refused or times out is usually a [network rule](/pages/namespaces?id=network-rules): add the
   missing line to your `config.yaml` in a pull request.
4. Still stuck? Email [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com).
