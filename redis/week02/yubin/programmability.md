# 레디스 Programmability

# Redis Functions

- redis 7 이상
- 왜 필요한데?
    - 이전 버전의 레디스는 `EVAL` 커맨드를 통해서만 스크립팅이 가능했음
        - eval script를 사용하려면 응용프로그램이 매번 실행을 위한 전체 스크립트를 전송해야함
            - 이 과정에서 네트워크/컴파일 오버헤드가 발생하므로 레디스는 최적화를 제공했음
        - 설계상 레디스는 로드된 스크립트만 캐싱함
            - 그러나 이 캐시는 서버 재시작등의 이유로 언제든지 손실될 수 있음
    - 인스턴스들은 모든 스크립트의 복사본을 유지해야함
    - 트랜잭션 컨텍스트 내에서 캐시된 스크립트를 호출하면 스크립트가 누락되어 트랜잭션이 실패할 확률이 높아짐
- 모든 함수의 실행은 원자단위
    - 함수를 실행하는 동안 모든 서버 활동을 차단
    - 스크립트의 효과가 아직 발생하지 않았거나/이미 발생

### Loading Libraries and functions

- 모든 라이브러리가 성공적으로 로드되려면 등록된 function을 하나 이상 포함해야함
- 등록된 function은 라이브러리의 진입점 역할을 함
- Lua 엔진은 로드시 라이브러리 소스코드를 컴파일하며, `redis.register_function()` 을 통해 function이 등록될거라고 기대함

### Input keys and regular arguments

```
#!lua name=mylib

local function my_hset(keys, args)
  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end

redis.register_function('my_hset', my_hset)
```

- 위와같은 식으로 function 구현 가능

```
$ cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE
```

- 위와 같은 방식으로 로드 가능

```
redis> FCALL my_hset 1 myhash myfield "some value" another_field "another value"
(integer) 3
```

- 위와같은 방식으로 호출 가능

# Lua Scripting

- lua
    - 대소문자 구분함

        ```
        127.0.0.1:6379> eval "return argv[1]" 0 Hello
        (error) ERR user_script:1: Script attempted to access nonexistent global variable 'argv' script: 254c4615d879dea0f348af879efaa4e87416e00b, on @user_script:1.
        127.0.0.1:6379> eval "return ARGV[1]" 0 Hello
        "Hello"
        ```

    - 자바처럼 가비지 컬렉션을 제공하므로, 사용하지 않는 변수를 제거하기위해 별도의 처리가 필요하지 않음
    - 변수형을 선언하지 않음 (입력된 값에 의해 변수형이 자동지정)
    - 배열의 인덱스는 1부터 시작
- Redis + Lua
    - `eval` 명령어를 사용해 수행하고자하는 스크립트를 레디스로 전송 (일회성)
    - 루아스크립트를 `script load` 명령을 통해 레디스 서버에 등록시킨 후 사용 가능
    - 명령은 원자적으로 처리됨

### Script Parameterization

- 키 이름인 입력인수 / 그렇지 않은 인수
    - Redis의 키 이름 : 데이터베이스의 키
    - 키 이름이 아닌 인수 : 정규 입력 인수
- 정규 입력 인수만 존재할 경우
    - `EVAL "return redis.call('set', 'key', 'value')" 0`
        - 이때 0은 인수로 입력할 키의 갯수
        - 없으면 0
- 키 이름인 입력인수가 존재할 경우

    ```
    redis> EVAL "return { KEYS[1], KEYS[2], ARGV[1], ARGV[2], ARGV[3] }" 2 key1 key2 arg1 arg2 arg3
    1) "key1"
    2) "key2"
    3) "arg1"
    4) "arg2"
    5) "arg3"
    ```

    - 키 이름인 입력인수의 갯수를 지정

### script로 redis와 interacting

- `redis.call()` 혹은 `redis.pcall()` 을 통해 레디스 커맨드를 호출가능

### Script Cache

- 레디스는 스크립트 캐싱을 제공
- `EVAL` 로 실행하는 스크립트는 서버의 전용 캐시에 저장됨
- 동적으로 생성되는 스크립트는 안티패턴임
    - 호스트 메모리 리소스 낭비

```
redis> SCRIPT LOAD "return 'Immabe a cached script'"
"c664a3bf70bd1d45c4284ffebb65a6f2299bfc9f"

redis> EVALSHA c664a3bf70bd1d45c4284ffebb65a6f2299bfc9f 0
"Immabe a cached script"
```

### Cache Volatility

- redis 스크립트 캐시는 휘발적임
- 서버는 script의 sha1 digest가 없는 경우 오류를 반환함
    - `(error) NOSCRIPT No matching script`
- 이럴경우 먼저 해당 스크립트를 로드한 다음 다시 호출하여 캐시된 스크립트를 SHA1 합계로 실행해야 함

### Script cache semantics(의미론)

- 일반적으로 애플리케이션의 스크립트는 캐시에 무기한으로 존재할것
- 스크립트 캐시를 flush하기위해서는 `SCRIPT FLUSH` 명령어를 명시적으로 호출해야함

### Lua Script 활용

- `keys myset*` : 키 리스트 조회

    ```
    127.0.0.1:6379> set myset1 myset1
    OK
    127.0.0.1:6379> set myset2 myset2
    OK
    127.0.0.1:6379> eval "return redis.call('keys', 'myset*')" 0
    1) "myset2"
    2) "myset1"
    ```

- 개수보기 : 앞에 #을 붙임

    ```
    127.0.0.1:6379> eval "return #redis.call('keys', 'myset*')" 0
    (integer) 2
    ```


# Lua API

### Global variable and functions

- 모든 변수/함수 정의를 로컬로 선언해야함 (`local`)

    ```
    local my_local_variable = 'some value'
    
    local function my_local_function()
      -- Do something else, but equally amazing
    end
    ```

    - 스크립트와 함수가 Redis에 저장된 데이터 이외의 런타임 컨텍스트를 유지하지 못하도록 전역 변수를 차단
    - 실행 사이에 문맥을 유지할 필요가 있는 이용사례에 있어서, 레디스의 키스페이스에 컨텍스트를 저장해야함
- `redis.call()`
    - redis command를 호출하고 응답을 반환
- `redis.pcall()`
    - redis 서버에서 발생하는 런타임 오류를 처리할 수 있음
- 이외에 `redis.error_reply`, `redis.status_reply` 등이 있음