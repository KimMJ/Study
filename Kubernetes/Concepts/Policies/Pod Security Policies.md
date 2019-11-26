# Pod Security Policies

**FEATURE STATE: ** `Kubernetes v1.16` - beta

Pod Security Policies는 파드의 생성과 업데이트 시 authorization을 세세하게 할 수 있도록 해준다.

## What is a Pod Security Policy?

Pod Security Policy는 pod specification에서 security sesitive인 면을 제어하는 cluster-level의 resource이다. `PodSecurityPolicy` 오브젝트는 파드가 반드시 system이 받아들여질 수 있고, 관련된 필드들에 대해 default를 설정하여 동작하도록 conditions를 정의하는 것이다. administrator는 다음을 제어할 수 있다.

| Control Aspect                                    | Field Names                                                  |
| :------------------------------------------------ | :----------------------------------------------------------- |
| Running of privileged containers                  | [`privileged`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#privileged) |
| Usage of host namespaces                          | [`hostPID`, `hostIPC`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces) |
| Usage of host networking and ports                | [`hostNetwork`, `hostPorts`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces) |
| Usage of volume types                             | [`volumes`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#volumes-and-file-systems) |
| Usage of the host filesystem                      | [`allowedHostPaths`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#volumes-and-file-systems) |
| White list of FlexVolume drivers                  | [`allowedFlexVolumes`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#flexvolume-drivers) |
| Allocating an FSGroup that owns the pod’s volumes | [`fsGroup`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#volumes-and-file-systems) |
| Requiring the use of a read only root file system | [`readOnlyRootFilesystem`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#volumes-and-file-systems) |
| The user and group IDs of the container           | [`runAsUser`, `runAsGroup`, `supplementalGroups`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#users-and-groups) |
| Restricting escalation to root privileges         | [`allowPrivilegeEscalation`, `defaultAllowPrivilegeEscalation`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#privilege-escalation) |
| Linux capabilities                                | [`defaultAddCapabilities`, `requiredDropCapabilities`, `allowedCapabilities`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#capabilities) |
| The SELinux context of the container              | [`seLinux`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#selinux) |
| The Allowed Proc Mount types for the container    | [`allowedProcMountTypes`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#allowedprocmounttypes) |
| The AppArmor profile used by containers           | [annotations](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#apparmor) |
| The seccomp profile used by containers            | [annotations](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#seccomp) |
| The sysctl profile used by containers             | [`forbiddenSysctls`,`allowedUnsafeSysctls`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#sysctl) |

## Enabling Pod Security Policies

Pod security policy control은 [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)에서 optional(하지만 추천하지는 않는다)로 구현되어있다. PodSecurityPolicies는 [enabling the admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-control-plug-in)에 의해서 시행될 수 있지만 어떤 policy에 대한 authorizing도 없이 이를 진행하는 것은 클러스터에서 모든 파드가 생성되는 것을 막을 수도 있다.

pod security policy API (`policy/v1beta1/podsecuritypolicy`)가 admission controller에 독립적으로 활성화되어있기 때문에, 클러스터는 admission controller에서 활성화 되기 전에 policy를 추가하고 authorize하는 것을 추천한다.

## Authorizing Policies

PodSecurityPolicy resource가 생성되었을 때에는 아무것도 하지 않는다. 이를 사용하기 위해선 유저나 해당 파드의 [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)가 반드시 policy를 사용하도록 policy에서 `use` verb를 허용하여 authorize 되어야 한다.

대부분의 Kubernetes 파드는 유저에 의해서 직접 생성되지 않는다. 대신, 파드들은 Deployment, ReplicaSet, 또는  controller manager를 통한 다른 templated controller에 의해 간접적으로 생성된다. policy에 대해 controller access를 주는 것은 controller에 의해 생성된 모든 파드에 대해 권한을 주는 것이며, 때문에 policy를 authorizing하는 좋은 방법은 파드의 service account에 대해 접근 권한을 주는 것이다. ([example](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#run-another-pod)참고)

### Via RBAC

RBAC은 standard Kubernetes authorization mode이며, policy의 사용을 authorize하는 데 쉽게 사용할 수 있다.

먼저 `Role`이나 `ClusterRole`은 원하는 policy에 대해 `use` access를 가지고 있어야 한다. access를 부여하는 규칙은 다음과 같다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role name>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <list of policies to authorize>
```

그러면 `(Cluster)Role`은 authorized user에게 바운드된다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <binding name>
roleRef:
  kind: ClusterRole
  name: <role name>
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize specific service accounts:
- kind: ServiceAccount
  name: <authorized service account name>
  namespace: <authorized pod namespace>
# Authorize specific users (not recommended):
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <authorized user name>
```

`RoleBinding`(`ClusterRoleBinding`이 아님)이 사용되면 binding으로 같은 네임스페이스에서 동작하고 있는 파드에 대해서만 usage를 부여할 것이다. 이는 system groups와 짝을 맞추어 namespace에서 동작하는 파드에 대해서 access를 부여할 수 있다.

```yaml
# Authorize all service accounts in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts
# Or equivalently, all authenticated users in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```

더 많은 RBAC bindings의 예시는 [Role Binding Examples](https://kubernetes.io/docs/reference/access-authn-authz/rbac#role-binding-examples)을 참고하라. PodSecurityPolicy에 대한 authorizing의 완전한 예시들은 [아래](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#example)를 보아라.

### Troubleshooting

[Controller Manager](https://kubernetes.io/docs/admin/kube-controller-manager/)는 반드시 [the secured API port](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)에서 동작해야 하고 superuser 권한이 반드시 없어야한다. 그렇지않으면 requests는 authentication과 authorization modules를 지나칠 것이고 모든 PodSecurityPolicy object는 허용되며 user는 privileged containers를 생성할 수 있게 된다. Controller Manager authorization을 설정하는 것에 대한 자세한 사항은 [Controller Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles)을 확인하라.

## Policy Order

파드의 생성과 업데이트에 대한 제한을 추가하기 위해 pod security policies는 이것이 관리하는 많은 필드에 대해서 default values를 제공할 수 있다. 여러 policies들이 사용가능할 경우 pod security policy controller는 다음의 criteria를 따라 policy를 선택한다.

1. pod를 as-is로 하게 해주는 PodSecurityPolicies는 파드의 default를 바꾸거나 변형하지 않는 것이 좋다. 이러한 non-mutating PodSecurityPolicies의 순서는 상관이 없다.
2. pod가 반드시 defaulted 되거나 mutated 되어야 한다면 파드를 허용하는 첫 PodSecurityPolicy가 선택된다.

> Note: 업데이트 동작 동안(pod specs의 변형이 허용되지 않을 때) non-mutating PodSecurityPolicies들만 파드를 validate하는 데 사용된다.

## Example

이 예시는 당신이 PodSecurityPolicy admission controller가 활성화 되어 있는 클러스터를 가지고 있고 cluster admin privileges를 가지고 있다고 가정한다.

### Set up

이 예시에서 사용할 namespace와 service account를 설정한다. 우리는 non-admin user인 mock을 service account로 사용할 것이다.

```shell
kubectl create namespace psp-example
kubectl create serviceaccount -n psp-example fake-user
kubectl create rolebinding -n psp-example fake-editor --clusterrole=edit --serviceaccount=psp-example:fake-user
```

어느 유저를 사용할지 명확하게 하고 몇가지 typing을 저장하기 위해 2개의 aliases를 생성한다.

```shell
alias kubectl-admin='kubectl -n psp-example'
alias kubectl-user='kubectl --as=system:serviceaccount:psp-example:fake-user -n psp-example'
```

### Create a policy and a pod

example PodSecurityPolicy object를 파일에서 정의한다. 이 policy는 간단하게 privileged pods의 생성을 막는 것이다.

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

그리고 kubectl로 생성한다.

```shell
kubectl-admin create -f example-psp.yaml
```

이제 unprivileged user로 simple pod를 생성해보자.

```shell
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: unable to validate against any pod security policy: []
```

**어떤 상황이 벌어졌는가?** PodSecurityPolicy가 생성되었음에도 파드의 service account나 `fake-user`도 새로운 policy에 대해 permission을 가지고 있지 않다.

```shell
kubectl-user auth can-i use podsecuritypolicy/example
no
```

example policy에서 `fake-user`에게 `use` verb를 부여하는 rolebinding을 생성하자.

> Note: 이는 추천하는 방법은 아니다. [next section](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#run-another-pod)이 더 선호되는 방식이다.

```shell
kubectl-admin create role psp:unprivileged \
    --verb=use \
    --resource=podsecuritypolicy \
    --resource-name=example
role "psp:unprivileged" created

kubectl-admin create rolebinding fake-user:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=psp-example:fake-user
rolebinding "fake-user:psp:unprivileged" created

kubectl-user auth can-i use podsecuritypolicy/example
yes
```

이제 다시 pod를 생성해보자.

```shell
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
EOF
pod "pause" created
```

예상한 대로 동작하였다. 하지만 privileged pod에 대한 생성 시도는 여전히 거부된다.

```shell
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      privileged
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
      securityContext:
        privileged: true
EOF
Error from server (Forbidden): error when creating "STDIN": pods "privileged" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```

다음으로 넘어가기 전에 파드를 삭제하자.

```shell
kubectl-user delete pod pause
```

### Run another pod

약간 다르게 다시 시도해보자.

```shell
kubectl-user run pause --image=k8s.gcr.io/pause
deployment "pause" created

kubectl-user get pods
No resources found.

kubectl-user get events | head -n 2
LASTSEEN   FIRSTSEEN   COUNT     NAME              KIND         SUBOBJECT                TYPE      REASON                  SOURCE                                  MESSAGE
1m         2m          15        pause-7774d79b5   ReplicaSet                            Warning   FailedCreate            replicaset-controller                   Error creating: pods "pause-7774d79b5-" is forbidden: no providers available to validate pod request
```

**어떤 일이 발생했는가?** 우리는 이미 `psp:unprivileged` role을 `fake-user`에 bound하였다. 왜 우리는 `Error creating: pods "pause-7774d79b5-" is forbidden: no providers available to validate pod request` 에러를 받았을까?

정답은 `replicaset-controller`의 소스에 있다. fake-user는 성공적으로 deployment를 생성했지만(성공적으로 replicaset을 생성하였다), replicaset이 파드를 생성하려 할 때 example podsecuritypolicy를 사용할 권한이 없기 때문이다.

이를 고치기 위해 `psp:unprivileged` role을 파드의 service account로 대신 bind시켜라. 이 경우에(아직 이를 지정하지 않았기 때문에) service account는 `default`이다.

```shell
kubectl-admin create rolebinding default:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=psp-example:default
rolebinding "default:psp:unprivileged" created
```

이제 몇분 후 재시도하면 replicaset-controller는 성공적으로 파드를 생성할 것이다.

```shell
kubectl-user get pods --watch
NAME                    READY     STATUS    RESTARTS   AGE
pause-7774d79b5-qrgcb   0/1       Pending   0         1s
pause-7774d79b5-qrgcb   0/1       Pending   0         1s
pause-7774d79b5-qrgcb   0/1       ContainerCreating   0         1s
pause-7774d79b5-qrgcb   1/1       Running   0         2s
```

### Clean up

대부분의 example resources를 제거하기 위해 namespace를 삭제하라.

```shell
kubectl-admin delete ns psp-example
namespace "psp-example" deleted
```

`PodSecurityPolicy` resources는 namespaced가 아니기 때문에 반드시 따로 지워져야 함을 주의하라.

```shell
kubectl-admin delete psp example
podsecuritypolicy "example" deleted
```

### Example Policies

pod security policy admission controller를 사용하지 않는 것과 같은 최소한의 restricted policy을 생성할 수 있다.

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

이는 유저가 unprivileged user로 동작하도록 하여 root권한을 가질 가능성을 막고, 몇가지 security mechanisms를 사용하도록 강제하는 restrictive policy의 예시이다.

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # Required to prevent escalations to root.
  allowPrivilegeEscalation: false
  # This is redundant with non-root + disallow privilege escalation,
  # but we can provide it for defense in depth.
  requiredDropCapabilities:
    - ALL
  # Allow core volume types.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # Assume that persistentVolumes set up by the cluster admin are safe to use.
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # Require the container to run without root privileges.
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

## Policy Reference

### Privileged

**Privileged** - 파드 내 컨테이너가 privileged mode를 활성화 할 수 있는지를 결정한다. default로 컨테이너는 호스트의 어떤 디바이스에도 접근할 수 없지만 "privileged" 컨테이너는 호스트의 모든 디바이스에 접근할 수 있다. 이는 컨테이너가 호스트에서 돌아가고 있는 프로세스와 같은 거의 모든 access를 가질 수 있도록 한다. 이는 network stack 제어나 디바이스 접근과 같은 linux capabilities를 사용하고자 할 때 유용하다.

### Host namespaces

**HostPID** - 파드 컨테이너가 host process ID namespace를 공유할 수 있는지 관리한다.  ptrace와 함께 짝을 이룰 때 이는 컨테이너 바깥의 privileges로 확대되도록 사용할 수 있음에 유의하라. (ptrace는 default로 forbidden이다.)

**HostIPC** - 파드 컨테이너가 host IPC namespace를 공유할 수 있을지 관리한다.

**HostNetwork** - 파드가 node network namespace를 사용할 수 있는지를 관리한다. 이는 loopback device, localhost를 listening하는 services 에 대한 pod access를 주게 되고, 동일 노드 내의 다른 파드의 network activity를 옅볼 수 있다.

**HostPorts** - host network namespace에서 사용가능한 포트의 whitelist 범위를 제공한다. `HostPortRange`의 리스트에 `min`(inclusive), `max`(inclusive)로 정의가 되어있다. defaults는 host ports를 사용가능하지 않다.

**AllowedHostPaths** - [Volumes and file systems](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#volumes-and-file-systems)를 확인하라.

### Volumes and file systems

**Volumes** - 허용되는 volume types에 대해 whitelist를 제공한다. 허용되는 values는 volume을 생성할 때 정의한 volume sources와 일치하다. volume types의 complete list를 알고싶으면 [Types of Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)를 참고하라. 추가적으로 `*`는 모든 volume types를 허용하는 것이다.

새로운 PSP에 대해 허용되는 volumes의 **recommend minimum set**은 다음과 같다.

- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- secret
- projected

> Warning: PodSecurityPolicy는 `PersistentVolumeClaim`에 의해 참조될 수 있는 `PersistentVolume` 오브젝트의 타입을 제한하지 않는다. 신뢰할 수 있는 유저만이 `PersistentVolume` 오브젝트를 생성할 권한을 가진다.

**FSGroup** - 몇가지 volumes에 적용할 supplemental group을 관리한다.

* MustRunAs - 최소 하나의 `range`가 지정되어야 한다. 첫 range의 minimum value를 default로 사용한다. 모든 range에 대해서 검증한다.
* MayRunAs - 최소 하나의 `range`가 지정되어야 한다. default를 제공하지 않고 `FSGroups`가 unset일 수 있도록 한다. `FSGroups`가 set이라면 모든 range에 대해 검증한다.
* RunAsAny - default를 제공하지 않는다. 어느 `fsGroup` IP던지 지정될 수 있다.

**AllowedHostPaths** - hostPath volumes에 의해 사용될 수 있는 host paths의 whitelist를 지정한다. empty list는 host paths의 사용에 제한이 없다는 것을 의미한다. 이는 단일 `pathPrefix` 필드가 있는 오브젝트의 리스트를 통해 정의되어있다. `pathPrefix`는 hostPath volumes가 허용된 prefix로 시작하는 path를 마운트 할 수 있도록 한다. `readOnly` 필드는 read-only로만 마운트 된다는 것을 의미한다.

```yaml
allowedHostPaths:
  # This allows "/foo", "/foo/", "/foo/bar" etc., but
  # disallows "/fool", "/etc/foo" etc.
  # "/foo/../" is never valid.
  - pathPrefix: "/foo"
    readOnly: true # only allow read-only mounts
```

> Warning: 
>
> host filesystem에 제한되지 않은 access를 가진 컨테이너가 privileges를 확장시킬 방법은 매우 많다. 그렇게 되면 다른 컨테이너의 데이터를 읽을수도 있고, kubelet같은 system service의 credentials를 어뷰징할 수도 있다.
>
> writeable hostPath directory volumes는 컨테이너가 `pathPrefix` 바깥의 host filesystem을 traverse할 수 있는 방법으로 filesystem에 작성할 수 있도록 한다. `readOnly: true`는 Kubernetes 1.11+에서 사용가능하며 반드시 모든 `allowedHostPaths`를 통해 지정된 `pathPrefix`에 대해 효과적으로 제한하도록 한다.

**ReadOnlyRootFilesystem** - 컨테이너가 반드시 read-only root filesystem(i.e. writable layer가 없는 상태)으로 동작하도록 한다.

### FlexVolume drivers

flexvolume에서 사용될 수 있도록 허용하기 위한 FlexVolume drivers의 whitelist를 지정한다. empty list나 nil은 driver에 대해 제약이 없다는 것을 의미한다. `volumes` 필드가 `flexVolume` volume type을 포함하고 있음을 확실히 하라; 그렇지 않으면 FlexVolume driver는 허용되지 않는다.

예를 들어:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: allow-flex-volumes
spec:
  # ... other spec fields
  volumes:
    - flexVolume
  allowedFlexVolumes:
    - driver: example/lvm
    - driver: example/cifs
```

### Users and groups

**RunAsUser** - 컨테이너가 동작할 user ID를 제어한다.

* MustRunAs - 최소 하나의 `range`가 지정되어야 한다. 첫 range의 minimum value가 default로 사용된다. 모든 범위에 대해서 검증한다.
* MustRunAsNonRoot - 이미지 안에서 `runAsUser`나 `USER` 지시자가 정의된 파드가 하나라도 submit되어야 한다. `runAsNonRoot`나 `runAsUser` 모두 지정되지 않은 파드는 `runAsNonRoot=true`로 수정이 된다. 따라서 컨테이너 내에서 `USER` 지시자를 사용한 것이 하나는 있어야 한다. default는 제공되지 않는다. `allowPrivilegeEscalation=false`를 설정하는 것은 이 strategy에서 강력 추천된다.
* RunAsAny - default를 제공하지 않는다. 어떤 `runAsUser`든 지정될 수 있도록 한다.

**RunAsGroup** - 어느 primary group ID로 컨테이너가 동작할 지 제어한다.

* MustRunAs - 최소 하나의 `range`를 지정해야한다. 첫 range의 minimum value가 default로 사용된다. 모든 range에 대해서 검증한다.
* MayRunAs - RunAsGroup이 지정될 필요가 없다. 하지만 RunAsGroup이 지정되면 정의된 range 내에 들어야 한다.
* RunAsAny - default를 제공하지 않는다. `runAsGroup`으로 어느 것이든 지정될 수 있다.

**SupplementalGroups** - 컨테이너가 추가하는 group ID를 제어한다.

* MustRunAs - 최소 하나의 `range`가 지정되어야 한다. 첫 range의 minimum value가 default로 사용된다. 모든 range에 대해서 검증한다.
* MayRunAs - 최소 하나의 `range`를 지정해야한다. `supplementalGroups`가 default를 제공하지 않고 unset일 수  있다. `supplementalGroups`가 설정되면 모든 range에 대해 검증한다.
* RunAsAny - default를 제공하지 않는다. 어느 `supplementalGroups`든 지정될 수 있다.

### Privilege Escalation

`allowPrivilegeEscalation` 컨테이너 옵션을 제어하는 옵션들이다. 이 boolean은 [`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt) 플래그가 컨테이너 프로세스에 설정되어있는지 아닌지를 직접 제어한다. 이 플래그는 `setuid` binaries를 efffective user ID로 변경하는 것을 막는다. 또한 파일이 추가적인 capabilities를 가능하게 하는 것을 막는다.(예 : `ping` tool의 사용을 막는다.) 이 동작은 `MustRunAsNonRoot`를 효과적으로 적용하기 위해 필요하다.

**AllowPrivilegeEscalation** - 유저가 컨테이너의 `allowPrivilegeEscalation=true`에 대한 security context를 설정할 수 있는지 없는지를 확인하는 게이트이다. 이는 setuid binaries를 망가뜨리지 않도록 하게 default 설정을 한다. 이를 `false`로 설정하는 것은 conatiner의 child process가 parenet보다 더 많은 privileges를 가질 수 없음을 보장한다.

**DefaultAllowPrivilegeEscalation** - `allowPrivilegeEscalation` 옵션에 대한 default를 설정한다. default behavior는  privilege escalation을 허용하여 setuid binaries를 망가뜨리지 않도록 한다. 그 behavior가 원하지 않는 것이라면 이 필드는 disallow로 default를 사용할 수 있지만 여전히 `allowPrivilegeEscalation`에 대한 권한은 명시적이어야한다.

### Capabilities

Linux capabilities는 superuser와 관련된 privileges를 세세하게 구분한다. 몇몇 이러한 capabilities는 container breakout을 위해 privileges를 escalate할 수 있고, PodSecurityPolicy에 의해서 제한될 것이다. Linux capabilities에 대해 자세히 알고 싶으면 [capabilities(7)](http://man7.org/linux/man-pages/man7/capabilities.7.html)를 보아라.

다음의 필드는 capabilities의 리스트이며 `CAP_` prefix 없이 전부 ALL_CAPS로 capability name이 지정되어있다.

**AllowedCapabilities** - 컨테이너에 추가될 수 있는 capabilities의 whitelist를 제공한다.  default capabilities들은  암묵적으로 allowed이다. empty set은 default set을 넘어선 추가적인 capabilities가 없음을 의미한다. `*`은 모든 capabilities를 의미한다.

**RequiredDropCapabilities** - 반드시 컨테이너에서 제외되어야 하는 capabilities들이다. 이 capabilities들은 default set에서 제거되고 반드시 추가되어서는 안되는 것들이다. `RequiredDropCapabilities` 안에 있는  capabilities는 반드시 `AllowedCapabilities`나 `DefaultAddCapabilities`에 포함되어서는 안된다.

**DefaultAddCapabilities** - 컨테이너에 default로 추가되는 capabilities로 runtime defaults에 추가된다. Docker runtime을 사용할 때 capabilities의 default list를 알아보려면 [Docker documentation](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)를 보아라.

### SELinux

* MustRunAs - `seLinuxOptions`가 설정되어야 한다. `seLinuxOptions`는 default이다. `seLinuxOptions`를 검증한다.
* RunAsAny - default가 제공되지 안흔낟. 어느 `seLinuxOptions`도 지정될 수 있다.

### AllowedProcMountTypes

`allowedProcMountTypes`는 허용된 ProcMountTypes의 whitelist이다. empty 또는 nil은 `DefaultProcMountType`만이 사용된다는 것을 의미한다.

`DefaultProcMount`는 container runtime을 readonly로 default처리하고 /proc에 대한 path에 mask한다. 대부분의 container runtime은 /proc에 특정한 마스크를 하여 특수 장비나 정보에 대해 security exposure 사고를 방지한다. 이는 string  `Default`로 지정된다.

다른 ProcMountType은 `UnmaskedProcMount`만 존재한다. 이는 container runtime에서 default masking behavior를 통과하고 새로 /proc 컨테이너를 생성하여 수정없이 보존되도록 한다. 이는 string `Unmasked`로 지정된다.

### AppArmor

PodSecurityPolicy에서의 annotations를 통해 제어한다. [AppArmor documentation](https://kubernetes.io/docs/tutorials/clusters/apparmor/#podsecuritypolicy-annotations)을 참조하라.

### Seccomp

파드 내에서 seccomp profiles를 사용하는 것은 PodSecurityPolicy를 annotations를 통해 제어가 가능하도록 한다. Seccomp는 Kubernetes의 alpha 기능이다.

**seccomp.security.alpha.kubernetes.io/defaultProfileName** - default seccomp profile을 container에 적용하기 위한 annotations이다. 가능한 values는 다음과 같다.

* `unconfined` - alternative가 제공되지 않으면 seccomp는 container proccess(Kubernetes에서 default이다.)에 적용되지 않는다. 
* `runtime/default` - default container runtime profile이 사용된다.
* `docker/default` - Docker default seccomp profile이 사용된다. Kubernetes 1.11에서부터 deprecated 되었다. `runtime/default`를 대신 사용해라.
* `localhost/<path>` - `<seccomp_root>`가 kubelet에서 `--seccomp-profile-root`를 통해 정의된  `<seccomp_root>/<path>`에 존재하는 노드에서 profile을 file처럼 지정한다.

**seccomp.security.alpha.kubernetes.io/allowedProfileNames** - pod seccomp annotations에 대해서 어떤 value가 허용되는지를 지정하는 annotation이다. 허용되는 value들을 comma-delimited list로 지정한다. 가능한 value들은 위에 list되고, `*`이 모든 프로파일에 대해서 허용된다. 이 annotation이 없다는 것은 default가 변경되지 않았다는 것이다.

### Sysctl

default로 모든 safe sysctls들을 허용된다.

* `forbiddenSysctls` - 지정된 sysctls들을 제외한다. list 내의 safe, unsafe combination들을 막는다. 어느 sysctls도 설정되는 것을 막기 위해 `*`을 사용하라.
* `allowedUnsafeSysctls` - `forbiddenSysctls` 에 적혀있지 않는 한 default list에서 허용되지 않는 sysctls들을 허용할 수 있도록 한다. `



















