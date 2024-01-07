# Redis Persistence
Redis는 SSD와 같은 저장소에 데이터를 저장하는 다양한 선택지를 제공한다.
이를 Redis Persistence라 하며 다음을 포함한다:
- RDB(Redis Database): RDB는 특정 인터벌의 데이터셋 스냅샷을 저장하는 방식으로 이루어진다.
- AOF(Append Only File): AOF는 서버로부터 받은 모든 write operation을 로깅하는 방식으로 이루어진다. 이 명령어들은 서버 시작시에 실행되어 데이터셋을 재구축하는데 사용될 수 있다. 명령어들이 로깅되는 방식은 레디스 프로토콜과 동일하다.
- No Persistence: 캐싱을 사용하는 경우, persistence를 하지 않을 수도 있다.
- RDB + AOF: 같이 사용할 수도 있다.

## RDB 장점
- 백업에 최적화된 방식이다.
- disaster recovory에 용이하다.
- Redis 성능을 극대화한다. 디스크 I/O와 같은 작업이 일어나지 않기 때문
- AOF에 비해 큰 데이터셋을 갖고 있을 경우의 재시작 속도가 빠르다.
- 복제본에서, RDB는 [partial resynchronizations after restarts and failovers](https://redis.io/docs/management/replication/)을 지원한다.

## RDB 단점
- 갑작스런 레디스 중지 시 데이터 손실 최소화를 원한다면, RDB는 좋은 선택지가 아니다. 보통 5분 혹은 그 이상의 주기로 스냅샷을 생성하기 때문에, 마지막 몇분의 데이터 손실은 발생할 수 밖에 없다.
- RDB는 fork()를 자주 호출하기 때문에, 데이터셋이 크다면 시간을 잡아먹을 수 있고, 데이터셋이 아주 크다면 서버가 클라이언트 상대를 적게는 millisec부터 크게는 sec단위까지 못할 수도 잇다. AOF 또한 fork()호출을 하지만 그렇게 자주 하지 않고, 튜닝 또한 가능하다.

## AOF 장점
- AOF가 더 튼튼하다. fsync 정책을 다양하게 설정할 수 있다. 하지 않거나, 매초 하거나(default), 매 명령마다 하거나. fsync는 백그라운드 쓰레드로 작동하기 때문에, 쓰기 성능에는 영향을 거의 주지 않는다. 
  - `fsync`: 파일의 내부 상태를 장치와 동기화시키는 함수(버퍼에 있는 데이터를 파일로 옮김)
- append-only log이기 때문에, seek 과정이 없고, 레디스의 갑작스런 종료 시에도 손실이 없다. 만약 특정 이유로 half-written 로그가 생긴다면, redis-check-aof-tool을 통해 쉽게 해결할 수 있다. 
- 로깅 파일이 너무 커지면, 백그라운드에서 새 파일에 다시 쓰는 작업이 이루어진다. 새 파일이 생기는 동안 레디스는 계속해서 이전 파일에 쓰기 작업을 실행하고, 새 파일에 작성하는 명령어는 현재 데이터셋을 구축하기 위한 최소의 명령어이다. 이 작업이 완료되면 레디스는 새 파일에 쓰기 시작한다.
- AOF는 모든 명령어를 하나하나 이해하기 쉽고 파싱하기 쉬운 방식으로 로깅한다. AOF 파일을 쉽게 추출할수도 있다. 실수로 `flushall` 키워드를 통해 모든 데이터를 flush했더라도, 서버를 잠시 중지하고, 마지막 커맨드를 제거하고, 서버를 재시작하면 레디스를 쉽게 복구할 수 있다.

## AOF 단점
- 같은 크기의 데이터셋일 때, AOF 파일이 보통 RDB 파일보다 크기가 크다
- 일반적으로 RDB보다 AOF가 속도가 느리다.
- Redis 7.0 이하일 경우
	- rewrite중 database에 write가 실행될 경우 많은 메모리를 사용할 수 있다.
    - rewrite중 database에 write하는 경우 두번씩 write된다.
    - 레디스는 write를 멈추고 rewrite중인 새 파일 끝에 해당 커맨드를 추가할 수 있다.
    
## So, what should I use?
- PostgreSQL이 제공하는 만큼의 데이터 safety를 필요로 한다면 둘다 사용한다.
- 데이터가 중요하긴 하지만, 몇분정도의 데이터 손실은 상관 없는 경우, RDB를 사용한다.
- AOF만 사용하는 유저도 많지만, RDB를 사용하는 것이 백업 측면이나, 빠른 재시작 관점이나, AOF 엔진 버그가 발생했을 시 큰 도움을 주므로, AOF만 사용하는 것은 권장하지 않는다.

## Snapshotting
- 기본적으로 레디스는 데이터셋 스냅샷을 디스크에 dump.rdb라는 이름으로 저장한다. n초마다 최소 m개의 변화가 생겼을 시 스냅샷을 저장하도록 설정할 수 있다.

```
save n m
```

### How it works
레디스가 데이터셋을 디스크에 저장할 때, 다음 과정이 이뤄진다.
- 레디스가 fork한다.
  - `fork`: 자식 프로세스를 생성하는 명령어
- 자식 프로세스가 데이터셋을 임시 RDB 파일에 작성한다.
- 자식 프로세스가 새 RDB 파일 작성을 완료하면, 기존 파일을 대체한다.

## Append-only file
- configuration 파일에서 다음을 설정해 AOF를 enable할 수 있다.
```
appendonly yes
```
- Redis 7.0부터, 레디스는 multi part AOF 매커니즘을 사용한다. 기존 하나의 AOF 파일이 아닌, 베이스 파일과 변경점 파일로 나뉘어 베이스 파일은 AOF rewritten 당시의 데이터셋 스냅샷, 변경점 파일은 생성한 이후의 변경점을 저장하는 방식이다.

## Log Rewritting
- 앞서 말했듯, 파일 크기를 줄이기 위해 rewrite가 진행될 수 있다.
- Redis 2.2라면 `BGREWRITEAOF` 명령어를 통해 백그라운드 rewrite를 실행 가능핟 .
- Redis 2.4 이상이라면 rewrite를 자동으로 실행하게 할 수 있다.
- Redis 7.0부터는 rewrite 스케쥴 시 부모 프로세스가 새 변경점 파일을 만들어 write를 지속하고, 자식 프로세스는 새 베이스 파일을 만든다. 너무 많은 변경점 파일이 만들어지는 것을 막기 위해, 실패한 AOF rewrite가 재시도될 때 더 천천히 실행되도록 한다.

## How durable is the AOF?
fsync 옵션에 따라 비교해보자.
- `appendfsync always`: 매우 느리지만 매우 안전하다. 매 명령마다 실행된다.
- `appendfsync everysec`: 충분히 빠르다. disaster시 1초 이내의 데이터 손실 가능성이 있다.
- `appendfsync no`: 빠르지만, 안전하지는 않다.

권장되는 방식은 매초 저장하는 방식이다.

## Interactions between AOF amd RDB persistence
- Redis >= 2.4
- 스냅샷을 뜨는 중 rewrite 요청이 들어오면, 먼저 OK 응답을 준 후, 스냅샷이 완료되면 실행된다.
- AOF와 RDB가 모두 활성화된 상태에서 restart하면 AOF 파일이 사용된다. 더 높은 완성도를 보장하기 때문이다.
