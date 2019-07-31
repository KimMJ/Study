# TTL(Time To Live) Controller for Finished Resources

**FEATURE STATE: Kuberetes v1.12 - alpha**

* TTL controller는 TTL 메커니즘을 제공하여 실행을 마친 리소스 오브젝트의 라이프타임을 제한한다.
* TTL controller는 현재 Jobs에 대해서만 적용할 수 있지만 파드나 커스텀 리소스처럼 실행을 마치는 다른 리소스들에 대해서도 확장될 것이다. 

## TTL Controller

* TTL controller는 현재 Jobs에 대해서만 적용된다.
* 클러스터 오퍼레이터는 이 feature를 완료된 Jobs를(Complete거나 Failed) Job 필드의 `.spec.ttlSecondsAfterFinished`필드를 지정하여 자동으로 처리하는데 사용할 것이다.
* TTL controller는 리소스가 종료되면 clean up 될때까지 TTL seconds 즉, TTL이 만료되는 시간을 가질 수 있다고 가정한다. 
* TTL이 리소스를 clean up할 때, cascade로 삭제한다. 다시말해 dependents 오브젝트를 함께 지운다.
* 리소스가 삭제될 때 finalizer처럼 라이프사이클이 보장된다.
* TTL seconds는 언제든지 설정될 수 있다.
* Job의 `.spec.ttlSecondsAfterFinished`필드를 세팅하는 예시
  * resource manifest 필드를 지정하여 Job이 종료되고나서 약간 후에 자동으로 삭제되도록 한다.
  * 존재하지만 이미 끝난 리소스에 대해서 이 필드를 설정해서 새로운 feature를 줄 수 있다.
  * mutating admission webhook을 이용하여 이 필드를 리소스가 생성될 때 동적으로 설정한다. 클러스터 관리자는 이를 종료되는 리소스에 대해 TTL 정책을 강제로 설정할 때 사용한다.
  * mutating admission webhook을 이용하여 리소스가 종료된 후에 이 필드를 동적으로 설정하고 리소스 상태, 라벨등에 따라서 TTL 값을 다르게 할 수 있다.

## Caveat

#### Updating TTL Seconds

* Jobs에서 `.spec.ttlSeondsAfterFinished`필드같은 TTL period가 리소스가 생성되거나 종료된 이후에 수정될 수 있음을 주의하라.
  * 하지만 한번 Job이 TTL이 만료되어 삭제되기로 되어있었다면 TTL을 연정하는 것이 API 응답이 성공이라고 나왔음에도 불구하고 시스템은 Job이 유지될 것이라는 보장을 하지않는다.

### Time Skew

* TTL controller는 TTL이 만료되었는지 아닌지를 판별하기 위해 Kubernetes resource에 있는 timestamp를 사용하기 때문에 이 feature은 클러스터의 time skew에 민감하다.
  * 이는 TTL controller가 잘못된 시간에 리소스 오브젝트를 삭제할 수 있다는 것을 의미한다.
* 쿠버네티스에서 time skew를 막기 위해서 NTP를 모든 노드에서 동작시키도록 요구된다. 시간은 항상 맞지는 않지만 차이가 매우 작다
* 이를 0이 아닌 TTL을 줄 때 주의해야 한다.