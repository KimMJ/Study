# Service

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
* CoreDNS같은 cluster-aware DNS server는 new Services와 각 DNS records의 생성을 Kubernetes API를 관찰하여