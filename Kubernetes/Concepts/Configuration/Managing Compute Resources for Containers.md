# Managing Compute Resources for Containers

* 파드를 지정하면 각 container에서 필요한 CPU와 memory (RAM)을 지정할 수 있다.
  * container에서 resource requests가 지정되면 스케쥴러는 파드를 위치시킬 노드에 대해 더 좋은 결정을 내린다.
  * container에서 limits가 지정되면 노드에 대한 리소스 contention은 지정된 방법으로 핸들링된다.
  * requests와 limits에 대한 자세한 사항은 [Resource QoS](https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md)를 확인하라.

## Resource types

* CPU와 memory는 resource type이다.
  * resource type은 기본 단위를 가지고 있다. 
  * CPU는 core라는 단위로 정해져있고, memory는 바이트 단위로 정해져있다.
* CPU와 memory는 둘다 compute resources또는 그냥 resources이다.
  * compute resources는 요청되고 할당되고 소비되는 측정가능한 양이다.
  * API resources와는 다르다.
  * Pods나 Services같은 API resources는 Kubernetes API server에 의해서 읽고 수정될 수 있는 오브젝트이다.

## Resource requests and limits of Pod and Container

* 각 파드의 컨테이너는 하나 이상의 다음 항목들을 지정할 수 있다.
  * `spec.containers[].resources.limits.cpu`
  * `spec.containers[].resources.limits.memory`
  * `spec.containers[].resources.requests.cpu`
  * `spec.containers[].resources.requests.memory`
* requests와 limits가 개별 컨테이너에 대해 지정된다 하더라도 파드 resource의 request, lmits에 대해 이야기하는 것이 편할 것이다.
  * 특정한 resource type의 Pod resource request/limit는 파드에서 각 컨테이너의 requests/limits의 합이다.

## Meaning of CPU

* CPU resources에 대한 limits, requests는 cpu 단위로 측정이 된다.
  * Kubernetes에서 1 cpu는 다음과 같다.
    * 1 AWS vCPU
    * 1 GCP Core
    * 1 Azure vCore
    * 1 IBM vCPU
    * 1 Hyperthread on a bare-metal Intel processor with Hyperthreading
* 단편적인 request도 가능하다.
  * `spec.containers[].resources.requests.cpu`가 `0.5`인 컨테이너는 하나의 CPU에 대한 요청에서 반만 처리하도록 보장된다.
  * `0.1`은 `100m`과 같은 의미이고 이는 "100 milicpu"로 읽는다.
    * 몇몇 사람들은 "100 milicores"로 읽기도 하며 이는 같은 것을 의미한다.
  * `0.1`처럼 소수점이 있으면 API에 의해서 `100m`으로 바뀌고 `1m`보다 더 작은 단위는 허용되지 않는다.
  * 이때문에 `100m`형식을 추천한다.
* CPU는 상대값이 아닌 항상 절대값을 요청받는다.
  * 0.1은 single-core이든, dual-core이든, 48-core이든 같은 CPU값이다.

## Meaning of memory

* `memory`에 대한 limits와 requests는 바이트로 측정된다.

  * 그냥 숫자나 E,P,T,G,M,K같은 suffixes를 사용하는 fixed-point integer로 memory를 표현할 수 있다.

  * 또한 이진접두어를 사용할 수 있다 : Ei, Pi, Ti, Gi, Mi, Ki

  * 예를 들어 다음의 표현은 다 같다.

    ```nothing
    128974848, 129e6, 129M, 123Mi
    ```

* 다음은 예시이다.

  * 이 파드는 두개의 컨테이너를 가지고 있다.
  * 각 컨테이너는 0.25 cpu를 요청하고 64MiB의 메모리를 요청한다. (2<sup>26</sup>bytes)
  * 각 컨테이너는 0.5 cpu와 128MiB의 제한이 있다.
  * 파드가 0.5 cpu와 128 MiB의 메모리를 요청한다고 해도 되며 limit는 1 cpu, 256MiB의 메모리를 제한으로 둔다.
  
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: frontend
  spec:
    containers:
    - name: db
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "password"
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
    - name: wp
      image: wordpress
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  ```

## How Pods with resource requests are scheduled

* 파드를 생성할 때 Kubernetes scheduler는 파드가 동작할 노드를 선택한다.
  * 각각의 노드는 각 resource type에 대한 최대 capacity를 가진다.
    * 파드에 대해 제공해줄수 있는 CPU와 memory의 양.
  * 각 resource type에 대해서 scheduler는 schedule된 contatiner의 resource requests의 합이 노드의 capacity모다 작게됨을 보장한다.
  * 노드에서 실제 memory나 CPU의 사용 resource의 양이 매우 적다 하더라도 scheduler는 용량 체크가 실패하면 노드에 파드를 배치하는것을 여전히 거부한다.
  * 이는 노드에서 request 비율에서 daily peak동안의 resource 사용량 증가하는 상황에서 노드가 resource shortage나지 않도록 해준다.

## How Pods with resources limits are run

* 파드의 컨테이너가 kubelet을 시작할 때 CPU와 memory limits를 container runtime에 전달한다.

* 도커를 사용한다면
  * `spec.containers[].resources.requests.cpu`는 코어 값으로 변경이 된다.
    * 값이 1보다 작으면 1024로 곱해진다.
    * 이 값보다 크거나 2이면 `docker run` 커맨드에서 `--cpu-shares`플래그의 값으로 쓰인다.
    
  * `spec.containers[].resources.limits.cpu`는 micicore 값으로 변경되고 100으로 곱해진다.
    
    * 결과값은 컨테이너가 100ms마다 사용할 수 있는 CPU 시간의 총 양이다.
  * 컨테이너는 이 기간동안 CPU 시간의 공유분 이상을 사용할 수 있다.
  
    > Note : default quota 기간은 100ms이다. 최소 CPU quota는 1ms이다.
    
  * `spec.containers[].resources.limits.memory`는 integer로 변환되고 `docker run`에서 `--memory` flag로 사용이 된다.
  
* 컨테이너가 memory limit을 초과하면 종료될 것이다.

  * restart가 가능하면 다른 runtime failure처럼 kubelet은 이를 restart할 것이다.

* 컨테이너가 memory request를 초과한다면 노드에서 메모리가 부족할 때 언제든 축출될 수 있다.

* 컨테이너는 약간의 시간동안 CPU limits를 초과하도록 허용할수도, 안할수도 있다.

  * 하지만 CPU의 추가 사용때문에 kill되지는 않을 것이다.

* 컨테이너가 schedule되지 않거나 resource limit때문에 kill되는 것을 결정하는 것은 [Troubleshooting](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#troubleshooting) 섹션 참고

## Monitoring compute resource usage

* 파드의 resource usage는 파드의 status의 일부분으로 보고된다.
* 클러스터에 대해 optional monitoring이 설정되었으면 파드의 resource usage는 monitoring system에 의해서 얻을 수 있다.

## Troubleshooting

### My Pods are pending with event message failedScheduling

* scheduler가 파드가 들어갈 수 있는 노드를 찾을 수 없을 경우 파드는 공간을 찾을 때까지 unscheduled 상태로 남아있다.

  * scheduler가 파드에 대한 공간 찾기를 실패할 때마다 이벤트가 다음과 같이 발생한다.

    ```shell
    kubectl describe pod frontend | grep -A 3 Events
    ```

    ```
    Events:
      FirstSeen LastSeen   Count  From          Subobject   PathReason      Message
      36s   5s     6      {scheduler }              FailedScheduling  Failed for reason PodExceedsFreeCPU and possibly others
    ```

* 이전의 예시에서 "frontend" 파드는 노드에서 CPU resource가 부족하기 때문에 schedule 되지 않았다.

  * 비슷한 에러 메시지가 memory가 부족할 때에도 생긴다. (PodExceedsFreeMemory)
  * 보통 파드가 이런 타입의 메시지로 pending이 되면 다음과 같은것을 해볼 수 있다.
    * 클러스터에 노드 추가하기
    * pending되는 파드를 위한 공간 확보를 위해 불필요한 파드 종료시키기
    * 파드가 전체 노드보다 크지 않음을 확인해라. 예를들어, 모든 노드가 `cpu: 1`씩의 capacity를 가지고 있다면 `cpu: 1.1`에 대한 요청은 절대 schedule되지 않을 것이다.

* 노드의 capacity와 할당된 양을 `kubectl describe nodes` 커맨드로 확인할 수 있다.

  ```shell
  kubectl describe nodes e2e-test-node-pool-4lw4
  ```

  ```
  Name:            e2e-test-node-pool-4lw4
  [ ... lines removed for clarity ...]
  Capacity:
   cpu:                               2
   memory:                            7679792Ki
   pods:                              110
  Allocatable:
   cpu:                               1800m
   memory:                            7474992Ki
   pods:                              110
  [ ... lines removed for clarity ...]
  Non-terminated Pods:        (5 in total)
    Namespace    Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits
    ---------    ----                                  ------------  ----------  ---------------  -------------
    kube-system  fluentd-gcp-v1.38-28bv1               100m (5%)     0 (0%)      200Mi (2%)       200Mi (2%)
    kube-system  kube-dns-3297075139-61lj3             260m (13%)    0 (0%)      100Mi (1%)       170Mi (2%)
    kube-system  kube-proxy-e2e-test-...               100m (5%)     0 (0%)      0 (0%)           0 (0%)
    kube-system  monitoring-influxdb-grafana-v4-z1m12  200m (10%)    200m (10%)  600Mi (8%)       600Mi (8%)
    kube-system  node-problem-detector-v0.1-fj7m3      20m (1%)      200m (10%)  20Mi (0%)        100Mi (1%)
  Allocated resources:
    (Total limits may be over 100 percent, i.e., overcommitted.)
    CPU Requests    CPU Limits    Memory Requests    Memory Limits
    ------------    ----------    ---------------    -------------
    680m (34%)      400m (20%)    920Mi (12%)        1070Mi (14%)
  ```

* 이전의 결과에서 파드의 request가 1200m CPUs를 넘거나 6.23Gi 메모리를 넘기는 것은 노드에 맞지 않는다는 것을 볼 수 있다.
* `Pods` 섹션을 보면 노드 안에서 어느 파드가 얼만큼의 공간을 차지하는지 알 수 있다.
* 시스템 데몬이 사용가능한 리소스의 일부분을 사용하기 때문에 파드에서 사용가능한 리소스의 총량은 노드의 capacity보다 작다.
  * `allocatable` 필드 [NodeStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#nodestatus-v1-core)는 파드에서 사용할 수 있는 리소스의 양을 보여준다.
  * 더 자세한 정보는 [Node Allocatable Resources](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md)참고
* [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) feature는 소비할 수 있는 resources의 총 량에 제한을 거는 설정이다.
  
  * namespace와 연결하여 사용하면 한 팀이 모든 리소스를 독차지 하는 것을 막아준다.

### My Container is terminated

* 컨테이너는 리소스의 부족으로 종료될 수 있다.

  * 컨테이너가 리소스 제한때문에 종료되었는지 알기위해서 `kubectl desribe pod`를 궁금한 파드에 대해서 해 보아라.

    ```shell
    kubectl describe pod simmemleak-hra99
    ```

    ```nothing
    Name:                           simmemleak-hra99
    Namespace:                      default
    Image(s):                       saadali/simmemleak
    Node:                           kubernetes-node-tf0f/10.240.216.66
    Labels:                         name=simmemleak
    Status:                         Running
    Reason:
    Message:
    IP:                             10.244.2.75
    Replication Controllers:        simmemleak (1/1 replicas created)
    Containers:
      simmemleak:
        Image:  saadali/simmemleak
        Limits:
          cpu:                      100m
          memory:                   50Mi
        State:                      Running
          Started:                  Tue, 07 Jul 2015 12:54:41 -0700
        Last Termination State:     Terminated
          Exit Code:                1
          Started:                  Fri, 07 Jul 2015 12:54:30 -0700
          Finished:                 Fri, 07 Jul 2015 12:54:33 -0700
        Ready:                      False
        Restart Count:              5
    Conditions:
      Type      Status
      Ready     False
    Events:
      FirstSeen                         LastSeen                         Count  From                              SubobjectPath                       Reason      Message
      Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {scheduler }                                                          scheduled   Successfully assigned simmemleak-hra99 to kubernetes-node-tf0f
      Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   pulled      Pod container image "k8s.gcr.io/pause:0.8.0" already present on machine
      Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   created     Created with docker id 6a41280f516d
      Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   started     Started with docker id 6a41280f516d
      Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    spec.containers{simmemleak}         created     Created with docker id 87348f12526a
    ```

* 이전의 예시에서 `Restart Count: 5`는 파드에서 `simmemleak` 컨테이너가 종료되었고 5번 재시작되었음을 알려준다.

* `kubectl get pod`를 `-o go-template=...` 옵션을 사용하여 호출해 이전에 종료된 컨테이너의 상태를 불러올 수 있다.

  ```shell
  kubectl get pod -o go-template='{{range.status.containerStatuses}}{{"Container Name: "}}{{.name}}{{"\r\nLastState: "}}{{.lastState}}{{end}}'  simmemleak-hra99
  ```

  ```nothing
  Container Name: simmemleak
  LastState: map[terminated:map[exitCode:137 reason:OOM Killed startedAt:2015-07-07T20:58:43Z finishedAt:2015-07-07T20:58:43Z containerID:docker://0e4095bba1feccdfe7ef9fb6ebffe972b4b14285d5acdec6f0d3ae8a22fad8b2]]
  ```

* 컨테이너가 `reason:OOM killed`로 인해서 종료되었음을 알 수 있다. `OOM`은 Out Of Memory이다.

## Local ephemeral storage

**FEATURE STATE: **`Kubernetes v1.15` - beta

* Kubernetes version 1.8에서 로 컬 단기 저장소를 관리하는 ephemeral-storage라는 새로운 리소스가 추가되었다.
  * 각 쿠버네티스 노드에서 kubelet의 root 디렉토리 (`/var/lib/kubelet`이 기본)와 log 디렉토리(`/var/log`)는 노드의 루트 파티션에 저장이 된다.
  * 이 파티션은 파드와 공유되고 emptyDir volumes, container logs, image layers, container wriatble layers로 파드가 사용할 수 있다.
* 이 파티션은 단기적이고 어플리케이션은 Disk IOPS같은 SLAs를 기대하지 말아야 한다.
* 로컬 단기 저장소 관리는 루트 파티션에 대해서만 적용이 된다.
  * 추가적인 image layer와 writable layer에 관한 파티션은 범위 밖 내용이다.

> Note : optional runtime partition이 사용 되면 루트 파티션은 어떤 image layer나 writable layers도 유지하지 않을 것이다.

### Requests and limits setting for local ephemeral storage

* 각 파드의 컨테이너는 하나 이상의 다음 것들을 지정할 수 있다.

  * `spec.containers[].resources.limits.ephemeral-storage`
  * `spec.containers[].resources.requests.ephemeral-storage`

* `ephemeral-storage`에 대한 limits와 requests는 bytes단위로 측정된다.

  * 숫자나 suffix로 부동점 숫자를 사용하여 표현할 수 있다: E, P, T, G, M, K

  * 또한 이진접두어를 사용할 수도 있다: Ei, Pi, Ti, Gi, Mi, Ki

  * 예를 들어, 다음의 표현은 거의 같은 값을 의미한다.

    ```
    128974848, 129e6, 129M, 123Mi
    ```

* 예를 들어 다음의 파드는 두개의 컨테이너를 가지고 있다.

  * 각 컨테이너는 2GiB의 local ephemeral storage를 요청한다.
  * 각 컨테이너는 4GiB의 local ephemeral storage에 대한 제한이 있다.
  * 따라서 파드는 4GiB의 local ephemeral storage를 요청할 것이고 8GiB의 제한이 있을 것이다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: frontend
  spec:
    containers:
    - name: db
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "password"
      resources:
        requests:
          ephemeral-storage: "2Gi"
        limits:
          ephemeral-storage: "4Gi"
    - name: wp
      image: wordpress
      resources:
        requests:
          ephemeral-storage: "2Gi"
        limits:
          ephemeral-storage: "4Gi"
  ```

### How Pods with ephemeral-storage requests are scheduled

* 파드를 생성할 때 쿠버네티스 스케쥴러는 파드가 농작할 노드를 선택한다.
  * 각 노드는 파드에 제공할 수 있는 최대 local ephemeral storage를 가지고 있다.
  * 자세한 사항은 [“Node Allocatable”](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable)참고.
* 스케쥴러는 스케쥴된 컨테이너의 resource requests의 합이 노드의 용량보다 작도록 한다.

### How  Pods with ephemeral-storage limits run

* 컨테이너 레벨의 isolation에서 컨테이너의 writable layer와 logs 사용이 storage limit을 넘는다면 파드는 축출될 것이다.
  * 파드 레벨의 isolation에서 모든 컨테이너와 파드의 emptyDir volumes의 local ephemeral storage의 사용 합이 제약을 넘으면 파드는 축출된다.

### Monitoring ephemeral-storage consumption

* local ephemeral storage가 사용될 때 kubelet에 의해서 지속적으로 모니터링 된다.
  * 모니터링은 각 emptyDir volume, log directories, writable layers를 일정 간격으로 스캐닝하는 것으로 수행된다.
  * Kubernetes 1.15부터 emptyDir volumes(로그 디렉토리나 writable layers는 제외)는 아마 cluster operator's의 옵션에서 [project quotas](http://xfs.org/docs/xfsdocs-xml-dev/XFS_User_Guide/tmp/en-US/html/xfs-quotas.html)를 사용하여 관리된다.
  * project quotas는 원래 XFS에 구현되어있고 최근에 extf4s에 생겼다.
  * project quotas는 모니터링과 enforcement 모두에 쓰인다.
    * Kubernetes 1.15에서는 alpha functionality로 모니터링만 가능하다.
* quotas는 디렉토리 스캐닝보다 더 빠르고 정확하다.
  * 디렉토리가 프로젝트에 할당되면 모든 파일은 그 프로젝트에 대해 생성된 디렉토리 아래에서 상성되고 커널은 단지 그 프로젝트에서 얼마나 많은 블락들이 파일에 의해 쓰이는지만 추적하면 된다.
  * 파일이 생성되고 삭제되지만 file descriptor가 열려있으면 공간을 계속 차지한다.
  * 이런 공간은 quota에 의해서 추적되지만 디렉토리 스캔으로는 감지되지 않는다.
* Kubernetes는 project IDs가 1048576으로 시작한다.
  * ID는 `/etc/project`/와 `/etc/projid`에 등록되어있다.
  * 이 범위의 project ID가 다른 시스템 상의 목적으로 쓰일 경우 이 project ID들은 쿠버네티스가 그것들을 쓰지 않도록 반드시 `/etc/projects`와 `/etc/projid`에 등록이 되어 있어야 한다.
* project quota를 활성화 하기 위해 cluster operator는 반드시 다음의 것들을 해야 한다.
  * kubelet 설정에서 `LocalStorageCapacityIsolationFSQuotaMonitoring=true` feature gate를 활성화 해야한다. 이는 쿠버네티스 1.15에서는 `false`로 기본 설정되어 있어서 반드시 `true`로 변경되어야 한다.
  * 루트 파티션 (또는 추가적인 runtime 파티션)이 project quotas가 활성화되어있도록 만들어져야 한다. 모든 XFS 파일시스템은 project quotas를 지원하지만 ext4 파일시스템은 따로 구성해야 한다.
  * 루트 파티션 (또는 추가적인 runtime 파티션)이 project quotas가 활성화되어있도록 마운트되어야 한다.

### Building and mounting filesystems with project quotas enabled

* XFS 파일시스템은 만들때 특별한 액션이 필요하지 않다.

  * 자동으로 project quotas가 사용가능하도록 되어있다.

* ext4fs 파일시스템은 반드시 project quotas가 활성화되도록 만들어야 하고 그러면 파일시스템에서 활성화 되어있을 것이다.

  ```shell
  % sudo mkfs.ext4 other_ext4fs_args... -E quotatype=prjquota /dev/block_device
  % sudo tune2fs -O project -Q prjquota /dev/block_device
  ```

* 파일시스템을 마운트하기 위해서 ext4fs와 XFS 모두 `prjquota` 옵션이 `/etc/fstab`에 설정되어있어야 한다.

  ```
  /dev/block_device	/var/kubernetes_data	defaults,prjquota	0	0
  ```

## Extended resources

* extended resources는 `kubernetes.io` 도메인 밖에서 fully-qualified resource names이다.
  * 이는 cluster operators가 advertise할 수 있도록 하고 유저가 non-Kuberentes-built-in 리소스를 사용할 수 있도록 한다.
* extended resources를 사용하기 위해서는 두가지 단계가 있다.
  * 먼저 cluster operator는 반드시 extended resource를 advertise해야 한다.
  * 두번째로 유저는 반드시 파드에서 extended resource를 요청해야 한다.

### Managing extended resources

#### Node-level extended resources

* 노드 레벨 extended resources는 노드와 엮여있다.

##### Device plugin managed resources

* [Device Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)을 참조하여 여떻게 각 노드에서 device plugin managed resources를 advertise하는지 확인해라.

##### Other resources

* 새로운 노드 레벨 extended resource를 advertise하려면 cluster operator는 `PATCH` HTTP request를 API 서버에 제출해서 클러스터 안의 노드에 대해 `status.capacity`의 available quantity를 지정할 수 있다.
  * 이 동작 후에 노드의 `status.capacity`는 새로운 리소스를 포함한다.
  * `status.allocatable` 필드는 kubelet에 의해 자동으로 asynchronously 하게 새로운 리소스를 업데이트한다.
  * 스케쥴러가 노드의 `status.allocatable` 값을 파드의 fitness를 측정할 때 사용하기 때문에 새로운 리소스로 node capacity를 patch하는 것과 처음 파드가 그 노드에 대해 스케쥴되기 위해 리소스를 요청하도록 하는 것 사이에 약간의 딜레이가 있을수 있음을 인지하여라.

##### Example:

* `k8s-master`가 마스터 노드이고 `k8s-node-1`인 상태에서 다섯개의 "example.com/foo" 리소스를 advertise하기 위해 어떻게 `curl`을 사용해서 HTTP request를 사용하는지 보여주는 예시이다.

  ```shell
  curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "add", "path": "/status/capacity/example.com~1foo", "value": "5"}]' \
  http://k8s-master:8080/api/v1/nodes/k8s-node-1/status
  ```

> Note : 이전의 요청에서 `~1`은 문자에 대한 인코딩이고 `/`는 patch 경로이다. JSON-Patch에서의 operation 경로 값은 JSON-Pointer로 해석된다. 자세한 사항은 [IETF RFC 6901, section 3](https://tools.ietf.org/html/rfc6901#section-3) 참조.

### Cluster-level extended resources

* 클러스터 레벨 extended resources는 노드와 엮여있지 않다.
  * 보통 리소스 소비와 리소스 쿼터를 관리하는 scheduler extenders에 의해 관리된다.
* [scheduler policy configuration](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/scheduler/api/v1/types.go#L31)에서 scheduler extneders가 관리하는 extended resources를 지정할 수 있다.

#### Example

* scheduler policy에 대한 다음 configuration은 클러스터 레벨 extended resource "example.com/foo"가 scheduler extender에 의해 관리된다는 것을 의미한다.

  * 스케쥴러는 파드가 "example.com/foo" 요청했을 때에만 파드를 scheduler extender에 전송한다.
  * `ignroedByScheduler` 필드는 스케쥴러가 "example.com/foo"를  `PodFitsResources`를 근거로 리소스를 체크하지 않도록 지정한다.

  ```json
  {
    "kind": "Policy",
    "apiVersion": "v1",
    "extenders": [
      {
        "urlPrefix":"<extender-endpoint>",
        "bindVerb": "bind",
        "managedResources": [
          {
            "name": "example.com/foo",
            "ignoredByScheduler": true
          }
        ]
      }
    ]
  }
  ```

### Consuming extended resources

* 사용자는 CPU와 메모리같은 파드 스펙에서의 extended resources를 사용할 수 있다.
  * 스케쥴러는 리소스 계산을 하여 가능한 양보다 더 많이 파드에 할당되지 않도록 한다.
* API 서버는 extended reousrces를 자연수 범위로 제한한다.
  * 유효한 quantity의 예시는 `3`, `3000m`, `3Ki`가 있다.
  * 유효하지 않는 quantity는 `0.5`, `1500m`이 있다.

> Note : extended resource는 Opaque Integer Resources를 대체한다. 사용자는 할당된 `kubernetes.io`와는 다른 도메인 이름 prefix를 사용할 수 있다.

* 파드에서 extended resource를 사용하려면 리소스 이름을 컨테이너 스펙의 `spec.containers[].resources.limits` 맵에 키로 포함시키면 된다.

> Note : extended resources는 overcommit될수 없으며 따라서 request와 limit은 반드시 컨테이너 스펙에서 이전에 이미 있다면 동일해야한다.

* 파드는 CPU, 메모리, extended resource를 포함한 모든 리소스 요청이 만족될때만 스케쥴된다.
  * 파드는 리소스 요청이 만족되지 않는동안 `PENDING` 상태를 유지한다.

#### Example:

* 아래의 파드는 2CPU와 1 "example.com/foo" (extended resource)를 요청한다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-pod
  spec:
    containers:
    - name: my-container
      image: myimage
      resources:
        requests:
          cpu: 2
          example.com/foo: 1
        limits:
          example.com/foo: 1
  ```

## Planned Improvements

* Kubernetes 1.5에서는 리소스 양이 컨테이너에서만 지정되도록 허용한다.
  * emptyDir volumes같은 파드의 모든 컨테이너가 공유하는 리소스에 대해서도 측정하도록 계획되어있다.
* 쿠버네티스 1.5는 CPU와 메모리에 대한 컨테이너의 requests, limits만 지원한다.
  * node disk space resource와 커스텀 리소스 타입에 대한 추가를 위한 framework를 포함한 새로운 리소스 타입을 추가하는 것을 계획으로 한다.
* 쿠버네티스는 [Quality of Service](http://issue.k8s.io/168)의 여러 레벨을 지원함으로써 리소스에 대한 overcommit을 지원한다.
* 쿠버네티스 1.5에서 CPU의 한 단위는 다른 cloud provider에서 다른 것을 의미하고 같은 cloud provider내의 다른 장비 타입에 대해서도 다른 것을 의미한다.
  * 예를 들어 AWS에서 노드의 capacity는 ECUs에서 리포트되지만 GCE에서는 logical core에서 리포트된다.
  * 우리는 provider와 platform에 일관되게 하도록 cpu 리소스의 정의를 수정하려 계획중이다.