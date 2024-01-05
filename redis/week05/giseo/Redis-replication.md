# Redis replication
- Redis는 replication을 통해 고가용성과 장애 조치를 지원하는 방법을 제공한다.

## replication 방식
1) 명령 스트림 전송: 마스터와 복제본이 연결되어 있을 때, 마스터는 데이터 세트 변경을 복제본에 반영하기 위해 명령 스트림을 복제본에 전송한다.
2) 부분 재동기화: 마스터와 복제본 사이의 연결이 끊어지면, 복제본은 부분 재동기화를 시도한다. 즉, 연결이 끊어진 동안 놓친 명령 스트림의 일부만 얻으려고 시도한다.
3) 전체 재동기화: 부분 재동기화가 불가능한 경우, 복제본은 전체 재동기화를 요청한다. 이는 마스터가 모든 데이터의 스냅샷을 생성하고 이를 복제본에 전송하고, 이후의 데이터 세트의 변경은 명령 스트림으로 전송한다.
---

### 부분 재동기화
- 마스터와 복제서버는 각 서버의 run id 와 replication offset을 가지고 있다.
- 마스터와 복제서버간 네트워크가 끊어지면 마스터는 복제서버에 전달할 데이터를 backlog-buffer에 저장한다.
- 다시 연결되었을 때 backlog-buffer가 넘치지 않았으면 run id와 offset을 비교해서 그 이후 부터 동기화를 진행한다.
- 네크워트 단절 시간이 길어져 마스터의 backlog-buffer가 넘치면 다시 연결되었을 때 전체 동기화를 진행한다.

### 전체 재동기화
- 마스터는 자식 프로세스를 시작해 백그라운드로 RDB파일에 데이터를 저장한다.
- 데이터를 저장하는 동안 마스터에 새로 들어온 명령들은 처리 후 복제버퍼에 저장된다.
- RDB 파일 저장이 완료되면, 마스터는 파일을 복제서버에게 전송한다.
- 복제서버는 파일을 받아 디스크에 저장하고, 메모리로 로드한다.
- 마스터는 복제버퍼에 저장된 명령을 복제서버에게 전송한다.,

## replication의 특징
1) Asynchronous replication: Redis는 기본적으로 비동기 복제를 사용한다. 복제본은 주기적으로 마스터에게 받은 데이터 양을 비동기적으로 확인한다.
2) 복제본 간 연결: 복제본은 다른 복제본과 connection을 맺을 수 있다. 이를 통해 캐스케이딩 구조로 복제본을 연결하는 것이 가능하다.
3) non-blocking: 복제 중에도 마스터는 다른 쿼리를 계속해서 처리할 수 있다.
4) non-blocking on the replica side: 복제본은 동기화 초기에는 이전 데이터 세트를 사용하여 쿼리를 처리할 수 있습니다. 그러나 이후에는 이전 데이터 세트를 삭제하고 새로운 데이터 세트를 로드해야 한다.
   - 이 과정은 복제본이 잠시 동안 차단될 수 있다. 
   - Redis 4.0 이상에서는 이전 데이터 세트의 삭제를 별도의 스레드에서 처리할 수 있도록 구성할 수 있으나, 새로운 데이터 세트의 로딩은 여전히 메인 스레드에서 이루어지므로, 이 과정 중에는 복제본이 차단된다.


## Replication ID
- Redis의 마스터는 고유하고 랜덤한 **Replication ID**를 가진다. 
- 연결된 Replica가 존재하지 않아도 **incremented offset**을 가지는데 이는 복제가 이루어져 Replica에 변경사항이 전달되면 증가한다.
  - Replication Id와 offset은 Master의 데이터에 대한 고유한 버전 식별로 사용 될 수 있다.

- Replica가 Master에 연결할 때, PSYNC 명령을 사용하여 이전 Master의 Replication Id와 지금까지 처리한 offset을 보내어 필요한 데이터 복제 부분만 보낼 수 있게 된다.
  - 만약 Master의 버퍼에 백로그가 없거나, Replication Id가 더 이상 유효하지 않은 Id를 참조하는 경우, 전체 동기화가 발생하게 된다.

- 전체 동기화가 발생하면 Master에서는 백그라운드에서 저장 프로세스가 시작되어 RDB파일을 생성하고, 동시에 클라이언트로부터 받고 있는 새로운 쓰기 명령을 버퍼링하기 시작한다.
- 이후 백그라운드 저장이 완료되면, RDB파일을 먼저 복제본에 전송하고 이후 버퍼에 담긴 쓰기 명령들을 보낸다.

### How Redis replication deals with expires on keys
- Replica 에서는 key를 스스로 만료시키지 않는다. Master와 Replica의 시계의 동기화로 의존한다면 데이터 일관성의 큰 결함을 초래할 수 있기 때문이다.
- Master에서 키를 만료시키면, DEL 명령을 모든 복제본에 전송하여 제거한다.
- 이러한 방식은 Replica에서 DEL 명령을 제공받지 못한 경우 문제가 발생 할 수 있다. 논리적으로 만료된 키가 유지 될 수 있는데, 이를 처리하기 위해 논리적 시계를 사용하여 DEL 명령을 수신하지 않았더라도 만료된 것처럼 처리한다.