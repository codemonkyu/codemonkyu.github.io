---
title: "[AWS] ddpt를 곁들인 EBS 스냅샷 크기 최적화"
date: 2025-03-02 16:41:03 +0900
categories: ["Cloud & Infrastructure", "AWS"]
tags: ["ebs", "optimized", "snapshot", "스냅샷 최적화", "전체 스냅샷 크기"]
tistory_url: https://codemonkyu.tistory.com/entry/AWS-ddpt%EB%A5%BC-%EA%B3%81%EB%93%A4%EC%9D%B8-EBS-%EC%8A%A4%EB%83%85%EC%83%B7%EC%9D%98-%ED%81%AC%EA%B8%B0-%ED%99%95%EC%9D%B8
---

![](/assets/img/posts/ebs-full-snapshot-size-ddpt/01.png)
_AWS - EBS Snapshot_
![](/assets/img/posts/ebs-full-snapshot-size-ddpt/02.png)
_AWS - EBS Snapshot_

오늘은 AWS EBS Console에서 새롭게 추가된 항목인 "전체 스냅샷 크기" 정보에 대해 EBS에서 실제 스냅샷 크기를 어떻게 측정하는지에 대하여 간단한 테스트를 통해 알아보려고 한다.

---

## EBS 전체 스냅샷 크기 정보란?

먼저 EBS의 전체 스냅샷 크기 정보에 대해서는 다음의 AWS 공식 문서에서 확인할 수 있다.  
------  
[공식문서](https://aws.amazon.com/ko/about-aws/whats-new/2025/02/amazon-ebs-full-snapshot-size-information-console-api/)

------

```

이제 Amazon Elastic Block Store(Amazon EBS)는 전체 스냅샷 크기의 EBS 스냅샷을 표시합니다. 이번 개선 사항으로 고객은 새로운 필드인 ull-snapshot-size-in-bytes를 사용하여 DescribeSnapshots API를 통해 프로그래밍 방식으로 전체 스냅샷 크기를 검색할 수 있게 되었습니다. 전체 스냅샷 크기는 EBS 스냅샷 콘솔에서 새로운 '전체 스냅샷 크기' 열에도 표시됩니다.

'전체 스냅샷 크기' 필드는 스냅샷을 구성하는 모든 블록의 총 크기, 즉 해당 스냅샷에 직접 저장된 블록과 이전 스냅샷에서 참조된 모든 블록을 보여줍니다. 예를 들어 100GB 볼륨에 50GB의 데이터가 있는 경우, 첫 번째 스냅샷이나 후속 스냅샷이나 관계없이 '전체 스냅샷 크기'는 50GB로 표시됩니다.

'전체 스냅샷 크기' 필드는 아카이브된 계층에 있는 스냅샷의 전체 크기, 스냅샷이 생성될 때 소스 볼륨에 기록된 데이터의 양 등 EBS 스냅샷 스토리지에 대한 중요한 정보를 제공합니다. 이는 특정 스냅샷에 저장된 새로 변경된 블록의 크기만을 나타내는 증분 스냅샷 크기와는 다르다는 점에 유의하시기 바랍니다.

```

AWS 공식문서에서 설명하고 있는 내용과 같이  "전체 스냅샷 크기"는 스냅샷이 생성될 때 소스 볼륨에 기록된 데이터양을 보여주는 지표로 보인다. 다만 동일한 볼륨에 대해 여러개의 스냅샷을 생성할 때 스냅샷별로 증분된(*변경된 블록) 크기를 보여주는것은 아님을 유의하자.

---

---

## 전체 스냅샷 크기 확인

실제 **"전체 스냅샷 크기"** 정보는 AWS EC2 Console -> EBS -> 스냅샷 -> 스냅샷 경로에서 확인할 수 있다. 실제 아래의 사진과 같이 스냅샷 세부 정보란에서는 **"전체 스냅샷 크기"**가 보여지고 있음을 알 수 있다.

![이미지 1](/assets/img/posts/ebs-full-snapshot-size-ddpt/03.png)
_이미지 1_

이외의 방법으로는 aws-cli를 사용하여 스냅샷을 조회(*describe)하여 **"fullSnapshotSizeInBytes"** 파라미터를 확인하는 방법으로도 전체 스냅샷 크기를 알 수 있다. 다만 해당 파라미터를 조회하기 위해서는 **aws-cli "2.24.2" 버전 이상**을 사용해야 한다.

```bash
#aws --version
aws-cli/2.24.7 Python/3.12.6 Linux/6.1.128-136.201.amzn2023.x86_64 exe/x86_64.amzn.2023

#aws ec2 describe-snapshots --snapshot-id snap-0c617aa24667f48bc
"Snapshots": [
        {
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "Snapshot1"
                }
            ],
            "StorageTier": "standard",
            "TransferType": "standard",
            "CompletionTime": "2025-02-24T07:07:54.324000+00:00",
            "FullSnapshotSizeInBytes": 5444206592, <--- 확인 !!
```

위 예시와 같이 "FullSnapshotSizeInBytes" 파라미터를 조회하여 실제 조회한 스냅샷이 갖고 있는 데이터 크기를 알 수 있다. 실제 스냅샷이 갖고 있는 데이터 크기를 확인해보면, **Bytes to GiB**으로 전환해보면 5444206592Bytes - > **5.07 GiB** 임을 알 수 있다.

---

---

## 테스트 - 전체 스냅샷 크기 동작 원리

이전에 "전체 스냅샷 크기" 정보에 대해서 AWS 공식 문서에서 아래와 같이 해당 정보는 증분된(*변경된 크기만 나타내는) 스냅샷 크기를 나타내는 것이 아님을 설명하고 있었다.

>  
> 이는 특정 스냅샷에 저장된 새로 변경된 블록의 크기만을 나타내는 증분 스냅샷 크기와는 다르다는 점에 유의하시기 바랍니다.

그렇다면 이를 어떻게 증명할 수 있을까 고민해보았으며, 다음의 [AWS 문서](https://aws.amazon.com/ko/blogs/apn/how-to-build-sparse-ebs-volumes-for-fun-and-easy-snapshotting/) 의 "Optimization" 부분에서 이용한 테스트 방법 활용하여 해당 파라미터가 변경된 블록 크기를 반영하지 않음을 확인할 수 있는 테스트를 진행하였다.

먼저 동일한 볼륨에 대하여 변경된 데이터 블록을 갖고 있는 스냅샷 2개를 생성하여 "전체 스냅샷 크기"에 변경된 데이터 블록에 대한 크기가 반영이 되는지 확인하고 이후 변경된 데이터 블록을 갖고 있는 볼륨의 블록 이미지를 생성하여 이를 새로운 볼륨에 복사하여 새 볼륨에 대한 세번째 스냅샷을 생성하여 마지막 스냅샷이 갖고 있는 데이터를 확인한다.

이러한 방법을 통해 스냅샷이 실제 EBS가 마운트된 환경 (*파일시스템 이하의 데이터 블록 정보)에 대한 정보를 반영하는지 아님 스냅샷이 생성되는 시점의 Data Block수만 Count 하는지 확인할 수 있게 된다.

테스트 순서를 자세히 정리하자면 아래와 같다.

-----

테스트 순서

1. 테스트 인스턴스 생성

2. 테스트 볼륨 A 생성

3. 테스트 볼륨 A에 dd 또는 파일 다운로드를 통해 볼륨 용량 사용

4. 테스트 볼륨 A에 대한 스냅샷 A 생성 후 콘솔에서 용량 확인

5. 테스트 볼륨 A에서 파일 삭제

6. 다시 테스트 볼륨 A에 대한 스냅샷 B 생성 후 콘솔에서 용량 확인
7. 테스트 볼륨 A에 대하여 dd 명령어를 사용하여 image 생성

8. 생성한 image를 루프백을 (*loop 옵션)을 사용하여 마운트

9. fstrim 명령어를 생성하여 변경된 데이터 블록(* unallocated block)을 정리

10. 새로운 테스트 볼륨 B 생성 후 해당 볼륨에 trim된 이미지를 복사

11. 이미지가 테스트 볼륨 B에 전부 복사되면 테스트 볼륨 B에 대한 스냅샷 C 생성

12. 스냅샷 C의 용량을 확인

------

```text
(1). 새로운 볼륨을 생성하여 장착 후 볼륨의 용량 사용을 위한 파일을 다운로드 합니다.
(*현재 예시에서는 ISO 파일을 다운로드 함.)

[root@-]# df -hT
Filesystem       Type      Size  Used Avail Use% Mounted on
..
..
/dev/nvme1n1     xfs        30G  6.7G   24G  22% /dd <--- 확인

[root@-]# ls -al
-rw-r--r--.  1 root root 4762707968 Sep 11 14:38 ubuntu-22.04.5-desktop-amd64.iso <--- 4.4GiB 용량의 파일
-rw-r--r--.  1 root root 2136926208 Sep 11 18:46 ubuntu-22.04.5-live-server-amd64.iso <--- 2.0GiB 용량의 파일

(2).nvme1n1 볼륨에 대한 스냅샷 A을 생성하고 크기를 관찰합니다.
전체 스냅샷 크기
6.4 GiB

(3). 작은 용량을 차지하는 파일을 삭제하고 스냅샷 B를 생성한 뒤 스냅샷 크기를 관찰합니다.
[root@ip-]# rm -f ubuntu-22.04.5-live-server-amd64.iso
[root@ip-]# df -hT
Filesystem       Type      Size  Used Avail Use% Mounted on
/dev/nvme1n1     xfs        30G  4.7G   25G  16% /dd

전체 스냅샷 크기
6.4 GiB

=> 해당 작업을 통해서 실제 마운트된 볼륨에서 파일 삭제를 진행, 변경된 블록 (*unallocated blocks) 를 생성합니다.

(4). dd 명령어를 sparse.img 파일을 생성하고, 루프백 장치로 마운트합니다. 이렇게 하여 실제 마운트 포인트를 이용해 이미지에 접근할 수 있게되며, 이후 "fstrim" 명령어를 통해 마운트된 이미지 파일의 비할당된 블록(삭제된 파일의 블록)을 제거합니다. 해당 과정에서 생성된 sparse.img 파일의 실제 용량은 줄어들게 됩니다.

[root@ip-]# dd if=/dev/nvme1n1 of=sparse.img bs=512 status=progress
32203947008 bytes (32 GB, 30 GiB) copied, 186 s, 173 MB/s
62914560+0 records in
62914560+0 records out
32212254720 bytes (32 GB, 30 GiB) copied, 186.047 s, 173 MB/s

[root@ip-]# ls -al
-rw-r--r--.  1 root root 32212254720 Feb 24 11:41 sparse.img <--- Check

# mount -o nouuid,loop,discard sparse.img /loop
* loop 옵션을 사용합니다.
* https://linux.die.net/man/8/mount

# fstrim /loop
* fstrim is used on a mounted filesystem to discard (or "trim") blocks which are not in use by the filesystem.
* https://man7.org/linux/man-pages/man8/fstrim.8.html

(7). ddpt 명령어를 통해 트림(*trim)된 이미지를 빈 볼륨 (*30GiB 용량의 새 볼륨)에 복사합니다. 해당 과정을 통해 실제  할당된 블록에 대한 데이터만 가진 볼륨이 생성 됩니다.

[root@ip-]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme1n1       259:1    0   30G  0 disk /dd
nvme5n1       259:8    0   30G  0 disk <--- 새 데이터 볼륨 B

[root@ip-]# ddpt of=/dev/nvme5n1 if=sparse.img bs=512 oflag=sparse
62914560+0 records in
9305984+0 records out
53608576 bypassed records out
time to transfer data: 39.785926 secs at 809.64 MB/sec

* ddpt - copies data between files and storage devices. Support for devices that understand the SCSI command set.
---

ddpt 명령어의 oflag=sparse 옵션은 이미지 복사 시 unallocation blocks을 건너뛰어 복사하는 기능을 제공합니다.

일반적인 dd 명령어는 볼륨의 모든 블록을 그대로 복사하기 때문에, 삭제된 파일의 블록(unallocation blocks)도 함께 복사됩니다. 이렇게 되면 복사된 볼륨의 크기가 실제 사용량보다 크게 나타나게 됩니다.

반면 ddpt 명령어의 oflag=sparse 옵션은 다음과 같은 역할을 합니다:

소스 이미지에서 할당되지 않은 블록(unallocation blocks)을 감지합니다.

이러한 unallocation blocks은 복사 과정에서 건너뛰게 됩니다.

결과적으로 실제 사용량에 해당하는 데이터만 타깃 볼륨에 복사됩니다.

---
* What is sparse wrriten ? : https://sg.danny.cz/sg/ddpt.html#sparse
* ddpt man : https://linux.die.net/man/8/ddpt

(8). 마지막으로 새 볼륨에 대한 스냅샷 C을 생성하고 크기를 관찰합니다.
전체 스냅샷 크기
4.44 GiB

4762707968 bytes -> 4.435GiB

=> (3) 작업에서 파일을 삭제하였지만, 스냅샷 B의 용량이 변경되지 않은것을 확인하였다. 하지만 trim 작업을 통해 allocated 되지 않은 block을 버린 후 다시 스냅샷 C를 생성하여 확인한 결과 정상적으로 스냅샷의 실제 전체 크기가 줄어든 것을 확인할 수 있다.
```

---

## 정리

EBS 볼륨은 블록 단위로 데이터를 관리합니다. 볼륨에 파일을 생성하면 해당 파일의 데이터가 볼륨의 블록에 기록됩니다. 문제는 이렇게 생성된 파일을 나중에 삭제하더라도, EBS는 그 블록들이 수정되었다는 사실만 기억하고 있다는 점입니다. EBS는 파일 시스템 수준의 정보를 모르기 때문에, 삭제된 파일의 블록들을 추적하지 못합니다. 이렇게 삭제된 파일의 블록들을 "unallocated blocks"라고 합니다. 이 블록들은 더 이상 파일 시스템에서 사용되지 않지만, EBS 볼륨 스냅샷에는 여전히 포함되어 있습니다. 문제는 이런 unallocated blocks이 스냅샷의 크기를 계속 증가시킨다는 점입니다. 예를 들어, 처음에 25GB 볼륨을 생성하고 20GB를 사용했다고 가정합시다. 이 경우 스냅샷 크기는 20GB가 됩니다. 그 후 5GB 파일을 삭제했다고 하더라도, EBS는 여전히 25GB 크기의 스냅샷을 생성합니다. 이는 삭제된 5GB 블록들이 unallocation blocks로 남아 있기 때문입니다. 이러한 unallocation blocks은 볼륨을 생성할 때 불필요한 데이터를 가져오게 되어, 초기 읽기 성능 저하의 원인이 됩니다. 따라서 이 unallocation blocks을 제거하는 것이 중요한 최적화 작업이 됩니다.
