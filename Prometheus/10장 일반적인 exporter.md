# 10장 일반적인 exporter

* exporter는 아무런 구성 없이 "작업만" 한다.
* 일반적으로 어떤 애플리케이션 인스턴스의 데이터를 수집할지 알려주기 위한 최소한의 설정이 익스포터에는 필요함.
* 반대의 경우 작업 중인 데이터가 매우 일반적이기 때문에 광범위한 구성이 필요한 익스포터도 있음.
* 익스포터가 필요한 모든 애플리케이션 인스턴스는 일반적으로 저마다 하나의 익스포터를 가짐
* 보편적인 프로메테우스 사용법은 모든 애플리케이션이 직접 계측한 다음, 프로메테우스가 이를 검색하고 직접 수집하는 것.
  * 이것이 불가능 할 때 exporter가 사용되며, 가능한 한 기존의 아키텍처를 유지함.

## Consul Exporter

* consul을 사용할 때 사용할 수 있는 exporter

## HAProxy Exporter

* 다음과 같이 haproxy.cfg를 작성한다.

  ```yaml
  defaults
    mode http
    timeout server 5s
    timeout connect 5s
    timeout client 5s
  
  frontend frontend
    bind *:1234
    use_backend backend
    
  backend backend
    server node_exporter 127.0.0.1:9100
    
  frontend monitoring
    bind *:1235
    no log
    stats uri /
    stats enable
  ```

* 이런 구성은 `http://localhost:1234`를 노드 익스포터로 프록시 처리한다.

* ```yaml
  frontend monitoring
    bind *:1235
    no log
    stats uri /
    stats enable
  ```

  이 설정을 통해 통계 보고 기능이 있는 HAProxy frontend가 `http://localhost:1235` 에서 수행되며, HAProxy exporter가 사용할 CSV 결과가 `http://localhost:1235/;csv`에 표시된다.

* haproxy가 정상적으로 동작하는지 확인할 수 있는 haproxy_up 메트릭이 있음.

* HAProxy는 frontend, backend, server로 구성되며 각각 `haproxy_frontend_`, `haproxy_backend_`, `haproxy_server_`를 prefix로 가지는 메트릭을 제공한다.

* backend에는 많은 서버가 있을 수 있기 때문에, `haproxy_server_` 메트릭의 cardinality는 문제를 야기할 수 있다.

  * `--haproxy.server-metric-fields` 플래그는 반환되는 매트릭을 제한할 수 있다.

* 프로메테우스가 HAProxy exporter를 수집하도록 구성할 수 있다.

  ```yaml
  global:
    scapre_interval: 10s
  scrape_configs:
    - job_name: haproxy
      static_configs:
        - targets:
          - localhost:9101
  ```

## Grok Exporter

* Grok Exporter는 로그를 메트릭으로 변환할 때 사용할 수 있다.
* grok은 일반적으로 Logstash(ELK에서 L)에서 사용하는 비정형적인 로그를 파싱하는 방법이다.
* grok exporter는 기존 패턴을 재사용하기 위해 동일한 패턴 언어를 재사용한다.

## 블랙박스

* 각 애플리케이션 인스턴스마다 하나의 익스포터가 실행되도록 배치하는 것을 권장하지만, 기술적인 이유로 불가능한 경우도 있다.
* 블랙박스 모니터링은 시스템 내부에 특별한 정보 없이 시스템 외부에서 모니터링하는 방법이다.

### ICMP

* ICMP는 Internet Control Message Protocol로 IP의 일부

### TCP

* TCP는 Transmission Control Protocol
* 블랙박스 익스포터는 실제로 발생한 내용에서 무엇이 잘못되었는지 확인하는 데 필요한 메트릭을 일부 제공함.

### HTTP

* HyperText Transfer Protocol
* probe_success 외에도 상태 코드, HTTP 버전 및 HTTP 요청의 여러 단계에 대한 타이밍 등 디버깅에 유용한 다수의 메트릭을 볼 수 있다.

### DNS

* 주로 DNS server를 테스트할 때 사용됨.
* dns server 테스트 외에도 dns 탐색을 이요해 DNS 해석을 통해서 특정 결과가 반환되는지 확인할 수 있다.

### 프로메테우스 구성

* 블랙박스 익스포터는 `/probe` endpoint에서 module과 target URL 파라미터를 사용한다.

