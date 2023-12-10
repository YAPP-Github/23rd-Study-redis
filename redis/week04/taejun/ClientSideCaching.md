# Client-Side Caching

- ClientSideCaching은 Server에서 높은 퍼포먼스를 내기위한 중요한 방법 중 하나나이다.
- DB와 같은 Storage가 아닌 별개의 공간에 (Memory)에 데이터를 저장하는 것이다.
    - 저장할 수 있는 데이터의 크기는 DB에 비해서 적지만, 속도에서 차이가 많이난다.
    - DB에 가해지는 부담 또한 줄일 수 있다.
- 자주 사용되는 것을 주로 저장한다.
- 여기서 말하는 ClientSide는 WebApplication, Server는 Redis

## Client-Side Cache 지원 전략

### Redis - Tracking
- Redis 6.0부터 지원한다.
    - CLIENT TRACKING이라는 명령어를 통해서 활성화 한다.
- Server는 특정 Client가 엑세스한 Key를 기억한다. (이래서 Tracking)
- 동일한 Key가 수정되면, Server는 Client에게무효화 Mesage를 보낸다.
    - Client가 Memory에 가지고있는 Key Set에 대해서만 무효화 메세지를 보낸다.

장점
- Redis(Server) 의 부하 감소
    - Cache를 Redis에서 질의하지 않기 떄문
- 응답시간 개선
- 네트워크 트래픽 감소
- 데이터 동기화 성능 향상

단점
- Application(Client)의 복잡성 증가 및 메모리 사용량 증가

### TwoConnections
- Redis6부터 지원하는 RESP3프로토콜을 사용하면, 같은 Conection내에서, Query 송신과 무효화 메세지 수신을 모두 수행 할 수 있다.
- 하지만 몇몇 Client는 2개의 Connection을 선호한다 (RESP2 프로토콜에서도 사용가능)
    - Conenction1 → 무효화 메세지
    - Connection2 → Query
- PubSub 채널을 구독한다.

```bash
(Connection 1 -- used for invalidations)
CLIENT ID # (1) 현재 Client의 Id를 가져온다.
:4
SUBSCRIBE __redis__:invalidate # (2) 무효화 메세지 채널 구독
# RESP 프로토콜을 사용하는 pubsub 형식
# *n은 n개의 요소로 이루어진다는 의미이다.
# $n은 n의 길이를 가진다는 의미이다.
# 마지막은 Client가 현재 구독하고 있는 Channel의 수를 나타낸다.
*3
$9
subscribe
$20
__redis__:invalidate
:1

# Get

(Connection 2 -- data connection)
CLIENT TRACKING on REDIRECT 4 # (2) Client Id를 입력하여 Tracking을 시작하고 키 변경사항을 1의 Client에 Reirect 시킨다.
+OK

GET foo
$3 #RESP 
bar

# Set
(Some other unrelated connection)
SET foo bar
+OK

# After Set

*3
$7
message
$20
__redis__:invalidate
*1
$3
foo # foo가 변경됐다.

```

### DefaultMode
- 전체 DataSet이 아닌, 변경된 부분의 정보에 대해서만 클라이언트에게 전송하여, 네트워크 트래픽을 최소화한다.
- 무효화 테이블에 Caching가능성이 있는 Client의 목록을 추가한다.
    - Client의 고유한 Id만을 저장한다.
- Client가 많아지면 성능상의 문제가 생길 수 있따.
    - Server는 ClientKey에 대한 더 많은 데이터를 유지해야한다.
    - Client는 실제로 Caching을 하지 않은 데이터에 대한 무효화 메세지를 자주 받을 수 있게 된다.

### Opt-in
- OPTIN명령을 통해서 트래킹을 활성화한다.
- 명시적으로 캐싱할 것을 전달하는 것이다.
    - Server의 부하가 줄어든다. (관리해야 할 것이 줄어든다)
    - 무효화 메세지를 받을 가능성도 줄어든다.
- 기본적으로 Read명령어를 수행해도 Cache가 되지않았다고 가정한다.
    - Cache를 한다는 것을 명시적으로 전달 한 후부터 추적을 한다.

```bash
CLIENT TRACKING on REDIRECT 1234 OPTIN

CLIENT CACHING YES
+OK
GET foo
"bar"
```

### BroadCastMode
- Redis 측에서 변경 정보에 대해서 기억하지 않는다.
- Client측에서 Prefix를 구독한다
  - 자기것이 아니면 버리는 Filtering 로직이 필요하다.
- BCAST 키워드를 통해서 활성화한다.
```kotlin
CLIENT TRACKING on REDIRECT 1234 BCAST PREFIX {prefix}
```
- opt-in같이 prefix를 등록하고, 해당 prefix에 해당하는 Key의 변경에 대해서 BroadCast한다.
- Client와 prefix를 연결 짓는 prefix table을 사용한다.