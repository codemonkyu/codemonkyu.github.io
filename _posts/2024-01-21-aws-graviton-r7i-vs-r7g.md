---
title: "[AWS] Graviton 프로세서를 사용해야하는 이유 (e.g. r7i vs r7g)"
date: 2024-01-21 15:19:37 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["arm", "AWS", "Gravition", "x86", "비용"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-Graviton-%ED%94%84%EB%A1%9C%EC%84%B8%EC%84%9C%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%B4%EC%95%BC%ED%95%98%EB%8A%94-%EC%9D%B4%EC%9C%A0-eg-r7i-vs-r7g
---

![](/assets/img/posts/aws-graviton-r7i-vs-r7g/01.png)

오늘은 AWS Graviton 프로세서에 대해 알아보려고 한다.

Gravition 이란 AWS가 제공하고 있는 ARM 아키텍처의 프로세서를 이용한 프로세서 패밀리를 말한다.

[인스턴스 이름](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/instance-types.html#instance-type-names)에 아래와 같이 "g"가 들어간 경우 Graviton 프로세서를 이용한 인스턴스 유형임을 알 수 있다.

> m6g.xlarge

r7g.xlarge

..

..

그럼 Graviton 프로세서(ARM)는 다른 프로세서(x86)와 어떠한 차이점이 있을까 ?

가장 큰 차이점은 바로  아키텍처 설계 시 **"RISC, CISC"**중 어떠한 방식을 채택하였는지이다.

각각의 아키텍처 설계 방식은 아래와 같은 특징이 있다.

---

### **CISC 와 RISC 비교**

| CISC (Complex Instruction Set Computer) | RISC(Reduced Instruction Set Computer) |
| --- | --- |

| 명령어의 수 | 많다 | 적다 |
| --- | --- | --- |
| 레지스터 | 적다 | 많다 |
| 처리속도 | 느리다 | 빠르다 |
| 설계(내부구조) | 복잡하다 | 간단하다 |
| 전력소모 | 많다 | 적다 |

* [출처](https://velog.io/@kjw2298/CISC-RISC-%EA%B0%9C%EB%85%90-%EB%B0%8F-%EC%B0%A8%EC%9D%B4)

두 설계방식에 대한 내용은 나중에 번외의 포스팅으로 더 자세히 알아볼 예정이다.

만약 지금 당장 궁금하다면 아래의 글을 참고해보면 좋을 것 같다.

 [ARM vs  x86](https://www.redhat.com/ko/topics/linux/ARM-vs-x86)

[ARM과 x86의 차이점은 무엇일까요?

비슷한 속성을 지닌 ARM과 x86은 둘 다 오늘날 가장 빠른 슈퍼컴퓨터에서 사용되지만 차이점도 다양합니다. ARM과 x86를 비교합니다.

www.redhat.com](https://www.redhat.com/ko/topics/linux/ARM-vs-x86)

다시 본론으로 들어와서 Graviton 프로세서는 ARM 아키텍처 방식(RISC)을 채택하였으며, AWS는 해당 아키텍처의 장점을 살려

아래와 같이 계속하여 Gravition 1세대 부터 가장 최신의 4세대 까지 프로세서를 출시하고 있다.

[Graviton 4세대 프로세서를 탑재한 R8g 인스턴스](https://aws.amazon.com/ko/blogs/korea/join-the-preview-for-new-memory-optimized-aws-graviton4-powered-amazon-ec2-instances-r8g/)

[Amazon EC2 인스턴스(R8g) 미리 보기 – AWS Graviton4 기반 메모리 최적화 인스턴스 타입 | Amazon Web Service

차세대 Amazon Elastic Compute Cloud(Amazon EC2) 인스턴스의 미리보기를 공개합니다. 새로운 Graviton4 프로세서가 장착된 새로운 R8g 인스턴스는 기존 메모리 최적화 인스턴스보다 더 나은 가격 대비 성능을

aws.amazon.com](https://aws.amazon.com/ko/blogs/korea/join-the-preview-for-new-memory-optimized-aws-graviton4-powered-amazon-ec2-instances-r8g/)

그렇다면 실제 Graviton 프로세서는 x86 프로세서와 성능에서 어떠한 차이를 보여줄까?

실제 인텔 프로세서를 사용한 R7i 인스턴스와 Graviton 3세대 프로세서를 사용한 R7g 인스턴스의 성능 비교를 통해 차이를 살펴보자

---

### **R7i .xlarge vs R7g.xlarge 인스턴스 스펙**

[간편 스펙]

| Instance type | vCPU | Memory(GiB) | Network Bandwidth | EBS Bandwidth |
| --- | --- | --- | --- | --- |
| r7i.xlarge | 4 | 32 | 최대 12.5(Gbps) | 최대 10(Gbps) |
| r7g.xlarge | 4 | 32 | 최대 12.5(Gbps) | 최대 10(Gbps) |

* [출처](https://aws.amazon.com/ko/ec2/instance-types/)

[상세 스펙]

- R7i.xlarge

- 최대 3.2GHz의 4세대 인텔 제온 스케일러블 프로세서(Sapphire Rapids 8488C)
- 새로운 AMX(Advanced Matrix Extensions)를 통해 행렬 곱셈 연산 가속화
- DDR4에 비해 더 많은 대역폭을 제공하는 최신 DDR5 메모리

- R7g.xlarge

- * 사용자 지정 방식으로 구축된 AWS Graviton3 프로세서로 구동
- DDR4에 비해 50% 더 많은 대역폭을 제공하는 DDR5 메모리 장착

---

### **R7i .xlarge vs R7g.xlarge 인스턴스 성능**

각 인스턴스에 대한 CPU 및 Memory 성능을 테스트하기 위하여 벤치마크 도구인 "[sysbench](https://github.com/akopytov/sysbench?tab=readme-ov-file)"를 사용했다.

#### **[CPU] 테스트**

**사용한 명령어**

```bash
# sysbench --test=cpu --cpu-max-prime=20000  --num-threads=4 --max-requests=10000 --max-time=100 run
```

**[* sysbench 옵션 설명](https://manpages.debian.org/testing/sysbench/sysbench.1.en.html)**

```bash
--max-requests=N : deprecated alias for --events [0]
--max-time=N : deprecated alias for --time [0]
--num-threads=N: deprecated alias for --threads [1]
--cpu-max-prime=N upper limit for primes generator [10000]
```

=> 아래의 결과를 확인해보면 Gravation 프로세서가 CPU 테스트에서 더 나은 성능을 보여주는 것을 확인할 수 있다.

**[1]. R7i.xlarge**

```bash
CPU speed:
    events per second:  2578.06

General statistics:
    total time:                          3.8778s
    total number of events:              10000

Latency (ms):
         min:                                    1.25
         avg:                                    1.55
         max:                                    1.85
         95th percentile:                        1.58
         sum:                                15505.04

Threads fairness:
    events (avg/stddev):           2500.0000/0.00
    execution time (avg/stddev):   3.8763/0.00
```

**[2]. R7g.xlarge**

```bash
CPU speed:
    events per second:  4574.63

General statistics:
    total time:                          2.1846s
    total number of events:              10000

Latency (ms):
         min:                                    0.87
         avg:                                    0.87
         max:                                    1.11
         95th percentile:                        0.87
         sum:                                 8734.03

Threads fairness:
    events (avg/stddev):           2500.0000/1.22
    execution time (avg/stddev):   2.1835/0.00
```

---

#### **[Memory] 테스트**

**사용한 명령어**

```bash
# sysbench memory --memory-block-size=1M --memory-total-size=100G --num-threads=8 run
```

**옵션 설명**

```bash
--memory-block-size=SIZE : size of memory block for test [1K]
--memory-total-size=SIZE : total size of data to transfer [100G]
```

=> 아래의 결과를 확인해보면 Gravation 프로세서가 Memory 테스트에서 더 나은 성능을 보여주는 것을 확인할 수 있다.

**[1]. R7i.xlarge**

```bash
General statistics:
    total time:                          2.4015s
    total number of events:              102400

Latency (ms):
         min:                                    0.03
         avg:                                    0.18
         max:                                   24.11
         95th percentile:                        0.10
         sum:                                18731.26

Threads fairness:
    events (avg/stddev):           12800.0000/0.00
    execution time (avg/stddev):   2.3414/0.04
```

****[2]. R7g.xlarge****

```bash
General statistics:
    total time:                          1.2638s
    total number of events:              102400

Latency (ms):
         min:                                    0.05
         avg:                                    0.09
         max:                                   36.05
         95th percentile:                        0.05
         sum:                                 9588.51

Threads fairness:
    events (avg/stddev):           12800.0000/0.00
    execution time (avg/stddev):   1.1986/0.06
```

---

#### **[가격]**

AWS의 Graviton 프로세서는 클라우드에 최고의 가격 대비 성능을 제공하도록 설계된 프로세서 제품군으로 주로 동일한 인스턴스 패밀리에서 가장 저렴한 가격을 제공한다. 실제 R7i.xlarge 와 R7g.xlarge 인스턴스를 아래의 [AWS price calualtor](https://calculator.aws/#/) 를 사용하여 비교해보면 이를 확실하게 확인 할 수 있다.

두 인스턴스 모두 On-demand 형식으로 한달 전체에 대한 예상 사용량을 약 80% 정도로 사용한다는 가정하에 아래와 같이 비용을 비교할  수 있다.

**[1]. R7i.xlarge**

![](/assets/img/posts/aws-graviton-r7i-vs-r7g/02.png)

**[2]. R7g.xlarge**

![](/assets/img/posts/aws-graviton-r7i-vs-r7g/03.png)

결론적으로, R7g.xlarge 인스턴스를 사용 시 R7i.xlarge를 사용 대비 약 20%의 비용 절감 효과를 볼  수 있다.

---

이번 포스팅을 통해서 AWS의 Graviton 프로세서와 x86 프로세서를 지원하는 인스턴스 유형을 비교해보았다. 확실히 Graviton 프로세서는 성능적으로나 비용적으로 더 매력적인 선택지 일 수 있다. 하지만 실제 이미 x86 아키텍처 기반으로 복잡하게 설계된 서비스의 워크로드

ARM 아키텍처로 마이그레이션하는 것은 쉽지 않은 일이며, 실제 서비스 운영에서는 테스트와 같이 극적인 성능차이가 나타나지 않을 수도 있다.

때문에 Gravition 인스턴스로 실제 운영 환경을 마이그레이션 할 예정이라면 이러한 부분을 고려하여 신중히 진행해야 한다.
