# Rancher Pipeline YAML

Your repo will need a .rancher-pipeline.yaml file which will facilitate the pipeline. 

Below is a basic Build and deploy example.

```yaml
stages:
- name: Publish
  steps:
  - publishImageConfig:
      dockerfilePath: ./Dockerfile
      buildContext: .
      tag: image-name:v1.0
    env:
      PLUGIN_MTU: "1230"
- name: Deploy
  steps:
  - applyYamlConfig:
      path: ./deploy.yaml
```

Above defines it will first build an image from the "Dockerfile" file at the root of your repo, push it to the local registry with the name image-name:v1.0. Following that, it will run kubectl apply -f deploy.yaml. 

!> Make sure your namespace is defined in your deploy file.

## Deploy config settings

In your deploy.yaml file you will need to add three special sections. 

1. Pipeline secrets
2. Image name
3. Git Commit

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: my-namespace
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
    imagePullSecrets:           # <----- 1.
      - name: pipeline-docker-registry
      containers:
      - name: nginx
        image: ${CICD_IMAGE}:v1.0 # <----- 2.
    env:
        - name: CICD_GIT_COMMIT   # <----- 3.
          value: "${CICD_GIT_COMMIT}"
        ports:
        - containerPort: 80
```