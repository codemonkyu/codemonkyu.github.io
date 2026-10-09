---
title: "[DNS] 내가 아는 DNS"
date: 2026-08-05 14:56:52 +0900
categories: ["Cloud & Infrastructure", "쉽게보는 IT"]
tags: ["dns", "DNS 정리", "udp", "전파"]
tistory_url: https://codemonkyu.tistory.com/entry/DNS-%EB%82%B4%EA%B0%80-%EC%95%84%EB%8A%94-DNS
---

주소창에 도메인을 치면 페이지가 열린다. 그 사이에 일어나는 일을 "DNS가 이름을 IP로 바꿔준다" 한 줄로만 알고 있었다.

언젠가는 자세히 정리해보자 했던 시점 마침 시간이 나서 DNS 에서 질의 하나가 시작해서 끝날 때까지의 순서를 정리했다.  
TCP를 정리했을 때처럼 각 단계를 직접 만든 GIF로 붙였고, 나오는 수치와 출력은 전부 이 글을 쓰면서 실제로 조회한 값이다.

다만 IP와 TTL 값은 조회 시점에 따라 달라지니, 직접 따라 해보면 숫자는 다르게 나오는 게 정상이다. 봐야 할 건 숫자가 아니라 구조다.

네트워크를 한 번쯤 훑었지만 DNS가 흐릿한 사람, 그리고 TTL이나 Alias 때문에 한 번 데어본 사람에게 맞는 글이다.

DNS(Domain Name System)는 사람이 읽는 이름을 기계가 쓰는 주소로 바꿔주는, 전 세계에 흩어진 분산 데이터베이스다.

기억할 건 세 가지다. 첫째, 아무도 전체를 알지 못하고 각자 자기 구역만 알며 나머지는 아래로 위임한다.

둘째, 그 탐색을 리졸버가 대신 해주고 결과를 캐시한다.

셋째, 그 캐시의 수명이 TTL이며 DNS의 거의 모든 실무 문제가 여기서 나온다.

---

### DNS는 어떻게 이름을 IP로 바꾸는가

IP 주소를 직접 외워서 쓴다고 생각해보자. 불편한 건 둘째 문제고, 더 큰 문제는 서버를 옮기면 주소가 바뀐다는 것이다. 이름과 주소를 분리해두면 이름은 그대로 두고 주소만 갈아끼울 수 있다. DNS의 본질은 이 분리다.

그럼 이름과 주소의 대응표를 어디에 둘까. 한 곳에 모아두면 그 서버가 죽는 순간 인터넷 전체가 멈춘다. 매일 수억 개가 바뀌는 걸 한 조직이 관리할 수도 없다. 그래서 DNS는 트리 구조로 쪼개고, 각 구역의 관리를 그 구역 주인에게 넘긴다. `.com`은 `.com` 관리자가, `example.com`은 그 도메인 소유자가 책임진다.

이 위임 구조 때문에 이름 하나를 찾으려면 트리를 따라 내려가야 한다. 그게 아래 과정이다.

브라우저가 `www.example.com`의 주소를 알아내는 전 과정이다.

![dns-recursive-lookup](/assets/img/posts/dns-deep-dive/01.gif)
_dns-recursive-lookup_

순서를 말로 풀면 이렇다.

1. 브라우저가 리졸버에게 묻는다. "www.example.com의 A 레코드 알려줘."  
2. 리졸버가 루트 네임서버에게 묻는다. 루트는 답을 모른다. 대신 "`.com`은 저 서버가 담당해"라고 알려준다.  
3. 리졸버가 `.com` TLD 서버에게 묻는다. 여기도 답은 모르고 "`example.com`은 저 서버가 담당해"라고만 한다.  
4. 리졸버가 그 권한 네임서버에게 묻는다. 여기가 실제 답을 가진 유일한 곳이다.  
5. 리졸버가 답을 캐시에 넣고 브라우저에게 돌려준다.

여기서 중요한 건 2번과 3번이 답이 아니라는 점이다. 루트도 TLD도 `www.example.com`의 IP를 모른다. 그들이 주는 건 "다음에 물어볼 곳"이고, 이걸 위임(referral)이라고 한다. 루트가 아는 건 TLD 목록뿐이다. 그래서 루트에 부하가 몰리지 않는다.

`dig +trace`로 실제 이 과정을 볼 수 있다. 내 환경에서 돌린 결과를 줄여 옮기면 이렇게 나온다.

```
$dig +trace amazon.com A

.   465162 IN NS e.root-servers.net.

.   465162 IN NS g.root-servers.net.

.   465162 IN NS m.root-servers.net.

;; Received 1097 bytes from 192.168.1.1#53(192.168.1.1) in 13 ms

com.   172800 IN NS j.gtld-servers.net.

com.   172800 IN NS k.gtld-servers.net.

com.   172800 IN NS l.gtld-servers.net.

;; Received 1170 bytes from 198.97.190.53#53(h.root-servers.net) in 45 ms

amazon.com.  172800 IN NS ns-521.awsdns-01.net.

amazon.com.  172800 IN NS ns-264.awsdns-33.com.

amazon.com.  172800 IN NS ns-1707.awsdns-21.co.uk.

amazon.com.  172800 IN NS ns-1447.awsdns-52.org.

;; Received 577 bytes from 192.31.80.30#53(d.gtld-servers.net) in 126 ms

```

응답을 준 서버가 매번 바뀌는 게 보인다. 루트에게 물었더니 `.net` 서버 목록을 주고, `.net` 서버에게 물었더니 'amazon.com`의 네임서버를 준다. 위임이 눈에 보이는 형태다.

---

### 재귀 질의와 반복 질의

위 과정에는 성격이 다른 두 종류의 질의가 섞여 있다. 이 둘을 구분하지 못하면 "재귀 리졸버"라는 이름이 왜 붙었는지 알 수 없다.

![dns-recursive-vs-iterative](/assets/img/posts/dns-deep-dive/02.gif)
_dns-recursive-vs-iterative_

브라우저가 리졸버에게 보내는 건 재귀 질의다. "답을 찾아서 가져다줘"라는 요청이고, 브라우저는 그 뒤로 중간 과정을 전혀 모른 채 결과만 기다린다. 일을 통째로 넘긴 것이다.

반면 리졸버가 루트, TLD, 권한 서버에게 보내는 건 반복 질의다. "네가 아는 데까지만 알려줘"이고, 받은 힌트로 다음 서버에 또 묻는다. 발품을 파는 쪽은 리졸버다.

헤더의 RD(Recursion Desired) 비트가 이 차이를 표시한다. 브라우저의 질의는 RD=1이고, 리졸버가 권한 서버들에게 보내는 질의는 RD=0이다. 브라우저는 1번 물었고 리졸버는 3번 물었다. 같은 하나의 이름 해석 안에서 벌어진 일이다.

이름이 헷갈리기 쉬운데, "재귀 리졸버"는 재귀 질의를 받아주는 서버라는 뜻이다. 자기가 하는 일은 오히려 반복 질의다.

---

### DNS 캐싱

매번 루트부터 내려간다면 DNS는 진작에 무너졌을 것이다. 실제로는 대부분의 질의가 트리에 닿기 전에 끝난다.

캐시가 여러 층으로 쌓여 있기 때문이다.

![DNS Cache](/assets/img/posts/dns-deep-dive/03.gif)
_DNS Cache_

위에서부터 브라우저 캐시, OS의 stub resolver 캐시, 재귀 리졸버 캐시, 그리고 권한 네임서버 순이다. 질의는 위에서부터 차례로 두드리고 맞는 곳에서 즉시 멈춘다. 맨 아래 권한 네임서버만 캐시가 아니라 원본이다.

주의할 점은 계층마다 수명과 관리 주체가 다르다는 것이다. 브라우저 캐시는 브라우저가 알아서 정하고, OS 캐시는 운영체제가, 리졸버 캐시는 그 리졸버 운영자가 관리한다. 그래서 "왜 나는 새 IP로 가는데 동료는 옛 IP로 가느냐"가 생긴다. 서로 다른 층에서 답을 받고 있는 것이다.

macOS에서 지금 어떤 리졸버를 쓰는지는 이렇게 확인한다.

```
$ scutil --dns | grep 'nameserver\[0\]'
  nameserver[0] : 192.168.1.1
```

OS 캐시를 비우고 싶으면 다음 명령을 쓴다. 다만 이건 OS 층만 지우는 것이고, 브라우저 캐시나 리졸버 캐시는 그대로 남는다.

```bash
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder
```

---

### 두 번째 요청이 빠른 이유

캐시의 효과는 같은 이름을 두 번 물어보면 바로 드러난다.

![](/assets/img/posts/dns-deep-dive/04.gif)

1회차는 리졸버 캐시가 비어 있으니 루트부터 세 번 외부 질의를 한다. 2회차는 리졸버가 캐시에서 바로 꺼내 준다. 외부 질의는 0회다.

실제로 해보면 이렇게 차이가 나온다.

```
$ dig @8.8.8.8 www.example.com A +noall +answer +stats
www.example.com. 300 IN  A   104.20.23.154
;; Query time: 47 msec

$ dig @192.168.1.1 www.rfc-editor.org A +noall +stats
;; Query time: 9 msec
```

캐시에서 온 답과 원본에서 온 답을 구분하는 방법이 있다. 헤더의 AA(Authoritative Answer) 플래그다.

권한 네임서버에 직접 물어보면 `aa`가 붙는다.

```
$ dig @hera.ns.cloudflare.com www.example.com A +noall +comments
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
```

`8.8.8.8`처럼 리졸버에 물으면 `aa`가 없다. 같은 값이라도 "원본이 방금 확인해준 답"과 "누가 아까 받아둔 답"은 다르다는 뜻이다.

---

### CNAME: "저기 가서 다시 물어봐"

이름을 물었는데 IP가 아니라 또 다른 이름이 돌아올 때가 있다. CNAME이다.

![](/assets/img/posts/dns-deep-dive/05.gif)

`www.microsoft.com`을 조회하면 CNAME이 두 번 이어진 뒤에야 A 레코드가 나온다. 실제 출력이다.

```
$ dig www.microsoft.com +noall +answer
www.microsoft.com. 3143 IN CNAME www.microsoft.com-c-3.edgekey.net.
www.microsoft.com-c-3.edgekey.net. 874 IN CNAME e13678.dscb.akamaiedge.net.
e13678.dscb.akamaiedge.net.        14 IN A     23.49.206.40
```

CNAME은 "이 이름은 저 이름의 별명이니 저기 가서 다시 물어봐"라는 뜻이다. CDN이 트래픽을 자기 인프라로 넘길 때 쓰는 전형적인 방식이다. 마이크로소프트가 Akamai로 넘기는 게 이 두 줄에 그대로 보인다.

비용도 함께 봐야 한다. 각 단계는 새로운 이름 해석이고, 캐시에 없으면 그 이름을 다시 루트부터 찾아 내려갈 수도 있다. 체인이 길면 첫 접속이 그만큼 느려진다. 맨 아래 TTL이 14초인 것도 눈에 걸린다. 이 값은 곧 만료되니 자주 다시 조회된다.

`dig www.amazon.com`도 CNAME 두 단이고, `dig docs.aws.amazon.com`은 세 단이다. 실무에서 흔한 구조다.

---

### TTL: 레코드를 바꿨는데 왜 바로 안 바뀌는가

여기가 실무에서 가장 자주 부딪히는 지점이고, 개인적으로 DNS에서 제일 중요하다고 생각하는 부분이다.

![dns-ttl-propagation](/assets/img/posts/dns-deep-dive/06.gif)
_dns-ttl-propagation_

권한 서버의 값을 바꾸는 순간 원본은 즉시 바뀐다. 하지만 이미 답을 캐시해둔 리졸버들은 자기 TTL이 끝나기 전까지 옛 값을 계속 내놓는다. 그리고 결정적으로 각 리졸버는 자기가 캐시한 시점부터 TTL을 센다.

세계 어딘가에는 방금 캐시한 리졸버가, 다른 곳에는 만료 직전인 리졸버가 있다.

그 결과가 위 자료처럼 리졸버 A / B / C 간의 TTL이 뒤죽박죽한 상태다. 같은 순간에 리졸버 C는 새 IP를, A와 B는 옛 IP를 준다. 사용자가 어느 리졸버를 쓰는지에 따라 새 서버로 가거나 옛 서버로 간다. "전파 중"이라고 부르는 상태가 이것이다.

여기서 흔히 오해하는 게 있다. 전파 지연은 DNS가 값을 서버들 사이에 밀어 보내는 시간이 아니다. DNS는 아무것도 밀어 보내지 않는다. 그냥 각 캐시의 TTL이 자기 속도로 만료되기를 기다리는 것이다. 그래서 최악의 대기 시간은 정확히 TTL 전체다.

TTL이 실제로 줄어드는 걸 눈으로 볼 수 있다. 같은 이름을 4초 간격으로 물어봤다.

```
$ dig @8.8.8.8 www.iana.org +noall +answer   # t=+0s
www.iana.org. 3387  IN  CNAME  www.iana.org.cdn.cloudflare.net.

$ dig @8.8.8.8 www.iana.org +noall +answer   # t=+4s
www.iana.org. 3383  IN  CNAME  www.iana.org.cdn.cloudflare.net.
```

3387에서 3383으로 딱 4가 줄었다. 리졸버가 새로 조회한 게 아니라 캐시에 남은 잔량을 알려주고 있다는 증거다. TTL이 0이 되면 그때 다시 원본에 물어본다.

그래서 실무 순서는 이렇게 된다. IP를 바꿀 계획이 있으면 변경하기 며칠 전에 TTL을 먼저 60초 같은 낮은 값으로 내려둔다. 기존 TTL(가령 3600초)이 만료되면서 낮은 값이 퍼지기를 기다린다. 그 다음에 실제 IP를 바꾸면 전파가 1분 안에 끝난다. 마이그레이션이 끝나면 TTL을 원래대로 올린다.

TTL을 낮게 유지하면 항상 좋을 것 같지만 그렇지 않다. 캐시 히트율이 떨어져 질의가 늘고, 리졸버 응답도 느려지고, 유료 DNS라면 질의 요금도 올라간다. 평소엔 넉넉하게, 변경 앞두고만 낮게가 실용적인 답이다.

---

### DNS 메시지의 구조

지금까지 나온 플래그(RD, AA, TC)들이 어디에 담겨 오는지 보자.

![](/assets/img/posts/dns-deep-dive/07.gif)

질의든 응답이든 같은 틀을 쓴다. Header, Question, Answer, Authority, Additional 다섯 칸이고, 어떤 칸이 채워졌는지가 메시지의 의미를 결정한다.

- Header (12바이트 고정): QR로 질의냐 응답이냐, RD로 재귀를 원하는지, AA로 권한 답인지, RCODE로 성공인지 실패인지를 표시한다  
- Question: 무엇을 물었나. 응답에도 그대로 복사되어 돌아온다  
- Answer: 실제 답  
- Authority: 답 대신 "다음 목적지"가 오는 칸. 앞에서 본 위임이 여기 담긴다  
- Additional: glue 레코드와 EDNS0가 오는 자리

여기서 구조가 하나로 이어진다. Answer는 비어 있고 Authority만 채워진 응답이 곧 위임이다. 재귀 질의 중간 단계에서 루트와 TLD가 준 게 바로 이 형태의 응답이었다.

`dig` 출력의 섹션 이름이 이 칸들과 그대로 대응한다. `;; flags:`가 Header고, `;; ANSWER SECTION:`이 Answer다. 이걸 알고 나면 `dig` 출력이 훨씬 편하게 읽힌다.

---

### UDP 512바이트와 TCP 폴백

DNS는 UDP를 쓴다고 배운다. 그런데 TCP도 쓴다. 언제 넘어가는지가 이 절의 주제다.

![](/assets/img/posts/dns-deep-dive/08.gif)

원래 DNS는 UDP 한 번의 왕복으로 끝내도록 설계됐고, 응답 크기 상한이 512바이트였다. 일반적인 A 레코드 응답은 76바이트 정도니 한참 여유가 있다.

문제는 DNSSEC처럼 서명이 붙는 경우다. 응답이 쉽게 512를 넘긴다.

그러면 서버는 답을 자르고 TC(Truncated) 플래그만 켜서 보낸다.

실제로 재현해봤다.

```
$ dig @8.8.8.8 cloudflare.com DNSKEY +dnssec +bufsize=512 +ignore +noall +comments
;; flags: qr tc rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
```

`tc`가 켜져 있고 ANSWER가 0이다. 답이 하나도 안 왔다는 뜻이다. 참고로 `+ignore` 옵션을 빼면 `dig`가 알아서 TCP로 재질의해버려서 이 상태를 볼 수 없다.

해결 방법은 두 가지다. 하나는 같은 질의를 TCP로 다시 보내는 것이다. 크기 제한이 사실상 없어지지만 연결을 맺어야 하니 왕복이 늘어난다. 다른 하나는 EDNS0로 UDP 버퍼 크기 자체를 키우는 것이다.

```
$ dig @8.8.8.8 cloudflare.com DNSKEY +dnssec +bufsize=4096 +noall +comments +stats
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; MSG SIZE  rcvd: 313
```

이번엔 ANSWER가 3개 다 왔고 `tc`도 없다. 요즘은 이쪽이 기본이라 권한 서버들이 EDNS0로 1232바이트 정도를 광고한다.

```
$ dig @hera.ns.cloudflare.com www.example.com A +noall +comments | grep EDNS
; EDNS: version: 0, flags:; udp: 1232
```

방화벽이 UDP 53만 열고 TCP 53을 막아둔 환경에서 큰 응답이 필요해지면 이름 해석이 실패한다.

DNS는 UDP만 쓴다고 알고 규칙을 짜면 이런 문제를 만난다.

---

### Route 53 Alias: apex에 CNAME을 못 쓰는 문제

마지막은 AWS 실무로 이어지는 이야기다.

도메인 루트(zone apex)에 CloudFront나 ELB를 붙이려다 막히는 경우가 흔하다.

![dns-route53-alias](/assets/img/posts/dns-deep-dive/09.gif)
_dns-route53-alias_

`example.com` 같은 zone apex에는 SOA와 NS 레코드가 반드시 있다. 존이 존재하기 위한 필수 조건이다.

그런데 CNAME의 의미는 "이 이름에 관한 건 전부 저쪽 것"이다. "전부 저쪽 것"과 "NS는 여기 것"이 동시에 참일 수 없다. 그래서 RFC 1034가 CNAME과 다른 레코드의 공존을 아예 금지한다.

규칙이 까다로운 게 아니라 의미가 모순되는 것이다.

Route 53의 Alias가 이 문제를 우회한다. Alias는 CNAME이 아니라 A(또는 AAAA) 타입으로 저장된다. 질의가 오면 Route 53이 내부에서 대상의 IP를 찾아 A 레코드처럼 응답한다. 타입이 A니까 SOA·NS와 충돌하지 않는다.

실제로 확인할 수 있다. `repost.aws`는 zone apex인데 A 레코드로 응답하면서 NS도 함께 갖고 있다.

```
$ dig repost.aws A +noall +answer
repost.aws.   60  IN  A  3.171.185.12
repost.aws.   60  IN  A  3.171.185.29

$ dig repost.aws NS +noall +answer
repost.aws.   27539  IN  NS  ns-410.awsdns-51.com.
repost.aws.   27539  IN  NS  ns-627.awsdns-14.net.
```

apex에 A와 NS가 공존한다. CNAME으로는 불가능한 조합이고, Alias이기 때문에 가능하다.

정리하면 이렇다.

| 구분 | CNAME | Route 53 Alias |<br>
|------|-------|----------------|<br>
| zone apex 사용 | 불가 | 가능 |<br>
| 저장되는 타입 | CNAME | A 또는 AAAA |<br>
| 해석 방식 | 클라이언트가 다시 질의 | Route 53이 내부에서 치환 |<br>
| 대상 | 임의의 이름 | AWS 리소스로 한정 (ELB, CloudFront, S3 등) |<br>
| 질의 요금 | 부과 | 무료 |<br>

주의할 점은 Alias가 DNS 표준이 아니라는 것이다. Route 53의 기능이고, 밖에서 보면 그냥 A 레코드다. 그래서 `dig`로는 Alias인지 아닌지 직접 구분할 수 없다. apex인데 A로 응답한다는 정황으로 짐작할 뿐이다.

---

정리

정리하면서, 그리고 예전에 내가 틀렸던 것들을 모았다.

루트 네임서버가 모든 도메인 정보를 가진 총본부라고 생각했다. 아니다. 루트가 아는 건 TLD 목록뿐이고, 실제 답은 맨 아래 권한 네임서버만 가지고 있다. 위임 구조 덕분에 루트에 부하가 몰리지 않는다.

전파 지연을 DNS가 값을 퍼뜨리는 시간으로 오해하기 쉽다. DNS는 아무것도 퍼뜨리지 않는다. 각 캐시의 TTL이 자기 속도로 만료되기를 기다리는 것뿐이다. 그래서 "전파를 빠르게 하는" 방법은 없고, "TTL을 미리 낮춰두는" 방법만 있다.

재귀 리졸버가 재귀 질의를 한다고 생각하기 쉬운데 반대다. 재귀 질의를 받아주는 서버이고, 자기가 밖으로 보내는 건 반복 질의다.

zone apex에 CNAME을 못 쓰는 걸 Route 53의 제약으로 아는 경우가 있다. DNS 표준의 제약이고, Alias는 그 제약을 A 타입으로 우회한 Route 53의 해법이다.

DNS는 UDP만 쓴다고 외우면 TC 플래그와 TCP 폴백을 만났을 때 당황한다. 방화벽에서 TCP 53을 막으면 큰 응답이 필요한 순간 이름 해석이 실패한다.

TTL을 낮게 두면 항상 유리하다고 생각했는데, 캐시 히트율이 떨어져 응답이 느려지고 질의 요금도 늘어난다. 평소엔 넉넉하게 두고 변경을 앞두고만 낮추는 게 낫다.

DNS는 아무도 전체를 알지 못하는 구조로 전 세계의 이름을 관리한다. 각자 자기 구역만 알고 나머지는 아래로 위임하며, 그 탐색을 재귀 리졸버가 대신 해준다. 리졸버는 결과를 TTL 동안 캐시하고, 그래서 대부분의 질의는 트리에 닿기 전에 끝난다. 헤더의 RD·AA·TC 플래그가 이 과정의 성격을 실어나른다. 실무 문제의 대부분은 캐시와 TTL에서 나오고, apex에 CNAME을 못 쓰는 문제는 Route 53 Alias가 A 타입으로 우회한다.

다음으로 볼 만한 것은 DNSSEC(응답이 위조되지 않았음을 서명으로 검증하는 방식, 위에서 응답 크기가 커진 원인)이나 DoH·DoT(DNS 질의 자체를 암호화하는 방식)다.

DNS가 평문 UDP라는 사실이 왜 문제가 되는지부터 보면 자연스럽게 이어진다.

## 참고 자료

- [RFC 1034 - Domain Names, Concepts and Facilities](<https://www.rfc-editor.org/rfc/rfc1034.html>) — DNS의 개념과 위임 구조, CNAME 단독 존재 규칙의 원문  
- [RFC 1035 - Domain Names, Implementation and Specification](<https://www.rfc-editor.org/rfc/rfc1035.html>) — 메시지 포맷, 512바이트 제한, TC 플래그 정의  
- [RFC 6891 - Extension Mechanisms for DNS (EDNS(0))](<https://www.rfc-editor.org/rfc/rfc6891.html>) — UDP 버퍼 크기를 확장하는 방법  
- [Route 53 - Alias 레코드와 비Alias 레코드 중에서 선택](<https://docs.aws.amazon.com/ko_kr/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html>) — Alias의 동작과 지원 대상, 요금 정책  
- [MDN - DNS](<https://developer.mozilla.org/ko/docs/Glossary/DNS>) — 짧고 쉬운 개요
