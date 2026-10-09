---
title: "[Network] & [AWS] EC2 인스턴스의 네트워크 처리량 벤치 마킹하기"
date: 2023-08-23 14:03:28 +0900
categories: ["Cloud & Infrastructure", "Network & Server"]
tags: ["AWS", "EC2", "iperf3", "Network Throughput"]
tistory_url: https://codemonkyu.tistory.com/entry/Network-AWS-EC2-%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%9D%98-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%B2%A4%EC%B9%98-%EB%A7%88%ED%82%B9%ED%95%98%EA%B8%B0
---

![네트워크 이미지 1](/assets/img/posts/ec2-network-throughput-benchmark/01.png)
_네트워크 이미지 1_

---

## Contents

1. 벤치마크의 목적  
2. 벤치마크 Tool - iperf3  
3. 네트워크 테스트 환경 구현  
4. 네트워크 테스트 진행  
5. 결과 분석

Amazon EC2 인스턴스는 유형별 상이한 네트워크 처리량(throughput)를 지원한다.  
이번 포스팅에서는 **"동일한 AWS VPC에 위차한 인스턴스간의 네트워크 처리량"** 벤치마크를 진행하고 이를 분석해보려고 한다.  
먼저 목차를 먼저 설명하고 진행해보겠다.

---

### 1.  벤치마크의 목적

동일한 VPC내에서 인스턴스간의 네트워크 워크로드를 구성할때 "인스턴스에서 지원되는 네트워크 성능"을 확인하는 단계에서 해당 벤치마크 방법을 활용할 수 있다. 또한 동일한 VPC내에서의 네트워크 성능과 Internet Gateway 등을 통한 서로 다른 VPC 에서의 성능 평가에 도움이 될 수 있다.

---

### 2.  iperf3 란?

*** [iperf3](https://docs.oracle.com/cd/E88353_01/html/E37839/iperf3-1.html)**

[iperf3 - man pages section 1: User Commands

iperf3 (1) Name iperf3 - perform network throughput tests Synopsis iperf3 -s [ options ] iperf3 -c server [ options ] Description IPERF3(1) User Manuals IPERF3(1) NAME iperf3 - perform network throughput tests SYNOPSIS iperf3 -s [ options ] iperf3 -c serve

docs.oracle.com](https://docs.oracle.com/cd/E88353_01/html/E37839/iperf3-1.html)
> - iperf3 is a tool for performing network throughput measurements. It can test TCP, UDP, or SCTP throughput.  
> - Iperf는 네트워크 성능 측정 및 튜닝 도구이다. 크로스 플랫폼 도구이며 모든 네트워크의 표준화된 성능 측정을 생성할 수 있다.

---

### 3. 네트워크 처리량 테스트 환경 구성

네트워크 성능 테스트를 진행하기 위해서는 **client - Sever** 역할을 위한 인스턴스 2대가 필요하다. 따라서 같은 VPC 내에 서브넷 A,B에 각각 1대의 인스턴스 A,B를 생성하여 테스트 환경을 구성한다.

#### 1. TEST 환경 A 를 아래와 같이 구성합니다.

**[TEST 환경 A]**

- **Test Instance A (Client)**

1. VPC : TEST VPC
2. Subnet : Public subnet A
3. Instnance type : c5.9xlarge

- vCPU : 36  
Memory (GiB) : 72  
Network Bandwidth (Gbps)*** : 10(Gbps)  
EBS Bandwidth (Mbps) : 9,500

    4. Security Group

- iperf3 는 기본적으로 port 5201을 사용하기 때문에 사용할 프로토콜 (TCP/UDP)에 대한 inbound rule을 적용시켜준다.

---

#### 2. TEST 환경 B 를 아래와 같이 구성합니다.

**[TEST 환경 B]**

- **Test Instance B (Server)**

1. VPC : TEST VPC
2. Subnet : Public subnet B
3. Instnance type : c5.9xlarge

- vCPU : 36  
Memory (GiB) : 72  
Network Bandwidth (Gbps)*** : 10(Gbps)  
EBS Bandwidth (Mbps) : 9,500

    4. Security Group

- iperf3 는 기본적으로 port 5201을 사용하기 때문에 사용할 프로토콜 (TCP/UDP)에 대한 inbound rule을 적용시켜준다.

---

### 4. 네트워크 테스트 진행

#### 1. Test Steps(1). Test Instance A & B에 접속하여 iperf3을 설치합니다.

> $ sudo yum install iperf3 -y

**(2). Test Instance B (Server) 에 접속하여 iperf3를 서버 모드로 실행해줍니다.**

> $ iperf3 -s

- 기본 포트는 5201 입니다.
- iperf3 --help 명령어를 통해 Server 옵션을 확인하고 필요하다면 추가합니다.

**(3). Test Instance A (Client) 에 접속하여 iperf3를 사용하여 서버 인스턴스에 테스트 패킷을 전송합니다.**

> $ iperf3 -c "test B instance private IP"

> e.g. $ iperf -c 10.0.0.x

**(4). 결과**

![iperf3 출력 결과1](/assets/img/posts/ec2-network-throughput-benchmark/02.png)
_iperf3 출력 결과1_

- **ID** is the ID of the iperf operation
- **Interval** is the time span the row refers to
- **Transfer** is the amount of data exchanged between client and server. In the end, speed test is about transferring files and measuring how long it took.
- **Bandwidth** is the measured bandwidth

위 기록은 Instance A에서 B로 iperf3을 사용하여 네트워크 성능을 테스트하여 나온 결과이다. 실제 전송한 데이터의 양은 5.78GBytes 이며 해당 테스트를 통해서 전송된 대역폭은 4.96 Gbits/sec 이다. 해당 서버 및 클라이언트 인스턴스는 "c5.9xlarge"의 **[네트워크 대역폭](https://aws.amazon.com/ko/ec2/instance-types/#Compute_Optimized)**은 **"10(Gbps)"** 임을 위해서 확인했다. 왜 AWS에서 보장하는 네트워크 대역폭이 테스트로 확인되어지지 않은것일까? 의문은 아래의 병렬 연결 옵션을 통해 해소될 수 있었다.

#### 2. 네트워크 대역폭 리밋 확인을 위한 -P 옵션 사용하기

> $ iperf3 -c 10.0.0.x -P 2  
> -P : --parallel # number of parallel client streams to run

위의 명령어를 통해 2개의 병렬 네트워크 커넥션을 생성한 뒤 테스트해 보았다.

![결과2](/assets/img/posts/ec2-network-throughput-benchmark/03.png)
_결과2_

결과는 9.93 Gbits/sec. 이렇듯 -P 옵션을 사용하면 **Network Bandwidth를 최대로** 사용하는것을 확인할 수 있다.

---

### 5. 다른 리전에 위치한 인스턴스간의 네트워크 대역폭 확인하기

AWS 공식 문서를 확인해보면, 다른 리전 및 인터넷 게이트웨이등을 통한 인스턴스간의 네트워크 대역폭은 아래와 같이 설정된다는 것을 알 수 있다.

> 다른 리전, 인터넷 게이트웨이, Direct Connect 또는 로컬 게이트웨이(LGW)로 - 최소 32개의 vCPU가 탑재된 현재 세대 인스턴스에 사용 가능한 네트워크 대역폭의 최대 50%를 트래픽에 활용할 수 있습니다. vCPU가 32개 미만인 현재 세대 인스턴스의 대역폭은 5Gbps로 제한됩니다.

따라서 현재 구성한 Test A와 B 인스턴스간에 인터넷 게이트 웨이를 통한 네트워크 성능 테스트를 통해 이를 증명해보자. 방법은 다른 리전의 VPC 끼리 통신을 또는 인터넷 게이트 웨이를 통한 통신 등으로 테스트할 수 있으며, 테스트의 간편함을 위해 위에 구성해놓은 테스트 인프라를 사용할 수 있는 "인터넷 게이트 웨이를 통한 통신" 방법을 사용해보자.

테스트 방법은 간단하다. 사설 IP가 아닌 공인 IP를 위해 Client에서 Server로 iperf3를 이용한 패킷을 보내면 된다. 방법은 아래와 같다.

#### Test Steps

**(1). Test Instance B (Server) 에 접속하여 iperf3를 서버 모드로 실행해줍니다.**

> $ iperf3 -s

**(2). Test Instance A (Client) 에 접속하여 iperf3를 사용하여 서버 인스턴스에 테스트 패킷을 전송합니다.**

> $ iperf3 -c "test B instance public IP"  
> $ iperf -c 3.34.138.xxx -P 5

![결과 3](/assets/img/posts/ec2-network-throughput-benchmark/04.png)
_결과 3_

**결과는 5.74 Gibts/sec 이다.** 테스트 인스턴스의 타입인 "c5.9xlarge" 는 36개의 vCPU를 제공하는 타입으로써 **최대 트래픽 10gbp의 50% 정도의 대역폭이 보장 되었음을 확인 할 수 있다.**

VPC에 대한 네트워크 대역폭이 궁금하다면 [해당 문서](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)를 참고하는 것을 권장한다.

---

이번 테스트를 통해 EC 인스턴스의 네트워크 성능을 올바르게 측정할 수 있는 방법을 알아 보았다. 성능을 테스트하고 나서 기대하는 성능이 나와주지 않는다면 아래의 **["Amazon EC2 네트워크 성능에 영향을 미칠 수 있는 몇 가지 요인"](https://repost.aws/ko/knowledge-center/network-throughput-benchmark-linux-ec2)**등을 확인해보는 것도 방법이다.

> **1. EC2 인스턴스의 물리적 근접성. 동일한 가용 영역에 있는 인스턴스는 서로 지리적으로 가장 가까운 반면, 동일한 리전의 다른 가용 영역에 있는 인스턴스, 동일한 대륙의 다른 리전에 있는 인스턴스, 다른 대륙의 다른 리전에 있는 인스턴스는 나열된 순서대로 서로 더 멀리 떨어져 있습니다.**

> **2. EC2 인스턴스 최대 전송 단위(MTU). 네트워크 연결의 MTU는 연결을 통해 전달할 수 있는 최대 허용 패킷 크기(바이트)입니다. 모든 EC2 인스턴스 유형은 1,500 MTU를 지원합니다. 모든 최신 세대 Amazon EC2 인스턴스는 점보 프레임을 지원합니다. 또한 이전 세대 인스턴스인 C3, G2, I2, M3 및 R3에서도 점보 프레임을 사용합니다. 점보 프레임은 1,500 MTU를 초과할 수 있습니다. 하지만 점보 프레임을 사용하는 경우에도 인스턴스가 1,500 MTU로 제한되는 시나리오가 있습니다. 자세한 내용은 점보 프레임(9001 MTU)을 참조하세요.**

> **3. EC2 인스턴스의 크기. 한 인스턴스 유형에 대해 인스턴스 크기가 더 크면 일반적으로 동일한 유형의 더 작은 인스턴스 크기보다 더 좋은 네트워크 성능을 제공합니다. 자세한 내용은 Amazon EC2 인스턴스 유형을 참조하십시오.**

> **4. Linux에 대한 Amazon EC2 확장 네트워킹 지원(T2 및 M3 인스턴스 유형 제외). 자세한 내용은 Linux에서의 향상된 네트워킹을 참조하십시오. 인스턴스에서 향상된 네트워킹을 활성화하는 방법에 대한 자세한 내용은 EC2 인스턴스에서 향상된 네트워킹을 활성화하고 구성하려면 어떻게 해야 합니까?를 참조하십시오.**

> **5. 배치 그룹을 사용한 Amazon EC2 HPC(고성능 컴퓨팅) 지원. HPC는 전체 바이섹션 대역폭과 낮은 지연 시간을 제공하며 인스턴스 유형에 따라 최대 100기가비트의 네트워크 속도를 지원합니다. 각 인스턴스 유형의 네트워크 성능을 확인하려면 Amazon Linux AMI 인스턴스 유형 표를 참조하세요. 자세한 내용은 배치 그룹으로 인스턴스 시작을 참조하세요.**

> **6. 인스턴스는 네트워크 I/O 크레딧 메커니즘을 사용하여 네트워크 대역폭을 할당합니다. 범용 인스턴스 - 네트워크 성능의 네트워크 성능 열에 † 기호가 있는 인스턴스는 지정된 최대 네트워크 성능에 도달할 수 있습니다. 그러나 이러한 인스턴스는 네트워크 I/O 크레딧 메커니즘을 사용하여 평균 대역폭 사용률을 기준으로 인스턴스에 대역폭을 할당합니다. 따라서 네트워크 성능은 이러한 인스턴스에 따라 다릅니다.**
