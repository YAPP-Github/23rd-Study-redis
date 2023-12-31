# Redis sentinel
- Redis Cluster 를 사용하지 않을 때 `고가용성 제공`
- 아래의 부수적인 기능들도 제공
    - Monitoring
    - Notification
    - Automatic failover
    - Configuration provider

# Sentinel as a distributed system
- Redis Sentinel은 `분산 시스템`
- Sentinel 자체는 `여러 Sentinel 프로세스가 함께 협력하는 구성에서 실행되도록 설계`
- 장점
    - 여러 Sentinel이 특정 마스터를 더 이상 사용할 수 없다는 사실에 동의하면 오류 감지 수행. 거짓 긍정 가능성이 낮아짐
    - Sentinel은 모든 Sentinel 프로세스가 작동하지 않는 경우에도 작동. 시스템이 장애에 대해 견고하게 유지

# Sentinel 실행 기본사항
- Sentinel을 실행할 때 구성 파일을 사용하는 것은 필수
- 설정 파일은 재시작 시 다시 로드될 현재 상태를 저장하기 위해 시스템에서 사용
- 기본적으로 TCP 26379 포트 사용

# Sentinel 구성
```
sentinel monitor <master-name> <ip> <port> <quorum>
```
- `quorum` 마스터를 실패로 표시하고 가능한 경우 결국 장애 조치 절차를 시작하기 위해 마스터에 연결할 수 없다는 사실에 `동의해야 하는 센티널의 수`
- 실패를 감지하는 데에만 사용
- 장애 조치를 수행하려면 Sentinel 중 하나가 장애 조치에 대한 리더로 선출되고 계속 진행할 수 있는 권한을 부여받아야 하는데, 이는 Sentinel 프로세스의 대다수 가 투표한 경우에만 발생
-  Sentinel 프로세스의 대부분이 통신할 수 없는 경우, Sentinel이 장애 조치를 시작하지 않는다는 것을 의미

# Adding or removing Sentinels
- Sentinel이 구현하는 자동 검색 메커니즘 덕분에 배포에 새 Sentinel을 추가하는 과정은 간단
    - 현재 활성화된 마스터를 모니터링하도록 구성된 새로운 Sentinel을 시작하면 됨
    - 여러 Sentinel을 추가해야 하는 경우 다음 Sentinel을 추가하기 전에 다른 모든 Sentinel이 첫 번째 Sentinel에 대해 이미 알고 있을 때까지 기다리면서 하나씩 추가하는 것이 좋다
- Sentinel을 제거하는 것은 좀 더 복잡
    - Sentinel은 오랜 시간 동안 연결할 수 없더라도 이미 본 Sentinels를 잊지 않음
    - 장애 조치 및 새 구성 생성을 승인하는 데 필요한 대부분을 동적으로 변경하고 싶지 않기 때문
    - 아래 단계를 통해 삭제해야 함
    1. 제거하려는 Sentinel의 Sentinel 프로세스를 중지
    2. 다른 모든 Sentinel 인스턴스에 `SENTINEL RESET *` 후, 인스턴스 간에 최소 30초 wait
    3. 모든 Sentinel의 출력을 검사하여 모든 Sentinel이 현재 활성화된 Sentinel 수에 동의하는지 확인

# Sentinels and replicas auto discovery
- Sentinel 은 서로의 가용성을 상호 확인하고 메시지를 교환하기 위해 다른 Sentinel 과 연결을 유지
- 다른 Sentinel 주소 목록을 구성할 필요는 없음
- 다른 Sentinel을 검색하기 위해 Redis 인스턴스 Pub/Sub 기능을 사용

# Replica selection and priority
- 복제본 선택 프로세스에서는 다음 정보를 평가
    1. 마스터와의 연결이 끊긴 시간
    2. 복제본 우선순위
    3. 처리한 데이터량
    4. Run ID