문제 출처 : SQLP 자격검정 핵심노트2 P.23
# 문제 
아래 SQL 튜닝하시오.
필요시 인덱스 재구성안을 제시하시오.
원하는 실행계획이 나오도록 힌트를 정확히 기술하시오.

# [데이터]
상품 : 1,000건
계약 : 5,000만 건
1년 간 계약건수는 500만 건
▶ 상품유형코드를 '=' 조건으로 검색할 때의 평균 카디널리티는 100

# [인덱스 구성]
상품_PK : 상품번호
상품_X1 : 상품유형코드
계약_PK : 계약번호
계약_X1 : 계약일자
계약_X2 : 상품번호

# [쿼리]

```sql
SQL > SELECT DISTINCT P. 상품번호, P. 상품명, P.상품가격, P.상품분류코드
FROM 상품 P, 계약 C
WHERE
P.상품유형코드 = :PCLSCD
AND C.상품번호 = P.상품번호
AND C.계약일자 >= TRUNC(ADD_MONTHS(SYSDATE, -12))

Execution Plan
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT Optimizer=ALL_ROWS (Cost=3 Card=1 Bytes=80)
|  1|  HASH (UNIQUE) (Cost=3 Card=1 Bytes=80)
|  2|   FILTER
|  3|    NESTED LOOPS
|  4|     NESTED LOOPS (Cost=2 Card=1 Bytes=80)
|  5|     	TABLE ACCESS (BY INDEX ROWID) OF '상품' (TABLE)
|  6|      		INDEX(RANGE SCAN) OF '상품_X1' (INDEX) (Cost=1 Card=1)
|  7|       INDEX (RANGE SCAN) OF '계약_X2' (INDEX) (Cost=1 Card=1)
|  8|     TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE) 
------------------------------------------------------------------------------
```