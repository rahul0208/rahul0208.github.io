---
layout: post
title:  Working with Istio Ingress Gateway
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, istio, microk8s]
---
Previously I configured [Istio on Microk8s](/2019-02-03-configure-istio/) and deployed a sample spring-greeting service on it. I explored Istio telemetary, looking through the various dashboards it made available in Grafana. As I accessed deployed [spring-greeting](http://10.152.183.60:8080/) service I could see the dashboard capturing the traffic.

![Spring-gs-telemetry](/img/istio-ingress-gateway/spring-gs-telemetry.png)

But! I am still accessing the service via NodePort. In a service mesh the services are not exposed outside the mesh. The service can be accessed in the cluster via the NodePort. In Microk8s I am running a K8s cluster on my workstation so I can still acess it. Istio offers an `IngressGateway` which can be used in such scenarios to access the service outside of the cluster. I will configure it and try to acess spring-gs using it. Let's first verify Istio IngressGateway service  using `microk8s.kubectl get svc  -n istio-system`

![Istio-services](/img/istio-ingress-gateway/svc.png)

The service list contains `istio-ingressgateway` of `LoadBalancer` type. The service exposes various port mappings out of which I will configure the `http port` for our spring-gs. Let's now `describe` the service to know the port mapping by using `microk8s.kubectl describe svc istio-ingressgateway -n istio-system`.

![Ingressgateway-port-mapping](/img/istio-ingress-gateway/port-mapping.png)

The output shows that `http 80 port` is mapped to `31380`. According to the docs if there is an external LoadBalancer the the `ingressgateway` will attach itself to the external IP provided by the LoadBalancer. I am running Microk8s on my workstation so all ports are exposed on the `localhost` (or the machine IP). Lets do a `curl` to find out :
```
$curl -v http://localhost:31380/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 31380 (#0)
> GET / HTTP/1.1
> Host: localhost:31380
> User-Agent: curl/7.58.0
> Accept: */*
>
* Recv failure: Connection reset by peer
* stopped the pause stream!
* Closing connection 0
curl: (56) Recv failure: Connection reset by peer
```

The output shows the URL exists but the connection was closed by the server-side. Istio has very fine grained permission model. It needs to be configured for permissioned requests. In order to do so I need to add a `Gataway`.

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
    name: gateway
spec:
    selector:
        istio: ingressgateway
    servers:
        - port:
              number: 80
              name: http
              protocol: HTTP
          hosts:
              - '*'
```
The above configuration creates a `Gateway` which allows access to all hosts. In order to limit the acess froma specific host wee must add its FQ name for `hosts` list. Lets do a `curl` now :
```
$ curl -v http://localhost:31380/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 31380 (#0)
> GET / HTTP/1.1
> Host: localhost:31380
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 404 Not Found
< date: Tue, 12 Feb 2019 05:13:05 GMT
< server: envoy
< content-length: 0
<
* Connection #0 to host localhost left intact
```
Now server returns a `404` error. This means the location is *Not Found*. Location mapping is configured using `VirtualService`. It can be configured in the following manner :

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sping-gs-vs
spec:
  hosts:
  - "*"
  gateways:
  - gateway
  http:
  - match:
    - uri:
        prefix: /spring-gs
    rewrite:
      uri: /
    route:
    - destination:
        port:
          number: 8080
        host: spring-gs
```

In the above configured we have created a `VirtualService` which is attached to a list of `gateways`. The `VirtualService` perfoms a URL rewrite from `/spring-gs` to `/` and sends it to `spring-gs:8080`. Lets do a `curl` and validate the configuration :

```
$ curl -v http://localhost:31380/spring-gs
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 31380 (#0)
> GET /spring-gs HTTP/1.1
> Host: localhost:31380
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< content-type: text/plain;charset=UTF-8
< content-length: 27
< date: Tue, 12 Feb 2019 05:20:11 GMT
< x-envoy-upstream-service-time: 8
< server: envoy
<
* Connection #0 to host localhost left intact
Greetings from Spring Boot!
```

The output above shows `HTTP 200` response. The above deployed compoenets are also availbe via `istioctl` commnad line : `microk8s.istioctl get all`

![istioctl-cli](/img/istio-ingress-gateway/istioctl-cli.png)
