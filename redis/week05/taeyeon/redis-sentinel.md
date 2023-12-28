# Redis Sentinel
Clustered가 아닌 환경에서 고가용성 제공하기
Redis Sentinel은 Redis 클러스터를 사용하지 않는 환경에서 고가용성을 제공하는 기능을 한다.

전체적으로, 아래와 같은 기능을 제공한다.
- `Monitoring`
Master와 Replica 인스턴스의 동작을 모니터링
- `Notification`
모니터링 API를 통해 Redis 인스턴스에서 문제가 발생하면 알림 제공
- `Automatic failover`
Master instance가 다운되는 경우, Sentinel이 Replica를 마스터로 승격시켜 장애 조치

Sentinel은 분산 시스템으로서 여러 프로세스가 함께 협력하는 system에서 실행되도록 설계되어있다.
여러 Sentinel을 사용하면 Master에 대한 장애 감지가 여러 Sentinel을 통해 관리되므로 오탐이 될 가능성이 낮아져 신뢰도가 높아진다.
또, Sentinel 자체에 대한 가용성도 높아지므로 단일 Sentinel의 장애가 발생해도 전체 Redis 시스템에 장애로 이어지지 않는다.

# Running Sentinel
Redis Sentinel을 배포하기 전에 고려해야할 몇가지 기본 사항들이 있다.
>
- 적어도 세 개의 Sentinel 인스턴스 사용하기
- Sentinel 인스턴스는 독립적인 환경에서 실행하기(SPOF 피하기)
- 기본적으로 Redis는 비동기 복제가 이루어지기 때문에 데이터 유실 가능성 존재
- 사용하는 Redis 클라이언트의 Sentinel 지원 여부
- 정기적인 테스트
- Sentinel과 Docker, NAT의 사용시 포트 매핑에 대한 [주의사항 숙지하기](https://redis.io/docs/management/sentinel/#sentinel-docker-nat-and-possible-issues)

Sentinel 실행 방법
```shell
> redis-sentinel /path/to/sentinel.conf 
# or
> redis-server /path/to/sentinel.conf --sentinel
```
위 명령어를 통해 Sentinel을 실행할 수 있다.

Sentinel을 실행할 때는 반드시 설정 파일을 사용해야 한다.
현재 상태를 저장하고, 재시작 시 다시 로드하는 데 사용된다.
설정 파일이 제공되지 않거나 설정 파일 경로에 **쓰기 권한이 없는 경우**, Sentinel은 시작되지 않는다.

기본적으로 Sentinel은 TCP 포트 26379를 점유하여 연결을 기다리기 때문에 Sentinel이 구동중인 서버의 26379 포트가 다른 Sentinel 인스턴스의 IP 주소로부터의 연결을 받을 수 있도록 열려 있어야 한다.

# Configuring Sentinel
Redis를 설치하면 `sentinel.conf`라는 파일이 기본적으로 포함되어 있다.
`sentinel.conf`
```yaml
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```
각각의 구성요소는 다음과 같다.
`sentinel monitor <master-name> <ip> <port> <quorum>`
Sentinel에게 주어진 이름, IP 주소, 포트를 가진 마스터를 모니터링 하도록 설정하는 명령이다.
`<quorum>`은 마스터가 도달할 수 없다고 판단하는 데 필요한 Sentinel의 최소 수를 지정하는 옵션.

`sentinel down-after-milliseconds <master-name> <time>`
지정한 `<time>`만큼 Master에 도달할 수 없을 때 Sentinel이 Master를 실패한 것으로 간주하는 시간을 설정하는 명령.

`sentinel failover-timeout <master-name> <time>`
failover 프로세스의 timeout 시간 설정

`sentinel parallel-syncs <master-name> <number>`
failover 후 동시에 동기화할 수 있는 슬레이브의 최대 수를 설정.

# Example Sentinel deployments
[Redis docs](https://redis.io/docs/management/sentinel/#example-sentinel-deployments)에서Sentinel을 얼마나, 어디에 구성하고 사용하는지에 대한 예제를 제공한다.
제공하는 ASCII 아트와 함께 간단한 예시들을 살펴보자.
`legend`
```shell
#Master 노드: M1, M2, M3, ...
#Replica 노드: R1, R2, R3, ...
#Sentinel 노드`: S1, S2, S3, ...
#Client 노드: C1, C2, C3, ...

#Sentinel의 개입으로 Master로 승격된 노드`: [M1]

+--------------------+
| This is a computer |
| or VM that fails   |
| independently. We  |
| call it a "box"    |
+--------------------+
```

### 2개의 Sentinel 구성
> 사용하면 안되는 이유를 보여주는 예제

```ASCII
+----+         +----+
| M1 |---------| R1 |
| S1 |         | S2 |
+----+         +----+

Configuration: quorum = 1
```
이러한 구성에서는 `M1`에서 장애가 발생하면 `S1`, `S2`가 합의하여 장애로 인식하고 `R1`을 승격할 수 있다.
하지만, `M1`이 구동중인 box(server)가 다운된 경우 `S1` 역시 작동을 멈춰 `S2`가 장애 판단을 내려도 합의가 되지 않았기 때문에 `R1`이 승격 될 수 없다.
하나의 Sentinel의 승인으로 failover를 감지하는 설정은 매우 위험하기 때문에 최소 3개의 독립적인 box에서 Sentinel을 구성하자.

### 3개의 Sentinel 구성 (basic practice)
```ASCII
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```
가장 기본적이고 간단한 구성 방식이다.
이전 예시처럼 `M1`의 `box`가 fail 되어도 `S2`, `S3`의 합의를 통해 `failover`가 수행 될 수 있다.

이 방식에도 한가지 문제가 발생 할 수 있다.
```ASCII
         +----+
         | M1 |
         | S1 | <- C1 (writes will be lost)
         +----+
            |
            /
            /
+------+    |    +----+
| [M2] |----+----| R3 |
| S2   |         | S3 |
+------+         +----+
```
위 그림처럼 failover가 수행되어 `[M2]`가 승격된 상황에서, 이전 `M1`에 `C1`의 쓰기작업이 비동기 Replica로 인해 전달되지 않고 유실 될 수 있다.

이러한 문제는 redis.conf에 다음과 같은 설정을 통해서 완화 할 수 있다.
```
min-replicas-to-write 1
min-replicas-max-lag 10
```
위 설정을 통해 `M1`이 적어도 하나의 `Replica`에 쓰기 작업을 하지 못하는 경우 Client의 쓰기 작업을 거부한다. `min-replicas-max-lag 10`을 통해서 복제를 하기 못하는 기준을 지정할 수 있다.

이 해결책은 `Master`의 쓰기 작업이 `Replica`에 의존적이라는 단점이 있다. `Master`는 정상인데 `Replica`가 다운되어 복제가 되지 않는 경우에도 `Master`에서는 쓰기 작업을 거부하는 상황이 발생 할 수 있다.

### Client에 Sentinel을 구성하는 경우
```ASCII
            +----+         +----+
            | M1 |----+----| R1 |
            | S1 |    |    | S2 |
            +----+    |    +----+
                      |
               +------+-----+
               |            |
               |            |
            +----+        +----+
            | C1 |        | C2 |
            | S3 |        | S4 |
            +----+        +----+

      Configuration: quorum = 3
```
이렇게 구성하는 경우, Client의 수에 따라 quorum의 값을 유동적으로 조절할 필요가 있다.

# Sentinel API
Sentinel은 상태를 검사하고, 모니터링되는 마스터 및 복제본의 상태를 확인하고, 특정 알림을 수신하기 위해 구독하고, 런타임에 Sentinel 구성을 변경하기 위한 API를 제공한다.

제공하는 API를 통해서 Sentinel에 직접 쿼리하여 모니터링되는 Redis 인스턴스의 상태를 확인하거나 Pub/Sub를 사용하여 장애 조치와 같은 이벤트에 대해 알림을 받을 수 있도록 구성할 수 있다.
다양한 API들이 제공되므로 문서에서 확인하자.
> https://redis.io/docs/management/sentinel/#sentinel-api

> 자주 사용되는 Sentinel 명령어
monitoring 시작: `SENTINEL MONITOR mymaster 127.0.0.1 6379 2`
failover 조치 시작: `SENTINEL FAILOVER mymaster`
master 상태 확인: `SENTINEL MASTER mymaster`
모니터링 중인 마스터 목록 보기: `SENTINEL MASTERS`
특정 마스터의 복제본 목록 확인: `SENTINEL REPLICAS mymaster`

## Reconfiguring Sentinel at Runtime (API)
API를 통해 Runtime중에 Sentinel의 구성을 수정할 수 있다.
하나의 Sentinel에 구성을 수정하는 경우, 전파가 되지 않으므로 모든 Sentinel구성을 고려하여 사용해야 한다.

- `SENTINEL MONITOR <name> <ip> <port> <quorum>`
새로운 마스터를 모니터링
- `SENTINEL REMOVE <name>` 지정된 마스터를 제거
- `SENTINEL SET <name> [<option> <value> ...]`
특정 마스터의 구성 매개변수를 변경.
	- `SENTINEL SET objects-cache-master down-after-milliseconds 1000`
    - `SENTINEL SET objects-cache-master quorum 5`

Redis 6.2 이후 버전 부터는 전역 config 매개변수를 가져오거나 설정 할 수 있다.
- `SENTINEL CONFIG GET <name>`
- `SENTINEL CONFIG SET <name> <value>`
> 조작 가능한 전역 매개변수
`resolve-hostnames, announce-hostnames` 
`announce-ip, announce-port`
`sentinel-user, sentinel-pass`


# Adding or removing Sentinels
새 Sentinel 추가: 새로운 Sentinel을 시작하고 현재 활성 마스터를 모니터링하도록 구성합니다. 10초 이내에 다른 Sentinel과 복제본 세트 목록을 획득합니다.
여러 Sentinel을 한 번에 추가할 때는 하나씩 추가하고, 다른 모든 Sentinel이 첫 번째 Sentinel에 대해 알고 나서 다음 Sentinel을 추가하는 것이 좋습니다.
Sentinel 제거: 제거하고자 하는 Sentinel 프로세스를 중지하고, 다른 모든 Sentinel 인스턴스에 SENTINEL RESET * 명령을 보냅니다.
오래된 마스터나 도달할 수 없는 복제본 제거:

Sentinel은 오랫동안 도달할 수 없더라도 주어진 마스터의 복제본을 잊지 않습니다.
복제본(예전 마스터일 수 있음)을 영구적으로 제거하려면, 모든 Sentinel에게 SENTINEL RESET mastername 명령을 보내야 합니다.

# Sentinels and replicas auto discovery
Sentinel은 다른 Sentinel과 연결되어 서로의 가용성을 확인하고 메시지를 교환한다.
Sentinel 인스턴스를 실행할 때 다른 Sentinel 주소 목록을 구성할 필요가 없는데, Sentinel은 Redis 인스턴스의 Pub/Sub 기능을 사용하여 동일한 Master와 Replica를 모니터링하여 다른 Sentinel을 자동으로 발견하기 때문이다.

이 기능은 `__sentinel__:hello` 채널에 'hello' 메시지 발행하는 방식으로 동작한다.
각 Sentinel은 모니터링하는 `Master`와 `Replica`의 Pub/Sub 채널 `__sentinel__:hello`에 매 2초마다 메시지를 발행하여 본인의 `IP`, `포트`, `runid`를 포함한 본인(Sentinel)을 알린다. 그리고 이 채널을 모든 `Sentinel`들이 구독하여 다른 `Sentinel`의 메세지를 확인하는 방식으로 찾는다.

# Replica selection and priority
`Master`가 ODown 상태일 때, Master로 승격할 Replica의 우선순위를 설정하여 지정할 수 있다.
이때, 다음과 같은 정보들을 평가하여 승격할 Replica를 선택한다.
- `Master와의 disconnection 시간`
- 지정된 우선순위
- 복제 정도
- Run id

특히 `Master와의 disconnection 시간`이 설정 되어있는 `down-after-milliseconds option` 보다 10배 이상의 시간이었다면 승격 후보에서 제외한다.
> `(down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state` 보다 작아야 한다

먼저 Redis 인스턴스의 `redis.conf` 파일에 설정된 `replica-priority`에 따라 선택된다.
예를 들어 `S1`을 우선순위 10으로, `S2`를 우선순위 100으로 설정한 경우 `S1`이 우선적으로 선택된다.
이후, 우선순위가 동일한 경우에는 Master로부터 더 많은 데이터를 받은 Replica가 선택된다.
우선순위도 동일하고, 동일한 양의 데이터를 받은 경우 Run-id의 사전 순으로 선택된다.

우선 순위에서 `replica-priority`를 0으로 설정하면 가장 최우선이 아닌 Sentinel에 의해 Master 승격에서 제외되도록 구성된다.
