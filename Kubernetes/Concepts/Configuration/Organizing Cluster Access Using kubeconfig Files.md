# Organizing Cluster Access Using kubeconfig Files

* kubeconfig 파일을 사용하여 클러스터, 유저, 네임스페이스, 인증 메카니즘을 관리할 수 있다.
  * `kubectl` 커멘트라인 툴은 kubeconfig 파일을 이용하여 어느 클러스터를 선택할지를 찾고 클러스터의 API 서버와 통신한다.

> Note: 클러스터에 접속하기 위해 설정된 파일을 kubeconfig 파일이라고 한다. 이는 설정 파일을 참조하는 일반적인 방법이다. 이는 파일의 이름이 `kubeconfig`라는 것을 의미하는 것이 아니다.

* 기본적으로 `kubectl`은 `$HOME/.kube` 디렉토리에서 `config`라고 이름지어진 파일을 찾는다.
  * `KUBECONFIG` 환경변수 또는 `--kubeconfig` 플래그를 설정해서 다른 kubeconfig 파일을 지정할 수 있다.
* 단계별 명령어나 kubeconfig 파일을 생성 및 지정하는 것은 [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters)을 참조하라.

## Supporting multiple clusters, users, and authentication mechanisms

* 여러개의 클러스터를 가지고 있고 유저와 컴포넌트가 다양한 방법으로 인증을 한다고 생각해보자. 예를 들어:
  * kubelet을 동작시키는 것은 certificates를 사용한 인증을 할 것이다.
  * 유저는 token을 통한 인증을 할 것이다.
  * 관리자는 유저 개인에게 제공하는 인증서들을 가지고 있을 것이다.
* kubeconfig파일은 클러스터, 유저, 네임스페이스를 관리할 수 있도록 한다.
  * 사용자는 contexts를 정의하여 빠르고 쉽게 클러스터와 네임스페이스를 변경할 수 있다.

## Context

* kubeconfig 파일에서의 context 엘리먼트는 그룹화된 접근 파라미터 친숙한 이름을 통해 사용할 수 있도록 한다.

  * 각각의 context는 세가지 파라미터가 있다.
    * 클러스터
    * 네임스페이스
    * 유저
  * 기본적으로 `kubectl` 커맨드라인 툴은 current context에서 파라미터를 가져오고 클러스터와 통신할 때 사용한다.

* current context를 선택하기 위해서는 다음과 같이 입력한다.

  ```shell
  kubectl config use-context
  ```

## The KUBECONFIG environment variable

* `KUBECONFIG` 환경 변수는 kubeconfig 파일들에 대한 리스트를 담고 있다.
  * Linux와 Mac에서는 colon으로 구분되어져 있다.
  * Windows는 세미콜론으로 구분되어져 있다.
  * `KUBECONFIG` 환경 변수는 필수가 아니다.
  * `KUBECONFIG` 환경 변수가 없다면 `kubectl`은 기본 kubeconfig 파일을 사용하며 `$HOME/.kube/config`파일을 사용한다.
* 만일 `KUBECONFIG` 환경 변수가 존재한다면 `kubectl`은 `KUBECONFIG` 환경 변수에 설정되어 있는 파일들을 병합한 것을 사용한다.

## Merging kubeconfig files

* 설정을 보려면 다음의 커멘드를 입력한다.

  ```shell
  kubectl config view
  ```

* 이전에 설명했듯이 결과는 하나의 kubeconfig파일이 될 수도 있고 여러 kubeconfig 파일을 병합한 것이 될수도 있다.

* `kubectl`이 kubeconfig 파일을 병합하는 데에는 다음과 같은 규칙을 따른다.

  * `--kubeconfig` 플래그가 설정되어있으면 지정된 파일만 사용한다. 병합하지 않는다. 이 플래그는 하나의 인스턴스만 허용한다.

* 그렇지 않으면 `KUBECONFIG` 환경 변수가 설정되었을 때 리스트된 파일들은 병합이 된다.

  * `KUBECONFIG` 환경 변수에 있는 파일 목록들은 다음과 같은 규칙으로 병합한다.
    * 빈 파일 이름은 무시한다.
    * deserialized가 불가능한 내용이 있는 파일은 에러를 발생시킨다.
    * 특정한 값이나 key의 매핑은 첫 파일에 적힌 값이 유효하다.
    * 값이나 key 매핑은 절대 변경하면 안된다. 예: `current-context` 설정이 된 context중 첫 파일에 기록된 것을 보호하라. 예: 만약 두 파일이 `red-user`를 지정하고 있다면 첫 파일의 `red-user`의 값만 사용된다. 두번째 파일이 `red-user`에 대해 충돌되지 않는 엔트리들이 있다고 하더라고 이는 버려진다.

* `KUBECONFIG` 환경 변수를 설정하는 예시는 [Setting the KUBECONFIG environment variable](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable)를 참조하라.

* 그렇지 않으면 기본 kubeconfig 파일은 `$HOME/.kube/config`이고 병합되지 않는다.

  * 다음과 같은 체인에서 처음으로 일치하는 컨텍스트를 고른다.
    * `--context` 커맨드라인 플래그가 존재하면 이를 사용한다.
    * 병합된 kubeconfig 파일에서 `current-context`를 사용한다.

* 다음의 경우에 비어있는 context가 가능하다.

  * 클러스터와 유저를 정의한다.
    * 이 경우 context가 있을수도, 없을수도 있다. 
    * 클러스터와 유저를 결정하는 것은 각각 다음의 체인에서 먼저 일치하는 것을 고른다.
      * `--user`나 `--cluster`같은 커맨드라인 플래그가 있으면 이를 쓴다.
      * context가 비어있지 않으면 context에서 유저와 클러스터를 가져온다.

* 유저와 클러스터는 다음의 경우에 비어있을 수 있다.

  * 사용할 실제 클러스터의 정보를 결정한다.
    * 이 관점에서 클러스터 정보가 있을수도, 없을수도 있다.
    * 각 클러스터 정보의 조각들은 다음의 체인으로 결정이 되고 먼저 일치하는 것이 선택된다.
      * `--server`, `--certificate-authority`, `--insecure-skip-tls-verify`플래그가 있으면 이를 이용한다.
      * 병합된 kubeconfig 파일에서 클러스터 정보에 대한 속성이 있으면 이를 사용한다.
      * 서버 위치가 없으면 실패한다.
  * 어떤 정보든 빠진게 있으면 기본값을 사용하며, 추후에 인증 정보에 관한 프롬프트가 생길 수 있다.

## File references

* kubeconfig 파일에서 파일과 경로 참조는 kubeconfig파일의 위치에 따라 상대적이다.
  * 커맨드 라인에서의 파일 참조는 현재 작업중인 디렉토리에 상대적이다.
  * `$HOME/.kube/config`에서는 상대 경로는 상대경로로 저장이 되며 절대 경로는 절대 경로로 저장이 된다.