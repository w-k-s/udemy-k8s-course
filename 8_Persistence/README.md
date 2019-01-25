# 8. Persistence

- When a pod is destroyed, its data is deleted.
- This data can be persisted using `volumes` or a cloud storage system

## 8.1 `MongoService`

- We introduce a mongodb service.
- The tracker will not only receive the position of a vehicle from the queue, but it will also save it to the mongodb
- Ideally, storing the vehicle history should be done by a seperate microservice (possibly called the `Telemtry` service).

---

## 8.2 Expanding the Minikube VM

- In this example, we update our position tracker to `release3`, which keeps track of the vehicle history. 
- The front-end receves the history and draws a track that the vehicle has travelled.
- This history is stored in a new `mongodb` pod.
- Once we have all these pods running (`position-simulator`,`position-tracker`,`api-gateway`,`webapp`,`queue` and `mongo`), the Virtual Machine falls short on memory.
- In case this happens, we can expand Minikube VM.

1. Power down Minikube (`minikube stop`)
2. Once powered down, launch `VirtualBox`
3. Right-click on the Minikube VM and press **Settings** > **System**
4. Increaase the RAM allocated e.g. 4GB
5. Restart minikube.

---

## 8.3 Volume Mounts

- Mongodb data is stored inside the mongodb container.
- If we delete the mongodb Pod, we lose all the vehicle history.
- It is desired that the vehicle history is stored permanently.
- One way to do this in Docker is to use volume mounts: **allows mapping a folder inside the container to a folder inside your host filesystem.**
- A similar concept exists in Kubernetes, called **Persistent Volumes**.

To define a Volume mount in the yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie 
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
```

- For the mongodb container, we define a volumeMount and a volume.

### 8.3.1 VolumeMount

- The `name` is used to map the `volumeMount` to a `volume`, which is a physical storage location. This could be a local filesystem or an AWS Elastic Block Storage (EBS is basically a hard disk on the cloud)

- The `mountPath` is the folder where mongodb stores its data in the container. 

### 8.3.2 Volume

- Volumes specify how volume mounts should be implemented.
- There are many kinds of [volume types](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie 
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      volumes:
        - name: mongo-persistent-storage
          hostPath: # specifies how mounting should be implemented ()
            path: /mnt/mydata/mydb # path in Linux version machine
            type: DirectoryOrCreate # If nothing exists at the given path, an empty directory will be create

```
- A hostPath volume mounts a file or directory from the host node’s filesystem into your Pod. This is not something that most Pods will need, but it offers a powerful escape hatch for some applications..

- To view the created directry structure, open VirtualBox, click on Minikube, and login to the shell using **default** credentials:

```
Username; docker
Password: tcuser
```

Once done, you should be able to navigate to /mnt/mydata/mydb

## 8.4 PersistentVolumeClaims

- In the previous example, we defined a storage volume specific to one pod.
- But what if we wanted to share a storage volume between multiple pods?
- **PersistentVolumeClaims** is a pointer to a storage volume implementation. This pointer is referenced by a pod.
- We replace the `hostPath` definition earlier with a `persistentVolumeClaim` pointer as follows:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie 
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      volumes:
        - name: mongo-persistent-storage
          persistentVolumeClaim: 
            claimName: mongo-pvc
```

- The claim name references a **`PersistentVolumeClaim`** definition which is ideally in a seperate file.
- The file defines two resources: **`PersistentVolumeClaim`** which defines the kind of storage we want and **`PersistentVolume`** which defines how we want the storage to be implemented.
- The `claimName` should match the `metadata.name` in the `PersistentVolumeClaim` definition.


- Both **`PersistentVolumeClaim`** and **`PersistentVolume`** require an `accessModes` definition, usually `ReadWriteOnce`
- Both **`PersistentVolumeClaim`** and **`PersistentVolume`** must also specify `capacity`. The capacity of the **`PersistentVolume`** must be equal to or greater than the storage requested in the **`PersistentVolumeClaim`**.
- Both **`PersistentVolumeClaim`** and **`PersistentVolume`** must have a `storageClassName`. `StorageClassName` allows a system administrator to define classes of storage e.g. Hard Disk, SSD, cloud etc. 
In order for a `PersistentVolume` to be mapped to a `PersistentVolumeClaim`, they must have the same `storageClassName`, the same `accessMode` and the `capacity` of the `persistentVolume` must be greater than or equal to the one requested by the `persistentVolumeClaim`.

### 8.4.1 Deploying a PersistentVolumeClaim

- `PersistentVolumes` will not appear when you run the command `kubectl get all`. This is because `PersistentVolume`s are resources.

- To view the `PersistentVolumeClaim`, use the command `kubectl get pv`


```
$ kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM               STORAGECLASS          REASON    AGE
local-storage   20Gi       RWO            Retain           Bound     default/mongo-pvc   local-storage-class             5m
```

```
$ kubectl get pvc
NAME        STATUS    VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS          AGE
mongo-pvc   Bound     local-storage   20Gi       RWO            local-storage-class   3m
```

We can also see the persistentVolumeClaim when we describe the pod:

```
$ kubectl describe pod/mongodb-747b7bfcc9-rtvn5
Name:               mongodb-747b7bfcc9-rtvn5
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               minikube/10.0.2.15
Start Time:         Fri, 25 Jan 2019 17:02:05 +0400
Labels:             app=mongodb
                    pod-template-hash=747b7bfcc9
Annotations:        <none>
Status:             Running
IP:                 172.17.0.11
Controlled By:      ReplicaSet/mongodb-747b7bfcc9
Containers:
  mongodb:
    Container ID:   docker://bc4b708a38b148811ad5c38390e711bfc04684da1f968120273bcc92aaf0eb1a
    Image:          mongo:3.6.5-jessie
    Image ID:       docker-pullable://mongo@sha256:3e00936a4fbd17003cfd33ca808f03ada736134774bfbc3069d3757905a4a326
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 25 Jan 2019 17:02:06 +0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /data/db from mongo-persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-445gl (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  mongo-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mongo-pvc
    ReadOnly:   false
  default-token-445gl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-445gl
    Optional:    false
QoS Class:       BestEffort
```