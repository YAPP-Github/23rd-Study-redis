<img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/2f9e3dc2-22db-405e-974e-52d509662e3b" width="500px">

# Redis Sentinel

- sentinel mode로 시작된 다수의 레디스 인스턴스로 구성된 분산 시스템
- sentinel 그룹은 primary 레디스와 replica들을 모니터링함
    - 이때 sentinel이 primary 인스턴스의 장애를 감지하면, sentinel 프로세스들은 가장 최신의 데이터를 가진 replica를 찾아서 새로운 primary로 승격시킴
- 레디스의 고가용성을 위해 사용

### Primary 인스턴스 장애 감지시

- sentinel들이 primary인스턴스가 다운되었다고 결정하기위해, 충분한 sentinel들의 동의가 필요함
- 장애에 조취를 취하기 위해 필요한 sentinel들의 동의 수를 **Reaching a quorum**이라고 함. sentinel들이 quorum에 도달하지 못하면, primary가 다운되었다고 할 수 없음. (이 때 숫자는 설정가능)

### 장애극복 트리거

- sentinel이 primary 인스턴스가 다운되었다고 결정하게되면, sentinel 인스턴스 중에서 그 다음 리더(sentinel 인스턴스)를 선출해야함. 과반수의 sentinel이 찬성하면 그 리더가 정해짐
- 그 다음, 리더는 주어진 레플리카가 primary가 되도록 재설정함 (`REPLICAOF NO ONE`)
- 그후에는 다른 replica들이 새로 승격된 primary를 따르도록 재설정

### Sentinel, Docker, NAT, and possible issues

- Docker에서 포트를 매핑시킬때 실제 프로세스의 포트와 외부에 공개되는 포트가 다를 수 있음
    - NAT(network address translation)도 비슷한 작업이 가능하고, 이때 IP가 바뀔수도 있음
- 이렇게되면…
    - 다른 sentinel에서 sentinel 자동발견이 잘 동작하지 않음.. sentinel은 자기 스스로 listening하는 ip와 포트 정보를 알리지만 다른 sentinel은 여기로 연결이 안됨
    - 레디스 마스터에서 `INFO` 를 사용하면 replica들의 정보가 나오는데 이떄 IP는 마스터가 확인할 수 있지만 포트는 replica와의 handshake과정에서 알수있으므로 연결이 안될것
    - 따라서 1:1 포트매핑을 사용하거나 network모드로 host를 이용할 수 있음
