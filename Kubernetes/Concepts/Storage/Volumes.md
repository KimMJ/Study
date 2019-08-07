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

#### AWS EBS Example configuration

[Edit This Page](https://github.com/kubernetes/website/edit/master/content/en/docs/concepts/storage/volumes.md)

# Volumes

On-disk files in a Container are ephemeral, which presents some problems for non-trivial applications when running in Containers. First, when a Container crashes, kubelet will restart it, but the files will be lost - the Container starts with a clean state. Second, when running Containers together in a `Pod` it is often necessary to share files between those Containers. The Kubernetes `Volume` abstraction solves both of these problems.

Familiarity with [Pods](https://kubernetes.io/docs/user-guide/pods) is suggested.

- [Background](https://kubernetes.io/docs/concepts/storage/volumes/#background)
- [Types of Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)
- [Using subPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath)
- [Resources](https://kubernetes.io/docs/concepts/storage/volumes/#resources)
- [Out-of-Tree Volume Plugins](https://kubernetes.io/docs/concepts/storage/volumes/#out-of-tree-volume-plugins)
- [Mount propagation](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation)
- [What's next](https://kubernetes.io/docs/concepts/storage/volumes/#what-s-next)

## Background

Docker also has a concept of [volumes](https://docs.docker.com/engine/admin/volumes/), though it is somewhat looser and less managed. In Docker, a volume is simply a directory on disk or in another Container. Lifetimes are not managed and until very recently there were only local-disk-backed volumes. Docker now provides volume drivers, but the functionality is very limited for now (e.g. as of Docker 1.7 only one volume driver is allowed per Container and there is no way to pass parameters to volumes).

A Kubernetes volume, on the other hand, has an explicit lifetime - the same as the Pod that encloses it. Consequently, a volume outlives any Containers that run within the Pod, and data is preserved across Container restarts. Of course, when a Pod ceases to exist, the volume will cease to exist, too. Perhaps more importantly than this, Kubernetes supports many types of volumes, and a Pod can use any number of them simultaneously.

At its core, a volume is just a directory, possibly with some data in it, which is accessible to the Containers in a Pod. How that directory comes to be, the medium that backs it, and the contents of it are determined by the particular volume type used.

To use a volume, a Pod specifies what volumes to provide for the Pod (the `.spec.volumes` field) and where to mount those into Containers (the `.spec.containers.volumeMounts` field).

A process in a container sees a filesystem view composed from their Docker image and volumes. The [Docker image](https://docs.docker.com/userguide/dockerimages/) is at the root of the filesystem hierarchy, and any volumes are mounted at the specified paths within the image. Volumes can not mount onto other volumes or have hard links to other volumes. Each Container in the Pod must independently specify where to mount each volume.

## Types of Volumes

Kubernetes supports several types of Volumes:

- [awsElasticBlockStore](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore)
- [azureDisk](https://kubernetes.io/docs/concepts/storage/volumes/#azuredisk)
- [azureFile](https://kubernetes.io/docs/concepts/storage/volumes/#azurefile)
- [cephfs](https://kubernetes.io/docs/concepts/storage/volumes/#cephfs)
- [cinder](https://kubernetes.io/docs/concepts/storage/volumes/#cinder)
- [configMap](https://kubernetes.io/docs/concepts/storage/volumes/#configmap)
- [csi](https://kubernetes.io/docs/concepts/storage/volumes/#csi)
- [downwardAPI](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi)
- [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [fc (fibre channel)](https://kubernetes.io/docs/concepts/storage/volumes/#fc)
- [flexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexVolume)
- [flocker](https://kubernetes.io/docs/concepts/storage/volumes/#flocker)
- [gcePersistentDisk](https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk)
- [gitRepo (deprecated)](https://kubernetes.io/docs/concepts/storage/volumes/#gitrepo)
- [glusterfs](https://kubernetes.io/docs/concepts/storage/volumes/#glusterfs)
- [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- [iscsi](https://kubernetes.io/docs/concepts/storage/volumes/#iscsi)
- [local](https://kubernetes.io/docs/concepts/storage/volumes/#local)
- [nfs](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)
- [persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim)
- [projected](https://kubernetes.io/docs/concepts/storage/volumes/#projected)
- [portworxVolume](https://kubernetes.io/docs/concepts/storage/volumes/#portworxvolume)
- [quobyte](https://kubernetes.io/docs/concepts/storage/volumes/#quobyte)
- [rbd](https://kubernetes.io/docs/concepts/storage/volumes/#rbd)
- [scaleIO](https://kubernetes.io/docs/concepts/storage/volumes/#scaleio)
- [secret](https://kubernetes.io/docs/concepts/storage/volumes/#secret)
- [storageos](https://kubernetes.io/docs/concepts/storage/volumes/#storageos)
- [vsphereVolume](https://kubernetes.io/docs/concepts/storage/volumes/#vspherevolume)

We welcome additional contributions.

### awsElasticBlockStore

An `awsElasticBlockStore` volume mounts an Amazon Web Services (AWS) [EBS Volume](http://aws.amazon.com/ebs/) into your Pod. Unlike `emptyDir`, which is erased when a Pod is removed, the contents of an EBS volume are preserved and the volume is merely unmounted. This means that an EBS volume can be pre-populated with data, and that data can be “handed off” between Pods.

> **Caution:** You must create an EBS volume using `aws ec2 create-volume` or the AWS API before you can use it.

There are some restrictions when using an `awsElasticBlockStore` volume:

- the nodes on which Pods are running must be AWS EC2 instances
- those instances need to be in the same region and availability-zone as the EBS volume
- EBS only supports a single EC2 instance mounting a volume

#### Creating an EBS volume

Before you can use an EBS volume with a Pod, you need to create it.

```shell
aws ec2 create-volume --availability-zone=eu-west-1a --size=10 --volume-type=gp2
```

Make sure the zone matches the zone you brought up your cluster in. (And also check that the size and EBS volume type are suitable for your use!)

#### AWS EBS Example configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

#### CSI Migration

**FEATURE STATE : ** `Kubernetes v1.14` - alpha

* awsElasticBlockStore의 CSI Migration feature가 활성화되면 존재하고 있는 in-tree plugin에서 `ebs.csi.aws.com` Container Storage Interface(CSI) Driver로 모든 plugin operations로 shim(redirect)한다.
  * 이 기능을 사용하기 위해선 AWS EBS CSI Driver가 cluster에서 반드시 설치되어 있어야하고 `CSIMigration`과 `CSIMigrationAWS` Alpha features가 반드시 활성화되어야 한다.



## Using subPath

* 가끔 