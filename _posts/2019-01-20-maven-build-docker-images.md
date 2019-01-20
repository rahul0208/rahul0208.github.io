---
layout: post
title:  Creating Docker images for Java projects
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, maven, docker]
---
I have been building a new Greenfield java application for quite sometime. This was a standard `Spring boot` application which packs an embeded servlet container. Most of my demos ran the application vai the `SpringBoot` plugin. This was good for development environment. But my workplace has declared `container` as the *go-forward* deployment platform and thus my application shall NOT be released into `staging` until it is containerized. Since this was a `SpringBoot` fat jar application so all that was required is to build a `Docker` image having `java -jar application.jar` command. Im my evaluation I found following ways of generating `Docker` images.

For the prupose of this blog let's assume we have a `Spring Boot` [Greeting-service](https://spring.io/guides/gs/rest-service/). The service can be built using `maven` or `gradle`. It makes no difference to the generated docker image. The docker image can be built in amy of the follwoing manner :

### Old School Scripting
`Docker` provides the essential commands to build docker images using a `Dockerfile`. So in my first attempt I added the following `Dockerfile`

```
FROM openjdk:8-alpine

ADD ./build/libs/ /opt/gs-service/

ARG service_version
ENV SERVICE_VERSION ${service_version}
CMD java -jar /opt/gs-service/gs-spring-boot-${SERVICE_VERSION}.jar
```
>> I picked java8 alpine base image as it has optimal binary size.Picking `ubuntu` or `centos`  increases the overall image size.

In this file I as coping files from `build` folder to `/opt` location.  I had to invoke `docker build` command to generate the image. Now every time I have to type two commands
- one for code compile (`gradlew clean build`)
- second for  image build (`docker build`)   

This was an overhead so I added both comannds to a simple `build.sh` file.
```
#!/bin/bash
set -o errexit

VERSION=0.1.0
BASE_DIR=$(pwd)

docker run --rm -v "$BASE_DIR":/home/spring-gs -w /home/spring-gs gradle:4.8.1 gradle clean build
docker build -t "spring-gs:${VERSION}" -t spring-gs:latest --build-arg service_version=${VERSION} "$BASE_DIR"

```

In the script I am compiling project in a docker container using gradle. The generated artifacts are then copied to a new docker image built using `Dockerfile` The above project generated docker images under the name `spring-gs`.

![Script-docker-image](/img/script-docker-image.png)

### Spotify Dockerfile Maven plugin

Well the above solution was *sub-optimal*. I did not like the idea of a separate build script. But the `Dockerfile` was definately the way we wanted to build a container. Thus, I configured the [`dockerfile-maven-plugin`](https://github.com/spotify/dockerfile-maven). The plugin has simple execution idea. It treats `Dockerfile` as the imput standard and build docker images from it. I had the `Dockerfile` in my project and I added the follwoing plugin configuration :

```
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>1.4.10</version>
  <executions>
    <execution>
      <id>default</id>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <tag>${project.version}</tag>
    <repository>spotify/spring-gs</repository>
    <buildArgs>
      <service_version>${project.version}</service_version>
    </buildArgs>
  </configuration>
</plugin>
```
The plugin also enables to build docker images in tar format which can be shared in offline manner. The plugin allowed me to pass a `repository name` and  a `tagname`. Apart from the limitation of a single tagename, I could pass all types of parameters to the underlying `docker build` command. Invoke `mvn clean dockerfile:build` to generate a docker image.

![spotify-docker-image](/img/spotify-docker-image.png)

### Fabric8 Docker Maven plugin

I also had a look at the `docker-maven-plugin`](https://dmp.fabric8.io/#introduction) from Fabric8. This is a complete docker plugin, it allows us to start / stop containers during various maven lifecycle phases. The plugin enables us to build multiple pages from a project. This is a complete docker toolkit for maven. I added the plugin to my maven pom

```
<plugin>
  <groupId>io.fabric8</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.28.0</version>
  <configuration>
    <images>
      <image>
        <name>dmp/spring-gs</name>
        <build>
          <args>
            <service_version>${project.version}</service_version>
          </args>
          <tags>
            <tag>latest</tag>
            <tag>${project.version}</tag>
          </tags>
          <dockerFile>${project.basedir}/Dockerfile</dockerFile>
          <filter>@</filter>
        </build>
      </image>
    </images>
  </configuration>
</plugin>
```
I added the minimal required configuration. It assumes the dockerfiles are present under `src/main/docker` path and filters the files to inject `project` variables form maven. I had no intentions of changing my `Dockerfile`, so configured the plugin accordingly. In the end I invoked `mvn clean docker:build` command to generate docker images.

![dmp-docker-image](/img/dmp-docker-image.png)

### Google Jib Maven plugin
Lastly, I worked with the google jib plugin. Jib is aimed at building cantainer images for your application. The overall result it produces is the same but the image it generates is very diffrent from the above approaches. the plugin DOES NOT build images by using `Dockerfile`.

The images it generates are **distroless** container images. This essatilly means that they pack just the Java runtime and **NOT** the OS. The plugin also enables us to build images for alternate container runtimes like oci.

>> (Introduction to Disroless docker)[https://www.youtube.com/watch?v=qhykcC94ukg]

```
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>0.9.0</version>
  <configuration>
    <to>
      <image>gcr.io/spring-gs</image>
    </to>
    <tags>
      <tag>${project.version}</tag>
      <tag>latest</tag>
    </tags>
  </configuration>
</plugin>
```
I have added minimal configuration. Since we are generating `SpringBoot` fat jar so the plugin generates image with `java -jar PROJECT.jar` command. The plugin provides configurable options to provide `MainClass`, `JvmArgs`, `additionalPorts` etc. It also enables us to upload our images to various container regiestries like dockerhup, aws container rgistry etc. But I am just uploading the images to hy dockerDemon so I involve `mvn clean  compile jib:dockerBuild`

![jib-docker-image](/img/jib-docker-image.png)

### Remarks
After working with each of the above ways, I was of the opinion to have a `Dockerfile`. In my oberstaion having `Dockerfile` enables me to accept *the container as my first-class deployment option and not some operational procedure*. To me the `dmp-maven-plugin` also enables me to leverage containers for my other development needs like integration testing etc. On the other hand `jib-maven-plugin` generated distroless images are the next leap in images. I would like to adapt them.  I would try to explore the `jib` tooling to see if there is way to build the configuration as a seperate file and not embed it in the plugin configuration.

All the plugins enabled me to export the image as `tar` and import it at other environment. To me this is quite handy in development and evaluation phase.
