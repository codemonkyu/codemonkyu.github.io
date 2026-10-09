---
title: "[AWS] 새롭게 업데이트된 \"io2 Block Express\" 사용하기"
date: 2023-12-10 14:17:13 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["AWS", "CloudWatch", "ebs", "Io2 Block Express"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-%EC%83%88%EB%A1%AD%EA%B2%8C-%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8%EB%90%9C-io2-Block-Express-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0
---

![](/assets/img/posts/ebs-io2-block-express/01.png)

해당 포스팅에서는 AWS의 블록 스토리지 서비스 EBS의 **"io2 Block Express"** 유형에 대하여

업데이트된 소식이 있어서 다루어보려고 한다.

사실 "io2 Block Express" 볼륨 유형은 2021년 7월에 출시된 볼륨 유형으로, 현재 제공되는 EBS 유형 중에서

가장 높은 IOPS 및 처리량 그리고 볼륨 크기를 제공하고 있다.

(*최대 : IOPS 256,000 / 처리량 4,000 Mib/s / 크기 64TiB )

다만, 해당 유형을 사용하기 위해서는 아래와 같이 해당 볼륨 유형을 지원하는 [**특정 인스턴스 유형**](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/provisioned-iops.html#EBSVolumeTypes_piops)에서만 사용이 가능했다.

> 1. io2 Block Express 볼륨은 현재 C6a, C6in, C7a, C7g, C7gd, C7gn, C7i, Inf2, M6a, M6in, M6idn, M7a, M7g, M7gd, M7i, M7i-flex, P5, R5b, R6a, R6in, R6idn, R7a, R7g, R7gd, R7i, R7iz, Trn1, Trn1n, X2idn, X2iedn 인스턴스에서 지원됩니다.

> 2. Block Express를 지원하는 인스턴스 유형을 사용하여 io2 볼륨으로 인스턴스를 시작하려면 볼륨은 볼륨 크기와 IOPS에 관계없이 Block Express에서 자동으로 실행됩니다.

즉,  위와 같이 io2 Block Express를 지원하는 인스턴스 유형에서 io2 볼륨 유형을 장착하면 자동으로 Block Express를 유형을 사용할 수 있었는데, 이번 업데이트를 통해 이제는 특정 인스턴스 패밀리가 아닌 **"Nitro System 에서 구동 되는 인스턴스"** 에서는 모두 지원이 가능해졌다. 아래는 관련된 내용이다.

[업데이트 된 정보](https://aws.amazon.com/ko/about-aws/whats-new/2023/11/amazon-elastic-block-store-io2-block-express-nitro/)

[Amazon Elastic Block Store, 모든 EC2 Nitro 인스턴스에서 io2 Block Express 볼륨 사용 가능

오늘 Amazon Elastic Block Store(EBS)는 Nitro 시스템에 구축된 모든 EC2 인스턴스에서 io2 Block Express 볼륨을 사용할 수 있다고 발표했습니다. EC2 Nitro 인스턴스에서 사용되는 모든 새로운 io2 볼륨은 밀리초

aws.amazon.com](https://aws.amazon.com/ko/about-aws/whats-new/2023/11/amazon-elastic-block-store-io2-block-express-nitro/)

---

그렇다면, 실제 Nitro 시스템을 지원하는 인스턴스 패밀리에서 "Io2 Block Express" 볼륨 유형이

사용 가능한지 테스트를 통해 확인해보자

## 테스트 목적

테스트를 통해 기존의 io2 Block Express를 지원하는 인스턴스 패밀리이외에 Nitro System에서 동작하는 인스턴스를 생성하고 "Io2 Block Express" 가 제공하는 IOPS 및 처리량 을 벤치마크를 통해서 실제 달성 가능한지 확인해본다.

**인스턴스 유형 결정**

테스트에 적합한 인스턴스 유형으로 "c5.18xlarge"를 선정하였다. 이유로는 해당 유형은 Nitro 시스템에서 구동되며, 아래와 같이 io2 Block Express가 제공하는 IOPS/처리량을 테스트할 수 있는  **"EBS 전용 대역폭"**이 보장 되기 때문이다. 볼륨에 프로비저닝 된  IOPS 및 처리량을 모두 사용하기 위해서는 볼륨이 장착된 인스턴스의 유형이 얼마만큼의 IOPS 및 처리량에 대한 대역폭을 갖고 있는지 확인하는 것이 중요하다.

["Amazon EBS 최적화 인스턴스"](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ebs-optimized.html#ebs-optimization-support)

[Amazon EBS 최적화 인스턴스 - Amazon Elastic Compute Cloud

인스턴스의 EBS 성능은 인스턴스의 성능 제한 또는 연결된 볼륨의 집계된 성능 중 작은 쪽의 제한을 받습니다. 최대한의 EBS 성능을 달성하려기 위해서는 인스턴스에 최대 인스턴스 성능과 같거

docs.aws.amazon.com](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ebs-optimized.html#ebs-optimization-support)

**"c5.18xlarge"**

![](/assets/img/posts/ebs-io2-block-express/02.png)

![](/assets/img/posts/ebs-io2-block-express/03.png)

해당 인스턴스 유형은 "io2 Block Express"가 제공하는 최대 IOPS 및 처리량 (각각 256,000 / 4,000 Mib/s)를 전부 사용할 수 있는 전용 대역폭을 갖고 있지는 않지만, io2 유형이 제공하는 최대 IOPS 및 처리량 (각각 64,000 / 1,000 Mib/s) 이상의 대역폭을 지원하기 때문에 해당 유형을 통해 충분히 Block Express의 유형을 제공하는지 확인이 가능하다.

**볼륨 설정**

C5.18xlarge 인스턴스 유형을 생성한 후, 아래와 같이 io2 Volume 유형을 생성하여 볼륨 용량 및 IOPS를 설정해준다. 현재 io2 볼륨 유형에서 프로비저닝 가능한 IOPS 및 볼륨 크기의 비율은 1000:1 이다. 볼륨 크기별 프로비저닝 가능한 IOPS의 비율은 아래의 공식 문서에서 확인이 가능하다.

["볼륨 성능"](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/provisioned-iops.html#EBSVolumeTypes_piops)

- IOPS: GiB 비율이 1,000:1인 최대 256,000의 프로비저닝된 IOPS. 최대 IOPS는 크기가 256GiB 이상인 볼륨으로 프로비저닝될 수 있습니다(1,000 IOPS x 256GiB = 256,000 IOPS).
- 요청된 볼륨 크기(GiB)에 대한 프로비저닝된 IOPS의 최대 비율은 io1 볼륨의 경우 50:1이고 io2 볼륨의 경우 500:1입니다.

[Provisioned IOPS SSD 볼륨 - Amazon Elastic Compute Cloud

io2 Block Express 볼륨은 C6a, C6in, C7a, C7g, C7gd, C7gn, C7i, Inf2, M6a, M6in, M6idn, M7a, M7g, M7gd, M7i, M7i-flex, P5, R5b, R6a, R6in, R6idn, R7a, R7g, R7gd, R7i, R7iz, Trn1, Trn1n, X2idn, X2iedn 인스턴스에서 지원됩니다.

docs.aws.amazon.com](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/provisioned-iops.html#EBSVolumeTypes_piops)

![](/assets/img/posts/ebs-io2-block-express/04.png)

위에서 확인해야할 부분은 바로 io2 볼륨에 대하여 1000:1 비율( 볼륨 크기 80GiB / IOPS 80,000)로 IOPS를 프로비저닝 가능한 부분과 해당 볼륨을 Nitro System 기반의 인스턴스에 장착해야만 프로비저닝한 IOPS를 전부 이용할 수 있음을 나타내는 문구이다. 이를 통해서 간접적으로 이제 Nitro 시스템에 구축된 인스턴스에서는 Io2 Block Express를 사용가능함을 알 수 있다. (* 볼륨은 장착할 인스턴스와 동일한 가용영역에 위치해 있어야 한다.)

### 볼륨 장착 및 성능 테스트

볼륨을 생성한 인스턴스에 장착하고 볼륨의 성능 테스트를 위해 "fio" 도구를 설치합니다.

```text
(1). "fio" 도구 설치
# yum install fio -y

(2). Volume mount
# mkdir /mnt1
# mkfs.xfs /dev/nvme1n1
# mount /dev/nvme1n1 /mnt1

(3). Check mount
# df -h
/dev/nvme1n1 80G 604M 80G 1% /mnt1

# lsblk -f
NAME                 FSTYPE               LABEL                UUID                            MOUNTPOINT
nvme1n1                  xfs                   9da100b2-9a8e-4b7e-9a79-ce3897ef299d                 /mnt1
```

볼륨이 정상적으로 마운트 되었으면 fio 도구를 사용하여 볼륨에 대한 i/o 부하를 발생시켜 벤치마크를 진행한다. 우선 볼륨에 설정된 80,000 IOPS를 사용하도록 아래의 명령어를 실행 시킨다.

> (1). 80,000 IOPS 사용  
>  
> $ fio --name=randread --ioengine=libaio --iodepth=16 --rw=randread --bs=4k --direct=1 --size=500m --numjobs=64 --runtime=300 --time_based --ramp_time=20 --rate_iops=80000 --directory=/mnt1

위 명령어의 파라미터는 다음과 같은 의미를 나타낸다. 더 많은 정보는 fio 명령어의 [Man page](https://linux.die.net/man/1/fio)를 확인해보면 좋다.

- --name: 임의의 작업 이름을 지정합니다.
- --ioengine: 입출력 엔진을 지정합니다. libaio는 Linux의 비동기 I/O 시스템을 사용합니다.
- --iodepth: I/O 큐의 깊이를 지정합니다. 높은 값은 더 많은 I/O 작업이 동시에 발생하도록 합니다.
- --rw: 입출력 패턴을 지정합니다. randread는 랜덤 읽기 작업을 의미합니다.
- --bs: 블록 크기를 지정합니다. 4k는 4KB를 의미합니다.
- --direct: Direct I/O를 사용하도록 설정합니다. 1은 활성화를 의미합니다.
- --size: 테스트 파일의 크기를 지정합니다.
- --numjobs: 동시에 실행할 작업의 개수를 지정합니다.
- --runtime: 테스트를 실행할 시간을 지정합니다. 단위는 초입니다.
- --time_based: 이 옵션을 사용하면 --runtime 동안 계속 테스트를 실행합니다.
- --ramp_time: 테스트 시작 전에 준비 시간을 지정합니다. 단위는 초입니다.
- --rate_iops: 초당 입출력 작업 수를 제한합니다.

해당 명령어를 통해 아래와 같이 "fio" 명령어 결과를 통해 사용된 전체 IOPS를 확인 가능하지만, CloudWatch를 이용한다면 조금 더 가시적인 그래프를 통해서 IOPS 및 처리량을 확인이 가능하다.

CloudWatch 에서 볼륨에 대한 Average Total IOPS 및 Total Throughput 을 구하기 위해서는 기본으로 AWS에서 제공하는 "EBS에 대한 CloudWatch 지표"를 사용하여 아래와 같은 수식을 구성하여 확인할 수 있다.

["CloudWatch 지표를 사용하여 EBS 볼륨이 제공하는 평균 처리량과 평균 IOPS 수를 계산하려면 어떻게 해야 하나요?"](https://repost.aws/ko/knowledge-center/ebs-cloudwatch-metrics-throughput-iops)

[CloudWatch 지표를 사용하여 EBS 볼륨의 평균 처리량과 IOPS 계산

Amazon Elastic Block Store(Amazon EBS) 볼륨의 평균 처리량과 IOPS 수를 계산하려고 합니다.

repost.aws](https://repost.aws/ko/knowledge-center/ebs-cloudwatch-metrics-throughput-iops)
> (1). Average Total IOPS (평균 전체 IOPS)  
> (VolumeReadOps)+(VolumeWriteOps))/(Period)  
>  
> (2). Average Total Throughput (평균 전체 처리량)  
> ((VolumeReadBytes)+(VolumeWriteBytes))/(Period)

실제로 확인한 결과 Total IOPS는 최대 80,000 까지 사용되었음을 확인할 수 있었다.

![](/assets/img/posts/ebs-io2-block-express/05.png)

최대 IOPS가 정상적으로 80,000에 도달하는것을 확인하였다. 그렇다면 최대 처리량은 정상적으로 4,000 Mib/s을 도달 가능한지 확인해보자. AWS에서 제공하는 아래의 그래프를 확인해보면 Io2 Block Express는 최소 16,000 IOPS 를 프로비저닝 했을때 부터 최대 처리량 4,000 Mib/s에 도달 가능하다. 이는 단순 계산으로도 확인이 가능한데, 처리량은 I/O Block Size x IOPS 으로 0.256 KiB x 16,000 IOPS를 계산하면 4,096 Mib/s 임을 확인 가능하다.  그렇다면 이론적으로 현재 해당 볼륨에 프로비저닝 된 IOPS는 80,000으로 블록 사이즈가 256 KiB 일때 충분히 최대 처리량 4,000MiB/s에 도달할 수 있어야 한다.

![](/assets/img/posts/ebs-io2-block-express/06.png)

최대 처리량을 테스트 하기 위하여 아래의 fio 명령어를 사용한다.

> (1). 최대 처리량 발생 명령어  
> $ fio --name=seqread --ioengine=libaio --iodepth=16 --rw=read --bs=256k --direct=1 --size=500m --numjobs=64 --runtime=300 --time_based --ramp_time=20 --rate=4000m --directory=/mnt1

이후 CloudWatch를 통해 처리량을 확인해본다.

![](/assets/img/posts/ebs-io2-block-express/07.png)

처리량이 2265 MiB/s 으로 고정된것을 확인할 수 있다. 따라서 IOPS도 9,061  정도에서 그 이상으로 활용하지 못하고 있는것을 확인할 수 있다. 그렇다면 왜 이런 현상이 발생하는 걸까? 정답은 바로 볼륨이 장착된 인스턴스 유형이 제공하는 대역폭에 있다.

처리량 = IOPS * I/O Size

위에서 "c5.18xlarge" 는 최대 2375 MiB/s 처리량을 제공한다는 것을 알 수 있다. 따라서 현재와 같은 경우 볼륨은 충분한 성능이 프로비비저닝 되어 있지만 인스턴스 유형이 이에 대한 충분한 "EBS 전용 대역폭"을 갖고 있지 않기 때문에 성능에 제한이 발생한것을 확인할 수 있다.

---

이번 포스팅을 통해 최근 업데이트 된 "io2 Block Express"에 업데이트된 내용을 직접 확인해보고, EBS 볼륨에 프로비저닝 된 IOPS 및 처리량을 모두 사용하기 위해서는 EC2 인스턴스의 유형별 IOPS 및 처리량에 대한 전용 대역폭 또한 중요한 요소임을 확인해보았다.
