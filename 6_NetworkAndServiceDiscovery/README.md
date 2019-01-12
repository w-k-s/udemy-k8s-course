# Networking & Service Discovery

## kube-dns

- If we were to deploy two containers in the same pod (e.g. a webapp container anad a mysql container), the two containers would be able communicate directly with each other on localhost.

- This is not recommneded as having a webapp container and a database container on the same pod would make the pod more difficult to manage (e.g. lack of sandboxing)

- It is recommended, as mentioned earler, that one pod should have one container, and services and pods typically have a one-to-one relationship.

- Each service has its own IP address. So how can the webapp service get hold of the IP address of the database service?

- The solution is Kubernetes' own DNS Service (`kube-dns`). This is a dictionary that maps the service names to ip addresses.

```
webapp		10.10.120.10
database	11.12.121.20
```

- The DNS enables services to communicate by using the service name as the hostname e.g. `database`. The hostname will be resolved by the dns

```
postgres://database:5432
```

---

## Namespaces

- If we run `kubectl get all`, we do not see the `kube-dns` service listed in the output

- This is because k8s only shows pods, services, deployments for a given namespace. e.g. we can separate pods relating to the frond and pods relating to the backend.

- By default, pods are added to the `default` namespace.

- To view all the namespaces, run `kubectl get namespace` or `kubectl get ns`

```
NAME          STATUS    AGE
default       Active    9d
kube-public   Active    9d
kube-system   Active    9d
```

- We can view the pods within a particular namespace by running: `kubectl get service -n kube-system`

```
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   9d
kubernetes-dashboard   ClusterIP   10.109.206.132   <none>        80/TCP          9d
```

- This applies to other commands as well, **To run a command on pods in a different namespace, the namespace must be specified**

```
kubectl describe svc kube-dns -n kube-system
```

## Exercise

- Apply the files in this folder `kubectl apply -f .`

- Once the database container is `Running`, we can enter the shell of the `webapp` and try to connect to the `database` container.

```
$ kubectl get all
NAME                          READY     STATUS    RESTARTS   AGE
pod/mysql                     1/1       Running   0          5m
pod/queue                     1/1       Running   2          4d
pod/webapp-74bd9697b4-9frpw   1/1       Running   1          15h
pod/webapp-74bd9697b4-fzd9g   1/1       Running   2          15h

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/database          ClusterIP   10.101.87.196   <none>        3306/TCP         5m
service/fleetman-queue    NodePort    10.98.215.77    <none>        8161:30010/TCP   5d
service/fleetman-webapp   NodePort    10.99.27.215    <none>        80:30080/TCP     5d
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP          9d

NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webapp   2         2         2            2           15h

NAME                                DESIRED   CURRENT   READY     AGE
replicaset.apps/webapp-7469fb7fd6   0         0         0         15h
replicaset.apps/webapp-74bd9697b4   2         2         2         15h

$ kubectl exec -it webapp-74bd9697b4-9frpw sh

# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5


# nslookup database
nslookup: can't resolve '(null)': Name does not resolve

Name:      database
Address 1: 10.101.87.196 database.default.svc.cluster.local
```

- **`cat /etc/resolv.conf`**: On a linux system, the `resolve.conf` stores the ip address of the dns. Look earlier and you'll see that that is the ip address of `kube-dns`

- **`nslookup database`**: This command returns the resolved ip of the hostname. Notice how it returns the ip of the database pod returned by `kubtectl get all`. 

- **`database.default.svc.cluster.local`**: This is the Fully qualified domain name of the service

## Fully Qualified Domain Names

- Fully qualified domain name is the full url of a service.
- Note that fully qualified domain name contains the namespace: `database`.**`default`**`.svc.cluster.local`.
- If we wanted to discover a service within a different namespace, we would have to write `database.<namespace>` instead of `database` in order for the service ip to be resolved.


