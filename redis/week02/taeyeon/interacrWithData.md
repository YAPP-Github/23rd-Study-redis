# Redis 프로그래밍
> docs 정리 https://redis.io/docs/interact/programmability/ 

`Redis`는 서버 자체에서 사용자가 작성한 스크립트를 실행할 수 있는 인터페이스가 제공 된다.
스크립트 뿐만 아니라 `Redis v7` 이상에서는 `function`을 제공한다.
`Redis v6.2` 이하에서는 `EVAL` 명령과 함께 `Lua scripting`을 사용하여 실행 할 수 있다.
> Nov, 2023 기준으로 Redis의 LTS는 `7.2v` 이다

단일적인 명령어를 반복 실행 하는 대신, 스크립트를 통해 비즈니스 로직을 캡슐화 하여 응집도 있게 구성하고, 네트워크 트래픽 감소와 전반적인 성능 향상을 기대 할 수 있다.

## 스크립트 실행
Redis는 스크립트 실행을 위한 두 가지 수단을 제공된다.
### Redis Eval Scripts
`Redis 2.6`부터 `EVAL` 명령으로 빠르고 간단하게 `server-side` 스크립트를 실행 시킬 수 있다.
스크립트가 `redis`가 아닌 어플리케이션에 존재하기 때문에 `Redis-server`의 확장으로 보진 않는다.
Redis가 실행하는 스크립트가 Redis를 사용하는 클라이언트(application)에 대해서만 관리되고 있기 때문이다. 만약 scale-out이 발생한다면, 공유하는 redis에 대한 스크립트 관리 및 유지가 어려워질 수 있다.

### Redis Function
`Redis 7.0`에 추가된 Redis Function은 Redis의 일급(first-class) 요소인 스크립트이다.
`Eval scripting`과 다르게 어플리케이션에서 로직을 관리 및 전달 하는 것이 아닌, 독립적으로 분리되어 존재 하며 테스트 및 배포도 가능하다.
독립적으로 존재하는 Function을 Redis 모듈에 포함하여 로드하여 모든 클라이언트에서 사용 할 수 있도록 구성한다.

모든 스크립트 실행은 Atomic 연산을 보장한다. 스크립트가 실행 되는 동안, redis를 점유하여 다른 client의 접근을 blocking 하기 때문에 성능 저하를 유발 할 수 있다는 점에 유의하여야 한다.

### Read-only scripts
읽기 전용 스크립트를 구성하고 실행할 수 있다. 스크립트 내부엔 쓰기 작업이 존재하면 안되고, `EVAL_RO`, `FACLL_RO` 처럼 Read-Only를 명시하고 실행시켜야 한다.
읽기 전용 스크립트를 통해 성능적 이점과 replica 기능 지원 및 ACL 사용자별 권한 제공등을 활용 할 수 있다.
읽기 전용 스크립트 및 실행 명령은 `Redis 7.0`에 도입되었다.

Redis 스크립트는 샌드박스라는 영역에 배치되는데, Redis의 파일 시스템, 네트워크에 접근하거나 API로 지원되는 것 이외의 다른 system call을 호출 하는 것을 방지하기 위함이다.

### Maximum excution time
스크립트는 최대 실행 시간을 가진다(default 5초). 실수로 생성된 무한 루프를 방지하기 위해 존재하는 제한.
`redis.conf`에서 설정하거나 `CONFIG SET` 명령을 통해 최대 실행 시간을 수정할 수 있다. 

고려해야 할 사항은 최대 실행 시간을 초과하더라도 timeout 처럼 스크립트가 자동으로 종료되지 않는다는 점이다.
만약 초과했다면, redis에선 로깅을 남기고 다른 요청들에 대해 `BUSY`라는 오류의 응답을 보낸다.
이러한 상황을 감지하였다면, 읽기 전용 스크립트에 대해선 `SCRIPT KILL`, `FUNCTION KILL` 명령을, 쓰기 작업이 포함된 경우 `SHUTDOWN NOSAVE` 의 명령으로 종료해야 한다.

# Lua scripts
Lua Scripting이란 script 언어 문법인 Lua를 Redis에서 실행하는 기술. Atomic한 연산이 보장된다.
Redis에서 local하게 실행되어 전반적인 지연 시간을 줄이고 네트워킹 리소스를 절약 할 수 있다.
스크립트는 클라이언트 측에서 관리되기 때문에 Redis Function을 권장하기도 한다.

### EVAL
Lua script는 `EVAL` 명령으로 실행 할 수 있다.
`EVAL`은 실행하고자 하는 스크립트와 파라미터를 인자로 받는다.
> eval [script] [numkeys] [key ...] [arg ...]

```shell
> EVAL "return 'Hello, scripting!'" 0
"Hello, scripting!

redis> EVAL "return ARGV[1]" 0 Hello
"Hello"
redis> EVAL "return ARGV[1]" 0 Parameterization!
"Parameterization!"

redis> EVAL "return { KEYS[1], KEYS[2], ARGV[1], ARGV[2], ARGV[3] }" 2 key1 key2 arg1 arg2 arg3
1) "key1"
2) "key2"
3) "arg1"
4) "arg2"
5) "arg3"
```

### redis call
lua script에 redis.call()을 통해 Redis 명령어를 실행 할 수 있다.
```shell
127.0.0.1:6379> EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 name royce
OK
127.0.0.1:6379> get name
"royce"
```

`redis.pcall()`은 비슷하게 동작하지만 런타임 오류를 처리하는데서 차이가 있다.
`redis.call()`은 발생한 오류를 함수를 실행시킨 클라이언트에 반환 하고, `redis.pcall()`은 스크립트 컨텍스트 내부로 반환되어 내부에서 처리할 수 있다.


### Script cache
매번 `EVAL`을 통해 스크립트를 전달하지 않고 자주 사용하는 script를 upload하여 해당 값을 실행 할 수 있다.
`SCIPRT LOAD`를 통해 스크립트를 저장 할 수 있다. Redis서버에 캐시되어 로드 하고, SHA1의 고유한 값을 반환한다.
이 값을 `EVALSHA`을 통해 실행 할 수 있다.
Load된 스크립트는 휘발성이므로 서버를 재시작하거나 `SCIPRT FLUSH` 를 통해 손실될 수 있다.

```shell
redis> SCRIPT LOAD "return 'Immabe a cached script'"
"c664a3bf70bd1d45c4284ffebb65a6f2299bfc9f"
redis> EVALSHA c664a3bf70bd1d45c4284ffebb65a6f2299bfc9f 0
"Immabe a cached script"
```

### Script replication
`Redis clustered` 운영인 경우, script는 두가지 방식으로 replica를 수행한다.
- `Verbatim replication`: Primary에서 script 복제본을 node들로 전송하고 실행시킵니다. Primary에서 수행된 동일한 script를 여러 node에서 반복 수행한다는 단점이 있다.
- `Effects replication`: script내의 쓰기 명령만 복제하여 수행한다. 쓰기 명령을 결과론적으로 수집하고 트랜잭션으로 래핑하여 replica와 aof 로 전송한다.
Redis 7.0이후 부턴 `Effects replication`만 제공하고 있다.

`redis.replicate_commands()` 을 통해 effect replication이 수행되는지 확인 할 수 있다.


# Lua API
앞서 언급한대로, lua는 sandbox라는 영역에서 직접적인 접근 권한 없이 수행 된다.
Redis에서 제공하는 API를 통해 수행 할 수 있다.
Lua 스크립트로는 Redis 전역 변수 선언은 불가하다. AOF 및 복제의 일관성에 영향을 미치기 때문에 redis에서도 하지 말라고 한다(`just don't do it.`)

- redis.call(command [,arg...])
Redis 명령을 호출하고 그 응답을 반환한다.
```
> EVAL "return redis.call('GET', KEYS[1])" 1 name royce
"royce"
```

- redis.pcall(command [,arg...])
`redis.call()`과 동일하게 동작하지만 런타임 예외를 던지지 않으며, 서버에서 런타임 예외를 던지는 경우 redis.error_reply를 대신 반환한다.
```
local reply = redis.pcall('ECHO', unpack(ARGV))
if reply['err'] ~= nil then
  -- Handle the error sometime, but for now just log it
  redis.log(redis.LOG_WARNING, reply['err'])
  reply['err'] = 'ERR Something is wrong, but no worries, everything is under control'
end
return reply

> EVAL "..." 0 hello world
(error) ERR Something is wrong, but no worries, everything is under control
```

- redis.error_reply(x): 오류 응답 반환
- redis.status_reply(x): 상태 응답 반환
- redis.log(level, message): 로그 레벨 설정
```
> redis.log(redis.LOG_WARNING, 'Something is terribly wrong')
[32343] 22 Mar 15:21:39 # Something is terribly wrong
```
[더 다양한 redis api](https://redis.io/docs/interact/programmability/lua-api/)

# Redis Function
EVAL로 `lua script`를 실행할 때, 매번 모든 스크립트를 전송해야 해서 네트워크 비용의 오버헤드가 발생한다. Script를 로드하여 EVALSHA를 통해 줄일 수 있지만, Redis는 해당 스크립트를 캐싱하기 때문에 flush가 호출되거나 재시작된 경우 휘발 될 수 있다. Redis에 여러 application(client)가 존재한다면 모든 application 마다 스크립트가 공유되어야 하며 관리되어야 한다는 불편함이 있다.

이러한 문제점을 해결하기 위해 Redis 7.0부터 `Function`이 도입되었다.

`Function`은 작성한 로직이 Redis 서버을 확장한다고 표현하는데, `Function`이 일종의 모듈 처럼 한 번 작성한 뒤 로드(Load)된 후 여러 client에서 동일하게 반복적으로 사용할 수 있기 때문이다.
`Function`에는 고유한 이름이 있기 때문에 호출 및 실행 하기가 쉽다.
작성한 함수는 단일 라이브러리에 속하며, 특정 라이브러리는 여러 함수가 포함될 수 있다.\
라이브러리를 Redis에 로드하려면 `FUNCTION LOAD` 명령을 통해 로드 할 수 있다.
`FUNCTION LOAD "#!lua name=mylib\n"` 를 통해 library를 로드 할 수 있고, 
```shell
#!lua name=mylib
redis.register_function(
  'knockknock',
  function() return 'Who\'s there?' end
)
```
처럼 `redis.register_function()`을 통해 Function을 등록 할 수 있다.

ex) 등록 및 호출
```shell
redis> FUNCTION LOAD "#!lua name=mylib\nredis.register_function('knockknock', function() return 'Who\\'s there?' end)"
mylib
redis> FCALL knockknock 0
"Who's there?"
```
호출 할 때는 Function에 필요한 key와 args를 명확하게 전달하여야 한다.

```shell
#!lua name=mylib
local function my_hset(keys, args)
  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end
redis.register_function('my_hset', my_hset)

redis> FCALL my_hset 1 myhash myfield "some value" another_field "another value"
(integer) 3
```
>  더 알아보기: https://redis.io/docs/interact/programmability/functions-intro/


# Transaction
Redis 트랜잭션을 통해 여러 명령을 한번에 실행할 수 있다.
- 트랜잭션내의 모든 명령은 직렬화되어 순차적으로 실행된다.
- 다른 클라이언트가 보낸 요청은 Redis 트랜잭션이 실행되는 도중에 처리되지 않는 격리성을 보장한다.

`MULTI` 를 통해서 Redis 트랜잭션을 시작하고, 내부의 명령은 항상 `OK`로 응답한다.
내부의 모든 명령은 실행되지 않고 큐에 대기시킨 후, `EXEC`가 호출되면 큐에 담긴 모든 명령을 실행 한다.
```shell
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```
대신 `DISCARD`를 호출하면 트랜잭션 큐가 `flush(삭제)`되고 트랜잭션이 종료된다.

하나의 `Transaction`내 명령들이 큐에 대기 될 때 명령이 잘 못 되었거나 서버의 메모리가 부족한 경우 오류가 발생 할 수 있다. 
Redis 서버는 명령이 누적되는 동안 오류를 감지하고, `EXEC` 중에 오류를 반환하는 트랜잭션의 실행을 거부하고 트랜잭션을 삭제합니다.

`EXEC`가 실행 된 후 실패하더라도 큐에 있는 다른 모든 명령은 처리되며, Redis는 명령 처리를 중지하지 않는다
![](https://velog.velcdn.com/images/roycewon/post/d4e95eed-fb4c-41ea-81aa-7e0a6d3e80cf/image.png)

롤백은 Redis의 단순성과 성능에 큰 영향을 미치기 때문에 지원하지 않는다고 한다.

`WATCH`를 통해 key의 동시 수정을 방지 할 수 있다.
키는 해당 키에 대한 변경을 감지하기 위해 모니터링 하게 된다.
하나의 트랙잭션에서 `EXEC` 명령 전에 `WATCH`로 설정한 키가 수정 되면 전체 트랜잭션이 중단 되고 트랜잭션이 실패하게 된다. (nil 반환)
![](https://velog.velcdn.com/images/roycewon/post/5e04612c-448a-4dcd-84c5-bbc7407470da/image.png)

# Pub/Sub
퍼블리셔(Publisher)와 구독자(Subcriber)를 분리하여 메시지를 발신 / 수신 한다.
구독자를 특정하지 않고 메시지를 발행하기 때문에 확장성이 향상 된다고 한다.

Redis의 Pub/Sub는 한번만 메세지를 전송한다. 만약 메시지를 수신할 구독자가 없거나 처리하는 과정에서 오류가 발생하면 메세지가 유실 될 수 있다. 전송 보장이 요구된다면, `Stream`을 활용하자.

간단하게 구독할 채널을 지정하고, 해당 채널로 메세지를 발행하면 전달된다.
![](https://velog.velcdn.com/images/roycewon/post/9356d1b7-d26e-4e34-bf19-fd5b0749946a/image.png)

Redis Pub/Sub 구현은 패턴 매칭을 지원하여 패턴과 일치하는 채널 이름으로도 여러 채널을 수신할 수 있다. 또, 패턴에 일치하는 메세지만 수신 할 수도 있다.
