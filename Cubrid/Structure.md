# CUBRID 구조의 이해

## 3 Tier 구조

* Interface, Brokers, Database Server 3개로 나누어짐

### Interface

* 드라이버들.
* JDBC, Python, Node.js, PHP 등..

### Brokers

* Middleware
* Interface와 DB Server 사이에 위치
* Connection Pooling(커넥션 처리), Query Parsing (질의 파싱), Optimizing (옵티마이징), Logging (로깅)

### DB Server

* Brokers에서 만들어진 계획대로 질의를 실행.
* 데이터의 일관성을 위한 락을 관리
* Transaction 관리
* 로깅

## 주요 프로세스

### Brokers

#### 브로커 프로세스 (cub_broker)

* 응용의 요청을 받아 CAS에 작업 할당 / CAS 프로세스 관리

#### CAS 프로세스 (cub_cas)

* 응용의 요청을 받아 DB 서버에 처리 요청
* 데이터베이스와의 연결 유지
* 프로세스 단위로 작업 수행
* idle, busy, client_wait, close_wait 상태 있음.

### DB Server

#### 마스터 프로세스 (cub_master)

* CAS와 CSQL의 요청을 받아 cub_server로 전달하는 중계자
* HA 환경에서 heartbeat 메시지 확인

#### DB 서버 프로세스 (cub_server)

* 데이터베이스 볼륨을 관리하며 client의 요청을 처리
* 데이터베이스마다 1개의 cub_server가 구동됨.

### CUBRID Manager Server

#### cub_auto

* CM 사용자 인증
* CM 클라이언트 접속관리
* 예약작업 자동 실행

#### cub_js

* CM 클라이언트 요청 실행

## 질의 처리 과정

1. 응용에서 broker에게 질의처리 요청
2. 질의 처리할 CAS 할당
3. 데이터베이스에 질의 요청
4. 질의처리결과를 CAS에 전달
5. 질의처리 결과를 응용에 전달
