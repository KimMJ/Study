# DNS for Services and Pods

## Introduction

* Kubernetes DNS는 cluster에서 DNS Pod와 Service를 스케쥴하고 kubelets를 설정하여 각각의 containers가 DNS Service의 IP로 DNS names를 해석하도록 한다.

### What things get DNS names?

* cluster 안에서 정의된 모든 Service(DNS server 그 자체도)는 DNS name을 할당받는다.
  * default로 client Pod의 DNS search list는 Pod 자신의 namespace와 cluster의 defulat domain을 포함할 것이다.
  * 이는 다음 예시에서 잘 설명되어 있다.
    * Kubernetes namespace `bar`안에 `foo`라는 서비스가 있다고 가정해보자.
    * `bar` namespace에서 동작하는 파드는 이 서비스를 `foo`에 대한 DNS query를 보내서 간단하게 검색할 수 있다.
    * `quux` namespace에서 동작하는 파드는 `foo.bar`에 대한 DNS query를 보내서 service를 검색할 수 있다.
* 아래 sections는 지원하는 record type과 layout에 대한 자세한 사항들이다.
  * 다른 동작하는 layout이나 name, queries는 자세한 구현에 대해 고려해 보아야 하고 경고없이 변경될 수 있다.
  * 최신버전의 specification은 [Kubernetes DNS-Based Service Discovery](https://github.com/kubernetes/dns/blob/master/docs/specification.md)을 참조하라.

## Services

### A records

* "Normal" (headless가 아닌) Service는 `my-svc.my-namespace.svc.cluster-domain.example`형식에 따른 이름에 대한 DNS A record를 할당받는다.
  * 이는 Service의 cluster IP로 풀이된다.
* "Headless" (cluster IP가 없는) Services는 또한 `my-svc.my-namespace.svc.cluster-domain.example`형식에 따른 이름에 대한 DNS A record를 할당받는다.
  * 일반적인 Service와는 다르게 Service에 의해 선택된 파드들의 IP 묶음으로 풀이된다.
  * Client는 그 묶음에서 consume하거나 표준 round-robin selection을 사용하도록 예상된다.

### SRV records

* SRV Records는 일반적인 또는 [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)의 일부분인 named ports로 생성된다.
  * 각각의 named port에 대해 SRV record는 `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`의 형식을 가진다.
  * regular Service에 대해서 이는 port number와 domain name으로 풀이된다 : `my-svc.my-namespace.svc.cluster-domain.example`
  * headless Service에서 이는 여러개의 답으로 풀이된다.
    * Service를 지원하는 각각의 파드들
    * 파드의 `auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example`형식에 해당하는 port number와 domain name을 포함하는 것

## Pods

### Pod's hostname and subdomain fields

* 현재 pod가 생성이 될 때 그 hostname은 Pod의 `metadata.name` 값이다.

* 파드의 spec은 `hostname`이라는 Pod의 hostname을 지정할 때 사용할 수 있는 optional 필드를 가지고 있다.

  * 지정을 하면 Pod의 name이 pod의 hostname이 되는 것이 중요시된다.
  * 예를 들어, 주어진 Pod의 `hostname`이 "`my-host`"로 설정이 되었으면 Pod는 hostname을 "`my-host`"로 설정할 것이다.

* Pod spec은 또한 subdomain을 지정할 때 사용하는 `subdomain` optional 필드를 가지고 있다.

  * 예를 들어, "`my-namespace`" namespace 안에서 Pod의 `hostname`이 "`foo`"로 설정되어 있고, `subdomain`이 "`bar`"로 설정이 되어있다면 `foo.bar.my-namespace.svc.cluster-domain.example`인 fully qualified domain name(FQDN)을 가지게 된다.

* 예시

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: default-subdomain
  spec:
    selector:
      name: busybox
    clusterIP: None
    ports:
    - name: foo # Actually, no port is needed.
      port: 1234
      targetPort: 1234
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox1
    labels:
      name: busybox
  spec:
    hostname: busybox-1
    subdomain: default-subdomain
    containers:
    - image: busybox:1.28
      command:
        - sleep
        - "3600"
      name: busybox
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox2
    labels:
      name: busybox
  spec:
    hostname: busybox-2
    subdomain: default-subdomain
    containers:
    - image: busybox:1.28
      command:
        - sleep
        - "3600"
      name: busybox
  ```

* 같은 namespace안에 headless service가 파드의 namespace가 같고 subdomain의 name이 같으면, cluster의 KubeDNS Server가 Pod의 fully qualified hostname에 대한 A record를 리턴한다.

  * 예를 들어 "`busybox-1`"이라는 hostname을 가지고 subdomain이 "`default-subdomain`"으로 설정된 파드가 주어졌을 때 같은 namespace 안에 headless Service가 "`default-subdomain`"으로 이름지어져 있으면 파드는 스스로의 FQDN을 "`busybox-1.default-subdomin.my-namespace.svc.cluster-domain.example`"을 사용하면 
  * DNS는 그 이름에 대한  Pod의 IP를 지정하는 A record를 제공한다.
  * "`busybox1`"와 "`busybox2`"는 그것들만의 구별가능한 A records를 가지고 있다.

* Endpoints 오브젝트는 어떤 endpoint 주소든 IP를 따라 `hostname`을 지정할 수 있다.

> Note : A records가 파드의 이름에 대해 생성되지 않기 때문에 `hostname`은 Pod의 A record가 생성되기 위해 필요하다. `hostname`이 없지만 `subdomain`이 있는 파드는 Pod의 IP 주소를 가르키는 headless service(default-subdomain.my-namespace.svc.cluster-domain.example)에 대한 A record만 생성한다. Pod는 `publishNotReadyAddresses=True`가 서비스에서 세팅이 되었더라도 record를 가지기 위해서 준비가 되어야 한다.

### Pod's DNS Policy

* DNS policies는 각 pod 기반으로 세팅이 된다.
  * 현재의 Kubernetes는 다음의 pod-specific DNS policies를 지원한다.
  * 이 policies는 Pod Spec의 `dnsPolicy` 필드에 지정된다.
    * "`Default`"
      * 파드는 그 파드가 동작하는 노드에서 name resolution configuration을 상속받는다.
      * 자세한 사항은 [related discussion](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#inheriting-dns-from-the-node)을 보라.
    * "`ClusterFirst`"
      * "`www.kubernetes.io`"같은 cluster domain suffix에 일치하지 않 DNS query는 노드로부터 상속된 upstream nameserver로 전달된다.
      * Cluster administrators는 여분의 stub-domain과 설정된 upstream DNS servers를 가질 수 있다.
      * 어떻게 DNS queries가 이 케이스에서 동작하는지 보려면 [related discussion](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#impacts-on-pods)를 참조하라.
    * "`ClusterFirstWithHostNet`"
      * hostNetwork에서 동작하는 파드에 대해서 반드시 명확하게 DNS policy를 "`ClusterFirstWithHostNet`"으로 설정해야 한다.
    * "`None`"
      * 파두가 Kubernetes 환경으로부터 DNS 세팅을 무시하도록 한다.
      * 모든 DNS 세팅은 Pod Spec안의 `dnsConfig` 필드를 사용하여 제공되는 것으로 가정된다.
      * 아래 [Pod’s DNS config](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config)을 보아라.
    
    > Note : "Default"는 default DNS policy가 아니다. `dnsPolicy`가 명확하게 지정되어 있지 않으면 "ClusterFirst"가 사용된다.
  
* 아래의 예시는 `hostNetwork`가 `true`로 설정되었기 때문에 DNS policy가 "`ClusterFirstWithHostNet`"으로 세팅이 된 파드를 보여준다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox
    namespace: default
  spec:
    containers:
    - image: busybox:1.28
      command:
        - sleep
        - "3600"
      imagePullPolicy: IfNotPresent
      name: busybox
    restartPolicy: Always
    hostNetwork: true
    dnsPolicy: ClusterFirstWithHostNet
  ```

### Pod's DNS Config

* Pod의 DNS Config는 사용자가 파드에 대한 DNS 세팅을 더 조절할 수 있도록 한다.

* `dnsConfig`필드는 optional이고 어떤 `dnsPolicy` 세팅과도 동작한다.

  * 하지만 Pod의 `dnsPolicy`가 "`None`"으로 설정되어 있으면 `dnsConfig` 필드는 반드시 지정되어야 한다.

* 아래의 속성들은 사용자가 `dnsConfig` 필드로 지정할 수 있는 것들이다.

  * `nameservers`
    * 파드에 대한 DNS servers로 사용될 IP 주소 리스트이다.
    * 최대 3개의 IP 주소가 지정될 수 있다.
    * 파드의 `dnsPolicy`가 "`None`"으로 설정되어있으면 최소 하나의 IP 주소를 포함해야 하고 그렇지 않으면 이 속성은 optional이다.
    * list 안에 있는 server들은 중복되는 주소가 지워진 지정된 DNS policy에서 생성하는 base nameservers와 합쳐진다.
  * `searches`
    * 파드에서 검색되는 hostname에 대한 DNS search domains
    * 이 속성은 optional이다.
    * 지정되었을 때 주어진 list는 선택된 DNS policy에서 생성된 base search domain names와 병합된다.
    * 중복되는 domain names는 삭제된다.
    * Kubernetes는 최대 6개의 search domains를 지원한다.
  * `options`
    * 각 오브젝트가 `name` 속성(필수)과 `value` 속성(선택)을 가질 수 있는 오브젝트들의 optional list.
    * 이 속성의 내용은 지정된 DNS policy에서 생성된 options와 병합될 것이다.
    * 중복되는 entry는 제거된다.

* 다음은 custom DNS 세팅을 하는 파드의 예시이다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    namespace: default
    name: dns-example
  spec:
    containers:
      - name: test
        image: nginx
    dnsPolicy: "None"
    dnsConfig:
      nameservers:
        - 1.2.3.4
      searches:
        - ns1.svc.cluster-domain.example
        - my.dns.search.suffix
      options:
        - name: ndots
          value: "2"
        - name: edns0
  ```

  * 위의 파드가 생성될 때 `test` 컨테이너는 다음 내용을 /etc/resolv.conf 파일에 얻는다.

    ```
    nameserver 1.2.3.4
    search ns1.svc.cluster-domain.example my.dns.search.suffix
    options ndots:2 edns0
    ```

* IPv6 셋업에서 search path와 name server는 다음과 같이 설정되어야 한다.

  ```bash
  kubectl exec -it dns-example -- cat /etc/resolv.conf
  ```

  * 결과는 다음과 같다.

    ```shell
    nameserver fd00:79:30::a
    search default.svc.cluster-domain.example svc.cluster-domain.example cluster-domain.example
    options ndots:5
    ```

### Feature availability

* 파드의 DNS Config avaiability는 아래 나와있다.

  | k8s version | Feature support      |
  | :---------- | :------------------- |
  | 1.14        | Stable               |
  | 1.10        | Beta (on by default) |
  | 1.9         | Alpha                |

















































