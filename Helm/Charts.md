# Charts

헬름은 차트라고 부르는 패키징 포멧을 사용한다. 차트는 쿠버네티스 리소스에 대한 관령 정보를 기술한 파일들의 묶음이다. 하나의 차트는 memcached 파드같은 어떤 간단한 것이나 HTTP 서버, 데이터베이스, 캐쉬 등을 사용하는 복잡한 full web app 배포하는데 사용할 수 있다.

차트는 특정한 디렉토리 트리에 있는 파일들로 생성이 되며 그것들은 배포되기 위해 버전화된 아카이브에 패키징된다.

이 문서는 차트 포멧에 대해 기술하며 Helm으로 차트를 작성하는데 기본적인 가이드를 준다.

## The Chart File Structure

차트는 디렉토리 내의 파일들의 모음으로 구성되어 있다. 디렉토리 이름은 (버전에 대한 정보가 없는) 차트의 이름이다. 즉, WordPress를 설명하는 차트는 `wordpress/` 디렉토리 내에 있어야 한다.

디렉토리 내부에서 Helm은 다음과 같은 구조를 생각한다.

```
wordpress/
  Chart.yaml          # 차트에 대한 정보를 담은 YAML 파일
  LICENSE             # OPTIONAL: 차트에 대한 라이센스를 포함하는 plain text 파일
  README.md           # OPTIONAL: 인간이 읽을 수 있는 README 파일
  requirements.yaml   # OPTIONAL: 차트에 대한 디펜던시를 나열한 YAML 파일
  values.yaml         # 이 차트의 기본값 설정들
  charts/             # 이 차트가 의존하고 있는 차트들의 디렉토리
  templates/          # 값들과 합쳐져서 유효한 쿠버네티스 매니페스트 파일을 생성하는 템플릿들의 디렉토리
  templates/NOTES.txt # OPTIONAL: 평문으로 작성된 짧은 사용법
```

Helm은 `charts/`와 `templates/` 디렉토리를 사용하도록 되어있고 파일들을 가지고 있다. 다른 파일들은 그래도 남아있을 것이다.

## The Chart.yaml file

차트에는 `Chart.yaml` 파일이 필요하다. 이는 다음과 같은 필드들을 가지고 있다.

```yaml
apiVersion: 차트의 API 버전으로 항상 "v1"이다. (필수)
name: 차트의 이름 (필수)
version: 시멘틱 버저닝 (필수)
kubeVersion: 지원하는 쿠버네티스 버전의 SemVer 범위 (옵션)
description: 프로젝트에 대한 간단한 설명 (옵션)
keywords:
  - 이 프로젝트에 대한 키워드 (옵션)
home: 이 프로젝트의 홈페이지 URL (옵션)
sources:
  - 이 프로젝트에 대한 소스 코드의 URL (옵션)
maintainers: # (옵션)
  - name: maintainer의 이름 (각각의 maintainer마다 필수)
    email: maintainer의 이메일 (maintainer에 대해 옵션) 
    url: maintainer의 URL (maintainer에 대해 옵션)
engine: gotpl # 템플릿 엔진의 이름 (옵션, 기본은 gotpl)
icon: icon으로 사용될 SVG나 PNG 이미지의 URL (옵션)
appVersion: 이 앱의 버전 (옵션). SemVer일 필요는 없음
deprecated: 이 차트가 deprecated인지 아닌지 표시 (옵션, boolean)
tillerVersion: 이 차트가 필요로 하는 tiller의 버전. 이는 SemVer 범위로 표현되어야 한다 : ">2.0.0" (옵션)
```

Helm Classic에 대한 `Chart.yaml` 파일 포멧과 친숙하다면 의존성을 지정하는 필드가 사라졌음을 유의해라. 이는 새로운 차트 포멧에서는 `charts/` 디렉토리에서 의존성을 표현하기 때문이다.

다른 필드는 그냥 무시된다.

### Chart and Versioning

모든 차트는 버전 번호가 반드시 있다. 버전은 반드시 SemVer2 표준을 따라야 한다. Helm Classic과는 다르게 쿠버네티스 Helm은 버전 번호를 릴리즈 마커(release marker)로 사용한다. 레파지토리의 패키지들은 이름과 버전으로 구분된다.

예를 들어, `version: 1.2.3`으로 버전이 설정된 `nginx` 차트는 다음과 같이 이름지어진다.

```
nginx-1.2.3.tgz
```

`version: 1.2.3-alpha.1+ef365`와 같은 더 복잡한 SemVer2 이름 또한 지원이 된다. 하지만 SemVer 이름이 아닌 것들은 시스템에 의해서 허용되지 않는다.

> Note : Helm Classic과 Deployment manager는 차트가 될 때 GitHub기반으로 되어졌지만 쿠버네티스 Helm은 GitHub나 Git도 필요로 하지 않다. 결과적으로 이것은 Git SHA를 버저닝에 전혀 사용하지 않는다.

`Chart.yaml` 안에 있는 `version`필드는 CLI와 Tiller server를 포함한 많은 Helm 툴에 의해 사용이 된다. 패키지를 생성할 때 `helm package` 커맨드는 패키지 이름으로 `Chart.yaml`에 있는 버전을 토큰으로 사용할 것이다. 시스템은 차트 패키지의 이름에 있는 버전 번호가 `Chart.yaml`의 버전 번호와 일치한다고 가정한다. 이 가정을 깨는 것은 에러를 발생시킨다.

### The appVersion field

`appVersion` 필드가 `version` 필드와 관련이 없다는 것에 유의하라. 이는 어플리케이션의 버전을 지정하는 것이다. 예를 들어, `drupal` 차트가 `appVersion: 8.2.1`을 가지고 있다고 하면 차트에 포함되어 있는 Drupal의 버전이 기본적으로 `8.2.1`이라는 것이다. 이 필드는 정보를 주는 것이며 차트 버전을 계산하는 데에는 영향을 주지 않는다.

### Deprecating Charts

차트 레파지토리에 있는 차트를 관리할 때 가끔 차트를 deprecate할 필요가 있다. `Chart.yaml` 필드의 `deprecated` 필드는 차트가 deprecated되었다고 마킹할 때 사용된다. 만약 레파지토리에 있는 최신 버전의 차트가 deprecated라면 전체 차트가 deprecated되었다고 결론짓는다. 차트의 이름은 deprecated되지 않은 것에 대해 새로운 버전을 만들어서 나중에 다시 사용될 수 있다. deprecating 차트에 대한 workflow는 `helm/charts` 프로젝트를 따른다.

- 차트의 `Chart.yaml`를 deprecated로 마킹하여 버전을 사용 못하게 한다.
- 차트 레파지토리에서 새로운 차트 버전을 릴리즈
- git과 같은 소스 레파지토리에서 차트 삭제

## Chart License, Readme and Notes

차트는 설치, 설정, 사용법과 라이센스에 대한 설명하는 파일을 가지고 있을 수 있다.

LICENSE는 차트에 대한 `license`를 포함하는 평문파일이다. 차트에는 템플릿 안에 프로그래밍 로직들이 들어있을 수 있기 때문에 설정에 대한 라이센스만 있는 것은 아니다. 차트에 의해서 설치되는 각각의 어플리케이션에 대해 각각 라이센스가 있을 수 있다.

차트에 대한 README는 Markdown 형태의 파일(README.md)이어야 한다. 그리고 일반적으로 다음을 포함한다.

- 차트가 제공하는 어플리케이션이나 서비스에 대한 설명
- 차트를 실행시키는데 필요한 요구사항
- `values.yaml`에 있는 옵션과 기본 값에 대한 설명들
- 설치나 차트의 설정에 관한 추가적인 정보들

차트는 설치 뒤에 출력이 되고 릴리즈의 상태를 볼 때 출력되는 짧은 평문의 `templates/NOTES.txt` 파일을 포함할 수 있다. 이 파일은 `template`으로 평가되며 사용법, 다음 단계, 또는 차트의 릴리즈에 관련된 추가적인 정보들이 올 수 있다. 예를 들어, 데이터베이스에 접속하기 위한 방법, 웹 UI에 접속하기 위한 방법들을 알려줄 수 있다. 이 파일이 `helm install`이나 `helm status`를 했을 때 STDOUT으로 출력이 되기 때문에 짧게 적는 것이 중요하고 README에 디테일한 부분들을 담는 것이 좋다.

## Chart Dependencies

Helm에서 하나의 차트는 다른 차트들에 대해 dependency가 있을 수 있다. 이런 dependency들은 `requrements.yaml`파일을 통해 동적으로 연될 수 도 있고 `charts/` 디렉토리 안에 가져와서 수동으로 관리할 수도 있다.

어떤 팀에게 있어서는 약간의 이점을 줄 수 있어서 수동으로 관리하게 된다 하더라도 의존성을 선언하는 것은 차트 내의 `requirements.yaml`을 통해 하는 것을 더욱 권장한다.

> Helm 클래식에서 `Chart.yaml`의 `dependencies:` 섹션은 완전하게 삭제되었다.

### Managing Dependencies with `requirements.yaml`

`requrements.yaml` 파일은 의존성을 나열하는 간단한 파일이다.

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

* `name` 필드는 필요한 차트의 이름이다.
* `version`은 필요한 차트의 버전이다.
* `repository` 필드는 차트 레파지토리에 대한 완전한 URL이다. 반드시 `helm repo add`를 통해서 repo를 로컬로 추가해야 함을 인지해라.

의존성 파일을 가지게 될 경우 `helm dependency update`를 실행하여 의존성으로 지정된 모든 차트들을 `charts/` 디렉토리에 다운로드할 것이다.

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete.
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

`helm dependency update`가 차트를 가지고 오면, 차트 아카이브를 `charts/` 디렉토리에 저장할 것이다. 위의 예시에서 charts 디렉토리에 다음과 같은 파일들이 올 것이라고 예상할 수 있다.

```
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

차트를 `requirements.yaml`로 관리하는 것은 차트를 계속해서 최신상태로 유지할 수 있고 팀 전체에 의존성에 대한 정보를 줄 수 있다.

### Alias field in requirements.yaml

위의 다른 필드들에 추가적으로 각각의 requirements 엔트리는 `alias`라는 옵션 필드를 가질 수 있다.

의존성 차트에 alias를 사용하는 것은 새로운 의존성으로 alias를 사용한 이름으로 의존성이 추가될 것이다.

`alias`를 사용하는 경우는 하나의 차트를 여러 이름으로 사용할 필요가 있는 경우가 있을수도 있다.

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

위의 예시에서 `parentchart`에 대해 3가지 dependency가 있다.

```
subchart
new-subchart-1
new-subchart-2
```

수동으로 이렇게 하려면 `charts/` 디렉토리에서 복사/붙여넣기로 다른 이름을 가진 여러개를 만들면 된다.

### Tags and Condition fields in requirements.yaml

위의 다른 필드들에 추가적으로 각각의 requirements 엔트리는 `tags`와 `condition`이라는 옵션 필드를 가질 수 있다.

모든 차트는 기본적으로 로딩이 된다. 만약 `tags`나 `condition` 필드가 존재한다면 이를 확인하고 적용하려는 차트에 로딩이 될지 말지 결정한다.

Condition - condition 필드는 하나 이상의 YAML 경로를 가진다.(comma로 구분) 만약 이 경로가 최상의 parent의 values로 있고, boolean으로 풀이가 되면 차트는 이 boolean값에 의해서 활성화될지, 비활성화 될지 결정한다. 리스트에서 처음으로 유효한 경로만 확인되고 경로가 존재하지 않다면 condition은 아무런 영향을 주지 않는다.

Tags - tags 필드는 이 차트와 연관된 label들을 가진 YAML 리스트이다. 최상위 parents의 values에서 tags를 가진 모든 차트는 tag와 boolean 값을 지정함으로써 활성화되거나 비활성화된다.

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled,global.subchart1.enabled
    tags:
      - front-end
      - subchart1

  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

위의 `front-end` 태그를 가진 모든 차트는 비활성화 되겠지만 `subchart1.enabled` 경로가 parents의 values에서 true라고 판명이 되었기 때문에 condition은 `front-end`태그를 오버라이딩 하고 `subchart1`은 활성화 될 것이다.

`subchart2`가 `back-end`라고 태그되어있고 그 태그가 `true`라고 판명되었으므로 `subchart2`는 활성화된다. `subchart2`가 condition이 `requirements.yaml`에서 지정되어 있음에도 불구하고 parents의 values에서 일치하는 경로와 값이 없기 때문에 condition은 아무런 영향을 주지 않는다.

#### Using the CLI with Tags and Conditions

`--set` 파라미터는 tag와 condition 값을 변경되는데 사용될 수 있다.

```console
helm install --set tags.front-end=true --set subchart2.enabled=false
```

#### Tags and Condition Resolution

* **Conditions(values에 세팅이 되어 있다면)는 항상 tags를 오버라이딩한다.**
* 처음으로 유효한 condition path가 적용이 되며 나머지 condition path들은 무시된다.
* 태그들은 '차트의 tags들 중 하나라도 참이면 차트를 활성화 한다.'라고 여겨진다.
* tags와 conditions values들은 반드시 parents의 values에 설정되어 있어야 한다.
* `tags` : values의 key들은 반드시 최상위 레벨의 key여야 한다. 글로벌, nested `tags:`는 현재 지원하지 않는다.

### Importing Child Values via requirements.yaml

몇몇 경우에 공통된 기본값을 가지기 위해 자식 차트의 values를 부모 차트에 적용시켜야 할 때가 있다. `exports` 포멧을 사용하는 추가적인 이점은 유저가 설정할 수 있는 값들을 파악할 수 있다는 것이다.

values를 가진 키들은 부모 차트의 `requirements.yaml`에 YAML리스트로 지정되어 있다. 리스트의 각각의 아이템들은 자식 차트의 `exports` 에서 가지고 올 키들이다.

`exports` 키에 포함되지 않은 값들을 가지고 오려면 `child-parent` 포멧을 사용하면 된다. 두 포멧의 예시들이 아래에 나와있다.

#### Using the exports format

자식 차트의 `values.yaml`파일이 `exports` 필드를 루트에 가지고 있으면 이 내용은 부모의 values에서 불러올 값의 키를 지정함으로써 직접 불러와진다.

```yaml
# parent's requirements.yaml file
    ...
    import-values:
      - data
```

```yaml
# child's values.yaml file
...
exports:
  data:
    myint: 99
```

우리가 import 리스트에서 `data`라는 키를 지정했기 때문에 Helm은 자식차트의 `exports` 필드에서 `data`라는 키를 찾을 것이고 이를 불러올 것이다.

위 예시에서 부모의 values는 이렇게 될 것이다.

```yaml
# parent's values file
...
myint: 99
```

부모의 `data`키는 부모의 최종 값을 가지고 있지 않음을 주의하라. 만약 부모의 키를 지정해야 한다면 `child-parents` 포멧을 사용하라.

#### Using the child-parent format

자식 차트의 values의 `exports` 키 안에 포함되지 않은 값에 접근하려면 자식에서 어떤 값을 가지고 올지를 지정하고 부모의 values 어느 경로에 저장할지를 지정하면 된다.

다음의 예시에서 `import-values`는 Helm이 `child:`에 보이는 모든 값들을 가지고 와서 부모의 `parents:`에 지정된 경로로 복사하게 한다.

```yaml
# parent's requirements.yaml file
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

위의 예시에서 subchart1의 값 중 `default.data`의 값들이 부모 차트의 `myimports` 키값으로 들어가게 된다. 아래는 자세한 예시이다.

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

이 경우에 부모 차트의 결과값은 다음과 같다.

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

부모의 최종 값은 `myint`와 `mybool`을 subchart1으로부터 가져온 것이다.

### Managing Dependencies manually via the `charts/` directory

의존성들을 원하는 대로 더 조절하고 싶으면 의존성이 있는 차트들을 직접 `charts/` 디렉토리에 복사해 넣음으로서 할 수 있다.

의존성은 `foo-1.2.3.tgz`같은 차트의 아카이브가 될 수도 있고 압축이 해제된 차트 디렉토리가 될 수도 있다. 하지만 이름은 `_`나 `.`으로는 시작하면 안된다. 이런 파일들은 차트 로더에 의해서 무시된다.

예를 들어 WordPress 차트는 Apache 차트에 의존성이 있고 알맞은 버전의 Apache 차트가 WordPress 차트의 `charts/` 디렉토리에 제공이 되어 있다.

```
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

위의 예시는 WordPress 차트가 이 Apache와 MySQL에 대한 의존성을  `charts/` 디렉토리안에 포함시킴으로써 의존성이 있음을 보여준다.

> Tip : `charts/` 디렉토리 안에서 의존성을 없애고 싶으면 `helm fetch` 명령어를 사용하라.

### Operational aspects of using dependencies

위의 섹션에서 어떻게 차트의 의존성이 표현되는지를 배웠었다. 그런데 어떻게 이러한 것들이 `helm install`과 `helm upgrade`에서 차트에 영향을 줄까?

"A"라는 이름을 가진 차트가 생성이 되었고 다음의 쿠버네티스 오브젝트를 가지고 있다고 해보자.

* 네임스페이스는 "A-Namespace"이다.
* statefulset은 "A-StatefulSet"이다.
* 서비스는 "A-Service"이다.

게다가 A는 차트 다음의 오브젝트를 생성하는 B에 의존하고 있다.

* 네임스페이스는 "B-Namespace"
* replicaset은 "B-ReplicaSet"
* 서비스는 "B-Service"

차트 A에 대한 설치/업그레이드가 끝나고 나면 하나의 Helm 릴리즈가 생성/수정이 된다. 릴리즈는 위의 쿠버네티스 오브젝트에 대해서 다음과 같은 순서로 생성/업데이트 한다.

* A-Namespace
* B-Namespace
* A-StatefulSet
* B-ReplicaSet
* A-Service
* B-Service

이는 Helm이 차트를 설치/업그레이드 할 때 차트의 쿠버네티스 오브젝트와 의존성들이

* 하나의 집합으로 통합이 되고 나서
* 타입으로 정렬되고 이름순으로 정렬한 후에
* 그 순서대로 생성/업데이트를 한다.

그렇게 하나의 릴리즈가 차트와 그 의존성 차트들의 모든 오브젝트를 생성한다.

쿠버네티스 타입의 설치 순서는 kind_sorter.go의 enumeration InstallOrder로 주어져 있다. ([Helm 소스코드](https://github.com/helm/helm/blob/master/pkg/tiller/kind_sorter.go#L26) 참조)

## Templates and Values

Helm 차트 템플릿은 Go 템플릿 언어에 추가로 [Sprig library](https://github.com/Masterminds/sprig)와 특별한 기능들을 추가한 템플릿으로 구성되어 있다.

모든 템플릿 파일은 차트의 `templates/` 폴더 내에 저장되어 있다. Helm이 차트를 렌더링할 때 그 디렉토리에 있는 모든 파일들을 템플릿 엔진으로 보낼 것이다.

템플릿의 값들은 두가지 방식으로 제공될 수 있다.

* 차트 개발자는 `values.yaml` 파일을 차트 안에서 제공할 수 있다. 이는 기본 값들을 가지고 있다.
* 차트 유저는 값들을 가지고 있는 YAML파일을 제공할 수 있다. 이는 `helm install` 커맨드와 함께 제공된다.

유저가 커스텀 값들을 제공할 때 이 값들은 차트의 `values.yaml` 파일을 오버라이딩 한다.

### Template Files

템플릿 파일들은 GO 템플릿의 표준 작성법을 준수한다.([the text/template Go package documentation](https://golang.org/pkg/text/template/)을 통해 자세한 사항 확인 가능) 템플릿 파일의 간단한 예시는 다음과 같을 것이다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

https://github.com/deis/charts를 기반으로 한 위의 예시는 쿠버네티스의 ReplicationController의 템플릿이다. 이는 다음과 같은 네가지의 템플릿 값들을 가지고 있다.(보통은 `values.yaml`에 정의되어있다.)

* `imageRegistry` : 도커 이미지에 대한 소스 레지스트리이다.
* `dockerTag` : 도커 이미지에 대한 태그이다.
* `pullPolicy` : 쿠버네티스의 pull policy이다.
* `storage` : 기본값이 "minio"로 설정이 된 storage backend이다.

이 모든 값들은 템플릿 저자에 의해서 정의가 되어져 있다. Helm은 파라미터를 요구하거나 지시하지 않는다.

많은 차트들을 보려면 [Helm Charts project](https://github.com/helm/charts)를 참조하라.

### Predefined Values

`values.yaml` 파일로 제공이 되는 값들은(또는 `--set` 플래그를 통해) 템플릿에서 `.Values` 오브젝트로부터 접근이 가능하다. 하지만 템플릿 내에서 접근할 수 있는 몇가지 pre-defined 데이터들이 있다.

다음의 값들은 pre-defined이며 모든 템플릿에서 이용이 가능하고 오버라이딩이 불가능하다. 이 모든 값들은 case sensitive하다.

* `Release.Name` : 릴리즈의 이름(차트의 이름이 아님)
* `Release.Time` : 차트 릴리즈가 마지막으로 업데이트 된 시간. 이는 Release 오브젝트에서 `Last Released` 시간과 같다.
* `Release.Namespace` : 차트가 릴리즈된 네임스페이스
* `Release.Service` : 릴리즈가 수행되는 서비스. 보통은 `Tiller`이다.
* `Release.IsUpgrade` : 현재의 동작이 업그레이드거나 롤백이면 참이다.
* `Release.IsInstall` : 현재의 동작이 설치라면 참이다.
* `Release.Revision` : revision 숫자이다. 1로 시작되어 `helm upgrade`할 때마다 1씩 증가한다.
* `Chart` : `Chart.yaml`의 내용. 따라서 차트의 버전은 `Chart.Version`으로 얻을 수 있고 maintainers는 `Chart.Maintainers`로 접근할 수 있다.
* `Files` : 차트안에 특별하지 않은 모든 파일을 포함하는 맵같은 오브젝트. 템플릿에 접근은 할 수 없지만 이는 존재하는 추가적인 파일들에 접근할 수 있도록 해준다.(`.helmignore`에 포함되어 있지 않다면) 파일은 `{{index .Files "file.name"}}`이나 `{{.Files.Get name}}`이나 `{{.Files.GetString name}}`으로 접근할 수 있다. 파일의 내용에 대해 `{{.Files.GetBytes}}`를 사용하여 `[]byte`형식으로 접근이 가능하다.
* `Capabilities` : 쿠버네티스 버전(`{{.Capabilities.KubeVersion}}`), Tiller 버전(`{{.Capabilites.TillerVersion}}`), 지원하는 쿠버네티스 API 버전(`{{.Capabilities.APIVersions.Has "batch/v1"}}`)에 대한 정보를 가지고 있는 맵같은 오브젝트.

> Note : 알 수 없는 Chart.yaml필드들은 무시된다. 이것들은 `Chart` 오브젝트 내에서 접근할 수 없다. 따라서 Chart.yaml은 임의의 구조화된 데이터를 템플릿에 보낼 수 없다. 이 values 파일들은 그렇게 사용할 수도 있다.

### Values files

이전 섹션에서 템플릿을 생각해보면 필수적인 값들을 제공하는 `values.yaml` 파일은 다음과 같을 것이다.

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

values 파일은 YAML 포멧이다. 차트는 기본 `values.yaml` 파일을 가질 수 있다. Helm install 명령어는 유저가 추가적인 YAML 값들을 제공할하여 값들을 오버라이딩 하게 할 수 있다.

```console
$ helm install --values=myvals.yaml wordpress
```

values가 이렇게 전달이 되면 default values 파일에 병합된다. 예를 들어, `myvals.yaml`파일을 다음과 같다고 해보자.

```yaml
storage: "gcs"
```

이것이 차트의 `values.yaml`파일과 병합이 되면 결과는 다음과 같을 것이다.

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

마지막 필드만 오버라이딩 됐음을 주의하라.

> Note : 차트 내의 default values 파일은 반드시 `values.yaml`로 이름지어져야 한다. 하지만 명령어에서 지정하는 파일들은 아무 이름이나 가능하다.

> Note : 만약 `--set` 플래그가 `helm install`이나 `helm upgrade`에 사용된다면 이런 값들은 클라이언트쪽에서 YAML로 변형된다.

> Note : 만약 필수 엔트리들이 values 파일에 있다면 [‘required’ function](https://helm.sh/docs/developing_charts/#chart-development-tips-and-tricks)을 사용하여 차트 템플릿에서 required라고 선언할 수 있다.

이 값들은 `.Values` 오브젝트로 템플릿 안에서 접근이 가능하다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

### Scope, Dependencies, and Values

Values 파일은 상위 레벨 차트에 대한 값들을 선언할 수 있다. 또한 차트의 `charts/` 디렉토리 내의 어떤 차트의 값도 포함할 수 있다. 다르게 말하면 values 파일은 의존성을 가지는 모든 차트들의 values를 제공할 수 있다. 예를 들어 WordPress의 데모 차트는 `mysql`과 `apache` 에 의존성을 가지고 있다. values 파일은 이런 모든 컴포넌트에 대해서 값을 제공할 수 있다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

상위 레벨의 차트는 아래에 정의된 모든 변수들에 대해 접근할 수 있다. 따라서 WordPress 차트는 MySQL 비밀번호를 `.Values.mysql.password`로 접근할 수 있다. 하지만 하위 레벨의 차트는 부모의 변수들에 대해 접근할 수 없으며 따라서 MySQL은 `title` 속성에 접근이 불가능하다. 그리고 그 이유로 `apache.port`에도 접근할 수 없다.

Values는 네임스페이스로 관리가 되지만 네임스페이스는 잘려져 있다. 따라서 WordPress 차트에서 MySQL의 비밀번호를 `.Values.mysql.password`로 접근할 수 있다. 하지만 MySQL 차트에서는 값의 스코프가 줄어들고 네임스페이스의 prefix가 삭제되어 패스워드를 간단하게 `.Values.password`로 접근할 수 있다.

### Global Values

2.0.0-Alpha.2부터 Helm은 특정한 "global" 값들을 지원한다. 이전의 예시에서 수정된 버전은 다음과 같다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

위의 `global` 섹션이 `app: MyWordPress`라는 값을 가지고 있다. 이 값은 모든 차트에서 `.Values.global.app`으로 접근할 수 있다.

예를 들어, `mysql` 템플릿은 `app`을 `{{.Values.global.app}}`으로 접근할 수 있으며 `apache` 차트또한 그러하다. 위의 차트가 다음과 같이 생성이 되는 것이다.

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

이는 하나의 상위 레벨의 변수를 모든 서브차트에서 공유하는 방법을 제공한다. 이는 라벨과 같은 `metadata` 속성들을 세팅할 때 좋다.

서브차트가 global 변수를 선언하면 global은 downward로 전달된다.(서브차트의 서브차트로) 하지만 upward로 부모 차트에게는 전달되지 않는다. subchart가 부모 차트의 값에 영향을 줄 수 있는 방법은 없다.

또한 부모 차트의 global 변수는 서브차트의 global 변수보다 더 우선순위를 가진다.

### Reference

템플릿과 values 파일을 작성할 때 다음의 표준들을 확인하면 도움이 될것이다.

- [Go templates](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [The YAML format](https://yaml.org/spec/)

## Using Helm To Manage Charts

`helm` 툴은 차트와 함꼐 사용하는 몇가지 명령어가 있다.

사용자는 새로운 차트를 다음과 같이 만들 수 있다.

```console
$ helm create mychart
Created mychart/
```

차트를 수정하고 나면 `helm`은 차트 아카이브로 패키징 시킬 수 있다.

```console
$ helm package mychart
Archived mychart-0.1.-.tgz
```

`helm`을 사용하여 차트의 포멧이나 정보에 문제점을 찾을 수 있다.

```console
$ helm lint mychart
No issues found
```

## Chart Repositories

차트 레파지토리는 HTTP 서버로 하나 이상의 패키지 파일을 가지고 있는 것이다. `helm`이 로컬 차트 디렉토리를 관리할 때 사용할 수 있다면 이 차트를 공유하려 할 때 선호되는 방법이 차트 레파지토리이다.

어느 HTTP 서버든지 YAML파일과 tar 파일을 제공할 수 있고 GET 요청에 응답하여 레파지토리 서버로 사용될 수 있다.

Helm은 개발자 테스트 용으로 빌트인 패키지 서버를 가지고 있다.(`helm serve`) Helm 팀은 웹사이트 모드가 활성화 된 Google Cloud Storage와 웹사이트 모드가 활성화 된 S3같은 다른 서버에 대해서도 테스트를 완료하였다.

레파지토리는 레파지토리가 제공하는 모든 패키지의 목록과 이런 패키지들을 얻고 검증할 수 있는 메타데이터들을 포함한 `index.yaml`이라고 불리는 특정한 파일의 존재로 구분된다.

클라이언트쪽에서 레파지토리는 `helm repo` 명령어로 관리가 된다. 하지만 Helm은 원격 레파지토리 서버에 업로딩 하는 것은 지원하지 않는다. 이는 그렇게 할 경우 서버를 구축하는 데 추가적인 요구사항이 생기게 되며 레파지토리를 구축하는 데 장벽이 될 수 있기 때문이다.

## Chart Starter Packs

`helm create` 명령어는 `--starter` 옵션으로 "starter chart"를 지정할 수 있다.

스타터는 그냥 일반적인 차트지만 `$HELM_HOME/starters`에 위치한 것이다. 차트 개발자는 스타터로 특정한 차트를 지정할 수 있다. 이런 차트는 다음과 같은 사항을 고려해야 한다.

* `Chart.yaml`은 generator에 의해 덮어씌워질 수 있다.
* 사용자가 이 차트의 내용을 수정한다고 기대되기 때문에 문서는 어떻게 유저가 이를 수정할 수 있을지 알려줘야한다.
* `templates` 디렉토리 안에 있는 파일에서 `<CHARTNAME>` 는 지정된 차트의 이름으로 대치가 되어 스타터 차트가 이를 템플릿으로 사용할 수 있을 것이다. 추가적으로 `values.yaml`에서 `<CHARTNAME>`도 대치가 될 수 있다.

현재 `$HELM_HOME/starters`에 차트를 추가하는 유일한 방법은 그곳으로 복사하는 것이다. 차트의 문서에서 이 프로세스를 설명하는 방법도 있다.

# Hooks

Helm은 hook 메커니즘을 제공하여 차트 개발자가 릴리즈의 라이프 사이클에 특정 포인트에 개입하도록 할 수 있다. 예를 들어 다음과 같은 hook을 사용할 수 있다.

* 설치하는 동안 다른 차트가 로딩되기 전에 ConfigMap이나 Secret을 불러온다.
* 데이터를 보존하기 위해 새로운 차트를 설치하기 전에 데이터베이스를 백업하는 동작을 실시하고 업그레이드 후에 그 다음 작업을 진행한다.
* 릴리즈를 삭제하기 전에 서서히 서비스를 줄여나가는 동작을 실행한다.

Hook은 정규 템플릿처럼 작동하지만 Helm이 다르게 인식하게 하는 특정한 어노테이션이 있다. 이 섹션에서는 hook의 일반적인 패턴에 대해 알아본다.

Hook은 manifest의 메타데이터 섹션에서 어노테이션으로 선언된다.

```yaml
apiVersion: ...
kind: ....
metadata:
  annotations:
    "helm.sh/hook": "pre-install"
# ...
```

## The Available Hooks

다음의 hook들이 가능하다.

* pre-install : 템플릿이 렌더링 되고 난 후, 리소스가 쿠버네티스에서 생성되기 전 실행한다.
* post-install : 모든 리소스가 쿠버네티스에서 로딩되고 난 후 실행한다.
* pre-delete : 삭제 요청에서 쿠버네티스에서 어떤 리소스도 삭제되기 전에 실행한다.
* post-delete : 삭제 요청 시 모든 리소스가 삭제되고 난 후 실행한다.
* pre-upgrade : 업그레이드 요청 시 템플릿이 렌더링 되고 난 후, 쿠버네티스에 어떤 리소스도 로딩되기 전에 실행한다. (예를 들어 쿠버네티스 apply 동작같은 것)
* post-upgrade : 업그레이드 시 모든 리소스가 업그레이드 되고 난 후 실행한다.
* pre-rollback : 롤백 요청시 템플릿이 렌더링 되고 난 후, 어떤 리소스도 아직 롤백되기 전에 실행한다.
* post-rollback : 롤백 요청시 모든 리소스가 수정되고 난 후 실행한다.
* crd-install : 다른 체크사항들이 동작하기 전에 CRD 리소스를 추가한다. 차트의 다른 manifest에서 사용되는 CRD 정의에만 쓸 수 있다.
* test-success : `helm test`를 실행할 때 실행하며 파드가 성공적으로 리턴되는 것을 기대한다. (return code == 0)
* test-failure : `helm test`를 실행할 때 실행하며 파드가 실패하기를 기대한다. (return code != 0)

## Hooks and The Release Lifecycle

Hook은 차트 개발자들에게 릴리즈 라이프사이클에서 중요한 지점에서 특정한 동작을 할 수 있도록 해준다. 예를 들어 `helm install`의 라이프사이클에 대해서 생각해보자. 기본적으로 라이프사이클은 다음과 같다.

1. 사용자는 `helm install foo`를 실행한다.
2. 차트가 Tiller에 로딩된다.
3. 몇가지 검증을 걸치고 Tiller는 `foo` 템플릿을 렌더링한다.
4. Tiller는 렌더링 된 리소스들을 쿠버네티스에 로딩한다.
5. Tiller는 클라이언트에게 릴리즈 이름(또는 다른 데이터)를 리턴한다.
6. 클라이언트가 종료된다.

Helm은 `install` 라이프사이클에서 두가지 훅을 정의한다: `pre-install`과 `post-install` `foo`차트의 개발자가 이 두가지 훅을 사용한다면 라이프사이클은 다음과 같이 될 것이다.

1. 사용자는 `helm install foo`를 실행한다.
2. 차트는 Tiller에 로딩된다.
3. 몇가지 검증을 걸치고 Tiller는 `foo` 템플릿을 렌더링한다.
4. Tiller는 `pre-install` 훅을 실행할 준비를 한다.(쿠버네티스에 훅 리소스를 로딩한다.)
5. Tiller는 훅을 weight로 정렬하고(기본값은 0으로 할당이 되어 있다.) 같은 weight의 경우 이름을 오름차순으로 정렬한다.
6. Tiller는 훅을 낮은 weight부터 로딩한다. (음수에서 양수)
7. Tiller는 훅이 "Ready"가 될 때까지 기다린다.(CRD는 제외)
8. Tiller는 렌더링된 리소스들을 쿠버네티스에 로딩한다. `--wait` 플래그가 설정이 되어 있다면 Tiller는 모든 리소스들이 ready 상태가 될 때까지 기다리고  `post-install` 훅은 그때까지 실행되지 않을 것이다.
9. Tiller는 `post-install` 훅을 실행한다.(훅 리소스를 로딩한다.)
10. Tiller는 훅이 "Ready"가 될때까지 기다린다.
11. Tiller는 클라이언트에게 릴리즈 이름(또는 다른 테이터)을 리턴한다.
12. 클라이언트는 종료된다.

훅이 준비될때까지 기다린다는 말은 무었일까? 이는 훅에 정의된 리소스에 따라 다르다. 리소스가 `Job` 종류라면 Tiller는 이 작업이 성공적으로 완료될 때까지 기다린다. 이 Job이 실패하게 되면 릴리즈도 실패한다. 이는 blocking operation이라고 하며 Helm 클라이언트는 이 Job이 동작하는 동안 멈춰있게 된다.

모든 다른 종류들에 대해서는 쿠버네티스가 리소스에 대해 로딩(추가 또는 업데이트)되었다고 표시하자마자 리소스는 "Ready" 상태가 된다. 많은 리소스들이 훅에 정의되어있는 경우에 리소스는 연속적으로 실행이 된다. 만약 훅 weight을 가지고 있다면 weight 순서대로 실행이 된다. 아니면 순서는 보장이 되지 않는다.(Helm 2.3.0이후부터는 알파벳 순으로 정렬이 된다. 하지만 그 동작은 보장되어있지 않으며 나중에 변경될 수도 있다.) 훅 weight을 추가하는 것은 좋은 활용법이 될 것이고 weight이 중요하지 않은 경우라면 `0`으로 설정하라.

### Hook resources are not managed with corresponding releases

훅이 생성한 리소스들은 릴리즈의 일부분으로 추적되고 관리되지 않는다. Tiller가 훅이 준비 상태로 도달했다고 확인이 되면 훅 리소스는 혼자 남겨져있게 된다.

실용적인 관점에서, 이는 훅에서 리소스를 생성하였다면 그 리소스를 `helm delete`로 삭제할 수 없다는 의미이다. 이런 리소스를 없애려면 이런 동작을 하는 코드를 `pre-delete`나 `post-delete`에 추가하거나 `"helm.sh/hook-delete-policy"` 어노테이션을 훅 템플릿 파일에 추가할 수 있다.

## Writing a Hook

훅은 단지 `metadata` 섹션에 특정한 어노테이션이 있는 쿠버네티스 manifest 파일일 뿐이다. 이것들은 단순 템플릿 파일이기 때문에 `.Values`, `.Release`, `.Template`을 읽을 수 있는 것을 포함한 모든 일반적인 템플릿의 기능을 사용할 수 있다.

예를 들어 `templates/post-install-job.yaml`에 저장된 이 템플릿은 `post-install`에서 실행하야 하는 job을 선언한다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    app.kubernetes.io/managed-by: {{.Release.Service | quote }}
    app.kubernetes.io/instance: {{.Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{.Release.Name}}"
      labels:
        app.kubernetes.io/managed-by: {{.Release.Service | quote }}
        app.kubernetes.io/instance: {{.Release.Name | quote }}
        helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default "10" .Values.sleepyTime}}"]
```

이 템플릿을 훅으로 만들어 주는 것은 다음의 어노테이션이다.

```yaml
  annotations:
    "helm.sh/hook": post-install
```

하나의 리소스는 여러가지 훅이 될 수 있다.

```yaml
  annotations:
    "helm.sh/hook": post-install,post-upgrade
```

이와 비슷하게 하나의 훅은 숫자의 제한 없이 여러 다른 리소스들을 구현할 수 있다. 예를 들어 하나의 훅이 pre-install 훅으로 secret과 config map을 선언할 수 있다.

서브차트가 훅을 선언하면 이것들 또한 사용된다. 상위 레벨의 차트가 서브차트에서 선언된 훅을 비활성화 할 방법은 없다.

실행 순서를 정의할 수 있도록 weight값을 지정할 수 있다. weight은 다음의 어노테이션으로 정의된다.

```yaml
  annotations:
    "helm.sh/hook-weight": "5"
```

훅의 weight은 양수 또는 음수일 수 있지만 반드시 string으로 표현이 되어야 한다.

Tiller가 어떤 종류의 훅 사이클(예: `pre-install`훅 또는 `post-install`훅 등)을 시작할 때 오름차순으로 정렬이 된다.

또한 이에 맞는 훅 리소스를 삭제할지를 결정하는 정책을 정의할 수 있다. 훅 삭제 정책은 다음의 어노테이션으로 정의할 수 있다.

```yaml
  annotations:
    "helm.sh/hook-delete-policy": hook-succeeded
```

하나 이상의 어노테이션 값을 고를 수 있다.

* `"hook-succeeded"`는 Tiller가 훅이 성공적으로 실행되고 나서 훅을 삭제해야 한다고 지정하는 것이다.
* `"hook-failed"`는 Tiller가 hook이 실행하는 동안 실패할 경우 훅을 삭제해야 한다고 지정하는 것이다.
* `"before-hook-creation"`는 Tiller가 새로운 훅을 런칭할 때 이전의 훅을 삭제해야 한다고 지정하는 것이다.

기본적으로 Tiller는 60초동안 타임아웃이 일어나기 전에 API 서버에서 훅이 더이상 존재하지 않기를 기다린다. 이 동작은 `helm.sh/hook-delete-timeout` 어노테이션을 사용하여 바꿀 수 있다. 이 값은 Tiller가 훅이 완전 삭제될 때까지 기다리는 초단위 시간이다. 0은 Tiller가 전혀 기다리지 않음을 의미한다.

### Defining a CRD with the `crd-install` Hook

Custom Resource Definitions (CRDs)는 쿠버네티스에서 특별한 종류이다. 이는 다른 종류들을 정의하는 방법을 제공한다.

때로 차트는 종류를 정의 하기도 하고 동시에 이를 사용할 필요가 있을 수 있다. 이것은 `crd-install` 훅으로 할 수 있다.

`crd-install` 훅은 설치시 나머지 manifest가 검증이 되기 전에 매우 일찍 실행이 된다. CRD는 이 훅으로 어노테이션 될 수 있고 따라서 어떤 인스턴스가 CRD를 참조하기 전에 설치될 수 있다. 이 방법으로 검증이 나중에 발생하면 CRD는 사용이 가능해진다.

CRD를 훅으로 정의한 예시이며 CRD 인스턴스는 다음과 같다.

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
  annotations:
    "helm.sh/hook": crd-install
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

그리고

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: {{ .Release.Name }}-inst
```

두가지 모두 같은 차트에 있으면 CRD는 올바르게 어노테이션 된 것이다.

### Automatically delete hook from previous release

훅을 사용하는 Helm 릴리즈가 업데이트 되면 훅 리소스는 이미 클러스터 내에 존재할 것이다. 이런 상황에서 기본적으로 helm은 훅 리소스를 설치하려는 시도에 대해서 `"... already exists"` 에러와 함께 시패할 것이다.

훅 리소스가 이미 존재하는 일반적인 이유는 다음번 설치/업데이터 이전에 삭제되도록   되어 있지 않기 때문이다. 사실  훅을 유지하려는 좋은 이유가 있다: 예를 들어, 뭔가 잘못 되었을 때 수동으로 디버깅하여 고치기 위함이다. 이경우에 다음번의 훅을 생성하려는 시도가 실패하지 않도록 보장해주는 추천 방법은 `"hook-delete-policy"`를 `"helm.sh/hook-delete-policy": "before-hook-creation"`으로 정의하는 것이다. 훅 어노테이션은 새로운 훅이 설치되기 전에 존재하는 훅들을 삭제한다.

각 훅이 실행되고 나서 이를 지우고 싶다면(위의 예시처럼 다음번 사용시 삭제하는 것이 아니라) 삭제 정책을 `"helm.sh/hook-delete-policy": "hook-succeeded,hook-failed"`로 설정하면 된다.

# Chart Development Tips and Tricks

이 가이드는 몇가지 팁과 Helm 차트 개발자가 프로덕션 퀄리티의 차트를 만들때 배워야 할 것들을 담고 있다.

## Know Your Template Functions

Helm은 Go 템플릿을 사용하여 리소스 파일을 템플릿화 시킨다. Go가 몇가지 빌트인 기능들을 가지고 있어서 Helm은 더 많은 것들을 추가해 두었다.

먼저, [Sprig library](https://godoc.org/github.com/Masterminds/sprig)에 있는 거의 모든 기능을 추가하였다. 여기서 두가지를 보안상 문제로 삭제하였다: `env`와 `expandenv` (이는 차트 작성자가 Tiller 환경에 접속할 수 있도록 한다.)

또한 두가지 템플릿 기능을 추가하였다: `include` 와 `required`. `include`는 다른 템플릿으로부터 가져올 수 있는 기능을 사용할 수 있고 그 결과를 다른 템플릿 기능들에 전달할 수 있다.

예를 들어 이 템플릿의 일부분은 `mytpl`이라는 템플릿을 include하고, 그 결과를 소문자로 변형하여 쌍따옴표로 감싼다.

```yaml
value: {{ include "mytpl" . | lower | quote }}
```

`required` 기능은 특정한 값을 템플릿 렌더링 시 반드시 필요하다고 설정하는 기능이다. 만약 그 값이 비어있다면 템플릿 렌더링은 사용자에게 에러 메시지를 보여주며 실패할 것이다.

다음의 예시는 `required` 기능을 `.Values.who`에 대해 선언하여 필수 요소라고 지정한 것이며 엔트리가 비어있다면 에러를 출력할 것이다.

```yaml
value: {{ required "A valid .Values.who entry required!" .Values.who }}
```

`include` 기능을 사용할 때, 현재 컨텍스트에서 `dict` 기능을 이용하여 생성이 된 커스텀 오브젝트 트리를 보낼 수 있다.

```yaml
{{- include "mytpl" (dict "key1" .Values.originalKey1 "key2" .Values.originalKey2) }}
```

## Quote Strings, Don't Quote Integers

string 데이터를 가지고 작업을 할 때 무조건 따옴표로 감싸는 것이 안전하다.

```yaml
name: {{ .Values.MyName | quote }}
```

그러나 integer로 작업할 때에는 따옴표를 사용하면 안된다. 이는 많은 경우 쿠버네티스에서 파싱 에러를 발생시킨다.

```yaml
port: {{ .Values.Port }}
```

이는 integer로 표현하지만 string으로 작성해야 하는 env 값들에 대해서는 적용되지 않는다.

## Using The "Include" Function

Go는 하나의 템플릿을 빌트인 `template`에 직접 include하는 기능을 제공한다. 하지만 빌트인 기능은 Go 템플릿 파이프 라인에서는 사용할 수 없다.

템플릿을 include하는 것을 가능하게 하여 템플릿의 결과를 수행하기 위해 Helm은 특별한 `inlucde` 기능을 제공한다.

```gotpl
{{- include "toYaml" $value | nindent 2 }}
```

위의 예시는 `toYaml`이라는 템플릿을 include하고 이를 `$value`로 전달하여 `nindent` 기능에 그 템플릿을 전달한다. `{{- ... | nindent _n_}}` 패턴을 사용하는 것은 공백을 지우고 `nindent`가 새로운 줄을 다시 생성하여 인덴트가 원하는 만큼 들어가 있는 방식으로 재생성하여 `include` 컨텍스트를 더 쉽게 읽을 수 있도록 한다.

YAML이 인덴테이션 레벨과 공백을 매우 중요시 여기기 때문에 이는 코드의 일부분을 include하면서 상대적인 인데트를 맞춰주는 데 굉장히 좋다.

## Using The "Required" Function

Go는 키로 인덱싱된 맵이 실제 맵 안에 없을 경우의 동작에 대해서 동작할 템플릿 옵션을 설정할 수 있다. 이는 보통 템플릿과 함께 지정이 되어 있다. 옵션("missingkey=option")은 기본값, zero, 또는 에러일 수 있다. 이 옵션을 에러로 선택하는 것은 실행을 에러와 함께 멈추도록 하지만 모든 맵의 missing key에 대해 적용할 수 있다. 차트 개발자는 이런 행동을 values.yaml에서 몇가지 선택된 값들에 대해서만 동작하도록 하고싶을 수도 있다.

`required` 기능은 개발자로 하여금 그 값이 템플릿을 렌더링할 때 반드시 필요한 값이라고 지정할 수 있도록 한다. 만약 엔트리가 values.yaml에 없다면 템플릿은 렌더링되지 않을 것이고 개발자가 제공한 에러메시지를 리턴할 것이다.

예를 들어:

```gotpl
{{ required "A valid foo is required!" .Values.foo }}
```

다음은 `.Values.foo`가 정의되어 있다면 렌더링 될 것이고 정의되어 있지 않다면 렌더링에 실패하여 종료될 것이다.

## Using The "TPL" Function

`tpl` 기능은 개발자가 템플릿 내에서 string을 템플릿으로 인식하게 만들 수 있다. 템플릿 string을 값으로 차트나 external configuration 파일에 넘기는 것은 유용할 수 있다. 문법은 `{{ tpl TEMPLATE_STRING VALUES}}` 이다.

예시

```yaml
# values
template: "{{ .Values.name }}"
name: "Tom"

# template
{{ tpl .Values.template . }}

# output
Tom
```

external configuration 파일 렌더링하기

```yaml
# external configuration file conf/app.conf
firstName={{ .Values.firstName }}
lastName={{ .Values.lastName }}

# values
firstName: Peter
lastName: Parker

# template
{{ tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```

## Creating Image Pull Secrets

이미지 풀 시크릿은 registry, username, password가 필요하다. 사용자는 어플리케이션을 배포할 때 이런 것들을 사용할 필요가 있을수도 있지만 이를 생성하려면 base64를 여러번 실행해야 한다. 우리는 헬퍼 템플릿을 작성하여 Docker configuration file을 생성하여 시크릿의 payload로 사용할 수 있다. 다음은 예시이다.

먼저, credentials가 `values.yaml`파일에 다음과 같이 정의되어 있다고 가정하자.

```yaml
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

우리는 헬퍼 템플릿을 다음과 같이 정의할 것이다.

```gotpl
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```

결과적으로 헬퍼 템플릿을 이용하여 시크릿 manifest를 생성하는 큰 템플릿을 만들 수 있다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

## Automatically Roll Deployments When ConfigMaps Or Secrets Change

대부분의 경우에 configmaps나 secrets는 컨테이너 안에서 설정 파일로 주입되어 있다. 재시작이 필요한 어플리케이션들은 이러한 것들을 `helm upgrade`로 업데이트할 수 있지만 deployment의 스펙이 바뀌지 않는다면 어플리케이션은 계속해서 이전에 설정한 값들을 가지고 동작할 것이며 이는 원하지 않은 배포가 될 것이다.

`sha256sum` 기능은 deployment의 어노테이션 섹션이 다른 파일들이 바뀜으로써 업데이트가 되지 않았음을 확인시켜준다.

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

이런 문제를 해결하는 약간 다른 방법인 `helm upgrade --recreate-pods` 플래그도 확인해 보아라.

## Tell Tiller Not To Delete A Resource

때로 Helm이 `helm delete`를 수행할 때 삭제되면 안되는 리소스가 있을 수 있다. 차트 개발자는 어노테이션을 추가하여 삭제될 때 리소스가 삭제되는 것을 막을 수 있다.

```yaml
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

(따옴표는 필수이다.)

`"helm.sh/resource-policy": kepp`은 Tiller가 `helm delete` 동작을 하는 동안 이 리소스는 무시하라고 지시한다. 하지만 리소스는 고아상태가 된다. Helm은 더이상  이를 관리하지 않는다. 이는 `helm install --replace`를 사용하여 이미 삭제된 릴리즈지만 리소스를 가지고 있는 것에 대해 설치 시 문제를 유발할 수 있다.

차트의 기본 어노테이션들을 오버라이딩 하기 위해 명시적으로 리소스를 삭제하려 할 경우 resource policy 어노테이션을 `delete`로 해라.

## Using "Partials" And Template Includes

때로 사용자는 차트에서 재사용할 수 있는 블럭이나 템플릿 일부분을 생성하고 싶을 것이다. 그리고 이를 고유의 파일로 가지고 있는 것이 더 깔끔하다.

`templates/` 디렉토리에서 underscore(`_`)로 시작하는 파일들은 쿠버네티스의 manifest 파일이 되지 않는다. 따라서 편의상 헬퍼 템플릿과 부분들은 `_helpers.tpl` 파일에 기록된다.

## Complex Charts With Many Dependencies

오피셜 차트 레파지토리에 있는 많은 차트들은 더 좋은 어플리케이션을 만들기 위해 블럭으로 만들어져 있다. 그러나 차트는 하나의 큰 규모의 어플리케이션으로도 인스턴스를 생성할 수도 있다. 이런 경우에는 하나의 umbrella 차트는 여러개의 서브차트를 가질 수 있고 각각의 차트는 전체의 일부분으로 동작한다.

여러 분리된 파트로부터 복잡한 어플리케이션을 구성하는 좋은 예시는 최상위 레벨의 umbrella 차트를 생성하여 global 설정들을 표현하고 이를 `charts/` 서브디렉토리ㅔㅇ서 사용하여 각 컴포넌트에 내장시키는 것이다.

두가지 디자인 패턴이 이 프로젝트에 대해 기술되어 설명되어 있다.

SAP's [Converged charts](https://github.com/sapcc/helm-charts) : 이 차트는 쿠버네티스의 오픈스택 IaaS인 SAP Converged Cloud를 설치하는 차트이다. 모든 차트는 몇가지 서브모듈을 제외하고는 함께 GitHub 레파지토리에 저장되어 있다.

Deis's [Workflow](https://github.com/deis/workflow/tree/master/charts/workflow) : 이 차트는 전체 Deis PaaS system을 하나의 차트로 묶은 것이다. 그러나 이는 SAP 차트와 다르게 umbrella 차트가 각 컴포넌트에서 만들어 졌으며 각 컴포넌트는 각각의 Git 레파지토리에서 추적이 된다. `requirements.yaml` 파일을 확인하여 어떻게 차트가 그들의 CI/CD 파이프라인을 구축했는지 확인해 보아라.

이 두가지 차트는 헬름을 통해 복잡한 환경을 어떻게 구성하는지 입증된 방식을 설명한다.

## YAML Is A Superset Of Json

YAML 규격에 따르면 YAML은 JSON의 상위개념이다. 이는 유효한 JSON 구조체가 있으면 이를 YAML에서도 유효다는 것을 의미한다.

이는 다음과 같은 이점이 있다: 가끔 템플릿 개발자는 데이터 구조를 YAML같은 공백에 민감한 문법보다는 JSON같은 문법으로 작성하는 것이 더 쉬울 수 있다.

JSON 문법이 변환 시 위험을 상당히 줄여주지 않는 한 YAML으로 템플릿을 작성하는 것이 가장 좋다.

## Be Careful With Generating Random Values

헬름에는 랜덤 데이터, 암호화 키같은 것들을 생성하는 기능이 있다. 이는 사용해도 좋다. 하지만 업데이트를 할 때 템플릿은 재실행됨에 유의하라. 템플릿이 지난번의 실행과는 다른 데이터를 생성할 경우 이는 리소스의 업데이트를 유발시킬 수 있다.

## Upgrade A Release Idempotently

설치와 업그레이드를 같은 명령어로 하고 싶으면 다음의 명령어를 사용하면 된다.

```shell
helm upgrade --install <release name> --values <values file> <chart directory>
```

# The Chart Repository Guide

이 섹션은 어떻게 헬름 차트 레파지토리를 만들고 사용하는지에 대해 알려줄 것이다. 상위 레벨에서 차트 레파지토리는 패키징 된 차트가 저장되고 공유될 수 있는 장소이다.

공식 차트 레파지토리는 Helm Charts에서 유지되고 있고 어떤 참여도 환영한다. 하지만 헬름은 또한 자신만의 차트 레파지토리를 생성하고 활용하기 쉽도록 만들기도 했다. 이 가이드는 어떻게 하는지에 관한 것이다.

## Prerequistes

* Quickstart 가이드를 볼 것
* Charts 문서를 쭉 읽어볼 것

## Create A Chart Repository

차트 레파지토리는 HTTP 서버로 `index.yaml` 파일에서 사용할 수 있으며 몇가지 패키징된 차트를 가질 수 있다. 차트를 공유할 준비가 되었다면 그 차트를 차트 레파지토리에 업로딩하는 것이 좋다.

> Note : 헬름 2.0.0부터 차트 레파지토리는 어떤 고유 인증도 필요하지 않다. 깃허브에 [이슈 트래킹](https://github.com/helm/helm/issues/1038)이 있다.

차틀 레파지토리가 YAML이나 tar 파일을 제공할 수 있고 GET 요청에 응답할 수 있는 어떤 HTTP도 될 수 있기 때문에 자신의 차트 레파지토리를 호스팅 하는 데에 엄청나게 많은 옵션들이 있다. 예를 들어 Google Cloud Storage(GCS) bucket을 사용할 수도 있고, Amazon S3 bucket, Github Pages, 또는 자신의 web server도 사용 가능하다.

### The chart repository structure

차트 레파지토리는 패키징된 차트와 레파지토리 내의 모든 차트의 인덱스를 포함한 `index.yaml`이라고 불리는 특정 파일로 구성되어 있다. 보통 `index.yaml`이 설명하는 차트는 같은 서버에 [provenance files](https://helm.sh/docs/developing_charts/#helm-provenance-and-integrity)로 호스팅 되어 있다.

예를 들어, `https://example.com/charts` 레파지토리의 레이아웃은 다음과 같을 것이다.

```
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```

이 경우에 인덱스 파일은 Alpine 차트 하나의 정보를 담고 있고 그 차트에 대한 다운로드 URL은 `https://example.com/charts/alpine-0.1.2.tgz`로 제공될 것이다.

차트 패키지가 `index.yaml`파일과 같은 서버에 위치할 필요는 없다. 하지만 이것이 가장 쉬운 방법이다.

### The index file

인덱스 팡리은 `index.yaml`이라고 불린다. 이는 `Chart.yaml` 파일의 내용을 포함하여 패키지에 관한 메타데이터를 포함하고 있다. 유효한 차트 레파지토리는 반드시 인덱스 파일을 가지고 있어야 한다. 인덱스 파일은 차트 레파지토리에서 각 차트에 대한 정보를 가지고 있다. `helm repo index` 명령어는 패키징 된 차트가 있는 로컬 디렉토리를 기반으로 인덱스 파일을 생성해줄 것이다.

다음은 인덱스 파일의 예시이다.

```yaml
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://k8s.io/helm
      name: nginx
      sources:
      - https://github.com/helm/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```

생성된 인덱스와 패키지들은 일반 웹 서버로부터 제공될 수 있다. `helm serve` 명령어로 로컬 서버를 시작하여 이를 로컬에서 테스트 할 수 있다.

```shell
$ helm serve --repo-path ./charts
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

## Hostring Chart Repositories

이 파트는 차트 레파지토리를 제공하는 몇가지 방법을 보여준다.

### ChartMuseum

헬름 프로젝트는 차트 뮤지엄이라고 불리는 스스로 호스팅 할 수 있는 오픈소스 헬름 레파지토리 서버를 제공한다.

차트 뮤지엄은 여러 클라우드 스토리지 백엔드를 지원한다. 이를 차트 패키지를 포함하고 있는 디렉토리나 버켓을 가리키도록 설정하면 index.yaml 파일이 동적으로 생성이 될 것이다.

이는 Helm Chart를 통해 쉽게 배포할 수 있다.

```shell
helm install stable/chartmuseum
```

그리고 Docker 이미지로도 가능하다.

```shell
docker run --rm -it \
  -p 8080:8080 \
  -v $(pwd)/charts:/charts \
  -e DEBUG=true \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  chartmuseum/chartmuseum
```

그 후 로컬 레파지토리 리스트에 이 레파지토리를 추가할 수 있다.

```shell
helm repo add chartmuseum http://localhost:8080
```

차트 뮤지엄은 차트 업로드를 위한 API를 제공하는 등의 다른 기능들을 가지고 있다. 자세한 사항은 [README](https://github.com/helm/chartmuseum) 참고.

### Google Cloud Storage

첫 단계는 GCS bucket을 생성하는 것이다. 이를 `fantastic-charts`라고 이름지어 보겠다.

![create-a-bucket](img\create-a-bucket.png)

그 다음 bucket을 editing the bucket permissions를 선택하여 public으로 만들어라.

![edit-permissions](img\edit-permissions.png)

bucket을 public으로 만들기 위해 다음의 아이템을 추가하라.

![make-bucket-public](img\make-bucket-public.png)

이제 차트를 제공할 수 있는 빈 GCS bucket이 생성되었다.

Google Cloud Storage 명령줄 도구나 GCS web UI를 이용하여 차트 레파지토리를 업로드 하고 싶을 것이다. 이런 것들은 공식 쿠버네티스 차트 레파지토리가 차트를 호스팅하는 방법이기도 하며 중간에 막히게 되면 [project](https://github.com/helm/charts)를 참조하면 될 것이다.

> Note : public GCS bucket은 간단하게 이 주소로 HTTPS 통신이 가능하다. `https://bucket-name.storage.googleapis.com/`

### JFrog Artifactory

JFrog Artifactory를 통해서 차트 레파지토리를 만들 수 있다. 자세한 내용은 [여기](https://www.jfrog.com/confluence/display/RTF/Helm+Chart+Repositories)를 참고하라.

### ProGet

헬름 차트 레파지토리는 ProGet을 지원한다. 더 자세한 정보는 Inedo 웹사이트의 [Helm repository documentation](https://inedo.com/support/documentation/proget/feeds/helm)를 확인해 보아라.

### Github Pages example

GitHub Pages를 이용하여 차트 레파지토리를 비슷한 방법으로 만들 수 있다.

GitHub는 두가지 방식으로 static 웹 페이지를 제공할 수 있도록 해준다.

* 프로젝트에서 `docs/` 디렉토리 내의 내용들을 제공하는 것으로 설정
* 프로젝트에서 특정한 브랜치를 제공하는 것으로 설정

첫번째 방식은 매우 쉽기 때문에 우리는 두번째 방식을 사용할 것이다.

가장 첫 단계는 gh-pages 브랜치를 생성하는 것이다. 로컬에서 다음과 같이 하면 된다.

```shell
$ git checkout -b gh-pages
```

또는 Github repository의 웹 브라우저의 Branch 버튼을 통해 생성할 수 있다.

![create-a-gh-page-button](img\create-a-gh-page-button.png)

그 다음 레파지토리에서 Settings를 클릭하고 스크롤을 내려 Github pages 섹션에서 아래와 같이 설정을 하여 gh-pages branch가 Github Pages로 설정되도록 할 수 있다.

![set-a-gh-page](img\set-a-gh-page.png)

기본적으로 Source는 gh-pages branch로 설정되어 있다. 기본값이 설정이 되어 있지 않다면 이를 선택해 주어라.

원한다면 custom domain을 설정할 수 있다.

HTTPS만으로 통신하도록 체크하였으면 차트가 제공될 때 HTTPS를 사용하게 될 것이다.

이런 셋업에서 master branch를 차트의 코드를 담는 데 사용하고 gh-pages branch를 차트 레파지토리를 담든데 사용할 수 있다. 예를 들어 `https://USERNAME.github.io/REPONAME`. 데모 레파지토리인 [TS Charts](https://github.com/technosophos/tscharts)는 `https://technosophos.github.io/tscharts/`로 접근 가능하다.

### Ordinary web servers

헬름 차트를 제공하기 위한 일반적인 웹 서버를 설정하기 위해서 단지 다음의 것들만 필요할 것이다.

* 제공할 서버의 디렉토리에 인덱스와 차트를 넣는다.
* `index.yaml` 파일이 어떤 인증 절차도 없이 접근 가능하도록 한다.
* `yaml` 파일들이 적절한 content type으로 제공되어야 한다. (`text/yaml`이나 `text/x-yaml`)

예를 들어, `$WEBROOT/charts`로 차트를 제공하고자 한다면 웹 루트에 `charts/` 디렉토리가 있어야 하고, 인덱스 파일과 차트들이 그 폴더 내에 있어야 한다.

## Managing Chart Repositories

이제 차트 레파지토리를 가지게 되었다. 이 가이드의 마지막 파트는 어떻게 레파지토리에서 차트를 유지보수하는지 설명할 것이다.

### Store charts in your chart repository

이제 차트 레파지토리를 가지게 되었으므로 차트와 인덱스 파일을 레파지토리에 올려보도록 하자. 차트 레파지토리의 차트는 반드시 패키징(`helm package chart-name/`) 되어야 하고 올바르게 버전화(`SemVer2` 가이드를 따르도록) 되어야 한다.

이 다음 절차는 아래의 워크 플로우로 구성이 되어 있지만 차트 레파지토리에 차트를 저장하는 어떤 저장 및 업데이트 워크플로우든 가능하다.

패키징 된 차트가 있다면 새로운 디렉토리를 만들어서 패키징 된 차트를 그 디렉토리로 옮긴다.

```shell
$ helm package docs/examples/alpine/
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
$ helm repo index fantastic-charts --url https://fantastic-charts.storage.googleapis.com
```

마지막 명령은 방금 생성한 로컬 디렉토리의 경로와 원격 차트 레파지토리의 URL을 의미하고 주어진 디렉토리 경로안에 `index.yaml` 파일이 있어야 한다.

이제 차트와 인덱스 파일을 차트 레파지토리에 sync 툴이나 수동으로 업로드 할 수 있다. 만약 Google Cloud Storage를 사용한다면 gsutil 클라이언트를 사용하는 [example workflow](https://helm.sh/docs/developing_charts/#developing_charts_sync_example)를 참조하라. GitHub라면 간단히 차트를 적절한 브런치로 보내면 된다.

### Add new charts to an existing repository

레파지토리에 새로운 차트를 추가하고 싶을 때마다 인덱스를 생성해야 한다. `helm repo index` 명령어는 완전히 `index.yaml` 파일을 로컬에서 찾을 수 있는 차트들만을 포함하여 스크래치로부터 재생성한다.

하지만 `--merge` 플래그를 사용하여 이미 존재하고 있는 `index.yaml`에 새로운 차트를 점직적으로 추가할 수도 있다.(GCS와 같은 원격 레파지토리를 사용할 때 좋은 옵션이다.) `helm repo index --help`를 통해 더 자세하게 배워라.

수정된 파일과 차트가 모두 업로드 되어야 한다. provenance 파일도 생성하였다면 이 역시 업로드 하라.

### Share your charts with others

차트를 남들과 공유할 준비가 되었다면 그 레파지토리의 URL을 알려주면 된다.

거기로부터 그들은 레파지토리를 헬름 클라이언트에 `helm repo add [NAME] [URL]}`를 통해 그 레파지토리를 참조할 때 사용할 아무 이름을 주어서 추가할 수 있다. 

```shell
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

차트들이 HTTP 기반 인증을 한다면 username과 password도 제공할 수 있다.

```shell
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com --username my-username --password my-password
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

> Note : 레파지토리가 유효한 `index.yaml`이 없다면 레파지토리는 추가되지 않을 것이다.

그 후에 사용자는 차트를 검색할 수 있다. 레파지토리를 업데이트 하고 나면 `helm repo update` 명령어로 최신의 차트 정보를 가지고 올 수 있다.

내부에서 `helm repo add`와 `helm repo update` 명령어는 `index.yaml` 파일을 fetching하고 이들을 `$HELM_HOME/repository/cache/` 디렉토리로 저장한다. 이는 `helm search` 기능이 차트에 대한 정보를 검색할 때 사용하는 장소이다.

# Syncing Your Chart Repository

> 이 예시는 차트 레파지토리를 제공하는 Google Cloud Storage(GCS) bucket을 기준으로 설명한다.

## Prerequisites

* `gsutil` 을 설치한다. gsutil rsync 기능에 의존한다.
* Helm binary에 접근할 수있도록 한다.
* _Optional : GCS bucket에 대해 실수로 뭔가를 지울 것을 대비하여 object versioning 을 추천한다.

## Set Up A Local Chart Repository Directory

[the chart repository guide](https://helm.sh/docs/developing_charts/#developing_charts)에서 한 것과 같이 로컬 디렉토리를 생성하고 패키징 된 차트를 그 디렉토리로 옮긴다.

예를 들어

```shell
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
```

## Generate An Updated `index.yaml`

헬름을 사용하여 디렉토리 경로와 원격 레파지토리의 url을 전달하여 업데이트 된 index.yaml파일을 생성한다. `helm repo index` 명령어는 다음과 같을 것이다.

```shell
$ helm repo index fantastic-charts/ --url https://fantastic-charts.storage.googleapis.com
```

이는 업데이트된 index.yaml 파일을 생성할 것이고 이를 `fantastic-charts/`에 저장할 것이다.

## Sync Your Local And Remote Chart Repositories

`scripts/sync-repo.sh`를 실행하여 GCS bucket의 디렉토리에 내용들을 업로드 하고 로컬 디렉토리 이름과 GCS bucket 이름을 전달한다.

예를 들어

```shell
$ pwd
/Users/funuser/go/src/github.com/helm/helm
$ scripts/sync-repo.sh fantastic-charts/ fantastic-charts
Getting ready to sync your local directory (fantastic-charts/) to a remote repository at gs://fantastic-charts
Verifying Prerequisites....
Thumbs up! Looks like you have gsutil. Let's continue.
Building synchronization state...
Starting synchronization
Would copy file://fantastic-charts/alpine-0.1.0.tgz to gs://fantastic-charts/alpine-0.1.0.tgz
Would copy file://fantastic-charts/index.yaml to gs://fantastic-charts/index.yaml
Are you sure you would like to continue with these changes? [y/N]} y
Building synchronization state...
Starting synchronization
Copying file://fantastic-charts/alpine-0.1.0.tgz [Content-Type=application/x-tar]...
Uploading   gs://fantastic-charts/alpine-0.1.0.tgz:              740 B/740 B
Copying file://fantastic-charts/index.yaml [Content-Type=application/octet-stream]...
Uploading   gs://fantastic-charts/index.yaml:                    347 B/347 B
Congratulations your remote chart repository now matches the contents of fantastic-charts/
```

## Updating Your Chart Repository

차트 레파지토리의 로컬 복사본을 보관하거나 `gsutil rsync`를 통해 원격 차트 레파지토리의 내용들을 로컬 디렉토리에 복사하고 싶을 것이다.

예를 들어

```shell
$ gsutil rsync -d -n gs://bucket-name local-dir/    # the -n flag does a dry run
Building synchronization state...
Starting synchronization
Would copy gs://bucket-name/alpine-0.1.0.tgz to file://local-dir/alpine-0.1.0.tgz
Would copy gs://bucket-name/index.yaml to file://local-dir/index.yaml

$ gsutil rsync -d gs://bucket-name local-dir/       # performs the copy actions
Building synchronization state...
Starting synchronization
Copying gs://bucket-name/alpine-0.1.0.tgz...
Downloading file://local-dir/alpine-0.1.0.tgz:                        740 B/740 B
Copying gs://bucket-name/index.yaml...
Downloading file://local-dir/index.yaml:                              346 B/346 B
```

도움이 되는 링크 : [gsutil rsync](https://cloud.google.com/storage/docs/gsutil/commands/rsync#description)의 문서, [The Chart Repository Guide](https://helm.sh/docs/developing_charts/#developing_charts), Google Cloud Storage의 [object versioning and concurrency control](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#overview) 문서

# Helm Provenance and Integrity

헬름은 provenance tool이 있어서 차트 유저가 패키지의 원 출처와 integrity를 검증할 수 있도록 한다. PKI, GnuPG, 높이 평가되는 패키지 매니저들을 기반으로 한 산업 표준 툴을 사용하여 헬름은 서명 파일을 생성하고 검증할 수 있다.

## Overview

Integrity는 차트와 provenance 기록을 비교하여 평가된다. provenance record는 패키징된 차트와 함께 저장되어 있는 provenance files에 저장되어 있다. 예를 들어 차트의 이름이 `myapp-1.2.3.tgz`라면 provenance file은 `myapp-1.2.3.tgz.prov`파일이 될 것이다.

provenance file은 패키징 할 때(`helm package --sign ...`) 생성이 된고 특히 `helm install --verify`같은 여러 명령어들에서 확인된다.

## The Workflow

이 섹션은 provenance 데이터를 효과적으로 사용할 수 있을만한 워크 플로우를 설명한다.

Prerequisites

* 바이너리(ASCII-armored가 아닌) 포멧으로 유효한 PGP keypair
* `helm` 명령줄 도구
* GnuPG >= 2.1 명령줄 도구 (옵션)
* Keybase 명령줄 도구 (옵션)

> Note : PGP private key가 passphrase를 가지고 있다면 `--sign` 옵션을 지원하는 모든 명령어에 대해서 그 passphrase를  입력하도록 할 것이다. HELM_KEY_PASSPHRASE 환경 변수를 설정하여 passphrase를 일일히 입력하지 않도록 할 수도 있다.

> Note : GnuPG의 keyfile 포멧이 2.1버전부터 변경되었다. 이전 릴리즈들은 GnuPG에서 key를 export하는 것을 필요로 하지 않았고, 대신 헬름이 `*.gpg` 파일을 가르키도록 할 수 있었다. 2.1부터 `.kbx` 포멧이 소개되었고, 이 포멧은 헬름에서 지원하지 않는다.

새로운 차트를 이전과 같이 생성한다.

```shell
$ helm create mychart
Creating mychart
```

패키징이 준비가 되면 `--sign` 플래그를 `helm package`에 추가한다. 또한 어떤 서명 키들이 알려져있는 이름을 지정할 수 있고 private key에 대응되는 keyring을 지정할 수 있다.

```shell
$ helm package --sign --key 'helm signing key' --keyring path/to/keyring.secret mychart
```

> Tip : GnuPG 유저라면 시크릿 keyring은 `~/.gnupg/secring.kbx`에 존재한다. `gpg --list-secret-keys`를 사용하여 가지고 있는 키들의 목록을 볼 수 있다.

> Warning : GnuPG v2.1은 시크릿 키링을 새로운 포멧인 'kbx'를 기본으로 '~/.gnupg/pubring.kbx'에 저장한다. 이 키링들을 gpg 포멧으로 변경하기 위해 다음의 명령어를 사용하라.

```shell
$ gpg --export-secret-keys >~/.gnupg/secring.gpg
```

여기서 `mychart-0.1.0.tgz`와 `mychart-0.1.0.tgz.prov` 파일 모두를 확인해야 한다. 두 파일은 원하는 차트 레파지토리에 업로드가 되어 있어야 한다.

차트를 `helm verify` 명령어를 통해서 검증할 수 있다.

```shell
$ helm verify mychart-0.1.0.tgz
```

검증이 실패하면 다음과 같을 것이다.

```shell
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

설치가 진행되는 동안 검증을 하기 위해서는 `--verify` 플래그를 사용하면 된다.

```shell
$ helm install --verify mychart-0.1.0.tgz
```

서명된 차트에 대한 public key를 가지고 있는 키링이 기본 위치에 없다면 이 키링을 `--keyring PATH`로 `helm package` 예시처럼 지정해주어야 한다.

검증이 실패하면 차트가 Tiller로 보내지기 전에 설치가 중단된다.

### Using Keybase.io credentials

Keybase.io 서비스는 암호화 아이디에 대한 신뢰 체인을 쉽게 설정할 수 있게 한다. Keybase credentials는 차트를 서명할 때 사용할 수 있다.

Prerequisites

* Keybase.io 계정
* GnuPG가 로컬에 설치되어 있음
* `keybase`CLI가 로컬에 설치되어 있음

### Signing packages

첫 단계는 eybase keys를 local GnuPG keyring에 임포트하는 것이다.

```
$ keybase pgp export -s > secring.gpg
```

이는 Keybase key를 OpenPGP 포멧으로 변환시킬 것이고 이를 `secring.gpg`파일로 로컬에 위치시킬 것이다.

> Tip : Keybase key를 이미 있는 keyring에 추가하고 싶다면 `keybase pgp export -s | gpg --import && gpg --export-secret-keys --outfile secring.gpg`를 하면 된다.

시크릿 키는 ID string을 가지게 될 것이다.

```
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

이는 키의 전체 이름이다.

그 다음 `helm package`로 차트를 서명하고 패키징 할 수 있다. 최소한 그 이름 string의 일부를 `--key`에서 사용해야 한다.

```shell
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

결과적으로 `package` 명령어는 `.tgz` 파일과 `.tgz.prov` 파일 모두를 생성할 것이다.

#### Verifying packages

비슷한 테크닉으로 다른 사람이 Keybase key로 서명한 차트를 인증할 수도 있다. `keybase.io/technosophos`로 서명이 된 패키지를 인증하려 한다고 해보자. 이를 하기 위해서는 `keybase` 도구를 사용하면 된다.

```shell
$ keybase follow technosophos
$ keybase pgp pull
```

첫 명령어는 `technosophos` 유저를 추적한다. 그 다음 `keybase pgp pull`은 사용자가 팔로우하는 OpenPGP 키를 다운로드 하고 이를 GnuPG keyring에 위치시킨다. (`~/.gnupg/pubring.gpg`)

이제부터 `helm verify`를 사용하거나 `--verify` 플래그를 사용할 수 있게 되었다.

```shell
$ helm verify somechart-1.2.3.tgz
```

### Reasons a chart may not verify

실패하는 몇가지 이유가 있다.

* prov 파일이 없거나 충돌한다. 이는 어떤 것이 잘못 설정되었거나 원작자가 provenance 파일을 생성하지 않았을 때 나타난다.
* 파일에 서명하기 위한 키가 키링에 없다. 이는 차트를 서명한 사람이 이전에 믿을 수 있다고 인정되는 사람이 아닐 때 나타난다.
* prov 파일의 인증이 실패한다. 이는 차트나 provenance 데이터 모두가 뭔가 잘못되었을 때 발생한다.
* provenance 파일의 해쉬가 아카이브 파일의 해쉬와 맞지 않는 경우. 이는 아카이브가 조작되었음을 나타낸다.

검증이 실패하게 되면 패키지를 믿을 수 없는 이유가 있는 것이다.

## The Provenance File

provenance 파일은 차트의 YAML파일에 몇가지 검증 정보들을 포함한 것이다. Provenance 파일은 자동으로 생성되도록 디자인되어 있다.

다음의 것들이 provenance 데이터에 추가가 된다.

* 차트 파일(Chart.yaml)은 인간과 툴 모두에게 차트의 내용을 쉽게 볼 수 있도록 포함되어 있다.
* 차트 패키지(.tgz 파일)의 서명(도커에서 처럼 SHA256)이 포함되고 차트 패키지의 integrity를 검증할 때 사용될 수 있다.
* 전체 바디는 PGP에 의한 알고리즘으로 서명이 된다.(암호 서명 및 확인을 쉽게하는 방법은 https://keybase.io를 참조하라.)

이를 조합하면 사용자는 다음과 같은 보장을 얻을 수 있다.

* 패키지가 조작되지 않았다.(checksum package tgz)
* 패키지를 릴리즈한 사람이 알려진 사람이다. (GnuPG/PGP 서명을 통해서)

파일의 포멧은 다음과 같다.

```yaml
-----BEGIN PGP SIGNED MESSAGE-----
name: nginx
description: The nginx web server as a replication controller and service pair.
version: 0.5.1
keywords:
  - https
  - http
  - web server
  - proxy
source:
- https://github.com/foo/bar
home: https://nginx.com

...
files:
        nginx-0.5.1.tgz: “sha256:9f5270f50fc842cfcb717f817e95178f”
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1.4.9 (GNU/Linux)

iEYEARECAAYFAkjilUEACgQkB01zfu119ZnHuQCdGCcg2YxF3XFscJLS4lzHlvte
WkQAmQGHuuoLEJuKhRNo+Wy7mhE7u1YG
=eifq
-----END PGP SIGNATURE-----
```

YAML 섹션이 두개의 문서(`...\n`을 통해)로 나누어짐을 유의하라. 처음은 Chart.yaml이다. 두번째는 체크섬으로써 파일의 이름을 SHA-256 digests(현재 보이는 값들은 가짜이고 짤려있다.)로 매핑한다.

서명 블럭은 [tamper resistance](https://www.rossde.com/PGP/pgp_signatures.html)를 제공하는 PGP 서명 표준이다.

## Chart Repositories

차트 레파지토리는 헬름 차트의 중앙집중식 저장소이다.

차트 레파지토리는 특정한 요청을 통해 HTTP로 provenance 파일을 제공할 수 있어야 하고 같은 URI에서 차트또한 가능하게 해야 한다.

예를 들어 패키지의 base URL이 `https://example.com/charts/mychart-1.2.3.tgz`가 provenance file이고 존재할 경우 반드시 `https://example.com/charts/mychart-1.2.3.tgz.prov`에 접근할 수 있어야 한다.

엔드 유저 관점에서 `helm install --verify myrepo/mychart-1.2.3`은 차트와 provenance 파일 모두를 추가적인 유저 설정이나 동작 없이 다운로드 할 수 있어야 한다.

## Establishing Authority And Authenticity

chain-of-trust 시스템을 다룰 때 서명자의 authority를 생성하는 것은 중요하다. 또는 쉽게 얘기하자면 그 위의 시스템은 차트의 서명자를 신뢰한다는 사실에 달려있다. 다시 말해서 서명자의 public key를 신뢰해야 한다는 것을 의미한다.

쿠버네티스 헬름의 디자인으로부터 헬름 프로젝트는 필요한 파티에 대해서 chain of trust를 스스로 주입하지 않는다. 우리는 모든 차트 서명자에 대해서 확인된 증명을 원하지는 않는다. 대신에 탈중앙화 된 모델을 강력히 선호하기 때문에 OpenPGP를 기초 기술로 선택하게 된 것이다. 따라서 authority를 발행할 때 헬름 2.0.0에서 제공하지 않는 약간의 단계를 남겨둔 것이다.

하지만 provenance 시스템을 사용하는데 흥미가 있는 사람들에게 몇가지 포인트와 추천할 사항이 있다.

* `Keybase` 플랫폼은 trust information에 대해서 public centralized repository를 제공한다.
  * Keybase를 사용자의 key를 저장하거나 다른 사람의 public key를 가져올 때 사용할 수 있다.
  * Keybase는 훌륭한 문서도 가지고 있다.
  * 아직 테스트해보지는 않았지만 Keybase의 "secure website" 기능은 헬름 차트에서 제공될 수 있을 것이다.
* [official Helm Charts project](https://github.com/helm/charts)는 공식 차트 레파지토리에 대한 문제점을 해결하려고 노력중이다.
  * [detailing the current thoughts](https://github.com/helm/charts/issues/23)에 오래전부터 있던 이슈가 있다.
  * 기본 아이디어는 공식 "차트 리뷰어"가 차트를 그의 key로 서명하고 그 provenance 파일을 차트 레파지토리에 업로드 시키는 것이다.
  * 유효한 signing key의 리스트를 레파지토리의 `index.yaml` 파일에 포함시키는 작업도 있었다.

최종적으로 chain-of-trust는 헬름의 개발중인 기능이며 몇몇 커뮤니티 멤버들은 서명에 OSI 모델을 적용할 것을 제안하고 있다. 이는 헬름 팀에게 공개적인 질문이다. 흥미롭다면 뛰어들어라.

# Charts Tests

차트는 함께 동작하는 여러 쿠버네티스 리소스와 컴포넌트들을 가지고 있다. 차트 제작자로서 차트가 설치 되었을 때 제대로 차트가 동작하는지 확인하기 위해 몇가지 테스트를 작성하고 싶을 것이다. 이런 테스트들은 차트 소비자로 하여금 차트가 어떤 것을 원하는지 이해할 수 있도록 해준다.

헬름 차트에서 테스트는 `templates/` 디렉토리 내에 있고 주어진 명령어를 동작시키는 지정된 컨테이너에 대한 파드 정의 형태이다. 컨테이너는 테스트가 성공적이라고 판명이 되면 0을 리턴하여 성공적으로 종료되어야 한다. 파드 정의는 헬름 테스트 훅 어노테이션 중 하나를 반드시 포함해야 한다: `helm.sh/hook: test-success`나 `helm.sh/hook: test-failure`

테스트 예시 : values.yaml파일로부터 설정한 파일이 적절하게 주입이 되었는지 검증하는 것.  username과 password가 잘 동작하는지 확인하기. 적절하지 않은 username과 password가 제대로 동작하지 않는지. 서비스가 제대로 켜지고 맞게 로드밸런싱 되었는지 등.

릴리즈에 대해 `helm test <RELEASE_NAME>` 명령어를 사용하여 Helm을 pre-defined 테스트 해볼 수 있다. 이를 통해 차트를 이용하는 사람은 차트 또는 어플리케이션의 릴리즈가 원하는대로 동작하는지 체크해 볼 수 있다.

## A Breakdown Of The Helm Test Hooks

헬름에서 두가지의 테스트 훅이 있다: `test-success`와 `test-failure`

`test-success`는 테스트 파드를 성공적으로 완료했음을 표현한다. 다시 말해서 파드의 컨테이너는 0으로 종료해야 한다. `test-failure`는 테스트 파드가 성공적으로 완료되면 안되는 것을 확인한다. 파드의 컨테이너가 0으로 종료하지 않으면 이는 성공한 것이다.

## Example Test

wordpress 차트에 대한 헬름 테스트 파드 정의의 예시가 있다. 테스트는 mariadb 데이터베이스에 접속하는 것을 테스트한다.

```
wordpress/
  Chart.yaml
  README.md
  values.yaml
  charts/
  templates/
  templates/tests/test-mariadb-connection.yaml
```

`wordpress/templates/tests/test-mariadb-connection.yaml`에서

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-credentials-test"
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
  - name: {{ .Release.Name }}-credentials-test
    image: {{ .Values.image }}
    env:
      - name: MARIADB_HOST
        value: {{ template "mariadb.fullname" . }}
      - name: MARIADB_PORT
        value: "3306"
      - name: WORDPRESS_DATABASE_NAME
        value: {{ default "" .Values.mariadb.mariadbDatabase | quote }}
      - name: WORDPRESS_DATABASE_USER
        value: {{ default "" .Values.mariadb.mariadbUser | quote }}
      - name: WORDPRESS_DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ template "mariadb.fullname" . }}
            key: mariadb-password
    command: ["sh", "-c", "mysql --host=$MARIADB_HOST --port=$MARIADB_PORT --user=$WORDPRESS_DATABASE_USER --password=$WORDPRESS_DATABASE_PASSWORD"]
  restartPolicy: Never
```

## Steps To Run A Test Suite On A Release

1. `$ helm install stable/wordpress`

   ```
   NAME:   quirky-walrus
   LAST DEPLOYED: Mon Feb 13 13:50:43 2017
   NAMESPACE: default
   STATUS: DEPLOYED
   ```

2. `$ helm test quirky-walrus`

   ```
   RUNNING: quirky-walrus-credentials-test
   SUCCESS: quirky-walrus-credentials-test
   ```

## Notes

* 많은 테스트 파일들을 하나의 yaml 파일로 정의하거나 `templates/` 디렉토리에 여러 파일로 나누어서 정의할 수 있다.
* 테스트 파일들을 `tests/` 디렉토리 안에 넣어서 `<chart-name>/templates/test/`로 더 격리 시킬 수 있다.



# Chart Repositories: Frequently Asked Questions

이 섹션은 차트 레파지토리를 이용하였을 때 빈번하게 발생할 수 있는 이슈들에 대해 담아 보았다.

이 문서를 더 작성하기 위해 여러분들의 도움이 필요하다. 추가하고, 수정하고, 삭제하기 위해서 `file an issue` 또는 pull requiest를 날려라.

## Fetching

### Q: Why do I get a `unsupported protocol scheme ""` error when trying to fetch a chart from my custom repo?

A: (Helm < 2.5.0) 이는 차트 레포 인덱스를 `--url` 플래그를 지정하지 않고 생성했기 때문에 발생하였을 것이다. `helm repo index --url http://my-repo/charts .`과 같은 명령어로 다시 `index.yaml`을 생성한 후에 다시 custom chart repo에 올려 보아라.

이 동작은 Helm 2.5.0버전에서 수정이 되었다.

