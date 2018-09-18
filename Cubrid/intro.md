# CUBRID란

2013년 기준

* 오픈소스
* NHN 개발 / cubrid.org
* 부담없는 라이센스 정책
  * Engine / Broker : GPL v2
  * Drivers / Tools : BSD
* NHN 서비스 적용 확대
  * 전체 DB 중 40% 이상이 CUBRID
  * 네이버 메일, N 드라이브, 네이버 사전 등에 적용
* 국내 / 해외 사용자 확대
  * 누적 다운로드 : 200,000 +
  * 월 평균 방문자 : 40,000 +
  * 유럽 / 러시아 / 미국 등 다수 해외사용자 확보

## CUBRID 특징

1. RDBMS 기본 기능 지원
  * SQL 92 표준 + SQL 확장 구문 지원
  * 다양한 데이터 타입 지원
  * 트랜잭션 지원 (ACID 보장 : commit, rollback, savepoint)
  * 다중 단위 잠금 : 테이블, 레코드 단위
2. 엔터프라이즈 기능 지원
  * HA (High Availability) : Master - Slave
  * DB 샤딩 (Sharding)
  * 테이블 분할 (Partitioning)
  * 온라인 / 오프라인 백업, 전체 / 증분 백업, 시점 복구 지원
3. 다양한 응용 환경 지원
  * JDBC, PHP, ODBC, OLEDB, ADO.NET, Python, Ruby, Node.js, C API
4. 다양한 CUBRID 편의 도구 지원
  * CM / CQB : 질의 실행 / 관리 유틸리티 실행
  * CMT : 원격 / 로컬 마이그레이션 실행
  * Web Manager : 설치패키지에 내장된 관리 도구
