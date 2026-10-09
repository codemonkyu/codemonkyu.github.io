---
title: "[Redis] 내가 아는 Redis"
date: 2026-08-07 13:18:07 +0900
categories: ["Cloud & Infrastructure", "쉽게보는 IT"]
tags: []
tistory_url: https://codemonkyu.tistory.com/entry/Redis-%EB%82%B4%EA%B0%80-%EC%95%84%EB%8A%94-Redis
---

## 한 문장 정의와 요약

Redis는 단일 스레드 이벤트 루프가 메모리 위의 자료구조를 조작하고, 그 변경을 디스크와 replica로 흘려보내는 엔진이다.

이 문장에 다섯 축이 들어 있다. 이벤트 루프(어떻게 처리하나), 자료구조(어떻게 담나), 메모리 관리(어떻게 지우나), 영속성(어떻게 남기나), 복제(어떻게 퍼뜨리나). 아래는 이 순서다.

기억할 건 세 가지다. 첫째, 스레드가 하나라서 락이 없는 대신 느린 명령 하나가 전부를 막는다. 둘째, 지우는 이유가 시간(만료)과 압박(축출) 두 가지고 서로 다른 메커니즘이다. 셋째, 저장과 복제는 fork와 스트리밍으로 서비스를 멈추지 않고 처리한다.

## 왜 단일 스레드인가

먼저 짚어야 할 게 있다. Redis가 단일 스레드라는 건 성능을 포기한 결과가 아니라 의도적 선택이다.

멀티스레드로 자료구조를 공유하려면 락이 필요하다. 락은 획득·해제 비용이 들고, 잘못 쓰면 데드락이 나고, 경쟁이 심하면 오히려 느려진다. Redis는 애초에 동시에 하나만 실행하는 쪽을 택했다. 그러면 락이 필요 없다.

부수 효과가 크다. `INCR`이 원자적인 이유가 이것이다. "읽고 더하고 쓰기"가 세 단계인데 그 사이에 아무도 끼어들 수 없으니, 별도 트랜잭션 없이도 안전하다. 컨텍스트 스위칭 비용도 없다.

Redis가 빠른 이유는 멀티스레드여서가 아니다. 대부분의 명령이 O(1)이고 메모리에서만 동작해서, 하나씩 처리해도 충분히 빠르기 때문이다. 문제는 O(1)이 아닌 명령이 섞일 때 생긴다.

---

### 잠깐, O(1)이 뭔가

이 글 전체가 이 표기에 기대고 있으니 짚고 가자. O 표기는 **데이터가 늘어날 때 소요 시간이 어떤 모양으로 늘어나는지**를 나타낸다. 절대 시간이 아니라 증가 방식이 관심사다.

책장에서 책을 찾는 일로 비유하면 이렇다.

```
O(1)  "3번째 칸, 왼쪽에서 5번째 책"
      → 책이 100권이든 100만 권이든 손 한 번 뻗으면 끝

O(N)  "제목에 '네트워크'가 들어간 책 전부 찾아줘"
      → 100권이면 100권 확인, 100만 권이면 100만 권 확인
```

`N`은 "다뤄야 할 데이터 개수"다. Redis 명령에 붙는 것들만 정리하면 이렇다.

| 표기 | 뜻 | Redis 예시 |
|---|---|---|
| O(1) | 데이터가 늘어도 시간이 그대로 | `GET`, `SET`, `HGET`, `LPUSH` |
| O(log N) | 아주 조금 늘어남 | `ZADD`, `ZRANGE` |
| O(N) | 데이터에 비례해 늘어남 | `KEYS`, `HGETALL`, `SMEMBERS` |

`GET`이 O(1)인 이유는 Redis가 키를 해시테이블에 담기 때문이다. 키 이름을 해시하면 저장 위치가 바로 나오니, 키가 100만 개여도 한 번에 찾는다. `KEYS`는 패턴에 맞는 걸 찾아야 하므로 전부 확인한다.

중요한 건 절대값이 아니라 **키가 늘어나면 어떻게 되느냐**다.

| 키 수 | `GET` | `KEYS *` |
|---|---|---|
| 1,000 | 2 µs | 약 0.1 ms |
| 500,000 | 2 µs | 42.6 ms |
| 5,000,000 | **2 µs** | **약 400 ms** |

`GET`은 변하지 않고 `KEYS *`만 늘어난다. 그래서 개발 환경(키 1,000개)에서는 `KEYS`가 멀쩡해 보이다가 운영에서 터진다.

정리하면 단일 스레드의 전제가 이것이다. **O(1) 명령만 있으면 줄을 세워도 문제가 없다.** 하나가 2마이크로초에 끝나니 초당 수십만 개를 처리한다. O(N) 명령 하나가 그 전제를 깨뜨리는 게 다음 절의 내용이다.

새 명령을 쓸 때는 복잡도를 확인하는 습관이 도움이 된다. 공식 문서의 각 명령 페이지에 명시돼 있고, 로컬에서도 볼 수 있다.

```
$ redis-cli COMMAND DOCS HGETALL | grep -A1 complexity
```

![redis-event-loop](/assets/img/posts/redis-internals/01.gif)
_redis-event-loop_

한 사이클은 다섯 단계다. 소켓 이벤트를 기다리고, 요청을 읽고, 명령을 실행하고, 응답을 쓰고, 마지막에 `serverCron`으로 살림(만료 처리, 리해싱)을 한다. 그리고 처음으로 돌아간다.

여기서 ③이 직렬화 지점이다. 세 클라이언트가 동시에 요청해도 실행은 한 줄로 선다. 그리고 ⑤도 같은 스레드라는 게 중요하다. 뒤에 나올 만료 처리가 스스로 시간을 제한하는 이유가 여기 있다. 안 그러면 자기가 이벤트 루프를 막는다.

---

## 느린 명령 하나가 전부를 멈춘다

단일 스레드의 대가는 여기서 나타난다.

![redis-blocking-on](/assets/img/posts/redis-internals/02.gif)
_redis-blocking-on_

`KEYS *` 하나가 도는 동안 다른 클라이언트의 `GET`은 그냥 기다린다. `GET` 자체는 마이크로초 단위로 끝나는 명령인데, 앞사람 때문에 40ms를 기다린 것으로 기록된다.

이게 실무에서 가장 헷갈리는 지점이다. 애플리케이션 로그에는 "Redis `GET` 응답 60ms"로 찍힌다. 그걸 보고 `GET`을 최적화하려 들면 헛수고다. 범인은 다른 클라이언트가 돌린 `KEYS *`거나, 백만 필드 Hash를 지운 `DEL`이거나, 수만 건을 순회하는 Lua 스크립트다.

내가 처음에 오해했던 게 있다. `io-threads`를 올리면 이 문제가 해결될 거라고 생각했다. 아니다. `io-threads`가 병렬화하는 건 소켓 read/write와 프로토콜 파싱뿐이고, **명령 실행은 여전히 단일 스레드다.** 값을 4로 올려도 `KEYS *`가 막는 건 그대로다.

피해야 할 명령과 대안을 정리하면 이렇다.

| 위험한 명령 | 대안 | 이유 |
|---|---|---|
| `KEYS pattern` | `SCAN cursor MATCH` | 커서로 조금씩 훑어 중간에 다른 요청을 처리 |
| `SMEMBERS` / `HGETALL` | `SSCAN` / `HSCAN` | 같은 이유 |
| `DEL bigkey` | `UNLINK` | 메모리 해제를 백그라운드 스레드에 넘김 |
| `FLUSHALL` | `FLUSHALL ASYNC` | 같은 이유 |

`SCAN`의 총 소요 시간은 `KEYS`보다 길 수 있다. 하지만 운영에서 중요한 건 총 시간이 아니라 최대 블로킹 시간이다. 한 번에 40ms 막는 것보다, 여러 번 나눠 24마이크로초씩 쓰는 게 낫다.

다만 `SCAN`에는 대가가 있다. 순회 중 키가 추가·삭제되면 중복이 나올 수 있고 일부를 놓칠 수도 있다. 정확한 스냅샷이 필요하면 다른 방법을 써야 한다.

---

## 같은 Hash인데 메모리가 3배 차이난다

다음은 자료구조다. 그런데 이 절을 읽으려면 알아야 할 게 두 개 있다. 먼저 그걸 짚고 간다.

### 사전 지식 1: Hash는 "키 안의 작은 딕셔너리"다

Redis의 키-값에서 값 자리에 여러 자료형이 올 수 있다. 그중 `Hash`는 **필드-값 쌍을 여러 개 담는 타입**이다. 파이썬 딕셔너리나 JSON 객체를 하나의 키 안에 넣은 셈이다.

```
# String — 값이 하나
$ redis-cli SET user:1:name "alice"

# Hash — 값이 여러 필드로 나뉜다
$ redis-cli HSET user:1 name "alice" age 30 city "Seoul"
$ redis-cli HGET user:1 name
"alice"
```

`user:1`이라는 키 하나에 `name`, `age`, `city` 세 **필드**가 들어 있다. 사용자 프로필처럼 관련 값을 묶어 둘 때 쓴다. String 키를 세 개 만드는 것보다 메모리도 적게 드는데, 재보면 이 정도 차이다.

```
# String 3개로 나눠 담으면
$ redis-cli MEMORY USAGE u:2:name   # 이런 식으로 3개 합계
122 bytes

# Hash 1개로 담으면
$ redis-cli MEMORY USAGE u:3
62 bytes                            # 약 2배 절약
```

키 이름이 반복되지 않고 관리 오버헤드도 한 번만 들어서다.

이 절에서 "필드 512개"라는 말이 나오면 그건 Hash 하나에 필드가 512개 들어 있다는 뜻이다.

### 사전 지식 2: 같은 타입도 저장 방식이 여러 개다

여기가 핵심이다. `Hash`는 **사용자가 보는 타입 이름**이고, Redis가 메모리에 실제로 담는 방식은 따로 있다. 그리고 그 방식이 크기에 따라 자동으로 바뀐다.

```
사용자가 쓰는 것 (논리 타입)      Redis가 담는 방식 (물리 인코딩)
──────────────────────         ────────────────────────────
Hash  ───────────────────┬──▶  listpack    (작을 때)
                         └──▶  hashtable   (커질 때)
```

두 방식이 어떻게 다른지가 이 절의 주제인데, 이름부터 풀어보자.

**`listpack`**

-- 값을 **연속된 메모리에 이어 붙인** 형태다. `[name][alice][age][30][city][Seoul]` 처럼 한 덩어리로 담는다. 주소를 가리키는 포인터가 없어 낭비가 없다. 대신 `age`를 찾으려면 앞에서부터 훑어야 한다.

**`hashtable`**
-- 이름 그대로 **해시테이블**이다. 필드 이름을 해시해서 저장 위치를 계산하니 바로 찾아간다.
-- 대신 버킷 배열과 포인터 같은 구조가 필요해 자리를 더 쓴다.

여기서 앞 절의 O 표기가 다시 나온다. `listpack`은 훑어야 하니 **O(N)**, `hashtable`은 계산해서 바로 가니 **O(1)** 이다.

정리하면 트레이드오프가 정반대인 두 방식이고, Redis는 크기에 따라 유리한 쪽을 고른다.

| | `listpack` | `hashtable` |
|---|---|---|
| 담는 방식 | 연속 메모리에 이어 붙임 | 버킷 배열 + 포인터 |
| 필드 조회 | O(N) — 훑는다 | O(1) — 계산해서 찾는다 |
| 메모리 | 적다 | 많다 |
| 언제 | 작을 때 | 커질 때 |

이제 본론이다.

![redis-encoding-switch](/assets/img/posts/redis-internals/03.gif)
_redis-encoding-switch_

작을 때는 `listpack`을 쓴다. 연속된 메모리에 값을 이어 붙이는 방식이라 포인터가 없고 밀도가 높다. 대신 특정 필드를 찾으려면 앞에서부터 훑어야 한다 — O(N)이다.

커지면 `hashtable`로 바뀐다. 버킷 배열과 엔트리 구조체, 포인터가 생겨 오버헤드가 커지는 대신 조회가 O(1)이 된다.

설계가 맞물려 있다. 작을 때 O(N)을 감수해 메모리를 아끼고, 커지면 메모리를 내주고 O(1)을 산다. 필드 10개를 순차 탐색하는 비용은 사실상 무시할 수 있고, 게다가 연속 메모리라 CPU 캐시에 여러 엔트리가 한꺼번에 올라와서 오히려 빠를 수도 있다.

전환 조건이 두 개인데, 여기서 함정이 하나 있다.

```
hash-max-listpack-entries  512   ← 필드 개수
hash-max-listpack-value     64   ← 값 크기 (바이트)
```

**둘 중 하나만 넘어도 전환된다.** 필드가 단 1개여도 값이 64바이트를 넘으면 hashtable로 간다.

주의할 게 하나 더 있다. 64바이트는 문자 수가 아니라 바이트다. 한글은 UTF-8에서 글자당 3바이트니 22글자만 넘어도 초과한다. 한글 데이터를 다룰 때 예상보다 훨씬 빨리 전환된다.

그리고 **전환은 단방향이다.** 필드를 지워 다시 작아져도 `listpack`으로 돌아오지 않는다. 일시적으로 커졌던 키가 영구히 큰 메모리를 쓰는 상태로 남는다. 되돌리려면 키를 삭제하고 다시 만들어야 한다.

`OBJECT ENCODING`으로 지금 상태를 확인할 수 있다.

```
$ redis-cli OBJECT ENCODING user:profile:12345
"hashtable"

$ redis-cli HLEN user:profile:12345
(integer) 12          # 필드는 12개뿐인데?
```

필드 12개인데 hashtable이면 값 크기가 원인이다. `HGETALL`로 각 값의 바이트를 확인해보면 범인이 나온다.

---

## 지우는 이유가 두 가지다

이제 메모리 관리다. Redis가 키를 지우는 계기는 둘이고, 섞으면 진단이 어긋난다.

![](/assets/img/posts/redis-internals/04.gif)

**만료(expiration)** 는 시간 때문이다. TTL이 지난 키만 대상이고, TTL 없는 키는 영향을 받지 않는다. 지표는 `expired_keys`다.

**축출(eviction)** 은 메모리 압박 때문이다. `maxmemory`에 도달하면 `maxmemory-policy`에 따라 대상을 골라 지운다. TTL이 없는 키도 대상이 될 수 있다. 지표는 `evicted_keys`다.

기본 정책이 `noeviction`이라는 점을 알아둘 만하다. 이 경우 지우지 않고 쓰기를 거부한다.

```
(error) OOM command not allowed when used memory > 'maxmemory'
```

정책별로 대상 선정이 다르다. `allkeys-lru`는 전체에서 오래 안 쓴 것을, `volatile-lru`는 TTL 있는 키 중에서만 고른다. 캐시 용도라면 `allkeys-*`, TTL 있는 것만 지우고 싶으면 `volatile-*`다.

진단은 이 순서다. `expired_keys`가 안 오르면 TTL 문제, `evicted_keys`가 오르면 용량 문제다.

---

## TTL이 지났는데 왜 아직 남아있나

만료를 좀 더 파보자. 여기가 실무에서 가장 자주 부딪히는 지점이다.

Redis는 만료 시각에 키를 즉시 지우지 않는다. 키마다 타이머를 두면 키가 수백만 개일 때 타이머도 수백만 개가 되기 때문이다. 그 비용을 내지 않는 대신 두 가지 방식을 쓴다.

![](/assets/img/posts/redis-internals/05.gif)

**lazy(passive)** 는 접근할 때 검사한다. 공식 문서 표현으로 "클라이언트가 접근을 시도하고 그 키가 시간 초과됐을 때 수동적으로 만료된다". `GET`이 삭제를 유발하는 셈이다. 읽기 명령인데 쓰기 부수효과가 있다.

그런데 이것만으로는 부족하다. 문서가 이유를 명확히 적어뒀다. "다시는 접근되지 않을 만료된 키들이 있기 때문이다." 세션 키를 생각해보면 된다. 사용자가 떠나면 그 키는 다시 읽히지 않는다.

그래서 **active** 가 있다. `serverCron`이 주기적으로 무작위 표본을 뽑아 만료된 것을 지운다. 알고리즘의 골자는 이렇다.

```
반복 {
    expires 딕셔너리에서 무작위로 최대 20개 표본 추출
    만료된 것을 삭제
    if (삭제된 비율 < 25%) break      ← 표본 대부분이 살아있으면 중단
    if (시간 예산 초과) break          ← CPU를 독점하지 않도록
}
```

표본 20개 중 5개 미만이 만료됐으면 "지금은 만료 키가 별로 없다"고 판단하고 멈춘다. 많이 만료됐으면 계속 돈다. 만료 키가 많을 때만 열심히 일하는 적응형 구조다. 그리고 이 시간 예산이 앞서 말한 "자기가 이벤트 루프를 막지 않으려는" 장치다.

직접 확인해볼 수 있다. TTL 1초짜리 키를 대량으로 만들고 **한 번도 접근하지 않은 채** 기다리면 된다.

```
$ redis-cli INFO stats | grep expired
expired_keys:1000
expired_keys_active:1000     # 전부 active가 처리했다
```

`expired_keys_active`가 전체와 같으면 lazy가 아니라 active가 다 했다는 뜻이다. 두 값의 비율로 접근 패턴을 짐작할 수도 있다. active가 압도적이면 "쓰고 안 읽는" 키가 많은 것이다.

### TTL이 사라지는 함정

여기서 실무에서 제일 자주 밟는 함정을 짚어야 한다. TTL은 키를 삭제하거나 **값을 새로 덮어쓰는** 명령에서 지워진다.

```
$ redis-cli SET mykey "Hello"
$ redis-cli EXPIRE mykey 100
$ redis-cli TTL mykey
(integer) 100

$ redis-cli SET mykey "World"      # 값을 덮어씀
$ redis-cli TTL mykey
(integer) -1                        # TTL이 사라졌다
```

반면 값을 바꾸지만 교체하지 않는 명령은 TTL을 유지한다.

| TTL이 지워짐 | TTL이 유지됨 |
|---|---|
| `SET`, `GETSET`, `*STORE` 계열 | `INCR` |
| `DEL` (키 자체가 사라짐) | `LPUSH` / `RPUSH` |
| `PERSIST` (명시적 제거) | `HSET` (기존 해시의 필드 변경) |

캐시 갱신 코드에서 `SET key value`를 쓰면서 TTL 재설정을 빠뜨리면, 그 키는 영구 키가 된다. `INFO keyspace`에서 `expires` 수가 예상보다 훨씬 작으면 이걸 의심해야 한다. 해결은 `SET key value EX 3600` 형태로 바꾸거나 `SET` 직후 `EXPIRE`를 다시 거는 것이다.

---

## 저장하는 동안 서비스를 멈추지 않는 방법

영속성으로 넘어간다. 메모리의 데이터를 디스크에 남겨야 하는데, 그동안 서비스를 멈출 수는 없다.

Redis는 자식 프로세스를 만들어 그쪽이 파일을 쓰게 한다. 그런데 데이터를 통째로 복사하면 메모리가 두 배가 되고 복사 시간도 오래 걸린다. 그래서 fork와 COW(Copy-On-Write)를 쓴다.

![redis-fork-cow](/assets/img/posts/redis-internals/06.gif)
_redis-fork-cow_

`fork()`는 데이터를 복사하지 않는다. 페이지 테이블만 복사한다. 그래서 수백 MB 데이터에서도 fork 자체는 수 밀리초로 끝난다.

fork 직후 부모와 자식은 같은 물리 페이지를 공유한다. 그러다 부모가 어떤 페이지를 수정하는 순간, **그 페이지만** 복사된다. 부모는 새 사본을 쓰고 자식은 원본을 계속 본다. 이래야 자식이 fork 시점의 일관된 스냅샷을 저장할 수 있다.

여기서 실무 함의가 나온다. **저장 중 쓰기가 많으면 복사되는 페이지가 늘어나 메모리가 최대 2배까지 갈 수 있다.** 그래서 Redis 인스턴스의 메모리를 100% 채워 쓰면 위험하다. BGSAVE가 돌 때 OOM이 날 수 있다. 여유를 남기라는 권고가 이 때문이다.

`latest_fork_usec`으로 fork 비용을 확인할 수 있다.

```
$ redis-cli INFO stats | grep fork
latest_fork_usec:4639        # 4.6ms
total_forks:2
```

메모리가 클수록 페이지 테이블도 커져서 fork 시간이 늘어난다. 이 값이 수백 ms를 넘으면 그 시간만큼 이벤트 루프가 멈춘 것이므로 문제가 된다.

---

## RDB와 AOF는 무엇을 잃고 무엇을 얻나

저장 방식은 두 가지다. 같은 시간축에 겹쳐 놓으면 차이가 선명하다.

![redis-rdb-aof](/assets/img/posts/redis-internals/07.gif)
_redis-rdb-aof_

**RDB**는 특정 시점의 전체를 덤프한다. 파일이 작고 복구가 빠르다. 대신 마지막 스냅샷 이후의 쓰기는 장애 시 전부 날아간다.

**AOF**는 쓰기 명령을 순차 기록한다. 기본 설정(`appendfsync everysec`)이면 유실이 최대 1초분이다. 대신 파일이 커지고 복구도 느리다.

```
$ redis-cli CONFIG GET appendfsync
1) "appendfsync"
2) "everysec"
```

Redis 7부터 AOF 구조가 바뀌어 multi-part 방식을 쓴다. 처음 켜보면 파일이 여러 개라 당황할 수 있다.

```
appendonlydir/
├── appendonly.aof.1.base.rdb     기준점 (RDB 형식이다)
├── appendonly.aof.1.incr.aof     그 이후 증분
└── appendonly.aof.manifest       조합 정보
```

실무에서는 둘을 함께 켜는 경우가 많다. AOF로 유실을 줄이고, 복구할 때는 RDB 기반으로 빠르게 올린다. 참고로 AOF rewrite도 fork를 쓰므로 위의 COW 이야기가 그대로 적용된다.

---

## 복제는 변경을 흘려보내는 일

마지막 단계이다. primary의 변경을 replica로 스트리밍한다.

![redis-replication](/assets/img/posts/redis-internals/08.gif)
_redis-replication_

기본 동작은 단순하다. primary에 쓰면 복제 스트림으로 replica에 전달되고, 오프셋이 맞으면 동기 상태다. replica는 읽기 전용이라 쓰기를 시도하면 거부한다.

```
READONLY You can't write against a read only replica.
```

여기서 흥미로운 게 만료 처리다. **replica는 TTL이 지난 걸 알고 있어도 스스로 지우지 않는다.** primary가 보내는 `DEL`을 기다린다.

공식 문서를 인용하면 이렇다.

> master에 연결된 replica는 키를 독립적으로 만료시키지 않는다 (master로부터 오는 `DEL`을 기다린다). 하지만 데이터셋에 존재하는 만료 정보의 전체 상태는 그대로 갖고 있어서, replica가 master로 승격되면 독립적으로 키를 만료시킬 수 있게 된다.

왜 이렇게 했을까. 각 노드가 알아서 만료를 판단하면 시각 차이나 처리 순서 때문에 불일치가 생긴다. 만료 판단을 primary에 몰아두면 그런 일관성 오류가 원리적으로 없어진다.

그래서 만료로 인한 삭제는 명시적 `DEL`로 변환되어 AOF와 replica에 전달된다. 앞에서 본 lazy든 active든 결국 이 경로를 탄다.

대가는 있다. 복제 지연이 크면 그 간극에서 replica가 옛 값을 보여줄 수 있다. 읽기를 replica로 분산하는 구성에서 "방금 만료됐어야 할 값이 읽힌다"는 현상이 발생할 수 있다.

---

## 키는 어느 노드로 가는가

단일 스레드라 멀티코어를 못 쓴다는 한계는 샤딩으로 푼다. 코어를 늘리는 대신 노드를 늘리는 것이다.

![redis-cluster-slots](/assets/img/posts/redis-internals/09.gif)
_redis-cluster-slots_

Redis Cluster는 해시 슬롯 16384개를 노드가 나눠 갖는다. 키가 어느 슬롯에 속하는지는 계산으로 정해진다.

```
HASH_SLOT = CRC16(key) mod 16384
```

`CLUSTER KEYSLOT`으로 확인할 수 있다.

```
$ redis-cli CLUSTER KEYSLOT foo
(integer) 12182

$ redis-cli CLUSTER KEYSLOT user:1
(integer) 10778
```

중앙 조정자가 없다는 게 핵심이다. 키 이름만으로 어느 노드인지 결정되니, 조회할 때 누구에게 물어볼지 클라이언트가 스스로 계산한다.

왜 슬롯이라는 중간 층을 뒀을까. 키를 노드에 직접 매핑하면 노드가 추가·제거될 때마다 전체 키를 재계산해야 한다. 슬롯을 두면 **슬롯의 담당 노드만 바꾸면 된다.** 키의 슬롯은 영원히 그대로다.

---

## MOVED와 ASK는 무엇이 다른가

클라이언트가 엉뚱한 노드에 물으면 리다이렉션이 온다. 그런데 두 종류가 있고, 이 차이가 클러스터를 이해하는 관문이다.

![redis-moved-ask](/assets/img/posts/redis-internals/10.gif)
_redis-moved-ask_

**MOVED**는 "이 슬롯은 이제부터 계속 저 노드 담당"이다. 클라이언트는 슬롯맵을 갱신하고, 이후 질의도 그 노드로 보낸다.

```
$ redis-cli -p 17001 SET foo bar
(error) MOVED 12182 127.0.0.1:17003
```

**ASK**는 "이번 질의만 저 노드로 보내라"다. 슬롯맵은 그대로 두고, 다음 질의는 다시 원래 노드로 간다.

왜 두 개가 필요한가. 공식 문서의 설명이 명확하다.

> MOVED는 해시 슬롯이 영구적으로 다른 노드에서 서비스된다고 판단하며 다음 질의들도 지정된 노드로 시도해야 한다는 뜻이다.

ASK는 **다음 질의만** 지정된 노드로 보내라는 뜻이다.

슬롯을 옮기는 중이라고 생각해보자. 슬롯 안의 키 일부는 이미 옮겨갔고 일부는 아직 원래 노드에 있다. 이때 슬롯맵을 통째로 바꿔버리면, 아직 안 옮긴 키를 찾을 수 없게 된다. 그래서 "이 키만 저쪽에 있다"를 표현할 방법이 필요했다.

이관 중 두 노드는 특별한 상태가 된다.

| 노드 | 상태 | 동작 |
|---|---|---|
| 원본 (A) | `MIGRATING` | 키가 있으면 직접 응답, 없으면 ASK로 넘김 |
| 대상 (B) | `IMPORTING` | `ASKING`을 먼저 받은 질의만 처리 |

`ASKING`이라는 명령이 여기서 등장한다. 문서 표현으로 "ASKING 명령은 클라이언트에 일회용 플래그를 설정해서, IMPORTING 상태인 슬롯에 대한 질의를 노드가 처리하도록 강제한다."

3노드 클러스터를 띄워 실제로 재현해보면 이렇게 나온다.

```
# 슬롯 12182를 A→B로 이관 중인 상태에서

# 아직 A에 있는 키 → A가 직접 응답
$ redis-cli -p 17003 GET foo
"bar"

# A에 없는 같은-슬롯 키 → ASK
$ redis-cli -p 17003 GET '{foo}notexist'
(error) ASK 12182 127.0.0.1:17001
```

`{foo}` 표기는 해시 태그다. 중괄호 안의 문자열만으로 슬롯을 계산하므로, 여러 키를 같은 슬롯에 몰아넣을 수 있다. 멀티키 연산이 필요할 때 쓴다.

이관이 끝나면 A가 MOVED를 보내고, 그때 클라이언트가 슬롯맵을 영구 갱신한다.

---

## 다섯 축이 만나는 지점
이제 위에서 정리한 5개의 컴포넌트에 대한 내용을 확인한 사태에서 아래의 장애 시나리오를 확인해보자

```
느린 명령 하나 (이벤트 루프)
   → serverCron 지연
   → active expire가 밀림 (만료)
   → 만료 키가 메모리에 남음
   → maxmemory 도달 (축출)
   → eviction 발생
   → 그 사이 BGSAVE가 돌면 fork + COW로 메모리 추가 증가 (영속성)
   → 복제 버퍼도 밀림 (복제)
```

"Redis가 느리다"는 한마디가 단순하지 않은 이유다. 단일 스레드라는 하나의 제약이 나머지 넷에 모두 영향을 준다.

진단 순서로 정리하면 이렇다.

```
① SLOWLOG GET 10        느린 명령이 있나
        ↓ 없으면
② redis-cli --latency   기저 지연인가 max만 튀는가
        ↓ max만 튀면
③ LATENCY LATEST        fork? expire-cycle? 무엇이 막았나
        ↓
④ INFO stats            expired_keys / evicted_keys 구분
⑤ INFO persistence      latest_fork_usec, rewrite 진행 여부
⑥ INFO replication      복제 지연
```

②가 필요한 이유가 있다. SLOWLOG는 **명령 실행 시간만** 잰다. 네트워크 대기, 클라이언트 출력 버퍼 지연, fork로 인한 정지는 안 잡힌다. SLOWLOG가 비어 있다고 Redis 문제가 아니라고 단정하면 안 된다.

## 흔한 오해와 헷갈리는 지점

정리하면서, 그리고 예전에 내가 틀렸던 것들을 모았다.

`io-threads`를 올리면 O(N) 명령 문제가 해결된다고 생각했다. 아니다. 병렬화되는 건 네트워크 I/O와 파싱이고 명령 실행은 단일 스레드다.

"단일 스레드라서 Redis는 느리다"는 오해도 있다. 대부분 명령이 O(1)이고 메모리에서 동작하므로 하나씩 처리해도 충분히 빠르다. 문제는 O(1)이 아닌 명령이 섞일 때다.

느린 `GET`이 로그에 찍혔으니 `GET`이 문제라고 보기 쉽다. 그 `GET`은 앞선 느린 명령을 기다린 피해자일 수 있다. 같은 시각의 다른 명령을 함께 봐야 한다.

Hash의 필드 수만 임계값 이내면 listpack이라고 생각했다. 값 크기도 조건이다. 필드 1개여도 값이 64바이트를 넘으면 전환된다. 그리고 한글은 글자당 3바이트다.

인코딩이 작아지면 다시 돌아온다고 오해했다. 단방향이다. 되돌리려면 키를 새로 만들어야 한다.

만료와 축출을 같은 것으로 보면 진단이 어긋난다. 만료는 시간, 축출은 메모리 압박이다. 지표도 `expired_keys`와 `evicted_keys`로 따로 있다.

TTL이 지나면 즉시 메모리에서 사라진다고 생각하기 쉽다. lazy든 active든 실제 삭제까지 시간차가 있다. 다만 클라이언트에게는 만료된 키가 보이지 않으니 논리적으로는 즉시 사라진 것과 같다.

replica도 알아서 만료시킨다고 오해했다. primary의 `DEL`을 기다린다. 승격 후에는 독립적으로 처리한다.

MOVED와 ASK를 같은 것으로 보면 클러스터 클라이언트 구현을 이해할 수 없다. MOVED는 슬롯맵 갱신, ASK는 이번 한 번이다.

## 요약

Redis는 단일 스레드 이벤트 루프로 명령을 하나씩 처리한다. 락이 없고 명령이 원자적인 대신, 느린 명령 하나가 전부를 막는다. 자료구조는 크기에 따라 인코딩을 자동 전환해 메모리와 조회 속도를 저울질하며, 그 전환은 단방향이다. 키를 지우는 계기는 시간(만료)과 압박(축출) 두 가지이고, 만료는 lazy와 active가 서로의 빈틈을 메운다. 저장은 fork와 COW로 서비스를 멈추지 않고 하되 메모리가 최대 2배까지 늘 수 있다. 복제는 변경을 스트리밍하며, 만료 판단은 primary에 중앙화해 일관성을 지킨다. 멀티코어 한계는 16384개 슬롯을 나누는 클러스터로 푼다.

다음으로 볼 만한 것은 클러스터의 failover 과정(replica가 어떻게 승격되는지, configEpoch가 왜 필요한지)이나 Valkey의 멀티스레드 실행이다. Redis의 단일 스레드 설계를 이해하면 왜 그런 시도가 어려운지가 보인다.

## 참고 자료

- [EXPIRE — Appendix: Redis expires](https://redis.io/docs/latest/commands/expire/) — passive/active 만료, 복제·AOF 처리, TTL이 지워지는 조건
- [Redis latency troubleshooting](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/) — 단일 스레드 특성과 진단 절차
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) — RDB·AOF 동작과 트레이드오프
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) — 복제 스트림과 오프셋
- [Redis cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) — 16384 슬롯, CRC16, MOVED/ASK, MIGRATING/IMPORTING
- [Key eviction](https://redis.io/docs/latest/develop/reference/eviction/) — maxmemory-policy별 대상 선정
- [SCAN](https://redis.io/docs/latest/commands/scan/) — 커서 기반 순회의 보장 범위

## 실측 환경

Redis 8.10.0 (arm64, Darwin), 로컬 단독 인스턴스.

문서의 수치는 이 환경에서 직접 측정한 값이다. 명령 실행 시간(`KEYS *` 42,594µs / `SCAN` 24µs / `GET` 2µs)은 키 50만 개 기준이고, 지연 비교(max 59.470ms vs 유휴 5.067ms)는 `redis-cli --latency`로 재면서 `KEYS *`를 반복 실행한 결과다. 인코딩 경계값(44바이트, 512엔트리, 64바이트)과 메모리 차이(listpack 5,815B vs hashtable 18,958B), Hash와 String 비교(122B vs 62B)도 실측이다. fork 비용은 244MB 데이터에서 `latest_fork_usec:4639`, eviction은 한도를 20MB 낮춰 39만 건이 축출된 값이다.

다만 O 표기 절의 키 500만 개 항목(약 400ms)은 50만 개 실측값에서 선형 외삽한 추정치다. 나머지는 전부 관측값이다.

프로덕션에서는 인스턴스 타입·키 크기·네트워크에 따라 절대값이 다르다. 봐야 할 것은 절대값이 아니라 배율이다.
