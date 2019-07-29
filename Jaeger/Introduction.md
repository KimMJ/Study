# Introduction

### About

Jaeger는 Dapper와 OpenZipkin에 영감을 받아 Uber Technologies에 의해서 release된 분산 트레이싱 시스템(분산 로그 추적, distributed tracing system)이다. 이는 distributed systems를 기반으로 하는 microservices에 대한 monitoring과 troubleshooting에 사용된다. 다음을 포함한다.

* Distributed context propagation
* Distributed transaction monitoring
* Root cause analysis
* Service dependency analysis
* Performance / latency optimization

Uber는 [Evolving Distributed Tracing at Uber](https://eng.uber.com/distributed-tracing/) 포스트를 작성했다. 이는 Jaeger에서 사용되는 구조적인 선택들에 대한 역사와 이유가 담겨있다.



### Features

* OpenTracing과 compatible(양립 가능한) data model과 instrumentation libraries
  * Go, Java, Node, Python, C++
* service/endpoint probabilities 각각에 대한 지속적인 upfront sampling 사용
* Multiple storage backends: Cassandra, Elasticsearch, memory
* ~~Adaptive sampling~~
* ~~Post-collection data processing pipeline~~

### Technical Specs

* Go로 작성된 Backend components
* React/Javascript UI
* 지원하는 stroage backends:
  * Cassandra 3.4+
  * Elasticsearch 5.x, 6.x
  * Kafka
  * memory storage

