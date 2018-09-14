# 스프링 JDBC

## 스프링 JDBC 개념

* JDBC는 자바에서 DB를 연동할 때 쓰는 기술
* 이것을 사용하려면 개발자가 작성해야 할 코드가 너무 방대해짐.
* 개발자가 실행되는 SQL 구문만 관리할 수 있도록 하는 것이 `JdbcTemplate` 클래스.

## JdbcTemplate

* 템플릿 메소드 패턴이 적용된 클래스.
* JDBC의 반복적인 코드를 제거하기 위해 사용.

## 스프링 JDBC 설정

* pom.xml파일에 DBCP 관련 dependency 추가.

### DataSource 설정

* `JdbcTemplate` 클래스가 JDBC API를 이용하여 DB연동을 처리하려면 반드시 데이터베이스로부터 커넥션을 얻어야 한다.
* `DataSource` 인터페이스를 구현한 클래스는 다양하지만 일반적으로 가장 많이 사용하는 `BasicDataSource` 객체는 연결에 필요한 property를 Setter 인젝션으로 설정.
* `BasicDataSource` 객체가 삭제되기 전에 연결을 해제하고자 할 경우 close()를 destroy-method속성으로 지정

### 프로퍼티 파일을 이용한 DataSource 설정

* `PropertyPlaceholderConfigurer` 를 이용하면 외부의 프로퍼티 파일을 참조하여 DataSource를 설정할 수 있음.

## `JdbcTemplate` 메소드

### `update()` 메소드

* `INSERT`, `UPDATE`, `DELETE` 구문을 처리할 때 `JdbcTemplate` 클래스의 `update()` 메소드 사용.
	1. SQL 구문에 설정된 "?" 수만큼 값들을 차례대로 나타내기
	2. Object 배열 객체에 SQL 구문에 설정된 "?" 수만큼의 값들을 세팅하여 배열 객체를 두 번째 인자로 전달하는 방식

### `queryForInt()` 메소드

* `SELECT` 구문으로 검색된 정수값을 리턴받을 때 `queryForInt()` 사용.

### `queryForObject()` 메소드

* `SELECT` 구문의 실행 결과를 특정 자바 객체로 매핑하여 리턴받을 때 `queryForObject()` 사용.
* 검색 결과를 자바 객체로 매핑할 `RowMapper` 객체를 반드시 지정해야 함.

### `query()` 메소드

* `queryForObject()`가 `SELECT` 문으로 객체 하나를 검색할 때 사용하는 메소드라면, `query()` 메소드는 `SELECT` 문의 실행 결과가 목록일 때 사용.
* `RowMapper` 객체 사용해야 함.

## DAO 클래스 구현

### 1. `JdbcDaoSupport` 클래스 상속

### 2. `JdbcTemplate` 클래스 `<bean>` 등록, 의존성 주입

* DAO 클래스에서 `JdbcTemplate` 객체를 얻는 두번째 방법은 `<bean>`에 등록하고 의존성 주입으로 처리하는 것.
* 일반적으로 이 방법 사용.
