# Spinnaker

## Application management

클라우드 리소스를 관리하고 보기 위해서 어플리케이션 매니지먼트를 사용한다.

현대의 기업은 서비스의 묶음으로 운영한다. - "어플리케이션"이나 "마이크로서비스"라고 부른다. 스피네이커는 이런 컨셉을 모델링한다.

어플리케이션, 클러스터, 서버 그룹은 서비스를 설명하기 위해 스피네이커가 사용하는 핵심 개념이다. 로드 발란서와 방화벽은 서비스를 어떻게 유저에게 노출시키는지에 관한 것이다.

![clusters](https://www.spinnaker.io/concepts/clusters.png)

### Server group

* 기본 리소스
* deployable artifact(VM 이미지, 도커 이미지, 소스 위치)와 인스턴스의 수, 오토 스케일링 정책, 메타데이타 등 configuration settings
* 로드밸런서와 방화벽을 추가할 수 있음.
* 서버 그룹은 동작하고 있는 소프트웨어의 인스턴스 모음이다.

### Account

A Spinnaker account corresponds to a physical Kubernetes cluster. If you are unsure which account to use, talk to your Spinnaker admin.

## Cluster

* V000 : configmap -> ?
* V001에서부터 올라가는 것은 replicaset이 변경되면 추가됨.
* 업데이트 느림.

--------

## Manifest 관련

* pipeline의 upstream에서 docker image / k8s configmap / k8s secret이 update 된다면 spinnaker는 자동으로 detect한다.
  * deploy할 manifest에 자동으로 적용한다.

## 