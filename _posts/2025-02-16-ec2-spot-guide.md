---
title: "[AWS] Amazon EC2 Spot 완벽 가이드"
date: 2025-02-16 20:06:33 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["asg", "EC2", "instance", "spot", "가이드", "요청 상태 확인", "할당 전략"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-Amazon-EC2-Spot-%EC%99%84%EB%B2%BD-%EA%B0%80%EC%9D%B4%EB%93%9C
---

![](/assets/img/posts/ec2-spot-guide/01.png)

금일 포스팅을 통해서 EC2 Spot 인스턴스의 작동 방식, 생성, 관리 및 모니터링 등에 대한 내용을 정리해 보려고 한다.  
해당 포스팅을 통해서 Spot 인스턴스에 대한 자세한 사용 방법을 공유 한다.

---

## 1. EC2 인스턴스 가격 옵션

- **On-Demand**: 언제든지 사용가능한 대신 사용한 만큼의 시간당 요금 지불
- **Reserved**: 할인된 시간당 요금과 용량 예약(선택 사항)을 제공
- **Spot**: EC2 여유 용량을 최대 90% 할인된 가격으로 사용

---

## 2. Spot 인스턴스 작동 방식

**기본 작동 원리**

- Spot 인스턴스는 AWS의 미사용 EC2 용량을 활용하는 방식으로 작동합니다. AWS가 가진 EC2 인스턴스의 여유 가용 용량은 실시간으로 변동하며, 이에 따라 Spot 가격도 동적으로 조정됩니다.

**스팟 인스턴스 중단 요소 - 용량 -**

Amazon EC2는 스팟 인스턴스가 다시 필요할 때 스팟 인스턴스를 중단할 수 있습니다. EC2는 주로 용량을 재활용하기 위해 인스턴스를 회수하지만 호스트 유지 관리 또는 하드웨어 중단과 같은 다른 이유로 회수할 수도 있습니다.

**- 가격 -**

스팟 가격이 최고 가격보다 높습니다.

스팟 요청에서 최고 가격을 지정할 수 있습니다. 그러나 최고 가격을 지정하면 지정하지 않을 때보다 인스턴스가 더 자주 중단됩니다.

**- 제약 조건 -**

요청에 시작 그룹 또는 가용 영역 그룹과 같은 제약 조건이 포함되는 경우 제약 조건을 더 이상 충족할 수 없으면 스팟 인스턴스가 그룹으로 종료됩니다.

**스팟 인스턴스 중단 동작**

- 스팟 요청을 생성할 때 아래의 중단 동작 중 하나를 지정할 수 있습니다. 기본 동작은 "Terminate" 입니다.

- [Stop](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/interruption-behavior.html#stop-spot-instances)
- [최대 절전 모드](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/interruption-behavior.html#hibernate-spot-instances)
- [Terminate](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/interruption-behavior.html#terminate-interrupted-spot-instances)

- 중지된 스팟 인스턴스에 대하여 Amazon EC2 인스턴스는 스팟 인스턴스를 다시 시작 할 수 있습니다. 다만 스팟 인스턴스는 요청 유형이 "Persistent" 여야 하며, EC2 플릿 및 스팟 플릿인 경우 요청 유형이 "maintain" 이어야 합니다.

- Amazon EC2만 중단된 중지된 스팟 인스턴스를 다시 시작할 수 있습니다.
- persistent 스팟 인스턴스 요청에 의해 시작된 스팟 인스턴스의 경우 Amazon EC2는 용량이 동일한 가용 영역에서 사용 가능한 경우 중지된 인스턴스와 동일한 인스턴스 유형에 대해 중지된 인스턴스를 다시 시작합니다(동일한 시작 사양을 사용해야 함).
- 스팟 인스턴스가 중지되었을 때 일부 인스턴스 속성을 수정할 수 있지만 인스턴스 유형은 수정할 수 없습니다. EBS 볼륨을 분리하거나 삭제한 경우에는 스팟 인스턴스를 시작해도 연결되지 않습니다. 루트 볼륨을 분리하고 Amazon EC2가 스팟 인스턴스를 시작하려고 하면 인스턴스가 시작되지 않고 Amazon EC2가 중지된 인스턴스를 종료합니다.
- 중지된 상태의 스팟 인스턴스를 종료할 수 있습니다.
- 스팟 인스턴스 요청이나 EC2 플릿 또는 스팟 플릿을 취소하면 Amazon EC2에서는 중지된 상태의 연결된 스팟 인스턴스를 모두 종료합니다.
- **중단된 스팟 인스턴스가 중지 상태인 동안에는 유지 중인 EBS 볼륨에 대한 요금만 부과됩니다.** EC2 플릿 및 스팟 플릿을 사용하는 경우 중지된 인스턴스가 많으면 해당 계정의 EBS 볼륨 수 제한을 초과할 수 있습니다.

---

## 3. Spot 인스턴스 생성하기

### 3-1. Spot Fleet 생성하기

(a). EC2 인스턴스 콘솔 접속 후 **"스팟 요청"** 탭으로 이동 후 **"스팟 플릿 요청 생성"**

![1](/assets/img/posts/ec2-spot-guide/02.png)
_1_

(b). Spot Fleet 요청 시 필요한 **"파라미터"**를 결정합니다.  
**- 인스턴스 유형 요구 사항**

![인스턴스 유형 요구 - 컴퓨팅 요구 사항](/assets/img/posts/ec2-spot-guide/03.png)
_인스턴스 유형 요구 - 컴퓨팅 요구 사항_

![인스턴스 유형 요구 - 인스턴스 유형](/assets/img/posts/ec2-spot-guide/04.png)
_인스턴스 유형 요구 - 인스턴스 유형_

**- 목표 용량**

![목표 용량](/assets/img/posts/ec2-spot-guide/05.png)
_목표 용량_

**- 할당방식**

![스팟 할당 방식](/assets/img/posts/ec2-spot-guide/06.png)
_스팟 할당 방식_

[할당 방식 참고 문서](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html#ec2-fleet-allocation-strategies-for-spot-instances)

**가격 용량 최적화(권장)**  
- 플릿은 시작하는 인스턴스의 수에 맞추어 용량 가용성이 가장 높은 풀을 가져옵니다. 즉, 가까운 시일 내에 중단될 가능성이 가장 낮다고 판단되는 풀에서 스팟 인스턴스를 요청합니다. 그러면 플릿이 해당 풀에서 가장 가격이 낮은 스팟 인스턴스를 요청합니다.

**용량 최적화**

플릿은 시작하는 인스턴스의 수에 맞추어 용량 가용성이 가장 높은 풀을 가져옵니다. 즉, 가까운 시일 내에 중단될 가능성이 가장 낮다고 판단되는 풀에서 스팟 인스턴스를 요청합니다. 플릿의 각 인스턴스 유형에 대해 우선순위를 설정할 수 있습니다. 이 경우 플릿은 용량을 먼저 최적화하지만 최선의 노력을 기준으로 인스턴스 유형 우선순위를 고려합니다.

**다각화**

스팟 인스턴스는 모든 스팟 용량 풀에 걸쳐 분산됩니다. AWS CLI를 사용하는 경우 파라미터 이름은 EC2 플릿과 스팟 플릿 모두에서 diversified입니다.

---

### 3-2. AutoScalingGroup 에서 Spot 인스턴스 생성하기

(a). ASG 생성에 사용 할 **"시작 템플릿"**을 생성합니다.

- 시작 템플릿 생성 시 "**고급 세부 정보"** 에서 **"구매 옵션"** 을 **"스팟 인스턴스"**로 설정합니다.  
- ASG 에 사용될 스팟 인스턴스 시작 템플릿에서 요청 유형은 **"일회성"**으로 제한 됩니다.

(b). EC2 인스턴스 콘솔에서 AutoScaling -> AutoScaling 그룹 생성 합니다.

![6](/assets/img/posts/ec2-spot-guide/07.png)
_6_

(c). AutoScaling 그룹 요청 시 필요한 **"파라미터"**를 결정합니다.

- 시작 템플릿 적용

(d). 인스턴스 시작 옵션 선택 및AutoScaling 그룹 크기 및 크기 조정 구성  
- 인스턴스 유지 관리 정책은 AutoScaling 그룹에서 인스턴스 교체 이벤트가 발생하는 경우 어떠한 교체 동작을 설정할지 확인하는 부분입니다. -  [참고 문헌](https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/instance-maintenance-policy-overview-and-considerations.html)

![7](/assets/img/posts/ec2-spot-guide/08.png)
_7_

---

## 4. 생성된 스팟 인스턴스 상태 확인 및 모니터링

- 스팟 인스턴스 요청을 추적하고 스팟 인스턴스 사용 계획을 세우는 데 도움이 되도록 Amazon EC2에서 제공하는 요청 상태를 사용합니다.

### 4-1. EC2 Spot Instance 요청 작동 방식

![스팟 인스턴스 작동 방식](/assets/img/posts/ec2-spot-guide/09.png)
_스팟 인스턴스 작동 방식_

4-2. EC2 Spot Instance 요청 상태 확인

![스팟 요청 상태 확인](/assets/img/posts/ec2-spot-guide/10.png)
_스팟 요청 상태 확인_

- 요청된 스팟 인스턴스에 대한 상세 정보는 "스팟 요청" 탭에서 확인이 가능합니다.

- 스팟 요청 (*ID)에 대한 "상태" 탭 부분을 확인하면 요청된 스팟 요청에 대한 상태를 확인 가능합니다. - [참고 문헌](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/spot-request-status.html#spot-instance-request-status-understand)

```
==fulfilled==

스팟 요청이 active이고 Amazon EC2에서 스팟 인스턴스를 시작하는 중입니다.

==instance-stopped-by-price==

스팟 가격이 최고 가격을 초과하여 인스턴스가 중지됩니다.

==instance-stopped-by-user==

사용자가 인스턴스를 중지했거나 인스턴스에서 종료 명령을 실행했기 때문에 인스턴스가 중지되었습니다

==instance-terminated-by-price==

스팟 가격이 최고 가격을 초과하여 인스턴스가 종료됩니다. 요청이 영구적일 경우 프로세스가 다시 시작되므로 요청은 평가 보류 상태입니다.
```

---

Spot 인스턴스는 최대 90%까지 비용을 절감할 수 있는 매우 효율적인 옵션이지만, 그만큼 신중한 계획과 관리가 필요합니다.

특히 주목할 만한 점들을 정리하면 다음과 같습니다:

1. **가격 최적화**
   - On-Demand 대비 최대 90% 비용 절감 가능
   - 다양한 할당 전략을 통한 비용 대비 안정성 확보
2. **안정성 확보**
   - Spot Fleet이나 Auto Scaling Group을 활용한 고가용성 구성
   - 중단 공지를 활용한 안정적인 워크로드 관리
3. **유연한 관리**
   - 다양한 인스턴스 타입 활용 가능
   - 자동화된 용량 관리 및 스케일링

Spot 인스턴스를 효과적으로 활용하기 위해서는 워크로드의 특성을 잘 이해하고, 적절한 구성 방식을 선택하는 것이 중요합니다. 특히 중단 허용성이 있는 워크로드나 유연한 시작/종료가 가능한 애플리케이션에 매우 적합합니다.
