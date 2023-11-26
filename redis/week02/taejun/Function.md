# [2] Function

### [1] LuaScript가 있는데 왜?

- 캐싱이되야한다.
- 장애에 따라서 언제든지 Cache가 소실될 수 있다.
- Aplication이 Script에 많이 의존하게되고, 복잡해지고 무거워질 경우 문제가 발생한다.
- Script간의 참조가 불가능하다. (모든 Script는 임시적이기 떄문에)

### [2] 무엇이 다른가

- 핵심기능은 LuaScript와 동일하다.
    - Function을 구성하는 언어는 Lua
    - Blocking된다. (atomic한 실행)
    - 즉, LuaScript를 확장 한 것이다.
- 데이터 지속성 및 복제를 통한 가용성확장
    - AOF파일에 지속
- 공유가 가능
    - 각각의 Function은 하나의 라이브러리에 포함된다.
    - 라이브러리는 여러 함수를 포함하고, 불변성을 띈다.