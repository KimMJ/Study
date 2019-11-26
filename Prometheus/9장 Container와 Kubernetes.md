# 9장 Container와 Kubernetes

모든 프로메테우스 컴포넌트는 컨테이너에서 잘 실행되지만, node exporter는 예외이다.

## cAdvisor

* cAdvisor Exporter는 cgroups에 대한 메트릭을 제공함.
* 컨테이너 메트릭은 `container_` prefix를 가지며 모두 `id` label을 가지고 있음.
* `/docker/` 로 시작하는 `id` label은 docker와 docker container에서 나왔으며 `/user.slice` 와 `/system.slice/` 로 시작하는 `id` label은  머신에서 실행되는 `systemd` 에서 나왔다.

### CPU

* 컨테이너 CPU에 대한 메트릭은 `container_cpu_usage_seconds_total` 과 `container_cpu_system_seconds_total`, `container_cpu_user_seconds_total` 이 있음.

### 메모리

* 메모리 사용 관련 메트릭들은 완벽히 명확하지는 않기 때문에 좀 더 깊이 이해하려면 코드와 [문서](http://bit.ly/2lijiBR)를 상세히 봐야 한다.
* `container_memory_cache` 는 컨테이너가 사용하는 페이지 캐시로 바이트 단위
* `container_memory_rss` 는 메모리에 상주하는 세트의 크기(RSS)로 바이트 단위
  * 동일한 RSS나 프로세스가 사용하는 물리적인 메모리가 아니므로 매핑된 파일의 크기를 제외함.
* `container_memory_usage_bytes` 는 RSS와 페이지 캐시이며 0으로 제한되지 않은 경우 `container_spec_memory_limit_bytes` 에 의해 제한됨.
* `container_memory_working_set_bytes` 는 `container_memory_usage_bytes` 에서 비활성 파일 기반 메모리 (커널의 `total_inactive_file`)을 빼서 계산됨.
* 일반적으로 cgroups의 메트릭보단 프로세스 자체의 `process_resident_memory_bytes` 와 같은 메트릭에 의존하는 것을 권장.

### label

* cgroups는 계층 구조의 루트에 `/` cgroup을 갖는 계층구조로 구조화되어있음.
* 각각의 cgroup의 메트릭들은 하위 cgroup의 사용량을 포함.
* 도커가 컨테이너에 대해 갖는 모든 메타데이터 레이블은 `container_label_` prefix와 함께 포함됨.
  * 이런 임의의 레이블은 데이터 수집 시 모니터링을 망칠 수 있으며, `labeldrop` 동작을 통해 제거할 수 있다.

## Kubernetes

### Kubernetes 실행하기

prometheus-deployment.yml

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
---
apiVersion: v1
data:
  prometheus.yml: |
    global:
      scrape_interval: 10s
    scrape_configs:
    - job_name: 'kubelet'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true  # Unfortunately required with Minikube.
    - job_name: 'cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true  # Unfortunately required with Minikube.
      metrics_path: /metrics/cadvisor
    - job_name: 'k8apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true  # Unfortunately required with Minikube.
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'k8services'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels:
          - __meta_kubernetes_namespace
          - __meta_kubernetes_service_name
        action: drop
        regex: default;kubernetes
      - source_labels:
          - __meta_kubernetes_namespace
        regex: default
        action: keep
      - source_labels: [__meta_kubernetes_service_name]
        target_label: job
    - job_name: 'k8pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        regex: metrics
        action: keep
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: job
kind: ConfigMap
metadata:
  name: prometheus-config
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.1.0
        ports:
        - containerPort: 9090
          name: default
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
      volumes:
      - name: config-volume
        configMap:
         name: prometheus-config
---
kind: Service
apiVersion: v1
metadata:
  name: prometheus
spec:
  selector:
    app: prometheus
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 9090
```

* 쿠버네티스에서 모니터링의 배경이 되는 핵심 아이디어를 보여주기 위한 기본 쿠버네티스 설정.
* 운영 환경에 바로 사용하기 위한 것은 아님.
  * 프메테우스가 재시작할 때마다 모든 데이터는 손실됨.

### Service Discovery

* 프로메테우스에서 사용할 수 있는 Kubernetes Service Discovery에는 node, endpoint, service, pod, ingress 5가지 타입이 있음.

#### Node

* node service discovery는 kubernetes를 구성하는 노드를 발견하는 데 쓰임.

* 쿠버네티스와 관련된 인프라 모니터링에서 사용됨.

* kubernetes cluster의 상태 모니터링의 일부로 kubelet에서 데이터를 수집해야 함.

  ```yaml
      scrape_configs:
      - job_name: 'kubelet'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true  # Unfortunately required with Minikube.
  ```

```yaml
 job_name: 'kubelet'
```

* 기본 작업 레이블이 제공되며, `relabel_configs` 가 없기 때문에 `kubelet` 이 작업 레이블이 됨.

```yaml
kubernetes_sd_configs:
- role: node
```

* 단일 Kubernetes Service Discovery는 node 역할과 함께 제공됨.
* node 역할은 각 kubelet에 대해 하나의 대상을 발견함.
* 프로메테우스가 클러스터 내부에서 실행되기 때문에 Kubernetes Service Discovery의 default는 Kubernetes API로 인증하도록 이미 설정되어있음.

```yaml
scheme: https
tls_config:
  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  insecure_skip_verify: true
```

* kubelet은 HTTPS를 통해 `/metrics` 를 제공한다.

  * 따라서 scheme을 지정해댜 한다.

* kubelet 자체 `/metrics`는 kubelet에 대한 메트릭만 포함

  * 컨테이너 수준의 정보는 포함하지 않음.

* kubelet은 `/metrics/cadvisor` endpoint를 갖는 cAdvisor를 내장하고 있음.

* 내장된 cAdvisor에서 데이터의 수집은 kubelet에 사용된 데이터 수집 구성에 `metrics_path`를 추가하면 됨.

  * 내장 cAdvisor는 Kubernetes의 namespace와 pod_name에 대한 레이블을 포함함.

* ```yaml
  scrape_configs:
    - job_name: 'cadvisor'
      kubernetes_sd_configs:
        - role: node
      scheme: https
      tls_configs:
        ca_file: /var/runm/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      metrics_path: /metrics/advisor
  ```

### Service

* Kubernetes상의 각 애플리케이션이 서로를 발견하는 방법으로 Services를 이용할 가능성이 있다.
* 프로메테우스가 매번 다른 애플리케이션 인스턴스에 대해서 데이터를 수집할 수 있기 때문에 로드밸런서를 통한 대상의 데이터 수집은 현명한 방법이 아님.

### Endpoint

* 프로메테우스는 각 애플리케이션 인스턴스가 대상을 갖도록 구성되어야 하며, endpoints 역할은 바로 이 기능을 제공함.

* service는 pod에 의해 지원된다. pod는 강하게 결합된 컨테이너의 그룹으로, 네트워크와 저장장치를 공유한다.

* 각 Kubernetes 서비스 포트에 대해 endpoint service discovery는 해당 서비스를 지원하는 각 파드의 대상을 반환한다.

  * 다른 모든 파드의 포트는 모니터링 대상으로 반환된다.

* API 서버를 발견하고 데이터를 수집하는 데이터 구성

  ```yaml
      scrape_configs:
      - job_name: 'k8apiserver'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true  # Unfortunately required with Minikube.
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
  ```

* ```yaml
  job_name : 'k8apiserver'
  ```

  job label은 k8apiserver가 된다.

* ```yaml
  kubernetes_sd_configs:
  - role: endpoints
  ```

  endpoints 역할을 사용하는 단일 Kubernetes Service Discovery가 있으며 이는 각 Service를 지원하는 모든 파드의 모든 포트를 Target으로 반환한다.

* ```yaml
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  ```

  kubelet과 마찬가지로 API server는 HTTPS를 통해 제공된다. 그리고 bearer_token_file을 제공하는 인증이 필요하다.

* ```yaml
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https
  ```

  이 reballing 구성은 default namespace에 있는 대상만 반환하며 https라는 포트를 갖는 kubernetes service의 일부이다.

* API 서버를 제외한 모든 쿠버네티스 서비스를 지원하는 파드 데이터를 수집하는 구성

  ```yaml
  scrape_config:
    - job_name: 'k8services'
      kubernetes_sd_configs:
        - role: endpoints
      relabel_configs:
        - source_labels:
          - __meta_kubernetes_namespace
          - __meta_kubernetes_service_name
          regex: default;kubernetes
          action: drop
        - source_labels:
          - __meta_kubernetes_namespace
          regex: default
          action: keep
        - source_labels: [__meta_kubernetes_service_name]
          target_label: job
  ```

* ```yaml
  job_name: 'k8services'
  kubernetes_sd_configs:
    - role: endpoints
  ```

  job 이름과 kubernetes endpoints 역할을 제공함. 우리가 알고있는 대상은 모두 HTTP이기 때문에 HTTPS 설정은 없다.

* ```yaml
      relabel_configs:
        - source_labels:
          - __meta_kubernetes_namespace
          - __meta_kubernetes_service_name
          regex: default;kubernetes
          action: drop
        - source_labels:
          - __meta_kubernetes_namespace
          regex: default
          action: keep
        - source_labels: [__meta_kubernetes_service_name]
          target_label: job
  ```

  API 서버는 제외하였고, 애플리케이션을 시작하는 default namespace에 대해서만 살펴봄.

* ```yaml
  - source_labels: [__meta_kubernetes_service_name]
    target_label: job
  ```

  kubernetes service name을 가지고 job label로 사용함.

### 파드

* endpoint service discovery는 service를 지원하는 주된 프로세스의 모니터링에 좋지만 service의 일부가 아닌 pod는 발견하지 못함.

* pod 역할은 파드를 검색하며, 파드의 모든 포트에 대한 대상을 반환함.

* 모든 파드는 서비스의 일부가 되고 나서 서비스 검색에 endpoint 역할을 사용해야 한다는 규약을 생성할 수 있음.

* kube-dns는 DNS 서비스를 제공함.

  * 프로메테우스 메트릭을 제공하는 metrics라는 포트를 비롯한 여러개의 포트를 가짐.

* metrics 이름을 갖는 모든 파드 포트를 검색하고 컨테이너 이름을 job label로 사용하기 위한 구성

  ```yaml
      - job_name: 'k8pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_container_port_name]
          regex: metrics
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_name]
          target_label: job
  ```

* 모니터링할 대상을 관리하는 또 다른 방법은 crd를 사용하는 prometheus-operator이다.

  * operator는 prometheus의 alertmanager 실행도 관리한다.

### Ingress

* ingress는 Kubernetes service를 cluster 외부에 표시하는 방법이다.
* ingress는 서비스의 최상위에 있는 계층이기 때문에, 기본적으로 service 역할과 비슷하게 ingress 역할도 load balancer이다.
* 블랙박스 모니터링에만 사용해야함

#### kube-state-metrics

* Kubernetes Service Discovery를 사용하여 프로메테우스가 애플리케이션과 kubernetes infra에서 데이터를 수집하게 할 수 있음.
* 하지만 여기에 service, pod, deployment, 기타 자원에 대해 kubernetes가 알고있는 사항에 대한 메트릭은 포함되지 않음.
* 대신 다른 endpoint에서 이러한 메트릭을 얻을 수 있음.
  * endpoint가 존재하지 않다면 관련 정보를 추출하는 exporter를 가질 수 있음.
  * `kube-state-metrics`가 이에 해당.