## Redis Replication이란?

- Sentinel, Clustering을 제외 했을 때, 간편하게 구성할 수 있는 고가용성(HA)확보 방법이다.
    - Replica는 Master와 연결이 끊어질 때마다 재연결을 시도한다.
    - master의 상태에 상관없이 항상 정확한 Replica를 얻기위해서 노력한다.
- 멀티 마스터구조의 replica는 불가능하다.
    - write연산은 master에서, read연산은 Replica에서 수행해야한다.
- Master는 N개의 Replica를 가질 수 있다.
- Replica의 Replica도 가질 수 있다. (특정환경에서 유리)
    - Replica Chaning, Replica Cascading …
    - Master의 복제연산에 대한 부하를 분산할 수 있다.
    - 지리적 분산 환경에서, 가까운 ReplicaNode끼리 복제를 수행하여 네트워크 성능을 최적화 할 수 있다.
    - 각각의 네트워크에서 수행될 것이기 떄문에 대역폭 최적화 관점에서도 유리하다.
- RDB는 Replicaion과정에서 주로 사용되지만, AOF는 사용되지 않는다.
    - RDB옵션을 꺼두어도, Replicaion이 Enable되있으면 RDB저장이 수행된다.
    - diskless옵션을 키면 RDB가 저장되지 않는다.

### 복제 Case

```bash
(1) Master와 Replica가 잘 연결되어 있을 때 - ReplicaionBuffer를 이용한 명령어 전달
(2) Master와 Replica의 연결이 끊어졌을 때  - 부분 재동기화 시도
(3) 부분 재동기화가 불가능 할 때            - 전체 재동기화 시도
```

### 부분 재동기화

- 연결이 끊어질 때마다 매번 RDB 스냅샷 파일을 전송하는 것은 매우 비효율적이다.
- Master는 Connection 유실에 대비하여, BacklogBuffer라는 메모리공간을 둔다.
    - 이 곳에는, Replica에 전달할  Command들을 저장해둔다.
    - BacklogBuffer Size에 대한 설증은 redis.conf에서 구성가능하다.
- ReplicaNode는 재연결이 되었을 때, MasterNode에게 자신의 ReplicaionId와 Offset을 전달한다.
- 충분한 Backlog가 없거나, 알 수 없는 ReplicaionId를 참조하는 경우 재동기화가 발생한다.

### 전체 재동기화

1. RDB 스냅샷 생성: 전체 재동기화 과정의 첫 단계로, 마스터 인스턴스는 자신의 전체 데이터셋에 대한 스냅샷을 생성한다.
2.  RDB 스냅샷 전송: 생성된 스냅샷은 복제 인스턴스로 전송한다. 네트워크 대역폭과 데이터 크기에 따라 시간이 소요될 수 있다.
3. 명령어 스트림 전송 재개: 스냅샷 전송 후, 마스터 인스턴스는 복제 인스턴스에 데이터셋 변경 사항을 반영하는 명령어 스트림의 전송을 재개한다.

### Command

```bash
# 복제 프로세스 시작
REPLICAOF <master-ip> <master-port>

# Master와 ReplciaNode가 연결되어 있을 때, 동기화 수행 (ReplicaNode가 내부적으로 수행)
# 기존에 Sync명령어가 존재했으나, 부분 재동기화를 지원하지 않는관계로 PSYNC가 사용된다.
PSYNC <replicaion-id> >offset>

# 설정 적용
CONFIG SET masterauth <master-node-password>
CONFIG REWRITE
```

- REPLICAOF명령어를 수행하는 RedisServer가 Replica가 되는 것이다.
- 기본적인 Password를 사용해서, 데이터를 복제 할 때는 masterauth옵션에 패스워드를 입력해야한다.
    - masterauth를 통해서, Replica가 master에 접근한다.
    - CONFIG REWRITE를 통해서, 설정을 적용시킨다.

### 과정

```
(1) REPLICAOF를 통해서, 복제 연결을 시도한다.
(2) MasterNode에서는, fork()를 통해서 자식 Process를 생헝 한 후 RDB Snapshot을 생성한다
(3) MaserNode에서 수행된 DataSet의 변경은 RESP형태로 Master ReplicationBuffer에 저장된다.
(4) RDB파일이 생성 완료되었다면, ReplicaNode에게 소켓통신을 통해서 diskless하게 전달된다.
(5) Replica는 받아온 RDB를 디스크에 저장한다.
(6) Replica에 저장되어있던 모든 내용을 삭제한 후, RDB파일을 이용하여 데이터를 로딩한다.
(7) Replica과정동안 Buffering됐던 데이터를 Replica에 전달하여 수행시킨다.
```

- RDB Snapshot은 Redis 전체의 복사본이다.
- RDB와 ReplicaionBuffer 두가지 방법을 사용하는 이유는 효율성 + 일관성의 목적이다.
    - RDB: 초기 DataSet을 제공하기 위해서 사용된다.
    - ReplicaionBuffer: 실시간으로 데이터를 추적하는데 사용된다.


### 비동기 (기본)

- 유실이 발생할 수 있다
    - Client와 Master간의 Command 및 응답이 선행된다.
    - 그 이후에 Master와 ReplicaNode간의 통신이 수행된다.
    - ReplicaNode와의 통신 이전에 Master의 비정상적인 종료의 경우 유실이 발생할 수 있다.
- Replica 속도가 굉장히 빠르기 떄문에, 유실이 빈번하게 발생하지는 않는다.

### 동기

- WAIT명령을 수행한다.
    - 지정된 수의 Replica가 Write명령을 받을 때 까지 대기한다.
- 데이터의 내구성은 향상시키지만, 성능을 저하시키게된다.
- 동기방식이 CP시스템을 보장하지는 않는다. (Consistency - Partition Tolerance)
    - 여전히 데이터 유실의 가능성은 있다는 것을 의미한다.
    - 장애발생 및 설정에 따라서 달라질 수 있기 떄문이다.

### Replicaion ID

```
> INFO Replication

# Replication 
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:277a883084000887a988cdfc32dff307444957e5
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:723
slave_priority:100
slave_read_only:1
connected_slaves:0
master_fail_over_state:no-failover
master_replid:277a883084000887a988cdfc32dff307444957e5
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

```

- RandomString값을 가지며, Offset과 쌍으로 유지된다.
- ReplicaionId가 같다는 것은 같은 DataSet을 구성하고 있다는 것을 의미한다.
    - ReplicaNode는 ReplicaitonConnection이 맺어지면 Master의 ReplicaionId를 상속받는다.
- Replication상태를 추적하는데 사용된다.
    - INFO REPLICAION 을 통해서 Replica상태를 파악 할 수 있다.
- Offset은 Replica에서 마지막으로 동기화된 Master의 Offset을 나타낸다.
    - Offset이 같으면 정확하게 일관성이 유지된 상태인 것이다.
- Secondary ReplicaionId는 Replica가 Master로 승격했을 떄를 구분하기 위함이다.
    - 새로운 Master가 선출 되면 새로운 ReplicaionId를 생성하게 된다.
    - 이전 Master노드의 ReplicationId는 SecondaryReplicaionId가 된다.
    - 단순히 Network문제로 인해서, 승격이 결정되고 이전 Master가 그대로 동작하고있다면 같은 DataSet을 가지고있다는 사실을 위반하게된다.

### Replica에서의 Expire

- ReplicaNode는 스스로 Key를 Expire시키지 않는다.
    - MasterNode가 Key를 만료시키거나, LRU를 통해서 제거될 때 까지 기다린다.
    - Master는 Expire가 된다면 ReplicaNode에게 DEL명령을 전달한다.
- 일관성이 깨질 수 있다.
    - Master의 DEL명령어가 전달되지 않는다면?
    - 읽기작업시 자체적인 논리적 시계를 사용하여, 만료되었다고 판단되는 데이터는 반환하지 않을 수 있다.
        - Master와는 독립적으로 동작하는 시계이며, Expire된 Key를 추적한다.
- Master의 DELETE명령어를 대체할 자체적인 프로세스는 가지고 있지 않다.