---
title: "[AWS] EC2 Spot Fleet : 비용 효율적인 클라우드 컴퓨팅 활용하기"
date: 2025-01-31 01:24:41 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["BASIC", "EC2", "spotfleet"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-EC2-Spot-Fleet-%EB%B9%84%EC%9A%A9-%ED%9A%A8%EC%9C%A8%EC%A0%81%EC%9D%B8-%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C-%EC%BB%B4%ED%93%A8%ED%8C%85-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B0
---

![](/assets/img/posts/ec2-spot-fleet/01.png)

## 1. Spot Fleet이란?

AWS Spot Fleet은 사용자가 지정한 기준에 따라 Spot 인스턴스와 온디맨드 인스턴스의 집합을 자동으로 요청하고 관리하는 서비스입니다. Spot Fleet을 사용하면 목표 용량을 설정하고, 다양한 인스턴스 유형과 가용 영역에 걸쳐 워크로드를 유연하게 실행할 수 있습니다.

### Spot Instance vs Spot Fleet

- Spot Instance: 개별 인스턴스 단위로 요청 및 관리
- Spot Fleet: 여러 인스턴스 유형과 가용 영역에 걸쳐 자동으로 관리

---

## 2. Spot Fleet의 주요 구성 요소(*옵션)

### 2-1. 할당 전략

**최저 가격(*lowestPrice)**

- 가장 낮은 가격의 인스턴스 풀에서 인스턴스를 시작
- 비용 최적화가 최우선 순위인 워크로드에 적합
- 단일 풀에 집중되어 가용성이 낮을 수 있음

**다각화(*Diversified)**

- 여러 인스턴스 풀에 걸쳐 인스턴스를 균등하게 분산
- 가용성과 유연성이 향상됨
- Spot 중단 위험을 분산시키는 효과

**용량 최적화(*CapacityOptimized)**

- 가용 용량이 가장 많은 풀을 선택
- Spot 인스턴스 중단 가능성 최소화
- 장기 실행 워크로드에 적합

**가격 용량 최적화 (*PriceCapacityOptimized)**

- 용량을 우선순위로 하고, 가격을 차선으로 고려
- 비용과 가용성의 균형을 맞춤

### 2-2. 인스턴스 가중치 부여

인스턴스 가중치를 사용하면 EC2 플릿 또는 스팟 플릿의 각 인스턴스 유형에 가중치를 할당하여 서로 상대적인 컴퓨팅 용량과 성능을 반영합니다. 가중치를 기반으로 플릿은 원하는 목표 용량을 충족할 수 있는 한 지정된 인스턴스 유형을 원하는 대로 조합하여 사용할 수 있습니다. 이는 플릿의 비용과 성능 관리에 도움이 될 수 있습니다.

[참고 문헌](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-fleet-instance-weighting.html)

### 2-3. 용량 재조정

용량 리밸런싱을 통해 EC2 플릿 또는 스팟 플릿은 중단 위험이 있는 스팟 인스턴스를 사전에 교체하여 원하는 스팟 용량을 유지할 수 있습니다. 스팟 인스턴스의 중단 위험이 높아질 때 Amazon EC2는 [재조정 권고](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/rebalance-recommendations.html)를 전송합니다. 용량 리밸런싱이 활성화된 경우 재조정 권고에 따라 위험이 있는 인스턴스가 중단되기 전에 새 스팟 인스턴스가 시작됩니다.

용량 리밸런싱을 사용하면 실행 중인 인스턴스가 Amazon EC2에 의해 중단되기 전에 미리 새 스팟 인스턴스로 플릿을 보강할 수 있으므로 워크로드 가용성을 유지하는 데 도움이 됩니다.

[참고 문헌](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-fleet-capacity-rebalance.html)

### 2-4. 용량 예약

온디맨드 용량 예약을 사용하면 특정 가용 영역의 온디맨드 인스턴스에 대해 원하는 기간만큼 컴퓨팅 용량을 예약할 수 있습니다. 먼저 온디맨드 인스턴스를 시작할 때 용량 예약을 사용하도록 **EC2 플릿**을 구성할 수 있습니다. (* Spot Fleet은 용량 예약을 사용할 수 없다.)

---

## 3. Spot Fleet (*스팟 플릿) 생성에 필요한 권한

Amazon EC2 콘솔을 사용하여 스팟 플릿을 생성하는 경우 **AWSServiceRoleForEC2SpotFleet** 및 **AWSServiceRoleForEC2Spot**이라는 서비스 연결 역할 2개와 스팟 플릿에 사용자 대신 리소스를 요청, 시작, 종료 및 태깅할 수 있는 권한을 부여하는 **aws-ec2-spot-fleet-tagging-role**이라는 역할이 생성됩니다.

### 3-1. 스팟 플릿의 사용자에게 권한 부여

**- 스팟 플릿을 생성하거나 관리하는 사용자에게 필요한 권한을 부여해야 합니다.**

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateTags",
                "ec2:RequestSpotFleet",
                "ec2:ModifySpotFleetRequest",
                "ec2:CancelSpotFleetRequests",
                "ec2:DescribeSpotFleetRequests",
                "ec2:DescribeSpotFleetInstances",
                "ec2:DescribeSpotFleetRequestHistory"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::*:role/aws-ec2-spot-fleet-tagging-role"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole",
                "iam:ListRoles",
                "iam:ListInstanceProfiles"
            ],
            "Resource": "*"
        }
    ]
}
```

### 3-2. 스팟 플릿의 서비스 연결 역할

Amazon EC2는 다른 AWS 서비스를 자동으로 호출하는 데 필요한 권한에 서비스 연결 역할을 사용합니다. 서비스 연결 역할은 AWS 서비스에 직접 연결된 고유한 유형의 IAM 역할입니다. 이러한 IAM 역할은 아래의 두 가지가 존재합니다.

**- AWSServiceRoleForEC2Spot - AWSServiceRoleForEC2SpotFleet  
(* 스팟 플릿에서 [암호화된 AMI](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/AMIEncryption.html) 또는 암호화된 Amazon EBS 스냅샷을 지정하는 경우 Amazon EC2에서 자동으로 인스턴스를 시작하려면 CMK를 사용할 권한을 AWSServiceRoleForEC2SpotFleet 역할에 부여해야 합니다.)**

---

## 4. 신규 또는 기존 스팟 플릿 요청과 이를 통해 시작되는 인스턴스 및 볼륨 태그 지정

스팟 플릿 요청과 이를 통해 시작되는 인스턴스와 볼륨을 분류 및 관리하는 데 도움이 되도록 사용자 지정 메타데이터로 이 요청에 태그를 지정할 수 있습니다. 스팟 플릿 요청을 생성할 때 또는 생성한 후 요청을 태깅할 수 있습니다. 마찬가지로 플릿에서 인스턴스와 볼륨이 시작될 때 또는 그 이후에 인스턴스와 볼륨에 태그를 할당할 수 있습니다.

플릿 요청에 태그를 지정할 때 플릿에서 시작한 인스턴스 및 볼륨에는 태그가 자동으로 지정되지 않습니다. 플릿에서 시작한 인스턴스 및 볼륨에 명시적으로 태그를 지정해야 합니다. 플릿 요청에만, 플릿에서 시작한 인스턴스에만, 플릿에서 시작한 인스턴스에 연결된 볼륨에만, 또는 모두에 태그를 할당하도록 선택할 수 있습니다.

*** 온디맨드 인스턴스에 연결된 볼륨에만 태그를 지정할 수 있습니다. 스팟 인스턴스에 연결된 볼륨에는 태그를 지정할 수 없습니다.**

### 4-1. 태그 지정

스팟 플릿 요청을 생성할 때 태깅하고 해당 스팟 플릿에서 시작할 때 인스턴스 및 볼륨을 태깅하려면 스팟 플릿 요청 구성을 다음과 같이 구성합니다.

스팟 플릿 요청 태그:

- SpotFleetRequestConfig에서 스팟 플릿 요청에 대한 태그를 지정합니다.
- ResourceType에 spot-fleet-request을 지정합니다. 다른 값을 지정하면 플릿 요청이 실패합니다.
- Tags에 대해 키-값 페어를 지정합니다. 둘 이상의 키-값 페어를 지정할 수 있습니다.

인스턴스 태그:

- LaunchSpecifications의 인스턴스에 대한 태그를 지정합니다.
- ResourceType에 instance을 지정합니다. 다른 값을 지정하면 플릿 요청이 실패합니다.
- Tags에 대해 키-값 페어를 지정합니다. 둘 이상의 키-값 페어를 지정할 수 있습니다.
- 또는 스팟 플릿 요청에서 참조되는 [시작 템플릿](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/create-launch-template.html)에서 인스턴스에 대한 태그를 지정할 수 있습니다.

---

## 5. 스팟 플릿 구성, 인스턴스, 이벤트 기록 설명

스팟 플릿 구성, 스팟 플릿의 인스턴스 및 스팟 플릿의 이벤트 기록을 설명할 수 있습니다.

(1). 스팟 플릿 설명(AWS CLI)

[describe-spot-fleet-requests](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-spot-fleet-requests.html) 명령을 사용하여 스팟 플릿 요청을 설명합니다.

```maxima
aws ec2 describe-spot-fleet-requests
```

[describe-spot-fleet-instances](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-spot-fleet-instances.html) 명령을 사용하여 지정된 스팟 플릿에 대한 스팟 인스턴스를 설명합니다.

```angelscript
aws ec2 describe-spot-fleet-instances \
    --spot-fleet-request-id sfr-73fbd2ce-aa30-494c-8788-1cee4EXAMPLE
```

[describe-spot-fleet-request-history](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-spot-fleet-request-history.html) 명령을 사용하여 지정된 스팟 플릿 요청에 대한 이벤트 기록을 설명합니다.

```angelscript
aws ec2 describe-spot-fleet-request-history \
    --spot-fleet-request-id sfr-73fbd2ce-aa30-494c-8788-1cee4EXAMPLE \
    --start-time 2015-05-18T00:00:00Z
```

---

EC2 Spot Fleet 에 대한 전반적인 기본 개념을 확인해 보았으며, 이후 해당 글에서 확인한 개념을 사용하여 Spot Fleet을 생성하고 모니터링 하는 방법에 대한 내용도 정리할 예정이다.
