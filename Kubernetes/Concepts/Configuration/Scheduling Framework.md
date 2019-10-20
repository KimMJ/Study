# Scheduling Framework

**FEATURE STATE: ** `Kubernetes 1.15` - alpha

scheduling framework은 Kubernetes Scheduler에 대한 새로운 plugable architecture이다. 이는 scheduler customizations를 쉽게 만들어준다. 이것은 현존하는 scheduler에 새로운 "plugin" APIs 세트를 추가한다. 플러그인은 scheduler과 합쳐진다. APIs는 대부분의 scheduling feature을 plugins로 구현될 수 있도록 해준다. 하지만 scheduling의 "core" 부분은 간단하고 유지가능하도록 남겨두었다.  [design proposal of the scheduling framework](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/20180409-scheduling-framework.md)을 참조하여 framework의 design에 대한 technical information을 확인해 보아라.

## Framework workflow

Scheduling Framework는 몇가지 extension points를 정의한다. Scheduler plugins는 하나 이상의 extension points를 적용시킨다. 이런 몇몇 plugins는 scheduling decisions를 바꾸기도 하고 어떤 plugins는 단순히 정보만을 제공하기도 한다.

하나의 파드에 대한 schedule의 각 시도는 두가지 phase로 나눌 수 있다: **scheduling cycle**, **binding cycle**

## Scheduling Cycle & Binding Cycle

scheduling cycle은 파드를 띄울 노드를 선택하고 binding cycle은 cluster에 결정을 적용시키는 것이다. scheduling cycle과 binding cycle은 함께 "scheduling context"라고 불린다.

Scheduling cycle는 연속적인 절차로 작동하지만 Binding cycle은 동시에 작동한다.

scheduling 또는 binding cycle은 파드가 unschedulable이라고 결정되거나 내부적인 에러가 있다면 버려질 수 있다. 파드는 queue로 돌아가고 재시도될 것이다.

## Extension points

다음의 그림은 파드의 scheduling context와 scheduling framework가 expose하는 extension points를 보여준다. 이 그림에서 "Filter"는 "Predicate" 와 같고, "Scoring"은 "Priority function"과 같다.

하나의 plugin은 더 복잡하거나 stateful tasks를 실행시키기 위해 여러개의 extension points를 등록할 수 있다.

![scheduling-framework-extensions](img/scheduling-framework-extensions.png)

<scheduling framework extension points>

### Queue sort

이 plugins는 scheduling queue에서 Pods를 정렬하는 데 사용된다. queue sort plugin은 반드시 "less(Pod1, Pod2)" 함수를 제공해야 한다. 동시에 하나의 queue sort plugin만 지원이 가능하다.

### Pre-filter

이 plugins는 Pod에 대한 info를 pre-process하는데 사용되거나 클러스터 또는 파드가 반드시 만족해야 하는 조건을 체크하는 데 쓰인다. 만약 pre-filter plugin이 에러를 리턴하면, scheduling cycle은 버려진다.

### Filter

이 plugins는 Pod를 실행시킬 수 없는 노드를 배제시키는 데 사용한다. 각 노드에서 scheduler는 설정한 순서에 따라서 filter plugins를 호출할 것이다. 어떤 filter라도 그 노드가 부적합하다고 판단하면 그 노드에 대해 나머지 plugins는 호출되지도 않을 것이다. 노드는 동시에 평가될 수도 있다.

### Post-filter

이는 informational extension point이다. Plugins는 filtering phase를 지나간 노드들의 리스트에서 호출될 수 있다. plugin은 이 데이터를 internal state를 업데이트 하거나 logs/metrics를 생성하는 데 사용될 수 있다.

> Note : "pre-scoring" 동작이 실행되기를 원하는 plugins는 반드시 post-filter extension point를 사용해야 한다.

### Scoring

이 plugins는 filtering phase를 통과한 노드들에 순위를 매기는 데 사용된다. scheduler는 각 노드들에 대해서 각각의 scoring plugin을 호출할 것이다. 최소 score와 최대 score의 well defined range가 정해져 있다. [normalize scoring](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#normalize-scoring)가 지나고 나면 scheduler는 모든 plugins에서 score를 각 plugin마다 설정된 weights에 따라 합칠 것이다.

### Normalize scoring

이 plugins는 scheduler가 최종적인 노드의 순위를 계산하기 전에 score를 수정하는 데 사용된다. 이 extension point로 등록된 plugin은 같은 plugin으로부터 scoring results와 함께 호출이 될 것이다. 이는 plugin마다 scheduling cycle마다 호출될 것이다.

예를 들어, plugin `BlinkingLightScorer`가 노드들을 얼마나 많은 blinking lights를 가지고 있는지에 따라 순위를 매긴다고 가정해보자.

```go
func ScoreNode(_ *v1.pod, n *v1.Node) (int, error) {
   return getBlinkingLightCount(n)
}
```

하지만, 최대 blinking lights 수는 `NodeScoreMax`보다 작을 것이다. 이를 해결하기 위해 `BlinkingLightScorer`는 이 extension point 또한 등록해야 한다.

```go
func NormalizeScores(scores map[string]int) {
   highest := 0
   for _, score := range scores {
      highest = max(highest, score)
   }
   for node, score := range scores {
      scores[node] = score*NodeScoreMax/highest
   }
}
```

normalize-scoring plugin이 에러를 리턴하게 되면 scheduling cycle은 버려진다.

> Note : "pre-reserve" 동작을 실행하고 싶어하는 Plugins는 normalize-scoring extension point를 사용하면 된다.

### Reserve

이는 informational extension point이다. runtime state을 유지하는 Plugins("stateful plugins"라고도 불린다)는 이 extension point를 사용하여 scheduler가 노드의 리소스가 주어진 Pod에 대해서 reserved되었을 때 알림을 받을 수 있도록 해야한다. 이는 scheduler가 실제로 파드를 노드에 bind하기 전에 발생하며, scheduler가 bind가 성공되는 것을 기다리는 동안 race conditions를 방지하기 위해서 존재한다. 

이는 scheduling cycle의 마지막 단계이다. 파드가 한번 reserved state로 들어가면 이는 binding cycle의 마지막에서 [Unreserve](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#unreserve) plugins(on failure) 또는 [Post-bind](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#post-bind) plugins(on success)를 트리거한다.

> Note : 이 컨셉은 보통 "assume"이라고 불린다.

### Permit

이러한 plugins는 파드의 binding을 막거나 지연시키는 데 사용한다. permit plugin은 다음의 셋 중 하나를 할 수 있다.

1. **approve**
   모든 permit plugins가 파드를 승인하면, 이는 binding으로 보내진다.
2. **deny**
   permit plugins중 하나라도 파드를 거부하면 이는 scheduling queue로 돌려보낸다. 이는 [Unreserve](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#unreserve) plugins를 트리거링 할 것이다.
3. **wait** (with a timeout)
   permit plugin이 "wait"을 리턴하게 되면 파드는 [plugin이 이를 승인할 때](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#frameworkhandle)까지 permit phase를 유지한다. timeout이 발생하면 **wait**은 **deny**가 되고 파드는 scheduling queue로 돌아가며  [Unreserve](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#unreserve) plugins을 트리거링 한다.

#### Approving a Pod binding

어느 plugin이라도 cache를 통해서 "waiting" 파드의 리스트에 접근할 수 있고 이를 승인한다면 ([`FrameworkHandle`](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#frameworkhandle) 참고) 우리는 단 하나의 permit plugins만 "waiting" state에 있는 reserved Pods에 대한 binding을 승인할 수 있다고 생각할 수 있다. 한번 파드가 승인되면 이는 pre-bind phase로 전송된다.

### Pre-bind

이 plugins는 파드가 bound 되기 전에 필요시 되는 어떤 작업이라도 수행할 때 사용된다. 예를 들어, pre-bind plugin은 network volume을 provision할 수 있고, 이를 파드가 거기서 작동하도록 허락하기 전에 target node에 mount할 수 있을 것이다.

어떤 pre-bind plugin이라도 에러를 리턴하면, 파드는  [rejected](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#unreserve)되고 scheduling queue로 돌아간다.

### Bind

이 plugins는 노드에 파드를 bind할 때 사용된다. bind plugins는 모든 pre-bind plugins들이 완료될 때까지 호출되지 않을 것이다. 각 bind plugin은 설정한 순서에 따라 호출된다. bind plugin은 주어진 파드를 다룰 수 있는지, 없는지를 선택한다. bind plugin이 파드를 다루기로 하였다면 **나머지 bind plugins는 스킵된다**.

### Post-bind

이는 informational extension point이다. post-bind plugins는 파드가 성공적으로 bound 되고 나서 호출된다. 이는 binding cycle의 마지막이며, 관련된 리소스들을 정리하는 데 사용된다.

### Unreserve

이는 informational extension point이다. 파드가 reserved되고 나중의 phase에서 rejected되면, unreserve plugins는 알림을 받게 된다. unreserve plugins는 reserved Pod와 연관된 state를 제거할 것이다.

이런 extension point를 사용하는 Plugins는 [Reserve](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/#reserve)도 사용하게 될 것이다.

## Plugin API

plugin API에는 두가지 단계가 있다. 먼저, plugins는 반드시 등록되고 설정되어야 하고, 그 다음에 extension point interfaces를 사용해야 한다. Extension point interfaces는 다음과 같은 형태를 띄고 있다.

```go
type Plugin interface {
   Name() string
}

type QueueSortPlugin interface {
   Plugin
   Less(*v1.pod, *v1.pod) bool
}

type PreFilterPlugin interface {
   Plugin
   PreFilter(PluginContext, *v1.pod) error
}

// ...
```

## Plugin Configuration

plugins는 scheduler configuration에서 활성화 될 수 있다. 또한 default plugins는 configuration에서 비활성화 될 수 있다. 1.15 버전에서는 scheduling framework에 대한 default plugins가 없다.

scheduler configuration은 plugins에 대한 configuration 또한 포함될 수 있다. 이런 configurations는 scheduler가 이들을 초기화 할 때 plugins로 전달이 된다. 이 configuration은 임의의 값이다. receiving plugin은 반드시 configuration을 decode하고 처리해야 한다.

다음의 예시는 `reserve`와 `preBind` extension points에서 몇가지 plugins를 활성화 한 것이고, 몇가지는 비활성화 한 scheduler configuration을 보여준다. 이는 또한 plugin `foo`라는 configuration을 제공한다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration

...

plugins:
  reserve:
    enabled:
    - name: foo
    - name: bar
    disabled:
    - name: baz
  preBind:
    enabled:
    - name: foo
    disabled:
    - name: baz
    
pluginConfig:
- name: foo
  args: >
    Arbitrary set of args to plugin foo
```

extension point가 configuration에서 생략되면 그 extension point에 대해 default plugins가 사용된다. extension point가 존재하고 `enabled`가 제공되면 `enabled` plugins는 default plugins를 추가할 때 호출될 수 있다. default plugins는 먼저 호출되고 나머지 추가적인 활성화된 plugins는 설정에 지정된 순서와 동일한 순서로 호출된다. 만약 default plugins를 다른 순서로 호출하기를 원한다면 default plugins는 반드시 `disabled`되고 원하는 순서에서 `enabled` 되어야 한다.

`reserve`에서 `foo`라는 default plugin을 호출한다고 가정하고 `foo`를 발생시키기 전에 plugin `bar`를 호출하고 싶다고 한다면, `foo`를 비활성화 하고 `bar`와 `foo`를 순서대로 호출시키면 된다. 다음의 예시는 이를 이용한 configuration을 보여준다:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration

...

plugins:
  reserve:
    enabled:
    - name: bar
    - name: foo
    disabled:
    - name: foo
```

