# Traffic Management

* Overview and terminology : Pilot, Istio의 코어 드래픽 관리 컴포넌트, Envoy 프록시에 대해 배우고 어떻게 그것들이 service discovery와 로드 밸런싱을 가능하게 하는지 배운다.
* Traffic routing and configuration: 라우팅을 하고 메쉬의 ingress, egress 트래픽을 컨트롤하는 Istio의 feature와 컴포넌트에 대해 배운다.
* Network resilience and testing: Istio의 동적 장애 복구 feature를 통해 장애가 나는 노드에 대해 견딜 수 있는 시스템을 설정할 수 있고 다른 노드로 장애가 전파되는 것을 막을 수 있다.

## Overview and terminology

Istio를 통해 서비스의 업데이트 없이 서비스 매쉬에서 트래픽 라우팅과 로드 밸런싱을 관리할 수 있다. Istio는 timeouts, retries같은 서비스 레벨의 속성들을 설정하는 것을 간단하게 하고, 간단하게 퍼센트 기반 트래픽 분산으로 staged rollout같은 일을 할 수 있게 한다.

Istio의 트래픽 관리 모델은 다음 두 컴포넌트에 의존한다.

* Pilot : 코어 트래픽 관리 컴포넌트
* Envoy proxies : Pilot을 통해 설정된 정책과 설정을 시행

이 컴포넌트들은 다음 두가지 high-level feature를 가능하게 한다.

* Service discovery
* Load balancing

### Pilot: Core traffic management

![Pilot Architecture](https://istio.io/docs/concepts/traffic-management/pilot-arch.svg)

그림에서 볼 수 있듯이 Pilot은 메쉬에서 모든 서비스에 대한 abstract model을 가진다. Pilot의 Platform-specific 어댑터가 abstract model을 적절한 플랫폼에 맞게 바꾸어 준다.

예를 들어, 쿠버네티스 어댑터는 컨트롤러가 pod의 등록 정보, ingress 리소스, 트래픽 관리 룰을 저장하는 CRDs(Custom Resource Definitions)같은 써드파티 리소스 의 변화를 감시하도록 한다. 쿠버네티스 어댑터는 이 데이터를 abstract model로 변환하여 Pilot이 적절한 Envoy-specific configuration을 생성하고 전달할 수 있도록 한다.

Pilot의 service discovery와 traffic rule은 abstract model을 통해 Envoy 프록시가 Envoy API를 통해 매쉬 안의 다른 프록시를 알도록 한다.

Networking과 Rules APIs를 통해서 서비스 매쉬의 트래픽에 대해 보다 세분와된 제어를 실행할 수 이싸.

### Envoy proxies

