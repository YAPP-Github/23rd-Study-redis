## String

- 캐싱에 많이 씀 (HTML 같은 문자열 저장)
- binary도 가능 (jpeg 파일 등등...)
- 최대 512MB
- `SET` 명령어는 기존 키가 존재하면 덮어씀. (기존 키의 값이 String이 아니어도 덮어씀)
- `SET` 명령어는 다양한 옵션 지원
  - 키 존재하면 실패 (nx)
  - 키 존재하지 않으면 실패 (xx)
- `SET` 을 하는 동시에 기존 값을 `GET` 하는 `GETSET` 도 존재 (두 동작을 atomic하게 처리할 수 있음)
- 여러개를 한번에 처리하는 `MSET`, `MGET` 도 있음 (latency 줄이는 용도)
- `INCR`, `INCRBY`, `DECR`, `DECRBY` 은 atomic함 -> 동시성 문제가 생기지 않음을 보장
- 대부분 동작이 O(N)
  - `SUBSTR(depracated)`, `GETRANGE`, `SETRANGE` 는 O(N)이라 주의
- 구조화된 데이터를 serialize해서 저장하는거라면 [Hash](#Hash)나 [JSON](#JSON)이 나을 수 있음

### 의문

- `SET` with option nx vs `SETNX` ?

## JSON

- index, query 지원 (Redis Stack 설치, Redis v6.x 이상 + RedisSearch v2.2, RedisJSON v2.0 이상)
  - 벡터 임베딩 인덱싱도 지원되는게 신기함
- 최대 depth가 128 (초과시 에러 뱉음)
- 그냥 [String](#String)에 저장하는 것에 비해 이점
  - 값 전체를 클라에 전송하지 않고도 하위의 nested된 값을 조회할 수 있음
  - atomic한 일부분 업데이트 지원 (값 전체를 조회하여 수정 후 다시 serialize하는 것 보다 적은 비용으로 처리 가능)
  - Index & Query

## List

- String 값의 Linked List
- 주 용도
  - stack & queue 구현
    - SNS 최근 업데이트된 포스트 목록 캐싱
    - Capped List
      - 최신 N개 항목만 기억
      - `LTRIM` 사용하여 구현 가능 (단, O(N)이라 주의)
  - worker queue
    - 프로세스간 통신 (consumer-producer 패턴)
    - blocking operations
      - `BRPOP`, `BLPOP`
      - list에 요소가 추가되거나, 지정한 timeout에 도달했을 때에만 caller에게 반환
- `LPUSH`, `LPOP`, `RPUSH`, `RPOP` 으로 좌/우에 값을 넣고 뺄 수 있으며, 한번에 여러개를 넣을 수도 있음. (단, N개 넣을 때 O(N))
- `LTRIM`, `LRANGE` 인덱스로 시작, 끝(해당 위치도 포함)을 인자로 받으며, 음수 인덱스도 가능함. (-1이 마지막)
- `LPUSH`에 사용하는 키가 존재하지 않으면 빈 list를 자동으로 생성하는 것 처럼 동작
- List의 최대 길이는 2^32 - 1 (4,294,967,295)임
- 불확정적인 일련의 이벤트를 저장하고 처리해야 할 때 List에 대한 대안으로 [Stream](#Stream)을 고려 (불확정적: 길이 또는 종점을 사전에 정확하게 예측할 수 없다는 뜻 = 길이가 미확정)

### 의문

- Redis는 싱글스레드인데 block될 경우 다른 operation에 영향을 안 미치는건가?

## Set

- 정렬되지 않은 문자열의 모음 (자료구조 Set과 동일한 역할)
- 주 용도
  - 유니크한 값 추적 (ex. 로그 게시물에 액세스하는 모든 고유 IP 주소 추적)
  - 연관 관계 표현 (ex. 특정 권한을 가진 모든 사용자 집합)
  - 집합 연산 수행
    - intersection (`SINTER`)
    - union (`SUNION`)
    - difference (`SDIFF`)
- Set의 최대 길이는 2^32 - 1 (4,294,967,295)임
- `SMEMBERS`는 O(N)이므로 사용 주의 (대신 iterative하게 반환할 수 있는 `SSCAN` 권장)

## Hash

- 필드 값 쌍의 컬렉션으로 구조화된 레코드
- Hash는 객체를 표현하는 데 편리하지만, 실제로 해시 안에 넣을 수 있는 필드 수에는 사용 가능한 메모리 외에는 실질적인 제한이 없음 = 다양한 방식으로 해시를 사용할 수 있음
- 작은 해시(즉, 작은 값을 가진 몇 개의 요소)는 메모리에서 특수한 방식으로 인코딩되어 메모리 효율이 매우 높음
- 최대 4,294,967,295개(2^32 - 1)의 필드 값 쌍을 저장할 수 있음

## Sorted Set

- score에 따라 정렬된 고유한 문자열(member)의 모음
- 주 용도
  - Leaderboard
  - Rate limiter (sliding-window rate limiter)
- 둘 이상의 문자열에 동일한 점수가 있는 경우 문자열은 사전순으로 정렬됨 (두 문자열은 서로 달라야 함)
- skip list와 해시 테이블을 모두 포함하는 dual-ported 자료 구조를 통해 구현됨
  - 요소를 추가할 때마다 Redis는 O(log(N)) 연산을 수행
  - 요소를 조회할 때는 O(1)
- `ZRANK`, `ZREVRANK` 명령어로 순위를 조회할 수 있음
- `ZRANGE`는 O(log(n) + m)임 (m = 반환하는 member 개수)

## Stream

- log 파일처럼 append only로 저장되는 구조
- 메시징 시스템인 Kafka와 비슷하게 동작하며, 메시지를 읽는 Consumer와 여러 Consumer를 관리하는 Consumer Group 또한 지원
- Stream을 사용하여 이벤트를 실시간으로 기록하고 동시에 발행할 수 있음
- 주 용도
  - 이벤트 소싱 (ex. tracking user actions, clicks, etc.)
  - 센서 모니터링 (ex. 디바이스 판독값)
  - 알림 (ex. 각 사용자의 알림 기록을 별도의 스트림에 저장)
- `XADD <stream-key> <message-id> <field> <name> ... <field> <name>` 와 같은 형태로 추가
  - `<message-id>`는 \*로 자동 생성 가능
  - 자동 생성된 id는 `<millliiseconds>-<sequenceNumber>`의 조합으로 되어 있음
  - 자동 생성된 id를 이용해 message를 time-range로 조회할 수 도 있음
  - O(1)
- single entry 조회는 O(n) (n = ID의 길이)
  - radix tree로 구현되어 거의 상수 시간에 조회 가능
- Subscribe(구독)형식으로 접근을 할때는 `XREAD` 사용
  - Stream은 데이터를 기다리는 여러 consumer들을 가질수 있음. 새로운 아이템은 데이터를 기다리는 모든 consumer에게 전달
  - 각 consumer가 다른 element를 받는 Blocking List와 다름. 하지만 fan-out 기능은 Redis Pub/Sub과 비슷
  - Redis Pub/Sub은 fire & forget으로 메시지 발행 후 저장되지 않는 형태, Blocking List을 사용할땐 client가 메시지를 받으면 popped. 하지만 Redis Stream은 모든 메시지는 append 되는 형태로, 다른 consumer들은 마지막으로 받은 메시지 id를 기억하는 걸로 새로운 메시지가 온걸 알 수 있음
  - 기본 non-blocking. blocking 옵션을 줄 수도 있음
- Consumer Group
  - 동일한 스트림에서 여러 클라이언트에 서로 다른 메시지 하위 집합을 제공
- `XPENDING`을 통해 pending상태의 메시지 조회를 지원하고, `XCLAIM`을 통해 메시지의 재처리를 지원
- `XCLAIM`을 이용하면 해당 message의 consumer를 바꿀 수 있음

## Geospatial

- geospatial index를 통해 좌표를 저장하고 검색할 수 있음
- `GEOADD`: geospatial index에 좌표 추가. longitude,latitude 순서대로 입력 (O(log(n)))
- `GEOSEARCH`: 지정한 범위 내의 radius 또는 bounding box를 기준으로 검색 (O(n + log(m)))
  - 경계 상자 영역 내의 요소 수(n)와 모양 안의 아이템 수(m)
  - 경계 상자 영역과 모양 안의 차이
    - 경계 상자 영역 (Bounding Box Area): 이것은 주어진 모양을 완전히 둘러싸는 가장 작은 직사각형을 말합니다. 이 상자는 그리드에 정렬되어 있으며, 모양의 외곽선을 따라가지 않고, 모양을 포함하는 직사각형 형태를 취합니다. 이 상자 안에는 모양 자체뿐만 아니라, 모양 주변의 추가 공간 또한 포함될 수 있습니다.
    - 모양 안 (Inside the Shape): 이는 특정한 지오메트릭 모양(예를 들어 원, 다각형 등)의 경계 안에 있는 공간을 의미합니다. 이 경우에는 모양의 경계선에 정확히 일치하는 영역만을 고려합니다. 즉, 모양의 실제 형태에 따라 정의되는 내부 공간을 말합니다.
  - `BYRADIUS`, `BYBOX` 로 모양을 결정할 수 있음

## Bitmaps

- 실제 데이터 타입이 아님
- String 타입에 대한 bit-oreiented operations의 집합임
  - String이 최대 512MB이므로 2^32개의 서로 다른 비트를 가질 수 있음
- 주 용도
  - 멤버가 정수 0~N에 매핑되는 경우 효율적으로 상태를 표현할 수 있음
  - 객체 권한: 파일 시스템이 권한을 저장하는 방식과 유사하게 각 비트가 특정 권한을 나타냄
- 정보를 저장할 때 공간을 크게 절약할 수 있음
  - 예를 들어, 서로 다른 사용자가 증분 사용자 ID로 표시되는 시스템에서는 512MB의 메모리만으로 40억 명의 사용자에 대한 단일 비트 정보(ex. 사용자가 뉴스레터 수신 여부를 파악하는 정보)를 기억할 수 있음
- `SETBIT`, `GETBIT` = O(1)
- `BITOP` = O(n) (n = 비교하는 String 둘 중 더 큰 길이)

## Bitfields

- 임의의 비트 길이의 정수 값을 set, increment, get할 수 있음
  - 1bit unsigned 정수부터 63bit signed 정수까지 가능
- atomic read, write, increment operations 지원 (counter로 사용하기 좋음)
