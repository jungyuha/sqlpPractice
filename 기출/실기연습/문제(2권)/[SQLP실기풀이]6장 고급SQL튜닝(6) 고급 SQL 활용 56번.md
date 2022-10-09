# \[SQLP실기풀이]6장 고급SQL튜닝(6) 고급 SQL 활용 56번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-고급SQL활용55번

### 1) 변경한 쿼리

```sql
SQL> SELECT 고객번호 , 고객명
, DECODE ( LVL, 1, 집전화번호, 2, 사무실전화번호, 3, 휴대폰번호 ) AS 연락처번호
, DECODE ( LVL, 1, '집전화번호', 2, '사무실전화번호', 3, '휴대폰번호' ) AS 연락처구분
FROM 고객 , (
    SELECT LEVEL AS LVL
    FROM DUAL
    WHERE CONNECT BY LEVEL <=3
	) T1
WHERE 고객구분코드 = 'VIP' 
AND LVL IN ((CASE WHEN LVL = 1 AND 집전화번호 IS NOT NULL THEN 1 END)
	,(CASE WHEN LVL = 2 AND 사무실전화번호 IS NOT NULL THEN 2 END)
    ,(CASE WHEN LVL =3 AND 휴대폰번호 IS NOT NULL THEN 3 END))
ORDER BY 고객번호 , LVL
;
```

### 2) 기타

**\[쿼리 변환 중간 디버깅 산출물]**

```sql
SELECT 고객번호 , 고객명
, 집전화번호 AS '연락처번호' ,'집전화번호' AS 연락처구분 
FROM 고객
WHERE 고객구분코드 = 'VIP' 
AND 집전화번호 IS NOT NULL;

SELECT 고객번호 , 고객명
, 사무실전화번호 AS '연락처번호' ,'사무실전화번호' AS 연락처구분
FROM 고객
WHERE 고객구분코드 = 'VIP'
AND 사무실전화번호 IS NOT NULL;

SELECT 고객번호 , 고객명 , 휴대폰번호 AS '연락처번호' ,'휴대폰번호' AS 연락처구분 
FROM 고객
WHERE 고객구분코드 = 'VIP'
AND 휴대폰번호 IS NOT NULL;

SELECT LEVEL AS LVL
FROM DUAL
WHERE CONNECT BY LEVEL <=3;
```

> **✅ IS NOT NULL 대상 컬럼이 각각 다른 경우**
>
> ```sql
> AND LVL IN ((CASE WHEN LVL = 1 AND 집전화번호 IS NOT NULL THEN 1 END)
> ```

```
,(CASE WHEN LVL = 2 AND 사무실전화번호 IS NOT NULL THEN 2 END)
```

```
,(CASE WHEN LVL =3 AND 휴대폰번호 IS NOT NULL THEN 3 END))
```

> ```
> ```

위 조건절과 같이 IN절로 포함 여부를 판단하는 방식으로 구현하도록 한다.
