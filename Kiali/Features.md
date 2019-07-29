# Features



Kiali는 Istio service mesh를 정의하고, 검증하고 관찰하는데 도움을 준다. Kiali는 다음과 같은 질문에 대한 해답을 제공한다. : 내 service mesh에는 어떤 microservice가 있고, 어떻게 그것들이 연결이 될까?

Kiali는 Kubernetes, Openshift 안에서 Istio와 함께 동작한다. 이것은 service mesh topology를 시각화 해주고, request routing, circuit breakers, request rates, latency 등등에 대한 시각정보를 제공한다. Kiali는 mesh component에 대해 abstract Application에서 Service와 Workload까지 다양한 시각으로 볼 수 있도록 한다.

* Kiali는 Jaeger Tracing을  통해 별도의 설치 없이 분산 추적이 가능하다.

### Observability Features

Observability는 operation에서 service mesh를 볼 수 있게 한다. 이는 topology, telemetry(원격 측정), traces(추적), logs, events, definition을 결합한다. 이는 mesh가 정상 동작을 하는지, 아니면 빠르게 어떤 곳에서 에러가 나는지를 확인할 수 있도록 도와준다. 이는 각각의 다른 정보들을 하나의 시스템으로 전체적으로 볼 수 있도록 한다.

#### Graph

graph는 service mesh의 topology(위상)을 시각화 하는데 좋다. 이는 어느 서비스가 어떤 서비스와 통신하는지, 그 사이에 **traffic rates**와 **latencies**는 어떤지 보여준다. 그래서 시각적으로 문제점을 확인할 수 있고 어떤 부분이 문제인지 빠르게 확인이 가능하다. Kiali는 네가지 타입의 그래프(서비스 상호작용에 대한 high-level view, pods에 대한 low-level view, application들에 대한 logical view)를 제공한다.

그래프는 또한 서비스가 어떤 것들(virtual services, circuit breakers)와 설정이 되었는지 보여준다. 또한 예측되지 않는 설정으로 들어오는 트래픽을 분류해서 보안도 확인할 수 있다. 그리고 component간의 traffic flow를 애니메이션이나 metrics를 봄으로써 관찰 가능하다.

원하는 데이터와 네임스페이스를 보도록 그래프를 설정할 수 있다. 그리고 우리의 needs에 맞게 변경할 수 있다.

##### Graph: Health

그래프의 색깔은 service mesh의 **상태(Health)**를 알려준다. 노드가 빨간색, 오렌지색일 경우 신경을 써야한다. 두 component 사이의 엣지는 두 component간의 **request**에 대한 상태이다. 노드의 모양은 component, services, workloads, apps 등의 타입을 보여준다.

노드와 엣지의 상태는 유저의 설정에 따라 자동으로 새로고침된다. 그래프는 또한 특정 상태에서 측정하기 위해 멈출 수 있다.

![Health](https://www.kiali.io/images/documentation/features/graph-health.png)



##### Graph: Drill-Down

service, workload, application에 관계 없이 하나의 component에만 집중하고 싶은 경우가 있을 수 있다. Kiali는 선택한 하나의 component에 대한 detail graph를 볼 수 있다.

그래프에서 노드를 더블클릭하면 Kiali는 해당 component에 대해서 디테일하게 보여준다. 이는 오직 incoming request와 outpoing request만을 보여준다. 해당 컴포넌트에 대한 모든 telemetry 정보이다.

다시 메인 그래프로 돌아가서 작업을 이어갈 수 있다.

![Drill down](https://www.kiali.io/images/documentation/features/graph-detailed.png)



##### Graph: Side-Panel

그래프에 대한 요약정보를 보고 싶을 경우 어떤 노드든 선택하면 side panel이 업데이트 되어 component에 대한 짧은 요약을 제공해준다. 다음과 같은 정보를 포함한다.

* **Charts** : 트래픽과 응답시간
* **Health** : 자세한 정보
* **Links** : fully-detailed 페이지
* **Response Code** : breakdowns(고장?)

혹은 그래프 배경을 선택하면 전체적인 그래프의 요약정보를 얻을 수 있다.

![Side Panel](https://www.kiali.io/images/documentation/features/graph-side-panel.png)

##### Graph: Traffic Animation

Kiali는 그래프에 대한 traffic animation을 포함한 몇가지 옵션을 제공한다.

HTTP 트래픽에 대해서는 원(circle)이 성공적인 request를 표현하고 빨간 다이아몬드는 에러를 표현한다. 원이나다이아몬드가 더 밀집하면 request rate이 높은 것이다. 애니메이션이 빠르면 response time이 빠른 것이다.

TCP 트래픽은 offset circle로 표현이 되고 원의 스피드는 트래픽의 스피드를 의미한다.

##### Graph: Graph Types

Kiali는 mesh telemetry를 보여주는 4가지 그래프를 제공한다. 각각의 그래프 타입은 트래픽에 대한 다른 view를 제공한다.

* **workload** 그래프는 workloads/pods에 대한 자세한 view를 제공한다.
* **app** 그래프는 더 logical한 관점에서 같은 app labeling에 대한 workload를 묶어 보여준다.
* **versioned app** 그래프는 app단위로 묶지만, version-specific한 트래픽 분기들을 다르게 분기시켜서 보여준다.
* **service** 그래프는 high-level 관점에서 정의된 서비스에 대한 모든 트래픽을 묶어서 보여준다.

![Graph Types](https://www.kiali.io/images/documentation/features/graph-types.png)



#### Detail Views

Kiali는 모든 service mesh definitions에 대한 리스트를 필터링해서 제공한다. 각각의 view는 health, detials, yamls, links를 제공하여 mesh를 시각화하는데 도움을 준다. 

* Services
* Applications
* Workloads
* Istio Configurations (Virtual Service, Gateways, etc)



##### Detail: Metrics

각각의 detail view는 **사전 정의된 metrics dashboard**이다. metrics dashboard는 relevant application, workload, service level에 맞게 구성이 되었다.

Application과 workload의 detail view는 request와 response metrics(volume, duration, size, tcp traffic)을 보여준다. 트래픽은 또한 inbound 또는 outbound 트래픽도 보여준다.

service detail view는 inbound 트래픽마다 request, response metrics를 보여준다.

##### Detail: Service

serivce detail view는 유저에게 현재 동작중인 service의 workloads를 보여준다. 이는 또한 service와 연관된 **Istio** traffic routing configuration(VirtualServices, DestinationRules)을 보여준다.

Kiali는 yaml파일에 접속해서 권한이 있는 유저에게 수정, 삭제를 할 수 있도록 한다. wizard를 통해서 잘못 설정된 route를 감시하기 위해 VirtualService에서 common configuration과 추가적인 검증을 도와준다.

**Detail : Workloads**

Kiali는 workload 설정에 대한 몇가지 검증을 수행한다.

* Istio sidecar는 배포가 되었는가?
* 적절한 app과 version 라벨이 설정 되었는가?

Workload detail은 어느 workload가 request를 핸들링하는지, 어떤 pods가 workload를 지원하는지 서비스 관점에서 보여준다.

Workload detail은 또한 **pod logs**에 접근할 수 있고, 자세한 traffic breakdown을 제공한다.

##### Detail: Runtimes Monitoring/Dashboards

Kiali는 Go, Node.js, Spring Boot, Thorntail, Vert.x를 포함한 몇몇 런타임 서비스에 대한 default dashboard를 제공한다.

이러한 dashboard는 간단한 Kubernetes resource이다. 따라서 선호하는 툴을 사용해서 생성, 수정, 삭제가 가능하다. 그러한 것들이 yaml 또는 json 파일로 정의가 되어 있어서 Git, track changes, share를 통해 source control할 수 있다.

자세한 사항은 [documentation page](https://www.kiali.io/documentation/runtimes-monitoring/) 에서 확인 가능하다.

![Runtimes Monitoring/Dashboards](https://www.kiali.io/images/documentation/features/runtimes_monitoring.png)

##### Distributed Tracing

Distributed Tracing 메뉴에서 아이템을 클릭하면 tracing service를 위한 Jaeger UI가 새로운 탭으로 열린다.

### Configuration and Validation Features

Kiali는 Istio service mesh를 단순히 관찰만 하는 것이 아니라 설정하고 업데이트하고, 검증하는데 도움을 준다.

#### Istio Configuration

Istio configuration view는 Virtual Services와 Gateways같은 Istio configuration object를 위한 필터링과 네비게이션을 제공한다.

Kiali는 Istio resource를 위한 inline config edtion과 semantic validation을 제공한다.

#### Validations Performed

이 섹션은 Kiali가 수행하는 Istio configuration에 대한 모든 검증 리스트이다. Most of these validations are done in addition to/on top of the existing ones performed by Istio’s Galley component (except those marked as deprecated). Most validations are done inside a single namespace only, any exceptions (such as gateways) are marked below. -- 해석어려움..

##### Istio Wizards

Kiali는 Wizards를 통해서 Istio configuration에 대한 create, update, delete 액션을 지원한다. 이것들은 Service Details page에서 Actions 메뉴에 위치한다.

![Istio Wizards](https://www.kiali.io/images/documentation/features/service-istio-actions.png)

이 액션들은 default로 동작할 수 있다.

Kiali는 또한 Istio configuration에 대한 쓰기 작업을 제한하는 view only 모드로도 설치가 가능하다.

이 옵션을 어떻게 설정하는지에 대해서는 [Kiali Operator CR](https://github.com/kiali/kiali/blob/master/operator/deploy/kiali/kiali_cr.yaml#L158)를 참조해라.

##### Weighted Routing Wizard

이 wizard는 특정 workload에 대해서 몇퍼센트로 트래픽을 보낼지 설정할 수 있다.

![Weighted Routing Wizard](https://www.kiali.io/images/documentation/features/wizard-weighted-routing.png)

Kiali는 destination workload에 대해 설정된 weights을 이용해서 single routing rule을 가지고 Istio resources(VirtualService, DestinationRule) 한 쌍을 생성할 것이다.

##### Matching Routing Wizard

Matching Routing Wizard는 multiple routing rule을 생성할 수 있도록 한다.

* 모든 rule은 Matching과 Routes section으로 구성된다.
* Matching section은 HEADERS, URI, SCHEME, METHOD, AUTHORITY같은 Http parameters를 사용해서 여러개의 필터링을 추가할 수 있다.
* Matching section은 비어있을 수 있고, 이 경우에 어느 http request도 받을 수 있다.
* Route section은 하나 또는 여러개의 Workloads를 선택할 수 있다.

Istio는 순서대로 routing rules를 제공한다. 즉, HTTP request에 대해서 첫번째로 매칭이 되는 룰이 routing을 수행해야 한다. Matching Routing Wizard는 rule의 순서를 변경할 수 있도록 한다.

![Matching Routing Wizard](https://www.kiali.io/images/documentation/features/wizard-matching-routing.png)

이전 Wizard에서와 같은 방식으로 Kiali는 생성된 VirtualService에 대해 정의된 routing rule에 매핑이 되는 한 쌍의 Istio resources를 생성한다.

Suspend Traffic Wizard

이 wizard는 유저가 service에 대해서 트래픽을 부분적으로 또는 전부 멈추도록 도와준다. 이는 어떤 workload가 트래픽을 받을지 정의하도록 해준다.

트래픽이 모든 workloads에 대해서 막혀있을때, Istio는 모든 Service request에 대해서 error code를 리턴한다.

![Suspend Traffic Wizard](https://www.kiali.io/images/documentation/features/wizard-suspend-traffic-thumb.png)

어떤 workload에 대해서 트래픽이 있으면 wizard는 weighted rule을 적용한다. 트래픽이 없으면 abort rule은 생성된 VirtualService와 DestinationRule의 Istio resource쌍에 적히게 된다.

##### Advanced Opotions

모든 이전의 wizard는 "advanced options" section이 있어서 유저가 TLS와 LoadBalancing에 대한 특정 설정을 정의할 수 있도록 한다.

![Advanced Options](https://www.kiali.io/images/documentation/features/wizard-advanced-options-thumb.png)

mTLS가 global cluster 또는 namespace에 대해 default로 설정이 되어있으면 이 옵션은 이미 선택된 상태이다.

##### More Wizard examples

다음 문서 [Kiali: Observability in Action for Istio Service Mesh](https://medium.com/kialiproject/kiali-observability-in-action-for-istio-service-mesh-69127f792103)는 더 다양한 예저와 어떻게 Kiali wizard로 Istio configuration을 설정하는지 보여준다.

