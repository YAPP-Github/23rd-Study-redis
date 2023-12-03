# 레디스 Search and Query

# 기본 구성 요소

### Documents

- 정보의 기본 단위
- hash / json 등
- key name으로 식별

### Fields

- string, number, geo-location, vector등의 속성
- 필드를 인덱싱하여 효율적인 query and search 가능
- 모든 document가 같은 필드를 가질 필요 없음

### Indexing Fields

- 모든 필드를 인덱싱하는것은 불필요한 오버헤드
- 어떤 필드를 인덱싱할지 생각하자

### Schema

- 스키마로 정의된 인덱스 구조
- 필드/인덱싱 방법 정의

# 스키마 정의

- 필드, 타입, 인덱스되어야할지/그냥저장할지 등을 특정지음
- 스키마를 정의해서 검색 성능을 최적화할 수 있음

```
FT.CREATE **idx**
    **ON HASH**
    **PREFIX** 1 blog:post:
SCHEMA
    title TEXT WEIGHT 5.0
    content TEXT
    author TAG
    created_date NUMERIC **SORTABLE**
    views NUMERIC
```

- 외에도 인덱스 태그에 구분자를 첨가하거나, 단일 필드를 다양한 방법으로 인덱싱하기, 여러 접두사를 가진 index document생성, filter 등등의 기능이 있음

# 필드와 타입옵션

### Number

- 범위기반 쿼리, 특정 숫자 조건을 걸어 문서 검색 가능
- `FT.CREATE ... SCHEMA ... {field_name} NUMBER [SORTABLE] [NOINDEX]`
- `FT.SEARCH products "@price:[200 300]"` : 200~300인 document search
- 조건
    - @field:[min max]  ==  min ≤ x ≤ max
    - @field:[-inf max] == x ≤ max
    - -@field:[val]  ==  x ≠ val
    - …

### GeoFields

- 지리적 좌표 저장
- `FT.CREATE ... SCHEMA ... {field_name} GEO [SORTABLE] [NOINDEX]`
- **`@<field_name>:[<lon> <lat> <radius> <unit>]`**
    - `FT.SEARCH cities "@coords:[2.34 48.86 1000 km]"` : 2.34,48.86지점에서 1000키로 이내의 document

### VectorFields

- 외부 기계학습 모델에 의해 생성되는 부동소수점 벡터
- 텍스트/이미지 등의 비정형 데이터
- 레디스 스택을 사용해 코사인 유사도, 유클리드 거리, 내적 등의 검색
- `FT.CREATE ... SCHEMA ... {field_name} VECTOR {algorithm} {count} [{attribute_name} {attribute_value} ...]`
    - {algorithm} - FLAT(brute force algorithm), HNSW(hierarchical, navigable, small world algorithm)
    - {count} - 인덱스 속성 수

### TagFields

- 데이터 태그 혹은 레이블의 모음을 나타내는 텍스트 데이터를 저장할 때 사용
- 텍스트 필드와 달리 토큰화되거나 stemming되지 않음
    - stemming : 단어의 기본 형식을 인덱스에 추가
        - “hiring”을 검색할때 hired, hire도 return하도록 함
- 데이터를 구성하고 분류하는데 유용
- `FT.CREATE ... SCHEMA ... {field_name} TAG [SEPARATOR {sep}] [CASESENSITIVE]`
    - CASESENSITIVE : 대/소문자 구분여부 (디폴트는 구분하지 않음)

### TextFields

- Redis Stack은 text field를 인덱싱할때 검색 기능을 최적화하기위해 여러 변환을 함
    - 모두 소문자로 바꿔서 대소문자구분 안하게 하기도 하고
    - 토큰화하기도 하고
    - 가중치를 부여할 수도 있고
    - 정렬할 수도 있고
    - 기준에 따라 검색 결과를 정렬할수도 있음
- `FT.CREATE ... SCHEMA ... {field_name} TEXT [WEIGHT] [NOSTEM] [PHONETIC {matcher}] [SORTABLE] [NOINDEX] [WITHSUFFIXTRIE]`

# 파라미터 구성

- 파라미터 중 일부는 로드 시간에만 설정할수있고, 어떤 파라미터는 런타임에도 설정 가능
- 사용할만한거 위주로 정리함
- 타임아웃 설정
    - `$ redis-server --loadmodule ./redisearch.so TIMEOUT 100` (밀리초)
    - 타임아웃 초과시 지금까지 누적된 상결과를 반환하거나 정책에 따라 설정된 오류를 반환
- 타임아웃 초과시 정책 설정
    - `$ redis-server --loadmodule ./redisearch.so ON_TIMEOUT {parameter}`
        - RETURN : 누적한 상위 결과 반환
        - FAIL : 오류 반환
- 병렬 쓰기 모드 설정
    - 토큰화 된 부분에 한하여 병렬로 쓰기 실행
    - 실제 쓰기 연산은 Redis Global Lock 필요
    - `$ redis-server --loadmodule ./redisearch.so CONCURRENT_WRITE_MODE`
- 검색 결과 갯수 제한
    - 반환할 최대 결과 수 제한
    - `$ redis-server --loadmodule ./redisearch.so MAXSEARCHRESULTS 3000000`

# Indexing

### Json 스키마로 인덱스 생성

- `FT.CREATE {index_name} ON JSON SCHEMA {json_path} AS {attribute} {type}`

### Json 문서 추가

- `JSON.SET ~`

### 인덱스 검색

- `FT.SEARCH itemIdx '@name:(earbuds)’`
    - 이름에 “earbuds”가 있는 항목 검색

### 그외

- JSON배열을 tag/텍스트/숫자/geo/vector 등으로 인덱싱 가능

# Query

### Basic syntax

- quote
    - 정확한 문구는 따옴표로 감싸짐
- multiword phrase
    - 다중 단어 구문은 토큰의 리스트임
    - AND를 암시함
- or union
    - pipe ( | )로 표현
- not
    - -로 표현
- *
    - prefix/infix/suffix 매칭 쿼리
    - prefix matching
        - 주어진 접두사로 시작하는 모든 용어
        - `hel* world`
    - infix/suffix matching
        - `*sun* *ing`
- wild card matching
    - ? : single character
    - * : 0이상 반복되는 문자
    - ‘와 \ : for escaping