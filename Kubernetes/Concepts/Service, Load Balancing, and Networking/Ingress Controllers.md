# Ingress Controllers

* Ingress resource가 동작하도록 하기 위해 cluster는 ingress controller가 반드시 실행하고 있어야 한다.
* `kube-controller-manager` 바이너리의 일부분으로 실행되는 다른 종류의 controllers와는 다르게 Ingress controllers는 cluster에서 자동으로 실행되지 않는다.
  * 이 페이지를 통해서 cluster에 가장 적합한 ingress controller implementation을 선택해라.
* Kubernetes는 GCE와 nginx controllers를 지원하고 유지하고 있다.

## Additional controllers

* Ambassador API Gateway는 Envoy 기반으로 Datawire에 의해 community, commercial이 지원되는 ingress controller이다.
* AppsCode Inc.는 Voyager ingress controller를 기반으로 HAProxy를 지원하고 유지한다.
* Contour는 Envoy 기반 ingress controller로 Heptio가 제공하고 지원한다.
* Citrix는 baremetal과 cloud 배포에 대한 하드웨어(MPX), 가상화(VPX), free containerized(CPX) ADC를 위한 Ingress Controller를 제공한다.
* F5 Networks는 F5 BIG-IP Controller for Kubernetes에 의해 지원, 유지된다.
* Gloo는 오픈소스 ingress controller로 Envoy를 기반으로 하여 solo.io에서 enterprise 지원을 받아 API Gateway functionality를 지원한다.
* HAProxy Technologies는 HAProxy Ingress Controller for Kubernetes의 지원을 받고 유지된다. official documentation 참조.
* Istio는 Control Ingress Traffic을 기반으로 한 ingress controller
* Kong은 Kong Ingress Controller for Kubernetes에 의해 community나 commercial에 대한 지원을 받고 유지되는 서비스.
* NGINX, Inc.는 NGINX Ingress Controller for Kubernetes에 대한 지원을 하고 유지한다.
* Kubernetes Ingress같은 use cases를 포함한 service composition에 대한 Skipper HTTP router와 reverse proxy는 custom proxy를 만들기 위한 라이브러리로 디자인되었다.
* Traefik은 fully featured ingress controller(Let's Encrypt, secrets, http2, websocket)이고 Containous에 의해 commercial 지원을 한다.

## Using multiple Ingress controllers

* cluster안에서 ingress controller를 몇개든지 배포할 수 있다.
  * ingress를 생성할 때, 하나 이상의 ingress controller가 cluster안에 있다면 각 ingress를 적절한 `ingress.class`로 annotate하여 어떤 ingress controller가 사용될지 정해주어야 한다.
* class를 정의해주지 않으면 provider는 default ingress controller를 사용할 것이다.
* 이상적으로는 모든 ingress controllers가 이 specification을 만족해야 하지만 다양한 ingress controllers들이 약간씩은 다르게 동작한다.

> Note : ingress controller의 documentation을 검토하여 이를 선택했을 때 어떤 문제가 있을지 잘 이해해라.