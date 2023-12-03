# 레디스 Pub/Sub

# Pub/Sub

- Publisher는 Subscriber가 누군지 모른채로 채널에 메시지를 보냄
- Subscriber는 Publisher가 누군지 모른채로 메시지를 받음
    - 구독자는 메시지가 publish된 순서대로 메시지를 받음
- redis의 pub/sub시스템은 매우 단순한 구조
    - 메시지를 던지는 시스템이라서, 메시지를 따로 보관하지도 않음
    - 즉, 수신자가 메시지를 받는것을 보장하지 않아 수신자가 하나도 없는 상황에서 메시지를 publish하면 사라짐

![image](https://github.com/Team-BuddyCon/BACKEND_V2/assets/69676101/a3e38902-904b-4e3e-b575-21dadaded371)

# Format of Pushed Messages

- subscribe
    - 채널 구독
- unsubscribe
    - 반환값의 마지막 인자가 0이면 구독중인 채널이 없다는뜻
- message
    - publish command의 결과로 받은 메시지

# Pattern-matching subscriptions

- 글로벌 스타일 패턴을 subscribe하여 그 패턴과 일치하는 채널의 이름으로 전송되는 모든 메시지 수신 가능
    - `PSUBSCRIBE channel.*`


# Pub/Sub - Kafka vs Redis

### kafka

![image](https://github.com/Team-BuddyCon/BACKEND_V2/assets/69676101/67137ec1-c7d5-4220-a907-d61df5642b0e)

- Producer는 Topic에 이벤트를 보내고, 이 이벤트는 Topic의 각 Partition에 분산되어 저장됨
- 토픽을 구독하는 수신자가 여러개라면, 먼저 발견하는 수신자가 가져가는 듯

### Redis

![image](https://github.com/Team-BuddyCon/BACKEND_V2/assets/69676101/e11b1086-1b26-4fe3-8178-35d0b8aa25bf)

- 그룹의 개념이 존재하지 않음
- 채널은 이벤트를 저장하지 않음
    - publish됐을때 수신자가 없다면 그 메시지는 사라짐

### 차이점

- 이벤트가 저장되는가/저장되지 않는가
- 발행된 이벤트에 대해, 특정 작업이 한 번만 발생하는가/모든 Subscriber에서 발생되는가