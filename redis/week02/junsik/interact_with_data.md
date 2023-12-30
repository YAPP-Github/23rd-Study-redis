# Programmability
- 사용자가 Redis의 기능을 확장하고 맞춤화할 수 있게 하는 특징
    - 이런 논리 조각들을 `스크립트` 라고 부름
    - 스크립트를 실행할 수 있는 방법은 두 가지
        1. EVAL
            - v7 이전에서 사용
        2. Functions
            - 스크립팅을 애플리케이션 로직에서 분리하고 스크립트의 독립적인 개발, 테스트 및 배포를 가능하게 함
    - 스크립트나 함수를 실행할 때 원자성 보장
        - 스크립트가 느린 경우 다른 모든 클라이언트가 차단되어 실행 중인 동안 어떤 명령도 실행할 수 없다는 점에 유의
- 주로 Lua 스크립팅, 모듈 시스템, 그리고 트랜잭션 기능을 통해 실행
- 이 기능들은 Redis를 단순한 키-값 저장소에서 더 복잡한 응용 프로그램과 시스템의 요구를 충족시키는 데이터베이스로 만들어줌
- 최대 실행시간 설정
    - 기본값은 5초

## Functions
- 사용자 정의 함수: Redis Functions를 사용하면, 사용자가 직접 작성한 함수를 Redis 서버에 로드하고, 이를 Redis의 다양한 데이터 구조에 적용할 수 있음. 이는 데이터 처리 로직을 데이터에 더 가까이 옮기므로, 네트워크 지연을 줄이고 성능을 향상시킬 수 있음.

- 서버 측 로직 실행: 클라이언트에서 여러 Redis 명령어를 실행하는 대신, 하나의 함수 호출로 복잡한 작업을 처리할 수 있음. 트랜잭션의 필요성을 줄이고, 네트워크 오버헤드 감소

- 함수 라이브러리 관리: 사용자는 여러 함수를 라이브러리 형태로 관리하고, 필요에 따라 이를 Redis 서버에 로드하거나 업데이트할 수 있음. 이를 통해 코드의 재사용성과 유지보수성 향상

- 원자성과 격리성: Redis Functions 내부에서 실행되는 로직은 원자적으로 처리. 즉, 함수 내부의 모든 명령어는 중단 없이 연속적으로 실행되며, 다른 클라이언트의 동시 접근으로부터 격리

## EVAL 과의 비교
1. 코드 관리 및 재사용성
    - Redis Functions
        - 모듈화 및 재사용 가능
        - Redis Functions는 여러 함수를 함께 그룹화하고 관리할 수 있게 해주어, 라이브러리처럼 재사용이 가능합니다.
    - EVAL
        - EVAL은 매번 전체 스크립트를 서버로 보내야 하며, 코드 재사용이나 모듈화가 어렵다
        - 같은 로직을 반복적으로 실행해야 할 때마다 전체 스크립트를 다시 보내야 함

2. 성능 및 효율성
    - Redis Functions
        - 이미 서버에 로드된 함수를 호출하기 때문에 네트워크 지연이 줄어듭니다. 함수를 한 번 로드하고 여러 번 호출 가능
        - 함수가 이미 컴파일되어 메모리에 저장되기 때문에 실행 시간 단축
    - EVAL
        - 스크립트를 전송해야 하므로, 네트워크 지연과 데이터 전송량이 증가합니다.
        - 매번 호출할 때마다 스크립트를 다시 해석해야 하므로, 실행 시간이 더 길어질 수 있음

3. 보안 및 격리
    - Redis Functions
        - 보안 모델 제공, 함수의 실행 권한 제어 가능
        - 격리된 환경에서 실행, 다른 Redis 데이터와의 간섭 최소화
    - EVAL
        - 보안 모델이나 실행 권한에 대한 명확한 구분 없음
        - 동일한 환경에서 실행, 데이터 간 간섭 가능성 존재

4. 유지보수 및 업데이트
    - Redis Functions
        - 함수를 중앙에서 관리하고 업데이트할 수 있으므로, 유지보수 용이
        - 함수 버전 관리 가능
    - EVAL
        - 각 클라이언트가 자체적으로 스크립트를 관리해야 함
        - 업데이트가 복잡해질 수 있음
        - 버전 관리 부재

# Lua scripting
- Lua를 사용하면 Redis 내에서 애플리케이션 로직 일부 실행 가능
- 여러 키에 걸쳐 조건부 업데이트를 수행 가능 
- 여러 가지 데이터 유형을 원자적으로 결합 가능
- 스크립트 소스 코드를 동적으로 생성하도록 하는 것이 가능
    - 스크립트 캐시 고려 사항으로 인해 안티 패턴
- 스크립트에 대한 캐싱 메커니즘을 제공
    - Redis 스크립트 캐시는 항상 휘발성
    - 캐시는 서버가 다시 시작될 때, 복제본이 마스터 역할을 맡을 때 장애 조치 중에 또는 에 의해 명시적으로 지워질 수 있음
    - 정상적인 작업 중에 응용 프로그램의 스크립트는 캐시에 무기한으로 유지
    - 스크립트 캐시를 플러시하는 유일한 방법은 SCRIPT FLUSH명령을 명시적으로 호출
- 스크립트 복제 가능
    - Redis는 복제를 사용하여 특정 마스터에 대해 하나 이상의 복제본 또는 정확한 복사본을 유지
- 스크립트를 복제하는 대신 명령 복제 가능

# Transaction
- 단일 단계로 명령 그룹을 실행할 수 있음
- 모든 명령은 직렬화되어 순차적으로 실행
- 명령이 단일 격리된 작업으로 실행되도록 보장
- 모든 명령어가 실행된 후에만 결과가 외부에 반영
- 롤백 기능 X
- MULTI
    - 트랜잭션의 시작 
    - 이후의 모든 명령어는 큐에 저장
- EXEC
    - 트랜잭션에 저장된 모든 명령어 실행
    - 만약 트랜잭션이 시작된 후에 키가 변경되었다면, EXEC는 트랜잭션을 실행하지 않고 중단
- DISCARD
    - 트랜잭션을 중단
    - MULTI 이후에 큐에 저장된 모든 명령어 삭제
- WATCH
    - 하나 이상의 키 감시 
    - WATCH 된 키가 트랜잭션 실행 전에 변경되면, EXEC는 트랜잭션을 실행하지 않고 중단
- UNWATCH
    - WATCH 로 설정된 모든 키 감시 해제

# Pub/sub
- 메시지 브로커
- 실시간 메시징, 데이터 스트리밍, 이벤트 중심의 아키텍처 등에 널리 사용
- 발행자(Publishers)는 특정 채널에 메시지를 발행
- 구독자(Subscribers)는 관심 있는 채널을 구독하여 그곳에 발행된 메시지 실시간 수신
- 메시지는 채널을 통해 비동기적으로 전달
- 하나의 채널에 여러 구독자가 있을 수 있으며, 모든 구독자는 채널에 발행된 메시지 수신
- 구독자는 특정 패턴을 사용하여 여러 채널 구독 가능
- Redis Pub/Sub은 메시지를 저장 X
    - 구독자가 연결되어 있지 않은 경우 메시지 X
    - 중요한 메시지는 다른 방법으로 관리해야 함