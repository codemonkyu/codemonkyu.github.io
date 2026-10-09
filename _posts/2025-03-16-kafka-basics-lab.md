---
title: "[Kafka] 간단한 LAB을 곁들인 Kafka 기본 개념 알아보기"
date: 2025-03-16 23:12:26 +0900
categories: ["Cloud & Infrastructure", "Kafka"]
tags: ["BASIC", "broker", "consumer", "instances", "KAFKA", "Producer"]
tistory_url: https://codemonkyu.tistory.com/entry/Kafka-%EA%B0%84%EB%8B%A8%ED%95%9C-LAB%EC%9D%84-%EA%B3%81%EB%93%A4%EC%9D%B8-Kafka-%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0
---

![](/assets/img/posts/kafka-basics-lab/01.png)

## Kafka 왜 써요?

메시지 큐는 비동기 메시지를 사용하여 서로 다른 시스템 간의 결합도를 낮추고, 효율적인 데이터 처리와 분산 시스템의 확장성을 제공하기 때문에 많은 서비스에 사용된다. 최근 데이터를 이용하는 서비스들이 많아졌는데 그러한 이유로 kafka와 같은 “메시지 큐” 방식의 데이터 처리 방식의 서비스에 대한 수요가 많아지게 되었다.

- **비동기적 통신**: 발신자와 수신자가 독립적으로 작업을 수행할 수 있도록 지원한다.
- **중간 저장소 역할**: 데이터를 큐(queue)에 임시로 저장하고, 소비자가 이를 처리한다.
- **분산 시스템 필수 구성 요소**: 작업 부하 분산, 내결함성, 확장성 제공.

![](/assets/img/posts/kafka-basics-lab/02.png)

---

## Kafka 주요 구성 요소

아래는 카프카의 데이터 스트림을 구성하고 있는 주요 요소에 대한 설명이다.

첫 번째는 **'토픽(Topic)' 이다.** 토픽은 카프카에서 데이터 스트림을 구분하는 단위로, 이벤트(*데이터)가 발행되는 장소라고 생각하면 된다. 토픽은 한 개 이상의 파티션으로 구성되며 파티션내에서 다시 오프셋으로 구분된다.  새로운 이벤트에 대한 토픽이 생성되면 몇개의 파티션에 이벤트(*데이터)를 저장할 것인지 정의할 수 있고 이는 또 다시 세그먼트 형태로 브로커의 로컬 디스크에 저장된다.

![토픽 / 파티션](/assets/img/posts/kafka-basics-lab/03.png)
_토픽 / 파티션_

![파티션 / 오프셋 / 세그먼트](/assets/img/posts/kafka-basics-lab/04.png)
_파티션 / 오프셋 / 세그먼트_

두 번째는 **'프로듀서(Producer)'와 '컨슈머(Consumer)' 이다.** 프로듀서는 데이터를 카프카 토픽에 발행하는 역할을 하며, 컨슈머는 토픽으로부터 데이터를 구독하여 처리하는 역할을 한다. 둘 다 카프카 클러스터 (*브로커 + 주키퍼)에 대한 클라이언트이다

![Producer &lt;--&gt; Kafka Cluster &lt;--&gt; Consumer](/assets/img/posts/kafka-basics-lab/05.png)
_Producer <--> Kafka Cluster <--> Consumer_

세 번째는 **'브로커(Broker)' 이다.** 브로커는 카프카 서버의 인스턴스로, 토픽의 메시지를 저장하고 컨슈머에게 전달하는 역할을 한다.

![Kafka Cluster Details](/assets/img/posts/kafka-basics-lab/06.png)
_Kafka Cluster Details_

> 카프카 용어 간단 정리  
>  
> - 주키퍼(ZooKeeper) : 카프카의 메타 데이터 관리 및 브로커의 점검 (*Health Check)을 담당한다.  
> - 카프카 클러스터(Kafka Cluster) : 여러대의 브로커로 이루어진 하나의 클러스트  
> - 브로커(Broker) : 카프카 어플리케이션이 설치된 서버(*노드)  
> - 프로듀서(Producer) : 카프카로 메시지를 보내는 역할을 하는 클라이언트  
> - 컨슈머 (Consumer) : 카프카에서 메시지를 꺼내가는 역할을 하는 클라이언트  
> - 토픽 (Topic) : 카프카는 메시지를 토픽으로 구분하고, 각 토픽은 카프카 클러스터 내에 저장됨  
> - 파티션 (Partition) : 병렬 처리 및 고성능을 얻기 위해 하나의 토픽을 여러 개로 구분 한 것  
> - 세그먼트 (Segment) : 프로듀서가 전송한 실제 메시지가 브로커의 로컬 디스크에 저장되는 파일  
> - 메시지 (Messages) 또는 (Record) : 프로듀서가 브로커로 전송하거나 컨슈머가 읽어가는 데이터
>
> ---

---

## AWS에서 Kafka Cluster 구성하기

AWS에서 Managed 서비스로 제공하는 Amazon MSK(Managed Streaming for Apache Kafka) 서비스가 존재하지만 이번 글에서는 위에서 설명한 Kafka Cluster를 EC2 인스턴스를 사용하여 간단하게 Zookeeper 및 Broker 구성 후 토픽을 발행한 후 확인하는 과정까지 해보려고 한다.

해당 LAB은 아래의 깃헙에서 설명하고 있는 책을 참고하여 진행하였다.

[링크](https://github.com/onlybooks/kafka2)

---

### LAB 구성도

![LAB 구성도](/assets/img/posts/kafka-basics-lab/07.png)
_LAB 구성도_

위는 진행하려는 LAB의 구성도이다. 구성도를 이루고 있는 요소들은 아래와 같다.

**Bastion Instance** - Zookeeper / Brokers를 Ansible를 이용하여 배포할 때 사용할 서버이다.

**NAT Gateway** - Private Subnet에 위치할 Kafka Cluster(*Zookeepers + Brokers)의 외부 통신을 위해 준비한다.

**Zookeepers** - Broker를 관리 할 Zookeeper 서비스를 설치할 T3.medium 사이즈의 인스턴스 3대를 준비한다.

**Brokers** - Broker로 사용할 T3.medium 사이즈의 인스턴스 3대를 준비한다.

---

### LAB 1 - Kafka 스크립트를 사용한 Topic 생성 및 Pub/Sub 테스트

윗 Github의 [Chapter2 명령어](https://github.com/onlybooks/kafka2/blob/main/chapter2/2_commands.txt) 를 통해서 Ansble를 활용한 준비된 인스턴스에 대하여 zookeeper와 kafka-server를 각각 설치해준 후 테스트를 위해 kafka server 01에 접속 후 kafka가 제공하는 쉘 스크립트를 통해 Topic 생성 그리고 동일한 노드에서 Producer / Consumer 쉘을 실행하여 토픽을 통해 정상적으로 메시지를 생성하고 읽어올 수 있는지 확인한다.

### LAB 2 - Docker로 Kafka Cluster 구축해보기

자료 : <https://westlife0615.tistory.com/474#8>

[Docker 로 Kafka Cluster 구축해보기.

- 목차 소개. 저는 로컬 환경에서 카프카 관련 테스트를 진행하는 경우가 많이 생기더군요. 그래서 docker-compose 를 활용하여 Kafka, ZooKeeper 클러스터를 구축하는 내용을 작성하려고 합니다. docker-com

westlife0615.tistory.com](https://westlife0615.tistory.com/474#8)

LAB 1 과 같은 방법 이외에 아래와 같이 Docker Compose 를 통해서 도커 이미지로 손 쉽게 Kafka Cluster + kafdrop을 구축 후 Topic을 생성하고 확인할 수 있다.

(1). 아래의 내용의 Docker Compose Yaml 파일 준비

```shell
version: '2'
services:
  kafdrop:
    image: obsidiandynamics/kafdrop:4.0.1
    container_name: kafdrop
    restart: "always"
    ports:
      - "9000:9000"
    environment:
      KAFKA_BROKERCONNECT: "kafka1:9092,kafka2:9092,kafka3:9092"
    depends_on:
      - "kafka1"
      - "kafka2"
      - "kafka3"
    networks:
      - kafka
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.3
    container_name: zookeeper
    restart: "always"
    environment:
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
    ports:
      - "22181:22181"
    networks:
      - kafka
  kafka1:
    image: confluentinc/cp-kafka:7.4.3
    container_name: kafka1
    restart: "always"
    depends_on:
      - zookeeper
    ports:
      - "29091:29091"
      - "29191:29191"
      - "29291:29291"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:9092,EXTERNAL://localhost:29091,EXTERNALDOCKER://host.docker.internal:29191,EXTERNALSERVER://192.168.45.32:29291
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,EXTERNALDOCKER:PLAINTEXT,EXTERNALSERVER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_MESSAGE_MAX_BYTES: 10000000
      KAFKA_SOCKET_REQUEST_MAX_BYTES: 100001200
      KAFKA_SOCKET_RECEIVE_BUFFER_BYTES: 10000000
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 3
    networks:
      - kafka
  kafka2:
    image: confluentinc/cp-kafka:7.4.3
    container_name: kafka2
    restart: "always"
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
      - "29192:29192"
      - "29292:29292"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka2:9092,EXTERNAL://localhost:29092,EXTERNALDOCKER://host.docker.internal:29192,EXTERNALSERVER://192.168.45.32:29292
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,EXTERNALDOCKER:PLAINTEXT,EXTERNALSERVER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_MESSAGE_MAX_BYTES: 10000000
      KAFKA_SOCKET_REQUEST_MAX_BYTES: 100001200
      KAFKA_SOCKET_RECEIVE_BUFFER_BYTES: 10000000
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 3  
    networks:
      - kafka
  kafka3:
    image: confluentinc/cp-kafka:7.4.3
    container_name: kafka3
    restart: "always"
    depends_on:
      - zookeeper
    ports:
      - "29093:29093"
      - "29193:29193"
      - "29293:29293"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka3:9092,EXTERNAL://localhost:29093,EXTERNALDOCKER://host.docker.internal:29193,EXTERNALSERVER://192.168.45.32:29293
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,EXTERNALDOCKER:PLAINTEXT,EXTERNALSERVER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_MESSAGE_MAX_BYTES: 10000000
      KAFKA_SOCKET_REQUEST_MAX_BYTES: 100001200
      KAFKA_SOCKET_RECEIVE_BUFFER_BYTES: 10000000
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 3  
    networks:
      - kafka
networks:
  kafka:
    driver: bridge
```

(2). Kafka 실행

$ docker-compose -f /tmp/kafka-docker-compose.yaml --project-name kafka up -d

```shell
# docker ps
CONTAINER ID   IMAGE                             COMMAND                  CREATED       STATUS       PORTS                                                                                                                                                   NAMES
2b042ccb8dc7   obsidiandynamics/kafdrop:4.0.1    "/kafdrop.sh"            3 hours ago   Up 3 hours   0.0.0.0:9000->9000/tcp, :::9000->9000/tcp                                                                                                               kafdrop
3df15e895e74   confluentinc/cp-kafka:7.4.3       "/etc/confluent/dock…"   3 hours ago   Up 3 hours   0.0.0.0:29093->29093/tcp, :::29093->29093/tcp, 0.0.0.0:29193->29193/tcp, :::29193->29193/tcp, 9092/tcp, 0.0.0.0:29293->29293/tcp, :::29293->29293/tcp   kafka3
2ffd57cad099   confluentinc/cp-kafka:7.4.3       "/etc/confluent/dock…"   3 hours ago   Up 3 hours   0.0.0.0:29091->29091/tcp, :::29091->29091/tcp, 0.0.0.0:29191->29191/tcp, :::29191->29191/tcp, 9092/tcp, 0.0.0.0:29291->29291/tcp, :::29291->29291/tcp   kafka1
81eb96ecd14f   confluentinc/cp-kafka:7.4.3       "/etc/confluent/dock…"   3 hours ago   Up 3 hours   0.0.0.0:29092->29092/tcp, :::29092->29092/tcp, 0.0.0.0:29192->29192/tcp, :::29192->29192/tcp, 9092/tcp, 0.0.0.0:29292->29292/tcp, :::29292->29292/tcp   kafka2
f646c888b259   confluentinc/cp-zookeeper:7.4.3   "/etc/confluent/dock…"   3 hours ago   Up 3 hours   2181/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:22181->22181/tcp, :::22181->22181/tcp                                                                             zookeeper
```

(3). [localhost:9000](http://localhost:9000) 포트로 “kafdrop” 모니터링 접속

Kafka Cluster 전체를 확인 - Zookeeper / Broker (Host 3대)

![](/assets/img/posts/kafka-basics-lab/08.png)

---
