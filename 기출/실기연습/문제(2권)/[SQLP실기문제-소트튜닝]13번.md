문제 출처 : SQLP 자격검정 핵심노트1 P.24
# 문제 
아래 SQL 튜닝하시오.
필요시 인덱스 재구성안을 제시하시오.
원하는 실행계획이 나오도록 힌트를 정확히 기술하시오.

# [데이터]
상품 : 100개
주문 : 1억 건
주문상품 : 2억 건
▶ 한달 주문건수 = 100만 건
▶ 거의 모든 상품에 골고루 주문 발생

# [인덱스 구성]
주문_PK : 주문번호
주문_X1 : 주문일자
주문상품_PK : 주문번호 + 상품코드

# [쿼리]

```sql
SQL > SELECT DISTINCT 0. 주문번호, 0.주문금액, 0.결제구분코드, 0.주문매체코드
FROM 주문 0, 주문상품 P
WHERE 0.주문일자 >= TRUNC(ADD_MONTHS(SYSDATE, -12))
AND P.주문번호 = 0.주문번호
AND P.상품코드 = :PRD_CD

Execution Plan
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=1 Bytes=80)
|  1|  HASH(UNIQUE) (Cost=3 Card=1 Bytes=80)
|  2|   FILTER
|  3|    NESTED LOOPS
|  4|     NESTED LOOPS (Cost=2 Card=1 Bytes=80)
|  5|     	TABLE ACCESS (BY INDEX ROWID) OF '주문' (TABLE) (Cost=1 … )
|  6|      		INDEX (RANGE SCAN) OF '주문_X1' (INDEX) (Cost=1 Card=1)
|  7|       INDEX (RANGE SCAN) OF '주문상품_X2' (INDEX) (Cost=1 Card=1)
------------------------------------------------------------------------------
```