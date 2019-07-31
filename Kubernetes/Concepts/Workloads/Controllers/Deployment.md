# Deployment



## Updating a Deployment

### Rollover (aka multiple updates in-flight)

* Deployment controller가 새로운 Deployment를 발견하면, `ReplicaSet`이 생성되어 파드를 원하는 상태로 만든다.
* Deployment가 업데이트 되면 파드 중 `.spec.selector`라벨이 일치하지만 template이 `.spec.template`과 일치하지 않는 파드를 컨트롤하는 `ReplicaSet`을 scale down한다.
  * 결과적으로 새로운 `ReplicaSet`은 `.spec.replicas`만큼 scale되고 이전의 `ReplicaSet`은 0으로 scale된다.
* 만약 rollout이 진행중일 때 Deployment를 업데이트 하면, Deployment는 새로운 ReplicaSet을 만들고 그것을 scaling up하기 시작한다. 이전에 scaling up 되고 있던 ReplicaSet은 roll over된다. (old ReplicaSet 리스트에 추가되고 scaling down되기 시작한다.)

### Label selector updates

* label selector를 업데이트 하는 것은 권장되지 않는다. 사전에 selector에 대한 계획을 해야한다.
* label selector를 업데이트 하는 어떤 경우에라도, 충분한 주의와 모든 영향에 대해서 고려해야한다.
* API version apps/v1에서, Deployment의 label selector는 생성되고 나서는 변경될 수 없다.
* selector를 추가하는 것은 Deployment spec의 Pod template label을 새 label로 변경하는 것이 필요하다. 그렇지 않으면 validation error가 리턴된다. 이 변경은 non-overlapping으로 새로운 selector는 이전의 selector가 생성한 ReplicaSet과 Pod를 선택하지 않는다는 것을 의미하고, 예전 ReplicaSet의 자식들을 제거하고 새로운 ReplicaSet을 생성해야 한다는 것을 의미한다.
* Selector는 selector key에 있는 값의 변경을 업데이트 한다. - 추가할때와 같은 결과
* Selector를 지우는 것은 Deployment selctor의 키를 지운다. - Pod template label의 어떤 변경도 필요 없다. 존재하던 ReplicaSet은 고아가 되지 않으며, 새로운 ReplicaSet이 생성되지 않지만, 제거된 label이 이미 존재하는 Pod와 ReplicaSet에 존재함을 주의하라.

## Rolling Back a Deployment

* 기본적으로 모든 Deployment의 rollout 히스토리는 시스템에 저장되어 있고 원하는 언제든 rollback할 수 있다. (revision history limit을 설정할 수 있다.)

> Note : Deployment의 revision은 Deployment rollout이 트리거 될 때 생성된다. 즉, template의 label이나 container image같은 Deployment의 template (`.spec.template`)이 변경 될 때만 새로운 revision이 생성이 된다. Deployment의 scaling같은 업데이트는 Deployment revision을 생성하지 않아 manual 또는 auto scaling을 동시에 가능하게 한다. 즉, 이전 revision으로 roll back할 경우 Deployment의 파드 template파트만 roll back 된다는 것을 의미한다.

### Checking Rollout History of a Deployment

* 이 Deployment의 Revision을 확인한다.
  ```bash
  kubectl rollout history deployment.v1.apps/nginx-deployment
  ```
  
* `CHANGE-CAUSE`은 Deployment annotation `kubernetes.io/change-cause`을 revision에 복사한 것이다. `CHANGE-CAUSE`을 다음과 같은 방법으로 정할 수 있다.

  * Annotating the Deployment

    ```
    kubectl annotate deployment.v1.apps/nginx-deployment kubernetes.io/change-cause="image updated to 1.9.1"
    ```

  * resource가 수정이 될 때 kubectl 명령어에 `--record`플래그를 붙여 저장하기

  * resource의 manifest를 직접 수정하기

* 각 revision에 대한 자세한 사항 확인

  ```
  kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2
  ```



### Rolling Back to a Previous Revision

* 이전 버전으로 undo하는 방법

  ```
  kubectl rollout undo deployment.v1.apps/nginx-deployment
  ```

  또는 `--to-revision`플래그를 통해 revision을 정해서 rollback할 수도 있다.

  ```
  kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
  ```

* rollback이 성공했는지 확인하고 Deployment가 예상대로 동작하는지 보려면:

  ```
  kubectl get deployment nginx-deployment
  ```

* Deployment의 description을 보려면

  ```
  kubectl describe deployment nginx-deployment
  ```

## Scaling a Deployment

*  다음 커맨드로 Deployment를 scale할 수 있다.

  ```
  kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
  ```

* horizontal Pod autoscaling이 cluster에서 가능하다면, Deployment에 autoscaler를 설정할 수 있고 동작중인 파드의 CPU utilization에 따른 minimum, maximum 파드 수를 설정할 수 있다.

  ```
  kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
  ```

### Proportional scaling

* RollingUpdate Deployment는 어플리케이션을 동시에 여러 버전으로 동작시키는 것을 지원한다.

* 사용자나 autoscaler가 rollout 중간에 RollingUpdate Deployment를 scale할 경우 Deployment controller는 현재 활성화된 ReplicaSet에서의 추가적인 replica를 조절하여 위험을 줄인다. 이를 proportional scaling이라고 한다.

* 예를 들어 10개의 replica, maxSurge=3, maxUnavailable=2인 Deployment를 동작한다 하자.

  * 10개의 replica가 잘 동작함을 확인한다.

  * 클러스터 내에서 unresolvable이 발생하도록 이미지를 업데이트 한다.

  * 이미지 업데이트는 ReplicaSet으로 rollout을 시작하지만 maxUnavailable에 막힌다. rollout status를 확인하라.

    ```
    kubectl get rs
    ```

  * 그후 Deployment에 대한 scaling 요청이 들어온다. autoscaler는 Deployment replica를 15로 증가시킨다. Deployment controller는 새로운 5 개의 replica를 더할 곳을 결정해야한다. proportional scaling을 하지 않을 경우, 5개의 파드는 모두 새로운 ReplicaSet에 더해질 것이다. proportional scaling이라면 추가적인 replica를 모든 ReplicaSet에 퍼뜨린다. 높은 비율로 많은 replica가 있는 ReplicaSet으로 할당이 되고 낮은 비율로 적은 replica가 있는 ReplicaSet에 할당이 된다. 나머지는 많은 replica가 있는 곳에 할당이 된다. ReplicaSet중 replica가 0인 것은 scale up 되지 않는다.

  * 

## Pausing and Resuming a Deployment

* Deployment를 하나가 트리거 되기 전 또는 나머지가 업데이트 되기 전에 멈출 수 있고 다시 재개할 수 있다.
* 이는 불필요한 rollout을 트리거하지 않고 여러 변경점을 적용시킬 수 있다.

> Note : 멈춰진 Deployment를 재개할 때까지 rollback할 수 없다.

## Deployment status

* lifecycle 동안 다양한 state를 가질 수 있따. ReplicaSet을 rollout하는 동안 progressing이 될수도, complete이 될수도, fail to progress가 될 수도 있다.

### Progressing Deployment

* 다음같은 상황에서 Deployment는 progressing으로 마킹된다.
  * Deployment가 새로운 ReplicaSet을 생성할 때
  * Deployment가 최신 ReplicaSet을 scaling up할 때
  * Deployment가 이전의 ReplicaSet을 scaling down할 때
  * 새로운 파드가 ready 또는 available이 되었을 때. (최소한 MinReadySeconds만큼 기다린다.)
* `kubectl rollout status`로 Deployment의 progress를 모니터 할 수 있다.

### Complete Deployment

* 다음같은 상황에서 Deployment는 complete로 마킹된다.
  * Deployment와 관련된 모든 replica가 최신 버전으로 업데이트가 되어 모든 업데이트 요청이 완료되었다.
  * Deployment와 관련된 모든 replica가 available이다.
  * Deployment의 이전 replica가 동작중이다.

### Failed Deployment

* 최신 ReplicaSet을 배포하는데 Deployment가 멈출 수 있다. 이는 다음과 같은 이유가 있다.

  * quota 부족
  * Readiness probe 실패
  * image pull 에러
  * 권한 부족
  * range 제한
  * application의 runtime misconfiguration

* 이 상태를 확인할 수 있는 방법은 Deployment spec(`spec.progressDeadlineSeconds`)에 있는 deadline parameter를 수정하는 것이다.

* `.spec.progressDeadlineSeconds`는 Deployment controller가 Deployment progress가 정지되었는지 Deployment status에서 확인하기까지 기다리는 시간이다.

* 다음 `kubectl`은 `progressDeadlineSeconds`를 controller가 10분단위로 체크하도록 하는 명령어이다.

  ```shell
  kubectl patch deployment.v1.apps/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
  ```

* Deadline을 초과하고 나면, Deployment controller는 DeploymentCondition의 `.status.conditions` 속성을 다음으로 변경한다.

  * Type=Progressing
  * Status=False
  * Reason=ProgressDealineExceeded

### Operating on a failed deployment

* complete Deployment에 할 수 있는 모든 동작이 failed Deployment에서도 할 수 있다. scale up/down, roll back, pause가 가능하다.

## Clean up Policy

* Deployment의 `rivisionHistoryLimit`필드를 통해서 얼마나 많은 해당 Deployment에 대해 얼마나 많은 old ReplicaSet을 유지할 지 결정할 수 있다. 나머지는 garbage-collected가 될 것이다. deafult는 10

## Canary Deployment

* 서버나 유저의 subset에 releases를 roll out하고 싶으면 release마다 하나의 Deployment를 생성하여 canary pattern을 통해서 roll out 할 수 있다.

## Writing a Deployment Spec

* 다른 kubernetes config에서처럼 Deployment는 `apiVersion`, `kind`, `metadata` 필드가 필요하다. 또한 `.spec`섹션도 필요하다.

### Pod Template

* `.spec.template`과 `.spec.selector`만 `.spec`에서 쓰인다.
* `.spec.template`은 Pod template이다. 이는 `apiVersion`, `kind`가 없고 nested인 점을 빼면 Pod의 schema와 일치한다.
* 이외의 Pod의 required field에서 Pod template은 반드시 적절한 라벨과 restart policy를 명시해야 한다. label에 대해서는 다른 controller와 일치하지 않음이 보장되어야 한다.
* `.spec.template.spec.restartPolicy`에서 `Always`만 가능고, 디폴트이다.

### Replicas

* `.spec.replicas`는 optional field이고 원하는 파드의 수를 기록한다. 
* 디폴트는 1 

### Selector

* `.spec.selector`은 Deployment가 지정하는 파드의 label selector를 명시하는 required field이다.
* `.spec.selector`는 반드시 `.spec.template.metadata.labels`와 일치해야하고 아니면 API에 의해 거부된다.
* API 버전 `apps/v1`에서는 `.spec.selector`와 `.metadata.labels`가 `.spec.template.metadata.labels`의 default가 아니다. 따라서 반드시 명확히 작성해야한다. `.spec.selector`는 `apps/v1`에서 생성되고 난 뒤 변경될 수 없다.
* Deployment는 라벨이 selector와 일치하고 그 template은 `.spec.template`과 다르거나 `.spec.replicas`를 넘는 Pod 개수가 있다면 파드를 종료시킬 것이다. 만약 원하는 숫자보다 적으면 `.spec.template`을 통해 새로운 파드를 생성할 것이다.
* 만약 여러 controller가 selector가 겹치게 될 경우 controller는 각각 모두 동작을 하고 정상 작동을 하지 않을 것이다.

### Strategy

* `.spec.strategy`는 strategy를 명시하여 오래된 파드를 새것으로 고친다.
* `.spec.strategy.type`은 `Recreate`, `RollingUpdate`가 될 수 있다.
* `RollingUpdate`는 디폴트다.

#### Recreate Deployment

* 새로 파드를 생성하기 전에 모든 존재하는 파드를 죽인다.

* ```
  .spec.strategy.type==Recreate
  ```

#### Rolling Update Deployment

* `.spec.strategy.type==RollingUpdate`일 때 Deployment는 파드를 rolling update방식으로 업데이트 한다.
* rolling update 프로세스를 관리하기 위해 `maxUnavailable`과 `maxSurge`를 명시해야한다.

##### Max Unavailable

* `.spec.strategy.rollingUpdate.maxUnavailable`은 update process가 진행되는 동안 unavailable상태로 할 수 있는 파드의 최대 개수를 적은 optional field이다. 
* 상수 또는 비율로 정할 수 있다. 퍼센트는 rounding down을 통해 절대값으로 계산된다.
* `.spec.strategy.rollingUpdate.maxSurge`가 0이면 `maxUnavailable`도 0이 될 수 없다.
* 디폴트는 25%이다.

##### Max Surge

* `.spec.strategy.rollingUpdate.maxSurge`는 원하는 파드의 개수보다 더 생성할 수 있는 파드의 최대 개수를 적은 것이다.
* 상수 또는 비율로 정할 수 있다. `MaxUnavailable`이 0이면 이 값도 0이 될 수 없다. 퍼센트는 rounding up으로 절대값으로 계산된다.
* 디폴트는 25%이다.

### Progress Deadline Seconds

* `.spec.progressDeadlineSeconds`는 Deployment가 시스템이 Deployment가 progressing에 실패했다고 알려주기 전에 progress를 기다리는 시간이다. 
  * `Type=Progressing`, `Status=False`, `Reason=ProgressDeadlineExceeded`로 나타냄.
* Deployment Controller는 계속 Deployment를 재시도 할 것이다.
* 나중에는 자동 rollback이 구현될 것이고 deployment controller는 이런 상황이 되었을 때 바로 roll back 할 것이다.
* 반드시 `.spec.minReadySeconds`보다는 커야한다.

### Min Ready Seconds

* `.spec.minReadySeconds`는 optional field로 새로 생성된 파드가 available이 되기 위해서 컨테이너 장애 없이 준비가 되는 최소의 시간이다.
* default는 0이다.
  * 파드는 준비가 되면 바로 available이다.

### Rollback To

* `.spec.rollbackTo`는 `extensions/v1beta1`과 `apps/v1beta1`에서 deprecated 되었다.
* `apps/v1beta2`에서는 지원하지 않는다.
* 대신 `kubectl rollout undo`를 사용해라.

### Revision History Limit

* Deployment의 revision history는 ReplicaSet에 저장된다.
* `.spec.revisionHistoryLimit`은 optional field로 rollback을 하기 위해 저장하는 이전의 ReplicaSet의 개수를 의미한다.
* 이 old ReplicaSet은 `etcd`에서 resource로 consume되고 `kubectl get rs`를 채운다.
* 각 Deployment revision의 configuration은 ReplicaSet에 저장되어 있다.
  * old ReplicaSet이 삭제가 되면 해당 Deployment revision으로 roll back할 수 없다.
* 디폴트는 10이지만 이상적인 값은 얼마나 자주, 안정성이 있는지 등에 따라서 달라진다.
* 특별하게 이 필드를 0으로 설정할 경우 replica가 0인 old ReplicaSet들은 지워진다. 이 경우 새로운 Deployment의 rollout은 되돌릴 수 없다.

### Pause

* `.spec.paused`는 optional boolean field로 Deployment를 pausing하고 resuming한다.
* pause된 Deployment와 pause되지 않은 Deployment의 차이점은 pause된 Deployment에서는 PodTemplateSpec의 변경이 pause가 되어있는 동안 새로운 rollout을 트리거하지 않는다는 것이다.
* Deployment는 생성될 때 default로 멈춰지지 않는다.

## Alternative to Deployments

### kubectl rolling update

* `kubectl rolling update`는 Pod와 ReplicationControllers를 비슷한 방식으로 update한다.
* 하지만 선언적이고, server side이고, rolling update가 끝나고 난 뒤 이전 버전으로 roll back하는 등의 추가적인 기능이 있기 때문에 Deployment가 추천된다.