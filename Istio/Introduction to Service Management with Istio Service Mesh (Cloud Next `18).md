# Introduction to Service Management with Istio Service Mesh (Cloud Next `18)

https://www.youtube.com/watch?v=wCJrdKdD6UM

모놀로틱은 하나의 코드에 모든것을 넣기 때문에 소스의 변경이 있을 때 모든 것을 함께 배포해야 한다. 마이크로 서비스는 이러한 것에 대해 하나의 비즈니스 로직 셋에 대해서만 배포하고 관리하면 되는 이점이 있다.

그런데 마이크로 서비스가 점점 커지면 시스템 안에서 장애가 났을 때 어느 부분이 실질적으로 장애를 유발하는지, 또 어떤 서비스가 어떤 서비스와 통신하는지 알기가 어렵다. 그리고 마이크로 서비스 별로 다른 언어로 작성되기 때문에 프로토콜이 조금씩 다를 수 있다. 따라서 서비스를 유지하는데 어렵다.

## What is Istio?

> An open services platform to manage service interactions across container- and VM- based workloads

