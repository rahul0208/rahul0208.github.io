---
layout: post
title: Using Azurite Emulator
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [Azure, blob, queue, Emulator, Azurite]
---
Cloud Storage is great for operation purpose. We need not think of upfront capacity planning. The infrastructure ca be provisioned on demand. There are host of operations which ca be used to configure the access policies. This is a long list of good things, but there are challenges as well. One of the biggest challenges is to get the storage working in the development environment. Ideally each developer should have their own storage. In addition to this there will be storage accounts for unit testing, functional testing etc. Each of these accounts will require setup and incur a billing. These costs can spiral out the budgeted costs.

## Local Infrastructure

Additional to the cost issues the cloud infrastructure can be problem while local development. Developers would often try to run everything on the local machine. But running on local is not possible for the cloud infrastructure. Moreover if the internet connection is poor then overall development takes a hit, as the application takes long time to connect to the cloud infrastructure.

In order to address these issues AWS comes with [`localstack`](https://github.com/localstack/localstack). The `localstack` is capable of running AWS mocked services on the local machine so that the developer need not to connect the cloud infrastructure for development. The `localstack`  is not a replacement of AWS services, it is just a mock of services. I would normally build the application using `localstack` and then connect to the actual services in the development environment.

In my last project we started working with `Azure` infrastructure. We started using the `blob store`, `queue` and `outlook` components. Unlike the AWS `localstack` there is no local suite of services for `Azure` infrastructure. Our customer provided the Azure infrastructure components. But then I needed to be on VPN to work with them. The VPN connectivity lead to overall slow access. Thus I created individual components for development, but each component had a free time period of 30 days. So every 30 days, I needed to create a new email account and created components using it. This was a pain.

### Azurite

So I started looking out ways to work around the issues. Azure docs explained [Azurite](https://github.com/azure/azurite) service which can be used to mock the storage and queue infrastructure. It was easy to get started with the service using its docker image :

```
$docker run -p 10000:10000 mcr.microsoft.com/azure-storage/azurite
```
It would run emulators for blob and queue services on port `10000` of `127.0.0.1`. Selectively we can run only the blob service or the queue service by supplying additional arguments.  

In order to connect to the above service the recommended way is to configure `UseDevelopmentStorage` flag. The flag enables a default account by the name of `devstoreaccount1` along-with the localhost settings.

```
StringBuilder connectionBuilder = new StringBuilder();
if (azureProperties.isDevelopmentMode()) {
    connectionBuilder
            .append("UseDevelopmentStorage=true;");
} else {
    connectionBuilder.append(String.format("DefaultEndpointsProtocol=%s;", blobStorage.getProtocol()))
            .append(String.format("AccountName=%s;", blobStorage.getAccountName()))
            .append(String.format("AccountKey=%s;" blobStorage.getAccountKey()))
            .append(String.format("EndpointSuffix=%s,", blobStorage.getSuffix()));
}
log.info("Generated Connection String ::: {} ", connectionBuilder.toString());
```
> If the service is hosted on a separate host it ca be configured using `DevelopmentStorageProxyUri` flags

The `UseDevelopmentStorage` is provided for convenience purpose. The emulator can be registered with user specified account and keys. The service can then be connected using `DefaultEndpointsProtocol`, `AccountName` and `AccountKey` only.

The above integration helped me to address Azure blob and queue issues. For the outlook service I ended up creating a test switch, which dumped the mail to a text file.
