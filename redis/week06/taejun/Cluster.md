# Redis Cluster
- Redis3.0부터 지원한다.
    - Cluster기능을 지원하는 Client를 사용해야 한다.
- 수평확장이다.
    - 최대 1000개의 MasterNode를 가질 수 있다.
- Sharding, Replication, FailOver사용이 가능해진다.
- 하나의 Database(0)만 사용가능하다.
    - StandAlone이나, Replication의 경우 16개 (0~15)가 기본적으로 사용가능하다.
    - 데이터 관리의 단순화와 성능최적화 목적이다.
## Cluster 설정
```shell
# redis.conf
cluster-enabled  yes                                         # Cluste Enable하여 ClusterMode로 실행한다.

# cli
# [host:port]부분은 여러개가 들어 갈 수 있으며, 공백으로 구분된다.
# 처읍부터 MasterNode로 구성되고 나머지가 ReplciaNode로 구성된다.
# 갯수가 맞지 않는다면 충분한 수의 Node가 제공되지 않았다는 오류가발생한다.
redis-cli -cluster create [ip:port] --cluster-replicas 1 # MasterNode에 ReplicaNode를 1개씩 할당한다.
```

## Cluster 상태보기
- Client Library(Lettuce...)는 이 Command를 통해서 Topology를 업데이트 한다.
    - Topology 업데이트에 따라서, Connection Pool도 업데이트한다.
```shell
# 순서없이 랜덤으로 Cluster내의 Node들을 출력한다.
redis-cli cluster nodes

<id> <ip:port@cport> <flags> <master> <pingsent> <pong-recv> <config-epoch> <link-state> <slot> ~ <slot>
```

| 필드명        | 설명                                                                   |
|---------------|----------------------------------------------------------------------|
| id            | Node가 생성될 떄 자동으로 만들어지는 RandomString ClusterId이다. (Immutable)         |
| ip:port@cport | Node의 IP 주소 및 Port, ClusterPort (RedisPort + 10000으로 자동설정된다)         |
| flags         | Node의 상태를 나타내는 Flag                                                  |
| master        | MasterNode 일 경우 '-'가, ReplicaNode일 경우 'MasterNode의 ID'가 표시된다.        |
| pingsent      | 마지막으로 PING 메시지를 보낸 시간, 보류중인게 있다면 0                                   |
| pong-recv     | 마지막으로 PONG 메시지를 받은 시간                                                |
| config-epoch  | 클러스터 구성 변경을 추적하는 Flag  (더 높은 값을 가질수록 우선순위를 가지며, 구성변경이 발생할 때 값이 증가한다) |
| link-state    | ClusterBus에서 아요되는 Link의 상태   (connected/disconnected)                |
| slot          | node가 담당하는 HashSlot 범위                                               |
- flag
    - myself: cli를 통해서 접근 가능한 Node
    - master: MasterNode
    - slave: ReplicaNode
    - fail?: PFAIL상태를 의미하며, 다른Node에 확인을 요청한 상태이다.
    - fail: FAIL인 상태 (과반수 이상의 Node가 Fail임을 인정, PFAIL의 다음상태이다.)
    - handshake: 새로운 Node를 인지하고, handshaking을 하는 단계이다.
    - nofailvoer: ReplicaNode가 FailOver를 시도하지 않을 것임을 의미한다.
    - noaddr: 해당 Node의 주소를 알 수 없음을 나타낸다.
    - noflags: 상태없음


## Sharding
- Sharding에 필요한 모든 기능은, Redis내부적으로 지원된다.
- 하나의 Key는 항상 하나의 MasterNode와 매핑된다.
    - StandAlone에서는 모든 Key가 한 Node에 존재하지만, Cluster는 그렇지 않다.
- Cluster의 모든 Node는 해당 Key에 맞는 Node를 알고있다. (HashSlot)
    - 자신이 담당할 데이터가 아니라면, 해당 Node에게 Redirect 시킨다.
    - 내부적으로 HashSlotMap을 가지고 있고, Redirect를 반영하고 어떤 HashSLot이 어떤 Node에 있는지를 저장한다.
        - ```shell
        # 일반모드
        redis-cli
        set user:1 true # Data Set
        (error) MOVED 10778 192.168.0.22:6379  # HashSlot은 10778이며 해당 HashSlot이 있는 192.168.0.22:6379로 Redirect하라
      
        # Cluster 모드
        rediss-cli -c
        set user:1 true # Data Set
        -> Redirected to slot [10778] located at 192.168.0.22:6379
        OK # Redirect 및 저장 완료
      ``` 
        - MOVED: 영구적인 Redirection (다음 쿼리는 Redirect Node에게 보내라 / HashSlotMap Update(O))
        - ASK: 임시적인 Redirection (Slot Migration 도중, 다음 쿼리도 자신이 받겠다. / HashSlotMap Update(X))

### Resharding
- MasterNode가 갖고있던 HashSlot을 다른 MasterNode로 이동시키는 것이다.
```shell
redis-cli --cluster reshard 192.168.112.4:6379
>>> Performing Cluster Check (using node 192.168.112.4:6379)
M: 8b35f9fa50b0f827408499d2dfb557620e74a9d4 192.168.112.4:6379
   slots:[0-5460] (5461 slots) master
S: 8bb7928fe3816ebd3ac058e69e939f8c8daa9eb9 192.168.112.2:6379
   slots: (0 slots) slave
   replicates 58c705bdd539946720ac648e1400563bdd59fcbe
M: 58c705bdd539946720ac648e1400563bdd59fcbe 192.168.112.3:6379
   slots:[5461-16383] (10923 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 100
What is the receiving node ID? 58c705bdd539946720ac648e1400563bdd59fcbe
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 8b35f9fa50b0f827408499d2dfb557620e74a9d4
Source node #2: all

Ready to move 100 slots.
  Source nodes:
    M: 8b35f9fa50b0f827408499d2dfb557620e74a9d4 192.168.112.4:6379
       slots:[0-5460] (5461 slots) master
  Destination node:
    M: 58c705bdd539946720ac648e1400563bdd59fcbe 192.168.112.3:6379
       slots:[5461-16383] (10923 slots) master
       1 additional replica(s)
```
- Cluster중 하나의 Node를 지정한다.
    - 해당 Node는 Orchestrator가 된다.
    - 해당 Node는 Cluster내부의 다른 Node들에 대한 정보를 가져온다.
    - Master가 아닌, ReplicaNode를 지정하더라도 Resharding의 동작은 동일하게 수행된다.
- **몇개를 옮길 것인지, 어디에서 어디로 옮길 것인지를 중점으로 한다.**
- 자동으로 일어나는 것이 아닌, 수동으로 관리자가 수행해야 한다.


### HashSlot
- 모든 데이터는 HashSlot에 저장된다.
    - HashSlot은 16384개이며, 0~16383으로 나뉘어 있다. (Memory갯수와 상관없는 고정 갯수)
    - 각 MasterNode들은 HashSlot을 나눠갖는다. (ex -> master1: 0~5460, master2: 5461~10922, master3:10923~16383)
    - ```shell
        # Key를 CRC16으로 암호화하고, modular 연산을 수행한다.
        # 각 MasterNode들은 HashSlot Range를 할당받는다.
        HASH_SLOT = CRC16(key) % 16384
    ```
    - Hashslot은 MasterNode에 해당되며, ReplicaNode에는 해당되지 않는다.
- Rebalancing 과정에서, Hashslot이 다른 MasterNode로 이동 할 수 있다.
    - Migration 도중에도 Relocation대상 데이터에 접근 가능하다.
- Gossip을 통해서 전파된다.
    - 하트비트 패킷: PING/PONG을 보낼 때, 자기가 갖고있는 HashSlot을 포함해서 보낸다.
    - 업데이트 메세지: config-epoch 값을 포함하여 업데이트 메세지를 보낸다.

### Rebalancing
```shell
# redis-cli --cluster rebalance 192.168.112.4:6379
>>> Performing Cluster Check (using node 192.168.112.4:6379)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Rebalancing across 2 nodes. Total weight = 2.00
Moving 2831 slots from 192.168.112.3:6379 to 192.168.112.4:6379
```
- Cluster에 있는 MasterNode사이에 HashSlot을 균등하게 재분배한다.
- 자동으로 일어나는 것이 아닌, 수동으로 관리자가 수행해야 한다.


### HashTag
- MGET, MSET과 같은 다중 Key Command는 ClusterMode에서 잘 동작하지 않는다.
    - 각 Key가 여러개의 HashSlot (= 여러개의 MasterNode)에 분산되어 저장되어 있을 수 있기 때문이다.
    - 해당 Node는 그 Key를 처리할 수 있는 다른 Node로 Redirect시키는데, Redirect 대상이 여러개이면 동작할 수 없다.
    - 다중 Key의 경우, 해당 Key들이 모두 같은 HashSlot에 위치해야 한다.
        - 같은 MasterNode에 위치하는 복수의 HashSlot들 이더라도, 복수 KeyCommand는 동작하지 않는다.
        - 다른 HashSlot에 분산되어있으면 원자성 보장이 어렵기 떄문에 간략화하기 위함이다.
- 위와 같은 다중 Key Command의 제약을 극복하는 방법이다.
    - ```shell
      # {} 사이에 있는 값을 해시하여, HashSlot을 배정한다.
      users:{123}:profile
      users:{123}:accout
    ```
        - Key중간을 포함한 어느위치에도 들어갈 수 있다.
- **HashTag기능의 무분별한 사용은 특정 Node의 데이터 쏠림을 야기할 수 있다.**

### Cluster Resharding
- Cluster 실행중에 Node 추가/삭제 가 가능하다.
- MasterNode가 갖고있던 HashSlot을 다른 MasterNode로 이동시키는 것을 의미한다.
    - 새로운 MasterNode의 추가 및 기존 MasterNode의 제거시에 발생한다.
    - Enterprise버전은 Auto-Resharding, Relocation을 지원하지만 Community버전은 지원하지 않는다.
- HashSlot이동에 따른 상태를 가진다.
    - MIGRATING: 해당 HashSlot에 대한 쿼리를 수행하지만, Key가 존재하는 경우에만 처리하고, 없다면 이동 대상 Node로 Redirect한다.
    - IMPORTING: ASKING명령이 선행된 경우에만, 해당 HashSlot 쿼리를 수락하고 아니라면 해당 HashSlot소유자에게 Redirect한다.

## High-Availability
- 최소 3대의 Master와 Replica를 갖게하는 것이 권장설정이다.
- Cluster에 속한 Node들은 서로를 모니터링한다.
    - 모든 Node가 연결되어있는 **Full-Mesh 토폴리지** 형태이다.
    - ClusterBus라는 독립적인 통신을 이용한다.
        - 추가적인 TCP포트를 사용한다.
            - 일반적인 포트에 10000을 더한다. (default 16379)
            - cluster_bus_port를 통해서 설정 가능하다.
    - Gossip Protocol(가십 프로토콜)을 사용하여 오버헤드가 적다.
        - Cluster가 정상적인 상태에서는 많은 Message를 교환하지 않는다.
        - Node의 추가 삭제와 같은 동적설정변경도 Gossip Prtocol을 통해서 전파된다.
        - 어떤 Node가 HashSlotRange를 가지는지도 Gossip을 통해 전파된다.

## FailOver
- Sentinel과 같은 기능을 제공한다.
    - MasterNode 장애시, ReplicaNode를  Master로 승격시킨다.
    - Sentinel은 별도의 Instance를 추가적으로 띄워야했으나 그럴 필요가 없다.
- ClusterBus를 통해서 모든 Instance가 FullMesh 통신하는 구조이며 문제가 생겼을 때는 자동으로 구조를 재조정한다.
- 두가지 종류가 있다.
    - Auto Failover: ReplicaNode를 MasterNode로 승격시킨다.
    - Replica Migration: 잉여 ReplicaNode를 다른 MasterNode에 연결시킨다.

### Master 승격
- 아래의 조건에 해당하면 Replica가 FailOver를 직접 시도한다.
    1. Master가 Fail상태이다.
    2. Master는 1개이상의 HashSlot을 가지고 있다.
    3. Master와 Replication이 끊어진지 오래다.
- 위 조건이 만족하면 FailOver를 수행한다.
    1. config-epoch값을 1 증가시킨다.
    2. Cluster내에 존재하는 모든 MasterNode에 FAILOVER_AUTH_REQUEST 패킷을 보내서 투표를 요청한다.
    - FAILOVER_AUTH_ACK 패킷은 긍정적인  응답을 나타낸다.
    - 현재 config-epoch보다 낮은 값으로 온 FAILOVER_AUTH_ACK는 무시한다.
    3. 과반수 이상의 ACK를 받게되면 MasterNode로 승격된다.
        - 과반수 이상의 ACK가 오지 않는다면 FailOver는 중단된다.
- 지연시간을 갖는다.
    - 고정지연 (500ms)
    - 랜덤지연 (0~500ms)
    - SlaveRank에 따른 지연
        - ```shell
        # Master로 승격될 우선순위가 높은대로 먼저 시도된다.
        DELAY = 500ms + 랜덤 지연시간(0~500ms) + SLAVE_RANK * 1000ms
      ```

### FailOver 설정
- ```shell
     # yes(default): Slave가 없는 MasterNode중 하나라도 장애가 발생했다면 전체 Cluster가 실패한다.
     # no          : 장애가 난 MasterNode를 제외하고는 다른 MastesrNode들은 Cluster에서 동작한다. 
    cluster-require-full-coverage yes     
  
    # Replica의 이동에 해당하는 설정이다.
    # 특정 MasterNode에 Replica가 집중되어있따면 다른 MasterNode로 옮겨서 균등하게 부하를 분산한다.
    # 밑의 Barrier를 초과하면 대상이된다.
    cluster-allow-replica-migration yes
  
    # MasterNode가 가지고 있어야하는 최소 ReplicaNode의 개수 (default: 1)
    cluster-migration-barrier    1
  ```
### 관련 플래그
- [1] PFAIL (Probable Fail)
    - 특정Node 에서 NODE_TIMEOUT이 발생하는 경우에 세워지는 Flag이다.
    - Master, Replica상관없이 PFAIL이 가능하다.
- [2] FAIL
    - 실제 MasterNode가 장애가 발생했다고 인지하며 FailOver를 Trigger하는 Flag이다.
    - 과반수 이상의 MasterNode가 PFAIL로 설정했다면 해당 Node는 FAIL로 변경된다.
    - Flag가 설정되면 FailOver프로세스가 시작된다.
