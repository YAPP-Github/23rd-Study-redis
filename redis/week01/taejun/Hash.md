# Hash
- prefix는 h
- flat-structure
    - Json과 같이 계층형 구조를 가질 수 없다.
- key-value 쌍의 컬렉션으로 구성된 레코드 유형
- 간단한 객체구조를 표현 할 수 있다.
- 필드 수에 대한 제한은 없다.

### UseCase
- Session Storage
- Data Storage (진짜 DB용으로 사용)

## 주요 Command
### [1] HSET
```bash
HSET KEY [...field value] {OPTION}# hset key f1 v1 f2 v2 {NX | XX}
```
- Hash에 값을 저장하는 것이다.
    - 한번에 N개의 쌍을 저장가능하다.
- Update와 Insert 모두 가능하다.
- 리턴 값은 새롭게 생성된 Field:Value의 갯수이다.
    - Update의 경우 0이 리턴

### [3] HGET
```bash
HGET [KEY] [FIELD]
```
- O(1)
- 단일 field-value 리턴

### [4] HMGET
```bash
HMGET [KEY] [...FIELD]
```
- O(N)
    - N은 검색하는 field의 갯수
- Hash에서 여러개의 필드에 대한 값을 검색하는 것이다
- 없으면 nil 리턴

## 주의해야할 명령어

### [1] HKEYS
```bash
HKEYS [KEY]
```
- O(N)
- Hash에 저장되어있는 모든 field를 가져온다.

### [2] HVALS
```bash
HVALS [KEY]
```
- O(N)
- Hash에 저장되어있는 모든 value를 가져온더.

### [3] HGETALL
```bash
HGETALL [KEY]
```
- O(N)
- Hash에 저장되어있는 모든 field-value 쌍을 가져온다. (HKEYS + HVALS)