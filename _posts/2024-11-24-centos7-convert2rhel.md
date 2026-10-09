---
title: "[Linux] CentOS7에서 RHEL OS로 전환 - convert2rhel 사용법"
date: 2024-11-24 10:18:23 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["centos7", "convert2rhel", "rhel7.9", "사용"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-CentOS7%EC%97%90%EC%84%9C-RHEL-OS%EB%A1%9C-%EC%A0%84%ED%99%98-convert2rhel-%EC%82%AC%EC%9A%A9%EB%B2%95
---

![](/assets/img/posts/centos7-convert2rhel/01.jpg)
_출처_

CentOS 7의 End of Support (EOS) 날짜는 2024년 6월 30일입니다.

주요 일정:
    2020년 12월 31일: CentOS 7의 전체 업데이트 지원 종료
    2024년 6월 30일: CentOS 7의 유지보수 업데이트 완전 종료 (EOS)

2024년 6월 30일 이후에는 CentOS 7에 대한 보안 패치, 버그 수정, 기술 지원이 더 이상 제공되지 않습니다. 이로 인해 많은 조직들이 RHEL이나 다른 대안으로의 마이그레이션을 계획하고 있습니다.

현재 시점(2024년 11월)에서는 CentOS 7이 이미 EOS에 도달했으므로, 보안 및 안정성을 위해 다른 운영 체제로의 전환이 매우 중요한 상황입니다.

---

위와 같은 이유로 현재 Centos7을 사용하고 있다면 다른 OS로 전환하는 것을 고려해보아야 한다.

해당 포스팅을 통해 RedHat 사의 command line tool 인 "convert2rhel" 사용하여 centos7에서 RHEL 7.9로 전환하는 방법에 대해 가이드를 진행하려고 한다.

---

1. Conversion 방법

참고 문헌
a). Converting by using the command line interface

- Use this method to convert a small number of servers, or to convert a large number of servers at scale by using custom automation.

b). Converting by using Red Hat Satellite

- Use this method to convert a large number of servers at scale. To convert by using Satellite, you must have a Satellite subscription. For more information about how to convert by using Satellite, see Converting a Host to Red Hat Enterprise Linux.

c). Converting by using Red Hat Insights

- Use this method to convert multiple systems in a user-friendly GUI interface without needing a Satellite subscription. Systems must be connected to the internet. Systems are fully onboarded with Red Hat Insights after the conversion.

2. 고려 사항

참고 문헌

**시스템을 RHEL로 변환하기 전에 다음을 고려해야 합니다.**
- **아키텍처** - 소스 OS는 64비트 Intel 아키텍처가 있는 시스템에 설치해야 합니다. 다른 시스템 아키텍처와 함께 변환할 수 없습니다.

- **Security** - FIPS 모드의 시스템은 변환에 지원되지 않습니다.

- **kernel** - RHEL 커널 모듈에 없는 커널 모듈을 사용하는 시스템은 현재 변환을 위해 지원되지 않습니다. 변환 전에 외부 커널 모듈을 비활성화하거나 제거한 다음 나중에 해당 커널 모듈을 활성화하거나 다시 설치하는 것이 좋습니다. 지원되지 않는 커널 모듈은 다음과 같습니다.

- 특수 애플리케이션, GPU, 네트워크 드라이버 또는 스토리지 드라이버용 커널 모듈

- DKMS에서 빌드한 사용자 정의 컴파일 커널 모듈

- **퍼블릭 클라우드** - 퍼블릭 클라우드의 변환은 다음과 같은 상황에서 지원됩니다.
    - Alma Linux, CentOS Linux, Rocky Linux - Using Red Hat Subscription Manager (RHSM) for the following
        - AWS(Amazon Web Services), Microsoft Azure 및 Google Cloud의 이미지(관련 소프트웨어 비용 없음).
        - 모든 퍼블릭 클라우드에서 사용자 정의 이미지 제공
        - Oracle Linux - 모든 퍼블릭 클라우드에서 사용자 지정 이미지에 RHSM 사용.
        `Convert2RHEL` 은 변환 중에 RHUI(Red Hat Update Infrastructure)를 통해 RHEL 패키지에 액세스할 수 없습니다.

- **고가용성** - Red Hat 또는 타사의 고가용성 클러스터 소프트웨어를 사용하는 시스템은 현재 RHEL로 전환하기 위해 테스트되거나 지원되지 않습니다. Red Hat은 이러한 환경의 무결성을 보장하기 위해 새로 설치된 RHEL 시스템으로 마이그레이션하는 것이 좋습니다.

- **Identity Management** - FreeIPA 서버의 인플레이스 변환을 수행하는 것은 지원되지 않습니다. FreeIPA 배포를 IdM으로 마이그레이션하는 방법에 대한 자세한 내용은 [RHEL이 아닌 Linux 배포판의 FreeIPA에서 FreeIPA 로 마이그레이션하고 RHEL 이외의 Linux 배포판 의 FreeIPA에서 IdM](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/linux_domain_identity_authentication_and_policy_guide/index#Migrating_to_IdM_on_RHEL_7_from_FreeIPA_on_non-RHEL_Linux_distributions) 으로 마이그레이션을 참조하십시오.

- **Foreman** - Katello 플러그인과 함께 Foreman을 사용하는 시스템 변환은 지원되지 않습니다. 지원되는 변환을 수행하려면 먼저 Red Hat Satellite로 마이그레이션한 다음 변환을 진행합니다.

- **RAID** - `mdadm`-managed RAID 장치를 사용하여 UEFI 기반 시스템을 변환할 수 없습니다.

3. 변환 준비

참고 문헌

3-1) 사전 요구 사항

**사전 요구 사항**
- 시스템이 RHEL로 변환될 수 있는지 확인했습니다. 자세한 내용은 [지원되는 변환 경로를] 참조하십시오.

- 데이터 무결성 문제의 위험을 줄이기 위해 중요한 애플리케이션, 데이터베이스 서비스 및 데이터를 저장하는 기타 서비스를 중지했습니다.

- 변환이 실패하지 않도록 일시적으로 바이러스 방지 소프트웨어를 비활성화했습니다.

- 원래 시스템을 복원하지 않도록 Salt, Chef, Puppet, Ansible과 같은 구성 관리 시스템을 비활성화하거나 적절하게 재구성했습니다.

- `sos` 패키지가 설치되어 있어야 합니다. 이 패키지를 사용하여 Red Hat 지원 팀의 지원 케이스를 열 때 필요한 `sosreport` 를 생성해야 합니다.

- Satellite 또는 RHSM에 활성화 키를 생성했습니다. 자세한 내용은 Satellite 문서의 [활성화 키 관리] 및 RHSM 문서의 [하이브리드 클라우드 콘솔의 활성화 키 시작하기] 를 참조하십시오.

- SCA[Simple Content Access]를 활성화했습니다. 2022년 7월 15일 이후에 생성된 Red Hat 계정의 SCA는 기본적으로 활성화되어 있습니다.

4. 전환 절차 진행

** 전환 진행전 최종 점검 사항 **
- 시스템을 백업하고 필요한 경우 복원할 수 있는지 확인합니다.

- [알려진 문제 및 제한 사항을] 확인하고 시스템 변환이 지원되는지 확인합니다. 해결방법을 적용하십시오.

- CentOS Linux 8에서 변환하는 경우 시스템에서 CentOS Stream 패키지를 제거하십시오. CentOS Stream은 현재 변환에 지원되지 않으며 시스템에 패키지가 있는 경우 변환이 실패할 수 있습니다.

- Red Hat Satellite를 사용하거나 프록시 서버를 통해 방화벽으로 변환하는 경우 다음 연결에 액세스할 수 있는지 확인합니다.

    - [https://cdn.redhat.com](https://cdn.redhat.com)
    - [https://cdn-public.redhat.com](https://cdn-public.redhat.com)
    - [https://subscription.rhsm.redhat.com](https://subscription.rhsm.redhat.com) - 방화벽이 있는 시스템에만 필요
    - HTTPS://*.akamaiedge.net - 방화벽이 있는 시스템에만 필요
    - [https://cert.console.redhat.com](https://cert.console.redhat.com)

4-1) 전환 절차

테스트 환경
- AWS EC2 인스턴스 사용
- AMI : ami-09e2a570cb404b37e

a. CentOS Linux에서 변환하는 경우 CentOS 리포지토리 URL을 업데이트 합니다.

sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=https://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

b. `Convert2 RHEL` 설치

0. Centos7 시스템에서 커널 및 패키지 버전 최신화를 진행해줍니다.
# yum update -y
# reboot

1. Red Hat GPG 키를 다운로드합니다.
# curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release https://security.access.redhat.com/data/fd431d51.txt

* https://security.access.redhat.com/data/fd431d51.txt *

2. RHEL 7으로의 변환의 경우 다음 명령을 입력합니다.
# curl -o /etc/yum.repos.d/convert2rhel.repo https://cdn-public.redhat.com/content/public/repofiles/convert2rhel-for-rhel-7-x86_64.repo

3. RHEL 8로의 변환의 경우 다음 명령을 입력합니다.
# curl -o /etc/yum.repos.d/convert2rhel.repo https://cdn-public.redhat.com/content/public/repofiles/convert2rhel-for-rhel-8-x86_64.repo

* 최신 버전의 `Convert2RHEL` 리포지토리 파일을 사용하여 변환을 수행해야 합니다. 이전 버전의 리포지토리 파일을 이전에 설치한 경우 이전 버전을 제거하고 현재 버전을 설치합니다. *

4. Convert2RHEL 설치
# yum -y install convert2rhel

c. convert2RHEL 실행 전 시스템 사전 분석

```
# convert2rhel analyze

Continue with the system conversion? [y/n]: y

...
...

========== Skip (Could not be checked due to other failures) ==========
(SKIP) ENSURE_KERNEL_MODULES_COMPATIBILITY::SKIP - Skipped action
     Description: This action was skipped due to another action failing.
     Diagnosis: Skipped because SUBSCRIBE_SYSTEM was not successful
     Remediations: Please ensure that the SUBSCRIBE_SYSTEM check passes so that this Action can evaluate
your system
(SKIP) VALIDATE_PACKAGE_MANAGER_TRANSACTION::SKIP - Skipped action
     Description: This action was skipped due to another action failing.
     Diagnosis: Skipped because ENSURE_KERNEL_MODULES_COMPATIBILITY and SUBSCRIBE_SYSTEM were not
successful
     Remediations: Please ensure that the ENSURE_KERNEL_MODULES_COMPATIBILITY and SUBSCRIBE_SYSTEM check
passes so that this Action can evaluate your system

========== Overridable (Review and either fix or ignore the failure) ==========
(OVERRIDABLE) IS_LOADED_KERNEL_LATEST::INVALID_KERNEL_VERSION - Invalid kernel version detected
     Description: The loaded kernel version mismatch the latest one available in system repositories
     Diagnosis: The version of the loaded kernel is different from the latest version in system
repositories.
     Latest kernel version available in updates: 3.10.0-1160.119.1.el7
     Loaded kernel version: 3.10.0-1160.76.1.el7
     Remediations: To proceed with the conversion, update the kernel version by executing the following
step:

    1. yum install kernel-3.10.0-1160.119.1.el7 -y
    2. reboot
    If you wish to ignore this message, set the environment variable
'CONVERT2RHEL_SKIP_KERNEL_CURRENCY_CHECK' to 1.

========== Error (Must fix before conversion) ==========   <--- Error 부분에 대해서는 반드시 조치를 진행해야함
(ERROR) SUBSCRIBE_SYSTEM::SYSTEM_NOT_REGISTERED - Not registered with RHSM
     Description: This system must be registered with rhsm in order to get access to the RHEL rpms. In
this case, the system was not already registered and no credentials were given to convert2rhel to
register it.
     Diagnosis: N/A
     Remediations: You may either register this system via subscription-manager before running
convert2rhel or give convert2rhel credentials to do that for you. The credentials convert2rhel would
need are either activation_key and organization or username and password. You can set these in a config
file and then pass the file to convert2rhel with the --config-file option.

```

d. Error List 에 대한 조치 진행

Convert2 RHEL로 Centos 를 RHEL로 변환하기 위해서는 RHSM(Red Hat Subscription Manager)을 통한 Red Hat CDN(Content Delivery Network)에 대한 접근 권한이 있어야 합니다. 이는 "subscription-manager" 를 통해서 아래와 같이 진행할 수 있습니다.

* Subscription-manager을 통해 등록을 진행할 Red Hat 계정이 존재해야합니다. Red Hat 계정에 Developer 이상의 Subscription이 존재해야합니다.

# yum install subscription-manager
# subscription-manager regiter

e. 사전 테스트 상태 검사 결과 해석

- **성공** - 테스트가 성공했으며 이 구성 요소에 대한 문제가 없습니다.
- **Error** - 테스트에서 시스템 상태가 저하될 가능성이 매우 높기 때문에 변환이 실패하는 문제가 발생했습니다. 변환하기 전에 이 문제를 해결해야 합니다.
- **Overridable** - 테스트에서 변환이 실패할 가능성이 매우 높기 때문에 시스템 상태가 저하될 수 있는 문제가 발생했습니다. 이 문제는 변환하기 전에 해결되거나 수동으로 재정의해야 합니다.
- **경고** - 테스트에 변환 후 시스템 및 애플리케이션 문제가 발생할 수 있는 문제가 발생했습니다. 그러나 이 문제로 인해 변환이 실패하지 않습니다.
- **skip** - 사전 요구 사항 테스트 실패로 인해 이 테스트를 실행할 수 없습니다. 변환이 실패할 수 있습니다.
- **info** - 시스템 또는 애플리케이션에 예상되는 영향을 미치지 않는 정보

f. RHEL 시스템으로 변환

1. `Convert2 RHEL` 유틸리티를 시작합니다.
# convert2rhel

2. 성공적으로 전환이 완료 되었는지 확인 합니다.
# cat /etc/os*
NAME="Red Hat Enterprise Linux Server"
VERSION="7.9 (Maipo)"
ID="rhel"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="7.9"
PRETTY_NAME="Red Hat Enterprise Linux Server 7.9 (Maipo)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:7.9:GA:server"
HOME_URL="https://www.redhat.com/"
