# Keyspace
Redis에서 키 관리하기(만료기간 설정, 스캐닝, 변경 및 쿼리)

잘 알려진대로, Redis의 키는 binary 시퀀스라면 모두 가능하다. `"foo"`같은 단순 문자열 뿐만 아니라 JPEG파일도 가능. `""` 같이 빈 문자열도 된다

Key를 지정할 때 고려하면 좋은 점
- 너무 긴 key는 메모리 측면에서 비용소모가 많고 조회할 때도 비교작업에 많은 비용이 들어서 권장하지 않는다.
- 반대로 너무 짧은 키의 경우 추가되는 공간과 가독성을 비교하여 적절하게 사용하면 좋다. ex) `"user:1000:follower"` vs `"u1000flw"`
- Schema를 사용하여 표현하기. `"object-type:id"` 
필드에 여러 단어가 포함되면 `.`이나 `-`를 활용하자ex) "comment:4321:reply-to", "comment
- Key의 최대 크기는 512MB이다.

## Altering and querying the key space
특정 타입과 관련없이 지원하는 명령들이 있다.
`EXISTS` 는 key가 존재하는지의 여부를 타입과 관련없이 확인 할 수 있다.
`DEL`은 타입과 관련없이 key와 value를 지운다.
`TYPE`은 key의 타입을 조회 할 수 있다.
```java
@Test
void KEY_타입과_관련_없이_존재_여부를_조회_할_수_있다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.hset("key", "field", "value");
    syncConnection.set("key2", "value");

    // when
    var existA = syncConnection.exists("key");
    var existB = syncConnection.exists("key2");
    var existC = syncConnection.exists("key3");

    // then
    assertSoftly(assertions -> {
        assertions.assertThat(existA).isEqualTo(1L);
        assertions.assertThat(existB).isEqualTo(1L);
        assertions.assertThat(existC).isEqualTo(0L);
    });
}

@Test
void KEY_타입과_관련_없이_KEY를_지울_수_있다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.hset("key", "field", "value");
    syncConnection.set("key2", "value");

    // when
    var existA = syncConnection.del("key");
    var existB = syncConnection.del("key3");

    // then
    var exist = syncConnection.exists("key");
    assertSoftly(assertions -> {
        assertions.assertThat(exist).isEqualTo(0L);
        assertions.assertThat(existA).isEqualTo(1L);
        assertions.assertThat(existB).isEqualTo(0L);
    });
}

@Test
void KEY_타입을_조회_할_수_있다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.hset("hashKey", "field", "value");
    syncConnection.pfadd("hyperloglogKey", "value");
    syncConnection.sadd("setKey", "value");
    syncConnection.zadd("sortedSetKey", 1, "value");

    // when
    var existA = syncConnection.type("hashKey");
    var existB = syncConnection.type("hyperloglogKey");
    var existC = syncConnection.type("setKey");
    var existD = syncConnection.type("sortedSetKey");

    // then
    assertSoftly(assertions -> {
        assertions.assertThat(existA).isEqualTo("hash");
        assertions.assertThat(existB).isEqualTo("string");
        assertions.assertThat(existC).isEqualTo("set");
        assertions.assertThat(existD).isEqualTo("zset");
    });
}
```

## Key expiration
키를 저장할 때, 만료 시간(TTL)을 지정할 수 있다. 해당 만료 시간이 지나면 자동으로 제거된다.
```shell
> set key some-value
OK
> expire key 5
(integer) 1
> get key (immediately)
"some-value"
> get key (after some time)

> set key 100 ex 10
OK
> ttl key
(integer) 9

(nil)
```
> persist로 ttl을 제거할 수 있다.
![](https://velog.velcdn.com/images/roycewon/post/18cff5de-4a88-4172-b463-b2a745067dcc/image.png)

계속 진행하기 전에, 저장하는 값의 유형에 관계없이 작동하는 중요한 Redis 기능인 키 만료에 대해 살펴보겠습니다. 키 만료를 사용하면 키에 대한 시간 제한을 설정할 수 있는데, 이를 "유효 기간" 또는 "TTL"이라고도 합니다. 이 시간이 경과하면 키는 자동으로 파기됩니다.

## Navigating the keyspace
### Scan
효율적으로 Redis의 키를 찾을때 사용.
count 옵션의 수 만큼 cursor기반으로 분산하여 검색하여 single-thread인 redis에서 많은 key를 조회 할 때 긴 블록킹을 하지 않아 권장된다.

### Keys
Redis 커넥션을 점유하며 모든 key를 순회하여 조회 한다.
운영 환경에서는 사용하지 말라고 권장한다.

```java
@Test
void 해쉬_전체_조회_성능_평가() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    Map<String, String> hashes = new LinkedHashMap<>();
    for (int i = 0; i < 100000; i++) {
        hashes.put("key" + i, "value" + i);
    }
    syncConnection.mset(hashes);

    // when
    Long hgetLatency = timeRater(() -> syncConnection.keys("*"));
    Long hsacnLatency = timeRater(() -> syncConnection.hscan("", Builder.matches("*")));

    System.out.println("hgetall latency: " + hgetLatency);
    System.out.println("hscan latency: " + hsacnLatency);
    // keys latency: 58
    // scan latency: 2
}

private Long timeRater(Supplier<?> supplier) {
    Instant now = Instant.now();
    supplier.get();
    Instant after = Instant.now();
    return Duration.between(now, after).toMillis();
}
```

# Client-side caching

클라이언트 사이드 캐싱은 고성능 서비스를 위해 사용되는 기술이다.
데이터의 일부를 클라이언트(어플리케이션)에 캐싱하여 클라이언트 서버의 메모리를 이용한다.

DB에 직접 조회/쿼리 하여 데이터에 요청하는 것과 달리, 어플리케이션 내에서 데이터를 직접 사용한다. 네트워크 비용이 획기적으로 줄어 들기 때문에 성능 뿐만 아니라 DB의 부하도 줄여준다.
자주 변하지 않는 데이터를 대상으로는 꽤나 효과적이다. 하지만 오래되거나 정합하지 않는 데이터가 제공되는 오류를 피하기 위해 어플리케이션에서의 invalidate(무효화)과정이 필요하다.
TTL을 사용하거나 Pub/Sub 패턴을 통해 실현 할 수 있다.

Redis에서 구현해둔 client-side caching은 `Tracking`이라 부르며 두가지 모드가 제공된다.
- default mode: 클라이언트가 액세스한 키를 기억해 두었다가, 해당 키가 수정되면 invalidate 메세지를 전송한다. 이 모드는 server에서 메모리 비용이 들지만 정확하게 클라이언트가 가지고 있는 키에 대해서만 무효 메세지를 보낼 수 있다.
- broadcast mode: `object:`, `user:` 와 같은 prefix를 구독해 두고, 해당 키가 변경될 때마다 메시지를 받는다.

우선, default mode에 대해 살펴보자.
클라이언트가 트래킹을 시작한다.
트래킹이 활성화 되면, 서버에서는 요청한 키를 기억한다.
해당 키가 수정되면 트래킹이 활성화된 클라이언트에 무효 메세지를 전달한다.
클라이언트가 해당 메세지를 수신하면 캐싱하는 키 값을 삭제한다.

```
Client 1 -> Server: CLIENT TRACKING ON
Client 1 -> Server: GET foo
(The server remembers that Client 1 may have the key "foo" cached)
(Client 1 may remember the value of "foo" inside its local memory)
Client 2 -> Server: SET foo SomeOtherValue
Server -> Client 1: INVALIDATE "foo"
```
이 과정은 클라이언트가 많아질 수록, 서버의 메모리 사용량이 크게 증가 할 수 있고, 이러한 문제점에 대해 Redis는 invalidation table을 사용한다. 트래킹을 활성화한 클라이언트 목록을 저장하는 글로벌 테이블인 `Invalidation Table`을 구성하고, 새로운 키가 추가되면 오래된 엔트리를 제거하여 무효화 메세지를 발송하여 서버의 메모리 사용량을 줄이는 방법이다.

## Two conncetions mode

Redis 6에서 지원하는 RESP3 프로토콜을 사용하면 하나의 커넥션으로 쿼리 실행과 무효화 메세지 수신이 가능하지만, 데이터 접근용 커넥션과 invalidation 커넥션을 따로 사용하여 캐싱을 구현할 수도 있다. 클라이언트가 트래킹을 활성화 할 때, 클라이언트ID를 지정하여 무효화 메세지를 다른 커넥션으로 리다이렉트 할 수 있다.
```shell
# (Connection 1 -- used for invalidations)
> CLIENT ID
:4
> SUBSCRIBE __redis__:invalidate
*3
$9
subscribe
$20
__redis__:invalidate
:1

# (Connection 2 -- data connection)
> CLIENT TRACKING on REDIRECT 4
+OK
> GET foo
$3
bar
```
Connection 2에서 Client ID가 4인 connection으로 `TRACKING on REDIRECT`를 지정했다.
이때 다른 커넥션에서 `foo`의 값을 변경하면 Conncetion 1이 정상적으로 무효화 메세지를 수신한다.

## Opt-in caching
트래킹을 활성화한 클라이언트는 모든 key가 아닌 선택적으로 캐시할 수 있다. `OPTIN`으로 사용 가능
`CLIENT TRACKING on REDIRECT 1234 OPTIN`


## Broadcasting mode
default mode와 달리, 서버의 메모리 사용량과 부하를 최대한 줄이는 데 초점이 맞춰진 모드.
클라이언트에서는 `BCAST` 옵션을 통해 broadcasting mode 캐싱을 활성화 할 수 있다.
`CLIENT TRACKING on REDIRECT 10 BCAST PREFIX object: PREFIX user:` 처럼, 구독하고자 하는 키의 prefix를 지정해야 한다. 지정하지 않으면 모든 키에 할당 된다.
이 모드에선 invalidation table이 아닌 prefixes table에 prefix를 지정한 클라이언트 목록을 저장하고, 해당 prefix에 해당하는 키의 값이 수정되면 table에 할당되어있는 클라이언트에 invalidation 메세지를 전송한다. prefix의 수 만큼 서버도 메모리를 소비한다.

## NOLOOP option
트래킹을 지정해두고 무효화 메세지를 받지 않도록 설정 할 수 있다.

# Redis Pipelining
Redis의 command를 일괄 처리 하여 RTT를 최적화 하는 방법

Redis는 TCP기반으로 통신하는 모델이기 때문에 간단한 로직을 처리하는 경우에도 네트워크 연결 시간이 소모된다.
만약 클라이언트가 연속적으로 많은 요청을 수행해야 하는 경우, 처리시간 뿐만 아니라 매 요청마다 네트워크 연결 시간이 소모되기 때문에 여러 명령을 수행하는 경우 개선이 필요하다.

Redis는 서버는 클라이언트가 요청의 응답을 수신하지 않고 계속 요청을 전송하여 서버에서 모든 명령을 처리한 뒤 응답을 한번에 수신할 수 있는 `파이프라이닝`을 지원한다.
만약 파이프라이닝을 통해 명령을 전송한다면, 서버에서는 해당 명령들을 처리하고 각각의 응답을 대기열에 넣기 때문에 많은 명령에 대해선 메모리를 사용하니 주의하여야 한다.

```shell
$ (printf "PING\r\nPING\r\nPING\r\n"; sleep 1) | nc localhost 6379
+PONG
+PONG
+PONG
```

파이프라이닝은 RTT latency 감소만아 아니라 Redis 서버에서 처리하는 작업의 수를 크게 늘린다.
파이프라이닝을 사용하면 일반적으로 한 번의 `read()` 시스템 호출을 통해 모든 명령을 읽고, 한 번의 `write()` 시스템 호출로 여러 개의 응답을 구성한다. 이러한 이유로 초당 수행되는 쿼리 수는 처음에는 파이프라인이 길어질수록 거의 선형적으로 증가하지만, 아래 그림과 같이 특정한 처리량으로 귀결된다.

>
![](https://redis.io/docs/manual/pipelining/pipeline_iops.png)https://redis.io/docs/manual/pipelining/

[Redis scripting](https://redis.io/commands/eval)을 통해 파이프라이닝처럼 여러 명령을 실행 할 수 있다.

# Redis keyspace notification
Keyspace 알림을 통해 key와 value의 변경 사항을 실시간으로 모니터링 할 수 있다.
클라이언트는 Pub/Sub을 통해 Redis 데이터에 영향을 미치는 이벤트를 수신 할 수 있다.
키를 수정하거나 추가 하는등의 데이터 연산 뿐만 아니라 만료되는 키들도 모니터링 가능하다.

Keyspcae 알림은 두 가지 유형의 이벤트를 전송하는 방식으로 구현된다.
예를 들어 데이터베이스 0의 "mykey"라는 키를 대상으로 하는 `DEL` 작업을 모니터링 하기 위해선 다음과 같은 두개의 메세지가 전달되어야 한다.
```shell
PUBLISH __keyspace@0__:mykey del
PUBLISH __keyevent@0__:del mykey
```

`__keyspace@0__` 채널은 "mykey"를 대상으로 하는 모든 이벤트를 수신한다. (Key-space notification)
`__keyevent@0__` 채널은 "mykey"에 대한 `DEL` 작업만 수신한다. (Key-event notification)

`Key-space` 채널은 이벤트의 이름을 메시지로 받는다. (`DEL`
`Key-event` 채널은 키의 이름을 메시지로 받습니다. "mykey"

![](https://velog.velcdn.com/images/roycewon/post/74bd96ac-b478-4975-9aa7-d7904c3b5693/image.jpg)

## Configuration
Keyspace notification은 CPU를 자원을 소모하기 때문에 기본적으로 비활성화되어 있다. `redis.conf`의 `notify-keyspace-events`를 사용하거나 `CONFIG SET`을 통해 알림을 활성화할 수 있다.

`CONFIG SET`에 옵션을 주지 않으면 비활성화 된다. 기능을 활성화하려면 `K` 혹은 `E` 옵션이 필수적으로 포함되어야 하고 여러 옵션도 추가하여 설정 할 수 있다.
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
d     Module key type events![](https://velog.velcdn.com/images/roycewon/post/c1327a35-7735-4587-8f46-4da3fb02ada0/image.png)

x     Expired events (events generated every time a key expires)
e     Evicted events (events generated when a key is evicted for maxmemory)
m     Key miss events (events generated when a key that doesn't exist is accessed)
n     New key events (Note: not included in the 'A' class)
A     Alias for "g$lshztxed", so that the "AKE" string means all the events except "m" and "n".
```

알림이 활성화 되면, 특정 명령마다 이벤트가 발생하는데 docs에 자세히, 많이 나와있다. 모든 명령 기반 이벤트는 실제로 키가 변경된 작업에 대해서만 발행된다.



## Timing of expired events
TTL이 지정된 key에 대해 만료 이벤트 발행은 다음과 같은 상황에서 발생한다.
- 명령을 통해 만료된 키에 접근 하는 경우 (ex `get`)
- 백그라운드에서 TTL이 만료된 키를 수집하는 경우

이러한 이유로, TTL이 연결된 키가 많은 경우에는 실제TTL이 만료 되었지만 아직 백그라운드에서 수집하지 못하는 경우 이벤트가 지연되서 발생 할 수 있다.


