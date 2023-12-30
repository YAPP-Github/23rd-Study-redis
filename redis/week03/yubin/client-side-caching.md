## Client-side caching in Redis

- client side caching의 이점
    - 매우 작은 지연 시간으로 데이터 접근
    - 디비가 쿼리를 덜 받음

## ****The Redis implementation of client-side caching****

- Tracking mode
    - default mode
        - 서버는 주어진 클라이언트가 접근한 키를 기억하고, 동일한 키가 수정되면 무효화 메시지를 보냄
        - 순서
            - 클라이언트를 tracking을 원할때 할 수 있음
            - 트래킹이 활성화되면 서버는 연결시간동안 클라이언트가 요청한 키를 기억하려고 함
            - 어떤 클라이언트에 의해 키가 수정되거나 TTL이 만료되었거나 최대치 메모리 정책에 걸린 경우 트래킹이 활성화 된 모든 클라이언트들은 invalidation message를 안내받음
            - invalidation message를 받으면 오래된 데이터를 제공하지 않도록 해당 키를 제거해야함
        - 예시
            - Client 1 **`>`** Server: CLIENT TRACKING ON
            - Client 1 **`>`** Server: GET foo
            - 서버는 클라이언트1이 캐시된 “foo” 키를 가지고있을 수 있다고 기억해둠
            - 클라이언트1은 로컬 메모리에 “foo”의 값을 기억하고 있을 수 있음
            - Client 2 **`>`** Server: SET foo SomeOtherValue
            - Server **`>`** Client 1: INVALIDATE "foo"
        - 단점
            - 클라이언트가 많아질수록, 키의 생명이 길수록 서버는 많은 정보를 가져야함
        - 해결방안
            - Invalidation Table
                - 엔트리의 최대 수를 가짐
                - 새로운 키가 추가되면 기존 오래된 엔트리를 제거함
                    - 이때 그 키가 수정된것처럼 행동
                - 이후 클라이언트에게 invalidation message를 전송
    - broadcasting mode
        - 서버는 주어진 클라이언트가 어떤 키로 접근했는지 기억하려하지않으므로 서버 측에서는 메모리 비용이 전혀 들지 않음
        - 클라이언트는 키 접두사를 구독하고(ex, user: ) 해당하는 키의 알림 메시지를 받음
        - 방법
            - 클라이언트는 `BCAST` 옵션을 통해 캐싱을 활성화
            - 서버는 Invalidation table대신 Prefixes Table을 사용
                - 각각의 접두사는 클라이언트의 리스트와 연관됨
            - 접두사와 매칭하는 키가 수정될때마다 그 접두사를 구독한 모든 클라이언트들은 invalidation message를 받음

## Two connections mode

- 대부분 클라이언트는 데이터 connection따로, invalidation message connection따로 구현하는걸 선호
    - 따라서 tracking을 활성화시킬때 리다이렉트시키기위해 다른 connection의 “client id”를 제공

## Opt-in caching

- 클라이언트는 키를 선택해서 캐시하길 원할 수 있음
    - 이는 새로운 오브젝트를 캐싱할때 더많은 대역폭을 필요로하지만, 서버가 기억할 데이터의 양을 줄임
- `CLIENT TRACKING on REDIRECT 1234 **OPTIN**`
- `CACHING` 커맨드로 캐시할 데이터를 지정해줄 수 있음
    - `PREFIX` 커맨드로 접두사를 지정

## The NOLOOP option

- 어떤 클라이언트는 로컬 인메모리에 쓴것도 캐시하고 싶을 수 있는데,이때 쓰고나서 바로 invalidation message를 수신하는 것은 클라이언트가 방금 캐시한 값을 강제로 제거하기 때문에 문제가됨
- `NOLOOP` 옵션은 클라이언트가 수정한 키에 대해 invalidation message를 받고싶지 않다고 서버에게 이를 말하는 역할

## Avoiding race conditions

- 커넥션이 여러개일때

```
[D] client -> server: GET foo
[I] server -> client: Invalidate foo (somebody else touched it)
[D] server -> client: "bar" (the reply of "GET foo")

(D: data connection, I: invalidation connection)
```

- GET의 요청이 클라이언트에게 늦게 닿아서, invalidation message가 먼저 받아짐
- 이 경우 그 뒤에 bar 값을 받아 stale version인 bar를 사용하는 문제 발생
- 해결책
    - invalidation message를 받고 난 후 해당하는 로컬 캐시 삭제
    - → 그 뒤에 받은 bar를 set하고자할때 이미 삭제되었기때문에 set 불가능

## ****Limiting the amount of memory used by Redis****

- BCAST를 사용하지 않을 때 Redis가 소비하는 메모리는 추적되는 키 수와 해당 키를 요청하는 클라이언트 수에 모두 비례함
- 따라서 Invalidation Table을 사용해 Redis가 기억하는 최대 키 수에 적합한 값을 구성하거나
- Redis 측에서 메모리를 전혀 소비하지 않는 BCAST 모드를 사용하는 것이 좋음