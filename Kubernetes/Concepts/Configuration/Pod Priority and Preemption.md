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

## How to use priority and preemption

priority와 preemption을 사용하러면 Kubernetes 1.11 이후의 버전부터 다음의 절차를 따라야 한다.

1. 하나 이상의 [PriorityClasses](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass) 생성
2. 추가한 PriorityClasses중 하나인 [`priorityClassName`](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#pod-priority)을 이용하여 파드 생성. 물론 파드를 직접 생성할 필요는 없다; 보통 `priorityClassName`을 Deployment같은 collection object의 pod template에 추가하면 된다.

이 절차에 대해서 더 자세히 알기 위해서는 계속해서 읽어 보아라.

만약 feature를 시도해보고 이를 비활성화 하기로 마음먹었다면 반드시 PorPriority command-line flag를 지우거나 이를 `false`로 설정해야 한다. 그리고 API server와 scheduler를 재시동한다. feature가 비활성화 되고 나면 이미 떠 있던 파드들은 priority field를 유지하겠지만 preemption이 비활성화 되어있고 priority field는 무시될 것이다. feature가 비활성화되면 새로운 파드에서 `priorityClassName`을 설정할 수 없다.

## How to disable preemption

> Note : Kubernetes 1.12+에서 클러스터가 resource pressure 상태일 때 critical pods는 스케쥴링 될 때 scheduler preemption을 사용한다. 이런 이유로 preemption을 비활성화 하는 것은 추천되지 않는다.

> Note : Kubernetes 1.15버전 이후로 `NonPreemptingPriority`가 활성화 되어있으면, PriorityClasses는 `preemptionPolicy: Never` 옵션이 설정되어있어야 한다. 이는 PriorityClass 파드가 다른 파드들을 preempt하는 것을 막아준다.

쿠버네티스 1.11 이후로 preemption은 kube-scheduler 플래그 `disablePreemption`으로 관리된다. 이는 `false`가 default이다. 위의 note에도 불구하고 preemption을 비활성화 할 경우 `disablePreemption`을 `true`로 설정해야 한다.

이 옵션은 component config에서 사용이 가능하고, old-style command line options에서는 사용할 수 없다. 아래의 예시는 component config로 preemption을 disable하는 것이다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider

...

disablePreemption: true
```

