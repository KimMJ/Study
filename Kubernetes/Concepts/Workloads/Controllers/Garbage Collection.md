# Garbage Collection

* 쿠버네티스의 garbage collector는 이전에 owner가 있었지만 더이상 owner가 없는 특정 오브젝트를 삭제하는 역할을 한다.

## Owners and dependents

* 몇몇 쿠버네티스 오브젝트는 다른 오브젝트의 owner이다.
  * ReplicaSet은 Pod 세트의 owner이다.
  * 소유되는 오브젝트들은 owner object에 dependents라고 한다.
  * 모든 dependent 오브젝트는 소유하는 오브젝트를 나타내는 `metadata.ownerReferences`필드를 가지고 있다.
* 가끔 쿠버네티스는 `ownerReference`를 자동으로 생성한다.
  * ReplicaSet을 생성하면 Kubernetes는 자동으로 ReplicaSet의 각 파드에서 `ownerReference`필드를 설정한다.
  * 1.8 버전에서 쿠버네티스는 ReplicationController, ReplicaSet, StatefulSet, DaemonSet, Depolyment, Job, CronJob에 의해 관리되거나 생성되는 오브젝트들에 대해서 자동으로 `ownerReference`값을 설정한다.
* 사용자는 또한 owner와 dependent의 관계를 `ownerReferece` 필드를 설정함으로써 수동으로 정해줄 수있다.

> Note : 다른 네임스페이스에 있는 owner는 디자인적으로 불가능하다. 즉, 1) 네임스페이스 스코프의 dependent가 같은 네임스페이스 또는 클러스터 스코프의 owner를 정할 수 있다. 2) 클러스터 스코프의 dependent는 클러스터 스코프의 owner만 지정할 수 있고 네임스페이스 스코프의 owner는 지정할 수 없다

## Controlling how the garbage collector deletes dependents

* 오브젝트를 삭제할 때, 오브젝트의 dependents또한 자동으로 지워지도록 지정할 수 있다.
  * 자동으로 dependents를 지우는 것을 cascading deletoin이라고 한다.
  * background, foreground 두가지 방식이 있다.
* dependents를 자동으로 삭제시키지 않고 오브젝트를 삭제시키면 dependents는 orphaned(고아)이 된다.

### Foreground cascading deletion

* foreground cascading deletion에서, 루트 오브젝트는 먼저 `deletion in progress` 상태가 된다. `deletion in progress` 상태는 다음이 참이다.
  * 오브젝트는 REST API를 통해서 여전히 볼 수 있다.
  * 오브젝트의 `deletionTimestap`가 설정되었다.
  * 오브젝트의 `metadata.finalizers`는 `forgroundDeletion`값을 포함한다.
* `deletion in progress` 상태가 설정이 되면 garbage collector는 오브젝트의 dependent를 삭제한다.
* garbage collector가 모든 "blocking" dependents를 삭제하면(`ownerReference.blockOwnerDeletion=true`인 오브젝트) owner 오브젝트를 삭제한다.
* `foregroundDeletion`에서는 `ownerReference.blockOwnerDeletion=true`인 dependents만 owner 오브젝트의 삭제를 막는다.
* 쿠버네티스 1.7에서 admission controller가 추가되어 owner 오브젝트에 대한 삭제 권한을 기준으로 유저가 `blockOwnerDeletion`이 참이 되도록 설정하는 접근을 관리한다.
  * 비인가된 dependent는 owner 오브젝트가 삭제되는 것을 지연시킬 수 없다.
* 만약 오브젝트의 `ownerReferences` 필드가 컨트롤러에 의해(Deployment, ReplicaSet) 설정되면, blockOwnerDeletion이 자동으로 설정되고 이를 수동으로 수정할 필요가 없다.

### Background cascading deletion

* background cascading deletion에서 kubernetes는 owner 오브젝트를 즉시 삭제하고 garbage collector는 background에서 dependents를 삭제한다.

### Setting the cascading deletion policy

* cascading deletion policy를 조작하기 위해서 오브젝트 삭제시 `deleteOptions` argument에서 `propagationPolicy`필드를 설정한다.

  * "Orphan", "Foreground", "Background"로 설정이 가능하다.

* 1.9 버전 전에는 많은 controller resource의 default garbage collection policy가 `orphan`이다.

  * ReplicationController, ReplicaSet, Statefulet, DaemonSet, Deployment가 이에 해당한다.
  * `extensions/v1beta1`, `apps/v1beta1`, `apps/v1beta2` 버전들에서는 지정하지 않더라도 dependent 오브젝트가 orphan으로 디폴트 처리되어있다.

* 1.9버전에서는 모든 `apps/v1`그룹 버전의 모든 종류에서 dependent 오브젝트는 기본적으로 지워진다.

* `kubectl`은 cascading deletion도 지원한다.

  * `kubectl`을 이용해서 자동으로 dependents를 지우고 싶으면 `--cascade`를 true로 설정하면 된다.

  * Orphan dependents는 `--cascade`를 false로 설정하면 된다.

    ```bash
    kubectl delete replicaset my-repset --cascade=false
    ```

  * `--cascade`의 기본값은 true이다.

### Additional note on Deployments

* 1.7 버전 전에는 Deployment에서 cascading delete를 할 때 생성된 ReplicaSet과 그 파드들을 삭제하기 위해서는 반드시 `propagationPolicy: Foreground`를 사용해서 지워야 했다.
* propagationPolicy가 사용되지 않으면 ReplicaSet만 지워지게 될 것이고 파드는 고아가 된다.