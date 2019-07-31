# CronJobs

* CronJobs는 시간 기반 일정에 따라 Job을 생성한다.
* 하나의 CronJobs 오브젝트는 crontab(크론 테이블)파일의 한 줄과 같다.
  * 이는 주어진 일정에 따라 Cron format으로 쓰인 작업을 실행한다.

> 모든 CronJob은 `schedule : ` 시간은 마스터가 처음 Job이 시작된 마스터의 시간을 기준으로 한다.

## Cron Job Limitations

* cron job은 일정 실행 시간마다 약 한번의 Job을 생성한다.

  * "약"이라고 하는 이유는 특정 환경에서 두 개의 Job이 만들어지거나 Job이 생성되지 않기도 하기 때문이다.
  * 이를 막아야 하지만 완벽하지는 못하다.
  * Job은 멱등원이어야 한다. => 공식 문서 한글판 수정필요?

* `startingDeadlineSeconds`가 큰 값으로 설정되거나 설정되지 않아 default를 사용하고  `concurrencyPolicy`가 `Allow`로 설정될 경우, Job은 적어도 한번은 실행된다.

* 모든 CronJob에 대해 Cronjob controller는 마지막 스케쥴된 시간부터 지금까지 얼마나 많은 스케쥴을 놓쳤는지 체크한다.

  * 만약 100회 이상 놓치면 job을 더 이상 실행하지 않고 에러 로그를 남긴다.

    ```
    Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
    ```

* 중요한 점은 `startingDeadlineSeconds` 필드가 설정되면 (`nil`이 아닌 값) 컨트롤러는 마지막 스케쥴부터 지금까지 대신 `startingDeadlineSeconds`값에서 얼마나 많은 Job이 누락되었는지 센다.

  * `startingDeadlineSeconds`가 `200`이면 컨트롤러는 최근 200초 내 몇 개의 Job이 누락되었는지 센다.

* CronJob은 정해진 스케쥴에 Job 실행이 실패하면 놓쳤다고 센다.

  * `concurrencyPolicy`가 `Forbid`로 설정되었고, CronJob이 스케쥴을 시작하려는데 이전 스케쥴이 아직 동작 중이라면 놓친것으로 센다

* 예를 들어 CronJob이 `08:30:00`에 시작하여 매 분 Job을 실행하도록 설정되어 있고, `startingDeadlineSeconds`가 설정되어 있지 않은 상태에서 CronJob controller가 `08:29:00` 부터 `10:21:00`까지 고장나면 놓친 일정이 100개를 초과하여 Job이 실행되지 않는다.

  * 여기서 `startingDeadlineSeconds`가 200이라고 하면 Job은 고장이 끝난 10:22:00에 다시 시작된다. 최근 200초에서 얼마나 놓쳤는지를 확인하고, 3개를 놓쳤다고 체크하기 때문이다.

* CronJob은 스케쥴에 따른 Job 생성에만 책임이 있고 Job은 해당 Pod의 관리에 대한 책임이 있다.