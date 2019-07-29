# ARCHITECTURE

Kiali는 두가지 component로 구성이 되어있다 - container application platform에서 실행되는 back-end application과 유저단에서의 front-end application. 그리고 Kiali는 container application platform과 Istio에서 제공하는 external service와 components에 대해 의존한다.

다음의 diagram은 Kiali에 포함된 components와 그 상호작용을 보여준다.

![Kiali Architecture](https://www.kiali.io/images/documentation/architecture/architecture.png)

### Kiali back-end

back-end는 container application platform에서 동작하는 application이다. Go 언어로 작성되어 있다. 코드는 깃허브 페이지에서 확인이 가능하다.

이 component는 Istio와 통신을 하고 데이터를 얻고 처리한다. 그리고 데이터를 front-end로 expose한다.

back-end는 storage가 필요 없다. back-end를 local에서 구동을 할 때, 인증에 필요한 유저와 비밀번호를 포함한 모든 설정이 파일과 환경 변수로 설정이 되어있다. docker images를 통해 back-end를 cluster에 배포하려면 configmaps와 secrets가 설정을 위해 사용이 된다.

### Kiali front-end

front-end는 하나의 web application이다. React, Typescript로 작성이 되었다. 마찬가지로 깃허브 페이지에서 확인이 가능하다.

standard deployment에서 back-end는 front-end를 지원한다. 그러면 front-end는 Kiali back-end에 쿼리를 보내서 data를 얻고, 유저에게 보여준다.

현재 personalizations에 대한 옵션이 없어서 stateless(이전 상태를 기록하지 않음)이다. session credentials와 같은 어떤 데이터는 보존이 될 수도 있지만 이 데이터는 브라우저 내에 저장이 되고 다른 browsers나 devices에서는 이용할 수 없다.

### Istio

Istio는 Kiali의 요구사항이다.(Istio is a Kiali requirement) 이것의 component는 service mesh를 제공하고 컨트롤한다. Kiali와 Istio가 따로 설치가 가능하지만 Kiali는 Istio가 없으면 동작하지 않을 것이다.

Kiali는 Istio data와 configuration을 얻어야 하고 이를 Prometheus와 cluster API를 통해서 가져온다. 이러한 이유로 diagram에서 점선으로 간접적인 의존성을 표현한 것이다.

### Prometheus

Prometheus는 Istio에 대해 종속적이다.(Prometheus is an Istio dependency.) Istio telemetry가 설정이 되면 metrics data는 Prometheus에 저장이 된다. Kiali는 Prometheus에 저장된 데이터를 사용하여 mesh topology를 알아내고 metrics를 보여주고 health를 계산하고 가능한 문제들을 보여주는 등의 일을 한다.

Kiali는 Prometheus와 직접적으로 통신을 하고 Istio Telemetry에 의해 사용되는 data schema를 가정한다. 이것은 Kiali의  hard dependency이고 대부분의 features는 이것 없이는 작동하지 않는다.

**현재 Kiali는 [Istio's default metrics](https://istio.io/docs/reference/config/policy-and-telemetry/metrics/)에 의존한다. 이러한 default metrics가 항상 잘 되어있는지 확실히 해야 한다. Custom metrics는 Kiali에서 지원하지 않는다. 만약 custom metrics나 default의 변형을 필요로 할 경우 새로운 Istio metrics를 custom names를 사용해서 생성해야 한다.**



### Cluster API

Kiali는 container application platform의 API를 사용하여 service mesh configurations에 fetch하고 resolve한다.

Kiali가 동작한다고 알려진 Container application platform은 OKD와 Kubernetes가 있다. Kiali는 이런 플랫폼들의 derivatives에서도 동작해야한다. 만약 cluster API에 대해 알고싶으면 [OKD REST API reference](https://docs.okd.io/latest/rest_api/index.html)와 [Kubernetes API reference](https://kubernetes.io/docs/reference/kubernetes-api/)를 클릭하라.

Kiali는 definitions와 같은 정보(namespaces, services, deployments, pods 등 기타 entities)를 얻기위해 cluster API 쿼리를 보낸다. Kiali는 또한 쿼리를 만들고 다른 cluster entities 사이의 관계를 해석(resolve)한다.

cluster API는 또한 virtual services, destination rules, route rules, gateways, quotas 등 Istio configurations를 얻는데 사용되기도 한다.

### Jaeger

Jaeger는 optional이다. 사용 가능하다면 Kiali는 유저를 Jaeger의 tracing data로 안내한다. 만약 이 feature가 필요하다면 [적절한 Jaeger integration configuration](https://github.com/kiali/kiali#jaeger)을 확인해라.

Tracing data는 [Istio의 distributed tracing](https://istio.io/docs/tasks/telemetry/distributed-tracing/)이 가능할 때만 사용할 수 있다.

### Grafana

Grafana는 optional이다. 사용 가능하다면 Kiali의 metrics pages는 Grafana에서 같은 metric을 보여주는 링크를 보여줄 것이다. 만약 이 feature가 필요하다면 적절한 [Grafana integration configuration](https://github.com/kiali/kiali#grafana)을 확인해라.

Kiali는 basic metric capabilities이다. 이는 workloads, apps, services에 대한 default Istio metrics를 보여준다. It allows to apply some groupings to the provided metrics and fetch metrics for different time ranges -- 해석 어려움. 하지만 Kiali는 views를 customize하거나 Prometheus queries를 customize할 수 없다. 만약 이것을 원한다면 Grafana를 설치해야 한다. 필요하다면 Istio documentation에서 Grafana를 설치해라.

