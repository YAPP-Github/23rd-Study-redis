## Redis KeySpace Notification

- PubSub을 통해서, 데이터에 영향을 미치는 이벤트를 구독 할 수 있다.
    - 실제로 Key가 변경 됐을 떄만 이벤트를 발행한다.
- 수신 할 수 있는 이벤트
    - Key에 영향을 미치는 모든 작업
    - LPUSH작업이 가능한 모든 Key
    - 만료가 되는 Key
- **PubSub은 Fire & Forget구조이고, Client가 받지 못했다면 추가로 할 수 있는 작업은 없다.**
- Server에 부하를 주는 구조이다,
    - Key변경 사항을 모니터링
    - 이벤트 생성 및 전송
    - Network 트래픽의 증가

### 이벤트 유형

- 2가지의 Event를 발생 시킨다.
    - KeySpace 채널
        - 이벤트 이름을 메세지로 받는다.
        - **PUBLISH __keyspace@0__:mykey del**
    - KeyEvent 채널
        - 특정 키에 대한 메세지를 받는다.
        - **PUBLISH __keyevent@0__:del mykey**

### 설정

- 기본적으로 Disable 되어있다.
    - CPU를 사용하기 떄문에, 조심해서 다루어야 하기 때문이다.
- redis.conf의 notify-keyspace-events 설정 또는, CONFIG SET을 통해서 활성화 된다.
    - CONFIG SET은 Runtime에 설정을 바꾸는 역할
    - redis.conf는 Server가 뜰 때 읽는 설정이기 때문에, 변경 할 경우 서버를 재시작 해야한다.

### TTL Event

- 아래와 같을 때 Event가 생성된다.
    - Key에 Access했을 떄, 만료된 것으로 확인 된 경우
    - 한번도 Access되지 않은 Key를 수집할 수 있도록 백그라운드에서 점진적으로 Key를 찾을 경우
- 즉 만료가 확정된 시점에 Event가 발행되는 것이 아니다.
    - 상당한 지연이 있을 수 있다.

### Cluster에서의 Event

- Cluster에서의 Event는 모든 Node에게 BroadCast되지 않는다.
- Client가 모든 Key에 대한 Event를 수신하려면, 각 노드에 대한 Subscription을 구현해야 한다.