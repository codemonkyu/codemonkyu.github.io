---
title: "[Linux] In-place upgrade of RHEL 7"
date: 2025-01-04 23:28:23 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["in-place", "Redhat", "rhel7", "Upgrade", "업그레이드", "인플레이스"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-In-place-upgrade-of-RHEL-7
---

![](/assets/img/posts/rhel7-in-place-upgrade/01.png)

이번 포스팅에서는 RHEL 7 에서 RHEL 8 으로 In-Place(*인-플레이스) 업그레이드를 진행하는 방법에 대하여 다루어 보려고 한다.

**인플레이스 업그레이드**는 기존의 시스템 환경을 유지 하면서 OS의 버전을 업그레이드 하는 방법을 의미한다.

따라서 인플레이스 업그레이드 방법을 사용하면 새로운 OS 버전을 신규 설치 할 때에 비해 아래와 같은 [이점이](https://www.redhat.com/ko/blog/what-are-rhel-place-upgrades) 있다.

![출처 : https://www.redhat.com/ko/blog/what-are-rhel-place-upgrades](/assets/img/posts/rhel7-in-place-upgrade/02.png)
_출처 : https://www.redhat.com/ko/blog/what-are-rhel-place-upgrades_

---

## 인플레이스 업그레이드 프로세스

* RHEL 7에서 RHEL 8로의 인플레이스 업그레이드 프로세스는 다음과 같이 요약할 수 있다.

1. **업그레이드 계획**- 시스템 요구 사항 및 제한 사항을 검토합니다. 시스템이 인플레이스 업그레이드에 적합한 후보인지 또는 대신 RHEL 8을 새로 설치해야 하는지 확인합니다.

2. **업그레이드 준비**- 업그레이드 프로세스를 시작하기 전에 RHEL 7 시스템의 백업 생성을 포함하여 필요한 준비 단계를 완료합니다.

3. **업그레이드 전 보고서를 실행하고 검토**- 발견된 문제를 해결하고 권장 솔루션을 적용합니다. 업그레이드 전 유틸리티를 다시 실행하여 모든 심각한 문제가 해결되었는지 확인합니다. 시스템이 업그레이드를 준비하기 전에 업그레이드 전 보고서를 실행하고 발견된 문제를 여러 번 해결해야 할 수 있습니다.  
   - 인플레이스 업그레이드를 수행하는 대신 RHEL 8의 새로 설치로 전환합니다.  
   - 업그레이드 전 유틸리티를 실행하여 업그레이드 전에 해결해야 하는 잠재적인 문제를 요약하는 보고서를 생성합니다.  
   - 발견된 문제의 심각도 및 영향에 따라 문제를 해결하는 데 필요한 작업 양에 따라 다음 결과 중 하나를 진행합니다.

4. **인플레이스 업그레이드를 수행**- RHEL 8로 업그레이드를 수행하고 업그레이드가 올바르게 완료되었는지 확인합니다. 업그레이드할 수 없는 문제가 발생하면 RHEL 7 백업으로 롤백하십시오.

5. **업그레이드 후 단계 수행**- 필요한 업그레이드 후 단계를 수행하여 RHEL 8 시스템이 올바르게 설정되었는지 확인합니다.

---

## 업그레이드 준비

인플레이스 업그레이드를 위해서는 System을 최신의 버전으로 업데이트 후 업그레이드에 제한이 존재하는 부분이 있는지 확인해야 한다.

확인 및 준비 절차는 아래의 순서대로 진행할 수 있다.

1. 시스템 업데이트:
   - 최신 마이너 릴리스로 시스템을 업데이트합니다.
   - yum update 명령을 사용하여 모든 패키지를 최신 버전으로 업데이트합니다.
2. 저장소 구성:
   - 필요한 저장소를 활성화합니다. 예: RHEL 7의 경우 'rhel-7-server-rpms'와 'rhel-7-server-extras-rpms'
   - subscription-manager repos --enable=<repo-name> 명령을 사용합니다.
3. 버전 잠금 해제:
   - subscription-manager의 버전 잠금을 해제합니다: subscription-manager release --unset
   - YUM의 버전 잠금을 해제합니다(해당되는 경우): yum versionlock clear
4. Leapp 도구 설치:
   - yum install leapp-upgrade 명령을 사용하여 Leapp 도구를 설치합니다.
5. 시스템 재부팅:
   - 모든 변경사항을 적용하기 위해 시스템을 재부팅합니다.
6. Leapp 도구 확인:
   - leapp --version으로 버전을 확인합니다.
   - leapp --help와 man leapp로 사용법을 검토합니다.

## 업그레이드 준비  - 명령어

**1. 업그레이드 하려는 시스템의 OS 정보를 확인 합니다.**

root@~]# **cat /etc/redhat-release**  
Red Hat Enterprise Linux Server release 7.9 (Maipo)

**2. Red Hat Subscription-Manager에 시스템이 등록되어 있는지 확인 합니다.**

[root@~]# subscription-manager identity  
system identity: 7b36d575-c923-4b4f-a689-886890f9acc3  
name: serverc  
org name: Operations  
org ID: Operations  
environment name: Library/leapp-upgrade

**3. 시스템 전체에 대한 업데이트를 진행합니다.**

[root@~]# yum update  
...output omitted...  
Is this ok [y/d/N]: y  
...output omitted...  
Complete!

**4. 인플레이스 업그레이드에 필요한 Repository가 존재하는지 확인 합니다.**

[root@~]# yum repolist  
...output omitted...  
repo id repo name status  
!rhel-7-server-rpms/7Server/x86_64 Red Hat Enterprise Linux 7 Server (RPMs) 34,270  
repolist: 34,270

*인플레이스 업그레이드를 위해서는 **'rhel-7-server-rpms'와 'rhel-7-server-extras-rpms' 두 레포지토리가 필요합니다. 5. 아래의 명령어를 통해 필요한 'rhel-7-server-extras-rpms' 레포지토리를 활성화 합니다.**

[root@~]# subscription-manager repos --enable rhel-7-server-extras-rpms  
Repository 'rhel-7-server-extras-rpms' is enabled for this system.

**6. 레포지토리가 정상적으로 활성화 되었는지 확인 합니다.**

[root@~]# yum repolist  
...output omitted...  
repo id repo name status  
rhel-7-server-extras-rpms/x86_64 Red Hat Enterprise Linux 7 Server - Extras (RPMs) 1,468  
rhel-7-server-rpms/7Server/x86_64 Red Hat Enterprise Linux 7 Server (RPMs) 34,270  
repolist: 35,738

**7. subscription-manager에 설정된 버전 고정이 있으면 정상적으로 업그레이드가 되지 않기 때문에 이를 해제 합니다.**

[root@~]# **subscription-manager release --unset**  
Release preference has been unset

**8. yum 패키지 관리자에 패키지 versionlock이 되어 있으면 이를 해제 합니다.**

[root@~]# yum versionlock clear  
Loaded plugins: product-id, search-disabled-repos, subscription-manager, versionlock  
versionlock cleared

**9. 인플레이스 업그레이드에 필요한 leapp-upgrade 패키지를 설치합니다.**

[root@~]# yum install leapp-upgrade  
...output omitted...  
Is this ok [y/d/N]: y  
...output omitted...  
Complete!

---

## 업그레이드 전 보고서를 실행하고 검토

이제 인플레이스 업그레이드를 위한 준비가 끝났으며, 업그레이드를 진행하기 전 '사전 업그레이드 보고서' 를 작성한 후 실제 업그레이드 시 문제가 발생할 부분이 없는지 확인을 진행한다.

1. 사전 업그레이드 보고서 생성:
   - leapp preupgrade --target 8.9 명령을 사용하여 RHEL 7.9에서 RHEL 8.9로의 업그레이드를 위한 사전 분석을 실행합니다.
2. 보고서 검토:
   - 생성된 보고서는 /var/log/leapp/leapp-report.txt에서 확인할 수 있습니다.
   - 보고서는 오류, 억제요인(inhibitors), 그리고 다양한 심각도의 보고 사항들을 포함합니다.
3. 주요 문제 해결: a) 커널 모듈 문제:
   - pata_acpi와 floppy 커널 모듈이 RHEL 8에서 제거되었음을 확인.
   - 이 모듈들이 필요하지 않다고 판단되면 rmmod 명령으로 제거.b) Leapp 답변 파일 문제:
   - /var/log/leapp/answerfile에서 필요한 사용자 선택 사항을 확인.
   - pam_pkcs11 모듈 비활성화 확인을 요구하는 경우, 답변 파일에서 confirm = True로 설정.
4. 머신 ID 문제:
   - /etc/machine-id 파일이 비어있는 경우 발생하는 오류 해결.
   - dbus-uuidgen --ensure=/etc/machine-id 명령으로 새 머신 ID 생성.
5. 재분석:
   - 문제 해결 후 leapp preupgrade --target 8.9 명령을 다시 실행하여 모든 문제가 해결되었는지 확인.

주요 포인트

- 사전 업그레이드 분석은 실제 업그레이드 전 잠재적 문제를 식별하고 해결하는 중요한 단계입니다.
- 보고된 모든 억제요인(inhibitors)은 반드시 해결해야 합니다.
- 일부 문제는 시스템 특정적일 수 있으며, 환경과 애플리케이션에 따라 해결 방법이 다를 수 있습니다.
- 문제 해결 후 항상 재분석을 수행하여 새로운 문제가 발생하지 않았는지 확인해야 합니다.

## 업그레이드 전 보고서를 실행하고 검토 - 명령어

**1. 아래 명령어를 통해서 업그레이 전 사전 보고서를 생성합니다.**

[root@serverc ~]# **leapp preupgrade --target 8.9**  
...output omitted...  
Debug output written to /var/log/leapp/leapp-preupgrade.log

**2. 사전 보고서가 생성된 해당 *경로에 Report 파일이 생성되며 Upgrade 성공 유무 및 조치해야 할 사항이 있으면 이를 알려준다.**

* /var/log/leapp/leapp-report.txt 파일에서 보고서를 검사하고 보고된 모든 문제를 수동으로 해결해야한다. 보고된 일부 문제에는 수정 제안 사항이 포함되어 있습니다. **억제*inhibitor*** 문제로 인해 해결될 때까지 업그레이드할 수 없습니다.

[root@~]# **cat /var/log/leapp/leapp-report.txt**  
Risk Factor: high (inhibitor)  
Title: Leapp detected loaded kernel drivers which have been removed in RHEL 8.  
Upgrade cannot proceed.  
Summary: Support for the following RHEL 7 device drivers has been removed in RHEL 8:  
- **pata_acpi**  
- **floppy**

[root@~]# **cat /var/log/leapp/leapp-report.txt**  
...output omitted...  
Risk Factor: high (inhibitor)  
Title: **Missing required answers in the answer file**  
Summary: One or more sections in answerfile are missing user choices: removepam_pkcs11_module_check.confirm  
For more information consult <https://red.ht/leapp-dialogs>.  
...output omitted..._

- Review the /var/log/leapp/answerfile file and confirm the tasks that the Leapp tool recommends.

**3. 특정 시스템 구성에서 Leapp은 수동으로 응답해야 하는 true 또는 false 질문을 생성합니다. 사전 업그레이드 보고서에 응답 파일 메시지에 Missing 필수 응답이 포함된 경우 다음 단계를 완료합니다.**

* /var/log/leapp/answerfile 파일을 수동으로 편집하고, # 기호를 삭제하여 파일의 confirm 줄의 주석을 제거하고, 응답을 True 또는 False 로 확인합니다.

[root@~]# cat /var/log/leapp/answerfile

```routeros
[remove_pam_pkcs11_module_check]
# Title:              None
# Reason:             Confirmation
# =================== remove_pam_pkcs11_module_check.confirm ==================
# Label:              Disable pam_pkcs11 module in PAM configuration?
                      If no, the upgrade process will be interrupted.
# Description:        PAM module pam_pkcs11 is no longer available in RHEL-8
                      since it was replaced by SSSD.
# Reason:             Leaving this module in PAM configuration may
                      lock out the system.
# Type:               bool
# Default:            None
# Available choices: True/False
# Unanswered question. Uncomment the following line with your answer
# confirm =

The Leapp tool requires confirmation to disable the pam_pkcs11 kernel module. The serverc system is not managed by Identity Management in Red Hat Enterprise Linux, which requires this module, so you can safely remove the pam_pkcs11 module.
```

**4. 사전 보고서에서 요구하는 모든 조치를 진행하였으면, 다시 pre-upgrade 분석을 진행합니다. - Errors 또는 Inhibitors가 발생하지 않음을 확인합니다.**

```bash
[root@ ~]# leapp preupgrade --target 8.9
...output omitted...
============================================================
                      REPORT OVERVIEW
============================================================

HIGH and MEDIUM severity reports:
    1. Packages available in excluded repositories will not be installed
    2. Difference in Python versions and support in RHEL 8
    3. GRUB2 core will be automatically updated during the upgrade
    4. Module pam_pkcs11 will be removed from PAM configuration
    5. chrony using default configuration

Reports summary:
    Errors:                      0
    Inhibitors:                  0
    HIGH severity reports:       3
```

---

## 업그레이드 진행 및 확인

**1. 업그레이드를 진행합니다.**

[root@~]# **leapp upgrade --target 8.9**  
...output omitted...

[root@~]# **reboot**  
...output omitted...

**2. 업그레이드 진행 후 아래의 작업을 통해 업그레이드 이후 시스템에 필요 없는 부분들을 정리하며 확인한다.

2-1. 아래의 명령어들을 확인하여 인플레이스 업그레이드가 정상적으로 진행 되었는지 시스템 확인**

```shell
# subscription-manager release
# hostnamectl
# uname -r
# cat /etc/redhat-release
```

**2-2. 시스템이 정상적으로 업그레이드 된 것을 확인하면 이하의 작업을 진행하여 업그레이든 된 시스템에 대한 조치를 진행한다.**

```vim
a) Leapp 패키지 제거:
    /etc/dnf/dnf.conf 파일에서 exclude 목록 확인 및 수정
    dnf remove 명령으로 Leapp 관련 패키지 제거

b) 오래된 패키지 제거:
    rpm -qa | grep 명령으로 이전 버전 패키지 확인 및 제거

c) 저장소 업데이트:
    불필요한 저장소 비활성화
    dnf config-manager 명령 사용

d) 복구 커널 및 RAM 디스크 업데이트:
    기존 복구 커널 제거 및 새 버전 설치

e) 보안 정책 복원:
    SELinux를 enforcing 모드로 설정
    시스템 전체 암호화 정책 검토 및 설정

운영 재개:
    애플리케이션 구성 변경
    구성 관리 도구 운영 재개
    시스템 모니터링 재개
    애플리케이션 테스트 및 로그 모니터링
```

* [참고 문서](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/8/html/upgrading_from_rhel_7_to_rhel_8/performing-post-upgrade-tasks-rhel-7-to-rhel-8_upgrading-from-rhel-7-to-rhel-8#performing-post-upgrade-tasks-rhel-7-to-rhel-8_upgrading-from-rhel-7-to-rhel-8)

[8장. 업그레이드 후 작업 수행 | Red Hat Product Documentation

형식멀티 페이지단일 페이지모든 문서를 PDF로 표시

docs.redhat.com](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/8/html/upgrading_from_rhel_7_to_rhel_8/performing-post-upgrade-tasks-rhel-7-to-rhel-8_upgrading-from-rhel-7-to-rhel-8#performing-post-upgrade-tasks-rhel-7-to-rhel-8_upgrading-from-rhel-7-to-rhel-8)

---

이번 포스팅을 통해 Red Hat Enterprise Linux의 In-place 업그레이드를 진행하는 방법 및 진행 후 시스템을 안정화 시키는 프로세스를 정리해 보았다. 실제 운영환경에 따라 인플레이스 업그레이드 시 발생할 수 있는 상황을 모두 정리할 수 없지만, 프로세스에 대한 기본적인 틀을 알고 있으면 모든 상황에 유연하게 대처할 수 있을 것으로 생각된다.
