- [Description](#description)
  - [Request routing](#request-routing)
- [Requirement](#requirement)
- [Deploy httpserver service with istio ingress gateway](#deploy-httpserver-service-with-istio-ingress-gateway)
  - [instruct Istio to automatically inject Envoy sidecar proxies](#instruct-istio-to-automatically-inject-envoy-sidecar-proxies)
  - [Deploy httpserver application](#deploy-httpserver-application)
  - [Associate application(httpserver) with the Istio gateway](#associate-applicationhttpserver-with-the-istio-gateway)
  - [Get istio-ingressgateway ingress IP and ports](#get-istio-ingressgateway-ingress-ip-and-ports)
  - [Verify external access](#verify-external-access)
- [安全](#安全)
  - [container dockerfile](#container-dockerfile)
  - [deployment yaml spec中引入 securityContext 再次确保](#deployment-yaml-spec中引入-securitycontext-再次确保)
  - [istio apply mutal tls (destination-rule)](#istio-apply-mutal-tls-destination-rule)
- [Tracing](#tracing)
  - [Deploy tracer (zipkin)](#deploy-tracer-zipkin)
  - [Deploy more services to mimic tracing chain](#deploy-more-services-to-mimic-tracing-chain)
    - [部署各dummy service](#部署各dummy-service)
    - [验证 httpserver 与 各dummy service的调用](#验证-httpserver-与-各dummy-service的调用)
  - [Generate test data](#generate-test-data)
  - [查看dashboard (zipkin dashboard)结果](#查看dashboard-zipkin-dashboard结果)
# Description
Deploy httpserver with istio.
## Request routing
There are two versions httpserver deployments, can tell the difference by response header: *version: v1.0.1* or *version: v2.0.1*.

If request header "tester" is prefixed with "tid", then the traffic will be routed to version 2. otherwise, it will be routed to version 1.
# Requirement
K8s cluster
istioctl installed
istio is installed with configuration: (istioctl install -f infra-deployment/istio/my-config.yaml -y)
- ✔ Istio core installed                          
- ✔ Istiod installed                              
- ✔ Ingress gateways installed    
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
service/httpserver created
sarah@sarah-510-p127c:~/k8s_ansible$ k get pods -l app=httpserver
NAME                             READY   STATUS    RESTARTS   AGE
httpserver-v1-7877b4c5c5-264fl   2/2     Running   0          10m
httpserver-v2-57d4897d64-v2rfd   2/2     Running   0          10m
```
## Associate application(httpserver) with the Istio gateway
Deploy destination rule
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k apply -f deployment/myhttpserver-destion-rule.yaml 
destinationrule.networking.istio.io/httpserver created
```
Deploy virtualservice and istio ingress gateway for httpserver
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k apply -f deployment/myhttpserver-gateway.yaml 
gateway.networking.istio.io/httpserver-gateway created
virtualservice.networking.istio.io/httpserver created
```
## Get istio-ingressgateway ingress IP and ports
Get ingress IP, my ingress IP is metallb pool ip: 192.168.34.200
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')

sarah@sarah-510-p127c:~/k8s_ansible$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
sarah@sarah-510-p127c:~/k8s_ansible$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```
## Verify external access
default route
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ curl $GATEWAY_URL/httpservervice/ -v
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< version: v1.0.1
```
version 2 route
```bash
curl -H 'tester: tid1' $GATEWAY_URL/httpservervice/ -v
> tester: tid1
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< version: v2.0.1
```
# 安全
## container dockerfile 
由于httpserver docker file 是基于distroless 无shell, 以及已经以 nonroot 身份运行，故暂时未加其它policy或者安全限制.
## deployment yaml spec中引入 securityContext 再次确保
在deployment yaml中，加入
>
          securityContext:
            runAsUser: 1000
## istio apply mutal tls (destination-rule)
部署istio基于default configuration profiles，所以mutal tls是enabled状态。但依然通过destion-rule declarative 声明为 mtls
```bash
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```
# Tracing
Based on the configuration, the default tracer backend is zipkin, so we just integrate zipkin
## Deploy tracer (zipkin)
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/extras/zipkin.yaml
```
## Deploy more services to mimic tracing chain
根据istio zipkin integration 文档，且前面的deployment已经部署httpserver service, VirtualService, 以及ingressgateway for httpserver。且ingressgateway会自动插入 x-b3- 相关请求头。
以下将模拟以下调用链，当请求 httpserver 服务的/service0 时，httpserver 会开启httpclient请求 dummy service。 各dummy service的调用链为: service0 -> service1 -> service2.
由于ingressgateway 已经插入相关头，在httpserver以及各dummyserver 以 httpclient 身份请求下一服务时，仅需要 forward 这些请求头即可。 
### 部署各dummy service
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ k apply -f deployment/dummyservice/
```
### 验证 httpserver 与 各dummy service的调用
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ curl $GATEWAY_URL/service0
Response from service0: Response from service1: ok
```
## Generate test data
默认的traceSample是1，即1%。
```bash
k -n istio-system get istiooperators.install.istio.io -oyaml | grep traceSampling:
        traceSampling: 1
```
为了在dashboard中看到结果,我们发起100个请求
```bash
for i in {1..100}; do curl $GATEWAY_URL/service0 -s > /dev/null ; done;
```
## 查看dashboard (zipkin dashboard)结果
1. Launch zipkin dashboard
```bash
sarah@sarah-510-p127c:~/k8s_ansible$ istioctl dashboard zipkin 
```
2. 在浏览器中 search by serviceName -> Run query, find the span -> Show 即可看到对应 traceid 以及耗时等。也可通过 Depedencies看到调用图。