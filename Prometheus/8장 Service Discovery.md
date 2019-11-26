# 8장 Service Discovery

* `static_configs` 를 활용한 정적 구성을 통해 프로메테우스가 어떤 정보를 수집할 수 있는지 알아보았다.
* 여기서는 동적으로 프로메테우스가 정보를 수집할 수 있도록 설정하는 방법은 알아본다.
* Service Discovery를 이용하면 정보를 저장해둔 모든 데이터베이스로부터 프로메테우스로 해당 정보를 제공할 수 있음.
* 프로메테우스는 별도의 설정 없이 Consul, EC2, Kubernetes같은 다양한 서비스 소스와의 연동을 지원
* target label을 지정해두면 모니터링 대상을 이해하기 쉬운 체계로 그룹화하고 구성할 수 있음.
  * 동일한 역할을 수행하거나 동일한 환경에 있는, 또는 같은 팀에 의해 운용되는 대상을 하나로 묶을 수 있음.

## Service Discovery 메커니즘

* 이미 확보된 머신과 서비스에 대한 데이터베이스와 통합되도록 설계되었음.
* 시스템 전반에 걸쳐 나타나는 좀 더 일반적인 관심사항이다.
* 바람직한 서비스 검색 메커니즘은 metadata를 제공함.
  * metadata는 이름, 설명, 서비스를 보유한 팀, 구조화된 태그, 또는 모든 유용하게 활용될 수 있는 모든 사항을 의미
  * metadata는 target label로 변환 가능

### 정적 서비스 검색

* 정적 구성 : `prometheus.yml` 파일에 직접 대상 정보를 입력하는 것.

### 파일 기반 서비스 검색

* 네트워크를 사용하지 않음
* 로컬 파일 시스템에 저장된 파일에서 모니터링 대상 정보를 읽어옴.
* 파일 경로는 상대경로.

### 컨설 (Consul)

* 네트워크를 사용하는 서비스 검색 메커니즘

### EC2

* 프로메테우스에서 추가 설정 없이 서비스 검색을 이용할 수 있는 cloud provider

## relabeling

* target과 target에 대한 metadata가 원시적으로 표시되고있음.
* relabeling을 통해 metadata를 대상과 매핑하는 방법을 알려줄 수 있음.
* 이상적인 환경에서는 새로운 머신과 애플리케이션이 자동으로 선택되고 모니터링되도록 서비스 검색과 레이블 재지정을 구성할 수 있음.

### 수집할 대상 선택하기

* 가장 먼저 구성해야할 사항은 어떤 대상에 대한 정보를 실제로 수집할지 결정하는 것.
* `relabel _configs` 를 통해 수집할 대상을 선택할 수 있음.
* 일치하지 않는 대상을 삭제하는 `keep` 동작 외에 `drop` 동작은 일치하는 대상을 제외함.
* `|` 은 두개 이상을 표현함.
* `;` 은 각 label을 연결시킴.

### target label

* target label은 수집을 통해 반환되는 모든 time series의 label에 추가되는 label임.
* target label은 수집 대상을 구분하며, 머신 소유자나 버전 번호처럼 시간이 지나면서 변경되어서는 안됨.
* 궁극적으로 target label은 PromQL에서 대상을 선택해 그룹화하고 집계할 수 있음.

#### replace 동작

* target label을 지정하기 위해 relabeling을 하려면 replace로 하면 된다.

#### job, instance, \__address__

#### labelmap 동작

* `drop`, `keep`, `replace` 동작과는 달리 label value가 아닌 label name에 적용됨.

#### 리스트

* 리스트에 있는 각 아이템들을 `,` 로 연결하고, 이렇게 연결된 아이템들을 레이블 값으로 사용할 수 있음.

## 수집 방법

```yml
scrape_configs:
  - job_name: example
    consul_sd_configs:
      - server: 'localhost:8500'
    scrape_timeout: 5s
    metrics_path: /admin/metrics
    params:
      foo: [bar]
    scheme: https
    tls_config:
      insecure_skip_verify: true
    basic_path:
      username: brian
      password: hunter2
```

* `metrics_path` 는 유일하게 URL 형식
* `scheme` 은 http나 https로 설정

### metric_relabel_configs

* 메트릭의 relabeling : 대상으로부터 수집된 time series에 relabeling을 적용.

#### labeldrop과 labelkeep 동작

* 계측 레이블을 대상 레이블과 혼동해 수집 대상 레이블이 되어야 하는 내용을 반환하는 경우에 사용.
* `labelmap` 과 유사하게 label name에 적용됨.

#### label 충돌과 honor_labels

* 수집 시 계측 레이블과 동일한 이름을 가지는 레이블이 있다면 대상 레이블이 우선시됨.
  * `job` 레이블에 충돌이 발생하면 계측 레이블은 `exported_job` 으로 rename
  * 계측 레이블이 우선시되게 하고 싶으면 `honor_labels: true` 로 설정하면 됨.



