# StatefulSets

* StatefulSet은 stateful application을 고나리하기 위한 workload API object이다.
* deployment와 pod의 scaling을 관리하고 이런 pod에 대해서 ordering과 uniqueness를 보장한다.
* Deployment처럼 StatefulSet은 container spec과 일치하는 파드를 관리한다.
* Deployment와는 다르게 StatefulSet은 각각의 파드에 sticky identity를 유지한다.
* 이런 파드들은 같은 스펙으로 생성이 되지만 교환돌 수 없다.
  * 각각은 rescheduling이 되는 동안에도 영구적인 ID를 가지고 있는다.
* StatefulSet은 다른 Controller와 같은 패턴으로 동작한다.
  * StatefulSet object에 원하는 상태를 기술하고, StatefulSet controller가 현재 상태에서 그 상태로 필요시 업데이트 해준다.

## Using StatefulSets

* StatefulSet은 다음과 같은 상황에서 쓰인다.
  * Stable, 유일한 네트워크 ID
  * Stable, 영구 스토리지
  * Ordered, 점진적인 deployment와 scaling
  * Ordered, 자동 rolling 업데이트
* stable : Pod의 (re)scheduling에도 지속적
* application이 stable ID나 ordered deployment, deletion, scaling이 필요하지 않다면 application을 stateless replica로 구성해야 한다.
* Deployment나 ReplicaSet이 stateless에 더 알맞다.

## Limitations

* StatefulSet은 1.9 버전 이전에는 beta였고 1.5 이전에는 사용 불가능하다.
* 파드의 storage는 필요한 `storage class` PersistentVolume Provisioner에 의해서 준비되거나 관리자에 의해서 준비되어야 한다.
* StatefulSet을 삭제하거나 scaling down하는 것은 StatefulSet과 관련된 volume을 삭제하지 않는다.
* StatefulSet은 파드의 network identity를 위한 Headless Service가 필요하다. 이 Service를 생성해 주어야 한다.
* StatefulSEt은 StatefulSet이 삭제될 때 파드의 종료를 제공하지 않는다. 파드의 ordered, graceful termination을 원하면 StatefulSet을 0으로 scale down하고 제거하라.
* default Pod Management Policy(`OrderedReady`)로 Rolling Update를 할 때 수리를 위해 manual intervention으로 broken state에 진입이 가능하다.

## Components

* Headless Service
* StatefulSet
* volumeClaimTemplates

## Pod Selector

* StatefulSet의 `.spec.selector`필드를 `.spec.template.metadata.labels`와 일치시켜야 한다.
* 1.8 이전 버전에서는 `.spec.selector`필드가 생략되면 디폴트로 채웠다.
* 1.8부터는 일치하는 pod selector가 없다면 StatefulSet 생성 시 validiation error를 일으킨다.

## Pod Identity

* StatefulSet 파드는 유일한 ID를 가지고 있어서 ordinal, stable network identity, stable storage에 구성된다.

### Ordinal Index

* N개의 replica를 가진 StatefulSet에서 각각의 파드는 순차적으로 숫자를 부여받는다. (0 ~ N-1). 이는 셋에서 유일하다.

### Stable Network ID

* StatefulSet의 각 파드는 hostname을 StatefulSet의 이름과 pod의 ordinal을 통해서 가져온다.
* hostname은 `$(statefulset name)-$(ordinal)`의 형식으로 구성된다.
* StatefulSet은 pod의 도메인을 관리하기 위해 Headless Service를 쓸 수 있다.
* 이 서비스에 의해 manage되는 도메인은 다음과 같은 형식이다. `$(service name).$(namespace).svc.cluster.local`
  * 여기서 `cluster.local`은 cluster domain이다.
* 각각의 파드가 생성이 되면 DNS subdomain을 매칭시키고 다음과 같은 형식을 따른다. `$(podname).$(governing service domain)`
  * 여기서 governing service는 StatefulSet의 `serviceName`필드에 정의되어 있다.
* limitation section에서, 파드의 network identity를 위해서 Headless Service를 생성해야 한다고 했다.

### Stable Storage

* 쿠버네티스는 각 VolumeClaimTemplate마다 하나의 PersistentVolume을 생성한다. 
* StorageClass가 정해지지 않으면, 디폴트 StorageClass가 사용될 것이다.
* 파드가 노드에 (re)schedule되면, `volumeMounts`가 PersistentVolumes를 PersistentVolume Claims로 마운트한다.
* 주의 : 파드의 PersistentVolume Claims와 관련된 PersistentVolumes는 파드, StatefulSet이 지워질 때 함께 지워지지 않는다. 이는 반드시 수동으로 삭제해야 한다.

### Pod Name Label

* StatefulSet controller가 파드를 생성하면 `statefulset.kubernetes.io/pod-name`라벨을 생성한다.
* 이 라벨은 Service를 StatefulSet의 특정한 파드로 연결되도록 한다.

## Deployment and Scaling Guarantees

* N개의 replicas를 가지고 있는 StatefulSet은 Pod가 deploy될 때 0 ... N-1로 순서대로 생성된다.
* 파드가 삭제될 때, N-1 ... 0의 순서로 거꾸로 삭제된다.
* 파드에 scaling operation이 적용되기 전에 모든 predecessors는 Running이고 Ready여야 한다.
* 파드가 종료되기 전에 successors는 반드시 완전히 꺼져야 한다.
* StatefulSet은 `pod.spec.TerminationGracePeriodSEconds`에 0지정하지 않아야 한다. 이 방식은 안전하지 않고 강력히 비추천한다.

### Pod Management Policies

* 쿠버네티스 1.7버전 이후에는 `.spec.podManagementPolicy`필드를 통해서 uniqueness와 identity를 보장하면서 ordering guarantee를 완화할 수 있다.

#### OrderedReadyPodManagement

* `OrderedReady`파트 매니지먼트는 StatefulSet의 디폴트이다.
* 이는 위에 나왔던 behavior로 작동한다.

#### Parallel Pod Management

* `Parallel`파트 매니지먼트는 StatefulSet controller에 모든 파트가 parallel하게 시작되거나 종료됨을 알려주고 파드가 Running & Ready가 되거나 다른 파드가 launching 또는 terminting하기 전에 완전히 종료되는 것을 기다려 주지 않는다.
* 이 옵션은 scaling operation에 영향을 준다.
* 업데이트는 적용되지 않는다.

## Update Strategies

* 쿠버네티스 1.7버전과 이후에서는 StatefulSet의 `.spec.updateStrategy`필드를 통해 사용자가 StatefulSet에서 컨테이너, 라벨, 리소스 request/limits, 파드의 annotation에 대한 automated rolling update를 설정하고 disable할 수 있다.

### On Delete

* `OnDelete`는 legacy이다. (1.6및 이전버전)
* StatefulSet의 `.spec.updateStrategy.type`이 `onDelete`로 설정이 되어있으면 StatefulSet controller는 StatefulSet의 파트를 자동으로 업데이트 하지 않는다.
* 사용자는 반드시 수동으로 파드를 삭제시켜서 controller가 새로운 파드를 만들고 그 변경사항이 StatefulSet의 `.spec.template`에 반영되도록 해야한다.

### Rolling Updates

* `RollingUpdate`는 StatefulSet의 파드를 rolling update를 자동화 한 update strategy이다.
* `.spec.updateStrategy`가 정의되지 않았으면 Rolling Update가 default이다.
* StatefulSet의 `.spec.updateStrategy.type`이 `RollingUpdate`로 설정이 되었으면, StatefulSet controller는 StatefulSet의 각 파드를 지우고 재생성 할 것이다.
* 이는 파드의 종료와 같은 순서로 진행이 되고(숫자가 큰 것에서부터 작은것 순으로) 동시에 각 파드를 업데이트 한다.
* predecessor가 업데이트 되기 전에 파드가 Running & Ready가 될때까지 기다린다.

#### Partitions

* `RollingUpdate`는 `.spec.updateStrategy.rollingUpdate.partition`에 명시되는 부분화 update strategy이다.
* partition으로 정해지면 모든 순서가 있는 파드 중 partition보다 크거나 같은 파드들은 StatefulSet의 `.spec.template`이 업데이트가 되면 업데이트가 된다.
* partition보다 작은 순서를 가진 pod들은 삭제가 되었다고 하더라도 이전 버전으로 재생성된다.
  * 만약 StatefulSet의 `.spec.updateStrategy.rollingUpdate.partition`이 `.spec.replicas`보다 크다면, `.spec.template`의 변경사항은 파드에 전달되지 않을 것이다.
* 대부분의 경우에 이 partition을 사용할 일은 없을 것이다. 하지만 stage an update, roll out a canary, perform a phased roll out의 상호아에서는 유용할 수 있다.

#### Forced Rollback

* default Pod Management Policy(`OrderedReady`)로 Rolling Update를 사용한다면, 수동으로 점검을 위해서 관리할 필요가 있을 때 state에 들어갈 수 있다.
* Running이나 Ready가 절대 되지 않는 configuration에 파드 template을 업데이트하면 StatefulSEt은 rollout을 멈추고 기다릴 것이다.
* 이 상태에서 Pod template을 좋은 configuration으로 교체하는 것은 어렵다.
  * StatefulSet은 동작하는 configuration으로 되돌리기 전에 망가진 파드가 Ready가 될때까지(절대 되지 않을 상태) 기다릴 것이다.
* 템플릿을 되돌리고 나면 사용자는 반드시 StatefulSet이 bad configuration으로 동작하고 있던 파드들을 지워주어야 한다.
* StatefulSet은 그 후에 파드를 되돌린 template으로 재생성할 것이다.

## 