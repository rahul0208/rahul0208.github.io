---
layout: post
title:  Configure Istio in Microk8s
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [k8s, istio, microk8s]
---

`microk8s.enable istio`

`microk8s.status`

`microk8s.kubectl get all -n istio-system`

`microk8s.kubectl label --overwrite namespace default istio-injection=true`

`microk8s.kubectl get pod`

`microk8s.kubectl delete pod spring-gs-555cd5cd47-i786d`
`microk8s.kubectl get pod`
`microk8s.kubectl describe pod spring-gs-555cd5cd47-h852d`

`microk8s.istioctl proxy-status`

`while true; do curl "http://10.152.183.60:8080/"; sleep 1; done`

grafana service
##telemetry http://10.152.183.33:3000/
