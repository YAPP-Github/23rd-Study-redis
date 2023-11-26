## String
- 512MB가 최대이다.
- Redis가 제공하는 가장 기본적인 DataType이다.
- BinarySafe하다. (모든 데이터를 저장 할 수 있다.)
- Redis가 제공하는 다른 DataType (List, Hash, Set, SortedSet)에 데이터가 적합하지 않다면, String이 좋은 대안일 것이다.
- 직렬화/역직렬화를 통해서 사용하면 된다.
- O(N)인 연산은 주의해야함 (Redis 공통)
  - ```text
    Most string operations are O(1), which means they're highly efficient.
    However, be careful with the [ SUBSTR, GETRANGE, and SETRANGE] commands, which can be O(n).
    These random-access string commands may cause performance issues when dealing with large strings.
    ```

### 비슷하게 동작할 수 있는 것
- JSON
- Hash

### 저장할 수 있는 것
- byte
- text
- Serailized Object
- ByteArray
- …

### UseCase
- Caching
- Static Page
- Counter
- Config

## 주요 Command

### [1] GET
```shell
GET [KEY]
```

### [2] GETDEL
```bash
GETDEL [key]
```
- GET 이후에 삭제

### [3] SET
```bash
SET key value [NX | XX] [GET] [EX seconds | PX milliseconds |

# key가 존재하지않을 때 key-value 설정하되, 200초의 TTL을 건다.
set [key] [value] nx ex 200
```
- set의 경우, 이미 데이터가 존재하면 덮어쓴다.
- 이를 방지하기 위한 다양한 옵션이 존재한다.
- 옵션
  1. nx: 존재하지 않을 때 SET (putIfAbsent)
  2. xx: 이미 존재할 때만 SET (Update)

### [4] MSET 
```shell
MSET [KEY] [VALUE] ...
```
- 한번에 여러개 Set을 한다. 
- atomic하다.
  - 하나라도 SET이 안되면 다 실패한다.
- msetnx도 존재한다.
  - 제공된 Key가 하나라도 이미 존재할 경우에는, 동작하지 않는다.

### [5] MGET
```shell
MGET [...KEYS]
```
- 한번에 여러개를 가져온다.
  - 없는 데이터의 경우 nil또는 null로 나온다.