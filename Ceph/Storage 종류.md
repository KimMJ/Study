# Storage 종류

## 3 Storage

1. Block Storage : 데이터를 임의로 구성해 **동일한 크기의 볼륨**으로 나눈다.
2. File Storage : 데이터를 폴더의 **파일 계층 구조**로 구성하고 표현한다.
3. Object Storage : 데이터를 관리하고 연결된 메타데이터와 이를 연결한다.

## What is File Storage?

* file-level, file-based storage

* 우리가 일반적으로 이해하고 있는 storage

* 데이터가 폴더 안에 단일 정보로 저장.

* 데이터에 엑세스 하기 위해서는 경로를 알아야 한다.

* 계층 구조 스토리지

  ![file storage](https://www.redhat.com/cms/managed-files/fileStorage_orange_320x242_0.png)

## What is Block Storage?

* 데이터를 블록으로 쪼갬
* 데이터를 별도의 조각으로 분리해 저장
* 각 데이터 블록은 고유 식별자(ID)를 받는다.
  * 스토리지 시스템이 더 작은 데이터 조각을 원하는 곳에 배치할 수 있도록 함.
  * 일부 데이터는 리눅스에 저장하고, 일부는 윈도우에 저장하는 것도 가능
* 데이터를 사용자의 환경에서 분리하여 분산시켰다가 데이터가 요청이 되면 스토리지 소프트웨어가 이 데이터 블록들을 조합해서 사용자에게 제공
  * SAN(Storage Area Network) 환경에서 배포되고 가동되는 서버에 연결해야함.
* 블록 스토리지는 파일 스토리지처럼 단일 데이터 경로에 의존하지 않아서 신속한 검색이 가능
* 각 블록은 독립적으로 존재하며 파티션으로 분할될 수 있음
  * 서로 다른 운영 체제에 액세스 가능
  * 사용자는 자유롭게 데이터 설정할 수 있음
  * 데이터를 효율적이고 안정적으로 저장하는 방법
* 사용과 관리도 간편
* 대규모 트랜잭션을 수행하는 기업과 대용량 데이터베이스를 배포하는 기업에서도 원활하게 동작
  * 더 많은 데이터를 저장해야 할수록 블록 스토리지를 사용하는 것이 유리
* 비용은 많이 듬

![block storage](https://www.redhat.com/cms/managed-files/blockStorage_orange_320x242_0.png)

## What is Object Storage?

* 파일들이 작게 나뉘어 여러 하드웨어에 분산되는 평면적인 구조
* 데이터는 오브젝트라 불리는 개별 단위로 나뉨
  * 서버의 블록이나 폴더에 파일을 보관하는 대신 단일 repository에 보관
* 오브젝트 스토리지의 볼륨은 모듈 단위로 동작
* 각각은 독립적인 repository이며 데이터, 오브젝트가 distributed system에 존재하도록 허용하는 ID, 해당 데이터를 설명하는 metadata를 가지고 있다.
  * metadata는 사용 기간, 개인 정보/ 보안 및 액세스 비상 대책등의 상세사항이 포함됨.
  * metadata는 매우 상세할 수 있음.
    * 영상 촬영 위치, 사용된 카메라 기종, 각 프레임에 출연한 배우 이름 등의 정보 저장 가능
* 검색 시 metadata와 ID를 사용
* HTTP API 사용
* 사용한 만큼만 비용을 지불하면 되어 비용 효율적
* 확장하기 쉬움
  * public cloud storage에 적합
* 정적 데이터에 적합
* 애플리케이션이 신속하게 데이터를 검색할 수 있음
* 비정형(unstructed) 데이터에 좋음
* 오브젝트는 수정이 불가능
  * 한번에 오브젝트 작성을 완료해야 함
* 전통적인 DB와 연동이 잘 되지 않음
  * 오브젝트 작성이 느림
  * 오브젝트 스토리지 API를 쓰기 위해 어플리케이션을 작성하는 것이 파일스토리지보다 어려움

-----

https://www.redhat.com/ko/topics/data-storage/file-block-object-storage