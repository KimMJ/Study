# How istio works

## How Istio works

* 사이드카 프록시(Envoy)가 워크로드에 붙어있음. (Kubernetes Pods, VMs)
* 프록시로 모든 inbound/outbound 트래픽이 지나간다.

쿠버네티스에서 pod는 하나 이상의 컨테이너를 가질 수 있음. 이 파드 내에서는 localhost로통신이 가능. 따라서 파드 안의 프록시와 서비스가 통신을 함. 

Envoy는 c++로 작성된 layer7의 고성능 프록시이며 오픈소스이다.

* Istio Control Plane은 쿠버네티스에 배포가 된다. 
* Istio API는 쿠버네티스의 CRD(custom resource definition)로 설치된다.
  * CRD는 custom resource definition이다. 즉, 우리가 Istio policy, rule들과 상호작용할 수 있고 이 모든것을 kubectl을 통해서 할 수 있다.

Istio는 모든 네트워크 로직을 떼어내서 operator가 획일화된 방법으로 관리할 수 있도록 한다.

## Istio Traffic Configuration

어떻게 Istio가 사이드카 프록시와 어떤 일을 한다고 할 수 있을까? Istio는 트래픽 API를 공개한다. 이는 여러 오브젝트를 공개한다.

#### Gateways in an Istio service mesh

![Gateways in an istio service mesh](https://istio.io/blog/2018/v1alpha3-routing/gateways.svg)

#### Relationship between different v1alpha3 elements

![Relationship between different v1alpha3 elements](https://istio.io/blog/2018/v1alpha3-routing/virtualservices-destrules.svg)

### DestinationRule

Destination ruledms 서비스를 subset으로 그룹짓는 방법이다. 서브셋은 이름이 지정된 버전이고 이를 통해 쿠버네티스 서비스를 할 수 있게하고 원하는 어떤 방법으로든 파티션지을 수 있게 한다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

이 destination rule은 virtual service를 뒷받침한다. 그리고 이 트래픽 룰은 Istio에서 꼭 알아야 할 API 오브젝트이다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend
spec:
  hosts:
  - frontend
  http:
  - route:
    - destination:
        host: frontend
        subset: v1
      weight: 90
    - destination:
        host: frontend
        subset: v2
      weight: 10
```

### Demo: Rollout

* Scale : 다양한 버전에 대해 트래픽을 전달하여 scale한다.
* Release new versions : 운영상 문제의 걱정 없이 새 버전을 release할 수 있다.

그냥 트래픽을 보내면 v1과 v2는 라운드로빈에 의해 50:50의 비율을 가질 것이다. 하지만 destination rule을 통해서 9:1의 비율로 트래픽을 보낼 수 있다.

Demo: Content-Based Routing

* HTTP header를 기반으로 트래픽을 전송
* production에서 트래픽의 우선순위를 정함.
* A/B 테스트를 수행.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend
spec:
  hosts:
  - "frontend"
  http:
  - match:
    - headers:
        foo:
          exact: bar1
    route:
    - destination:
        host: frontend
        subset: v1
  - route:
    - destination:
        host: frontend
        subset: v2
```

### Demo: Circuit Breaking

* castcading failures를 막는데 도움을 준다.
* 연속적인 failure -> circuit breaker를 이용 -> 즉시 fail 처리

### Demo: Chaos Testing

* 능동적으로 서비스 매쉬의 취약점을 찾는다.
* 장애를 발생시키는 실험을 한다. -> 결과를 분석

Istio에서는 topology상의 한 엣지에 에러를 심을 수 있다.

```yaml
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: productcatalogservice
spec:
  host: productcatalogservice
  subsets:
  - name: v1
    labels:
      version: v1
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productcatalogservice
spec:
  hosts:
  - productcatalogservice
  http:
  - match: #checkout -> productcatalog fails 100% of the time(HTTP status 400)
    - sourceLabels:
        app: checkoutservice
    fault:
      abort:
        percent: 100
        httpStatus: 400
    route:
    - destination:
        host: productcatalogservice
        subset: v1
  - route: # all other productcatalog requests (like from the frontend) work normally
    - destination:
        host: productcatalogservice
        subset: v1
```

# Traffic In

## Istio Ingress Model

* Inbound Istio 트래픽은 Envoy 프록시이다. (Gateway)
* ingress 트래픽을 트래픽 룰로 in-mesh 트래픽과 같은 방법으로 설정할 수 있다. (yaml로 똑같이)

### Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

ingressgateway의 IP로 DNS설정해서 서비스에 연결하도록 할 수 있다.

# Traffic Out

default로 Istio는 outbound 트래픽을 lock down한다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: eubank
spec:
  hosts:
  - www.ecb.europa.eu
  - ecb.europa.eu
  prots:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

### Secure Egress Traffic

* Egress Gateway를 outbound 트래픽을 intercept하는데 사용해라.
* Why?
  * zero-trust 네트워크를 구성하기 위해
  * egress 트래픽이 in-mesh 트래픽보다 더 중요하게 모니터링 할 필요가 있을 때
  * TLS(transport layer security)를 적용시키거나 라우팅 룰을 추가하기 위해.

## Istio 1.1 - Ready for Production

* 1.1 릴리즈는 성능과 scalability에 초점을 두었다.
* latency를 30%정도 줄였다.
* pod의 startup time을 40%정도 빠르게 했다.
* Pilot의 CPU 점유율을 90% 줄였다.
* Pilot의 메모리 점유율을 50% 줄였다.





---

### Load balancing

4가지 타입이 있다.

* Round Robin
* Random
* Weighted
* Least request

## Traffic routing and configuration

* Virtual services
  * 서비스 매쉬에서 라우팅 룰을 어떻게 적용할 지에 대해 ordered list로 설정한다.
* Destination rules
  * virtual service의 라우팅 룰이 시행되고 나서 어떤 정책을 적용하고 싶을 때 사용한다.
* Gateways
  * Envoy 프록시가 HTTP, TCP, gRPC 트래픽에 대해 어떻게 로드 밸런싱을 할지 설정할 때 쓴다.
* Service entries
  * 메쉬의 외부 종속성에 대한 라우팅 룰을 설정하는 entry를 abstract model에 추가할 때 사용한다.
* Sidecars
  * Envoy 프록시가 namespace같은 어떤 feature에 동작하도록 할지 설정한다.

