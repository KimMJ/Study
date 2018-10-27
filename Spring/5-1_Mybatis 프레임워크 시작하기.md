# Mybatis 프레임워크 시작하기

## Mybatis 프레임워크 특징

장점
1. 한두 줄의 자바 코드로 DB 연동을 처리한다.
2. SQL 명령어를 자바 코드에서 분리하여 XML 파일에 따로 관리한다.

* 유지 보수관점에서 보면 DB 연동에 사용된 복잡한 자바코드는 중요해지지 않는다.
* 개발자는 실행되는 SQL만 관리하면 되며, Mybatis는 개발자가 이 SQL 관리에만 집중할 수 있도록 도와준다.
* Mybatis는 XML 파일에 저장된 SQL 명령어를 대신 실행하고 실행 결과를 VO 같은 자바 객체에 자동으로 매핑까지 해준다.
* Mybatis 프레임워크는 데이터 맵퍼(Data Mapper)라고 부른다.

## SqlSession 객체 생성하기

* Mybatis를 이용하여 DAO를 구현하려면 `SqlSession` 객체가 필요하다.
* 이를 얻기 위해서는 `SqlSessionFactory` 가 필요하다.

```java
Reader reader = Resources.getResourceAsReader("sql-map-config.xml");
SqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
```

* 우선 Mybatis 메인 설정 파일인 `sql-map-config.xml` 파일로부터 설정 정보를 읽어 들이기 위한 입력 스트림을 생성해야 한다.
* 그러고 나서 입력 스트림을 통해 `sql-map-config.xml` 파일을 읽어 `SqlSessionFactory` 객체를 생성한다.
