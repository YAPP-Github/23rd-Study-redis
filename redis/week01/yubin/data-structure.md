## Strings

- 키가 이미 존재하는 경우 성공/실패 여부를 제어할 수 있음
    - `set key newValue nx` → 실패
    - `set key newValue xx` → 성공

        ```jsx
        127.0.0.1:6379> set key1 1
        OK
        127.0.0.1:6379> set key1 2 nx
        (nil)
        127.0.0.1:6379> get key1
        "1"
        127.0.0.1:6379> set key1 2 xx
        OK
        127.0.0.1:6379> get key1
        "2"
        ```

- `incrby` 를 통해서 소수점을 더하려고 할 경우 에러

    ```jsx
    127.0.0.1:6379> set key2 "1"
    OK
    127.0.0.1:6379> incrby key2 1.5
    (error) ERR value is not an integer or out of range
    ```

- `setnx`
    - 키가 이미 존재하는 경우 아무 작업도 수행하지 않음
    - `lock`을 구현하는데에 유용
- docs
    - 512MB이하만 가능
    - 어떠한 binary data도 가능
    - 성능
        - 대부분의경우 O(1)
        - 그러나 substr, getrange, setrange같은 경우 O(n)

## JSON

- `$` : json document의 path
- json의 요소에 추가,삭제 가능
- commands
    - `json.set animal & “dog”` : animal에 dog를 집어넣어라

## Lists

- redis의 list는 arraylist가아닌, `linkedlist`
    - linked list이기 때문에 추가/삭제 연산이 일정적인 시간으로 수행됨 O(1)
    - 그러나 조회는 느림
    - 데이터베이스에서 매우 긴 리스트를 빠르게 추가하는 것은 매우 중요한 요소이므로, 레디스는 linkedlist를 선택
    - 조회가 빠르게 이루어져야하는 경우, 인덱스 기반의 접근을 제공하는 `Sorted sets`를 사용하자.
- 리스트를 사용해 스택,큐 구현 가능
- Blocking commands
    - `Producer-Consumer`에서 리스트에 element가 없으면 Consumer는 리스트가 차거나, timeout이 발생할때까지 기다리게할 수 있음
        - 그냥 nil을 반환할 경우 지속적으로 Polling방식으로 consuming을 요구하는 과정에서 불필요한 네트워크 I/O가 발생할 수 있기때문
    - `BLPOP` : 리스트의 head에서 element를 제거하고 반환함. 이때 리스트가 비어있다면 리스트가 차거나, timeout에 도달할때까지 blocking됨
    - `BLMOVE`
- 버전 3.2 이전
    - `Zip List`
        - element가 작을때(각 element가 64byte이하, 총 갯수 128개 이하)
        - 메모리 절약에 최적화된 구조
    - `Linked List`
        - 엔트리 갯수 512 초과할때 사용
        - 메모리 오버헤드 (한 엔트리당 앞뒤포인터,밸류 저장)
- 버전 3.2~
    - `Quick List`로 통합
    - 하나의 quick list node가 ziplist를 갖는 구조 (맨앞/맨뒤의 노드 몇개를 제외하고 남은 가운데 노드들을 압축하는 등의 기법을 사용)

      <img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/ca777720-e480-48ee-8dfb-ca3d5c6451cf" width="400px">

    - Zip list에 중간 노드 삽입시 발생하는 메모리 재할당/복사/이동의 문제를 해결
    - Linked List의 오버헤드 문제 해결

## Sets

- 정렬되지 않는 자료구조
- 교집합, 합집합, 차집합 등등의 역할 수행
- SADD, SINTER, SUNION, SDIFF …
- 추가,제거,확인 → 모두 O(1)

## Hashes

- 필드-값 쌍으로 이루어진 컬렉션
- key-value의 value가 (key-value)s인 느낌
- hash에는 실질적으로 넣을 수 있는 필드 수에 한계가 없다 (VM의 메모리에 의해서만 제한됨)
- key,value들을 모두 가져오는 명령어(HKEYS, HVALS), HGETALL 등을 제외하면 대부분의 해시 명령어는 O(1)
- hash에서 column(?)을 추가하려면 사전작업 필요 없이 자유롭게 필드를 추가하고 삭제할 수 있음
    - 또한, 필드의 추가/삭제는 해당 key에만 영향을 미침

## Sorted sets

- 집합 내의 element가 score라는 float 수에 매핑되어있음
- 읽을 때 정렬되지 않고, 저장될때 정렬됨 (O(logN))
- 내부적으로 Skip List 사용
    - 원리
        - 링크드 리스트의 단점 : n번째 노드를 찾으려면 n번 비교해야함
        - 탐색시간을 줄이기위해 비교횟수를 줄였음
        - 하나의 노드가 여러개의 포인터를 가지고 스킵해가면서 탐색함
        - 예시) 레벨 3를 갖는 스킵 리스트

          <img src="https://github.com/YAPP-Github/23rd-Study-redis/assets/69676101/da6277d4-f7af-4a19-bfdb-e4031bd96509" width="400px">

            - 타겟이 80 → 처음에 레벨3가 가리키는 값 20이므로, 80보다 작으므로 이동 → 70과비교하는데 80보다 작으므로 이동 → 그다음 포인터가 null이므로 레벨을 하나 낮춰서 90을 보는데 80보다 큼 → 레벨을 하나 더 낮춰서 비교 → 80 발견 (총 비교횟수 4)

## Streams

- 로그파일처럼 append only임
- 이벤트 소싱, 센서 모니터링, 알림 등의 동작에 사용됨
- 각 스트림 항목에 대해 고유 id를 생성하여 이 id를 통해 검색등의 동작을 처리함
- 성능
    - 스트림에 항목을 추가: O(1)
    - 단일 항목에 접근: O(n) - 시간 지정으로 더 소요시간을 줄일 수 있음

## Geospatial

- 좌표 저장, 검색
- 주어진 반지름/영역 내에서 가까운 점을 찾는등의 목적으로 사용
- `GEOADD`, `GEOSEARCH`

## Bitmaps

- String 자료형에서 확장된 표현
- 정수 0부터 N으로 구성된 형태의 SET을 효율적으로 표현 가능
- 파일시스템의 권한(rwx)과 비슷하게, 각 비트가 특정한 역할을 갖도록 하는 등의 경우 사용됨
- 메모리 성능 개선 가능
    - 그러나 상대적으로 큰 오프셋을 가지면서 - 보관할 데이터의 대상이 적다면 bitmap보다 set을 사용하는게 메모리를 더 효율적으로 사용한다고함
    - 최대 유저 ID가 1백만이면서 특정 유저 10명만 bit를 1로 표현하는 경우
- 사용예시

  [유저 목록을 Redis Bitmap 구조로 저장하여 메모리 절약하기 - DRAMA&COMPANY](https://blog.dramancompany.com/2022/10/유저-목록을-redis-bitmap-구조로-저장하여-메모리-절약하기/)


## Bitfields

- 임의의 비트 길이의 정수를 set, increment, get할 수 있음
- 1비트 정수부터 63비트 정수까지 어느것이든 operate할 수 있음
- 성능 : O(n)