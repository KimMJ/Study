# Jobs - Run to Completion

* Job은 하나 이상의 파드를 생성하여 지정된 숫자만큼 성공적으로 완료되도록 보장한다.
  * 파드가 성공적으로 완료되면 Job은 이를 기록한다.
  * 지정된 수만큼 성공적으로 완료될 경우, Job같은 task는 완료된다.
  * Job을 지우는 것은 생성된 파드를 clean up 한다.
* 하나의 파드가 성공적으로 완료되는 것을 보장하기 위해서 하나의 Job오브젝트를 만드는 것이 가장 간단한 케이스이다.
  * 첫 파드가 실패하거나 삭제될 경우(하드웨어의 failure 또는 노드의 재부팅) 새로운 파드를 실행한다.
* Job을 통해 다수의 파드를 병렬로 실행할 수 있다.

## Running an example Job

* Job의 성공된 파드를 보기 위해서는 `kubectl get pods`를 사용하라.

* 파드의 standard output을 보려면 다음을 써라.

  ```bash
  kubectl logs $pods
  ```

## Writing a Job Spec

* 다른 쿠버네티스 config처럼 Job은 `apiVersion`, `kind`, `metadata`필드가 필요하다.
* 또한 `.spec` 섹션도 있어야 한다.

### Pod Template

* `.spec`에서 `.spec.template`필드만 필수 필드이다.
* `.spec.tempalte`은 파드 템플릿이다. 이는 `apiVersion`이나 `kind`가 없고 nested가 아닌 것을 제외하면 파드와 스키마가 같아야 한다.
* 파드의 필수 필드와 더불어 Job에서 파드 템플릿은 반드시 적절한 라벨과 적절한 restart policy를 필요로 한다.
  * `RestartPolicy`는 `Never` 또는 `OnFailure`만 쓴 수 있다.

### Pod Selector

* `.spec.selector`필드는 옵셔널이다.  
* 대부분의 경우 특별히 지정하지 않는다.

### Parallel Jobs

* Job으로 동작하는 것이 좋은 몇가지 타입의 테스크가 있다.
  * Non-parallel Jobs
    * 파트가 실패해도 일반적으로 하나의 파드만 시작된다.
    * Job은 파드가 성공적으로 종료되면 즉시 완료된다.
  * Parallel Jobs with a fixed completion count
    * `.spec.completion`에 1 이상의 파라미터를 지정해 주어야 한다.
    * Job은 전체 테스크에 요청하고 1부터 `.spec.completions`의 value들 각각 하나의 successful Pod가 있으면 complete가 된다.
    * **not implemented yet** 각 파드는 1부터 `.spec.completions`에서 다른 index를 가진다.
  * Parallel Jobs with a work queue
    * `.spec.parallelism`으로 default값이 지정된 `.spec.completions`를 지정하지 말라.
    * 파드는 반드시 파드끼리 조직화하거나 외부 서비스로 어떻게 일을 할지 결정해야 한다. 예를 들어 파드는 work queue에서 N개 아이템까지 작업을 가저올 수 있다.
    * 각 파드는 개별적으로 peer가 끝이 났는지 안났는지 결정할 수 있어야한다. 이를 통해 전체 작업이 끝났는지 판단한다.
    * Job에서의 어떤 파던지 성공하면 새로운 파드는 생성되지 않는다.
    * 최소 하나의 파드가 성공적으로 끝나고 모든 파드가 종료되면 Job은 성공적으로 끝난 것이다.
    * 어떤 파드던지 성공으로 종료가 되고 나머지는  종료되지 않았다면 파드는 계속해서 일을 하고 어떤 output을 쓰고 있다. 모든 파드가 끝이 나야한다.
* non-parallel Job에 대해서 `.spec.completions`와  `.spec.paralleism`을 설정하지 않아도 된다.
  * 둘다 설정하지 않으면 둘다 default로 1이다.
* work queue Job에서 `.spec.completions`를 설정하면 안되고 `.spec.parallelism`은 음이 아닌 정수여야 한다.

### Controlling Parallelism

* parallelism에 대한 필수 조건(`.spec.parallelism`)은 음이 아닌 정수로 설정하는 것이다. 
  * 지정하지 않으면 default는 1이다.
  * 0으로 지정할 경우 Job은 이 수가 늘어날때까지 멈춰있는다.
* 실제 병렬처리는 요청한 병렬처리보다 다음과 같은 이유로 많거나 적을 수 있다.
  * 고정된 count를 가진 Job에서 병렬로 동작하는 실제 파드는 남은 completion보다 많을 수 없다. 이보다 더 큰 `.spec.parallelism`은 무시된다.
  * work queue Job에서 어떤 파드가 성공하고 나면 새로운 파드가 생성되지 않는다. 하지만 나머지 파드는 완료될수 있다.
  * 컨트롤러가 반응할 시간이 없었다.
  * 컨트롤러가 어떤 이유에선가 파드를 생성하는데 실패하여(`ResourceQuota`, 권한 부족 등) 요청한 것보다 더 적은 파드가 있다.
  * 컨트롤러가 같은 Job에서 과도한 파드의 failure 때문에 새로운 파드의 생성을 막는다.
  * 파드가 graceful하게 종료되어 멈추는데 시간이 걸린다.

## Handling Pod and Container Failures

*  파드의 컨테이너는 프로세스가 0이 아닌 종료 코드로 종료되었거나 컨테이너가 메모리 초과되어 죽는 등의 많은 이유로 실패할 수 있다.
  * 이 경우에 `.spec.template.spec.restartPolicy = "OnFailure"`인 상태이면 파드는 노드에 계속있지만 컨테이너는 재실행된다.
  * 따라서 직접 이 케이스를 처리하거나 `.spec.template.restartPolicy = "Never"`로 설정해야 한다.
  * `restartPolicy`에 대한 lifecycle 참조
* 전체 파드는 또한 파드가 노드에서 방출되거나 파드의 컨테이너가 실패하고, `spec.template.spec.restartPolicy = "Never"`과 같은 많은 이유로 실패할 수 있다.
  * 파드가 실패하면 파드 컨트롤러는 새로운 파드를 시작한다.
  * 어플리케이션은 새로운 파드로 재시작하는 경우에 대해 처리해야 한다.
  * 이전의 실행으로 인한 임시 파일, 락, 비정상 출력등을 처리해야 한다.
* `.spec.parallelism = 1`, `.spec.completions = 1`, `.spec.template.spec.restartPolicy = "Never"`로 지정했다 하더라도 같은 프로그램이 가끔 두번 실행될 수 있다.
* `.spec.parallelism`과 `.spec.completions`가 둘다 1보다 크게 지정했으면 여러 파드가 동시에 동작할 것이다.
  * 따라서 concurrency에 대해 처리해야한다.

### Pod backoff failure policy

* configuration의 논리적인 에러때문에 몇번의 재시도 후에는 Job을 fail로 하고 싶은 상황이 있다.
  * `.spec.backoffLimit`을 지정하여 재시도의 수를 정한다.
  * back-off 제한은 기본 6으로 지정되어 있다.
* Job과 관련된 실패된 파드는 Job controller에 의해 exponential back-off delay로(10s, 20s, 30s.....) 재생성된다.
* Job이 다음 status를 체크하기까지 실패한 파드가 없을 경우 back-off count를 초기화한다.

> Note : `restartPolicy = "OnFailure"`인 job을 가지고 있다면 컨테이너가 잡을 실행한 것은 backoff limit에 다다른 것임을 기억해라. 이는 Job의 실행을 디버깅하기가 더 어렵도록 만든다. `restartPolicy = "Never"`를 이용해서 Job이나 logging system으로 Job의 출력이 부주의로 사라지지 않게 해야한다.

## Job Termination and Cleanup

* Job이 완료되면 파드는 더이상 생성되지 않지만 파드가 지워지지도 않는다.
  * 완료된 파드에 대해서 에러나 경고, 다른 진단할 수 있는 로그를 볼 수 있도록 해준다.
  * job 오브젝트 또한 완료된 후에도 남아있어서 상태를 볼 수 있다.
  * 상태를 보고 나서 job을 지울지 말지는 사용자가 결정한다.
  * `kubectl`을 통해 job을 지운다. (`kubectl delete jobs/pi` 또는 `kubectl delete -f ./job.yaml`.)
  * `kubectl`을 통해 job을 지우면 그에 해당하는 파드 또한 지워진다.
* 기본적으로 Job은 파드가 실패하거나(`restartPolicy=Never`) 컨테이너가 에러(`restartPolicy=OnFailure`)가 아니면 위에서 설명한 `.spec.backoffLimit`의 관점에서 interrupted 되지 않는다. `.spec.backoffLimit`이 되면 Job은 실패로 마킹되고 동작중이던 파드는 종료된다.
* Job을 끝내는 다른 방법은 active deadline을 설정하는 것이다.
  * `.spec.activeDeadlineSeconds`필드로 이를 설정할 수 있다.
  * `activeDeadlineSeconds`는 얼마나 많은 파드를 생성했는지에 상관 없이 job의 지속시간을 적용한다.
  * `activeDeadlineSeconds`에 도달하면 모든 진행중인 Pod는 종료되고 Job의 상태는 `type: Failed`, `reason: DeadlineExceeded`로 된다.
* Job의 `.spec.activeDeadlineSeconds`는 `.spec.backoffLimit`보다 우선순위이다.
  * 한번 이상 재시도하거나 실패한 파드는 `activeDeadlineSeconds`에 지정된 시간 제한을 넘으면 `backoffLimit`에 도달하지 않았더라도 새로운 파드를 만들지 않을 것이다.

## Clean Up Finished Jobs Automatically

* 완료된 Job은 더이상 시스템에서 필요하지 않다. 이를 시스템에 두는 것은 API 서버에 부담을 줄 수 있다.
* Job이 CronJob같은 더 high level의 컨트롤러에 의해서 직접 관리가 되면, Job은 지정된 capacity-based cleanup 정책에 따라서 clean up 된다.

### TTL Mechanism for Finished Jobs

**FEATURE STATE: `Kubernetes v1.12` - alpha**

*  성공했든 실패했든 완료된 Job을 자동으로 clean up하는 다른 방법은 TTL controller가 Job의 `.spec.ttlSecondsAfterFinished`필드를 지정하여 리소스를 끝내는데 사용하는 TTL 메카니즘을 이용하는 것이다.
* TTL 컨트롤러가 Job을 clean up할 때, cascade하게 Job을 삭제한다.
  * 파드와 같은 dependent 오브젝트를 Job과 함께 지운다.
  * Job이 삭제될 때 finalizer같은 lifecycle이 보장된다.

## Job patterns

* Job 오브젝트는 파드를 병렬로 실행하는 것을 지원하기 위해 사용된다.
  * Job은 보통의 scientific computing에서처럼 closely-communicating 병렬 처리를 지원하기로 디자인되지 않았다.
  * 분리되었지만 관련된 작업에 대한 병렬 처리를 지원한다.
* 병렬 연산에는 여러 패턴이 있고 각각 장단점이 있다.
  * 각각의 작업에 대해 하나의 Job 오브젝트를 두기 / 모든 작업에 대해 하나의 Job 오브젝트 두기
    * 후자는 작업의 수가 많을 때 좋다.
    * 전자는 사용자와 시스템에 많은 수의 Job 오브젝트 관리에 대한 오버헤드를 준다.
  * 파드의 개수가 작업의 수와 같게 하기 / 각 파드가 여러 작업을 수행하게 하기
    * 전자는 현재 코드와 컨테이너의 약간의 수정만 필요하다.
    * 후자는 작업의 수가 많을 때 좋으며 이는 이전의 이유와 같다.
  * work queue를 이용하는 여러 방법들.
    * queue 서비스를 실행시키고 현재 프로그램 또는 컨테이너를 work queue를 이용하도록 수정해야한다.
    * 다른 방법은 현재 컨테이너화된 어플리케이션에 적용하기 쉽다.

|                                                              |                   |                             |                     |                    |
| :----------------------------------------------------------- | :---------------- | :-------------------------- | :------------------ | :----------------- |
| Pattern                                                      | Single Job object | Fewer pods than work items? | Use app unmodified? | Works in Kube 1.1? |
| [Job Template Expansion](https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/) |                   |                             | ✓                   | ✓                  |
| [Queue with Pod Per Work Item](https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/) | ✓                 |                             | sometimes           | ✓                  |
| [Queue with Variable Pod Count](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/) | ✓                 | ✓                           |                     | ✓                  |
| Single Job with Static Work Assignment                       | ✓                 |                             | ✓                   |                    |

* `.spec.completions`로 completion을 지정하면 각 파드는 Job 컨트롤러에 의해 고유 spec을 가지게 된다.
  * task를 위한 모든 파드는 같은 command line, 같은 이미지, 같은 볼륨, 거의 같은 환경변수를 가진다는 것을 의미한다.
  * 이 패턴은 파드가 각기 다른것에 대해 수행하도록 하는것과는 다른 방법이다.

| Pattern                                                      | `.spec.completions` | `.spec.parallelism` |
| :----------------------------------------------------------- | :------------------ | :------------------ |
| [Job Template Expansion](https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/) | 1                   | should be 1         |
| [Queue with Pod Per Work Item](https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/) | #Work items         | any                 |
| [Queue with Variable Pod Count](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/) | 1                   | any                 |
| Single Job with Static Work Assignment                       | #Work items         | any                 |

## Advanced Usage

### Specifying your own pod selector

* 보통 Job 오브젝트를 생성하면 `.spec.selector`를 지정하지 않는다.
  * 시스템이 Job이 생성될때 로직으로 채워넣는다.
  * 다른 job과 겹치지 않는 selector를 선택한다.
* 몇몇 경우에서 이 자동으로 생성되는 selector를 덮어써야할 수도 있다.
  * 이는 Job의 `.spec.selector`에서 지정할 수 있따.
* 이는 주의가 필요하다. Job의 파드에서 유일하지 않은 label selector를 지정하면, 그리고 관계 없는 파드가 매칭이 되면 관계 없는 파드는 지워지거나 Job이 다른 파드를 completing하다고 생각하거나 하나 또는 두개의 Job이 모두 파드의 생성이나 실행을 거부할 수 있다.
  * 만일 유일하지 않은 selector가 선택이 되면 다른 컨트롤러(ReplicationController)와 그 파드들은 예상하지 못하는 행동을 한다. 쿠버네티스는 `.spec.selector`를 지정할 때 실수를 방지해주지 않는다.

## Alternatives

### Bare Pods

* 파드가 동작하고 있는 노드가 리부팅되거나 실패할 때, 파드는 종료되고 재시작되지 않는다.
* Job은 새로운 파드를 생성하여 종료된 파드를 대체한다.
* 따라서 bare Pod보다는 Job을 권장한다.

### Replication Controller

* Job은 Replication Controller와 상호 보완적이다.
  * Replication Controller는 종료되기로 예상되지 않은 파드에 대해 관리한다
  * Job은 종료하기로 예상되는 파드에 대해 관리한다.
* Pod Lifecycle에서 보았듯이 `Job`은 `RestartPolicy`가 `OnFailure` 또는 `Never`일때 적절하다. (`RestartPolicy`가 설정되지 않으면 기본적으로 Always이다.)

### Single Job starts Controller Pod

* 파드를 생성하기 위한 하나의 Job에 대한 다른 패턴은 생성하고자 하는 파드에 대한 일종의 커스텀 컨트롤러와 같이 행동하는 파드를 생성하는 것이다.
  * 더 유연하지만 시작하기에 어렵고 쿠버네티스와 잘 통합되지는 않는다.
* 이 방법의 이점은 전체 프로세스는 Job 오브젝트에 의해 성공함이 보장이 되지만 어떤 파드가 생성되고 어떻게 동작하는지에 관한 복잡한 컨트롤은 해당 파드에 넘겨주는 것이다.

## Cron Jobs

* `CronJob`을 생성하여 특정한 시간/날짜에 Job을 생성하도록 할 수 있다.
  * Unix의 `cron`과 비슷