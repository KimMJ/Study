# Cloud Providers

이 페이지는 특정한 cloud provider에서 동작하고 있는 Kubernetes를 어떻게 관리하는지에 대해서 설명한다.

### kubeadm

[kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)은 kubernetes clusters를 생성하는데 인기있는 옵션이다.  kubeadm은 cloud providers에 대한 configuration information을 지정하는 configuration option을 가지고 있다. 예를 들어 일반적인 in-tree cloud provider는 아래와 같이 kubeadm을 사용해서 설정할 수 있다.

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "openstack"
    cloud-config: "/etc/kubernetes/cloud.conf"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.13.0
apiServer:
  extraArgs:
    cloud-provider: "openstack"
    cloud-config: "/etc/kubernetes/cloud.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud.conf"
    mountPath: "/etc/kubernetes/cloud.conf"
controllerManager:
  extraArgs:
    cloud-provider: "openstack"
    cloud-config: "/etc/kubernetes/cloud.conf"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud.conf"
    mountPath: "/etc/kubernetes/cloud.conf"
```

in-tree cloud providers는 일반적으로  [kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver/), [kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/), [kubelet](https://kubernetes.io/docs/admin/kubelet/)에 대해 command line에 지정된 `--cloud-provider`와 `--cloud-config`를 필요로 한다. 각 cloud provider에 대해 `--cloud-config`에서 지정된 내용은 아래에 문서화되어있다.

---

생략

