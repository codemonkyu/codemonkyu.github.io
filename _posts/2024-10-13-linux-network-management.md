---
title: "[Network] Linux Network Management"
date: 2024-10-13 20:20:42 +0900
categories: ["Cloud & Infrastructure", "Network & Server"]
tags: ["ARP", "EC2", "iptables", "Network"]
tistory_url: https://codemonkyu.tistory.com/entry/Network-Linux-Network-Management
---

오늘 포스팅할 글의 주제는 "Linux Network Management"와 관련된 글이다.

최근 들어 서비스의 환경이 복잡해지며, OS의 네트워크 환경을 이해하는 것이 더욱 더 중요해졌다.

따라서 해당 포스팅을 통해 Linux OS 시스템에서 네트워크 환경 / 흐름 / 제어에 관련된 내용을 정리하고 살펴보는 시간을 갖으려고 한다.

---

## 1. Structure and Checking from Layer 1 to Layer 7

![](/assets/img/posts/linux-network-management/01.png)

위 그림은 Linux OS에 대한 여러 요소에 대한 성능 측정 시 사용될 수 있는 Tool(*도구)를 정리한 것이다. 해당 포스팅에서는 빨간색으로 강조한 부분인 "Net Device", "IP", "TPC/UDP", "Sockets" 등과 관련된 부분에 대하여 다룰 예정이다.

---

## 2. Network Management, Monitoring and Call Flow

### 1. Network Management - Information

해당 부분에서는 기본적으로 Linux OS 에서 네트워크 인터페이스 카드에 대한 설정 및 정보를 확인하는 방법을 알아본다. 이를 통해 실제 OS 구성된 네트워크 인터페이스 정보 및 route table의 내용을 확인할 수 있다.

> (1). Network Physical Layer / Network Device  
> > ip link show eth0  
> 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>  
> mtu  
> 9001  
>  qdisc mq state  
> UP  
>  mode DEFAULT group default qlen 1000  
> link/ether  
> 02:ce:4b:72:77:  
> fd  
>  brd ff:ff:ff:ff:ff:ff  
>  
> (2). ARP  
> ❯ ip neigh show  
> 172.31.0.1 dev eth0 lladdr 02:64:39:bd:62:74  
> REACHABLE  
> 172.31.0.2 dev eth0 lladdr 02:64:39:bd:62:74 REACHABLE  
> 172.31.14.92 dev eth0 lladdr 02:8d:7c:7e:6e:d3 REACHABLE  
> 172.31.5.6 dev eth0 lladdr 02:0f:bf:2b:d8:43 STALE  
> 172.31.13.152 dev eth0 lladdr 02:63:40:3d:54:bf REACHABLE  
>  
> (3). IP protocol address  
> ❯ ip address show eth0  
> 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000  
> link/ether 02:ce:4b:72:77:fd brd ff:ff:ff:ff:ff:ff  
> inet  
> 172.31.13.141/20  
>  brd 172.31.15.255 scope global dynamic eth0  
> valid_lft 2895sec preferred_lft 2895sec  
> inet6 fe80::ce:4bff:fe72:77fd/64 scope link  
> valid_lft forever preferred_lft forever  
>  
> (4). IP routing table entry  
> ❯ ip route show  
> default via 172.31.0.1 dev eth0  
> 169.254.169.254 dev eth0  
> 172.31.0.0/20 dev eth0 proto kernel scope link src 172.31.13.141  
> or  
> ❯ netstat -rn

---

### 2. Network Management -  IP Configuration & DHCP Transaction

해당 부분에서는 Linux OS에서 Network interface card의 설정 및 DHCP Transaction 과정에 대해 확인 한다.  
DHCP Transaction 과정 확인을 네트워크 관련 문제 발생 시 DHCP 갱신 오류등 문제를 확인 후 진단할 수 있다.

> **(1). NIC configuration**  
> ❯ cat /etc/sysconfig/network-scripts/ifcfg-eth0  
> DEVICE=eth0  
> BOOTPROTO=  
> dhcp  
> ONBOOT=yes  
> TYPE=Ethernet  
> USERCTL=yes  
> PEERDNS=yes  
> DHCPV6C=yes  
> DHCPV6C_OPTIONS=-nw  
> PERSISTENT_DHCLIENT=yes  
> RES_OPTIONS="timeout:2 attempts:5"  
> DHCP_ARP_CHECK=no  
>  
> **(2). 2nd NIC**  
> ❯ sudo dhcpdump -i eth1  
> TIME: 2024-10-07 07:51:23.409  
> IP: 172.31.2.253 (02:d7:0b:8f:d4:5f) > 172.31.2.253 (02:64:39:bd:62:74)  
> OP: 1 (BOOTPREQUEST)  
> HTYPE: 1 (Ethernet)  
> HLEN: 6  
> HOPS: 0  
> XID: 455ce67f  
> SECS: 0  
> FLAGS: 0  
> **CIADDR: 172.31.2.253  - IP 주소**  
> YIADDR: 0.0.0.0  
> SIADDR: 0.0.0.0  
> GIADDR: 0.0.0.0  
> CHADDR: 02:d7:0b:8f:d4:5f:00:00:00:00:00:00:00:00:00:00  
> SNAME: .  
> FNAME: .  
> OPTION: 53 ( 1) DHCP message type 3 (DHCPREQUEST)  
> OPTION: 81 ( 3) Client FQDN 1-0-0  
> **OPTION: 55 ( 13) Parameter Request List 1 (Subnet mask) - 클라이언트가 요청한 DHCP 옵션**  
> 28 (Broadcast address)  
> 2 (Time offset)  
> 121 (Classless Static Route)  
> 15 (Domain name)  
> 6 (DNS server)  
> 12 (Hostname)  
> 40 (NIS domain)  
> 41 (NIS servers)  
> 42 (NTP servers)  
> 26 (Interface MTU size)  
> 119 (Domain Search)  
> 3 (Router)  
> --------------------------------------------------------------------------------  
>  
> TIME: 2024-10-07 07:51:23.409  
> IP: 172.31.0.1 (02:64:39:bd:62:74) > 172.31.0.1 (02:d7:0b:8f:d4:5f)  
> OP: 2 (BOOTPREPLY)  
> HTYPE: 1 (Ethernet)  
> HLEN: 6  
> HOPS: 0  
> XID: 455ce67f  
> SECS: 0  
> FLAGS: 0  
> CIADDR: 0.0.0.0  
> **YIADDR: 172.31.2.253 - DHCP 서버가 할당한 IP 주소**  
> SIADDR: 0.0.0.0  
> GIADDR: 0.0.0.0  
> CHADDR: 02:d7:0b:8f:d4:5f:00:00:00:00:00:00:00:00:00:00  
> SNAME: .  
> FNAME: .  
> **OPTION: 53 ( 1) DHCP message type 5 (DHCPACK) - DHCP 응답에 따른 옵션에 대한 값**  
> **OPTION: 54 ( 4) DHCP Server identifier 172.31.0.1**  
> **OPTION: 51 ( 4) IP address leasetime 3600 (60m)**  
> **OPTION: 1 ( 4) Subnet mask 255.255.240.0**  
> **OPTION: 28 ( 4) Broadcast address 172.31.15.255**  
> **OPTION: 15 ( 31) Domain name ap-northeast-2.compute.internal**  
> **OPTION: 6 ( 4) DNS server 172.31.0.2**  
> **OPTION: 12 ( 15) Hostname ip-172-31-2-253**  
> **OPTION: 26 ( 2) Interface MTU size 9001**  
> **OPTION: 3 ( 4) Router 172.31.0.1**  
> --------------------------------------------------------------------------------  
>  
> **(3). DNS Server from DHCP lease**  
> ❯ nslookup  
> > server  
> Default server: 172.31.0.2  
> Address: 172.31.0.2#53

---

### 3. Network Management  - MTU

해당 글에서는 AWS의 EC2 Instance에서의 MTU를 확인한다.

MTU (Maximum Transmission Unit)는 네트워크 인터페이스에서 한 번에 전송할 수 있는 최대 데이터 크기이다. 다시 말해, 네트워크 프레임의 최대 크기를 나타내며 네트워크 성능에 큰 부분을 차지 하기 때문에 최적의 MTU 크기를 구성하는것이 중요하다. 이를 위해서는 서비스가 위치한 호스트에서 MTU를 어느정도 까지 지원하는지 그리고 네트워크 구간에서 호환성이 보장되는지를 확인하는 과정이 필요하다.

![](/assets/img/posts/linux-network-management/02.png)

> **(1). MTU check (in the same Subnet)**  
> ❯ tracepath 172.31.13.141  
>  
> 1?: [LOCALHOST] pmtu 9001  
> 1: ip-172-31-13-141.ap-northeast-2.compute.internal 1.708ms reached  
> 1: ip-172-31-13-141.ap-northeast-2.compute.internal 0.377ms reached  
> **Resume :  pmtu 9001 hops 1 back 1 <--- 확인**  
>  
> **(2). MTU check (in the differen Regions)  
> ❯ tracepath 35.165.74.115**  
> **1?: [LOCALHOST] pmtu 9001**  
> **1: ip-172-31-0-1.ap-northeast-2.compute.internal 0.111ms pmtu 1500**  
> **1: no reply**  
> **..**  
> **...**  
> **..**  
> **30: no reply**  
> **Too many hops: pmtu 1500  
> Resume : *pmtu* *1500 <--- 확인  
>**(3). MTU check toward Internet  
> ❯ tracepath google.com  
>  
> 1?: [LOCALHOST] pmtu 9001  
> 1: ip-172-31-0-1.ap-northeast-2.compute.internal 0.171ms pmtu 1500  
> ...  
> ..  
> 30: no reply  
> Too many hops: pmtu 1500  
> Resume: *pmtu**1500* <--- 확인***

참고 문헌 :

Amazon EC2 인스턴스의 MTU 설정 - <https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-instance-mtu.html>

---

### 4. Performance & Monitoring  - Using Iperf3

이 부분에서는 네트워크 처리량 벤치마크 툴인 Iperf3을 이용하여 임의로 Client 와 Server 간의 트래픽을 발생 후

해당 구간을 실제로 OS에서 확인하는 방법을 정리하였다.

Iperf3를 이용한 Client 와 Server 간의 테스트 구성은 이전에 설명한 포스팅 및 문서를 참고하여 준비한다.

[이전 포스팅](https://codemonkyu.tistory.com/entry/Network-AWS-EC2-%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%9D%98-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%B2%98%EB%A6%AC%EB%9F%89-%EB%B2%A4%EC%B9%98-%EB%A7%88%ED%82%B9%ED%95%98%EA%B8%B0)

[AWS에서 제공하는 공식 문서](https://repost.aws/ko/knowledge-center/network-throughput-benchmark-linux-ec2)

아래 방법을 통해, 실제 서비스간의 네트워크 트래픽이 발생하였을 때 Socket 또는 Packet 단위에서 네트워크 통신을 분석할 수 있다.

> **(1). Socket / TCP / IP level view (* Iperf는 기본적으로 5201 포트를 사용한다.)**  
> ❯ sudo ss -tanp | grep 5201 | grep ESTAB  
>  
> ESTAB 0 1566075 172.31.14.92:53818 172.31.13.141:5201 users:(("iperf3",pid=421181,fd=21))  
> ESTAB 0 1798749 172.31.14.92:53832 172.31.13.141:5201 users:(("iperf3",pid=421181,fd=23))  
> ESTAB 0 1691361 172.31.14.92:53768 172.31.13.141:5201 users:(("iperf3",pid=421181,fd=5))  
> ESTAB 0 0 172.31.14.92:53756 172.31.13.141:5201 users:(("iperf3",pid=421181,fd=4))  
> ESTAB 0 1619769 172.31.14.92:53792 172.31.13.141:5201 users:(("iperf3",pid=421181,fd=13))  
>  
>  
> **(2). Packet Capture**  
> ❯ sudo tcpdump -i any -nn -N -c 10 -s 1500 -w test.pcap tcp port 5201  
>  
> tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 1500 bytes  
> 10 packets captured  
> 17 packets received by filter  
> 0 packets dropped by kernel  
>  
> **(2-1). read PCAP file**  
> ❯ sudo tcpdump -e -nn -N -vvv -r test.pcap  
>  
> reading from file test.pcap, link-type LINUX_SLL (Linux cooked)  
> 04:04:30.350544 In 02:8d:7c:7e:6e:d3 ethertype IPv4 (0x0800), length 9017: (tos 0x0, ttl 127, id 8739, offset 0, flags [DF], proto TCP (6), length 9001)  
>  
>  
> **(4). with PID**  
> ❯ sudo lsof -p 17513  
> COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME  
> iperf3 17513 ec2-user cwd DIR 259,1 4096 12669 /etc/sysconfig/network-scripts  
> ...  
> ...  
> ...  
>  
> **(5). with port number / refresh per 5 sec**  
> ❯ lsof -i :5201 -n -r 5  
> COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME  
> iperf3 17096 ec2-user 3u IPv6 817999 0t0 TCP *:targus-getdata1 (LISTEN)  
> iperf3 17096 ec2-user 4u IPv6 818099 0t0 TCP 172.31.13.141:targus-getdata1->172.31.13.141:39626 (ESTABLISHED)  
> iperf3 17096 ec2-user 5u IPv6 819075 0t0 TCP 172.31.13.141:targus-getdata1->172.31.13.141:39638 (ESTABLISHED)  
>

---

### 5. IPTables (OS level firewall)

> **(1). pre-requisites - port scan / from Server**  
> ❯ sudo nmap localhost  
>  
> Not shown: 995 closed ports  
> PORT STATE SERVICE  
> 22/tcp open ssh  
> 25/tcp open smtp  
> **80/tcp open http**  
> 111/tcp open rpcbind  
> 9000/tcp open cslistener  
> Nmap done: 1 IP address (1 host up) scanned in 0.12 seconds  
>  
> => Localhost의 포트 상태를 스캐닝한다. 테스트에 사용할 80번 포트 상태를 확인한다.  
>  
> **(2). IPTables**  
> **(2-1). iptables rule Backup**  
> # save  
> ❯  iptables-save > /etc/sysconfig/iptables  
> # restore  
> ❯ sudo iptables-restore < /etc/sysconfig/iptables  
>  
>  
> ## Logging  
> **(3). drop & logging**  
> iptables -A INPUT -p tcp --dport 80 --syn -j LOG --log-prefix 'IPTABLES_TEST_LOG'  
> # test traffic from Client  
> ❯ curl -O http://al2/test.100MB  
> ...  
>  
> => 테스트를 위해 80번 포트에 대한 네트워크 트래픽을 발생시킨다.  
>  
>  
> **(4). check logging/logs via journalctl**  
> ❯ sudo journalctl -k -f  
> ..  
> Oct 08 00:18:52 ip-172-31-13-141.ap-northeast-2.compute.internal kernel:  
> IPTABLES_TEST_LOGIN  
> =eth0 OUT= MAC=02:ce:4b:72:77:fd:02:64:39:bd:62:74:08:00 SRC=203.249.187.71 DST=172.31.13.141 LEN=64 TOS=0x00 PREC=0x00 TTL=44 ID=0 DF PROTO=TCP SPT=62589 DPT=80 WINDOW=65535 RES=0x00 SYN URGP=0  
> OPT  
> (020405B4010303060101080A731FF31A0000000004020000)  
>  
>  
> **(5). Counter**  
> **(5-1). check counter**  
> ❯ sudo iptables -L -nv  
> Chain INPUT (policy ACCEPT 29588 packets, 1822K bytes)  
> pkts bytes target prot opt in out source destination  
> 3  
>  172 LOG tcp -- * * 0.0.0.0/0 0.0.0.0/0 tcp dpt:80 flags:0x17/0x02 LOG flags 0 level 4 prefix "IPTABLES_TEST_LOG"  
>  
> **(6-1). reset counter**  
> sudo iptables --zero INPUT  
> ❯ sudo iptables -L -nv  
> Chain INPUT (policy ACCEPT 28 packets, 1856 bytes)  
> pkts bytes target prot opt in out source destination  
> 0  
>  0 LOG tcp -- * * 0.0.0.0/0 0.0.0.0/0 tcp dpt:80 flags:0x17/0x02 LOG flags 0 level 4 prefix "IPTABLES_TEST_LOG"  
>  
> **(7). Advanced**  
> -m geoip --src-cc KR -j ACCEPT # need to install extra module  
> -m connlimit --connlimit-above 10 # block after 10 conncurrent tcp session  
> -m limit --limit 100/m --limit-burst 5 # a little complicated

---

### 글을 마치며..

평소 네트워크 관련하여 문제가 발생하면 어떠한 구간 (*네트워크 홉) 또는 client와 server에서의 어떠한 방법으로 문제를 진단해야할 지 막막할 때가 있다. 그럴때 해당 포스팅에서 소개한 방법들을 상황에 따라 사용하여 진단해본다면 도움이 될 것이라 생각한다.

다만 해당 포스팅은 Linux OS에서의 네트워크와 관련된 전반적 부분에 대한 가이드를 담고 있기 때문에 특정 상황의 문제에 대한 명쾌한 답은 되지 않을 수 있다.
