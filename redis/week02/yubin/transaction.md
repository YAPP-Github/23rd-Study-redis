# 레디스 Transaction

# Transaction

- 하나의 단계에서 여러 명령어 그룹을 실행할 수 있음
- 두가지 보증
    - 트랜잭션의 모든 명령은 직렬화되어 실행됨
        - 다른 클라이언트가 보낸 요청은 레디스 트랜잭션 실행 도중에 절대 serving되지 않음
    - `EXEC` 커맨드는 트랜잭션의 모든 command 실행을 트리거하므로, 클라이언트가 EXEC 커맨드를 호출하기 전에 서버와 연결이 끊기면 operation은 단 하나도 수행되지 않음
        - 반대로 exec커맨드가 호출되었으면 모든 operation이 수행됨

# Usage

- `MULTI` 커맨드로 트랜잭션 시작(?)
    - 이때 사용자는 여러 명령을 시도할 수 있고, 레디스는 이 여러개를 실행하는대신 대기열에 넣어놓고 `EXEC`가 호출되면 실행됨
- `DISCARD` 로 트랜잭션 대기열 플러시시키고 종료 가능

```
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

- foo와 bar를 원자적으로 증가시킴

# 트랜잭션 내부의 에러

- 에러 발생 시나리오
    - 명령이 대기열에 넣어지는 과정에서 실패하여 exec가 호출되기전에 에러가 발생하는 경우
        - ex) 명령어가 구문적으로 오류, …
    - exec가 호출되고 난 후 실패
        - ex) 잘못된 값을 가진 키에 대한 작업
- 레디스 2.6.5부터 서버는 명령어를 누적하는 동안 오류를 감지하고, exec 명령어를 수행하는 동안 에러를 반환하면서 트랜잭션 실행을 거부할것
- 명령이 실패해도 대기열의 다른 모든 명령은 처리됨 (명령을 중지하지 않음)

# 롤백은 없다

- 레디스의 단순함과 성능에 해가 가기때문에 지원하지 않는다

# Check-and-Set으로 낙관적 락

- `WATCH` 는 레디스 트랜잭션에게 cas동작을 제공할 때 사용
- watched된 키는 변경사항을 감지하기 위해 모니터링됨
    - exec 커맨드 전에 적어도 하나의 watched key가 수정된다면 모든 트랜잭션이 중단되고 null reply를 반환하여 트랜잭션 실패를 알릴것
- 아래 command는 클라이언트가 1개일때만 정상동작

    ```
    val = GET mykey
    val = val + 1
    SET mykey $val
    ```

    - 만약 여러 클라이언트가 저걸 수행하려고하면 race condition이 될것
- WATCH를 사용하여 문제해결

    ```
    WATCH mykey
    val = GET mykey
    val = val + 1
    MULTI
    SET mykey $val
    EXEC
    ```

    - WATCH와 EXEC 호출 사이에 다른 클라이언트가 val을 수정했다면 race condition이 감지되어 트랜잭션이 실패할것