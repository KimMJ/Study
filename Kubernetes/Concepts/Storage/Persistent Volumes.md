# Persistent Volumes

* 이 문서는 Kubernetes의 `PersistentVolumes`의 현 상태를 기술한다.
* 그 전에 volumes에 익숙해 지는 것을 추천한다.

## Introduction

* storage를 관리하는 것은 compute를 관리하는 것과는 다른 문제이다.
  * `PersistentVolume` subsystem은 사용자와 운영자에게 어떻게 어떻게 소비하는지로부터 어떻게 제공되는지에 대한 자세한 사항을 추상화한 API를 제공한다.
  * 이를 하기 위해서는 두가지 새로운 API resource를 소개한다. : `PersistentVolume`과 `PersistentVolumeClaim`
* `PersistentVolume`(PV)는 administrator에 의해 provisioned되거나 Storage Classes를 사용하여 동적으로 provisioned된 cluster의 하나의 storage 조각이다.
  * 노드가 cluster resource인 것처럼 이것도 cluster resource이다.
  * PVs는 Volumes처럼 volume plugins이지만 PV를 사용하는 어떤 각각의 파드와 독립된 lifecycle을 가지고 있다.
  * 이 API object는 NFS, iSCSI, cloud-provider-specific storage system같은 storage의 implementation의 자세한 사항을 저장한다.
* `PersistentVolumeClaim`(PVC)는 유저에 의해 요구되는 storage이다.
  * 이는 Pod와 비슷하다.
  * 파드는 node resource를 소비하고 PVCs는 PV resource를 소비한다.
  * 파드는 리소스의 특정한 레벨(CPU와 Memory)을 요청할 수 있다.
  * Claims는 특정한 크기와 접속 모드(e.g. once read/write나 many times read-only로 mount될 수 있다.)를 요청할 수 있다.
* `PersistentVolumeClaims`는 사용자가 추상적인 storage resource를 소비하도록 하는지만 다른 문제에 대해 유저가 performance같은 다양한 속성의 `PersistentVolumes`를 필요로 하는것은 흔한 일이다.
  * Cluster administrators는 유저에게 어떻게 volume이 implement되었는지 자세한 정보를 노출시키지 않고 사이즈나 접속 모드와는 다른 방법으로 `PersistentVolumes`의 다양성을 제공할 수 있을 필요가 있다.
  * `StorageClass` resource는 이 필요성에 부합한다.
* [detailed walkthrough with working examples](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)를 보아라.

## Lifecycle of a volume and claim

* PVs는 cluster에서 resource이다.
  * PVCs는 이런 resource를 요청하고 이 resource 요청에 대한 체크도 한다.
  * PVs와 PVCs 사이의 interaction은 이러한 lifecycle을 가진다.

### Provisioning

* PVs가 provisioned되는 두가지 방법이 있다. : statically, dynamically

#### Static

 * cluster administrator는 몇개의 PVs를 생성한다.
   	* cluster user에 대해 어떤 real storage를 사용할 수 있는 지에 대한 자세한 사항을 다룬다.
   	* 그것들은 Kubernetes API에 존재하고 consumption에 대해 사용가능하다.

#### Dynamic

* administrator가 생성한 static PV가 사용자의 `PersistentVolumeClaim`과 하나도 일치하지 않으면 cluster는 PVC를 위한 동적으로 volume을 provision할 것이다.
  * 이 provisioning은 `StorageClasses`를 기반으로 한다.: PVC는 반드시 storage class를 요청해야 하고 administrator는 반드시 dynamic provisioning이 발생하기 위해 그 class를 생성하고 설정해야 한다.
  * class `""`를 요청하는 claim은 효과적으로 dynamic provisioning을 스스로 비활성화 하도록 한다.
* storage class를 기반으로 dynamic storage provisioning을 활성화 하기 위해 cluster administrator는 API server에서 `DefaultStorageClass` admission controller를 활성화 할 필요가 있다.
  * 예를들어 이는 `DefaultStorageClass`가 comma-delimited이고 API server component의 `--enable-admission-plugins` 플래그 값의 ordered list를 확인함으로써 행해질 수 있다.
  * API server command line flag에 대한 더 많은 정보는 [kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver/)를 참조하라.

### Binding

* 사용자는 생성하거나 이미 dynamic provisioning의 케이스로 지정된 요청 storage 양을 가지고 있고 특정한 접속 모드가 있는`PersistentVolumeClaim`을 가지고 생성한다.
* master에서의 control loop는 새로운 PVC를 관찰하고 가능하다면 일치하는 PV가 있는지 찾고 그 그것들을 bound한다.
  * 만일 PV가 동적으로 새로운 PVC에 대해 provisioned되면 loop을 항상 그 PV와 PVC를 bound할 것이다.
  * 반면에 사용자는 최소한 요청하는 것에 대해서 항상 얻을 수 있지만 volume은 요청한 것을 초과할 수 있다.
  * 한번 연결되면 `PersistentVolumeClaim` bind는 어떻게 연결이 되었는지에 상관 없이 배제가 된다.
  * PVC에서 PV로의 연결은 일대일 매핑이다.
* claim은 일치하는 volume이 존재하지 않는다면 무기한으로 unbound인 채로 남아있다.
  * claims는 일치하는 volume이 사용가능해지면 bound된다.
  * 예를 들어 많은 50Gi PVs로 provisioned된 cluster는 100Gi를 요청하는 PVC와 매칭되지 않는다.
  * PVC는 100Gi PV가 cluster에 추가될 때 bound될 것이다.

### Using

* 파드는 claim을 volume처럼 사용한다.
  * cluster는 bound volume을 찾기 위해 claim을 조사하고 파드에 대해 그 volume을 mount한다.
  * multiple access modes를 지원하는 volume에 대해 사용자는 파드에서 volume으로써 claim을 사용할 때 어떤 mode를 원하는지 지정한다.
* 사용자가 claim을 가지고 있고, bound되면 bound PV는 사용자가 원하는 만큼의 시간동안 사용자에게 속하게 된다.
  * 사용자는 파드를 스케쥴하고 파드의 volume block에서 `persistentVolumeClaim`을 포함함으로써 claimed PV에 접속할 수 있다.
  * [See below for syntax details](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes).

### Storage Object in Use Protection

* Use Protection feature에서 Storage Object의 목적은 파드와 PVCs에 bound된 Persistent Volume(PVs)에 의해 active use인 Persistent Volume Claims(PVCs)가 데이터 손실을 야기할 수 있기 때문에 시스템에서 삭제되지 않도록 해준다.

> Note : PVC를 사용하는 파드 오브젝트가 존재하면 PVC는 파드에 의해 active use이다.

* 사용자가 파드에서 active use인 PVC를 삭제하면 PVC는 즉시 지워지지 않는다.

  * PVC 삭제는 PVC가 어떤 파드에 의해서도 사용되지 않을때까지 지연되고 admin이 PVC와 bound된 PV를 지우면 PV또한 즉시 지워지지 않는다.
  * PV의 삭제는 PV가 더이상 PVC에 bound되지 않을때까지 지연된다.

* PVC의 상태가 `Terminating`이고 `Finalizers` list가 다음과 같은 `kubernetes.io/pvc-protection`일 때 PVC가 보호됨을 볼 수 있다.

  ```shell
  kubectl describe pvc hostpath
  Name:          hostpath
  Namespace:     default
  StorageClass:  example-hostpath
  Status:        Terminating
  Volume:        
  Labels:        <none>
  Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
                 volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
  Finalizers:    [kubernetes.io/pvc-protection]
  ...
  ```

* PV는 PV상태가 `Terminating`이고 `Finalizers` list가 `kubernetes.io/pv-protection`을 포함하였을 때 PV가 보호됨을 볼 수 있다.

  ```shell
  kubectl describe pv task-pv-volume
  Name:            task-pv-volume
  Labels:          type=local
  Annotations:     <none>
  Finalizers:      [kubernetes.io/pv-protection]
  StorageClass:    standard
  Status:          Available
  Claim:           
  Reclaim Policy:  Delete
  Access Modes:    RWO
  Capacity:        1Gi
  Message:         
  Source:
      Type:          HostPath (bare host directory volume)
      Path:          /tmp/data
      HostPathType:  
  Events:            <none>
  ```

### Reclaiming

* 사용자가 volume을 다 사용하였을 때 resource의 reclamation을 허용하는 API에서 PVC 오브젝트를 삭제한다.
  * `PersistentVolume`에 대한 reclaim policy는 cluster가 claim이 release된 후에 volume으로 무엇을 할지 알려준다.
  * 현재 volume은 Retained, Recycled, Deleted될 수 있다.

#### Retain

* `Retain` reclaim policy는 리소스의 수동 reclamation을 가능하게 한다.
  * `PersistentVolumeClaim`이 삭제될 때 `PersistentVolume`은 여전히 존재하고 volume은 "released"로 간주된다.
  * 하지만 다른 claim에서는 아직 사용가능하지 않은데 이는 이전의 claimant의 데이터가 volume에 남아있기 때문이다.
  * administrator는 수동적으로 다음 절차를 통해 volume을 reclaim해야한다.
    * `PersistentVolume` 삭제.
      * AWS EBS, GCE PD, Azure Disk, Cinder volume같은 external infrastructure에서의 associated storage asset은 여전히 PV가 지워지고 난 후에도 존재한다.
    * 적절하게 associated storage asset의 데이터를 수동으로 지운다.
    * associated storage asset을 수동으로 지운거나 같은 storage asset을 재사용하기를 원한다면 새로운 `PersistentVolume`을 storage asset definition으로 생성하라.

#### Delete

* `Delete` reclaim policy를 지원하는 volume plugins에 대해 deletion은 `PersistentVolume`오브젝트를 Kubernetes로부터 지우고 또한 AWS EBS, GCE PD, Azure Disk, Cinder volume같은 external infrastructure에서의 associated storage asset을 지운다.
  * dynamic하게 provisioned된 volumes는 default가 `Delete`인 `StorageClass`에 대한 reclaim policy를 상속한다.
  * administrator는 `StorageClass`를 사용자의 기대에 맞게 설정해야하는 반면 PV는 반드시 이것이 생성되고 난 후 수정되거나 패치되어야 한다.
  * [Change the Reclaim Policy of a PersistentVolume](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/)을 보아라.

#### Recycle

> Warning : `Recycle` reclaim policy는 deprecated이다. 대신, dynamic provisioning을 사용하는 접근법을 추천한다.

* 만약 기반 volume plugin이 지원되면 `Recycle` reclaim policy는 volume에서 basic scrub(rm -rf /thevolume/*)을 실행하고 새로운 claim으로 사용될 수 있도록 만든다.

* 하지만 administrator가 [여기](https://kubernetes.io/docs/admin/kube-controller-manager/)기술된 Kubernetes controller manager command line argument를 사용하여 custom recycler pod template을 설정할 수 있다.

  * custom recycler pod template은 다음의 예시처럼 반드시 `volumes` specification을 가지고 있어야 한다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pv-recycler
      namespace: default
    spec:
      restartPolicy: Never
      volumes:
      - name: vol
        hostPath:
          path: /any/path/it/will/be/replaced
      containers:
      - name: pv-recycler
        image: "k8s.gcr.io/busybox"
        command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
        volumeMounts:
        - name: vol
          mountPath: /scrub
    ```

### Expanding Persistent Volumes Claims

**FEATURE STATE : ** `Kubernetes v1.11` - beta

* PersistentVolumeClaims(PVCs)의 확장된 지원은 기본으로 활성화되어있다.
  * 다음 타입의 volume에 대해서 확장할 수 있다.
    * gcePersistentDisk
    * awsElasticBlockStore
    * Cinder
    * glusterfs
    * rbd
    * Azure File
    * Azure Disk
    * Portworx
    * FlexVolumes
    * CSI
* 

