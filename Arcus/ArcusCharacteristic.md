# Arcus 특징

## Memcached

* 아커스는 Memcached를 확장한 구조.
* Memcached + Zookeeper(클라우드를 관리하는 admin 서버) + Collection

## Arcus 특징과 구현

1. cluster-aware server / client : cluster의 변경사항을 서버, 클라이언트가 바로바로 알 수 있다.
2. partial fault-tolerant
3. data distribution
4. collections (b+tree, list, set)
5. data rebalancing
6. no replication
7. no persistence

## Cluster-aware client / server

### 서버

* 서버가 실행되면 자동으로 cloud 에 참여한다.
* 서버에 문제가 생기면 알아서 cloud 에서 탈퇴한다.

### 클라이언트

* cloud 에 참여하는 Arcus 서버 목록을 알 수 있다.
* cloud 에 참여하는 Arcus 서버 목록의 변경을 알 수 있다.

이러한 모든 것을 통제하는 admin controller가 존재.

## 구성 요소별 역할

## Arcus 서버

1. Cache 데이터 저장소
2. Arcus 서버 프로세스가 start/ stop 될 때 Admin에 통보
3. 문제가 발생한 Arcus 프로세스는 스스로 중지

## Admin (Zookeeper ensemble)

1. 서비스코드 별로 Arcus 서버 목록을 관리하고 변경사항이 있을 때 클라이언트에게 통지
2. Arcus 서버와의 세션이 만료되면 Arcus 서버 목록에서 해당 서버를 제거

## Arcus 클라이언트

1. Admin 으로부터 서비스코드에 할당된 사용가능한 Arcus 서버 목록을 받아온다.
2. Arcus 서버 목록의 변경을 지속적으로 감시한다. (zk watcher)
3. Arcus 서버 목록 변경시, 최신 서버목록 기반으로 요청을 rehashing 한다.
4. Arcus 서버와 통신
