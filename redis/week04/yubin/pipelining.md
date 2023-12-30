## Pipelining이란?

- 여러 명령을 응답을 기다리지않고 한번에 처리하도록 하는것
- RTT(round trip time)을 줄이는 방법, 일정 시간동안 수행하는 명령의 수를 크게 증가시킴(유저모드에서 커널모드로 전환하는 context swith와 Socket I/O의 비용을 줄임)
- 파이프라인을 사용할 경우 서버는 메모리를 사용해서 응답을 대기시킴. 따라서 많은 명령을 보낼때는 배치로 일정 청크만큼 나눠서 보내는것이 좋음
- 명령들은 단일 read() 시스템콜로 읽히고, 단일 write() 시스템콜로 응답들이 반환됨

  ![image](https://github.com/YAPP-Github/21st-Android-Team-1-BE/assets/69676101/a276bee8-4f53-40af-ad04-413a532d3e4f)


### Jedis Pipeline 실습

![image](https://github.com/YAPP-Github/21st-Android-Team-1-BE/assets/69676101/5332de5f-828d-48b3-ad6a-7ee6cc431f86)

![image](https://github.com/YAPP-Github/21st-Android-Team-1-BE/assets/69676101/900d1a39-7687-4351-8aac-56a928bc4e38)

### Pipelining VS Scripting

- 스크립트를 사용하면 서버사이드에서 필요한 작업을 더 효율적으로 처리 가능
- 스크립트 사용시 최소한의 지연으로 읽기/쓰기를 동시에 할 수 있음
    - 파이프라이닝은 쓰기 명령 호출 전에 읽기 명령이 필요하기 때문에