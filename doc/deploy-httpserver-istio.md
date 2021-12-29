- [Description](#description)
- [Requirement](#requirement)
- [Deploy httpserver service with istio ingress gateway](#deploy-httpserver-service-with-istio-ingress-gateway)
  - [instruct Istio to automatically inject Envoy sidecar proxies](#instruct-istio-to-automatically-inject-envoy-sidecar-proxies)
  - [Deploy httpserver application](#deploy-httpserver-application)
  - [Associate application(httpserver) with the Istio gateway](#associate-applicationhttpserver-with-the-istio-gateway)
  - [Get istio-ingressgateway ingress IP and ports](#get-istio-ingressgateway-ingress-ip-and-ports)
  - [Verify external access](#verify-external-access)
- [安全](#安全)
- [Tracing](#tracing)
# Description
Deploy httpserver with istio
# Requirement
K8s cluster
loadbalancer is deploy (optional, in this lab, I have deployed metallb)
istioctl installed
istio is installed with configuration:
- ✔ Istio core installed                          
- ✔ Istiod installed                              
- ✔ Ingress gateways installed                    
- ✔ Egress gateways installed   
# Deploy httpserver service with istio ingress gateway
## instruct Istio to automatically inject Envoy sidecar proxies
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k label namespaces default istio-injection=enabled
namespace/default labeled
```
## Deploy httpserver application
与普通service部署yaml无差，故重用以前的yaml文件
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k apply -f deployment/myhttpserver.yaml 
deployment.apps/httpserver-deployment created
service/httpserver-service created
sarah@sarah-510-p127c:~/k8s_ansible$ k get pods -l app.kubernetes.io/name=httpserver
NAME                                    READY   STATUS    RESTARTS   AGE
httpserver-deployment-757b546dd-cbnzg   2/2     Running   0          117s
httpserver-deployment-757b546dd-nxsjb   2/2     Running   0          118s
```
## Associate application(httpserver) with the Istio gateway
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k apply -f deployment/myhttpserver-gateway.yaml 
gateway.networking.istio.io/httpserver-gateway created
virtualservice.networking.istio.io/httpserver created
```
## Get istio-ingressgateway ingress IP and ports
Get ingress IP, my ingress IP is metallb pool ip: 192.168.34.200
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

sarah@sarah-510-p127c:~/k8s_ansible$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
sarah@sarah-510-p127c:~/k8s_ansible$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```
## Verify external access
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ curl $GATEWAY_URL/httpservervice/
ok
```
# 安全
由于httpserver docker file 是基于distroless 无shell, 以及已经以 nonroot 身份运行，故暂时未加其它policy或者安全限制.
# Tracing
