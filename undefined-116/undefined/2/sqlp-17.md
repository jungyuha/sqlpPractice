문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-소트튜닝17번


### ⭐️ 쿼리 작성 포인트 
1. 인덱스 정렬을 사용해 TOP 1개만 추출하도록 한다.

##  튜닝한 SQL문
   
```sql
SQL > SELECT 변경일시 AS 최종변경일시 
FROM (
  SELECT 변경일시
  FROM 상품변경이력
  WHERE 상품번호 = 'ZE367'
      AND 변경구분코드 ='C2'
  ORDER BY 상품번호 , 변경일시 DESC
)
WHERE ROWNUM =1 ;
```

> #### 🍎 정리
- 인덱스가 상품번호+변경일시 구성이므로 상품번호+변경일시 정렬순으로 가장 최근 변경일시 1건을 추출한다.

   

   