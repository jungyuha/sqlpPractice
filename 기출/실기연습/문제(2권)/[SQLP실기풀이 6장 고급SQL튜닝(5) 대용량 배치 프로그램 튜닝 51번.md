문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대용량배치프로그램튜닝51번

## 1) 내가 생각한 튜닝 포인트🤔
1. 기존 쿼리는 PARALLEL 모드로 가져온 데이터를 ROWNUM을 사용해 한번에 합치려한다.
   - 병렬 조회 쿼리에 ROWNUM을 사용하면 중복 값이 생긴다.따라서 중복 없이 처리하기 위해 QC가 Unique 처리를 하는데 이 과정에서 병목현상이 생긴다.
   - 👉 ROWNUM이 아닌 ROW_NUMBER()을 사용해 데이터 크기로 정렬해 일련번호를 부여하면 QC가 Unique 처리를 하는 과정을 생략할 수 있다.(다만 데이터 재분배는 필요하다.)
       
## 2) 튜닝한 쿼리
#### 튜닝 전 쿼리
```sql
SQL > CREATE TABLE T
PARALLEL 4
AS
SELECT ROWNUM AS 주문일련번호, 주문일자, 주문순번
FROM (
  SELECT /*+ PARALLEL(주문 4) */ 고객번호, 주문일자, 상품번호, 주문량, 주문금액
  FROM 주문
  ORDER BY 고객번호, 주문일자, 주문순번
);
```
#### 튜닝 후 쿼리
```sql
SQL > CREATE TABLE T
PARALLEL 4
AS
  SELECT /*+ PARALLEL(주문 4) */ 고객번호, 주문일자, 상품번호, 주문량, 주문금액
  , ROW_NUMBER() OVER ( ORDER BY 고객번호, 주문일자, 주문순번) AS 주문일련번호
  FROM 주문;
```
> #### 🍎 정리
- ROWNUM이 아닌 ROW_NUMBER()을 사용해 데이터 크기로 정렬해 일련번호를 부여하여 QC가 Unique 처리를 하는 과정을 생략한다.
