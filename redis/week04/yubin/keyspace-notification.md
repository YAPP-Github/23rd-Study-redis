## Keyspace Notifications

- key,value 변경사항 이벤트를 받고자 할때 Pub/Sub 채널을 구독할 수 있게해줌
    - 예를들어, 주어진 키에 영향을 받는 모든 명령, LPUSH 명령을 받는 모든 키, database 0(?)에서 만료되는 모든 키

### 이벤트 유형

- 2가지
    - Keyspace 채널은 이벤트의 이름을 메시지로 수신
    - Keyevent 채널은 키의 이름을 메시지로 수신
- 오직 하나의 종류의 notification만 활성화 가능

### Configuration

- 기본적으로 Keyspace 이벤트 notification은 비활성화
    - 눈에 띌 만큼은 아니지만 CPU 자원을 일부 사용하기 때문
- redis.conf의 notify-keyspace-events 옵션을 사용하거나 CONFIG SET을 통해서 활성화

    ```
    K     Keyspace events, published with __keyspace@<db>__ prefix.
    E     Keyevent events, published with __keyevent@<db>__ prefix.
    g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
    $     String commands
    l     List commands
    s     Set commands
    h     Hash commands
    z     Sorted set commands
    t     Stream commands
    x     Expired events (events generated every time a key expires)
    e     Evicted events (events generated when a key is evicted for maxmemory)
    A     Alias for g$lshztxe, so that the "AKE" string means all the events.
    ```


### Events

- 모든 명령들은 대상 키가 실제로 변경된 경우에만 이벤트를 발생시킴
    - `srem` (set remove, 집합의 멤버를 제거)이 실제로 없는 키를 제거하려고하는 경우, 실제로는 키의 값을 변경시키지 않으므로 아무런 이벤트도 발생하지 않음

### Timing of expired events

- TTL이 설정된 키들의 경우 2가지 방법으로 만료됨
    1. 특정 키에 대해 command에 의한 접근이 발생했고, 만료된 사실이 발견되었을 때
    2. 순차적으로 만료된 키를 찾는 백그라운드시스템으로
- `expired` 이벤트는 위 중 하나에 의해서 키로 접근이 발생하고, 만료된 사실이 발견되었을 때 발생
    - 그 결과, 레디스 서버가 키의 TTL이 0에 도달한 시점에 `expired` 이벤트를 발생시키는 것은 보장되지 않음
    - 따라서 키의 TTL이 0이 된 시간과 `expired` 이벤트가 발생한 시간 사이에는 상당한 딜레이가 있을 수 있음