---
title: "[AWS] EC2에서 루트볼륨교체 사용해보기"
date: 2023-05-06 12:28:35 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["AWS", "EC2", "RootVolume"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-EC2%EC%97%90%EC%84%9C-%EB%A3%A8%ED%8A%B8%EB%B3%BC%EB%A5%A8%EA%B5%90%EC%B2%B4-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0
---

![AWS](/assets/img/posts/ec2-replace-root-volume/01.png)
_AWS_

OS 볼륨에 문제가 있어서 인스턴스가 정상 작동하지 않는다면 매우 난감하다.

이러한 경우 AWS에서 제시하는 방법으로, 새로운 복구 인스턴스(Rescue Instance)를 생성하여 문제의 루트볼륨을 데이터 볼륨으로 마운트한 뒤 루트 볼륨의 OS로그를 분석하여 해결하는 방법이 있다.

## *  Why is my EC2 Linux instance not booting and going into emergency mode?

-> <https://repost.aws/knowledge-center/ec2-linux-emergency-mode>

인스턴스가 정상적으로 부팅이 되지 않는 현상에 대해 원인 분석을 위해서는 위의 방법을 통해 OS로그를 살펴보고, 문제점을 파악한 후 수정하는것이 옳은 방법이다. 하지만 급하게 인스턴스를 복구해야하는 경우에는 기존에 생성해놓은 AMI을 통해 아래의 방법을 통해 실행 중인 인스턴스의 루트 볼륨을 교체할 수 있다.

## * 루트 볼륨 교체

-> <https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/replace-root.html>

오늘은 위의 "루트 볼륨 교체"를 통해 Amazon linux 2 OS를 가진 인스턴스에서 Amazon linux 2023의 루트볼륨으로 교체해보는 작업을 진행해보겠다.

(  지난 3월 AWS에서 새로운 정기배포판 OS인 AL2023을 출시하였다. 이전의 AL2에 비해 보안/패지키관리자/지원하는 레포지토리 등이 매우 달라졌으니 자세한 내용을 아래의 URL을 통해서 확인해보자. )

* Amazon Linux 2023 – 클라우드에 최적화된 Linux 배포판 출시 (장기 지원 제공)

-> <https://aws.amazon.com/ko/blogs/korea/amazon-linux-2023-a-cloud-optimized-linux-distribution-with-long-term-support/>

* Comparing Amazon Linux 2 and Amazon Linux 2023

-> <https://docs.aws.amazon.com/ko_kr/linux/al2023/ug/compare-with-al2.html>

----------------

## 루트 볼륨 교체 이점

"루트 볼륨 교체" 기능을 통해 볼륨을 교체하면 아래와 같은 장점이 있다.

> Amazon EC2를 사용하면 다음을 유지하면서 실행 중인 인스턴스의 루트 Amazon EBS 볼륨을 바꿀 수 있습니다.

> * 인스턴스 스토어 볼륨에 저장된 데이터 - 인스턴스 스토어 볼륨은 루트 볼륨이 복원된 후에도 인스턴스에 연결된 상태로 유지됩니다.

> * 데이터(루트가 아닌) Amazon EBS 볼륨에 저장된 데이터 - 루트가 아닌 Amazon EBS 볼륨은 루트 볼륨이 복원된 후에도 인스턴스에 연결된 상태로 유지됩니다.

> * 네트워크 구성 — 모든 네트워크 인터페이스는 인스턴스에 연결된 상태로 유지되며 IP 주소, 식별자, 첨부 파일 ID를 유지합니다. 인스턴스를 사용할 수 있게 되면 보류 중인 모든 네트워크 트래픽이 플러시됩니다. 또한 인스턴스는 동일한 물리적 호스트에 유지되므로 퍼블릭 및 프라이빗 IP 주소와 DNS 이름을 유지합니다.

> * IAM 정책 — IAM 프로필 및 인스턴스와 연결된 정책(예: 태그 기반 정책)은 유지되고 적용됩니다.

## 루트 볼륨 교체 방법 및 고려사항

**교체 방법**

"루트 볼륨 교체" 방법은 아래와 같이 3가지 방법이 존재한다.

1. **초기 시작 상태**로 - 볼륨이 인스턴스 시작 시 초기 상태로 복원됩니다. 자세한 내용은 [시작 상태로 루트 볼륨 복원 섹션](<https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/replace-root.html#replace-launchstate)을> 참조하세요.

2. 현재 루트 볼륨과 **동일한 계보의 스냅샷**에서 - 루트 볼륨 손상 또는 게스트 운영 체제 네트워크 구성 오류와 같은 문제를 수정할 수 있습니다. 자세한 내용은 [스냅샷을 사용하여 루트 볼륨 바꾸기 섹션](<https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/replace-root.html#replace-snapshot)을> 참조하세요.

3. 인스턴스와 **키 속성이 동일한 AMI**에서 - 운영 체제 및 애플리케이션 패치 또는 업그레이드를 수행할 수 있습니다. 자세한 내용은 [AMI를 사용하여 루트 볼륨 바꾸기 섹션](<https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/replace-root.html#replace-ami)을> 참조하세요.

**고려 사항**

> 1. 인스턴스는 running 상태여야 합니다.
>
> 2. 프로세스 중에 인스턴스가 자동으로 재부팅됩니다. 재부팅 중에는 메모리(RAM)의 내용이 지워집니다. 수동 재부팅은 필요하지 않습니다.
>
> 3. 루트 볼륨이 인스턴스 스토어 볼륨이면 교체할 수 없습니다. Amazon EBS 루트 볼륨이 있는 인스턴스만 지원됩니다.
>
> 4. 모든 가상화된 인스턴스 유형과 EC2 Mac 베어 메탈 인스턴스의 루트 볼륨을 바꿀 수 있습니다. 다른 모든 베어 메탈 인스턴스 유형은 지원되지 않습니다.
>
> 5. Amazon EC2 Mac 베어 메탈 인스턴스는 루트 볼륨을 시작 상태로 복원하고 스냅샷을 사용하여 루트 볼륨을 대체할 수 있습니다. Amazon EC2 Mac 베어 메탈 인스턴스는 현재 AMI를 사용한 루트 볼륨 대체를 지원하지 않습니다.
>
> 6. 인스턴스의 이전 루트 볼륨과 동일한 계보에 속하는 모든 스냅샷을 사용할 수 있습니다.

## AMI을 이용한 루트 볼륨 교체 진행

AMI을 이용한 루트 볼륨 교체를 진행해 보겠다. 우선 Amazon linux 2 OS를 이용해 인스턴스 A를 생성해주고 Amazon linux 2023을 이용하여 인스턴스 B를 생성해준다. 이후 인스턴스 B에 대한 AMI 생성 후 해당 AMI을 사용하여 인스턴스 A의 루트볼륨을 교체해준다.

**1. Amazon Linux 2를 사용한 Instance A를 아래의 조건으로 생성해준다.**

> * 인스턴스 타입 : t2.medium  
> * 리전 : 서울 리전 ( ap-norhteast-2)  
> * VPC : 디폴트 A  
> * 서브넷 : 디폴트 A-3  
> * AMI : ami-0e52aed83baf3e36a (public)  
> * AMI 이름 : amzn2-ami-kernel-5.10-hvm-2.0.20230404.0-x86_64-gp2  
> * 루트 볼륨 : 20GiB GP2  
> * Private IP : 172.31.43.120  <---- 교체되지 않는다.  
> * Publick IP : 43.200.5.134   <---- 교체되지 않는다.

**2. Amazon Linux 2023을 사용한 Instance B를 아래의 조건으로 생성해준다.**

> * 리전 : 서울 리전 ( ap-norhteast-2)  
> * VPC : 디폴트 A  
> * 서브넷 : 디폴트 A-3  
> * AMI : ami-0e52aed83baf3e36a (public)  
> * AMI 이름 : amzn2-ami-kernel-5.10-hvm-2.0.20230404.0-x86_64-gp2  
> * 루트 볼륨 : 20GiB GP2  
> * Private IP : 172.31.43.120  <---- 교체되지 않는다.  
> * Publick IP : 43.200.5.134   <---- 교체되지 않는다.

![](/assets/img/posts/ec2-replace-root-volume/02.png)

**3. Instance B에 대한 AMI를 생성해준다.**

* 인스턴스 선택 -> 작업 -> 이미지 및 템플릿 -> AMI 생성

**4. AMI가 아래와 같이 정상적으로 생성되었는지 확인해준다. (* 최근 루트 볼륨 대체 작업의 작업 상태를 통해 확인할 수 있다.)**

![](/assets/img/posts/ec2-replace-root-volume/03.png)

**5. Instance A에서 루트 볼륨 교체 진행**

* 인스턴스 A 선택 -> 작업 -> 모니터링 및 문제해결 -> 루트 볼륨 대체 -> 복원 -> 이미지 -> AMI ID 선택

* 대체된 루트 볼륨 삭제 해제 (기존의 Instance A의 루트볼륨이 분리된 후에도 삭제되지 않도록 설정 해제를 해준다.)

**6. Instance A의 상세설명 -> 스토리지에서 확인**

-> 볼륨이 바뀌었는지 확인 !

**7. SSH 접속을 통해 확인 인스턴스 A 접속하여 볼륨이 정상적으로 바뀌었는지 확인**

- SSH 접속

> $ ssh -i "key" ec2-user@instance public IPv4 DNS

- 볼륨 용량이 변경되었는지 확인 (20GIB -> 25GIB로 변경)

> $ df -h  
> Filesystem      Size  Used Avail Use% Mounted on  
> devtmpfs        4.0M     0  4.0M   0% /dev  
> tmpfs           2.0G     0  2.0G   0% /dev/shm  
> tmpfs           782M  8.4M  774M   2% /run  
> /dev/xvda1       25G  1.7G   24G   7% /  
> tmpfs           2.0G     0  2.0G   0% /tmp  
> tmpfs           391M     0  391M   0% /run/user/1000

- OS가 변경되었는지 확인

> $ cat /etc/*release  
> Amazon Linux release 2023 (Amazon Linux)  
> NAME="Amazon Linux"  
> VERSION="2023"  
> ID="amzn"  
> ID_LIKE="fedora"  
> VERSION_ID="2023"  
> PLATFORM_ID="platform:al2023"  
> PRETTY_NAME="Amazon Linux 2023"  
> ANSI_COLOR="0;33"  
> CPE_NAME="cpe:2.3:o:amazon:amazon_linux:2023"  
> HOME_URL="https://aws.amazon.com/linux/"  
> BUG_REPORT_URL="https://github.com/amazonlinux/amazon-linux-2023"  
> SUPPORT_END="2028-03-01"  
> Amazon Linux release 2023 (Amazon Linux)

---

"루트 볼륨 대체"기능을 통해서 중단 없이 성공적으로 Instance A의 루트볼륨을 교체할 수 있었다.

해당 기능을 잘 활용하면 OS볼륨에 대한 교체, 복구 작업을 조금 더 유용하게 진행할 수 있을 것 같다.
