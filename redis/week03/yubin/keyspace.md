# Keyspace

## Altering and querying the key space

- `EXISTS`
    - 키가 존재한다면 1 반환
- `DEL`
    - 키가 존재했다면 1 반환, 없었다면 0 반환
- `TYPE`
    - 키의 타입 반환
        - 존재하지 않는다면 “none” 반환

## Key expiration

- TTL 설정
- 만료에 대한 정보는 디스크에 복제/저장됨
- 레디스 서버가 멈추더라도 만료시간은 가상으로 지나감 (레디스가 키의 만료시각을 저장하기 때문)
- `expire ‘key’ ‘time’`
- `set ‘key’ ‘value’ ex ‘time’`

# Navigating the keyspace

## Scan

- 레디스 디비의 키들을 iterate하고싶을때
- 서버를 오래동안 블락시킬 수 있는 `KEYS` , `SMEMBERS` 와 달리 블락시키지 않고 iterate하는 scan 명령어는 iteration 과정에서 collection이 바뀌었을수 있음

## Keys

- keyspace를 도는 또다른 방법
- 레디스 서버를 블락시킴
- 프로덕션 환경에서 사용할때는 신중을 기해야함
- 보통 디버깅 용도로 사용