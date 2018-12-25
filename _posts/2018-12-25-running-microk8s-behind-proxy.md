---
layout: post
title: Running microk8s behind proxy
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
#tags: [test]
---
 My work workplace is an enterprise with all kinds of administrative controls. Access to the outside network is restricted and routed via a proxy. Now, one day I thougtout of doing some POC with `microk8s`. But the chanllange is to get it rubbibg on our local box(typically windows). So I bootstarpped a VM using `Vagrant`. Now the chanllange is to get `microk8s` running on the box.

 First steps were easy, microk8s is bundled via `snap`. So I installed it using `apt`. Here as well `apt` needed proxy setting so I first exported the required variables :
 ```
 export http_proxy='http://user:password@proxy.net:8080/'
 export https_proxy='http://user:password@proxy.net:8080/'
 sudo -E apt install snap
 ```  

Now, `snap` is on my box, now I tried installing `microk8s` but it failed to reach the `snapcraft.io` repository.
```
sudo snap install microk8s --classic
```

 `snap` runs a daemon `snapd` to manage its applications. So we need to configure the daemon to bun with proper environment variables, rather then passing them via command line (as in `apt`). Thus we need to add `proxy` settings to `/etc/environment`.

```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
http_proxy='http://user:password@proxy.net:8080/'
https_proxy='http://user:password@proxy.net:8080/'
HTTP_PROXY='http://user:password@proxy.net:8080/'
HTTPS_PROXY='http://user:password@proxy.net:8080/'
no_proxy=127.0.01,:::1,localhost
```

Now reload the `snapd` service and we are good to install `microk8s`

```
sudo systemctl restart snapd
sudo snap install microk8s
```

Now that I have `microk8s` running on my box, I checked its status using `microk8s.status`. It said `service failed: unable to resolve gcr.io`. Now there must be something missing as I have uanble to reach google container registry. After spending sometime on it, I realised `microk8s` is running `docker` for running `kubernetes`. At first it felt, I need to add proxy varibles to docker daemon. That's true, but still `unable to resolve gcr.io` looks to be something else.

Later I found out the `unable to resolve gcr.io` is a `dns` error. Basically, my VM depends on host dns details, but these details are not known to the docker daemon. It took `dns` setting from VM, for which the dns server was `localhost`. These settings can be added to a `docker.json` file. Now, lets first find the docker daemon config file :

```
ps -ef | grep dockerd
root     25567     1  1 Dec06 ?        00:15:01 /snap/microk8s/340/usr/bin/dockerd --add-runtime nvidia=/snap/microk8s/340/usr/bin/nvidia-container-runtime -H unix:///var/snap/microk8s/340/docker.sock --exec-root /var/snap/microk8s/common/var/run/docker --graph /var/snap/microk8s/common/var/lib/docker --pidfile /var/snap/microk8s/common/docker-pid --config-file=/var/snap/microk8s/340/args/docker-daemon.json
```

We need to update `/var/snap/microk8s/340/args/docker-daemon.json` for `dns` and `dns-search` attributes :
```
{
  "insecure-registries" : ["localhost:32000"],
  "dns":["11.0.0.1", "11.0.1.1"],
  "dns-search":["fx.xlsgrp.net"]
}
```
Post the update restart docker usig `sudo systemctl reload snap.microk8s.daemon-docker.service`. Once started we can validate the change by executing `microk8s.docker run --env HTTPS_PROXY=http://user:password@proxy.net:8080/ --env HTTP_PROXY=http://user:password@proxy.net:8080/ -it ubuntu`. The command should work as expected. The proxy variables can be supplied in a dockerfile as follows :
```
ENV HTTPS_PROXY http://user:password@proxy.net:8080/
ENV HTTP_PROXY http://user:password@proxy.net:8080/
```

Now, all we are left is to provide these variables to the docker runtime from k8s engine. As per the docs these variables must be set in `kubelet` configuration. So now we edit `/etc/systemd/system/snap.microk8s.daemon-kubelet.service` and add these varibles :
```
[Service]
ExecStart=/usr/bin/snap run microk8s.daemon-kubelet
SyslogIdentifier=microk8s.daemon-kubelet
Restart=on-failure
WorkingDirectory=/var/snap/microk8s/340
TimeoutStopSec=30
Type=simple
Environment="HTTPS_PROXY=http://user:password@proxy.net:8080/" "HTTP_PROXY=http://user:password@proxy.net:8080/"
```

All we are now left is to save the file, reload the configuration and restart microk8s.kubelet service
```
sudo systemctl daemon-reload
sudo systemctl reload snap.microk8s.daemon-kubelet.service
```

It took quite a while to get this setup working !
