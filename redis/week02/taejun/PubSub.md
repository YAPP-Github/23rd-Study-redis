# [4] Redis PubSub

- 빠르고 경량화되어있다.
- Fire & Forget
    - 특정 Channel에 Publish하면 Subscribe하던 Consumer는 Data를 바로 소비할 수 있다.
    - Message의 유실 가능성이 있다.
        - Redis는 Message를 영속화 하지 않는다.
        - Subscriber가 미작동하는 상태
        - Publishing 패킷 유실
        - Redis 서버 과부화 및 Buffer Overflow
- Push 방식이다.
    - Message가 Publish된 순서대로 Subscribe한다.
    - 일반적으로 순서가 보장된다.
    - Network지연, Cluster에서의 Node들간의 동기화로 인해서 순서가 알맞지 않을 수 있다.
- Redis에 접근하는 모든 Node는 Publisher / Subscriber가 될 수 있다.
- MetaData를 갖고있지 않다.
    - 어떤 Subscriber가 읽어갔는지
    - Subscriber에게 정상적으로 전달 됐는지 (ACK)
    - Message가 언제생성되었는지
    - 어떤 Publisher가 Message를 생성했는지
    - …
- KeySpace와 별도이다
    - Key-Value 등 데이터구조와는 아무런 연관성이 없으며 서로 영향을 주지 않는다.

## [0] Cluster구조에서의 Pub/Sub

- Publish를 하게되면 Cluster에 속한 모든 Node에게 전달된다.
    - 그렇기 때문에, Subscriber는 Cluster상의 아무 Node에 연결하여 처리 할 수 있다.
    - 간단한 방법이지만, 비효율을적이다. (불필요한 리소스 사용 및 네트워크 부하)
- Redis7.0부터 shared pub/sub을 지원한다.
    - 각 Channel은 Slot에 매핑된다.
    - Key가 Slot에 할당되는 것과 동일하며, 같은 Slot간에만 pub/sub을 전달한다.

## [1] Command

### [1] PUBLISH

```
> PUBLISH [CHANNEL] [MESSAGE]
```

- 특정 Channel에 Message를 Publish한다.
- 해당 Channel을 구독하던 Subscriber의 수가 리턴된다.

### [2] SUBSCRIBE

```
> SUBSCRIBE [...CHANNELS]

1) "subscribe"
2) "event1"
3) (integer) 1
1) "subscribe"
2) "event2"
3) (integer) 2
```

- 동시에 여러개의 Channel을 subscribe 할 수 있다.
- Subscribe를 수행하여 구독자가 되면 pub-sub이외의 Command들은 수행할 수 없다.
    - SUBSCRIBE
    - SSUBSCRIBE
    - SUNSUBSCRIBE
    - PSUBSCRIBE
    - UNSUBSCRIBE
    - PING
    - RESET
      QUIT
- 3개의 리턴 값을 가진다.
    - MessageType
    - 채널이름
    - 구독중인 Channel의 수

### [3] PSUBSCRIBE

```
> PSUBSCRIBE mail-*

1)"psubscribe"
2) "mail-*"
3) (integer) 1
```

- 패턴을 통해서 여러 Channel을 subscribe 하는 것이다.