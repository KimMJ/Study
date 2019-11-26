# 13장 PromQL 활용

* label은 PromQL의 핵심.
* PromQL로 임의의 집계를 수행할 수 있을 뿐 아니라 레이블에 대한 산술 연산을 위해 서로 다른 메트릭을 함께 조인할 수 있다.

## 집계 기본 사항

* PromQL은 필요한 모든 것을 처리할 수 있을 정도로 강력하지만, 대부분은 상당히 단순한 작업만 처리한다.

### Gauge

* gauge는 state의 스냅샷이다.
* 일반적으로 합계, 평균, 최솟값, 최댓값을 구하기 위해 게이지를 집계한다.
* 어떤 label이 반환될 것인가에 대한 예측 가능성은 연산자를 포함하는 벡터 매칭에 중요하다.

### Counter

* 이벤트 개수나 크기를 추적함.
* `/metrics` 경로에 표시하는 값은 애플리케이션이 시작한 이후의 총합이다.
  * 실제로 사용자가 알고싶어하는 값은 보통 얼마나 빠르게 증가하는지에 관한 것이다.
  * 일반적으로 `rate` 함수를 사용한다.
* `rate` 의 결과는 gauge이다. 따라서 gauge와 동일한 집계 연산을 적용할 수 있다.

### Summary

* 일반적으로 summary 메트릭은 `_sum`, `_count` prefix를 포함한다.
* 간혹 prefix 없이 quantile label이 있는 time series가 포함됨.
* `_sum` 과 `_count` 는 모두 카운터이다.
* 서머리의 강점은 이벤트의 평균 크기를 계산할 수 있다는 점이다.
* 나누기 연산자는 동일한 label이 있는 time series를 일치시키고 나눈다.
* 평균을 계산하는 경우, 합과 카운트를 먼저 집계하고 마지막 단계에서 나누는 것이 중요하다.
  * 그렇지 않으면 통계적으로 유효하지 않은 평균을 구하게 된다.

### Histogram

* histogram 메트릭으로 이벤트 크기에 대한 분포를 추적할 수 있다.
* quantile을 사용하는 경우 몇분위가 몇번째인지 사전에 알고있어야 한다.

## 선택기

* 일반적으로 어떤 time series를 가지고 작업하는지 파악하길 원할 것이며, 대부분의 경우 job 레이블에 따라, 그리고 어떤 사항에 의존하는가에 따라 제한하고 싶을 것이다.
* 이런 label에 의한 제한은 selector를 통해서 수행된다.
* vector는 기본적으로 1차원 리스트를 의미
* `job="node"` 와 같은 것을 matcher라고 부르며 하나의 selector에 AND로 함께 처리된 많은 matcher가 있을 수 있다.

### Matcher

* `=` : equality matcher. `job="node"`. 없음을 지정하려면 `foo=""`와 같이 사용할 수 있음.
* `!=` : negative equality matcher. `job!="node"`. 
* `=~` : regular expression matcher. `job=~"n.*"`.
* `!~` : negative regular expression matcher.
* `{}`는 오류를 반환한다.
  * 좀 더 자세히 말하자면 최소한 selector에 있는 matcher 중 하나는 빈 문자열과 일치해야한다.

### Instance Vector

* instance vector selector는 쿼리 평가 시점 전 가장 최근 샘플의 instant vector를 반환하며, 0개 이상의 time series list로 표현된다.
* 각 time series에는 샘플이 하나씩 있고, 샘플에는 value와 timestamp가 모두 포함된다.

### 범위 벡터

* time seires마다 하나의 샘플 메트릭을 제공하는 instance vector와는 다르게, range vector selector는 각 time series에 대한 샘플을 반환할 수 있다.
* 프로메테우스같은 메트릭 기반 모니터링 시스템은 정확한 답보단 추정치를 만들기 때문에 인위적인 결과를 만들어낼 수 있다.

### 오프셋

* offset이라 불리는 vector selector 유형과 함께 사용할 수 있는 것으로 modifier가 있다.
* 쿼리에 대한 평가 시간을 가질 수 있게 하며, 선택기 단위로 시간을 더 되돌린다.

## HTTP API

* 프로메테우스는 많은 HTTP API를 제공함.
* 대부분의 API는 PromQL에 접근하는 query와 query_range이다.

### Query

* query endpoint.
* `/api/v1/query`
* 쿼리 해석 시간을 URL 파라미터로 줄 수 있다.

### query_range

* `/api/v1/query_range`
* query_range endpoint
* 주의사항
  * 각 결과는 서로 다른 인스턴스 쿼리 해석에서 나오기 때문에, 샘플 타임스탬프들은 start 시간과 step이 일치하고, 인스턴스 쿼리 결과는 항상 쿼리의 해석 시간을 결과에 대한 타임스탬프로 사용해야 한다.
  * 결과의 마지막 샘플은 end 시간에 있어야 한다.
  * rate 함수에 대해 5분의 범윌르 선택했는데, 5분의 범위는 step보다 크다.
    * 최소한 step보다 1~2배 더 큰 데이터 수집 간격 범위를 사용해야함.
  * 모든 숫자는 정확한 값이 아닌 근삿값이다.

#### 정렬된 데이터

* 그라파나 같은 도구를 사용하는 경우, 현재 시간을 기반으로 query_range를 정렬하는 것이 일반적임.
* query_range에는 정렬을 지정하는 옵션이 없으며, 그 대신 사용자가 정렬을 위해 적절한 start 파라미터를 지정할 수 있다.