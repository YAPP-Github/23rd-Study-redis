```
Redis 는 caching, queueing, event processing 에 주로 사용된다.
```

# String
- Redis String 은 바이트 시퀀스를 저장
    - 문자열이면 뭐든 저장 가능(이미지 등)
- HTML 조각이나 페이지 캐싱과 같은 다양한 사용 사례에 유용
- get, set 명령어를 이용해 data 를 읽고 씀
    - set 은 기본적으로 이미 key-value 가 존재해도 overwrite 함
    - `nx=True` 옵션을 주면 overwrite 방지
```python
res3 = r.set("bike:1", "bike", nx=True)
print(res3)  # None
print(r.get("bike:1"))  # Deimos
res4 = r.set("bike:1", "bike", xx=True)
print(res4)  # True
```
- mget, mset 명령어는 단일 명령으로 bulk 처리
```python
res5 = r.mset({"bike:1": "Deimos", "bike:2": "Ares", "bike:3": "Vanth"})
print(res5)  # True
res6 = r.mget(["bike:1", "bike:2", "bike:3"])
print(res6)  # ['Deimos', 'Ares', 'Vanth']
```
- incr, incrby, decr, decrby 명령어로 `원자적`으로 정수값 핸들링 가능
```python
r.set("total_crashes", 0)
res7 = r.incr("total_crashes")
print(res7)  # 1
res8 = r.incrby("total_crashes", 10)
print(res8)  # 11
```
- 최대 512MB 까지 저장 가능
- 대부분의 문자열 연산은 O(1)이므로 매우 효율적
- 그러나 O(n)이 될 수 있는 SUBSTR, GETRANGE, 명령에는 주의

# JSON


# Lists
- 양방향 연결 리스트
- LPUSH / RPUSH
- LPOP / RPOP
```python
res1 = r.lpush("bikes:repairs", "bike:1")
print(res1)  # >>> 1

res2 = r.lpush("bikes:repairs", "bike:2")
print(res2)  # >>> 2

res3 = r.rpop("bikes:repairs")
print(res3)  # >>> bike:1

res4 = r.rpop("bikes:repairs")
print(res4)  # >>> bike:2
```
- LLEN
- LMOVE
```python
res12 = r.lmove("bikes:repairs", "bikes:finished", "LEFT", "LEFT")
print(res12)  # >>> 'bike:2'
```
- LRANGE
```python
res50 = r.lrange("bikes:repairs", 0, -1)
print(res50)  # >>> ['bike:5', 'bike:4', 'bike:3']
```
- LTRIM
```
res49 = r.ltrim("bikes:repairs", 0, 2)
print(res49)  # >>> True
```

# Sets
- 일반적인 set 과 동일
- SADD -> add
- SREM -> remove
- SISMEMBER -> in
```python
r.sadd("bikes:racing:france", "bike:1", "bike:2", "bike:3")
r.sadd("bikes:racing:usa", "bike:1", "bike:4")
res5 = r.sismember("bikes:racing:usa", "bike:1")
print(res5)  # >>> 1

res6 = r.sismember("bikes:racing:usa", "bike:2")
print(res6)  # >>> 1 <- 왜 1임? 오타인가?
```
- SINTER -> &
- SCARD -> len()
- SMEMBERS -> print all
- SMISMEMBER -> mget 과 비슷
- SDIFF -> difference, -
- SUNION -> union, |
- SPOP -> pop
- SRANMEMBER
- 최대 크기: 2^32 - 1(4,294,967,295)
- 추가, 제거, 항목이 집합 멤버인지 확인하는 등 대부분의 집합 작업은 O(1)
- 대규모 세트의 경우 명령을 실행할 때 주의 해야함 SMEMBERS 는 이 경우 `O(n)`

# Hashes
- key 에 구조화된 데이터를 저장할 수 있음
```python
res1 = r.hset(
    "bike:1",
    mapping={
        "model": "Deimos",
        "brand": "Ergonom",
        "type": "Enduro bikes",
        "price": 4972,
    },
)
print(res1)
# >>> 4

res2 = r.hget("bike:1", "model")
print(res2)
# >>> 'Deimos'

res3 = r.hget("bike:1", "price")
print(res3)
# >>> '4972'

res4 = r.hgetall("bike:1")
print(res4)
# >>> {'model': 'Deimos', 'brand': 'Ergonom', 'type': 'Enduro bikes', 'price': '4972'}
```

# Sorted sets
- 그냥 sets 는 순서가 없지만 Sorted sets 는 이름처럼 순서가 존재하고 저장할때 정렬 된다
- 대부분의 정렬 집합 연산은 O(log(n))
- ZRANGE 큰 반환 값을 사용하여 명령을 실행할 때는 주의
    - 명령의 시간 복잡도는 O(log(n) + m) 
    - 여기서 m 은 반환된 결과 수

# Streams
- Redis 5.0 부터 도입
- append-only
- 시간 순서대로 데이터를 저장
- 다양한 소스에서 발생하는 이벤트나 메시지를 실시간으로 처리하는데 매우 유용


# Geospatial
- 좌표 저장
- 주어진 반경이나 경계 내에서 가까운 지점을 찾는데 유용
```python
res1 = r.geoadd("bikes:rentable", [-122.27652, 37.805186, "station:1"])
print(res1)  # >>> 1

res2 = r.geoadd("bikes:rentable", [-122.2674626, 37.8062344, "station:2"])
print(res2)  # >>> 1

res3 = r.geoadd("bikes:rentable", [-122.2469854, 37.8104049, "station:3"])
print(res3)  # >>> 1

res4 = r.geosearch(
    "bikes:rentable",
    longitude=-122.27652,
    latitude=37.805186,
    radius=5,
    unit="km",
)
print(res4)  # >>> ['station:1', 'station:2', 'station:3']
```

# Bitmaps
- 비트 배열로 작동하는 데이터 구조
- 간단한 단일 비트의 읽기/쓰기
- 문자열의 형태로 저장
- 크기가 매우 작은 공간에서 많은 불린 데이터를 저장하는데 효율적
- 사용자 로그인 상태 추적, DAU 계산 등

# Bitfields
- 비트맵의 확장된 형태, 더 복잡한 비트 수준의 조작 가능
- 여러 비트를 그룹화하여 복잡한 방식으로 조작
- Bitfields를 사용하면 여러 비트를 하나의 정수로 취급하여 읽고 쓸 수 있으며, 비트 필드 내에서 정수형 데이터를 저장하고 검색할 수 있음
- 임의의 비트 길이의 정수 값 사용(부호 없는 1비트 정수부터 부호 있는 63비트 정수까지)
- 바이너리로 인코딩된 Redis 문자열을 사용하여 저장
- 원자성 읽기, 쓰기 및 증분 작업을 지원
- O(n)
    - 여기서 n 은 액세스된 카운터 수