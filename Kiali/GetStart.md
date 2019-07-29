# All about Kiali



### 설치

Kiali는 OpenShift나 Kubernetes 클러스터 환경에 설치될 수 있다.

Istio에서 깔린 Kiali는 최신버전이 아닐 수 있으며, 추후에 최신버전으로 수동 업데이트가 가능하다.



#### Kiali CR(Custom Resource)의 생성 또는 수정

Kiali Operator는 Kiali CR을 바라본다. Kiali CR의 생성, 업데이트, 삭제는 Kilai Operator의 설치, 업데이트, 삭제를 유발한다. 





-------------

### Login Options

Kiali는 세가지 login option이 있다.

**login**: 이 옵션은 유저가 Kiali로 username, password를 통해 접속하도록 한다. Kubernetes를 사용시 가장 기본 옵션이다.

**anonymous** : 이 옵션은 로그인을 하지 않는다. 유저는 로그인 페이지에 바로 Kiali로 접속하게 되고 따로 자격을 증명하지 않는다.

openshift : openshift로 배포시 할 수 있는 옵션. 생략.

* anonymous 옵션은 Kiali를 보안에 취약하게 한다.

만약 설치 시 *login*으로 설정을 해 두었다면, 감춰진 Kiali login 인증 정보가 배포시 필요하다. 이 때, install script는 username과 암호를 필요로 한다. install script는 이런 정보를 Kiali가 설치된 네임스페이스에 secret하게 저장된다.



### Namespace 관리

#### 접근 가능한 namespace

Kiali CR(custom resource)은 Kiali Operator에게 어떤 namespace에 접근할 수 있는지 알려준다. 이것은 CR에 depolyment 섹션에 accessible_namespaces로 정의가 된다.

specified namespace란 Kiali에 의해 보여지는 service mesh components를 가지고 있는 것이다. 각각의 list entry는 operator가 볼 수 있는 namespace를 정규식으로 매칭한다. 만약 default를 만들지 않을 경우 반드시 무시되어야 하는 경우인 몇개의 namespace를 제외하고 아니면 모든 namespace에 접근이 가능하게 된다.

#### 접근 불가능한 namespace

마찬가지로 exclude 설정을 통해서 Kiali Operator가 접근하면 안되는 namespace를 설정할 수 있다.

