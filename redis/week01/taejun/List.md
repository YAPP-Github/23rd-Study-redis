## [3] List
- String들의 리스트
- 순서가 지정된 요소들의 Sequence
- LinkedList이기 때문에, Index를 통한 접근은 올바르지 않다.
- insert 요청 시, 빈 list가 없다면 직접 생성하고, list가 비어있다면 삭제한다.

### UseCase
- Queue
- Stack

## 주요 Command
### [1] LPUSH
```shell
LPUSH [KEY] [VALUE]
```
- O(1)
- List왼쪽에 새로운 Element를 push

### [2] LPOP
```shell
LPOP [KEY]
```
- O(1)
- List 맨왼쪽에 있는 Element를 제거 후 리턴

### [3] LLEN
```shell
LLEN [KEY]
```
- O(1)
- List의 길이 (Element의 갯수)를 리턴

### [4] LMOVE
```bash
LMOVE [SOURCE_LIST_KEY] [DESTINATION_LIST_KEY] [LEFT | RIGHT] [LEFT | RIGHT]
```
- O(N)
- SourceList를 Desination List로 옮긴다.
- LEFT, RIGHT 옵션을 통해서 방향을 조절 할 수 있다.

### [5] LTRIM
```shell
LTRIM [KEY] [START] [STOP]
```
- O(N)
- START부터, STOP까지만 남기고 다 삭제한다.

### [6] LRANGE
```shell
LRANGE [KEY] [START] [STOP]
```
- O(N)
- 범위 검색이다.
  - 0은 왼쪽 끝, -1은 오른쪽 끝이다.

### [7] LINDEX
```shell
LINSERT [KEY] [INDEX]
```
- O(N)
- Index에 해당하는 Elemet를 반환한다.

### [8] LINSERT
```shell
LINSERT [KEY] [BEFORE | AFTER] [PIVOT ELEMENT]
```
- O(N)
- Pivot Elemet 앞 혹은 뒤에 Elemet를 추가한다.

### [9] LSET
```shell
LSET [KEY] [INDEX] [VALUE]
```
- O(N)
  - List의 첫번쨰 혹은 마지막 Elemet를 바꿀 때는 O(1)
- Index에 해당하는 Elemet의 Value를 비꾼다.

## Blocking 연산
- UseCase
    - MessageQueue와 같은 기능?
    - 실시간 Consume?
- Spin방식이 아닌 Event방식
    - 내부적인 EventLoop가 존재
- Polling을 하면되는데 왜 Blocking연산이 좋을 수도 있는가?
    - 일정시간을 Waiting후 접근하는데 이게 더 느릴 수 있다.
- TimeOut 또한 인자로받는데, TimeOut이 나거나, 실제 작업할 대상을 찾기 전까지 Blocking이 걸린다.
- Redis전체에 대한 Blocking이 아닌, 해당 Client(혹은 Session)에 Blocking이 걸린다.
- Consumer - Producer 패턴
    - 한쪽에서 push, 반대쪽에서 Consume → like RateLimiter
