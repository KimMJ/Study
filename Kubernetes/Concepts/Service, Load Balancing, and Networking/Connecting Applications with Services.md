# Connecting Applications with Services

## The Kubernetes model for connecting containers

* 이제 계속적으로 네트워크에 노출할 수 있는 동작하고 복제하는 어플리케이션을 가지게 되었다.
  * 네트워크에 대한 Kubernetes의 접근법에 대해 논의하기 전에 도커가 네트워크에서 동작하는 "보통의" 방법에 대해 비교하여 알아보는 것은 가치있을 것이다.
* 기본적으로 도커는 host-private 네트워크를 사용하여 컨테이너가 다른 컨테이너로 같은 머신 위에 있을 때에만 통신할 수 있다.
  * 도커의 컨테이너가 노드 사이에서 통신하도록 하기 위해 머신의 IP 주소에 컨테이너로 foward하거나 proxy하는 포트를 할당해 주어야 한다.
  * 이는 컨테이너가 반드시 사용하는 포트에 대해 조심스럽게 구성해야 하고 포트는 동적으로 할당되어야 함을 의미한다.
* 여러 개발자가 포트를 함께 구성하는 것은 규모의 면에서 매우 어렵고 제어하지 못하는 cluster-level의 이슈에 노출되게 된다.
  * Kubernetes는 파드가 어느 host에 있더라도 다른 파드와 통신할 수 있음을 가정한다.
  * 모든 파드에 대해 그 자신의 cluster-private-IP 주소를 부여하여 파드 사이의 링크를 명확하게 생성하거나 컨테이너 포트를 호스트 포트로 매핑할 필요가 없도록 한다.
  * 이는 파드 안에 있는 컨테이너가 서로의 포트에 localhost로 통신할 수 있음을 의미하고 모든 cluster 안에 있는 파드는 다른 파드를 NAT가 없이 볼 수 있다는 것을 의미한다.
  * 이 문서의 나머지 부분은 어떻게 그러한 네트워크 모델에서 reliable services가 동작할 수 있는지에 대한 자세한 설명이 있을 것이다.
* 이 가이드는 컨셉의 증명을 보여주기 위해 간단한 nginx 서버를 예시로 사용할 것이다.
  * 이는 Jenkins CI application같은 더 복잡한 어플리케이션의 원리를 포함한다.

## Exposing pods to the cluster

* 이전의 예시에서 했지만 다시한번 해보고 네트워킹 관점에 초점을 맞추어 보자.

  * nginx Pod를 생성하고 container port specification이 있음을 주목하자.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx
    spec:
      selector:
        matchLabels:
          run: my-nginx
      replicas: 2
      template:
        metadata:
          labels:
            run: my-nginx
        spec:
          containers:
          - name: my-nginx
            image: nginx
            ports:
            - containerPort: 80
    ```

  * 이는 사용자의 cluster에 어떤 노드에서도 접속이 가능하게 한다. 파드가 동작하는 노드를 확인해보자.

    ```shell
    kubectl apply -f ./run-my-nginx.yaml
    kubectl get pods -l run=my-nginx -o wide
    ```

    ```
    NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
    my-nginx-3800858182-jr4a2   1/1       Running   0          13s       10.244.3.4    kubernetes-minion-905m
    my-nginx-3800858182-kna2y   1/1       Running   0          13s       10.244.2.5    kubernetes-minion-ljyd
    ```

  * 파드의 IP를 확인해보자.

    ```shell
    kubectl get pods -l run=my-nginx -o yaml | grep podIP
        podIP: 10.244.3.4
        podIP: 10.244.2.5
    ```

  * cluster 안에서 모든 노드에 대해 ssh로 접속할 수 있고 두 IP에 curl을 보낼 수 있다.

## Creating a Service

* 따라서 우리는 평평하게, cluster에 전반에 걸친 주소 공간에서 nginx가 돌아가는 파드를 가지고 있다.

  * 이론상 우리는 이 파드와 직접 통신할 수 있지만 노드가 죽으면 어떤 일이 발생할까?
  * 그 안의 파드도 죽게 되고 Deployment는 다른 IP를 가진 새로운 파드를 생성할 것이다.
  * 이 문제는 Services가 해결해준다.

* Kubernetes의 Service는 모든 동일한 기능을 제공해주는 cluster안 어딘가에서 동작하는 파드의 논리적인 묶음을 정의하는 추상화된 개념이다.

  * 생성이 되었을 때, 각 Service는 유일한 IP 주소(clusterIP)를 할당한다.
  * 이 주소는 Service의 lifespan과 연결되고 Service가 살아있는 동안 변경되지 않는다.
  * 파드는 Service에 연결하도록 설정될 수 있고, Service와의 통신은 자동으로 Service 멤버중 어떤 하나의 파드로 load-balanced될 것임을 안다.

* 2 nginx replicas가 도는 Service를 `kubectl expose`를 통해 생성할 수 있다.

  ```bash
  kubectl expose deployment/my-nginx
  ```

  ```
  service/my-nginx exposed
  ```

* 이는 `kubectl apply -f`를 통해 다음 yaml을 지정해주는 것과 같다.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nginx
    labels:
      run: my-nginx
  spec:
    ports:
    - port: 80
      protocol: TCP
    selector:
      run: my-nginx
  ```

* 이 specification은 TCP port 80을 run: my-nginx 라벨이 있는 어느 파드에나 타게팅하고 이를 추상화된 Service port(`targetPort` : 트래픽이 들어오도록 허락하는 container의 port, `port` : 다른 파드들이 Service에 접속할 때 사용할 수 있는 모든 추상화된 Service port)로 노출시키는 Service를 생성할 것이다.

  * Service API 오브젝트를 확인하여 Service에서 지원하는 필드들의 리스트를 확인하라.

  * ```yaml
    kubectl get svc my-nginx
    ```

    ```
    NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    my-nginx   ClusterIP   10.0.162.149   <none>        80/TCP    21s
    ```

* 전에 언급 했듯이, Service는 파드 그룹에 의해 지원받는다.

  * 이런 파드는 `endpoints`를 통해서 노출이 된다.
  * Service의 selector는 지속적으로 evalutated될 것이고 결과는 마찬가지로 `my-nginx`이름을 가진 Endpoints 오브젝트에 POST될 것이다.
  * Pod가 죽으면 이는 자동적으로 endpoints에서 삭제가 되고 Service의 selector와 일치하는 새로운 Pod는 자동적으로 endpoints에 추가가 될 것이다.
  * endpoints를 확인하고 IPs가 첫번째 단계에서 Pods가 생성되었을 때와 같음을 확인해라.

  ```shell
  kubectl describe svc my-nginx
  ```

  ```
  Name:                my-nginx
  Namespace:           default
  Labels:              run=my-nginx
  Annotations:         <none>
  Selector:            run=my-nginx
  Type:                ClusterIP
  IP:                  10.0.162.149
  Port:                <unset> 80/TCP
  Endpoints:           10.244.2.5:80,10.244.3.4:80
  Session Affinity:    None
  Events:              <none>
  ```

  ```shell
  kubectl get ep my-nginx
  ```

  ```
  NAME       ENDPOINTS                     AGE
  my-nginx   10.244.2.5:80,10.244.3.4:80   1m
  ```

* 이제 cluster의 어느 노드에서나 nginx Service의 `<CLUSTER-IP>:<PORT>`로 curl을 보낼 수 있다.

  * Service IP가 선으로 연결된 것이 아닌 완전한 virtual임을 확인해라.
  * 이것이 어떻게 동작하는지 궁금하다면 [service proxy](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)를 읽어보아라.

## Accessing the Service

* Kuberntes는 Service를 찾는데 두가지 주요 mode를 제공한다. - environment variables, DNS
  * 전자는 설치가 필요 없지만 후자는 [CoreDNS cluster addon](http://releases.k8s.io/master/cluster/addons/dns/coredns)를 필요로 한다.

### Environment Variables

* Pod가 Node에서 동작할때 kubelet은 각각의 활성화된 Service에 대한 environment variables를 추가한다.

  * 여기서는 ordering problem을 소개한다.
  * 이유를 알기 위해서는 동작중인 nginx Pods의 environment를 확인해라.

  ```shell
  kubectl exec my-nginx-3800858182-jr4a2 -- printenv | grep SERVICE
  ```

  ```
  KUBERNETES_SERVICE_HOST=10.0.0.1
  KUBERNETES_SERVICE_PORT=443
  KUBERNETES_SERVICE_PORT_HTTPS=443
  ```

* Service에 대한 언급이 없음을 보아라.

  * 이는 사용자가 Service 생성 전에 replicas를 생성했기 때문이다.
  * 이렇게 사용하는 것의 다른 단점은 sheduler가 두 Pod를 같은 Machine에 놓아 Pod가 죽으면 전체 시스템이 down될수도 있다는 것이다.
  * 우리는 이를 두개의 Pod를 죽이고 Deployment가 Pod가 재생성되도록 기다리게 함으로써 해볼 수 있다.
  * 이때쯤 Service는 replicas 전에 존재한다.
  * 이는 Pods에 퍼져있고, 올바른 environment variables를 가진 scheduler-level Service를 제공할 것이다.

  ```shell
  kubectl scale deployment my-nginx --replicas=0; kubectl scale deployment my-nginx --replicas=2;
  
  kubectl get pods -l run=my-nginx -o wide
  ```

  ```
  NAME                        READY     STATUS    RESTARTS   AGE     IP            NODE
  my-nginx-3800858182-e9ihh   1/1       Running   0          5s      10.244.2.7    kubernetes-minion-ljyd
  my-nginx-3800858182-j4rm4   1/1       Running   0          5s      10.244.3.8    kubernetes-minion-905m
  ```

* 파드가 죽고 재생성 되었으니 파드가 다른 이름을 가졌음을 주의하라.

  ```shell
  kubectl exec my-nginx-3800858182-e9ihh -- printenv | grep SERVICE
  ```

  ```
  KUBERNETES_SERVICE_PORT=443
  MY_NGINX_SERVICE_HOST=10.0.162.149
  KUBERNETES_SERVICE_HOST=10.0.0.1
  MY_NGINX_SERVICE_PORT=80
  KUBERNETES_SERVICE_PORT_HTTPS=443
  ```

### DNS

* Kubernetes는 자동으로 dns names를 다른 Services에 할당하는 DNS cluster addon Service를 제공한다.

  * cluster에서 이것이 동작하는지 확인할 수 있다.

  ```shell
  kubectl get services kube-dns --namespace=kube-system
  ```

  ```
  NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
  kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   8m
  ```

* 이것이 동작하고 있지 않다면 [활성화 시킬 수 있다.](http://releases.k8s.io/master/cluster/addons/dns/kube-dns/README.md#how-do-i-configure-it)

  * 이 섹션의 나머지 부분은 오랫동안 존재하는 IP(my-nginx) Service가 있고 그 IP로 이름을 할당한 DNS server(CoreDNS cluster addon)가 있어서 cluster의 어느 파드에서든 표준 방식으로(e.g. gethostbyname) Service와 통신할 수 있다고 가정한다.
  * 이를 테스트 하기 위해 다른 curl application을 실행시켜보자.

  ```shell
  kubectl run curl --image=radial/busyboxplus:curl -i --tty
  ```

  ```
  Waiting for pod default/curl-131556218-9fnch to be running, status is Pending, pod ready: false
  Hit enter for command prompt
  ```

  * 그러고 enter를 입력 후 `nslookup my-nginx`를 실행해 보아라.

  ```
  [ root@curl-131556218-9fnch:/ ]$ nslookup my-nginx
  Server:    10.0.0.10
  Address 1: 10.0.0.10
  
  Name:      my-nginx
  Address 1: 10.0.162.149
  ```

## Securing the Service

* 이제까지 우리는 cluster 안에서만 nginx server에 접속했다.

  * Service를 인터넷에 expose하기 전에 communication channel이 안전한지 확인하고 싶을 것이다.
  * 이를 위해 이것들이 필요할 것이다.
    * https를 위한 Self signed certificates(이미 identity certificate를 가지고 있지 않는 한)
    * certificates를 사용하도록 설정되어 있는 nginx server
    * 파드에서 certificates에 접속할 수 있도록 해주는 [secret](https://kubernetes.io/docs/concepts/configuration/secret/)

* 이러한 것들을 [nginx https example](https://github.com/kubernetes/examples/tree/master/staging/https-nginx/)에서 얻을 수 있다.

  * `go`와 `make`가 설치되어 있어야 한다.
  * 이런 것들을 설치하고 싶지 않다면 뒤의 수동 절차멧를 따라 해라.

  ```shell
  make keys secret KEY=/tmp/nginx.key CERT=/tmp/nginx.crt SECRET=/tmp/secret.json
  kubectl apply -f /tmp/secret.json
  ```

  ```
  secret/nginxsecret created
  ```

  ```shell
  kubectl get secrets
  ```

  ```
  NAME                  TYPE                                  DATA      AGE
  default-token-il9rc   kubernetes.io/service-account-token   1         1d
  nginxsecret           Opaque                                2         1m
  ```

* 다음은 make를 실행하는데 문제가 있는 경우에 수동으로 하는 절차이다.

  ```shell
  #create a public private key pair
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /d/tmp/nginx.key -out /d/tmp/nginx.crt -subj "/CN=my-nginx/O=my-nginx"
  #convert the keys to base64 encoding
  cat /d/tmp/nginx.crt | base64
  cat /d/tmp/nginx.key | base64
  ```

* 이전 커맨드의 output을 사용하여 다음과 같은 yaml을 생성하라.

  * base64로 인코딩된 값은 한줄이어야 한다.

  ```yaml
  apiVersion: "v1"
  kind: "Secret"
  metadata:
    name: "nginxsecret"
    namespace: "default"
  data:
    nginx.crt: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURIekNDQWdlZ0F3SUJBZ0lKQUp5M3lQK0pzMlpJTUEwR0NTcUdTSWIzRFFFQkJRVUFNQ1l4RVRBUEJnTlYKQkFNVENHNW5hVzU0YzNaak1SRXdEd1lEVlFRS0V3aHVaMmx1ZUhOMll6QWVGdzB4TnpFd01qWXdOekEzTVRKYQpGdzB4T0RFd01qWXdOekEzTVRKYU1DWXhFVEFQQmdOVkJBTVRDRzVuYVc1NGMzWmpNUkV3RHdZRFZRUUtFd2h1CloybHVlSE4yWXpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSjFxSU1SOVdWM0IKMlZIQlRMRmtobDRONXljMEJxYUhIQktMSnJMcy8vdzZhU3hRS29GbHlJSU94NGUrMlN5ajBFcndCLzlYTnBwbQppeW1CL3JkRldkOXg5UWhBQUxCZkVaTmNiV3NsTVFVcnhBZW50VWt1dk1vLzgvMHRpbGhjc3paenJEYVJ4NEo5Ci82UVRtVVI3a0ZTWUpOWTVQZkR3cGc3dlVvaDZmZ1Voam92VG42eHNVR0M2QURVODBpNXFlZWhNeVI1N2lmU2YKNHZpaXdIY3hnL3lZR1JBRS9mRTRqakxCdmdONjc2SU90S01rZXV3R0ljNDFhd05tNnNTSzRqYUNGeGpYSnZaZQp2by9kTlEybHhHWCtKT2l3SEhXbXNhdGp4WTRaNVk3R1ZoK0QrWnYvcW1mMFgvbVY0Rmo1NzV3ajFMWVBocWtsCmdhSXZYRyt4U1FVQ0F3RUFBYU5RTUU0d0hRWURWUjBPQkJZRUZPNG9OWkI3YXc1OUlsYkROMzhIYkduYnhFVjcKTUI4R0ExVWRJd1FZTUJhQUZPNG9OWkI3YXc1OUlsYkROMzhIYkduYnhFVjdNQXdHQTFVZEV3UUZNQU1CQWY4dwpEUVlKS29aSWh2Y05BUUVGQlFBRGdnRUJBRVhTMW9FU0lFaXdyMDhWcVA0K2NwTHI3TW5FMTducDBvMm14alFvCjRGb0RvRjdRZnZqeE04Tzd2TjB0clcxb2pGSW0vWDE4ZnZaL3k4ZzVaWG40Vm8zc3hKVmRBcStNZC9jTStzUGEKNmJjTkNUekZqeFpUV0UrKzE5NS9zb2dmOUZ3VDVDK3U2Q3B5N0M3MTZvUXRUakViV05VdEt4cXI0Nk1OZWNCMApwRFhWZmdWQTRadkR4NFo3S2RiZDY5eXM3OVFHYmg5ZW1PZ05NZFlsSUswSGt0ejF5WU4vbVpmK3FqTkJqbWZjCkNnMnlwbGQ0Wi8rUUNQZjl3SkoybFIrY2FnT0R4elBWcGxNSEcybzgvTHFDdnh6elZPUDUxeXdLZEtxaUMwSVEKQ0I5T2wwWW5scE9UNEh1b2hSUzBPOStlMm9KdFZsNUIyczRpbDlhZ3RTVXFxUlU9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
    nginx.key: "LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRQ2RhaURFZlZsZHdkbFIKd1V5eFpJWmVEZWNuTkFhbWh4d1NpeWF5N1AvOE9ta3NVQ3FCWmNpQ0RzZUh2dGtzbzlCSzhBZi9WemFhWm9zcApnZjYzUlZuZmNmVUlRQUN3WHhHVFhHMXJKVEVGSzhRSHA3VkpMcnpLUC9QOUxZcFlYTE0yYzZ3MmtjZUNmZitrCkU1bEVlNUJVbUNUV09UM3c4S1lPNzFLSWVuNEZJWTZMMDUrc2JGQmd1Z0ExUE5JdWFubm9UTWtlZTRuMG4rTDQKb3NCM01ZUDhtQmtRQlAzeE9JNHl3YjREZXUraURyU2pKSHJzQmlIT05Xc0RadXJFaXVJMmdoY1kxeWIyWHI2UAozVFVOcGNSbC9pVG9zQngxcHJHclk4V09HZVdPeGxZZmcvbWIvNnBuOUYvNWxlQlkrZStjSTlTMkQ0YXBKWUdpCkwxeHZzVWtGQWdNQkFBRUNnZ0VBZFhCK0xkbk8ySElOTGo5bWRsb25IUGlHWWVzZ294RGQwci9hQ1Zkank4dlEKTjIwL3FQWkUxek1yall6Ry9kVGhTMmMwc0QxaTBXSjdwR1lGb0xtdXlWTjltY0FXUTM5SjM0VHZaU2FFSWZWNgo5TE1jUHhNTmFsNjRLMFRVbUFQZytGam9QSFlhUUxLOERLOUtnNXNrSE5pOWNzMlY5ckd6VWlVZWtBL0RBUlBTClI3L2ZjUFBacDRuRWVBZmI3WTk1R1llb1p5V21SU3VKdlNyblBESGtUdW1vVlVWdkxMRHRzaG9reUxiTWVtN3oKMmJzVmpwSW1GTHJqbGtmQXlpNHg0WjJrV3YyMFRrdWtsZU1jaVlMbjk4QWxiRi9DSmRLM3QraTRoMTVlR2ZQegpoTnh3bk9QdlVTaDR2Q0o3c2Q5TmtEUGJvS2JneVVHOXBYamZhRGR2UVFLQmdRRFFLM01nUkhkQ1pKNVFqZWFKClFGdXF4cHdnNzhZTjQyL1NwenlUYmtGcVFoQWtyczJxWGx1MDZBRzhrZzIzQkswaHkzaE9zSGgxcXRVK3NHZVAKOWRERHBsUWV0ODZsY2FlR3hoc0V0L1R6cEdtNGFKSm5oNzVVaTVGZk9QTDhPTm1FZ3MxMVRhUldhNzZxelRyMgphRlpjQ2pWV1g0YnRSTHVwSkgrMjZnY0FhUUtCZ1FEQmxVSUUzTnNVOFBBZEYvL25sQVB5VWs1T3lDdWc3dmVyClUycXlrdXFzYnBkSi9hODViT1JhM05IVmpVM25uRGpHVHBWaE9JeXg5TEFrc2RwZEFjVmxvcG9HODhXYk9lMTAKMUdqbnkySmdDK3JVWUZiRGtpUGx1K09IYnRnOXFYcGJMSHBzUVpsMGhucDBYSFNYVm9CMUliQndnMGEyOFVadApCbFBtWmc2d1BRS0JnRHVIUVV2SDZHYTNDVUsxNFdmOFhIcFFnMU16M2VvWTBPQm5iSDRvZUZKZmcraEppSXlnCm9RN3hqWldVR3BIc3AyblRtcHErQWlSNzdyRVhsdlhtOElVU2FsbkNiRGlKY01Pc29RdFBZNS9NczJMRm5LQTQKaENmL0pWb2FtZm1nZEN0ZGtFMXNINE9MR2lJVHdEbTRpb0dWZGIwMllnbzFyb2htNUpLMUI3MkpBb0dBUW01UQpHNDhXOTVhL0w1eSt5dCsyZ3YvUHM2VnBvMjZlTzRNQ3lJazJVem9ZWE9IYnNkODJkaC8xT2sybGdHZlI2K3VuCnc1YytZUXRSTHlhQmd3MUtpbGhFZDBKTWU3cGpUSVpnQWJ0LzVPbnlDak9OVXN2aDJjS2lrQ1Z2dTZsZlBjNkQKckliT2ZIaHhxV0RZK2Q1TGN1YSt2NzJ0RkxhenJsSlBsRzlOZHhrQ2dZRUF5elIzT3UyMDNRVVV6bUlCRkwzZAp4Wm5XZ0JLSEo3TnNxcGFWb2RjL0d5aGVycjFDZzE2MmJaSjJDV2RsZkI0VEdtUjZZdmxTZEFOOFRwUWhFbUtKCnFBLzVzdHdxNWd0WGVLOVJmMWxXK29xNThRNTBxMmk1NVdUTThoSDZhTjlaMTltZ0FGdE5VdGNqQUx2dFYxdEYKWSs4WFJkSHJaRnBIWll2NWkwVW1VbGc9Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K"
  ```

* 이제 파일을 이용해 secrets를 생성해보자.

  ```shell
  kubectl apply -f nginxsecrets.yaml
  kubectl get secrets
  ```

  ```
  NAME                  TYPE                                  DATA      AGE
  default-token-il9rc   kubernetes.io/service-account-token   1         1d
  nginxsecret           Opaque                                2         1m
  ```

* nginx replicas를 수정하여 Service와 secret의 certificate를 사용하고, 80, 443 ports를 expose하는 https server를 실행해보자.

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nginx
    labels:
      run: my-nginx
  spec:
    type: NodePort
    ports:
    - port: 8080
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      protocol: TCP
      name: https
    selector:
      run: my-nginx
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-nginx
  spec:
    selector:
      matchLabels:
        run: my-nginx
    replicas: 1
    template:
      metadata:
        labels:
          run: my-nginx
      spec:
        volumes:
        - name: secret-volume
          secret:
            secretName: nginxsecret
        containers:
        - name: nginxhttps
          image: bprashanth/nginxhttps:1.0
          ports:
          - containerPort: 443
          - containerPort: 80
          volumeMounts:
          - mountPath: /etc/nginx/ssl
            name: secret-volume
  ```

* 주목할만한 부분은 nginx-secure-app manifest이다.

  * Deployment와 Service specification은 같은 파일 안에 포함되어있다.
  * [nginx server](https://github.com/kubernetes/examples/tree/master/staging/https-nginx/default.conf)는 HTTP 트래픽을 80포트에서 서비스하고 HTTPS 트래픽을 443포트에서 서비스하며 nginx Service는 둘 모두를 expose한다.

  ```shell
  kubectl delete deployments,svc my-nginx; kubectl create -f ./nginx-secure-app.yaml
  ```

* 이 때 어느 노드에서나 nginx server로 연결할 수 있다.

  ```shell
  kubectl get pods -o yaml | grep -i podip
      podIP: 10.244.3.5
  node $ curl -k https://10.244.3.5
  ...
  <h1>Welcome to nginx!</h1>
  ```

* 마지막 단계에서 `-k` 파라미터를 curl에 사용하였는지 봐보자. 이는 certificate generation time에 nginx를 실행하는 파드에 대해서 우리가 아무것도 모르기 때문이고 따라서 우리는 curl에게 CName mismatch를 무시하라고 알려주어야 하기 때문이다.

  * Service를 생성함으로써 우리는 실제 Service lookup하는 동안 파드에 의해 사용되는 DNS 이름을 가진 certificate에 쓰이는 CName을 연결할 수 있다.
  * 이를 파드(간단하게 같은 secret 다시 사용해보자. 파드는 Service에 접속하는 데 nginx.crt만 필요하다.)을 로 테스트 해보자.

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: curl-deployment
  spec:
    selector:
      matchLabels:
        app: curlpod
    replicas: 1
    template:
      metadata:
        labels:
          app: curlpod
      spec:
        volumes:
        - name: secret-volume
          secret:
            secretName: nginxsecret
        containers:
        - name: curlpod
          command:
          - sh
          - -c
          - while true; do sleep 1; done
          image: radial/busyboxplus:curl
          volumeMounts:
          - mountPath: /etc/nginx/ssl
            name: secret-volume
  ```

  ```shell
  kubectl apply -f ./curlpod.yaml
  kubectl get pods -l app=curlpod
  ```

  ```
  NAME                               READY     STATUS    RESTARTS   AGE
  curl-deployment-1515033274-1410r   1/1       Running   0          1m
  ```

  ```shell
  kubectl exec curl-deployment-1515033274-1410r -- curl https://my-nginx --cacert /etc/nginx/ssl/nginx.crt
  ...
  <title>Welcome to nginx!</title>
  ...
  ```

## Exposing the Service

* application의 어떤 부분에서는 Service를 external IP주소로 expose하고 싶을 것이다.

  * Kubernetes는 두가지 방식을 지원한다. : NodePorts, LoadBalancers
  * 지난 section에서 생성된 Service는 이미 `NodePort`를 사용하고 있어서 nginx HTTPS replica는 public IP를 가지고 있으면 인터넷에 트래픽을 서비스할 준비가 되어있다.

  ```shell
  kubectl get svc my-nginx -o yaml | grep nodePort -C 5
    uid: 07191fb3-f61a-11e5-8ae5-42010af00002
  spec:
    clusterIP: 10.0.162.149
    ports:
    - name: http
      nodePort: 31704
      port: 8080
      protocol: TCP
      targetPort: 80
    - name: https
      nodePort: 32453
      port: 443
      protocol: TCP
      targetPort: 443
    selector:
      run: my-nginx
  ```

  ```shell
  kubectl get nodes -o yaml | grep ExternalIP -C 1
      - address: 104.197.41.11
        type: ExternalIP
      allocatable:
  --
      - address: 23.251.152.56
        type: ExternalIP
      allocatable:
  ...
  
  $ curl https://<EXTERNAL-IP>:<NODE-PORT> -k
  ...
  <h1>Welcome to nginx!</h1>
  ```

* 이제 `my-nginx` Service의 `Type`을 `NodePort`에서 `LoadBalancer`로 변경한 후 다시 Service를 재생성하여 cloud load balancer를 사용하도록 하자.

  ```shell
  kubectl edit svc my-nginx
  kubectl get svc my-nginx
  ```
  
  ```
  NAME       TYPE        CLUSTER-IP     EXTERNAL-IP        PORT(S)               AGE
  my-nginx   ClusterIP   10.0.162.149   162.222.184.144    80/TCP,81/TCP,82/TCP  21s
  ```
  
  ```
  curl https://<EXTERNAL-IP> -k
  ...
  <title>Welcome to nginx!</title>
  ```
  
* `EXTERNAL-IP` 컬럼에 있는 IP 주소는 pulblic internet에서 사용가능한 것 중 하나이다.

  * `CLUSTER-IP`는 cluster 내부 / private cloud network에서만 사용할 수 있다.

* AWS에서는 `LoadBalancer` 타입이 IP가 아닌 (긴)hostname을 사용하는 ELB를 생성한다는 것에 유의하자.

  * 이는 사실 표준의 `kubectl get svc` 출력에는 너무 길어서 `kubectl describe service my-nginx`를 사용해서 봐야한다.

  * 다음과 같이 보일 것이다.

    ```shell
    kubectl describe service my-nginx
    ...
    LoadBalancer Ingress:   a320587ffd19711e5a37606cf4a74574-1142138393.us-east-1.elb.amazonaws.com
    ...
    ```

