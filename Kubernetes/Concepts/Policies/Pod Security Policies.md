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

Pod security policy control은 