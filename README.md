Instructions to build a single node cluster on Minikube with a private repository running on http. It runs a spring boot application container. This container is built from an image pulled from the private repository.

* [Pre-requisites](#pre-requisites)
* [Setup](#setup)
* [Getting Started](#getting-started)
* [Deploy to Kubernetes](#deploy-to-kubernetes)

## Pre-requisites

- `jdk11`
- `docker`
- `minikube`
- `kubectl`

## Setup

### Start Docker Daemon

* Get *IP* address or url of where your registry runs. IP address can be obtained from cmd `ip addr`

* Replace *IP* in below command with the actual ip or url.

```
$ echo '{"insecure-registries": ["IP:5000"]}' > /etc/docker/daemon.json
```
* Start docker daemon
```
$ sudo systemctl start docker
```
* Verify insecure registry

```
$ docker info -f '{{json .RegistryConfig.IndexConfigs}}'
```

This should list as:
{"*IP*:5000":{"Name":"*IP*:5000","Mirrors":[],"Secure":false,"Official":false},"docker.io":{"Name":"docker.io","Mirrors":[],"Secure":true,"Official":true}}

### Start Minikube



## Getting Started

Create a basic Spring Boot application:

```
$ curl https://start.spring.io/starter.tgz -d dependencies=webflux -d dependencies=actuator | tar -xzvf -
```

Add an endpoint (`src/main/java/com/example/demo/Home.java`):

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class Home {
    @GetMapping("/")
    public String home() {
        return "Hello World";
    }
}
```



Containerize (`Dockerfile`):

```
FROM openjdk:8-jdk-alpine as build
WORKDIR /workspace/app

COPY target/*.jar app.jar

RUN mkdir target && cd target && jar -xf ../*.jar

FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG DEPENDENCY=/workspace/app/target
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","com.example.demo.DemoApplication"]
```

Run and test...

```
$ ./mvnw package
$ docker build -t localhost:5000/apps/demo .
$ docker run -p 8080:8080 localhost:5000/apps/demo
$ curl localhost:8080
Hello World
```

Stash the image for later in our local repository (which was started with `kind-setup` if you used that):

```
$ docker push localhost:5000/apps/demo
```

## Deploy to Kubernetes

Create a basic manifest:

```
$ kubectl create deployment demo --image=localhost:5000/apps/demo --dry-run -o=yaml > deployment.yaml
$ echo --- >> deployment.yaml
$ kubectl create service clusterip demo --tcp=80:8080 --dry-run -o=yaml >> deployment.yaml
```

Apply it:

```
$ kubectl apply -f deployment.yaml
$ kubectl port-forward svc/demo 8080:80
$ curl localhost:8080
Hello World
```

