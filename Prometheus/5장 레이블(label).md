# 5장 레이블 (label)

프로메테우스의 핵심과도 같다.

## 레이블의 정의

time series와 연관된 key-value 쌍이다. 메트릭 이름뿐 아니라 time series를 고유하게 식별한다.

HTTP와 관련된 메트릭을 뽑을 때 경로를 쓰고 싶은 경우가 있을 것이다. 하지만 이는 잠재적으로 많은 자원을 사용하는 일부 유형을 모든 메트릭 이름에 매칭시켜야 하기 때문에 안티패턴이다. 대신에 이런 상황을 위해서 프로메테우스는 label을 사용한다.

```
http_requests_total{path="/login"}
http_requests_total{path="/logout"}
http_requests_total{path="/adduser"}
http_requests_total{path="/comment"}
http_requests_total{path="/view"}
```

위와 같이 작성하면 하나의 http_requests_total 메트릭으로 작업할 수 있다. 따라서 전체 경로에 대한 특정 경로의 비율을 확인할 수 있다.

메트릭은 label을 하나 이상 가질 수 있다. 그리고 순서가 없다.

## 계측 레이블과 대상 레이블 (instrumentation label, target label)

* PromQL에서 작업할 때에는 둘간의 차이는 없다.
  * 하지만 label로 최상의 효과를 얻으려면 구분지어야 한다.
* instrumentation label
  * 애플리케이션이나 라이브러리 내부에서 알 수 있는 내용을 담고 있음.
* target label
  * 프로메테우스가 데이터를 수집하는 특정 모니터링 대상을 식별함.
  * 아키텍쳐와 관련이 깊음.
  * 프로메테우스는 target label을 데이터 수집 과정의 일부로 추가할 수 있음.
  * Service Discovery, Relabelling에서 유래.

## 계측 (instrumentation)

계측에서 메트릭을 사용하는 경우, label 값에 대한 인자와 더불어 반드시 labels 메소드에 대한 호출을 추가해야한다.

```python
import http.server
from prometheus_client import start_http_server, Counter

REQUESTS = Counter('hello_worlds_total',
					'Hello Worlds requested.',
					labelnames=['path'])

class MyHandler(http.server.BaseHTTPRequestsHandler):
  def do_GET(self):
    REQUESTS.labels(self.path).inc()
    self.send_response(200)
    self.end_headers()
    self.wfile.write(b"Hello World")
    
if __name__ == "__main__":
  start_http_server(8000)
  server = http.server.HTTPServer(('localhost', 8081), MyHandler)
  server.serve_forever()
```

* 레이블 이름은 첫 글자가 문자로 시작되어야하고 `:`을 쓰면 안된다.
* 메트릭 이름과 달리 레이블 이름은 네임스페이스에 속하지 않는다.
* 계측 레이블을 정의할 때는 `env`, `cluster`, `service`, `team`, `zone`, `region` 같은 이름은 피한다.
* `type` 은 너무 일반적이므로 사용하지 않는다.
* snake_case 사용

### 메트릭

메트릭은 다음과 같은 뜻을 가질 수 있다.

* metric family
* metric child
* time series

```
# HELP latency_seconds Latency in seconds.
# TYPE latency_seconds summary
latency_seconds_sum{path="/foo"} 1.0
latency_seconds_count{path="/foo"} 2.0
latency_seconds_sum{path="/bar"} 3.0
latency_seconds_count{path="/bar"} 4.0
```

* `latency_seconds_sum{path="/bar"}` 는 이름과 레이블로 구분되는 time series이며, PromQL과 함께 동작한다.
* `latency_seconds{path="/bar"}` 는 차일드이며, 파이썬 클라이언트에서 labels()의 반환값을 나타낸다.
  * summary의 경우에 차일드는 이런 레이블과 함께 _sum과 _count time series를 포함한다.
* `latency_seconds` 는 메트릭 패밀리로 메트릭 이름 혹은 그와 관련된 타입이며 클라이언트 라이브러리를 사용하는 경우의 메트릭 definition이다.
* label이 없는 gauge 메트릭의 경우 matric family, matric child, time series는 모두 동일함.

### 다중 레이블

메트릭을 정의할 때 레이블 개수를 지정하고 레이블 호출 시 같은 순서로 값을 지정할 수 있다.
다음의 예시에서는 hello_worlds_total이 path, method 두가지 label이 있다.

```python
REQUESTS = Counter('hello_worlds_total',
        'Hello Worlds requested.',
        labelnames=['path', 'method'])

class MyHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        REQUESTS.labels(self.path, self.command).inc()
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello World")
```

* 중요한 메트릭을 갖고 작업하면 관련 레이블이 무엇인지 알 수 있다.
* 직접 계측을 수행하는 경우에도 사전에 레이블 이름을 알고 있어야 함.

### 차일드 (Child)

* 파이썬에서는 label 메소드의 반환 값을 child라고 부름.
* 객체가 메트릭에 대해 하나의 차일드만 참조하는 경우 labels를 한번 호출한 다음 해당 객체에 저장하는 것이 일반적이다.
* 차일드는 labels를 호출한 후 /metrics에만 표시됨.
  * 따라서 사라질 수 있기 때문에 PromQL에서 작업시 어려움.
  * 가능하다면 시작 시점에 차일드를 초기화시켜줌.

## 집계 (aggregation)

PromQL에서 레이블을 사용해보자.

**input:**

```
rate(hello_worlds_total[5m])
```

**result:**

```
{job="myjob",instance="localhost:1234",path="/foo",method"GET"} 1
{job="myjob",instance="localhost:1234",path="/foo",method"POST"} 2
{job="myjob",instance="localhost:1234",path="/bar",method"GET"} 4
{job="myjob",instance="localhost:5678",path="/foo",method"GET"} 8
{job="myjob",instance="localhost:5678",path="/foo",method"POST"} 16
{job="myjob",instance="localhost:5678",path="/bar",method"GET"} 32
```

path label로 집계를 해본다. sum은 합계, without은 제거할 label을 나타낸다.

**input:**

```
sum without(path)(rate(hello_worlds_total[5m]))
```

**result:**

```
{job="myjob",instance="localhost:1234",method="GET"} 5
{job="myjob",instance="localhost:1234",method="POST"} 2
{job="myjob",instance="localhost:5678",method="GET"} 40
{job="myjob",instance="localhost:5678",method="POST"} 16
```

두개의 label을 without에 포함시킬 수 있다.

**input:**

```
sum without(path,instance)(rate(hello_worlds_total[5m]))
```

**result:**

```
{job="myjob",method="GET"} 45
{job="myjob",method="POST"} 18
```

label은 정렬되지 않기 때문에 method label도 삭제 가능하다.

**input:**

```
{job="myjob",path="/foo"} 27
{job="myjob",path="/bar"} 36
```

지정한 레이블만 유지하는 `by` 도 있다. 모든 job이 공유하는 env나 region 같은 추가 레이블이 있는 경우에는 `without` 을 더 선호한다.

## 레이블 패턴 (label pattern)

* 프로메테우스에서는 time series로 string 같은 타입은 지원하지 않고 64bit float만 지원한다.
* 하지만 label은 문자열이다.
  * 이에 대한 몇가지 사용 예가 있다.

### enum

* 문자열을 사용하는 하나의 일반적인 경우

* 예시

  * STARTING, RUNNING, STOPPING, TERMINATED를 가지는 enum type

  * 이 때 순서대로 0, 1, 2, 3

  * 이 때 PromQL에서 작업하기는 조금 까다롭다. 숫자를 사용하는 것은 부정확하다.

  * 자원이 STARTING일 때 사용한 시간에 비율을 알고싶을 때 쓸 수 있는 표현식이 없다.

  * 따라서 child가 될 수 있는 각각의 경우의 수에 대해서 gauge에 대한 label을 추가하는 방식으로 한다.

  * 이 때 참인 경우는 1일 것이고 아닌 경우는 모두 0이 될 것이다.

    ```
    # HELP gauge The current state of resources
    # TYPE gauge resource_state
    resource_state{reousrce_state="STARTING",resource="blaa"} 0
    resource_state{reousrce_state="RUNNING",resource="blaa"} 1
    resource_state{reousrce_state="STOPPING",resource="blaa"} 0
    resource_state{reousrce_state="TERMINATED",resource="blaa"} 0
    ```

  * `avg_over_time(resource_state[1h])` 는 각 상태에서 사용한 시간의 비율을 제공한다.

  * `sum without(resource)(resource_state)` 를 사용하여 resource_state에 따라 집계 가능하다.

* 이런 메트릭 생성에 `set` 을 사용할 수도 있지만 경쟁 조건을 발생시킬 수 있다.

  * 이에 대한 해결책은 Custom Collector를 사용하는 것이다.

* 다른 레이블의 개수와 결합 상태의 개수가 너무 많으면 성능 문제가 발생할 수 있다.

### info

* machine roles approach 라고도 부름.
* 1의 값을 가진 gauge와 target의 label로 annotation하고 싶은 모든 문자열을 사용하는 것이 규칙이다.
* gauge는 suffix로 _info를 사용해야함.
* info 메트릭은 곱셈 연산자와 group_left 변경자를 이용하여 다른 모든 메트릭과 join 할 수 있다.
  * 메트릭 조인 시 어떤 연산자든 사용할 수 있지만, 정보 메트릭의 값은 1이기 때문에 다른 메트릭의 값을 변경시키지 않는다.

## label 사용 시점

* 메트릭이 유용해지기 위해서는 어떻게든 집계를 할 수 있어야 한다.

* 메트릭의 합계나 평균을 구하는 것이 의미있는 결과를 도출하는데 기본적인 규칙이다.

* label을 사용하면 안되는 경우

  * PromQL에서 메트릭을 사용할 때마다 해당 label을 지정해야 한다면 계측 label은 적절하지 않다.

    * 오히려 label을 사용하는 대신 매트릭 이름으로 이동시켜야 한다.

  * 나머지 메트릭의 전체 합인 time series를 만드는 경우.

    ```
    some_metric{label="foo"} 7
    some_metric{label="bar"} 13
    some_metric{label="total"} 20
    ```

    ```
    some_metric{label="foo"} 7
    some_metric{label="bar"} 13
    some_metric{} 20
    ```

    * 이 경우 둘 다 두번의 계산이 있기 때문에 PromQL에서 sum에 대한 집계가 이상하게 된다.

### Cardinality

* time series나 monitoring이 많다고 좋은 것이 아님.
* Cardinality : 프로메테우스에서 확보한 time series의 갯수
* 일반적으로 사용자로부터 오는 증가된 트래픽은 더 많은 애플리케이션 인스턴스를 의미함.
* 일반적으로 프로메테우스에서 가장 큰 10개의 메트릭은 자원 사용량의 절반 이상을 차지함.
  * 대부분은 label cardinality 때문에 발생

