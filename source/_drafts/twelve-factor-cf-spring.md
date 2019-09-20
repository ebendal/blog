---
title: Twelve-Factor App with Spring and Cloud Foundry
---

## 1. Codebase
##### One codebase tracked in revision control, many deploys

Git is the VCS of choice. Create a local copy (deploy) of the repository.

```console
git clone https://github.com/ebendal/twelve-factor.git
git checkout one
```

## 2. Dependencies
##### Explicitly declare and isolate dependencies

Open [http://start.spring.io](http://start.spring.io) and choose `Gradle Project` and `Java`. Use `twelve-factor` as artifact name. Generate the project and unpack the contents in the previously cloned Git repo. 

Replace the contents of `build.gradle` with the following:

```groovy
plugins {
	id 'org.springframework.boot' version '2.1.8.RELEASE'
	id 'io.spring.dependency-management' version '1.0.8.RELEASE'
	id 'java'
}

group = 'com.example'
sourceCompatibility = '1.8'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-data-rest'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.flywaydb:flyway-core'
	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'mysql:mysql-connector-java'
	runtimeOnly 'com.h2database:h2'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

Gradle is the build tool of choice. To let Gradle fetch all the dependencies, compile the source code, and create an executable fat Jar, run:

```console
./gradlew assemble
```

The Jar file can be found at `build/libs/twelve-factor.jar`.

Before running the application give the application a name and disable some defaults. Rename the file `application.properties` to `application.yml` and add the following:

```yml
spring:
  application:
    name: twelve-factor
  jpa:
    open-in-view: false
  flyway:
    enabled: false
```

Run the application on your development machine:

```console
./gradlew bootRun
```

Check the proper functioning of the application by going to `http://localhost:8080/actuator/health` with your favourite HTTP request tool. For example Curl:

```console
curl http://localhost:8080/actuator/health
```
Response:
```json
{
  "status":"UP"
}
```

After this step the application looks like [this](https://github.com/ebendal/twelve-factor/tree/two).

## 3. Config
##### Store config in the environment

The environments in which this application should run is the developer machine and on an arbitrary `space` in Cloud Foundry (for example a `development`,`test`,`staging`, and `production` space). Spring profiles can be used to make the distinction between a local machine and a cloud environment, but should not be used for distinction between different Cloud Foundry spaces.


Edit the `application.yml` so it will look like this:
 
```yml
spring:
  application:
    name: twelve-factor
  jpa:
    open-in-view: false
  flyway:
    enabled: false
---
spring:
  profiles: local
info:
  application:
    environment: local
    platform: ${os.name}
---
spring:
  profiles: cloud
info:
  application:
    environment: ${vcap.application.space_name}
    platform: ${platform.name}
```

The profile specific properties will only be set when that profile is active. In Cloud Foundry the `cloud` profile is always active. In the `application.yml` we configure the Spring actuator info endpoint. The `${}` values instruct Spring to find the values, among others, in environment variables. 

To run the application on your local machine you can now use:

```console
./gradlew bootRun --args='--spring.profiles.active=local'
```

Check the response of the actuator info endpoint:
```console
$ curl localhost:8080/actuator/info
```
```json
{
  "application": {
    "environment": "local",
    "platform": "Mac OS X"
  }
}
```

In Cloud Foundry there are two ways to get information from the environment.

- Standard environment variables found in the `VCAP` environment variable. The `vcap.application.space_name` is an example.
- Custom environment variables created when deploying the application. The `platform.name` is an example.

To deploy an application to Cloud Foundry you need to give it at least a name. One of the deployment features of Cloud Foundry is to use a manifest file. Create the following `manifest.yml` in the root of the repository:
```yaml
applications:
  - name: twelve-factor
    path: build/libs/twelve-factor.jar
    env:
      PLATFORM_NAME: Cloud Foundry
```

In this file the custom environment variable `PLATFORM_NAME` is also defined. 

Now the application can be built and deployed:
```console
./gradlew build
cf push --random-route
```
The two environment variables the application uses can be found when you display the environment of the app:
```console
cf env twelve-factor
```
Compare the configuration in the `application.yml` with the displayed environment variables.

Check the response of the actuator info endpoint. The response should be filled with the values of the environment variables.
```console
curl <random-generated-uri>/actuator/info
```
```json
{
  "application": {
    "environment": "<space-name>",
    "platform": "Cloud Foundry"
  }
}
```
After this step the application looks like [this](https://github.com/ebendal/twelve-factor/tree/three).


## 4. Backing services
##### Treat backing services as attached resources


In Cloud Foundry (backing) services are available in the `marketplace`.
```console
cf marketplace
```
A list of available services is visible and every service has a name and associated plans. To create a service you choose the name of the service, the plan, and a name for the service instance you are creating. In this example we will create a free MySQL database.
```console 
cf create-service cleardb spark twelve-factor-db 
```
We now have a backing service. In Cloud Foundry you can bind the service to an app to make it an attached resource of that app.  
```console 
cf bind-service twelve-factor twelve-factor-db
```
By binding a service to an app, a set of credentials is created for the app to access the service. These credentials are available to the app and can be accessed by the app in the same manner as factor 3.
```console
cf env twelve-factor
```
App specific configuration of the platform is something you want to have in version control as well. The current space is now ready, but factor 1 stated there could be many deploys. Therefore we want to add the service to the `manifest.yml`.
```yaml
applications:
  - name: twelve-factor
    path: build/libs/twelve-factor.jar
    env:
      PLATFORM_NAME: Cloud Foundry
    services:
      - twelve-factor-db
```

Now we can configure Spring to find the credentials of the backing service in the environment variables when the `cloud` profile is active. For now we will also let Hibernate generate the schema for us. The `application.yml` now looks like this.

```yaml
spring:
  profiles: cloud
  datasource:
    url: ${vcap.services.twelve-factor-db.credentials.jdbc-url}
  jpa:
    hibernate:
      ddl-auto: update
info:
  application:
    environment: ${vcap.application.space_name}
    platform: ${platform.name}
```
This is the most simple example of a REST service that performs CRUD operations. For more information see the Spring documentation. Make sure the class is on the classpath of the application and Spring will wire it.
```java
import lombok.Data;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

import static javax.persistence.GenerationType.IDENTITY;

@RepositoryRestResource
public interface FactorRepository extends CrudRepository<FactorRepository.Factor, Long> {

    @Data
    @Entity(name = "factor")
    class Factor {

        @Id
        @GeneratedValue(strategy = IDENTITY)
        private Long id;
        private Integer number;
        private String name;
    }
}
```
Check whether the application is working in your local machine.
```shell script
./gradlew bootRun --args='--spring.profiles.active=local'
curl -d '{"number":"1", "name":"Codebase"}' -H "Content-Type: application/json" -X POST http://localhost:8080/factors
curl http://localhost:8080/factors
```
Now we can build and deploy the app to Cloud Foundry.

```shell script
./gradlew build
cf push
```
Check whether the application is working in Cloud Foundry.
```shell script
curl -d '{"number":"1", "name":"Codebase"}' -H "Content-Type: application/json" -X POST https://<app-uri>/factors
cf restart twelve-factor
curl http://<app-uri>/factors
```
## 5. Build, release, run
##### Strictly separate build and run stages
We have seen the combination of the commands `./gradlew build` and `cf push` a lot. 

The `./gradlew build` command is building an artifact from the source files and is referred to as the build step.

The `cf push` command does a lot as you can see in the output of the console. First it uploads the artifact to the platform. Cloud Foundry separates what happens next in 2 stages:
1. staging
2. running

The staging stage is called release stage by Twelve-Factor. The naming might be different the stage is the same. The artifact is combined with an OS and JRE into an image. This image is called a droplet. A new container is started when the staging stage begins. After the droplet is created and saved, the container gets destroyed.

The running stage is where the droplet is used to create a new container and the start command of the application is executed. The app is monitored during startup and upon the first positive health response the app gets the status running.
## 6. Processes 
##### Execute the app as one or more stateless processes
An app in Cloud Foundry should be stateless. Cloud Foundry can restart an app at any moment. It could be triggered by a new CF push, by an upgrade of the buildpack, by any unknown failure. With every request that has any state change involved, the change should be persisted to a service. Other requests can pick up this state change once the transaction is committed. Caching can be done by a service that has the appropriate underlying hardware. Such product are available in the marketplace.
## 7. Port binding
##### Export services via port binding
The best evidence that Cloud Foundry uses port binding is that in the start command of the app the port is configured by an environmental variable of the container it is running in. Also the router that handles the requests routes the traffic to a container by forwarding it to the right port of the container.
## 8. Concurrency
##### Scale out via the process model
In Cloud Foundry you can scale your app vertically (increasing memory) and horizontally (increasing instances). 

The memory available to an app is by default `1G`. The required memory for an application needs to be found out by a mild load performance test. For our app `768M` is sufficient. Most Spring Boot apps require more and `1G` is a good starting point. To scale the app down we need to define the memory that should be available. The right place is in the `manifest.yml`. Every new push of the application will automatically know the right amount of memory.

```yaml
applications:
  - name: twelve-factor
    path: build/libs/twelve-factor.jar
    memory: 768M
    env:
      PLATFORM_NAME: Cloud Foundry
    services:
      - twelve-factor-db
```

Now the app is good for a mild load. When the load starts increasing while the app is running, you want to scale horizontally.

```shell script
cf scale twelve-factor -i 2
cf app twelve-factor
```

## 9. Disposability
##### Maximize robustness with fast startup and graceful shutdown

Spring Boot starts typically in a bit more than 10 seconds. By default Cloud Foundry requires an app to start within 60 seconds otherwise it considers the startup failed and will try again. You can extend this timeout period up to 120 seconds.

Shutdown of an app happens by Cloud Foundry sending the `SIGINT` signal to the running process. Spring Boot will pickup this signal and tries to let the running requests finish but does not accept new ones.

## 10. Dev/prod parity
##### Keep development, staging, and production as similar as possible

A space in Cloud Foundry is just a logical separation. Apps in these spaces can run on any underlying VM. All spaces run on the same Cloud Foundry instance. A space is typically named after the environment it represents (i.e. test, staging, production).

That leaves the parity on a local machine. Up till now the app uses an in memory database when it is not able to find a real MySQL database. This violates the parity principle. Queries can perform different and might not even be supported by the H2 database. To make the local environment as similar as possible to production we will use docker-compose to start a container with a MySQL database.

Create a file called `docker-compose.yml` in the root of the repository with the following content:

```yaml
version: '3'
services:
  mysql:
    image: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=root-password
      - MYSQL_DATABASE=database
      - MYSQL_USER=user
      - MYSQL_PASSWORD=password
```

Now configure the datasource of the app when the `local` profile is active in the `application.yml`:
```yaml
spring:
  profiles: local
  datasource:
    url: jdbc:mysql://localhost:3306/database?user=user&password=password
info:
  application:
    environment: local
    platform: ${os.name}
```

Start the container and the application:

```shell script
docker-compose up mysql
./gradlew bootRun --args='--spring.profiles.active=local'
```

## 11. Logs
##### Treat logs as event streams

In Cloud Foundry your app should just log to the standard system out and system error. An agent will pick up those logs and send it to the log aggregator that aggregates all the logs from all the apps. As a developer you cannot do anything with those logs. The administrators of the Cloud Foundry instance will be able to tap in on that stream. Also the API that is used by the the CLI gets logs from that stream.

You can choose to let the app logging be forwarded to your own log aggregator as well. To do this first create an account on a cloud service that aggregates logging (for example Papertrail). Get a syslog uri and use it in the following commands:

```shell script
cf create-user-provided-service logging -l <your-syslog-uri>
```

Now add this service to you `manifest.yml`:

```yaml
applications:
  - name: twelve-factor
    path: build/libs/twelve-factor.jar
    memory: 768M
    env:
      PLATFORM_NAME: Cloud Foundry
    services:
      - twelve-factor-db
      - logging
```
## 12. Admin processes
##### Run admin/management tasks as one-off processes

Up till now we used Hibernate to migrate our database. We will use a different library called Flyway to take of that for us. By default it will migrate the app on startup, which for now we will leave like this. It would be even better to let the migration be triggered by an admin trigger.

Enable flyway in the `apllication.yml` and configure Hibernate to only validate the schema:
```yaml
spring:
  application:
    name: twelve-factor
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
  flyway:
    baseline-on-migrate: true
```

By default Spring will look in the `db/migration` folder for any migrations. Our current schema will be the first migration. Create a the file `src/resources/db/migration/V1__Initial_Schema.sql` with the following content:
```sql
CREATE TABLE factor
(
	id BIGINT AUTO_INCREMENT PRIMARY KEY,
	name VARCHAR(255),
	number INT
)
```

We want to add an additional column to our factor table. Create the file `src/main/resources/db/migration/V2__Add_Statement.sql` with the following content:
```sql
ALTER TABLE factor ADD statement VARCHAR(255)
```

Start the app:

```sql
./gradlew bootRun --args='--spring.profiles.active=local'
```

The database now has an additional column. Our app did not change the domain models yet. Let's do that:

```java
import lombok.Data;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

import static javax.persistence.GenerationType.IDENTITY;

@RepositoryRestResource
public interface FactorRepository extends CrudRepository<FactorRepository.Factor, Long> {

    @Data
    @Entity(name = "factor")
    class Factor {

        @Id
        @GeneratedValue(strategy = IDENTITY)
        private Long id;
        private Integer number;
        private String name;
        private String statement;
    }
}
```
###### Sources  
1. [The Twelve-Factor App](https://12factor.net/)
2. [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)