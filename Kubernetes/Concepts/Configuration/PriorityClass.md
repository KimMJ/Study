## PriorityClass

PriorityClass는 non-namespaced object이다. 이는 priority class의 name에 priority에 대한 integer 값을 부여한다. 이 name은 PriorityClass 오브젝트의 메타데이터에서  `name` 필에 지정되어 있다. 이 value는 필수 필드인  `value` 필드에서 지정된다. 더 높은 값을 가질수록 우선순위가 높다.

PriorityClass 오브젝트는 1백만보다 작거나 같은 값의 32비트 integer를 가질 수 있다. 더 큰 값은 일반적으로 preempted 되거나 evicted 되지 않아야 하는 critical system pod로 reserved 되어 있다. cluster admin은 반드시 매핑에서 원하는 만큼 각각 PriorityClass 오브젝트를 생성해야 한다.

PriorityClass는 또한 두가지 옵션 필드가 있다: `globalDefault` 와 `description` 이다. `globalDefault` 필드는 PriorityClass의 value가 `priorityClassName` 없이 파드에서 사용될 때 사용할 수 있다. 하나의 시스템에서는 오직 하나의 `globalDefault`가 true로 설정된 PriorityClass가 존재할 수 있다. `globalDefault`가 설정 된 PriorityClass가 없다면 `priorityClassName`이 없는 파드의 priority는 0이다.

`description`필드는 임의의 string이다. 이는 클러스터의 유저에게 언제 그 PriorityClass를 사용할 지 알려준다.

### Notes about PodPriority and existing clusters

* 이미 있던 클러스터를 업그레이드 하고 이 기능을 활성화 한다면 이미 존재하던 파드들의 priority는 균등하게 0이 될 것이다.
* `globalDefault`를 `true`로 설정한 PriorityClass를 추가한다고 해서 이미 있던 파드들의 priority가 변경되지는 않는다. 이런 PriorityClass의 value는 PriorityClass가 추가된 뒤로 생성되는 파드들에 대해서만 영향을 준다.
* PriorityClass를 삭제한다면, 이미 그 삭제된 PriorityClass의 name을 사용하던 파드는 변경되지 않겠지만, 그 삭제된 PriorityClass의 name을 사용하는 파드는 더이상 생성될 수 없다.

### Example PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

### Non-preempting PriorityClasses (alpha)

1.15에서 `PreemptionPolicy` 필드가 알파 기능으로 추가되었다. 이는 1.15에서는 기본적으로 비활성화 되어 있고 `NonPreemptingPriority` feature gate를 활성화 해야한다.

`PreemptionPolicy: Never`인 파드는 lower-priority pods의 에 있는 scheduling queue에 위치될 것이다. 하지만 이것들은 다른 파드를 preempt하지 않는다. non-preempting pod는 스케쥴링 될 때까지 scheduling queue에서 머무르며 대기할 것이다. Non-preempting pods는 다른 파드처럼 scheduler back-off의 주체이다. 이 말은 스케쥴러가 이 파드들을 띄우고자 하지만 스케쥴되지 않는다면, lower priority로 재시도 하여 다른 lower priority의 파드들이 그것 전에 스케쥴링 되도록 한다.

여전히 Non-preempting 파드는 다른 high-priority pods에 의해 preempted될 수 있다.

`PreemptionPolicy`는 기본적으로 `PreemptLowerPriority`이다. 이는 PriorityClass를 가진 파드들이 lower-priority 파드들을 preempt할 수 있도록 한다. (기본 동작으로 존재하고 있다.) `PreemptionPolicy`가 `Never`로 설정되어 있으면, 해당 PriorityClass의 파드는 non-preempting이 될 것이다.

하나의 사용 예시는 data science workloads이다. 유저는 다른 workloads보다 우선시 되어야 하는 job을 제출하고자 하지만 현재 동작하고 있는 파드들을 preempt하지 않고싶을 수 있다. 이러한 `PreemptionPolicy: Never`인 high priority job은 리소스가 "자연스럽게" free가 되자마자 다른 queued pods보다 먼저 스케쥴이 될 것이다.

### Example Non-preepting PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

## Pod priority

하나 이상의 PriorityClasses를 가지게 된다면 이런 PriorityClass의 specifications에서의 name중 하나를 지정하여 파드를 생성할 수 있다. priority admission controller는 `priorityClassName` 필드를 사용하고 priority의 integer value를 발행할 것이다. priority class를 못찾으면 파드는 거부된다.

 다음의 YAML은 앞선 예시에서 생성한 PriorityClass를 사용하는 Pod configuration의 예시이다. priority admission controller는 specification을 검사하여 pod의 priority를 1000000까지 생성한다.

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
    images: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

### Effect of Pod priority on scheduling order

Kubernetes 1.9 이후부터는 Pod priority가 활성화 되면 스케쥴러는 pending Pods를 priority에 따라 순서를 매기고 pending pods들은 scheduling queue에서 다른 lower priority의 pending pods보다 앞선 순서가 된다. 결과적으로 higher priority Pod는 스케쥴링 조건이 만족되었을 떄 lower priority Pods보다 더 먼저 스케쥴링 될 것이다. 이러한 파드가 스케쥴링이 될 수 없으면 스케쥴러는 계속하여 다른 lower priority Pods를 스케쥴링하려고 시도할 것이다.

## Preemption

파드가 생성될 때 파드는 queue로 들어가서 스케쥴링되기를 기다린다. 스케쥴러는 queue에서 파드를 뽑아서 노드에 뜨도록 시도한다. 파드의 모든 지정된 requirements를 만족하는 노드가 없다면 pending pod에 대해서 preemption logic이 트리거된다. pending Pod를 P라고 해보자. Preemption logic은 P보다 낮은 priority 파드 하나 이상을 삭제하였을 때 그 노드에 P가 뜰 수 있게 되는 노드를 찾는다. 이러한 노드를 찾았다면 하나 이상의 lower priority Pods는 그 노드로부터 evicted 될 수 있다. 파드가 사라지면 P는 그 노드에 스케쥴링 될 수 있다.

### User exposed information

Pod P가 Node N에서 하나 이상의 파드를 preempts하였을 때 Pod P의 status에서 `nominatedNodeName` 필드는 Node N의 name으로 설정된다. 이 필드는 scheduler가 Pod P에 대해 reserved된 리소스를 추적하는데 도움을 주고 유저에게 클러스터 내에서 일어난 preemptions에 대한 정보를 준다.

Pod P가 "nominated Node"로 반드시 스케쥴 되는 것은 아님을 주의하라. victim Pods가 preemted 되었을 때 graceful termination period가 적용된다. scheduler가 victim Pods들이 종료되기를 기다리는 동안 다른 노드가 available 상태가 되면, scheduler는 Pod P를 그 노드로 띄울 것이다. 결과적으로 Pod spec에서 `nominatedNodeName`과 `nodeName`은 항상 같지는 않다. 또한 scheduler가 Node N에서 Pods를 preempts 하였지만 Pod P 보다 higher priority Pod가 생기면 scheduler는 Node N에 대해서 새로운 higher priority Pod를 띄우게 될 것이다. 이러한 경우에 scheduler는 Pod P에 대해 `nominatedNodeName`을 지운고 Pod P가 다른 노드에서 적합하도록 Pods를 preempt한다.

### Limitations of preemption

#### Graceful termination of preemption victims

파드가 preempted 되었을 때 victims는 [graceful termination period](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)를 가진다. 그것들은 작업을 치고 종료하는 데 많은 시간을 사용한다. 그렇지 못할 경우 kill된다. graceful termination period는 scheduler preempts Pods와 pending Pod (P)가 Node (N)에서 스케쥴 되기까지의 시간차이를 발생시킨다. 그동안 스케쥴러는 다른 pending Pods를 스케쥴링 한다. victims가 exit(정상 종료)되거나 terminated(비정상 종료) 되었을 때, 스케쥴러는 pending queue에서 파드 스케쥴링을 시도한다. 그래서 스케쥴러가 victims를 preempts하는 것과 Pod P가 스케쥴되는 시간에 차이가 발생한다. 이러한 갭을 최소화 하는 하나의 방법은 lower priority Pods에 대한 graceful termination period를 0이나 작은 수로 지정하는 것이다.

#### PodDisruptionBudget is supported, but not guaranteed!

[Pod Disruption Budget (PDB)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)는 application owner가 voluntary disruptions(예측이 가능한 장애)에서 동시에 down되는 replicated application의 숫자를 제한할 수 있도록 한다. Kubernetes 1.9는 preempting Pods에서 PDB를 지원하지만 PDB는 best effort일 뿐이다. 스케쥴러는 preemption에 의해서 PDB를 violated하지 않는 victims를 찾으려 노력하지만 이런 victims를 찾을 수 없을 때에는 preemption이 여전히 발생할 수 있고, PDB가 violated 되더라고 lower priority pods는 삭제될 수 있다.

#### Inter-Pod affinity on lower-priority Pods

이 질문에 대한 답이 yes일 때 노드에 대해서 preemption을 고려할 수 있다: "pending pods보다 lower priority인 모든 파드가 노드에서 삭제되면 pending Pod는 그 노드에서 스케쥴 될 수 있는가?"

> Note : Preemption은 반드시 모든 lower-priority Pods를 지우는 것은 아니다. pending Pod가 모든 lower-priority Pods를 지우지 않고 더 적은 수의 파드를 삭제해도 스케쥴링이 가능하다면, 일부분만 삭제될 것이다. 그렇긴 하지만 앞선 질문의 답은 yes여야 한다. 만약 no라면, 노드는 preemption이 전혀 고려되지 않는다.

pending Pod가 노드에서 하나 이상의 lower-priority Pods에 대해 inter-pod affinity를 가지고 있다면 inter-Pod affinity rule은 이런 lower-priority Pods가 없이는 절대 만족할 수 없다. 이러한 경우에 스케쥴러는 그 파드에서 어떤 파드도 preempt하지 않는다. 대신에 다른 노드를 찾는다. 스케쥴러는 적절한 노드를 찾거나 찾지 못할수도 있다. pending Pod가 스케쥴 되리라는 보장은 없게 된다.

이 문제의 추천하는 해결책은 inter-Pod affinity를 priority가 같거나 더 높은 Pod에 대해서만 설정하는 것이다.

#### Cross node preemption

Node N이 preemption 대상으로 고려되고 있고 따라서 pending Pod P가 N에 스케쥴 될 수 있다고 해보자. P는 다른 노드에 있는 파드가 preempted 되어야 N에 뜰수 있게 되는 상황이 있다. 다음은 예시이다:

* Pod P는 Node N에 뜨려고 한다.
* Pod Q는 Node N과 같은 존에 뜬 다른 Node에서 실행중이다.
* Pod P는 Pod Q에 대해 Zone-wide anti-affinity를 가지고 있다.
  (`topologyKey: failure-domain.beta.kubernetes.io/zone`)
* Zone 내에서 Pod P와 다른 파드들 간에 anti-affinity는 없다.
* Pod P를 Node N에 뜨게 하기 위해서는 Pod Q가 preempted 되어야 하지만 scheduler는 cross-node preemption을 하지 않는다. 따라서 Pod P는 Node N에 뜰 수 없다고 간주된다.

Pod Q가 이 노드에서 삭제가 된다면 Pod anti-affinity violation은 사라지게 될 것이고 Pod P는 아마 Node N에 뜰 수 있게 될 것이다.

충분한 수요가 있고 합리적인 퍼포먼스를 낼 수 있는 알고리즘을 찾게 된다면, 미래의 버전에서는 cross Node preemption이 가능해지도록 고려하고 있다.

## Debugging Pod Priority and Preemption

Pod Priority와 Preemption은 버그를 가지고 있게 될 경우 파드의 스케쥴링을 망가뜨릴 가능성을 가지고 있는 주요 기능이다.

### Potential Problems caused by Priority and Preemption

다음은 기능의 구현에서 생길 수 있는 버그가 야기할 수 있는 문제점들이다. 이 리스트들은 완벽하지 않다.

### Pods are preempted unnecessarily

Preemption은 resource pressure 상황일 때 클러스터에서 현재 동작중인 Pods를 삭제하여 higher priority pending Pods가 뜰 수 있는 공간을 마련한다. 유저가 특정한 파드에 대해서 실수로 높은 priority를 주게 된다면 의도하지않게 priority가 높은 Pod는 클러스터에서 preemption을 발생시킬 것이다. 위에 언급한 것처럼 Pod priority는 `podSpec`의 `priorityClassName`을 설정함으로써 지정된다. priority의 integer value는 해석되어 `podSpec`에 `priority`로 발행된다.

이한 문제를 해결하기 위해 파드의 `priorityClassName`은 lower priority class로 변경되거나 비어있도록 변경되어야 한다. 비어있는 `priorityClassName`은 기본적으로 0으로 풀이된다.

Pod가 preempted될 때 그 이벤트들은 기록이 된다. Preemption은 반드시 클러스터가 Pod에 할당할 충분한 리소스가 없을 때 발생해야 한다. 이러한 경우에 preemption은 pending Pod(preemptor)가 victims Pods보다 higher priority를 가졌을 때에만 발생한다. Preemption은 pending Pods가 없을 때, pending Pods가 victims보다 priority가 같거나 낮을 때에는 반드시 일어나면 안된다. preemption이 이러한 시나리오에서 발생한다면, 이슈를 제기해 주길 바란다.

### Pods are preempted, but the preemptor is not scheduled

파드가 preempted 되었을 때 이들은 기본적으로는 30초로 설정이 되었지만 PodSpec에는 다른 값으로 지정될 수도 있는 graceful termination period를 요청받게 된다.victim Pods가 이 기간 내에 종료되지 않는다면 강제로 종료된다. 모든 victims가 사라지고 나면 preemptor Pod는 스케쥴링 될 수 있다.

preemptor Podrㅏ victims가 사라지기를 기다리는 동안에 higher priority Pod가 생성되어 같은 노드에 뜨기 적합한 상태가 될 수도 있다. 이러한 경우에 scheduler는 preemptor 대신에 higher priority Pod를 띄우게 된다.

이런 higher priority Pod가 없으면 우리는 preemptor Pod가 victims의 graceful termination period 이후에 스케쥴링 될 것이라고 예상해볼 수 있다.

### Higher priority Pods are preempted before lower priority pods

scheduler는 pending Pod가 뜰 수 있는 노드가 없을 때 lower priority를 노드에서 삭제하여 pending Pod를 위한 공간을 마련하게 되면 동작은 할 수 있는 노드를 찾는다. 노드가 low priority Pods를 지워도 pending Pods를 띄울 수 없게 되면 scheduler는 (다른 노드의 Pods와 비교하였을 때) higher priority Pods를 가진 다른 노드를 선택하여 preemption으로 선택한다. victims는 여전히 preemptor Pod보다 낮은 priority를 가지고 있어야 한다.

여러개의 노드가 preemption으로 적합하였을 때 scheduler는 가장 낮은 priority의 Pod 세트를 가진 노드를 선택한다. 하지만 그런 파드가 PodDisruptionBudget을 가지고 있어서 preempted 되었을 때 PDB가 violated 될 수 있다면 scheduler는 higher priority Pods를 가진 다른 노드를 선택할 것이다.

여개의 노드가 preemption의 후보이지만 위의 시나리오중 어느것에도 해당되지 않을 때 우리는 scheduler가 lowest priority의 node를 선택한다고 생각해볼 수 있다. 이 경우가 아니라면 scheduler의 버그이다.

## Interactions of Pod priority and QoS

Pod priority와 QoS는 상호 연관이 거의 없는 orthogonal한 기능이다. QoS class를 기반으로 한 Pod의 priority 설정에는 어떤 기본적인 제약도 없다. scheduler의 preemption logic은 preemption targets를 고를 때 QoS를 고려하지 않는다. Preemption은 Pod priority를 고려하고 lowest priority인 targets의 세트를 선택하려고 한다. Higher-priority Pods는 lowest priority Pods를 지워도 scheduler가 preemptor Pod를 스케쥴링 하기 위해서 충분하지 않을 때나 lowest priority Pods가 `PodDisruptionBudget`에 의해서 보호될 때 고려되는 사항이다.

QoS와 Pod priority를 모두 고려하는 유일한 component는 [Kubelet out-of-resource eviction](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)이다. kubelet은 evction을 위해 Pods에 대하여 첫번째로 starved resource의 사용이 requests를 초과했는지, Pods의 scheduling request에 대한 상대적인 starved compute resource의 소비를 기준으로 순위를 내린다. 자세한 사항은 [Evicting end-user pods](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#evicting-end-user-pods) 참조. kubelet out-of-resource eviction은 사용량이 requests를 초과하지 않은 Pods를 evict하지 않는다. lower priority인 Pod가 requests를 초과하지 않는다면, 이는 evicted 되지 않을 것이다. higher priority를 가진 다른 파드가 requests를 초과하였다면, evicted 될 것이다.

