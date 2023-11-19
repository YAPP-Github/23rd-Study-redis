# GeoSpatial
- 지리적 위치 데이터를 효율적으로 저장하고 쿼리하는 데 사용
- 좌표를 저장하고 검색(변경, 경계)에 용이하다. 

## 주요 Command
### [1] GEOADD
 ```
 GEOADD [GEO-KEY] [LONGITUDE] [LATITUDE] [MEMBER]
 ```
- O(logN) (N은 Geo Set에 있는 요소의 수)
- 여러개 저장이 가능하다. 

### [2] GEODIST
```shell
GEODIST [GEO-KEY] [MEMBER1] [MEMBER2]
```
- O(logN)
- 두 위치 간의 거리를 계산

### [3] GEOHASH
```shell
GEOHASH [GEO-KEY ][...MEMBERS]
```
- O(1)
- Member를 geoHash 문자열로 반환
  - 일반 Hash문자열과 다르다.
  - 복호화가 가능하다.
  - 위치 데이터를 geoHash화 하면서 전송효율을 높인다.
- geoHash를 사용하여, DB에 데이터 Indexing 효율을 높일 수 있다.

### [4] GEORADIUS
```shell
GEORADIUS [GEO-KEY] [LONGITUDE] [LATITUDE] [RADIUS][m|km|ft|mi]
```
- O(N) (N은 결과에 반환되는 요소의 수)
- 반경 N [단위]에 있는 Member들을 반환한다.