# Secrets

* Kubernetes `secret` 오브젝트는 패스워드, OAuth 토큰, ssh key같은 민감한 정보를 관리하고 저장할 수 있도록 해준다.
  * 이런 정보들을 `secret`에 저장하는 것은 파드 정의나 컨테이너 이미지 안에 그대로 적는것보다 더 안전하고 유연한 방법이다.
  * [Secrets design document](https://git.k8s.io/community/contributors/design-proposals/auth/secrets.md)에서 더 많은 정보를 볼 수 있다.

## Overview of Secrets

* Secret은 비밀번호, 토큰, 키같은 작은 양의 민감한 정보를 담을 수 있는 오브젝트이다.
  * 이런 정보들은 파드의 규격이나 이미지 안에 직접 넣어질수도 있지만 Secret 오브젝트에 놓는 것은 이것들이 어떻게 사용되는지에 대한 관리가 용이하도록 하고 노출되는 사고의 위험성을 줄여준다.
* 유저는 secret을 생성할 수 있고 시스템도 몇몇의 secret을 생성할 수 있다.
* secret을 사용하려면 파드는 secret을 참조해야한다.
  * 시크릿은 파드에서 두가지 방법으로 사용될 수 있다.
    * 파드 내에서 하나 이상의 container가 마운트 한 볼륨 안에 있는 파일
    * 파드때문에 이미지를 pull할 때 kubelet이 사용하는 것

### Built-in Secrets

#### Service Accounts Automatically Create and Attach Secrets with API Credentials

* 쿠버네티스는 API에 접속하기 위한 credentials를 가진 secret을 자동으로 생성하고 이 타입의 secret을 사용할 수 있도록 자동으로 파드를 수정한다.
* 자동 생성하여 API credential를 사용하는 것은 필요에 따라 비활성화 되거나 오버라이드 될 수 있다.
  * 하지만 apiserver에 안전하게 접속하려면 이는 추천되는 방법이다.
* [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 문서를 확인하여 더 많은 정보와 어떻게 Service Account가 동작하는지를 볼 수 있다.

### Creating your own Secrets

#### Creating a Secret Using kubectl create secret

* 몇몇 파드가 데이터베이스에 접속해야 한다고 해보자.

  * 파드가 사용해야 할 username과 password는 로컬 컴퓨터의 `./username.txt`와 `./password.txt`에 저장되어 있다.

    ```bash
    # Create files needed for rest of example.
    echo -n 'admin' > ./username.txt
    echo -n '1f2d1e2e67df' > ./password.txt
    ```

* `kubectl create secret` 명령어는 이 파일을 Secret에 저장시키고 Apiserverㅇ에서 오브젝트를 생성한다.

  ```shell
  kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
  ```

  ```
  secret "db-user-pass" created
  ```

> Note:
>
> `$`, `\`, `*`, `!`같은 특수문자는 shell에 의해 해석되므로 escape 문자가 필요하다. 일반적인 쉘에서 가장 쉽게 암호를 escape하는 것은 이를 `'`로 감싸는 것이다. 예를 들어, 실제 비밀번호가 `S!B\*d$zDsb` 라면 다음과 같이 명령어를 치면 된다.
>
> ```shell
> kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb'
> ```
>
> 파일에 있는 패스워드는 escape 문자가 필요 없다. (`--from-file`)
>

* 생성한 secret은 다음과 같이 확인이 가능하다.

  ```shell
  kubectl get secrets
  ```

  ```
  NAME                  TYPE                                  DATA      AGE
  db-user-pass          Opaque                                2         51s
  ```

  ```shell
  kubectl describe secrets/db-user-pass
  ```

  ```
  Name:            db-user-pass
  Namespace:       default
  Labels:          <none>
  Annotations:     <none>
  
  Type:            Opaque
  
  Data
  ====
  password.txt:    12 bytes
  username.txt:    5 bytes
  ```

> Note : `kubectl get`과 `kubectl describe`는 기본적으로 secret의 내용을 보여주지 않는다. 이는 실수로 secret이 외부로 노출되지 않는 것을 방지하고 터미널 로그에 남지 않도록 한다.

* [decoding a secret](https://kubernetes.io/docs/concepts/configuration/secret/#decoding-a-secret)에서는 어떻게 secret을 보는지 알 수 있다.

### Creating a Secret Manually

* 사용자는 또한 Secret을 json이나 yaml파일 형식으로 파일에 먼저 생성하고 나서 그 오브젝트를 생성할 수 있다.

  * Secret은 두가지 map을 가지고 있다.(data, stringData)
    * data 필드는 임의의 데이터를 저장할 때 사용되고, base64를 통해 인코딩 되어있다.
    * stringData 필드는 편의성을 위해 제공이 되고 사용자에게 secret data를 인코딩하지 않은 string으로 제공할 수 있도록 한다.

* 예를 들어, data field를 사용하여 두개의 string을 Secret에 저장하기 위해서 다음과 같이 base64로 변환한다.

  ```shell
  echo -n 'admin' | base64
  YWRtaW4=
  echo -n '1f2d1e2e67df' | base64
  MWYyZDFlMmU2N2Rm
  ```

* Secret을 다음과 같이 작성한다.

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: mysecret
  type: Opaque
  data:
    username: YWRtaW4=
    password: MWYyZDFlMmU2N2Rm
  ```

* `kubectl apply`를 이용하여 Secret을 생성하자.

  ```shell
  kubectl apply -f ./secret.yaml
  ```

  ```
  secret "mysecret" created
  ```

* 특정 시나리오에서는 stringData 필드를 사용하고 싶을 것이다.

  * 이 필드는 base64로 인코딩되지 않은 string을 직접 Secret에 넣을 수 있고 string은 Secret이 생성되거나 업데이트 될 때 인코딩 될 것이다.

* 이를 사용하는 실질적인 예시는 애플리케이션을 배포할 때 Secret을 이용하여 configuration 파일을 저장하려 하고 배포가 진행되는 동안에 configuration 파일의 일부분을 생성하려 하는 경우이다.

* 어플리케이션이 다음과 같은 configuration 파일을 사용하고 있다면

  ```yaml
  apiUrl: "https://my.api.com/api/v1"
  username: "user"
  password: "password"
  ```

  다음과 같은 방법으로 Secret을 이 안에 넣을 수 있다.

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: mysecret
  type: Opaque
  stringData:
    config.yaml: |-
      apiUrl: "https://my.api.com/api/v1"
      username: {{username}}
      password: {{password}}
  ```

* deployment tool은 template value `{{username}}`과 `{{password}}`을 `kubectl apply`가 동작하기 전에 치환할 것이다.

* stringData는 write-only인 필드이다.

  * 이는 Secret을 출력할 때 절대 나오지 않는다.

  * 예를들어, 다음과 같이 명령어를 입력하면

    ```shell
    kubectl get secret mysecret -o yaml
    ```

    출력은 다음과 같다.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      creationTimestamp: 2018-11-15T20:40:59Z
      name: mysecret
      namespace: default
      resourceVersion: "7225"
      uid: c280ad2e-e916-11e8-98f2-025000000001
    type: Opaque
    data:
      config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19
    ```

* 필드가 data와 stringData 모두 지정이 되어 있다면, stringData의 값이 사용된다.

  * 예를 들어, 다음의 Secret 정의에서

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      username: YWRtaW4=
    stringData:
      username: administrator
    ```

  * 결과는 다음과 같다.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      creationTimestamp: 2018-11-15T20:46:46Z
      name: mysecret
      namespace: default
      resourceVersion: "7579"
      uid: 91460ecb-e917-11e8-98f2-025000000001
    type: Opaque
    data:
      username: YWRtaW5pc3RyYXRvcg==
    ```

  * `YWRtaW5pc3RyYXRvcg==`는 `administrator`로 디코딩된다.

* data와 stringData의 key는 반드시 alphanumeric 문자와 '-', '_', '.'으로만 이루어져야 한다.

* **Encoding Note**: secret 데이터의 JSON과 YAML값을 직렬화 하는 것은 base64로 인코딩되어진다.

  * 줄바꿈은 이 문자에서 허용되지 않으며, 반드시 생략되어야 한다.
  * Darwin/macOS에서 `base64` 유틸리티를 사용하면 유저는 긴 줄을 나누는 `-b` 옵션을 반드시 사용하지 않아야 한다.
  * 반대로 Linux 유저는 `base64` 커맨드에 `-w 0`옵션을 넣어주거나 `-w`옵션이 없는 경우 `base64 | tr -d '\n'` 옵션을 주어야 한다.

### Creating a Secret from Generator

* kubectl은 [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)를 이용한 오브젝트 관리를 1.14부터 지원한다.

  * 이 새로운 기능으로 사용자는 generator를 통해서 Secret을 생성할 수 있고 이를 적용하여 Apiserver에 오브젝트를 생성할 수 있다.
  * generator는 반드시 디렉토리 내의 `kustomization.yaml` 안에 지정되어야 한다.

* 예를 들어, Secret을 `./username.txt`와 `./password.txt`를 통해서 생성하려 한다면

  ```shell
  # Create a kustomization.yaml file with SecretGenerator
  cat <<EOF >./kustomization.yaml
  secretGenerator:
  - name: db-user-pass
    files:
    - username.txt
    - password.txt
  EOF
  ```

  이를 kubesomization directory를 apply하여 Secret 오브젝트를 생성하면 된다.

  ```shell
  $ kubectl apply -k .
  secret/db-user-pass-96mffmfh4k created
  ```

* 이 secret이 생성된 것을 다음과 같이 확인할 수 있다.

  ```shell
  $ kubectl get secrets
  NAME                             TYPE                                  DATA      AGE
  db-user-pass-96mffmfh4k          Opaque                                2         51s
  
  $ kubectl describe secrets/db-user-pass-96mffmfh4k
  Name:            db-user-pass
  Namespace:       default
  Labels:          <none>
  Annotations:     <none>
  
  Type:            Opaque
  
  Data
  ====
  password.txt:    12 bytes
  username.txt:    5 bytes
  ```

* 예를 들어, Secret을 `username=admin`과 `password=secret`으로 생성하려면 `kustomization.yaml`안에 secret generator를 다음과 같이 적을 수 있다.

  ```shell
  # Create a kustomization.yaml file with SecretGenerator
  $ cat <<EOF >./kustomization.yaml
  secretGenerator:
  - name: db-user-pass
    literals:
    - username=admin
    - password=secret
  EOF
  ```

  kustomization 디렉토리를 apply하여 Secret 오브젝트를 생성할 수 있다.

  ```shell
  $ kubectl apply -k .
  secret/db-user-pass-dddghtt9b5 created
  ```

> Note: 생성된 Secret은 내용을 hasing한 suffix가 달린 이름으로 생성이 된다. 이를 통해 각 내용이 수정될 때마다 새로운 Secret이 생성됨을 확인할 수 있다.

### Decoding a Secret

* Secret은 `kubectl get secret` 명령어를 통해 얻을 수 있다.

  * 예를 들어, 이전에 생성했던 secret을 얻고자 하면

    ```shell
    kubectl get secret mysecret -o yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      creationTimestamp: 2016-01-22T18:41:56Z
      name: mysecret
      namespace: default
      resourceVersion: "164619"
      uid: cfee02d6-c137-11e5-8d73-42010af00002
    type: Opaque
    data:
      username: YWRtaW4=
      password: MWYyZDFlMmU2N2Rm
    ```

  * password 필드를 디코딩하려면

    ```shell
    echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
    ```

    ```
    1f2d1e2e67df
    ```

### Editing a Secret

* 이미 존재하는 secret은 다음의 명령어로 수정할 수 있다.

  ```shell
  kubectl edit secrets mysecret
  ```

* 이는 기본 설정된 에디터를 열게되고 base64로 인코딩 된 secret 값을 `data` 필드 안에 보여준다.

  ```yaml
  # Please edit the object below. Lines beginning with a '#' will be ignored,
  # and an empty file will abort the edit. If an error occurs while saving this file will be
  # reopened with the relevant failures.
  #
  apiVersion: v1
  data:
    username: YWRtaW4=
    password: MWYyZDFlMmU2N2Rm
  kind: Secret
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: { ... }
    creationTimestamp: 2016-01-22T18:41:56Z
    name: mysecret
    namespace: default
    resourceVersion: "164619"
    uid: cfee02d6-c137-11e5-8d73-42010af00002
  type: Opaque
  ```

## Using Secrets

* Secret은 data volume으로 마운트 될 수 있고 아니면 환경 변수로 파드 속 컨테이너에서 사용될 수 있다.
  * 또한 파드에 직접적으로 노출이 되지 않으면서 다른 시스템의 일부에서 사용될 수도 있다.
  * 예를 들어, 내 파드 대신에 외부의 시스템과 상호작용하기 위해서 다른 시스템에 일부에서 credential을 가지고 있을 수 있다.

### Using Secrets as Files from a Pod

* 파드 내에서 volume 안에있는 Secret을 사용하기 위해서는

  * secret을 생성하거나 이미 존재하는 것을 사용한다. 여러 파드는 같은 secret을 동시에 참조할 수 있다.
  * 파드의 정의에서 `.spec.volumes[]` 아래에 volume을 추가한다. volume이름은 아무거나 해도 되고, `.spec.volumes[].secret.secretName` 필드는 secret 오브젝트의 이름과 동일해야 한다.
  * `.spec.containers[].volumeMounts[]`를 secret이 필요한 각 container에 추가한다. `.spec.containers[].volumeMounts[].readOnly = true`를 지정하고 `.spec.containsers[].volumeMounts[].mountPath`를 사용중이지 않은 디렉토리 이름으로 하여 원하는 곳에 secret이 나타나도록 한다.
  * image와 command를 수정하여 프로그램이 그 디렉토리에서 해당 파일을 찾도록 한다. 각 secret `data` map에 있는 key는 `mountPath` 아래에 파일이름이 된다.

* 다음은 volume 안에있는 secret을 마운트하는 파드의 예시이다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mypod
      image: redis
      volumeMounts:
      - name: foo
        mountPath: "/etc/foo"
        readOnly: true
    volumes:
    - name: foo
      secret:
        secretName: mysecret
  ```

* 사용하고자 하는 각각의 secret은 `.spec.volumes`에 언급되어야 한다.

* 파드 내에 여러개의 컨테이너가 있을 경우 각 컨테이너는 각각 `volumeMounts` 블락이 있어야 하지만 `.spec.volumes`는 secret마다 파드내에 하나씩 있어야 한다.

* 편한대로 많은 파일을 하나의 secret으로 패키징할 수도 있고 여러개의 secret을 사용할 수도 있다.

#### Projection of secret keys to specific paths

* volume 내에서 어느 위치에 Secret의 key들이 투영되는지 경로를 변경할 수 있다.

  * `.spec.volumes[].secret.items`를 사용하여 각 key의 target path를 변경할 수 있다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: mypod
        image: redis
        volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
      volumes:
      - name: foo
        secret:
          secretName: mysecret
          items:
          - key: username
            path: my-group/my-username
    ```

* 이는 다음과 같이 동작한다.

  * `username` secret은 `/etc/foo/username`이 아니라 `/etc/foo/my-group/my-username` 파일 아래에 저장이 된다.
  * `password` secret은 투영되지 않는다.

* 만약 `.spec.volumes[].secret.items`가 사용된다면 `items` 안에 지정이 된 key들만 투영된다.

  * secret의 모든 key들을 사용하려면 모두 `items` 필드 안에 작성이 되어야 한다.
  * 모든 key들은 반드시 일치하는 secret을 가지고 있어야 한다.
    * 그렇지 않으면, volume자체가 생성되지 않는다.

#### Secret file permissions

* secret이 가지게 될 일부분의 파일에 permission mode bit를 지정할 수 있다.

  * 만약 아무것도 지정하지 않는다면 `0644`를 기본값으로 한다.
  * 필요에 따라 전체 secret volume에 대한 기본 모드를 지정할 수도 있고 각각 key에 대해서 override할 수도 있다.

* 예를 들어, 다음과 같이 기본 모드를 지정할 수 있다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mypod
      image: redis
      volumeMounts:
      - name: foo
        mountPath: "/etc/foo"
    volumes:
    - name: foo
      secret:
        secretName: mysecret
        defaultMode: 256
  ```

* 그러면 secret은 `/etc/foo`에 마운트가 되고 secret volume mount에 의해 생성된 모든 파일들은 `0400` 권한을 가지게 된다.

* JSON이 8진법을 지원하지 않기 때문에 0400 권한은 256값을 사용해야 한다.

  * 만약 yaml을 사용한다면, 더 익숙한 방법인 8진법을 이용하여 권한을 부여할 수 있다.

* 이전의 예시에서 mapping을 이용할 수 있고 다른 file에 대해서 다른 권한을 줄 수 있다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mypod
      image: redis
      volumeMounts:
      - name: foo
        mountPath: "/etc/foo"
    volumes:
    - name: foo
      secret:
        secretName: mysecret
        items:
        - key: username
          path: my-group/my-username
          mode: 511
  ```

* 이 경우에 `/etc/foo/my-group/my-username`으로 생성된 파일은 `0777`의 권한을 가질 것이다.

  * JSON의 제약때문에 10진법을 사용해서 모드를 지정해야 한다.

* 이 permission value가 나중에 읽었을 때 10진법으로 출력이 됨을 참고하라.

#### Consuming Secret Values from Volumes

* secret volume을 마운트한 컨테이너 안에서 secret key들은 파일로 나타나고 secret value들은 base64로 디코딩되어 이 파일들 안에 저장된다.

  * 다음은 위의 예시에서 컨테이너 안에서 다음의 명령어를 사용했을 때의 출력들이다.

    ```shell
    ls /etc/foo/
    ```

    ```
    username
    password
    ```

    ```shell
    cat /etc/foo/username
    ```

    ```
    admin
    ```

    ```shell
    cat /etc/foo/password
    ```

    ```
    1f2d1e2e67df
    ```

* 컨테이너 내의 프로그램은 이 파일로부터 secret을 읽어야 한다.

#### Mounted Secrets are updated automatically

* 이미 volume 안에서 사용되고 있는 secret이 업데이트가 되면, 투영된 키들도 또한 업데이트 된다.
  * kubelet은 주기적으로 마운트된 secret 정보가 최신화한다.
  * 하지만 현재의 Secret 값을 가지고 올 때에는 local cache를 이용한다.
  * 이 캐쉬의 타입은 ([KubeletConfiguration struct](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go)에서 `ConfigMapAndSecretChangeDetectionStrategy`필드를 통해) 설정가능하다.
  * watch(default), ttl기반, 단순히 모든 요청을 직접 kube-apiserver로 리다이렉팅하는 방법을 통해서 전파가 된다.
  * 결과적으로 Secret이 업데이트가 되는 시점, kubelet의 sync period와 캐시가 전파될때 지연이 있다면 cache propagation delay까지 되어 파드에 반영되는 모든 지연은 캐시의 종류에 따라 다르다.
    * watch propagation delay, 캐시의 ttl, zero corespondingly와 같다.(?)

> Note: subPath volume 마운트로 Secret을 사용하는 컨테이너는 Secret 업데이트를 받지 못한다.

### Using Secrets as Environment Variables

* 파드내의 환경변수에서 secret을 사용하기 위해서는

  * secret을 생성하거나 기존에 있는 것을 사용한다. 여러 파드가 같은 secret을 참조할 수 있다.
  * 환경변수로 secret을 사용하고 싶은 각 컨테이너에 대한 파드의 정의를 수정하여 원하는 secret key를 사용하도록 수정하라.
    * secret key를 사용하는 환경변수는 secret의 name과 key를 `env[].valueFrom.secretKeyRef`안에 반영되도록 한다.
  * 이미지를 수정하하고 커맨드 라인을 수정하여 프로그램이 지정된 환경 변수를 보도록 수정한다.

* 다음은 환경변수로 secret을 사용하는 것의 예시이다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: secret-env-pod
  spec:
    containers:
    - name: mycontainer
      image: redis
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
    restartPolicy: Never
  ```

#### Consuming Secret Values from Environment Variables

* 환경 변수 안에 있는 secret을 사용하는 컨테이너 내부에서 secret key는 secret data의 값이 base64로 디코딩 된 값을 가지는 일반적인 환경변수로 나타난다.

  * 아래는 위의 예시에서 다음과 같은 명령어의 결과들을 보여준다.

    ```shell
    echo $SECRET_USERNAME
    ```

    ```
    admin
    ```

    ```shell
    echo $SECRET_PASSWORD
    ```

    ```
    1f2d1e2e67df
    ```

### Using imagePullSecrets

* imagePullSecret은 Docker(또는 다른) image registry의 비밀번호를 포함한 secret을 전달하여 Kubelet이 파드 대신하여 private 이미지를 pull하도록 한다.

#### Manually specifying an imagePullSecret

* imagePullSecrets의 사용법 등은 [images documentation](https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod)에 정리되어 있다.

### Arranging for imagePullSecrets to be Automatically Attached

* 수동으로 imagePullSecret을 생성하고 이를 serviceAccount에서 참조하도록 할 수 있다.
  * 그 serviceAccount 또는 기본적으로 그 secret을 사용하는 serviceAccount로 생성이 되는 어떤 파드던지 imagePullSecret 필드가 service account로 변경될 것이다.
  * [Add ImagePullSecrets to a service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)을 통해 이 프로세스에 대해 자세한 설명을 보아라.

### Automatic Mounting of Manually Created Secrets

* 수동으로 생성한 secret(예: github 계정에 접속하기 위한 token을 가진 것)은 자동으로 service account에 따라 파드에 접속될 수 있다.
  * [Injecting Information into Pods Using a PodPreset](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)을 통해서 이 프로세스에 대한 자세한 설명을 확인하라.

## Details

### Restrictions

* Secret volume sources는 지정된 오브젝트 참조가 실제로 `Secret` 오브젝트를 참조하는지 확인하기 위해 검증된다.

  * 그러므로 secret은 어떤 파드가 그것을 참조하기 전에 생성되어 있어야 한다.

* Secret API 오브젝트는 namespace기반으로 있다.

  * 그것들은 같은 namespace 안에있는 파드에 의해서만 참조될 수 있다.

* 각각의 secret은 1MiB로 제한이 된다.

  * 이는 매우 큰 secret을 만들어서 apiserver와 kubelet 메모리를 힘들게 하는 것을 막는다.
  * 하지만 매우 작은 secret을 많이 생성하는 것 또한 메모리를 많이 사용한다.
  * secret은 planned feature이기 때문에 더 많은 복잡한 메모리 사용량에 대한 제약이 있다.

* Kubelet은 API server로부터 받은 secret을 파드에서 사용하는 방식만 지원한다.

  * 이는 kubectl이나 replication controller로 간접적으로 생성된 모든 파드를 포함한다.
  * 또 kubelet의 `--manifest-url` 플래그, `--config` 플래그, REST API를 통해 생성된 파드들은 제외된다. (이들은 pod를 생성하는 일반적인 방식이 아니다.)

* Secret은 optional로 표기하지 않았다면 반드시 환경변수로 파드에서 사용되기 전에 생성이 되어야 한다.

* Secret이 존재하지 않을 때 `secretKeyRef`를 통해 키의 참조는 파드가 시작되는 것을 막는다.

* Secret은 유효하지 않은 환경변수 이름을 가진 것들은 제외하고 `envFrom`을 통해 환경 변수에 값을 전파한다.

  * 파드는 시작할 수는 있다.

  * `InvalidVariableNames`라는 이유를 가진 이벤트가 생성될 것이고 메시지는 생략된 유효하지 않은 키들을 포함할 것이다.

  * 아래의 예시는 파드가 2개의 유효하지 않은 키(1badkey, 2alsobad)를 가진 default/mysecret을 참조하는 것을 보여준다.

    ```shell
    kubectl get events
    ```

    ```
    LASTSEEN   FIRSTSEEN   COUNT     NAME            KIND      SUBOBJECT                         TYPE      REASON
    0s         0s          1         dapi-test-pod   Pod                                         Warning   InvalidEnvironmentVariableNames   kubelet, 127.0.0.1      Keys [1badkey, 2alsobad] from the EnvFrom secret default/mysecret were skipped since they are considered invalid environment variable names.
    ```



### Secret and Pod Lifetime interaction

* 파드가 API를 통해 생성이 될 때 참조된 secret이 존재하는지는 검사하지 않는다.
  * 파드가 뜨고 나면 kubelet은 secret 값을 fetch하려 할 것이다.
  * secret이 존재하지 않거나 API server의 일시적인 장애로 fetch되지 않으면 kubelet은 계속해서 재시도 한다.
  * 이는 파드가 아직 시작되지 않았기 때문이라고 보고된다.
  * secret이 fetch되면 kubelet은 이를 담고있는 volume을 생성하고 마운트한다.
  * 어떤 파드의 컨테이너도 이런 파드의 volume이 마운트 되기 전까지 시작되지 않는다.

## Use cases

### Use-Case: Pod with ssh keys

* ssh key들을 포함한 kustomization.yaml을 SecretGenerator를 통해 생성하라.

  ```shell
  kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub
  ```

  ```
  secret "ssh-key-secret" created
  ```

> Caution : ssh key를 전송하기 전에 주의깊게 생각해야 한다. 클러스터 내의 다른 유저들이 이 secret에 접근할 수도 있다. 쿠버네티스 클러스터 내의 모든 유저가 접근할 수 있고 필요에 의해 삭제할 수도 있는 service account를 사용하라.

* 이제 우리는 ssh key를 가진 secret을 참조하고 volume 내에서 소비할 수 있는 파드를 생성할 수 있다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: secret-test-pod
    labels:
      name: secret-test
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: ssh-key-secret
    containers:
    - name: ssh-test-container
      image: mySshImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
  ```

* 컨테이너의 명령어가 실행되면 key들을 사용할 수 있을 것이다.

  ```shell
  /etc/secret-volume/ssh-publickey
  /etc/secret-volume/ssh-privatekey
  ```

* 컨테이너는 ssh 연결을 위한 secret data 사용에 자유로워진다.

### Use-Case: Pods with prod / test credentials

* 이 예시는 prod credentials를 가지는 secret을 사용하는 파드와 테스트 환경의 credential을 가진 secret을 사용하는 파드에 대해 알아볼 것이다.

* ```shell
  kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11
  ```

  ```
  secret "prod-db-secret" created
  ```

  ```shell
  kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests
  ```

  ```
  secret "test-db-secret" created
  ```

> Note: 
>
> `$`, `\`, `*`, `!` 같은 특수 문자는 shell에 의해 해석이 되므로 escape 문자가 필요하다. 대부분의 shell에서 가장 쉽게 password를 escape하는 방법은 이를 `'`을 통해 감싸는 것이다. 예를 들어, 실제 패스워드가 `S!B\*d$zDsb`라면, 다음과 같이 명령어를 작성하면 된다.
>
> ```shell
> kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb'
> ```
>
> 파일에 있는 패스워드를 사용할 때에는 특수문자를 escape하지 않아도 된다. (`--from-file`)

* 이제 파드를 생성한다.

  ```shell
  $ cat <<EOF > pod.yaml
  apiVersion: v1
  kind: List
  items:
  - kind: Pod
    apiVersion: v1
    metadata:
      name: prod-db-client-pod
      labels:
        name: prod-db-client
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: prod-db-secret
      containers:
      - name: db-client-container
        image: myClientImage
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
  - kind: Pod
    apiVersion: v1
    metadata:
      name: test-db-client-pod
      labels:
        name: test-db-client
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: test-db-secret
      containers:
      - name: db-client-container
        image: myClientImage
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
  EOF
  ```

* 파드를 같은 kustomization.yaml에 추가한다.

  ```shell
  $ cat <<EOF >> kustomization.yaml
  resources:
  - pod.yaml
  EOF
  ```

* 이 오브젝트를 Apiserver에 적용한다.

  ```shell
  kubectl apply -k .
  ```

* 두 컨테이너는 다음의 파일을 각 컨테이너의 환경에 따라 filesystem에 가지고 있게 된다.

  ```shell
  /etc/secret-volume/username
  /etc/secret-volume/password
  ```

* 두 파드의 스펙이 하나의 필드만 다름을 인지하라.

  * 이는 공통의 파드 config 템플릿과는 다른 파드를 생성해버리기 쉽다.

* 사용자는 두개의 service account를 통해서 파드의 specification을 기반으로 간단하게 할 수 있다.

  * 하나는 `prod-db-secret`이라는 secret을 가진 `prod-user`이라고 하고 하나는 `test-db-secret`을 가진 `test-user`라고 하자.

* 그러면 파드의 spec은 다음과 같이 간단해진다.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: prod-db-client-pod
    labels:
      name: prod-db-client
  spec:
    serviceAccount: prod-db-client
    containers:
    - name: db-client-container
      image: myClientImage
  ```

### Use-case: Dotfiles in secret volume

* 데이터를 숨김처리 하기 위해(예: .문자로 시작되는 이름을 가진 파일들) key를 .으로 시작하여 숨김처리 할 수 있다.

  * 예를 들어 다음은 volume에 마운트되는 시크릿이다.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: dotfile-secret
    data:
      .secret-file: dmFsdWUtMg0KDQo=
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-dotfiles-pod
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: dotfile-secret
      containers:
      - name: dotfile-test-container
        image: k8s.gcr.io/busybox
        command:
        - ls
        - "-l"
        - "/etc/secret-volume"
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
    ```

* `secret-volume`은 `.secret-file`이라고 하는 하나의 파일을 가지고 있고 `dotfile-test-container`는 이 파일을 `/etc/secret-volume/.secret-file`에 가지게 된다.

> Note: .문자로 시작되는 파일들은 `ls -l` 명령어로부터 숨겨진다. `ls -la`를 통해서만 디렉토리안에서 볼 수 있다.

### Use-case: Secret visible to one container in a pod

* 프로그램이 HTTP 요청을 받아야 한다고 가정해보면 몇가지 복잡한 비즈니스 로직을 한 후에 HMAC으로 메시지들에 대해 서명할 것이다.
  * 이는 복잡한 어플리케이션 로직을 가지고 있어서 미리 알리지 않고 서버에 있는 파일에 대해 읽을수도 있으며, 이는 해커에 의해 private key가 노출되는 상황이 발생할 수도 있기 때문이다.
* 이는 두 컨테이너 안에서 두가지 프로세스로 나뉠 수 있다.
  * 유저와 상호작용을 하고 비즈니스 로직을 하지만 private key를 볼 수 없는  프론트엔드 컨테이너
  * private key를 가지고 있어서, 프론트엔드로부터 온 요청(예: localhost 네트워킹)에 대해 간단히 서명하는 signer 컨테이너 
  * 이 분할된 접근방식을 통해 해커가 파일을 읽기 더 어렵게 어플리케이션 서버가 임의의 동작을 하도록 하는게 아니라 어떤 동작을 하도록 속여야 한다.

## Best practices

### Clients that use the secrets API

* secret API를 통해 어플리케이션을 배포할 때, RBAC같은 authorization policies를 사용하여 접속을 제한해야 한다.
* Secret은 쿠버네티스 내에서(ex: service account token), 외부의 시스템으로의 escalation을 유발하는 많은 중요한 값들을 가지고 있다.
* 이러한 이유로 namespace 내에서 secret에 대한 `watch`와 `list` 요청은 매우 강력한 능력을 가지고 있고, secret의 목록을 보는 클라이언트가 네임스페이스 안에 있는 모든 시크릿에 대해 값을 볼 수 있기 때문에 피해야 한다.
  * 클러스터에서 모든 secret에 대한 `watch`와 `list`는 가장 특권을 가진 system-level componets에서만 가능해야 한다.
* secret API에 접속하려는 어플리케이션은 반드시 `get` 요청을 사용하고자 하는  secret에다 해야한다.
  * 이는 관리자가 앱이 필요로 하는 [white-listing access to individual instances](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)을 제외하고 모든 시크릿에 대한 접근을 막는다.
* 반복되는 `get`요청을 개선하려면 클라이언트는 리소스를 `watch`하기보다는 secret을 참조하도록 디자인하고 참조가 변경될 때 secret에 대한 요청을 다시하는 방식으로 디자인할 수 있다.
  * 추가적으로 [“bulk watch” API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/bulk_watch.md)는 클라이언트가 원하는 각 리소스를 `watch`할 수 있도록 하고 나중의 Kubernetes 릴리즈에서 사용가능할 것이다.

## Security Properties

### Protections

* `secret` 오브젝트가 그것을 사용하는 `pods`와는 독립적으로 생성이 되기 때문에 secret이 파드가 생성되고, 수정되고, 볼 때 노출되는 위험이 적다.
  * 이 시스템은 `secret` 오브젝트에 대해 가능한 곳에서 secret을 디스크에 사용하는것을 막는 것과 같은 추가적인 보호를 해준다.
* secret은 이를 사용하려는 파드가 동작하는 노드로만 보내진다.
  * kubelet은 secret을 `tmpfs`에 저장하여 secret이 디스크 저장공간에 쓰이지 않도록 한다.
  * secret을 참조하던 파드가 삭제되면 kubelet은 secret의 local copy 또한 삭제한다.
* 같은 노드에 여러 파드가 사용하는 여러 secret이 있을수 있다.
  * 하지만 secret에 대한 파드의 요청은 컨테이너 내에서만 보일 것이다.
  * 그러므로 하나의 파드는 다른 파드의 secret에 접근할 수 없다.
* 파드 안에는 여러개의 컨테이너가 있을 수 있다.
  * 하지만 파드 안의 각 컨테이너는 secret volume에 대한 요청을 해당 컨테이너의 `volumeMounts`에서 요청해야 하고 이를 통해 컨테이너 내에서 secret을 볼 수 있게 된다.
  * [security partitions at the Pod level](https://kubernetes.io/docs/concepts/configuration/secret/#use-case-secret-visible-to-one-container-in-a-pod)을 구성하는데 도움이 된다.
* 대부분의 kubernetes-project-maintained 분포에서 유저와 apiserver와의 통신, apiserver에서 kubelet으로의 통신은 SSL/TLS로 보호가 된다.
  * secret은 이 채널을 통해 전송이 될 때 보호를 해준다.

**FEATURE STATE**: `Kubernetes v1.13` - beta

* secret 데이터에 대해서 [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)을 활성화할 수 있다.
  * 그러면 secret은 etcd에 평문으로 저장되지 않는다.

### Risks

* API server에서 secret 데이터는 etcd에 저장이 된다. 따라서:
  * 관리자는 encryption at rest를 cluster data에 대해서 활성화해야 한다. (버전 1.13 이상)
  * 관리자는 etcd에 대한 접속을 admin 유저로만 제한해야한다.
  * 관리자는 etcd에서 더이상 사용하지 않는 정보들을 정리해야한다.
  * etcd가 클러스터 안에서 동작하고 있다면 관리자는 etcd peer-to-peer 통신에 SSL/TLS를 사용하도록 해야한다.
* secret 데이터가 base64로 인코딩 되어져 있는 manifest(JSON이나 YAML)를 통해서 secret을 설정하면 이 파일을 공유하거나 소스 레파지토리에서 이를 확인하는 것은 secret이 제대로 동작하지 않는 것이다.
  * base64 인코딩은 암호화 방법이 아니며 이는 평문과 같음을 인지해야 한다.
* 어플리케이션은 여전히 volume에서 secret을 읽고난 뒤로 실수로 로그를 남기거나 신뢰하지 못하는 곳을 전송하는 것을 막는 것과 같이 그 값을 보호해야 한다.
* secret을 사용하는 파드를 만드는 사용자는 secret의 값을 볼 수 있다.
  * spiserver 정책이 사용자가 secret 오브젝트를 보지 못하도록 하더라도 user는 secret을 노출시키는 파드를 실행시킬 수 있다.
* 현재 어떤 노드에서던지 root권한을 가진 사람은 kubelet과 비슷하게 apiserver로부터 모든 secret을 읽을 수 있다.
  * 이는 나중에 실제로 secret을 소유한 사람만 노드로 secret을 보낼 수 있고, 단일 노드에 대한 root권한에 대한 영향도를 제한하는 기능을 계획중에 있다.