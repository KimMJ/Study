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
  
* storage class의 `allowVolumeExpansion` 필드가 true로 설정되었을 때만 PVC를 확장할 수 있다.

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gluster-vol-default
  provisioner: kubernetes.io/glusterfs
  parameters:
    resturl: "http://192.168.10.100:8080"
    restuser: ""
    secretNamespace: ""
    secretName: ""
  allowVolumeExpansion: true
  ```

* PVC에 대해 더 큰 크기의 volume을 요청하려면 PVC 오브젝트를 수정하고 더 큰 사이즈를 지정해라.

  * 이는 `PersistentVolume` 기반을 지원하는 volume의 확장을 트리거한다.
  * 새로운 `PersistentVolume`은 그 claim을 만족하기 위해 절대 생성되지 않는다.
  * 대신 존재하던 volume이 resize된다.

#### CSI Volume expansion

**FEATURE STATE : ** `Kubernetes v1.14` - alpha

* CSI volume 확장은 `ExpandCSIVolumes` feature gate의 활성화가 필요하고 또한 지정된 CSI driver가 volume expansion을 지원해야한다.
  * 더 많은 정보를 위해 지정된 CSI driver 문서를 확인하라.

#### Resizing a volume containing a file system

* file system이 XFS, Ext3, Ext4인 경우에만 volume을 resize할 수 있다.
* volume이 filesystem을 가지고 있으면 file system은 새로운 파드가 `PresistentVolumeClaim`을 ReadWrite 모드로 사용할 때만 resize한다.
  * File system 확장은 파드가 시작할때나 파드가 동작하고 기반 file system이 online expansion을 지원할 때 모두 할수 있다.
* FlexVolumes는 driver에서 `RequiresFSResize` capability가 true가 되었을 때 resize를 허용한다.
  * FlexVolume은 파드가 재시작될 때 resize된다.

#### Resizing an in-use PersistentVolumeClaim

**FEATURE STATE : ** `Kubernetes v1.15` - beta

> Note : Expanding in-use PVC는 1.15부터 베타버전이 되었으며 1.11부터는 알파였다. `ExpandInUsePersistentVolumes` feature는 반드시 활성화 되어야 하며 이는 베타 기능의 많은 클러스터에 대해서 자동으로 실행되어야 한다. 더 많은 정보를 위해 feature gate documentation을 참조하라.

* 이 경우에 사용자는 PVC를 사용하고 있던 파드나 deployment를 삭제하고 재생성 할 필요가 없다.
  * in-use PVC는 파일 시스템이 커지면 자동적으로 파드에서 사용가능 할 수 있게 된다.
  * 이 feature는 파드나 deployment가 사용하지 않는 PVC에서 영향을 주지 않는다.
  * 반드시 확장이 성공할 수 있기 전에 그 PVC를 사용하는 파드를 생성해야한다.
* 다른 volume type과 비슷하게 FlexVolume volume은 파드에 의해 in-use일 때 확장할 수 있다.

> Note : FlexVolume resize는 기반 driver가 resize를 지원할때만 사용가능하다.

> Note : EBS volumes를 확장하는 것은 시간이 걸리는 작업이다. 또한 6시간마다 한번 수정에 per-volume quota가 있다.

## Types of Persistent Volumes

* `PersistentVolume`타입은 plugin으로서 이식된다.
  * Kubernetes는 현재 다음의 plugin을 지원한다.
    * GCEPersistentDisk
    * AWSElasticBlockStore
    * AzureFile
    * AzureDisk
    * CSI
    * FC (Fibre Channel)
    * Flexvolume
    * Flocker
    * NFS
    * iSCSI
    * RBD (Ceph Block Device)
    * CephFS
    * Cinder (OpenStack block storage)
    * Glusterfs
    * VsphereVolume
    * Quobyte Volumes
    * HostPath (Single node testing only – local storage is not supported in any way and WILL NOT WORK in a multi-node cluster)
    * Portworx Volumes
    * ScaleIO Volumes
    * StorageOS

## PersistentVolumes

* 각각의 PV는 volume의 specification과 status인 spec과 status를 가지고 있다.

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv0003
  spec:
    capacity:
      storage: 5Gi
    volumeMode: Filesystem
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    storageClassName: slow
    mountOptions:
      - hard
      - nfsvers=4.1
    nfs:
      path: /tmp
      server: 172.17.0.2
  ```

### Capacity

* 일반적으로 PV는 지정된 storage capacity를 가진다.
  * 이는 PV의 `capacity` attribute을 사용하여 설정된다.
  * Kubernetes [Resource Model](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md)를 참조하여 `capacity`에 의해 unit이 어떻게 되는지를 이해할 수 있다.
* 현재 storage size는 단 하나의 설정될 수 있는, 그리고 필수적인 리소스이다.
  * 추후에는 IOPS, throughput 등이 추가될 수 있다.

### Volume Mode

**FEATURE STATE : ** `Kubernetes v1.13` - beta

* Kubernetes 1.9 이전에는 모든 volume plugin들은 persistent volume에 filesystem을 생성하였다.
  * 이제 사용자는 `volumeMode`의 값을 `block`으로 설정하여 raw block device를 사용할 수 있고 또는 `filesystem`을 사용하여 filesystem을 쓸 수 있도록 설정할 수 있다.
  * `filesystem`은 value가 생략되었을 때 기본값이다.
  * 이는 optional API parameter이다.

### Access Modes

* `PersistentVolume`은 resource provider에 의해서 어떤방식으로든 host에 mount될 수 있다.
  * 아래의 표에서 보이듯이 provider는 다른 capability를 가질 것이고 각 PV의 access mode는 particular volume에 의해 지원되는 지정된 mode로 설정된다.
  * 예를 들어, NFS는 multiple read/write client를 지원할 수 있지만 특정 NFS PV는 서버에 read-only로 노출이 될 것이다.
  * 각 PV는 지정된 PV의 capability를 설명하는 access mode의 세트를 가질 것이다.
* access mode들
  * ReadWriteOnce
    * volume은 싱글 노드에 의해 read-write로 mount될 수 있다.
  * ReadOnlyMany
    * volume은 멀티 노드에 의해 read-only로 mount될 수 있다.
  * ReadWriteMany
    * volume은 멀티 노드에 의해 read-write로 mount될 수 있다.

* CLI에서 access modes는 다음처럼 줄여쓴다.
  * RWO - ReadWriteOnce
  * ROX - ReadOnlyMany
  * RWX - ReadWriteMany

* **중요!** volume은 many를 지원하더라도 한번에 하나의 access mode를 사용하여 mount될 수 있다.
  *  예를 들어 GCEPersistentDisk는 단일 노드에서 ReadWriteOnce로 mount되거나 ReadOnlyMany 에 의해 많은 노드에서 ReadOnlyMany에 mount될 수 있지만 동시에 하지는 않는다.

| Volume Plugin        | ReadWriteOnce         | ReadOnlyMany          | ReadWriteMany                      |
| :------------------- | :-------------------- | :-------------------- | :--------------------------------- |
| AWSElasticBlockStore | ✓                     | -                     | -                                  |
| AzureFile            | ✓                     | ✓                     | ✓                                  |
| AzureDisk            | ✓                     | -                     | -                                  |
| CephFS               | ✓                     | ✓                     | ✓                                  |
| Cinder               | ✓                     | -                     | -                                  |
| CSI                  | depends on the driver | depends on the driver | depends on the driver              |
| FC                   | ✓                     | ✓                     | -                                  |
| Flexvolume           | ✓                     | ✓                     | depends on the driver              |
| Flocker              | ✓                     | -                     | -                                  |
| GCEPersistentDisk    | ✓                     | ✓                     | -                                  |
| Glusterfs            | ✓                     | ✓                     | ✓                                  |
| HostPath             | ✓                     | -                     | -                                  |
| iSCSI                | ✓                     | ✓                     | -                                  |
| Quobyte              | ✓                     | ✓                     | ✓                                  |
| NFS                  | ✓                     | ✓                     | ✓                                  |
| RBD                  | ✓                     | ✓                     | -                                  |
| VsphereVolume        | ✓                     | -                     | - (works when pods are collocated) |
| PortworxVolume       | ✓                     | -                     | ✓                                  |
| ScaleIO              | ✓                     | ✓                     | -                                  |
| StorageOS            | ✓                     | -                     | -                                  |

### Class

* PV는 `storageClassName` attribute를 StorageClass의 이름으로 설정하여 지정되는 class를 가질 수 있다.
  * 특정한 class의 PV는 그 class를 요청하는 PVC에만 bind될 수 있다.
  * `storageClassName`이 없는 PV는 class가 없고 특정한 class가 없는 요청을 하는 PVC에만 bind될 수 있다.
* 이전에는 `volume.beta.kubernetes.io/storage-class` annotation이 `storageClassName` attribute 대신에 사용된다.
  * 이 annotation은 아직 작동하지만 후의 Kubernetes release에서 완전히 deprecate될 것이다.

### Reclaim Policy

* 현재 reclaim policy들이다.
  * Retain
    * 수동 reclamation
  * Recycle
    * 기본 삭제 (`rm -rf /thevolume/*`)
  * Delete
    * AWS EBS, GCE PD, Azure Disk, OpenStack Cinder volume같은 연관된 storage asset들이 삭제된다.
* 현재 NFS와 HostPath만 recycling을 지원한다.
  * AWS EBS, GCE PD, Azure Disk, Cinder volume은 deletion을 지원한다.

### Mount Options

* Kubernetes administrator는 노드에 Persistent Volume이 mount되었을 때 추가적인 mount options를 지정할 수 있다.

> Note : 모든 Persistent volume 타입이 mount options를 지원하는 것이 아니다.

* 다음 volume 타입은 mount options를 지원한다.
  * AWSElasticBlockStore
  * AzureDisk
  * AzureFile
  * CephFS
  * Cinder (OpenStack block storage)
  * GCEPersistentDisk
  * Glusterfs
  * NFS
  * Quobyte Volumes
  * RBD (Ceph Block Device)
  * StorageOS
  * VsphereVolume
  * iSCSI
* mount options는 입증되지 않았으며 따라서 mount는 유효하지 않다면 그냥 실패할 것이다.
* 이전에는 `volume.beta.kubernetes.io/mount-options` annotation이 `mountOptions` attribute 대신에 사용되었다.
  * 이 annotation은 아직 동작하지만 미래의 Kubernetes release에서는 완전히 deprecate될 것이다.

### Node Affinity

> Note : 대부분의 volume type에 대해 이 필드를 설정할 필요가 없다. 이는 AWS EBS, GCE PD와 Azure Disk volume block type에 대해 자동적으로 생성된다. 사용자는 local volume에 대해서만 이를 명백히 설정해주어야 한다.

* PV는 node affinity를 지정하여 이 volume으로 접속할 수 있는 node에 대해서 제한사항을 정의할 수 있다.
  * PV를 사용하는 파드는 node affinity에 의해 선택된 노드에서만 스케쥴되어야 한다.

### Phase

* volume은 다음의 phase중 하나이다.
  * Available
    * claim으로 아직 bind되지 않은 free resource
  * Bound
    * claim으로 bind된 volume
  * Released
    * claim이 삭제되었지만 cluster에 의해서 아직 reclaim되지 않은 resource
  * Failed
    * automatic reclamation이 실패한 volume
* CLI는 PV에 bind된 PVC의 이름을 보여줄 것이다.

## PersistentVolumeClaims

* 각각의 PVC는 claim의 specification과 status를 설명하는 spec과 status를 포함한다.

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: myclaim
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 8Gi
    storageClassName: slow
    selector:
      matchLabels:
        release: "stable"
      matchExpressions:
        - {key: environment, operator: In, values: [dev]}
  ```

### Access Modes

* claim은 지정된 access modes로 storage를 요청할 때  volumes과 같은 conventions를 사용한다.

### Voulme Modes

* claim은 filesystem이나 block devices같은 volume의 consumption을 나타내기 위해 volume과 같은 conventions를 사용한다.

### Resources

* 파드같은 claim은 지정된 양의 resource를 요청할 수 있다.
  * 이 경우에 request는 storage에 대한 것이다.
  * 같은 resource model이 volume과 claim 모두에 적용된다.

### Selector

* claim은 volume의 세트에 대한 필터링을 위해 label selector를 지정할 수 있다.
  * selector와 일치하는 label을 가진 volume만이 claim으로 bind될 수 있다.
  * selector는 두개의 필드로 구성이 된다.
    * `matchLabels`
      * volume은 반드시 이 값으로 label을 가져야 한다.
    * `matchExpressions`
      * 지정된 키, value의 리스트와 key, value와 관련된 operator로 만들어진 요청들의 list
      * 유요한 operator는 In, NotIn, Exists, DoesNotExists를 포함한다.

* `matchLabels`와 `matchExpressions`의 모든 requirements가 AND로 구성이 된다.
  * 매칭되기 위해서는 모든것을 만족해야한다.

### Class

* claim은 `storageClassName` attribute를 사용하여 StorageClass의 이름을 지정함으로써 특정한 class를 요청할 수 있다.
  * 요청된 class의 PV중 PVC와 같은 `storageClassName`인 것들만 PVC와 bind된다.
* PVC는 class를 요청할 필요가 없다.
  * `storageClassName`을 `""`으로 설정한 PVC는 항상 class가 없는 PV를 요청한것으로 인식되고 따라서 class가 없는(annotation이 없거나 `""`로 설정된 것) PV로만 bind될 수 있다.
  * `storageClassName`이 없는 PVC과는 약간 다르며 cluster에서 `DefaultStorageClass` admission plugin이 켜졌는지에 따라 다르게 처리가 될 것이다.
    * admission plugin이 켜져있으면 administrator는 default `StorageClass`를 지정할 수 있다.
      * `storageClassName`을 가지고 있지 않은 모든 PVC는 default인 PV들과만 bind될 수 있다.
      * default `StorageClass`를 오브젝트 내에서 true로 `storageclass.kubernetes.io/is-default-class` annotation을 설정함으로써  지정할 수 있다.
      * administrator가 default를 지정하지 않으면 cluster는 admission plugin이 꺼져있는 상태인 것처럼 PVC creation을 한다.
      * admission plugin은 모든 PVC의 생성을 거부한다.
    * admission plugin이 꺼지면 default `StorageClass`에 대한 개념이 없다.
      * 모든 `storageClassName`을 가지고 있지 않는 PVC는 class가 없는 PV와만 bind된다.
      * 이 경우에 `storageClassName`이 없는  PVC는 `storageClassName`이 `""`로 지정된 PVC와 같은 방식으로 취급된다.

* installation method와 관련하여 default StorageClass는 installation동안 addon manager를 통해 Kubernetes cluster에 deploy될 것이다.
* PVC가 `StorageClass`를 요청하는 것과 더불어 `selector`를 지정하면 requirement는 함께 AND가 된다.
  * 요청된 class, label인 PV만 PVC에 bind될 수 있다.

> Note : 현재 `selector`가 비어있지 않은 PVC는 이를 위해 동적으로 준비되지 않은 PV를 가질 수 없다.

* 이전에는 `volume.beta.kubernetes.io/storage-class`가 `storageClassName` attribute대신에 사용되었다.
  * 이 annotation은 아직 동작하지만 후의 Kubernetes release에서는 지원되지 않을 것이다.

## Claims As Volumes

* 파드는 storage에 volume으로 claim을 사용하여 접속한다.

  * claim은 반드시 claim을 사용하는 파드와 같은 namespace에 있어야 한다.

  * cluster는 파드의 namespace안에서 claim을 찾고 `PersistentVolume`을 claim으로부터 가져오는데 사용한다.

  * volume은 그러면 host와 파드 안에 mount된다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
        - name: myfrontend
          image: nginx
          volumeMounts:
          - mountPath: "/var/www/html"
            name: mypd
      volumes:
        - name: mypd
          persistentVolumeClaim:
            claimName: myclaim
    ```

### A Note on Namespaces

* `PersistentVolumes` binds는 배타적이고 `PersistentVolumeClaims`는 namespaced object이기 때문에 claim을 "Many" 모드(`ROX`, `RWX`)로 mount하는 것은 하나의 namespace안에서만 가능하다.

## Raw Block Volume Support

**FEATURE STATE : ** `Kubernetes v1.13` - beta

* 다음의 volume plugin은 적용 가능한 곳에서는 dynamic provisioning을 포함한 raw block volume을 지원한다.
  * AWSElasticBlockStore
  * AzureDisk
  * FC (Fibre Channel)
  * GCEPersistentDisk
  * iSCSI
  * Local volume
  * RBD (Ceph Block Device)
  * VsphereVolume (alpha)

> Note : FC와 iSCSI volume만 raw block volume를 Kubernetes 1.9에서 지원한다. 추가적인 plugin들은 1.10에서 추가되었다.

### Persistent Volumes using a Raw Block Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### Persistent Volume Claim requesting a Raw Block Volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

### Pod specification adding Raw Block Device path in container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

> Note : 파드에 raw block device를 추가할 때 우리는 mount path대신에 container에서의 device path를 지정한다.

### Binding Block Volumes

* 만일 사용자가 `PersistentVolumeClaim` spec에 있는 `volumeMode` 필드를 사용하여 raw block volume을 지정하면 spec의 일부를 이 mode에 대해 고려하지 않았던 이전의 release와는 약간 다른 binding rule이 된다.

  * 다음 리스트는 user와 admin이 raw block device를 요청할 때 지정할 수 있는 가능한 조합들이다.

  * 표는 volume이 주어진 combinatino에서 bound 되는지 안되는지를 보여준다.

    * 정적으로 준비된 volume에 대한 volume binding matrix

    | PV volumeMode | PVC volumeMode | Result  |
    | :------------ | :------------- | :------ |
    | unspecified   | unspecified    | BIND    |
    | unspecified   | Block          | NO BIND |
    | unspecified   | Filesystem     | BIND    |
    | Block         | unspecified    | NO BIND |
    | Block         | Block          | BIND    |
    | Block         | Filesystem     | NO BIND |
    | Filesystem    | Filesystem     | BIND    |
    | Filesystem    | Block          | NO BIND |
    | Filesystem    | unspecified    | BIND    |

> Note : statically provisioned volume만이 alpha release에 대해 지원된다. administrator는 raw block device로 작업을 할 때 이런 값들에 대해서 생각해야 한다.

## Volume Snapshot and Restore Volume from Snapshot Support

**FEATURE STATE : ** `Kubernetes v1.12` - alpha

* olume snapshot feature는 CSI Volume plugin가 지원되는 곳에만 추가되었다.
  * 세한 사항은 [volume snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)참조
* volume snapshot data source에서 volume에 대한 복구 지원을 활성화하려면 `VolumeSnapshotDataSource` feature gate를 apiserver와 controller-manager에서 활성화시켜라.

### Create Persistent Volume Claim from Volume Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume Cloning

**FEATURE STATE : **`Kubernetes v1.15` - alpha

* volume clone feature는 CSI Volume Plugins를 지원하는 곳에만 추가되었다.
  * 자세한 사항은 [volume cloning](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)참고
* pvc data source에서 volume을 cloning하는 것에 대한 지원을 활성화하려면 apiserver와 controller-manager에 있는 `VolumePVCDataSource` feature gate에서 활성화하라.

### Create Persistent Volume Claim from an existing pvc

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Writing Portable Configuration

* cluster의 넓은 반경에서 작동하고 persistent storage를 필요로하는 configuration template이나 example을 작성하고자 한다면 우리는 다음의 패턴을 추천하고 싶다.
  * PersistentVolumeClaim 오브젝트를 config 번들에 추가하라. (Deployments, ConfigMaps, 등)
  * config를 예시로 사용하는 사람들이 PersistentVolumes를 생성하는데 권한을 가지지 않을 것이기 때문에 config에서 PersistentVolume 오브젝트를 포함하지 말아라.
  * template을 사용할 때 storage class name에 대한 option을 사용자에게 주어라.
    * 사용자가 storage class name을 주면 그 값을 `persistentVolumeClaim.storageClassName` 필드에 넣어라.
      * admin에 의해서 활성화된 StorageClasses를 가지고 있는 cluster가 있을 때 이는 PVC가 올바른 storage class와 매칭되도록 할 것이다.
    * 사용자가 storage class name을 주지 않으면 `persistentVolumeClaim.storageClassName`을 nul로 놔두어라.
    * 이는 PV가 자동적으로 cluster의 default StorageClass로 사용자가 provisioned되도록 할 것이다.
      * 많은 cluster environment는 default StorageClass가 설치되어 있거나 administrator는 고유의 default StorageClass를 생성할 수 있다.
  * tooling에서 조금의 시간 후에도 bind되지 않는 PVC를 관찰하고 이를 사용자에게 노출시켜서 cluster가 dynamic storage support를 가지고 있지 않음(사용자가 일치하는 PV를 생성할 때)을 알리거나 cluster에 storage system이 없다(사용자가 요청된 PVC에 config를 배포할 수 없는 경우에)는 것을 알려라.