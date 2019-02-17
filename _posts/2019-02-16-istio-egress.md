---
layout: post
title:  Working with Istio Ingress Gateway
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, istio, microk8s]
---

spring-v2 ver 0.2.0


```
RestTemplate restTemplate = new RestTemplate();
JsonNode data = restTemplate.getForObject("https://en.wikipedia.org/w/api.php?format=json" +
        "&action=query&generator=random&" +
        "grnnamespace=0&prop=revisions|images" +
        "&rvprop=content&grnlimit=1", ObjectNode.class);

List<JsonNode> title = data.findValues("title");
String name= title.isEmpty() ? "I didn't find one" : title.get(0).asText();
return "Greetings ! Here is an interesting topic for you :" + name;
```

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

`microk8s.kubectl get all`

`microk8s.kubectl describe pod/spring-gs-v2-db88849fc-9222l`

`curl http://10.1.1.28:8080/`

```
{"timestamp":"2019-02-16T14:18:19.996+0000","status":500,"error":"Internal Server Error","message":"I/O error on GET request for \"https://en.wikipedia.org/w/api.php\": Remote host closed connection during handshake; nested exception is javax.net.ssl.SSLHandshakeException: Remote host closed connection during handshake","path":"/"}
```


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

```
$ microk8s.kubectl create -f istio-egress.yam
serviceentry.networking.istio.io/wikipedia created
virtualservice.networking.istio.io/wikipedia created
```

`curl http://10.1.1.28:8080/`
`Greetings ! Here is an interesting topic for you :Priory Church of St George, Dunster`
