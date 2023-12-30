## [1] BulkLoading

- 한번에 대량의 데이터를 Insert할 때 사용된다.
    - Redis프로토콜 형식의 파일을 생성하여 Server에 전달한다.
    - PipeLining보다 더 빠른 데이터 Insert속도를 보여준다.
- 단순히 Redis제공 명령어를 통해서 다량의 데이터를 Insert하는 것은 권장되지 않는다.
    - RTT가 증가한다.
    - PipeLining을 사용할 수 있지만, 대량의 데이터를 Write하려면, Write하는 동시에 Read도 가능해야한다.
- 초기 Data를 Setting하는데 특화되어 있다.

### Redis Protocol

- Redis Client가 구현해야 하는 프로토콜이다.
    - 구현이 간단하다.
    - 분석속도가 빠르다.
    - 사람이 읽을 수 있다.
- 정수, 문자열, 배열 등 다양한 데이터 유형을 직렬화할 수 있다.
- 문자열 배열을 RedisServer에게 보낸다.
    - 명령
    - 인수
- Client와 Server간의 통신에만 이용된다.
    - ClusterNode간의 통신은 별개이다.

### PipeLining과의 차이점은?

- 상호보완적인 관계라고 볼 수 있다.
- Usecase의 차이점
    - Pipelining: 일반적인 경우에 적당한 양의 데이터를 하나의 단위로 묶어서 보낼 때 유용
    - Bulkloading: 파일 형태로 대량의 데이터를 묶어서 전송
- 보내는 형태
    - Pipelining: 명령어를 연속적으로 보냄
    - Bulkloading: RedisProtocol을 구현한 형태의 파일을 전송

### netcat을 통한 전송

```bash
cat data.txt | nc localhost 6379
```

- netcat을 통해서 해당 서버에 데이터를 전송한다.
- 전송하는 데이터(파일)은 Redis프로토콜 형식에 맞추어야한다.
- 명령들이 개별적으로 실행된다.
- netcat은 RedisServer의 처리결과에 대해서 알지 못하기 떄문에 개발자가 직접 찾아야한다.

### redis-cli pipe모드

```bash
cat commands.txt | redis-cli --pipe

All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 1000000
```

- 2.6 이상버전부터 제공한다,
- 명령어를 pipelining(일괄처리) 한다.
    - 어느 명령에서 오류가 발생했는지 알기 힘들다.
    - 데이터(명령, 인수)를 읽음과 동시에, 파싱한다.
- netcat만큼 빠르다.

## [2] Distributed Lock

- Redis를 분산락매니저 (DLM)으로 많이사용한다.
- 아래의 3가지 특성을 만족해야한다.
    - MutualExclusion(상호배제): Lock은 동시에 하나의 Client만 소유 할 수 있다.
    - DeadLock-Free(교착상태X): 어떤 상태에서도 Deadlock이 발생해서는 안되며, 모든 Client는 언제든지 Lock을 Acquire하고 Release할 수 있어야한다.
    - Fault Tolerance(장애 내구성): 일부시스템에 장애가 발생해도, 노드가 정상적으로 동작해야 한다. 즉, 일부가 실패하는 경우 대다수가 정상적으로 동작하기 때문에 Lock의 Acquire과 Release에 문제가 생기면 안된다.

### FailOver에 대해서 좀더 생각해보자!

- 가장 쉬운 방법은 Instance에 Key를 생성 하는 것
    - Key에는 Expire가 있고, 장애가 생기더라도 언젠가는 삭제될 것이기 떄문이다.
    - 하지만 Replica환경에서 MutualExclusion이 무너질 수 있다.
        - Redis는 SPOF가 될 수 있다.
        - MasterNode에 장애가 생긴다면?
            - Redis의 복제는 Async하다.
                1. Replica로 Lock을 획득하기 위해서 Write한 Key가 복제되기 전 MasterNode가 다운된다.
                2. Replica는 Master로 승격된다.
                3. 승격된 Master에는 Key가 Write되지 않은 상태이기 때문에, 동시에 Lock이 획득 될 수 있다.


### 단일 Instance에서의 적합한 구현

1. Key를 통한 Lock Acquire

    ```bash
    # Lock Acquire
    SET resource_name my_random_value NX PX 30000 
    
    # Lock Release with LuaScript
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
    ```

    - 모든 요청에서 Unique한 Key를 30000ms동안 획득 할 수 있다.
    - 안전한 방식으로 LuaScript를 사용하여 Lock을 Release한다.
        - 다른Client가 생성한 Lock을 지우지 않기 위해서 중요하다.
        - value에는 random한 값이 들어있는데 그 값은 Lock을 얻은 Client밖에 모르기 때문에 일치할 때만 Lock을 Release할 수 있는 권한을 부여한다.

### Redlock

- SingleInstance가 아닌, 여러개의 Master가 있는 환경 (Cluster)에서는 불완전하다.
- Redis에서 구현할 것을 권장하는 알고리즘이다. (물론 완벽하게 안전하지는 않다)
- 진행 순서
    1. 현재시간을 ms 단위로 가져온다.
    2. 모든 Instance에 동일한 Key이름과 Random한 Value를 통해서 순차적으로 잠금을 얻으려고 시도한다.
    3. 당연하게도 2에서 여러Node에 대해서 순차적으로 잠금을 시도하기 위한 시간은 Release시간보다 적어야 한다.
    4. Lock Release시간보다, Lock을 Acquire를 하는 시간이 적고, 정족수(Quorum: N/2 + 1)이상의 Node에서  Lock을 얻었다면 전체적으로 Lock을 획득 한 것으로 판단한다.
    5. Lock획득이 실패한 경우,  모든 Instance에서의 Lock Release를 시도한다.
    6. Lock Acquire에 실패한 경우, 경쟁을 방지하기 위해서 무작위 시간 지연 후에 재시도한다.
    7. 부분적으로만 Lock을 획득(일부 Instance에서만 획득) 했을 경우, 빠르게 Lock을 해제 해야 다른 Client가 Key의 만료 이전에 빠르게 Lock을 획득 할 가능성을 높일 수 있다.
- 한계점
    1. 모든 Node가 일관된 시간을 가지고 있지 않다. (조금의 정합성이 틀어질 수 있다)
        - 거의 근접한 시간에 갱신된다는 것을 가정한다.
    2. Application의 중단문제
        - GC 등 별도의 사유로 인해서 Application이 로직 실행도중 중단 될 수 있다.
        - Lock을 획득 한 시점에서의 중지 후 재 실행은 MutalExclusion을 보장 하지 못하게 될 수 있다.

- 보완점
    - FencingToken의 구현
        - FencingToken은 각 Lock의 순서를 식별 할 수 있는 고유한 식별자로 동작한다.
            - 잠깐의 중단이 있고 재개할 때 이미 Lock은 Release되어 있을 수 있다. 이때 FencingToken의 최신성 여부 판단으로 인해서 Lock유효성 판단이 가능하다.
        - 이미 Lock을 획득한 자원이 다시 Lock을 획득하려는 중복작업을 방지한다.
    - 시간을 변경해서는 안된다.
        - 수동적인 시간 변경은 MutualExclusion을 깰 수 있기 때문이다.