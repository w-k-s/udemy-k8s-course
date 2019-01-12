# Replica Sets

## Introduction

- Pods are rarely worked with directly in production. Instead, we use **ReplicaSets** or **Deployments**.

- Pods are short-lived. If a pod is created manually, the the architect is responsible for ensuring the life-time of the Pod and the wellness of the pod. If the pod goes down, it will not be recreated.

- To test this, we can delete a pod:

```
kubectl delete pod webapp-release-0-5
```

The pod will not be recreated.

## Replica Set

- A `ReplicaSet` is a small wrapper around a pod specifying how many replicas of the pod should exist at any given time.

- For example, if we set the number of replicas to 1, k8s will ensure that there is always one replica of the pod that is running.

- Every microservice pod should be part of a replica set.

- A `ReplicaSet` is referenced from the `Service`, instead of a `Pod`.

## Defining a ReplicaSet

ReplicaSets and Pods are defined in the same file. The Pod specification is nested in the ReplicaSet specification


```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: webapp
spec:
    replicas: 1
    selector:
        matchLabels:
            app: webapp
    template:
        metadata:
            labels:
                app: webapp
        spec:
            containers: 
            - name: webapp
              image: richardchesterwood/k8s-fleetman-webapp-angular:release0

```

- When defining a replica set, a pod does not need to have a name. The pod will acquire the name of the replica set it belongs to.
- The ReplicaSet needs a selector so that it can match with the pod.
- The pod still needs the label in order to match with a service.

## Running the ReplicaSet

```
kubectl apply -f .
```

- If we have one replica for a pod and we delete the pod, a new replica of the pod will immediately be initiated.
- It will take some time for the replica to be ready so there will be some downtime.

- If we we have two replicas for a pod and we delete one pod, there will be no downtime as there is still one more to hanlde requests.

- It is the job of the architect to decide how many replicas of a given service are required e.g. the frontend for example would need 2 replicas.

## Tips

**Delete all pods**: `kubectl delete pod --all`
**Describe ReplicaSet**: `kubectl describe replicaset webapp`, `kubectl describe rs webapp`