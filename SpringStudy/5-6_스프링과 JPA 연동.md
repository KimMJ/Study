# 스프링과 JPA 연동

## 스프링과 JPA 연동 설정

* 스프링과 JPA 연동을 위해 가장 먼저 `JpaVendorAdapter` 클래스를 등록해야 한다.
* 이 클래스는 실제로 DB 연동에 사용할 JPA 벤터를 지정할 때 사용.
* 두번쨰로 등록할 클래스는 `EntityManagerFactoryBean` 이다.
* JPA를 이용하여 DAO 클래스를 구현하려면 최종적으로 `EntityManager` 객체가 필요.
* 이 `EntityManager` 객체를 생성하려면 `LocalContainerEntityManagerFactoryBean` 클래스를 `<bean>` 등록 해야함.

## 트랜잭션 설정 수정

* 트랜잭션 관리자를 `JpaTransactionManager` 로 변경.

## DAO 클래스 구현

* JPA를 단독으로 사용하지 않고 스프링과 연동할 때는 `EntityManagerFactory` 에서 `EntityManager` 를 직접 생성하는 것이 아니라 스프링 컨테이너가 제공하는 `EntityManager` 를 사용해야 한다.
* `@PersistenceContext` 는 스프링 컨테이너가 관리하는 `EntityManager` 객체를 의존성 주입할 때 사용하는 어노테이션.
* 스프링 컨테이너는 이 객체를 이용하여 `@PersistenceContext` 가 설정된 `EntityManager` 타입의 변수에 `EntityManager` 객체를 의존성 주입 시켜준다.
