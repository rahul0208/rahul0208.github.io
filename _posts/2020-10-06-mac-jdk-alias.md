---
layout: post
title: JDK Alias in Mac
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [mac, bash]
---

Mac OS has a a useful utility `java_home`. The utility helps us to determine which Java versions are available and  setup the `JAVA_HOME` environment variable. It can be used to enable a versions of Java from the available versions. We can execute the following command, to list the versions :

```
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (6):
    14.0.2, x86_64:	"OpenJDK 14.0.2"	/Library/Java/JavaVirtualMachines/jdk-14.0.2.jdk/Contents/Home
    11.0.6, x86_64:	"AdoptOpenJDK 11"	/Library/Java/JavaVirtualMachines/adoptopenjdk-11.jdk/Contents/Home
    11.0.5, x86_64:	"Java SE 11.0.5"	/Library/Java/JavaVirtualMachines/jdk-11.0.5.jdk/Contents/Home
    9, x86_64:	"AdoptOpenJDK 9"	/Library/Java/JavaVirtualMachines/adoptopenjdk-9.jdk/Contents/Home
    1.8.0_251, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
    1.8.0_242, x86_64:	"AdoptOpenJDK 8"	/Library/Java/JavaVirtualMachines/adoptopenjdk-8.jdk/Contents/Home
```

In case you are working across different versions. I had to work on JDK 8 for one of out support project, while JDK 14 is used for another greenfield project. In order to do this add the following lines to the `~/.bash_profile` :

```
alias j14="export JAVA_HOME=`/usr/libexec/java_home -v 14`; java -version"
alias j11="export JAVA_HOME=`/usr/libexec/java_home -v 11`; java -version"
alias j8="export JAVA_HOME=`/usr/libexec/java_home -v 1.8`; java -version"
```

 We can switch different versions in the following manner :
```
$ j14
openjdk version "14.0.2" 2020-07-14
OpenJDK Runtime Environment (build 14.0.2+12-46)
OpenJDK 64-Bit Server VM (build 14.0.2+12-46, mixed mode, sharing)

$ j8
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```
