# Managing Compute Resources for Containers

* 파드를 지정하면 각 container에서 필요한 CPU와 memory (RAM)을 지정할 수 있다.
  * container에서 resource requests가 지정되면 스케쥴러는 파드를 위치시킬 노드에 대해 더 좋은 결정을 내린다.
  * container에서 limits가 지정되면 노드에 대한 리소스 contention은 지정된 방법으로 핸들링된다.
  * requests와 limits에 대한 자세한 사항은 [Resource QoS](https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md)를 확인하라.

