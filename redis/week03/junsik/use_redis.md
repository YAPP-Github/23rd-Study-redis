# Keyspace
- Redis key 는 binary safe 하며 어떤 binary sequence 든 key 로 사용할 수 있다. empty string 도 valid key 다.
- 길이가 너무 긴 key 는 적합하지 않다.
    - 쓸데없이 메모리만 많이 잡아먹기 때문
- 길이가 너무 짧은 key 도 적합하지 않다.
    - 너무 줄이느라 readable 하지 않을 수 있기 때문
- `object-type:id` 의 형태가 가장 적합한 best practice
- maximum key size 는 512MB
- command
    - set {key_name} {value}
    - exists
    - del
    - type
- 만료기한 설정 가능(TTL: Time to live)
    - expire {key_name} {expire_value}
    - set {key_name} {value} ex {expire_value}
    - ttl {key_name}
- 탐색
    - Scan
    - Keys -> 사용 X
        - 모든 key 를 return 할 때 까지 redis server 가 block 됨

# Client-side caching
- high performance 를 위해 redis 도 client-side caching 을 지원한다.
- DB resource 를 조회할 때 매번 DB 에 query 를 요청하지 않고, client application 메모리에 저장해두었다가 동일한 query 에 대한 결과가 저장되어 있다면 그걸 그대로 사용한다.
- client 측에서는 latency 가 줄어서 좋고, DB 는 query 요청을 덜 받아서 좋다.
- 하지만 DB 상에서는 데이터가 변경되었는데, client 측에서는 이전 데이터를 그대로 사용하는 문제가 발생한다. 그래서 redis 에서는 `Tracking` 을 제공한다.
    - 두 가지 모드(`default`, `broadcasting`)가 존재한다.
```
default mode protocol 예시

Client 1 -> Server: CLIENT TRACKING ON
Client 1 -> Server: GET foo
(The server remembers that Client 1 may have the key "foo" cached)
(Client 1 may remember the value of "foo" inside its local memory)
Client 2 -> Server: SET foo SomeOtherValue
Server -> Client 1: INVALIDATE "foo"
```
- 위 방식은 client 가 많아질수록 server memory 사용량도 늘어나게 된다. 그래서 `Invalidation Table` 을 사용한다.
    - 최대 entry 개수를 정해놓고, 새로운 key 가 들어왔을 때 공간이 충분하지 않다면 오래된 key 를 제거하는 방식이다.
- Two connections mode
    - Redis 6에서 지원되는 새로운 버전의 Redis 프로토콜 RESP3을 사용하면 동일한 연결에서 데이터 쿼리를 실행하고 무효화 메시지를 수신할 수 있다.
    - 두 개의 분리된 연결(하나는 데이터용, 다른 하나는 무효화 메시지용)을 사용하여 클라이언트측 캐싱을 구현할 수도 있다.
- 모든 key 에 대해 caching 하는 것은 부담이 될 수 있기 때문에, client 에서는 OPTION 명령어로 특정 key 만 caching 하게 할 수 있다.
- 아래 명령어는 기본적으로 읽기 쿼리에 언급된 키가 캐시되지 않는 것으로 간주하게 한다.
```
CLIENT TRACKING on REDIRECT 1234 OPTIN
```
- 그리고 client 측에서 caching 을 원한다면 아래처럼 특수 명령어를 먼저 실행해야 한다.
```
CLIENT CACHING YES
+OK
GET foo
"bar"
```
- `broadcasting` 모드는 server memory 사용량을 줄이는 데에 초점이 있다.
    - 아래 명령어를 사용하면 모든 key 를 tracking 할 수 있다.
    ```
    CLIENT TRACKING on BROADCAST
    ```
    - 또한, prefix 를 사용하여 특정 패턴을 만족하는 것들만 tracking 할 수도 있다.
- `NOLOOP` 옵션을 사용하면 무효화 메시지를 무시할 수 있다.
- server 와 connection 이 끊어졌을 때 대처 방법
    - 연결이 끊어지면 local cache 가 flush 되는지 확인한다.
    - Pub/Sub와 함께 RESP2를 사용하거나 RESP3을 사용하는 경우 둘 다 무효화 채널에 주기적으로 ping 을 보낸다. 연결이 끊어진 것으로 보이고 ping back 을 수신할 수 없는 경우 최대 시간이 지난 후 연결을 닫고 cache 를 flush 한다. 
- 어떤 것들을 caching 할지 잘 정해야 한다.
    - 변화가 많은 key 들은 지양
    - 요청이 드문 key 들은 지양
    - 잘 변하지 않지만 요청이 많다면 best

# Pipelining
- Redis Pipelining 은 개별 명령에 대한 응답을 기다리지 않고 한 번에 여러 명령을 실행하여 성능을 향상시키는 기술이다.
- Pipelining 을 사용하면 server 에서는 해당 response 들을 queue 에 저장해두기 때문에 메모리 사용에 대해 주의해야한다.

# Keyspace notifications
- 클라이언트가 어떤 방식으로든 Redis 데이터 세트에 영향을 미치는 이벤트를 수신하기 위해 Pub/Sub 채널을 구독할 수 있다.
- 모든 명령은 대상 키가 `실제로 수정된 경우에만` 이벤트를 생성한다.
