# Bulk Loading
- 대량의 기존 데이터를 Redis에 로딩하는 프로세스
- 일반적인 Redis 클라이언트를 사용하여 대량 로드를 수행하는 것은 좋지 않다.
- `keyN -> ValueN' 형식의 수십억 개의 키가 있는 대규모 데이터 세트를 생성해야 하는 경우 Redis protocol 형식으로 다음 명령을 포함하는 파일을 생성하고 파이프 모드를 사용하여 명령을 실행한다
```
SET Key0 Value0
SET Key1 Value1
...
SET KeyN ValueN
```
```
* netcat 명령어로도 할 수 있지만, 모든 데이터가 언제 전송되었는지 실제로 알지 못하고 오류를 확인할 수 없기 때문에 --pipe 를 통해 하는 것을 권장

cat data.txt | redis-cli --pipe
```
- Redis protocol 은 아래처럼 생성할 수 있다
```
*<args><cr><lf>
$<len><cr><lf>
<arg0><cr><lf>
<arg1><cr><lf>
...
<argN><cr><lf>
```
```
예시

*3<cr><lf>
$3<cr><lf>
SET<cr><lf>
$3<cr><lf>
key<cr><lf>
$5<cr><lf>
value<cr><lf>

--- 

*3<cr><lf>: 이 줄은 명령이 세 부분으로 구성되어 있음을 나타냅니다 (SET, key, value). *는 배열을 나타내며, 3은 배열의 요소 개수입니다. <cr><lf>는 캐리지 리턴(\r)과 라인 피드(\n)를 나타내며, 새로운 라인의 시작을 의미합니다.

$3<cr><lf>: 이 줄은 첫 번째 인자의 길이를 나타냅니다. 여기서 $는 문자열의 길이를 나타내는 데 사용되며, 3은 문자열 SET의 길이입니다.

SET<cr><lf>: 명령의 첫 번째 부분인 SET입니다.

$3<cr><lf>: 두 번째 인자의 길이를 나타냅니다. 이 경우 key라는 문자열의 길이는 3입니다.

key<cr><lf>: 명령의 두 번째 부분인 키 key입니다.

$5<cr><lf>: 세 번째 인자의 길이를 나타냅니다. 이 경우 value라는 문자열의 길이는 5입니다.

value<cr><lf>: 명령의 세 번째 부분인 값 value입니다.
```
- `--pipe` 모드가 내부적으로 동작하는 방식
    - netcat만큼 빠르면서도 동시에 서버에서 마지막 응답이 언제 전송되었는지 이해할 수 있어야 한다는 것
    - 아래와 같은 방법으로 동작됨
        - redis-cli --pipe는 가능한 한 빨리 서버에 데이터를 전송하려고 시도한다.
        - 동시에 가능한 경우 데이터를 읽고 구문 분석 시도한다.
        - stdin에서 더 이상 읽을 데이터가 없으면 임의의 20바이트 문자열이 포함된 특별한 ECHO 명령을 보낸다. 
        - 바이트를 대량 응답으로 보낸다.
        - 최종 명령이 전송되면 응답을 수신하는 코드는 응답을 이 20바이트와 일치시키기 시작한다. 
        - 일치하는 응답에 도달하면 성공하여 종료된다.

# Distributed locks
- 분산락은 서로 다른 프로세스가 상호 배타적인 방식으로 공유 리소스를 사용하여 작동해야 하는 많은 환경에서 매우 유용한 기본 요소다.

## Why Failover-based Implementations Are Not Enough
- Redis를 사용하여 리소스를 잠그는 가장 간단한 방법은 인스턴스에 키를 생성하는 것
- replica 를 만들더라도 아래 문제가 발생할 수 있다.
```
1. 클라이언트 A는 마스터에서 잠금을 획득합니다.
2. 키에 대한 쓰기가 복제본으로 전송되기 전에 마스터가 충돌합니다.
3. 복제본이 마스터로 승격됩니다.
4. 클라이언트 B는 A가 이미 잠금을 보유하고 있는 동일한 리소스에 대한 잠금을 획득합니다.
```

## Correct Implementation with a Single Instance
- 키가 존재하고 키에 저장된 값이 정확히 내가 기대하는 값인 경우에만 키를 제거하라는 스크립트를 사용하여 안전한 방법으로 잠금을 해제하게 한다.
- 이는 다른 클라이언트가 생성한 잠금이 제거되는 것을 방지하기 위해 중요하다.

## The Redlock Algorithm
Redlock 알고리즘의 작동 원리
- 여러 Redis 인스턴스 사용: Redlock 알고리즘은 여러 개의 독립적인 Redis 인스턴스 (일반적으로 5개)를 사용한다. 이 인스턴스들은 서로의 상태나 데이터를 공유하지 않는다.
- 락 시도: 클라이언트는 모든 Redis 인스턴스에 동시에 락을 획득하려고 시도한다. 각 인스턴스에서는 SET 명령어와 NX (키가 존재하지 않을 때만 설정), PX (락의 유효 시간을 밀리초 단위로 설정) 옵션을 사용한다.
- 다수결 원칙: 클라이언트는 전체 Redis 인스턴스 중 과반수 이상에서 락을 성공적으로 획득하고, 전체 시도 시간이 락의 유효 시간의 절반을 넘지 않았을 때만 락을 성공적으로 획득한 것으로 간주한다.
- 락 해제: 작업이 완료되면, 클라이언트는 모든 Redis 인스턴스에서 락을 해제한다.
- 락 획득 실패: 만약 과반수 이상에서 락을 획득하는 데 실패하거나, 락 획득 시간이 너무 길면 클라이언트는 모든 인스턴스에서 락 획득 시도를 취소(해제)하고 잠시 후 다시 시도한다.

# Secondary indexing
- Redis는 데이터 구조 서버이므로 복합(다중 열) 인덱스를 포함하여 다양한 종류의 보조 인덱스를 생성하기 위해 해당 기능을 인덱싱에 사용할 수 있다.
## Simple numerical indexes with sorted sets
```
ZADD myindex 25 Manuel
ZADD myindex 18 Anna
ZADD myindex 35 Jon
ZADD myindex 67 Helen

ZRANGE myindex 20 40 BYSCORE
```
## Using objects IDs as associated values
```
HMSET user:1 id 1 username antirez ctime 1444809424 age 38
HMSET user:2 id 2 username maria ctime 1444808132 age 42
HMSET user:3 id 3 username jballard ctime 1443246218 age 33

ZADD user.age.index 38 1
ZADD user.age.index 42 2
ZADD user.age.index 33 3
```
## Updating simple sorted set indexes
- ZADD 은 다른 점수와 동일한 값을 가진 요소를 다시 추가하면 단순히 점수를 업데이트하고 요소를 올바른 위치로 이동하므로 간단한 인덱스 업데이트를 매우 간단히 할 수 있다.
```
HSET user:1 age 39
ZADD user.age.index 39 1
```
- 선형 방식으로 다차원적인 것을 효율적으로 표현할 수 있다면, 다차원 데이터도 인덱스로 쓸 수 있다.
## Lexicographical indexes
```
ZADD myindex 0 baaa
ZADD myindex 0 abbb
ZADD myindex 0 aaaa
ZADD myindex 0 bbbb

ZRANGE myindex 0 -1
1) "aaaa"
2) "abbb"
3) "baaa"
4) "bbbb"

ZRANGE myindex [a (b BYLEX
1) "aaaa"
2) "abbb"
```
- 아래 명령어 예시처럼 자동완성을 구현할 수 있다
```
ZRANGE myindex "[bit" "[bit\xff" BYLEX
```
- 위의 예시에서 아래처럼 빈도수 로직을 추가할 수도 있다
```
ZRANGE myindex "[banana:" + BYLEX LIMIT 0 1

ZREM myindex 0 banana:1
ZADD myindex 0 banana:2

동시 업데이트가 있을 수 있기 때문에 위의 세 가지 명령은 대신 Lua 스크립트를 통해 전송되어야 한다.
```
## Normalizing strings for case and accents
```
ZADD myindex 0 {normalized}:{frequency}:{original}
```

## Adding auxiliary information in the index
```
ZADD myindex 0 {mykey}:{myvalue}
```
- 콜론 문자는 키 자체의 일부일 수 있으므로 추가한 키와 충돌하지 않도록 선택해야 한다.

## Numerical padding
- 문자열 숫자에 패딩값을 넣음으로써 결과적으로 숫자 값을 기준으로 정렬되게 할 수 있다
```
ZADD myindex 0 00324823481:foo
ZADD myindex 0 12838349234:bar
ZADD myindex 0 00000000111:zap

ZRANGE myindex 0 -1
1) "00000000111:zap"
2) "00324823481:foo"
3) "12838349234:bar"
```
```
01000000000000.11000000000000
01000000000000.02200000000000
00000002121241.34893482930000
00999999999999.00000000000000
```

## Composite indexes
```
ZADD myindex 0 0056:0028.44:90
ZADD myindex 0 0034:0011.00:832

ZRANGE myindex [0056:0010.00 [0056:0030.00 BYLEX
```

## Updating lexicographical indexes
- 더 많은 메모리를 사용하면서 인덱스 처리를 단순화 하는 방법이다.
- 항상 필요한 것은 아니지만 인덱스 업데이트 작업을 단순화 할 수 있다.
```
MULTI
ZADD myindex 0 0056:0028.44:90
HSET index.content 90 0056:0028.44:90
EXEC
```

## Representing and querying graphs using a hexastore
- `o`bjects, formed by a `s`ubject, a `p`redicate and an object
```
antirez is-friend-of matteocollina

ZADD myindex 0 spo:antirez:is-friend-of:matteocollina
ZADD myindex 0 sop:antirez:matteocollina:is-friend-of
ZADD myindex 0 ops:matteocollina:is-friend-of:antirez
ZADD myindex 0 osp:matteocollina:antirez:is-friend-of
ZADD myindex 0 pso:is-friend-of:antirez:matteocollina
ZADD myindex 0 pos:is-friend-of:matteocollina:antirez

ZRANGE myindex "[spo:antirez:is-friend-of:" "[spo:antirez:is-friend-of:\xff" BYLEX
```

## Multi dimensional indexes
