# SortedSet
- prefix는 z
- set에 정렬이가능한 score를 추가한 것
    - 기본적으로 Score 내림차순이다.
    - score가 동일하다면, key값에 따라서 “사전순”으로 정렬된다.
- RankingBoard, RateLimiter등에 사용 가능하다.
- SkipList와 HashTable로 구성되어있다.
- 대부분의 연산이 수행되는데 O(logN)의 시간복잡도를 가진다.

## 주요 Command

### [1] ZADD
```bash
ZADD [KEY] [SCORE][MEMBER]
```
- O(logN)
- SortedSet에 Member를 추가한다.

### [2] ZRANGE
```bash
ZRANGE [KEY] [START] [STOP] {OPTIONS}
```
- O(log(N)+M)
- 기본적으로 Score를 기준으로 오름차순으로 정렬되어있다.
- START와 STOP은 Index를 나타낸다.
    - ‘[’  : 포함 (default)
    - ‘(‘: 미포함
- -1은 END부터 첫번째 요소를 나타낸다.
    - ZRANGE [KEY] 0 -1 은 전체요소를 나타낸다.
- OPTIONS
    - WITHSCORES: 점수도 함께 출력
    - REV: 내림차순으로 출력
    - BYSCORE: 점수를 기준으로 조회
        - O(logN)
    - BYLEX: lexicographical의 약자, Value를 사전순으로 정렬한다
        - 정렬 시, Score는 무시된다.
        - O(logN+M)
            - N: 정렬해야 할 요소들의 수
            - M: 반환하는 요소들의 수
    - LIMIT OFFSET COUNT
        - offset 부터 시작하여 count 갯수를 가져온다.

### [3] ZINCRBY
```bash
ZINCRBY [KEY] [SCORE] [MEMBER]
```
- Score를 증가시킨다.
- 값이 존재하지 않는다면, 0.0을 기준으로 증가시킨다. (새롭게 Insert)

### [4] ZRANK
```bash
ZRANK [KEY] [MEMBER] {WITHSCORE}
```
- O(logN)
- 순위를 가지고온다.
    - 순위는 0부터 시작한다.
- 없으면 nil이 뜬다.
- 반대 명령어는 **ZREVRANK** 이다.

### [5] ZREM
```bash
ZREM [KEY] [...MEMBERS]
```
- O(M * logN)
    - 하나를 삭제하는데 O(logN)의 시간이 걸리고, 삭제할 갯수 M을 곱한만큼 걸린다.

### [6] ZREMRANGEBYSCORE
```bash
ZREMRANGEBYSCORE [KEY] [MIN] [MAX]
```
- O(logN+M)
- Score를 기반으로 삭제한다
- -inf ~ inf까지 지원한다.
    - 위 옵션 사용시 모두 지운다.

### [7] ZCARD
```bash
ZCARD [KEY]
```
- O(1)
- SortedSet에 존재하는 Member들의 수를 리턴한다.

### [8] ZCOUNT
```bash
ZCOUNT [KEY] [MIN] [MAX]
```
- O(logN)
- Score내에 있는 요소의 갯수를 리턴한다.
- 오직 Score만 지원한다. (byLex는 지원하지 않는다)