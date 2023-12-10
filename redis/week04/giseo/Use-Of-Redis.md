# Patterns

## 1. Bulk Loading
- 일반적인 Redis 클라이언트로 bulk 데이터를 처리하는 것은 RTT 측면에서 비효율적이다.
- pipelining을 적용할 수 있지만 많은 레코드를 대량으로 로드하기에는 부적합하다.
- 소수의 클라이언트만 non-blocking I/O를 지원하기 때문에 모든 클라이언트에서 bulk쓰기 처리량을 최대화 하기 위해선 Redis 프로토콜이 포함된 text 파일을 생성해야 한다.


- 10억 개의 키가 있는 대규모 데이터를 생성해야 하는 경우 해당 명령을 Redis 프로토콜 형식으로 생성하고 서버에 전송하면 된다. 
```
SET Key0 Value0
SET Key1 Value1
...
SET KeyN ValueN
```

- `netcat` 명령어는이 모든 데이터가 언제 전송되었는지 실제로 알지 못하고 오류를 확인할 수 없다.
- 아래의 `pipe mode`로 데이터를 전송하는 것을 권장한다.

```bash
cat data.txt | redis-cli --pipe
```

### Generating Redis Protocol
- SET key value 명령은 명령어(SET), 키(key), 값(value)의 세 부분이 인자로 전달된다. 
```
*<args><cr><lf>         # *: 배열의 시작 ||  <args>: 명령어에 전달될 인자의 개수     || <cr><lf>: 라인의 끝
$<len><cr><lf>          # $: 바이너리 문자열의 시작
<arg0><cr><lf>          # <arg0>: 첫 번째 인자의 내용
<arg1><cr><lf>
...
<argN><cr><lf>
```

- SET keyA valueA 를 프로토콜로 작성하면 아래와 같다.
```
*3<cr><lf>         # 총 3개의 인자가 전달된다. SET, key, value
$3<cr><lf>         # $3: 다음에 오는 문자열이 3바이트 길이임을 나타낸다.
SET<cr><lf>
$4<cr><lf> 
key<cr><lf>
$6<cr><lf>
valueA<cr><lf>
```


## 2. Distributed Lock
- 서로 다른 프로세스에서 동일한 공유자원에 접근할 때, 데이터의 원자성을 보장하기위해 활용되는 방법
- 스프링에서 레디스를 사용하기 위한 클라이언트는 Lettuce, Redisson, Jedis 등등 이 있는데, **Redisson**이 Redlock 알고리즘을 사용해 분산락을 구현하고 있다.

## Safety and Liveness Guarantees
- Safety property: 상호배제. 어떤 순간에도 하나의 클라이언트만 락을 획득할 수 있어야 한다.
- Liveness property A: 데드락이 발생하지 않는 것. 락을 획득한 클라이언트에 문제가 생겨 락을 풀지 못하더라도 결국에는 락을 획득할 수 있어야 한다.
- Liveness property B: 내결함성. 특정 노드에 장애가 발생해도 클라이언트는 락을 획득하고 해제할 수 있어야 한다.

## Single Instance
- 락을 획득하기 위한 방법은 다음과 같다.
```
 SET resource_name my_random_value NX PX 30000
```
- 위 명령어는 락이 존재하지 않는 다면 키를 세팅하고 30000ms의 타임아웃 시간을 가진다.

- 락을 해제하는 방법은 다음과 같다(Lua Script)

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end

```
- 락을 해제하기 전에, 현재 키의 값(KEYS[1]) 이 예상된 값(ARGV[1]) 과 일치하는지 확인한다.
  - 락을 획득한 클라이언트만 락을 해제할 수 있다.

- 이 방법을 사용하면 분산 시스템이 아닌 단일 시스템의 대부분의 경우에서 안전하다.
- 다만, 이러한 방식으로 구축된 단일 레디스 노드는 단일 장애 지점(SPOF, Single Point Of Failure)이 될 수 있다.

## Why Failover-based Implementations Are Not Enough
- Redis를 사용하여 리소스를 잠그는 가장 간단한 방법은 인스턴스에 키와 만료 시간을 생성하는 것이다.
- 하지만 이 방식은 아래의 문제가 존재한다.
    - Redis Master가 고장날 경우 Replica가 Master로 승격된다.
    - Replica에 잠금 정보가 복제되기 전에 다른 클라이언트에서 동일한 키에 잠금을 획득한다.

## Redlock Algorithm
- 클라이언트는 분산 환경에서 락을 획득하기 위해 다음 작업을 수행한다.

> 1. 현재 시간을 ms 단위로 가져온다.
> 2. 모든 인스턴스에서 순차적으로 잠금을 획득하려고 시도한다. 각 인스턴스에 잠금을 설정할 때 클라이언트는 전체 잠금 자동 해제 시간에 비해 작은 타임아웃을 사용하여 잠금을 획득한다. 예를 들어 자동 해제 시간이 10s인 경우, 타임아웃은 5~50ms가 될 수 있다. 이를 통해 클라이언트가 다운된 Redis 노드와 통신하려고 오랫동안 블로킹되는 것을 방지할 수 있다.
> 3. 클라이언트는 (현재 시간 - 1단계에서 얻은 타임스탬프)를 통해 잠금을 획득하기 위해 경과한 시간을 계산한다. 클라이언트가 과반이 넘는(N/2 + 1) 인스턴스에서 잠금을 획득했고, 총 경과 시간이 잠금 유효 시간보다 적다면 분산락을 획득한 것으로 간주한다.
> 4. 분산락을 획득한 경우, 잠금 유효 시간은 3단계에서 계산한 시간으로 간주한다.
> 5. 분산락을 획득하지 못한 경우(과반이 넘는 인스턴스를 잠글 수 없거나 유효 시간이 음수인 경우), 클라이언트는 모든 인스턴스에서 잠금을 해제하려고 시도한다.

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
        "if (redis.call('exists', KEYS[1]) == 0) then " +     // LOCK KEY가 존재하는지 확인한다(없으면 0, 있으면 1)
        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +      // LOCK KEY가 존재하지 않으면 LOCK KEY와 현재 쓰레드 아이디를 기반으로 값을 1 증가시켜준다
        "redis.call('pexpire', KEYS[1], ARGV[1]); " +         // LOCK KEY에 유효시간을 설정한다
        "return nil; " +
        "end; " +
        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +     // 해시맵 기반으로 LOCK KEY와 쓰레드 아이디로 존재하면 0이고, 존재하지 않으면 저장하고 1을 리턴한다
        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                // LOCK KEY가 존재하지 않으면 LOCK KEY와 현재 쓰레드 아이디를 기반으로 값을 1 증가시켜준다
        "redis.call('pexpire', KEYS[1], ARGV[1]); " +                   // LOCK KEY에 유효시간을 설정한다
        "return nil; " +
        "end; " +
        "return redis.call('pttl', KEYS[1]);",            // 위의 조건들이 모두 false 이면 현재 존재하는 LOCK KEY의 TTL 시간을 리턴한다
        Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
        redis.call('exists', KEYS[1]) == 0
}
        
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);                                              // 락 획득 대기시간
    long current = System.currentTimeMillis();                                        // 현재 시간
    long threadId = Thread.currentThread().getId();
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {                                          // 다른 스레드가 보유한 Lock의 TTL이 null인 경우
        return true;
    }
    
    time -= System.currentTimeMillis() - current;               // 락 획득 대기 시간보다 초과되었는지 확인하고 초과되었으면 false를 리턴
    if (time <= 0) {                     
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
    
    current = System.currentTimeMillis();
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);       // thread id로 구독한 채널에서 lock 획득이 유효할때까지 대기한다. 세마포어를 사용한다.
    try {
        subscribeFuture.get(time, TimeUnit.MILLISECONDS);                          // 이벤트가 수신될 때 까지 block
        } catch (TimeoutException e) {                                             // 대기 시간을 초과할 경우 lock 획득에 실패
        if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
                "Unable to acquire subscription lock after " + time + "ms. " +
                        "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
            subscribeFuture.whenComplete((res, ex) -> {
                if (ex == null) {
                    unsubscribe(res, threadId);
                }
            });
        }
        acquireFailed(waitTime, unit, threadId);
        return false;
    } catch (ExecutionException e) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    try {
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
    
        while (true) {                                                    // 무한루프를 수행하면서 대기 시간이 남아있는지 체크한다.
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                return true;
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }

            // 남은시간까지 lock이 avaliable한지 구독한다
            currentTime = System.currentTimeMillis();
            if (ttl >= 0 && ttl < time) {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    } finally {
        unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);
    }
//        return get(tryLockAsync(waitTime, leaseTime, unit));
}

```


## Clock Drift
- RedLock 알고리즘은 노드들 간에 동기화된 시계(synchronized clock)는 없지만, 로컬 시간이 거의 동일한 속도로 갱신된다는 가정에 의존한다. 
- 하지만 현실에서는 클럭이 정확한 속도로 동작하지 않는 클럭 드리프트(Clock Drift) 현상으로 인해 레드락 알고리즘에 문제가 생길 수 있다.
  - 예를 들어 시스템에 Redis 노드 5개(A, B, C, D, E)와 클라이언트 2개(1, 2)가 있다고 하자. 이때 클럭 드리프트 현상이 발생한다면 분산락 알고리즘은 깨질 수 있다.

> - 클라이언트 1이 노드 A, B, C에서 잠금을 획득하지만, 네트워크 문제로 인해 D와 E에서는 잠금 획득에 실패한다.
> - 이때 노드 C의 시계가 앞으로 이동하여 잠금이 만료된다.
> - 클라이언트 2가 노드 C, D, E에서 잠금을 획득하지만, 네트워크 문제로 인해 A와 B에서는 잠금 획득에 실패한다.
> - 이제 클라이언트 1과 2는 모두 자신이 잠금을 획득했다고 믿는다.



## 3. Secondary Indexing
- Redis는 API 수준에서는 **primary key**를 통해 Data에 접근하는 구조를 가지고 있다.
  - 다양한 종류의 **보조 인덱스**와 **복합 인덱스**를 생성하는 기능을 가지고 있다.
  1. Sorted sets을 이용한 numerical, ID indexes
  2. Sorted Sets을 이용한 보조 인덱스, 복합 인덱스, 그래프 탐색 인덱스
  3. Sets을 이용한 임의의 인덱스
  4. Lists을 이용한 단순 반복 인덱스, 최근 N개 아이템 인덱스

1. Sorted sets을 이용한 numerical, ID indexes
- ZADD를 사용해 데이터를 정렬된 집합에 추가하는 순간부터, 그 데이터는 일종의 인덱스 역할을 할 수 있다.

- 사용자 나이를 기반으로 생성한 보조인덱스
```
ZADD myindex 25 Manuel
ZADD myindex 18 Anna
ZADD myindex 35 Jon
ZADD myindex 67 Helen

ZRANGE myindex 20 40 BYSCORE
1) "Manuel"
2) "Jon"
```

```
HMSET user:1 id 1 username antirez ctime 1444809424 age 38
HMSET user:2 id 2 username maria ctime 1444808132 age 42
HMSET user:3 id 3 username jballard ctime 1443246218 age 33

ZADD user.age.index 38 1
ZADD user.age.index 42 2
ZADD user.age.index 33 3

ZRANGE user.age.index 20 40 BYSCORE
1) "1"
2) "3"
```


2. Sorted Sets의 lexicographical index를 이용한 보조 인덱스, 복합 인덱스, 그래프 탐색 인덱스
- Sorted Sets을 이용하면 특정 필드(사용자의 나이, 상품의 가격)에 대한 보조 인덱스를 생성할 수 있다. 
- Sorted Sets에 점수가 같은 요소들을 추가하면, 이들은 사전 순으로 정렬된다.
  - 문자열을 이진 데이터로 변환하여 정렬하기때문
  - 이를 이용해 사전식 인덱스를 활용할 수 있다.

- ZRANGE 명령어와 BYLEX 인수를 사용하여 사전식 범위 내의 요소를 쿼리한다.
  - 범위는 특수 문자 [와 (를 사용하여 지정
  - 
```
ZADD myindex 0 baaa
ZADD myindex 0 abbb
ZADD myindex 0 aaaa
ZADD myindex 0 bbbb

ZRANGE myindex [a (b BYLEX  // 'a'로 시작하는 모든 요소를 반환, 'b'로 시작하는 모든 요소는 반환 X

"aaaa"
"abbb"
```

- 활용 사례
  - 완성 기능
  ```
  ZRANGE myindex "[bit" "[bit\xff" BYLEX  // 'bit'로 시작하는 모든 문자열 반환
  ```
  - 빈도수 추가
    - 검색어의 빈도수에 따라 자동완성을 제공할 수 있음.
  ```
  ZADD myindex 0 banana:1  // banana가 한 번 검색되었음을 나타낸다.
  ```

- 사용자의 검색 행동을 기반으로 자동 완성 기능을 효과적으로 구현하고, 검색어의 빈도수에 따라 결과를 조정할 수 있다.


3. Sets을 이용한 임의의 인덱스
- 집합은 순서가 없는 문자열의 컬렉션으로, 각 요소는 집합 내에서 유일하다.

```
SADD myset "element1" "element2" "element3"

SRANDMEMBER myset  // 랜덤 요소 선택

SPOP myset  // "myset" 집합에서 임의의 요소를 제거하고 그 요소를 반환 
```

- 사용 사례
  - 세션 관리: 사용자 세션 ID를 랜덤하게 선택할 때
  - 데이터 샘플링: 데이터 분석이나 실험을 위해 무작위 데이터를 선택할 때
  - 게임 개발: 게임에서 무작위 이벤트를 생성할 때
  
- 순서가 중요하지 않고 중복되지 않는 랜덤한 요소가 필요한 경우


4. Lists을 이용한 단순 반복 인덱스, 최근 N개 아이템 인덱스
- 순서가 있는 문자열의 컬렉션으로, 각 요소는 리스트 내에서 특정 순서를 가진다.
- LRANGE 명령어를 사용하여 리스트의 특정 범위의 요소를 검색할 수 있다.
```
LPUSH mylist "element1"
RPUSH mylist "element2"

LRANGE mylist 0 -1  
LRANGE mylist -N -1  // 최근 N개 아이템 인덱스
```

- 사용 사례
  - 로그 관리: 최근 이벤트나 로그 메시지를 순서대로 저장하고 검색
