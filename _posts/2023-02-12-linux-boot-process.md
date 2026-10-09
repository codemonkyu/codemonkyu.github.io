---
title: "[Linux] Linux 부팅 과정 알아보기"
date: 2023-02-12 11:38:48 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: []
tistory_url: https://codemonkyu.tistory.com/entry/Linux-Linux-%EB%B6%80%ED%8C%85-%EA%B3%BC%EC%A0%95-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0
---

리눅스 시스템에서 부팅이라 부르는 과정을 정리해보겠다.

부팅과정을 알면 아래의 상황에서 도움이 될 수 있다.

> 1) 부팅 관련 장애 발생에서 원인을 찾을 때 도움이 된다.  
> 2) 부트로더 에러 및 커널패닉 같은 자주 발생하는 부팅에러를 해결할 때 도움이 된다.  
> 3) 시스템이 어떻게 하드웨어를 인식하는지 알 수 있다.

부팅 과정은 아래의 순서로 진행된다.

![Linux Boot Process](/assets/img/posts/linux-boot-process/01.png)
_Linux Boot Process_

---

## BIOS (Basic INPUT / OUTPUT System)

- - This is the first step toward the Linux boot process.
- - BIOS is a very important and very first part of booting any operating system like Linux, Windows, or macOS.
- - Now we have  UEFI which stands for Unified Extensible Firmware Interface.
- - BIOS first initializes CPU because this must function at first and then RAM. BIOS placed its copy to RAM and starts running from RAM. Next, it then starts all hardware like PCI bus, USB interfaces, etc.
- - BIOS also runs a test called POST to ensure all hardware is working properly.

>  시스템에 전원이 공급되면, 메인보드의 ROM에 저장된 BIOS프로그램이 실행된다. BIOS프로그램은 이때 메모리(RAM)의 특정 부분(ffff:0000)에 로드된다. 그래서 CPU는 전원공급이 되면 메모리의 특정 부분을 읽어 실행한다. 이렇게 BIOS는 실행된다.  
>  BIOS가 실행되면서 POST(Power on Self Test)를 진행하면서 CMOS검사, CPU,Memory, 그래픽카드등 디바이스의 이상 유무를 검사하고, 사용할 수 있게 초기화 시킨다. 이상이 없으면 부팅매체의 GRUB을 읽어온다. 부팅매체를 및 순서를 BIOS 및 UEFI를 사용해서 세팅할 수 있는데, 1번 하드디스크가 부팅 순서가 1순위일 경우 해당 하드디스크의 0번섹터(MBR, Master Boot Record)에 있는 부트로더(GRUB)을 읽어 메모리로 불러온다.

( * ROM, read-only memory  고정 기억 장치 또는 롬은 반도체 기억 장치의 하나로 컴퓨터를 구동하기 위한 기본적인 정보가 담겨져 있다.)

---

## MBR / GPT

- - This is the second step teoward the Linux boot process.
- - MBR (Master Boot Record), It's a legacy standard for wokring with disk partitioning. And it can have a max of 4 partitions with each not more than 2TB.
- - GPT stands for Grd partition table.
- - With GPT we can have up to 128 partition tables on a single hard disk.
- - In Linux, we work with MBR using the command-line utility "gdisk"
- - MBR is located at the sector of the bootable disk. Example are /dev/sda
- - Most disk is divided in to sectors and the default sector size is 512 bytes. So MBR is a smaal file of 512 bytes in size.
- - Now this 512-byte file called MBR contains information about the boot loader called GRUB. Once GRUB found we hand over control to it.

>  대부분의 디스크는 섹터로 나뉘어져 있으며 한 섹터의 사이즈는 512bytes이다. 즉 GRUB이라 불리는 부트로더에 대한 정보를 담고 있는 곳을 **MBR(Master Boot Record)**이라 부르며 MBR은 부팅 디스크의 첫번째 섹터에 위치해있다. GRUB을 읽어와 실행 되고 나서는 GRUB에게 컨트롤권이 넘어간다.

---

## Bootloader (GRUB/LILO)

- - This is the third step toward the Linux boot process.
- - GRUB stands for Grand Unified Bootloader.
- - In Linux, this partition. points to Kernel and initrd(init ram disk) files which are typically stored in /boot.

![](/assets/img/posts/linux-boot-process/02.png)

![*($ ls /boot command on Amazon Linux 2022)](/assets/img/posts/linux-boot-process/03.png)
_*($ ls /boot command on Amazon Linux 2022)_

- - Mostly, the GRUB2 configuration is located at /boot/grub2/grub.cfg and we should not edit this file.
- - If you want to add custom changes to  the GRUB config you can edit the file like 40_custom or 41_custom located in the /etc/grub.d directory. Once you have added your customer GRUB config, run grub2-mkconfig command to build a new configuration.
- - So, at first, Bootloader finds the kernel and loads it in RAM and then looks for file initrd or Initramfs which basically root file system. The kernel mounts the Initramfs which then accesses all files and then turns control over to the kernel.
- - Both kernel and initramfs shoud have the same version number as initramfs file is build specifically to the kernel.

![MBR to GRUB](/assets/img/posts/linux-boot-process/04.png)
_MBR to GRUB_

>  GRUB은 /boot/grub 파일 시스템에 직접 접근하여 커널(vmlinuz)의 압축을 풀어 메모리에 로드하고, 커널이 필요로 하는 모든 드라이버와 모듈, 파일시스템(ext2, ext3, ext4...)등이 담긴 RAM 디스크 파일(initrd.img)를  메모리에 로드 한다. Kernel과 initramfs은 initramfs 파일이 커널에 맞게 빌드되었기 때문에 서로 동일한 버전이 로드되어야 한다.  
>

* Vmliuz file

<https://askubuntu.com/questions/452070/what-is-vmlinuz-file-on-live-ubuntu>

[what is Vmlinuz file on live ubuntu?

I use Ubuntu 14.04 live on flash. In my computer directory there is a file name vmlinuz What is this file? I did not have that on my 12.04 on hard derive.

askubuntu.com](https://askubuntu.com/questions/452070/what-is-vmlinuz-file-on-live-ubuntu)

---

## Kernel

- This is the 4th step toward the Linux boot process.
- After the kernel gets control, its job is to launch or execute initialization scripts called **systemd** or **init**.
- Systemd is a new program that is used in all Latest Linux operating systems.
- The Kernel uses the initial RAM disk (**initramfs**) as a temporary file system until the physical file system is mounted.
- Since the system process is first executed by the kernel, it has a process id of **1**.

![](/assets/img/posts/linux-boot-process/05.png)

> 커널이 제어권을 얻은 후에는 systemd 또는 init라는 초기화 스크립트를 실행하거나 실행한다. 또한 커널은 잠시동안 initial RAM disk (initramfs)를 임시 파일 시스템으로 사용한다. 또한 커널은 하드웨어 드라이버를 포함하거나 initramfs에서 하드웨어 모듈을 로드합니다. (버전이 같아야 하는 이유)

* What is the Linux kernel?  
<https://www.redhat.com/en/topics/linux/what-is-the-linux-kernel>

---

## Systemd

- This is the fifth step toward the Linux boot process.
- **Systemd** is a next-generation daemon designed to replace the old SysVinit process.
- Its job is to detect the hardware changes happening in real-time on the system in coordination with Kernel.
- As said Kernel initializes the Systemd process which then starts all software in the user environment.
- Systemd process is located in **/lib/system/systemd**.
- Systemd then loads the rest of the scripts to make the operating system fully functional for the user. These scripts are called units located under directory **/lib/systemd/system**.
- These unit contain scripts that load networking stuff, user environment (graphical or CLI), ssh, etc

>  Systemd는 리눅스 운영을 위한 서비스 관리자이다. 시스템 부팅 시 첫번째 프로세스(PID 1)으로 실행된다.  
> systemd(system daemon)은 Unix 시스템이 부팅후에 가장 먼저 생성된 후에 다른 프로세스를 실행하는 init 역할을 대체하는 데몬입니다. Red Hat 에서 주도적으로 개발을 시작했고 지금은 RHEL/CentOS 와 Ubuntu 나 Arch 등 대부분의 리눅스 시스템에 공식적으로 채택되었다.  
>  Systemd는 리눅스 운영을 위한 서비스 관리자이다. 시스템 부팅 시 첫번째 프로세스(PID 1)으로 실행된다.  또한 부팅시에 병렬로 실행되어서 부팅속도가 빠른 장점이 있다.

* systemd(1) — Linux manual page

<https://man7.org/linux/man-pages/man1/init.1.html>

[systemd(1) - Linux manual page

systemd(1) — Linux manual page SYSTEMD(1) systemd SYSTEMD(1) NAME         top systemd, init - systemd system and service manager SYNOPSIS         top /usr/lib/systemd/systemd [OPTIONS...] init [OPTIONS...] {COMMAND} DESCRIPTION         top sy

man7.org](https://man7.org/linux/man-pages/man1/init.1.html)

---

## Runlevel

- These are the services which are started or stopped based on the numeric values from 0 to 6 or 10 to 16 in new Linux distros as below

![the numeric value of the run level](/assets/img/posts/linux-boot-process/06.jpg)
_the numeric value of the run level_

These are called target files which run services based on what target is defined.

![](/assets/img/posts/linux-boot-process/07.png)

>  리눅스 시스템을 부팅할 때 시작하는 프로그램의 실행 순서를 정의한 것을 "runlevel"이라고 정의한다. 리눅스에서 런레벨은 위 처럼 7개로 구분되어진다.

---

리눅스는 오픈소스 OS로 현재는 스마트폰 에서 고성능 서버에 이르는 다양한 하드웨어에서 실행되고있다.

이렇게 많이 사용되는 OS이지만 주로 인프라단에서 많이 사용되기 때문에 일반 사용자들이 접할 일이 적었었다.

하지만 최근 클라우드 컴퓨팅의 보편화로 점점 더 많은 사용자들이 리눅스 OS를 사용해서 인프라를 손쉽게 구성하고 있으며, 위의 부팅 프로세스를 이해하고 있으며 도움이 되지 않을까 싶다.
