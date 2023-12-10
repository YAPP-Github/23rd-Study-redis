# [1] Lua Script

- 대부분의 명령어를 사용 가능하다.
- Server내부에서 실행되기 때문에, Server내부의 데이터에 접근하여 Read / Write하는데 매우 적합하다.
- Rollback이 없다. (즉, Script중간에 Exception이 발생해도 그전까지의 쓰기는 유지된다)

    ```bash
    > EVAL "redis.call('SET', 'key3', 'value3'); assert(false, '의도적인 에러 발생'); redis.call('SET', 'key2', 'value2');" 0
    "ERR Error running script: @user_script:1: 의도적인 에러 발생 stack traceback:  [G]: in function 'assert'  @user_script:1: in main chunk  [G]: ?"
    
    > get key1
    value1
    ```

- Blocking되는 Command (기본TimeOut 5초)

## 환경

```
Sandboxed LuaScript

샌드박스 환경은 스크립트가 서버의 기타 시스템 자원에 접근하는 것을 제한하여 잠재적인 오용을 방지하고 보안 위험을 줄인다.

1. 시스템 자원 접근 제한 
   - Lua 스크립트는 Redis 서버의 호스트 시스템에 접근해서는 안된다. 
   - 파일 시스템, 네트워크, 그리고 Redis API가 지원하지 않는 기타 모든 시스템 호출을 시도하는 것을 포함한다.
   - 모든 변수는 local이여야하며, 전역변수 선언시 차단된다.

2. 데이터 처리 제한
   - 스크립트는 오직 Redis 데이터베이스에 저장된 데이터와 스크립트 실행 시 제공되는 인자들에 대해서만 작업해야 한다. 
   - 스크립트가 Redis 서버의 내부 데이터에만 영향을 미치고, 외부 시스템이나 서비스에 영향을 주지 않도록 함을 의미한다. 
   - Lua모듈을 가져오는 것(require)은 제한된다.
```

## Lua가 제공해줄 수 있는 이점

- 지역성을 제공하기 떄문에, 전체 대기 시간을 줄이고 네트워킹 리소스를 절약한다.
- Script 단위의 원자성을 제공한다.
- Redis에 포함되기에는 너무 틈새적인 간단한 기능의 구성을 가능하게 한다.

## 문법

### 1. EVAL

```
redis> EVAL "return { KEYS[1], KEYS[2], ARGV[1], ARGV[2], ARGV[3] }" 2 key1 key2 arg1 arg2 arg3
1) "key1"
2) "key2"
3) "arg1"
4) "arg2"
5) "arg3"
```

KEYS와 ARGV 변수를 넘기기 이전에 존재하는 숫자는 KEYS배열의 갯수를 의미한다.

### KEY vs ARGV

둘다 Index는 1부터 시작

- **KEYS**: Script의 안정성과 효율성을 보장하기 위해 스크립트가 작업을 수행할 키는 Keys에 포함되어야 한다. Redis 클러스터 환경에서는 Keys를 통해서 스크립트가 실행될 노드를 결정한다.
  따라서, 스크립트가 작업할 모든 키는 Keys배열에 명시적으로 지정되어야 한다.
  동적으로 생성(프로그래밍적)되는 Key가 제공되어서는 안된다.
- **ARGV**:  값이나 명령에 대한 매개변수 등을 스크립트에 전달하는 데 사용된다.  ARGV를 통해 전달된 값은 스크립트 내에서 데이터 처리나 계산에 사용될 수 있다.

## Script내에서의 Command

- BLPOP, BRPOP, BZPOPMIN, BZPOPMAX같은 Blocking 명령은 사용할 수 없다.
- PUBLISH 명령은 사용가능하나, SUBSCRIBE 명령은 사용할 수 없다.
- EVAL, SCRIPT, MULTI, EXEC 명령은 사용할 수 없다.
- 관리(admin) 명령 사용 불가: SAVE, BGSAVE, BEREWRITEAOF, LATENCY, SLOWLOG, REPLICAOF, DEBUG, ROLE, AUTH, SHUTDOWN, ...

****Script parameterization (안티패턴)****

- 두개의 Parameter를 제공한 것이다.
    - parameter1: LuaScript
    - parameter2: 3번째 Parameter부터 끝 Parameter까지 존재하는 Parameter의 수에 대한 정의
- 권장되지 않는 이유
    1. **성능 저하**: Redis는 LuaScript를 캐시하여 재사용.  동적으로 생성된 스크립트는 매번 다르게 인식되어 캐시 효율이 떨어지고, 이는 성능 저하로 이어진다. 반면에 미리 정의된 스크립트는 한 번 캐시되면 재사용이 가능하여 더 효율적이다.
    2. **보안 문제**: 동적 스크립트는 예측 불가능하고 검증이 어려울 수 있다. 악의적인 사용자가 공격 코드를 스크립트에 삽입할 위험이 있으며, 이는 보안 취약점을 야기할 수 있다.
    3. **유지보수의 어려움**: 동적으로 생성된 스크립트는 재사용성, 가독성이 낮다. 코드가 분산되어 있거나 문서화되지 않은 경우, 시스템의 복잡성이 증가하고 유지보수가 어려워진다.

### 사용예시 (Redisson)

- tryLock
    - <img width="998" alt="스크린샷 2023-11-26 오후 3 29 36" src="https://github.com/ktj1997/TIL/assets/57896918/d8dc99d5-282e-40de-8542-c691a4a4de3a">
    - Hash 자료구조 사용
          - LockKey가 존재하지 않음(최초진입)  OR Hash내에 Value가 존재 (Reentrant) 할 때만
              - hincrby → Reentrant Count 증가
              - Expire지정
              - Lock을 정상적으로 설정했다면 nil리리턴
                  - 보통 nil은 조건에 만족하는 요소가 없을 경우를 나타내지만, 구현별로 조건분기를 위해서 사용할 때도 있다.
    - Lock을 얻지못했다면, Lock의 남은 만료시간을 조회

- unLock
  - <img width="565" alt="스크린샷 2023-11-26 오후 4 21 35" src="https://github.com/ktj1997/TIL/assets/57896918/dce48450-9cd5-4bf5-bfcd-d2733db19144">
  - key가 존재하지 않거나, hash에도 존재하지 않으면 그대로 리턴
      - LockCount—
      - LockCount가 0보다 크다면,
          - 다시 기본 expire로 설정
          - set (MetaData성? 의미모르겠음)
  - LockCount가 0이라면
      - key 삭제
      - UnLock Message Publish
      - set (MetaData성? 의미모르겠음)

### EVAL vs EVALSHA vs SCRIPT LOAD

1. EVAL
    - Script전체를 받아온다.
    - 내부적으로 Text를 sha1로 해싱해본다.
        - 나온 해시값을 통해서 캐시에 접근해서 있다면 캐시를 실행
        - 없다면 캐시에 저장
    - 내부적으로 캐싱하지만 해시값을 리턴해주지는 않는다. (오로지 내부해시목적)
    - Redis 서버에 Lua 스크립트를 실행하도록 지시하는 명령어
    - Lua 스크립트를 실행하고, 그 결과를 Redis 클라이언트로 반환
2. EVALSHA
    - SHA1을 이용한 해시를 참조하여 실행한다.
    - 최소 한번 실행됐었어야한다. (캐시에 올라가 있었어야한다)
    - 바로 캐시에 접근하기 때문에 (해싱하는 과정이 생략되기 때문에) 성능상으로 유리하다.

    ```bash
    > EVALSHA d8f2fad9f8e86a53d2a6ebd960b33c4972cacc3 1 foo bar
    (error) NOSCRIPT No matching script
    
    > EVALSHA d8f2fad9f8e86a53d2a6ebd960b33c4972cacc37 1 foo bar
    "OK"
    ```

3. SCRIPT LOAD
    - Redis Server에 Script를 올리고 캐싱한 후 Hash값을 리턴받는다.
    - 이후 EVAL SHA를 통해서 호출하면된다.

    ```bash
    > SCRIPT LOAD "return redis.call('SET', KEYS[1], ARGV[1])"
    "d8f2fad9f8e86a53d2a6ebd960b33c4972cacc37"
    ```


### redis. call vs redis.pcall

1. redis.call
    - Lua 스크립트 내에서 Redis 명령어를 실행하는 함수
    - redis.call을 통해서 Redis의 데이터를 읽거나 수정할 수 있습니다.
    - 오류가 발생하면 Script 실행을 중단
2. redis.pcall
    - 오류 처리를 하기위해서 사용한다.
    - 일종의 try-catch

### Script Cache

- Redis 인스턴스가 살아있는 동안 유지된다.
    - Redis 캐시는 항상 휘발성이다. (영속적이지 않다)
    - 캐시의 내용이 언제든지 휘발될 수 있음을 명심해야 한다.
    - 인스턴스 다운 OR SCRIPT FLUSH
    - ARGV와 KEY의 변동은 캐싱의 고려 사항이 아니다. (텍스트 자체가 캐싱되는 것)
- EVAL의 경우, Script와 인자가 모두 전달된다.
- 네트워크 대역폭이 늘어날 뿐만 아니라, Redis에도 오버헤드가 발생한다.

## Script Command

### [1] SCRIPT FLUSH

- Cache에서 Script 명시적제거
- Script 변경이 발생해서 기존 캐시를 제거할 때 유용하다.

### [2] SCRIPT EXISTS

- 스크립트의 해시값 (sha key)를 input으로 받고 존재여부에 따라서 1, 0을 리턴한다.
- 0의 경우, 캐시에 올라가있지 않은 것을 의미한다.

### [3] SCRIT LOAD

- Script를 캐시에 로드한다.
- EVALSHA가 실패하지 않게한다.

### [4] SCRIPT KILL

- 오래실행되는 Script를 명시적으로 KILL할 수 있다.
- read-only 스크립트에만 사용 가능하다.
    - 내부적으로 데이터를 변경하는 스크립트의 경우 사용할 수 없다.
    - 정합성이 깨질 수 있기 때문이다.

### [5] SCRIPT DEBUG

- LuaScript를 디버깅하는 목적으로 사용된다.

### [6] SCRIPT INFO

- Script관련된 정보를 조회한다.
    - Lua Interpereter가 사용하는 메모리
    - Script Sha1 hash
    - Cache 상태
    - …