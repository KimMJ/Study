# Kiali 요약

Kiali는 OpenShift나 Kubernetes 클러스터 환경에 설치됩니다.

Istio에서 깔린 Kiali는 최신버전이 아닐 수 있으며, 깔고 나서 최신버전으로 수동 업데이트가 가능합니다.

### Kiali CR(Custom Resource)

Kiali 설치를 했다면 Kiali Operator를 이용해 설치한 것입니다. Kiali Operator는 Kiali CR(Custom Resource, Kiali 설정파일에 대한 yaml)을 확인합니다. Kiali CR을 수정하면 필요에 따라 Kiali가 설치, 업데이트, 삭제됩니다.

### Namespace 관리

#### 접근 가능한 namespace

Kiali CR은 Kiali Operator가 어떤 namespace에 접근할 수 있는지 알려줍니다. 이는 CR의 deployment 섹션에서 accessible_namespaces로 정의되어있습니다.

#### 접근 불가능한 namespace

마찬가지로 exclude 설정을 통해서 Kiali Operator가 접근하면 안되는 namespace를 설정할 수 있습니다.



## Features

Kiali는 Istio service mesh를 define, validate, observe하는데 도움을 줍니다. Kiali는 내  service mesh에 어떤 microservice가 있고 어떻게 그것들이 연결되는지에 대한 답을 보여줍니다.

Kiali는 Kubernetes, Openshift안에서 Istio와 함께 동작합니다. 이는 service mesh topology를 시각화 해주고, request routing, circuit breakers, request rates, latency 등등에 대한 정보를 제공합니다. Kiali는 mesh component에 대해 abstract Application에서 Service와 Workload까지 다양한 뷰를 보여줍니다.

또한 Kiali는 Jaeger Tracing을 통해 별도의 추가 설치 없이 분산 추적(Distributed Tracing)이 가능합니다.

### Observability Features

operation에서 service mesh를 볼 수 있도록 해줍니다. topology, telemetry, traces, logs, events, definition들을 보여줍니다. 이를 통해 mesh가 정상 동작을 하는지, 아니면 어떤 곳에서 에러가 나는지를 빠르게 확인할 수 있도록 해줍니다. 따라서 하나의 시스템을 전체적으로 볼 수 있도록 해줍니다.

#### Graph Views(Graph 메뉴)

그래프는 service mesh의 topology를 보기에 좋습니다. 어느 서비스가 어떤 서비스와 통신하는지, 그 사이에 traffic rates와 latency는 어떤지 확인할 수 있습니다. 이를통해 시각적으로 문제점을 확인할 수 있고 문제점을 파악할 수 있습니다. Kiali는 4가지 타입의 그래프를 제공합니다.

사용자는 원하는 데이터와 네임스페이스를 보도록 그래프를 설정할 수 있습니다.

##### Graph: Health

그래프의 색깔은 service mesh의 Health를 알려줍니다. 노드가 빨간색, 오렌지색일 경우 확인이 필요합니다. 두 component 사이의 선은 두 component간의 request에 대한 상태입니다. 노드의 모양을 통해 component, service, workload, app을 구분합니다.

##### Graph: Drill-Down

어떤 component 하나에만 집중하고 싶을경우 이를 더블클릭하여 자세히 볼 수 있습니다. 그렇게 하면 해당 component에 대해 incoming request와 outgoing request만 보여줍니다. 모든것은 component의 telemetry(원격 디바이스에서 얻는 데이터의 자동화된 감지 및 측정) 관점입니다.

##### Graph: Side-Panel

그래프에 대한 요약정보를 보고싶다면 어떤 노드는 싱글클릭으로 확인할 수 있습니다. Charts, Health, Links, Response Code를 포함합니다.

또한 그래프의 배경을 선택하면 전체적인 그래프의 요약정보도 볼 수 있습니다.

##### Graph: Traffic Animation

HTTP 트래픽에 대해서는 circle이 성공적인 request를 표현하고, 빨간 다이아몬드는 에러를 표현합니다. 원이나 다이아몬드가 더 밀집할 경우 request rate이 빠르다고 볼 수 있습니다. 애니메이션이 빠르면 response time이 빠른 것입니다.

TCP 트래픽은 offset circle로 표현이 되고 circle의 스피드는 트래픽의 스피드를 의미합니다.

그래프 아래의 legend를 클릭하시면 어떤 모양인지 확인할 수 있습니다.

Graph: Graph Types

* **workload** : workloads, pods에 대한 자세한 view를 제공합니다.
* **app** : 더 logical한 관점에서 app labeling에 대한 workload는 묶어서 보여줍니다.
* **versioned app** : app 단위로 묶지만, version-specific한 트래픽 분기를 다르게 표현하여 보여줍니다. (ex. rating에서 db를 조회하는것, 안하는것)
* **service** : high-level관점에서 정의된 서비스에 대한 모든 트래픽을 묶어서 보여줍니다.

#### Detail Views(다른 detail 메뉴들)

##### Detail: Metrics

각각의 detail view는 **<u>사전에 정의를 한 metrics dashboard</u>**입니다. 

##### Detail: Service (Service 메뉴)

service detail view는 현재 동작중인 service의 workloads를 보여줍니다. 또한 service와 연관된 Istio traffic routing configuration을 보여줍니다.

Kiali에서는 yaml파일에 접속하여 권한이 있다면 파일을 수정, 삭제할 수 있습니다. 또한 wizard 기능을 통해 검증까지 도와줍니다.

##### Detail: Workloads (Workloads 메뉴)

Kiali는 Istio sidecar의 배포가 되었는지, 적절한 app과 version 라벨이 설정되었는지 검증을 합니다.

Workload detail은 어느 workload가 request를 핸들링하는지, 어떤 pods가 workload를 지원하는지 서비스 관점에서 보여줍니다.

Workload detail은 또한 pod logs에 접근할 수 있고, 자세한 traffic breakdown을 제공합니다.

##### Detail: Runtimes Monitoring/Dashboard

Kiali는 Go, Node.js등등 몇몇 런타임 서비스에 대한 default dashboard를 제공합니다. (component를 클릭하여 나타나는 메뉴에서 확인이 가능합니다.)

##### Distributed Tracing

Distributed Tracing 메뉴에서 아이템을 클릭하면 tracing service를 위한 Jaeger UI가 새로운 탭으로 열립니다.

#### Configuration and Validation Features

Kiali에서는 Istio service mesh를 보는것만 하는것이 아니라 설정, 업데이트, 검증에 도움을 주기도 합니다.

## Architecture

Kiali는 container application platform에서 동작하는 back-end application과 front-end application을 구성이 됩니다. Kiali는 container application platform과 Istio에서 제공하는 external service, components에 대해 의존합니다.

![Kiali Architecture](https://www.kiali.io/images/documentation/architecture/architecture.png)

#### Kiali back-end

Istio와 통신을 하여 데이터를 얻고 처리합니다. 그리고 데이터를 front-end로 expose합니다.

#### Kiali front-end

하나의 web-application을 생각하시면 됩니다.

front-end는 Kiali back-end에 쿼리를 보내서 data를 얻고 유저에게 뿌려줍니다.

stateless(이전 상태를 기록하지 않음)으로 통신합니다.

#### Istio

Kiali를 동작할 때 필수사항입니다. service mesh를 제공하고 컨트롤 합니다.

Kiali는 Istio data와 configuration을 Prometheus, cluster API를 통해서 가져옵니다.

#### Prometheus

prometheus는 Istio에 대해 종속적입니다.(Istio가 있어야 prometheus가 동작합니다.) Istio telemtry가 설정이 되면 metrics data는 Prometheus에 저장이 됩니다. Kiali는 이 저장된 데이터를 사용하여 mesh topology를 알아내고, 통계정보를 보여주고, health를 파악합니다.

Kiali는 Prometheus와 직접 통신을 하고 Istio Telemetry에 사용되는 data schema를 가정해서 통신합니다. 이것은 Kiali의 hard dependency로써, 대부분의 feature가 이것 없이는 동작하지 않습니다.

**현재의 Kaili는 Istio의 default metrics에 의존합니다. Custom metrics는 Kiali에서 지원하지 않습니다. 만약 custom metrics를 사용해야 할 경우 새로운 Istio metrics를 custom names를 사용해서 생성해야 합니다.**

#### Cluster API

Kiali는 container application platform의 API를 사용해서 service mesh configurations에 fetch하고 resolve합니다.

Kiali는 definitions와 같은 정보를 얻기 위해 cluster API로 쿼리를 보냅니다. 

cluster API는 Istio configurations를 얻는데 사용되기도 합니다.

#### Jaeger

옵션입니다. Istio의 distributed tracing이 가능할 때만 tracing data를 사용할 수 있습니다.

#### Grafana

옵션입니다. 사용할 경우 Kiali의 metrics pages는 Grafana에서 동일한 metrics를 보여주는 링크를 줍니다.



Kiali는 basic metric capabilites입니다. workloads, apps, services에 대한 default Istio metrics를 보여줍니다. **하지만 Kiali는 view를 커스터마이징하거나 Prometheus query를 커스터마이징 할 수 없습니다. 이를 원할 경우 Grafana를 이용해야 합니다.** 



----

### Summary

간단히 요약하자면, Kiali는 Prometheus에서 저장하는 Istio service mesh에 대한 metrics data(통계 자료)를 그래프 등을 통해 시각적으로 확인할 수 있도록 도와줍니다. 또한 yaml 파일, configuration을 Kiali를 통해서 확인, 수정할 수 있습니다. 그러나 customize metrics를 사용할 수 없으므로 이런 기능이 필요하다면 grafana에서 동작하도록 해야합니다.

전체적인 service mesh를 시각화 하고, default metrics에 대해서 확인하는 데 좋습니다. 또한 복잡한 service mesh 안에서 장애가 난 부분을 그래프를 통해 빠르게 캐치할 수 있습니다. dashboard 또한 default dashboard를 사용하여 변경이 불가능한 것 같습니다.

Kiali는 Istio, Prometheus, cluster API가 함께 동작을 해야합니다.

Distribute Tracing에 관한 부분은 Jaeger에 위임합니다.