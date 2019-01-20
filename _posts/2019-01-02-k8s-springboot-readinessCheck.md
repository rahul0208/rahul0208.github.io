---
layout: post
title: Configure Pod readiness with Spring Actuator
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, spring]
---
Sometime back I worked on adapating a bank's application runbook using Ansible. Ansible was great it did need any external agents. So we started creating `playbook`s and deploying applications. But then the automation challange was *how do we determine an application is ready ?* so that we proceed deployment of next microservice. In our stack some application were age old spring 2.x applications while some were new age `SpringBoot` 2.0 based applications. SpingBoot provides `actuator endpoints` which lead us to the eventual solution. We added extensions to the `/health` endpoint, checking the availabily of our application dependencis. In older appliations we added `dropwizard` and created a similar ecosystem. In the end we added health check to each our service `Ansible` startup. Well that was Ansible but now we are adapting `Kubernetes`.

On kubernetes platform most of our containers are running a `java` application. These are springBoot applications with embeded servlet engines executed with the command `java -jar service.jar`. In the container world as well we have the challage of application readiness. So we can hack our way adapting the `Ansible` approach, by adding a `/health` endpoint check to our startup script. But this is *NOT* the recomended K8s best practice. Instead of this K8s provide container probes to determine a `container` status. There are two kinds of probes :
- **liveliness probe** : This is meant to determine is the container is running.
- **readiness probe** : This is meant to determine is the container is ready to accept connections.

The probe defination can be added to the `deployment.yaml` and thus we can leverage the readiness check. Lets look at the following `deployment.yaml`
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-gs
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: spring-gs
        version: v0.1.0
    spec:
      containers:
      - name: spring-gs
        image: k8s-series/spring-gs:0.1.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
---
```
In the above configuration we are deploying v0.1.0 of a sample `sprng-greetingService`. This is a SpringBoot based application to it exposed the following endpoints :
```
curl -o http://localhost:8080
{"_links":{"self":{"href":"http://localhost:8080/actuator","templated":false},"health":{"href":"http://localhost:8080/actuator/health","templated":false},"info":{"href":"http://localhost:8080/actuator/info","templated":false}}}
```
Now we added the same `/actuator/health` check to our application.
```
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```
K8s perfoms a period check determined by `periodSeconds`. We can `describe` our pod to know if the container has performed the check or not.
![Springboot-readiness](/img/springbook_k8s_readiness.png)

For our application `http` check was used but we can also perform check for `tcp` services as well. Probes can also execute custom commands to determine the state.
