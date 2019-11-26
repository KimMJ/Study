# Cluster Administration Overview

cluster administration overview는 Kubernetes cluster를 생성하거나 관리하는 모두를 위한 것이다. 이는 Kubernetes concepts에 대해 잘 알고있다고 가정한다.

## Planning a cluster

[Setup](https://kubernetes.io/docs/setup/)에 있는 가이드를 통해서 어떻게 Kubernetes clusters를 계획하고 셋업하고 구성하는지 알아보아라. 이 article에 있는 솔루션들은 distros라고 불린다.

가이드를 선택하기 전에 다음의 고려사항들이 있다.

* 단순히 컴퓨터에 쿠버네티스를 테스트하려고 설치하는 것인가 아니면 고가용성으로 multi-cluster 구성을 원하는가? 니즈에 맞는 distros를 선택하라.
* high-availability를 디자인하고 있다면, [clusters in multiple zones](https://kubernetes.io/docs/concepts/cluster-administration/federation/)에 대해 학습해 보아라.
* Google Kubernetes Engine같은 hosted Kubernetes cluster를 사용할 것인지 자신의 cluster를 호스팅할 것인지.
* cluster가 on-premises가 될 것인가 cloud (IaaS)가 될 것인가? Kubernetes는 hybrid cluster를 직접 지원하지 않는다. 대신, multiple cluster를 구성할 수는 있다.
* Kubernetes on-premises를 설정한다면 가장 잘 맞는 [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)을 선택하라.
* 쿠버네티스를 "bare metal" hardware에 동작시킬 것인가 virtual machines (VMs)에 동작시킬 것인가?
* 단순히 cluster를 동작시킬 것인지 아니면 쿠버네티스로 실제 배포를 하게 될 것인지. 후자라면 actively-developed distro를 선택하라. 몇몇 distros는 binary releases만 사용하지만 더 다양한 선택권이 있다.
* 클러스터를 동작시키는 데 필요한 [components](https://kubernetes.io/docs/admin/cluster-components/)에 대해서 익숙해저라.

Note : 모든 distros들이 actively maintained 되는 것은 아니다. 최신 쿠버네티스 버전에서 테스트가 된 distros를 선택하라.

## Managing a cluster

* [Managing a cluster](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/)는 cluster의 lifecycle에 관련된 몇가지 주제를 설명한다: 새로운 클러스터를 생성하기, cluster의 마스터와 워커 노드를 업그레이드 하기, 노드 maintenance 수행 하기 (e.g. kernel upgrades), 동작중인 cluster의 Kubernetes API version 업그레이드 하기
* [manage nodes](https://kubernetes.io/docs/concepts/nodes/node/)에 대해 배워라.
* 공유 클러스터에 대해서 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)를 어떻게 설정하고 관리하는지 배워라.

## Securing a cluster

* [Certificates](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)는 다른 툴 체인을 사용하여 certificates를 생성하는 방법에 대해 기술한다.
* [Kubernetes Container Environment](https://kubernetes.io/docs/concepts/containers/container-environment-variables/)는 Kubernetes node의 컨테이너를 관리하는 Kubelet에 대한 환경설정에 대해서 설명한다.
* [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)는 어떻게 유저와 service accounts에 대한 권한을 설정하는지 설명한다.
* [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)은 다양한 인증 옵션을 포함한 Kubernetes의 인증에 관해 설명한다.
* [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)은 authentication에서 분리되어 어떻게 HTTP 호출을 핸들링하는지를 제어한다.
* [Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)는 authentication과 authorization 이후의 Kubernetes API server로의 요청을 가로채는 plug-ins에 대해서 설명할 것이다.
* [Using Sysctls in a Kubernetes Cluster](https://kubernetes.io/docs/concepts/cluster-administration/sysctl-cluster/)은 kernel parameters를 설정하기 위해 어떻게 `sysctl` command-line tool을 사용하는지에 대해서 설명한다.
* [Auditing](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)은 어떻게 Kubernetes의 audit logs와 상호작용하는지를 설명한다.

### Securing the kubelet

- [Master-Node communication](https://kubernetes.io/docs/concepts/architecture/master-node-communication/)
- [TLS bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
- [Kubelet authentication/authorization](https://kubernetes.io/docs/admin/kubelet-authentication-authorization/)

## Optional Cluster Services

* [DNS Integration](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)은 DNS name을 어떻게 Kubernetes service로 풀이하는지에 대해 설명한다.
* [Logging and Monitoring Cluster Activity](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 은 어떻게 Kubernetes가 동작하고 어떻게 이를 구현하는지에 대해 설명한다.