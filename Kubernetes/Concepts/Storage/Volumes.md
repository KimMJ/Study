# Volumes

* container의 on-disk file은 일시적로 non-trivial applications가 container에서 동작할 때 몇몇 문제를 발생시킨다.
  * 먼저, container가 충돌이 되면 kubelet은 이를 재시작하지만 파일들은 사라질 것이다. - container는 깨끗한 상태로 시작된다.
  * 둘째로 `Pod`에서 container를 동시에 실행시킬 때 종종 이 container사이에 파일을 공유할 필요성이 생긴다.
  * Kubernetes `Volume` abstraction은 이런 문제들을 모두 해결한다.
* 파드에 대해 익숙해지는 것을 추천한다.

## Background

* Docker역시 volumes라는 개념을 가지고 있지만 이는 약간 느슨하고 덜 관리가 된다.
  * Docker에서 volume은 간단히 disk나 다른 container에서의 directory이다.
  * lifetime은 관리되지 않고 최근까지 local-disk-backed volumes만 있었다.
  * Docker는 이제 volume drivers를 제공하지만 기능들이 아직까지는 훨씬 제한적이다.(예를 들어 Docker 1.7과 같이 각 container마다 하나의 volume driver만 허용이 되고 parameter를 volume으로 보낼 방법이 없다.)
* 반면에 Kubernetes volume은 명시적인 lifetime이 있다. - 이를 사용하는 파드와 같다.
  * 결과적으로 파드 안에서 동작하는 모든 conatiner 밖에서 volume이 존재하고 data는 container가 재시작될 때 보존이 된다.
  * 물론 파드가 중지될 때 volume 또한 중지된다.
  * 이보다 더 중요한 것은 아마도 Kubernetes는 많은 종류의 volumes를 지우너하고 파드는 동시에 몇개든지 volume을 사용할 수 있다.
* core에서 volume은 단지 directory이고 아마 파드의 container에서 접근할 수 있는 어떤 data가 들어있을 것이다.
  * 어떻게 그런 directory가 될수 있는지, 이를 지원하는 중간 매체, 그리고 이것의 내용은 사용된 특정한 volume type에 의해 결정된다.
* volume을 사용하기 위해 파드는 어떤 volume을 파드에 제공할지(`.spec.volumes` 필드)와 container에서 어느 지점에 mount할지(`.spec.containers.volumeMounts` 필드) 지정한다. 
* container에서 프로세스는 Docker image와 volumes로 구성된 filesystem view를 볼 수 있다.
  * Docker image는 filesystem hierarchy의 root에 있고 모든 volumes는 image에서 지정된 path로 mount된다.
  * volumes는 다른 volumes이나 다른 volumes에 hard link를 가지는 곳 위에 mount 될 수 없다.
  * 파드에서 각 container는 독립적으로 어디에 각 volume을 mount할지 지정해야한다.

## Types of Volumes

* Kubernetes는 몇몇 타입의 volumes를 지원한다.
  * [awsElasticBlockStore](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore)
  * [azureDisk](https://kubernetes.io/docs/concepts/storage/volumes/#azuredisk)
  * [azureFile](https://kubernetes.io/docs/concepts/storage/volumes/#azurefile)
  * [cephfs](https://kubernetes.io/docs/concepts/storage/volumes/#cephfs)
  * [cinder](https://kubernetes.io/docs/concepts/storage/volumes/#cinder)
  * [configMap](https://kubernetes.io/docs/concepts/storage/volumes/#configmap)
  * [csi](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
  * [downwardAPI](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi)
  * [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
  * [fc (fibre channel)](https://kubernetes.io/docs/concepts/storage/volumes/#fc)
  * [flexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexVolume)
  * [flocker](https://kubernetes.io/docs/concepts/storage/volumes/#flocker)
  * [gcePersistentDisk](https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk)
  * [gitRepo (deprecated)](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo)
  * [glusterfs](https://kubernetes.io/docs/concepts/storage/volumes/#glusterfs)
  * [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
  * [iscsi](https://kubernetes.io/docs/concepts/storage/volumes/#iscsi)
  * [local](https://kubernetes.io/docs/concepts/storage/volumes/#local)
  * [nfs](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)
  * [persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim)
  * [projected](https://kubernetes.io/docs/concepts/storage/volumes/#projected)
  * [portworxVolume](https://kubernetes.io/docs/concepts/storage/volumes/#portworxvolume)
  * [quobyte](https://kubernetes.io/docs/concepts/storage/volumes/#quobyte)
  * [rbd](https://kubernetes.io/docs/concepts/storage/volumes/#rbd)
  * [scaleIO](https://kubernetes.io/docs/concepts/storage/volumes/#scaleio)
  * [secret](https://kubernetes.io/docs/concepts/storage/volumes/#secret)
  * [storageos](https://kubernetes.io/docs/concepts/storage/volumes/#storageos)
  * [vsphereVolume](https://kubernetes.io/docs/concepts/storage/volumes/#vspherevolume)
* 추가적인 contribution은 환영한다.

### awsElasticBlockStore

* `awsElasticBlockStore` volume은 Amazon Web Services(AWS) EBS Volume을 파드에 mount한다.
  * 파드가 삭제될 때 지워지는 `emtpyDir`과는 다르게 EBS volume의 내용들은 보존되고 volume은 그저 unmount될 뿐이다.
  * 이는 EBS volume은 data로 pre-populated될 수 있고 그 data는 파드간에 "handed off" 될 수 있음을 의미한다.

> Caustion : 이를 사용하기 전에 반드시 `aws ec2 create-volume`이나 AWS API를 이용해서 EBS volume을 생성해야한다.

* `awsElasticBlockStore` volume을 사용할 때 몇가지 제약사항이 있다.
  * 파드가 동작중인 노드는 반드시 AWS EC2 instances여야 한다.
  * 이 instances는 EBS volume과 같은 region과 availability-zone에 있어야 한다.
  * EBS는 volume mounting에 하나의 EC2 instance만을 지원한다.

#### Creating an EBS volume

* 파드에 EBS volume을 사용하기 전에 먼저 생성해야 한다.

  ```shell
  aws ec2 create-volume --availability-zone=eu-west-1a --size=10 --volume-type=gp2
  ```

* zone이 cluster가 들어있는 zone과 일치함을 확실히 해라. (그리고 크기와 EBS volume type이 사용하기에 적합한지도 확인해라.)

#### CSI Migration

**FEATURE STATE : ** `Kubernetes v1.14` - alpha

* awsElasticBlockStore의 CSI Migration feature가 활성화되면 존재하고 있는 in-tree plugin에서 `ebs.csi.aws.com` Container Storage Interface(CSI) Driver로 모든 plugin operations로 shim(redirect)한다.
  * 이 기능을 사용하기 위해선 AWS EBS CSI Driver가 cluster에서 반드시 설치되어 있어야하고 `CSIMigration`과 `CSIMigrationAWS` Alpha features가 반드시 활성화되어야 한다.

### azureDisk

### cephfs

* `cephfs` volume은 존재하는 CephFS volume을 파드에 mount되도록 한다.
  * 파드가 삭제되면 지우는 `emptyDir`과는 다르게 `cephfs` volume은 보존되고 드물게 unmounted된다.
  * 이는 CephFS volume이 데이터를 가지고 pre-populated될 수 있고 그 데이터는 파드간에 "handed off"될 수 있음을 의미한다.
  * CephFS는 동시에 여러 writer에 의해 mount될 수 있다.

> Caution : 사용하기 전에 share exported이고 작동중인 Ceph server를 가지고 있어야 한다.

* 자세한 사항은 [CephFS example](https://github.com/kubernetes/examples/tree/master/volumes/cephfs/) 참고

### configMap

* `configMap` resource는 configuration data를 파드에 주입시키는 방법을 제공한다.

  * `ConfigMap` object에 저장된 데이터는 `configMap` 타입의 volume에서 참조될 수 있고 그러면 파드에서 동작중인 containerized applications에 의해 소비될 수 있다.

* `configMap` object를 참조할 때 이를 참조하기 위해 volume에서 이름을 제공해주어야 한다.

  * 또한 ConfigMap에서 지정된 entry를 사용하기 위해 path를 customize할 수 있다.

  * 예를 들어 `configmap-pod`라고 불리는 파드 위에 `log-config` ConfigMap을 mount하기 위해 다음과 같이 YAML을 작성해야 한다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: configmap-pod
    spec:
      containers:
        - name: test
          image: busybox
          volumeMounts:
            - name: config-vol
              mountPath: /etc/config
      volumes:
        - name: config-vol
          configMap:
            name: log-config
            items:
              - key: log_level
                path: log_level
    ```

* `log-config` ConfigMap은 volume으로써 mount되었고 모든 `log_level` entry 안의 내용들은 파드의 "`/etc/config/log_level`" path에 mount 되었다.
  
  * 이 path가 volume의 `mountPath`와 key가 `log_level`인 `path`로부터 왔음을 주목하라.

> Caution : 반드시 사용전에 ConfigMap을 생성해야 한다.

> Note : subPath volume mount로써 ConfigMap을 사용하는 container는 ConfigMap 업데이트를 받지 못할 것이다.

### rbd

* `rbd` volume은 [Rados Block Device](http://ceph.com/docs/master/rbd/rbd/) volume을 파드에 mount되게 해준다.
  * 파드가 삭제되면 지워지는 `emtpyDir`과는 다르게 `rbd` volume의 내용들은 보존되고 volume은 드물게 unmount된다.
  * 이는 RBD volume이 데이터로 pre-populated될 수 있고 그 데이터는 파드간에 "handed off"될 수 있음을 의미한다.

> Caution : RBD를 사용하기 전에 반드시 Ceph installation이 동작하도록 하 해야한다.

* RBD의 feautre는 동시에 여러 consumer에 의해서 read-only로 mount될 수 있다는 것이다.
  * 이는 volume을 dataset으로 pre-populate할 수 있고 이를 원하는 만큼 많은 파드에서 병렬적으로 서비스할 수 있음을 의미한다.
  * 불행히도 RBD volume은 오직 하나의 consumer에서만 read-write mode로 mount될 수 있다. - 동시에 여러 writer가 있는것은 허용되지 않는다.
* 자세한 사항은 [RBD example](https://github.com/kubernetes/examples/tree/master/volumes/rbd) 참고

### secret

* `secret` volume은 password같은 민감한 정보를 파드에 보내기 위해 사용된다.
  * Kubernetes API에 secrets를 저장할 수 있고 이를 Kubernetes에 직접 연관되지 않고 파드가 사용하도록 하기 위해 file로 mount할 수 있다.
  * `secret` volume은 tmpfs(RAM기반 filesystem)기반으로 구성되어있어 non-volatile storage에 절대 적히지 않는다.

> Caution : 사용 전에 반드시 Kubernetes API에 secret을 생성해야 한다.

> Note : subPath volume mount로써 Secret을 사용하는 container는 Secret 업데이트를 받지 못할 것이다.

* 자세한 사항은 [여기](https://kubernetes.io/docs/user-guide/secrets)에 설명되어있다.

## Using subPath

* 가끔 하나의 파드 안에서 하나의 volume을 다수의 사용자에 대해서 공유하는 것이 유용할 수 있다.

  * `volumeMounts.subPath` 속성이 root 대신에 참조된 volume안에서 sub-path를 지정하는데 사용된다.

* 하나의 공유된 volume을 사용하는 LAMP stack (Linux Apache Mysql PHP)인 파드에서의 예시이다.

  * HTML contents는 `html` 폴더로 매핑이 되고 database는 `mysql` 폴더에 저장이 된다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-lamp-site
  spec:
      containers:
      - name: mysql
        image: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: &quot;rootpasswd&quot;
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: site-data
          subPath: mysql
      - name: php
        image: php:7.0-apache
        volumeMounts:
        - mountPath: /var/www/html
          name: site-data
          subPath: html
      volumes:
      - name: site-data
        persistentVolumeClaim:
          claimName: my-lamp-site-data
  ```

### Using subPath with expanded environment variables

**FEATURE STATE : ** `Kubernetes v1.15` - beta

* `subPathExpr` 필드를 사용하여 `subPath` directory names를 Downward API environment variables로부터 구성할 수 있다.

  * 이 feature를 사용하기 전에 반드시 `VolumeSubpathEnvExpansion` feature gate를 활성화 해야한다.
  * `subPath`와 `subPathExpr` 속성은 상호 배타적이다.

* 이 예시에서 파드는 `subPathExpr`를 사용하여 directory `pod1`을 hostPath volume `/var/log/pods`에 생성하고 Downward API로부터 파드 이름을 사용한다.

* host directory `/var/log/pods/pod1`는 container에서 `/logs`에 mount된다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod1
  spec:
    containers:
    - name: container1
      env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            apiVersion: v1
            fieldPath: metadata.name
      image: busybox
      command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
      volumeMounts:
      - name: workdir1
        mountPath: /logs
        subPathExpr: $(POD_NAME)
    restartPolicy: Never
    volumes:
    - name: workdir1
      hostPath:
        path: /var/log/pods
  ```

## Resources

* `emptyDir` volume의 storage media(Disk, SSD 등)는 kubelet root dir(보통 `/var/lib/kubelet`)를 가지고있는 filesystem의 medium에 의해 결정된다.
  * 얼마나 많은 `emptyDir`이나 `hostPath` volume 공간을 소비할지는 제한이 없으며 container나 파드들 같의 isolation이 없다.
* 나중에는 `emptyDir`과 `hostPath` volume이 resource specification을 사용하여 특정한 양만큼의 공간을 요청하도록 할 수 있을 것이며 사용할 몇몇 media 타입을 가지는 cluster에 대해 media 종류를 선택할 수 있을 것이다.

## Out-of-Tree Volume Plugins

* Out-of-tree volume plugin은 Container Storage Interface(CSI)와 Flexvolume을 포함한다.
  * 이것들은 storage vendor가 Kubernetes repository에 custom storage plugin을 추가하지 않고도 이를 생성할 수 있게 해준다.
* CSI와 Flexvolume을 소개하기 전에 모든 volume plugins(위에 리스트된 volume 타입처럼)들은 core Kubernetes binary와 build, link, compile, ship되는 "in-tree"이며 Kubernetes API를 확장하는 것들이다.
  * 이는 새로운 storage system을 Kubernetes(volume plugin)에 추가하는 것이 core Kubernetes code repository에서 코드를 확인하는 것을 필요로 한다는 것을 의미한다.
* CSI와 Flexvolume 모두 volume plugins가 Kubernetes code base와 별개로 개발되도록 하고 extension으로써 Kubernetes cluster에 deploy(설치)될 수 있도록 한다.
* storage vendor가 out-of-tree volume plugin을 생성하는 방법을 찾는다면 [이 FAQ](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md)를 보아라.

### CSI

* [Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)(CSI) 임의의 storage system을 container workload에 노출시키기 위해 Kubernetes같은 container orchestration systems에 대한 표준 interface를 정의한다.
* 자세한 정보는 [CSI design proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)를 참조하라.
* CSI에 대한 지원은 Kubernetes v1.9에서 alpha로 소개되었고 Kubernetes v1.10에서 beta로 옮겨졌으며 Kubernetes v1.13에서 GA가 되었다.

> Note : CSI spec version 0.2와 0.3에 대한 지원은 Kubernetes v1.13에서 deprecated되었고 다음 release에서 삭제될 것이다.

> Note : CSI drivers는 아마 모든 Kubernetes releases에서 사용가능하지는 않을 것이다. 

* CSI가 호환되는 volume driver가 Kubernetes cluster에 한번 배포되면 사용자는 `csi` volume type을 사용하여 attach, mount 등을 할 수 있다.
  * volume은 CSI driver에 의해 노출된다.
* `csi` volume type은 파드에서 직접적인 참조를 지원하지 않고 `PersistentVolumeClaim` 오브젝트를 통해 파드에서 참조될 것이다.
* 다음 필드는 storage administrator가 CSI persistent volume을 configure하도록 해준다.
  * `driver`
    * string 값으로 사용할 volume driver의 이름을 지정한다.
    * 이 값은 반드시 CSI spec에 정의된 CSI drver에 의해 `GetPluginInfoResponse`에 리턴된 값과 일치해야 한다.
    * 이는 Kubernetes에 의해 어느 CSI driver로 호출을 할지 확인하는데 사용되고, CSI driver component에 의해 어느 PV 오브젝트가 CSI driver에 속해있는지 확인하는데 사용된다.
  * `volumeHandle`
    * string 값으로 volume을 유일하게 지정하는 것이다.
    * 이 값은 반드시 CSI spec에서 정의된 CSI driver에 의한 `CreateVolumeResponse`필드의 `volume.id`에서 리턴된 값과 일치해야 한다.
    * volume을 참조할 때 이 값은 CSI volume driver로 향하는 모든 호출에서 `volume_id`로 전달된다.
  * `readOnly`
    * optional boolean 값으로 volume이 read only로 "ControllerPublished"(attached) 될 수 있는지에 대한 값이다.
    * 기본값은 false이다.
    * 이 값은 `ControllerPublishVolumeRequest`에서 `readonly`필드를 통해서 CSI driver로 전달된다.
  * `fsType`
    * PV의 `VolumeMode`가 `FileSystem`이면 이 필드는 아마 volume을 mount하는 데 사용될 filesystem을 지정하는데 쓰일 것이다.
    * volume이 포멧되지 않았고 포멧을 지원한다면 이 값은 volume을 포멧하는데 쓰인다.
    * 이 값은 `ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, `NodePublishVolumeRequest`를 통해 CSI driver로 전달될 것이다.
  * `volumeAttributes`
    * (string : string) map으로 volume의 정적인 속정을 지정한다.
    * 이 맵은 반드시 CSI spec에서 정의한 CSI driver의 `CreateVolumeResponse`필드의 `volume.attributes`에서 리턴된 map과 일치해야한다.
    * 이 map은 `ControllerPublicshVolumeRequest`, `NodeStageVolumeRequest`, `NodePublishVolumeRequest`에서 `volume_attributes`필드를 통해서 CSI driver로 전달된다.
  * `controllerPublishSecretRef`
    * CSI `ControllerPublishVolume`과 `ControllerUnpublishVolume` 호출을 완료하기 위해 CSI driver로 보낼 때 민감한 정보를  가진 secret object에 대한 참조
    * 이 필드는 optional로 secret이 필요하지 않을 때는 비어있을 것이다.
    * secret 오브젝트가 둘 이상의 secret을 가지고 있다면 모든 secret은 전달된다.
  * `nodeStageSecretRef`
    * CSI `NodeStageVolume` 호출을 완료하기 위해 CSI driver로 보내는 민감한 정보를 포함한 secret 오브젝트에 대한 참조
    * 이 필드는 optional로 secret이 필요하지 않다면 비어있을 것이다.
    * secret 오브젝트가 둘 이상의 secret을 포함하면 모든 secret은 전달된다.
  * `nodePublishSecretRef`
    * CSI `NodePublishVolume` 호출을 완료하기 위해 CSI driver로 전달하는 민감한 정보를 포함한 secret 오브젝트에 대한 참조
    * 이 필드는 optional로 secret이 필요하지 않다면 비어있을 것이다.
    * secret 오브젝트가 둘 이상의 secret을 필요로 한다면 모든 secret은 전달된다.

#### CSI raw block volume support

**FEATURE STATE : ** `Kubernetes v1.14` - beta

* version 1.11에서부터 CSI는 이전의 Kubernetes 버전에서 소개된 raw block volume feature에 의존하는 raw block volume에 대한 지원을 시작했다.
  * 이 feature는 external CSI driver를 가진 vendor가 Kubernetes workload에서 raw block volume에 대한 지원을 implement할 수 있도록 한다.
* CSI block volume에 대한 지원은 feature-gated지만 기본적으로 활성화 되어있다.
  * 이 feature를 위해서 `BlockVolume`과 `CSIBlockVolume`은 반드시 활성화 되어야 하는 feature gate이다.
* 어떻게 [PV/PVC를 raw block volume support로 세팅하는지](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support) 배워보아라.

#### CSI ephemeral volumes

**FEATURE STATE : ** `Kubernetes v.15` - alpha

* 이 feature는 CSI volume을 PersistentVolume대신에 Pod specification에 직접 embedded되어 있도록 한다.

  * 이 방법으로 지정된 volume은 emphemeral이고 파드가 재시작할 때 유지되지 않는다.

* 예시

  ```yaml
  kind: Pod
  apiVersion: v1
  metadata:
    name: my-csi-app
  spec:
    containers:
      - name: my-frontend
        image: busybox
        volumeMounts:
        - mountPath: "/data"
          name: my-csi-inline-vol
        command: [ "sleep", "1000000" ]
    volumes:
      - name: my-csi-inline-vol
        csi:
          driver: inline.storage.kubernetes.io
          volumeAttributes:
                foo: bar
  ```

* 이 feature는 CSIInlineVolume feature gate가 활성화 되어있어야 한다.

  ```
  --feature-gates=CSIInlineVolume=true
  ```

* CSI ephemeral volume은 CSI driver의 subset만 지원한다.

  * CSI driver의 리스트는 [이곳](https://kubernetes-csi.github.io/docs/drivers.html)에서 확인하라.

## Developer resources

* CSI driver를 개발하는 방법에 대한 더 많은 정보는 [kubernetes-csi documentation](https://kubernetes-csi.github.io/docs/)를 참조하라.

### Migrating to CSI drivers from in-tree plugins

**FEATURE STATE : ** `Kubernetes v1.13` - alpha

* CSI Migration feature는 활성화 되었을 때 in-tree plugin에 대한 operation을 이와 일치하는 CSI plugins(설치되었고 configure되었다고 여겨지는)으로 direct한다.
  * feature는 seamless fashion에서 operation을 re-route하기 위해 필요한 translation logic과 shim를 implements한다.
  * 결과적으로 operator는 in-tree plugin을 대체하는 CSI driver를 transition할 때 현재 Storage Classes, PVs나 PVCs(in-tree plugins를 참조하는)들에 대해 configuration 변화를 줄 필요가 없다.
* alpha 상태에서 지원하는 operations과 feature는 provisioning/delete, attach/detach/, mount/unmount와 volumes의 resizing을 포함한다.
* CSI Migration을 지원하고 이에 대한 CSI driver를 가진 In-tree plugins는 위의 "Types of Volumes" 섹션에 리스트가 있다.

### Flexvolume

* Flexvolume은 out-of-tree plugin interface로 Kubernetes 1.2버전(CSI가 있기 전)부터 존재하였다.
  * driver로 interface에 exec-based model을 사용한다.
  * Flexvolume driver binaries는 각 노드(어떨 땐 마스터)에 반드시 사전 정의된 volume plugin path에 설치되어야 한다.
* 파드는 `flexvolume` in-tree plugin을 통해서 Flexvolume drvier와 함께 상호작용한다.
  * 더 자세한 사항은 [여기](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md)에서.

## Mount propagation

* Mount propagation은 container에 의해 mount된 volume에 대해서 같은 pod 안의 다른 container에게 공휴하거나 심지어 같은 node의 다른 pods에 대해 공유하도록 할 수 있다.

* volume의 Mount propagation은 `Container.volumeMounts`의 `mountPropagation` 필드에 의해서 관리된다.

  * `None`

    * 이 volume mount는 이 volume이나 host에서 모든 subdirectory에대해 mount된 그 어떤 mount도 받지 않는다.
    * 비슷한 방식으로 conatiner에 의해 생성된 no mounts는 host에 의해서 보일것이다.
    * default mode이다.
    * 이 mode는 `private` mount propagation으로서 Linux kernel documentation과 일치한다.

  * `HostToContainer`

    * 이 volume mount는 이 volume에 또는 어느 subdirectory에 mount된 모든 mounte들을 받을 것이다.
    * 다시 말해서 host가 volume mount안의 어떤것이든 mount하면 container는 그곳에 mount된 것을 볼 수 있다.
    * 비슷하게 같은 volume에 `Bidrectional` mount propagation인 어떤 파드에 어떤것이든 mount하면 `HostToContainer` mount propagation인 container는 이를 볼 수 있다.
    * 이 모드는 Linux kernal documentation과 `rslave` mount propagation과 일치한다.

  * `Bidirectional`

    * 이 volume mount는 `HostToContainer` mount와 동일한 동작을 한다.
    * 게다가 container에 의해 생성된 모든 volume mounts는 host, 같은 volume을 사용하는 모든 파드의 모든 container 로 propagated back할 것이고 
    * 이 mode의 전형적인 use case는 Flexvolume이나 CSI driver가 있는 파드나 `hostPath` volume을 이용하여 host에 어떤 것을 mount할 필요가 있는 파드이다.
    * 이 모드는 Linux kernel documentation에 기술된 `rshared` mount propagation과 같다.

    > Caution : `Bidirectional` mount propagation은 위험할 수 있다. 이는 host 운영체제에 피해를 줄 수 있고 그래서 privileged container에서만 허용된다. Linux kernel behavior를 잘 다룰 수 있어야 한다. 게다가 파드에서 container에 의해 생성된 모든 volume mount들은 반드시 termination일 때 container에 의해서 destroyed(unmounted) 되어야 한다.

### Configuration

* mount propagation가 몇몇 deployments (CoreOS, RedHat/Centos, Ubuntu) mount share에서 적절하게 동작하기 전에 아래와 같이 Docker에서 올바르게 설정해주어야 한다.

* Docker의 systemd service file을 수정해라.

  * `MountFlags`를 다음과 같이 하라.

    ```shell
    MountFlags=shared
    ```

  * 혹은 `MountFlags=slave`가 있다면 지워라.

  * 그 후 Docker daemon을 재시작하라.

    ```shell
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

