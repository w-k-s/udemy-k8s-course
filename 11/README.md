# Pods

## Introduction

- Pods and containers typically have a one-to-one relationship.
- In some rare cases, there might be two containers in the same pod where the second container is a helper pod. This is quite rare and unnecessary.
- One pod should contain one service

## Writing a pod

1. Pull image

We will write a pod to run the following docker image in a pod

```
docker image pull richardchesterwood/k8s-fleetman-webapp-angular:release0
```

2. Writing a `pod.yaml` file:

```
apiVersion: v1
kind: Pod # defining a kubernets object of type Pod
metadata:
  name: webapp # pod name
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0

```

## Deploying a Pod

1. Start Minikube `minikube start`.

2. Find out the ip address of the virtual machine: `minikube ip`

3. Navigate to the directory containing the `pod.yaml` file.

4. We need to ask kubernetes to add the pod to the kubernetes cluster

5. **To see what's already present in the k8s cluster**, run:

```
$: kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   2d
```

6. **To deploy the pod to the cluster**:

```
kubectl apply -f pod.yaml
```

After a few seconds, the pod will be running:

```
NAME         READY     STATUS    RESTARTS   AGE
pod/webapp   1/1       Running   0          17s
```

- The webapp in the pod will be running inside the kubernets cluster (inside the VM). 

- One might think that they could now get the ip address of the cluster and navigate to `http://<kubernetes-ip>:80`, but this wouldn't work.

- **Pods are not intended to visible from outside the cluster. They are isolated and only accessible from within the cluster**

- **Servies**, another k8s concept, enables us to access pods outside the cluster.

## Details about the running pod

- Run `kubectl describe pod <pod name>`

```
Name:               webapp
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               minikube/10.0.2.15
Start Time:         Sat, 05 Jan 2019 20:47:26 +0400
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"webapp","namespace":"default"},"spec":{"containers":[{"image":"richardchesterwood/...
Status:             Running
IP:                 172.17.0.5
Containers:
  webapp:
    Container ID:   docker://cfd8a43b55d069f91da63deb0405e66cf54b1c3e89be2d532ad3d4d37b4af48a
    Image:          richardchesterwood/k8s-fleetman-webapp-angular:release0
    Image ID:       docker-pullable://richardchesterwood/k8s-fleetman-webapp-angular@sha256:9b98fec20772bd1d7d4c9085048f28af35b31ad3a7b7d3ba395fb512c5c359e6
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 05 Jan 2019 20:47:37 +0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-4pldl (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-4pldl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-4pldl
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/webapp to minikube
  Normal  Pulling    10m   kubelet, minikube  pulling image "richardchesterwood/k8s-fleetman-webapp-angular:release0"
  Normal  Pulled     10m   kubelet, minikube  Successfully pulled image "richardchesterwood/k8s-fleetman-webapp-angular:release0"
  Normal  Created    10m   kubelet, minikube  Created container
  Normal  Started    10m   kubelet, minikube  Started container
```

At the end, we see a list of events that occurred when starting the pod. This is a useful debugging tool.

## Connecting to the pod

```
kubectl exec <podname e.g. webapp> <bash command e.g ls>
```

```
$ kubectl exec webapp ls
bin
dev
etc
home
lib
media
mnt
proc
root
run
sbin
srv
sys
tmp
usr
var

```

---- 

```
kubectl exec <podname e.g. webapp> sh -it
```

```
$ kubectl exec webapp sh -it

/ # wget http://localhost
Connecting to localhost (127.0.0.1:80)
index.html           100% |*******************************|   585   0:00:00 ETA

/ # cat index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Fleet Management</title>
  <base href="/">

  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.png">
</head>
<body>
  <app-root></app-root>
<script type="text/javascript" src="runtime.js"></script><script type="text/javascript" src="polyfills.js"></script><script type="text/javascript" src="styles.js"></script><script type="text/javascript" src="vendor.js"></script><script type="text/javascript" src="main.js"></script></body>
</html>
```

# References

1. [API reference](https://kubernetes.io/docs/reference)