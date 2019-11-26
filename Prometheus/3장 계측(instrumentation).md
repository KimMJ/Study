# 3장 계측(instrumentation)

프로메테우스에서는 직접 계측 (direct instrumentation)과 client library를 사용해 애플리케이션을 계측할 수 있다.

## 간단한 예제 프로그램

간단하게 http 서버를 만듦.

```python
import http.server
from prometheus_client import start_http_server

class MyHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(self):
    self.send_response(200)
    self.end_headers()
    self.wfile.write(b"Hello World")
    
if __name__ == "__main__":
  start_http_server(8000)
  server = http.server.HTTPServer(('localhost', 8001), MyHandler)
  server.serve_forever()
```

* start_http_server(8000)은 8000포트에서 프로메테우스 메트릭을 제공.

(prometheus.yml)

```yaml
global:
  scrape_interval: 10s
scrape_configs:
- job_name: example
  static_configs:
  - targets:
    - localhost:8000
```

* 해당 포트로 접속하여 매트릭을 가져옴.

## 카운터

* 계측시 가장 자주 사용
* 이벤트 개수나 크기를 추적.

```yaml
from prometheus_client import Counter

REQUESTS = Counter('hello_worlds_total', 'Hello Words requested')

class MyHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(self):
    REQUESTS.inc()
    self.send_response(200)
    self.end_headers()
    self.wfile.write(b"Hello World")
```

코드는 세부분으로 이루어짐

* 임포트 (Import)
  * prometheus_client에서 Counter 클래스를 임포트
* 메트릭 정의 (Metric definition)
  * 프로메테우스에서 메트릭은 사용 전에 정의가 되어야 함.
  * 여기에서는 hello_worlds_total이 카운터.
  * 이 카운터의 도움말이 'Hello Worlds requested'
  * 도움말이 /metric 페이지에 표시됨.
* 계측 (Instrumentation)
  * inc 메소드는 카운터를 1 증가시킴.

위와 같이 설정할 경우 URL에 접속시마다 카운터가 1씩 증가한다. 이를 PromQL로 `rate(hello_worlds_total[1m])`을 통해 사용할 수 있다.

### 카운팅 예외 처리

```python
import random
from prometheus_client import Counter

REQUESTS = Counter('hello_worlds_total', 'Hello Words requested')
EXCEPTIONS = Counter('hello_worlds_exceptions_total', 'Exceptions serving Hello World.')

class MyHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(self):
    REQUESTS.inc()
    with EXCEPTIONS.count_exceptions():
      if random.random() < 0.2:
        raise Exception
    self.send_response(200)
    self.end_headers()
    self.wfile.write(b"Hello World")
```

count_exceptions는 예외처리 사항이 발생하면 예외를 전달한다. 따라서 애플리케이션 로직을 방해하지는 않는다. 이를 `rate(hello_world_exceptions_total[1m])/rate(hello_worlds_total[1m])`

```python
EXCEPTIONS = Counter('hello_worlds_exceptions_total', 'Exceptions serving Hello World.')

class MyHandler(http.server.BaseHTTPRequestHandler):
  @EXCEPTIONS.count_exceptions()
  def do_GET(self):
    ...
```

### 크기 계산

* 값을 표현하기 위해 64비트 부동소수점 방식의 숫자를 이용함.
* 증가하는 크기는 1이 아니어도 상관 없음. 음수는 안됨.
* 따라서 카운터가 0으로 변경되는 것은 애플리케이션이 재시작하는 경우임.

## 게이지 (gauge)

* 현재 상태에 대한 스냅샷
* 게이지의 실제값이 중요.

### 게이지 사용

* inc, dec, set 메소드 사용 가능.

> 메트릭 접미어 (suffix)
>
> 메트릭 단위 + _total, _count, _sum, _bucket로 접미어를 설정하는 것이 convention.
> ex) `myapp_requests_processed_bytes_total`

### Callback

set_function 메소드는 게시 시점에 호출되어야 하는 함수를 지정할 수 있다.

## Summary

* 시스템의 성능을 이해하려고 할 때, 애플리케이션이 요청에 응답하는 데 걸리는 시간이나 백엔드의 대기 시간은 중요한 메트릭이다.

* Summary는 observe라는 메소드를 통해 이벤트의 크기를 넘겨줌. time.time()으로 대기시간도 추적 가능.

  ```python
  import time
  from prometheus_client import Summary
  
  LATENCY = Summary('hello_world_latency_seconds', 'Time for a request Hello World')
  
  class MyHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
      start = time.time()
      self.send_response(200)
      self.end_headers()
      self.wfile.write(b"Hello World")
      LATENCY.observe(time.time() - start)
  ```

## 히스토그램

* quantile(분위수)는 특정 이벤트의 비율이 주어진 값보다 크기가 작다는 것을 의미.
  * 300밀리초의 0.95 quantile은 95%가 300밀리초 미만의 시간이 걸린다는 것을 의미.
* Histogram을 통해 bucket을 생성하여 quantile 계산 가능.

### Bucket

* 기본 bucket은 1밀리초에서 10초의 범위를 다룸.
* 10개정도의 버켓이면 충분.
* 메트릭 기반은 100% 정확한 quantile을 보여주지는 않음.

quantile은 최종 사용자 경험을 기술하는 데에는 적합하지만 디버깅에 이용하기는 어려움. 대기시간 문제는 quantile보다는 평균으로 디버깅 하는 것이 권장됨.

## 단위 테스팅 계측

* 시간이 지나며 코드가 변경됨에 따라 실수로 코드가 망가지는 것을 방지해줌.
* 대부분 트랜잭션 로그의 로그 문장만 단위 테스트를 하고, 때때로 요청 로그만 테스트를 할 것임.

## 계측 적용 방법

### 계측 대상

#### 서비스 계측

* 온라인 서비스 시스템
  * 사람이나 다른 서비스가 응답을 기다리는 시스템
  * 웹서버, 데이터베이스
  * request rate, latency, error rate. -> RED 메소드
* 오프라인 서비스 시스템
  * 응답을 기다리는 대기자가 없음.
  * 작업을 일괄처리하고, 여러 단계를 가지는 파이프라인이 있음.
  * Utilisation, Saturation, Error (USE) 메소드
* 일괄처리 작업 (batch job)
  * 정기적으로 실행되지만 항상 실행되는 것은 아님.
    * 따라서 데이터 수집은 잘 동작하지 않음.
  * Pushgateway같은 기법들 사용.
  * batch job의 마지막에 실행하는 데 걸린 시간, 작업의 각 단계에 걸린 시간, 작업이 마지막으로 성공한 시간이 기록되어야 함.

#### 라이브러리 계측

* 서비스 보다 작은 서비스인 라이브러리에 대한 계측.
* 요청과 대기시간, 오류같은 동일한 메트릭으로부터 이득을 얻음.

### 적정한 계측 빈도

* 프로메테우스는 처리할 수 있는 메트릭의 개수에 제한이 있음
* 모든 기능이 지속되는 동안 메트릭을 자동으로 추가하면 빠르게 추가될 수 있다.
* 카디널리티 (Cardinality)에 대한 고민 필요

### 매트릭 명명 규칙

일반적으로는 `library_name_unit_suffix` 형태를 띔.

#### 문자

* 메트릭 이름은 문자로 시작해야함. 그 뒤에는 상관 없음.
* `:`은 recording rule에서 사용하므로 쓰면 안됨.
* `_`로 시작하는 메트릭 이름은 프로메테우스 내부 사용을 위해 예약되어있음.

#### snake_case

* convenction에서 snake_case 사용하도록 가이드.

#### metric suffix

* counter, summary, histogram 메트릭에 대해 `_total`, `_count`, `_sum`, `_bucket` suffix 사용됨.
* counter에는 그 앞에 _total까지 붙임.

#### 단위

* prefix가 없는 단위를 사용하는 것이 좋다.
* 단위를 메트릭 이름에 넣어주기도 함.
* 갯수를 사용한다고 count를 단위로 쓰면 안된다.

#### 이름

* 관련 메트릭에 대해서는 같은 prefix를 두어서 관계가 쉽게 파악되도록 해야함.
* 이름은 왼쪽에서 오른쪽으로 갈수록 구체적이어야 함.
* 메트릭 이름에 label을 넣어서는 안됨.
  * PromQL로 label에 대해 집계할 때 올바르게 집계되지 않음.
* direct instrumentatin을 구현하는 경우 메트릭이나 메트릭 이름을 절차적으로 생성해서는 안된다.

#### 라이브러리

* 메트릭 이름은 전역 namespace이기 때문에 라이브러리 사이의 충돌을 피하고 메트릭이 어디에서 온 것인지를 표시하는 것이 중요함.
* 혼동을 피하기 위해서는 메트릭 이름의 라이브러리 부분은 확실하게 구분하여야 한다.

애플리케이션의 메트릭 이름 앞에 애플리케이션의 이름을 붙여서는 안 된다. 각기 다른 애플리케이션에서 메트릭을 구별하는 방법은 메트릭 이름이 아닌 대상 레이블 (target label)을 사용하는 것이다.

