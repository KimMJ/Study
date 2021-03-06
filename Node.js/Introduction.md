# Node.js Introduction

* Node.js는 Chrome V8 JavaScript 엔진으로 빌드된 JavaScript 런타임이다.

## 특징

##### 비동기 I/O 처리, 이벤트 위주

* Node.js 라이브러리의 모든 API는 비동기식. (Nonblocking)
* Node.js 기반 서버는 API가 실행되었을때, 데이터를 반환할때까지 기다리지 않고 다음 API를 실행시킴.
* 이전에 실행했던 API가 결과값을 반환할 시, Node.js의 이벤트 알림 메커니즘을 통해 결과값을 받아옴.

##### 빠른 속도

* 구글 크롬의 V8 JavaScript 엔진을 사용하여 빠른 코드 실행 제공.

##### 단일 쓰레드, 뛰어난 확장성

* 이벤트 루프와 함께 단일 쓰레드 모델 사용.
  * 서버가 멈추지않고 반응하도록 해주어 서버의 확장성을 키워줌.


##### 노 버퍼링

* Node.js 어플리케이션은 데이터 버퍼링이 없고, 데이터를 chunk로 출력함.

##### 라이센스

* MIT License 적용

## 사용 분야

* 입출력이 잦은 어플리케이션
* 데이터 스트리밍 어플리케이션
* 데이터를 실시간으로 다루는 어플리케이션
* JSON API 기반 어플리케이션
* 싱글페이지 어플리케이션

**CPU 사용율이 높은 어플리케이션에선 Node.js 사용을 권장하지 않음**
