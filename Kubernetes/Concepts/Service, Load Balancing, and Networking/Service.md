# Service

* Pod 세트에서 동작하는 application을 네트워크 서비스로 노출시키는 추상적 방법
* 쿠버네티스를 사용하면 사용자는 친숙하지 않은 service discovery 메카니즘을 사용하기 위해서 application을 수정하지 않아도 된다.
  * 쿠버네티스는 파드와 그 IP 주소와 파드 세트에 대한 하나의 DNS name을 부여하고 load-balance를 한다.

## Motivation

* 쿠버네티스 파드는 영원하지 않다(mortal).
  * 생성되고 죽고 부활하지 않는다.
  * 만약 Deployment로 app을 구동하고 있다면 이는 파드를 동적으로 생성하고 없앨 수 있다.
* 각 파드는 그 IP 주소를 가지고 있지만 Deployment에서는 한 순간에 함께 동작중인 파드의 세트는 나중에 동작하는 파드의 세트와는 그 정보가 다르다.
* 이는 문제를 야기한다. 파드의 세트가 (backend라고 하자) 클러스터 안에있는 다른 파드(frontend라고 하자)에 기능을 제공한다면 어떻게 프론트엔드는 어떤 IP 주소에 연결해야하는지 추적하여 워크로드의 백엔드 파트를 사용할 수 있을까?
* 이를 위해 Service가 태어났다.

## Service resources

* Kubernetes에서 서비스는 논리적인 파드의 세트와 어떻게 연결할지(이런 패턴을 micro-service라고 한다)에 대한 정책을 정의하는 abstraction이다.
  * Service가 타게팅한 파드의 세트는 보통 selector에 의해 결정된다.
* 3개의 replica로 동작하는 stateless 이미지 프로세싱 backend가 있다고 생각해보자. 이런 replica는 대체가 가능하다. 즉, frontend는 어떤 backend가 사용되는지 알 필요 없다. 백엔드를 구성하는 실제 파드가 바뀔수 있지만, 프론트엔드 클라이언트는 이를 알 필요도, 벡엔드에 대해 추적관리를 할 필요도 없다.
* Service abstraction은 이런 decoupling을 할 수 있게 해준다.

### Cloud-native service discovery

* 사용자가 쿠버네티스 API로 application에서 service discovery를 할 수 있다면, 서비스가 바뀔때마다 업데이트가 되는 Endpoint를 위한 API server로 쿼리를 보낼 수 있다.
* non-native application에서 쿠버네티스는 application과 backend 파드간 로드벨런서나 네트워크 포트를 위치시키는 방법을 제공한다.

## Defining a Service

* 쿠버네티스의 Service는 파드와 비슷하게 REST 오브젝트이다.

  * 모든 REST 오브젝트가 그렇듯 사용자는 Service definition을 API 서버로 `POST`하여 새로운 인스턴스를 생성할 수 있다.

* TCP 포트 9376을 listen하고 label이 `app=Myapp`인 파드 세트가 있다고 가정하자.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    selector:
      app: MyApp
    ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  ```

  * 이 specification은 "my-service"라는 이름을 가지고 `app=MyApp` 라벨을 가진 파드에 대해 TCP 포트 9376을 지정하는 Service 오브젝트를 생성한다.
  * 쿠버네티스는 이 Service에 IP주소를 할당하고(이를 "cluster IP"라고도 부른다.), 이를 Service proxy가 사용하게 한다.
  * Service selector에 대한 컨트롤러는 기속적으로 selector가 일치하는 파드를 스캔하여 "my-service"라는 이름을 가진 Endpoint 오브젝트에 업데이트를 POST한다

> Note : Service는 어떤 incoming `port`도 `targetPort`로 매핑할 수 있다. 편의상 default로 `targetPort`는 `port`필드와 같은 값으로 설정된다.

* 파드에서 Port 정의는 이름을 가지고있고 사용자는 이 이름을 Service의 `targetPort` 속성으로 참조할 수 있다.
  * 이는 Service에서 같은 네트워크 프로토콜을 다른 포트 번호를 통해 사용할수있는 하나의 configured name을 사용하는 혼합된 파드에서도 동작한다.
  * 이는 Service를 배포하고 발전하는데 많은 유연성을 제공한다.
  * 예를 들어, 사용자는 클라이언트를 망가뜨리지 않고 backend 소프트웨어의 다음 버전에서 파드가 노출하는 포트 번호를 바꿀 수 있다.
* 많은 서비스들이 하나 이상의 port를 노출해야 함에 따라 쿠버네티스는 Service 오브젝트에서 multiple port definition을 지원한다.
  * 각 port definition은 같은 `protocol`을 사용할수도 있고 다른 것을 사용할수도 있다.

### Services without selectors

* Service는 보통 쿠버네티스 파드로의 접속을 추상화하지만 다른 종류의 backend에도 추상화 할 수 있다. 예를 들어:

  * 사용자가 production의 external database cluster를 사용하려고 하지만 테스트 환경에서는 own database를 사용하려고 할 때
  * 사용자가 Service를 다른 네임스페이스나 다른 클러스터에 있는 Service로 point하려 할 때
  * 쿠버네티스에 워크로드를 migrate할 때. approach를 평가하는 동안 사용자는 쿠버네티스의 backend의 일부분만 실행시킨다.

* 어떤 시나리오던지 Service를 Pod selecter 없이 생성할 수 있다.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  ```

* 이 Service는 selector가 없기 때문에, 해당하는 Endpoint 오브젝트는 자동으로 생성되지 않는다. 

  * 사용자가 직접 Endpoint 오브젝트를 생성해서 Service를 어디서 동작하는지에 대한 네트워크 주소와 포트를 매핑해야 한다.

    ```yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: my-service
    subsets:
      - addresses:
          - ip: 192.0.2.42
        ports:
          - port: 9376
    ```

> Note : 
>
> endpoint IP는 반드시 loopback(127.0.0.0/8 for IPv4, ::1/128 for IPv6)이나 link-local(169.254.0.0/16, 224.0.0.0/24 for IPv4, fe80::/64 for IPv6)이 되어서는 안된다.
>
> endpoint IP 주소는 다른 Kubernetes Service의 cluster IP가 될 수 없다. kube-proxy가 destination으로 virtual IP를 지원하지 않기 때문이다.

* selector가 없는 Service로 접속하는 것은 selector가 있는 것과 같은 방식으로 동작한다. 위의 예시에서 traffic은 YAML에서 `192.0.2.42:9376(TCP)`로 정의된 single endpoint로 전송된다.
* ExternalName Service는 Service의 특이 케이스로 selector를 가지지 않고 대신 DNS 이름을 사용한다.

## Virtual IPs and service proxies

* 모든 쿠버네티스 클러스터의 노드는 `kube-proxy`를 실행하고 있다. `kube-proxy`는 `ExternalName`과는 다른 타입의 `Service`를 위해 virtual IP의 형태를 시행할 책임을 가진다.

### Why not use round-robin DNS?

* 매일 올라오고 질문은 Kubernetes가 왜 proxy에 의존하여 inbound traffic을 backend로 전달하는가에 대한 것이다.
  * 다른 방법으로는 DNS record를 설정하여 여러개의 `A value`(IPv6에서는 AAAA)를 가지도록 하여 round-robin name resolution을 사용하면 어떨까?
* `Services`에 프록시를 사용하는 이유
  * record TTL, 만료되고 난 후의 name lookup의 결과를 캐싱을 고려하지 않은 DNS의 구현에는 오랜 역사가 있다.
  * 어떤 app은 DNS lookup을 한번만 하고 영원히 결과값을 캐싱한다.
  * app과 library들이 적절한 re-resolution을 하더라도 DNS record에 대한 낮거나 0인 TTL은 DNS에 큰 부하를 주고 관리하기 어려워 지게 된다.

### Version compatibility

* 쿠버네티스 v1.0 이후로 userspace proxy mode를 사용할 수 있다.
* 쿠버네티스 v1.1에서는 iptables mode proxying이 추가되었다.
* 쿠버네티스 v1.2에서는 kube-proxy의 iptables mode가 기본값이 되었다.
* 쿠버네티스 v1.8에서는 ipvs proxy mode가 추가되었다.

### User space proxy mode

* 이 모드에서는 kube-proxy가 Kubernetes master를 관찰하여 Service와 Endpoint 오브젝트의 추가나 삭제를 확인한다.
  * 각각의 서비스는 local node에 대해 랜덤으로 선택된 port를 연다.
  * 이 "proxy port"로의 연결은 Endpoints를 통해 Service의 backend Pods중 하나로 프록시 된다.
  * `kube-proxy`는 Service의 세팅인 `SessionAffinity`를 고려하여 어느 backend Pod가 사용될지 결정한다.
* 마지막으로, user-space proxy는 iptables rule을 설치하여 Service의 `clusterIP`(virtual)와 `port`로 가는 트래픽을 감지한다.
  * rule은 그 트래픽을 backend Pod로 프록시하는 proxy port로 redirect한다.
* 기본적으로 userspace mode에서 `kube-proxy`는 backend를 round-robin 알고리즘으로 선택한다.

![services-userspace-overview](img\services-userspace-overview.png)

### iptables proxy mode

* 이 모드에서는 kube-proxy가 Kubernetes control plane을 관찰하여 Serivce와  Endpoint 오브젝트의 추가나 삭제를 확인한다.

  * 각각의 Service는 iptables rules를 설치하여 Service의 `clusterIP`와 `port`로 가는 트래픽을 감지하고 이를 Service의 backend set중 하나로 트래픽을 redirect한다.
  * 각각의 Endpoint 오브젝트는 iptables rules를 설치하고 어느 backend Pod를 선택할지를 결정한다.

* default로 iptables mode에서 kube-porxy는는 backend를 랜덤으로 선택한다.

* iptable을 사용하여 트래픽을 관리하는 것은 트래픽이 userspace와 kernel spcae를 바꿀 필요 없이 Linux netfilter를 사용하여 관리되기 때문에 system overhead가 더 적다.

  * 이 접근법은 또한 더 믿을수 있다.

* kube-proxy가 iptabels mode에서 동작하고 있고 선택된 첫번째 pod가 응답을 하지 않는다면, connection은 실패한다.

  * 이는 userspace mode와는 다르다
    * userspace mode에서는 kube-proxy가 그 첫번째 파드로의 연결이 실패했음을 감지하고 자동으로 다른 backend Pod로 연결을 재시도 할 것이다.

* Pod의 readiness probe를 사용하여 backend Pod가 제대로 동작하는지 확인할 수 있어 iptables mode를 사용하는 kube-proxy가 healthy한 backend만 볼 수 있도록 해준다.

  * 이는 kube-proxy를 통하여 failed된 파드로 트래픽을 보내는 것을 방지해준다.

  ![services-iptables-overview](img\services-iptables-overview.png)

### IPVS proxy mode

**FEATURE STATE:** `Kubernetes v1.11` - stable

* `ipvs` mode에서는 kube-proxy가 Kubernetes Service와 Endpoint를 관찰하여 `netlink`라고 불리는 인터페이스를 호출하여 IPVS rules를 이에 맞게 생성하고 IPVS rules를 Kubernetes Services와 Endpoint랑 주기적으로 synchronize한다.

  * 이 control loop은 IPVS 상태가 원하는 상태가 되도록 한다.
  * Service에 접속할 때, IPVS는 backend Pods 중 하나로 트래픽을 direct한다.

* IPVS proxy mode는  iptables mode와 비슷한 netfilter hook function을 기반으로 하지만 hash table을 data structre로 사용하고 kernel space에서 동작한다.

  * 이는 IPVS mode의 kube-proxy가 iptables mode의 kube-proxy보다 트래픽을 더 낮은 지연속도를 통해서 redirect하고 proxy rule을 synchronising하는 데 훨씬 더 좋은 성능을 낸다는 것을 의미한다.
  * 다른 proxy mode와 비교했을 때, IPVS mode는 network traffic에서 더 좋은 throughput을 제공한다.

* IPVS는 backend Pods로 트래픽을 balancing하는 것에 더 많은 옵션이 있다.

  * `rr` : round-robin
  * `lc` : least connection (최소 개수의 open connection)
  * `dh` : destination hashing
  * `sh` : source hashing
  * `sed` : shortest expected delay
  * `nq` : never queue

  > Note : IPVS mode에서 kube-proxy를 동작시키기 위해서는 kube-proxy를 실행하기 전에 반드시 IPVS Linux를 node에서 사용할 수 있도록 해야한다.
  >
  > kube-proxy가 IPVS proxy mode를 시작할 때 IPVS kernel module이 사용가능한지 확인한다. 만약 IPVS kernel module이 감지되지 않으면 kube-proxy는 iptables proxy mode로 바뀌어 동작한다.

  ![services-ipvs-overview](img\services-ipvs-overview.png)

* 이 proxy model에서 Service의 IP:Port에 대한 traffic bound는 client가 Kubernetes, Services, Pods에 대해 아무런 정보도 없어도 적절한 backend로 프록시 된다.

* 만약 특정한 client로부터 오는 connection이 매번 같은 pod로 보내지도록 하고싶으면 "ClientIP"(default는 "None")에 `service.spec.sessionAffinity`를 설정하여 client의 IP 주소에 기반을 한 session affinity를 선택할 수 있다.

  * 또한 `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`를 적절하게 설정하여 maximum session sticky time을 설정할 수 있다. (default는 10800이며 3시간동안 동작하는 것이다.)

## Multi-Port Services

* 어떤 Service에서는 하나 이상의 port를 열어야 할 필요가 있다.

  * Kubernetes는 Service 오브젝트에서 multiple port definition을 설정할 수 있다.

  * Service에서 multiple port를 사용하면 사용자는 반드시 모든 port의 이름을 주어 모호하지 않도록 해야한다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
      - name: http
        protocol: TCP
        port: 80
        targetPort: 9376
      - name: https
        protocol: TCP
        port: 443
        targetPort: 9377
    ```

> Note : 일반적인 Kubernetes 이름들처럼 port를 위한 이름은 소문자 영어, 숫자, `-`로만 이루어져야 한다. Port 이름은 영문자 또는 숫자로만 시작하고 끝나야 한다.
>
> 예를 들어 `123-abc`와 `web`은 유효하지만 `123_abc`와 `-web`은 유효하지 않다.

## Choosing your own IP address

* 사용자는 고유한 cluster IP 주소를 `Service` 생성요청의 일부분으로 명시할 수 있다.
  * 이를 위해 `spec.clusterIP` 필드를 설정하면 된다.
  * 예를 들어, 사용자가 이미 DNS entry를 가지고 있고 이를 reuse하기를 원하거나 특정한 IP 주소에 대해 configure되어있는 legacy system이 있어 re-configure하기 어려울 때.
* 사용자가 고른 IP 주소는 API server로 설정이 된 `service-cluster-ip-range` CIDR range내에서 반드시 유효한 IPv4 또는 IPv6 주소여야 한다.
  * 만약 사용자가 Service를 유효하지 않은 cluster IP 주소값으로 생성하려 한다면 API server는 422 HTTP status code를 리턴하여 문제가 있음을 알릴 것이다.

## Discovering services

* Kubernetes는 Service를 검색하는데 2가지 primary mode를 가지고 있다.
  * environment variables
  * DNS

### Environment variables

* Node에서 파드가 동작할 때 kubelet은 각각의 active Service에 환경 변수를 추가한다.

  * 이는 Docker links compatible variables([makeLinkVariables](http://releases.k8s.io/master/pkg/kubelet/envvars/envvars.go#L49)참조)와 더 간단한 Service 이름이 대문자로 되어있고 dash가 밑줄로 변경된  `{SVCNAME}_SERVICE_HOST`와 `{SVCNAME}_SERVICE_PORT` variables를 지원한다.

* 예를 들어 TCP port 6379를 여는 Service `"redis-master"`는 cluster IP 주소가 10.0.0.11로 할당되었고 다음의 환경변수를 생성한다.

  ```bash
  REDIS_MASTER_SERVICE_HOST=10.0.0.11
  REDIS_MASTER_SERVICE_PORT=6379
  REDIS_MASTER_PORT=tcp://10.0.0.11:6379
  REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
  REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
  REDIS_MASTER_PORT_6379_TCP_PORT=6379
  REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
  ```

> Note : Service에 접속해야 하는 파드가 있고, 환경 변수를 이용하여 port와 cluster IP를 client Pod로 전송하고 싶으면 반드시 Service를 Pods가 존재하기 전에 생성해야 한다. 그렇지 않으면 client Pods는 환경 변수를 가지지 않는다.

### DNS

* 사용자는 아마도 거의 항상 add-on을 이용하여 Kubernetes cluster에 DNS service를 set up할 것이다.
* CoreDNS같은 cluster-aware DNS server는 new Services와 각 DNS records의 생성을 Kubernetes API를 관찰한다.
  * DNS가 cluster 전체에 enable 되어있다면 파드는 자동으로 DNS name을 통해 Service를 해석한다.
* 예를 들어 `"my-service"`라고 하는 Service가 Kubernetes Namespace `"ms-ns"`에 있다면, control plane과 DNS Service는 함께 `"my-service.my-ns"`에 대한 DNS record를 생성한다.
  * `"my-ns"` namespace에 있는 파드는 `my-service`(`"my-service.my-ns"`도 가능)에 대한 이름 검색을 통해서 쉽게 찾을 수 있어야 한다.
* 다른 namespace에 있는 파드는 반드시 `my-service.my-ns`로 사용해야 한다.
  * 이 이름은 Service에 할당된 cluster IP로 해석될 것이다.
* Kubernetes는 또한 named ports를 위한 DNS SRV (Service) records를 지원한다.
  * `"my-service.my-ns"` Service가 `"http"` 이름을 가지고 있고, protocol은 `TCP`로 설정이 되어 있으면, `_http._tcp.my-service.my-ns` DNS SRV query를 보내서 `"http"`의 port 번호와 IP 주소를 알 수 있다.
* Kubernetes DNS server는 `ExternalName` Service에 접속하는 유일한 방법이다.
  * `ExternalName`에 대한 더 많은 정보는 [DNS Pods and Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)참조.

## Headless Services

* 때때로 load-balancing이나 single Service IP가 필요없을 때가 있다.
  *  이 경우에 cluster IP(`.spec.clusterIP`)를 명시적으로 `"None"`으로 설정하여 "headless"라 불리는 Service를 생성할 수 있다.
* Kubernetes의 구현과 연결되지 않은 다른 service discovery 메카니즘과의 인터페이스를 위해 headless Service를 사용할 수 있다.
  * 예를 들어, 이 API로 custom Operator를 사용할 수 있다.
* cluster IP가 할당되지 않은 `Services`에서는 kube-proxy는 이 Service를 다루지 않고 load balancing이나 proxy가 플랫폼에 의해 동작하지 않는다.
  * 어떻게 DNS가 자동으로 configure하는지는 Service에 selector가 정의되었는지에 따라 달라진다.

### With selectors

* selector가 정의된 headleadd Service에서는 endpoint controller가 `Endpoints` records를 API에 생성하고 DNS configuration을 수정하여 `Service`를 지원하는 `Pods`를 직접 가르키는 records(addresses)를 리턴한다.

### Without selectors

* selectors가 정의되지 않은 headless Services에서는 endpoints controller가 `Endpoints` records를 생성하지 않는다.
  * 하지만 DNS 시스템은 다음과 같은 것들을 찾아 설정한다.
    * `ExternalName` 타입의 Service에서 CNAME records
    * Service와 이름을 공유하는 `Endpoints`의 records, 기타 모든 타입들

## Publishing Services (Service Types)

* application(예를 들어, frontend)의 어떤 부분에서 사용자는 Service를 cluster 밖에 있는 외부 IP 주소에 노출시키고 싶을 것이다.

* Kubernetes의 `ServiceTypes`는 사용자가 어떤 Service를 원하는지 지정할 수 있다.
  
  * default는 `ClusterIP`이다.
  
* `Type`값에 따른 동작은 다음과 같다.
  * `ClusterIP`
    * Service를 cluster-internal IP에 노출시킨다.
    * 이 값을 선택하는 것은 Service가 cluster 안에서만 연결될 수 있도록 한다.
    * `ServiceType`의 default값이다.
  * [`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)
    * static port(`NodePort`)로 각 노드의 IP에 Service를 노출시키는 것.
    * `NodePort` Service가 라우팅하는 ` ClusterIP` Service는 자동으로 생성이 된다.
    * `<NodeIP>:<NodePort>`를 요청하여 cluster 밖에서 `NodePort` Service로 연결할 수 있다.
  * [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
    * cloud 제공자의 load balancer를 이용하여 Service를 외부로 노출한다.
    * external load balancer가 라우팅하는 `NodePort`와 `ClusterIP` Service는 자동으로 생성된다.
  * [`ExternalName`](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)
    * `CNAME` record를 리턴하여 `externalName` 필드(예를 들어 `foo.bar.example.com`)의 컨텐츠에 서비스를 매핑한다.
  
* 어떤 종류의 proxying도 설정되지 않았다.

  > Note : `ExternalName` 타입을 사용하기 위해서는 1.7 이상 버전의 CoreDNS가 필요하다.

* 사용자는 또한 Service를 노출시키기 위해 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)를 사용할 수도 있다.

  * Ingress는 Service 타입은 아니지만 cluster에 대해서 entry point로 작동한다.
  * 이는 routing rules를 하나의 리소스로 통합하여 같은 IP 주소에 대해서 multiple services를 노출할 수 있도록 한다.

### Type NodePort

* `type` 필드를 `NodePort`로 설정하면 Kubernetes control plane은 `--service-node-port-range` flag(default: 30000~32767)에 의해 정해진 range에서 port를 할당할 것이다.
  * 각 노드는 그 port(모든 노드에서 같은 port 번호)로 Service에 프록시 할 것이다.
  * Service는 `.spec.ports[*].nodePort` 필드에 할당된 port를 기록할 것이다.
* 특정한 IP(s)를 통해서 port에 프록시를 하고 싶으면 `--nodeport-address` 플래그를 특정한 IP block(s)로 kube-proxy에 세팅할 수 있다.
  * 이는 Kubernetes v1.10부터 지원이 된다.
  * 이 플래그는 IP blocks의 comma-delimited 리스트(e.g. 10.0.0.0/8, 192.0.2.0/25)를 가져서 kube-proxy가 이 노드의 local이라고 생각하는 IP 주소 범위를 지정할 수 있다.
* 예를 들어, kube-proxy를 `--nodeport-addresses=127.0.0.0/8` 플래그를 통해서 시작하면 kube-proxy는 NodePort Services에 대해 loopback 인터페이스만 선택한다.
  * `--nodeport-addresses`의 default는 빈 리스트이다.
  * 이는 kube-proxy가 NodePort에 대해 모든 가능한 network interface를 고려해야한다는 것을 의미한다. (이전의 Kubernetes releases와 연동이 된다.)
* 만약 특정한 port 번호를 원한다면 `nodePort` 필드에 값을 지정할 수 있다.
  * control plane은 그 port에 대해 할당하거나 API transcation이 실패했다고 알린다.
  * 이는 스스로 가능한 port의 충동에 대해 잘 생각해야 한다는 것을 의미한다.
  * 또한 NodePort가 사용하도록 configure된 범위안에서 유효한 port 번호를 사용해야 한다.
* NodePort는 Kubernetes가 완전히 지원하지 않거나 하나 또는 그 이상의 노드 IP에 대해 직접적으로 노출하는 환경을 설정해주는 own load balancing solution을 설치하는 것으로부터 자유롭게 해준다.
* Service가 `<NodeIP>:spec.ports[*].nodePort`와 `.spec.clusterIP:spec.ports[*].port`로 보인다는 점을 인지하라.
  * 만약 kube-proxy에서 `--nodeport-addresses` flag가 세팅되어 있으면 NodeIP(s)로 필터링 될 것이다.

### Type LoadBalancer

* external load balancer를 지원하는 cloud provider에서 `type`필드를 `LoadBalancer`로 세팅하여 Service에 대한 load balancer를 준비할 수 있다.

  * 실질적인 load balancer의 생성은 asynchronous로 일어나고 provisioned balancer에 대한 정보는 Service의 `.status.loadBalancer`필드로 배포가 된다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
      - protocol: TCP
        port: 80
        targetPort: 9376
      clusterIP: 10.0.171.239
      loadBalancerIP: 78.11.24.19
      type: LoadBalancer
    status:
      loadBalancer:
        ingress:
        - ip: 146.148.47.155
    ```

* external load balancer로부터 오는 트래픽은 backend Pods로 유도된다.

  * cloud provider는 어떻게 load balanced될지 결정한다.

* 몇몇 cloud provider는 `loadBalancerIP`를 지정할 수 있다.

  * 이 케이스에서 load-balancer는 유저가 지정한 `loadBalancerIP`로 생성할 수 있다.

  * `loadBalancerIP` 필드가 지정되지 않으면 loadBalancer는 ephemeral IP 주소로 설정된다.

  * `loadBalancerIP`를 지정했지만 cloud provider가 이를 지원하지 않으면 설정한 `loadBalancerIP`는 무시된다.

    > Note : SCTP를 사용하고 있지 않다면 `LoadBalancer` Service 타입에 대한 [caveat](https://kubernetes.io/docs/concepts/services-networking/service/#caveat-sctp-loadbalancer-service-type)을 참조하라.

    > Note : Azure에서는 유저가 지정한 public 타입의 `loadBalancerIP`를 사용하고 싶으면 먼저 static type의 IP주소 리소스를 생성해야 한다. 이 public IP 주소 리소스는 반드시 다른 cluster의 자동으로 생성되는 리소스들과 같은 리소스 그룹에 속해야 한다. 예를 들어, `MC_myResourceGroup_myAKSCluster_eastus`. 할당받은 IP 주소를 loadBalancerIP로 지정하라. cloud provider의 configuration file에서 securityGroupName을 업데이트 했는지 확인해라. `CreatingLoadBalancerFailed` permission 이슈에 대한 troubleshooting에 대한 정보는  [Use a static IP address with the Azure Kubernetes Service (AKS) load balancer](https://docs.microsoft.com/en-us/azure/aks/static-ip) 또는 [CreatingLoadBalancerFailed on AKS cluster with advanced networking](https://github.com/Azure/AKS/issues/357)를 참조하라.

### Internal load balancer

* mixed environment에서는 가끔 Service에서 오는 트래픽을 같은 (virtual) 네트워크 주소 블록으로 라우팅할 필요가 있다.
* split-horizon 환경에서는 사용자가 두개의 Service가 external, internal 트래픽 모두가 endpoint로 라우팅 되도록 할 필요가 있다.
* 이는 다음의 annotations를 Service에 추가하여 사용할 수 있다.
  * annotation은 사용자가 쓰는 cloud Service provider에 따라 다르다. ([공식 문서](https://kubernetes.io/docs/concepts/services-networking/service/#discovering-services) 참조)

### TLS support on AWS

* AWS에서 동작하는 클러스터에 대한 TLS / SSL을 사용하려면 `LoadBalancer` Serivce에 세가지 annotations를 추가하면 된다.	

    ```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
    ```

  * 첫번째는 사용할 certificate ARN을 지정한다.

    * IAM에 업로드 된 third party issuer에 대한 certificate 이거나 AWS Certificate Manager에서 생성된 것이면 된다.

  ```yaml
  metadata:
    name: my-service
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
  ```

  * 두번째는 Pod가 speak할 프로토콜을 지정하는 annotation이다.
    * HTTPS와 SSL에서 ELB(Elastic Load Balancing)는 Pod가 certificate을 이용하여 스스로 암호화된 connection에서 인증을 하기를 기대한다.
  * HTTP와 HTTPS는 layer 7 proxying을 사용한다.
    * ELB는 유저와의 connection을 종료하고 haeders를 파싱하고 포워딩이 요청되었을 때 유저의 IP 주소(Pod는 ELB의 IP 주소를 다른 connection의 끝에서만 볼 수 있다.)로`X-Forwarded-For` header를 주입한다.
  * TCP와 SSL은 layer 4 proxying을 사용한다.
    * ELB는 트래픽을 header의 수정 없이 포워딩한다.
  
* 어떤 부분은 보안되고 다른 것은 암호화 되지 않은 mixed-use 환경에서 사용자는 다음의 annotation을 사용할 수 있다.

    ```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
    ```

    * 위의 예시에서 Service가 `80`, `443`, `8443` 이 세개의 port를 가진다면, `443`과 `8443`은 SSL certificate를 사용하지만 `80`은 HTTP로 프록싱 될 것이다.

* Kubernetes v1.9부터 사용자는 Service에 HTTPS나 SSl listener로 [predefined AWS SSL policies](http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html)를 사용할 수 있다.

    * 어떤 정책을 사용할 수 있는지 보려면 `aws` command line tool에서 

        ```yaml
        aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
        ```

        를 사용하면 된다.

    * 그 후에 "**`service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy`**"를 사용하여 하나의 annotation을 지정할 수 있다.

        ```yaml
        metadata:
          name: my-service
          annotations:
            service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
        ```

#### PROXY protocol support on AWS

* AWS에서 작동하고 있는 cluster에서 [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)을 사용하기 위해서는 다음의 service annotation을 사용할 수 있다.

  ```yaml
  metadata:
    name: my-service
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
  ```

* 1.3.0버전부터 annotation을 사용하는 것은 ELB에 의해 proxy 되는 모든 포트에 적용되고 다른 방법으로는 configure 될 수 없다.

#### ELB Access Logs on AWS

* AWS에서 작동하는 ELB Services에 대한 access logs를 관리하는 annotatinos가 있다.

* **`service.beta.kubernetes.io/aws-load-balancer-access-log-enabled`** annotation은 로그의 접속이 가능한지 제어 가능하다.

* **`service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval`**는 access logs를 발행하는 것의 interval를 분단위로 관리한다.

  * interval을 5분에서 60분으로 지정 가능하다.

* **`service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name`** annotation은 load balancer의 access logs가 저장되는 Amazon S3 버켓의 이름을 지정한다.

* **`service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix`**는 Amazon S3 버켓에 생성한 logical hierachy를 지정한다.

  ```yaml
  metadata:
    name: my-service
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
      # Specifies whether access logs are enabled for the load balancer
      service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"
      # The interval for publishing the access logs. You can specify an interval of either 5 or 60 (minutes).
      service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"
      # The name of the Amazon S3 bucket where the access logs are stored
      service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"
      # The logical hierarchy you created for your Amazon S3 bucket, for example `my-bucket-prefix/prod`
  ```

#### Connection Draining on AWS

* Classic ELBs에서 Connection draining은 **`service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled`** annotation을 `"true"`로 설정하여 관리할 수 있다.

  * **`service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout`** annotation은 또한 instances가 deregistering되기 전에 존재하고 있는 connectiosn를 열어두기 위해 초단위 최대 시간(60초)으로 사용될 수 있다.

    ```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"
    ```

#### Other ELB annotations

* Classic Elastic Load Balancers를 관리하는 다른 annotations가 있다.

  ```yaml
  metadata:
    name: my-service
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
      # The time, in seconds, that the connection is allowed to be idle (no data has been sent over the connection) before it is closed by the load balancer
  
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      # Specifies whether cross-zone load balancing is enabled for the load balancer
  
      service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "environment=prod,owner=devops"
      # A comma-separated list of key-value pairs which will be recorded as
      # additional tags in the ELB.
  
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: ""
      # The number of successive successful health checks required for a backend to
      # be considered healthy for traffic. Defaults to 2, must be between 2 and 10
  
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
      # The number of unsuccessful health checks required for a backend to be
      # considered unhealthy for traffic. Defaults to 6, must be between 2 and 10
  
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"
      # The approximate interval, in seconds, between health checks of an
      # individual instance. Defaults to 10, must be between 5 and 300
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
      # The amount of time, in seconds, during which no response means a failed
      # health check. This value must be less than the service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval
      # value. Defaults to 5, must be between 2 and 60
  
      service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f,sg-42efd82e"
      # A list of additional security groups to be added to the ELB
  ```

#### Network Load Balancer support on AWS [alpha]

> Warning : 이 기능은 alpha로 production cluster에서는 사용하지 않길 권장합니다.

* Kubernetes v1.9.0에서부터 AWS Network Load Balancer(NLB)를 Service로 사용이 가능하다.

  * 이 Network Load Balancer를 사용하려면 **`service.beta.kubernetes.io/aws-load-balancer-type`**를 `nlb`로 설정하면 된다.

    ```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    ```

* Classic Elastic Load Blancers와 다르게 Network Load Balancers (NLBs)는 client의 IP 주소를 node로 포워딩 한다.

  * Service의 `.spec.externalTrafficPolicy`가 `Cluster`로 설정이 되어 있으면, client의 IP 주소는 end Pods로 전파되지 않는다.

* `.spec.externalTrafficPolicy`를 `Local`로 설정하여 client IP 주소가 end Pods로 전파할 수 있지만 이는 트래픽의 불균등한 분배를 초래할 수 있다.

  * 특정한 LoadBalancer Service Pods가 없는 노드는 NLB Target Group의  `.spec.healthCehckNodePort`에서 자동할당된 health check에서 실패할 것이다.

* 균일한 트래픽을 얻기 위해서는 DaemonSet을 사용하거나 [pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)를 통해 같은 node에 위치하지 않도록 해야한다.

* 사용자는 NLB Service를 [internal load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer) annotation으로 사용할 수 있다.

* client의 트래픽이 NLB 뒷단의 instance에 도달하기 위해서는 Node security groups가 다음 IP 규칙으로 설정되어야 한다.

  | Rule           | Protocol | Port(s)                                                      | IpRange(s)                                                 | IpRange Description                                |
  | -------------- | -------- | ------------------------------------------------------------ | ---------------------------------------------------------- | -------------------------------------------------- |
  | Health Check   | TCP      | NodePort(s) (`.spec.healthCheckNodePort` for `.spec.externalTrafficPolicy = Local`) | VPC CIDR                                                   | kubernetes.io/rule/nlb/health=\<loadBalancerName\> |
  | Client Traffic | TCP      | NodePort(s)                                                  | `.spec.loadBalancerSourceRanges` (defaults to `0.0.0.0/0`) | kubernetes.io/rule/nlb/client=\<loadBalancerName\> |
  | MTU Discovery  | ICMP     | 3,4                                                          | `.spec.loadBalancerSourceRanges` (defaults to `0.0.0.0/0`) | kubernetes.io/rule/nlb/mtu=\<loadBalancerName\>    |

* Network Load Balancer에 접속할 수 있는 client IP를 제한하려면 `loadBalancerSrouceRanges`를 지정하면 된다.

  ```yaml
  spec:
    loadBalancerSourceRanges:
    - "143.231.0.0/16"
  ```

> Note : `.spec.loadBalancerSourceRanges`가 설정되어 있지 않으면 Kubernetes는 `0.0.0.0/0`에서 들어오는 트래픽을 Node Security Group(s)로 허용한다. 노드가 public IP 주소를 가지고 있으면 non-NLB 트래픽은 이 수정된 security groups 안에 있는 모든 instance에 도달할 수 있음을 주의해라.

### Type ExternalName

* ExternalName 타입의 Service는 Service를 `my-service`나 `cassandra`같은 일반적인 selector가 아닌 DNS 이름으로 매핑한다.

  * 이 Service를 `spec.externalName` 파라미터로 지정할 수 있다.

* 예를 들어 이 Service definition은 `prod` namespace에 있는 `my-service` Service를 `my.database.example.com`으로 매핑한다.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
    namespace: prod
  spec:
    type: ExternalName
    externalName: my.database.example.com
  ```

> Note : ExternalName은 IPv4 주소 string은 사용가능하지만 IP 주소가 아닌 숫자로 구성된 DNS names로 인식이 된다. IPv4 주소와 유사한 ExternalNames는 ExternalName이 canonical DNS name을 지정하려고 의도되었기 때문에 CoreDNS나 ingress-nginx에 의해서 해석되지 않는다. IP 주소를 하드코딩하려면 [headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)를 사용하는 것을 고려해 보아라.

* `my-service.prod.svc.cluster.local` host를 검색할 때 cluster DNS Service는 `my.database.example.com`으로 `CNAME` record를 리턴한다.
  * `my-service`로의 접속은 다른 Service와 같은 방식으로 작동하지만 proxying이나 forwading을 통한 것이 아니라 DNS 단계에서 redirection이 일어나는 것이 차이점이다.
  * 나중에 database를 cluster 안으로 옮기자고 결정한 경우 Pods에서 시작하여 적절한 selectors와 endpoints를 추가하고 Service의 `type`을 변경하여 할 수 있다.

> Note : 이 section은  [Alen Komljen](https://akomljen.com/)가 작성한 [Kubernetes Tips - Part 1](https://akomljen.com/kubernetes-tips-part-1/)에게 감사함을 표한다.

### External IPs

* 하나 이상의 cluster nodes에 라우팅하는 external IPs가 있다면 Kubernetes Service는 이 `externalIPs`를 노출할 수 있다.

  * Service port에서 external IP(destination IP)를 통해 cluster로 ingresses하는 트래픽은 Service endpoints중 하나로 라우팅 될 것이다.
  * `externalIPs`는 Kubernetes에 의해 관리되지 않고 cluster administrator에게 책임이 있다.

* Service spec에서는 `externalIPs`는 아무 `ServiceTypes`를 통해 지정될 수 있다.

  * 아래 예시에서 "`my-service`"는 "`80.11.12.10:80`"(`externalIP:port)의 client에 의해서 접속될 수 있다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
      - name: http
        protocol: TCP
        port: 80
        targetPort: 9376
      externalIPs:
      - 80.11.12.10
    ```

## Shortcomings

* VIPs를 이용하는 userspace proxy 사용할 때 작은 범위에서 중간 범위까지는 동작하지만 수천개의 Services를 가지는 매우 큰 cluster로는 스케일 되지 않는다.
  * [original design proposal for portals](http://issue.k8s.io/1107)에 더 자세히 기술되어 있다.
* userspace proxy를 사용하는 것은 Service에 접속하는 패킷의 source IP 주소를 알 수 없게 한다.
  * 이는 network filtering (firewalling)을 불가능하게 할 수 있다.
  * iptables proxy mode는 in-cluster souce IPs를 알수 없게 하지 않지만 여전히 client가 load balancer나 node-port를 통해 들어오는 데 영향을 준다.
* `Type` 필드는 중첩된 기능으로 구현되어 있다.
  * 각각의 level은 이전의 level에 추가되어 있다.
  * 이는 모든 cloud providers에 대해 필수 요구사항이 아니지만(예를 들어 Google Compute Engine은 `NodePort`를 할당하여 `LoadBalancer`가 동작하도록 할 필요는 없지만 AWS는 할당해야 한다.) 현재의 API는 이를 요구한다.

## Virtual IP implementation

* 이전의 정보는 Services를 사용하고 싶어하는 많은 사람들에게 충분할 것이다.
  * 하지만 이해하면 좋은 더 많은 뒷 이야기들이 있다.

### Avoiding collisions

* Kubernetes의 주요 철학중 하나는 사용자의 실수가 아닌 상황때문에 fail이 나지 않도록 하는 것이다.
  * Service resource의 디자인에서 이는 다른 사람이 선택한 port number와 사용자의 port number가 충돌하는 것을 선택하도록 하지 않는다.
* Service에 대한 port number를 사용자가 선택할 수 있도록 하기 위해 두 Service가 충돌하지 않는 것을 확인해야 한다.
  * Kubernetes는 각 Service에 고유의 IP 주소를 할당하여 구현한다.
* 각 Service가 유일한 IP를 가지는 것을 확실히 하기 위해 각 Service를 생성하기 전에 internal allocator가 atomic하게 etcd에 있는 global allocation map을 업데이트 한다.
  * 이 map 오브젝트는 IP 주소를 할당받기 위해 Service에 대해 registry에 존재해야 하고 그렇지 않으면 IP 주소를 할당받을 수 없다는 메시지를 주며 실패한다.
* control plane에서는 background controller가 그 map을 생성하는 데 책임이 있다.(in-memory locking을 사용하는 이전의 Kubernetes 버전을 사용하는 것으로부터 migrating이 지원되어야 한다.)
  * Kubernetes는 똫나 controller가 유효하지 않은 할당을 하는지 체크해야 하고 할당되었지만 어떤 Service에서도 사용하지 않는 IP 주소를 cleaning up 해야한다.

### Service IP addresses

* 실제로 고정된 destination으로 라우팅 되는 Pod의 IP 주소와는 달리 Service IPs는 실제로 single host에 의해 응답되지 않는다.
  * 대신 kube-proxy는 iptables(Linux에서의 packet processing logic)를 이용하여 필요시 투명하게 redirect를 하는 virtual IP 주소를 정의한다.
  * clients가 VIP로 연결을 할 때 트래픽은 자동으로 적절한 endpoint로 전송이 된다.
  * Service에 대한 DNS의 환경 변수는 실제로 Service의 virtual IP address(와 port)로 제공된다.
* kube-proxy는 약간씩 다르게 동작하는 세가지 proxy modes -userspace, iptables, IPVS- 를 제공한다.

#### Userspace

* 예시로 위에 설명한 이미지 처리 application을 들어보자
  * backend Service가 생성이 되면 Kubernetes master는 10.0.0.1같은 virtual IP 주소를 할당한다.
  * Service의 port가 1234라고 하면 Service는 cluster 안의 모든 kube-proxy instances에 의해 보여진다.
  * proxy가 새로운 Service를 발견하면 새로운 random port를 열고 virtual IP 주소에서 이 포트로 redirect하는 iptables를 생성하고 여기로의 connection을 받기 시작한다.
* clients가 Service의 virtual IP 주소로 연결하려고 하면 iptables rules가 발현되어 패킷을 proxy 고유의 port로 redirect한다. 
  * "Service porxy"는 backend를 선택하고 트래픽을 client에서 backend로 proxying 하기 시작한다.
* 이는 Service의 owner가 collision의 위험 없이 어떤 port도 선택할 수 있음을 의미한다.
  * Clients는 어떤 파드에 실제로 접속하는지 알 필요 없이 간단히 IP와 port를 연결할 수 있다.

#### iptables

* 다시 위에 설명한 이미지 처리 application을 예시로 들어보자.
  * backend Service가 생성이 되면, Kubernetes control plane은 10.0.0.1같은 virtual IP 주소를 할당한다.
  * Service port가 1234라고 하면 Service는 cluster의 모든 kube-proxy instances에 의해 보일 것이다.
  * proxy가 새로운 Service를 발견하면 virtual IP 주소를 per-Service rules로 redirect하는 series of iptables를 설치한다.
  * per-Service rules는 (destination NAT를 사용하는)트래픽을 backend로 redirect하는 per-Endpoint rules에 연결된다.
* client가 Service의 virtual IP 주소로 연결하려 할 때 iptables가 발현이 된다.
  * backend는 (session affinity나 임의로)선택되고 패킷은 backend로 redirect된다.
  * userspace proxy와는 다르게 패킷은 userspace에 절대 복사되지 않고 kube-proxy는 동작하기 위해 virtual IP 주소를 사용할 필요가 없고 노드는 바뀌지 않은 client IP 주소로부터 도착한 트래픽을 볼 수 있다.
* 이 같은 basic flow는 이런 케이스에서 client IP가 바뀐다고 하더라도 트래픽이 node-port나 load-balancer를 통해 들어올 때 작동을 한다.

#### IPVS

* iptables 동작은 예를 들어 10,000개의 Services가 있는 큰 스케일의 cluster에서 급격하게 느려진다.
  * IPVS는 load balancing을 위해 디자인 되었고 in-kernel hash tables에 기반한다.
  * 따라서 IPVS-based kube-proxy에서 많은 수의 Services를 사용해도 일관적인 퍼포먼스를 얻을 수 있다.
  * IPVS-based kube-proxy는 더 복잡한 load balancing algorithms을 사용한다. (least conns, locality, weighted, persistence)

## API Obejct

* Service는 Kubernetes REST API에서 tol-level resource이다.
  * API 오브젝트에 관한 더 자세한 사항은 [Service API object](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#service-v1-core)에서 확인 가능하다.

## Supported protocols

### TCP

**FEATURE STATE: `kubernetes v1.0`** - stable

* 모든 종류의 Service에 대해서 TCP를 사용할 수 있다.
* default network protocol이다.

### UDP

**FEATURE STATE: `Kubernetes v1.0`** - stable

* 대부분의 Service에서 UPD를 사용할 수 있다.
* type=LoadBalancer인 Services에서 UDP 지원은 cloud provider가 제공하는 기능에 달려있다.

### HTTP

**FEATURE STATE: `Kubernetes v1.1`** - stable

* cloud privoder가 이를 지원한다면 Service를 LoadBalancer mode로 사용하여 external HTTP / HTTPS reverse proxing을 설정하고 Service의 Endpoints로 포워딩 할 수 있다.

> Note : Service에서 Ingress를 사용하여 HTTP / HTTPS Services를 노출할 수도 있다.

### PROXY protocol

**FEATURE STATE: `Kubernetes v1.1`** - stable

* cloud provider가 이를 지원한다면 (AWS처럼), 사용자는 LoadBalancer mode의 Service를 사용하여 Kubernetes 밖에 있는 load balancer를 configure할 수 있고 [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)의 prefix를 가지는 connection을 포워딩할 것이다.
* load balancer는 client로부터 오는 데이터 뒤에 오는 `PROXY TCP4 192.0.2.202 10.0.42.7 12345 7\r\n`같은 incoming connection을 설명하는 octets의 initial series를 보낼 것이다. 

### SCTP

**FEATURE STATE: `Kubernetes v1.12`** - alpha

* Kubernetes는 alpha 기능으로 SCTP를 Service, Endpoint, NetworkPolicy, Pod deficitions에서 `protocol` 값으로 지원한다.
  * 이 기능을 사용하려면 cluster administrator는 `SCTPSupport` apiserver의 feature gate에 `--feature-gates=SCTPSupport=true, ...`처럼 enable해야 한다.
* feature gate가 enable되면 Service, Endpoint, NetworkPolicy나 파드의 `protocol` 필드를 `SCTP`로 설정할 수 있다.
  * Kubernetes는 TCP connections처럼 SCTP associations에 따라 network를 세팅한다.

#### Warnings

##### Support for multihomed SCTP associations

> **Warning** : 
>
> multihomed SCTP associations의 지원은 CNI plugin이 multiple interfaces와 IP 주소를 파드에 할당하는 것을 지원할 것을 필요로한다.
>
> multihomed SCTP associations에 대한 NAT은 이에 상응하는 kernel modules의 특별한 로직을 필요로 한다.

##### Service with type=LoadBalancer

> **Warning** : 
>
> LoadBalancer `type`에 cloud provider가 load balancer implemetation에서 SCTP를 프로토콜로 지원하는 경우에 `protocol`을 SCTP를 사용하는 것만 Service로만 생성할 수 있다. 현재의 cloud load balancer providers(Azure, AWS, CloudStack, GCE, OpenStack)는 모두 SCTP에 대한 지원이 부족하다.

##### Windows

> **Warning** :
>
> SCTP는 Windows 기반의 노드에서는 지원되지 않는다.

##### Userspace kube-proxy

> Warning:
>
> kube-proxy는 userspace mode에서 SCTP associations의 management를 지원하지 않는다.

## Future work

* 미래에는 Services에 대한 proxy policy가 master-elected나 sharded처럼 round-robin balancing보다 더 미묘하게(nuanced) 될 것이다.
  * virtual IP 주소가 간단하게 그곳으로 패킷이 전송되는 경우에서 Services가 "real" load balancers를 가지도록 계획중이다.
* Kubernetes 프로젝트는 L7(HTTP) Services에 대한 지원을 개선하도록 의도되었다.
* Kubernetes 프로젝트는 현재의 ClusterIP, NodePort, LoadBalancer modes와 더 많은 것들을 포괄하는 Services에 대한 더 많은 ingress modes를 가지게 될 것이다.

