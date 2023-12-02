## Programmability
- Redis는 서버 자체에서 사용가능한 프로그래밍 인터페이스를 제공한다.
  - v7: Redis Functions
  - ~v6.2: Lua scripting

## Lua scripting
- Redis 서버에 내장된 Lua 인터프리터에서 EVAL 명령어(또는 EVALSHA 명령어)를 이용하여 임의의 명령어를 조합한 처리를 수행하는 기능
- 여러 가지 명령어를 조합할 수 있을 뿐 아니라, 스크립트 자체가 하나의 큰 명령어로 해석되기 때문에 스크립트가 atomic하게 처리된다.

### EVALSHA
- 스크립트가 길어질 경우, 스크립트 전체를 매번 EVAL 명령어로 전송하기에는 네트워크 대역이 아까울 수 있다.
- SCRIPT LOAD 명령어로 스크립트를 서버 측에 캐싱한 다음, 캐싱 반환 값으로 받은 키를 이용하여 EVALSHA 명령어로 스크립트를 실행할 수 있다.
> - cluster 환경에서는 스크립트를 클러스터 내의 모든 노드에 캐싱해야 한다.
> - 캐시된 스크립트의 신뢰성 문제로, 트랜잭션이 실패할 가능성이 존재한다.
> - 스크립트 간 코드 공유가 어렵고, 그 결과 중복된 코드를 작성하거나 클라이언트 측에서의 복잡한 전처리 작업을 수행해야 한다.

### Redis Function
- Redis 7에 도입되어 Lua 스크립팅을 발전시킨 형태
- 스크립트와 달리 .rdb, .aof 파일뿐만 아니라 모든 replica에 복제된다.
- 생성한 function은 라이브러리에 속하게 되며, 코드 공유도 가능하다.

```bash
127.0.0.1:6379> FUNCTION LOAD "#!lua name=mylib\nredis.register_function('knockknock', function() return 'Who\\'s there?' end)"
"mylib"

127.0.0.1:6379> FCALL knockknock 0
"Who's there?"
```
---

```
#!lua name=mylib

local function my_hset(keys, args)
  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end

redis.register_function('my_hset', my_hset)
```

```
$ cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE
```

```
redis-cli> FCALL my_hset 1 myhash myfield "some value" another_field "another value"
(integer) 3

redis-cli> HGETALL myhash
1) "_last_modified_"
2) "1640772721"
3) "myfield"
4) "some value"
5) "another_field"
6) "another value"
```
