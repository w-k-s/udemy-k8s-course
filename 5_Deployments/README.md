# Deployments

- Deployments are similar to ReplicaSets in that they allow you to specify how many replicas of a pod there should be at any given moment.
- However, Deployments have two additional important features: **updates without downtime** and **rollbacks**

## Defining a Deployment

The Definiton of a `Deployment` is very similar to the definition of a `ReplicaSet`. So much infact that we can copy our earlier definition of a `ReplicaSet` and simply replace the kind with `Deployment`.


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

## Running the Deployment

```
kubectl apply -f .
```

- When we run a `Deployment`, a corresponding `ReplicaSet` will automatically be created. In other words, `Deployments` manage `ReplicaSet`s for you.

- If pods in a deployment do not respond or fail to set up, then the deployment will not succeed. The deployment will try recreate the pod in an infinite loop

## Updating the Pod

- We can updae the image in the deployment

```
image: richardchesterwood/k8s-fleetman-webapp-angular:release0 -> 
image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5
```

- When we do this, the `Deployment` will create a new `ReplicaSet` with the updated pods.
- Once the updated pods respond to requests, the replica count on the old replica set will be set to 0.
- **The old ReplicaSet will not be removed. This is so to making rolling back to the previous version possible**.

## Keeping track of the rollout

- We can monitor the state of a rollout with the folloign command:

```
kubectl rollout status deployment <deployment-name>
```

e.g. 

```
kubectl rollout status deployment webapp
```

Sample output:

```
Waiting for rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for rollout to finish: 1 old replicas are pending termination...
Waiting for rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
```

## Rollout history

```
kubectl rollout history deployment webapp
```
Sample output:

```
deployments "webapp"
REVISION  CHANGE-CAUSE
3         <none>
4         <none>
```

This shows that the current revision is 4 and the one before this was 3.
We can go back to a previous revision using the command

```
kubectl undo deployment webapp
```

To rollback to a specific revision, we can add a flag

```
kubectl undo deployment webapp -to-revision=3
```

- The undo command is not recommneded because the state of the live running kubernetes system does not match the state described by the yaml files. It is only suitable for emergency situations.