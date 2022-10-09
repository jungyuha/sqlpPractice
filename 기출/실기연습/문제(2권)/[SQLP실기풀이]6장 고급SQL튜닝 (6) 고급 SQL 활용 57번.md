문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-고급SQL활용57번

## 1) 변경한 쿼리
```sql
SQL> 
  SELECT 고객번호 , MIN(A.고객명) 고객명
  , MIN(DECODE(B.연락처구분코드 , 'HOM', B.연락처번호 )) AS 집 
  , MIN(DECODE(B.연락처구분코드 , 'OFC', B.연락처번호 )) AS 사무실
  , MIN(DECODE(B.연락처구분코드 , 'MBL', B.연락처번호 )) AS 휴대폰 
  FROM 고객 A , 고객연락처 B
  WHERE A.고객구분코드 = 'VIP'
  AND A.고객번호 = B.고객번호(+)
  GROUP BY 고객번호;
```

## 2) 기타 
#### [쿼리 변환 중간 디버깅 산출물]
```sql
SELECT 고객번호 , 고객명
FROM 고객
WHERE 고객구분코드 = 'VIP' ;

SELECT 연락처구분코드 , 연락처번호
FROM 고객연락처
WHERE 고객번호 =: 고객번호;

SELECT 고객번호 , 고객명 , 연락처번호 AS 집 ,NULL AS 사무실 , NULL AS 휴대폰 
FROM 고객 A , 고객연락처 B
WHERE A.고객구분코드 = 'VIP'
AND A.고객번호 = B.고객번호(+)
AND B.연락처구분코드 ='HOM';

SELECT 고객번호 , 고객명 , NULL AS 집 , 연락처번호 AS 사무실 , NULL AS 휴대폰 
FROM 고객 A , 고객연락처 B
WHERE A.고객구분코드 = 'VIP'
AND A.고객번호 = B.고객번호(+)
AND B.연락처구분코드 ='OFC';

SELECT 고객번호 , 고객명 , NULL AS 집 ,NULL AS 사무실 , 연락처번호 AS 휴대폰 
FROM 고객 A , 고객연락처 B
WHERE A.고객구분코드 = 'VIP'
AND A.고객번호 = B.고객번호(+)
AND B.연락처구분코드 ='MBL';

SELECT 고객번호 , MIN(고객명) 고객명 , MIN(집) 집 , MIN(사무실) 사무실 , MIN(휴대폰) 휴대폰 
FROM (
  SELECT 고객번호 , 고객명 
  , DECODE(B.연락처구분코드 , 'HOM', B.연락처번호 ) AS 집 
  , DECODE(B.연락처구분코드 , 'OFC', B.연락처번호 ) AS 사무실
  , DECODE(B.연락처구분코드 , 'MBL', B.연락처번호 ) AS 휴대폰 
  FROM 고객 A , 고객연락처 B
  WHERE A.고객구분코드 = 'VIP'
  AND A.고객번호 = B.고객번호(+)
)
GROUP BY 고객번호;

SELECT 고객번호 , MIN(A.고객명) 고객명
, MIN(DECODE(B.연락처구분코드 , 'HOM', B.연락처번호 )) AS 집 
, MIN(DECODE(B.연락처구분코드 , 'OFC', B.연락처번호 )) AS 사무실
, MIN(DECODE(B.연락처구분코드 , 'MBL', B.연락처번호 )) AS 휴대폰 
FROM 고객 A , 고객연락처 B
WHERE A.고객구분코드 = 'VIP'
AND A.고객번호 = B.고객번호(+)
GROUP BY 고객번호;
```

> #### ✅ 헷갈렸던 점
> - group by를 하지 않으면 고객연락처에 해당되는 로우수만큼 추출되므로 group by 고객번호는 필수다.
> - group by 절 이외의 컬럼을 추출하려는 경우 MIN(컬럼)을 이용한다.
(단 집계함수를 써도 영향받지 않는 데이터여야한다.)
