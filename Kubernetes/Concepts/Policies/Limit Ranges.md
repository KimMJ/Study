

# Limit Ranges

기본적으로 Kubernetes cluster에서 [compute resources](https://kubernetes.io/docs/user-guide/compute-resources)가 unbound된 채로 containers를 동작시키게 된다. Resource quotas가 설정되어 있으면 cluster administrator는 namespace를 기반으로 resource consumption과 creation을 제한할 수 있다. namespace 안에서, 파드나 컨테이너는 namespace의 resource quota로 정의된 CPU와 memory까지 소비할 수 있다. 하나의 파드나 컨테이너가 모든 resource를 독점할 수도 있다는 걱정이 생길 수 있다. Limit Range는 namespace 안에서 Pod나 Container가 사용할 수 있는 resource의 양을 제한하는 policy이다.

`LimitRange` 오브젝트로 정의된 Limit Range는 다음과 같은 제약을 줄 수 있다.

* namespace 안에서 각 파드나 컨테이너 마다 최소, 최대의 compute resources를 제한할 수 있다.
* namespace 내에서 PersistentVolumeClaim마다 minimum, maximum storage request를 제한할 수 있다.
* namespace 내에서 resource에 대한 request, limit 사이의 비율을 제한할 수 있다.
* namespace 내에서 compute resources에 대하여 컨테이너가 runtime에 자동으로 default request, limits에 대한 설정을 주입할 수 있다.

## Enabling Limit Range

Limit Range는 많은 Kubernetes distributions에 대해서 default로 활성화되어 있다. 이는 apiserver가 `--enable-admission-plugins=` 플래그가 `LimitRanger` admission controller를 하나의 arguments로 사용할 때 활성화 된다.

limit range는 `LimitRange` 오브젝트가 있는 특정한 namespace에서 사용할 수 있다.

### Overview of Limit Range:

* administrator는 하나의 namespace에 대해 하나의 `LimitRange`를 생성한다.
* 유저는 namespace 내에서 Pods, Containers, PersistentVolumeClaims와 같은 resources를 생성한다.
* `LimitRanger` admission controller는 compute resource requirements를 설정하지 않은 모든 파드와 컨테이너에 대해서 defaults limits를 강제설정하고, 사용량을 측정하여 이것이 resource minimum, maximum과 namespace에서 정해진 `LimitRange` 범위를 초과하지 않는지 확인한다.
* 파드, 컨테이너, PVC같은 resource를 생성하거나 업데이트 하는 것이 limit range constraint를 위반하게 될 경우 API server로의 요청은 HTTP status code `403 FORBIDDEN`으로 실패하게 되고 메시지는 그 위반한 constraint에 대한 설명이 될 것이다.
* limit range가 `cpu`나 `memory`같은 compute resources에 대해 namespace에서 활성화 되어 있다면 유저는 반드시 이러한 values에 대해 request나 limits를 지정해야 한다; 그렇지 않을 경우 시스템은 파드의 생성을 거부할 것이다.
* LimitRange의 위배는 이미 동작중인 pods에서가 아니라 Pod Admission stage에서만 발생할 수 있다.

다음은 limit range를 통해서 생성할 수 있는 policy에 대한 예시이다.

* 각 노드가 8GiB RAM을 가지고, 16 코어인 2개의 노드를 가신 클러스터에서 namespace 내의 파드에 대해 CPU에서 request를 100m으로 하고 500m을 초과하지 못하도록 하고 request가 200Mid이고 600Mi를 넘지 못하게 한다.
* spec에서 cpu와 memory request가 정의되지 않았을 때 default CPU limits와 request를 150m으로 정의하고 컨테이너에 대해 Memory default request를 300Mi로 지정한다.

namespace의 total limits가 파드/컨테이너의 limits의 sum보다 작은 경우에 resource에 대한 경합이 일어날 수 있다; 컨테이너나 파드가 생성되지 않을 것이다.

LimitRange에 대한 경합이나 변경은 이미 생성된 resources에 대해 영향을 주지 않을 것이다.

## Limiting Container compute resources

다음 섹션에서는 container level에서 작용하는 LimitRange의 생성에 대해 이야기 하도록 하겠다. 4개의 컨테이너를 가진 파드가 먼저 생성된다; 파드 내의 각 컨테이너는  `spec.resource` configuration이 지정되어 있고 파드 내의 각 컨테이너는  LimitRanger admission controller에 의해서 다르게 다루어진다.

다음의 kubectl 명령어를 통하여 `limitrange-demo`라는 namespace를 생성한다.

```bash
kubectl create namespace limitrange-demo
```

다음은 LimitRange object에 대한 configuration file이다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-container
spec:
  limits:
  - max:
      cpu: "800m"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "99Mi"
    default:
      cpu: "700m"
      memory: "900Mi"
    defaultRequest:
      cpu: "110m"
      memory: "111Mi"
    type: Container
```

이 오브젝트는 컨테이너에 적용하는 Memory/CPU limits에 대한 minimum과 maximum, default cpu/Memory requests, CPU/Memory resources에 대한 default limits을 정의한다.

다음의 kubectl 명령어를 통해 `limit-mem-cpu-per-container` LimitRange를 `limitrange-demo` namespace 안에 생성한다.

```shell
kubectl create -f https://k8s.io/examples/admin/resource/limit-mem-cpu-container.yaml -n limitrange-demo
```

```shell
 kubectl describe limitrange/limit-mem-cpu-per-container -n limitrange-demo
```

```shell
Type        Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---   ---------------  -------------  -----------------------
Container   cpu       100m  800m  110m             700m           -
Container   memory    99Mi  1Gi   111Mi            900Mi          -
```

다음은 LimitRange 기능을 시연하기 위한 4개의 컨테이너를 가진 파드에 대한 configuration file이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt01; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt02
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt02; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
  - name: busybox-cnt03
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt03; sleep 10;done"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt04
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt04; sleep 10;done"]
```

`busybox1` 파드를 생성해보자.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/limit-range-pod-1.yaml -n limitrange-demo
```

### Container spec with valid CPU/Memory requests and limits

`busybox-cnt01` resource configuration을 보자.

```shell
kubectl get  po/busybox1 -n limitrange-demo -o json | jq ".spec.containers[0].resources"
```

```json
{
  "limits": {
    "cpu": "500m",
    "memory": "200Mi"
  },
  "requests": {
    "cpu": "100m",
    "memory": "100Mi"
  }
}
```

* `busybox` 파드 내의 `busybox-cnt01` container는 `requests.cpu=100m`과 `requests.memory=100Mi`를 정의하였다.
* `100m <= 500m <= 800m`이여서 container의 cpu limit (500m)은 허용된 CPU limit range 안에 들어가 있다.
* `99Mi <= 200Mi <= 1Gi`이여서 container의 memory limit (200Mi)는 허용된 Memory limit range 안에 들어가 있다.
* CPU/Memory에 대한 request/limits 비율에 관한 확인절차가 따로 없으므로 container는 유효하고 생성된다.

### Container spec with a valid CPU/Memory requests but no limits

`busybox-cnt02`의 resource configuration을 보도록 하자.

```shell
kubectl get  po/busybox1 -n limitrange-demo -o json | jq ".spec.containers[1].resources"
```

```json
{
  "limits": {
    "cpu": "700m",
    "memory": "900Mi"
  },
  "requests": {
    "cpu": "100m",
    "memory": "100Mi"
  }
}
```

* `busybox1` 파드 내의 `busybox-cnt02` Container는 `requests.cpu=100m`과 `requests.memory=100Mi`를 정의하고 있지만 cpu와 memory에 대한 limits가 없다.
* limits section을 가지고 있지 않는 컨테이너는 limit-mem-cpu-per-container LimitRange 오브젝트에서 정의된 default limits인 `limits.cpu=700m`와 `limits.memory=900Mi`를 이 컨테이너에 주입한다.
* `100m <= 700m <= 800m`이기 때문에 container cpu limit (700m)은 허용된 CPU limit range 내에 있다.
* `99Mi <= 900Mi <= 1Gi`이기 때문에 conatiner memory limit (900Mi)는 허용된 Memory limit range 내에 있다.
* request/limits ratio가 설정되지 않았기 때문에 container는 유효하고 생성된다.

### Container spec with a valid CPU/Memory limits but no requests

`busybox-cnt03`의 resource configuration을 보도록 하자.

```shell
kubectl get  po/busybox1 -n limitrange-demo -o json | jq ".spec.containers[2].resources"
```

```json
{
  "limits": {
    "cpu": "500m",
    "memory": "200Mi"
  },
  "requests": {
    "cpu": "500m",
    "memory": "200Mi"
  }
}
```

* `busybox1` 파드 안에 있는 `busybox-cnt03` 컨테이너는 `limits.cpu=500m`과 `limits.memory=200Mi`로 정의가 되어있지만 cpu와 memory에 대한 `requests`는 정의되어있지 않다.
* request section을 정의하지 않은 컨테이너는 limit-mem-cpu-per-container LimitRange에서 정의된 defaultRequest가 이 limits section을 채우는 데 이용되지는 않지만 container에 의해 정의된 limits가 `limits.cpu=500m`과 `limits.memory=200Mi`로 정의되어 있다.
* `100m <= 500m <= 800m`이기 때문에 container의 cpu limit (500m)은 허용된 CPU limit range 범위 내에 존재한다.
* `99Mi <= 200Mi <= 1Gi`이기 때문에, container의 memory limit (200Mi)는 허용된 Memory limit range 범위 내에 존재한다.
* request/limits의 비율이 설정되지 않았기 때문에 container는 유효하고 생성될 수 있다.

### Container spec with no CPU/Memory requests/limits

`busybox-cnt04`의 resource configuration을 보도록 하자.

```shell
kubectl get  po/busybox1 -n limitrange-demo -o json | jq ".spec.containers[3].resources"
```

```json
{
  "limits": {
    "cpu": "700m",
    "memory": "900Mi"
  },
  "requests": {
    "cpu": "110m",
    "memory": "111Mi"
  }
}
```

* `busybox1` 안의 `busybox-cnt04` Container는 `limits`나 `requests`를 정의하지 않았다.
* limit section을 정의하지 않은 container는 이 request에 대해 limit-mem-cpu-per-container LimitRange에서 정의된 default limit(`limits.cpu=700m`, `limits.memory=900Mi`)이 사용된다.
* request section을 정의하지 않은 container는 limit-mem-cpu-per-container LimitRange에서 정의된 defaultRequest(`requests.cpu=110m`, `requests.memory=111Mi`)가 이 request section을 채우기 위해 사용되었다.
* `100m <= 700m <= 800m`이기 때문에 container cpu limit (700m)은 허용된 CPU limit range 내에 존재한다.
* `99Mi <= 900Mi <= 1Gi`이기 때문에 container memory limit (900Mi)는 허용된 Memory limit range 내에 존재한다.
* request/limits ratio가 설정되지 않았기 때문에 container는 유효하며 생성된다.

`busybox` 파드 내에 있는 모든 컨테이너는 LimitRange validations를 통과하고 이 파드는 유효하며 namespace 내에서 생성된다.

## Limiting Pod compute resources

다음 섹션에서는 Pod level에서 어떻게 resources를 제한하는지에 대해 알아보도록 하겠다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-pod
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    type: Pod
```

`busybox1` 파드를 삭제하지 않고 `limit-mem-cpu-pod` LimitRange를 `limitrange-demo` namespace 안에 생성해보자.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/limit-mem-cpu-pod.yaml -n limitrange-demo
```

limitrange가 생성되고 각 파드에 대해 CPU를 2 Core로 제한하고 Memory를 2Gi로 제한한다.

```shell
limitrange/limit-mem-cpu-per-pod created
```

`limit-mem-cpu-per-pod` limit 오브젝트를 다음의 kubectl 명령어를 통해서 확인해보자.

```shell
kubectl describe limitrange/limit-mem-cpu-per-pod -n limitrange-demo
```

이제 `busybox2` 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt01; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt02
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt02; sleep 10;done"]
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
  - name: busybox-cnt03
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt03; sleep 10;done"]
    resources:
      limits:
        memory: "200Mi"
        cpu: "500m"
  - name: busybox-cnt04
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello from cnt04; sleep 10;done"]
```

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/limit-range-pod-2.yaml -n limitrange-demo
```

`busybox2` Pod definition은 `busybox1`과 동일하지만 이제 파드의 resource가 제한되기 때문에 에러가 발생한다.

```shell
Error from server (Forbidden): error when creating "limit-range-pod-2.yaml": pods "busybox2" is forbidden: [maximum cpu usage per Pod is 2, but limit is 2400m., maximum memory usage per Pod is 2Gi, but limit is 2306867200.]
```

```shell
kubectl get  po/busybox1  -n limitrange-demo -o json | jq ".spec.containers[].resources.limits.memory" 
"200Mi"
"900Mi"
"200Mi"
"900Mi"
```

`busybox2` 파드는 container의 총 memory limit가 LimitRange에서 정의된 limit보다 크기 때문에 클러스터에서 허용되지 않을 것이다. `busybox1`은 LimitRange를 생성하기 전에 cluster에서 생성되고 허용되었기 때문에 evicted 되지 않을 것이다.

## Limiting Storage resources

사용자는 LimitRange를 사용하여 namespace 내에 있는 각 PersistentVolumeClaim에서 requested된 storage resources의 minimum, maximum size를 제한할 수 있다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi
```

`kubectl create` 명령어를 통하여 YAML을 등록한다.

```shell
kubectl create -f https://k8s.io/examples/admin/resource/storagelimits.yaml -n limitrange-demo
```

```shell
limitrange/storagelimits created
```

생성된 오브젝트를 확인해보자.

```shell
kubectl describe limits/storagelimits  
```

결과는 다음과 같을 것이다.

```shell
Name:                  storagelimits
Namespace:             limitrange-demo
Type                   Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----                   --------  ---  ---  ---------------  -------------  -----------------------
PersistentVolumeClaim  storage   1Gi  2Gi  -                -              -
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-limit-lower
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```shell
kubectl create -f https://k8s.io/examples/admin/resource//pvc-limit-lower.yaml -n limitrange-demo
```

`requests.storage`가 LimitRange의 Min value보다 작은 값으로 PVC를 생성하려 하면 server에서 에러가 발생한다.

```shell
Error from server (Forbidden): error when creating "pvc-limit-lower.yaml": persistentvolumeclaims "pvc-limit-lower" is forbidden: minimum storage usage per PersistentVolumeClaim is 1Gi, but request is 500Mi.
```

마찬가지로 `requests.storage`가 LimitRange에서의 Max value보다 크게 될 경우 에러를 발생한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-limit-greater
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```shell
kubectl create -f https://k8s.io/examples/admin/resource/pvc-limit-greater.yaml -n limitrange-demo
```

```shell
Error from server (Forbidden): error when creating "pvc-limit-greater.yaml": persistentvolumeclaims "pvc-limit-greater" is forbidden: maximum storage usage per PersistentVolumeClaim is 2Gi, but request is 5Gi.
```

##  Limits/Requests Ratio

`LimitRangeItem.maxLimitRequestRatio`가 `LimitRangeSpec`에 지정되어 있다면 named resource는 *limit/request*가 정해진 값보다 작거나 같지 않은, 0이 아닌 request와 limit을 가지고 있어야 한다.

다음의 `LimitRange`는 namespace 내의 어떤 파드던지 memory limit이 memory request의 최대 2배가 되도록 제한하는 것이다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-memory-ratio-pod
spec:
  limits:
  - maxLimitRequestRatio:
      memory: 2
    type: Pod
```

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/limit-memory-ratio-pod.yaml -n limitrange-demo
```

```shell
kubectl describe limitrange/limit-memory-ratio-pod -n limitrange-demo
```

```shell
Name:       limit-memory-ratio-pod
Namespace:  limitrange-demo
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Pod         memory    -    -    -                -              2
```

`requests.memory=100Mi`이고 `limits.memory=300Mi`인 파드를 생성해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox3
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    resources:
      limits:
        memory: "300Mi"
      requests:
        memory: "100Mi"
```

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/limit-range-pod-3.yaml -n limitrange-demo
```

파드의 생성이 ratio(3)가 `limit-memory-ratio-pod` LimitRange(2)보다 크기 때문에 실패한다.

```shell
Error from server (Forbidden): error when creating "limit-range-pod-3.yaml": pods "busybox3" is forbidden: memory max limit to request ratio per Pod is 2, but provided ratio is 3.000000.
```

### Clean up

모든 resource를 free하기 위해 `limitrange-demo` namespace를 삭제한다.

```shell
kubectl delete ns limitrange-demo
```

## Examples

- [namespace별로 어떻게 compute resources를 제한하는지에 대한 튜토리얼](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)을 보아라.
- [어떻게 storage consumption을 제한하는지](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/#limitrange-to-limit-requests-for-storage) 확인해보아라.
- [namespace별로 quota를 설정하는 자세한 예시](https://kubernetes.io/docs/tasks/administer-cluster/quota-memory-cpu-namespace/)를 보아라.

## What's next

더 많은 정보는 [LimitRanger design doc](https://git.k8s.io/community/contributors/design-proposals/resource-management/admission_control_limit_range.md)를 참조하라.