---
title: Twelve-Factor App with Spring and Cloud Foundry
---

## 1. Codebase
##### One codebase tracked in revision control, many deploys

The source code of an application should live in a version control system (VCS). In this context an application is an independently runnable component in a system. Every application in a system should have a separate VCS.

###### Implementation

Git is the VCS of choice. Create a local copy of the repository:

```console
git clone https://github.com/ebendal/twelve-factor.git
git checkout one
```

## 2. Dependencies
##### Explicitly declare and isolate dependencies

An application should have a dependency management system (DMS). All dependencies are listed in a configuration file including the version number of the dependency. The DMS is responsible for importing all the dependencies and creating an artifact that includes all the dependencies. The DMS is often incorporated in a build tool. 

###### Implementation

Open [http://start.spring.io](http://start.spring.io) and choose `Gradle Project` and `Java`. Use `twelve-factor` as artifact name. Generate the project and unpack the contents in the previously cloned Git repo. 

Replace the contents of `build.gradle` with the following:

```groovy
buildscript {
	ext {
		springBootVersion = '2.1.1.RELEASE'
	}
	repositories {
		mavenCentral()
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'com.example'
sourceCompatibility = '1.8'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'mysql:mysql-connector-java'
	runtimeOnly 'com.h2database:h2'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

Gradle is the build tool of choice. To let Gradle fetch all the dependencies, compile the source code, and create an executable fat Jar, run:

```console
./gradlew assemble
```

The Jar file can be found at `build/libs/twelve-factor.jar`.

Before running the application give the application a name and get rid of a Spring Boot warning. Rename the file `application.properties` to `application.yml` and add the following:

```yml
spring:
  application:
    name: twelve-factor
  jpa:
    open-in-view: false
```

Run the application on your development machine:

```console
./gradlew bootRun
```

Check the proper functioning of the application by going to `http://localhost:8080/actuator/health` with your favourite HTTP request tool. For example Curl:

```console
curl http://localhost:8080
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



###### Implementation
wewfwgwg
```yaml
---
spring:
  profiles: local
info:
  application:
    name: ${spring.application.name}
    platform: ${os.name}
---
spring:
  profiles: cloud
info:
  application:
    name: ${vcap.application.name}
    platform: ${platform.name}
```
jnjn
```console
./gradlew bootRun --args='--spring.profiles.active=local'
```

```console
$ curl localhost:8080/actuator/info
```

```json
{
  "application": {
    "name": "twelve-factor",
    "platform": "Mac OS X"
  }
}
```

```yaml
applications:
  - name: twelve-factor
    path: build/libs/twelve-factor.jar
    env:
      PLATFORM_NAME: Cloud Foundry
```


```console
cf push --random-route
```

```console
cf env twelve-factor
```

```console
curl <random-generated-uri>/actuator/info
```

```json
{
  "application": {
    "name": "twelve-factor",
    "platform": "Cloud Foundry"
  }
}
```


After this step the application looks like [this](https://github.com/ebendal/twelve-factor/tree/three).


## 4. Backing services
##### Treat backing services as attached resources

```java
import lombok.Data;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@RepositoryRestResource
public interface FactorRepository extends CrudRepository<FactorRepository.Factor, Long> {

    @Data
    @Entity
    class Factor {

        @Id
        @GeneratedValue
        private Long id;
        private Integer number;
        private String name;
        private String description;
    }
}
```


```console 
cf create-service cleardb spark twelve-factor-db 
```

```console 
cf bind-service twelve-factor twelve-factor-db
```

```console
cf env twelve-factor
```


## 5. Build, release, run
##### Strictly separate build and run stages
## 6. Processes
##### Execute the app as one or more stateless processes
## 7. Port binding
##### Export services via port binding
## 8. Concurrency
##### Scale out via the process model
## 9. Disposability
##### Maximize robustness with fast startup and graceful shutdown
## 10. Dev/prod parity
##### Keep development, staging, and production as similar as possible
## 11. Logs
##### Treat logs as event streams
## 12. Admin processes
##### Run admin/management tasks as one-off processes

###### Sources  
1. [The Twelve-Factor App](https://12factor.net/)
2. [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)