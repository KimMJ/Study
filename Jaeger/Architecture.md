# Architecture

Jaeger clients는 OpenTracing standard에 맞는 data model에 붙어있다. [specification](https://github.com/opentracing/specification/blob/master/specification.md)을 읽으면 이 section을 이해하는데 도움을 줄 것이다.

### Terminology

OpenTracing Specification에 정의된 terminology를 간단히 확인하자.

### Span

**span**은 Jaeger에서 논리적인 일의 단위를 의미한다. 이는 operation name, operation의 start time, duration을 가지고 있다. spans는 nested되었고 model causal relationships에 관한 순서가 있다.

![Span](https://www.jaegertracing.io/img/spans-traces.png)

### Trace

trace는 system에서 data/execution path이다. 그리고 **spans**의 directed acyclic graph라고 생각할 수 있다.

### Components

Jaeger는 모든 Jaeger backend component가 하나의 process에서 동작하는 all-in-one binary파일로 배포할 수도 있고 아래에서 설명할 scalable distributed system에서도 배포가 가능하다. main deployment에는 두가지 옵션이 있다.

1. Collectors가 storage에 직접 쓴다.
   ![directly to storage](https://www.jaegertracing.io/img/architecture-v1.png)
2. Collectors가 preliminary buffer로 Kafka를 사용한다.
   ![using Kafka](https://www.jaegertracing.io/img/architecture-v2.png)

이 section은 Jaeger의 구성요소에 대해 설명하고 어떻게 그것들이 관계되는지를 설명한다. 그것들과 상호작용하는 우리 application의 spans에 있는 순서로 정렬하여 설명한다.

### Jaeger client libraries

Jaeger clients는 OpenTracing API의 language specific implementations이다. Jaeger clients는 이미 OpenTracing로 통합이 된 Flask, Dropwizard, gRPC와 같은 많은 존재하는 오픈소스 프레임워크 또는 수동으로 distributed tracing에 instrument applications으로 사용될 수 있다.

instrumented service는 새로운 request를 받고 context information(trace id, span id, baggage)을 outgoing request에 더할 때 spans를 생성한다. ids와 baggage만 requests에서 전파된다. spans를 구성하는 다른 정보들 (operation name, logs 등)은 전파되지 않는다. 대신 sampled spans가 process 밖으로 백그라운드에서 비동기식으로 Jaeger Agents에 전송이 된다.

instrumentation은 overhead가 매우 작다. 그리고 production에서 항상 동작하도록 고안되었다.

모든 traces가 생성이 된 동안 약간만 sample이 된다는 것을 인지해라.  Sampling a trace marks the trace for further processing and storage. - 해석이.. 기본적으로 Jaeger client는 traces의 0.1%를 샘플링한다. 그리고 agent로부터 sampling strategy를 얻어올 수 있다.

![Illustration of context propagation](https://www.jaegertracing.io/img/context-prop.png)

### Agent

Jaeger agent는 네트워크 데몬으로 UDP를 통해 전송되는 collector로 batch되고 보내지는 spans를 listen한다. 이것은 infrastructure component로 모든 호스트에 대해 배포되도록 디자인되었다. agent는 라우팅과 collectors의 discovery를 클라이언트로부터 추상화한다.

### Collector

Jaeger collector는 Jaeger agents로부터 traces를 받고 processing pipeline을 통해 그것들을 작동시킨다. 현재 Jaeger의 pipeline은 validates traces, indexing, 모든 transformations 처리 그리고 저장의 그것들을 저장하는 일을 한다.

Jaeger의 storage는 pluggable component이고 Cassandra, Elasticsearch, Kafka를 지원한다.

### Query

Query는 storage로부터 traces를 얻어 UI로 display하는 service이다.

### Ingester

Ingester는 Kafka topic에서 읽어서 다른 storage backend로 쓰기를 하는 service이다.

