# Kubernetes Scheduler

Kubernetes에서 scheduling은 Pods가 Nodes에 매칭이 되어 kubelet이 이를 실행할 수 있도록 하는 것을 의미한다.

## Scheduling overview

scheduler는 노드가 할당되지 않은 새로 생성된 Pods를 관찰한다. scheduler가 발견한 모든 파드들에 대해서 scheduler는 파드가 뜰 수 있는 최적의 노드를 찾는 데에 책임을 가지고 있다. scheduler는 아래에서 기술하는 스케쥴링 원리에 따라서 placement decision을 수행하게 된다.

왜 파드가 특정한 Node에서 뜨는지 이해하고 싶거나, custom scheduler를 스스로 구현하고 싶다면 이 페이지는 scheduling에 대하여 학습하는 데 도움을 줄 수 있다.

## kube-scheduler

[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)는 Kubernetes에서 default scheduler이고, control plane의 일부분으로 동작한다. kube-scheduler는 사용자가 원한다면 그만의 scheduling component를 작성하고 이를 대신 사용할 수 있도록 디자인되어 있다.

모든 새로이 생성되는 파드나 스케쥴링 되지 않은 파드들에 대해서, kube-scheduler는 이들이 실행될 수 있는 최적의 노드를 선택한다. 하지만 파드 내의 모든 container는 resource에 대해서 다른 requirements를 가지고 있고, 또한 모든 파드는 다른 requirements를 가지고 있다. 따라서, 존재하는 노드들은 특정한 scheduling requirements에 따라서 필터링 될 필요가 있다.

Cluster에서, 파드의 scheduling requirements를 만족하는 노드들은 feasible nodes라고 불린다. 어떤 노드도 적합하다고 판단되지 않으면, 파드는 스케쥴러가 뜰 수 있는 노드를 찾는 동안 unscheduled 상태로 남아있게 된다.

스케쥴러는 파드에 대해서 feasible Nodes를 찾고 정해진 함수들을 실행시켜서 feasible Nodes에 대한 점수를 매기고, 파드가 동작하는 데 적합한 노드들 중에서 가장 높은 점수를 가진 것을 선택한다. 그 후 스케쥴러는 이러한 결정에 대해 API server에 알림을 주고, 이 process를 binding이라고 한다.

scheduling decisions에서 고려해야 하는 요소들은 individual와 collective resource requirements, hardware / software / policy constraints, affinity와 anti-affinity specifications, data locality, inter-workload interference 등을 포함한다.

## Scheduling with kube-scheduler

kube-scheduler는 두 단계를 통해서 파드를 띄울 노드를 선택한다.

1. Filtering
2. Scoring

filtering 단계는 파드를 띄울 수 있는 적합한 노드들을 찾는 과정이다. 예를 들어, PodFitsResources 필터는 후보 노드들이 파드의 지정된 resource requests를 만족할 수 있는 충분한 resource를 가지고 있는지 검사한다. 이 단계 이후에, node list는 적합한 모든 노드들을 포함한다; 보통, 하나 이상이다. list가 비어있게 될 경우 파드는 스케쥴링이 아직 불가능하다.

scoring 단계에서 scheduler는 남아있는 노드들에 순위를 매겨서 파드가 위치하기에 가장 적합한 노드를 선택한다. scheduler는 필터링을 통과한 각 노드에 대해서 활성화된 scoring rules를 기반으로 점수를 할당한다.

최종적으로 kube-scheduler는 파드에게 가장 높은 등수인 노드를 할당하게 된다. 같은 점수를 가진 노드가 하나 이상이라면, kube-scheduler는 이들 중 하나를 랜덤으로 선택한다.

### Default policies

kube-scheduler는 scheduling policies의 default set을 가지고 있다.

### Filtering

* `PodFitsHostPorts`: 노드가 파드가 요구하는 Pod ports에 대해서 free ports(network protocol kind)를 가지고 있는지 확인한다.
* `PodFitsHost`: 파드가 hostname으로 지정한 특정한 노드가 있는지 확인한다.
* `PodFitsResources`: Node가 파드의 requirement를 만족하는 free resource(예: CPU, Memory)를 가지고 있는지 확인한다.
* `PodMatchNodeSelector`: 파드의 Node Selector와 일치하는 Node의 label이 있는지 확인한다.
* `NoVolumeZoneConflict`: 파드가 요청하는 Volumes가 그 storage에 대해서 failure zone restrictions에 따라 노드에서 사용 가능한지 평가한다.
* `NoDiskConflict`: 파드가 요청한 volumes에 따라 노드에 적합한지, 그것이 이미 마운트 되었는지 평가한다.
* `MaxCSIVoilumeCount`: 얼마나 많은 CSI(Container Storage Interface) volumes가 붙을 수 있는지 결정하고, 이것이 설정한 제한을 넘었는지 확인한다.
* `CheckNodeMemoryPressure`: 노드가 memory pressure를 보고하고, 설정된 예외사항이 없다면 파드는 그곳에 스케쥴링 되지 않을 것이다.
* `CheckNodePIDPressure`: 노드가 process IDs가 부족하다고 보고하고 예외사항이 없다면, 파드는 그곳에 스케쥴링 되지 않을 것이다.
* `CheckNodeDiskPressure`: 노드가 storage pressure(filesystem이 꽉차거나 거의 차있는 상태)를 보고하고 설정한 예외사항이 없다면 파드는 그곳에서 스케쥴링 되지 않을 것이다.
* `CheckNodeCondition`: 노드는 네트워크가 불가능하거나 kubelet이 파드를 실행시키는 데 준비가 되지 않았거나 filesystem이 완전히 꽉찼다고 보고할 수 있다. 노드에 대해서 이러한 상태가 설정되어있다면 설정된 예외사항이 없을 경우 파드는 그곳에 스케쥴링 되지 않을 것이다.
* `PodToleratesNodeTaints`: 파드의 tolerations는 노드의 taints를 tolerate할 수 있는지 확인한다.
* `CheckVolumeBinding`: 파드가 요청한 volumes를 만족하는지 평가한다. 이는 bound, unbound PVC 모두에 적용된다.

### Scoring

* `SelectorSpreadPriority`: 같은 Service, StatefulSet, ReplicaSet을 가진 파드들에 대해서 파드를 호스트 전체적으로 뿌리는 것이다.
* `InterPodAffinityPriority`: weightedPodAffinityTerm의 elements를 순회하며 연관된 PodAffinityTerm이 그 노드에 만족할 경우 "weight"를 sum에 더한다; 가장 높은 sum을 가지는 노드가 선호된다.
* `LeastRequestedPriority`: requested resource가 적은 노드를 선호한다. 다시말해, 노드에 더 많은 파드가 위치할수록, 그리고 그러한 파드들이 더 많은 resource를 사용할수록 이 policy는 낮은 순위를 매긴다.
* `MostRequestedPriority`: requested resource가 가장 많은 노드를 선호한다. 이 policy는 scheduled Pods들이 그 전체 workloads를 동작시키는 데 필요한 Nodes의 수를 최소화하도록 한다.
* `RequestedToCapacityRatioPriority`: default resource scoring function shape를 사용하는 ResourceAllocationPriority를 기반으로 한 requestedToCapacity를 생성한다.
* `BalancedResourceAllocation`: balanced resource usage를 가진 노드를 선호한다.
* `NodePreferAvoidPodsPriority`: node annotation이 `scheduler.alpha.kubertes.io/preferAvoidPods`인 것에 따라 우선순위를 매긴다. 이 hint를 두개의 다른 파드가 같은 노드에서 동작하지 않도록 하고 싶을 때 사용할 수 있다.
* `NodeAffinityPriority`: 노드를 PreferredDuringSchedulingIgnoredDuringExecution에 있는 node affinity scheduling preferences에 따라서 우선순위를 매긴다. 더 자세한 사항은 [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)에서 확인 가능하다.
* `TaintTolerationPriority`: 노드에서 intolerable taints의 갯수를 기준으로 모든 노드의 priority list를 준비한다. 이 policy는 그 list를 고려하여 노드의 순위를 조정한다.
* `ImageLocalityPriority`: 이미 container images의 파드에 대해 local cache를 가지고 있는 노드를 선호한다.
* `ServiceSpreadingPriority`: Service가 주어졌을 때, 이 policy는 Service에 대한 파드가 다른 노드에서 실행될 수 있도록 한다. 이는 서비스에 대한 파드가 아직 할당되지 않은 노드를 선호한다. 전체적으로 Service가 single Node failure에 대해서 더 잘 견딜 수 있게 된다.
* `CalculateAntiAffinityPriorityMap`: 이 policy는 [pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)를 구현하는 데 도움을 준다.
* `EqualPriorityMap`: 모든 노드에 대해서 동일한 wight을 준다.