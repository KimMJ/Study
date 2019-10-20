# Resource Quotas

몇몇 유저나 팀이 클러스터의 고정된 수의 노드들을 공유하고 있다고 할 때 하나의 팀이 공정한 resource의 share보다 더 많은 양을 사용할 수도 있다.

Resource quotas는 administrators가 이를 해결하는 툴이다.

`ResourceQuota` 오브젝트로 정의된 resource quota는 namespace별로 resource consumption의 총합을 제한하는 constraints를 제공한다. 이는 namespace에서 생성될 수 있는 오브젝트의 양을 type별로 제한할 수 있다. 또한 프로젝트 내에서 소비될 수 있는 compute resources의 총량을 제한할 수 있다.

resource quotas는 다음과 같이 동작한다.

* 다른 팀은 다른 namespace에서 작업한다. 현재 이는 자발적으로 행해지고 있지만 계획된 ACL(Access Control List)을 통하여 이를 의무로 변경하도록 지원해야 한다.
* 각 namespace에 대해서 administrator는 하나의 `ResourceQuota`를 생성한다.
* 유저는 namespace 안에서 파드나 서비스 등의 resources를 생성하고 quota system은 사용량을 추적하여 `ResourceQuota`에 정의된 hard resource limits를 초과하지 않는지 확인한다.
* quota constraint를 위반하는 resource를 생성하거나 업데이트 하면 request는 HTTP status code `403 FORBIDDEN`으로 실패할 것이고 위반한 constraint에 대한 설명이 메시지로 출력될 것이다.
* quota가 `cpu`나 `memory`같은 compute resources에 대해 namespace에서 활성화 되어 있다면 유저는 반드시 이러한 values에 대해서 requests나 limits를 지정해 주어야 한다; 그렇지 않으면, quota system은 파드의 생성을 거부할 것이다. Hint: `LimitRanger` admission controller를 사용하여 conpute resource requirements가 없는 파드에 대해 defaults를 강제로 생성하라. 어떻게 이 문제를 피하는지에 대해 [walkthrough](https://kubernetes.io/docs/tasks/administer-cluster/quota-memory-cpu-namespace/)에서의 예시를 확인해 보아라.

namespaces와 quotas를 사용하여 생성된 policy의 예시는 다음과 같을 수 있다.

* 32 GiB RAM, 16 cores를 가지고 있는 클러스터에서 team A가 20 GiB, 10 cores를 사용하고 B가 10 GiB, 4 cores를 사용하면 2 GiB와 2 cores는 나중의 allocation을 위해서 reserve 되어 있다.
* "testing" namespace를 1 core, 1 GiB RAM을 사용하도록 제한한다. "production" namespace를 어떤 양이든 사용할 수 있도록 한다.

클러스터에서 총 capacity가 namespaces에서의 quotas의 합보다 적은 경우에 resource에 대한 경합이 있을 수 있다. 이는 first-come-first-served를 기준으로 다루어진다.

quota에 대한 contention이나 changes는 이미 생성된 resources에 대해서는 영향을 주지 않는다.

## Enabling Resource Quota

Resource Quota 지원은 많은 Kubernetes distributions에서 default로 활성화되어있다. 이는 apiserver가 `--enable-admission-plugin=` 플래그가 arguments중 하나로 `ResourceQuota`를 가지고 있을 때 활성화된다.

resource quota는 특정한 namespace에서 `ResourceQuota`가 있을 때 해당 namespace에서 집행된다.

## Compute Resource Quota

주어진 namespace에서 requested 될 수 있는 [compute resources](https://kubernetes.io/docs/user-guide/compute-resources)의 총합을 제한할 수 있다.

다음의 resource type들을 지원한다.

| Resource Name     | Description                                                  |
| :---------------- | :----------------------------------------------------------- |
| `limits.cpu`      | 모든 non-terminal state에서의 파드들에서 CPU limits의 합계는 이 값을 넘을 수 없다. |
| `limits.memory`   | 모든 non-terminal state에서의 파드들에서 memory limits의 합계는 이 값을 넘을 수 없다. |
| `requests.cpu`    | 모든 non-terminal state에서의 파드들에서 CPU requests의 합계는 이 값을 넘을 수 없다. |
| `requests.memory` | 모든 non-terminal stated에서의 파드들에서 memory requests의 합계는 이 값을 넘을 수 없다. |

### Resource Quota For Extended Resources

release 1.10에서 위에 언급한 resources에 추가적으로 [extended resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)에 대한 quota 지원이 추가되었다.

extended resources에 대한 overcommit이 허용되지 않아서 quota에서 동일한 extended resource에 대한 `requests`와 `limits`를 지정하는 것은 말이 되지 않는다. 따라서 extended resources에 대해서는 현재 `requests.` prefix만을 지원한다.

GPU resource를 예로 들어보면 `nvidia.com/gpu`라는 resource name을 가지고 있다고 했을 때 사용자는 namespace에서 총 GPU requested 수가 4로 제한이 되게 하고 싶을 경우 quota를 다음과 같이 정의하면 된다.

- `requests.nvidia.com/gpu: 4`

자세한 정보는 [Viewing and Setting Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/#viewing-and-setting-quotas)를 참조하라.

## Storage Resource Quota

사용자는 주어진 namespace에서 requested 될 수 있는 [storage resources](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)의 총합을 제한할 수 있다.

추가적으로 사용자는 연관된 storage-class를 기반으로 storage resources의 consumption에 제한을 둘 수 있다.

| Resource Name                                         | Description                                                  |
| :---------------------------------------------------- | :----------------------------------------------------------- |
| `requests.storage`                                    | 모든 persistent volume claims에 걸쳐서 storage requests의 총합은 이 값을 넘을 수 없다. |
| `persistentvolumeclaims`                              | namespace 내에서 존재할 수 있는 [persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)의 숫자이다. |
| `.storageclass.storage.k8s.io/requests.storage`       | storage-class-name과 관련된 모든 persistent volume claims에 대해서 storage requests의 총합은 이 값을 넘을 수 없다. |
| `.storageclass.storage.k8s.io/persistentvolumeclaims` | storage-class-name과 관련된 모든 persistent volume claims에서 namespace 내에서 존재할 수 있는 [persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)의 총 개수이다. |

예를 들어, operator가 `gold` storage class를 가진 quota storage를 `bronze` storage class와 분리하고 싶으면 operator는 quota를 다음과 같이 정의할 수 있다.

- `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
- `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

1.8 release에서 ephemeral storage에 대한 quota가 alpha 기능으로 추가되었다.

| Resource Name                | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `requests.ephemeral-storage` | namespace 내의 모든 파드에 대해 local ephemeral storage requests의 총합은 이 값을 넘을 수 없다. |
| `limits.ephemeral-storage`   | namespace 내의 모든 파드에 대해 local ephemeral storage limits의 총합은 이 값을 넘을 수 없다. |

## Object Count Quota

1.9 release에서는 다음의 syntax를 사용하는 모든 standard namespaced resource types에 대해서 quota를 지원한다.

- `count/<resource>.<group>`

다음은 유저가 object count quota로 설정할 수 있는 resources들의 예시이다.

- `count/persistentvolumeclaims`
- `count/services`
- `count/secrets`
- `count/configmaps`
- `count/replicationcontrollers`
- `count/deployments.apps`
- `count/replicasets.apps`
- `count/statefulsets.apps`
- `count/jobs.batch`
- `count/cronjobs.batch`
- `count/deployments.extensions`

1.15 release에서는 동일한 syntax로 custom resources에 대한 지원을 한다. 예를 들어, `example.com` API group 안에서의 `widgets` custom resource에서의 quota를 생성하기 위해선 `count/widgets.example.com`을 사용하면 된다.

`count/*` resource quota를 사용하면 오브젝트는 server storage에 존재할 경우 quota가 부과되게 된다. 이런 quotas에 대한 종류는 storage resources의 exhaustion을 방지하는 데 유용할 수 있다. 예를 들어, server의 크기에 따라서 secrets의 quota 수를 조절하고 싶을 것이다. 클러스터 안에서의 너무 많은 secrets는 servers와 controllers가 시작하길 방해한다! 사용자는 poorly configured cronjob이 너무 많은 jobs를 namespace안에 생성하여 denial of service를 야기하지 않도록 하기위해 quota jobs를 선택할 것이다.

1.9 release 이전 버전에서는 제한된 resources에 대해서만 generic object count quota를 할 수 있었다. 여기에 추가적으로 특정 resources의 quota를 type에 따라서 constraint를 줄 수 있다.

다음은 지원하는 type들이다.

| Resource Name            | Description                                                  |
| :----------------------- | :----------------------------------------------------------- |
| `configmaps`             | namespace에서 존재할 수 있는 config maps의 총 숫자이다.      |
| `persistentvolumeclaims` | namespace에서 존재할 수 있는 [persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)의 총 숫자이다. |
| `pods`                   | namespace내에서 존재하는 non-terminal state인 파드의 총 숫자이다. `.status.phase in (Failed, Succeeded)`가 true이면 파드는 terminal state이다. |
| `replicationcontrollers` | namespace 내에서 존재할 수 있는 replication controllers의 총 숫자이다. |
| `resourcequotas`         | namespace 내에서 존재할 수 있는 [resource quotas](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#resourcequota)의 총 수 이다. |
| `services`               | namespace 내에서 존재할 수 있는 services의 총 갯수이다.      |
| `services.loadbalancers` | namespace 내에서 존재할 수 있는 load balancer type의 service의 총 갯수이다. |
| `services.nodeports`     | namespace 내에서 존재할 수 있는 node port type의 service의 총 갯수이다. |
| `secrets`                | namespace 내에서 존재할 수 있는 secrets의 총 갯수이다.       |

예를 들어, `pods` quota는 count를 하고 terminal이 아닌 단일 namespace에서 생성된 `pods`의 최대 갯수를 집행한다. namespace에서의 `pods` quota를 설정하여 유저가 작지만 많은 pods를 생성하여 클러스터의 Pod IPs를 exhausts하는 것을 방지하고 싶을 것이다.

## Quota Scopes

각 quota는 관련된 스코프를 가진다. quota는 나열된 스코프와의 교집합과 일치하는 resource의 사용량을 측정하기만 할 것이다.

스코프가 quota에 추가되면 해당 스코프와 관련되어 지원하는 resource의 수를 제한한다. resources는 allowed set의 밖인 quota에서 지정되어 있어 validiation error의 결과를 불러일으킨다.

| Scope            | Description                                          |
| :--------------- | :--------------------------------------------------- |
| `Terminating`    | `.spec.activeDeadlineSeconds >= 0`인 파드와 매칭됨   |
| `NotTerminating` | `.spec.activeDeadlineSeconds is nil`인 파드와 매칭됨 |
| `BestEffort`     | service의 quality가 best effort인 파드와 매칭됨      |
| `NotBestEffort`  | service의 quality가 best effort가 아닌 파드와 매칭됨 |

`BestEffort` 스코프는 다음의 resource를 추적하여 quota의 제한을 준다: `pods`. `Termination`, `NotTerminating`, `NotBestEffort` 스코프는 다음의 resources를 추적하여 quota의 제한을 준다.

- `cpu`
- `limits.cpu`
- `limits.memory`
- `memory`
- `pods`
- `requests.cpu`
- `requests.memory`

### Resource Quota Per PriorityClass

**FEATURE STATE: ** `Kubernetes 1.12` - beta

파드는 지정된 [priority](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#pod-priority)로 생성될 수 있다. 당신은 파드의 priority에 따라서 파드가 system resources를 소비하는 양을 조절할 수 있다. 이는 quota spec의 `scopeSelector` 필드를 사용하여 할 수 있다.

quota는 quota spec에서의 `scopeSelector`가 파드를 선택하였을 때에만 매칭이 되고 소비된다.

이 예시는 quota object를 생성하여 이를 지정한 priorities에 맞게 매칭한다. 예시의 동작은 다음과 같다.

* 클러스터에서의 파드는 3가지 priority classes가 있다. "low", "medium", "high"
* 하나의 quota object는 각 priority에 대해 생성된다.

다음의 YAML 파일을 `quota.yml`로 저장하라.

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
```

 YAML을 `kubectl create`를 통해 apply 하라.

```shell
kubectl create -f ./quota.yml
```

```shell
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
```

`kubectl describe quota`를 사용하여 `Used` quota가 `0`임을 확인하라.

```shell
kubectl describe quota
```

```shell
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     1k
memory      0     200Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

"high" priority를 가지는 파드를 생성한다. 다음의 YAML을 `high-priority-pod.yml`로 저장하라.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

이를 `kubectl create`로 apply하라.

```shell
kubectl create -f ./high-priority-pod.yml
```

"high" priority quota인 `pods-high`에 대해 "Used"가 바뀌었고 나머지 두 quota는 바뀌지 않았음을 확인하라.

```shell
kubectl describe quota
```

```shell
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         500m  1k
memory      10Gi  200Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

`scopeSelector`는 `operator` 필드에서 다음의 value들을 지원한다.

- `In`
- `NotIn`
- `Exist`
- `DoesNotExist`

## Requests vs Limits

compute resources를 할당할 때 각 컨테이너는 CPU나 memory에 대해 request와 limit value를 설정할 수 있다. quota는 이러한 value들에 대해 quota를 설정할 수 있다.

quota가 `requests.cpu`나 `requests.memory`에 대해서 지정된 value를 가질 때 모든 incoming containers는 명시적으로 이런 resources에 대해 request를 설정해 주어야 한다. quota가 `limits.cpu`나 `limits.memory`에 대해서 지정된 value를 가질 때 모든 incoming container는 명시적으로 이러한 resources에 대해 limit를 설정해 주어야 한다.

## Viewing and Setting Quotas

kubectl은 quota에 대해 생성, 업데이트, 보기가 가능하다.

```shell
kubectl create namespace myspace
```

```shell
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
EOF
```

```shell
kubectl create -f ./compute-resources.yaml --namespace=myspace
```

```shell
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    pods: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
```

```shell
kubectl create -f ./object-counts.yaml --namespace=myspace
```

```shell
kubectl get quota --namespace=myspace
```

```shell
NAME                    AGE
compute-resources       30s
object-counts           32s
```

```shell
kubectl describe quota compute-resources --namespace=myspace
```

```shell
Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
```

```shell
kubectl describe quota object-counts --namespace=myspace
```

```shell
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
pods                    0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

kubectl은 모든 표준 namespaced resources에 대해서 `count/<resource>.<group>`의 문법을 사용하여 object count quota를 지원한다.

```shell
kubectl create namespace myspace
```

```shell
kubectl create quota test --hard=count/deployments.extensions=2,count/replicasets.extensions=4,count/pods=3,count/secrets=4 --namespace=myspace
```

```shell
kubectl run nginx --image=nginx --replicas=2 --namespace=myspace
```

```shell
kubectl describe quota --namespace=myspace
```

```shell
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.extensions  1     2
count/pods                    2     3
count/replicasets.extensions  1     4
count/secrets                 1     4
```

## Quota and Cluster Capacity

`ResourceQuotas`는 cluster capacity와 독립적이다. 절대적인 unit으로 표현이 된다. 따라서 클러스터에 노드를 추가할 때 이는 자동으로 각 namespace에 대해서 더 많은 resource를 소비할 수 있도록 해주지 않는다.

다음과 같이 가끔 더 복잡한 policy가 필요한 경우가 있다.

* 몇몇 팀에 대해서 전체 클러스터 resources를 균등하게 나누기.
* 각 tenant에 대해 필요에 따라 resource의 사용량을 늘리지만 실수로 resource exhaustion이 발생하지 않도록 일반적인 상한선을 두는 것.
* 어떤 namespace에서의 요구를 감지하여 노드를 추가하고 quota를 늘리는 것.

이러한 policy는 `ResourceQuotas`를 building blocks처럼 사용하여 구현할 수 있다. "controller"가 quota usage를 관찰하고 다른 시그널에 따라서 각 namespace의 quota hard limits를 조정하는 식으로 작성하면 된다.

resource quota는 cluster resources의 총합을 나누는 것이지만 노드에 대한 제한을 생성하지 않는다는 것에 주의하라: 몇몇 namespaces에서 오는 파드들은 같은 노드에서 뜰 것이다.

## Limit Priority Class consumption by default

"cluster-services"처럼 특정 priority에서 파드가 일치하는 quota object가 존재할 때만 namespace 내에서 허용될 필요가 있을 수 있다.

이러한 매커니즘으로 operators는 특정한 high priority classes의 사용을 제한된 수의 namespaces로 제한할 수 있다. 그리고 모든 namespace는 이 priority classes를 기default로 소비할 수 있게 될 것이다.

이를 적용하기 위해 kube-apiserver flag `--admission-control-config-file`은 반드시 다음의 configuration file에 path를 전달해야한다.

```yaml
apiVersion: apiserver.k8s.io/v1alpha1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: resourcequota.admission.k8s.io/v1beta1
    kind: Configuration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: PriorityClass 
        operator: In
        values: ["cluster-services"]
```

이제 "cluster-services" 파드는 `scopeSelector`로 매칭이 되는 quota object가 존재할 때에만 이러한 namespaces에 대해서만 허용이 된다.

```yaml
    scopeSelector:
      matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values: ["cluster-services"]
```

더 많은 정보는 [LimitedResources](https://github.com/kubernetes/kubernetes/pull/36765) 와 [Quota support for priority class design doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/pod-priority-resourcequota.md) 를 참고하라.

## Example

[detailed example for how to use resource quota](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)를 보아라.

## What's next

[ResourceQuota design doc](https://git.k8s.io/community/contributors/design-proposals/resource-management/admission_control_resource_quota.md)를 통해 더 많은 정보를 얻을 수 있다.