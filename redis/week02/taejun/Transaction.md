# [3] Redis Transaction

- 명령의 직렬화 및 순차적인 실행을 보장한다.
- 다른 클라이언트의 요청은 실행중간에 처리되지 않으며, 격리된 작업으로 수행한다.
- OptimisitcLock을 제공한다.
    - Redis2.2, CAS매커니즘으로 동작한다.
    - 트랜잭션 중 데이터 변경이 감지되면 트랜잭션을 중지 시킬 수 있다.
- 전통적인 RDB 트랜잭션의 개념과는 다르다.
    - Client레벨에서, Atomic한 단위의 순차적인 실행을 보장한다.
    - 해당 Client에 국한되기 때문에, 다른 Client와의 RaceCondition이 발생될 수 있다.
    - 실패시에 Rollback을 하지 않는다.
- 트랜잭션 안에서의 검증은 실제 명령어가 수행될 때 일어난다.
    - 모든 검증은 EXEC가 호출될 때 발생한다.
- Transaction 중간에 명령이 실패할 수 있다.
    - Command Syntax오류
    - DataType 불일치
    - Redis Server의 Memory 및 리소스 부족
    - WATCH Command와의 충돌
- 트랜잭션이 중간에 실패해도 나머지 명령이 계속 수행된다.

## [1] Cluster환경에서의 Transaction

- **단일 노드 트랜잭션**
    - 트랜잭션은 단일 노드에서만 실행될 수 있다.
    - 트랜잭션에 포함된 모든 키는 같은 노드에 존재해야 한다.
    - 여러 노드에 걸쳐 있는 데이터를 동시에 변경하는 것이 불가능하다.
- **{Hash Tag} 사용**
    - 여러 키가 같은 노드에 위치하도록 하기 위해서는 '해시 태그'를 사용할 수 있다.
    - 중괄호 {} 안에 있는 문자열을 기반으로 키가 같은 슬롯에 할당되도록 한다.
        - ex) key1{tag}와 key2{tag} 는 동일한 {tag} 를 가지므로 같은 노드에 위치하게 된다.

### 발생할 수 있는 문제점

- **크로스 슬롯 오류**:
    - 여러 해시 슬롯에 걸쳐 있는 키들을 사용하려고 하면 '크로스 슬롯' 오류가 발생한다.
    - 클러스터 환경에서 한 트랜잭션 내에서 여러 노드의 데이터를 동시에 조작할 수 없음을 의미합니다.
- **네트워크 지연과 일관성**
    - 네트워크 지연과 노드 간의 커뮤니케이션이 트랜잭션의 성능과 일관성에 영향을 미칠 수 있다.
    - 클러스터 환경에서 트랜잭션을 사용할 때는 이러한 요소들을 고려해야 한다.

## [2] Command

### [1] MULTI & EXEC

- 트랜잭션을 시작하는 명령어이다.
- Syntax오류가 있을 경우, Error를 리턴하며 이전까지의 트랜잭션을 버린다.
- Queue에 담긴 순서대로 명령이 수행된다.
- Atomic & Blocking이 발생하는 것은 해당 Client 한정이다.
    - MULTI가 입력되고 Queue에 Command가 적재되는 중간에도, 다른 Client는 Command를 수행할 수 있다.

```bash
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

### [2] DISCARD

- 트랜잭션을 취소하고 모든 명령어를 삭제한다.
- 다시 일반적인 모드로 변경한다.

```bash
> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
```

### [3] WATCH

- WATCH를 통해서, 감시할 대상을 선정한다.
    - CAS (Compare And Set)을 통한 Optimistic Lock을 제공한다
    - 변경이 감지되면 Transaction이 중단되고, NULL을 리턴함으로 하여 Transaction 실패를 알린다.
- DISCARD 또는 EXEC 이후 UNWATCH 상태로 변경된다.
- UNWATCH 명령어를 통해서 명시적으로 상태를 변경 할 수 있다.

```bash
> WATCH foo # Transaction용 WATCH의 시작
OK

> incrby foo 3 # Transaction 외부에서 WATCH 대상의 value 변경
(integer) 3

> MULTI # Transaction 시작
OK

> incrby foo 100 #Transaction 내부 명령어
QUEUED

> EXEC # 실행
(nli) # Transaction 실패 (watch를 시작한 시점 이후에 해당 Key에 해당하는 Value가 변경되었기 때문이다)

```