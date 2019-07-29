# What is Istio?

클라우드 플랫폼은 이를 사용하는 사람들에게 많은 이점을 주었다. 하지만 이는 DevOps 팀에 큰 부담이 되었다는 사실은 부정할 수 없다. 개발자들은 portability를 위해 마이크로 서비스를 사용해야 하지만 operator는 매우 큰 복합적인 멀티 클라우드에 배포하는 것을 관리해야한다. Istio는 connect, secure, control, observe하는 서비스이다.

하이레벨 관점에서 Istio는 배포의 복잡성을 줄여주어 배포팀의 부담을 줄여준다. Istio는 완전한 오픈소스 서비스 메쉬로 기존에 있는 분산 어플리케이션에 투명하게 층을 이룬다. 이는 또한 플랫폼으로써 API를 제공하여 로깅 플랫폼, telemetry, policy 시스템을 통합시킬 수 있다. Istio의 다양한 feature는 성공적으로, 효과적으로 마이크로 서비스를 동작할 수 있게 해주고 마이크로 서비스에 대한 보안, 연결, 관리를 제공한다.

## What is a service mesh?

Istio는 monolithic 어플리케이션 분산 마이크로서비스 아키텍쳐로 바꿀 때 생기는 문제들을 해결한다. 어떻게 해결했는지 알기 위해서는 Istio의 서비스 메쉬를 자세히 보는 것이 도움이 된다.

서비스 메쉬라는 용어는 어플리케이션과 그 사이의 상호작용을 구성하는 마이크로 서비스의 네트워크를 설명하는데 쓰인다. 서비스 메쉬가 커지고 복잡해짐에 따라, 더 이해하기 어렵고 관리하기 어렵게 되었다. 요구사항으로 discovery, load balancing, failure recovery, metrics, monitoring가 포함된다. 서비스 메쉬는 또한 A/B testing, canary rollouts, rate limiting, access control, end-to-end authentication같은 더 많은 복잡한 운용상의 요구사항을 종종 포함하게 된다.

Istio는 전체 서비스 메쉬에 대한 behavioral insighs와 operational control을 제공하여 마이크로 서비스 어플리케이션의 다양한 요구사항을 만족시키는 완전한 솔루션을 제공한다.

## Why use Istio?

Istio는 <u>load balancing, service-to-service authentication, monitoring등의 매우 적은 또는 코드의 변화 없이 쉽게 서비스 네트워크를 만들 수 있도록 한다</u>. 당신의 환경 전체에 사이드카 프록시를 배포함으로써 Istio support를 서비스에 추가할 수 있다.  이를 통해 <u>마이크로 서비스 간의 모든 네트워크를 intercept하여 configure하고 Istio의 control plane 기능을 활용하여 관리할 수 있다.</u>

* HTTP, gRPC, WebSocket, TCP traffic에 대한 자동 load balancing
* 풍부한 라우팅 규칙, 재시도, 장애 해결, fault injection을 통해서 트래픽 동작을 미세하게 제어할 수 있다.
* access control, 속도 제한, 할당량을 지원하는 pluggable policy layer, configuration API
* ingress, egress를 포함한 모든 클러스터 사이의 트래픽에 대한 metrics, logs, traces
* 강한 ID 기반 인증 및 권한을 가진 클러스터에서 service-to-service communication을 보호한다.

Istio는 확장성을 위해 고안되었고 다양한 deployment 니즈를 만족시킨다.

## Core features

Istio는 서비스 네트워크를  통해 많은 주요한 기능을 제공한다.

### Traffic management

Istio의 쉬운 configuration과 traffic routing은 트래픽의 흐름과 서비스간의 API 호출을 제어할 수 있도록 해준다. Istio는 circuit breakers, timeouts, retries같은 서비스 레벨의 속성에 대한 configuration을 단순화하고, A/B testing, canary rollouts, 퍼센트 기반 트래픽 분산으로 staged rollouts 같은 중요한 일들을 쉽게 셋업할 수 있도록 한다.

트래픽에 대한 더 나은 visibility와 out-of-box failure recovery를 이용해서 이슈를 문제가 되기 전에 알아낼 수 있고, call을 더 reliable하게 할 수 있고, 네트워크를 더욱 견고하게 할 수 있다. - 어떤 상황에 처하더라도.

### Security

Istio의 보안 기능은 개발자들이 어플리케이션 레벨에서의 보안에만 집중할 수 있도록 한다. Istio는 밑단의 보안 통신 채널을 제공하고 인증, 권한, 서비스 통신에서의 암호화를 제공한다. Istio는 서비스 통신을 기본적으로 보호하고, 어플리케이션의 변경 없이 다양한 프로토콜과 런타임에 일관되게 정책을 집행할 수 있다.

Istio가 플랫폼에 종속되지는 않지만, 쿠버네티스 네트워크 policy와 함께 사용하면 이점은 더욱 커진다. pod-to-pod의 보안과, service-to-service 네트워크와 어플리케이션 레이어에서의 통신을 보호하는 것을 포함한다.

### Observability

Istio의 견고한 tracing, monitoring, logging은 서비스 메쉬 배포에 큰 인사이트를 준다. 실제로 어떻게 서비스 퍼포먼스가 Istio의 모니터링을 통해서 upstream, downstream에 영향을 주는지 알게 되고, 커스텀 대쉬보드는 모든 서비스 성능에 대한 visibility와 어떻게 다른 프로세스에 성능이 영향을 주는지 알수있게 된다.

Istio의 Mixer component는 policy control과 telemetry collection의 역할을 한다. 이것은 backend 추상화와 중재를 제공하고, individual infrastructure backend의 구현 세부사항으로부터 Istio의 나머지 부분을 분리하여 operator에게 메쉬와 infrastructure backend간의 모든 상호작용에 대한 정교한 control을 제공한다.

이 모든 feature을 통해 서비스에 대한 SLO를 보다 효과적으로 설정하고, 모니터링하고 enforce할 수 있다. 물론, 핵심은 빠르고 효과적으로 이슈를 감지하고 해결할 수 있다는 것이다.

### Platform support

Istio는 플랫폼 독립적이며 클라우드, on-premise, K8s, Mesos등 다양한 환경에서 작동하도록 디자인 되었다. Istio를 쿠버네이트사 Nomad with Consul에 배포할 수 있다. Istio는 다음을 지원한다.

* Kubernetes에서의 서비스 배포
* Consul로 서비스 등록
* 개인 VM에서의 서비스 동작

### Integration and customization

Istio의 policy enforcement component는 기존 ACL, logging, monitoring, quotas, auditing 등과 같은 솔루션과 통합되도록 확장 및 커스터마이징 할 수 있다.

## Architecture

Istio 서비스 메쉬는 논리적으로 `data plane`과 `control plane`으로 구분되어있다.

* `data plane`은 사이드카 형식으로 배포가 된 지능형 프록시(Envoy)로 구성되어있다. 이 프록시들은 범용 목적의  policy, telemetry hub인 `Mixer`를 이용해 마이크로 서비스의 모든 네트워크 통신들을 중재하고 관리한다.
* `control plane`은 route traffic에 대한 프록시를 설정하고 관리한다. 추가적으로 `control plane`은 Mixer의 설정을 하고 policy를 집행하고 telemetry를 수집한다.

다음의 diagram은 각각의 플레인을 구성하는 컴포넌트를 보여준다.

![Istio Architecture](https://istio.io/docs/concepts/what-is-istio/arch.svg)

### Envoy

Istio는 Envoy 프록시의 확장 버전을 사용한다. Envoy는 c++로 개발된 고성능의 프록시로써 서비스 메쉬의 모든 서비스의 inbound와 outbound 트래픽을 중재한다. Istio는 Envoy의 여러 기능들을 활용한다. 예를 들어:

* 동적 서비스 discovery
* Load balancing
* TLS termination
* HTTP/2, gRPC 프록시
* Circuit breaker
* Health 체크
* %기반 트래픽 분산을 통한 Staged rollout
* Fault injection
* Rich metric

Envoy는 같은 Kubernetes pod 내의 연관된 서비스에 사이트카 형식으로 배포된다. 이를 통해 Istio는 트래픽 동작에 대한 풍부한 신호를 속성으로 추출할 수 있다. Istio는 이런 속성을 `Mixer`에서 policy 결정을 하는데 쓰고, 그것들을 모니터링 시스템에 보내서 전체 메쉬의 동작에 대한 정보를 제공하는데 사용할 수 있다.

사이드카 프록시 모델은 또한 이미 존재하는 deployment에서 구조를 바꾸거나 코드를 다시 쓸 필요가 없이 Istio의 기능을 사용할 수 있도록 한다. 왜 이러한 방식을 채택했는지는 [Design Goals](https://istio.io/docs/concepts/what-is-istio/#design-goals)를 참조하자.

### Mixer - access control, telemetry

`Mixer`는 플랫폼 독립적인 컴포넌트이다. <u>`Mixer`는 서비스 매쉬에서 `access control`과 사용 policy를 집행하고, telemetry data를 `Envoy` 와 다른 서비스로부터 모은다.</u> 프록시는 request level의 속성들을 추출하고 이를 `Mixer`로 보내서 측정한다. 더 많은 attribute extraction과 policy evaluation에 대한 정보는 [Mixer Configuration documentation](https://istio.io/docs/concepts/policies-and-telemetry/#configuration-model)에서 얻을 수 있다.

`Mixer`는 유연한 플러그인 모델이다. 이 모델은 Istio가 다양한 호스트 환경과 infrastructure backend에 접속할 수 있도록 한다. 그래서 Istio는 Envoy 프록시와 `Istio-managed` 서비스를 디테일로부터 추상화한다.

### Pilot - traffic management, service discovery

<u>`Pilot`은 `Envoy` 사이드카, 지능형 routing에 대한 트래픽 관리 기능(A/B test, canary rollout 등), resiliency(timeouts, retries, circuit breakers 등)을 위한 service discovery를 제공한다.</u>

<u>`Pilot`은 트래픽 동작을 제어하는 high level 라우팅 룰을 Envoy-specific configurations으로 변환하고, 런타임에 사이드카로 전파한다.</u> `Pilot`은 플랫폼 별 servce discovery 메카니즘을 추상화하고, `Envoy data plane APIs`를 준수하는 사이드카가 사용할 수 있는 표준 형식으로 합성한다. 이 loose coupling은 Istio가 Kubernetes, Consul, Nomad같은 다향한 환경에서 작동할 수 있도록 하면서 트래픽 관리에 대한 동일한 operator 인터페이스를 유지하도록 한다.

### Citadel - security

`Citadel`은 내장 ID와 credential 관리를 통해 강력한 service-to-service, end-user authentication을 가능하게 한다. 서비스 메쉬에서 암호화 되지 않은 트래픽을 업그레이드 하는데 `Citadel`을 사용할 수 있다. `Citadel`을 사용하면, operator는 상대적으로 불안정한 layer 3나 layer 4 네트워크 식별자가 아닌 서비스 ID를 기반으로 policy를 적용할 수 있다. release 0.5부터 [Istio's authorization feature](https://istio.io/docs/concepts/security/#authorization)을 통해 서비스에 누가 접속할 수 있는지 제어할 수 있다.

### Galley - configuration

`Galley`는 Istio의 configuration validation, ingestion, processing, distribution 컴포넌트이다. 이는 Kubernetes같은 기반 플랫폼에서 user configuration을 얻는 것을 Istio 컴포넌트로부터 분리한다.

## Design Goals

몇개의 design goal이 Istio 구조에서 알려졌다. 이러한 goal들은 서비스가 scale, high performance할 수 있도록 만들어주는데 필수적이다.

* **Maximize Transparency** : Istio를 사용하기 위해서는 operator나 developer가 시스템으로부터 실제 값을 얻어낼 수 있도록 최소한의 일을 수행하면 된다. 이는 결국 Istio가 자동적으로 서비스 사이의 네트워크 path에 주입이 되도록 한다. Istio는 사이드카 프록시를 통해 트래픽을 캐치하고 가능한 곳에서는 이미 배포된 어플리케이션의 어떤 코드 변화도 없이 자동으로 네트워크 레이어를 트래픽이 프록시로 라우트되도록 한다. Kubernetes에서 프록시는 pod에 주입되고 트래픽은 `iptables` rule을 프로그래밍하여 캡쳐된다. 사이드카 프록시가 주입되고 트래핑 라우팅이 프로그래밍되면, Istio는 모든 트래픽을 중재할 수 있다. 이 원리는 또한 성능에도 적용이 된다. Istio를 deployment에 적용하면 operator는 기능들이 제공되는데 최소한의 resource cost의 증가로 할 수 있다. 컴포넌트와 API는 반드시 성능과 scale을 생각하여 디자인되어야 한다**.**

* **Extensibility** : operator와 developer는 Istio가 제공하는 기능에 더욱 의존적이게 됨으로써 시스템은 그들의 니즈에 맞게 성장해야 한다. Istio가 계속해서 새로운 기능을 추가하는 동안 가장 중요한 것은 policy system을 확장하고, 다른 policy와 control 소스와 통합하고 메쉬 동작에 대한 신호를 분석을 위한 다른 시스템에 전파시키는 것이다. policy runtime은 다른 서비스를 연결하기 위한 standard extension mechanism을 지원한다. 또한 이는 메쉬가 생성하는 새로운 신호를 기반으로 policy를 시행할 수 있도록 vocabulary의 확장을 지원한다.
* **Portability** :  Istio가 사용되는 생태계는 다양한 dimension을 거쳐 변화한다. Istio는 클라우드나 on-premise 환경에서 최소한의 노력으로 동작해야 한다. Istio기반 서비스를 새로운 환경에서 포팅하는 일은 쉬워야 한다. Istio를 사용하면 다양한 환경에 배포된 single service를 운영할 수 있다. 예를 들어, redundancy를 위해 여러 클라우드에 배포할 수 있다.
* **Policy Uniformity** : 어플리케이션의 서비스간 API 호출 정책은 매쉬 동작에 대한 많은 제어를 준다. 하지만 이는 API 레벨에서 반드시 표현되지 않는 리소스에 policy를 적용하는 것도 똑같이 중요할 수 있다. 예를 들어 머신러닝 트레이닝에 사용되는 CPU 할당량을 주는 것은 작업을 시작한 호출에 대한 할당을 주는것 보다 중요하다. 이를 위해 Istio는 사이드카 프록시에 policy system을 굽지 않고 그 자체의 API로써 개별적인 서비스로 policy system을 유지한다. 따라서 서비스가 필요에 따라 통합될 수 있도록 한다.