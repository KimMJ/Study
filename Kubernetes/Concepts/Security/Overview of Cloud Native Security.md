# Overview of Cloud Native Security

Kubernetes Security(와 일반적인 security)는 연관된 파트가 정말 많은 엄청난 주제이다. web application이 동작하는 것을 돕기 위해 오픈 소스 소프트웨어가 많은 시스템 속에 들어가있는 요즘같은 시대에는 어떻게 보안에 대해서 생각하는지에 대한 직관을 도와줄 가이드가 중요한 콘셉으로 자리잡았다. 이 가이드는 Cloud Native Security 전반에 걸친 일반적인 컨셉의 mental model을 제시한다. mental model은 완전히 임의적이며 당신은 이를 당신의 software stack에서 어떻게 보안을 챙길 지 생각하는 것에 대한 도움을 얻는 정도로만 사용해야 한다.

## The 4C's of Cloud Native Sercurity

어떻게 security가 layer를 이루고 있는지 이해하기 쉽도록 도와주는 다음의 diagram을 살펴보자.

> Note: 이 layered approach는 security에 대한 [defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) approach 증대시켜준다. 이는 software system의 보호에 가장 좋은 것으로 여겨진다. 4C's는 CLoud, Clusters, Containers, Code를 의미한다.

![4c](img/4c.png)

### The 4C's of Cloud Native Security

위의 그림에서 볼 수 있듯이 각각의 4C's들은 그것이 있는 사각형의 security에 의존한다. Cloud, Container, Code의 poor security standards를 가지고는 코드레벨에서의 보안만으로 보호하는 것은 거의 불가능하다. 하지만 이러한 area들이 적절하게 보호가 되면 코드에 보안을 추가하는 것은 이미 강력한 기반의 보안을 더 강하게 해줄 것이다. 이러한 관심 영역은 아래에서 더 자세히 설명될 것이다.

## Cloud

많은 면에서 Cloud (또는 co-located servers나 corporate datacenter)는 Kubernetes cluster의  [trusted computing base](https://en.wikipedia.org/wiki/Trusted_computing_base)이다. 이러한 components가 그 자체로 취약하다면 (또는 취약한 방식으로 설정이 되어있다면) 이를 기반으로 만들어진 어떤 components던지 보안을 보장할 수 있는 방법은 없다고 봐도 된다. 각각의 cloud provider는 그들이 만든 최선의 security recommendations를 가지고 있다. 이를 통해서 customer들이 어떻게 그들의 환경에서 workloads를 안전하게 동작할 수 있게 할지를 알려준다. cloud security에 관한 recommendations를 주는 것은 이 문서의 범위 밖이다. cloud provider와 workload는 모두가 다르기 때문이다. 몇가지 security에 관한 유명한 cloud provider의 documentation의 링크를 제공한다. 또한 Kubernetes cluster를 구성하는 인프라를 보호하기 위한 일반적인 가이드를 제공한다.

### Cloud Provider Security Table

| IaaS Provider         | Link                                                         |
| :-------------------- | :----------------------------------------------------------- |
| Alibaba Cloud         | https://www.alibabacloud.com/trust-center                    |
| Amazon Web Services   | https://aws.amazon.com/security/                             |
| Google Cloud Platform | https://cloud.google.com/security/                           |
| IBM Cloud             | https://www.ibm.com/cloud/security                           |
| Microsoft Azure       | https://docs.microsoft.com/en-us/azure/security/azure-security |
| VMWare VSphere        | https://www.vmware.com/security/hardening-guides.html        |

사용자만의 하드웨어나 다른 cloud provider를 사용하고 있다면 security best practice에 대한 문서를 통한 컨설팅이 필요할 것이다.

### General Infrastructure Guidance Table

| Area of Concern for Kubernetes Infrastructure | Recommendation                                               |
| :-------------------------------------------- | :----------------------------------------------------------- |
| Network access to API Server (Masters)        | 이상적으로 모든 Kubernetes Master로의 access는 인터넷에서 public으로 허용되어서는 안되며 이는 cluster를 관리하기 위해 필요한 IP 주소들의 리스트를 제한하여 network access control을 해야한다. |
| Network access to Nodes (Worker Servers)      | 노드는 반드시 마스터로부터 지정된 포트로의 connections(network access control lists를 통하여)만 허용하도록 설정이 되어있어야 한다. 그리고 Kubernetes의 NodePort와 LoadBalancer로부터의 connections만 허용해야 한다. 가능하다면 이러한 노드들은 public internet에 전체적으로 expose되어서는 안된다. |
| Kubernetes access to Cloud Provider API       | 각 cloud provider는 Kubernetes Master와 Nodes에 대해 다른 permissions를 필요로 할 수 있다. 따라서 이러한 recommendation은 더 일반적으로 되어야 한다. 관리할 때 필요로 하는 리소스에 대한 [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)를 따르는 cloud provider access만을 cluster에게 제공하는 것이 가장 좋다. AWS에서의 Kops를 예시로 보면 다음을 보면 된다: https://github.com/kubernetes/kops/blob/master/docs/iam_roles.md#iam-roles |
| Access to etcd                                | etcd(Kubernetes의 datastore)에 대한 접근은 master에게만 허용이 되어야 한다. configuration에 따라 etcd를 TLS로 시도하도록 해야할 수도 있다. 더 자세한 정보는 다음을 통해 알아보아라: https://github.com/etcd-io/etcd/tree/master/Documentation#security |
| etcd Encryption                               | 가능하다면 rest상태의 모든 드라이브를 암호화 하는 것이 좋은 방법이지만, etcd는 전체 cluster에 관한 state(Secrets를 포함하여)를 가지고 있으므로 이 디스크는 특별히 rest 상태에서 암호화를 해야 한다. |

## Cluster

이 섹션은 쿠버네티스에서의 securing workloads에 대한 링크를 제공한다. securing Kubernetes에는 두가지 관심 영역이 있다.

* cluster를 구성하고 있는 설정이 가능한 components를 보호하는 것.
* 클러스터에서 동작하고 있는 components들을 보호하는 것.

### Components *of* the Cluster

사고나 악의적인 접근으로부터 cluster를 보호하고 싶고, 좋은 예시에 대한 정보를 얻고자 한다면 다음의 [securing your cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)에 관한 조언을 읽고 따르도록 해라.

### Components *in* the Cluster (your application)

application의 attack surface에 따라서 사용자는 security의 특정한 면에 포커스를 맞추고 싶어할 수 있다. 예를 들어 chain of other resources에 중요한 Service A라는 서비스를 실행중이고, resource exhaustion attack에 취약한 Service B라는 분리된 workload를 실행중일 때, Service B에 대해서 resource limits를 주지 않는 것은 Service A에도 위협을 줄 수 있다. 아래의 표는 Kubernetes에서 실행중인 workloads를 보호할 때 고려해야 하는 사항들에 대한 링크이다.

| Area of Concern for Workload Security                        | Recommendation                                               |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| RBAC Authorization (Access to the Kubernetes API)            | https://kubernetes.io/docs/reference/access-authn-authz/rbac/ |
| Authentication                                               | https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/ |
| Application secrets management (and encrypting them in etcd at rest) | https://kubernetes.io/docs/concepts/configuration/secret/ https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/ |
| Pod Security Policies                                        | https://kubernetes.io/docs/concepts/policy/pod-security-policy/ |
| Quality of Service (and Cluster resource management)         | https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/ |
| Network Policies                                             | https://kubernetes.io/docs/concepts/services-networking/network-policies/ |
| TLS For Kubernetes Ingress                                   | https://kubernetes.io/docs/concepts/services-networking/ingress/#tls |

## Container

Kubernetes에서 소프트웨어를 실행하기 위해서는 반드시 컨테이너로 해야한다. 이 때문에, 쿠버네티스의 주요한 workloads security로부터 이점을 얻기 위해서 반드시 고려해야할 security considerations가 있다. Conatiner security는 또한 이 가이드의 범위 바깥이지만, 이 주제에 대한 recommendations와 링크를 표로 정리해 보았다.

| Area of Concern for Containers                              | Recommendation                                               |
| :---------------------------------------------------------- | :----------------------------------------------------------- |
| Container Vulnerability Scanning and OS Dependency Security | image build 단계나 정기적으로 [CoreOS’s Clair](https://github.com/coreos/clair/)같은 툴을 통해서 conatiner의 알려진 취약점을 스캔하라. |
| Image Signing and Enforcement                               | TUF와 Notary같은 두 CNCF 프로젝트는 container images에 대한 서명이나 conatiners의 내용에 관한 system의 신뢰성을 유지하는 데 유용한 툴이다. Docker를 사용하고 있다면,[Docker Content Trust](https://docs.docker.com/engine/security/trust/content_trust/)로 Docker Engine에 빌드가 되어있다. enforcement piece에서,  [IBM’s Portieris](https://github.com/IBM/portieris) 프로젝트는 Kubernetes Dynamic Admission Controller처럼 동작하여 클러스터에 승인이 되기 전에 image가 적절하게 Notary를 통해서 서명이 되었는지를 확인한다. |
| Disallow privileged users                                   | containers를 구성할 때, 문서를 참고하여 어떻게 containers 내에서 conatiners의 목표를 수행하기 위해 OS의 최소 privilege necessary를 가지는 유저를 생성하는지 알아보아라. |

## Code

최종적으로 application code level로 내려가게 되면 중요한 attack surfaces의 하나는 가장 많은 제어 권한을 가지는 것이다. 이는 Kubernetes의 범위 밖이기는 하지만 몇가지 추천하는 사항들이 있다.

### General Code Security Guidance Table

| Area of Concern for Code              | Recommendation                                               |
| :------------------------------------ | :----------------------------------------------------------- |
| Access over TLS only                  | 당신의 코드가 TCP를 통한 통신을 필요로 한다면, client와 미리 TLS handshake를 수행하는 것이 이상적일 것이다. 몇가지 케이스를 제외하고는 기본 동작은 반드시 보든 전송을 암호화해야 한다. 한단계 더 앞서나가면 VPC에서의 "behind the firewall" 조차도 서비스간에 network traffic을 암호화하는 것이 더 좋다. 이는 mutual 또는 [mTLS](https://en.wikipedia.org/wiki/Mutual_authentication)라고 불리는 서비스를 가지고 있는 두 certificate 사이의 통신에 대한 양방향 verification을 실행하는 것을 통한 프로세스를 통해서 할 수 있다. Kubernetes에서는 [Linkerd](https://linkerd.io/) 와 [Istio](https://istio.io/) 같은 것으로 이를 할 수 있는 여러 툴들이 있다. |
| Limiting port ranges of communication | 이 recommendation은 약간 self-explanatory(자가 설명) 일수도 있지만, 가능하다면 통신이나 metric을 수집하는 데 정말 필요한 서비스에 대한 포트만을 expose하는 것이 좋다. |
| 3rd Party Dependency Security         | 우리의 applications가 우리의 codebase 바깥에 dependency가 있는 경향이 많기 때문에 CVE(Common Vulnerabilities and Exposures)가 현재는 문제가 되지 않더라도 code의 dependencies에 대한 정기적인 스캔을 확실히 하는 것은 좋은 방법이다. 각 언어는 이 체크를 자동으로 해주는 툴을 가지고 있다. |
| Static Code Analysis                  | 대부분의 언어는 코드의 일부분을 분석하여 안전하지 않을 수 있는 코딩 습관을 분석해준다. 자동화된 툴을 사용하여 체크를 할 수 있을 때마다 시행해주어 일반적인 보안 에러에 대해 codebase를 스캔할 수 있다. 몇가지 툴은 이곳에서 확인할 수 있다: https://www.owasp.org/index.php/Source_Code_Analysis_Tools |
| Dynamic probing attacks               | 몇가지 자동화 툴은 서비스에 안좋은 영향을 줄 수 있는 잘 알려진 공격들을 서비스에서 수행할 수 있다. 이들은 SQL injecdtion, CSRF, XSS를 포함한다. 유명한 dynamic analysis tools중 하나는 [OWASP Zed Attack proxy](https://www.owasp.org/index.php/OWASP_Zed_Attack_Proxy_Project)이다. |

## Robust automation

웨서 언급한 제안사항들의 대부분은 사실상 code delivery pipeline에서 security를 체크하는 과정 중에 자동화 될 수 있는 것들이다. software delivery에 관한 접근인 "Continuous Hacking"을 더 자세히 알아보려면 [this article](https://thenewstack.io/beyond-ci-cd-how-continuous-hacking-of-docker-containers-and-pipeline-driven-security-keeps-ygrene-secure/)이 더 자세한 사항을 알려줄 것이다.