# Taints and Tolerations

* node affinity는 파드가 어떤 노드의 묶음에 더 잘 올라간다는 뜻이다.
  * 선호도가 될 수도 있고 필요조건이 될수도 있다.
  * Taints는 반대이다.
    * 파드가 노드에서 쫒겨나도록 한다.
* taints와 toleration은 함께 동작하여 파드가 부적절한 노드에 뜨지 않도록 한다.
  * 하나 이상의 taints가 노드에 적용된다.
    * 이는 taints를 극복하지 못하는 어떤 파드도 노드에서 받아들여지지 않도록 표시한다.
  * toleration은 파드에 적용이 되고, 파드가 매칭이되는 taints를 가진 노드에 뜨도록 허락한다(반드시는 아니다.).

## Concepts

* kubectl taint를 통해서 노드에 taint를 추가할 수 있다.

  * 예시)

    ```bash
    kubectl taint nodes node1 key=value:NoSchedule
    ```

    위의 명령어는 `node`에 대해  taint를 설정한다.

  * taint는 `key`라는 키를 가지고 있고 value라는 값을 가지고있으며 `NoSchedule`이라는 taint effect이 있다.

  * 이는 매칭이 되는 toleration이 없다면 `node1` 으로 뜨지 않는다는 것을 의미한다.

* 위의 명령어로 추가된 taint를 없애고 싶다면, 다음과 같이 할 수 있다.

  ```bash
  kubectl taint nodes node1 key:NoSchedule-
  ```

* PodSpec에서 파드의 toleration을 지정할 수 있다.

  * 다음의 toleration들은 위의 `kubectl taint` 명령으로 생성된 taint와 매칭이 되는 toleration이며 따라서 이 toleration을 가진 파드는 `node1`에 뜰 수 있다.

    ```yaml
    tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
    ```

    ```yaml
    tolerations:
    - key: "key"
      operator: "Exists"
      effect: "NoSchedule"
    ```

  * 다음은 toleration을 사용하는 파드의 예시이다.

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
      tolerations:
      - key: "example-key"
        operator: "Exists"
        effect: "NoSchedule"
    ```

  * toleration은 key와 effect가 같은 상황에서 다음의 경우에 taint와 매칭이 된다.

    * `operator`가 `Exists`인 경우 (`value`가 지정되면 안된다. )
    * `operator`가 `Equal`이며 `value`가 같을 때
    * `operator`가 지정되지 않았을 경우 기본값은 `Equal`이다.

> Note:
> 두가지 특이 케이스가 있다.
>
> * operator가 `Exists`이고 `key`가 비어있다면 모든 key, value와 매칭이 되며 이는 모든 것을 tolerate한다는 의미이다.
>
> ```yaml
> tolerations:
> - operator: "Exists"
> ```
>
> * `effect`가 비어있을 경우 `key`에 해당하는 모든 effect와 매칭이 된다.

* 위 예시는 `effect`가 `NoSchedule`인 것을 보여주었다.

  * 이와는 다르게 `effect`를 `PreferNoSchedule`로 설정할 수도 있다.
  * 이는 "선호도" 또는 "soft" 버전의 `NoSchedule`이다.
    * 시스템은 taint를 tolerate하지 못하는 노드에 대해서 파드를 띄우지 않도록 노력하겠지만, 반드시는 아니다.
  * 세번째 `effect`는 `NoExecute`로, 후에 기술한다.

* 하나의 노드에 대해서 여러개의 taints를 설정할 수도 있고 하나의 파드에 대해서 여러개의 toleration을 설정할 수도 있다.

  * Kubernetes는 여러개의 taints와 toleration을 필터처럼 처리한다.
    * 노드의 모든 taints에서 시작하여 파드의 toleration과 매칭되는 노드의 taint는 무시한다.
    * 그 후 남아있는 taints들은 파드에 effect를 행사한다.
  * 특히,
    * `NoSchedule` effect를 가진 최소 하나의 무시되지 않은 taint가 있으면 Kubernetes는 그 노드에서 해당 파드가 뜨지 않도록 한다. 
    * `NoSchedule` effect를 가진 무시되지 않은 taint가 있지만 최소 하나의 `PreferNoSchedule` effect를 가진 무시되지 않은 taint가 있다면 Kubernetes는 해당 파드를 그 노드에 띄우지 않도록 *노력* 할 것이다.
    * 최소 하나의 `NoExecute` effect를 가진 무시되지 않은 taints가 있다면 파드는 이미 그 노드에 떠있다면 해당 노드로부터 축출이 될 것이고 아직 그 노드에서 작동하고 있지 않았다면 그 노드로 뜨지 않을 것이다.

* 예를 들어 node에 다음과 같이 taint 처리를 했다고 생각해보자.

  ```bash
  kubectl taint nodes node1 key1=value1:NoSchedule
  kubectl taint nodes node1 key1=value1:NoExecute
  kubectl taint nodes node1 key2=value2:NoSchedule
  ```

* 그리고 파드에는 두가지의 toleration이 있다고 가정하자.

  ```yaml
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
  ```

* 이 경우에 세번째 taint와 매칭이 되는 toleration이 없기 때문에 파드는 노드로 뜨지 않을 것이다.

  * 하지만 taint가 추가되는 시점에 노드에 이미 파드가 떠있다면 계속해서 동작할 것이다.
    * 세번째 taint는 파드에서 tolerate할 수 없는 셋중 유일한 하나이기 때문이다.

* 일반적으로 `NoExecute` effect를 가진 taint가 노드에 추가되면 taint를 만족하지 못하는 어떤 파드든지 즉시 축출이 될 것이고 taint를 tolerate을 하는 파드는 축출되지 않을 것이다.

  * 하지만, `NoExecute` effect를 가진 toleration은 `tolerationSeconds`라는 옵션을 지정할 수 있는데, 이는 taint가 추가되고 나서 얼마나 파드가 노드에 머무를 수 있는지를 지정해 주는 것이다.

  * 예시

    ```yaml
    tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoExecute"
      tolerationSeconds: 3600
    ```

    위의 toleration은 파드가 동작하고 있고 toleration과 매칭이 되는 taint가 노드에 추가가 되면 파드는 3600초동안 유지되다가 축출이 된다.

  * 만일 taint가 그 대기시간 전에 사라지게 되면 파드는 축출되지 않는다.

## Example Use Cases

* taints와 toleration은 파드가 노드에 뜨지 않게 하거나 뜨면 안되는 파드를 노드에서 축출하는 유연한 방법을 제공한다.

  * Use case

    * Dedicated Nodes

      * 만약 어떤 노드들을 특정 유저들만 따로 사용하도록 하고싶을 때 이 노드들에 대해 taint를 추가할 수 있고, 이에 맞는 toleration을 파드에 추가하여 그렇게 할 수 있다.

        ```bash
        kubectl taint nodes nodename dedicated=groupName:NoSchedule
        ```

        * 이는 [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)을 작성하여 쉽게 구현할 수 있다.

      * toleration을 가진 파드는 다른 노드들 뿐만 아니라 taint처리된 노드(할당된 노드)를 사용할 수 있게 된다.

      * 노드를 할당해주고, 그 노드만 사용하도록 하고 싶다면 taint와 비슷한 label을 그 할당된 노드에 추가해주면 된다. (예를 들어 `dedicated=groupName`)

        * 그 후 admission controller는 추가적으로 node affinity를 설정하여 `dedicated=groupName`을 라벨로 가지고 있는 노드에만 파드가 뜨도록 설정하면 된다.

    * Nodes with Special Hardware

      * 클러스터에서 노드들의 작은 그룹이 특정한 하드웨어를 가지고 있다면 (예를 들어, GPU), 그 하드웨어가 필요없는 파드들은 해당 노드에 뜨지 않는것이 바람직하며 따라서 그 하드웨어를 필요로 하는 나중에 뜨려하는 파드들이 그 노드에 뜰 수 있는 공간을 남겨두어야 한다.

      * 이는 그 특정한 하드웨어를 가진 노드들에 대해서 taint 처리를 하여 구현할 수 있다.

        * 예)

          ```bash
          kubectl taint nodes nodename special=true:NoSchedule
          ```

           또는

          ```bash
          kubectl taint nodes nodename special=true:PreferNoSchedule
          ```

      * 그리고 이에 해당하는 toleration을 그 하드웨어가 필요한 파드에 추가하면 된다.

      * dedicated nodes use case에서처럼, 이는 [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)를 사용하여 쉽게 적용할 수 있을 것이다.

      * 예를 들어, 특정한 하드웨어를 나타낼 때 [Extended Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)를 이용하는 것을 추천하고, 그 하드웨어를 쓰는 노드들을 resource name으로 taint 처리를 하고 [ExtendedResourceToleration](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration) admission controller를 동작시킨다.

      * 이제 노드들이 taint처리가 되었기 때문에 toleration이 업슨 파드들은 그곳에 뜨지 않는다.

      * 하지만 extended resource를 요청하는 파드를 띄우려 할 때, `ExtendedResourceToleration` admission controller가 자동적으로 올바른 toleration을 파드에 추가하여 파드가 그 특정 하드웨어를 쓰는 노드로 뜨도록 할 것이다.

      * 이는 이 특정한 하드웨어 노드들이 그 하드웨어를 요청하는 파드들에 대해서 할당이 되어있고, 파드들에 대해 일일히 toleration을 추가하지 않아도 됨을 의미한다. 

    * Taint based Evictions (beta feature)

      * 노드에 문제가 있을 때 각 파드당 설정할 수 있는 축출 동작방식을 다음 섹션에서 설명하도록 한다.

## Taint based Evictions

* 이전에 우리는 이미 파드가 노드에 동작하고 있는 경우에 `NoExecute` taint effect에 대해 언급을 했다.
  * taint를 tolerate할 수 없는 파드들은 즉시 축출된다.
  * toleration 규격에 `tolerationSeconds`를 지정하지 않고 taint를 tolerate할 수 있는 파드는  계속해서 남아있는다.
  * 지정된 `tolerationSeconds`를 가지고 taint를 tolerate할 수 있는 파드는 해당 시간동안 남아있는다.
* 추가적으로 Kubernetes 1.6에서는 노드의 문제점들을 나타내는 것을 alpha버전으로 지원한다.
  * 다른 말로, node controller는 특정 상황이 되면 자동적으로 node를 taint처리한다.
  * 다음의 taint들은 내장되어있다.
    * `node.kubernetes.io/not-ready`
      * 노드가 아직 준비되지 않은 것이다.
      * 이는 NodeCondition의 `Ready`가 "`False`"인 상태이다.
    * `node.kubernetes.io/unreachable`
      * 노드가 node controller에 의해서 unreachable한 상태이다.
      * 이는 NodeCondition의 `Ready`가 "`Unknown`"인 상태이다.
    * `node.kubernetes.io/out-of-disk`
      * 노드에서 disk가 없는 상태이다.
    * `node.kubernetes.io/memory-pressure`
      * 노드에서 memory pressure인 상태이다.
    * `node.kubernetes.io/disk-pressure`
      * 노드에서 disk가 부족한 상태이다.
    * `node.kubernetes.io/network-unavailable`
      * 노드의 네트워크가 사용불가능한 상태이다.
    * `node.kubernetes.io/unschedulable`
      * 노드가 스케쥴이 가능하지 않은 상태이다.
    * `node.cloudprovider.kubernetes.io/uninitialized`
      * Kubelet이 외부 cloud provider에서 시작될 때 이 taint는 노드를 사용불가능이라고 표시한다.
      * cloud-controller-manager에서 controller가 이 노드를 초기화하고 나면 kubelet은 taint를 삭제한다.
  * 1.13 버전에서 `TaintBasedEvctions` 기능은 beta로 승격되었고 기본적으로 활성화 되어있으므로 taint는 자동적으로 NodeController(또는 kubelet)에 의해서 추가되고 일반적인 Ready NodeCondition을 기반으로 한 파드 축출은 비활성화 되어있다.

> Note: 노드 문제로 인한 파드의 축출 동작에 비율을 통한 제약을 유지하기 위해 시스템은 사실상 taint를 제한된 비율로 추가한다. 이는 master가 노드로부터 제거되는 것처럼 엄청난 파드 축출 시나리오를 방지해준다.

*  `tolerationSeconds`와 결합된 이 beta 기능은 파드가 이런 하나 이상의 문제를 가진 노드들에서 얼마나 남아있을지를 지정할 수 있다.

* 예를 들어, 많은 local state를 가진 어플리케이션들은 네트워크 분할의 상황에서 오랫동안 남아있기를 원하고, 그 분할이 해결되어서 파드 축출이 되도록 안되도록 하고싶다.

  * 이 경우에 파드에서 다음과 같은 toleration을 쓸 수 있다.

    ```yaml
    tolerations:
    - key: "node.kubernetes.io/unreachable"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 6000
    ```

* Kubernetes는 자동으로 `node.kubernetes.io/not-ready`에 대해 유저가 따로 `node.kubernetes.io/not-ready`에 대한 toleration을 설정하지 않았다면 `tolerationSeconds=300`을 추가고 있다는 것을 참고하라.

  * 이처럼 따로 toleration 설정을 안했을 경우 `node.kubernetes.io/unreachable`을 `tolerationSeconds=300`으로 지정한다.

* 이런 자동으로 추가되는 toleration들은 기본적으로 파드가 이런 문제점이 발생했을 때 5분동안 남아있도록 한다.

  * 이 두 기본 toleration들은 [DefaultTolerationSeconds admission controller](https://git.k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds)에 의해서 추가된다.

* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 파드들은 `NoExecute` toleration을 다음의 taint에서 `tolerationSeconds`가 없이 생성된다.

  * `node.kubernetes.io/unreachable`
  * `node.kubernetes.io/not-ready`

* 이는 DaemonSet 파드가 이 기능이 활성화되지 않은 것과 같이 이런 문제들로 인해서 절대 축출되지 않도록한다.

## Taint Nodes by Condition

* 1.12버전에서 `TaintNodesByCondition` 기능이 베타버전으로 승격되어 node lifecycle controller가 자동적으로 노드의 상태에 상응하는 taints를 생성한다.
  * 이와 비슷하게 스케쥴러는 노드의 상태를 체크하지 않는 대신 taints를 체크한다.
  * 이는 노드의 상태가 노드에 어떤것이 뜰지에 대해 영향을 주지 않는다는 것을 보장한다.
  * 유저는 적절한 파드 toleration을 추가함으로써 몇몇 노드 문제들(노드의 상태로 표현이 된)에 대해서 무시하도록 선택할 수 있다.
  * `TaintNodesByCondition`은 `NoSchedule` effect로만 노드를 taint 처리함을 참고하라.
  * `NoExecute` effect는 1.13버전부터 기본적으로 활성화되었고, 베타버전인 `TaintBasedEviction`에 의해서 처리된다.
* Kubernetes 1.8에서부터 DaemonSet controller는 자동으로 다음의 `NoSchedule` toleration들을 모든 daemon에 추가하여 DaemonSet들이 멈추는 것을 방지한다.
  * `node.kubernetes.io/memory-pressure`
  * `node.kubernetes.io/disk-pressure`
  * `node.kubernetes.io/out-of-disk` (중요한 파드들만)
  * `node.kubernetes.io/unschedulable` (1.10버전 이후)
  * `node.kubernetes.io/network-unavailable` (host network만)
* 이런 toleration을 추가하는 것은 하위 호환성을 보장한다.
  * DaemonSet에 임의의 toleration을 추가하는 것도 가능하다.