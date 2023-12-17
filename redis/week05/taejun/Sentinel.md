# Sentinel이란?

- 고가용성을 유지하는 방법 중 하나이다.
- Replica구성에서의 자동 FailOver기능을 제공한다.

### Sentinel이 제공하는 기능

1. 모니터링
    - master node
    - replicaNode에 대한 간접 모니터링
2. FailOver
    - MasterNode의 비정상적인 상태를 감지 후, ReplicationNode를 Master로 승격시킨다.
    - 기존 MasterNode에 연결되어있었던 Replica들은 승격된 Master에게로 다시 연결된다.
3. Instance구성정보 제공
    - Client에게 현재 구성에서의 MasterNode의 정보를 알려준다.
    - 변경된 Master의 정보를 알려주기 때문에, EndPoint변경을 어플리케이션코드에 구성할 필요 없다.
4. Notification
    - 감시하고 있는 RedisInstance들이 문제가 생겼을 때, 알려준다.
    - pubsub을 통해서 Client에게 알리거나, shell script를 통해서 E-Mail, SMS알림도 가능하다,.

### Sentinel이 없을 때, Replication의 FailOver

1. ReplicaNode에 접속한다.
2. REPLICA OF NO ONE 커맨드를 통해서 ReadOnly모드를 해제한다.
3. Application에서, 바라보고 있던 Master Redis의 EndPoint를 ReplicaNode의 것으로 변경 후 배포

### Sentinel의 FailOver

1. 수동 FailOver

    ```bash
    SENTINEL FAILOVER <master-name>
    ```

2. 자동 FailOver
    - 주기적으로 masterNode에 PING을 보내서 정상여부를 판단한다.
    - sentinel.conf에 구성되어 있는 down-after-milliseconds시간동안 정상응답이 오지 않으면, DOWN상태로 판단하고 FailOver가 Trigger된다
    - Master승격은 아래의 우선수위를 가진다.
        1. redis.conf에 구성된 priority가 낮은 ReplicaNode
        2. Master와 가장 일관성이 근접한 Node (master_repl_offset 기준)
        3. 2번까지의 기준이 동일하면 run_id가 사전순으로 작은 ReplicaNode

### Sentinel의 Down판단

1. SDOWN
    - Sentinel의 주관적인 판단을 의미한다.
    - 각각의 Sentinel들의 판단이다.
    - FailOver시, SDOWN상태인 ReplicaNode는 승격대상이 되지 않는다.

    ```bash
    # 다른 Sentinel에게 Down여부 질의
    SENTINEL is-master-down-by-addr
    ```

2. ODOWN
    - SDOWN이 Quoroum값(정족수)을 넘으면 ODOWN 상태가 된다.
    - ODOWN은 MasterNode에만 해당되며, 다른 Node들은 이상태를 갖지않는다.

### Sentinel의 FailOver 이후의 모니터링

1. 모니터링 설정 초기화
    - FailOver이후에도, 비정상적으로 판단되는 Node에 대한 모니터링 시도를 계속한다.
    - 만약 모니터링에서 제외하고싶다면 상태정보를 초기화 해야한다.
    - SENTINEL끼리는 down여부를 판단하지 않기 때문에 꼭 초기화 해주어야한다.

    ```bash
    SENTINEL RESET <master-name>
    ```

2. Sentinel 노드 추가 / 제거
    - 모니터링하고자하는 master-name을 지정하기만 하면된다.
    - 자동검색 매커니즘을 통해서 다른 Sentinel들의 Known-list에 추가된다.
        - pubsub으로 구성된다.

### Sentinel의 인스턴스 구성정보 제공

1. Client가 Sentinel에게 MasterNode의 구성정보를 질의
2. Sentinel이 MasterNode의 정보 반환
3. Client가 Master와 Connection

### Sentinel 구성

- 기본적으로 23679포트를 사용한다.
- 단일 Instance로 구성되어있을 경우, SPOF가 될 수 있다.
- 최소3대 이상으로 구성하는 것이 권장되며, 하나의 Sentinel에 문제가생겨도 나머지는 정상적으로 동작한다.
    - 보통 3대로 구성하고, 안정성을 위해서 5대로 구성하는 경우도 많다.
    - 각각 다른 물리적 Instance 혹은 AZ에 구성한다.
- Quorum(정족수)를 사용한다.
    - Master가 비정상적이라는 것의 판단은 과반수가 넘어야 인정이된다.

### Sentinel Leader

- Sentinel의 특정 결정을 내리는 역할을 한다.
- FailOver과정에서 주도적인 역할을 한다.

### Sentinel 환경설정

```
# sentinel.conf
port 23679
sentinel monitor <monitoring 할 master-node 명> <master-ip> <master-port> <quorum-count>
```

- 일부설정은 CONFIG SET을 통해서 변경이 가능하지만, 재시작해야 하는 설정들도 있다.
    - 대부분의 주요설정 변경, Cluster구성이나 Master Monitoring대상 변경은 재시작해야한다.
- 시작하는 방법은 2가지이다.

    ```bash
    # redis-sentinel
    redis-sentinel <path-to-sentinel.conf>
    
    #redis-server
    redis-server <path-to-sentinel.conf> --sentinel
    
    # 접속
    redis-cli =p <sentinel-port>
    ```

- Sentinel 정보 (info sentinel)

    ```bash
    > info sentinel
    # Sentinel
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_tilt_since_seconds:-1
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=mymaster,status=sdown,address=:6379,slaves=0,sentinels=1
    ```


### Sentinel 제공 정보

1. master 정보

    ```bash
    SENTINEL master <master-name>
    ```

2. Replica 정보

    ```bash
    SENTINEL replicas <master-name>
    ```

3. Sentinel  정보

    ```bash
    SENTINEL sentinels <master-name>
    ```

4. Sentinel Quorum 가용성 여부

    ```bash
    SENTINEL chquoroum <master-name>
    ```

    - 작업 수행가능한 Sentinel의 수와, Quoroum 및 FailOver이 수행가능한지를 리턴한다.