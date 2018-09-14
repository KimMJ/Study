# Mapper XML 파일 설정

## SQL Mapper XML 기본 설정

* `SqlMapConfig.xml` 파일은 Mybatis 메인 환경설정 파일.
  * 이 파일을 읽어들여 어떤 DBMS와 커넥션을 맺을지, 어떤 SQL Mapper XML 파일들이 등록되어 있는지 알 수 있다.
* Mybatis는 `SqlMap.xml` 파일에 등록된 각 SQL 명령어들을 Map 구조로 저장하여 관리한다.
* SQL이 실행될 때 필요한 값은 input 형태의 데이터로 할당하고, 실행된 SQL이 SELECT 구문일 때는 output 형태로 데이터를 리턴한다.

### Mapper XML 파일 구조

* Mybatis 프레임워크에서 가장 중요한 파일은 SQL 명령어들이 저장되는 SQL Mapper XML 파일.

### `<select>` 엘리먼트

* `SELECT` 구문을 작성할 때 사용.
* `parameterType` 과 `resultType` 속성을 사용할 수 있음.
  * `parameterType` 은 SQL 실행에 필요한 데이터를 외부로부터 가져오는 것.
  * `resultType` 은 검색관련 SQL 구문이 실행되면 `ResultSet` 이 리턴되며, `ResultSet` 에 저장된 검색 결과를 어떤 자바 객체에 매핑할지 지정.

### `<insert>` 엘리먼트

* `INSERT` 구문을 작성할 때 사용.
* `<selectKey>` 엘리먼트를 사용할 수 있음.
  * `<selectKey>` 는 생성된 키를 쉽게 가져올 수 있다.

### `<update>` 엘리먼트

* `UPDATE` 구문을 작성할 때 사용.

### `<delete>` 엘리먼트

* `DELETE` 구문을 작성할 때 사용.

## SQL Mapper XML 추가 설정

### `resultMap` 속성 사용

* 검색 결과를 `parameterType` 속성으로 매핑할 수 없는 몇몇 사례가 있음.
* 이럴 때 `resultMap` 속성을 사용하여 처리.

### `CDATA Section` 사용

* XML의 고유 문법으로서, CDATA 영역에 명시된 데이터는 단순한 문자 데이터이므로 XML 파서가 해석하지 않도록 한다.

## Mybatis JAVA API

### `SqlSessionFactoryBuilder` 클래스

* Mybatis로 DAO 클래스의 CRUD 메소드를 구현하려면 Mybatis에서 제공하는 `SqlSession` 객체를 사용해야 함.
* 이 객체는 `SqlSessionFactory` 로부터 얻어야 함.
* `SqlSessionFactory` 는 `SqlSessionFactoryBuilder` 의 `build()` 메소드를 이용.
* `build()` 는 Mybatis 설정파일 (`sql-map-config.xml`) 을 로딩하여 `SqlSessionFactory` 객체를 생성.
* 이때, 로딩하기 위해서 스트림인 `Reader` 객체가 필요

### `SqlSessionFactory` 클래스

* `SqlSession` 객체에 대한 공장 역할.
* `openSession()` 이라는 메소드를 제공하여 이 메소드를 통해서 `SqlSession` 객체를 얻을 수 있음.

### 유틸리티 클래스 작성

### `SqlSession` 객체

* `selectOne()` 메소드
  * 오직 하나의 데이터를 검색하는 SQL 구문을 실행할 때 사용.
  * 두개 이상의 데이터는 예외 발생.
* `selectList()` 메소드
  * 여러개의 데이터가 검색되는 SQL 구문을 실행할 때 사용.
* `insert()`, `update()`, `delete()` 메소드
  * 각각 `INSERT`, `UPDATE`, `DELETE` 구문을 실행할 때 사용.
  * 몇개의 데이터가 처리되었는지 리턴.
