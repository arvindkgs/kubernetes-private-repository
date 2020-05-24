Instructions to build a single node cluster on Minikube with a private repository running on http. It runs a spring boot application container. This container is built from an image pulled from the private repository.

* [Pre-requisites](#pre-requisites)
* [Setup](#setup)
* [Getting Started](#getting-started)
* [Deploy to Kubernetes](#deploy-to-kubernetes)
* [Test](#test)
* [Consume Service](#consume-service)
* [Alternatives](#alternatives)
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

* To start minikube configured to use the private registry, run below command. NOTE: *Replace DRIVER with correct driver which is one of: [virtualbox, vmwarefusion, kvm2, kvm, hyperkit]*
```
$ minikube start --vm-driver=DRIVER --insecure-registry="IP:5000"
```

### Start private registry

* If running on local start registry as:
```
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

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
$ docker build -t IP:5000/apps/demo .
$ docker run -p 8080:8080 IP:5000/apps/demo
$ curl localhost:8080
Hello World
```

Stash the image to our private repository:

```
$ docker push IP:5000/apps/demo
```

## Deploy to Kubernetes

Create a basic manifest:

```
$ kubectl create deployment demo --image=IP:5000/apps/demo --dry-run -o=yaml > deployment.yaml
$ echo --- >> deployment.yaml
$ kubectl create service nodeport demo --tcp=8080:8080 --dry-run -o=yaml >> deployment.yaml
```

Apply it:

```
$ kubectl apply -f deployment.yaml
```

## Test

Check if deployment is up
```
$ kubectl get deployments.app demo
```
Next check if service is running
```
$ kubectl get service demo
```
NOTE: *Make a note of the port. For example if port is mentioned as 8080:32614, then copy 32614. This is the port on which the service is available*

## Consume service

Get IP
```
$ kubectl get endpoints kubernetes
```
Make a note of the IP, ignore the port number. Example - if endpoint: '192.168.39.138:8443', then IP: 192.168.39.138

Get Port
```
$ kubectl get service demo
```
Port is the second part of ':' delimiter. Example - if PORT is '8080:32614/TCP', then our port is 32614

Call springboot app, substituting IP below with ip obtained above and PORT with port obtained above
```
$ curl IP:PORT
Hello World
```

## Alternatives

As a corollary, if you are running a test or doing development and don't want to push your image to a private repository or dockerhub, then you can reuse Minikube's built-in docker daemon. So you can build images inside the same docker daemon as Minikube, which speeds up local experiments.

For this, run last line from
```
$ minikube docker-env
```
You can now use Docker at the command line of your host Mac/Linux machine to communicate with the Docker daemon inside the Minikube VM:

```
$ docker ps
```

More info [here](https://kubernetes.io/docs/setup/learning-environment/minikube/#use-local-images-by-re-using-the-docker-daemon)
