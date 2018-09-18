# CUBRID HA

## HA(Hight Availability)

1. 고가용성 (무중단 서비스)
2. 읽기부하 분산 (대용량 처리)

### 특징

* 기본키 기반으로 마스터 DB로부터 복제 수행.
* 자동 failover 지원. 장애발생시 감지하여 자동으로 slave로 연결.
* 다양한 복제모드 설정 가능.

## HA 기본 구성 (DB 이중화)

* master - slave 로 구성.
* master는 Transaction 로그를 복사하여 slave에 전달.
* slave는 DB에 반영.
* master가 다운시 slave가 master가 됨. 이전 master는 복구시 slave로 복구됨.

### 브로커 이중화

* 브로커도 이중화를 해야 안전.
* althosts로 예비 브로커도 연결.
