# Ingress

**FEATURE STATE: **`Kubernetes v1.1` - beta

* HTTP같은 cluster 내부 service로의외부 접속을 관리하는 API 오브젝트
* Ingress는 load balancing, SSL termination과 name-based virtual hosting을 제공해준다.

## Terminology

* 명확하게 하기 위해서 이 가이드는 다음의 용어를 정의한다.
* Node
  * cluster의 부분으로써 Kubernetes의 worker machine
* Cluster
  * Kubernetes에 의해 관리되는 containerized applications를 실행시키는 Nodes의 묶음.
  * 이 예시에서, 그리고 보통의 Kubernetes deployments와 cluster의 nodes에서 public internet의 일부분이 아니다.
* Edge router
  * cluster에 대한 firewall policy를 시행하는 router.
  * cloud provider에 의해 관리되는 gateway이거나 hardware의 물리적 부분이 될 수 있다.
* Cluster network
  * 논리적 또는 물리적 links의 묶음으로 Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)을 따라 cluster 내부에서 communication을 가능하게 하는 것
* Service
  * Kubernetes Service는 label selectors를 이용하여 파드의 묶음을 확인한다.
  * 따로 언급되지 않으면 Service는 cluster network안에서만 routable한 virtual IPs를 가진다고 가정된다.

## What is Ingress?

* Ingress는 cluster 외부에서 cluster 내부의 services로의 HTTP와 HTTPS routes를 expose한다.

  * Traffic routing은 Ingress resource에 정의된 rules에 의해 관리된다.

  ```none
      internet
          |
     [ Ingress ]
     --|-----|--
     [ Services ]
  ```

* Ingress는 Services에게 externally-reachable URLs, load balance traffic, terminate SSL / TLS를 주도록 configured 되어있고 name 기반 virtual hosting을 제공한다.

  * [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)은 이것이 edge router나 추가적인 frontends가 트래픽을 관리하는 데 도움을 주도록 configure되어있긴 하지만 보통 load balancer를 통해 Ingress를 채우는데 책임이 있다.

* Ingress는 임의의 ports나 protocols를 expose하지 않는다.

  * 인터넷으로의 HTTP나 HTTPS가 아닌 Exposing services는 전형적으로  [Service.Type=NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)나 [Service.Type=LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)같은 타입의 service를 사용한다.

## Prerequisites

* Ingress를 만족시키기 위해서 반드시 [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)를 가져야 한다.
  * Ingress resource를 생성하기만 하는 것은 효과가 없다.
* [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)같은 Ingress controller를 배포해야한다.
  * 여러 [Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)중에 선택할 수 있다.
* 이상적으로 모든 Ingress controllers는 reference specification을 만족해야 한다.
  * 사실상 다양한 Ingress controllers가 약간 다르게 동작한다.

> Note : 사용할 Ingress controller의 documentation을 보고 선택했을 때의 주의할 점들을 이해해라.

## The Ingress Resource

* 경량 Ingress resource 예시

  ```yaml
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: test-ingress
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - http:
        paths:
        - path: /testpath
          backend:
            serviceName: test
            servicePort: 80
  ```

* 다른 Kubernetes resources처럼 Ingress는 `apiVersion`, `kind`, `metadata` 필드를 필요로 한다.

  * config files의 동작에 대한 일반적인 정보는 [deploying applications](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/), [configuring containers](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/), [managing resources](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)를 확인해라.
  * Ingress는 [rewrite-target annotation](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/rewrite/README.md)의 예시처럼 Ingress controller에 따라 몇몇 옵션을 configure하는 데 annotations를 자주 사용한다.
  * 다른 Ingress controller는 다른 annotations를 지원한다.
  * Ingress controller로 선택한 documentation을 검토하여 어떤 annotations가 지원되는지 배워라.

* Ingress [spec](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)은 load balancer나 proxy server를 configure하는 모든 정보를 가진다.

  * 가장 중요한 것은 모든 incoming requests에 대해 매칭되는 rules의 리스트를 가지고 있다는 것이다.
  * Ingress resource는 직접적인 HTTP 트래픽에 대한 rule만 지원한다.

### Ingress rules

* 각각의 HTTP rule은 다음의 정보를 가지고 있다.
  * optional host. 
    * 여기 예시에서 host는 지정되지 않았으며 따라서 지정된 IP 주소를 따라 모든 inboud HTTP 트래픽에 rule이 적용된다.
    * 만약 host가 (예를 들어 foo.bar.com)를 제공받으면 그 host에 rules가 적용이 된다.
  * paths(예를 들어 `/testpath`)의 리스트로서 `serviceName`과 `servicePort`로 정의된 관련 backend를 가진 것.
    * 모든 host와 path는 load balancer가 트래픽을 참조된 service로 direct 하기 전에 반드시 incoming request의 content와 일치해야 한다.
  * [Service doc](https://kubernetes.io/docs/concepts/services-networking/service/)에 설명되어 있듯이 backend는 Service와 port names의 결합이다.
    * host와 rule의 path가 일치하는 Ingress로의 HTTP(그리고 HTTPS) requests는 listed backend로 전송된다.
* default backend는 spec에서의 path가 일치하지 않는 어떤 request도 서비스 하기 위해 Ingress controller에서 configured된다.

### Default Backend

* rule이 없는 Ingress는 모든 트래픽을 single default backend로 전송한다.
  * default backend는 전형적으로 [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)의 configuration option이고 Ingress resources에서 지정되지 않는다.
* Ingress 오브젝트에서 HTTP request가 어떤 posts나 paths에도 일치하지 않으면 트래픽은 default backend로 route된다.

## Type of Ingress

### Single Service Ingress

* single Service ([alternatives](https://kubernetes.io/docs/concepts/services-networking/ingress/#alternatives)를 보아라)를 expose하기 위한 Kubernetes concepts가 있다.

  * 이를 default backend를 rule이 없이 지정해주어 Ingress 똑같이 할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: test-ingress
  spec:
    backend:
      serviceName: testsvc
      servicePort: 80
  ```

* `kubectl apply -f`를 이용해서 이를 생성하면 방금 추가한 Ingress의 state를 확인할 수 있다.

  ```shell
  kubectl get ingress test-ingress
  ```

  ```
  NAME           HOSTS     ADDRESS           PORTS     AGE
  test-ingress   *         107.178.254.228   80        59s
  ```

* `107.178.254.228`이 이 Ingress를 만족시키는 Ingress controller에 의해 할당된 IP이다.

> Note : Ingress controllers와 laod balancers는 IP 주소를 할당하는데 1~2분이 걸릴 수 있다. 그 시간동안 주소가 `<pending.`으로 리스트 된 것을 볼 수 있다.

### Simple fanout

* fanout configuration은 요청된 HTTP URI를 기반으로 single IP 주소에서 들어온 트래픽을 하나 이상의 Service로 route한다.

  * Ingress는 load balancer의 수를 최소한으로 유지하도록 할 것이다.

  * 예를 들어 이렇게 세팅하면

    ```none
    foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                     / bar    service2:8080
    ```

    이런 Ingress가 필요하다

    ```yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: simple-fanout-example
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: foo.bar.com
        http:
          paths:
          - path: /foo
            backend:
              serviceName: service1
              servicePort: 4200
          - path: /bar
            backend:
              serviceName: service2
              servicePort: 8080
    ```

  * Ingress를 `kubectl apply -f`로 생성하면

    ```shell
    kubectl describe ingress simple-fanout-example
    ```

    ```
    Name:             simple-fanout-example
    Namespace:        default
    Address:          178.91.123.132
    Default backend:  default-http-backend:80 (10.8.2.3:8080)
    Rules:
      Host         Path  Backends
      ----         ----  --------
      foo.bar.com
                   /foo   service1:4200 (10.8.0.90:4200)
                   /bar   service2:8080 (10.8.0.91:8080)
    Annotations:
      nginx.ingress.kubernetes.io/rewrite-target:  /
    Events:
      Type     Reason  Age                From                     Message
      ----     ------  ----               ----                     -------
      Normal   ADD     22s                loadbalancer-controller  default/test
    ```

  * Ingress controller는 Services(`s1`, `s2`)가 존재하는 한 Ingress를 만족시키는 implementation-specific load balancer를 제공한다.

    * 이렇게 하면 주소 필드에서 load balancer의 주소를 볼 수 있다.

> Note : 사용하는 Ingress controller에 따라 default-http-backend Service를 생성해야 할수도 있다.

### Name based virtual hosting

* Name-based virtual hosts는 HTTP 트래픽을 같은 IP 주소를 가진 여러 host names로 라우팅하는 것을 지원한다.

  ```none
  foo.bar.com --|                 |-> foo.bar.com s1:80
                | 178.91.123.132  |
  bar.foo.com --|                 |-> bar.foo.com s2:80
  ```

* 다음의 Ingress는 뒷단의 load balancer가 Host header를 기반으로 request를 route하도록 한다.

  ```yaml
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: name-virtual-host-ingress
  spec:
    rules:
    - host: foo.bar.com
      http:
        paths:
        - backend:
            serviceName: service1
            servicePort: 80
    - host: bar.foo.com
      http:
        paths:
        - backend:
            serviceName: service2
            servicePort: 80
  ```

* rule에 정의된 어떤 hosts도 없이 Ingress resource를 생성하면 Ingress controller의 IP 주소로의 어떤 웹 트래픽도 요구되는 virtual host를 기반으로 한 이름 없이 매칭될 수 있다.

* 예를 들어 다음 Ingress resource는 `first.bar.com` 트래픽을 `service1`으로 라우팅할 것이고 `second.foo.com`을 `service1`로 라우팅 할 것이다. 그리고 request에 정의된 hostname이 없는 모든 IP 주소에 대한 트래픽(즉, 존재하는 request header가 없는 것)은 `service3`로 라우팅 된다.

  ```yaml
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: name-virtual-host-ingress
  spec:
    rules:
    - host: first.bar.com
      http:
        paths:
        - backend:
            serviceName: service1
            servicePort: 80
    - host: second.foo.com
      http:
        paths:
        - backend:
            serviceName: service2
            servicePort: 80
    - http:
        paths:
        - backend:
            serviceName: service3
            servicePort: 80
  ```

### TLS

* Ingress를 TLS private key와 certificate를 포함한 Secret을 지정하여 보안을 적용할 수 있다.

  * 현재 Ingress는 하나의 TLS port 443만 지원하고 TLS termination을 가정한다.

  * Ingress안의 TLS configuration section에서 다른 hosts를 지정하면 SNI TLS extension(Ingress controller가 SNI를 지원하도록 하는 것)을 통해 지정된 hostname을 따라 같은 포트로 multiplexed될 것이다.

  * TLS secret은 반드시 TLS에 사용할 certificate와 private key를 포함한 `tls.crt`와 `tls.key`의 이름을 가진 키를 포함해야 한다.

  * 예시

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: testsecret-tls
      namespace: default
    data:
      tls.crt: base64 encoded cert
      tls.key: base64 encoded key
    type: kubernetes.io/tls
    ```

  * Ingress에서 이 secret을 참조하는 것은 Ingress controller에게 TLS를 이용하여 client에서 load balancer로 가는 채널의 보안을 적용하라고 한다.

    * 생성한 TLS secret이 `sslexample.foo.com`에 대한  CN을 포함하는 certificate로부터 생성되었음을 확인해야 한다.

    ```yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: tls-example-ingress
    spec:
      tls:
      - hosts:
        - sslexample.foo.com
        secretName: testsecret-tls
      rules:
        - host: sslexample.foo.com
          http:
            paths:
            - path: /
              backend:
                serviceName: service1
                servicePort: 80
    ```

> Note : 다양한 Ingress controllers에 의해 지원되는 TLS features 사이에 gap이 있다. [nginx](https://git.k8s.io/ingress-nginx/README.md#https), [GCE](https://git.k8s.io/ingress-gce/README.md#frontend-https), 혹은 다른 Ingress controller를 지정하는 플랫폼의 문서를 검토하여 어떻게 TLS가 환경에서 동작하는지 이해해라.

### Loadbalancing

* Ingress controller는 load balancing algorithm, backend weight scheme, 등과 같은 모든 Ingress에 적용되는 몇몇 load balancing policy setting으로 bootstrapped된다.
  * 더 자세한 load balancing concepts(e.g. persistent sessions, dynamic weights)는 아직 Ingress를 통해 expose 되지 않았다.
  * Service에 대한 load balancer를 통해서 이러한 feature를 대신 얻을 수 있다.
* heal check가 Ingress를 통해서 직접적으로 expose되지 않았다고 하더라도 같은 end result를 얻을 수 있는 readiness probes처럼 Kubernetes 내부에 동일한 컨셉이 있음을 알아두어라.
  * nginx, GCE같은 것들이 어떻게 health checks를 다루는지 controller specific documentation을 검토해 보아라.

## Updating an Ingress

* resource를 업데이트하면 새로운 Host에 이미 있는 Ingress를 업데이트 할 수 있다.

  ```shell
  kubectl describe ingress test
  ```

  ```
  Name:             test
  Namespace:        default
  Address:          178.91.123.132
  Default backend:  default-http-backend:80 (10.8.2.3:8080)
  Rules:
    Host         Path  Backends
    ----         ----  --------
    foo.bar.com
                 /foo   s1:80 (10.8.0.90:80)
  Annotations:
    nginx.ingress.kubernetes.io/rewrite-target:  /
  Events:
    Type     Reason  Age                From                     Message
    ----     ------  ----               ----                     -------
    Normal   ADD     35s                loadbalancer-controller  default/test
  ```

  ```shell
  kubectl edit ingress test
  ```

* YAML 형식의 configuration을 에디트 할 수있다. 새로운 Host를 포함하도록 수정해보자.

  ```yaml
  spec:
    rules:
    - host: foo.bar.com
      http:
        paths:
        - backend:
            serviceName: s1
            servicePort: 80
          path: /foo
    - host: bar.baz.com
      http:
        paths:
        - backend:
            serviceName: s2
            servicePort: 80
          path: /foo
  ..
  ```

* 변경사항을 저장하고 난 후 kubectl은 Ingress controller에게 load balancer를 reconfigure하라고 지시하는 API server의 리소스를 업데이트한다.

* 확인

  ```shell
  kubectl describe ingress test
  ```

  ```
  Name:             test
  Namespace:        default
  Address:          178.91.123.132
  Default backend:  default-http-backend:80 (10.8.2.3:8080)
  Rules:
    Host         Path  Backends
    ----         ----  --------
    foo.bar.com
                 /foo   s1:80 (10.8.0.90:80)
    bar.baz.com
                 /foo   s2:80 (10.8.0.91:80)
  Annotations:
    nginx.ingress.kubernetes.io/rewrite-target:  /
  Events:
    Type     Reason  Age                From                     Message
    ----     ------  ----               ----                     -------
    Normal   ADD     45s                loadbalancer-controller  default/test
  ```

* 같은 결과를 수정된 Ingress YAML파일을 가지고 `kubectl replace -f`를 호출하여 얻을 수 있다.

## Failing across availability zones

* failure domains를 너머 트래픽을 분산하는 기순은 cloud providers마다 다르다.
  * 관련된 [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers)의 documentation을 자세히 확인하라.
  * federated cluster에서 Ingress deploying의 자세한 사항에 대해서는 [federation documentation](https://kubernetes.io/docs/concepts/cluster-administration/federation/)를 참조해라.

## Future Work

* Ingress와 관련된 리소스의 진화에 관해 [SIG Network](https://github.com/kubernetes/community/tree/master/sig-network)의 자세한 사항을 확인하라.
  * 다양한 Ingress controllers에 대한 진화에 관해 [Ingress repository](https://github.com/kubernetes/ingress/tree/master)의 자세한 사항을 확인하라

## Alternatives

* 직접적으로 Ingress 리소스를 포함하지 않는 여러 방법으로 Service를 expose할 수 있다.
  * [Service.Type=LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)를 사용하라
  * [Service.Type=NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)를 사용하라