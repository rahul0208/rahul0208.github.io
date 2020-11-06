---
layout: post
title: Service Containers in Azure Pipelines
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [Azure Devops, service containers]
---
Testing a system in isolation is not easy. A system usually has many dependencies which must be plugged for its proper function.  During development Developers have the flexibility of deploying mock services. These services allow every developer to run the application on their own boxes. These mock services are required during component / system testing as well. The challenge is how do we provision the infrastructure for tests ?

The above stated challenge is a classic issue. Previously when I had faced this we would extend the build phase / steps. In one of the project where Gradle was used, we achieved this with `start-server` and `stop-server` servers. The approach solved our task, but there were issues in the integration. We often had orphaned services, which asked for cleanup.

In another project we asked the developers to have these services running on their boxes. On Jenkins we created a build pipeline where we deployed the services as a pre-step. The services were shutdown as a post-step.

In my recent project we had a similar requirement, but instead of Jenkins we had Azure Devops as our build system. We could have added a pre `step` and a post `step` by following the traditional approach.

Alternatively, Azure Devops provides the concept of `services`. These `services` will be started pre build(`Initialize Containers`) and stoped post the project build steps (`Stop Containers`). Azure Devops takes care of the startup and shutdown.  

![Azure-services](/img/azure-service/steps.png)

We can start as many `services` as required by our build process. The `services` can be defined using [pipeline resources](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema#resources). These `resources` can be containers or other pipelines, defined in the same file.  We can configure the different options for [containers](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema)


```
trigger:
- master

resources:
  containers:
    - container: azurite
      image: mcr.microsoft.com/azure-storage/azurite
      ports:
      - 10000:10000

jobs:
- job: BuildJob
  timeoutInMinutes: 0
  pool:
    vmImage : 'ubuntu-latest'

  services:
    azurite: azurite

  steps:
    - task: Cache@2
      inputs:
      ## Removed for Brevity  

    - task: Maven@3
      inputs:
      ## Removed for Brevity        

## Rest Removed for Brevity  
```

I the configuration above I added a `Azurite` container. The service is a mock for Azure blob service. The service is the started before project build. The container is started on `localhost` address. If we are using multiple containers then we can refer them using service name.
