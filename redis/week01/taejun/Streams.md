# [7] Streams
- appeny-only-log방식
    - 수정이나 삭제가 되지않고, 계속해서 쌓이는방식
    - Log식으로 계속 쌓이기 떄문에 높은 내구성, 데이터 무결성 및 이력에 대한 추적이 용이하다.
- Fire & Forget이 아니다.
    - 모든 Consumer에게 Data를 전달하는 것이 목적이아니다.
        - ConsumerGroup이라는 개념을 제공한다.
    - 분산처리를 목적으로 한다.
- Kafka의 Topic == Stream 으로 볼 수 있다.

### PubSub과의 비교
- 영속성
    - PubSub - fire & forget
    - Stream - append-only
- Listen
    - PubSub  - Subscriber
        - Push기반
            - 각각의 Topic에 Publisher가 직접 넣는다.
        - subscriber는 event혹은 callback을 통해서 새로운 Message를 인식한다.
    - Consumer
        - Polling 기반
            - Publisher는 중앙화된 Stream에 Message를 보내고, Consumer가 Stream에서 Polling해간다.
        - Consumer가 주기적으로 Polling한다.
- ConsumerGroup
    - PubSub: 없음 → 모든 Subscriber가 읽음
    - Stream: 있음 → 같은 ConsumerGroup내에서는 같은 Message를 Consume하지 않음
- MetaData
    - pubsub: 알수 없다
        - publisher는 subscriber가 읽어갔는지 알 수 없다.
        - subscriber는 Message가언제 만들어졌는지, 어떤 Publisher가 만들었는지 알 수 없다.
    - Streams

## Kafka와의 비교
- 속도
    - Redis - InMemory
    - Kafka - Disk
- 확장성
    - Redis - 수직확장에 용이
    - Kafka - 수평확장에 용이
- 사용사례
    - Redis - 실시간분석, IOT
    - Kafka - 로그 처리, 대용량 데이터 실시간 처리
- Topic(혹은 Stream) 생성
    - Kafka: 사전에 필요함
    - Redis: XADD 시점에 생성 가능
- Message의 순서
    - Kafka: Partition에서만 보장
    - Reids: Global하게 보장

## 주요 Command

### [1] XADD
```bash
XADD [STREAM-KEY] [MESSAGE- -ID] [FIELD][VALUE] ... {OPTIONS}
```
- O(logN)
- Stream에 데이터를 추가한다.
- Default STREAM-ID는  **<millisecondsTime>-<sequenceNumber> 로** 구성된다.
    - * 를 대신 넣으면 Default MESSAGE-ID가 생성된다.
    - millisecondsTime은 timeStamp숫자이다.
        - 저장되는 시점의 RedisNode의 LocalTime이다.
    - sequenceNumber는 8Byte 정수이다.
        - 중복을 방지하기 위한 정책으로, miliSecondTime까지 같다면, sequence는 1씩증가한다.
- STREAM-ID는 이전보다 작은 값을 설정 할 수 없다.
- OPTIONS
    - NOMKSTRAM
        - 새로운 Key를 만들지 않는다.
        - 이미 존재하는 Key에만 추가한다.
    - MAXLEN
        - Stream의 최대 갯수를 제한하고 싶을 때 정의한다.
        - Add를 할 때, 이미 maxLen에 정의된 수치이거나 그 이상일 때는 가장 오래된 메세지를 삭제한다.
            - Stream의 갯수가 현재 maxLen보다 많다면, 우선 오래된 순서대로 maxLen -1 개 까지 삭제하고 새로운 메세지를 추가한다.
        - add와 delete가 한 Operation에서 실행되기 떄문에, 오래걸릴 수도 있다.
    - MINID
        - Stream에서 유지되어야할 Id의 최소를 지정한다.
        - minId보다 낮은 메세지들은 Stream에서 제거된다.

### [2] XREAD
```bash
XREAD {COUNT count} {BLOCK milliSec} STREAMS [...STREAM-KEY] [...MESSAGE-ID] 
```
- 일종의 Subscribe
    - 모든 Client에게 전달된다.
- Count를 지정하지 않으면 모두 가져온다.
- StreamId보다 큰 id들로 가져온다.
    - STREAM-ID를 0으로 지정하면 맨앞부터 가져온다.
- 메세지의 읽음상태 (ACK) 를 추적하지 않는다.
    - 단순 읽기연산을 목표로 하기 때문이다.
- BLOCK 옵션을 0으로주면, 새로운 데이터가 올 때 까지 계속 Blocking한다.

### [3] XREADGROUP

```bash
XREADGROUP [GROUP GROUP-NAME] [CONSUMER]{COUNT count} {BLOCK milliSec} STREAMS [...STREAM-KEY] [...MESSAGE-ID} {>]
```
- O(M): M은 반환된 요소의 수
- XREAD에 ConsumerGroup 개념이 더해진 것이다.
- **ConsumerGroup내에서 각 Consumer는 고유하게 식별 가능해야 한다.**
    - XACK명령을 통해서, 어떤 Consumer가 처리했는지 알아야 하기 때문이다.
    - 분산처리가 필요하지 않은경우 사용하지 않아도 된다. (모든 Client가 수신해야 하는경우)
    - 어디까지 읽었는지 lastId도 ConsumerGroup단위로 관리된다.
    - pendingList를 읽고 읽은 후 ACK처리가 되면 lastId를 업데이트 한다.
- > 옵션은 다른 Consumer가 읽지않은 Message를 가져온다. (거의 99% 사용)
- Group을 명시적으로 생성해주어야 한다.
    ```bash
    XGROUP CREATE [MESSAGE-ID] [GROUP-NAME] {$}
    ```

    - $ 옵션을 주면 생성 이후시점의 Stream부터 읽는다.
    - 생략하면 맨처음부터 읽는다.

### [4] XRANGE
```shell
XRANGE [KEY] [START] [END] [COUNT count]
```
- 범위에서 가장적은걸 입력하고싶으면 -, 많은걸 입력하고싶으면 + 를 사용한다.
    - XRANGE sampe - +
- XREVRANGE도 존재한다.

### [5] XACK
```shell
XACK [KEY] [GROUP_NAME] [MESSAGE-ID] 
```
- 내부적으로 pendingList를 가지고 있다.
    - pendingList는 ACK를 아직 받지못한 Message들이 저장된 곳이다.

### [6] XPENDING
```shell
> XPENDING [KEY] [GROUP_NAME] {}

(1) (integer) 9.       # Pending Message 갯수
(2) 1659114481311-0    # Pending Message 최솟값
(3) 1659170735630-0    # Pending Message 최댓값
(4) ...                # 각 소비자별 Pending중인 Message
```
- ConsumerGroup단위로 관리되는 pendingList에서 아직 ACK처리가 안된 Messae가 있는지 확인한다.
- Consumer별로 Pending이 구분된다.
    - Consumer가 할당되는 시점은 Runtime (XREAD, XREADGROUP) 이다.
    - 읽어간 후, 아직 ACK를 받지못한 Message가 대상이다.

### [7] XCLAIM
```shell
XCLAIM [KEY] [GROUP] [CONSUMER] [min-idle-time] [...message-id]
```

- PendingList에 있는 Message의 소유권을 다른 Consumer에게 넘긴다.
    - 같은 ConsumerGroup내에서 가능
- Consumer가 장애상황에서 복구가 되지 않을 때, 메세지를 재할당 하는 방법이다.
- min-idletime보다 pendingList에 있었던 시간이 길어야지 Message가 재할당된다.

### [8] XLEN
```shell
XLEN [KEY]
```
- Stream의 길이를 구한다.
- ID의 갯수를 구하는 것이다.

### [9] XINFO
- Stream에 대한 정보들을 얻을 수 있다.
- consumer, group등에 대한 정보도 확인 가능하다.