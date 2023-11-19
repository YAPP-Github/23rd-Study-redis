# BitMap
- 실질적인 DataType이라고 볼수는 없다.
- String에 정의되고, 일종의 BitVector처럼 사용된다.
- 적은 공간을 사용하여, 다량의 Flag를 관리하는데 용이하다.
- 512MB까지 정의 가능하다.

## 주요 Command

### [1] SETBIT
```shell
SETBIT [BIT-KEY] [OFFSET] [VALUE]
```
- O(1)
- bitMap에 flag를 설정한다.
  - 1: enable
  - 0: disable

### [2] GETBIT
```shell
GETBIT [BIT-KEY] [OFFSET]
```
- O(1)
- OFFSET에 해당하는 Bit를 가져온다.

### [3] BITCOUNT
```shell
BITCOUNT [BIT-KEY] {START STOP}
```
- O(N)
- BitMap에서 Enable된 Bit의 수를 리턴한다.

### [4] BITPOS
```shell
BITPOS [BIT-KEY] [0|1] {START STOP}
```
- O(N)
- 해당되는 bit가 처음 나타내는 Position을 리턴한다.

### [5] BITOP
```shell
BITOP [OPERATION] [RESULT-KEY] [KEY1] [KEY2]
```
- O(N)
- Bit연산을 수행한다.
  - XOR
  - OR
  - AND
  - NOT
- RESULT-KEY에 KEY1과 KEY2를 Bit연산한 결과를 저장한다.