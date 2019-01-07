# 2. Service

## 2.1 Introduction

- Pods are short-lived
- **Services** on the other hand is a long-running object, they have a IP address and a fixed port.
- Services are connected to pods.

## 2.2 Connecting services to pods

- A pod can be assigned a **label**. A label consist of a **key, value pair**.
e.g. `app:webapp`.

- Similarly, a service can be assigned a **Selector**. A selector also consists of a **key, value pair**. e.g. `app/webapp`

- A pod and a service that contain the same key-value pairs are implicitly connected to each other.

- In  both cases, the key-value pair are arbitrary, as long as all the key-value pairs match.

## 2.3 Service Types

1. **ClusterIP**

- Service will only be accessible from within the cluster. It will not be accessible from browsers, for example
- Used for microservices or any service that we do not wish to be publicly accessible.

2. **NodePort**

- Expose a port through the node (node = 1 kubernetes cluster)
- When using `type: NodePort`, the port can be specified in `spec.ports.nodePort`
- The port must be a number greater than **30,000** (Avoids collision of service ports with kubernetes internally-used ports)

## 2.4 Deploying the service

```
kubectl apply -f webapp-service.yaml
```

## 2.4 Labels and Selector are Powerful: A Case Study

- Labels and Selectors can be used as one way of updating a service with zero downtime.

- The simple way of doing it would be to update the version of the image in the Pod yaml file and then wait for the pod to reload which will result in downtime.

- A much better approach is as follows:

1. Create a new label and selector, `release: 0`.

2. Create a new pod with the updated image and label `release: 0.1`.

3. Once the pod is running, we simply update the selector of the service to `release: 0.5` and the service immediately reroutes traffic to the updated pod without any downtime.

> This is just an example. A common pattern to upgrade a pod without downtime will be covered later in the course.


