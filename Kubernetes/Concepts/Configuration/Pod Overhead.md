# Pod Overhead

## FEATURE STATE: `Kubernetes v1.16` - alpha

* 파드가 노드에서 동작하고 있을 때, 파드 스스로가 시스템 자원을 사용한다.
  * 이런 리소스는 파드 내의 컨테이너들이 동작할 때 사용되는 리소스에 추가된다.
  * 파드 오버헤드는 파드 인프라가 컨테이너 requests와 limits에 걸쳐 소비하는 리소스를 측정하기 위한 기능이다.

## Pod Overhead

* 쿠버네티스에서 파드의 오버헤드는 파드의 RuntimeClass와 관련된 오버헤드에 대한 admission 시간에 정해진다.
* 파드 오버헤드가 활성화되면, 파드를 스케쥴링할 때 오버헤드는 컨테이너 리소스 request에 합해진다.
  * 이와 비슷하게 kubelet은 파드의 cgroup을 조절할 때, 파드 축출 순위를 결정할 때 파드 오버헤드를 포함시킬 것이다.

### Set Up

* 클러스터 내에서 `PodOverhead` feature gate가 활성화 되어 있는지 확인한다. (기본값은 꺼져있다.)
  * 즉:
    * [kube-scheduler](https://kubernetes.io/docs/reference/generated/kube-scheduler/) 안에서
    * [kube-apiserver ](https://kubernetes.io/docs/reference/generated/kube-apiserver/)안에서
    * 각 노드의 [kubelet ](https://kubernetes.io/docs/reference/generated/kubelet)안에서
    * feature gate를 사용하는 어떤 custom API server 안에서

> Note : RuntimeClass 리소스를 작성할 수 있는 유저는 workload performance에서 클러스터 전역에 영향을 줄 수 있다. 사용자는 쿠버네티스 access controls를 사용하여 이런 능력에 대해 접근 제한을 두어야 한다.

