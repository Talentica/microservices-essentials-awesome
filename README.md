
Recently I worked on a project where we set up microservice architecture. Through this article, I would like to share the experience and the approaches we took to build a microservice application.


# Defining a boundary for microservice

This involves how to break down the large system into its microservice. When we started, a lot of approaches came to mind like Should we break down based on lines of code, team geography, I/O bound component can benefit from the nonblocking I/O of technology such as Node.js and therefore they are a subsystem of their own, computationally heavy services need to be written in C or Rust or Go and therefore they are a separate subsystem?

To define the boundary of a microservice we took Domain-Driven design approach. Eric Evans, in his wonderful book **Implementing Domain-Driven Design**, explains very well how to define boundary based on the business model. Reading Eric Evans' **Implementing Domain-Driven design** book is highly recommend if you are going to set up microservice architecture. He also wrote **Domain-Driven Desing Distilled**, this book summarizes the whole concept of DDD and is perfect reference material for those who have less time but still wants to grasp the concept of DDD. There are other recommended articles which I would like to share

- [Tutorials DDD](https://vaadin.com/tutorials/ddd)

In the project, we have only used bounded context to define the boundary for microservice. Other concepts of DDD like aggregate, domain service, domain event, etc are not being used in the project because the project demands CRUD, incorporating DDD in the whole application would be overkill.

# Communication between the service.

Communication between the service.

- ***Synchronous***: With synchronous communication, a call is made to a remote server, which blocks until the operation completes.
- ***Asynchronous***: With asynchronous communication, the caller doesn’t wait for the operation to complete before returning, and may not even care whether or not the operation completes at all.

We have used both styles of communication in our project. In general, asynchronous communication is suitable for the long-running process and synchronous is suitable when instance response is required.

### Synchronous
We used **Feign library** for defining both server-side and client-side interface for REST base synchronous call between the services. Each service is a multi-module java project. One of the modules is the client module which defines the interface or API contract. The service uses the module for implementing the service and the client uses the same module to call the service. 
***CAUTIONS:*** *Ideally, the client should not depend upon the external library for calling the other service. This will make the service more coupled.* Before moving to production this dependency should be removed and each service should define its code to call another service.
The approach we choose helps us to quickly get to the point where we can show the feature to the client and get the feedback. But we were planning to remove those dependencies before going live.

### Asynchronous

We used **RabbitMQ** for enabling asynchronous calls. Whenever a major thing happens, a service generates an event and the listeners to that event can take appropriate action. For example, let say we have two service one is *User service*, and another is *Field Service*. The tenant registration will happen in *User service* and once the tenant is registered the field definition for that tenant will happen in *Field Service*. Here, once a tenant is successfully registred, *User service* will generate an event **tenant.created**. The *Field service* is listening to **tenant.created** event and takes appropriate action. The set up we did in our application is as follows :

During service boot up, it checks if its exchange and its queue are set up or not. If it is not set up, it creates exchange and queue for that service. The exchange is used to publish the event and queues is used to persist messages for that service which the service is interested in.

# Packing and deployment

We are packaging our services in docker images, and for orchestration, we are using Kubernetes. The Kubernetes helps us in scaling the application. The Kubernetes resources packaging is done using helm charts which also allows versioning the deployment.

For CD/CI, we used GOCD. GOCD natively support defining pipeline, stage, and job. It also gives an ability to define a template which is very handy in case of microservice architecture, as each microservice is having same deployment pipeline.

# API Gateway
We used Spring cloud gateway for implementing API Gateway. The main responsibility of the API gateway is as follows:
- Authentication
- Routing

### Authentication
Every API request is expected to have JWT token which user will get after he successfully logs in to the system. The API Gateway validates the JWT token with User service, If the token is valid the request will be routed to designated service. If the token is not valid, Gateway will return an error to the user.

### Routing
All of our services are using swagger for documentation. We used swagger API for setting up routing in API Gateway. The way it works is as follows:

- During API Gateway boot up, it first inquiry Kubernetes for list of services running in the cluster.
- Then API Gateway, inquiry each service for the list of REST APIs available using swagger API.
- Then API Gateway, is configured with filter, predicates and route (spring cloud gateway terminology)


