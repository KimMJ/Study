# Using Helm

## Three Big Concepts

### Chart

Chart는 helm 패키지이다. 쿠버네티스 클러스터 내에서 어플리케이션, 툴, 서비스들을 동작시키는데 필요한 모든 리소스를 가지고 있다.

### Repository

Repository는 차트들이 모이고 공유되는 장소이다.

### Release

Release는 쿠버네티스 클러스터 안에서 동작하는 차트들의 인스턴스이다. 하나의 차트는 같은 클러스터에 여러번 설치될 수 있다. 그리고 각 차트가 설치될 때마다 새로운 release가 생성이 된다. MySQL을 생각하면 편하다. 클러스터 내에 두개의 데이터베이스가 필요하다면 차트를 두번 설치하면 된다. 각각의 설치는 각각의 release를 가지고 있어서 각자 고유의 release name을 가진다.



위의 3가지 개념을 가지고 Helm을 이와같이 설명할 수 있다.

Helm은 차트를 쿠버네티스에 설치하고 각 설치마다 새로운 릴리즈를 생성한다. 그리고 새로운 차트를 찾기 위해서는 helm 차트 레파지토리에 검색할 수 있다.

## Helm Search : Finding Charts

처음 헬름을 설치하면 쿠버네티스의 오피셜 차트 레피지토리와 통신하도록 설정이 되어 있다. 이 레파지토리는 잘 유지되고 관리되는 차트들을 가지고 있다. 이 차트는 기본적으로 `stable`이라고 이름지어진다.

`helm search`를 통해서 차트를 검색할 수 있다.

```console
$ helm search
NAME                 	VERSION 	DESCRIPTION
stable/drupal   	0.3.2   	One of the most versatile open source content m...
stable/jenkins  	0.1.0   	A Jenkins Helm chart for Kubernetes.
stable/mariadb  	0.5.1   	Chart for MariaDB
stable/mysql    	0.1.0   	Chart for MySQL
...
```

필터링 없이 `helm search`는 모든 가능한 차트를 보여준다. 이를 필터를 추가해서 범위를 좁힐 수 있다.

```console
$ helm search mysql
NAME               	VERSION	DESCRIPTION
stable/mysql  	0.1.0  	Chart for MySQL
stable/mariadb	0.5.1  	Chart for MariaDB
```

이제 필터와 맞는 결과값만 보여준다.

왜 `mariadb`가 결과에 있을까? 이는 이 패키지가 이를 MySQL과 연관되어있다고 설명했기 때문이다. `helm inspect <chart>`를 통해 이를 확인할 수 있다.

```console
$ helm inspect stable/mariadb
Fetched stable/mariadb to mariadb-0.5.1.tgz
description: Chart for MariaDB
engine: gotpl
home: https://mariadb.org
keywords:
- mariadb
- mysql
- database
- sql
...
```

search는 가능한 패키지를 찾는 좋은 방법이다. 한번 설치하고자 하는 패키지를 찾으면 `helm install`로 이를 설치할 수 있다.

## Helm Install : Installing A Package

새로운 패키지를 설치하려면 `helm install` 커맨드를 사용하면 된다. 간단하게 이는 하나의 파라미터만을 가진다: 차트의 이름

```console
$ helm install stable/mariadb
Fetched stable/mariadb-0.3.0 to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
NAME: happy-panda
LAST DEPLOYED: Wed Sep 28 12:32:28 2016
NAMESPACE: default
STATUS: DEPLOYED

Resources:
==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         0         0            0           1s

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         1s

==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   1s


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

이제 `mariadb`가 설치되었다. 차트를 설치하는 것이 새로운 릴리즈 오브젝트를 생성했음에 주목하라. 이 릴리즈는 위에서 `happy-panda`라고 이름지어졌다. (만약 사용자가 원하는 릴리즈 이름을 주고싶다면 `--name` 플래그를 `helm install`에 사용하라.)

설치하는 동안 `helm` 클라이언트는 어떤 리소스가 생성되었는지, 릴리즈의 상태는 어떠한지, 또 사용자가 해야되거나 할 수 있는 추가적인 설정이 있는지에 대한 유용한 정보들을 보여준다.

helm은 모든 리소스가 동작할때까지 기다리지 않는다. 많은 차트들이 600M가 넘는 사이즈의 도커 이미지를 사용하고 있고 클러스터에 설치하는데 많은 시간이 걸린다.

릴리즈의 상태를 보거나 설정 정보를 다시 보려면 `helm status`를 입력하면 된다.

```console
$ helm status happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   4m

==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         1         1            1           4m

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         4m


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

위에서 릴리즈의 현재 상태를 볼 수 있다.

### Customizing the chart Before Installing

여기서 본 설치 절차는 차트의 기본 설정 옵션만을 사용한다. 많은 경우에 사용자는 선호하는 설정으로 차트를 커스터마이징 하고 싶어한다.

어떠한 옵션들이 차트에서 설정가능한지 보려면 `helm inspect values`를 사용한다.

```console
helm inspect values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: https://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
```

우리는 이러한 설정들을 YAML 포멧 파일로 오버라이드 할 수 있고 이를 설치시에 사용할 수 있다.

```console
$ cat << EOF > config.yaml
mariadbUser: user0
mariadbDatabase: user0db
EOF
$ helm install -f config.yaml stable/mariadb
```

위에서 기본 MariaDB 유저의 이름을 `user0`로 설정하였고 이를 새로 생성한 `user0db` 데이터베이스에 접속할 때 사용하게 하였지만 나머지 값들은 차트의 기본값을 사용하도록 되어있다.

설치할 때 설정 정보를 보내는 두가지 방법이 있다.

* `--values`(또는 `-f`) : 오버라이드할 YAML을 지정한다. 이는 여러번 지정될 수 있으며 가장 오른쪽에 있는 파일이 우선순위를 가진다.
* `--set` (그리고 이의 변형인 `--set-string`과 `--set-file`) : 커멘드 라인에서 오버라이딩 할 수 있다.

둘다 사용할 경우 `--set` 값이 `--values`보다 높은 우선순위로 병합된다. `--set`으로 오버라이딩된 값은 configmap에 남아있게 된다. 릴리즈에 대해서 `--set`으로 설정한 값들은 `helm get values <release-name>`으로 확인할 수 있다. `--set`으로 설정된 값들은 `helm upgrade`를 `--reset-values`로 지정함으로써 삭제할 수 있다.

#### The Format and Limitation of  `--set`

`--set` 옵션은 0개 이상의 name/value 쌍을 가질 수 있다. 간단하게 이는 다음과 같이 사용된다:`--set name=value` YAML은 다음과 같다.

```yaml
name: value
```

여러개의 값은 `,`로 나눌 수 있다. 따라서 `--set a=b,c=d`는 다음과 같다.

```yaml
a: b
c: d
```

더 복잡한 방식의 표현도 사용 가능하다. 예를 들어 `--set outer.inner=value`는 다음과 같이 변환된다.

```yaml
outer:
  inner: value
```

리스트는 `{`과 `}`로 감싸서 표현할 수 있다. 예를 들어 `--set name={a,b,c}`는 다음과 같이 변환된다.

```yaml
name:
  - a
  - b
  - c
```

Helm 2.5.0부터, array index 문법을 사용하여 베열의 아이템에 접근할 수 있다. 예를 들어 `--set servers[0].port=80`은 다음과 같다.

```yaml
servers:
  - port: 80
```

여러 값이 이 방법으로 설정될 수 있다. `--set servers[0].port=80,servers[0].host=example`은 다음과 같이 변형된다.

```yaml
servers:
  - port: 80
    host: example
```

가끔 `--set` 줄에 특수문자를 사용해야 할 수 있다. 백슬래쉬로 이스케이핑하면 된다. `--set name="value1\,value2"`는 다음과 같다.

```yaml
name: "value1,value2"
```

이처럼 차트를 `toYaml` 함수를 사용하여 어노테이션, 라벨, 노드 셀렉터을 파싱할 때 유용할 수 있는 '.' 또한 이스케이핑할 수 있다. `--set nodeSelector."kubernetes\.io/role=master"`에 대한 문법은 다음과 같이 변형된다.

```yaml
nodeSelector:
  kubernetes.io/role: master
```

깊게 연관되어 있는 데이터 구조는 `--set`을 통해서 표현하기 어려울 수 있다. 차트 디자니어는 `--set` 사용을 고려하여 `values.yaml`파일의 포멧을 디자인해야한다.

Helm은 `--set`으로 지정된 특정한 값들을 integer로 변환한다. 예를 들어, `--set foo=true`는 Helm이 `true`를 int64값으로 변환하도록 한다. 이 경우에 만약 string을 원한다면 `--set`의 변형인 `--set-string`을 사용한다. `--set-string foo=true`는 `"true"`라는 string 값이 된다.

`--set-file key=filepath`는 `--set`의 다른 변형이다. 이는 파일에서 불러와서 내용들을 값으로 사용한다. 예시에서 이를 사용하여 YAML에서의 들여쓰기와는 관련 없이 여러 줄의 텍스트를 값에 넣는 예시를 보여준다. `brigade` 프로젝트를 특정한 5개의 줄을 가진 javascript 코드로 생성하고 싶다면 `values.yaml`을 다음과 같이 작성할 수 있다.

```yaml
defaultScript: |
  const { events, Job } = require("brigadier")
  function run(e, project) {
    console.log("hello default script")
  }
  events.on("run", run)
```

YAML에 내장되어 코드를 작성하는 것에 대한 지원은 IDE 기능을 사용하고 프레임와크를 테스트하기 어렵게 만든다.?? 대신 `--set-file defaultScript=brigade.js`를 `brigade.js`와 함께 사용할 수 있다.

```javascript
const { events, Job } = require("brigadier")
function run(e, project) {
  console.log("hello default script")
}
events.on("run", run)
```

### More Installation Methods

`helm install` 커맨드는 몇가지 소스들로부터 설치할 수 있다.

* 차트 레파지토리 (위에서 본 것들)
* 로컬 차트 아카이브 (`helm install foo-0.1.1.tgz`)
* 압축해제된 차트 디렉토리 (`helm install path/to/foo`)
* URL (helm install https://example.com/charts/foo-1.2.3.tgz)

## Helm Upgrade and Helm Rollback : Upgrading a Release and recovering on failure

새로운 버전의 차트가 릴리즈되거나 릴리즈의 설정을 바꾸고 싶으면 `helm upgrade` 커맨드를 사용할 수 있다.

업그레이드는 현재의 릴리즈를 가지고 이를 사용자가 제공한 정보에 맞게 업그레이드 한다. 쿠버네티스 차트가 크고 복잡할 수 있기 때문에 Helm은 최소한으로 업그레이드를 할 것이다. 지난 릴리즈에서 변경된 사항만 업그레이드 된다.

```console
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded.
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

위의 예시에서 `happy-panda` 릴리즈는 같은 차트로 업그레이드 되었지만 새로운 YAML파일과 함께 업그레이드 한다.

```yaml
mariadbUser: user1
```

우리는 `helm get values`로 어떤 세팅이 새로 설정되었는지 확인할 수 있다.

```console
$ helm get values happy-panda
mariadbUser: user1
```

`helm get` 커맨드는 클러스터내에서 릴리즈를 볼 때 유용하다. 위에서 볼 수 있듯이 `panda.yaml`에서의 새로운 값이 클러스터에 배포가 되었다.

이제 릴리즈했을 때 뭔가 계획대로 되지 않았다면 이전 릴리즈로 `helm rollback [RELEASE] [REVISION]`을 통해 롤백할 수 있다.

```console
$ helm rollback happy-panda 1
```

위의 롤백은 happy-panda를 가장 첫 릴리즈 버전으로 롤백시킨다. 릴리즈 버전은 증가하는 revision이다. 설치, 업그레이드, 롤백이 일어날 때마다 revision 수는 1씩 증가한다. 가장 첫 revision은 항상 1이다. 그리고 `helm history [RELEASE]`를 통해 특정 릴리즈에서 revision을 볼 수 있다.

## Helpful Option for Install/Upgrade/Rollback

