# Replication

- 데이터는 primary 인스턴스에만 기록되며, 복제본은 primary인스턴스의 복사본이 되도록 동기화 상태로 유지됨
- 복제본은 생성하려면 primary인스턴스의 주소와 포트로 설정된 replicaof 지시어를 사용해 redis 서버 인스턴스를 생성
    - `replicaof 192.168.1.1 6379`

<img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/9e8ee373-776b-4af0-a3dc-5e23aff4c49f" width="500px">

- 복제본 인스턴스가 실행되면 복제본은 primary 인스턴스와 싱크를 시도
    - 모든 데이터를 최대한 효율적으로 전송하기 위해 primary 인스턴스는 압축된 버전의 데이터를 RDB파일의 스냅샷으로 생성하여 replica에게 보냄

      <img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/07c920fd-2763-48fe-aa9c-6d1235ad50c9" width="500px">

    - 복제본은 스냅샷 파일을 읽고 모든 데이터를 메모리에 로드
    - RDB파일을 생성하는 순간 primary 인스턴스와 동일한 상태로 만듦
    - 로딩 단계가 완료되면 primary 인스턴스는 스냅샷이 생성된 이후 실행된 모든 쓰기 명령의 백로그를 보냄
    - 마지막으로 primary 인스턴스는 모든 후속 명령의 live stream을 replica에게 보냄
    - 동기화 매커니즘
        - master-replica가 연결되면 master는 data에 변경을 주는 모든 명령을 replica에게 전달
        - 이때 둘간의 연결이 끊기면 replica가 다시 연결됐을때 부분 재동기화 시도 (연결이 끊어진 동안의 누락된 명령 stream을 가져옴)
        - 부분 재동기화가 불가능하면 replica는 전체 재동기화 요청 (마스터의 스냅샷을 받아옴)
- 기본적으로 복제는 비동기식
    - redis에 명령을 보내면 ACK 반응을 받고, 그 명령이 복제본에 복제됨.
    - 쓰기를 승인한 후 쓰기가 복제되기 전에 primary가 다운되면 데이터가 손실될 수 있음
    - 이를 방지하기 위해 클라이언트는 Wait명령을 사용할 수 있음
        - 모든 이전 쓰기 명령이 성공적으로 전송되고 최소한 지정된 수의 복제본에 의해 승인될때까지 현재 클라이언트를 차단
    - 보통 비동기지만, `WAIT` 커맨드를 통해 동기식으로 요청할 수 있음
        - 예를들어 `wait 2 0` 명령을 보내면 클라이언트는 해당 연결에서 실행된 모든 이전 쓰기 명령이 최소 두개의 복제본에 기록될 때까지 클라이언트에 대한 응답을 차단하거나 반환하지 않음. (이때 0은 서버에 무기한 차단을 지시함을 의미)
- 복제본은 읽기 전용
    - 클라이언트에서 읽을 수 있지만 애초에 모든 쓰기 명령을 거부하므로 쓸수는 없음
        - 그러나 `DEBUG`, `CONFIG` 명령은 가능하기때문에 외부에 공개하진 말자

###### Replication 동작방식

- replica는 마스터와 연결할때 `PSYNC` 커맨드와 함께 자기 replication id와 지금까지 처리한 offset을 전달하여 필요한 부분만 넘겨받을 수 있음
- 이때 offset은 논리적인 시간을 의미함. 이를통해 누가 가장 최신의 데이터를 가지고 있는지 알수있음

###### Replication ID

- 마스터 인스턴스가 재시작하거나 replica가 마스터로 승격되는 경우 새로운 replication id가 부여됨
- replica가 마스터로 승격되는 경우
    - 이전 마스터의 Replication id와 새로운 Replication id 두개를 갖게됨
    - 다른 replica가 새로 승격된 마스터와 연결할때 이전 Replication id를 통해 어디까지 데이터를 받았는지 부분 재동기화할 수 있음

###### 만료시간이 있는 키의 경우

- 마스터에서 보낸 명령어가 replica한테 늦게 도착해가지구 도착했을때 이미 만료시간이 지났다면?
    - replica는 키를 만료시키지않고 마스터가 만료시킬때까지 기다림. 마스터는 만료시키고 `DEL` 명령을 모든 replica한테 보냄
    - 이때 마스터가 `DEL` 명령을 늦게 보냈다면 replica들은 logical clock을 사용해서 읽기 명령에대해서 키가 존재하지 않는다고 응답함

### Replication test

<img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/f63ab904-eb97-484a-8e36-11c5ba28a0e0" width="500px">

- 마스터, 슬레이브 노드 실행

<img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/ef91b196-7577-485b-b7b2-7f7eeeb6815a" width="500px">

- 슬레이브 노드에서 마스터 노드 관계 설정

<img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/1fe7e6d0-c96d-4b48-a01b-b95bb13580ef" width="500px">

- 마스터 노드에서 write수행시 슬레이브 노드에서 read가능

마스터 노드가 다운됐다면? → sentinel 기능으로 슬레이브를 마스터로 승격시킬 수 있음
