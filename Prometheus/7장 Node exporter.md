# 7장 Node exporter

* node exporter는 CPU, memory, Disk size, Disk I/O, 네트워크 대역, 마더보드 온도 등 주로 운영 시스템 커널에서 오는 머신 수준의 메트릭을 게시함.
* 머신 자체를 모니터링하기 위한 것으로 개별 프로세스나 서비스를 모니터링 하기 위한 것은 아님.
* 프로메테우스 아키텍처에서 각 서비스는 exporter를 사용해 자체 메트릭을 expose함.
  * 필요한 경우 프로메테우스가 직접 데이터를 수집하기도 함.
* 노드 익스포터는 nonroot user가 실행할 수 있도록 설계되었으며 머신에서 직접 실행해야 함.
* 노드 익스포터는 도커에서 동작시키는 것을 권장하지 않음.
* 운영체제에서 사용할 수 있는 많은 메트릭을 갖고 있는 노드 익스포터는 가져오는 메트릭의 카테로리를 구성하는 것이 가능하다.

## CPU 수집기

* `node_cpu_seconds_total` 은 각 모드에서 CPU마다 얼마나 많은 시간을 소비했는지 나타내는 카운터이다.
  * label은 `cpu` 와 `mode` 이다.
* `avg without(cpu, mode)(rate(node_cpu_seconds_total{mode="idle"}[1m]))`
  * 이 표현식은 CPU마다 초당 유휴시간을 계산한 다음 머신에서 모든 CPU의 평균을 계산한다.
* `avg without(cpu)(rate(node_cpu_seconds_total[1m]))`
  * 하나의 머신에 대해 각 모드에서 사용한 시간 비율에 대한 계산

## filesystem 수집기

* `df` 명령어에서 얻을 수 있는 것처럼 마운트된 파일 시스템의 메트릭을 수집함.

* 이 수집기의 모든 메트릭에는 `node_filesystem_` prefix가 붙어있음.

* `device`, `fstype`, `mountpoint` label이 있음.

* `node_filesystem_avail_bytes` 는 일반 사용자들이 이용할 수 있는 공간이고 사용된 디스크 공간을 계산하려면 다음과 같이 작성한다.

  ```
  node_filesystem_avail_bytes
  /
  node_filesystem_size_bytes
  ```

## diskstats 수집기

* `/proc/diskstats` 에서 얻은 디스크 I/O 메트릭을 게시함.

* 모든 메트릭은 `device` label을 가지고 있음.

* 디스크 I/O 사용률을 계산하려면 `iostat -x` 에 의해 표시된 것처럼 `node_disk_io_time_seconds_total` 을 사용할 수 있다.
  `rate(node_disk_to_time_seconds_total[1m])`

* 다음과 같이 읽기 I/O의 평균 시간을 계산할 수 있다.

  ```
  rate(node_disk_read_time_seconds_total[1m])
  /
  rate(node_disk_read_completed_total[1m])
  ```

## netdev 수집기

* `node_network_` prefix와 `device` label을 사용해 네트워크 디바이스에 대한 메트릭을 게시함.

## meminfo 수집기

* `node_memory_` prefix를 갖는 모든 메모리 관련 표준 메트릭을 갖고 있음.
* `/proc/meminfo` 에서 메트릭을 얻음.

## hwmon 수집기

* baremetal에서 hwmon 수집기는 온도, 팬 속도 등 `node_hwmon_` prefix를 갖는 메트릭을 제공함.
* `sensors` 명령으로 얻을 수 있는 내용과 동일한 정보.

## stat 수집기

* `/proc/stat` 에서 메트릭을 제공하기 때문에 다양한 성격의 메트릭이 혼재되어 있음.

## uname 수집기

* `node_uname_info` 를 expose함.

## loadavg 수집기

* 1분, 5분, 15분 부하 평균을 의미하는 `node_load1`, `node_load5`, `node_load15` 메트릭을 제공

## textfile 수집기

* textfile 수집기는 앞서 설명한 수집기들과는 다르게 커널에서 메트릭을 얻어오지 않고 우리가 만든 파일에서 가져옴.
* `smartctl` 명령어를 실행하기 위해서는 루트 사용자 권한이 필요함.
* `iptables` 명령어를 실행하면 일부 정보만 얻을 수 있음.
* `smartctl`, `iptables` 를 정기적으로 실행하는 cronjob을 생성하고 그 결과를 프로메테우스 표현 형식으로 변환해야함.
  * 그리고 특정 디렉토리의 파일에 atomically 기록해야함.

### textfile 수집기 사용

* 노드 익스포터에서 동작시키려면 `--collector.textfile.directory` 명령행 플래그를 사용해야함.
* textfile 수집기는 `.prom` 확장자를 갖는 파일만 찾음.
* `.prom` 파일은 cronjob과 함께 생성되고 업데이트됨.

### timestamp

* exposition format은 timestamp를 지원하지만 textfile 수집기와 함께 사용할 수는 없음.
* 대신 `mtime` 을 `node_textfile_mtime_seconds` 에서 사용할 수 있음

