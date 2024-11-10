# Storage

GuldenTech provides only one type of storage.

1. local-path storage class

## Local path

Local path creates a pvc that is linked to a folder on the server, each time the pod starts it will be tied to that node. To use this storage class, refrence the example below.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

!> If you use local path, please note that if the node goes down, your pod will not move nodes, it will be in pending state until the node comes back online. Your pod is tied to the node the local-stoage is created on.

!> We do offer data backup! This is offered through [k8up](https://k8up.io/). Please read through the documentation on how to setup your backup job. We very much suggest to use [schedules](https://docs.k8up.io/k8up/2.11/how-tos/schedules.html).