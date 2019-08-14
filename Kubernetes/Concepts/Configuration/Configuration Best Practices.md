# Configuration Best Practices

## General Configuration Tips

* configurations를 정의할 때 최신의 stable API version을 지정하라.
* configuration file은 cluster에 push하기 전에 version control로 저장되어야 한다.
  * 이는 필요시 빠르게 configuration change를 롤백할 수 있게 한다.
  * 또한 cluster 재생성, 복구에 도움을 준다.
* configuration을 JSON이 아닌 YAML으로 작성하라.
  * 이 두 포맷이 거의 모든 시나리오에서 서로 변환 가능하지만 YAML이 더 user-friendly하다.
* 가능한 관련된 오브젝트를 하나의 파일로 만들어라.
  * 하나의 파일은 여러개일때 보다 더욱 관리하기 편하다.
  * [guestbook-all-in-one.yaml](https://github.com/kubernetes/examples/tree/master/guestbook/all-in-one/guestbook-all-in-one.yaml)참고
* 많은 `kubectl` 명령어가 디렉토리로 호출될 수 있음을 기억해라.
  * 예를 들어 `kubectl apply`를 config file의 디렉토리에서 호출할 수 있다.
* 필요하지 않다면 default 값을 지정하지 마라.
  * 간단하고, 최소한의 configuration이 더 적은 error를 만든다.
  * 오브젝트 description을 annotations 안에 넣어 내부를 확인하기에 더 좋게 하라.

## "Naked" Pods vs ReplicaSets, Deployments, and Jobs

* 피할 수 있다면 naked Pods(즉, 파드가 ReplicaSet이나 Deployment에 연결되지 않은 것들)를 사용하지 말라.
  * naked pods는 노드 장애시에 rescheduled되지 않는다.
* ReplicaSet을 생성해서 원하는 숫자의 파드를 항상 이용가능하도록 해주고 RollingUpdate같은 파드의 교체정책을 지정하는 Deployment는 `restartPolicy: Never`같은 예외 시나리오를 제외하면 직접 파드를 생성하는 것보다 항상 더 좋다.
  * Job 또한 적절하다.

## Services

* 어떤 workload가 backend workloads(Deployments나 ReplicaSets)에 접속하려 하기 전에 해당하는 Service를 먼저 생성하라.

  * Kubernetes가 container를 시작하면 container가 시작하는 시점에 동작하고 있는 모든 서비스를 가리키는 환경 변수를 제공한다.

  * 예를 들어, `foo`라는 Service가 있으면 모든 container는 다음 환경변수를 초기 환경으로 가진다.

    ```shell
    FOO_SERVICE_HOST=<the host the Service is running on>
    FOO_SERVICE_PORT=<the port the Service is running on>
    ```

* 이는 ordering requirement를 내포한다.

  * `Pod`가 접속하려는 모든 `Service`는 반드시 `Pod`가 생성되기 전에 생성되어야 하고 아니면 환경변수는 생성되지 않는다.
  * DNS는 이런 제약사항이 없다.

* optional cluster add-on(강력히 추천한다)은 DNS server이다.

  * DNS server는 Kubernetes API를 관찰하여 새로운 `Services`가 있는지 보고 각각에 대해서 DNS records를 생성한다.
  * 만약 DNS가 cluster 전반에 걸쳐 활성화되었다면 모든 `Pods`는 `Services`에 대해 자동으로 name resolution을 할 수 있다.

* 절대적으로 필요한 상황이 아니라면 파드에 대해서 `hostPort`를 지정하지 마라.

  * `hostPort`에 파드를 연결시키면 각 <`hostIP`, `hostPort`, `protocol`>의 조합이 유일해야하기 때문에 파드가 스케쥴링 될 수 있는 곳의 수를 제한한다.
  * `hostIP`와 `protocol`을 따로 지정하지 않으면 Kubernetes는 `0.0.0.0`을 `hostIP`의 기본값으로 사용하고 `TCP`를 `protocol`의 기본값으로 사용한다.

* 디버깅의 목적으로 포트에 접속해야한다면 apiserver proxy나 `kubectl port-forward`를 사용할 수 있다.

* 노드에서 파드의 포트를 반드시 노출시켜야 한다면 `hostPort`에 의지하기보다는 NodePort Service를 사용하는 것을 고려해 보아라.

  * 같은 이유로 `hostNetwork`의 사용을 피하라.
  * headless Services(`ClusterIP`가 `None`인 것)을 사용하여 `kube-proxy`의 load balancing이 필요 없을 때 쉽게 service discovery를 사용할 수 있다.

## Using Labels

* `{ app: myapp, tier: frontend, phase: test, deployment: v3 }`처럼 labels를 만들고 이르 사용하여 application이나 deployment의 semantic attributes를 나타내라.
  * 이런 labels를 이용하여 다른 리소스에 대해서 적절한 파드를 선택할 수 있다.
  * 예를 들어 `tier: frontend` 파드를 선택하는 모든 Service나 `app: myapp`에 대한 모든 `phase: test` 컴포넌트를 선택할 수 있다.
  * [guestbook](https://github.com/kubernetes/examples/tree/master/guestbook/) app은 이런 예시를 사용하였다.
* Service를 release-specific labels를 selector에서 생략함으로써 복수의 Deployments를 span하도록 할 수 있다.
  * Deployments는 downtime없이 동작하는 서비스를 업데이트 하기 쉽다.
* 오브젝트에 대한 원하는 state는 Deployment에 기록되어있고 spec에 대한 변경이 적용되면 deployment controller는 정해진 rate에 따라 실제 state를 원하는 state로 변경한다.
* label을 디버깅을 위해 만들 수 있다.
  * ReplicaSet같은 Kubernetes controller와 Services가 selector labels를 이용해서 파드와 매칭하기 때문에 파드에서 관련된 labels를 삭제하는 것은 controller에 의해서 관리되지 않게 되고 Service에 의해 트래픽을 받지 못할 것이다.
  * 존재하던 파드에서 labels를 삭제하면 controller는 새로운 파드를 만들어 대체할 것이다.
  * 격리된 환경에서 동작하는 파드를 미리 디버깅하는 좋은 방법일 것이다.
  * 실시간으로 labels를 삭제하거나 추가하고 싶으면 `kubectl label`을 사용하라.

## Container Images

* imagePullPolicy와 image의 tag는 kubelet이 지정된 이미지를 pull하려 할 때 영향을 준다.
  * `imagePullPolicy: IfNotPresent` : 이미지가 local에 없을때만 pull을 한다.
  * `imagePullPolicy: Always` : 파드가 시작될 때 항상 이미지를 pull한다.
  * `imagePullPolicy`가 생략되었고 이미지 tag가 `:latest`이거나 생략되었을 때 : `Always`로 동작
  * `imagePullPolicy`가 생략되었고 이미지 tag가 있지만 `:latest`가 아닐 때 : `IfNotPresent`가 적용된다.
  * `imagePullPolicy : Never` : 이미지가 local에 있다고 가정. 이미지를 pull하지 않음

> Note : container가 항상 이미지의 같은 버전을 사용한다는 것을 확실히 하기 위해 **`sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2`**같은 digest를 지정할 수 있다. digest는 이미지의 버전을 유일하게 하는 것으로 사용자가 digest 값을 변경하지 않는이상 Kubernetes에 의해 update되지 않는다.

> Note : production에서 배포시 어떤 이미지가 동작하는지 추적하기 어렵고 적절하게 롤백하기 어렵기 때문에 `:latest` 태그의 사용을 피해야 한다.

> Note : image provider의 캐싱정책은 `imagePullPolicy: Always`를 효과적으로 하도록 한다. 예를 들어 도커에서는 이미지가 존재한다면 모든 이미지 레이어가 캐시되고 이미지 다운로드가 필요없기 때문에 pull 시도가 빠르다.

## Using kubectl

* `kubectl apply -f <directory>`를 사용하라.
  * 이는 `<directory>`내에서 `apply`할 수 있는 `.yaml`, `.yml`, `json`파일로 된 모든 Kubernetes configuration을 검색한다.
* `get`, `delete` operation에 대해 특정한 오브젝트의 이름이 아닌 label selector를 사용하라.
  * 이 section에 대해 [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)와 [using labels effectively](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively)를 참고하라.
* `kubectl run`과 `kubectl expose`를 사용하여 빠르게 single-container Deployments와 Services를 생성하라.
  * 예시로 [Use a Service to Access an Application in a Cluster](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)를 참고하라.

