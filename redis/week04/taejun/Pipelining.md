## Pipelining

- 초창기 Redis버전부터 지원했기 때문에, 버전에 상관없이 사용 가능하다.
- Redis명령을 일괄 처리하여 RTT를 최소화한다.
- Redis는 TCP Server이며, Request와 Response를 사용하는 Client-Server모델이다.
    - Client가 Server에 쿼리를 날린다.
    - Server는 해당 명령을 수행하고, 응답을 Client에게 다시 보낸다.
- Client와 Server는 네트워크를 통해서 연결되어있다.
    - 빠를수도 있고, (LoopBack)
    - 느릴 수도 있다. (Host사이에 많은 Network Hop이 껴 있을 경우)
- Client가 Server로 요청을 보내고, 응답을 받아오는데까지 걸리는 시간을 RTT(왕복시간) 이라고 한다.
    - 만약 Redis가 하나의 요청을 250ms에 걸려서 처리한다면 Server가 100K의 요청을 받을 수 있어도, 4개의 요청밖에 처리하지 못할 것이다.
- RTT뿐만 아니라 I/O관점에서도 유리하다.
    - read(), write()와 같은 SystemCall을 하나의 요청으로 다 처리할 수 있기 때문이다.

### 진행과정

- Client가 다수의 명령을 Server로 일괄적으로 보낸다.
    - 보냈던 요청을 한번에 받는다.
    - 각각의 요청을 대기하는 Latency를 줄일 수 있기 떄문에 성능향상에 유리하다.
- 너무 많은 명령어를 한번에 PipeLine에 보내면 서버의 부하가 증가 할 수 있다.
    - Server가 보낼 응답을 Memory에 저장해두고있기 떄문이다.
    - 적절한 크기의 배치 사이즈를 찾는 것이 중요하다.