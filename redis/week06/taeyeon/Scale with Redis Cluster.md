> https://redis.io/docs/management/scaling/


# Scale with Redis Cluster
`Redis 클러스터`라는 배포 topology로 수평적 확장이 가능하다.


# Redis Cluster 101
`Redis Clutser`는 데이터가 여러 Redis 노드에 자동으로 샤딩되며 Redis를 운영/실행하는 방법을 제공한다. `Redis Cluster` 노드의 일부에 문제가 발생해도 운영가능하며 어느 정도의 가용성을 기대할 수 있다. `Master` 다운되면 `Redis Cluster`를 사용할 수 없다.

이러한 목적으로 다음과 같은 기능을 사용할 수 있다.
1. 데이터를 여러 노드에 자동으로 분할
2. 노드의 하위 집합이 다운되도 Client와의 작업을 계속 수행 할 수 있다.(가용성)

### Redis Cluster TCP ports
모든 `Redis Cluster`는 클라이언트와 통신하는(e.g. 6379) 포트와 _`bus port`_라고 하는 포트가 필요하다. 일반적으로 `bus port`는 데이터 포트에 10000을 추가하여 설정되지만(e.g. 16379) 변경할 수 있다.

`bus port`를 통해 `Cluster`는 장애 감지, 구성 업데이트, 장애 조치 권한 부여 등을 통신한다.그렇기 때문에 Client에서 해당 `bus port`를 사용하지 않도록 하는 것이 좋다.

Redis Cluster가 정상적으로 동작하기 위해선 `data port`(for client)와 `bus port`가 모두 개방되어 있어야 한다.

### Redis Cluster data sharding
`Redis Cluster`는 일관된 해싱 방식을 사용하지 않고, 데이터의 키가 개념적으로 해시 슬롯이 되는 형태로 샤딩을 제공한다. 16384개의 해시 슬롯이 있는데, 키에 대한 [CRC16](http://wiki.hash.kr/index.php/CRC-16)를 연산한 뒤 16384로 모듈로(modulo, `%`)연산을 수행하여 할당한다.

`Redis Cluster`의 각 노드는 해시 슬롯의 하위 집합을 담당하게 되고, 3개 노드로 구성된 `Cluster`에서 `노드 A`는 `0 ~ 5500`, `노드 B`는 `5501 ~ 11000`, `노드 C`는 `11001 ~ 16383` 까지의 해시 슬롯을 담당하게 된다.
이렇게 구성하여 노드의 변동에 대해 해시 슬롯 재분재를 통해 작업이 간단하며 중단될 필요가 없어진다.

또, `Redis cluster`는 node가 자체적으로 `master-replica` 모델을 사용한다. 만약 위 예시에서 `노드 B` 에 장애가 발생하면 `5501 ~ 11000`의 해시 슬롯 사용이 불가하여 `Cluster`에 문제가 발생할 수 있기 때문에 replica 노드를 운용하여 가용성 높은 Cluster 환경을 제공한다.

### Redis Cluster consistency guarantees
Redis Cluster 는 강력한 일관성(strong consistency)를 보장하지 않는다고 한다. 특정 조건에서 승인된 쓰기 작업이 `Redis Cluster`에서 손실 될 수 있다는 것이다.

아래와 같은 이유들로 `"강력한 일관성"`이 보장되지 않는다.
- 비동기 복제 사용
Redis Cluster는 비동기 복제를 사용하기 때문에 복제 대상에게 잘 전달되었는지에 대한 응답 없이 client에게 쓰기 응답을 전달한다. 이 상황에서 Master만 쓰기 작업을 수행한 후, 복제본에 쓰기 전파를 하기 전에 다운되고 이를 감지한 다른 노드가 master로 승격되면서 영구적으로 손실될 수 있다.

flush(일종의 동기화 작업)하기 전에 클라이언트에 응답하는 것은 성능이 매우 낮아지므로 `Redis Cluster`는 비동기 방식으로 복사한다.

`Redis Cluster`도 `WAIT` 명령을 지원하지만, 완전하게 일관성을 보장하진 않는다고 한다.

또, node timeout이 발생하면 master는 해당 노드에 문제가 발생했다고 간주하고 Cluster 구성을 변경 하면서 유실 될 수 있다.

### Redis Cluster configuration parameters

`Redis Cluster`를 구성할 때는 각 Redis 노드의 redis.conf 파일에서 여러 설정을 지정할 수 있다.

- cluster-enabled <yes/no>
Redis 인스턴스에서 Redis Cluster 지원을 활성화 한다.

- cluster-node-timeout <milliseconds>:

Redis 노드가 실패한 것으로 간주되기 전까지 허용되는 최대 시간 설정
  
- cluster-slave-validity-factor <factor>:
복제본이 마스터의 장애 조치를 자격 여부 설정. 0으로 설정하면 복제본은 항상 마스터의 장애 조치를 시도한다.
  
- cluster-allow-reads-when-down <yes/no>:

클러스터가 실패로 표시될 때 노드가 읽기 쿼리를 제공할 수 있는지 여부 설정. 'yes'로 설정하면, 실패 상태에서도 노드에서 읽기 작업은 허용된다.
