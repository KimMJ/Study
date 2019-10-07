# Pod Priority and Preemption

**FEATURE STATE**: `Kubernetes 1.14` - stable

* 파드는 priority를 가질 수 있다. 
  * priority는 파드가 상대적으로 다른 파드에 비해 얼마나 중요한지를 나타낸다.
  * 만약 파드가 스케쥴되지 않으면 스케쥴러는 낮은 priority의 파드를 선점(축출)하여 대기중인 파드가 시케쥴될 수 있도록 한다.
* 쿠버네티스 1.9버전 이후부터 priority는 파드의 스케쥴링 순서와 노드에서 리소스 부족시 축출되는 순서 모두에 영향을 미친다.
* 파드 priority와 Preemption은 쿠버네티스 1.11에 베타로 되었고 1.14에서 GA가 되었다.
  * 1.11버전 이후로 기본값은 활성화이다.
* 파드의 priority와 preemption이 아직 알파버전인 쿠버네티스 버전을 이용중이라면 이를 명시적으로 활성화시켜야 한다.
  * 이전 버전에서 이 기능을 사용하고자 한다면 쿠버네티스 버전에 맞는 문서상 절차를 따라야 한다

| Kubernetes Version | Priority and Preemption State | Enabled by default |
| :----------------- | :---------------------------- | :----------------- |
| 1.8                | alpha                         | no                 |
| 1.9                | alpha                         | no                 |
| 1.10               | alpha                         | no                 |
| 1.11               | beta                          | yes                |
| 1.14               | stable                        | yes                |

> 주의 : 모든 유저가 신뢰되는 것은 아닌 클러스터에서는 악성 유저가 가장 높은 priority의 파드를 생성하여 다른 파드들이 축출되고 스케쥴되지 않도록 하게 할 수 있다. 이 이슈를 해결하려면 ResourceQuota를 설정하여 파드의 priority를 이용해라. admin은 유저에 대해 priority 레벨을 지정하는 ResourceQuota를 생성하여 높은 priority의 파드를 생성하는 것으로부터 막는다. 이 기능은 쿠버네티스 1.12부터 베타이다.

