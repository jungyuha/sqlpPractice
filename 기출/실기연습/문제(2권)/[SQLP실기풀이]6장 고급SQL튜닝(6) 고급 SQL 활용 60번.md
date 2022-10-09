# \[SQLP실기풀이]6장 고급SQL튜닝(6) 고급 SQL 활용 60번

문제 출처 : SQLP 자격검정 핵심노트2 P.119

## 문제

아래 SQL을 Union All을 활용한 Full Outer Join으로 재작성하시오.

### \[SQL]

**변환 전 쿼리**

```sql
SELECT NVL(A.고객ID, B.고객ID) 고객ID, A.입금액, B.출금액
FROM (
	SELECT 고객ID, SUM(입금액) 입금액
    FROM 입금
    GROUP BY 고객ID
    ) A
FULL OUTER JOIN
  (SELECT 고객ID, SUM(출금액) 출금액
  FROM 출금
  GROUP BY 고객ID) B
ON A. 고객ID = B. 고객ID
```

### \[풀이]

**변환 후 쿼리**

```sql
SELECT 고객ID , SUM(입금액) 입금액 , SUM(출금액) 출금액 
FROM(
  SELECT 고객ID, 입금액 , NULL 출금액
  FROM 입금
  UNION ALL
  SELECT 고객ID, NULL 입금액 , 출금액
  FROM 출금
  )
GROUP BY 고객ID ;
```
