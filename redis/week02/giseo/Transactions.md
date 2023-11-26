## Transaction
- 데이터베이스의 상태를 변환시키는 하나의 논리적 기능을 수행하기 위한 작업의 단위 또는 한꺼번에 모두 수행되어야 할 일련의 연산
- All Or Nothing

## Redis Transaction
- 트랜잭션을 유지하기 위해서는 순차성을 가져야 하고 도중에 명령어가 치고 들어오지 못하게 Lock이 필요하다.
- MULTI, EXEC, DISCARD 그리고 WATCH

### MULTI
- Redis의 트랜잭션을 시작하는 커맨드
- 트랜잭션을 시작하면 이후 입력되는 커맨드는 바로 실행되지 않고 Queue에 쌓인다.

### EXEC
- Queue에 쌓인 명령어를 일괄 실행한다.

### DISCARD
- Queue에 쌓인 명령어를 일괄 폐기한다.

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET chicken bbq
QUEUED
127.0.0.1:6379(TX)> set chicken bhc
QUEUED
127.0.0.1:6379(TX)> set chicken hosik
QUEUED
127.0.0.1:6379(TX)> get chicken
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) OK
3) OK
4) "hosik"
```

```
127.0.0.1:6379> get chicken
"hosik"

127.0.0.1:6379> MULTI
OK

127.0.0.1:6379(TX)> set chicken kyochon
QUEUED

127.0.0.1:6379(TX)> get chicken
QUEUED

127.0.0.1:6379(TX)> discard
OK

127.0.0.1:6379> get chicken
"hosik"
```

```
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET mySet 100

QUEUED
127.0.0.1:6379(TX)> HSET mySet chicken hosik
QUEUED

127.0.0.1:6379(TX)> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value

127.0.0.1:6379> get mySet
"100"
```
- 잘못된 자료구조의 명령어를 사용할 경우 해당 커맨드만 Rollback한다.

---

### WATCH / UNWATCH
- Redis에서 낙관적 Lock을 거는 명령어
- 이후 UNWATCH가 실행되기 전에 1번의 EXEC나 Transaction이 아닌 다른 커맨드만 허용한다.
- EXEC을 실행하면 묵시적으로 UNWATCH가 실행된다.

```
127.0.0.1:6379> watch chicken
OK
127.0.0.1:6379> set chicken kyochon   //트랜잭션 외부
OK
127.0.0.1:6379> MULTI           // 트랜잭션 시작
OK
127.0.0.1:6379(TX)> set chicken bbq
QUEUED
127.0.0.1:6379(TX)> exec       // 트랜잭션 종료, unwatch
(nil)
```
