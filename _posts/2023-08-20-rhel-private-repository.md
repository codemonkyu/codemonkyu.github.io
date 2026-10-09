---
title: "[Linux] 개인 RHEL Repository 생성하여 사용해보기"
date: 2023-08-20 13:49:45 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["AWS", "Linux", "Repo", "repolist", "rpm", "yum"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-%EA%B0%9C%EC%9D%B8-RHEL-Repository-%EC%83%9D%EC%84%B1%ED%95%98%EC%97%AC-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0
math: true
---

![](/assets/img/posts/rhel-private-repository/01.jpg)

### 목차

- yum.repo.d 알아보기  
- RPM vs YUM  
- 특정 패키지 다운로드하기  
- 사설 Repository 생성  
- 사설 Repository 사용

RHEL 리눅스를 사용하는 경우 Red hat 레포지토리에서 원하는 패키지를 편리하게 설치하고 사용할 수 있다.  
레포지토리에서 패키지를 가져오는 과정이 어떻게 이루어지는지 궁금하였고, 해당 부분을 직접 설정하여 필요한 패키지만 모아 놓은 사설 레포지토리를 생성하고 해당 레포지토리를 외부에서도 사용이 가능하도록 해보고 싶었다.

또한 보안을 목적으로 외부와 단절되어 있는 서버인 경우 외부와 인터넷을 통해 통신할 수 없기 때문에 레포지토리를 사용할 수 없는 환경일 수 있는데, 이러한 상황에서도 위의 과정을 알고 있으면 도움이 될 수 있을 것이다.  
(* 물론 클라우드 환경에서는 NAT Gateway를 통해 Private 망에 있는 인스턴스 또한 인터넷 액세스가 가능하다. 하지만 폐쇠적인  
네트워크 망을 구축하여 사용중이라면 위의 과정을 알아두어서 나쁠것은 없어보인다.)

---

### "/etc/yum.repo.d" 살펴보기

**경로 : /etc/yum.repos.d**  
위의 경로에는 " *.repo " 형태의 파일이 존재하며, 해당 파일에는 아래와 같이 레포지토리에 대한 정보가 정의 되어 있다.

*** RHEL 9의 /etc/yum.repos.d 경로에 존재 하는 파일**  
---  
redhat-rhui-beta.repo.disabled  
redhat-rhui-client-config.repo  
redhat-rhui.repo  
---

**- *.redhat-rhui.repo 파일 -**

> ```  
> [rhel-9-baseos-rhui-rpms]  
> name=Red Hat Enterprise Linux 9 for $basearch - BaseOS from RHUI (RPMs)  
> mirrorlist=  
> https://rhui.REGION.aws.ce.redhat.com/pulp/mirror/content/dist/rhel9/rhui/$releasever/$basearch/baseos/os  
> enabled=1  
> gpgcheck=1  
> gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release  
> sslverify=1  
> sslclientkey=/etc/pki/rhui/content-rhel9.key  
> sslclientcert=/etc/pki/rhui/product/content-rhel9.crt  
> sslcacert=/etc/pki/rhui/cdn.redhat.com-chain.crt  
> ```

**-  *.repo 파일의 주요 구성 요소 -**  
(1). Repository ID : A one word unique repository ID e.g. [localrepo].  
(2). Name : A human readable name for the repository e.g. name=Awesome Local Repo  
(3). Baseurl : A URL to the repodata directory (where the actual files are kept). file://path, ftp://link, http://link, and https://link addresses are all valid options.  
(4). Enabled : Whether or not to enable the repository for use when performing updates and installs e.g. enabled=1 (1 - "use this repository", 0 - "do not use this repository").  
(5). gpgcheck : GPG(GNU Privacy Guard) 키가 들어있는 저장소의 URL을 적습니다. GPG 키는 rpm 패키지를 인증하는데 사용하는 암호화 서명입니다.

(6). mirrorlist :  mirrorlist에 등록된 경로를 자동으로 찾고 그 사이트를 이용하여 업데이트

참고자료 : [What's inside an RPM .repo file?](https://developers.redhat.com/articles/2022/10/07/whats-inside-rpm-repo-file)

---

**RPM vs YUM**

- RPM(Red Hat Package Manager) vs YUM(Yellow Dog Updater, Modified) [링크](https://phoenixnap.com/kb/rpm-vs-yum)

패키지 관리자 RPM과 YUM의 차이점

[RPM vs. YUM: Learn the Difference

RPM and YUM are package managers used in the Red Hat Linux distros. This comparison article lists the key differences between them.

phoenixnap.com](https://phoenixnap.com/kb/rpm-vs-yum)

#### 참고

- RPM은 리포지토리를 지원하지 않으므로 패키지를 설치하려면 전체 패키지 파일 이름과 위치가 필요합니다.RPM은 설치된 패키지의 내부 데이터베이스를 보관하며 나중에 패키지 이름을 사용하여 설치된 패키지를 관리할 수 있도록 합니다.
- 반면 YUM으로 패키지를 설치하려면 패키지 이름만 필요하고 패키지 위치는 필요하지 않습니다.이 도구는 /etc/yum.repos.d/ 디렉터리에서 repo 파일에 나열된 리포지토리를 검색하고 사용 가능한 최신 패키지 버전을 설치합니다.

### 특정 패키지 다운로드하기

> 특정 패키지를 설치할 때 private 망의 서버에서는 패키지 관리자 "yum" 을 사용하여 레포지토리에 등록된 패키지를 설치할 수 없다. 때문에 이러한 상황에서, 인터넷이 가능한 서버에서 rpm 패키지를 다운로드 받아 Private 서버로 이동하여 패키지 설치를 진행하는데 이때 필요한 "yumdownloader" 명령어를 알아보자. ### yum-utils 패키지 설치 yumdownloader 명령어가 있는 yum-utils 패키지를 설치해야 합니다.

```bash
$ sudo yum install yum-utils
```

yumdownloader 명령어에 --downloadonly 옵션을 주고 다운받을 패키지를 지정하면 현재 폴더에 해당 패키지만 다운로드 됩니다.

```bash
$ yumdownloader --downloadonly gcc
```

- --downloadonly 옵션을 사용하면 특정 패키지만 다운로드 되기 때문에 해당 패키지 설치를 위해 --resolve 옵션 사용하여 의존성 패키지도 같이 다운로드 해준다.  
- yum deplist 명령어를 통해 특정 패키지의 의존성 패키지가 무엇이 있는지 확인할 수 있다.

```bash
$ yum deplist yum-utils

dependency: /usr/bin/python
   provider: python.x86_64 2.7.18-1.amzn2.0.6
  dependency: libxml2-python
   provider: libxml2-python.x86_64 2.9.1-6.amzn2.5.8
  dependency: python(abi) = 2.7
   provider: python.x86_64 2.7.18-1.amzn2.0.6
  dependency: python-kitchen
   provider: python-kitchen.noarch 1.1.1-5.amzn2
  dependency: yum >= 3.4.3-143
   provider: yum.noarch 3.4.3-158.amzn2.0.6
```

```bash
$ yumdownloader --resolve gcc

Finished Dependency Resolution
(1/13): cpp-7.3.1-15.amzn2.x86_64.rpm                                                           | 9.2 MB  00:00:00
(2/13): glibc-devel-2.26-63.amzn2.x86_64.rpm                                                    | 995 kB  00:00:00
(3/13): glibc-headers-2.26-63.amzn2.x86_64.rpm                                                  | 516 kB  00:00:00
(4/13): libatomic-7.3.1-15.amzn2.x86_64.rpm                                                     |  46 kB  00:00:00
(5/13): libcilkrts-7.3.1-15.amzn2.x86_64.rpm                                                    |  85 kB  00:00:00
(6/13): gcc-7.3.1-15.amzn2.x86_64.rpm                                                           |  22 MB  00:00:00
(7/13): libitm-7.3.1-15.amzn2.x86_64.rpm                                                        |  85 kB  00:00:00
(8/13): kernel-headers-5.10.186-179.751.amzn2.x86_64.rpm                                        | 1.4 MB  00:00:00
(9/13): libmpc-1.0.1-3.amzn2.0.2.x86_64.rpm                                                     |  52 kB  00:00:00
(10/13): libmpx-7.3.1-15.amzn2.x86_64.rpm                                                       |  51 kB  00:00:00
(11/13): libquadmath-7.3.1-15.amzn2.x86_64.rpm                                                  | 189 kB  00:00:00
(12/13): libsanitizer-7.3.1-15.amzn2.x86_64.rpm                                                 | 642 kB  00:00:00
(13/13): mpfr-3.1.1-4.amzn2.0.2.x86_64.rpm
```

#### gcc 패키지 및 의존성 패키지 설치

```bash
$sudo yum localinstall *.rpm

=======================================================================================================================
 Package             Arch        Version                      Repository                                          Size
=======================================================================================================================
Installing:
 cpp                 x86_64      7.3.1-15.amzn2               /cpp-7.3.1-15.amzn2.x86_64                          24 M
 gcc                 x86_64      7.3.1-15.amzn2               /gcc-7.3.1-15.amzn2.x86_64                          59 M
 glibc-devel         x86_64      2.26-63.amzn2                /glibc-devel-2.26-63.amzn2.x86_64                  1.1 M
 glibc-headers       x86_64      2.26-63.amzn2                /glibc-headers-2.26-63.amzn2.x86_64                2.2 M
 kernel-headers      x86_64      5.10.186-179.751.amzn2       /kernel-headers-5.10.186-179.751.amzn2.x86_64      5.1 M
 libatomic           x86_64      7.3.1-15.amzn2               /libatomic-7.3.1-15.amzn2.x86_64                    26 k
 libcilkrts          x86_64      7.3.1-15.amzn2               /libcilkrts-7.3.1-15.amzn2.x86_64                  124 k
 libitm              x86_64      7.3.1-15.amzn2               /libitm-7.3.1-15.amzn2.x86_64                      125 k
 libmpc              x86_64      1.0.1-3.amzn2.0.2            /libmpc-1.0.1-3.amzn2.0.2.x86_64                   111 k
 libmpx              x86_64      7.3.1-15.amzn2               /libmpx-7.3.1-15.amzn2.x86_64                       34 k
 libquadmath         x86_64      7.3.1-15.amzn2               /libquadmath-7.3.1-15.amzn2.x86_64                 284 k
 libsanitizer        x86_64      7.3.1-15.amzn2               /libsanitizer-7.3.1-15.amzn2.x86_64                2.8 M
 mpfr                x86_64      3.1.1-4.amzn2.0.2            /mpfr-3.1.1-4.amzn2.0.2.x86_64                     553 k

Transaction Summary
=======================================================================================================================
Install  13 Packages

Total size: 95 M
Installed size: 95 M
Is this ok [y/d/N]:
```

---

### yum 명령어의 동작 방식

![](/assets/img/posts/rhel-private-repository/02.png)

```basic
1. yum install 로 패키지 설치 명령을 실행합니다.
2. /etc/yum.repos.d/ 디렉터리의 여러 설정파일들 및 CentOS-Base.repo 파일을 참고하여 인터넷상의 패키지 저장소 서버 주소를 얻습니다.
3 ~ 4. 패키지 목록을 저장소 서버로부터 얻습니다.(실제 패키지는 받지 않고 목록만 가져옴)
5. 가져온 패키지 목록을 터미널에 출력하고 설치할 것인지 여부(Y/N)를 묻습니다.
6. Y를 입력하면 패키지를 다운로드하여 자동으로 설치합니다. 만약 yum install -y 옵션을 사용했다면 여부를 묻지 않고 모든 과정을 자동으로 진행합니다.
```

(* 출처 : <https://dololak.tistory.com/332>)

---

### 사설 Repository 생성

외부에 존재하는 레포지토리 전체를 "reposync" 명령어를 통해 로컬에 다운로드 한다.  
reposync하는 시점에 해당 레포지토리에 release된 패키지들은 다운로드 하게 된다.

#### reposync 를 이용하여 특정 레포지토리에 release 되어 있는 패키지 다운로드하기

```bash
$ mkdir testrepo
$ cd testrepo
$ reposync -r [repo-name]
$ tree | more
```

```angelscript
[root@ip-172-31-7-80 testrepo]# tree | more
.
└── rhel-9-baseos-rhui-rpms
    └── Packages
        ├── a
        │   ├── accel-config-3.4.2-2.el9.i686.rpm
        │   ├── accel-config-3.4.2-2.el9.x86_64.rpm
        │   ├── accel-config-3.4.6.3-1.el9.i686.rpm
        │   ├── accel-config-3.4.6.3-1.el9.x86_64.rpm
        │   ├── accel-config-libs-3.4.2-2.el9.i686.rpm
        │   ├── accel-config-libs-3.4.2-2.el9.x86_64.rpm
        │   ├── accel-config-libs-3.4.6.3-1.el9.i686.rpm
        │   ├── accel-config-libs-3.4.6.3-1.el9.x86_64.rpm
        │   ├── acl-2.3.1-3.el9.x86_64.rpm
        │   ├── acpica-tools-20210604-3.el9.x86_64.rpm
```

이렇게 다운로드가 완료된 패키지들은 **Packages** 라는 디렉터리 안에 저장이 되어 있다.  
reposync을 통해 다운로드한 패키지들을 rpm / localinstall 명령어를 사용하여 설치는 가능하지만 패키지 관리자인 yum을 이용해서 dependency까지 설치할 수는 없는 상태이다. (* 단순히 레포지토리에 있는 패키지들만 가져온 상태라고 생각하면 된다.)

때문에 해당 패키지들에 대해서 "repodata"를 생성해야지 **사설 레포지토리** 로 사용이 가능하다. repodata를 생성하기 위해서는 명령어 **createrepo**를 사용해야하며, 이를 위해 아래 명령어를 통해 해당 패키지를 설치해준다.

#### createrepo 를 사용하여 repodata 생성하기

```cmake
$ yum install createrepo -y
```

```angelscript
$ [root@ip-172-31-7-80 rhel-9-baseos-rhui-rpms]# pwd
/root/testrepo/rhel-9-baseos-rhui-rpms

$ [root@ip-172-31-7-80 rhel-9-baseos-rhui-rpms]# ls
Packages

$ [root@ip-172-31-7-80 rhel-9-baseos-rhui-rpms]# createrepo .
Directory walk started
Directory walk done - 2926 packages
Temporary output repo path: ./.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished

$ [root@ip-172-31-7-80 rhel-9-baseos-rhui-rpms]# ls
Packages  repodata
```

---

## 사설 Repository 사용

아래와 같이 **/etc/yum.repos.d** 경로 밑에 repo 파일을 생성합니다.

```ini
[root@ip-172-31-7-80 yum.repos.d]# cat test.repo
[private-testrepo]
name="TEST rhui base - testropo"
baseurl=file:////root/testrepo/rhel-9-baseos-rhui-rpms
gpgcheck=0
enable=1
```

repolist 명령어를 통해 레포지토리가 정상적으로 사용가능한지 확인합니다.

```angelscript
[root@ip-172-31-7-80 yum.repos.d]# yum repolist
repo id                              repo name
private-testrepo                     TEST rhui base - testropo
```

---

### 사설 Repository를 외부에서 이용 가능하도록  변경

내가 만든 사설 레포지토리를 다른 클라이언트에서 사용하고 싶다면 해당 레포지토리를 간단하게 httpd을 사용해 다른 클라이언트에서 이를 사용할 수 있도록 해주면 된다.

1. httpd 설치 및 repodata 생성

- $yum install httpd
- $cd /var/www/html
- $mkdir repo <---repo폴더 생성
- $cp packages /var/www/html/repo
- $Createrepo . <- /var/www/html 경로에서 다시 repodata를 만들어준다.
- http://인스턴스공인IP/repo 로 접속 - (e.g. 3.34.43.6/repo/)

![](/assets/img/posts/rhel-private-repository/03.png)

2. 해당 repodata를 사용할 클라이언트 인스턴스 생성 및 레포지토리 등록 설정

- $cd /etc/yum.repos.d
- $vim test.repo  
-baseurl을 사설 레포지토리가 존재하는 인스턴스의 주소로 변경

```ini
# cat test.repo
[private-testrepo]
name="TEST rhui base - testropo"
baseurl=http://3.34.43.6/repo
gpgcheck=0
enable=1
```
- $yum list & yum repolist 으로 확인

```applescript
repo id                        repo name
private-testrepo               TEST rhui     base - testropo
```
