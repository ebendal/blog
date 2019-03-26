---
title: Twelve-Factor App
---

Running a system on a cloud platform requires a component within the system to be designed for running on a cloud platform. When components are built this way they can utilize the platform to the full extent by using available resources, fast scaling, and full automation, but also deal with the complications of the cloud like maintaining state and resilience against failures. The "Twelve-Factor App"[1] principles are a set of rules that, when followed, will result in such a component. I will give a brief summary of every factor.

## 1. Codebase
##### One codebase tracked in revision control, many deploys

The source code of an application should live in a version control system (VCS). Every application in a system should have a separate VCS. In this context an application is an independently runnable component in a system. This breaks with conventional approaches where a big project is in one repository that consists of several components.

## 2. Dependencies
##### Explicitly declare and isolate dependencies

An application should have a dependency management system (DMS). All dependencies are listed in a configuration file including the version number of the dependency. The DMS is responsible for importing all the dependencies and creating an artifact that includes all the dependencies. This artifact can run on a runtime environment without pre-installed packages or libraries. This breaks with conventional approaches where an applicationserver has a lot of pre-installed libraries. The DMS is often incorporated in a build tool.

## 3. Config
##### Store config in the environment
 
An application has a different configuration in every environment it runs. This configuration of the application should not be inside the produced artifact by the build tool. The configuration should be obtained upon startup of the application from environment variables or an external service. In this way the some artifact can be used for an arbitrary amount of environments.
 
## 4. Backing services
##### Treat backing services as attached resources

An application often uses backing services, like a data store, an external cache, or a message broker. When deploying and starting an application the assumption is that these services are already running and ready for being used. In this way the application is decoupled from the actual manifestation of the service which could be a docker image on a local machine or a cluster in the cloud. Credentials for accessing the backing service is configuration therefore should be obtained from environment variables (factor 3).

## 5. Build, release, run
##### Strictly separate build and run stages

An application has three different stages. In the build stage the codebase (factor 1) and the dependencies (factor 2) are combined into an artifact by a build tool. This artifact may be deployed to different environments and for every environment a different release should be created so the configuration (factor 3) and backing services credentials (factor 4) are in the runtime environment. This release should be available at the cloud platform and can be started by a start command. Upon starting a release the run phase begins.

## 6. Processes
##### Execute the app as one or more stateless processes

An application consists of one or more processes. These processes can handle triggers from the outside by exposing service endpoints or can run scheduled tasks. In both cases a process can utilize computing power, memory, and disk space. It should use those resources for optimizations and short state transfers (i.e. save an incoming stream on disk to process a file after upload is complete). Apart from that the application should be stateless and any state should be stored in a stateful backing service.

## 7. Port binding
##### Export services via port binding

An application can expose a service by binding to a port and listening on it. Let the port on which the application listens be configured in the release phase (factor 5).

## 8. Concurrency
##### Scale out via the process model

An application can scale by using horizontal scaling. Because the application is stateless (factor 6) there is no need te share state between instances and scaling is just a matter of starting additional instances. The current instances finish the already running work. An additional instance helps with new load and work gets distributed equally over all instances. 

## 9. Disposability
##### Maximize robustness with fast startup and graceful shutdown

An application running in a cloud platform must be able to restart at any time. A failing instance, an upgrade in different layers of virtualization or software, or a new release are all reasons for a potential restart. To make this as seemless as possible an applications should try to finish all in progress work after a shutdown signal is received. The startup of an application should be as fast as feasible. No heavy tasks should be performed during startup but should be started after startup.

## 10. Dev/prod parity
##### Keep development, staging, and production as similar as possible

An application should be developed and tested against an environment that is as similar as possible to production. Therefore try to run real backing services such as a database on a development machine instead of using a fake in-memory one. Also test an application in the running stage on the cloud.

## 11. Logs
##### Treat logs as event streams

An application should not be spending resources on handling logging. Logs are written to standard out and the cloud platform will have a mechanism in place to capture those streams. A special log aggregation tool will take care of storing and retrieving log entries.

## 12. Admin processes
##### Run admin/management tasks as one-off processes

An application might require some management tasks like for example a database migration. Such a task should be incorporated in the codebase of the application and be inside the artifact produced by the build phase. The task should be configured in the release phase and be able to run in the running phase. This should not happen on startup because startup times could be increased significantly (factor 9). Instead it should be a one-off process that can be triggered by an exposed service.

## Conclusions

The Twelve-Factor App principals is a stepwise buildup towards an application that is designed to run on the cloud. It shows how an application can be build that is scalable, portable, and reliable.

## Sources

1. [Twelve-Factor App](https://12factor.net)