# Adding entries to Pod /etc/hosts with HostAliases

* DNS나 다른 옵션들이 적용될 수 없을 때 entry들을 파드의 /etc/hosts파일에 추가하는 것은 파드 레벨에서 hostname resolution을 override한다.
  * 1.7에서 사용자는 이런 custom entries를 PodSpec의 HostAliases로 추가할 수 있다.
* 파일이 Kubelet에 의해 관리되고 파드의 생성/재시작 하는 동안 overwritten될 수 있기 때문에 HostAliases를 사용하지 않는 수정은 추천하지 않는다.

## Default Hosts File Content

* Pod IP가 할당된 Nginx Pod를 시작해보자.

  ```shell
  kubectl run nginx --image nginx --generator=run-pod/v1
  ```

  ```shell
  pod/nginx created
  ```

* Pod IP를 확인하자

  ```shell
  kubectl get pods --output=wide
  ```

  ```shell
  NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
  nginx    1/1       Running   0          13s    10.200.0.4   worker0
  ```

* hosts 파일의 내용은 다음과 같다.

  ```shell
  kubectl exec nginx -- cat /etc/hosts
  ```

  ```none
  # Kubernetes-managed hosts file.
  127.0.0.1	localhost
  ::1	localhost ip6-localhost ip6-loopback
  fe00::0	ip6-localnet
  fe00::0	ip6-mcastprefix
  fe00::1	ip6-allnodes
  fe00::2	ip6-allrouters
  10.200.0.4	nginx
  ```

* 기본적으로 `hosts`는 `localhost`와 자신의 hostname같은 IPv4와 IPv6 boilerplates만 포함한다.

## Adding Additional Entries with HostAliases

* default boilerplate를 추가하는 것과 더불어 우리는 HostAliases를 파드의 `.spec.hostAliases`에 추가함으로써 additional entries를 `hosts`파일에 추가하여 `foo.local`, `bar.local`을 `127.0.0.1`로, 그리고 `foo.remote`, `bar.remote`를 `10.1.2.3`으로 해석할 수있다. 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: hostaliases-pod
  spec:
    restartPolicy: Never
    hostAliases:
    - ip: "127.0.0.1"
      hostnames:
      - "foo.local"
      - "bar.local"
    - ip: "10.1.2.3"
      hostnames:
      - "foo.remote"
      - "bar.remote"
    containers:
    - name: cat-hosts
      image: busybox
      command:
      - cat
      args:
      - "/etc/hosts"
  ```

* 이 파드는 다음의 명령어로 시작할 수 있다.

  ```shell
  kubectl apply -f hostaliases-pod.yaml
  ```

  ```shell
  pod/hostaliases-pod created
  ```

* Pod IP와 status를 확인하자

  ```shell
  kubectl get pod --output=wide
  ```

  ```shell
  NAME                           READY     STATUS      RESTARTS   AGE       IP              NODE
  hostaliases-pod                0/1       Completed   0          6s        10.200.0.5      worker0
  ```

* `hosts`파일의 내용은 이럴 것이다.

  ```shell
  kubectl logs hostaliases-pod
  ```

  ```none
  # Kubernetes-managed hosts file.
  127.0.0.1	localhost
  ::1	localhost ip6-localhost ip6-loopback
  fe00::0	ip6-localnet
  fe00::0	ip6-mcastprefix
  fe00::1	ip6-allnodes
  fe00::2	ip6-allrouters
  10.200.0.5	hostaliases-pod
  
  # Entries added by HostAliases.
  127.0.0.1	foo.local	bar.local
  10.1.2.3	foo.remote	bar.remote
  ```

* 
    

## Why Does Kubelet Manage the Hosts File?

* container가 이미 시작되고 난 후로 Docker에서 파일이 수정되는 것을 막기 위해 Kubelet은 파드의 각 conatiner에 대해 `hosts` 파일을 관리한다.
* 파일의 managed-nature때문에 사용자가 적은 내용들은 언제든지 `hosts` 파일이 container가 재시작되거나 Pod가 reschedule되는 경우에 Kubelet에 의해서 remounted될 때마다 overwritten될 수 있다.
  * 따라서 파일의 내용을 수정하는 것은 추천하지 않는다.

