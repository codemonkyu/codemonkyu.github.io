---
title: "[Linux] Disk  I/O 스케줄러 (I/O Scheduler) 알고 있어?"
date: 2024-03-31 19:48:39 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["bfq", "Deadline", "I/O 스케줄러", "Linux"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-Disk-IO-%EC%8A%A4%EC%BC%80%EC%A4%84%EB%9F%AC-IO-Scheduler-%EC%95%8C%EA%B3%A0-%EC%9E%88%EC%96%B4
---

![I/O Scheduler](/assets/img/posts/linux-io-scheduler/01.png)
_I/O Scheduler_

Linux 운영체제를 사용하다 보면 I/O 스케줄러를 다룰 일이 발생한다.

I/O 스케줄러는 Disk Device에 대한 I/O 요청을 정렬 및 병합하며 처리 순서를 결정한다.

Linux OS의 배포판 별로 지원하는 I/O 스케줄러의 종류는 다양하며, 오늘은 AWS의 Amazon Linux2에서 지원하는 아래의 3개의 I/O 스케줄러 유형에 대하여 알아보려고 한다. Amazon Linux 2는 RHEL 에서 지원하는 [I/O 스케줄러](https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/setting-the-disk-scheduler_monitoring-and-managing-system-status-and-performance)를 사용하고 있는것을 확인하였다.

------

![출처 : RHEL8 공식 사이트](/assets/img/posts/linux-io-scheduler/02.png)
_출처 : RHEL8 공식 사이트_

------

각각의 I/O 스케줄러는 I/O 요청을 처리하는 방식이 다르며 장착된 물리적 디스크의 종류 (HDD 또는 SSD)에 따라서 적용하는 I/O 스케줄러도 달라지게 된다. 오늘 글에서는 아래의 순서를 통해서 현재 OS의 I/O 스케줄러를 확인하는 방법 및 변경하는 방법, 워크로드 별 적합한 I/O 스케줄러에 대하여 살펴보려고 한다.

---

## I/O 스케줄러 확인 및 변경 방법

Linux 서버에 장착된 디스크별 설정된 I/O 스케줄러는 아래와 같은 방법으로 확인 및 수정 할 수 있다.

1. 확인

> [root@al2 ~]# cat /sys/block/nvme0n1/queue/scheduler  
> [none] mq-deadline kyber bfq

2. 변경

```text
[root@al2 ~]# echo bfq > /sys/block/nvme0n1/queue/scheduler
[root@al2 ~]# cat /sys/block/nvme0n1/queue/scheduler
mq-deadline kyber [bfq] none
```

3. Iosched 폴더 확인

I/O 스케줄러를 변경하게 되면, 이에 따라서 Iosched 폴더의 파일의 내용이 변경되게 된다. 이러한 파라미터의 변경으로 각각의 스케줄러는 서로 다르게 동작하는 것을 알 수 있다.

```text
(1). bfq
[root@al2 queue]# ls iosched
back_seek_max fifo_expire_async low_latency slice_idle strict_guarantees back_seek_penalty fifo_expire_sync max_budget slice_idle_us timeout_sync

(2). mq-deadline
[root@al2 queue]# ls iosched
fifo_batch front_merges read_expire write_expire writes_starved

(3). kyber
[root@al2 queue]# ls iosched
read_lat_nsec write_lat_nsec
```

---

## 워크로드 별 I/O 스케줄러

디스크 유형인 SSD, HDD에 따라 권장되는 I/O 스케줄러가 다르게 존재한다. 그 이유로는 디스크 유형별 데이터를 읽고 쓰는 물리적 차이에 따라 발생한다. HDD와 같은 경우 직접 데이터가 저장되어 있는 또는 저장될 플래터의 위치로 헤드가 움직여야 하는 물리적 한계가

존재하기 때문에 I/O 큐가 랜덤하게 요청되는 작업보다는 I/O 순차 작업에 유리하다. 반면 SSD는 이러한 한계(*탐색지연, 회전지연등)로 부터 자유롭기 때문에  랜덤 I/O 요청에 유리하게 된다.

[다양한 사용 사례에 맞는 다른 디스크 스케줄러](https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/different-disk-schedulers-for-different-use-cases_setting-the-disk-scheduler)

| SCSI 인터페이스가 있는 기존 HDD | **mq-deadline** 또는 **bfq** 를 사용합니다. |
| --- | --- |
| 고성능 SSD 또는 빠른 스토리지가 있는 CPU 바운드 시스템 | 특히 엔터프라이즈 애플리케이션을 실행할 때는 **none** 을 사용합니다. 또는 **kyber** 를 사용합니다. |
| 데스크탑 또는 대화형 작업 | **bfq** 사용. |
| 가상 게스트 | **mq-deadline** 사용. 멀티 큐를 사용할 수 있는 HBA(호스트 버스 어댑터) 드라이버에서는 **none** 을 사용합니다. |

---

## 1. NOOP (No Operation)

- **특성**: FIFO(First In, First Out) 큐를 사용하여, 최소한의 오버헤드로 요청을 처리합니다. 별도의 정렬이나 복잡한 로직 없이 들어온 순서대로 처리합니다.
- **권장 워크로드**: SSD 및 SAN(Storage Area Network)과 같은 고속 스토리지 시스템에서 권장됩니다. 이러한 시스템들은 자체적으로 I/O 최적화를 수행하기 때문에, 추가적인 스케줄링 로직이 오히려 성능을 저하시킬 수 있습니다.

## 2. Deadline

- **특성**: 읽기/쓰기 요청이 데드라인을 넘기지 않도록 보장합니다. 데드라인을 기반으로 요청을 정렬하여, 요청이 오래 대기하지 않도록 합니다.
- **권장 워크로드**: 대부분의 일반적인 워크로드에 적합하며, 특히 읽기/쓰기 요청의 지연 시간을 최소화해야 하는 데이터베이스 서버와 같은 환경에서 좋습니다.

## 3. CFQ (Completely Fair Queuing)

- **특성**: 프로세스별로 I/O 대역폭을 공평하게 분배합니다. 각 프로세스에 대한 I/O 요청을 별도로 관리하여, 한 프로세스의 I/O 요청이 다른 프로세스의 성능에 영향을 미치지 않도록 합니다.
- **권장 워크로드**: 멀티태스킹 환경에서 여러 프로세스가 동시에 다양한 I/O 작업을 수행할 때 적합합니다. 일반적인 데스크탑 환경이나 대화형 시스템에서 유용합니다.

## 4. BFQ (Budget Fair Queuing)

- **특성**: CFQ와 유사하게 프로세스별로 I/O 대역폭을 공평하게 분배하지만, 더 정교한 방식으로 I/O 성능을 관리합니다.
- **권장 워크로드**: 멀티미디어 처리 및 실시간 애플리케이션과 같이 일관된 응답 시간이 중요한 환경에서 좋습니다.

## 5. mq-deadline (멀티 큐 데드라인)

- **특성**: Deadline 스케줄러의 멀티 큐 버전으로, 멀티코어 시스템에서 각 코어가 자체 I/O 큐를 관리할 수 있도록 합니다.
- **권장 워크로드** :
  - 멀티코어 시스템에서의 고성능 I/O 처리
  - 읽기/쓰기 요청의 시간 민감성이 중요한 환경
  - 높은 I/O 처리량과 낮은 지연 시간이 요구되는 환경

대표적으로 많이 사용하는 Deadline / CFQ 에 대해서는 아래의 그림을 참고하여 동작 원리를 살펴 볼 수 있다.

**Deadline**

![](/assets/img/posts/linux-io-scheduler/03.png)

**CFQ**

![](/assets/img/posts/linux-io-scheduler/04.png)

---

Linux I/O 스케줄러를 잘 이용한다면 운영하는 서비스의 워크로드의 I/O 작업을 최적화 할 수 있을 것으로 보인다.

나중에 기회가 된다면 직접 이를 테스트 해보고 I/O 스케줄별 벤치마크 결과를 정리해 보려고 한다.
