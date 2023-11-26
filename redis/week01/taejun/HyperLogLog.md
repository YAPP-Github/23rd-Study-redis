# [8] HyperLogLog

- Redis에만 있는 개념이 아니고, 다른 곳에서도 사용한다. (ksql, druid, presto …)
- hash함수를 사용한다.
- 매우 큰 데이터 Set의 고유성 (카디널리티)을 추정하는 확률적 데이터구조
    - 12KB를 사용 0.81%의 표준오류를 제공한다.
    - 최대 2^64개의 요소에 대한 카디널리티를 제공할 수 있다.
    - cardinality를 N이라 할 때, 각 레지스터 크기는 log2log2N + O(1)이다.
        - 당연하게도 메모리를 많이 사용할수록 실제에 근사한 값을 얻을 수 있다.
- UseCase
    - 빅데이터 애플리케이션
    - 실시간 분석
    - 로그 처리
    - 웹페이지 고유 방문자 수
- **HyperLogLog**는 2007년 Philippe Flajolet 발표한 알고리즘입니다. HyperLogLog라는 이름은 이전 알고리즘인 Loglog Counting을 발전시켰기 때문에 Hyper를 붙친것입니다.
- 살바토르(Salvatore)는 Philippe Flajolet를 기리기 위해 HyperLogLog 관련 명령에 PF 접두어를 사용했습니다.
- HyperLogLog 데이터는 별도 data structure를 사용하지 않고 String을 사용합니다. String 내부는 HyperLogLog data structure입니다.
- Bit패턴 분석을 사용하고, 어느정도의 오차를 허용한다.
