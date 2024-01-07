Redis의 복제(Replication)는 Redis 클러스터나 Redis Sentinel과 같은 추가적인 고가용성 기능을 제외하고 고가용성(High Availability)과 장애 조치(Failover)를 지원 한다.

`Master-replica` 시스템으로 제공된다.

### `Master-Replica`

`Master-Replica` 가 원활하게 연결되어 있을 경우, Master에서 데이터가 변경되면 Replica에 해당 변경사항을 반영하기 위해 명령 스트림을 보낸다.
Replica는 Master와의 연결이 끊어질 때마다 자동으로 다시 연결을 시도하며, 최대한 Master와 동일한 복제본을 유지하기 위해 노력한다.

만약, `Master-Replica` 간의 연결이 네트워크 문제나 타임아웃으로 인해 끊어지면, Replica는 부분 동기적으로 동기화를 시도하는데, 연결이 끊어진 동안 놓친 명령 스트림만 얻으려는 시도한다.
부분 동기화가 불가능할 경우, Replica은 전체 동기화를 요청하는데, Master가 데이터의 전체 Snapshot을 만들어 Replication에 보낸다.

### Asynchronous replication (비동기 복제), WAIT(동기 복제)
Redis는 낮은 대기 시간과 높은 성능을 제공하기 위해 기본적으로 비동기적으로 Master의 데이터를 복제한다. 이는.
복제의 신뢰도는 Replication에서 주기적으로 Master로부터 수신한 데이터의 양을 비동기적으로 확인하여 어떤 명령 스트림들을 처리하였는지 확인한다.

WAIT 명령을 통해 특정 데이터들에 대해 동기적 복제를 요청할 수 있다.

# Important facts about Redis replication
- 비동기 복제
Redis는 기본적으로 비동기 복제를 사용한다.

- 하나의 마스터는 여러 복제본을 가질 수 있다.

- 복제본 간의 연결
`Replica`는 다른 `Replica`를 가질 수 있다. 하위 `Replica`를 같은 마스터에 연결하는 것 외에도, 계단식 구조로 복제 할 수 있다.
Redis 4.0부터 모든 하위 복제본은 마스터로부터 동일한 복제 스트림을 수신한다고 한다.

- Non-blocking
복제는 Master와 Replica에서 모두 non-blocking으로 동작한다. Master에서 replica 동기화나 복제를 수행하는 중에도 쿼리를 계속 처리할 수 있다.

- 확장성과 고가용성
Replication을 확장성을 위해 사용될 수 있으며, 여러 복제본을 읽기 전용 쿼리(예: 느린 O(N) 연산)에 사용할 수도 있다. 또한 데이터 안전성과 고가용성을 향상시키기 위해 사용하기도 한다.

- Master의 쓰기 비용 절감:
전체 데이터를 persistence하는 비용(쓰기 비용)을 절감하기 위해 복제를 사용할 수 있다. Master의 `redis.conf`를 지속하지 않도록 구성하고, 간헐적으로 저장하거나 AOF(append-only file)를 활성화한 Replica를 통해 데이터를 디스크에 쓰는 방식으로 사용한다.
Master에서 persistence를 끄고 사용하는 경우 다음과 같은 상황에 주의하여야 한다.
> Master A가 영속화 옵션 없이 replica에 복제만 하는 경우, A가 다운 되었을 때 failover를 통해 재로딩 된다. 이때, Master A는 비어있는 데이터 셋으로 로딩 되고, 이를 replication 하는 노드들이 동기화를 하면서 이전 데이터 셋을 소실 시킬 수도 있다.

# How Redis replication works
Redis 복제 작업은 다음과 같은 방식으로 동작한다.

Redis의 마스터는 고유하고 랜덤한 Replication ID를 가진다. 또, 연결된 Replica가 존재하지 않아도 incremented offset을 가지는데 이는 복제가 이루어져 Replica에 변경사항이 전달되면 증가한다.

기본적으로 `Replication Id`와 `offset`은 Master의 데이터에 대한 고유한 버전 식별로 사용 될 수 있다.

`Replica`가 `Master`에 연결할 때, `PSYNC` 명령을 사용하여 이전 `Master`의 `Replication Id`와 지금까지 처리한 `offset`을 보내어 필요한 데이터 복제 부분만 보낼 수 있게 된다.
만약 `Master`의 버퍼에 백로그가 없거나, `Replication Id`가 더 이상 유효하지 않은 Id를 참조하는 경우, 전체 동기화가 발생하게 된다.

전체 동기화가 발생하면 Master에서는 백그라운드에서 저장 프로세스가 시작되어 RDB파일을 생성하고, 동시에 클라이언트로부터 받고 있는 새로운 쓰기 명령을 버퍼링하기 시작한다. 
이후 백그라운드 저장이 완료되면, RDB파일을 먼저 복제본에 전송하고 이후 버퍼에 담긴 쓰기 명령들을 보낸다.

# Read-only replica

Redis 2.6부터 replica는 읽기전용 모드가 기본으로 되어 있다. redis.conf 파일에 `replica-read-only` 옵션이나 runtime 중에는 `CONFIG SET`을 통해 설정할 수 있다.

읽기 전용 복제본은 모든 쓰기 명령을 거부합니다. 실수로 복제본에 데이터가 쓰여지는 것을 방지하기 위해 모든 쓰기 명령은 거부한다. 그렇다고 해서 Replica를 인터넷과 같은 public한 네트워크에 노출해도 된다는 의미는 아니라고 한다.

쓰기 가능한 Replica는 Master와의 데이터 정합성에 큰 문제가 발생할 수 있으므로 권장하지 않는다.

# Allow writes only with N attached replicas
Redis 2.8 부터 Redis master에 N개의 Replica가 연결된 경우에만 쓰기 쿼리를 수행한다.
하지만, Redis는 비동기 복제를 사용하기 때문에 복제본이 특정 쓰기를 실제로 받았는지를 보장할 수 없기 때문에 데이터가 유실 될 수 있다.

이러한 문제에 대해 Replica와 Master가 `PING-PONG`을 주고 받으며 replica에서 처리한 복제 스트림의 양을 확인한다.
그리고 Master에서는 각 Replica으로부터 마지막으로 핑을 받은 시간을 기억 해 두어, 지연 시간을 측정한다.
사용자나 지정한 N개의 복제본과, M초 미만의 지연의 경우에만 쓰기 작업이 허용되도록 구성 할 수 있다.
```
min-replicas-to-write <N>
min-replicas-max-lag <M>
```

조건이 충족되지 않으면 Master는 오류로 응답하고 쓰기를 작업을 수행하지 않는다.

# How Redis replication deals with expires on keys
Replica 에서는 key를 스스로 만료시키지 않는다. Master와 Replica의 시계의 동기화로 의존한다면 데이터 일관성의 큰 결함을 초래할 수 있기 때문이다.

다음과 같은 기술을 통해 만료시킨다.

Master에서 키를 만료(또는 LRU로 인해 제거)되면, `DEL` 명령을 모든 복제본에 전송하여 제거한다.

이러한 방식은 Replica에서 `DEL` 명령을 제공받지 못한 경우 문제가 발생 할 수 있다. 논리적으로 만료된 키가 유지 될 수 있는데, 이를 처리하기 위해 논리적 시계를 사용하여 일관성을 위반하지 않는 읽기 작업에 대해서만 키 만료를 전달한다.


# Partial sync after restarts and failovers

Redis 4.0부터, 장애 조치(Failover) 후 Master로 승격된 인스턴스는 이전 마스터의 복제본과 부분 동기화를 수행할 수 있다. 이 과정에서 승격된 Replica는 이전 Master의 Replication ID와 offset을 통해 백로그의 일부를 제공할 수 있다.

이때 승격된 Master는 새로운 복제 ID를 가지게 된다. 이전 Master가 복구되어 돌아오면, 중복되는 `Replication Id, offset` 쌍이 존재할 수 있기 때문이다.

Graceful 종료 후 재시작된 경우 RDB 파일에 Master와 재동기화하는 데 필요한 정보를 저장하여 쉽게 부분 동기화가 이루어질 수 있다.

AOF 파일을 통해 재시작된 복제본은 부분 동기화를 수행할 수 없다고 한다. 그러므로 인스턴스를 종료하기 전에 RDB 지속성으로 전환한 다음 재시작을 수행하고, 다시 AOF 방식을 활성화 하여야 한다.
