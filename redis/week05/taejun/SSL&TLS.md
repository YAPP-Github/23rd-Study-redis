### SSL/TLS

- 버전 6부터 지원을 시작했다.
- 두가지 모두 암호화를 위한 인터넷 보안 프로토콜
    - SSL은 1996년 이후로 업데이트 되지 않고있다.
    - 두가지용어가 혼용되어 사용되고 있다.
- TLS 1.0, 1.1은 안전하지 않다고 판단되고있고 1.2사용이 권장된다.
- Redis는 Network를 통해서 빠르게 데이터를 주고받는데, 이 과정에서 평문일 경우 문제가 생길 수 있다.

### SSL/TLS 설정 Enable

- 기본적으로는 Disable되어있다.
- SSL개발 라이브러리가필요하다.
- Redis빌드를 해주어야한다.

```bash
make BUILD_TLS=yes
```

- Server와 Client가 동일한 인증서를 사용한다.
- TLS port를 설정해주어야한다.

    ```bash
    # 일반 포트 비활성화
    port 0
    
    # TLS 연결만 허용
    tls-port 6379
    ```


### 명시적으로 Server 실행시키기

```bash
./src/redis-server --tls-port 6379 --port 0 \
    --tls-cert-file ./tests/tls/redis.crt \
    --tls-key-file ./tests/tls/redis.key \
    --tls-ca-cert-file ./tests/tls/ca.crt
```

### cli를 통한 접근

```bash
./src/redis-cli --tls \
    --cert ./tests/tls/redis.crt \
    --key ./tests/tls/redis.key \
    --cacert ./tests/tls/ca.crt
```

### Replicaion에서의 TLS

- Client와의 설정이 Replicaion에도 똑같이 적용된다.
- MasterNode 설정
    - tls-port 정의되어있어야 한다.
    - tls-auth-clients(Optional)를 통해서, Client에게 인증서를 요구 할 수 있다.
- ClientNode 설정
    - tls-replicaion을 enable해야 한다.
    - ReplicaionNode → MasterNode로의 통신이 TLS로 연결된다.

### Sentinel에서의 TLS

- Sentinel은 Redis의 네트워크 구성을 상속받는다.
- 설정한 모든내용은 Sentinel에게도 전파된다.
- Sentinel ←→MasterNode간의 연결 시, tls-replicaion을 통해서 지원할 지 여부를 결정한다.
    - enable되어있으면 Sentinel과의 연결도 tls-port에 매핑된 포트로 TLS로 되어야한다는 것이다.

### Cluster에서의 TLS

- tls-cluster 옵션이 enable되어있다면, cluster-bus간의 통신도 tls를 통해서 이루어진다.

### 고려해 볼 점

- Read / Write에 암호화, 복호화를 수행하는 통신계층이 추가되는 것이다.
- 결과적으로 Instance 하나 당 처리량은 줄어든다.