# DaemonSet

* DaemonSet은 모든 노드 또는 몇몇 노드가 파드의 복제본을 동작하도록 하는 것이다.
* 노드가 cluster에 추가될 때, 파드도 거기에 추가된다.
* 노드가 cluster에서 삭제될 때, 파드도 garbage collected 된다.
* DaemonSet을 지우는 것은 생성한 파드를 지운다.
* DaemonSet이 사용되는 상황
  * `glusterd`, `ceph`같은 cluster storage daemon을 각각의 노드에 실행시킨다.
  * 모든 노드에 `fluentd`, `logstash`같은 log collection을 실행시킨다.
  * 모든 노드에 `Prometheus Node Exprter`, `Sysdig Agent`, `collectd`,` Dynatrace OneAgent`, `AppDynamics Agent`, `Datadog agent`, New Relic agent`, `Ganglia gmond`, `Instana Agent`같은 node monitoring을 실행시킨다.
* 모든 daemon의 타입에서 하나의 DaemonSet이 모든 node를 커버하는 간단한 케이스가 사용될 것이다.
* 더 복잡한 셋업은 daemon의 한 타입에 여러개의 플래그가 다르거나 다른 하드웨어 타입에 따라 메모리 또는 cpu의 request가 다른 DaemonSet을 사용하는 것이다.

## Writing a DaemonSet Spec

### Create a DaemonSet

* YAML 파일로 DaemonSet을 정의할 수 있다.
* YAML 파일 기반으로 DaemonSet 생성
  `kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml`

### Required Fields

* 다른 kubernetes config처럼, DaemonSet은 `apiVersion`, `kind`, `metadata` 필드를 필요로 한다.
* 또한 `.spec` 섹션을 필요로 한다.

### Pod Template

* `.spec.template`은 `.spec`필드에서 required field중 하나다.
* `.spec.template`은 파드 템플릿이다. `apiVersion`이나 `kind`를 가지지 않고 nested된 것을 빼면 정확히 pod와 같은 스키마를 가지고 있다.
* DaemonSet의 Pod Template은 `RestartPolicy`가 `Always`여야 한다. 명시되지 않을 경우 기본적으로 `Always`이다.

### Pod Selector

* `.spec.selector`필드는 파드 셀렉터이다. Job에서의 `.spec.selector`와 똑같이 동작한다.
* 쿠버네티스 1.8 버전에서 사용자는 `.spec.template`과 일치하는 파드 셀렉터를 정해주어야 한다. 비었다고 default를 넣어주지 않는다.
* Selector의 defaulting은 `kubectl apply`와 호환되지 않는다.
* DaemonSet이 생성되면 `.spec.selctor`는 수정될 수 없다.
  * pod selector의 수정은 이상한 동작을 할 수 있으며 사용자가 헷갈리게 된다.
* `.spec.selector`는 두개의 필드로 나뉜다.
  * `matchLabels` - ReplicationController의 `.spec.selector`와 똑같이 동작한다.
  * `matchExpressions` - 키, 벨류 배열을 이용하여 더 복잡한 selector를 만들 수 있게 한다.
  * 둘 다 적히면 AND로 처리된다.
* `.spec.selector`가 적힐 경우 `.spec.template.metadata.labels`와 일치해야 한다. 일치하지 않을 경우 API에 의해 거절된다.
* 또한 이 selector와 라벨이 일치하는 파드를 직접, 다른 Daemonset을 통해서, ReplicaSet같은 다른 controller를 통해서 만들어서는 안된다. 
  * 그렇지 않으면 DaemonSet은 그 파드가 DaemonSet에 의해 생성된 것으로 생각한다.
  * 그런 상황은 테스트 용도로만 써라.

### Running Pods on Only Some Nodes

* `.spec.template.spec.nodeSelector`를 적었을 경우 DaemonSet controller는 node selector와 일치하는 노드에 파드를 생성할 것이다.
* 비슷하게 `.spec.template.spec.affinity`를 설정할 경우 DaemonSet controller는 node affinity와 일치하는 노드에 파드를 생성할 것이다.
* 둘다 설정하지 않았다면 DaemonSet controller는 모든 노드에 파드를 생성할 것이다.

## How Daemon  Pods are Scheduled

### Scheduled by DaemonSet controller (disabled by default since 1.12)

* 보통 파드가 동작하고 있는 머신은 kubernetes scheduler에 의해 선택된다. 하지만 DaemonSet controller에 의해 생성된 파드는 이미 머신이 선택되어있다. (`.spec.nodeName`이 파드가 생성될 때 설정되어 해당 파드는 schedular에 의해 무시된다.)
  * 노드의 `unschedulable`필드는 DaemonSet controller에서 무시된다.
  * DaemonSet controller는 scheduler가 시작되기 전에 파드를 만들 수 있다. 이를 통해 cluster bootstrap에 도움을 준다.

### Scheduled by DaemonSet controller (enabled by default since 1.12)

**FEATURE STATE: `Kubernetes v1.15` - beta**

* DaemonSet은 모든 가능한 노드가 pod의 복사본을 실행할 수 있도록 한다.
* 보통 파드가 동작하는 노드는 Kubernetes scheduler에 의해서 결정된다.
* 하지만 DaemonSet 파드는 DaemonSet controller가 대신해서 생성하고 스케쥴링한다. 이는 다음과 같은 이슈가 있다.
  * Inconsistent Pod behavior : 보통의 파드는 `Pending`상태에서 스케쥴링과 생성되기를 기다리지만 DaemonSet 파드는 `Pending`상태로 생성되지 않는다. 이는 유저에게 혼란을 준다.
  * Pod preemption은 default schedular에 의해서 관리된다. preemption이 가능하면 DaemonSet controller는 pod priority와 preemption을 고려하지 않고 스케쥴링을 결정할 것이다.
* `ScheduleDaemonSetPods`는 DaemonSet pod에 `.spec.nodeName`대신에 `NodeAffinity`를 추가함으로써 DaemonSet이 DaemonSet controller가 아니라 default controller에 의해 스케쥴 되도록 한다.
* default scheduler는 타겟이 되는 호스트에 파드를 바인딩한다.
* DaemonSet파드의 만약 node affinity가 이미 존재하다면 교체된다.
* DaemonSet controller는 DaemonSet 파드를 생성하거나 수정할 때만 이런 동작을 수행하고 DaemonSet의 `spec.template`은 수정되지 않는다.
* 추가로 `node.kubernetes.io/unschedulable:NoSchedule`이 자동으로 DaemonSet 파드에 추가된다. default scheduler는 DaemonSet 파드를 스케쥴링 할 때 `unschedulable` 노드를 무시한다.

### Taints and Tolerations

* 문서 참조

## Communicating with Daemon Pods

* DaemonSet에 있는 파드들간의 통신에는 몇가지 가능한 패턴이 있다.
  * Push : DaemonSet의 파드가 다른 통계 데이터베이스같은 서비스들에 업데이트하도록 설정되어 있다.
  * NodeIP and Known Port : DaemonSet안에 있는 파드는 `hostPort`를 사용할 수 있어서 파드가 node IP를 통해서 연결될 수 있다. 클라이언트는 node IP와 포트를 알 수 있다.
  * DNS : headless service를 같은 pod select로 생성하여 DaemonSet을 `endpoints` 리소스를 사용하거나 DNS의 기록을 통해서 discover될 수 있다.
  * Service : 같은 Pod selector로 서비스를 생성하고 서비스를 이용하여 임의의 노드에 있는 daemon에 연결할 수 있다.(특정한 node를 고를 수 없다)

## Updating a DaemonSet

* 노드의 라벨이 변경되면 DaemonSet은 즉시 새로 매치되는 노드에 파드를 생성하고 새로 매치되지 않는 노드에 대해서 파드를 지운다.
* DaemonSet이 생성한 파드를 수정할 수 있다. 하지만 파드는 모든 필드에 대해 업데이트 되지 않는다. 또한 DaemonSet controller는 다음에 노드가 생성될 때에도 기존의 template을 이용할 것이다.
* DaemonSet을 지울 수 있다. 만약 `kubectl`로 `--cascade=false`를 설정하면 파드는 노드에서 사라진다. 그럴 경우 다른 template을 가진 새로운 DaemonSet을 생성할 수 있다.
* 다른 template을 가지는 새로운 DaemonSet은 라벨과 매치되는 현존하는 파드를 인식한다. 하지만 pod template이 다르다고 해서 수정하거나 삭제하지 않는다. 강제로 파드를 지우거나 노드를 지워서 새로운 파드가 생성되도록 해야한다.
* kubernetes 1.6버전부터 DaemonSet을 통해서 rolling update를 수행할 수 있다.

## Alternatives to DaemonSet

### Init Scripts

* 직접 `init`, `upstartd`, `systemd`의 명령어등을 사용해서 노드에 직접 daemon process를 실행시킬 수 있다.
* 그러나 DaemonSet을 사용하면 더 이점이 많다.
  * application과 같은 방식으로 daemon을 모니터하고 로그를 관리할 수 있다.
  * 같은 config 언어와 툴로 (Pod template의 `kubectl`) daemon과 application에 사용할 수 있다.
  * 데몬을 제한된 자원을 가진 컨테이너에 실행시켜서 daemon과 app container를 격리성을 증가시킬 수 있다. 이는 daemon을 도커로 직접 실행시키는 것 처럼 파드에서가 아니라 container에서 동작하도록 해도 똑같이 할 수 있다.

### Bare Pods

* 파드를 특정한 노드에서 실행하도록 생성할 수 있다.
* 하지만 DaemonSet은 node의 failure나 커널 업데이트같이 node의 유지에 지장을 주는 상황에서 삭제되거나 종료된 파드를 다시 생성한다.
* 이러한 이유로 각기의 pod를 생성하는 것보다 DaemonSet을 사용하는 것이 더 좋다.

### Static Pods

* Kubelet에 의해 사용되는 특정한 디렉토리에 파일을 쓰는 파드를 생성하는 것이 가능하다. - static pod라고 부른다.
* DaemonSet과 다르게 static pod는 kubectl이나 다른 Kubernetes API client에 의해 관리되지 않는다.
* Static Pods는 apiserver에 의존하지 않아 cluster bootstrapping의 상황에 유용하다.
* 또한 static pod는 후에 사라질 것이다.

### Deployments

* DaemonSet은 Deployment와 둘다 파드를 생성하고 예상되지 않은 종료에서 처리를 한다는 점이 비슷하다.
* 프론트엔드처럼 replica의 scaling up, down을 하고 update를 roll out하는 것이 어떤 호스트에서 파드가 동작하는지 컨트롤하는 것보다 중요한 stateless service에는 Deployment를 사용하라.
* 파드의 복사본이 모든 또는 특정한 호스트에서 항상 동작하도록 하는것이 중요하고, 다른 파드가 시작되기 전에 시작되어야 한다면 DaemonSet을 사용하라.



