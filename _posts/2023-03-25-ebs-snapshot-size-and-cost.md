---
title: "[AWS] 스냅샷의 작동 방식 및 실제 크기 구하기"
date: 2023-03-25 18:07:08 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["개념", "사이즈 구하기", "생성", "스냅샷", "작동 방식"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-%EC%8A%A4%EB%83%85%EC%83%B7%EC%9D%98-%EC%9E%91%EB%8F%99-%EB%B0%A9%EC%8B%9D-%EB%B0%8F-%EC%82%AC%EC%9D%B4%EC%A6%88-%EA%B5%AC%ED%95%98%EA%B8%B0
---

![](/assets/img/posts/ebs-snapshot-size-and-cost/01.png)

AWS에서는 EBS볼륨에 대한 스냅샷(Snapshot)기능을 제공하고 있다.

이번 글에서는 스냅샷의 작동 방식 / 생성 / 실제 스냅샷이 가지고 있는 데이터의 크기 계산을 통해 스냅샷의 비용까지 계산해보는 내용에 대한 글을 포스팅을 해보려고한다.

바로 본론으로 들어가보자!

## 스냅샷의 작동 방식

스냅샷의 작동 방식을 쉽게 이해하기 위해서 제일 먼저 알아야 할 가장 중요한 개념은 스냅샷은 "증분식"으로 작동한다는 사실이다.

증분식이란 동일한 볼륨에 대해 여러개의 스냅샷을 생성하는 경우 가장 최신의 스냅샷(마지막 스냅샷)은 이전의 스냅샷이 가진 데이터에서 변경된 부분만 가지게 된다는 것을 의미한다. 조금 더 정확한 이해를 돕기 위한 그림 및 공식 문서를 첨부한다.

[AWS 공식 문서 - 스냅샷 작동 방식](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/EBSSnapshots.html#how_snapshots_work)

### [동일한 볼륨에 대한 여러 스냅샷 생성]

![(* 출처: AWS 공식 문서 )](/assets/img/posts/ebs-snapshot-size-and-cost/02.png)
_(* 출처: AWS 공식 문서 )_

위의 그림을 보면 동일 볼륨에 대한 스냅샷 생성 방식을 한눈에 이해할 수 있다.

맨 왼쪽부터 스냅샷이 생성된 순서에 따라 **"Snapshots A-> B-> C"**로 표현하고 있으며, 볼륨에 대한 데이터가 변경될 때마다 스냅샷을 생성하는 순서를 **"State"**로 표현하고 있다.

**State 1** 부터 살펴보면, 15GiB 용량을 가진 EBS 볼륨에서 10GiB의 데이터를 가지고 있고, 해당 볼륨에 대한 첫번째 스냅샷을 생성한다면 스냅샷 A가 가진 용량은 10GiB가 된다.

**State 2** 에서는 기존 EBS 볼륨의 10GiB 데이터에서 4GiB의 데이터가 변경(changed)되었는데, 해당 시점에서 두번째 스냅샷을 생성한다면 스냅샷 B는 변경된 4GiB의 데이터를 가지며, 변경되지 않은 6GiB에 대해서는 스냅샷 A의 부분을 참조(referenced)한다.

(* 이 시점에서 10GiB 데이터를 가진 볼륨에 대한 스냅샷이 2개 생성되었지만 실제로 스냅샷이 가진 데이터의 크기는 A -> 10GiB + B -> 4 GiB으로 14GiB가 된다.)

**State 3** 에서는 기존 10GiB 데이터를 가지고 있는 볼륨에 2GiB 크기의 새로운 데이터가 추가(added)되었으며, 해당 시점에서 세번째 스냅샷을 생성한다면 스냅샷 C는 새로 추가된 2GiB의 데이터를 가지며, 스냅샷 A의 6GiB, 스냅샷 B의 4GiB의 데이터를 참조(referenced)한다.

위 과정을 정리해보면 아래와 같다.

1. Snapshot A : 10GiB 데이터를 가짐 (EBS 볼륨이 가지고 있는 데이터 용량 : 10GiB)
2. Snapshot B : 4GiB 데이터를 가짐 (EBS 볼륨이 가지고 있는 데이터 용량 : 기존의 6GiB + 변경된 4GiB)
3. Snapshot C : 2GiB 데이터를 가짐 (EBS 볼륨이 가지고 있는 데이터 용량 : 기존의 10GIB + 추가된 2GIB 으로 12GiB)

Snapshot A + B + C 저장에 필요한 S3의 용량은 10 + 4 + 2 로 16GiB이 된다. 따라서 사용자는 16GiB에 대한 스냅샷 비용을 지불하면된다. (* 스냅샷은 EBS의 데이터를 S3에 백업하는 방식으로 생성된다.)

### [예상 비용]

[AWS 공식 문서 - Amazon EBS 요금](https://aws.amazon.com/ko/ebs/pricing/)

1. 스탠다드 스냅샷을 사용할 경우 (*일반적인 경우)
   1. 예상 월별 요금 : 16GiB x 0.05 USD = 0.8 USD
2. 아카이브 스냅샷을 사용할 경우
   1. 예상 월별 요금 : 16GiB x 0.0125 USD = 0.2 USD

---

### [서로 다른 볼륨에서의 증분 스냅샷]

![(* 출처: AWS 공식 문서 )](/assets/img/posts/ebs-snapshot-size-and-cost/03.png)
_(* 출처: AWS 공식 문서 )_

이전 처럼 **같은 볼륨** 에 대한 연속적인 스냅샷이 아니라 서로 다른 볼륨(처음 동일한 스냅샷으로 생성된 볼륨)에 대한 스냅샷일 경우 어떻게 증분형식으로 생성되는지 위의 그림을 통해 이해할 수 있다.

위 그림에서 14GiB의 용량과 10GiB의 데이터를 갖고 있는 "볼륨 1" 에 대하여 첫 스냅샷 A를 생성하였다. 따라서 스냅샷 A는 10GiB의 크기를 가지게 된다. 이후 스냅샷 A를 이용해서 새로운 볼륨인 "볼륨 2"를 생성한다. "볼륨 2"은 생성되며 스냅샷 A의 10GiB의 데이터를 갖고 있게 된다. 이후 "볼륨 2"에 4GiB 의 데이터가 추가한 후 해당볼륨(볼륨 2)에 대한 첫 스냅샷인 스냅샷 B를 생성해도 스냅샷의 용량은 14GiB가 아닌 4GiB가 된다. 왜냐하면 스냅샷 B는 기존 10GiB의 데이터에 대해 스냅샷 A를 참조하고 있기 때문이다.

참조라는 것이 어려울 수 있지만, 쉽게 생각하면 스냅샷이 생성될 때 중복된 데이터는 기존의 것을 따르며, 변경된 데이터 블록만 저장하는 방식으로 새로운 스냅샷을 생성하기 때문에 어찌보면 당연한 방식이다. 그렇기 때문에 제일 마지막에 생성되는 스냅샷은 이전의 스냅샷에 대한 모든 중복된 데이터를 참조하게된다. 최신의 스냅샷이 아닌 이전의 스냅샷을 지워도 최신 스냅샷으로 볼륨생성시 해당 볼륨의 이전 데이터를 포함하여 전부 복구할 수 있어야 되기 때문이다.

> 용량이 20 GiB 이며 데이터 10GiB를 갖고 있는 볼륨 A에 대한 연속적인 스냅샷의 참조관계  
> e.g 스냅샷 N (데이터 크기 / 참조 )

> 볼륨 A에 대한 첫 스냅샷 A 생성 -> 스냅샷 A (10 GiB)  
>  
> 볼륨 A에 4GiB의 데이터를 변경 후 스냅샷 B 생성 -> 스냅샷 B ( 4 GiB / 스냅샷 A 참조)
>
> 볼륨 A에 5GiB의 데이터를 추가 후 스냅샷 C 생성 -> 스냅샷 C ( 5 GiB / 스냅샷 A & B 참조)  
>  
>
> 볼륨 A에 3GiB의 데이터를 추가 후 스냅샷 D 생성 -> 스냅샷 D ( 3 GiB / 스냅샷 A & B & C 참조)  
>  
>
> 스냅샷 A를 삭제시 스냅샷 A의 10GiB의 데이터는 이를 참조하고 있는 스냅샷 B로 이동 -> 스냅샷 B 의 데이터 크기 변경 ( 4GiB -> 10 GiB )  
>  
>
> 스냅샷 C는 스냅샷 B만 참조하게 된다.  
>  
>
> 스냅샷 D는 스냅샷 B & C 만 참조하게 된다.  
>  
>
> 스냅샷 C를 삭제하면 스냅샷 D는 ( 8 GiB / 스냅샷 B 참조) 하게 된다. 결국 스냅샷 D로 새로운 볼륨을 생성하여도 스냅샷 B를 참조하고 스냅샷 C에 대한 참조하는 부분이 스냅샷 D로 넘어왔기 때문에 18GiB 데이터를 갖고 있는 새로운 볼륨을 정상적으로 생성 가능하다. (데이터 보장)

## 스냅샷 생성

스냅샷을 생성하는 방법에 대해서 설명하는 글을 이미 많이 존재하기 때문에 이 생성 방법은 간단하게 공식문서로 대체하며, 스냅샷을 생성하는 경우 주의해야할 점 또는 생성시 발생할 수 있는 상황에 대하여 정리해본다.

### [스냅샷 생성 방법]

[AWS 공식 문서 - 스냅샷 생성](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ebs-creating-snapshot.html#ebs-create-snapshot)

### [스냅샷 생성 시 주의해야할 부분]

- 스냅샷은 비동기적으로 생성된다. 따라서 특정 시점에 대해서 여러개의 스냅샷 생성이 가능하지만, 스냅샷이 완료될 때 까지 (수정 / 추가된 데이터 블록이 모두 S3로 정상적으로 전송이 완료될 경우  
) 스냅샷의 상태는 "pending" 상태이다. 변경되는 데이터의 블록의 크기가 크다면 스냅샷 생성까지 수 시간이 소요될 수 있다. 또한 같은 볼륨에 대한 여러개의 "pending" 상태의 스냅샷이 있는 경우 볼륨 성능에 저하가 일어날 수 있다.
- "pending" 상태의 스냅샷은 st1 & sc1 볼륨 유형에 대해서는 한개로 제한되며, 다른 유형의 볼륨에 대해서는 5개로 제한된다. 만약 스냅샷 생성시 "ConcurrentSnapshotLimitExceedEdEd" 에러가 발생하면, 생성이 진행중 인 스냅샷이 완료된 후 다시 스냅샹 생성을 시도해야한다.
- 스냅샷이 생성되는 동안 해당 스냅샷은 볼륨의 읽기 및 쓰기 작업에 영향을 주지 않는다.
- 지속적인 쓰기 작업이 진행되는 사용중인 볼륨에 대한 스냅샷을 생성할 때는 데이터의 일관성을 위해 해당 볼륨을 인스턴스에서 떼어낸다음 진행하는 것도 방법이다. 이후 스냅샷을 생성하고 스냅샷의 상태가 "pending"일 때 다시 해당 볼륨을 부착하여 사용하면 된다.
- **OS가 설치된 루트 볼륨 (데이터 볼륨이 아닌)에 대한 스냅샷을 생성할때는 반드시 인스턴스를 중지한 후 스냅샷을 생성해야 한다.**

###

## 실제 스냅샷의 데이터 크기 구하기

EC2 managemnet console에서 생성된 스냅샷을 확인할 수 있지만, 아쉽게도 스냅샷이 가진 데이터의 크기를 알수는 없다... (* 현재는 스냅샷에 대한 볼륨의 크기만 제공되고 있다.)

따라서 스냅샷의 실제 크기는 AWS CLI 명령을 통해 스냅샷이 가지고 있는 블록 데이터 수를 구한다음 블록의 크기 (512KiB)를 곱하여 전체 스냅샷의 데이터 크기를 구해야 한다.

### [첫번째로 생성하는 스냅샷의 크기 구하기]

"list-snapshot-blocks" 명령어를 통해 스냅샷이 가지고 있는 블록 수를 구할 수 있다.

- [list-snapshot-blocks](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ebs/list-snapshot-blocks.html#list-snapshot-blocks)  
  - Returns information about the blocks in an Amazon Elastic Block Store snapshot.

**스냅샷 데이터 크기 구하기 명령어**

> $ aws ebs list-snapshot-blocks --snapshot-id your snapshot ID

위의 명령어를 통해 "BlockIndex"의 수를 직접 구한 다음 블록 한개의 크기 (512KiB)를 곱해주면 스냅샷의 데이터 크기를 알 수 있다.

> 스냅샷 크기 : BlockIndex 수 x 512 KiB

만약 명령어의 결과로 10000개 이상의 블록이 반환되면, "Next Token"의 항목값을 통해 아래의 명령어를 통해 다음 토큰에 대해 반환된 블록의 수를 구할 수 있다.

> aws ebs list-snapshot-blocks --snapshot-id your snapshot ID --next-token your token ID

### [같은 볼륨에 대해 여러개의 스냅샷 생성하는 경우 스냅샷의 크기 구하기]

같은 볼륨에 대해 여러개의 스냅샷을 생성한 경우에는 변경된 블록수만 구하여 계산하는 방법을 통해 스냅샷의 데이터 크기를 알 수 있다.

- [list-changed-blocks](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ebs/list-changed-blocks.html)  
  - Returns information about the blocks that are different between two Amazon Elastic Block Store snapshots of the same volume/snapshot lineage.

아래의 명령어로 첫번째와 두번째 스냅샷의 크기 차이를 구할 수 있다.

> $ aws ebs list-changed-blocks --first-snapshot-id "value" --second-snapshot-id "value"

위의 명령어로 구한 블록 수에 블록의 크기를 곱하면 두번째 스냅샷의 데이터 크기를 구할 수 있다.

> 스냅샷 크기 : BlockIndex 수 x 512 KiB
