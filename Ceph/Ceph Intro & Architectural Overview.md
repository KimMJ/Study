# Ceph Intro & Architectural Overview

Cloud Services는 Compute, Network, Storage 분야로 나눌 수 있고 그 중 ceph는 storage 분야이다. 

`You - Technology - Your Data`

처음에는 몇몇 사람들이 하나의 컴퓨터를 통해서 storage에 접속을 했다. 그러다가 점점 이용자 수가 늘어나서 엄청 많은 사람들이 하나의 컴퓨터를 통해 여러대의 storage에 접속을 하게 되었고, 이 트래픽을 감당하기 위해서 scale-up(컴퓨터 자체의 성능 향상)을 했다. 그러나 이러한 것은 한계점이 있었고, scale-out을 생각하게 되었다. 이 scale-out된 다수의 컴퓨터들을 하나의 박스로 묶어 관리하기 쉽도록 하였다. 이를 `Storage Appliance`라고 한다.

Storage Appliance는 Proprietary hardware - Proprietary software - support and maintenance로 구성되었다. 여기서 Cloud로 바꾸면 Standard hardware - Open Source Software(Ceph) - (Enterprise subscription) 으로 구성이 된다.

## Design Consideration

|    philosophy     |           design           |
| :---------------: | :------------------------: |
|    open source    |          scalable          |
| community-focused | no single point of failure |
|                   |       software based       |
|                   |       self-managing        |

## Marketing Diagram

![Marketing Diagram](img\marketing diagram.PNG)

3가지 인터페이스로 cluster의 데이터에 접근할 수 있다. 

## Technical Overview

![technical overview](img\technical overview.PNG)

### RADOS

* reliable, autonomous, distributed object store
* self-healing, self-managing, intelligent storage nodes

![RADOS](img\RADOS.PNG)

OSD : object storage demon

disk, file system, osd가 하나의 노드를 이루고 (sw layer) 이 노드들이 모여서 클러스터를 이룬다. 

## Type of cluster member

#### 파란색 (OSDs)

* providing access for data
* 10s to 10000s in cluster
* 보통 디스크당 하나
* 클라이언트에게 저장된 오브젝트 제공
* replication이나 recovery 를 위한 intelligence peer 제공

#### 검정색 (Monitors)

* cluster의 membership과 state 관리
* distributed decision-making을 위한 의견 제시. 
* small, odd number (투표해야 하기 때문)
* 클라이언트에게 저장된 오브젝트를 제공하지 않음.

### LIBRADOS

![LIBRADOS](img\LIBRADOS.PNG)

* RADOS에 직접적으로 앱이 접근할 수 있도록 하는 libaray
* C, C++, Java, Python, Ruby, PHP, Erlang 지원 
* 스토리지 노드에 직접 접속
* HTTP 오버헤드는 없음

### RADOSGW

![RADOSGW](img\RADOSGW.PNG)

* REST 기반 object storage **proxy**
* object를 저장하기 위해 RADOS 사용
* bucket, account에 대한 API 제공
* billing을 위한 사용량 측정
* S3, Swift 어플리케이션과 호환가능

### RBD (RADOS Block Device)

![RBD](img\RBD.PNG)

* RADOS에서 디스크 이미지의 storage
* VM을 호스트로부터 분리
* Image들은 클러스터에 스트라이핑(물리적인 디스크에 분산 저장) 된다.
* 스냅샷
* Copy-on-write 복제
* 지원
  * Mainline Linux Kernel
  * Qeme/KVM, native Xen - comming soon
  * OpenStack, CloudStack, Nebula, Proxmox

### CEPH FS

![Ceph Fs](img\CEPH FS.PNG)

#### Metadata Server

* POSIX-기반 shared filesystem을 위한 메타데이터 관리
  * directory hierarchy
  * 파일 메타데이터(timestamp, mode, etc..) 
* RADOS에 metadata 저장
* 클라이언트에게 데이터를 제공하지 않음
* shared filesystem에서만 필요

## What Makes Ceph Unique?

데이터가 어디에 저장되어있는지 아는 방법

1. 메타데이터를 통해서 어디에 저장되어있는지 작성해놓는다.
2. 특정한 기준으로 구간을 나누어서 저장한다.
3. CRUSH - ceph방식

### CRUSH

* Pseudo-random placement 알고리즘
  * 빠른 계산, no lookup
  * Repeatable, deterministic
* 통계적으로 uniform distribution (정규분포)
* stable mapping (input 데이터가 조금 바뀌면 output이 조금 바뀐다)
  * limited data migration on change
* rule-based 설정
  * infrastructure topology aware
  * adjustable replication
  * weighting

노드 중 하나가 장애가 났을 때는 동일한 데이터를 가지고 있던 노드에서 데이터를 다른 노드로 복사한다. 그 후 CRUSH가 잘 동작하도록 설정한다.

### Thin Provisioning

아직 이해하지 못함.

### Clustered metadata

metadata를 관리하는 노드들 또한 scale-out되어 있다. 여기서 어떻게 하나의 tree를 만들까 -> dynamic subtree partitioning



-----

https://www.youtube.com/watch?v=7I9uxoEhUdY&t=857s