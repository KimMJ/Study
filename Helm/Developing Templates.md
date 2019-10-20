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

`values.yaml`의 기본 값들을 지우고 하나의 파라미터만 남겨보자.

```yaml
favoriteDrink: coffee
```

이제 템플릿을 다음과 같이 써보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

마지막 줄로 `favoriteDrink`를 통해 `Values`: `{{.Values.favoriteDrink}}`으로 접근 가능하다.

어떻게 이것이 렌더링 되는지 확인해보자.

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   geared-marsupi
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: geared-marsupi-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

`favoriteDrink`가 `values.yaml`에 `coffee`로 설정되어있기 때문에 템플릿 안에서 그 값이 보이게 된다. 이를 `helm install`을 호출할 때 `--set` 플래그를 통해 간단하게 오버라이딩 할 수 있다.

```shell
helm install --dry-run --debug --set favoriteDrink=slurm ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   solid-vulture
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

`--set`은 `values.yaml`보다 더 우선순위를 가진다. 따라서 템플릿은 `drink: slurm`을 생성하였다.

Values 파일은 하나 이상의 구조화된 내용을 가질 수도 있다. 예를 들어 `values.yaml`파일에서 `favorite` 섹션은 몇가지 키를 더할 수도 있다.

```yaml
favorite:
  drink: coffee
  food: pizza
```

이제 템플릿을 약간 변경해보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

이런 방식으로 데이터를 만드는 것은 가능하지만 최대한 평탄하게 트리를 얕게 하는 것을 추천한다. 서브차트에 할당된 값을 보면 어떻게 값들이 트리 구조에서 사용되는지를 알게될 것이다.

## Deleting a Default Key

기본 값들에서 키를 삭제하려고 한다면 그 키를 `null`로 오버라이딩 하면 된다. 그러면 헬름은 그 키를 병합된 values에서 삭제한다.

예를 들어 stable Drupal 차트가 liveness probe를 설정할 수 있도록 지원하고 사용자가 커스텀 이미지를 설정한다고 해보자. 기본값들은 다음과 같다.

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

livenessProbe 핸들러가 `httpGet`대신 `--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`를 사용하여 `exec`하도록 오버라이딩하려하면 헬름은 기본값과 오버라이딩 된 키들을 합치고 다음과 같은 YAML을 생성한다.

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

하지만 쿠버네티스는 둘 이상의 livenessProbe 핸들러가 선언되면 안되기 때문에 실패할 것이다. 이를 해결하기 위해 헬름이 `livenessProbe.httpGet`을 null로 설정하여 삭제하도록 지시할 수 있다.

```sh
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

여기에서 우리는 몇가지 빌트인 오브젝트를 보았고 이 정보들을 템플릿에 주입시켜보았다. 이제 우리는 템플릿 엔진의 다른 면들을 보게 될 것이다: 함수와 파이프라인

# Template Functions and Pipelines

이제까지 템플릿에 어떻게 정보들을 놓는지 알아보았다. 하지만 그 정보는 수정되지 않은 상태로 템플릿에 주입된다. 가끔 받은 데이터를 사용할 수 있게 수정하는 작업이 필요할 수 있다.

모범 사례에서부터 시작해보자: `.Values` 오브젝트로부터 스트링을 템플릿에 주입할 때 이 스트링에 따옴표를 추가하고 싶을 수 있다. 이를  템플릿 지시사에서 `quote` 라고 부르는 함수를 호출할 수 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

템플릿 함수는 `functionName arg1 arg2 ...`의 문법을 따른다. 위의 일부분을 보면 `quote .Values.favorite.dirnk`는 `quote`함수를 호출하고 하나의 argument를 전달한다.

헬름은 60개가 넘는 함수를 가지고 있다. 몇개는 Go 템플릿 언어 그 자체에 정의된 것이다. 다른 대부분의 것들은 Spring template library에서 가지고 온 것들이다. 예시를 통해 그에 대해 많은 것들을 알아보도록 하겠다.

> "Helm template language"가 헬름에만 특화된 언어라고 말하고는 있지만 이는 사실 Go 템플릿 언어에다가 추가 함수들, 템플릿에 특정 오브젝트를 노출시키는 다양한 wrapper들을 포함한 것이다. 템플릿에 대해 학습할 때 Go 템플릿에 많은 노력을 들이는 것은 도움이 될 것이다.

## Pipelines 

템플릿 언어의 강력한 기능들 중 하나는 pipeline이라는 컨셉이다. UNIX의 컨셉으로부터 따와서 파이프라인은 일련의 변환을 간결하게 표현하기 위해 일련의 커맨드들을 연결시키는 툴이다. 다시말해서 파이프라인은 연속적인 것들을 실행하는 효과적인 방법이다. 위의 예시를 파이프라인을 통해 다시한번 작성해보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

이 예시에서 `quote ARGUMENT` 를 호출하는 대신에 순서를 바꾸었다. 우리는 argument를 파이프라인 (`|`)을 통해 함수로 보낸다: `.Values.favorite.drink | quote`. 파이프라인을 이용하여 우리는 몇가지 함수를 연결할 수 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

> 순서를 뒤바꾸는 것은 템플릿에서 자주 있는 예시이다. `.val | quote`를 `quote .val`보다 더 많이 보게 될 것이다. 다른 예시도 괜찮다.

이를 확인해보면 템플릿은 다음과 같이 생성한다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

원래 `pizza`였던 것이 `"PIZZA"`로 변경되었음을 확인할 수 있다.

arguments를 파이프라인을 이용해서 다룰 때 첫번째의 결과(`.Values.favorite.drink`)가 함수의 마지막 argument로 전달이 된다. 위의 drink 예시를 두개의 argument를 가지는 함수로 변경해보았다: `repeat COUNT STRING`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

`repeat` 함수는 주어진 스트링을 주어진 숫자만큼 반복하여 출력할 것이고 다음과 같은 결과가 나온다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

## Using The `Default` Function

템플릿 안에서 자주 사용되는 함수중 하나는 `default` 함수이다: `default DEFAULT_VALUE GIVEN_VALUE`. 이 함수는 value가 생략되었을 때 사용자가 템플릿 내에서 기본 값을 설정할 수 있도록 해준다. drink 예시를 다음과 같이 수정해보도록 하자.

```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

이를 그냥 실행해보면 `coffee`라는 값을 얻을 수 있다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

이제 favorite drink 설정을 `values.yaml`에서 삭제해보자.

```yaml
favorite:
  #drink: coffee
  food: pizza
```

이제 다시 `helm install --dry-run --debug ./mychart`를 실행해보면 이런 YAML을 생성할 것이다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

실제 차트에서는 모든 static default values가 values.yaml안에 있어야 하고 `default` 명령어를 다시 사용하는 일은 없어야 한다. 하지만 `default` 명령어는 values.yaml에는 없는 값을 계산하는 데 완벽하다. 예를 들어

```yaml
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

어떤 곳에서는 `if` 조건문이 `default`보다 더 좋게 방어를 할 것이다. 이에 대해서는 다음 섹션에서 보도록 하자.

템플릿 함수와 파이프라인은 정보를 변형하고 YAML에 이를 삽입하는 데 매우 좋다. 하지만 때때로 단순한 스트링을 삽입할 때 보다 약간 복잡한 몇가지 템플릿 로직을 추가할 필요가 있다. 다음 섹션에서 우리는 템플릿 언어에 의해 제공되는 제어 구조(control structures)에 대해서 알아보도록 하겠다.

## Operators Are Functions

오퍼레이터들은 boolean 값을 리턴하는 함수처럼 구현되어 있다. `eq`, `ne`, `lt`, `gt`, `and`, `or`, `not` 등의 operator는 statement 앞에 와서 함수를 사용하듯이 파라미터가 뒤따라온다. 여러개의 operation을 묶을려면 각 함수를 괄호로 감싸면 된다.

```yaml
{{/* include the body of this if statement when the variable .Values.fooString exists and is set to "foo" */}}
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}


{{/* include the body of this if statement when the variable .Values.anUnsetVariable is set or .values.aSetVariable is not set */}}
{{ if or .Values.anUnsetVariable (not .Values.aSetVariable) }}
   {{ ... }}
{{ end }}
```

이제 우리는 함수와 파이프 라인에서 conditions, loops, scope modifiers가 있는 flow control로 전환할 수 있다.

# Flow Control

템플릿 용어에서 "action"이라고 불리는 control structure는 템플릿 제작자가 템플릿 생성의 플로우를 컨트롤 할 수 있도록 한다. 헬름의 템플릿 언어는 다음의 control structure를 제공한다.

* conditional block을 생성하기 위한 `if`/`else` 
* 스코프를 지정하기 위한 `with`
* "for-each" 스타일의 루프를 제공하기 위한 `range`

여기에 더불어 named template segments를 정의하는 데 몇가지 action을 제공한다.

* `define`은 템플릿 안에서 새로운 named template을 정의한다.
* `template`은 named template을 임포트한다.
* `block`은 특정 종류의 템플릿 구역으로 채워넣을 수 있다.

이 섹션에서 `if`, `with`, `range`에 대해서 이야기해 볼 것이다. 다른 것들은 이 가이드의 "Named Templates" 섹션에서 알아보도록 하겠다.

## If/Else

우리가 처음으로 본 control structure는 텍스트 블록을 조건부로 포함시키는 것이다. 이는 `if`/`else` 블록으로 할 수 있다.

조건부의 기본 구조는 다음과 같다.

```yaml
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

이제 우리는 values가 아닌 pipelines에 대해서 이야기 해 보고자 한다. control structure가 단순히 값을 계산하기 보다는 전체 파이프 라인을 실행할 수 있다는 것을 명확하게 하기 위해서이다.

파이프라인은 다음과 같을 때 false이다.

* boolean이 false일 때
* 0일 때
* 빈 스트링일 때
* `nil`일 때 (empty이거나 null)
* 빈 collection(`map`,`slice`,`tuple`,`dict`,`array`)

이와 다른 경우들에 대해서는 조건이 참으로 판명되고 파이프라인이 실행된다.

ConfigMap에 간단한 조건을 추가해 보도록 하자. drink가 coffee로 설정이 되면 실행할 다른 설정을 추가할 것이다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if and .Values.favorite.drink (eq .Values.favorite.drink "coffee") }}mug:
```

`.Values.favorite.drink`는 반드시 정의 되어 있어야 하고 아니라면 "coffee"와 비교할 때 에러를 발생시킬 것이다. 우리의 지난 예시에서 `drink: coffee`를 주석처리하면 결과에 `mug: true` 플래그가 포함되지 않을 것이다.  `value.yaml`에 다시 추가하면 결과는 다음과 같다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

## Controlling Whitespace

조건문을 보게 되면 템플릿에서 공백이 어떻게 다루어지는지 알 수 있다. 이전의 예시를 보고 이를 더 읽기 쉽게 만들어 보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
```

처음에 이는 괜찮아 보인다. 하지만 템플릿 엔진에 이를 주었을 때 불행하게도 다음과 같은 결과를 보게 된다.

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

어떤 일이 일어났을까? 우리는 공백때문에 올바르지 않은 YAML파일을 생성했다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

`mug`는 올바르지 않게 인덴트되었다. 간단하게 인덴트 하나를 지워서 재실행해보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{end}}
```

이 요청을 보내면 유효한 YAML을 얻게 되지만 약간 웃기게 생성된다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true
```

YAML에 몇가지 빈 줄이 생긴 것을 확인할 수 있다. 왜일까? 템플릿 엔진이 동작하면 `{{`와 `}}`를 지운다. 하지만 whitespace는 그대로 남긴다.

YAML은 whitespace에도 의미를 부여하기 때문에 whitespace을 관리하는 것은 매우 중요하다. 운좋게도 Helm 템플릿은 몇가지 도움을 주는 툴을 가지고 있다.

먼저, 템플릿 정의의 유연한 문법은 특수 문자로 템플릿 엔진이 whitespace을 지우도록 하게 수정할 수 있다. `{{-`(대쉬와 공백이 추가되었다)는 왼쪽 whitespace를 없앤다는 의미이고 `-}}`는 오른쪽 whitespace를 없앤다는 의미이다. **newline도 whitespace임을 주의하라**

> `-`와 나머지 지시자 사이에 공백이 있어야 한다. `{{- 3 }}`은 "왼쪽 whitespace를 제거하고 3을 프린트해라"라는 의미이지만 `{{-3}}`dms "-3을 프린트하라"는 의미이다.

이 문법을 사용하여 우리는 템플릿에서 new line을 제거할 수 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

이것을 확실히 알아보기 위해 위의 예시에서 각 whitespace에 `*`을 넣어서 이 룰에 의해 어떻게 whitespace가 지워지는지 확인해보자. 줄 마지막에 있는 `*`는 newline을 의미하며, 지워질 것이다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}*
**{{- if eq .Values.favorite.drink "coffee"}}
  mug: true*
**{{- end}}
```

이를 생각해서 우리의 템플릿을 Helm을 통해 실행하고 결과를 보자.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clunky-cat-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

chomping 수정자를 사용할 때에는 주의가 필요하다. 실수로 다음과 같이 할 수도 있다.

```yaml
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: true
  {{- end -}}
```

이는 양쪽의 newline을 삭제하기 때문에 `food: "PIZZA"mug: true`를 생성한다.

> 템플릿에서 whitespace에 대해서 자세한 사항은 [Official Go template documentation](https://godoc.org/text/template)를 참조하라.

최종적으로 템플릿 시스템에게 템플릿 지시자를 통해 인덴트를 직접 하는 것보다 어떻게 인덴트를 할지 알려주는 것이 더 편할 수 있다. 이러한 이유로 `indent` 함수를 사용하는 것이 더 유용할 수 있다.(`{{indent 2 "mug: true"}}`)

## Modifying Scope Using `with`

다음 control structure는 `with` action이다. 이는 변수의 스코프를 컨트롤한다. `.`은 현재의 스코프라는 것을 생각해보자. 따라서 `.Values`는 현재 스코프에서 `Values` 오브젝트를 찾는다.

`with`는 `if`문과 비슷하다.

```yaml
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

스코프는 변경될 수 있다. `with`는 현재의 스코프(`.`)를 특정 오브젝트로 변경시켜준다. 예를 들어, `.Values.favorites`를 보도록 하자. 우리의 ConfigMap에서 `.` 스코프를 `.Values.favorites`를 바라보도록 해보자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

(`if` 조건문을 삭제한 것에 유의하라.)

우리는 이제 `.drink`와 `.food`로 참조가 가능하다. 이는 `with` 문으로 `.`이 `.Values.favorite`을 가르키도록 하였기 때문이다. `.`은 `{{ end }}`이후에 이전의 스코프로 돌아온다.

하지만 여기에 주의할 점이 있다. 제한된 스코프 안에서 우리는 부모 스코프의 다른 오브젝트로 접근할 수 없다. 예를 들어 다음은 실패할 것이다.

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```

`.`안에 `Release.Name`가 없기 때문에 에러를 발생시킬 것이다. 하지만 마지막 두 줄의 위치를 서로 바꾸면 원하는 대로 동작할 것이다. 이는 스코프가 `{{- end }}`에 의해서 돌아왔기 때문이다.

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  release: {{ .Release.Name }}
```

`range`에 대해 보고나서 template variables에 대해 살펴본다. 이는 위의 스코프 이슈를 해결하는 하나의 방법이다.

## Looping With The `Range` Action

많은 프로그래밍 언어가 `for` 루프와 `foreach` 루프 또는 비슷한 매카니즘을 제공한다. 헬름 템플릿 언어에서 collection을 iterate하는 방법은 `range` operator를 사용하는 것이다.

시작에 앞서 우리는 `values.yaml`파일에 pizza toppings 리스트를 추가할 것이다.

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

이제 우리는 `pizzaToppings`라는 리스트(템플릿에서는 `slice`라고 표현한다)를 가지게 되었다. 이제 템플릿을 수정하여 이 리스트들을 ConfigMap에 출력할 수 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

`toppings` 리스트에 대해서 자세히 보도록 하자. `range` 함수는 `pizzaToppings` 리스트를 "range over"(iterate한다) 한다. 하지만 흥미로운 일이 발생했다. `with`가 스코프를 `.`으로 설정한 것처럼 `range` 또한 그렇게 한다. 각 루프를 실행할 때 `.`은 현재의 pizza topping이 된다. 즉, 처음에 `.`은 `mushrooms`로 설정이 된다. 두번째 반복에서 `cheese`가 되고 계속 그런식으로 동작한다.

---

<data insert>

---------

이 config map은 이전 섹션에서 다룬 몇가지 테크닉들을 사용한다. 예를 들어 우리는 `$files` 변수를 생성하여 `.Files` 오브젝트를 참조하게 했다. 또한 `list` 함수를 통해서 루프를 돌 리스트를 생성하였다. 그 후 우리는 각 파일의 이름을 출력(`{{.}}: |-`)한 뒤 파일의 내용을 출력(`{{ $files.Get . }}`)하였다.

이 템플릿을 실행하면 세 파일의 내용이 담긴 하나의 ConfigMap이 생성된다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quieting-giraf-configmap
data:
  config1.toml: |-
    message = "Hello from config 1"

  config2.toml: |-
    message = "This is config 2"

  config3.toml: |-
    message = "Goodbye from config 3"
```

## Path Helpers

파일을 사용할 때 파일 경로에 대한 표준 동작들을 사용하는 것은 매우 유용할 것이다. 이를 돕기 위해 헬름은 Go의 많은 [path](https://golang.org/pkg/path/) 패키지들을 임포트하였다. 이들은 고 패키지에서와 같은 이름으로 접근할 수 있지만 첫글자가 소문자이다. 예를 들어 `Base`는 `base`가 되었다.

임포트한 함수는 다음과 같다.

* Base
* Dir
* Ext
* IsAbs
* Clean

## Glob Patterns

차트가 성숙해지면서 아마 파일들을 구성하는 일이 많아질 것이다. 따라서 우리는 `Files.Glob(pattern string)` 메소드를 제공하여 특정 파일을 [glob patterns](https://godoc.org/github.com/gobwas/glob)으로 유연하게 추출할 수 있도록 한다.

`.Glob`은 `Files` 타입을 리턴하기 때문에 사용자는 리턴된 오브젝트로부터 어떤 `Files` 메소드던지 호출할 수 있을 것이다.

예를 들어 다음과 같은 디렉토리 구조를 생각해보자.

```txt
foo/:
  foo.txt foo.yaml

bar/:
  bar.go bar.conf baz.yaml
```

Globs에는 여러개의 옵션을 줄 수 있다.

```yaml
{{ $root := . }}
{{ range $path, $bytes := .Files.Glob "**.yaml" }}
{{ $path }}: |-
{{ $root.Files.Get $path }}
{{ end }}
```

또는

```yaml
{{ $root := . }}
{{ range $path, $bytes := .Files.Glob "foo/*" }}
{{ base $path }}: '{{ $root.Files.Get $path | b64enc }}'
{{ end }}
```

## Configmap And Secrets Utility Functions

(2.0.2 이전 버전에는 존재하지 않는다.)

런타임에 파드로 마운팅하기 위해 파일의 내용을 configmap과 secret 두 곳에 모두 넣고 싶은 경우는 흔한 일이다. 이를 위해 우리는 `Files` 타입에 대한 두가지 유틸리티 메소드를 제공한다.

다음의 추가적인 구성에서 `Glob` 메소드와 함께 이 메소드들을 사용하는 것은 유용하다.

위의 `Glob` 예제에서 주어진 디렉토리를 고려해 보았을 때 다음과 같다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
  {{- (.Files.Glob "foo/*").AsConfig | nindent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
  {{- (.Files.Glob "bar/*").AsSecrets | nindent 2 }}
```

## Encoding

사용자는 파일을 임포트하고 성공적인 전송을 확인하기 위해 base-64로 인코딩 된 템플릿을 가져올 수 있다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  token: |-
    {{ .Files.Get "config1.toml" | b64enc }}
```

위의 예시는 전과 같은 `config1.toml` 파일을 가지고 오고 이를 인코딩한다.

```yaml
# Source: mychart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucky-turkey-secret
type: Opaque
data:
  token: |-
    bWVzc2FnZSA9IEhlbGxvIGZyb20gY29uZmlnIDEK
```

## Lines

가끔 템플릿에 있는 파일의 각 줄에 접근할 필요가 있을 수 있다. 우리는 `Lines` 메소드라는 편리한 기능을 제공한다.

```yaml
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```

현재 `helm install`을 하는 동안 외부의 파일을 전달하는 방법은 없다. 따라서 유저에게 데이터를 제공하라고 요청하려면 반드시 `helm install -f`나 `helm install --set`으로 로딩되어야 한다.

이 토론은 헬름 템플릿을 작성하기 위한 툴과 테크닉에 대한 설명을 마무리한다. 다음 섹션에서 우리는 이 `template/NOTES.txt`라는 특수한 파일을 사용하는지에 대해 알아보고 차트의 사용자에게 post-installation instructions를 보낼 수 있도록 해보자.

# Creating A NOTES.txt File

이 섹션에서 우리는 차트 사용자에게 지시를 내리는 헬름의 툴에 대해서 알아볼 것이다. `helm install`이나 `helm upgrade` 마지막에서 헬름은 유저에게 도움을 줄 수 있는 정보들을 출력할 수 있다. 이 정보는 템플릿을 사용하여 커스터마이징이 가능하다.

차트에 노트를 설치하려면 단순히 `template/NOTES.txt` 파일을 추가하면 된다. 이 파일은 평문이지만 템플릿처럼 처리가 되며 일반적인 템플릿 함수들과 오브젝트가 사용 가능하다.

다음과 같은 간단한 `NOTES.txt` 파일을 생성해 보자.

```gotpl
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get {{ .Release.Name }}
```

이제 `helm install ./mychart`를 실행해보면 아래에 메시지가 보일 것이다.

```
RESOURCES:
==> v1/Secret
NAME                   TYPE      DATA      AGE
rude-cardinal-secret   Opaque    1         0s

==> v1/ConfigMap
NAME                      DATA      AGE
rude-cardinal-configmap   3         0s


NOTES:
Thank you for installing mychart.

Your release is named rude-cardinal.

To learn more about the release, try:

  $ helm status rude-cardinal
  $ helm get rude-cardinal
```

`NOTES.txt`를 이렇게 사용하는 것은 유저에게 그들이 새로 설치한 차트에 대한 자세한 정보를 제공할 수 있다. `NOTES.txt` 파일을 생성하는 것은 강력추천되지만 필수는 아니다.

# SubCharts and Global Values

이제까지 우리는 하나의 차트로만 작업을 해왔다. 하지만 차트는 서브차트라고 부르는 dependency를 가질 수 있다. 그리고 그 서브차트 또한 고유의 value와 template을 가지고 있다. 이 섹션에서 우리는 subcart를 생성하고 템플릿 내에서 value에 접근하는 다른 방법들을 알아볼 것이다.

코드로 들어가기 전에 서브차트에 대해서 알아야할 몇가지 중요한 정보가 있다.

1. 서브 차트는 "stand-alone"으로 취급이 된다. 즉, 서브차트는 절대 부모 차트에 외부적인 의존성이 없다.
2. 그 이유로 서브차트는 부모의 value에 접근할 수 없다.
3. 부모 차트는 서브차트의 value를 오버라이딩 할 수 있다.
4. 헬름은 모든 차트에서 접근할 수 있는 global value라는 개념을 가지고 있다.

이번 섹션에서 예시들을 보며 이런 많은 개념들이 명확해질 것이다.

## Creating a Subchart

이번 연습에서 우리는 이 가이드의 처음 부분에서 만든 `mychart/` 차트와 함께 시작할 것이다. 그리고 그 안에 새로운 차트를 추가할 것이다.

```console
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*.*
```

이전에 했던것과 같이 우리는 모든 base template을 삭제하여 처음부터 시작할 수 있도록 하였다. 이 가이드에서 우리는 dependency를 관리하는 측면이 아니라 어떻게 템플릿이 동작하는지에 대해 포커스를 맞출 것이다. 하지만 [Charts Guide](https://helm.sh/docs/chart_template_guide/#../charts)는 어떻게 서브차트가 동작하는지에 대해 더 많은 정보가 있다. 

## Adding Values and a Template to the Subchart

다음으로 `mysubchart` 차트에 간단한 차트와 values 파일을 만들 것이다. `mychart/charts/mysubchart` 안에는 이미 `values.yaml`파일이 존재한다. 우리는 이를 다음과 같이 설정할 것이다.

```yaml
dessert: cake
```

그 후, 새로운 ConfigMap 템플릿을 `mychart/charts/mysubchart/templates/configmap.yaml`에 추가해보도록 하자.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

모든 서브차트는 stand-alone chart이기 때문에 우리는 `mysubchart`를 그 자체로 테스트해볼 수 있다.

```console
$ helm install --dry-run --debug mychart/charts/mysubchart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart/charts/mysubchart
NAME:   newbie-elk
TARGET NAMESPACE:   default
CHART:  mysubchart 0.1.0
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: newbie-elk-cfgmap2
data:
  dessert: cake
```

## Overriding Values of a Child Chart

원래 차트에서 `mychart`는 이제 `mysubchart`의 부모 차트가되었다. 이 관계는 전적으로 `mysubchart`가 `mychart/charts`에 있기 때문에 생긴 것이다.

`mychart`가 부모이기 때문에 우리는 `mychart`의 설정을 지정할 수 있고 설정을 `mysubchart`로 보낼 수도 있다. 예를 들어 우리는 `mychart/values.yaml`을 다음과 같이 설정할 수 있다.

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

마지막 두줄을 주목하라. `mysubchart` 섹션에서의 어떤 지시자던지 `mysubchart` 차트로 보내질 것이다. 따라서 `helm install --dry-run --debug mychart`를 실행하면 우리가 볼 수 있는 `mysubchart` ConfigMap은 다음과 같을 것이다.

```yaml
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-bee-cfgmap2
data:
  dessert: ice cream
```

상위 레벨의 value가 서브차트의 value를 오버라이딩하였다.

여기서 자세히 알아야 할 사항이 있다. 우리는 `mychart/charts/mysubchart/templates/configmap.yaml` 의 어떤 부분도 `.Values.mysubchart.dessert`를 가르키고 있던 것을 변경하지 않았다. 그 템플릿을 유지하며 value는 여전히 `.Values.dessert`에 존재한다. 템플릿 엔진이 value를 전달하는 동안 스코프를 설정한다. 따라서 `mysubchart` 템플릿에 대해서 `mysubchart`로 지정된 value만이 `.Values`에서 사용가능하다.

그러나 때로는 특정한 값들이 모든 템플릿에서 사용가능할 수 있도록 하고 싶을 것이다. 이는 global chart values를 사용하여 가능하다.

## Global Chart Values

global value는 어느 차트나 서브차트에서 정확히 같은 이름으로 접근할 수 있는 value이다. global은 명확한 정의가 필요하다. global이 아닌 것을 global처럼 사용할 수 없다.

Value data type은 global value를 설정할 수 있는 `Values.global`이라고 부르는 reserved section이 있다. `mychart/values.yaml` 파일에서부터 시작하자.

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

global이 동작하는 방식 때문에 `mychart/templates/configmap.yaml`과 `mychart/charts/mysubchart/templates/configmap.yaml`은 `{{ .Values.global.salad }}`로 접근이 가능해야한다.

`mychart/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  salad: {{ .Values.global.salad }}
```

`mychart/charts/mysubchart/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

이제 dry run install을 해보면 둘다 같은 value를 출력함을 알 수 있다.

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-configmap
data:
  salad: caesar

---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```

global은 이런 정보를 전달할 때 유용하지만 적절한 템플릿이 global을 사용하도록 설정 되어있음을 확실히 하는 계획이 필요하다.

## Sharing Templates With Subcharts

부모 차트와 서브차트는 템플릿을 공유할 수 있다. 어느 차트에서 정의된 블록이든지 다른 차트에서 사용될 수 있다.

예를 들어 다음과 같은 간단한 템플릿을 정의할 수 있다.

```yaml
{{- define "labels" }}from: mychart{{ end }}
```

어떻게 템플릿에서 labels가 글로벌로 공유되는지 회상해보자. 따라서 `labels` 차트는 다른 어느 차트에서든지 include 될 수 있다.

차트 개발자가 `inlcude`와 `template`중 선택권을 가지고 있는데, `include`를 사용하는 것의 이점 중 하나는 `include`는 동적으로 템플릿을 참조할 수 있다는 것이다.

```yaml
{{ include $mytemplate }}
```

위는 `$mytemplate`을 역참조한다. `template` 함수는 반대로 string literal만 사용할 수 있다.

## Avoid Using Block

Go 템플릿 언어는 `block` 키워드를 제공한다. 이는 개발자가 나중에 오버라이딩 될 수 있는 default implementation을 제공할 수 있도록 한다. 헬름 차트에서 블록은 오버라이딩 하는 데 최선의 방안이 아니다. 왜냐하면 동일한 블록에 대해 복수의 implementaion이 제공된다면 선택되는 것은 예측이 불가능하기 때문이다.

추천하는 것은 `include`를 사용하는 것이다.

# Debugging Templates

템플릿을 디버깅하는 것은 간단하지는 않은 작업이다. 템플릿을 렌더링하는 것은 헬름 클라이언트가 아니라 틸러 서버에서 이루어지기 때문이다. 그리고 렌더링된 템플릿은 YAML포멧과 다르다는 이유로 거부할 수 있는 쿠버네티스 API 서버로 전송이 된다.

디버그를 도와주는 몇가지 명령어가 있다.

* `helm lint`는 차트가 모범 사례를 잘 따르는지를 검사해 줄 수 있는 도움을 주는 툴이다.
* `helm install --dry-run --debug` : 우리는 이미 약간의 트릭을 보았다. 사용자의 템플릿을 렌더링해서 manifest 파일 결과를 보여주는 좋은 방법이다.
* `helm get manifest` : 서버에 설치된 템플릿을 볼 수 있는 좋은 방법이다.

YAML이 파싱때문에 실패하게 되었지만 무엇이 생성되었는지 보고싶으면 YAML을 받는 쉬운 방법은 문제되는 섹션을 템플릿에서 주석처리하고 다시 `helm install --dry-run --debug`를 실행하는 것이다.

```yaml
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

위의 것은 주석처리가 된 상태로 렌더링하여 리턴된다.

```yaml
apiVersion: v1
# some: problem section
#  "bar"
```

이는 YAML의 파싱 에러가 없이 생성된 내용을 볼 수 있는 빠른 방법이다.

# Wrapping Up

이 가이드는 차트 개발자이며 헬름 템플릿 언어를 사용하는 데 이해가 깊은 사람에게 제공되기 위한 목적이다. 이 가이드는 템플릿 개발의 테크닉적인 요소에 초점이 맞춰져 있다.

이 가이드에는 실제 일상적인 차트 개발의 측면에서 다뤄지지 않은 것들이 많이 있다. 몇가지 유용한 참조 문서들이 있다. 이들은 새로운 차트를 개발할 때 도움을 줄 것이다.

* [Helm Charts project](https://github.com/helm/charts)는 차트의 필수 요소이다. 이 프로젝트는 차트 개발의 모범 사례들의 표준을 제시한다.
* 쿠버네티스 [Documentation](https://kubernetes.io/docs/home/)은 ConfigMap과 Secrets에서부터 DaemonSets, Deployments까지 다양한 종류의 리소스 예시들을 제공한다.
* Helm [Charts Guide](https://helm.sh/docs/chart_template_guide/#../charts)는 차트를 사용하는 데의 워크플로우를 설명한다.
* Helm [Chart Hooks Guide](https://helm.sh/docs/chart_template_guide/#../charts_hooks)는 어떻게 lifecycle hook이 생성되는지를 설명한다.
* Helm [Charts Tips and Tricks](https://helm.sh/docs/chart_template_guide/#../chart-development-tips-and-tricks)은 차트를 작성하는 데 유용한 팁들을 제공한다.
* [Sprig documentation](https://github.com/Masterminds/sprig) 문서는 60개가 넘는 템플릿 기능들에 대해서 설명한다.
* [Go template docs](https://godoc.org/text/template)는 템플릿 문법에 대해 자세한 사항을 설명한다.
* [Schelm tool](https://github.com/databus23/schelm)은 차트를 디버깅하는데 유용한 유틸리티이다.

가끔 숙련된 개발자에게 질문을 하고 답을 얻는 것이 더 쉬울 수 있다. 가장 적합한 장소는 [Kubernetes Slack](https://kubernetes.slack.com/) Helm channels이다.

- [#helm-users](https://kubernetes.slack.com/messages/helm-users)
- [#helm-dev](https://kubernetes.slack.com/messages/helm-dev)
- [#charts](https://kubernetes.slack.com/messages/charts)

최종적으로, 이 문서의 에러나 오탈자를 발견하였거나 새로운 컨텐츠를 제안하고자 한다면, 또는 컨트리뷰트를 원할 경우 [The Helm Project](https://github.com/helm/helm)를 방문하라.

# YAML Techniques

이 가이드의 대부분은 템플릿 언어를 작성하는 데 초점이 맞추어져 있다. 여기서 우리는 YAML 포멧에 대해서 알아볼 것이다. YAML은 template authors와 같은 유용한 기능을 가지고 있어서 템플릿을 에러가 발생할 확률이 적게 만들어주고 읽기 쉽도록 해준다.

## Scalars and Collections

[YAML spec](https://yaml.org/spec/1.2/spec.html)에 따르면 두가지 종류의 collection들이 있고 여러개의 scalar 타입이 있다.

collections의 두 타입은 map과 sequence이다.

```yaml
map:
  one: 1
  two: 2
  three: 3

sequence:
  - one
  - two
  - three
```

scalar 값은 개별적인 값이다. (collections와는 다르다.)

### Scalar Types in YAML

Helm은 YAML의 방언과 같다. Helm에서 value의 scalar data type은 리소스 정의에 대한 쿠버네티스의 스키마를 포함한 복잡한 규칙에 의해서 결정이 된다. 그러나 타입을 언급할 때 다음의 규칙들은 참으로 여겨진다.

integer나 float이 unquoted bare word일 때, 이는 numeric type으로 여겨진다.

```yaml
count: 1
size: 2.34
```

하지만 quoted 되었으면 string으로 취급한다.

```yaml
count: "1" # <-- string, not int
size: '2.34' # <-- string, not float
```

booleans에 대해서도 같다.

```yaml
isGood: true   # bool
answer: "true" # string
```

empty value에 대한 단어는 `null`이다. (`nil`이 아니다.)

`port: "80"`은 유효한 YAML이라는 것에 유의하라. 그리고 이는 템플릿 엔진과 YAML 파서를 모두 통과할 수 있다. 그러나 쿠버네티스는 `port`가 integer이길 원하기 때문에 실패한다.

몇가지 케이스에서 YAML node tags를 사용하여 특정한 타입 추론을 강제할 수 있다.

```yaml
coffee: "yes, please"
age: !!str 21
port: !!int "80"
```

위에서 `!!str`은 파서가 `age`가 int처럼 보일지라도 string이라고 말해준다. 그리고 `port`는 quoted되었지만 int로 처리된다.

## Strings in YAML

YAML 문서에 작성한 대부분의 데이터는 string이다. YAML은 string을 표현하는 방법이 한가지 더 있다. 이 섹션에서는 그 방법을 사용하고 어떻게 그것을 이용하는지를 설명할 것이다.

string을 선언하는 데에는 세가지 "inline" 방식이 있다.

```yaml
way1: bare words
way2: "double-quoted strings"
way3: 'single-quoted strings'
```

모든 inline 스타일은 반드시 한 줄이어야 한다.

* bare word는 unquoted이고 escaped되지 않았다. 따라서 사용하는 글자에 대해 주의가 필요하다.
* Double-quoted string은 `\`로 라는 특수문자로 escape할 수 있다. 예를 들어 `"\"Hello\", she said"`라고 작성할 수 있다. `\n`으로 줄바꿈 할 수 있다.
* Single-quoted string은 "literal" string이다. 그리고 `\`을 escaping character로 사용하지 않는다. 유일한 escape sequence는 `''`이다. 이는 `'`으로 풀이된다.

one-line string에 추가적으로 multi-line string도 선언할 수 있다.

```yaml
coffee: |
  Latte
  Cappuccino
  Espresso
```

위의 것은 `coffee`의 value를 `Latte\nCappuccino\nEspresso\n`처럼 single string으로 취급한다.

`|` 뒤의 첫 줄은 반드시 올바르게 인덴트 되어야 함을 유의하라. 우리는 위의 예시에서 이렇게 바꾸어 볼 수 있다.

```yaml
coffee: |
         Latte
  Cappuccino
  Espresso
```

`Latte`는 잘못 인덴트 되었기 때문에 우리는 다음과 같은 에러를 볼 수 있을 것이다.

```
Error parsing file: error converting YAML to JSON: yaml: line 7: did not find expected key
```

위와같은 에러를 방지하기 위해 템플릿에 가짜 "첫 줄"을 넣는 것은 multi-line document에서 안전할 수 있다.

```yaml
coffee: |
  # Commented first line
         Latte
  Cappuccino
  Espresso
```

첫 줄이 무엇이든지, 이는 결과 string에서 보존될 것이다. 따라서 예를들어 당신이 파일의 내용을 ConfigMap에 주입시키고자 할 경우, 주석줄은 그 엔트리를 읽는 것이 어떤 값이든 사용할 수 있도록 한다.

## Controlling Spaces in Multi-line Strings

위의 예시에서 우리는 `|`를 사용하여 multi-line string을 나타내었다. 하지만 우리의 string은 `\n`으로 잘려있는 스트링이다. 만일 우리가 YAML 프로세서가 newline으로 잘리는 것을 제거하도록 하길 원한다면 `|` 뒤에 `-`를 추가하면 된다.

```yaml
coffee: |-
  Latte
  Cappuccino
  Espresso
```

이제 `coffee` value는 `Latte\nCappuccino\nEspresso`가 될 것이다. (`\n`에 의해 잘리지 않는다.)

어떤 때에는 whitespace로 잘리는 것을 보존하고 싶을 것이다. 우리는 이를 `|+`로 표현할 수 있다.

```yaml
coffee: |+
  Latte
  Cappuccino
  Espresso


another: value
```

이제 `coffee`의 value는 `Latte\nCappuccino\nEscpresso\n\n\n`이 된다.

텍스트 블록의 인덴트는 보존이 되고 line break를 보존하는 결과가 된다.

```yaml
coffee: |-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

위의 경우에서 `coffee`는 `Latte\n 12 oz \n 16 oz\nCappuccino\nEspresso`가 된다.

## Indenting and Templates

템플릿을 작성할 때, 템플릿 안에 파일의 내용을 넣고싶을 수 있다. 이전 챕터에서 보았듯이 두가지 방법이 있다.

* `{{ .Files.Get "FILENAME" }}`을 사용하여 차트에서 파일의 내용을 가져온다.
* `{{ include "TEMPLATE" . }}`을 사용하여 템플릿을 렌더링하고 그 내용을 차트 안에 위치시킨다.

파일을 YAML안에 삽입할 때 위에서의 multi-line rules를 이해하는 것이 좋다. 대부분의 경우에 static file을 삽입하는 가장 쉬운 방법은 다음과 같은 방법이다.

```yaml
myfile: |
{{ .Files.Get "myfile.txt" | indent 2 }}
```

어떻게 인덴트를 하였는지 보아라: `indent 2`는 템플릿 엔진에게 "myfile.txt"의 모든 줄이 두개의 공백으로 인덴트 처리되었음을 알려준다. 우리가 그 템플릿 라인을 인덴트하지 않았음을 유의하라. 첫 줄에 인덴트를 주게 되면 두번 인덴트하는 결과를 불러오기 때문이다.

## Folded Multi-line Strings

때로 사용자는 YAML에 multiple line으로 string을 표현하고 싶지만 해석할 때에는 하나의 긴 줄로 다루어지길 원할 수 있다. 이는 "folding"이라고 한다. folded block을 선언하려면 `|`대신 `>`을 사용하면 된다.

```yaml
coffee: >
  Latte
  Cappuccino
  Espresso
```

`coffee`의 값은 `Latte Cappuccino Espresso\n`가 된다. 마지막 줄을 제외한 것들은 공백으로 구분함을 유의하라. 사용자는 whitespace를 folded text marker로 조절할 수 있으며, `>-`는 모든 newline 을 trim시킬 수 있다.

folded syntax에서 텍스트에서 텍스트에 대한 인덴트는 line이 유지되도록 할 것이다.

```yaml
coffee: >-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

위의 예시는 `Latte\n 12 oz\n 16 oz\nCappuccino Espresso`를 출력할 것이다. 공백과 줄바꿈이 여전히 존재함을 유의해라.

## Embedding Multiple Documents in one file

하나 이상의 YAML 문서를 하나의 파일에 위치시킬 수 있다. 이는 새 문서의 시작을 `---`로 하고 문서의 끝을 `...`으로 하여 구현가능하다.

```yaml
---
document:1
...
---
document: 2
...
```

많은 경우에 `---`나 `...`는 생략이 가능하다.

헬름에서 몇몇 파일은 둘 이상의 문서를 포함할 수 없다. 예를 들어 하나 이상의 문서가 `values.yaml` 파일 내에 제공된다면 첫번째만 사용이 될 것이다.

하지만 템플릿 파일은 하나 이상의 문서를 가질 수 있다. 이는 파일(그리고 이에 대한 모든 문서)는 템플릿 렌더링을 하는 동안 하나의 오브젝트로 취급된다. 하지만 결과 YAML파일은 여러 문서로 나뉘어져서 쿠버네티스로 제공이 된다.

우리는 정확히 필요한 상황이 있는 경우에만 multiple documents를 사용할 것을 권장한다. 파일에서 multiple documents를 가지는 것은 디버깅을 하는 데 어려울 수 있다.

## YAML is a Superset of Json

YAML이 JSON의 superset이기 때문에 유효한 JSON 문서는 유효한 YAML 문서로 변환이 가능해야 한다.

```json
{
  "coffee": "yes, please",
  "coffees": [
    "Latte", "Cappuccino", "Espresso"
  ]
}
```

위의 예시는 다음처럼 표현이 가능하다.

```yaml
coffee: yes, please
coffees:
- Latte
- Cappuccino
- Espresso
```

그리고 이들 (주의 깊게)둘을 섞어 쓸수도 있다.

```yaml
coffee: "yes, please"
coffees: [ "Latte", "Cappuccino", "Espresso"]
```

이 세가지들은 하나의 동일한 internal representation으로 파싱된다.

이는 `values.yaml`이 JSON 데이터를 포함할 수 있다는 것을 의미하기 때문에 헬름은 file extension `.json`을 유효한 suffix로 취급하지 않는다.

## YAML Anchors

YAML 스펙은 value를 참조하기 위해 저장하는 방식을 제공하고, 후에 이를 참조하여 value를 가져온다. YAML은 이를 "anchoring"이라고 부른다.

```yaml
coffee: "yes, please"
favorite: &favoriteCoffee "Cappucino"
coffees:
  - Latte
  - *favoriteCoffee
  - Espresso
```

위의 예시에서 `&favoriteCoffee`는 `Cappuccino`로 설정이 되었다. 후에 이 참조는 `*favoriteCoffee`로 사용된다. 따라서 `coffees`는 `Latte, Cappuccino, Espresso`가 된다.

anchor가 유용한 몇가지 경우들이 있지만 이들은 감지하기 어려운 버그를 만드는 경향이 있다: 먼저 YAML이 이를 사용할 때 참조는 확장된 후 버려진다.

만약 위의 예시를 디코드하고 다시 인코딩 할 경우 YAML은 다음과 같아진다.

```yaml
coffee: yes, please
favorite: Cappucino
coffees:
- Latte
- Cappucino
- Espresso
```

헬름과 쿠버네티스는 YAML파일을 자주 읽고, 수정하고 재작성 하기 때문에 anchors는 사라질 수 있다.

# Appendix: Go Data Types and Templates

헬름 템플릿 언어는 Go 템플릿 언어로 작성이 되었다. 이런 이유로 템플릿에서 variables는 typed이다. 대부분의 파트에서 variables는 다음의 타입 중 하나로 표현됩니다.

* string: 텍스트의 스트링
* bool: `true` 또는 `false`
* int: integer 값(8, 16, 32, 64비트의 signed, unsigned가 있다)
* float64: 64-bit floating point value (8,16,32비트도 있다.)
* 바이너리 데이터를 가져올 때 자주 사용되는 byte slice([]byte)
* struct: properties와 methods를 가지는 오브젝트
* 이전의 타입에서 하나의 slice(indexed list)
* 이전의 타입 중 하나의 value인 string-keyed map (`map[string]interface{}`)

Go에는 다른 많은 타입들이 있고 때때로 사용자는 템플릿 안에서 convert해야할 수도 잇다. 가장 쉽게 오브젝트의 타입을 디버깅하는 방법은 템플릿 내에서 이를 `printf "%t"`에 전달하여 프린트하는 것이다. 또한 `typeOf`와 `kindOf` 함수도 확인해 보아라.