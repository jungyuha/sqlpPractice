# \[SQLP실기풀이]6장 고급SQL튜닝 (6) 고급 SQL 활용 58번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-고급SQL활용58번

### 1) 변경한 쿼리

```sql
SQL> 
SELECT 고객번호 , MIN(B.고객명) 고객명
, LIST_AGG('(' || A.연락처구분코드 || ')'||A.연락처번호,',')
	WITHIN GROUP (ORDER BY 연락처구분코드) 연락처
FROM 고객연락처 A , 고객 B
WHERE B.고객구분코드 = 'VIP'
AND B.고객번호 = A.고객번호
GROUP BY A.고객번호
;
```

### 2) 기타

**\[쿼리 변환 중간 디버깅 산출물]**

```sql
SELECT 고객번호 , 고객명
FROM 고객
WHERE 고객구분코드 = 'VIP' ;

SELECT 연락처구분코드 , 연락처번호
FROM 고객연락처
WHERE 고객번호 =: 고객번호;

SELECT 고객번호 , B.고객명 , 연락처번호 , 연락처구분코드
, '(' || 연락처구분코드 || ')'||연락처번호 AS 연락처
FROM 고객연락처 A , 고객 B
WHERE B.고객구분코드 = 'VIP'
AND B.고객번호 = A.고객번호;
```

> **✅ 해법**
>
> * group by 처리 후 list\_agg 함수를 이용한다...근데 첨보는 함수임...시험에 안 나올거가틈....
