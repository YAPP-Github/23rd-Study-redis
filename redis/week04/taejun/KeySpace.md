# Keyspace

## Key에 대한 규칙
Key의 최대 길이는 512MB이다.

1. 긴 Key는 좋은 방법이 아니다.
    1. 메모리 측면
    2. Key를 조회하는데 많은 비용이 든다.
    3. Key가 길다면 sha1과 같은 것으로 해싱하는게 더 좋은 방식이다.
2. 짧은 Key 또한 좋은 방법이 아니다.
    1. 가독성이 더 중요하다.
    2. 짧은 Key를 사용하는 것이 메모리 측면에서 더 좋을 수 있으나, 올바른 균형을 찾는 것이 중요하다.
3. 일정한 Schema를 사용하는 것이 좋다.
    1. ex) object-type:id (comment:4321:reply:to)

## Key에 대한 질의

### [1] EXISTS
- Key 존재 유무에 대해서 쿼리한다.
- 있으면 1 없으면 0

### [2] DEL
- Key와 Value를 모두 삭제
- Key에 해당하는 값이 있었다면 1, 없었다면 0을 리턴
- 동기적인 명령어이다.

### [3] TYPE
- Key에 해당하는 Value의 타입에 대해서 리턴한다.

### [4] EXPIRE
- Key를 만료시킨다.
- 시간 해상도는 초(Sec) 또는 밀리초(milisSec)으로설정할 수 있다.
    - expire에 설정되는 해상도는 항상 밀리초이다.
- Expire정보는 Disk에 저장되고 Replication된다.
    - 언제 만료가 될지 저장이 되있다.
    - 즉, Redis가 종료되어있어도 시간은 흐르는 것이다.

### [5] TTL
- 남은 만료시간 체크
- 시간해상도는 항상 밀리초이다.

### [6] UNLINK
- 비동기적인 Key삭제
- 별도의 쓰레드에서 백그라운드로 삭제한다.
- 논리적 처리시간은 O(1), 실제 처리시간은 O(N)
- del은 동기적이기 떄문에, 많은 양의 Key를 삭제하게 된다면 시스템에 장애를 유발 할 수 있다.

### [7] PERSIST
- 만료 옵션이 지정되있는 Key를 영구보관으로 바꾼다.
- 만료옵션을 제거한다.
  - 0: 키가 존재하지 않거나, TimeOut옵션이 지정되어있지 않음
  - 1: TimeOut옵션 삭제
## Key 탐색

### [1] Scan
- 호출 당 소수의 Key만을 가져온다.
    - O(N)명령어들 보다 훨씬 좋다.
        - KEYS
        - SMEMBERS
- 중복을 허용한다.
  - 중복 Key에 대한 중복제거는 Application의 몫이다.

### [2] KEYS
- O(N)명령어다.
    - 결과를 리턴하기 전까지 RedisServer에 해당하는 모든 명령어가 Blocking된다.
    - 사용하면 안된다.
- KeySpace를 관리하고 싶다면, SCAN명령어나 Set자료구조를 사용하는것이 더 적합하다.
### KEYS Expression
- ?: 하나의 문자 혹은 숫자와 일치하는 모든 Key를 보여준다.
    - ```shell
        keys h?llo;
        - hello
        - hallo
        - hqllo
        - ...
      ```
- *: 나머지 조건에 일치하는 모든 Key를 보여준다.
    - ```shell
        keys h*llo
        - heeeeeeeeeeeeeeeello
        - hello
        - hasdcadfqwaerqasdfasdfllo
        - ...
      ```
- []: 사이에 있는 문자와 일치하는 것들의 Key를 보여준다.
    - ```shell
          keys h[ae]llo
          - hallo
          - hello
      ```
    - 괄호 내부에 있는 것들의 조합을 의미하는 것이 아니다.
- [^]: 괄호에 포함된 것들을 제외하고 검색한다.
    - ```shell
        keys h[^e]llo
        - hallo
        - hillo
        - ...
      ```
- [-]: 범위에 포함된 것들을 검색한다.
    - ```shell
        keys h[a-c]llo
        - hallo
        - hbllo
        - hcllo
      ```