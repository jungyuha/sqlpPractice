# 문제
아래 SQL을 튜닝하세요.
-결과 집합을 모두 출력하는 애플리케이션 환경을 기준으로 튜닝하세요.
-필요하다면, 인덱스 재구성 안을 제시하세요.
-튜닝 의도대로 정확히 실행되도록 옵티마이저 힌트를 기술하세요.
## [1] SQL
```sql
SQL > SELECT P.상품코드 , MIN(P.상품명) 상품명 , MIN(P.상품가격) 상품가격
, SUM(O.주문수량) 총주문수량 , SUM(O.주문금액) 총주문금액
FROM 상품 P , 주문상품 O
WHERE O.주문일시 >= ADD_MONTHS(SYSDATE,-1)
AND O.할인유형코드 = 'K890'
AND P.상품코드 = O.상품코드
GROUP BY P.상품코드
ORDER BY 총 주문금액 DESC , 상품코드

Execution Plan
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT Optimizer=ALL_ROWS (cost = 1M card=20k Bytes=645k)
|  1|  SORT (ORDER BY) (cost = 1M card=20k Bytes=645k)
|  2|   HASH (GROUP BY) (cost = 1M card=20k Bytes=645k)
|  3|    NESTED LOOPS (cost = 1M card=497k Bytes=16M) 
|  4|     NESTED LOOPS (cost = 1M card=497k Bytes=16M) 
|  5|      TABLE ACCESS (BY INDEX ROWID) OF '주문상품' (TABLE) (cost = 505K...)
|  6|       INDEX (RANGE SCAN) OF '주문상품_X1' (INDEX) (cost = 3K card=500k)
|  7|      INDEX (UNIQUE SCAN) OF '상품_PK' (INDEX (UNIQUE)) (cost = 0 card=1)
|  8|     TABLE ACCESS (BY INDEX ROWID) OF '상품' (TABLE) (cost = 1 card=1...)
------------------------------------------------------------------------------
```
## [2] 테이블 구성 및 데이터
- 주문 상품은 비파티션 테이블
- 한 달 주문상품 = 100만건
- 주문상품의 보관 기관 = 10년
- 주문상품 총 건수 = 총 1억 2천만 건 (=100만 X 120개월)
- 할인유형코드 조건을 만족하는 데이터 비중 = 20%
- 등록된 상품 = 2만개
- 2만 개 상품을 고르게 주문

## [3] 인덱스 구성
- 상품_PK : 상품코드
- 주문상품_PK : 고객번호 + 상품코드 + 주문일시
- 주문상품_X1 : 주문일시 + 할인유형코드