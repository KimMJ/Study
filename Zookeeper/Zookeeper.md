# Zookeeper

### Distibuted System이란?

> A distributed system consists of multiple computers that commnunicate through a computer network and interact with each other to achieve a common goal. 

분산 어플리케이션을 위한 분산 협력 시스템

### Distributed Computing에 대한 잘못된 생각

* 네트워크는 reliable하다.
* Latency가 없다
* Bandwidth가 무한하다
* 네트워크는 secure하다
* Topology가 변경되지 않는다
* 하나의 administrator가 있다
* 전송 비용이 없다.
* 네트워크는 homogeneous하다

### 분산 시스템에서의 Coordination

여러개의 노드들은 반드시 함께 동작해야 한다. 이를 맞추는 것은 꽤 어렵다.

### Zookeeper

> Zookeeper allows distributed proccesses to coordinate with each other through a shared hierarchical name space of data registers.

* 오픈소스로써, 분산 어플리케이션에서 coordination에 높은 퍼포먼스를 낸다.

#### Use Cases

* Configuration Management
  * Cluster member node들은 중앙에 있는 소스로부터 configuration을 받아온다.
  * 쉽고 간단하게 deployment/provisioning 할 수 있다.
* Distributed Cluster Management
  * Node의 join/leave
  * Node의 실시간 상태
* Naming service - DNS
* Distrubted synchronization - locks, barriers, queues
* 분산 시스템에서의 리더 선출
* 집중화 되어있고 reliable한 data registry

### Zookeeper Service

![Zookeeper Service](https://t1.daumcdn.net/cfile/tistory/998D8B495BB9AB403F)

* Zookeeper service는 머신들에 복제가 된다.
* 모든 머신들은 data의 copy를 저장한다.(in-memory)
* 서비스 시작시에 리더가 선출된다.
* 클라이언트는 하나의 주키퍼 서버에만 연결을 하고 TCP 연결을 유지한다.
* 클라이언트는 주키퍼 서버에서 읽을 수 있고 리더를 통해서 쓴다. 이 때 majority consensus 방법을 이용한다.

### Zookeeper Data Model

* Zookeeper는 계층형 namespace를 가지고 있다.
* 네임스페이스에서 각각의 노드는 ZNode라고 부른다.
* 모든 ZNode는 데이터를 가지고 있다. 그리고 추가적으로 자식을 가질수도 있다.

#### Zookeeper Reads & Writes

* Read 요청은 클라이언트가 연결이 된 주키퍼 서버에서 local로 이루어진다.
* Write 요청은 리더에게 forward되고 과반수 체크를 통해서 쓸 수 있다는 응답을 받으면 쓰기를 한다.

### Guarantees

* Sequential Consistency : 업데이트가 순차적으로 적용된다.
* Atomicity : 업데이트는 성공/실패만 있다.
* Single System Image : 클라이언트는 어느 Zookeeper서버에 연결되어 있던지 상관 없이 동일한 service view를 본다.
* Reliability : 업데이트는 한번 적용이 되면 다른 클라이언트가 write하기 전까지 계속 유지가 된다.
* Timeliness : 클라이언트의 시스템에 대한 view는 eventual consistency(궁극적 일관성)이다.



ZooKeeper는 데이터를 디스크에 영구 저장하긴 하지만, 빠른 처리를 위해 모든 트리 노드를 메모리에 올려놓고 처리한다. 즉 대규모의 데이터를 처리하기엔 무리가 있다. 따라서 Memcached나 Redis와 같은 캐시 서버와는 다른 용도로 쓰인다.

ZooKeeper의 여러 기능 가운데 가장 활용도가 높은 부분이 바로 데이터 변경을 감시하여 콜백을 실행하는 감시자(Watcher)다. 감시자를 트리 내의 특정 노드에 등록하면 별도로 데이터 폴링을 하지 않더라도 노드 변경 사항을 전달받을 수 있다. 따라서 서비스에 동적으로 설정을 반영하고자 할 때 ZooKeeper를 많이 사용한다.

## Zookeeper-docker

```shell
docker run --name some-zookeeper --restart always -d zookeeper
```









---

  이 글에서는 구축과정을 따라가기 쉽도록 Kafka 클러스터와 Zookeeper 클러스터를 동일한 장비에 구축하고 있는데, 실무에서는 Zookeeper 클러스터와 Kafka를 별도로 구축하길 바란다. 또한, Kafka에서는 Zookeeper에 쓰기 연산이 대량으로 발생하므로 다른 서비스에서 사용하고 있는 Zookeeper 클러스터를 공유해서 쓰는것 또한 권장하지 않는다. 정리하자면 Kafka를 위한 Zookeeper 클러스터를 별도의 장비에 구축하고, 해당 Zookeeper 클러스터는 Kafka만 사용하도록 하는 것을 권장한다.

출처: https://epicdevs.com/20 [Epic Developer]  