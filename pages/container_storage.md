# Container storage (Harbor)

Guldentech provides a container storage option for its users. This is leveraging [Harbor](https://goharbor.io/).

## Projects

When you onboard you will get a project and a `robot` user. You will use the `robot` credentials to push and pull images.

## Creating image pull secret

You can create your pull secret on rancher by running the following `kubectl` command. Create this secret in all namespaces that need access. A Guldentech admin will provide you with your username and password.

```bash
rancher kubectl create secret docker-registry guldentech-harbor-registry \
    --docker-server=harbor.guldentech.com \
    --docker-username=UPDATE_HERE \
    --docker-password=UPDATE_HERE \
    -n UPDATE_HERE
```

Then reference it from your pods with `imagePullSecrets: [{name: guldentech-harbor-registry}]`
(see [Getting started](/pages/getting-started)).

## Vulnerability scanning

Harbor scans every image you push (and re-scans everything weekly, on Sundays) for known vulnerabilities. Open your
project in [Harbor](https://harbor.guldentech.com/), click an image tag, and the scan result is on that page, with the
package and the version that fixes each finding. Most findings go away by rebuilding on a newer base image.

Scanning only reports. It doesn't block a push or a deploy.

