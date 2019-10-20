# Assigning Pods to Nodes

* 파드가 특정한 노드에서만 동작하도록 제한을 두거나 특정 노드에서 동작하는 것을 선호하도록 설정할 수 있다.
  * 여러가지 방법이 있으며, 추천하는 방법은 label selector를 이용하여 선택하는 것이다.
  * 일반적으로 이런 constrains는 스케쥴러가 자동적으로 합리적으로 위치시키기 때문에(파드를 노드 전체에 분배하고 남은 리소스가 충분하지 않은 노드에는 파드를 위치시키지 않는다.) 불필요하지만 파드가 어느 노드에 있어야 하는지 조작해야할 상황이 있다.
    * 예를 들어 파드가 SSD가 붙어있는 장비에 있어야 하거나 두개의 다른 서비스에서 서로 연관된 파드가 같은 availability zone 내에서 많이 통신하는 경우

## nodeSelector

* `nodeSelector`는 node selection constraint로 추천하는 가장 간단한 형태이다.
  * `nodeSelector`는 PodSpec의 필드이다.
  * 이는 key-value 쌍으로 지정되어있다.
  * 파드가 노드에서 동작하기 적합해지도록 노드는 반드시 각 key-value 쌍을 label로 표시해야 한다. (이는 또한 추가적인 라벨이 될 수 있다.)
  * 가장 일반적인 사용법은 하나의 key-value 쌍이다.
* 예시를 통해 어떻게 `nodeSelector`를 사용하는지 살펴보자.

### Step Zero: Prerequisites

* 이 예시는 쿠버네티스에 대한 기본적인 이해를 가지고 있고 쿠버네티스 클러스터를 구성했다고 가정한다.

### Step One: Attach label to the node

* `kubectl get nodes`를 실행하여 클러스터 노드의 이름을 가지고 온다.
  * 라벨을 추가하고 싶은 노드를 하나 선택하고 `kubectl label nodes <node-name> <label-key>=<label-value>`를 실행하여 선택한 노드에 라벨을 추가한다.
  * 예를 들어, 노드의 이름이 'kubernetes-foo-node-1.c.a-robinson.internal'이고 내가 원하는 label이 'disktype=ssd'라면, `kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd`를 실행하면 된다.
  * 이를 `kubectl get nodes --show-labels`를 다시 실행하고 모드가 label을 가지고 있는지 확인함으로써 검수할 수 있다.
  * 또한 `kubectl describe node "nodename"`을 사용하여 주어진 노드의 모든 라벨 리스트를 확인할 수 있다.

### Step Two: Add a nodeSelector field to your pod configuration

* 실행시키고 싶은 아무 파드의 config 파일을 가지고 nodeSelector 섹션을 이처럼 추가해라. 예를 들어:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels:
      env: test
  spec:
    containers:
    - name: nginx
      image: nginx
  ```

* 여기서 nodeSelector를 추가하면:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels:
      env: test
  spec:
    containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
    nodeSelector:
      disktype: ssd
  ```

* `kubectl apply -f https://k8s.io/examples/pods/pod-nginx.yaml`을 실행하면 파드는 라벨을 붙인 노드에서 스케쥴링 될 것이다.

  * `kubectl get pods -o wide`를 실행시키고 파드가 어느 노드에서 실행되고 있는지를 확인하여 검수할 수 있다.

## Interlude: built-in node labels

* 사용자가 추가한 label에 추가적으로 노드는 이미 발행된 표준 라벨 쌍들이 있다.
  * [`kubernetes.io/hostname`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-hostname)
  * [`failure-domain.beta.kubernetes.io/zone`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domain-beta-kubernetes-io-zone)
  * [`failure-domain.beta.kubernetes.io/region`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domain-beta-kubernetes-io-region)
  * [`beta.kubernetes.io/instance-type`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#beta-kubernetes-io-instance-type)
  * [`kubernetes.io/os`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os)
  * [`kubernetes.io/arch`](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-arch)

> Note : 이 라벨들의 값은 cloud provider에 따라 다르고 전적으로 믿을 수 있지는 않다. 예를 들어 `kubernetes.io/hostname`값은 어떤 환경에서는 노드의 이름과 같지만 어떤 환경에서는 다른 값이다.

## Node isolation/restriction

* 노드 오브젝트에 라벨을 추가하는 것은 특정한 노드나 노드의 그룹에 대해서 파드가 타게팅할 수 있도록 한다.
  * 이는 특정한 파드가 명확한 분리, 보안 또는 규제의 속성을 가진 노드에서 동작하도록 정해줄 때 쓰인다.
  * 이러한 목적으로 라벨을 쓸 때, 노드의 kubelet process에 의해서 수정되지 않는 리벨 key를 고르는 것이 강력하게 추천된다.
    * 이는 compromised node가  kubelet credential을 통해 자기 자신의 노드 오브젝트에 라벨을 설정하고 compromised node에 워크로드를 스케쥴링 하는 것에 영향을 주는 것으로부터 방지한다.
* `NodeRestriction` admission 플러그인은 kubelet이 node-`restriction.kubernetes.io/` prefix를 가진 라벨을 설정하거나 수정하는것을 막아준다.
  * 그 prefix를 가진 라벨을 node isolation에서 사용하도록 하기 위해서는
    * Kubernetes v1.11+이어서 NodeRestriction이 사용가능한지 확인해야한다.
    * Node authorizer를 사용하고 있고 NodeRestriction admission plugin을 활성화했는지 확인한다.
    * node object에서 `node-restriction.kubernetes.io/` prefix 아래에 라벨을 추가하고 이 라벨을 node selectors에서 사용한다. 예를 들어 `example.com.node-restriction.kubernetes.io/fips=true`나 `example.com.node-restriction.kubernetes.io/pci-dss=true`

## Affinity and anti-affinity

* `nodeSelector`는 특정한 라벨을 가진 노드에 파드를 제한할 수 있는 매우 간단한 방법을 제공한다.
  * affinity/anti-affinity 기능은 표헌할 수 있는 제약사항의 종류를 엄청나게 능려준다.
  * 주요 향상점
    * language가 더 expressive하다. (단순한 "정확한 일치의 AND"가 아니다.)
    * hard requirement가 아니라 "soft"/"preference" 룰을 설정할 수 있어서 스케쥴러가 이를 만족하지 않으면 파드는 여전히 스케쥴된다.
    * 노드 자체에 라벨에 대한 것이 아니라 노드에서 동작하는 다른 파드들(또는 다른 topological domain)에 대해서 제한을 걸 수 있어서 어느 파드가 함께 공존하고, 공존하지 않도록 설정할 수 있다.
* affinity 기능은 "node affinity"와 "inter-pod affinity/anti-affinity" 두가지 타입으로 구성된다.
  * node affinity는 `nodeSelector`와 비슷하면서 위의 두가지 장점을 가지고 있는 반면 inter-pod affnity/anti-affinity는 위의 3번째 아이템에 기술한대로 노드 라벨이 아닌 파드 라벨에 대한 제한사항이면서 1번째, 2번째의 속성도 가지고 있다.

### Node affinity

* node affinity는 개념적으로 `nodeSelector`과 비슷하다.

  * 이는 노드의 라벨을 기준으로 파드가 어느 노드에서 스케쥴링 되는 것이 적합한지 제한을 둘 수 있다.

* 현재 `requiredDuringSchedulingIgnoredDuringExecution`과 `preferredDuringSchedulingIgnoredDuringExecution` 두가지 타입의 node affinity가 있다.

  * 이를 전자는 노드에 파드가 스케쥴 될 때 반드시 지정된 룰을 만족해야 함(`nodeSelector`와 같지만 더 expressive syntax를 사용한다.)을 명시하는 반면 후자는 선호로써 스케쥴러가 이를 적용하려고 노력하지만   보장되지는 않는다는 점에서 각각 "hard"와 "soft"라고 생각하면 된다.
  * 이름에서 "IngnoredDuringExecution" 파트는 `nodeSelector`가 동작하는 방법과 비슷하게 런타임에 노드의 라벨이 변경될 경우 그 파드의 affinity 룰은 만족하지 않게 되어도 파드는 여전히 그 노드에서 동작함을 의미한다.
  * 나중에는 `requiredDuringSchedulingIgnoredDuringExecution`처럼 동작하지만 파드의 node affinity 조건을 더 이상 만족하지 않는 노드에서 파드를 추출하는 `requiredDuringSchedulingRequiredDuringExecution`를 제공하는 것을 목표로 한다.

* 따라서 `requiredDuringSchedulingIgnoredDuringExecution`의 예시는 "Intel CPUs를 사용하는 노드에서만 동작하는 파드"이고 `preferredDuringSchedulingIgnoredDuringExecution`는 "파드의 묶음이 failure zone XYZ에서 동작하도록 노력하지만 가능하지 않다면 아무데서나 동작하도록 허락한다"가 될 것이다.

* Node affinity는 PodSpec에서 `affinity` 필드의 `nodeAffinity` 필드에 지정되어 있다.

* node affinity를 사용하는 예시

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-node-affinity
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/e2e-az-name
              operator: In
              values:
              - e2e-az1
              - e2e-az2
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
            - key: another-node-label-key
              operator: In
              values:
              - another-node-label-value
    containers:
    - name: with-node-affinity
      image: k8s.gcr.io/pause:2.0
  ```

* 이 node affinity 룰은 파드가 `kubernetes.io/e2e-az-name`의 키를 가지고 값이 `e2e-az1`이나 `e2e-az2`인 라벨을 가진 노드에서만 위치할 수 있음을 이야기한다.

  * 추가적으로 그 기준을 만족하는 노드들에서 `another-node-label-key` 키를 가지고 있고 `another-node-label-value`를 가지고 있는 노드를 선호한다.

* 예시에서 `In` operator가 사용된 것이 보인다.

  * 새로운 node affinity syntax로 다음의 operator를 지원한다: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.
  * `NotIn`과 `DoesNotExist`를 사용하여 node anti-affinity 동작을 할 수 있고 또는 node taints로 파드를 특정 노드에서 배제할 수 있다.

* `nodeSelector`와 `nodeAffinity` 모두를 지정한다면 둘 다 후보 노드에 대해서 만족해야 스케쥴링이 된다.

* 여러개의 `nodeAffinity`와 연관된 `nodeSelectorTerms`를 지정할 경우 파드는 `nodeSelectorTerms`를 만족하는 노드중 하나에 스케쥴링된다.

* 파드가 스케쥴된 노드에서 라벨을 지우거나 변경했을 때 파드는 지워지지 않을 수 있다.

  * 다른말로 하면 affinity selecton은 파드가 스케쥴링 될때만 동작을 한다.

* `preferredDuringSchedulingIgnoredDuringExecution`에서의 `weight` 필드는 1부터 100까지가 범위이다.

  * 모든 스케쥴링 조건을 만족하는 각각에 노드에 대해 스케쥴러는 이 필드를 통해서 반복하여 노드가 MatchExpressions와 맞으면 "weight"를 sum에 더한다.
  * 이 점수는 노드에 대한 다른 priority functions의 점수와 합해진다.
  * 총 점수가 가장 높은 노드가 가장 선호된다.

### Inter-pod affinity and anti-affinity

* inter-pod affinity와 anity-affinity는 노드의 라벨 기준이 아니라 노드에서 이미 동작하고 있는 파드들의 라벨을 기준으로 파드가 어느 노드에서 스케쥴링 될지를 경정하는 것이다.
  * "이 파드는 반드시 Y룰을 만족하는 파드가 하나 이상 동작하고 있는 X에서 동작해야 한다.(또는 anity-affinity라면, 하면 안된다.)"의 형식으로 룰이 형성된다.
  * Y는 네임스페이스와 연관된 추가적인 리스트와 함께 LabelSelector로 표현된다.
    * 이는 노드와 다른데 파드가 네임스페이스를 기반으로 동작하고(그래서 파드의 라벨도 암묵적으로 네임스페이스 기반이다.) 파드 라벨에서의 라벨 셀렉터가 반드시 어느 네임스페이스에서 셀렉터가 적용되는지를 지정해야 하기 때문이다.
  * 개념적으로 X는 node, rack, cloud provider zone, cloud provider region 등과 같은 topology domain이다.
  * 시스템이 그런 topology domain을 나타내기 위한 노드 라벨에 대한 키인 `topologyKey`를 이용해서 표현한다.
    * 예들 들어 위에 섹션에서 리스트된 [Interlude: built-in node labels](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#built-in-node-labels)를 보아라.

> Note: Inter-pod affinity와 anti-affinity는 실질적인 양의 프로세싱을 필요로 하여 큰 클러스터에서는 스케쥴링의 속도를 줄일 수 있다. 수백개의 노드를 가진 클러스터에서는 사용하지 않기를 권장한다.

> Note: 파드 anti-affinity는 노드가 지속적으로 라벨이 되어있도록 요구한다. 예를 들어 모든 클러스터의 노드가 `topologyKey`를 만족하는 적절한 라벨을 가지고 있어야 한다. 만약 일부 또는 모든 노드가 `topologyKey` 라벨이 없으면 의도치 않은 동작을 한다.

* node affinity에서 각각 "hard"와 "soft" 조건을 의미하는 `requiredDuringSchedulingIgnoredDuringExecution`와 `preferredDuringSchedulingIgnoredDuringExecution` 두가지 종류의 pod affinity와 anti-affinity가 있다.
  * 이전의 node affinity section을 참고하라.
  * `requiredDuringSchedulingIgnoredDuringExecution` affinity는 "service A와 service B는 서로 통신이 많기 때문에 같은 존에 파드가 위치해야 한다."가 될 것이고 `preferredDuringSchedulingIgnoredDuringExecution` anti affinity는 "이 서비스를 zone에 걸쳐서 퍼뜨려라"가 될 것이다.(zone의 개수보다 파드가 더 많을 수도 있기 때문에 hard requirement는 말이 되지 않는다.)
* Inter-pod affinity는 PodSpec에서 `affinity` 필드의 `podAffinity` 필드에 지정되어 있다.
  * 그리고 inter-pod anti-affinity는 PodSpec에서 `affinity` 필드의 `podAntiAffinity` 필드에 지정되어 있다.

## An Example of a pod that uses pod affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: failure-domain.beta.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

* 이 파드의 affinity는 하나의 pod affinity rule과 pod anti-affinity rule을 가지고 있다.
  * 이 예시에서 `podAffinity`는 `requiredDuringSchedulingIgnoredDuringExecution`인 반면 `podAntiAffinity`는 `preferredDuringSchedulingIgnoredDuringExecution`이다.
  * pod affinity rule은 최소 하나의 파드가 key가 "security"이고 value가 "S1"인 라벨을 가지고 있는 존과 같은 곳에 위치한 노드에만 스케쥴링 될 수 있다는 것을 의미한다.
    * 더 정확하게는 노드 N의 라벨 중 key가 `failure-domain.beta.kubernetes.io/zone`인 것이 있고 클러스터 내에 key가 `failure-domain.beta.kubernetes.io/zone`인 노드가 최소 하나 있으면서 값이 V인 것이 key가 "security"이고 value가 "S1"인 라벨을 가진 파드를 동작시키면 파드는 노드 N에서 동작하기에 적합하다는 의미이다.
  * pod anti-affinity 룰은 key가 "security"이고 value가 "S2"인 라벨을 가진 파드가 동작하고 있는 노드에는 스케쥴되지 않기를 선호한다는 것을 의미한다.
    * 만약 `topologyKey`가 `failure-domain.beta.kubernetes.io/zone`이면 key가 "security"이고 value가 "S2"인 라벨을 가진 파드와 같은 존에 있는 노드에 스케쥴 되지 않는다는 것을 의미한다.
  * pod affinity와 pod anti-affinity에 대한 다양한 예시는 [design doc](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md)에서 볼수 있다.
* pod affinity와 anity-affinity에 대한 가능한 operator는 `In`, `NotIn`, `Exists`, `DoesNotExist`가 있다.
* 원칙적으로 `topologyKey`는 적합한 label-key 아무것이나 가능하다. 하지만 퍼포먼스와 보안상 이유로 topologyKey에 대한 몇가지 제약사항이 있다.
  * affinity와 `requiredDuringSchedulingIgnoredDuringExecution` pod anti-affinity에 대해서 빈 `topologyKey`는 허용되지 않는다.
  * `requiredDuringSchedulingIgnoredDuringExecution` pod anti-affinity에 대해 admission controller `LimitPodHardAntiAffinityTopology`는 `topologyKey`가 `kubernetes.io/hostname` 제한되도록 한다. 만약 custom topologies에 대해서 허용하도록 하고 싶다면 admission controller를 수정하거나 간단히 비활성화 하면 된다.
  * `preferredDuringSchedulingIgnoredDuringExecution` pod anti-affinity에 대해 빈 `topologyKey`는 "all topologies"로 해석된다. ("all topologies"는 여기서 `kubernetes.io/hostname`, `failure-domain.beta.kubernetes.io/zone`, `failure-domain.beta.kubernetes.io/region`으로 제한된다.
  * 위의 케이스를 제외하면 `topologyKey`는 적합한 label-key가 될 수 있다.
* `labelSelector`와 `topologyKey`에 추가적으로 `labelSelector`가 매칭되어야 하는 `namespaces`의 리스트를 지정할 수 있다.(이는 `labelSelector`와 `topologyKey`와 같은 레벨로 동작한다.)
  * 만약 비어있거나 생략되었다면 이는 affinity/anti-affinity 정의가 있던 파드의 네임스페이스로 지정이 된다.
* `requiredDuringSchedulingIgnoredDuringExecution` affinity와 anti-affinity와 연관된 모든 `matchExpressions`는 노드에 스케쥴링 될 때 파드에 대해서 만족해야 한다.

## More Practical Use-cases

* 파드간의 Affinity와 AntiAffinity는 ReplicaSets, StatefulSets, Deployments, 등 상위 레벨의 컬렉션과 함께 사용될 때 더 유용할 수 있다.
  * 같은 노드내 처럼 같은 topology에 함께 위치한 곳에 워크로드가 위치될 수 있도록 쉽게 설정할 수 있다.

### Always co-located in the same node

* 세 노드를 가진 클러스터에서 웹 어플리케이션은 redis와 같은 in-memory 캐시를 가진다.
  * 우리는 web-servers가 가능한 cache와 함께 위치하도록 하고 싶다.
* 다음은 세개의 replicas와 `app=store` selector 라벨을 가진 간단한 redis deployment의 일부분이다.
  * deployment는 `PodAntiAffinity` 설정을 가지고 있어서 스케쥴러가 하나의 노드에 replicas를 위치시키지 않도록 보장해준다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

* 다음의 yaml은 `podAntiAffinity`와 `podAffinity` 설정이 된 webserver deployment의 일부이다.
  * 이는 스케쥴러에게 모든 이 replicas가 `app=store` selector label을 가진 파드와 함께 위치하도록 알려준다.
  * 또한 각 web-server replica는 하나의 노드에 위치되지 않도록 해준다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

* 위의 두 deployments로 생성을 했다면 세개의 노드를 가진 클러스터는 다음과 같이 보일 것이다.


| node-1      | node-2      | node-3      |
| ----------- | ----------- | ----------- |
| webserver-1 | webserver-2 | webserver-3 |
| cache-1     | cache-2     | cache-3     |

* 위에서 볼 수 있듯이 `web-server`의 3 replicas는 예상했던 대로 자동으로 캐쉬와 같이 위치한다.

  ```bash
  kubectl get pods -o wide	
  ```

* 결과는 다음과 비슷하다.

  ```
  NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
  redis-cache-1450370735-6dzlj   1/1       Running   0          8m        10.192.4.2   kube-node-3
  redis-cache-1450370735-j2j96   1/1       Running   0          8m        10.192.2.2   kube-node-1
  redis-cache-1450370735-z73mh   1/1       Running   0          8m        10.192.3.1   kube-node-2
  web-server-1287567482-5d4dz    1/1       Running   0          7m        10.192.2.3   kube-node-1
  web-server-1287567482-6f7v5    1/1       Running   0          7m        10.192.4.3   kube-node-3
  web-server-1287567482-s330j    1/1       Running   0          7m        10.192.3.2   kube-node-2
  ```

### Never co-located in the same node

* 위의 예시는 redis cluster 배포하기 위해 `PodAntiAffinity` 룰을 `topologyKey: "kubernetes.io/hostname"`으로 사용하였고 어떤 두 인스턴스도 같은 호스트에 존재하지 않도록 했다.
  * 같은 기술을 사용하여 HA를 위한 anti-affinity StatefulSet의 예시를 보려면 [ZooKeeper tutorial](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure)을 참조하라.

## nodeName

* `nodeName`은 node selection constraint의 가장 간단한 형태이지만 그 제한사항 때문에 일반적으로 사용하지 않는다.

  * `nodeName`은 PodSpec의 필드이다.
  * 비어있지 않다면 스케쥴러는 파드가 동작하려 하는 named node에서 동작중인 파드와 kubelet을 무시한다.
  * 그래서 `nodeName`이 PodSpec에 있으면 node selection보다 상위로 동작한다.

* `nodeName`을 이용하여 노드를 선택하는데 몇가지 제한사항들이다.

  * named node가 존재하지 않다면 파드는 동작하지 않을 것이고 몇몇 경우에는 자동적으로 삭제된다.
  * named node가 파드에 적절한 리소스를 가지고 있지 않다면 파드는 실패할 것이고 그 이유는 OutOfMemory나 OfOfCpu가 될 것이다.
  * 클라우드 환경에서 노드의 이름은 항상 예측이 가능하거나 안정적인게 아니다.

* `nodeName` 필드를 사용한 파드 config file의 예시이다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - name: nginx
      image: nginx
    nodeName: kube-01
  ```

* 위의 파드는 kube-01 노드에서 동작할 것이다.

## What's next

* Taints는 노드가 파드의 묶음을 없애도록 한다.
* [node affinity](https://git.k8s.io/community/contributors/design-proposals/scheduling/nodeaffinity.md)와 [inter-pod affinity/anti-affinity](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md)에 대한 디자인 문서는 이 기능에 대한 배경 정보를 포함한다.
* 파드가 노드에 할당이 되고 나면 kubelet은 파드를 동작시키고 node-local 리소스를 할당한다.
  * [topology manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)는 node-level resource 할당을 결정하는데 일부분을 차지한다.