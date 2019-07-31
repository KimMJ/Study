# ReplicaSet



## How a ReplicaSet works

* 어떻게 Pod를 얻을지, replica의 개수는 몇개가 유지되어야 하는지, 그 기준에 맞으려면 복제를 해야하는지를 포함한다.
* 생성, 삭제를 통해 그 목적을 달성
* 새로운 파드를 생성할 때, 파드 템플릿을 사용
* ReplicaSet을 통해 생성된 파드는 ownerReference가 ReplicaSet이다.
* 파드가 OwnerReference가 없거나 컨트롤러가 아니면서 ReplicaSet의 selector와 일치하면 그 ReplicaSet으로 획득된다.

## When to use a ReplicaSet

* 직접 ReplicaSet을 사용하는 것보다 Deployments를 사용하는 것을 추천.
  * custom update를 하거나 update를 하지 않는다면 사용해도 된다.

## Non-Template Pod acquisitions

* bare Pod를 문제없이 생성했다면, label이 ReplicaSet과 일치하지 않도록 해야한다.
  * ReplicaSet이 Template에 적힌 것만 획득하는 것이 아니라 이전의 section에서도 획득하기 때문.

## Writing a ReplicaSet manifest

* `apiVersion`, `kind`, `metadata` 필드 필요

### Pod Template

* `restartPolicy`는 Always만 가능, default

### Pod Selector

* ReplicaSet에서 `.spec.template.metadata.labels`는  `spec.selctor`와 일치해야한다. 아니면 API에서 거절됨.

### Replicas

* `.spec.replicas`로 얼마나 많은 pod가 동시에 실행될지 결정가능.
* default는 1

## Working with ReplicaSets

### Deleting a ReplicaSet and its Pods

* `kubectl delete`로 삭제
* `Garbage collector`가 삭제할 것이다.

### Deleting just a ReplicaSet

* `kubectl delete`를 `--cascade=false`옵션을 줘서 ReplicaSet을 지울 수 있다.
* 삭제하고 나면, 이를 대체할 ReplicaSet을 생성할 수 있다.
* `.spec.selector`가 같다면, 새로운 ReplicaSet은 예전 파드를 획득할 것이다.
* 그러나 기존의 파드가 새로운 템플릿에 맞춰지도록 하지는 않을 것이다.
* 파드를 새로운 스펙으로 변경하고 싶으면 `Deployment`를 사용해라.
* ReplicaSet은 Rolling update를 지원하지 않는다.

### Isolating Pods from a ReplicaSet

* 파드를 라벨을 바꿔 ReplicaSet에서 지울 수 있다.
* 디버깅이나 데이터 복구 등의 이유로 서비스에서 파드를 지울 때 사용
* 이 방식으로 지워진 파드는 자동으로 대체된다.

### Scaling a ReplicaSet

* `.spec.replicas`필드를 업데이트해서 scale up, down이 쉽게 가능

### ReplicaSet as a Horizontal Pod Autoscaler Target

* ReplicaSet은 `Horizontal Pod Autoscalers(HPA)`로 사용될 수 있다.
* 즉, ReplicaSet은 HPA에 의해 auto-scaled된다.
* `kubectl autoscale rs frontend --max=10`로도 가능

## Alternatives to ReplicaSet

### Deployment (추천)

* Deployment는 RelicaSets를 소유할 수 있고 ReplicaSet과 Pod를 선언적으로 업데이트 할 수 있고 서버사이드의 rolling update를 지원한다.
* ReplicaSet을 따로 할 수도 있지만 보통 Deployment쓴다
* Deployment를 쓰면 ReplicaSet에 대해 고민할 필요 없다.
* Deployment가 ReplicaSet을 소유하고 관리한다.
* 따라서 ReplicaSet쓸거면 Deployment를 추천한다.

### Bare Pods

* 유저가 직접 파드를 생성하는 케이스와는 다르게 ReplicaSet은 삭제되거나 종료된 파드를 대체한다.
  * 커널 업그레이드같은 노드의 failure나 disruptive node maintenance 등 상황에서도.
* 그렇기 때문에 하나의 파드만 생성하더라고 ReplicaSet을 쓰기를 권유한다.

### Job

* 스스로 종료되는 파드에 대해서 (batch jobs) Job을 써라.

### DaemonSet

* machine monitoringㅇ나 machine logging같은 machine-level function을 제공하는 파드에 대해서는 ReplicaSet보다는 DaemonSet을 써라.
* 이런 파드들은 machine의 lifetime에 종속되어있다.
  * 파드는 다른 파드가 시작하기 전에 머신에서 실행되어야 하고 reboot, shudown할 때 안전하게 종료되어야 한다.

### ReplicationController

* ReplicaSet은 ReplicationController의 상속자이다.
* 같은 목적으로 제공되고 비슷하게 동작한다.
* ReplicationController는 set-based selector를 지원하지 않는다.
  * 따라서 ReplicaSet이 더 선호된다.

