## ACL (AccessControl)

- Redis 6.0버전부터 사용가능하다.
- User마다 다른 Password와 권한을 줄 수 있다.
- ACL List를 통해서 부여받은 제한을 확인 가능하다.

    ```bash
    ACL List
    # default라는 user는 active 되어있고 패스워드가 없으며 
    # 모든 카테고리 enable,모든 pubsub enable, 모든 key pattern enable
    "user default on nopass ~* &* +@all" 
    ```

- 기본적인 설정은 redis.conf에 저장된다.
    - 파일 분리도 가능하다.

    ```bash
    # redis.conf에 acl파일의 위치를 지정해주면 된다.
    aclfile <path-to-acl-file>
    
    # redis.conf의 aclfile 설정에 저장되있는 설정을 Load한다.
    ACL LOAD
    
    # 초기 파일의 형태와 달라졌을 때 (ACL SET USER, ...) 해당 설정은 메모리에 위치하게 된다.
    # 이걸 ACL파일에 다시 저장하여 메모리와 설정을 동기화 시킨다.
    ACL SAVE
    ```


### Connection 제어

- redis.conf 혹은 cli상에 파라미터를 통해서 설정을 넘길 수 있다.
- Redis Server는 여러개의 NetworkInterface를 가질 수 있다.
    - 여러 IP중에 어떤 IP로 들어오는 요청만 받을 것인지 설정한다.
- Client의 접근을 관리 할 수 있다.
    - 특정 IP만 Ciient에게 허용하기 떄문에 보안을 높일 수 있다.
- Network대역폭환경에 맞게 제한하여, 성능최적화도 가능하다.

### Password

- 1개이상 지정 가능하다.
    - 추가 삭제 모두 가능하다.
- SHA256으로 해시되어있다.
- 2가지 방식이 있다.
    - ACL이전처럼 Node에 직접 Password를 매핑
        - requirepass

            ```bash
            # 명령어를 통한 전달
            CONFIG SET requirepass <password>
            
            # -a옵션을 통한 접근
            redis-cli -a <password>
            
            # 우선 접근 후 AUTH
            redis-cli
            
            # username 명시적 지정
            AUTH <user-name> <password>
            
            # username은 default로 가정된다.
            AUTH <password>
            ```

        - redis.conf에 저장
    - ACL 기능을 사용
        - nopass: 비밀번호 없이 접근 가능
        - resetpass: 비밀번호 초기화
            - nopass에 대한 권한도 사라지게 된다.
            - 새로운 비밀번호를 부여하기 전까지는 아무도 접근 할 수 없다.

### protectedMode

- Redis를 운영할 것이라면 사용하는 것이 권장된다.
- Password를 입력하지 않았다면, local(127.0.0.1)에서 접근 하는것만 허용된다
    - 일종의 이중장치라고 볼 수 있다.

### Command제어

- redis-command를 통해서, 이름을 변경하거나 활성화 / 비활성화가 가능하다.
    - redis.conf에서 변경이 간으하다.
    - CONFIG GET을 통해서 보거나, CONFIG SET을 통해서 변경하는 것은 불가능하다.
- 세밀한 설정을 통해서 보한 강화가 가능하다.
    - ex) CONFIG SET과 같은 설정변경은 매우 위험한데, ACL을 통해서 특정 User의 접근을 Disable할 수 있다.
- command이름도 변경 가능하다.
    - 예를들어서 다른 기능과 함께 사용되는 command가 변경되면 같이 설정해주어야한다.
    - ex) REPLICAOF, CONFIG가 변경되면 sentinel.conf도 변경되어야한다.
- rule은 왼쪽 → 오른쪽 순서로 적용된다.

| 분류 | 규칙 | 설명 |
| --- | --- | --- |
| 사용자 활성화/비활성화 | on, off | on: 사용자 활성화, off: 사용자 비활성화 (이미 인증된 연결은 계속 작동) |
| 명령어 접근 제어 | +<command>, -<command> | +<command>: 특정 명령어 허용, -<command>: 특정 명령어 거부. Redis 7.0부터 서브커맨드 차단 가능 (예: `-config |
| 카테고리별 접근 제어 | +@<category>, -@<category> | +@<category>: 특정 카테고리의 모든 명령어 허용, -@<category>: 특정 카테고리의 모든 명령어 거부. 예: @admin, @set, @sortedset 등 |
| 키 패턴 제한 | ~<pattern>, %R~<pattern>, %W~<pattern> | ~<pattern>: 특정 패턴의 키 접근 허용. Redis 7.0부터 읽기/쓰기 키 패턴 (%R~, %W~) 지원 |
| Pub/Sub 채널 접근 제어 | &<pattern>, allchannels, resetchannels | &<pattern>: 특정 패턴의 Pub/Sub 채널 접근 허용. allchannels는 모든 채널 허용, resetchannels는 채널 패턴 목록 초기화 |
| 비밀번호 설정 | ><password>, #<hash>, nopass, resetpass | ><password>: 비밀번호 추가, #<hash>: SHA-256 해시 값으로 비밀번호 추가. nopass: 비밀번호 없이 접근 허용, resetpass: 비밀번호 목록 초기화 |
| 선택기 설정 | (<rule list>), clearselectors | Redis 7.0부터 사용 가능. (<rule list>): 새 선택기 생성 및 규칙 매칭, clearselectors: 모든 선택기 삭제 |
| 사용자 초기화 | reset | resetpass, resetkeys, resetchannels, allchannels, off, clearselectors, -@all을 포함하여 사용자를 초기 상태로 재설정 |
- 카테고리: 명령어의 집합

```bash
# 모든 카테고리목록을 보여준다.
ACL CAT 
```

- @read: 데이터를 읽는 명령어의 집합
- @write: 데이터를 쓰는 명령어들의 집합
- @dangerous: 보안이나, 데이터 무결성에 영향을 줄 수 있는 명령어 집합

### Command 실행환경 제어

- 7버전부터, Command가 실행될 수 있는 환경도 제어가 가능하다.
    - Redis Server가 실행중일 때 변경되면 위험한 설정은 실행불가능하게 막았다.
    - redis.conf에 설정 가능하다.
- 위험한 설정을 실행중에 변경 하는 것을 enable / disable / 로컬에서만 허용 등의 설정이 가능하다.
    - no | yes | local의 설정이 가능하다.

```bash
enable-protected-configs no # CONFIG SET, CONFIG REWRITE, CONFIG RESETSTAT, ...
enable-debug-command no     # DEBUG SETFAULT, DEBUG RELOAD, DEBUG OBJECT, ...
enable-module-command no    # MODULE LOAD, MODULE UNLOAD, MODULE LIST, ...
```

### User 생성 및 삭제

```bash
# 유저 생성 
#(user-name: tj, password: password, 접근가능 Key Pattern: cached:*, pubsub: *, dangerous 카테고리 제외)
ACL  SETUSER tj on >password ~cached:* &* +@all -@dangerous

# 유저 삭제
ACL DELUSER

# 유저 ACL 목록 조회
ACL GETUSER <user-name>
```