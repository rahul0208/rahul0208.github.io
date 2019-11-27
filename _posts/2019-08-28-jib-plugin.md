---
layout: post
title: Build container image using Maven-jib-plugin
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [maven, jib]
---

Current I am building containers for my project using `DockerFile`. In order to build such a project I would run with `dockerBuild` step in my CI pipeline. Since `Dockerfile` is another piece of code so it needs to be given its share of attention as well. We add steps to execute our app, but there is no one stopping adding further steps. Thus  there are  best practices which must be obeyed while creating such a file.  But do we need such a file ? Most of the time we are excuting the projetc as exBut often in my `spring-boot` projects, I end up creating a jar file which needs to be exectuted as a main program. In this case the `Dockerfile` is quite simple with a `copy` and `extrypoint` step.  So lately I am having an inclination of avoiding the docker file and instead build the `image ` with [google-jib-plugin](https://github.com/GoogleContainerTools/jib). Besides building the image in the build lifecycle there is additional advantage of my CI independent of docker demon.

The following is a example configuration which I used in my project :

```xml
<plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>1.8.0</version>
                <configuration>
                    <from>
                        <image>tomcat:8.5-jre8-alpine</image>
                    </from>
                    <outputPaths>
                        <tar>${project.build.directory}/${artifactId}-${version}.tar</tar>
                    </outputPaths>
                    <container>
                        <creationTime>${maven.build.timestamp}</creationTime>
                    </container>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>buildTar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

There are a few important pieces here :

- I am working with `buildTar` goal of the plugin. The plugin has a number of goals which can allow iyt to push images to docker registry / google registry etc. 
- The generated `tar` file is in `OCI`format so can be imported in Docker / containerd or anyother runtime.
- I had customed the `tar` name using the `outputPaths`
- Lastly the plugin stamps EPOCH time for conatiner creation. This would be shown as `image` build time( which is wrong). So I provided the `creationTime` value to set the correct time.

The above configuration starts the image building as part of `package` lifecycle. 

```
[INFO] --- jib-maven-plugin:1.8.0:buildTar (default) @ guestbook-spring ---
[INFO] Tagging image with generated image reference guestbook-spring:1.0-SNAPSHOT. If you'd like to specify a different tag, you can set the <to><image> parameter in your pom.xml, or use the -Dimage=<MY IMAGE> commandline flag.
[INFO] 
[INFO] Containerizing application to file at '/Users/rahul/Projects/guestbook/target/guestbook-spring-1.0-SNAPSHOT.tar'...
[WARNING] Base image 'tomcat:8.5-jre8-alpine' does not use a specific image digest - build may not be reproducible
[INFO] The base image requires auth. Trying again for tomcat:8.5-jre8-alpine...
[INFO] Using base image with digest: sha256:3998c88afb2c2751c18db3fb0b45cb4f8bb1d575591a96dcd9f071ce12651242
[INFO] Container program arguments set to [catalina.sh, run] (inherited from base image)
[INFO] 
[INFO] Built image tarball at /Users/rahul/Projects/guestbook/target/guestbook-spring-1.0-SNAPSHOT.tar
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO] 
[INFO] 
[INFO] --- maven-install-plugin:2.4:install (default-install) @ guestbook-spring ---
```

There is one more thing to note Google has a best practice of using distroless images. Thus if the base image is unspecifed the plugin selects `gcr.io/distroless/java:8`, for `jar` packaging  and `gcr.io/distroless/java/jetty:java8` for `war` packaging.  The above created archive cab be imported in docker / containered using `docker load --input guestbook-spring-1.0-SNAPSHOT.tar`

`$ docker images`

```
REPOSITORY     TAG         IMAGE ID      CREATED       SIZE
guestbook-spring  1.0-SNAPSHOT    5dcfcd266211    44 seconds ago   132MB
ubuntu       18.04        775349758637    3 weeks ago     64.2MB 
```

 

