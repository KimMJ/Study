# Features

Jaeger는 distributed systems를 기반으로 하는 microservices에 대한 monitoring과 troubleshooting에 사용된다. 다음을 포함한다.

- Distributed context propagation
- Distributed transaction monitoring
- Root cause analysis
- Service dependency analysis
- Performance / latency optimization

### High Scalability

Jaeger backend는 failure의 single points(시스템 구성 요소 중에서 동작이 정지되면 전체 시스템이 중단되는요소)가 없도록, business needs에 따라 scale 할 수 있도록 고안되었다. 예를 들어 Uber에서 제공하는 Jaeger installation은 기본적으로 보통 하루에 몇십억의 span(Jaeger에서 논리적인 work 단위)을 처리할 수 있다. 

### Native support for OpenTracing

### Multiple storage backends

### Modern Web UI

### Cloud Native Deployment

### Observability

모든 Jaeger backend components는 Prometheus metrics를 default로 expose한다. (다른 metric backends도 지원한다.) Logs는 구조화된 logging library이 zap을 통해 작성된다.

