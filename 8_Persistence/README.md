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