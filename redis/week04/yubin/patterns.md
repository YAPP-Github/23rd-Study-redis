# Bulk loading

- 일반적인 레디스 클라이언트로 bulk loading을 하는건 안좋음
    - 매 명령마다 소요되는 RTT … 등
- 선호되는 방식은 아래와 같은 ‘text file’을 생성하는것

    ```
    SET Key0 Value0
    SET Key1 Value1
    ...
    SET KeyN ValueN
    ```

- 이 파일을 보낼때 과거에는 netcat(nc)을 사용했지만, netcat은 언제 데이터가 전송되었는지 알지 못하고, 에러를 체크하지 못함
- 레디스는 2.6버전부터 bulk loading을 위해 설계된 `pipe mode` 를 지원함
    - 입력 : `cat data.txt | redis-cli --pipe`
    - 출력 :

        ```
        All data transferred. Waiting for the last reply...
        Last reply received from server.
        errors: 0, replies: 1000000
        ```


### Redis protocol

- Redis 클라이언트는 **RESP**(REdis Serialization Protocol) 프로토콜을 이용하여 Redis 서버와 통신
- RESP 데이터 타입
    - **단순 문자열** *Simple Strings*: 응답의 첫 바이트가 +
    - **에러** *Errors*: 응답의 첫 바이트가 -
    - **정수** *Integers*: 응답의 첫 바이트가 :
    - **벌크 문자열** *Bulk Strings*: 응답의 첫 바이트가 $
    - **배열** *Arrays*: 응답의 첫 바이트가 *
- RESP내에서 서로 다른 부분을 구분할 때에는 마지막을 항상 `\r\n` (CRLF)로 끝내야 함

# Distributed Locks

### Safety and Liveness Guarantees

- 안전 속성
    - 상호 배제. 어떤 순간에도 하나의 클라이언트만 락을 잡고있을 수 있음
- 명 보장A..?(Liveness Guarantee)
    - 교착상태가 없음
    - 자원을 락으로 잡고있던 클라이언트가 충돌하거나 파티셔닝된 경우에도 항상 락을 잡는것이 가능함.
- Liveness Guarantee B
    - 결함(장애) 허용
    - 대부분의 레디스 노드가 가동되고있는한, 클라이언트는 락을 잡고 풀수 있음

### 장애 극복(Failover)의 구현이 충분하지 않은 이유

- 레디스로 자원에 락을 잡는 가장 단순한 방법은 사전에 TTL을 가진 키를 만들어놓는것
    - 아래와 같은 방식으로 리소스에 값을 설정하여 값이 설정된 경우 다른 리소스의 접근을 차단하도록 구현 가능

        ```
        // key가 존재하지 않는다면, value를 저장하고, 30000ms 동안 유지
        SET key value NX PX 30000
        ```

    - 리소스에 대한 잠금 해제의 경우, 아래와 같이 1. 키가 존재하며 2. 키에 저장된 값이 클라이언트의 값과 일치하는 경우에만 해제할 수 있어야함.

        ```
        if redis.call("get",KEYS[1]) == ARGV[1] then
            return redis.call("del",KEYS[1])
        else
            return 0
        end
        ```

    - 이 방식으로 진행하면 결국엔 언젠가 락은 풀어짐
    - 그러나 위 방식은 표면적으로는 정상적으로 작동하지만, SPOF가 될 수 있으며, SPOF를 해결하기위해 Replication을 두는 순간 다음과 같은 문제가 생길 수 있음
        1. 클라이언트A가 마스터에서 락을 잡음
        2. 마스터는 키에 대한 쓰기가 replica로 전송되기 전에 충돌
        3. replica가 마스터로 승격
        4. 클라이언트B는 A가 잠그고있던 동일한 리소스의 락을 획득 → 문제
    - 다수의 클라이언트가 락을 잡고있어도 된다면 위 방법을 써도되지만, 아니라면 다른 방법을 사용하자

### Redlock 알고리즘

- 레드락은 N개의 단일 레디스 노드들을 이용하여, Quorum 이상의 노드에서 잠금을 획득하면 분산락을 획득한 것으로 판단
- 클라이언트는 분산 환경에서 락을 획득하기 위해 다음 작업을 수행
    1. 현재 시간을 ms 단위로 가져옴
    2. 모든 인스턴스에서 순차적으로 잠금을 획득하려고 시도. 각 인스턴스에 잠금을 설정할 때 클라이언트는 전체 잠금 자동 해제 시간에 비해 작은 타임아웃을 사용하여 잠금을 획득. 예를 들어 자동 해제 시간이 10s인 경우, 타임아웃은 5~50ms가 될 수 있음. 이를 통해 클라이언트가 다운된 Redis 노드와 통신하려고 오랫동안 블로킹되는 것을 방지할 수 있음.
    3. 클라이언트는 (현재 시간 - 1단계에서 얻은 타임스탬프)를 통해 잠금을 획득하기 위해 경과한 시간을 계산. 클라이언트가 과반이 넘는(N/2 + 1) 인스턴스에서 잠금을 획득했고, 총 경과 시간이 잠금 유효 시간보다 적다면 분산락을 획득한 것으로 간주.
    4. 분산락을 획득한 경우, 잠금 유효 시간은 3단계에서 계산한 시간으로 간주
    5. 분산락을 획득하지 못한 경우(과반이 넘는 인스턴스를 잠글 수 없거나 유효 시간이 음수인 경우), 클라이언트는 모든 인스턴스에서 잠금을 해제하려고 시도
    6. random interval 이후에 재시도

- 요약
    - N대의 싱글 노드 레디스가 존재
    - 클라이언트에서 잠금을 위해 N대의 노드에 동시 요청(setnx)를 보냄
    - N대 중 N/2 + 1의 잠금을 획득했다면, 분산락 획득에 성공하게 됨
    - 만약 실패했다면, 모든 N대의 노드에 해제 요청(delete)을 보냄
    - random interval 이후에 재시도함

### 레드락 알고리즘의 한계

- Clock Drift
    - 레드락 알고리즘은 노드들간에 로컬시간이 거의 동일한 속도로 갱신된다는 가정에 의존
    - 하지만 현실에서는 클락이 정확한 속도로 동작하지 않는 Clock Drift 혀상으로 인해 레드락알고리즘에 문제가 있을 수 있음
    - 예
        1. 클라이언트 1이 노드 A, B, C에서 잠금을 획득하지만, 네트워크 문제로 인해 D와 E에서는 잠금 획득에 실패
        2. 이때 노드 C의 시계가 앞으로 이동하여 잠금이 만료
        3. 클라이언트 2가 노드 C, D, E에서 잠금을 획득하지만, 네트워크 문제로 인해 A와 B에서는 잠금 획득에 실패
        4. 이제 클라이언트 1과 2는 모두 자신이 잠금을 획득했다고 생각 → 문제
- 애플리케이션 중지시 문제 시나리오
    1. 클라이언트 1이 분산락을 획득
    2. 이때 클라이언트1에서 애플리케이션 중지가 발생하고, 그 사이에 분산락이 만료
    3. 클라이언트 2는 분산락을 획득하고 파일을 갱신
    4. 클라이언트 1의 애플리케이션이 복구되고 파일을 갱신 → 동시성 문제 발생

# 보조 인덱스

- 레디스는 정확한 key-value store는 아님
    - 복잡한 값도 저장하기 때문
- 그러나 외부 key-value shell을 가짐
    - API level에서 데이터는 key name으로 저장됨
    - pk로만 접근 가능
- 아래와 같은 데이터구조로 인덱스를 만들수있음
    - ID나 숫자 필드로 보조 인덱스를 만드는 Sorted sets
    - 더욱 향상된 보조 인덱스(composite index, graph traversal index)를 만들기 위한 Sorted sets
    - 무작위 인덱스를 만드는 Sets
    - iterable한 인덱스, last N items 인덱스 등을 생성하는 Lists

### Simple numerical indexes with sorted sets

- 가장 쉬운 인덱스 생성 방법은 각 element가 부동소수점 숫자로 정렬된 sorted sets를 사용하는것
- ZADD, ZRANGE 명령어 사용

    ```
    ZADD myindex 25 Manuel
    ZADD myindex 18 Anna
    ZADD myindex 35 Jon
    ZADD myindex 67 Helen
    ```

    ```
    // 나이가 20에서 40사이인 사람
    ZRANGE myindex 20 40 BYSCORE
    1) "Manuel"
    2) "Jon"
    ```


### Using objects IDs as associated values

- 다른곳에 저장된 필드를 인덱싱하고싶을때
- Sorted sets에 ID만 저장

    ```
    HMSET user:1 id 1 username antirez ctime 1444809424 age 38
    HMSET user:2 id 2 username maria ctime 1444808132 age 42
    HMSET user:3 id 3 username jballard ctime 1443246218 age 33
    ```

    ```
    ZADD user.age.index 38 1
    ZADD user.age.index 42 2
    ZADD user.age.index 33 3
    ```


### Lexicographical indexes

- b-tree 구조와 유사
- 사용 1: Completion
    - 검색엔진에서 자동완성되는 기능
- 사용 2: frequency 추가
    - `ZADD myindex 0 banana:1`
        - 1 : frequency
    - 자주 사용되는 (frequency가 높은)게 가중치가 높음

        ```
        ZRANGE myindex "[banana:" + BYLEX LIMIT 0 10
        1) "banana:123"
        2) "banaooo:1"
        3) "banned user:49"
        4) "banning:89"
        ```

- 사용 3: Numerical padding
    - Lexicographical indexes는 문자열을 인덱스 하는 경우에만 좋아보일 수 있지만, 실제로 임의 정밀도의 숫자를 인덱스 할 때에도 괜찮음
    - 숫자 앞뒤에 적절한 길이의 0 padding을 추가하는 방법

        ```
        01000000000000.11000000000000
        01000000000000.02200000000000
        00000002121241.34893482930000
        00999999999999.00000000000000
        ```