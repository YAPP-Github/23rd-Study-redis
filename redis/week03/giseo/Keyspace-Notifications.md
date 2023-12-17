# Redis Keyspace Notification
- Redis key/value의 변경 사항을 실시간으로 수신할 수 있는 Pub/Sub 기능
- 새로운 키 입력/변경 등의 이벤트가 발생할 때 알려주는 기능을 제공하며, redis.conf 에서 이를 관리할 수 있다.

## 이벤트 종류
- 키(데이터) 입력/변경 : set 같은 명령으로 키/값이 새로 입력되거나 수정될 때
- 키 삭제 : del 같은 명령으로 키가 삭제될 때
- 키 만료 시간 설정으로 키가 삭제될 때
- Maxmemory 정책으로 키가 퇴출(eviction)될 때


## Pub/Sub Channel
- 이벤트가 발생했을 때 메시지를 주고받는 channel 유형에는 키 중심과 명령 중심이 있다.
  - 키 중심 : __keyspace@<dbid>__:key command
  - 명령 중심 : __keyevent@<dbid>__:command key

- 키 중심의 채널은 이벤트가 발생하면 **이벤트 이름**을 주고 받고, 명령 중심의 채널은 이벤트가 발생하면 **키 이름**을 메세지로 주고 받습니다.

``` bash
#키 중심 (mykey가 삭제 되었을 때 이벤트 이름을 메세지로 보냄)
publish __keyspace@0__:mykey del

#명령중심 (mykey가 삭제 되었을 때 키 이름을 메세지로 보냄)
publish __keyevent@0__:del mykey
```

- 메세지를 받는 subcribe는 다음과 같이 설정할 수 있다.

```
#모든 key에 대해서 이벤트가 발생했을 때 이벤트 이름을 받음
psubscribe __keyspace@0__:*

#tag로 시작하는 키에 대해서 이벤트가 발생했을 때 이벤트 이름을 받음
psubscribe __keyspace@0__:tag*

#특정 키 tag0001를 지정해서 이벤트가 발생했을 때 이벤트 이름을 받음
subscribe __keyspace@0__:tag0001

#tag0001 키가 만료되는 이벤트가 발생했을 때 이벤트 이름을 받음
subscribe __keyspace@0__:tag0001 expired

#모든 쓰기 명령이 발생했을 때 키 이름을 받음
psubscribe __keyevent@0__:*

#set 명령이 발생했을 때 키 이름을 받음
subscribe __keyevent@0__:set
```


## notify-keyspace-events
- 이벤트를 활성화하는 방법은 notify-keyspace-events 으로 하며, 이벤트 종류는 다음과 같다.
  - K : Keyspace events, publish prefix "__keyspace@<dbid>__:key command"
  - E : Keyevent events, publish prefix "__keyevent@<dbid>__:command key"
  - g : 공통 명령: del, expire, rename ...
  - $ : 스트링(String) 명령
  - l : 리스트(List) 명령
  - s : 셋(Set) 명령
  - h: 해시(Hash) 명령
  - z: 소트 셋(Sorted set) 명령
  - x: 만료(Expired) 이벤트 (키가 만료될 때마다 생성되는 이벤트)
  - e: 퇴출(Evicted) 이벤트(Maxmemory 정책으로 키가 삭제될 때 생성되는 이벤트)

```
#모든 이벤트를 발생
notify-keyspace-events "KEA"

#이벤트 제거
notify-keyspace-events ""
```

## Timing of expired events
- 기본적으로 expired 이벤트는 레디스 서버가 키를 삭제할 때 발생되는 이벤트다.
- 따라서 TTL이 0으로 도달했을 때 즉각적으로 발생하지는 않는다.

- TTL이 설정된 키들은 레디스에 의해 다음의 2가지 방법으로 만료된다. 
  - 특정 키에 대해 커맨드에 의한 접근이 발생했고, 만료된 사실이 발견되었을 때
  - 백그라운드 시스템에 의해 만료된 사실이 발견되었을 때

- expired 이벤트는 위의 방법을 통해 key가 만료된 사실이 발견되었을 때 발생한다.
