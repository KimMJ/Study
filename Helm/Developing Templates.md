# The Chart Template Developer's Guide

이 가이드는 헬름 차트 템플릿에 대한 소개와 템플릿 언어에서 강조할 사항을 포함한다.

템플릿은 쿠버네티스가 알아들을 수 있는 YAML 포멧을 가진 리소스에 대한 설명인 manifest 파일을 생성한다. 우리는 어떻게 템플릿이 구성되어 있는지 살펴볼 것이며 어떻게 그것들이 사용되는지, 어떻게 Go 템플릿으로 쓰이는지, 어떻게 디버깅을 할 것인지 알아볼 것이다.

이 가이드는 다음의 컨셉에 초점을 둘 것이다.

* 헬름 템플릿 언어
* values의 사용
* 템플릿을 사용하는 테크닉

이 가이드는 헬름 템플릿 언어의 내/외부를 학습하는 데 초점을 두고 있다. 또 소개 자료들, 예시 및 모범사례를 제공한다.

# Getting Started with a Chart Template

가이드의 이 섹션에서 우리는 차트를 생성하고 첫번째 템플릿에 추가할 것이다. 여기서 생성된 차트는 나머지 가이드에서 계속해서 사용될 것이다.

계속하기 전에 헬름 차트를 짧게 알아보자.

## Charts

Charts Guide에 설명되어 있듯이 헬름 차트는 다음과 같이 구성되어 있다.

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/` 디렉토리는 템플릿 파일들을 위한 디렉토리이다. 틸러가 차트를 측정할 때 `templates/` 내의 모든 파일들은 템플릿 렌더링 엔진으로 보내진다. 틸러는 그런 템플릿들의 결과를 취합하고 쿠버네티스로 전송한다.

`values.yaml` 파일은 또한 템플릿에서 중요한 파일이다. 이 파일은 차트의 기본 값들을 가지고 있다. 이런 값들은 `helm install`이나 `helm upgrade`를 하는 동안 유저에 의해서 오버라이딩 될 수 있다.

`Chart.yaml` 파일은 차트에 대한 설명을 가지고 있다. 이를 템플릿 내에서 접근할 수 있다. `charts/` 디렉토리는 다른 차트들도 가질 수 있다. 이를 서브차트라고 부른다. 이 가이드의 나중에는 템플릿 렌더링 시 어떻게 동작하는지 알아볼 것이다.

## A Starter Chart

이 가이드에서 우리는 `mychart`라고 불리는 간단한 차트를 생성할 것이다. 그리고 그 차트에서 몇가지 템플릿을 생성해 볼 것이다.

### A Quick Glimpse of  `mychart/templates/`

`mychart/templates/` 디렉토리를 보게 되면 이미 몇가지 파일들이 있는 것을 알 수 있다.

* `NOTES.txt` : 차트에 대한 "help text"이다. 이는 유저가 `helm install`을 실행할 때 출력될 것이다.
* `deployment.yaml` : 쿠버네티스 deployment를 생성하는 기본적인 manifest이다.
* `service.yaml` : deployment의 service endpoint를 생성하기 위한 기본적인 manifest이다.
* `_helpers.ypl` : 차트 전체에서 재사용될 수 있는 템플릿들을 놓는 장소이다.

이제 해야할 일은... 그것들을 전부 지우는 것이다! 그렇게 하면 처음부터 튜토리얼을 진행할 수 있다. 우리는 우리만의 `NOTES.txt`와 `_helpers.ypl`을 생성할 것이다.

```shell
$ rm -rf mychart/templates/*.*
```

프로덕션 등급의 차트를 작성할 때 이런 기본 버전의 차트들이 매우 유용할 수 있다. 따라서 차트를 제작하는 동안 이것들을 지우고 싶어하지 않을 것이다.

## A First Template

첫 템플릿은 `ConfigMap`을 생성하는 것이 될 것이다. 쿠버네티스에서 ConfigMap은 설정 데이터를 담는 간단한 컨테이너이다. 파드와 같이 다른 것들은 ConfigMap의 데이터에 접근할 수 있다.

ConfigMaps가 기본 리소스이기 때문에 굉장히 좋은 시작점을 제공한다.

`mychart/templates/configmap.yaml`이라고 불리는 파일을 생성하는 것부터 시작하자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

> Tip : 템플릿 이름은 엄격한 명명 규칙을 따르지 않는다. 하지만 YAML 파일은 `.yaml` suffix를 사용할 것을, helpers에는 `.tpl`을 사용할 것을 권장한다.

위의 YAML 파일은 최소한의 필드만 가지고 있는 빈약한 ConfigMap이다. 이 파일은 `templates/` 디렉토리 내부에 있기 때문에 템플릿 엔진으로 전송될 것이다.

`templates/` 디렉토리 안에 일반 YAML파일을 넣는 것은 괜찮다. 틸러가 이 템플릿을 읽을 때, 이는 쿠버네티스의 as-is로 전송된다.

이 간단한 템플릿에서 우리는 설치할 수 있는 차트를 가지게 되었다. 설치는 다음과 같이 한다.

```shell
$ helm install ./mychart
NAME: full-coral
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA      AGE
mychart-configmap   1         1m
```

결과는 위와 같고 우리는 ConfigMap이 생성되었음을 볼 수 있다. Helm을 이용하면 우리는 릴리즈와 어떻게 실제 템플릿이 로딩되었는지를 볼 수 있다.

```shell
$ helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

`helm get manifest` 명령어는 릴리즈 이름(`full-coral`)을 사용하고 서버로부터 업로드 된 모든 쿠버네티스 리소스들을 출력한다. 각 파일은 `---`로 YAML 문서의 시작을 알리며 그 다음에 자동으로 생성된 주석이 어떤 템플릿 YAML 파일이 생성되었는지 알려준다.

여기서 우리는 YAML 데이터가 우리가 `configmap.yaml` 파일에 적었던 것과 정확하게 같음을 알 수 있다.

이제 우리는 릴리즈를 삭제할 수 있다: `helm delete full-coral`

### Adding a Simple Template Call

`name:`을 리소스에 하드코딩하는것은 안좋은 예시이다. 이름은 릴리즈별로 유일해야한다. 따라서 릴리즈 이름을 삽입함으로써 name 필드를 생성하고 싶을 것이다.

> Tip : `name:` 필드는 DNS 시스템의 제약때문에 63글자로 제한되어 있다. 이 이유로 릴리즈 이름은 53글자로 제한된다. 쿠버네티스 1.3과 그 이전버전은 24글자로만 제한된다. (따라서 name으로 14글자만 사용할 수 있다.)

`configmap.yaml`을 적절하게 바꾸어 보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

가장 큰 변화는 `name:` 필드의 값이다. 이는 `{{ .Release.Name }}-configmap`으로 변경되었다.

> 템플릿 지시자는 `{{`와 `}}`로 감싸져야 한다.

템플릿 지시자 `{{ .Release.Name }}`은 릴리즈의 이름을 템플릿 안에 주입한다. 템플릿으로 전달된 값은 namespaced objects라고 생각될 수 있다. 점(`.`)이 각 네임스페이스화된 엘리먼트들을 구분지어준다.

`Release` 이전의 제일 처음 점은 이 스코프의 가장 상위의 네임스페이스로부터 시작함을 알려준다.(나중에 스코프에 대해서 간단히 알아본다.) 따라서 `.Release.Name`을 "상위 네임스페이스에서 시작하여, `Release` 오브젝트를 찾고, 그 안에서 `Name`이라고 불리는 오브젝트를 찾아라."라고 해석할 수 있다.

`Release` 오브젝트는 헬름의 빌트인 오브젝트 중 하나이며 나중에 더 자세히 알아볼 것이다. 하지만 지금은 이것이 틸러가 우리의 릴리즈에 부여하는 릴리즈 이름을 출력한다고 아는 것만으로도 충분하다.

이제 우리의 리소스가 설치가 되면 우리는 즉시 템플릿 지시자의 결과를 볼 수 있다.

```shell
$ helm install ./mychart
NAME: clunky-serval
LAST DEPLOYED: Tue Nov  1 17:45:37 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                      DATA      AGE
clunky-serval-configmap   1         1m
```

`RESOURCES` 섹션에서, 이름이 기존 `mychart-configmap`에서 `clunky-serval-configmap`으로 된 것에 주목하라.

`helm get manifest clunky-serval`을 통해서 전체 생성된 YAML을 볼 수 있다.

여기에서 우리는 우리가 본 가장 간단한 템플릿은: `{{`와 `}}`로 감싸진 템플릿 지시자를 가지고 있는 YAML 파일이다. 다음 파트에서 우리는 템플릿에 대해서 자세히 알아볼 것이다. 넘어가기 전에, 템플릿을 빠르게 생성할 수 있는 트릭을 알아볼 것이다: 템플릿 렌더링을 테스트하려 하면서 실제로는 아무것도 설치하고 싶지 않을 때, `helm install ./mychart --debug --dry-run`을 실행할 수 있다. 이는 차트를 틸러 서버에 전달하여 템플릿을 렌더링하게 한다. 차트를 설치하는 대신에 렌더링 도니 템플릿을 리턴하여 결과값으로 볼 수 있게 된다.

```shell
$ helm install ./mychart --debug --dry-run
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   goodly-guppy
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"
```

`--dry-run`을 사용하는 것은 코드를 테스트 하는것을 더욱 쉽게 해주지만, 쿠버네티스가 사용자가 생성한 템플릿을 받아들일수 있을지는 확인시켜주지 않는다. `--dry-run`이 제대로 동작했다고 해서 차트가 잘 설치될 것이라고 생각하면 안된다.

다음 몇가지 섹션에서 우리는 여기서 정의한 기본 차트를 사용할 것이고 헬름 템플릿 언어에 대해서 자세히 탐구해볼 것이다. 빌트인 오브젝트부터 시작해 보자.

# Built-in Objects

오브젝트는 템플릿 엔진으로부터 템플릿으로 전달이 된다. 그리고 코드는 오브젝트를 전달할 수 있다.(`with`와 `range`를 사용한 예시를 보게 될 것이다.) 나중에 볼 `list` 기능과 같이 템플릿에서 새로운 오브젝트를 생성하는 몇가지 방법이 있다.

오브젝트는 간단해질 수 있고 오직 하나의 값만을 가진다. 또는 오브젝트는 다른 오브젝트나 함수들도 포함할 수 있다. 예를 들어 `Release` 오브젝트는 몇가지 `Release.Name` 같은 오브젝트를 포함할 수 있고 `Files` 오브젝트는 몇가지 함수들을 가질 수 있다.

이전 섹션에서 우리는 `{{.Release.Name}}`으로 릴리즈의 이름을 템프릿에 삽입했다. `Release`는 템플릿에서 접근할 수 있는 최상위 레벨의 오브젝트 중 하나이다.

* `Release`: 이 오브젝트는 릴리즈 그 자체를 설명한다. 
  * `Release.Name` : 릴리즈의 이름이다.
  * `Release.Time` : 릴리즈한 시간이다.
  * `Release.Namespace` : 릴리즈된 네임스페이스(manifest가 오버라이딩 되지 않았을 경우)
  * `Release.Service` : 릴리징 서비스의 이름(항상 `Tiller`)
  * `Release.Revision` : 이 릴리즈에 대한 revision number. 1에서 시작하여 `helm upgrade` 할 때마다 1씩 증가한다.
  * `Release.IsUpgrade` : 현재의 동작이 upgrade거나 rollback일 때 참이 된다.
  * `Release.IsInstall` : 현재 동장이 install인 경우 참이 된다.
* `Values` : values는 `values.yaml`로부터 템플릿으로 전달된 값이고 사용자에 의해 제공된다. 기본적으로 `Values`는 비어있다.
* `Chart` : `Chart.yaml` 파일의 내용. `Chart.yaml`의 모든 데이터는 여기를 통해 접근가능하다. 예를 들어 `{{.Chart.Name}}-{{.Chart.Version}}`은 `mychart-0.1.0`을 출력할 것이다.
  * 가능한 필드들은 [Charts Guide](https://github.com/helm/helm/blob/master/docs/charts.md#the-chartyaml-file)를 통해 확인 가능하다.
* `Files` : 이는 차트에서 특정한 파일들이 아닌 모든 파일들에 대해 접근할 수 있도록 한다. 이를 access 템플릿에 사용할 수 없더라도 이를 통해 차트 안에서 다른 파일들에 접근할 수 있다. 더 자세한 사항은 Accessing Files 섹션을 보아라.
  * `Files.Get`은 이름으로 파일을 가지고 오는 함수이다. (`.Files.Get config.ini`)
  * `Files.GetBytes`는 파일의 내용을 스트링 대신 바이트의 배열로 가지고 오는 함수이다. 이미지 같은 것에 대해서는 이게 유용하다.
* `Capabilities` : 쿠버네티스 클러스터가 지원하는 기능이 무엇인지에 대한 정보를 제공한다.
  * `Capabilities.APIVersions`는 버전의 집합이다.
  * `Capabilities.APIVersions.Has $version`은 버전(예: `batch/v1`)이나 리소스(예: `apps/v1/Deployment`)가 클러스터에서 가능한지를 알려준다. 리소스는 헬름 v2.15 이전에는 지원하지 않음에 유의하라.
  * `Capabilities.KubeVersion`은 쿠버네티스 버전을 확인할 수 있도록 한다. 이는 다음의 값들을 가지고 있다: `Major`, `Minor`, `GitVersion`, `GitCommit`, `GitTreeState`, `BuildDate`, `GoVersion`, `Compiler`, `Platform`
  * `Capabilities.TillerVersion`은 틸러의 버전을 확인할 수 있게 한다. 이는 다음의 값들을 가지고 있다: `SemVer`, `GitCommit`, `GitTreeState`
* `Template` : 현재 실행중인 템플릿에 대한 정보를 가지고 있다.
  * `Name` : 현재 템플릿에 대한 네임스페이스화된 파일 경로이다. (예: `mychart/templates/mytemplate.yaml`)
  * `BasePath` : 현재 차트에 대한 템플릿 디렉토리의 네임스페이스화된 경로이다. (예: `mychart/template`)

이 값은 어느 최상위 템플릿에서도 사용 가능하다. 나중에 볼 수 있듯이 이는 반드시 어디에서나 사용할 수 있다는 것을 의미하는 것은 아니다.

빌트인 values는 항상 대문자로 시작한다. 이는 Go의 네이밍 규칙을 따른 것이다. 사용자가 고유의 이름을 생성했을 때 팀에 맞는 명명 규칙을 사용하는 것은 자유룝다. 헬름 차트 팀과 같은 어떤 팀은 빌트인과 로컬 네임을 구별하기 위해 항상 소문자로 이름을 시작한다. 이 가이드에서는 이 명명규칙을 따르도록 하겠다.

# Values Files

이전 섹션에서 헬름 템플릿이 제공하는 빌트인 오브젝트에 대해서 알아보았다. 이 빌트인 오브젝트 중 하나는 `Values`이다. 오브젝트는 차트에서 values에 접근할 수 있도록 전달해준다. 그 내용은 다음의 네가지 소스로부터 가져온다.

* 차트 내의 `values.yaml`파일
* 이것이 서브차트라면, 부모 차트의 `values.yaml`파일
* `helm install`이나 `helm upgrade`의 `-f` 플래그로 전달되는 values 파일(`helm install -f myvals.yaml ./mychart`)
* `--set`으로 전송된 개별 파라미터들(`helm install --set foo=bar ./mychart`같은 것들)

위의 리스트들은 순서가 지정되어있다: `values.yaml`이 기본으로 사용되고 부모의 `values.yaml`에 의해서 오버라이딩 된다. 그 후 사용자가 제공하는 values 파일로 오버라이딩 되고 그 후에 `--set` 파라미터에 의해서 오버라이딩 된다.

Values 파일은 순수 YAML 파일이다. `mychart/values.yaml`을 수정하여 우리의 ConfigMap 템플릿을 고쳐보자.