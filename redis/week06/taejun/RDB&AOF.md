# Redis 영속성
- 둘 중 하나만 사용하는 것도 가능하다.
- Cache 용도로 사용하는경우, 두가지 다 사용하지 않을 수도 있다.
- RDB + AOF를 같이 사용할 수도 있다.
    - RDB와 AOF가 모두 존재할 때, AOF파일의 복구 우선순위가 높다. (더 신뢰성이 높다고 판단한다.)
- Redis가 복구에 필요한 Data File을 읽어오는 시점은 시작 시점 밖에 없다.
    - 즉, 두가지 복구옵션 모두 Redis Server 재시작 시점에만 가능하다.

## [1] RDB (Redis DataBase)
- 일반적으로 dump.rdb라는 파일에 저장된다.
    - binary형식이며, 사람이 읽을 수 있는 형태가 아니다.

### 장점
- 복구가 빠르다.
- 특정시점의 Snapshot으로, BackUp을 하기에 유리하다.
    - Data의 Snapshot이다. (Command의 Snapshot 아님)
- Disk에 저장하기 때문에, 충분한 공간을 확보해야 한다.
    - 별도의 분산스토리지(S3, ...)에 저장하는 것도 가능하다.
- 장애 복구, 초기 데이터, Replica 수행에서 Snapshot 파일을 다시 사용하면 되기 때문에 복구가 빠르다.

### 단점
- RDB는 Snapshot을 찍는 주기를 가지기 떄문에, 유실이 발생할 수 있다.
    - 다음 Snapshot을 준비하는 과정에서 장애가 발생하면, 데이터 유실이 발생 할 수 있다.
    - 일반적으로 Snapshot은 5분단위로 생성된다.
- RDB의 수행은 자식 Process가 수행하며, fork()비용이 높다.
    - Snapshot이 클 경우, 병목 현상이 발생 할 수 있다.

## RDB 설정
- redis.conf 파일에 저장되어있다.
- 기본 설정은 다음과 같다.
    - ```shell
        # 여기에 있는 save는 Command의 Save (동기 방식의 수동 RDB)와 다르다.
        save 900 1       # 900초(15분) 동안 1개 이상의 키가 변경되었을 경우
        save 300 10      # 300초(5분) 동안 10개 이상의 키가 변경되었을 경우
        save 60 10000    # 60초(1분) 동안 10,000개 이상의 키가 변경되었을 경우
        save ""          # RDB옵션 비활성화 / CONFIG SET으로 지정 가능하다.
    ```
    - rdbcompression (yes)
        - RDB 파일을 저장하기 전에 데이터 압축을 사용
        - 기본적으로 LZF 압축 알고리즘을 사용
    - rdbchecksum (yes)
        - RDB 파일 저장 시 체크섬을 사용하여 데이터 무결성을 검사
    - dbfilename (dump.rdb)
        - RDB 파일의 이름
    - dir (./)
        - RDB 파일과 AOF 파일이 저장될 디렉토리를 지정
        - 기본적으로 현재 디렉토리에 저장

## 수동 RDB Command

### [1] SAVE
- 동기 방식이다.
```text
[1] 부모 Process가 임시 RDB파일에 DataSet을 Write 한다.
[2] 부모 Process의 Write가 끝나면, 이전 RDB파일을 대체한다.
```

### [2] BGSAVE
- 비동기 방식이다.
```text
[1] Redis 부모 Process가 fork()를 통해서 자식 Process를 생성한다.
[2] 자식 Process는 임시 RDB파일에 DataSet을 Write한다.
[3] 자식 Process의 Write가 끝나면, 이전 RDB파일을 대체한다.
```
- SAVE는 동기방식이다.
    - 다른 Client의 요청을 차단한다.

### Copy-On-Write
- Linux(Unix)에서, 부모가 자식 Process를 fork()할 경우, 같은 메모리 공간을 공유한다.
- 이 때, 부모 Process의 데이터 수정 (CUD)가 발생하면, 메모리의 공유가 불가능해진다.
- 이 때, 자식 Process를 위한 Memory Page를 복사해야하기 떄문에, 부하가 발생한다.

## [2] AOF
- 실행된 Write Command가 Immutable하게 append된 형태이다.
    - RESP형태로 저장된다.
    - **데이터가 변경된 Command만 저장된다.**
    - BRPOP과 같은 Blocking Command는 RPOP과 같은 NonBlocking Command로 변경된다
        - AOF파일에 Blocking을 굳이 명시할 필요가 없기 때문이다.
- 시간이 지남에 따라 AOF 파일의 크기가 커질 수 있다.
    - 주기적으로 파일 재작성 (bgrewriteaof)프로세스를 수행한다.
- 우선적으로 AOF Buffer에 명령어들이 기록된다.
    - 옵션에 따라서 fsync를 수행하며 Buffer를 비운다.

### 설정
```shell
appendonly yes                        # AOF 설정 Enable
appendfilename "appendonly.aof"       # appendonly.aof (default)
appenddirname "appendonlydir"         # aof파일 저장 디렉토리
appendfsync   everysync               # 매초 저장
aof-use-rdb-preamble yes              # AOF Rewrite 시, AOF파일의 포맷을 rdb와 같은 포맷으로 가져감
```
- appendonly: AOF 설정 Enable 여부
- no-appendfsync-on-rewrite: Rewrite설정 여부
- aof-load-truncated: AOF파일 손상 시 로드 여부
    - yes: 손상된 부분을 무시하고 다른 부분 로드
    - no: 손상된 부분이 발견되면 로드를 중지
- appendfsync: AOF에 기록하는 시점을 나타낸다.
    - always: 명령 실행마다 수행
    - everysync: 1초마다 AOF에 기록한다.
    - no: AOF가 기록하는 시점을 OS가 결정한다.
- aof-use-rdb-preamble: AOF Rewrite시의 파일 포맷
    - yes: RDB포맷도 함꼐 사용된다.
        - RDB 포맷
        - AOF 포맷
    - no: 순수 AOF파일 포맷 (Rewrite는 그대로 실행, 포맷만 AOF 형식)

### 장점
- 원하는 시점으로의 복구가 가능하다.
    - 예를들어서, FLUSHALL을 통해서 데이터를 다 날렸더라도, 그 이전까지의 AOF를 사용할 경우 복구가 가능하다.
- fsync를 사용하기 떄문에 내구성이 매우 좋다.
    - Buffer에 있는 데이터를 디스크에 실제로 Write하게 강제한다.
    - 실제 Disk I/O작업이기 때문에 장애에도 비교적 안전 할 수 있다.
- 하나의 파일이 커질 때를 대비하여 Rewrite를 제공한다.

### 단점
- Write가 많은 경우 많은 Memory를 사용하게된다.
- 동일한 양의 정보여도 RDB파일보다 크기가 커진다.


### Rewrite (압축)
- AOF를 더 안정적으로 사용하는 기능이다.
    - 특정 조건에 따라 재구성하게 할 수 있다.
- 커져가는 파일을 분할하고 압축한다.
    - **압축은 기존에 존재하는 AOF파일을 사용하지 않는다.**
    - 현재 메모리를 읽어와서 새로운 데이터로 저장하는 과정이 발생한다.
- RDB와 마찬가지로 자식 Process를 fork()하고 자식 Process가 AOF파일을 Rewrite한다.
- Rewrite과정은 모두 Sequential I/O로 구성된다.
- 똑같이 fork()와 COW(Copy-On-Write)의 영향을 받는다.

### Rewrite 후 AOF 파일의 구조
```text
[Redis 버전 7.0 이전]

+-----------------------------------+
| RDB Preamble (Optional)           |
+-----------------------------------+
| AOF Logs (RESP Commands)          |
+-----------------------------------+
- 'RDB Preamble'은 'aof-use-rdb-preamble' 설정이 'yes'로 활성화되었을 때만 포함됩니다.
- 'AOF Logs'는 모든 데이터 변경 명령어를 순차적으로 기록합니다.

[Redis 버전 7.0 이후]

Manifest File
   |
   |----> AOF File(s)
   |       +
   |
   |----> RDB File

- 'Manifest File': AOF 및 RDB 파일의 상태와 순서를 기술합니다.
  - 어떤 AOF 파일 시퀀스와 RDB 파일 시퀀스를 결합해야 하는지 명시합니다.
- 'AOF File(s)': 데이터 변경 명령어를 기록한 하나 이상의 AOF 파일을 포함합니다.
- 'RDB File': 특정 시점에서의 데이터베이스 전체 스냅샷을 저장하는 RDB 파일을 포함합니다.
```
### 수동 Rewrite
- BGREWRITEOF를 통해서 실행 가능하다.
    - MainThread를 차단하지 않고 백그라운드에서 수행한다.
- **REWRITEOF는 존재하지 않는다.**


### Rewrite 수행과정
```text
[Redis 버전 7.0 이전]

[1] fork()를 통해서 자식 Process를 생성한다.
[2] 자식 Process는 Memory의 데이터를 읽어와서 임시 RDB파일을 생성 후 저장한다.
[3] 부모 Process는 1~2의 작업이 진행되는 동안 기존 AOF파일과 InMemoryBuffer에 데이터를 저장한다.
[4] 임시 RDB파일의 마지막에 InMemoryBuffer의 내용을 append한다.
[5] 임시 RDB파일을 기존 AOF파일에 덮어씌운다.


[Redis 버전 7.0 이후]
[1] fork()를 통해서 자식 Process를 생성한다.
[2] 백그라운드에서 자식 Process를 만드는 동안, 변경분은 새로운 AOF파일에 저장된다.
[3] 임시 Manifeset파일을 생성하고, 내용을 저장하고 기존 Manifest파일을 수정한다.
[4] 임시 Mainfest파일을 삭제하고, 이전버전의 AOF & RDB파일을 삭제한다.
```
### Rewrite 자동 설정
```shell
auto-aof-rewrite-percentage 100       # 이전 AOF파일과 비교해서 두배가 되었을 떄 Rewrite를 수행한다.
auto-aof-rewrite-min-size 64mb        # AOF파일의 크기가 64mb를 넘어가면 Rewrite를 수행한다.
```
- INFO PERSISTENCE를 통해서 AOF파일의 현황을 알 수 있다.


### AOF가 잘려있다면?
- 파일 기록도중 충돌하거나, 볼륨이 가득차면 발생할 수 있다.
- AOF는 이러한 경우, 명령을 버리고 계속 동작한다.

### AOF 파일이 손상되었다면?
- 파일 내부에, 유요하지 않은 byte sequence가 존재한다면 손상된 것으로 간주한다.
- AOF는 작업을 중단하고 복구과정을 갖는다.
    1. AOF 백업본을 만든다.
    2. redis-check-aof 도구를 사용하여 검사 후 수정한다.
- 자동복구과정이 적합하지 않다면, 수동 복구과정이 필요하다.