# Set
- prefix는 s
- 순서가 없는 데이터구조
- 중복없이 Element를 관리하는 것이 목적이다.

## 주요 Command

### [1] SADD
- O(1)
- Set에 데이터 추가

```bash
SADD [KEY] [...MEMBERS]
```

### [2] SREM
- O(N) 
  - N은 제거되는 요소의 갯수
- Set에서 요소 제거

```bash
SREM [KEY] [...MEMBERS]
```

### [3] SISMEMBER
```bash
SISMEMBER [KEY] [MEMBER]
```
- O(1)
- Set에 요소가 있는지 Check
    - 있으면 1, 없으면 0

### [4] SINTER
```bash
SINTER [...KEYS]
```
- SET간의 교집합을 반환한다.
- O(N*M) → N은 가장작은 Set의 크기, M은 Set들의 갯수

### [5] SCARD
```bash
SCARD [KEY]
```
- SET에 저장되어있는 요소의 갯수를 구한다.
- O(1)


### [6] SPOP
```bash
SPOP [KEY]
SPOP [KEY] [COUNT]
```
- Set에서 **무작위** 요소 삭제
  - 3.2부터 N개 꺼내올 수 있다.


### [7] SMOVE
```bash
SMOVE [SRC_SET_KEY] [DEST_SET_KEY]
```
- Source에 해당하는 Set의 요소를 Destinaton으로 옮김
- DEST에, SRC의 것을 추가하고, 중복을 제거


## 쓰면안되는 Command
### [1] SMEMBERS
```bash
SMEMBERS [KEY] [SORT]
```
- O(N)
- Set에 포함되어있는 모든 요소들을 가져온다.
    - SSCAN을 사용하라고하지만, 이것도 결국은 O(N  / M(chunkSize))기 때문에 안쓰는게 좋아보임
    - SMEMBERS를 ChunkSize로 잘게쪼개서 도는 방식으로 이해함


### [2] SUNION
```shell
SUNION [...SET_KEYS]
```
- O(N1+…+Nk)
- SET들의 합집합을 구한다.

### [3] SDIFF
```bash
SDIFF [KEY1] [keys...]
```
- N = 모든 Set들의 요소를 더한 크기
- KEY1에만 존재하는 것들을 구한다.

### [4] SINTER
```bash
SINTER [KEYS]
```
- SET간의 교집합을 반환한다.
- O(N*M) → N은 가장작은 Set의 크기, M은 Set들의 갯수