# Habor container storage

Guldentech provides a container storage option for its users. This is leveraging [Harbor](https://goharbor.io/).

## Projects

When you onboard you will get a project and a `robot` user. You will use the `robot` credentials to push and pull images.

## Creating image pull secret

You can create your pull secret on rancher by running the following `kubectl` command. Create this secret in all namespaces that need access. A Guldentech admin will provide you with your username and password.

```bash
kubectl create secret docker-registry guldentech-harbor-registry \
    --docker-server=harbor2.guldentech.com \
    --docker-username=UPDATE_HERE \
    --docker-password=UPDATE_HERE \
    -n UPDATE_HERE
```
