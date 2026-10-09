---
title: "[Linux] Processor Power Management C-states & P-states"
date: 2024-11-10 21:48:49 +0900
categories: ["Cloud & Infrastructure", "Linux"]
tags: ["c-state", "Linux", "p-state", "PowerManagement"]
tistory_url: https://codemonkyu.tistory.com/entry/Linux-Processor-Power-Management-C-states-P-states
---

오늘 포스팅할 주제는 프로세서의 상태 제어에 대한 내용을 정리해 볼 예정이다.

제어할 수 있는 프로세서의 상태 제어는 "C-state"와 "P-state"로 구분할 수 있다. 각각의 상태 제어는 서로 다른 부분에서 CPU(*프로세서)의 상태를 제어하는데, 쉽게 구분하자면 C-state는 주로 코어가 여러 단계의 유휴 상태로 전환되게 하여 전력 소비를 줄이는 방식의 상태 제어 이며, P-state는 주로 CPU의 성능 (*주파수)를 직접 제어 하는 방식이다.

모든 프로세서가 두 상태 제어 모드를 지원하는 것은 아니며, 주로 x86 아키텍처로 빌드된 프로세서(*intel, AMD 사)에서는 두 상태 제어 모드를 대부분 지원하고 있다.

다시 오늘의 포스팅 주제로 돌아와 위에서 설명한 C-state, P-state에 대한 개념을 정리해보고 실제 인텔의 "하스웰 E5-2666 v3" 프로세서를 기반으로 호스팅 되고 있는 EC2 유형 "c4.8xlarge"를 통해 C-state, P-state를 활성화하고 사용하는 방법에 대하여 확인해 볼 것이다.

---

## C-state

> CPU가 유휴 상태일 때 에너지를 절약하기 위해 CPU에 저전력 모드로 들어가도록 명령할 수 있습니다. 각 CPU에는 여러 가지 전력 모드가 있으며 이를 통틀어 "C-상태" 또는 "C-모드"라고 합니다.

C-state는 위의 설명과 같이 CPU가 가지고 있는 여러가지 전력 모드이다. CPU에 여러개의 코어가 존재하는 경우 모든 코어가 유휴 상태에서 깨어나 있을 필요가 없을 수 있기 때문에 유휴 상태로 전환되는 코어들은 상황에 따라 특정 C-state 모드로 전환될 수 있다. 더 깊은 유휴 상태로 빠질 수록 해당 코어가 다시 100%의 성능으로 전환되기 위해서는 더 많은 시간(*지연)이 발생하게 된다. 아래의 그림을 참고하면 C-state의 메커니즘을 이해하는데 도움이 될 수 있다.

![그림1](/assets/img/posts/linux-cpu-c-states-p-states/01.png)
_그림1_

**Typical C-states**  
C0 – Active Mode: Code is executed, in this state the P-States (see above) are also relevant.  
C1 – Auto Halt  
C1E – Auto halt, low frequency, low voltage  
C2 – Temporary state before C3. Memory path open  
C3 – L1/L2 caches flush, clocks off  
C6 – Save core states before shutdown and PLL off  
C7 – C6 + LLC may be flushed  
C8 – C7 + LLC must be flushed

---

## P-state

> P-상태를 사용하면 프로세서 주파수의 가변성을 줄일 수 있습니다. P-상태는 코어에서 원하는 성능(CPU 주파수)을 제어합니다.

P-state 는 CPU 사용률에 따라 CPU 코어의 물리적 주파수를 조정하여 적은 전압으로 또는 많은 전압으로 CPU가 동작할 수 있도록 조정할 수 있다. 모든 워크로드가 CPU가 제공하는 최대 전압 (*터보부스트 주파수)에서 가장 빠르게 처리되는 것이 아닌 더 낮은 전압에서도 CPU가 충분한 성능으로 워크로드를 처리할 수 있으며, 이러한 상황에서 P-state를 사용하게 된다.

![그림2](/assets/img/posts/linux-cpu-c-states-p-states/02.jpg)
_그림2_

---

## Linux 에서 프로세서 상태 제어

간략하게 C-state 그리고 P-state의 개념을 정리하였으니 실제 Linux 에서 프로세서의 상태 제어를 조정하는 법 그리고 상태 별 부하 테스트에 대한 결과 비교를 정리할 예정이다.

우선 테스트 하기 위해 두 상태 제어 모두를 지원하는 프로세서에서 호스팅 되는 가상 서버가 필요하다. 나는 AWS사의 EC2 에서 "m4.16xlarge" 유형을 선택하여 테스트를 진행하였다. 모든 인스턴스 유형이 두 상태 제어를 지원하는 것이 아니기 때문에 아래의 AWS 공식 문서를 확인해볼 것을 권장한다,

[Amazon EC2 Linux 인스턴스에 대한 프로세서 상태 제어](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/processor_state_control.html)

### (1). m4.16xlarge의 프로세서 확인

아래의 명령어를 확인하여 해당 인스턴스의 프로세서 정보를 조회한다.

프로세서 정보를 통해 Thread(s) per core /  Core(s) per socket 및 최소,최대 주파수를 확인할 수 있다.

> **# lscpu**  
> Architecture:             x86_64  
>   CPU op-mode(s):         32-bit, 64-bit  
>   Address sizes:          46 bits physical, 48 bits virtual  
>   Byte Order:             Little Endian  
> **CPU(s):                   36**  
>   On-line CPU(s) list:    0-35  
> Vendor ID:                GenuineIntel  
>   BIOS Vendor ID:         Intel  
>  **Model name:             Intel(R) Xeon(R) CPU E5-2666 v3 @ 2.90GHz**  
>     CPU family:           6  
>     Model:                63  
>     Thread(s) per core:   2  
>     Core(s) per socket:   9  
>     Socket(s):            2  
>     Stepping:             2  
>    **CPU max MHz:          3500.0000**  
> **CPU min MHz:          1200.0000**...  
> ...  
> ...

### (2).  지원하는 C-state 확인

아래의 명령어를 통해 현재 프로세서에서 지원하고 있는 C-state 종류 및 각 종류별 latency를 확인할 수 있다.

> **# cpupower idle-info**  
> CPUidle driver: intel_idle  
> CPUidle governor: menu  
> analyzing CPU 0:  
>  
> Number of idle states: 5  
> **Available idle states: POLL C1 C1E C3 C6**  
> **POLL**:  
> Flags/Description: CPUIDLE CORE POLL IDLE  
> Latency: 0  
> Usage: 478  
> Duration: 7748  
> **C1**:  
> Flags/Description: MWAIT 0x00  
> Latency: 2  
> Usage: 656  
> Duration: 83073  
> **C1E**:  
> Flags/Description: MWAIT 0x01  
> Latency: 10  
> Usage: 2981  
> Duration: 494452  
> **C3**:  
> Flags/Description: MWAIT 0x10  
> Latency: 33  
> Usage: 6730  
> Duration: 2975511  
> **C6**:  
> Flags/Description: MWAIT 0x20  
> Latency: 133  
> Usage: 91646  
> Duration: 182905918

### (3). 테스트 1 (C-state 비활성화 전)

아래 테스트를 통해 확인시 가장 낮은 c1 state에 위치했던 vCPU 21, 28이 가장 높은 주파수 (*Turbo Boost)로 동작한 것을 확인 가능하다.

```bash
$ sudo turbostat stress -c 2 -t 10
stress: info: [30680] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
stress: info: [30680] successful run completed in 10s
pk cor CPU    %c0  GHz  TSC SMI    %c1    %c3    %c6    %c7   %pc2   %pc3   %pc6   %pc7  Pkg_W RAM_W PKG_% RAM_%
             5.54 3.44 2.90   0   9.18   0.00  85.28   0.00   0.00   0.00   0.00   0.00  94.04 32.70 54.18  0.00
 0   0   0   0.12 3.26 2.90   0   3.61   0.00  96.27   0.00   0.00   0.00   0.00   0.00  48.12 18.88 26.02  0.00
 0   0  18   0.12 3.26 2.90   0   3.61
 0   1   1   0.12 3.26 2.90   0   4.11   0.00  95.77   0.00
 0   1  19   0.13 3.27 2.90   0   4.11
 0   2   2   0.13 3.28 2.90   0   4.45   0.00  95.42   0.00
 0   2  20   0.11 3.27 2.90   0   4.47
 0   3   3   0.05 3.42 2.90   0  99.91   0.00   0.05   0.00
 0   3  21  97.84 3.45 2.90   0   2.11
...
 1   1  10   0.06 3.33 2.90   0  99.88   0.01   0.06   0.00
 1   1  28  97.61 3.44 2.90   0   2.32
...
10.002556 sec
```

반면, 모든 CPU 코어가 부하 테스트에 동작에 사용되도록 c4.8xlarge가 제공하는 32 vCPU에 맞추어 옵션을 설정 후 테스트해보면 아래와 같이 모든 코어에서 균등한 주파수로 동작함을 알 수 있다. 이는 모든 코어가 Max 주파수로 동작할 수 있는 여분의 Headroom이 없기 때문에 발생한 현상이며, 모든 코어가 Max는 아니지만 프로세서가 지원하는 기본 주파수 2.9Ghz 이상으로 Turboboost 성능을 균등하게 내고 있다.

즉 프로세서의 코어에 대해 C-state를 활성화 하면 특정 프로세스의 요청에 응답하는 코어가 Max 주파수로 동작 가능할 수 있는 Headroom이 충분하기 때문에 몇몇의 작업에서는 더 빠른 동작 속도로 작업을 처리할 수 있다. 다만 특정 코어에 위치한 vCPU에 대해 요청이 존재하는 경우 C-state 모드에 따라 동작 속도에 큰 차이가 발생할 수 있다.

```bash
$ sudo turbostat stress -c 36 -t 10
stress: info: [30685] dispatching hogs: 36 cpu, 0 io, 0 vm, 0 hdd
stress: info: [30685] successful run completed in 10s
pk cor CPU    %c0  GHz  TSC SMI    %c1    %c3    %c6    %c7   %pc2   %pc3   %pc6   %pc7  Pkg_W RAM_W PKG_% RAM_%
            99.27 3.20 2.90   0   0.26   0.00   0.47   0.00   0.00   0.00   0.00   0.00 228.59 31.33 199.26  0.00
 0   0   0  99.08 3.20 2.90   0   0.27   0.01   0.64   0.00   0.00   0.00   0.00   0.00 114.69 18.55 99.32  0.00
 0   0  18  98.74 3.20 2.90   0   0.62
 0   1   1  99.14 3.20 2.90   0   0.09   0.00   0.76   0.00
 0   1  19  98.75 3.20 2.90   0   0.49
 0   2   2  99.07 3.20 2.90   0   0.10   0.02   0.81   0.00
 0   2  20  98.73 3.20 2.90   0   0.44
 0   3   3  99.02 3.20 2.90   0   0.24   0.00   0.74   0.00
 0   3  21  99.13 3.20 2.90   0   0.13
 0   4   4  99.26 3.20 2.90   0   0.09   0.00   0.65   0.00
 0   4  22  98.68 3.20 2.90   0   0.67
 0   5   5  99.19 3.20 2.90   0   0.08   0.00   0.73   0.00
 0   5  23  98.58 3.20 2.90   0   0.69
 0   6   6  99.01 3.20 2.90   0   0.11   0.00   0.89   0.00
 0   6  24  98.72 3.20 2.90   0   0.39
...
```

### (4). C-state 비 활성화

만약 일관된 성능을 위해 C-state를 비 활성화를 하고 싶은 경우 아래의 방법을 통해 가능하다.

1. 텍스트 편집기(vim)로 /etc/default/grub 파일을 엽니다.

```bash
[ec2-user ~]$ sudo vim /etc/default/grub
```

2. GRUB_CMDLINE_LINUX_DEFAULT 행을 편집하여 intel_idle.max_cstate=1 및 processor.max_cstate=1 옵션을 추가합니다. 이렇게 하면 idle 코어의 가장 깊은 C-state로 C1이 설정됩니다.

```bash
GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 nvme_core.io_timeout=4294967295 intel_idle.max_cstate=1 processor.max_cstate=1"
GRUB_TIMEOUT=0
```

3. 파일을 저장하고 편집기를 종료합니다.

4. 부팅 구성을 다시 빌드합니다.

```bash
[ec2-user ~]$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

5. 인스턴스를 재부팅하여 새로운 커널 옵션을 활성화합니다.

```bash
[ec2-user ~]$ sudo reboot
```

### (5). 테스트 2 (C-state 비활성화 후)

C-state를 비활성화 후 이전과 동일한 명령어로 테스트를 진행해보면 최대 주파수가 **3.2GHz** 로 고정되고 있는 것을 확인 가능하다. 이는 C-state 모드를 비활성화 하여 코어별 유휴 상태를 이용한 Headroom이 존재하지 않기 때문에 특정 코어에 부하가 집중되고 있음에도 프로세서가 지원하는 최대 주파수인 **3.5Ghz를**전부 사용하지 않고 있는 것이다.

```bash
$ sudo turbostat stress -c 2 -t 10
stress: info: [5322] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
stress: info: [5322] successful run completed in 10s
pk cor CPU    %c0  GHz  TSC SMI    %c1    %c3    %c6    %c7   %pc2   %pc3   %pc6   %pc7  Pkg_W RAM_W PKG_% RAM_%
             5.56 3.20 2.90   0  94.44   0.00   0.00   0.00   0.00   0.00   0.00   0.00 131.90 31.11 199.47  0.00
 0   0   0   0.03 2.08 2.90   0  99.97   0.00   0.00   0.00   0.00   0.00   0.00   0.00  67.23 17.11 99.76  0.00
 0   0  18   0.01 1.93 2.90   0  99.99
 0   1   1   0.02 1.96 2.90   0  99.98   0.00   0.00   0.00
 0   1  19  99.70 3.20 2.90   0   0.30
...
 1   1  10   0.02 1.97 2.90   0  99.98   0.00   0.00   0.00
 1   1  28  99.67 3.20 2.90   0   0.33
 1   2  11   0.04 2.63 2.90   0  99.96   0.00   0.00   0.00
 1   2  29   0.02 2.11 2.90   0  99.98
```

---

### (6).  P-state 설정

만약 프로세서가 제공하고 있는 모든 코어에 대한 Workload의 작업량이 어느정도 일정한 경우 코어 별 C-state 설정 또는 P-state를 통한 turboboost 동작보다 일관된 동작속도로 꾸준하게 작업을 처리하는 것이 더 나을 수 있다.

따라서 이러한 경우 C-state 및 P-state를 비활성화 하여 이러한 워크로드에 맞게 프로세서의 상태를 조절할 수 있다.

C-state의 비활성화 방법은 위에서 설명하였음으로, 아래에 P-state를 비활성화 하는 방법만 정리하였다.

1. P-state 를 이용하여 Turboboost 비활성화

```bash
sudo sh -c "echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo"
```

2. Turboboost 활성화

```bash
sudo sh -c "echo 0 > /sys/devices/system/cpu/intel_pstate/no_turbo"
```

3. Turboboost 비활성화 후 테스트

프로세서의 모든 vCPU가 제공되는 기본 동작 속도로 작동하고 있음을 확인 가능하다.

```angelscript
sudo turbostat stress -c 2 -t 10
stress: info: [5389] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
stress: info: [5389] successful run completed in 10s
pk cor CPU    %c0  GHz  TSC SMI    %c1    %c3    %c6    %c7   %pc2   %pc3   %pc6   %pc7  Pkg_W RAM_W PKG_% RAM_%
             5.59 2.90 2.90   0  94.41   0.00   0.00   0.00   0.00   0.00   0.00   0.00 128.48 33.54 200.00  0.00
 0   0   0   0.04 2.90 2.90   0  99.96   0.00   0.00   0.00   0.00   0.00   0.00   0.00  65.33 19.02 100.00  0.00
 0   0  18   0.04 2.90 2.90   0  99.96
 0   1   1   0.05 2.90 2.90   0  99.95   0.00   0.00   0.00
 0   1  19   0.04 2.90 2.90   0  99.96
 0   2   2   0.04 2.90 2.90   0  99.96   0.00   0.00   0.00
 0   2  20   0.04 2.90 2.90   0  99.96
 0   3   3   0.05 2.90 2.90   0  99.95   0.00   0.00   0.00
 0   3  21  99.95 2.90 2.90   0   0.05
...
 1   1  28  99.92 2.90 2.90   0   0.08
 1   2  11   0.06 2.90 2.90   0  99.94   0.00   0.00   0.00
 1   2  29   0.05 2.90 2.90   0  99.95
```
