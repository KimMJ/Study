# JPA 환경설정

* VO 객체와 테이블을 매핑하기 위한 다양한 설정 정보를 `persistence.xml` 파일로 환경설정 사용.
* `persistence.xml` 파일은 `<persistence>` 를 루트 엘리먼트로 사용하며, 영속성 유닛과 관련된 다양한 정보가 설정된다.

## 영속성 유닛 설정

* 영속성 유닛은 연동할 데이터베이스당 하나씩 등록
* 영속성 유닛에 설정된 이름은 나중에 DAO 클래스를 구현할 때 `EntityManagerFactory` 객체 생성에 사용된다.
* 가장 먼저 영속석 유닛을 이용하여 `EntityManagerFactory` 객체를 생성한다.
* JPA를 이용하여 CRUD를 구현하려면 `EntityManager` 객체를 사용해야하는데, 이는 `EntityManagerFactory` 로부터 구할 수 있기 때문.

### 영속성 유닛 프로퍼티 설정

* DB 커넥션을 설정하는 것.
* 이 설정을 바탕으로 Hibernate같은 JPA 구현체가 특정 데이터베이스와 커넥션을 맺을 수 있음

### Dialect 클래스 설정

* JPA는 특정 DBMS에 최적화된 SQL을 제공하기 위해서 DBMS마다 다른 Dialect 클래스를 제공.
* DBMS가 변경되는 경우 Dialect 클래스만 변경하면 SQL이 자동으로 변경되어 생성되므로 유지보수가 좋아짐.

## 엔티티 클래스 기본 매핑

* JPA의 기본은 엔티티 클래스를 기반으로 관계형 데이터베이스에 저장된 데이터를 관리하는 것.
* 엔티티 매핑에서 가장 복잡하고 중요한 설정은 연관 매핑 설정.

### `@Entity, @Id`

* `@Entity` 는 특정 클래스를 JPA가 관리하는 엔티티 클래스로 인식하는 가장 중요한 어노테이션.
* 엔티티 클래스 선언부 위에 `@Entity` 를 붙이면 JPA가 이 클래스를 엔티티 클래스로 인식하여 관련된 테이블과 자동으로 매핑 처리
* 엔티티 클래스와 매핑되는 테이블은 각 ROW를 식별하기 위한 PK(Primary Key) 칼럼을 가지고 있다.
* 이것과 매핑되는 엔티티 클래스 역시 PK와 매핑되는 변수가 필요하다.
* 이 변수를 식별자 필드라고 하고, 이 식별자 필드는 엔티티 클래스라면 무조건 가지고 있어야 하며, `@Id` 를 이용하여 선언한다.
* `@Entity` 를 사용하면 클래스의 이름과 같은 테이블이 매핑이 된다. 만약 다른 테이블과 매핑하려면 `@Table` 을 사용해야 한다.

### `@Table`

* 엔티티 클래스 이름과 테이블 이름이 다른 경우에 사용.
* 다른 `name`, `catalog`, `schema`, `uniqueConstraints` 속성들도 있음.

### `@Column`

* `@Column` 은 엔티티 클래스의 변수와 테이블의 칼럼을 매핑할 때 사용.
* 일반적으로는 엔티티 클래스의 변수 이름과 칼럼 이름이 다를 때 사용.
* 생략시 기본적으로 같은 칼럼이름을 매핑.
* 다양한 속성들이 존재하지만 보통 `name`, `nullable` 속성을 사용.

### `@GeneratedValue`

* `@GeneratedValue` 는 `@Id` 로 지정된 식별자 필드에 PK 값을 생성하여 저장할 때 사용.

### `@Transient`

* 엔티티 클래스의 변수들은 대부분 테이블의 칼럼과 매핑이 된다.
* 그러나 몇몇 변수는 매핑되는 칼럼이 없거나 아예 매핑에서 제외해야만 하는 경우가 있다.
* `@Transient` 는 엔티티 클래스 내의 특정 변수를 영속 필드에서 제외할 때 사용.

### `@Temporal`

* `@Temporal` 은 `java.util.Date` 타입의 날짜 데이터를 매핑할 때 사용.

## JPA API

* 애플리케이션에서 JPA를 이용하여 CRUD 기능을 처리하려면 `EntityManager` 객체를 사용해야함.
* 이 `EntityManager` 객체는 `EntityManagerFactory` 로부터 얻어냄.
* `EntityManagerFactory` 객체를 생성할 때는 영속성 유닛이 필요하므로 `persistence.xml` 파일에서 설정한 영속성 유닛 이름을 지정하여 `EntityManagerFactory` 객체를 생성한다.

```java
//EntityManager 설정
EntityManagerFactory emf = persistence.createEntityManagerFactory("JPAProject");
EntityManager em = emf.createEntityManager();
```

* `EntityManagerFactory` 객체로부터 `EntityManager` 객체를 얻었으면, `EntityManager` 를 통해서
`EntityTransaction` 객체를 얻을 수 있다.
* 이 `EntityTransaction` 을 통해서 트랜잭션을 제어할 수 있다.

### JPA API 사용

* 단순히 엔티티 객체를 생성하고 여기에 값을 저장했다고 해서 이 객체가 테이블과 자동으로 매핑이 되지는 않음.
* `EntityManager` 의 `persist()` 메소드로 엔티티 객체를 영속화해야만 `INSERT` 작업이 처리됨.
