# JPA 개념

* ORM(Object-Relation Mapping)은 자바 객체에 저장된 데이터를 테이블의 Row 정보로 저장하고, 반대로 테이블에 저장된 Row 정보를 자바 객체로 매핑해준다.
* ORM 프레임워크의 가장 큰 특징이자 장점은 DB 연동에 필요한 SQL을 자동으로 생성한다는 것.
* DBMS가 변경되면 이 SQL 또한 변경됨.
* 단지 ORM 환경설정 파일 어딘가 DBMS가 변경되었다는 것만 수정해주면 됨.
* Hibernate 프레임워크는 완벽한 ORM 프레임워크이며 자바 객체와 테이블의 ROW를 매핑하는 역할을 수행

## JPA의 특징

* JPA는 모든 ORM 구현체들의 공통 인터페이스를 제공.
* 애플리케이션을 구현할 때, JPA API를 이용하면 ORM 프레임워크를 변경하기 쉽다.

## JPA 시작하기

* 일반적인 프로그램에서는 객체를 식별하기 위해서 유일 식별자를 사용하지는 않지만, 영속 객체가 테이블과 매핑될 때 객체 식별 방법이 필요하므로 유일 식별자를 소유하는 클래스로 작성한다.

### `persistence.xml` 파일 작성

* JPA는 `persistence.xml` 파일을 사용하여 필요한 설정 정보를 관리한다.

### 클라이언트 프로그램 작성

* 가장 먼저 영속석 유닛을 이용하여 `EntityManagerFactory` 객체를 생성한다.
* JPA를 이용하여 CRUD를 구현하려면 `EntityManger` 객체를 사용해야하는데, 이는 `EntityManagerFactory` 로부터 구할 수 있기 때문.
