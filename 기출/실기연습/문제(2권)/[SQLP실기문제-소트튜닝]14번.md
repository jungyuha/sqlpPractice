문제 출처 : SQLP 자격검정 핵심노트2 P.25
# 문제 
아래 SQL 튜닝하시오.
필요시 인덱스 재구성안을 제시하시오.
원하는 실행계획이 나오도록 힌트를 정확히 기술하시오.

# [인덱스 구성]
상품_PK : 상품코드
계약_PK : 계약번호
계약_X1 : 지점ID

# [쿼리]

```sql
SQL > SELECT *
FROM (
	SELECT C. 계약번호, C. 상품코드, P. 상품명, P.상품구분코드, C. 계약일시, C.계약금액
	FROM 계약 C, 상품 P
	WHERE C.지점 ID = :BRCH_ID
	AND P.상품코드 = C.상품코드
	ORDER BY C.계약일시 DESC ) X
WHERE ROWNUM <= 50

Execution Plan
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT Optimizer=ALL_ROWS
|  1|  COUNT(STOPKEY)
|  2|   VIEW (Cost=5 Card=14 Bytes=1K)
|  3|    SORT (ORDER BY)
|  4|     HASH JOIN
|  5|     	TABLE ACCESS (FULL) OF '상품' (TABLE)
|  6|      	TABLE ACCESS (BY INDEX ROWID) OF '계약' (TABLE)
|  7|       	INDEX(RANGE SCAN) OF '계약_X1' (INDEX)	
------------------------------------------------------------------------------
```