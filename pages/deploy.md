# Example Deployment


Create a file called "deploy.yaml" and copy the below yaml into it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```


To apply the deployment resource to your project run the following.

```bash
# Make sure your_namespace is created!
kubectl apply -f deploy.yaml -n your_namespace
```