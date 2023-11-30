# String
- 문자열은 텍스트, 직렬화된 개체, 바이너리 배열을 포함한 바이트 시퀀스(jpeg 같은 파일 포함) 등 모든 binary safe한 타입을 저장 한다. 
- 문자열은 **캐싱** 뿐만 아니라 **카운터**를 구현하고 비트 단위 연산을 수행할 수 있는 기능도 지원. 최대 크기는 512MB이다.
- GET과 SET을 통해 문자열 값을 저장, 조회 한다.
이때, 이미 있는 키를 SET 하면 대체 된다. 추가로, 다른 인자나 명령들을 활용하여 중복된 키에 대한 정책을 정할 수 있다.

증감 연산에 대해서는 Atomic 연산을 보장한다고 한다.

문자열 조작 test code
`(with lettuce)`
```java
@Test
void 간단한_저장_및_조회() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.set("key", "value");

    // when
    String value = syncConnection.get("key");

    // then
    assertThat(value).isEqualTo("value");
}

@Test
void 키가_이미_존재_하는_경우_값은_대체_된다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.set("key", "value");
    syncConnection.set("key", "newValue");

    // when
    String value = syncConnection.get("key");

    // then
    assertThat(value).isEqualTo("newValue");
}

@Test
void SETNX는_키가_이미_존재_하면_저장_하지_않는다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.set("key", "value");

    // when
    // deprecated
    // syncConnection.setnx("key", "newValue");
    syncConnection.set("key", "newValue", SetArgs.Builder.nx());

    // then
    String value = syncConnection.get("key");
    assertThat(value).isEqualTo("value");
}

@Test
void SETXX는_키가_이미_존재_하는_경우에만_저장_한다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();

    // when
    syncConnection.set("key", "value", SetArgs.Builder.xx());

    // then
    String value = syncConnection.get("key");
    assertThat(value).isNull();
}

@Test
void 단일_명령으로_여러_데이터를_저장_조회_한다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();

    // when
    syncConnection.mset(Map.of(
            "key", "value",
            "key2", "value2",
            "key3", "value3"
    ));

    // then
    var mget = syncConnection.mget("key", "key2", "key3");
    assertThat(mget).hasSize(3);
    List<String> values = mget.stream().map(KeyValue::getValue).toList();
    assertThat(values).containsExactly("value", "value2", "value3");
}

@Test
void MSET시_key가_중복_되면_예외가_발생_한다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();

    // when
    assertThatThrownBy(() -> syncConnection.mset(Map.of(
            "key", "value",
            "key", "value2"
    ))).isInstanceOf(IllegalArgumentException.class);
}

@Test
void 숫자로_저장된_값에_대하여_증감_연산이_가능_하다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.set("user:royce:score", "80");

    // when & then
    syncConnection.incr("user:royce:score");
    assertThat(syncConnection.get("user:royce:score")).isEqualTo("81");

    syncConnection.incrby("user:royce:score", 20);
    assertThat(syncConnection.get("user:royce:score")).isEqualTo("101");

    syncConnection.decr("user:royce:score");
    assertThat(syncConnection.get("user:royce:score")).isEqualTo("100");

    syncConnection.decrby("user:royce:score", 20);
    assertThat(syncConnection.get("user:royce:score")).isEqualTo("80");
    
    syncConnection.incrbyfloat("user:royce:score", 0.5);
    assertThat(syncConnection.get("user:royce:score")).isEqualTo("80.5");
}
```
[더 다양한 String 타입 연산](https://redis.io/commands/?group=string)

증감 연산의 경우, 원자성 연산을 보장하여 대부분의 상황에서 `thread`간 경합이 발생하지 않는다고 한다.
```java
@Nested
class AtomicIncrementTest {

    static class RedisIncr implements Callable<Void> {

        @Override
        public Void call() {
            for (int i = 0; i < 10000; i++) {
                RedisCommands<String, String> syncConnection = RedisConnectionProvider.getSync();
                syncConnection.incr("test");
            }

            return null;
        }
    }
    
    @Test
    void 증감_연산은_원자적_연산을_보장_한다() throws InterruptedException {
        // given
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        List<RedisIncr> runners = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            runners.add(new RedisIncr());
        }

        // when
        executorService.invokeAll(runners);

        // then
        var syncConnection = RedisConnectionProvider.getSync();
        assertThat(syncConnection.get("test")).isEqualTo("100000");
    }
}
```

대부분의 String 연산은 $O(1)$이므로 매우 효율적이지만 `SUBSTR`, `GETRANGE`, `SETRANGE` 와 같은 특정 command는 $O(n)$ 의 성능이 날 수 있다.

# Hashes
Hashes는 key의 값으로 {key-value} 형태의 구조를 가지는 자료구조다. 필드-값 쌍을 추가, 변경, 증감 연산 및 제거 가능.
Hash구조는 mutable하기 때문에 언제든지 수정이 가능하다.

```java

@Test
void 해쉬_생성() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.hset("endpoint:/admin/{id}", "caller", "user:123");

    // when
    String value = syncConnection.hget(":endpoint:/admin/{id}", "caller");

    // then
    assertThat(value).isEqualTo("user:123");
}

@Test
void 해쉬_수정() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    String endpointKey = "endpoint:/admin/{id}";
    syncConnection.hset(endpointKey, Map.of(
            "caller", "user:123",
            "initAccessIp", "127.0.0.1"
            )
    ); //

    // when
    syncConnection.hset(endpointKey, "caller", "user:456");
    syncConnection.hset(endpointKey, "lastAccessTime", Instant.now().toString());
    syncConnection.hsetnx(endpointKey, "initAccessIp", "192.1.1.1");
    syncConnection.hincrby(endpointKey, "count", 1);

    // then
    String caller = syncConnection.hget(endpointKey, "caller");
    String count = syncConnection.hget(endpointKey, "count");
    String lastAccessTime = syncConnection.hget(endpointKey, "lastAccessTime");
    String initAccessIp = syncConnection.hget(endpointKey, "initAccessIp");
    assertThat(caller).isEqualTo("user:456");
    assertThat(count).isEqualTo("1");
    assertThat(lastAccessTime).isNotNull();
    assertThat(initAccessIp).isNotEqualTo("192.1.1.1");
}

@Test
void 해쉬_여러_필드_조회() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    String endpointKey = "endpoint:/admin/{id}";
    syncConnection.hset(endpointKey, Map.of(
                    "caller", "user:123",
                    "initAccessIp", "127.0.0.1"
            )
    );

    // when
    var result = syncConnection.hmget(endpointKey,
            "caller", "initAccessIp", "lastAccessTime", "count"
    );

    // then
    assertThat(result).hasSize(4);
    assertThat(result.get(0).getValue()).isEqualTo("user:123");
    assertThat(result.get(1).getValue()).isEqualTo("127.0.0.1");

    assertThat(result.get(2).isEmpty()).isTrue();
    assertThat(result.get(3).isEmpty()).isTrue();
}

@Test
void 다른_데이터_타입으로_필드_조회() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    String endpointKey = "endpoint:/admin/{id}";
    syncConnection.hset(endpointKey, Map.of(
                    "caller", "user:123",
                    "initAccessIp", "127.0.0.1"
            )
    );

    // when
    assertThatThrownBy(() -> syncConnection.get(endpointKey))
            .isInstanceOf(RedisCommandExecutionException.class)
            .hasMessage("WRONGTYPE Operation against a key holding the wrong kind of value");
}

@Test
void 해쉬_전체_조회_성능_평가() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    String endpointKey = "endpoint:/admin/{id}";
    Map<String, String> hashes = new LinkedHashMap<>();
    for (int i = 0; i < 100000; i++) {
        hashes.put("key" + i, "value" + i);
    }
    syncConnection.hset(endpointKey, hashes);

    // when
    Long hgetLatency = timeRater(() -> syncConnection.hgetall(endpointKey));
    Long hsacnLatency = timeRater(() -> syncConnection.hscan(endpointKey, Builder.matches("*")));

    System.out.println("hgetall latency: " + hgetLatency);
    System.out.println("hscan latency: " + hsacnLatency);
    // hgetall latency: 107
    // hscan latency: 2
}

private Long timeRater(Supplier<?> supplier) {
    Instant now = Instant.now();
    supplier.get();
    Instant after = Instant.now();
    return Duration.between(now, after).toMillis();
}
```
대부분의 Hash 연산은 $O(1)$이므로 매우 효율적이지만 HKEYS, HVALS, HGETALL과 같은 일부 명령은 $O(n)$이다.
저장 용량은 43억개의 field-value pair가 저장 가능 하지만 실행되는 호스트의 메모리에 제한 받는다.

# Lists
List는 값으로 문자열의 연속을 저장한다. Double-Linked-List 자료구조로 구성 되어 있다([메모리 절약을 위해 zip-list도 차용했다](https://redis.com/glossary/redis-ziplist/)).
자료구조 특성상 빠른 데이터 추가/삭제가 가능하다. 범위 조회가 빈번한 경우 `Sorted sets`를 권장한다.
사용 가능한 명령어를 통해 큐, 스택처럼 활용 할 수 있다.

간단한 element 조작 명령 뿐만 아니라, blocking과 관련한 기능들도 제공한다.

```java
@Test
void 리스트_생성() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.lpush("playlists", "playlist:1");
    assertThat(syncConnection.type("playlists")).isEqualTo("list");

    // when
    var value = syncConnection.lrange("playlists", 0, -1).get(0);

    // then
    assertThat(value).isEqualTo("playlist:1");
}

@Test
void 리스트는_순서를_보장_하며_중복된_값을_저장_한다() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.lpush("playlists", "playlist:1");

    // when
    syncConnection.rpush("playlists", "playlist:2");
    syncConnection.lpush("playlists", "playlist:0");
    syncConnection.rpush("playlists", "playlist:2");

    // then
    var all = syncConnection.lrange("playlists", 0, -1);
    assertThat(all).containsExactly(
            "playlist:0", "playlist:1", "playlist:2", "playlist:2"
    );
}

@Test
void 리스트_요소_제거() {
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.rpush("playlists", "playlist:1");
    syncConnection.rpush("playlists", "playlist:2");
    syncConnection.rpush("playlists", "playlist:0");
    syncConnection.rpush("playlists", "playlist:2");
    syncConnection.rpush("playlists", "playlist:1");
    syncConnection.rpush("playlists", "playlist:2");
    assertThat(syncConnection.lrange("playlists", 0, -1)).containsExactly(
            "playlist:1", "playlist:2", "playlist:0", "playlist:2", "playlist:1", "playlist:2"
    );

    assertThat(syncConnection.lpop("playlists")).isEqualTo("playlist:1");
    assertThat(syncConnection.lpop("playlists")).isEqualTo("playlist:2");
    assertThat(syncConnection.lrange("playlists", 0, -1)).containsExactly(
            "playlist:0", "playlist:2", "playlist:1", "playlist:2"
    );

    syncConnection.ltrim("playlists", 0, 1);
    assertThat(syncConnection.lrange("playlists", 0, -1)).containsExactly(
            "playlist:0", "playlist:2"
    );
}
```

--- 
`Blocking` 연산을 통해 `messaging queue`와 같은 작업에서 유용하게 사용 할 수 있다.
이벤트가 존재하는지를 매번 확인하는 폴링 방식 대신, 지원하는 blocking을 통해 발행된 작업을 수행하는 식으로 활용 할 수 있다.

`Provider`가 queue로 사용하는 `list`에 `job`을 등록하는 경우, `Cosumer`는 `BRPOP list-key`를 통해 대기하다가 해당 값을 가져가면 된다.
이 때, connection을 점유하고 있으므로 timeout을 설정하며 무기한적인 대기를 방지해야한다. (0으로 설정하면 무한 대기)

- ex) `BRPOP` 

```java

@Test
void 블로킹_연산() {
    // given
    // redis에 connection 개수 count용 client
    RedisClient clientA = RedisConnectionProvider.client();
    var counterConnection = clientA.connect().sync();
    int connectionCountBeforeBlocking = RedisConnectionProvider.getConnectionCount(counterConnection);
    
    // when
    // blocking 작업을 하는 client
    RedisClient clientB = RedisConnectionProvider.client();
    var syncConnection = clientB.connect().async();
    syncConnection.brpop(10, "playlists");

    // then
    // 아직 "playlists"에 tail이 존재하지 않아 connection 수가 1 증가해 있음
    int connectionCountAfterBlocking = RedisConnectionProvider.getConnectionCount(counterConnection);
    assertThat(connectionCountAfterBlocking).isEqualTo(connectionCountBeforeBlocking + 1);
    
    var otherConnection = RedisConnectionProvider.getSync(); // insert element by other connection
    otherConnection.rpush("playlists", "playlist:1");
    assertThat(otherConnection.lrange("playlists", 0, -1)).isEmpty();
}
```

[다양한 커맨드들](https://redis.io/commands/?group=list)


# Sets
중복이 없고, 순서가 보장 되지 않는 자료구조이다.
단순히 값을 저장하는 것 뿐만 아니라 집합간의 연산도 가능하다.
`SINTER`, `SDIFF`, `SUNION`

많은 메모리를 사용할 수 있으므로, 정밀도를 조금 포기하고 메모리 사용량을 고려한다면 `Bloom filter` or `Cuckoo filter`를 권장한다.

```java
@Test
void Set_생성() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.sadd("user:online", "user:1", "user:2", "user:3");

    // when
    List<String> values = syncConnection.sscan("user:online", Builder.matches("user:*")).getValues();

    // then
    assertThat(values).hasSize(3);
    assertThat(values).containsExactlyInAnyOrder("user:1", "user:2", "user:3");
}

@Test
void Set_연산() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.sadd("user:online", "user:1", "user:2", "user:3");
    syncConnection.sadd("user:online:redis-study", "user:2", "user:3");

    // when
    Set<String> onlineUserInRedisStudy = syncConnection.sinter("user:online", "user:online:redis-study");

    // then
    assertThat(onlineUserInRedisStudy).hasSize(2);
    assertThat(onlineUserInRedisStudy).containsExactlyInAnyOrder("user:2", "user:3");
}

@Test
void Set_제거() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.sadd("user:online",
            "user:1", "user:2", "user:3", "temp:user:1", "temp:user:2", "temp:user:3"
    );


    // when
    syncConnection.srem("user:online",
            "temp:user:1", "temp:user:2", "temp:user:3"
    );

    // then
    Long sizeOfSet = syncConnection.scard("user:online");
    assertThat(sizeOfSet).isEqualTo(3);
}

@Test
void Set_랜덤으로_제거() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.sadd("user:online",
            "user:1", "user:2", "user:3", "temp:user:1", "temp:user:2", "temp:user:3"
    );


    // when
    syncConnection.spop("user:online", 2);

    // then
    Long sizeOfSet = syncConnection.scard("user:online");
    assertThat(sizeOfSet).isEqualTo(4);
}
```

# Sorted Sets
Set에 자료구조에서 가중치(score)가 추가되어 정렬을 유지하는 구조. 가중치가 같다면, key로 정렬된다.(Set의 특성에 따라 key는 같을 수 없다)

가중치를 기준으로 정렬되기 때문에, 우선 순위 큐나 실시간 랭킹, 인덱스로도 활용하기 좋다.

`Sorted Set`은 데이터가 저장될 때 저장되며 `Lists`와 비슷하게, 상황에 따라 내부적으로 두 가지의 데이터 구조로 저장된다.
- member 모두 64 byte 이하이며 member 수가 128개 이하이면 Zip List 데이터 구조로 저장
- member 중 하나라도 65byte 이상이거나 member 수가 129개 이상이면 [Skip List](http://redisgate.kr/redis/configuration/internal_skiplist.php) 데이터 구조로 변환하여 저장

정렬된 범위를 조회하거나, Top10 등 다양한 command를 제공한다.
대부분의 정렬 집합 연산은 $O(log(n))$이다.

```java
@Test
void SortedSet_생성() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();

    // when
    syncConnection.zadd("user:rank", 5, "user:1");
    syncConnection.zadd("user:rank", 4, "user:2");
    syncConnection.zadd("user:rank", 3, "user:3");

    // then
    var ranks = syncConnection.zrange("user:rank", 0, -1);
    assertThat(ranks).containsExactly(
            "user:3", "user:2", "user:1"
    );

    Long rank = syncConnection.zrank("user:rank", "user:1");
    assertThat(rank).isEqualTo(2L);

    Long reverseRank = syncConnection.zrevrank("user:rank", "user:1");
    assertThat(reverseRank).isEqualTo(0L);

    Double score = syncConnection.zscore("user:rank", "user:1");
    assertThat(score).isEqualTo(5.0);

    syncConnection.zremrangebyrank("user:rank", 0, 1);
    ranks = syncConnection.zrange("user:rank", 0, -1);
    assertThat(ranks).containsExactly("user:1");
}
```

# Stream
Redis Stream(레디스 스트림)은 Redis 5.0부터 추가 된 자료구조로, log 파일처럼 append only로 저장되는 구조를 가지고 있다.
Pub/Sub 메커니즘보다 더 복잡한 스트림 처리 및 메시지 유지를 지원 한다.

기본적으로 Redis Stream은 타임라인에 따라 정렬된 메시지의 일련의 데이터로 구성됩니다. Provider가 메시지를 스트림에 추가하고, Consumer(혹은 group)이 스트림에서 메시지를 소비할 수 있는 구조.

Redis 스트림은 추가 전용 로그처럼 작동하지만 일반적인 추가 전용 로그의 몇 가지 한계를 극복하기 위해 몇 가지 작업을 구현하는 데이터 구조.
$O(1)$의 시간 복잡도로 접근 및 추가 작업이 이루어지도록 했다고 한다.

Redis 스트림 사용 사례
- 이벤트 소싱(예: 사용자 작업, 클릭 등 추적)
- 센서 모니터링(예: 현장의 장치에서 판독값)
- 알림(예: 각 사용자의 알림 기록을 별도의 스트림에 저장)
- 로깅

Redis는 각 스트림 항목에 대해 시간과 관련된 고유 `ID`를 생성한다. `ID`를 사용하여 나중에 관련 항목을 검색하거나 소비 할 수 있다.

# HyperLogLog

`HyperLogLog`는 `Sets`과 동일하게 순서와 상관없이 중복되지 않는 데이터들의 카디널리티를 추정하는 확률적 데이터 구조이다.

[HLL 알고리즘](https://d2.naver.com/helloworld/711301)을 사용하여 최대 12KB의 적은 메모리 사용량과 함께 $0.81%$의 표준 오류가 발생 할 수 있다.

기술적으로는 다른 데이터 구조이지만 `HLL`은 `String`으로 인코딩되기 때문에 값을 직렬화하고 SET을 호출하여 값을 확인 할 수도 있다.

이러한 특성을 바탕으로 정확성은 조금 떨어져도 효율적으로 카디널리티 측정이 필요한 상황에서 사용 하면 좋다. (counter)
- 페이지 방문자 수
- 요청 사용자 수
- 특정 구간을 지나간 차량 수

```java
@Test
void HLL_생성_테스트() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.pfadd("endpoint:/admin", "user:1", "user:2", "user:3");

    // when
    long count = syncConnection.pfcount("endpoint:/admin");

    // then
    assertThat(count).isEqualTo(3);
}

@Test
void HLL_MERGE_테스트() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();
    syncConnection.pfadd("endpoint:/admin/move", "user:1", "user:2", "user:3", "user:4");
    syncConnection.pfadd("endpoint:/admin/enroll", "user:1", "user:2");
    syncConnection.pfadd("endpoint:/admin/remove", "user:3", "user:4", "user:5");

    // when
    syncConnection.pfmerge( "endpoint:/admin", "endpoint:/admin/enroll", "endpoint:/admin/remove");

    // then
    long count = syncConnection.pfcount("endpoint:/admin");
    assertThat(count).isEqualTo(5);
}
```

`SET`과 성능 및 메모리 비교
```java
@Test
void SET과_성능_비교() {
    // given
    var syncConnection = RedisConnectionProvider.getSync();

    // when
    Long setTime = timeRater(() -> {
        for (int i = 0; i < 10000; i++) {
            syncConnection.sadd("user:online:set", "user:" + i);
        }
        return syncConnection.scard("user:online");
    });

    Long hllTime = timeRater(() -> {
        for (int i = 0; i < 10000; i++) {
            syncConnection.pfadd("user:online:hll", "user:" + i);
        }
        return syncConnection.pfcount("user:online:hll");
    });

    // then
    Long setMemory = syncConnection.memoryUsage("user:online:set");
    Long hllMemory = syncConnection.memoryUsage("user:online:hll");
    System.out.println("setTime = " + setTime + ", setMemory = " + setMemory);
    System.out.println("hllTime = " + hllTime + ", hllMemory = " + hllMemory);
//        setTime = 6557, setMemory = 596728
//        hllTime = 6476, hllMemory = 14400
}

private Long timeRater(Supplier<?> supplier) {
    Instant now = Instant.now();
    supplier.get();
    Instant after = Instant.now();
    return Duration.between(now, after).toMillis();
}
```
메모리 사용량에서 엄청 큰차이가 발생 하는 것을 확인할 수 있다.
