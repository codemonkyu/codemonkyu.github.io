---
title: "[Redis] 내가 아는 Redis IO MultiFlexing"
date: 2026-08-10 18:01:03 +0900
categories: ["Cloud & Infrastructure", "쉽게보는 IT"]
tags: []
tistory_url: https://codemonkyu.tistory.com/entry/Redis%EB%82%B4%EA%B0%80%EC%95%84%EB%8A%94-IO-MultiFlexing
---

[Redis 엔진 동작 원리](/posts/redis-internals/)를 정리하면서 한 가지를 얼버무리고 넘어갔다. "단일 스레드 이벤트 루프가 소켓 이벤트를 기다린다"고 썼는데, 스레드가 하나뿐인데 연결 수천 개를 동시에 어떻게 지켜보는지는 설명하지 않았다.

답이 I/O 멀티플렉싱이다. 그리고 이걸 알아야 풀리는 실무 질문이 몇 개 있다.

연결을 많이 열어두면 메모리가 얼마나 드는지, `maxclients`를 왜 올려야 하는지, ElastiCache의 "I/O Multiplexing" 기능은 이것과 같은 건지 다른 건지.

전부 로컬 Redis 8.10에 실제로 연결을 붙여 측정했다. Redis 내부가 궁금한 사람, 연결 수가 많은 환경을 운영하는 사람에게 맞는 글이다.

## 한 문장 정의와 요약

I/O 멀티플렉싱은 **여러 소켓을 한 번의 시스템 호출로 감시하다가, 준비된 것만 알려받아 처리하는** 방식이다.

기억할 건 세 가지다. 첫째, 병렬 처리가 아니다 — 실행은 여전히 하나씩이고 기다리는 비용만 없앤 것이다. 둘째, 연결당 비용이 스레드 방식보다 수백 배 낮아서 수천 연결이 가능해진다. 셋째, ElastiCache의 "I/O Multiplexing"은 이 OS 기능과 이름만 같고 층이 다르다.

## 왜 필요한가: 기다리는 비용

소켓에서 데이터를 읽는 가장 단순한 코드는 이렇다.

```
read(fd, buf, size);   // 데이터가 올 때까지 여기서 멈춘다
```

문제는 이게 **블로킹**이라는 점이다. 데이터가 안 오면 그 자리에서 기다린다. 연결이 하나면 괜찮은데, 여러 개면 곤란해진다.

A를 기다리는 동안 B에 데이터가 와도 처리하지 못한다.

전통적인 해법이 연결마다 스레드를 두는 것이다. 각 스레드가 자기 연결에서 블로킹하면, 다른 연결은 다른 스레드가 처리한다. 동작은 한다. 그런데 비용이 크다.

스레드 하나의 기본 스택이 보통 8MB다. 연결 1,000개면 스택만 8GB다. 실제로 그만큼 물리 메모리를 다 쓰는 건 아니지만, 컨텍스트 스위칭 비용과 스케줄러 부담이 그대로 남는다. 연결 대부분이 아무 일 없이 대기만 하는데도 그렇다.

그래서 나온 게 멀티플렉싱이다. **소켓을 논블로킹으로 열어두고, "이 중에 준비된 게 있으면 알려줘"를 OS에 맡긴다.**

---

## 어떻게 동작하나

![redis-io-multiplexing](/assets/img/posts/redis-io-multiplexing/01.gif)
_redis-io-multiplexing_

왼쪽이 연결당 스레드, 오른쪽이 멀티플렉싱이다.

오른쪽을 보면 스레드 하나가 6개 fd를 통째로 감시하다가, 데이터가 도착한 두 개만 처리한다. 나머지 넷은 건드리지 않는다.

핵심은 **전부 순회하지 않는다**는 점이다.  순회한다면 연결이 늘어날수록 느려진다. epoll이나 kqueue는 준비된 것만 골라 돌려준다.

Redis가 어떤 API를 쓰는지는 직접 확인할 수 있다.

```
$ redis-cli INFO server | grep multiplexing
multiplexing_api:kqueue
```

내 환경은 macOS라 `kqueue`다. Linux에서 돌리면 `epoll`이 나온다. Redis는 빌드 시점에 플랫폼에 맞는 걸 고른다.

| API | 플랫폼 | 특징 |
|---|---|---|
| `epoll` | Linux | 준비된 fd만 반환. 연결 수와 무관하게 O(1) |
| `kqueue` | macOS, BSD | epoll과 같은 계열 |
| `select` | 어디서나 | fd를 전부 순회. 1024개 제한 |
| `evport` | Solaris | — |

`select`는 오래된 방식인데, 이게 왜 문제인지가 멀티플렉싱을 이해하는 열쇠다. `select`는 매번 감시 대상 전체를 커널에 넘기고 커널도 전체를 검사한다. 연결이 1만 개면 1만 개를 훑는다. 준비된 게 하나뿐이어도 그렇다.

`epoll`은 감시 목록을 커널에 등록해두고, 이벤트가 생긴 것만 큐에 쌓아 돌려준다. 그래서 연결 수가 늘어도 반환 비용이 준비된 개수에만 비례한다. Redis가 수만 연결을 감당하는 이유가 여기 있다.

---

## 연결 하나의 실제 비용

이론은 그렇고, 실제로 얼마나 드는지 재봤다. 연결을 늘려가며 `used_memory` 증가분을 나눴다.

```
기준(연결 1개):  956 KB

연결  200개:    3,468 KB 증가  → 연결당 약 17.3 KB
연결  500개:    7,056 KB 증가  → 연결당 약 14.1 KB
연결 1000개:   13,123 KB 증가  → 연결당 약 13.1 KB
```

연결당 **약 13KB**로 수렴한다. 초기에 17KB로 나온 건 고정 오버헤드가 섞여서다.

스레드 방식과 비교하면 이렇다.

| 방식 | 연결당 비용 | 1,000 연결 |
|---|---|---|
| I/O 멀티플렉싱 | 약 13 KB | 약 13 MB |
| 연결당 스레드 (스택만) | 약 8 MB | 약 8 GB |

**약 600배 차이다.** 이 숫자가 "수천 연결이 가능하다"의 근거다.

서버 쪽에서 본 모습도 확인했다. 연결 801개를 붙인 상태다.

```
$ redis-cli INFO clients
connected_clients:801
maxclients:10000
blocked_clients:0

$ redis-cli INFO memory | grep mem_clients_normal
mem_clients_normal:1843200        # 약 1.8MB
```

그리고 프로세스 스레드 수를 세보면 5개다. 그중 명령을 실행하는 건 하나뿐이고 나머지는 백그라운드 작업용이다. **연결 801개를 스레드 1개가 처리하고 있다.**

---

## CLIENT LIST로 들여다보기

각 연결의 상태를 직접 볼 수 있다. 이 명령이 멀티플렉싱의 실체를 가장 잘 보여준다.

```
$ redis-cli CLIENT LIST
id=2318 addr=127.0.0.1:56250 fd=11 age=2 idle=2 flags=N db=0
  qbuf=0 qbuf-free=0 rbs=1024 obl=0 oll=0 omem=0 tot-mem=2304
  events=r cmd=NULL io-thread=0 tot-net-in=0 tot-net-out=0
```

봐야 할 필드는 이렇다.

| 필드 | 의미 |
|---|---|
| `fd=11` | 이 연결의 파일 디스크립터. 멀티플렉싱이 감시하는 대상 |
| `events=r` | 읽기 이벤트를 감시 중 (`w`면 쓰기, `rw`면 둘 다) |
| `tot-mem=2304` | 이 연결이 쓰는 메모리 (약 2.3KB) |
| `io-thread=0` | 담당 I/O 스레드 번호 |
| `idle=2` | 2초간 유휴 |
| `qbuf` / `obl` / `oll` | 입력 버퍼, 출력 버퍼 |

여러 연결을 붙여놓고 보면 `fd`가 10, 11, 12, 13, 14로 연속 배정되는 게 보인다. 그리고 유휴 연결도 `events=r` 상태로 감시 목록에 남아 있다. **아무 일도 안 하지만 등록은 되어 있는 상태**, 이게 멀티플렉싱이 관리하는 것이다.

`tot-mem`이 2,304바이트인데 위에서 잰 연결당 13KB보다 작다. 차이는 커널 쪽 소켓 버퍼와 Redis 내부 구조체 등 `tot-mem`에 잡히지 않는 부분이다.

---

## 실무에서 걸리는 것들

### maxclients와 파일 디스크립터

연결 하나가 fd 하나를 쓰므로, **OS의 fd 한도가 곧 연결 한도**다.

```bash
$ redis-cli CONFIG GET maxclients
1) "maxclients"
2) "10000"
```

여기서 함정이 있다. `maxclients`를 올려도 OS의 `ulimit -n`이 낮으면 Redis가 시작할 때 값을 자동으로 낮춘다. 로그에 경고가 남는다. 그런데 그 로그를 안 보면 "설정했는데 왜 안 되지" 하게 된다.

Redis는 자기 몫으로 32개를 예약한다(복제, AOF, 클러스터 버스 등). 그래서 실제 클라이언트 한도는 그만큼 적다.

ElastiCache에서는 `maxclients`가 노드 타입에 따라 정해져 있고 직접 올릴 수 없는 경우가 있다. 연결이 한도에 가까우면 노드 타입 상향을 검토해야 한다.

### 연결이 많으면 무엇이 느려지나

멀티플렉싱 덕에 연결 수 자체는 부담이 적다. 그런데 완전히 공짜는 아니다.

**메모리.** 연결당 13KB니 1만 연결이면 130MB다. 데이터 메모리와 별개로 나가는 비용이다. `INFO memory`의 `mem_clients_normal`로 볼 수 있다.

**출력 버퍼.** 큰 응답을 보내는 연결이 많으면 출력 버퍼가 쌓인다. 이건 연결 수보다 응답 크기 문제인데, 한도를 넘으면 Redis가 그 클라이언트를 끊는다. 정리해두신 Client Output Buffer 노트의 주제다.

**연결 수립 비용.** 연결을 유지하는 건 싸지만 새로 맺는 건 그렇지 않다. 실측해보면 500개 수립에 0.47초, 초당 약 1,074개였다. TLS를 쓰면 핸드셰이크 때문에 더 느려진다. 그래서 **커넥션 풀을 쓰는 게 중요하다.** 매 요청마다 연결을 맺으면 이 비용을 계속 낸다.

반면 이미 맺어진 연결로 명령을 보내는 건 빠르다. 500개 연결에 동시에 `PING`을 보내고 전부 응답받는 데 29밀리초였다.

### 멀티플렉싱이 못 막는 것

멀티플렉싱은 **기다리는 비용**을 없앤다. 실행 비용은 그대로다.

그래서 느린 명령 하나가 이벤트 루프를 막는 문제는 여전히 남는다. `KEYS *`가 40ms를 쓰면 그동안 다른 fd가 준비돼 있어도 처리하지 못한다. 멀티플렉싱이 "준비됐다"고 알려준 것들이 순서를 기다린다.

이게 앞서 [Redis 엔진 편]에서 다룬 O(N) 명령 문제다. 멀티플렉싱과 별개의 층이라 서로를 해결해주지 않는다.

## ElastiCache의 "I/O Multiplexing"은 다른 것이다

여기서 이름 때문에 헷갈리는 지점을 정리해야 한다.

AWS ElastiCache 문서에도 "I/O Multiplexing"이 나오는데, 지금까지 설명한 OS 기능과 층이 다르다.

| | OS 레벨 (이 글의 주제) | ElastiCache 기능 |
| 무엇 | `epoll` / `kqueue` 시스템 호출 | ElastiCache의 성능 최적화 |
| 어디 | Redis 프로세스 내부 | AWS 관리형 계층 |
| 켜고 끄기 | 불가 (Redis 기본 동작) | 노드 타입에 따라 제공 |
| 관련 기능 | — | Enhanced I/O, TLS Offload |

Redis는 **원래부터** OS 멀티플렉싱을 쓴다. 켜고 끄는 옵션이 아니다.

ElastiCache가 말하는 건 그 위에서 AWS가 추가한 최적화 계층이다. Enhanced I/O는 네트워크 처리를 별도로 분리하고, TLS Offload는 암호화를 다른 곳으로 넘긴다.

기본 멀티플렉싱은 이미 동작하고 있고, ElastiCache 기능은 처리량 쪽 이야기다.

## 흔한 오해와 헷갈리는 지점

정리하면서, 그리고 내가 흐릿했던 것들을 모았다.

멀티플렉싱이 여러 요청을 동시에 처리하는 거라고 생각했다. 아니다. 감시를 동시에 하는 것이고 처리는 하나씩이다.

 병렬성이 아니라 **대기 비용 제거**다.

`select`와 `epoll`이 같은 거라고 봤다. `select`는 fd를 전부 순회해서 연결이 늘면 느려지고 1024개 제한도 있다. `epoll`은 준비된 것만 반환한다. Redis가 수만 연결을 감당하는 건 후자 덕이다.

`maxclients`만 올리면 연결이 늘어난다고 생각했다. OS의 `ulimit -n`이 낮으면 Redis가 시작 시 값을 낮춘다. 로그에 경고가 남는데 안 보면 놓친다.

연결이 유휴면 비용이 없다고 봤다. 유휴여도 `events=r`로 감시 목록에 남아 있고 연결당 약 13KB를 계속 쓴다. 안 쓰는 연결은 닫는 게 맞다.

멀티플렉싱이 있으니 느린 명령도 괜찮다고 생각했다. 별개의 층이다. `KEYS *`가 이벤트 루프를 잡으면 준비된 fd들이 그대로 기다린다.

ElastiCache의 I/O Multiplexing을 OS 기능과 같은 것으로 봤다. 이름만 같다. OS 멀티플렉싱은 Redis 기본 동작이고 끌 수 없다.

## 요약

I/O 멀티플렉싱은 여러 소켓을 한 번의 시스템 호출로 감시하다 준비된 것만 처리하는 방식이다. 연결마다 스레드를 두는 대신 하나가 전부를 보므로, 연결당 비용이 약 13KB로 스레드 스택(약 8MB)의 600분의 1 수준이다. `INFO server`의 `multiplexing_api`로 어떤 API를 쓰는지, `CLIENT LIST`의 `fd`와 `events`로 무엇을 감시하는지 볼 수 있다. 다만 이건 대기 비용만 없앤 것이고, 느린 명령이 이벤트 루프를 막는 문제는 그대로 남는다. ElastiCache 문서의 "I/O Multiplexing"은 이 OS 기능과 층이 다른 별개의 최적화다.

다음으로 볼 만한 것은 커넥션 풀 설계(풀 크기를 얼마로 둘지, 유휴 연결을 언제 닫을지)나 Client Output Buffer 한도다. 연결 수가 많은 환경에서 실제로 문제가 되는 건 연결 자체보다 이쪽이다.

---

## 참고 자료

- [Redis 설정](https://redis.io/docs/latest/operate/oss_and_stack/management/config/) — `maxclients`, `tcp-keepalive` 등
- [CLIENT LIST](https://redis.io/docs/latest/commands/client-list/) — 각 필드의 의미
- [INFO](https://redis.io/docs/latest/commands/info/) — `multiplexing_api`, `connected_clients`, `mem_clients_normal`
- [ElastiCache 노드 크기 선택](https://docs.aws.amazon.com/ko_kr/AmazonElastiCache/latest/dg/nodes-select-size.html) — 노드 타입별 연결 한도

## 실측 환경

Redis 8.10.0 (arm64, Darwin), 로컬 단독 인스턴스, `maxclients=10000` 기본값.
연결당 메모리(13KB), 수립 속도(1,074 conn/s), 동시 PING 왕복(29ms), `CLIENT LIST` 출력은 이 환경에서 직접 측정한 값이다.
Linux에서는 `multiplexing_api`가 `epoll`로 나오고, 연결당 메모리도 커널 설정에 따라 달라진다.
