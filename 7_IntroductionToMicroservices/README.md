# Introduction to Microservices

## Introduction

**Monolith**

- Entire application is deployed as a single unit (e.g. one compiled binary/ WAR). 
- One global database for entire application (Integration Database)

_Problems_

- Difficult to coordinate
- Difficult to make changes as coordination is required between teams
- Slow deployments
- Delayed and long deployments

**Microservices**

- Modules: Self-contained components
- Each service can be developed and deployed on their own.
- Services can communicate with each other (e.g. using rest calls or grpc)

_How big should a microservice be_

- Each service should only deal with one specific area of business funcitonality (e.g. auth, orders, delivery)
- There may be some overlap in service functionality
- **Two Pizza Rule**: Heuristic measure of how big a microservice should be; Can we feed the entire development team of the microservice with two pizzas.

_Microservices should be highly cohesive and loosely coupled_

- **Highly Cohesive**: Each service should only deal with one specific area of business funcitonality
- **Lously Couples**: Where possible, we need to minimise interfaces between microservices

## Microservice Databases

- Integration Databases are bad
- Integration Database are not highly cohesive (handle multiple business functionalities)
- Integration Database are not loosely coupled (any part of the system can read write to the db )

- Each microservice must have its own datastore

_[Bounded Context](https://www.martinfowler.com/bliki/BoundedContext.html)_

- Each microservice can use the best database best suited for its role e.g. an OTAP database for analytics and a OTTP database for transactions.
- E.g. [User Service with user table with fields username, password] and [AddressBook Service with user service address]

> Microservice architecture is emergent. it is not immediately clear that the architecture is correct. Eventually you realise that one service is doing too much or that a role is more suited to a different service.

## Exercise

- Fleet tracking project.
- Each service is a docker container.

__Services__

**PositionSimulator**: 

- Simulates vehicles moving around the country. Generated positions posts latitude and longitude to the activemq container.

**Active mq**:

- Simple Docker container hosting a mq.

**PositionTracker**:

- Read positions from queue, do calculations (e.g. speed, bearing)
- Store history of where vehicles have been.

**FrontEnd**:

- Angular application

**API Gateway**:

- Backend system will change consistently.
- An API Gateway provides a consistent interface for the frontend application.
- API Gateway is single point of entry to entire backend application.

### Starting the Exercise

- Clear up your minikubectl with `kubectl delete -f pods.yaml`
- `Workloads` is the term used in k8s for the pods, deployments, replicasets and so on.

### Deploying the Queue

- **Expose 8161 on 30010**: Admin of queue.
- **Expose 61616 in cluster**: To send and receive messgaes.
- **Deployment**: want to be able to update queue safely without downtime

### Deploying the Position Simulator

- **No ports needed**: As it only send messages to the queue and does not receive anything. It does not need a Service definition, it can simply be a Deployment.

- This Deployment requires an environemnt variable to be set: `SPRING_ACTIVE_PROFILE: production_microservice`.

- As an experiment, let's misspell the environment variable value.

```
NAME                                      READY     STATUS    RESTARTS   AGE
pod/api-gateway-84f6bb8fbb-4r9sf          1/1       Running   1          1d
pod/position-simulator-77b9f8b6df-9mv8s   0/1       Error     0          18s
pod/position-tracker-86d694f997-vwjd9     1/1       Running   0          1d
pod/queue-9668b9bb4-98zjs                 1/1       Running   1          1d
pod/webapp-84654c48fc-k98gb               1/1       Running   4          1d

$ kubectl describe pod position-simulator-77b9f8b6df-9mv8s

Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  59s                default-scheduler  Successfully assigned default/position-simulator-77b9f8b6df-9mv8s to minikube
  Normal   Pulled     24s (x3 over 55s)  kubelet, minikube  Container image "richardchesterwood/k8s-fleetman-position-simulator:release1" already present on machine
  Normal   Created    24s (x3 over 55s)  kubelet, minikube  Created container
  Normal   Started    24s (x3 over 55s)  kubelet, minikube  Started container
  Warning  BackOff    4s (x3 over 36s)   kubelet, minikube  Back-off restarting failed container
```

- Let's see pod logs to find out what exactly the error is

```
$ kubectl logs position-simulator-77b9f8b6df-9mv8s

 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.0.RELEASE)

2019-01-14 18:54:59.754  INFO 1 --- [           main] c.v.s.PositionsimulatorApplication       : Starting PositionsimulatorApplication v0.0.1-SNAPSHOT on position-simulator-77b9f8b6df-9mv8s with PID 1 (/webapp.jar started by root in /)
2019-01-14 18:54:59.787  INFO 1 --- [           main] c.v.s.PositionsimulatorApplication       : The following profiles are active: production-microservice222
2019-01-14 18:55:01.140  INFO 1 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@5f4da5c3: startup date [Mon Jan 14 18:55:01 UTC 2019]; root of context hierarchy
2019-01-14 18:55:09.700  WARN 1 --- [           main] s.c.a.AnnotationConfigApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'journeySimulator': Injection of autowired dependencies failed; nested exception is java.lang.IllegalArgumentException: Could not resolve placeholder 'fleetman.position.queue' in string value "${fleetman.position.queue}"
2019-01-14 18:55:09.721  INFO 1 --- [           main] utoConfigurationReportLoggingInitializer : 

Error starting ApplicationContext. To display the auto-configuration report enable debug logging (start with --debug)


2019-01-14 18:55:09.927 ERROR 1 --- [           main] o.s.boot.SpringApplication               : Application startup failed

org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'journeySimulator': Injection of autowired dependencies failed; nested exception is java.lang.IllegalArgumentException: Could not resolve placeholder 'fleetman.position.queue' in string value "${fleetman.position.queue}"
```

- Notice we don't need to add the workload type when retrieving logs. This is because logs are only available for `Pod` workload types.

- Once we correct the environment variable, we can log in to the message queue admin panel and verify that position messages are enqueued.


### Deploying the Position Tracker

- We will expose the REST API on port 30015 to ensure it is working, and then update the yaml file to unexpose it.

```
$ curl http://192.168.99.102:30015/vehicles/City%20Truck
{
  "name": "City Truck",
  "lat": 53.379783388227224,
  "longitude": -1.4662444777786732,
  "timestamp": "2019-01-14T19:14:02.070+0000",
  "speed": 0.20170576099195223
}
```

- It works!

### Deploying the API Gateway

- We will expose the REST API on port 30020 to ensure it is working, and then update the yaml file to unexpose it.

```
$ curl http://192.168.99.102:30020
<p>Fleetman API Gateway at Mon Jan 14 19:19:52 GMT 2019</p
```

### Deploying the web app

- Expose port 80