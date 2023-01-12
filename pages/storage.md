# Storage

GuldenTech provides two types of storage.

1. local-path storage class
2. longhorn distributed block storage

Depending on your choice, you will have different availability. Read below for availability insight.

!> Email GuldenTech admins if you are still not sure one which storage class to use for your persistent data. [guldentechjobs@gmail.com](mailto:guldentechjobs@gmail.com)

!> All GuldenTech physical servers that hold storage are running 15k SAS drives in RAID 10.

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

##  Longhorn distributed block storage

For apps that need high uptime, please create a longhorn pvc

Example:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

!> Longhorn will create two replicas of your PVC and save them on different nodes. So if node `A` goes down, your pod can easily move to node `B` and start running with the same data available.

!> Warning for statefulsets: If workload is a statefulset, you will need to set the `terminationGracePeriodSeconds` to 0. The reason being is, statefulset pods will not be rescheduled unless the orginal pod is terminated. Setting it to 0 will forefull delete the pod on the node that is down, allowing it to spawn on the other node.

!> Although I have no reason to believe data loss will occur with `lognhorn`, I beleive if the data you are persiting can not be lost or corrupted, the best bet would be to use `local-path`
