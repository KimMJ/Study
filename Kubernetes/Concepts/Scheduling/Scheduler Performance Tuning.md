## Scheduler Performance Tuning

**FEATURE STATE: ** `Kubernetes 1.14` - beta

[kube-scheduler](https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/#kube-scheduler)는 Kubernetes의 default scheduler이다. 이는 클러스터 내에서 노드에 대해 파드를 할당하는 역할을 한다.

파드의 scheduling requirements를 만족하는 클러스터 내의 노드는 그 파드에 대해 feasible Nodes라고 불린다. scheduler는 파드에 대해 feasible Nodes를 찾아서 점수를 매기기 위해 함수들을 실행시키고 이 feasible Nodes 중에서 파드를 실행시키기 위한 점수가 가장 높은 노드를 선택한다. 그러면 scheduler는 API server에 Binding이라고 불리는 process에서의 결정을 알린다.

이 페이지는 큰 Kubernetes clusters에서 performance tuning optimizations에 대해 설명할 것이다.

## Percentage of Nodes to Score

Kubernetes 1.12 이전에, kube-scheduler는 클러스터 내에서 가능한 모든 노드들에 대해서 검사를 하고 적합한 것들에 대해 점수를 매겼다. Kubernetes 1.12에서는 scheduler가 일정 수의 feasible nodes를 찾게 되면 더이상 feasible nodes를 검색하는 것을 중단하는 기능을 추가하였다. 이는 large cluster에서의 scheduler's performance를 높인다. 이 숫자는 cluster size의 비율로 지정될 수 있다. 이 비율은 `percentageOfNodesToScore`라고 불리는 configuration option으로 조절이 가능하다. 범위는 1부터 100까지이다. 더 큰 값은 100%로 간주된다. 0은 config option을 제공하지 않는 것과 같다. Kubernetes 1.14에서는 configuration을 따로 지정하지 않았을 때 cluster size를 기반으로 노드의 비율을 찾는 로직을 가지고 있다. 이는 100개의 노드를 가진 클러스터에서는 50%가 되는 선형 식을 사용한다. 이 식은 5000개의 노드를 가진 클러스터에 대해서는 10%이다. automatic value의 lower bound는 5%이다. 다시 말해, scheduler는 클러스터가 얼마나 크던지 유저가 5보다 작은 값으로 config option을 주지 않는 한 항상 클러스터에서 최소 5%의 점수를 확인한다는 것을 의미한다.

아래의 예시는 `percentageOfNodesToScore`를 50%로 설정한 것이다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider

...

percentageOfNodesToScore: 50
```

> Note : feasible nodes가 50개보다 적은 클러스터의 경우 scheduler는 여전히 모든 노드들을 확인할 것이다. scheduler의 검색을 미리 중단시킬 충분한 feasible nodes가 생성되지 않기 때문이다.

**이 기능을 비활성화 하려면** `percentageOfNodesToScore`를 100으로 설정하면 된다.

### Tuning percentageOfNodesToScore

`percentageOfNodesToScore`은 클러스터의 사이즈를 기반으로 계산이 된 default value로 반드시 1에서 100사이의 값이어야 한다. 또한 50개의 노드에 대해서 하드코딩된 최소값이 존재한다. 이는 이 옵션을 수백개의 노드를 가진 클러스터에서 더 작은 값으로 변경할 경우 스케쥴러가 검색하는 feasible nodes의 수에 크게 영향을 미치지는 않는다는 것을 의미한다. 이는 이 옵션이 소규모의 클러스터에서는 퍼포먼스를 눈에띄게 증가시키지 않을 것이기 때문에 의도한 것이다. 1000개의 노드가 넘는 대규모의 클러스터에서 이 값을 작은 수로 변경하는 것은 퍼포먼스를 눈에띄게 증가시킬 수 있다.

클러스터에서 feasibility를 검사하는 노드의 수를 줄일 때 설정할 값을 고려할 때 주의깊게 생각해야할 점은 몇몇 노드들은 주어진 파드에 대해서 점수를 제공하지 않게될 것이라는 점이다. 결과적으로 그 파드에 대해서 더 높은 점수로 파드를 실행시킬 가능성이 있는 노드들이 scoring phase마저도 통과하지 않는다는 이야기가 된다. 이는 파드의 위치선정에서 이상적인 상황은 아닐 수 있다. 이러한 이유로 value는 너무 낮은 비율로 설정되어서는 안된다. 일반적으로 대두되는 방법은 10보다 작은 값으로는 설정하지 않는 것이다. 더 작은 값은 스케쥴러의 throughput이 application에 매우 중요하고, 노드의 점수들이 중요하지 않은 경우에만 사용되어야 한다. 다시말해, 사용자는 feasible인 아무 노드에서나 파드를 실행시키고자 하는 경우이다.

클러스터가 수백개의 노드이거나 더 적은 수의 노드를 가진다면, 이 configuration option에서 default value를 줄이는 것을 추천하지는 않는다. 이는 스케쥴러의 퍼포먼스를 거의 증가시키지 않는다.

### How the scheduler iterates over Nodes

이번 섹션은 이 기능에 대해서 더 자세하게 이해하고 싶은 사람들을 위한 것이다.

클러스터에서 모든 노드들에게 파드를 실행시킬 수 있는지를 고려할 동일한 기회를 주기 위해, 스케쥴러는 모든 노드들을 라운드 로빈 방식으로 돈다. 노드들이 배열로 있다고 상상해도 된다. 스케쥴러는 배열의 처음부터 node의  `percentageOfNodesToScore`를 만족할만한 충분한 수의 노드를 찾을 때까지 feasibility를 체크한다. 다음 파드에 대해서 스케쥴러는 노드 배열에서 이전 파드가 노드의 feasibility를 체크할 때 멈춘 부분부터 이어서 검사하기 시작한다.

노드들이 multiple zone에 있다면 스케쥴러는 다양한 zone에 있는 노드들을 번갈아 순회한다. 이를 통해 다른 zone에 있는 노드들도 feasibility를 검사했음을 보장한다. 예를 들어 2개의 zone에 6개의 노드가 있다고 생각해보자.

```
Zone 1: Node 1, Node 2, Node 3, Node 4
Zone 2: Node 5, Node 6
```

스케쥴러는 다음과 같은 순서로 feasibility를 검사한다.

```
Node 1, Node 5, Node 2, Node 6, Node 3, Node 4
```

모든 노드를 순회하였다면 다시 Node 1으로 돌아간다.