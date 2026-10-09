---
title: "[kubernetes] 쿠버네티스의 디플로이먼트(deployment)란?"
date: 2022-04-12 03:54:44 +0900
categories: ["Cloud & Infrastructure", "kubernetes"]
tags: ["Deployment", "Kubernetes", "livenessProbe", "MinReadySeconds", "readinessProbe", "디플로이먼트"]
tistory_url: https://codemonkyu.tistory.com/entry/kubernetes-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EC%9D%98-%EB%94%94%ED%94%8C%EB%A1%9C%EC%9D%B4%EB%A8%BC%ED%8A%B8deployment%EB%9E%80
---

![](/assets/img/posts/kubernetes-deployment/01.png)

### 애플리케이션 배포 유형과 관리 (디플로이먼트)

**1. 디플로이먼트 리소스로 파드와 레플리카셋(ReplicaSet)에 대한 선언적 업데이트를 제공한다.  
2. 디플로이먼트를 사용하는 경우 실제 파드는 디플로이먼트가 아닌 레플리카셋에 의해 생성되고 관리된다.  
3. 디플로이먼트를 생성하면 레플리카셋 리소스가 그 아래에 생성된다.  
파드를 감시하는 레플리카셋이 디플로이먼트를 지원한다.  
레플리카셋은 파드를 생성하고 롤아웃 상태를 체크해서 성공 여부를 확인한다.  
4. 디플로이먼트의 상태가 에러가 생길 경우 이전 버전으로 롤백한다.  각 롤백은 디플로이먼트의 수정 버전에 따라 업데이트한다.**

![](/assets/img/posts/kubernetes-deployment/02.png)

## 디플로이먼트의 배포와 업데이트 방법 3가지 소개

**1. 블루-그린 디플로이먼트 2. 롤링 업데이트 3. 카나리 배포**

---

### 1. 블루-그린 디플로이먼트 동작방법

1. 새 버전(그린)을 실행하는 파드를 불러오는 동안 클라이언트 연결을 위한 트래픽 서비스는 파드의 이전 버전(블루)에 연결된다.  
2. 새 버전(그린)의 파드가 실행되면 서비스의 레이블 셀렉터를 변경하고 클라이언트 연결을 위한 서비스를 새 버전(그린) 파드로 전환할 수 있다.  
3. **새 버전이 올바르게 작동하면 이전 버전(블루)을 삭제하고 업데이트를 완료한다.**

![](/assets/img/posts/kubernetes-deployment/03.png)

---

### 2. 롤링 업데이트

1. 새 버전의 파드를 만들어가면서 트래픽을 구 버전 파드에서 신 버전 파드로 점차적으로 옮기는 형태이다.  
**2. 순차적으로 새 파드를 추가하고 기존 파드를 점진적으로 제거해 이 작업을 수행할 수 있다.**  
3. Blue/Green 배포 방식은 인프라 리소스가 2배로 필요한데 비해, **롤링 업데이트 배포는 서버 자원이 한정적인 경우 유리하다.**  
4. 장단점

장점: 다운타임이 없다. 버전의 업데이트와 테스팅을 동시에 수행할 수 있다.  
**단점: 애플리케이션이 동시에 2가지 버전을 실행해야 한다. 새 버전이 이전 버전을 손상시킬 수 있다.**  
5. 서비스는 롤링 업데이트 중에 요청을 이전 파드와 새로운 파드 모두에 전달한다.

![](/assets/img/posts/kubernetes-deployment/04.png)

![](/assets/img/posts/kubernetes-deployment/05.png)

#### - 롤링업데이트 실습

> ***디플로이먼트 배포**  
> [root@master1 deploy]# cat deploy-v1.yaml  
> apiVersion: apps/v1  
> **kind: Deployment**  
> metadata:  
>   name: test1   **#디플로이먼트 이름 정의**  
> spec:  
>   replicas: 3  #3개의 레플리카 파드를 생성한다.  
>   selector:   **#디플로이먼트가 관리할 파드를 찾는 방법을 정의한다.** 파드 템플릿에 정의된 레이블을 선택한다.  
>     matchLabels:    #key,value의 쌍으로 매핑된다.  
>       **app: test1**  
>   template:    #파드 템플릿  
>     metadata:  
>       name: test1  
>       labels:    #파드에 붙일 레이블은 app: test1이다.  
>         **app: test1**  
>     spec:  
>       containers:  
>       - image: hewon16/test1:1  
>         name: test1  
>

> *클러스터내 클라이언트 접속을 테스트하기 위해 서비스 구성하기  
>  
> [root@master1 deploy]# cat test-svc.yaml  
> apiVersion: v1  
> **kind: Service**  
> metadata:  
>   name: test-svc  
> spec:  
>   ports:  
>   - port: 80  
>     targetPort: 8080  
>  **selector:   #디플로이먼트의 레이블과 같아야 한다.**  
>     app: test1  
>

> #서비스를 먼저 구성하고 파드를 구성한다.  
> [root@master1 deploy]# **kubectl apply -f test-svc.yaml**  
> service/test-svc created  
>  
> [root@master1 deploy]# **kubectl apply -f deploy-v1.yaml**  
> deployment.apps/test1 created

> * 서비스, 파드, 디플로이먼트, 레플리카셋의 리스트 확인  
> [root@master1 deploy]# kubectl get all -o wide  
>  
> NAME          READY   STATUS    RESTARTS   AGE    IP               NODE      NOMINATED NODE   READINESS GATES  
>  
> pod/test1-**55d95bbb65**-cw8k2  1/1     Running   0      115s   172.16.166.171   node1  
> pod/test1-**55d95bbb65**-nnpmt  1/1     Running   0      115s   172.16.180.57    master2  
> pod/test1-**55d95bbb65**-shlll    1/1     Running   0       115s   172.16.136.28    master3  
>  
> => 파드의 목록 확인  
> => 파드 이름 중간의 문자열은 레플리카셋의 해시 값을 의미하며 레플리카셋이 이러한 파드를 관리함을 뜻함  
> => (디플로이먼트이름 – 레플리카셋 해시값 – 파드 해시값 )으로 이뤄짐  
>  
>  
>  
> NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE    SELECTOR  
> service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   4m2s   <none>  
> **service/test-svc     ClusterIP   10.107.193.248   <none>        80/TCP    2m6s   app=test1**  
> =>  서비스  
>  
> NAME                    READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES            SELECTOR  
> **deployment.apps/test1**   3/3     3            3           115s   test1        hewon16/test1:1   **app=test1**  
> =>  디플로이먼트 이름 확인  
>  
>  
>  
> NAME                               DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES            SELECTOR  
>  
> replicaset.apps/**test1**-55d95bbb65   3         3         3       115s   test1        hewon16/test1:1   app=test1,pod-template-hash=55d95bbb65  
>  
> => 레플리카셋 이름 확인  
> => 디플로이먼트는 **파드 템플릿의 각 버전마다 하나씩 여러 개의 레플리카셋을 만든다.**  
> **파드 템플릿의 해시값**을 사용하면 디플로이먼트에서 지정된 버전의 파드 템플릿에 관해 항상 동일한(기존의) 레플리카셋을 사용할 수 있다.

---

#### - annotations을 활용한 deployment 개정정보(history) 관리

> *** rollout history명령어**로 확인  
> [root@master1 deploy]# kubectl rollout history deployment test1  
> deployment.apps/test1  
> REVISION  **CHANGE-CAUSE**  
> 1                 **<none>**  
> =>히스토리의 상세정보가 없어 불편  
>  
> * 히스토리 확인 시 상세정보가 보여 작업이 편하므로 어노테이션을 세팅해준다!!  
> * 아까 위에서 사용한 deploy-v1.yaml 파일에 metadata에 annotation을 추가해준다.  
> [root@master1 deploy]# cat deploy-v1.yaml  
> apiVersion: apps/v1  
> kind: Deployment  
> metadata:  
>   name: test1  
>   annotations:     **kubernetes.io/change-cause**: "test rolling update version 1:1"  #내용추가하기  
>  
> spec:  
>   replicas: 3  
>   selector:  
>     matchLabels:  
>       app: test1  
>   template:  
>     metadata:  
>       name: test1  
>       labels:  
>         app: test1  
>     spec:  
>       containers:  
>       - image: hewon16/test1:1  
>         name: test1

>  
> [root@master1 deploy]# kubectl apply -f deploy-v1.yaml  
> deployment.apps/test1 **configured**  
>  
>  
> [root@master1 deploy]# kubectl rollout history deployment test1  
> deployment.apps/test1  
> REVISION  **CHANGE-CAUSE**  
> 1         **test rolling update version 1:1**  
>  
>  
> [root@master1 deploy]# kubectl describe deployment  
> Name:                   test1  
> Annotations:            deployment.kubernetes.io/revision: 1  
>                         kubernetes.io/change-cause: **test rolling update version 1:1**

---

### 3. 카나리 배포

1.  새 버전을 일부 특정 클라이언트 사용자에게만 접근 허용하고 배포한다. **서비스 안정성과 정상이 확인되면 모든 사용자들에게 서비스를 제공한다.**

2.  기존 버전을 유지하고 일부만 새버전으로 올려서 이상이 없나 확인 및 테스트하여 문제가 있다면 쉽게 롤백 한다.

3.  **새 버전에 문제가 없는지 확인할 때 좋은 배포 방법이다.**

4.  레이블을 이용한 카나리 배포를 할 수 있다.

5.  단점은 버전 관리를 위해 동시에 여러 개의 버전을 관리해야 하는 관리 부담이 있다. 동시에 2개 이상의 버전이 운영 환경에 배포되어야 하므로 배포 버전의 개수를 최대한 작게 하여 관리 부담을 최소화한다.

![](/assets/img/posts/kubernetes-deployment/06.png)

---

## minReadySeconds, Readiness Probe, Liveness Probe 개념과 실습

### minReadySeconds개념

1. 롤링업데이트 작업을 수행할 때 **최초의 새 버전 파드가 Running 상태라 해도 ready상태가 지정된 시간(minReadySeconds)동안 체크되어 지정된 시간이 지나야 롤아웃 되도록 한다.**

즉 업데이트 배포 속도를 늦추는 효과처럼 보인다.

2.  Pod의 상태가 Ready 상태임을 확인하는 최소 시간으로 본다.  파드가 생성된 후 사용 가능한 ready 상태로 있어야 하는 시간 (초단위)

주의: 레디니스 프로브가 완료되면, minReadySeconds에 설정된 시간이 남아있어도 이는 무시되고 서비스 트래픽을 보낸다.  **즉 모든 파드의 readinessProbe가 성공하면 클라이언트는 연결할 수 있다. 단지 다음 롤아웃 작업이 지정된 minReadySeconds간격으로 진행될 뿐이다.**

3.  minReadySeconds가 지나기 전에 새 파드가 제대로 작동하지 않고 **레디니스 프로브가 실패하기 시작하면 새 버전의 롤아웃이 효과적으로 차단된다.**

4.  파드가 준비 상태를 계속 보고할 수 있도록 minReadySeconds를 높게 설정한다.

5.  **minReadySeconds를 올바르게 설정하지 않고 레디니스 프로브만 정의하는 경우 레디니스 프로브의 첫 번째 호출이 성공하면 즉시 새 파드가 네트웍 서비스가 가능하여 클라이언트 연결이 시도된다.**

레디니스 프로브가 곧 실패하면 모든 파드에서 잘못된 버전이 롤아웃 된다. 따라서 minReadySeconds를 적절하게 설정해야 한다.

6.  minReadySeconds는 Deployment와 DaemonSet에서 사용가능하다.

### Readiness Probe, Liveness Probe개념

1. **파드의 컨테이너가 살아 있는지 체크하는 라이브니스 프로브**와 **컨테이너가 서비스 가능한 상태인지 확인하는 레디니스 프로브는 주기적으로 파드의 헬스 체크를 확인한다**.

2. 라이브니스 프로브는 컨테이너가 살아 있는지 확인하고 실패하면 kubelet이 컨테이너를 다시 시작한다.

3. 레디니스 프로브는 파드가 서비스 요청을 처리할 준비가 되어있는지 확인하고 정상이면 파드와 서비스간의 트래픽을 정상적으로 수행하도록 한다. **따라서 실제 서비스가 준비된 상태인지를 확인하는 레디니스 프로브 설정이 매우 중요하다.** 라이브니스 프로브는 파드의 컨테이너가 비정상이라면 해당 파드를 재시작하고 레디니스 프로브는 해당 파드를 사용할 수 없도록 서비스의 endpoint 오브젝트에서 제거한다.

### Readiness Probe, Liveness Probe사용 방법 (공통)

1.  라이브니스 프로브와 레디니스 프로브를 실행하는 3가지 방법

-  **HTTP GET 프로브**: HTTP GET 요청을 수행하여 응답코드가 2xx 또는 3xx 즉 200~300 사이면 성공 , 참으로 판단하고 이외의 값은 실패, 거짓으로 판단한다.

-  **TCP 소켓 프로브**: 컨테이너의 지정된 포트에 TCP 연결을 하여 성공하면 참 아니면 거짓으로 판단한다.

-   **EXEC (Command) 프로브**:  컨테이너 내에 특정 명령어를 실행하고 리턴 코드를 확인한다. ( 리턴 코드 값 0: 참     0이 아닌 값: 거짓 )

### 레디니스 프로브와 minReadySeconds의 필요성

> 새 파드에서 레디니스 프로브의 실패 실습을 위한 yaml파일 구성하기  
>  
> [root@master1 deploy]# cat deploy-v3.yaml  
> apiVersion: apps/v1  
> kind: Deployment  
> metadata:  
>   name: test1  
> spec:  
>   replicas: 3  
>   minReadySeconds: 10   #**설정하기**  
>   strategy:  
>     rollingUpdate:  
>       maxSurge: 1  
>       maxUnavailable: 0 **#디플로이먼트가 파드를 하나씩 교체하도록 maxUnavailable을 0으로 설정**  
>     type: RollingUpdate  
>   selector:  
>     matchLabels:  
>       app: test1  
>   template:  
>     metadata:  
>       name: test1  
>       labels:  
>         app: test1  
>     spec:  
>       containers:  
>       - image: hewon16/test1:3 **#이 이미지는 에러가 발생하도록 만들어진 샘플**  
>         name: test1  
>         readinessProbe:  
>             periodSeconds: 1      **# 매초마다 실행될 레디니스 프로브를 정의한다.**  
>             httpGet:      #**레디니스 프로브는 컨테이너에 HTTP GET요청을 수행한다.**  
>                 path: /  
>                 port: 8080

> [root@master1 deploy]# kubectl apply -f deploy-v3.yaml  
>  
> [root@master1 deploy]# kubectl rollout status deployment test1  
>  
> [root@master1 deploy]# kubectl rollout status deployment test1  
> Waiting for deployment "test1" rollout to finish: 1 out of 3 new replicas have been updated...  
> error: deployment "test1" exceeded its progress deadline  
>  
>  
>
> ---
>
>  
> [root@master1 ~]# kubectl get all -o wide  
> NAME   READY   STATUS    RESTARTS   AGE    IP     NODE      NOMINATED NODE  
> pod/test1-6bc7bd84c9-2c8vh   **0/1**     Running   0          109s   172.16.166.144   node1  
> => 사용 가능할 때까지 롤아웃 프로세스는 새 파드를 만들지 않으며 **maxUnavailable속성값 0 때문에 원래 파드도 제거되지 않는다.**  
>  
> ......  
>  
> NAME          DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES            SELECTOR  
> **replicaset.apps/test1-6bc7bd84c9   1         1         0       109s   test1        hewon16/test1:3   app=test1,pod-template-hash=6bc7bd84c9**  
>  
> =**> 새 레플리카만 하나 시작되고 롤아웃 프로세스는 계속되지 않는다.**  
>  
> **replicaset.apps/test1-7b9d5b9588   3         3         3       35h    test1        hewon16/test1:2   app=test1,pod-template-hash=7b9d5b9588**  
> replicaset.apps/test1-84b5cd9887   0         0         0       35h    test1        hewon16/test1:3   app=test1,pod-template-hash=84b5cd9887  
>  
>  
>  
> 중요:  
>   **minReadySeconds를 올바르게 설정하지 않고 레디니스 프로브만 정의하는 경우 레디니스 프로브의 첫 번째 호출이 성공하면 즉시 새 파드가 사용 가능한 것으로 간주된다.**  
>   **레디니스 프로브가 곧 실패하면 모든 파드에서 잘못된 버전이 롤아웃된다.** 따라서 minReadySeconds를 적절하게 설정해야 한다.  
>

---

## readiness probe 소개

1.  pod가 연결을 수락할 준비가 됐을 때 신호 보내기

2.  Pod 구성에 시간이 필요할 수 있고, 준비 절차를 수행해야할 수도 있다. 완전한 준비가 될 때까지 준비 중인 pod에 요청을 전달하지 않는 방법이 필요하다.

3.  주기적 호출로 특정 pod가 클라이언트 요청을 수신할 수 있는지를 결정한다.

4.  컨테이너의 readiness probe가 성공을 반환하면 컨테이너가 요청을 수락할 준비가 됐다는 신호다.

### readiness porbe 3가지 메카니즘

1.  **exec프로브**: 컨테이너의 상태를 프로세스의 종료 상태 코드로 결정한다. 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인한다.  코드가 0이면 성공, 0이 아닌 값이면 실패로 간주된다.

2.   **HTTP GET  프로브:**  HTTP GET 요청을 컨테이너로 보내고 응답의 HTTP 상태 코드를 보고 컨테이너가 준비됐는지 여부를 결정한다.  지정한 IP주소, 포트 경로에 HTTP GET 요청을 수행한다.

프로브가 응답을 수신하고 HTTP응답코드가 2xx 또는 3xx인 경우 성공이다.

서버가 오류 응답 코드가 4xx 또는 5xx인 경우 실패이다.

3.   **TCP 소켓 프로브:** 컨테이너의 지정된 포트로 TCP 연결을 연다. 소켓이 연결되면 컨테이너가 준비된 것으로 간주된다.

### 레디니스 프로브의 동작

1. **1단계:** 컨테이너가 시작될 때 첫 번째 레디니스 점검을 수행하기 전에 구성 가능한 시간을 기다린다. (initialDelaySeconds )

2. **2단계:** 주기적으로 프로브를 호출하고 레디니스 프로브의 결과에 따라 작동한다. **Pod가 준비되지 않았다고 하면 서비스에서 엔드포인트는 제거된다.** Pod가 준비되면 서비스에 다시 엔드포인트가 추가된다.

3.  컨테이너가 준비 상태 점검(레디니스 프로브)에 실패하더라도 **컨테이너가 종료되거나 다시 시작되지 않는다.**

4.  레디니스 프로브가 실패한 파드는 서비스의 엔드포인트에서 제거되어 클라이언트가 **정상 상태인 pod하고만 통신 가능하다.**

---

## liveness probe 소개

1. 컨테이너가 살아 있는지 확인하 수 있다.

2. pod의 spec에 각 컨테이너의 liveness probe를 지정할 수 있다.

3. 주기적으로 probe를 실행하고 probe가 실패할 경우 **컨테이너를 다시 시작한다**.

### liveness probe의 3가지 메커니즘

1.  **HTTP GET probe:** 지정한 IP주소, 포트 경로에 HTTP GET 요청을 수행한다.

프로브가 응답을 수신하고 HTTP응답코드가 2xx 또는 3xx인 경우 성공으로 간주된다.

서버가 오류 응답 코드가 4xx 또는 5xx인 경우 실패한 것으로 간주된다.

전혀 응답하지 않으면 프로브가 실패한 것으로 간주돼 컨테이너를 다시 시작한다.

2.  **TCP소켓 probe**: 컨테이너의 지정된 포트에 TCP연결을 시도한다. 연결하면 성공, 못하면 실패로 간주 컨테이너를 다시 시작한다.

3.  **Exec probe**: 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인한다.  **코드가 0이면 성공, 0이 아닌 값이면 실패로 간주된다.**

* 실습

> [root@master1 deploy]# cat deploy-liveness-probe.yaml  
> apiVersion: apps/v1  
> kind: Deployment  
> metadata:  
>   name: test1  
> spec:  
>   replicas: 3  
>   selector:  
>     matchLabels:  
>       app: test1  
>   template:  
>     metadata:  
>       name: test1  
>       labels:  
>         app: test1  
>     spec:  
>       containers:  
>       - image: hewon16/test1:1  
>         name: test1  
>         ports:  
>         - name: test1  
>           containerPort: 8080  
>         l**ivenessProbe:**  
>  **exec:**  
>  **command:**  
>  **- ls**  
>  **- /var/ready**

> [root@master1 deploy]# kubectl apply -f  deploy-liveness-probe.yaml  
> deployment.apps/test1 created  
>  
> [root@master1 deploy]# kubectl get pod  
> NAME                     READY   STATUS             RESTARTS   AGE  
> test1-6bb7db76cd-c2bg8   0/1     CrashLoopBackOff   17         48m  
> test1-6bb7db76cd-pq2l2   0/1     CrashLoopBackOff   17         48m  
> test1-6bb7db76cd-qdnsg   0/1     CrashLoopBackOff   17         48m  
>  
> #에러가 날것이다 상세정보를 통해 속성값 및 에러를 알아보자!  
> [root@master1 deploy]# kubectl describe pod **test1-6bb7db76cd-c2bg8**  
>  
>  State:  Waiting  
>  **Reason: CrashLoopBackOff**  
> Last State:     **Terminated**  
> Reason:       **Error**  
>  Exit Code:    137  
> Started:      Mon, 10 May 2021 13:50:55 +0900  
> Finished:     Mon, 10 May 2021 13:51:55 +0900  
> Ready:          False  
> Restart Count:  17
>
> **Liveness:**       exec [ls /var/ready] delay=0s timeout=1s period=10s #success=1 #failure=3
>
> -Delay ( 지연=0초 ) : 컨테이너가 시작된 후 바로 프로브가 시작된다는 것을 나타냄
>
> -Timeout ( 제한시간=1초 ): 제한 시간이 1초로 설정돼 있어 컨테이너가 1초 안에 응답           해야함 그렇지 않으면 프로브가 실패한 것으로 카운트된다.
>
> -Period ( 기간=10초 ): 컨테이너는 10초마다 프로브를 수행
>
> -Failure ( 실패=연속3회 ): 프로브가 연속 3번 실패하면 컨테이너가 다시 시작된다.

---

## minReadySeconds와 레디니스 프로브 연관성

#### 1. 롤아웃 속도 제어 (maxSurge와 maxUnavailable 옵션)

1. 새 파드가 생성되고 사용 가능해지면 이전 파드가 삭제된다.

2. 오래된 파드가 다 없어질 때까지 진행한다. 이러한 배포가 무 중단 배포가 되기 위해 파드 배포 과정에 옵션 2가지를 사용한다.  pod의 롤링업데이트를 위한 maxSurge와 maxUnavailable 옵션 사용

3.  항상 일정 개수의 pod를 이용 가능하게 되기 때문에 배포 중 트래픽 중단을 방지한다.

4.  **2가지 옵션값을 한꺼번에 0으로 설정할 수는 없다.** 파드가 존재하지 않을 수 있다.

**maxSurge**

1. maxSurge는 deployment의 의도하는 replica에서 지정된 pod개수 보다 **여분의 pod가 몇개 더 추가될 수 있는지 지정한다.**

2. **기본: 레플리카 설정 값의 25% 예) 의도하는 레플리카 수가 4라면 업데이트 중에 동시에 5개 이상의 파드가 실행되지 않아야 한다.**

3.  %나 숫자로 설정 가능, 백분율을 절대 숫자로 변환할 때 숫자가 반올림된다.

**maxUnavailable**

1.  maxUnavailable는 **업데이트 하는 동안 몇 개의 파드가 사용할 수 없는지 개수를 설정하는데 사용**

2.  롤링 업데이트 중에 의도하는 레플리카 수를 기준으로 **동시에 삭제할 수 있는 파드 개수를 결정한다.** 즉 사용할 수 없는 파드 개수 결정

3.  **기본 : 25%**

4.  예) 의도하는 레플리카 수가 4로 설정되고 백분율이 25%이면 하나의 파드만 사용할 수 없다.  
즉 25% 는 1/4이므로 1개의 파드만 사용할 수 없는 상태로 만들고 파드 인스턴스 3개가 항상 사용할 수 있어야 한다.

5.   %나 숫자로 설정 가능. 백분율이 절대 숫자로 변환될 때 숫자가 내림 된다.

이 두개의 옵션을 운영중인 서비스의 특성에 맞게 적절히 조절해 주어 항상 일정 개수 이상의 pod가 이용 가능하게 되기 때문에 배포 중 트래픽 유실이 없게 된다. 둘 다 한꺼번에 0으로 설정되면 pod가 존재하지 않는 경우가 발생하기 때문에 한꺼번에 0으로 설정할 수는 없다.

#### 2. 실습

**1단계: v1버전이 처음 배포되고 파드들이 레디니스프로브를 만족할 딜레이타임을 240초로 길게 설정하고** 각 파드가 240초 안에 레디니스 프로브를 만족하게 되면 3개의 파드가 정상적으로 클라이언트 연결을 가능하게 한다.

(수동으로 3개의 파드마다 /var/ready파일을 생성하면서 실제 서비스 준비를 위한 시간이 필요하다고 가정한다.)

**2단계:** 롤링업데이트를 시켜 레디니스프로브를 만족하지 못하는 새버전의 파드 상태를 관찰한다. 240초가 지날 경우 레디니스 프로브를 만족하는 파드가 어떻게 되는지 확인하고 360초 후에 롤아웃이 된 새버전의 파드는 레디니스 프로브를 만족하지 못할 경우 어떻게 될까? 즉 롤링업데이트가 멈추게 되는지 확인하자.

> **1단계**  
> #v1버전 yaml파일 만들기  
> [root@master1 deploy]# cat deploy-minready-readiness-probe.yaml  
> apiVersion: apps/v1  
> kind: Deployment  
> metadata:  
>   name: test1  
> spec:  
>   replicas: 3  
>   **minReadySeconds: 360**  
>   strategy  
>     **rollingUpdate:**  
>  **maxSurge: 1**  
>  **maxUnavailable: 0**  
>     type: RollingUpdate  
>   selector:  
>     matchLabels:  
>       app: test1  
>   template:  
>     metadata:  
>       name: test1  
>       labels:  
>         app: test1  
>     spec:  
>       containers:  
>       - image: hewon16/test1:1  
>         name: test1  
>         ports:  
>         - name: test1  
>           containerPort: 8080  
>        **readinessProbe:**  
>  **initialDelaySeconds: 240**  
>  **periodSeconds: 5**  
>  **exec:**  
>  **command:**  
>  **- ls**  
>  **- /var/ready**  
>  
>  
> [root@master1 deploy]# kubectl apply -f deploy-minready-readiness-probe.yaml  
> deployment.apps/test1 created  
>  
> #생성된 pod를 확인한다.  
> [root@master1 deploy]# kubectl get pod  
> NAME                     READY   STATUS    RESTARTS   AGE  
> test1-7c99bd78c4-2d9rb   0/1     Running   0          112s  
> test1-7c99bd78c4-wcdx5   0/1     Running   0          112s  
> test1-7c99bd78c4-wrcwt   0/1     Running   0          112s  
>  
> #생성된 pod에 /var/ready를 만들어준다.  
> [root@master1 deploy]# kubectl exec test1-7c99bd78c4-2d9rb **-- touch /var/ready**  
> [root@master1 deploy]# kubectl exec test1-7c99bd78c4-wcdx5 -- touch /var/ready  
> [root@master1 deploy]# kubectl exec test1-7c99bd78c4-wrcwt  -- touch /var/ready  
>  
>
> ---
>
>  
> **2단계**  
> **#새버전 v2를 배포하여본다.**  
> [root@master1 deploy]# kubectl set image deployment test1 **test1=hewon16/test1:2  
>**
>
> [root@master1 deploy]# kubectl get pod  
> NAME                     READY   STATUS    RESTARTS   AGE
>
> **test1-6cccdd977f-2ltwx   0/1     Running   0          18s**
>
> test1-7c99bd78c4-2d9rb   1/1     Running   0          14m
>
> test1-7c99bd78c4-wcdx5   1/1     Running   0          14m
>
> test1-7c99bd78c4-wrcwt   1/1     Running   0          14m
>
>  
> => 새버전의 v2파드가 ready 상태가 아니다.
>
>  
>
> **새 파드인 v2파드에 /var/ready파일을 만들어준후 240초 기다리면 ready로 바뀐다.**
>
> [root@master1 deploy]# kubectl exec test1-6cccdd977f-2ltwx -- touch /var/ready  
>  
>
> **그리고 다음 차례의 롤아웃작업은 최소 ready 상태를 유지할 minReadySeconds 360초(6분)가 더 지나야 가능함을 정확히 확인한다.  총 걸리는 시간: ( initialDelaySeconds: 240 + minReadySeconds: 360 = 600(10분))**
>
>  
>

**결과**

- livenessProbe는 컨테이너가 살아 있는지 확인하고 이 헬스체크가 실패하면 kubelet이 컨테이너의 restart policy에 따라 컨테이너가 재시작 된다.

- 무 중단 배포에서 중요한 설정은 readinessProbe다. readinessProbe는 실제로 컨테이너가 서비스 요청을 처리할 준비가 되었는지를 확인하는데 사용된다.

- readinessProbe가 정상일때 파드와 연결된 service에 파드의 ip가 엔드포인트에 추가되고 실제 서비스가 가능해진다.

- readinessProbe는 초기화 시간이 오래 걸리는 컨테이너에 대해서 사용하면 컨테이너가 준비될 때까지 일정시간동안 트래픽을 받지 않고 대기할 수 있기 때문에 좋다.

- readinessProbe가 완료되면 minReadySeconds에 설정된 시간이 남아 있어도 무시되고 네트웍 클라이언트와 서비스가 연결된다.

- MinReadySeconds 옵션: 롤링업데이트시 롤아웃 작업에 의해 생성된 새 버전의 파드가  ready상태로 체크되는 최소 대기 시간으로 롤아웃 간격조절의 효과가 있다. 레디니스가 최초 성공시 바로 클라이언트가 서비스 연결이 가능하고 그러다 레디니스가 실패가 발생할 경우 롤아웃 작업이 차단되지 못할 수 있어 일정 시간 동안을 ready상태로 있는지 체크를 하는 minReadySeconds가 함께 설정되는 것이 안정적이다. 이로 인해 잘못된 버전의 롤아웃 작업은 중단될 것이다.
