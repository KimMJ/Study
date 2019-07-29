# Kafka

*<https://kafka.apache.org/intro>*

카프카는 분산 스트리밍 플랫폼이다.

스트리밍 플랫폼은 세가지 특징이 있다.

* Message Queue나 Enterpirse Messaging System처럼 *records*에 대해 publish하고 subscribe한다.
* fault-tolerant durable way로 *records*를 저장한다.
* *records*가 발생할 때 이를 스트림 형태로 처리한다.

카프카는 보통 두개의 범주에서 사용될 수 있다.

* 시스템과 어플리케이션 사이에서 실시간 데이터 파이프라인을 구축할 수 있다.
* 데이터의 스트림에 따라서 동작하는 실시간 스트리밍 어플리케이션을 만들 수 있다.

카프카가 어떻게 이러한 것들을 하는지 이해하기 위해서는 카프카의 대해 a-z 알아보자.

먼저 컨셉이다.

* 카프카는 multiple datacenter에서 동작하는 클러스터 위에서 동작한다.
* 카프카는 *records*를 *topics*라 불리는 카테고리에 저장한다.
* 각각의 *record*는 key, value, timestamp로 구성되어 있다.

카프카는 4개의 core API가 있다.

* **Producer API**는 *records*를 하나 이상의 Kafka *topic*으로 publish하는 API이다.
* **Consumer API**는 어플리케이션이 하나 이상의 *topic*을 subscribe하고 그곳에서 생성된 *records* 스트림을 처리할 수 있도록 한다.
* **Stream API**는 어플리케이션이 *stream processor*(하나 이상의 *topic*에서 input stream을 consume하고, 하나 이상의 *topic*에 대해 output stream을 produce함)처럼 동작할 수 있도록 하여 효과적으로 input stream을 output stream으로 변경할 수 있도록 한다**.** 
* **Connector API**는 존재하는 어플리케이션이나 데이터 시스템에 연결하는Kafka topic에 대해 재사용 가능한 producer, consumer를 만들고 동작할 수 있도록 한다. 예를 들어, 관계 데이터베이스에 대한 connector는 테이블의 모든 변경을 기록할 것이다.

Kafka에서는 클라이언트와 서버의 커뮤니케이션이 TCP protocol을 통해 간단하고, 고성능으로 처리가 된다. 이 프로토콜은 버전화 되어있고, 이전 버전에 대한 지원도 하고 있다.

#### <u>Topics and Logs</u>

Kafka가 어떻게 *topic*에 대한 *records stream*을 제공하는지 알아보자.

Topic은 발행할 *record*의 카테고리이거나 feed name이다. Kafka에서 *topic*은 항상 multi-subscriber이다. 즉, 토픽은 그곳에 쓰여진 데이터를 subscribe하는  0개 이상의 consumer를 가질 수 있다.

각각의 topic에 대해 kafka 클러스터는 다음과 같은 파티션된 로그를 가지고 있다. : ![Anatomy of a Topic](https://kafka.apache.org/22/images/log_anatomy.png)

각각의 파티션은 변경할 수 없는 정렬된 순서의 *record*이다. 이는 계속해서 구조화된 commit log에 추가된다. 한 파티션에 있는 *record*는 각각의 sequential id number를 가지게되고 이를 파티션 내에서 *record*를 독립적으로 구분지어주는 *offset*이라고 한다.

Kafka 클러스터는 모든 publish된 *record*를 consume이 되거나 말거나 한 설정가능한 보존 기간동안 유지한다. 예를 들어, 만약 보존 정책이 2일로 설정이 되어있으면 publish된지 2일동안은 계속 consume할 수 있고 그 후에는 이를 버려서 공간을 늘린다. Kafka의 performance는 데이터 사이즈에 대해서는 효과적으로 유지가 된다. 따라서 데이터를 오랜기간 저장하는 것은 문제가 발생되지 않는다. 

![log_consumer](https://kafka.apache.org/22/images/log_consumer.png)

사실, consumer마다 가지고있는 메타데이터는 offset이나 로그 안에서 그 consumer의 위치이다. 이 offset은 consumer에 의해 조작된다. 보통 consumer는 선형적으로 offset을 전진시키며 *record*를 읽는다. 하지만 사실 위치가 consumer에 의해서 조작이 되어 원하는 대로 어떠한 순서든 상관없이 *record*를 consumer할 수 있다. 예를 들어 consumer는 지나간 데이터를 다시 처리하기 위해 이전의 offset으로 돌아갈 수도 있고 가장 최근의 record로 skip하여 "지금" 순간부터 consume을 시작할 수 있다.

이러한 특징들은 Kafka의 consumer가 매우 가볍다는 것을 의미한다. 이것들은 cluster와 다른 consumer에게 큰 영향을 끼치지 않으면서 오고가고 할 수 있다. 예를 들어 당신은 우리의 command line tool로 어떤 topic의 내용이든 존재하고 있는 consumer가 consume하는 것들을 수정하지 않고 따라다닐 수 있다.

파티션의 로그는 몇가지 목표를 제공한다. 첫째, 그것들은 로그가 단일 서버에 맞도록 사이즈보다 크게 scale을 늘릴 수 있다. 각각의 파티션은 호스트를 하고 있는 서버에 맞추어야 하지만 topic은 많은 파티션을 가질 수 있으므로 임의의 양의 데이터를 다룰 수 있어야 한다. 두번째로 parallel하게 동작한다.

#### <u>Distribution</u>

로그의 파티션들은 카프카 클러스터에 있는 서버에 배포가 된다. 각각의 서버는 데이터를 다루고 파티션의 공유에 대한 요청을 한다. 각각의 파티션은 fault tolerance를 위해 구성 가능한 숫자의 서버들에 대해서 복제를 한다.

각각의 파티션은 leader라고 불리는 하나의 서버를 가지고 있고, 0개 이상의 follower를 가지고 있다. leader는 파티션에 대한 모든 읽기와 쓰기를 담당하는 반면 follower는 수동적으로 leader를 복제한다. 만약 leader가 죽는다면, 하나의 follower가 자동적으로 새로운 leader가 된다. 각각의 서버는 어떤 파티션에 대해서는 leader로 동작하고 또 다른 파티션에 대해서는 follower로 동작하여 클러스터 내에서 load balance가 적절히 이루어진다.

#### <u>Geo-Replication</u> // ??

Kafka MirrorMaker는 당신의 클러스터에 대해 geo-replication을 제공한다. MirrorMaker로 메시지를 다수의 데이터센터나 클라우드로 복제할 수 있다. 당신은 이것을 백업과 복구에 대한 상황에서 액티브/패시브로 사용할 수 있다. 또는 데이터를 당신의 유저에게 가깝게 위치시킬수도 있고, 각각의 요구에 대한 데이터를 제공할 수 있다.

