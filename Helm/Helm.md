# Helm

## Basic

### helm이란

* kubernetes 패키지 매니저.
* yaml의 집합 : chart
* 차트 관리 도구 : helm
* tiller : chart 레파지토리 연계

### helm 용어

* chart : 쿠버네티스 애플리케이션을 만들기 위해 필요한 정보들의 묶음
* config : 배포 가능한 객체들을 만들기 위해 패키지된 차트에 넣어서 사용할 수 있는 설정들
* release : kubernetes에 배포된 어플리케이션

### helm 구성요소

1. helm client : 사용자가 직접 사용할 수 있는 명령줄 도구
2. tiller server : 클러스터 내부에서 실행되어 helm 클라이언트에서 오는 명령을 받아 쿠버네티스 api와 통신하는 역할. pod로 배포됨. (namespace : kube-system)

![helm architecture](https://sktelecom-oslab.github.io/Virtualization-Software-Lab/assets/img/helm-architecture.png)

* client는 gRPC 프로토콜을 이용해서 tiller와 통신.
* chart를 배포하기 위해서 `helm install` 명령어를 사용하면 helm client는 chart를 tiller에게 전송.
* tiller는 받은 chart를 rendering해서 kubernetes 객체 정보를 정의하는 manifest 파일을 생성.
* 이 manifest 파일을 kubernetes api를 통해 kubernetes 위에 배포.

### helm 명령어

#### helm init

Tiller를 설치하는 명령어입니다.

* --service-account : service account의 이름을 지정해 주는 것입니다.
* **11.3.2 helm 관련 추가 설치 및 설정**에서 helm-service-account.yaml을 kubernetes에 배포했습니다.
* 이는 Tiller를 배포한 것입니다. (namespace : kube-system, name : tiller)
* 이것과 helm을 연결시키는 작업이라고 생각하면 될 것 같습니다.

