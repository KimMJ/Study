# 스프링과 Mybatis 연동

## 스프링 연동 설정

* `SqlSessionFactoryBean` 객체가 `SqlSession` 객체를 생산하려면 반드시 `DataSource` 와 `SQL Mapper` 정보가 필요하다.
* 따라서 `DataSource` 를 `Setter` 인젝션을 참조하고, `SQL Mapper` 가 등록된 `sql-map-config.xml` 파일도 `Setter` 인젝션으로 설정해야 한다.

## DAO 클래스 구현 - 방법1

* `SqlSessionDaoSupport` 클래스 상속
* 재정의한 `setSqlSessionFactory()` 메소드 위에 `@Autowired`를 붙여서 스프링 컨테이너가 `setSqlSessionFactory()` 메소드를 자동으로 호출하도록 함.
* 이때, 스프링 설정 파일에 `<bean>` 등록된 `SqlSessionFactoryBean` 객체를 인자로 받아 부모인 `SqlSessionDaoSupport` 에 `setSqlSessionFactory()` 메소드로 설정해준다.

## DAO 클래스 구현 - 방법2

* `SqlSessionTemplate` 클래스를 `<bean>` 등록하여 사용하는 것.
* 스프링 설정 파일에서 `SqlSessionTemplate` 클래스를 `SqlSessionFactoryBean` 아래에 `<bean>` 등록하는 것.
* 여기서 `SqlSessionTemplate` 클래스에는 `Setter` 메소드가 없어서 `Setter` 인젝션을 할 수 없다.
* 따라서 생성자 메소드를 이용한 `Constructor` 주입으로 처리해야한다.
* 그 후 DAO 클래스를 구현할 때, `SqlSessionTemplate` 객체를 `Autowired` 를 이용하여 의존성 주입 처리하면 `SqlSessionTemplate` 객체로 DB 연동을 처리할 수 있다.

## Dynamic SQL으로 검색 처리

* Mybatis는 SQL의 재사용성과 유연성을 향상하고자 Dynamic SQL을 지원한다.
