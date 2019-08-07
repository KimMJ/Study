# Network Policies

* network policy는 파드의 그룹이 서로와 다른 network endpoints간에 어떻게 통신하는지에 대한 specification이다.
* `NetworkPolicy` resources는 파드를 선택할 때 사용되고 어떤 트래픽이 선택된 파드에 허용되는지를 지정해주는 룰을 정의한다.

## Prerequisites

* Network policies는 network plugin에 의해 구현되었고 따라서 사용자는 반드시 `NetworkPolicy`(간단히 이를 구현해주는 controller 없이 리소스를 생성하는 것은 아무런 영향을 주지 않을 것이다.) 를 지원하는 networking solution을 사용해야 한다.

## Isolated and Non-isolated Pods

* 기본적으로 pods는 non-isolated이다.
  * 모든 source로부터 트래픽을 받는다.
* 파드가 그것들을 선택하는 NetworkPolicy를 가지면 isolated가 된다.
  * 특정한 파드를 선택하는 namespace에 NetworkPolicy가 한번이라도 있게되면 파드는 NetworkPolicy에 의해 허용되지 않는 모든 연결을 거부할 것이다. (어느 NetworkPolicy에게도 선택되지 않은 namespace안의 다른 파드들은 계속해서 모든 트래픽을 받을 것이다.)

## The NetworkPolicy Resource

* [NetworkPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#networkpolicy-v1-networking-k8s-io)를 봐서 resource의 full definition을 확인하라.

* `NetworkPolicy`의 예시는 다음과 같을 것이다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: test-network-policy
    namespace: default
  spec:
    podSelector:
      matchLabels:
        role: db
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - ipBlock:
          cidr: 172.17.0.0/16
          except:
          - 172.17.1.0/24
      - namespaceSelector:
          matchLabels:
            project: myproject
      - podSelector:
          matchLabels:
            role: frontend
      ports:
      - protocol: TCP
        port: 6379
    egress:
    - to:
      - ipBlock:
          cidr: 10.0.0.0/24
      ports:
      - protocol: TCP
        port: 5978
  ```

* 이 API server에 POST하는 것은 선택한 networking solution이 network policy를 지원하지 않는다면 아무런 영향도 주지 않을 것이다.

* **Mandatory Fields**

  * 다른 Kubernetes config처럼 `NetworkPolicy`는 `apiVersion`, `kind`, `metadata` 필드를 필요로 한다.
  * config files로 작동하는 일반적인 정보에 대해서는  [Configure Containers Using a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/), [Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management) 참조.

* **spec**

  * `NetworkPolicy` spec은 주어진 namespace에서 특정한 network policy를 정의하는 데 필요한 모든 정보를 가지고 있다.

* **podSelector**

  * 각 `NetworkPolicy`는 policy가 적용될 pods의 그룹을 결정하는 `podSelector`를 포함한다.
  * 예시 policy는 label이 "role=db"인 파드를 선택한다.
  * `podSelector`가 비어있으면 네임스페이스의 모든 pods를 선택한다.

* **policyTypes**

  * 각 `NetworkPolicy`는 `Ingress`, `Egress`, 혹은 둘 다를 포함한 `policyTypes` 리스트를 포함한다.
  * `policyTypes` 필드는 주어진 policy가 선택된 파드로의 ingress 트래픽에 적용될지, 선택된 파드로부터의 egress 트래픽에 적용될지 또는 둘 다일지를 나타낸다.
  * 만약 `policyTypes`가 NetworkPolicy에 지정이 되면 기본적으로 `Ingress`는 항상 설정되고 `Egress`는 NetworkPolicy가 egress rules가 있을때 생성될 거이다.

* **ingress**

  * 각 `NetworkPolicy`는 `ingress` rules의 화이트리스트를 포함한다.
  * 각 룰은 `from`, `ports` 섹션이 둘다 일치하는 트래픽을 허용한다.
  * 예시의 policy는 하나의 포트에서 첫번째는 `ipBlock`으로 지정되고 두번째는 `namespaceSelector`, 세번째는 `podSelector`를 통해 지정된 세개의 소스로부터 트래픽이 오는 하나의 rule을 가지고 있다.

* **egress**

  * 각 `NetworkPolicy`는 `egress` rules의 화이트리스트를 포함한다.
  * 각 rule은 `to`, `ports` 섹션이 모두 일치하는 트래픽을 허용한다.
  * 예시의 policy는 single port에서 `10.0.0.0/24`로 가는 모든 트래픽과 일치하는 하나의 rule을 포함한다.

* 따라서 NetworkPolicy의 예시에서는:

  * "default" namespace에서 ingress와 egress 트래픽에 대해 "role=db" 파드를 분리시킨다. (아직 isolated가 되지 않았다면)
  * (Ingress rules) "default" 네임스페이스를 가지고 다음에서 들어와 TCP prot 6379에서 label이 "role=db"인 모든 파드로의 연결을 허용한다.
    * "role=fronted"인 "default" 네임스페이스의 모든 파드
    * "project=myproject"의 라벨을 가지는 namespace안의 모든 파드
    * 172.17.0.0 ~ 172.17.0.255. 172.17.2.0 ~ 172.17.255.255 범위의 IP 주소(ie, 모든 172.17.0.0/16에서 172.17.11.0/24를 제외한 것) 
  * (Egress rules) "default" 네임스페이스에서 "role=db"인 label을 가진 파드에서 TCP port가 5878이고 CIDR 10.0.0.0/24인 곳으로 가는 연결을 허용한다.

* [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)를 통해 예제를 사용해 보아라.

## Behavior of `to` and `from` selectors

* `ingress`의 `from` 섹션이나 `egress`의 `to` 섹션에 지정할 수 있는 네가지 종류의 selector가 있다.

  * podSelector

    * 같은 네임스페이스 내에서 ingress sources나 egress destinations로 허용될 수 있는 `NetworkPolicy`로써 특정한 파드를 선택한다.

  * namespaceSelector

    * 모든 파드가 ingress sources나 egress destinations로 허용할 수 있는 특정한 namespaces를 선택한다.

  * namespaceSelector와 podSelector

    * `namespaceSelector`와 `podSelector` 모두를 지정하는 하나의 `to`/`from` entry로 특정한 namespace에서 특정한 파드를 선택한다.

    * 올바른 YAML syntax를 사용하는 것에 유의하라.

    * ```yaml
       ...
        ingress:
        - from:
          - namespaceSelector:
              matchLabels:
                user: alice
            podSelector:
              matchLabels:
                role: client
        ...
      ```

      `user=alice` 라벨을 가진 namespace 내에서 `role=client` 라벨을 가진 파드로부터의 연결을 허용하는 하나의 `from` element를 포함한다.

    * ```yaml
       ...
        ingress:
        - from:
          - namespaceSelector:
              matchLabels:
                user: alice
          - podSelector:
              matchLabels:
                role: client
        ...
      ```

      두개의 `from` array를 포함하고 `role-client` 라벨을 가진 local Namespace안의 파드로부터 온 연결과 `user=alice` 라벨을 가진 네임스페이스의 모든 파드에서부터 온 연결을 허용한다.
      
    * 의심이 된다면 `kubectl describe`를 사용해서 Kubernetes가 policy를 어떻게 해석하는지 확인해 보아라.
    
  * ipBlock
  
    * 특정한 IP CIDR range를 선택하여 ingress source나 egress destination을 허용한다.
      * cluster-external IP에 대해서 해야하며 그 이유는 파드의 IP는 ephemeral이고 unpredictable이기 때문이다.
    * cluster ingress와 egress 메카니즘은 패킷의 source나 destination IP를 재작성하는 것을 필요로 한다.
      * 이 때 NetworkPolicy 전에 발생하는지 후에 발생하는지 정의되지 않았고 network plugin, cloud provider, `Service` implementation 등의 각각 조합에 따라 다르게 동작할 수 있다.
    * ingress의 경우 어떤 케이스에서는 incoming packet을 actual original source IP기반으로 필터링 할 수 있고 다른 케이스에서는 NetworkPolicy가 동작하는 "source IP"가 `LoadBalancer`나 파드의 노드 IP일 수 있다.
    * egress에 대해서는 파드에서 cluster-external IP로 재작성된 `Service` IP로 가는 연결이 `ipBlock`-based policy에 의해 적용 될수도 안될수도 있음을 의미한다.

## Default policies

* 기본적으로 네임스페이스에서 policy가 없다면 모든 ingress와 egress traffic은 그 네임스페이스의 파드에 대해 허용된다.
  * 다음의 예시는 그 네임스페이스에서 default 동작을 변경할 수 있도록 해준다.

### Default deny all ingress traffic

* 네임스페이스에 대해 모든 파드를 선택하지만 그 파드들로 어떤 ingress traffic도 허용하지 않는 NetworkPolicy를 생성함으로써 "default" isolation policy를 생성할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
  ```

* 이는 파드가 다른 NetworkPolicy에 의해서 선택되지 않더라도 isolated 되도록 해준다.

  * 이 policy는 default egress isolation 동작을 변경하지 않는다.

### Default allow all ingress traffic

* 네임스페이스 내부의 모든 파드에 대해서 트래픽을 허용하고 싶으면(policy를 추가하면 어떤 파드들이 "isolated"로 여겨지게 되더라도) 명시적으로 그 네임스페이스 안의 모든 트래픽을 허용하는 policy를 생성할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-all
  spec:
    podSelector: {}
    ingress:
    - {}
    policyTypes:
    - Ingress
  ```

### Default deny all egress traffic

* 네임스페이스에 대해 모든 파드를 선택하지만 그 파드로부터의 어떤 egress traffic도 허용하지 않는 NetworkPolicy를 생성함으로써 "default" egress isloation policy를 생성할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny
  spec:
    podSelector: {}
    policyTypes:
    - Egress
  ```

* 이는 파드가 다른 NetworkPolicy에 의해서 선택되지 않더라도 egress traffic을 허용하지 않도록 해준다.

  * 이 policy는 default ingress isolation 동작을 변경하지 않는다.

### Default allow all egress traffic

* 네임스페이스의 모든 파드로부터의 트래픽을 허용하고 싶으면(policy가 추가되면 몇몇 파드는 "isolated"로 여겨지게 되더라도) 명시적으로 그 네임스페이스의 모든 egress traffic을 허용하는 policy를 생성할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-all
  spec:
    podSelector: {}
    egress:
    - {}
    policyTypes:
    - Egress
  ```

### Default deny all ingress and egress traffic

* 네임스페이스에 대한 "default" policy를 그 네임스페이스에 다음의 NetworkPolicy를 생성함으로써 모든 ingress와 egress traffic을 막도록 생성할 수 있다.

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
    - Egress
  ```

* 다른 NetworkPolicy에 의해서 선택되지 않은 파드더라도 ingress나 egress traffic에 대해 허용되지 않도록 해준다.

## SCTP support

**FEATURE STATE : ** `Kubernetes v1.12` - alpha

* Kubernetes는 SCTP를 alpha feature로 `NetworkPolicy`의 `protocol` 값으로써 지원한다.
  * 이 feature를 활성화 하려면 cluster administrator는 `"--feature-gates=SCTPSupport=true,..."`처럼 `SCTPSupport` feature gate를 apiserver에서 활성화 해야한다.
  * feature gate가 활성화될 때 사용자는 `NetworkPolicy`의 `protocol` 필드를 `SCTP`로 설정할 수 있다.
  * Kubernetes는 TCP connection과 동일한 방식으로 SCTP associations에 따라 네트워크를 설정할 수 있다.
* CNI plugin은 SCTP를 `NetworkPolicy`에서의 `protocol` 값으로 지원해야한다.