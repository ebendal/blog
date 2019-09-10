---
title: Service Discovery in a Microservices Architecture
---

Service Discovery is not a new problem. When the client-server pattern is used in an architecture there is often a well known URL that resolves via DNS to a load balancer and the load balancer knows how to route traffic towards the application server. When multiple instances exist of the application server the load balancer has the responsibility to balance the load using a certain strategy. This can referred to as server-side service discovery and server-side load balancing because the load-balancer acts as the server.

##### The Discovery Problem of Microservices

In a microservices architecture an application is deployed on a certain location and exposes a service on a port. I will refer to this location and port as the address from now on. One of the benefits of a microservice architecture is that services can scale independently by horizontal scaling. Now assume that a consumer wants to consume a service of a producer. This producer has potentially multiple instances. How does a consumer know the address of the producer? 

The server-side service discovery solution is not a perfect fit for this problem. Resolving a DNS record (or maybe the consumer knows it already) and having another network hop towards a load balancer adds unnecessary latency. Choosing between a single load balancer in the system or a load balancer per application is also choosing between two evils. Having a single load balancer means a single point of failure for the entire system. A load balancer per application adds unnecessary complexity. 

##### Discovery Server pattern

The discovery server pattern is a good solution for the discovery problem of microservices. A discovery server is an application that keeps a registry of all running instances of producers. A producer tells the discovery server it is alive upon startup and keeps sending heartbeats to tell the discovery server it is still up and running. A consumer has a local copy of the service registry and updates it regularly by requesting the latest state from the discovery server. Now the consumer knows all addresses of the producer. The client takes care of the service discovery and therefore you can refer to this as client-side service discovery.

Using the discovery server pattern does not require an extra network hop to a load balancer. Temporary downtime of the discovery server does not bring the whole system down because consumers can still use their local copy of the registry. Load balancing can now be taken care of by the consumer. This pattern uses client-side load balancing.

##### Conslusion

With choosing a microservices architecture for a system the traditional service discovery problem changes. With the Discovery Server pattern service discovery shifts from server-side to client-side and with it also load balancing shifts from server-side to client-side. 


### 
<img src="discovery-problem.png" alt="Discovery Problem [1]" width="450"/>
<img src="self-registration.png" alt="Registering Instance [1]" width="350"/>
<img src="client-side-discovery.png" alt="Service Discovery [1]" width="400"/>

###### Sources  
1. [Service Discovery in a Microservices Architecture](https://www.nginx.com/blog/service-discovery-in-a-microservices-architecture/)
2. [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html)
