# \[SQLP실기풀이]6장 고급SQL튜닝(5) 대용량 배치 프로그램 튜닝 52번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대용량배치프로그램튜닝52번

### 1) 내가 생각한 튜닝 포인트🤔

1. 기존 쿼리는 PARALLEL 모드로 데이터를 UPDATE하는데 ROWNUM을 사용해 병목현상이 일어난다.
   * 병렬 조회 쿼리에 ROWNUM을 사용하면 중복 값이 생긴다.따라서 중복 없이 처리하기 위해 QC가 Unique 처리를 하는데 이 과정에서 병목현상이 생긴다.
   * 👉 ROWNUM이 아닌 ROW\_NUMBER()을 사용해 PK컬럼으로 정렬해 일련번호를 부여하면 QC가 Unique 처리를 하는 과정을 생략할 수 있다.(다만 데이터 재분배는 필요하다.)
   * 이 때 데이터 재분배 과정을 모두 마쳐 ROW\_NUMBER()값이 모두 부여되어 완성된 뷰를 가지고 UPDATE 하기 때문에 MERGE INTO 구문을 써야한다.

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL > ALTER SESSION ENABLE PARALLEL DML;
UPDATE /*+ PARALLEL (주문 4) */ 주문
SET 주문일련번호
주문일련번호 = ROWNUM
WHERE 주문일자 = TO_CHAR(SYSDATE, 'YYYYMMDD');
```

**튜닝 후 쿼리**

```sql
SQL > MERGE INTO 주문 T1
USING ( SELECT 고객번호 , 주문순번
		,ROW_NUMBER() OVER ( ORDER BY 고객번호 , 주문순번 ) AS 주문일련번호
		FROM 주문
		WHERE 주문일자 = TO_CHAR(SYSDATE, 'YYYYMMDD') ) T2
ON ( T1.주문일자 = TO_CHAR(SYSDATE, 'YYYYMMDD')
	AND T1.고객번호 = T2.고객번호
    AND T1.주문순번 = T2.주문순번 )
WHEN MATCHED THEN UPDATE
	SET T1.주문일련번호 = T2.주문일련번호;
```

> **🍎 정리**

* ROWNUM이 아닌 ROW\_NUMBER()을 사용해 일련번호를 부여하여 QC가 Unique 처리를 하는 과정을 생략한다.
