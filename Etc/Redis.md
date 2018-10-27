# RequestDispatcher

## Redis란?

* 오픈소스
* 데이터베이스(NoSQL DBMS)로  분류되기도 하고 Memcached같은 인메모리 솔루션으로 분류되기도 함.
* 데이터 저장소로 가장 입/출력이 빠른 메모리를 채택
* 단순한 구조의 데이터 모델인 key-value 방식을 통해 빠른 속도를 보장
* 캐시 및 데이터스토어에 유리
* 다양한 API 지원
* 급격히 사용자가 집중되는 상황이나 대규모의 확장이 예정되어 있는 환경에 적합.
* global cache 방식이 적용되어 was 인스턴스 확장에는 유리하지만 cache 및 redis 세션 등 관리 포인트가 늘어난다는 단점이 있음

## Redis 특징

### key-value 저장방식

* 특정 키값을 저장하는 방식으로 구성.
* PUT/GET 명령어 지원
* 데이터는 메모리에 존재해서 write와 read의 속도가 매우 빠름.

### 다양한 데이터 타입

* memcached는 단순하게 key-value를 저장하는 방식.
* redis는 value가 단순한 Object가 아니라 자료구조를 가진다.

#### 지원 대이터 형

1. String
  * 일반적인 문자열. 최대 512Mb까지 지원.
  * Text 문자열 뿐 아니라 Integer나 JPEG 같은 Binary File까지 지원
2. Set
  * String의 집합.
  * 여러개의 값을 하나의 Value에 넣을 수 있음.
  * Set간의 연산 지원 (교집합, 차집합, 차이)
3. Sorted set
  * set에 score라는 필드가 추가된 데이터형.
  * Sorted set에서는 데이터를 오름차순으로 내부 정렬.
  * score 값의 범위에 따른 쿼리 (range query), top rank에 따른 쿼리 등 가능
4. hashes
  * value 내에 field/string value 쌍으로 이루어진 테이블을 저장하는 데이타 구조체.
  * RDBMS에서 PK 1개와 String 필드 하나로 이루어진 테이블같은 것.
5. list
  * String들의 집합으로 저장되는 데이터 형태는 set가 유사하지만, 일종의 양방향 linked list.
  * list 앞과 뒤에서 PUSH/POP 연산을 이용해서 데이터를 넣거나 뺄 수 있다.
  * 지정된 INDEX 값을 통해서 지정된 위치에 데이터를 넣거나 뺄 수 있다.
