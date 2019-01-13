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