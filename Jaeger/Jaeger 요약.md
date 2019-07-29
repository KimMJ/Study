# Jaeger 요약

Jager는 distributed systems를 기반으로 하는 microservices에 대한 monitoring과 troubleshooting에 사용이 되는 distribted tracing system(분산 로그 추적)입니다.

OpenTracing에 영감을 받아 만들어진 프로젝트입니다.

### Span

Jaeger에서 논리적인 일의 단위입니다.

operation name, operation의 start time, duration을 포함합니다.

모델의 인과관계(model causal relationships)에 따른 순서가 있습니다.

![Span](https://www.jaegertracing.io/img/spans-traces.png)

### Trace

system에서의 data/ execution path입니다. spans의 DAG(directed acyclic graph)라고 봐도 됩니다. 위의 그림에서 왼쪽을 생각하시면 될 것 같습니다.

### Components

Jaeger는 모든 Jaeger backend component가 하나의 process에서 동작하는 all-in-one binary파일로 배포할 수도 있고, scalable distributed system에도 배포가 가능합니다. 두가지 옵션이 있습니다.

1. Collectors가 storage에 직접 쓴다.
   ![directly to storage](https://www.jaegertracing.io/img/architecture-v1.png)
2. Collectors가 preliminary buffer로 Kafka를 사용한다.
   ![using Kafka](https://www.jaegertracing.io/img/architecture-v2.png)



-----

Jager의 구성요소를 application의 spans의 순서로 정렬하여 설명합니다.

#### Jaeger client libraries

Jaeger client는 OpenTracing API의 language specific implementations입니다. 다양한 오픈소스 프레임워크 또는 수동으로 distributed tracing에 측정 도구로 이용할 수 있습니다.

trace 중에서 일부만 샘플링하여 사용합니다. default는 0.1%입니다. agent를 통해서 sampling strategy를 얻을 수 있습니다.

#### Agent

UDP를 통해 전송되는 span들을 listen하여 collector로 보냅니다. 모든 호스트에 대해 배포가 됩니다.

#### Collector

Jaeger agent로부터 trace를 받고 그것들을 processing pipeline을 통해 작동시킵니다. pluggable component이고, Cassandra, Elasticsearch, Kafka를 지원합니다.

#### Query

Storage로부터 trace를 얻어내여 UI로 보여주는 service입니다.

#### Ingester

kafka topic에서 읽어서 다른 storage backend로 쓰기를 해주는 service입니다.



----

## Summary

Jaeger는 Distributed Tracing System입니다. Distributed Tracing System은 마이크로 서비스와 같은 분산 시스템에서 디버깅하고 모니터링하는데 도움을 주는 것을 의미합니다. 어디에서 오류가 발생했는지 등을 알 수 있습니다.

하나의 큰 요청을 trace라고 생각하고 각각의 마이크로 서비스에 대한 요청을 span이라고 생각하면 될 것 같습니다. 따라서 어디에서 오류가 발생했느지 등을 추적하기 위해서 span id 정보를 메타데이터로 전송하고, 큰 틀에서 어떤 trace에 대한 요청인지 확인하기 위해 trace id를 전송합니다. 이러한 방식으로 한 요청이 각각의 어떤 서비스에서 어떤 순서로 동작하는지 등을 확인할 수 있습니다.

Jaeger Query는 이를 시각화하여 보여주는 것입니다. 따라서 이 곳에서 trace를 확인하면 어떤 span들이 있었는지 어떻게 처리가 되었는지 확인 할 수 있고 이것이 분산 로그 추적을 의미합니다.

