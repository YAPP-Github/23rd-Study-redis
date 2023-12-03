### HyperLogLog

- **확률적 자료구조를 이용한 추정**
- 확률적으로 집합의 크기를 추정
- **정확하지 않지만 최대한 정확한 값(근사값)을 상대적으로 적은 메모리만 사용해 얻고 싶을 때 사용할 수 있는 방법**
- 오차 : 0.81%
- 일정 메모리양만을 사용하여 계산 가능 (최악의 경우 12k bytes)
- redis string으로 인코딩됨
    - `GET`, `SET`
- 개념적으로 set 자료구조와 유사
- 명령어
    - `PFADD` : 새로운 element일때 추가
    - `PFCOUNT` : 확률적으로 집합의 크기 추정하여 반환
    - `PFMERGE` : 두개의 다른 HLL을 합치고자할때 사용

### 확률적 자료구조

<img width="638" alt="image" src="https://github.com/Team-BuddyCon/BACKEND_V2/assets/69676101/d029a341-da28-44ab-9733-eaa9b7c47931">


https://d2.naver.com/helloworld/711301

- 위 자료구조들은 랜덤함수가 아닌 해시함수를 사용
    - 해시함수 → 해시 충돌에 대한 보안책을 둘 수 있음