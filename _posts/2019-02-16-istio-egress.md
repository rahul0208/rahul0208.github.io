---
layout: post
title:  Configure External access with Istio Egress
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, istio, microk8s]
---
Istio offers rich set of traffic management capabilities. Recently I tried to access service outside of cluster as a result I had to copnfigure Istio Egress resource. Continuing the journey with `spring-greeting` service, I added a next version of the servcice which will fetch a random page title from `Wikipedia`. `Wikipedia` offers a Rest api to accomplish the same in the following manner :

```
RestTemplate restTemplate = new RestTemplate();
JsonNode data = restTemplate.getForObject("https://en.wikipedia.org/w/api.php?format=json" +
        "&action=query&generator=random&" +
        "grnnamespace=0&prop=revisions|images" +
        "&rvprop=content&grnlimit=1", ObjectNode.class);

List<JsonNode> title = data.findValues("title");
String titleName= title.isEmpty() ? "" : title.get(0).asText();
return "Greetings ! Here is an interesting topic for you : " + titleName;
```

The above code is shipped as `spring-gs:0.2.0`. I had to build the `docker` image for the same with tag as `spring-gs:0.2.0`.  Post this I updated the deployment file for the *said tag* and deployed it to my `microk8s` cluster.
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-gs-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: spring-gs
        version: v0.2.0
    spec:
      containers:
      - name: spring-gs
        image: spring-gs:0.2.0
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
Now when I look at the cluster(`microk8s.kubectl get all`), I can see both version of `spring-gs` running. All left is to lookup the URL of `spring-gs-v2`. This can be done is many ways, but I will lookup IP of the container to do so (`microk8s.kubectl describe pod/spring-gs-v2-db88849fc-9222l`). The service is running on `8080` port of the respective container. Now `curl` to the said location `curl http://10.1.1.28:8080/`. The request fails with the following error .

```
{"timestamp":"2019-02-16T14:18:19.996+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"https://en.wikipedia.org/w/api.php\": Remote host closed connection during handshake; nested exception is javax.net.ssl.SSLHandshakeException: Remote host closed connection during handshake","path":"/"}
```
The failure is also reported in the *Istio dashboard*.

![access denied](/img/istio-egress/egress-denied.png)

As per Istio docs :
>>> By default, Istio-enabled services are unable to access URLs outside of the cluster because the pod uses iptables to transparently redirect all outbound traffic to the sidecar proxy, which only handles intra-cluster destinations.

Had the service been deployed in K8s cluster without Istio it would have worked fine. But in-order to allow external service first I would define it using a `ServiceEntry`.

```
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: wikipedia
spec:
  hosts:
  - en.wikipedia.org
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
```
Now, like any our deployed Istio service I woud need to define a `VirtualService` for its access by components of the cluster.

```
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: wikipedia
spec:
  hosts:
  - en.wikipedia.org
  tls:
  - match:
    - port: 443
      sni_hosts:
      - en.wikipedia.org
    route:
    - destination:
        host: en.wikipedia.org
        port:
          number: 443
      weight: 100
---
```
I added the above two components to my `microk8s` cluster.

```
$ microk8s.kubectl create -f istio-egress.yaml
serviceentry.networking.istio.io/wikipedia created
virtualservice.networking.istio.io/wikipedia created
```
Now I again accessed the said URL :`curl http://10.1.1.28:8080/`. The srevice now responds back with an expected response `Greetings ! Here is an interesting topic for you :Priory Church of St George, Dunster`. This can also be validated in *Istio dashboard*.

![access allowed](/img/istio-egress/egress-allowed.png)


