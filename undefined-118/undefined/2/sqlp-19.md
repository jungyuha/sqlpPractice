문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-소트튜닝19번

# 문제

아래 상품할인율 테이블에서 상품번호 = 'R0014' 인 상품의 2021년 3월 한 달간 일별 최종 할인율 (= 기준일자별 마지막 변경순번의 할인율)을 찾는 SQL을 작성하세요.

### ⭐️ 쿼리 작성 포인트 
1. 인덱스 정렬을 사용해 TOP 1개만 추출하는 부분범위 처리를 활용한다.

##  튜닝한 SQL문
   
```sql
SQL > SELECT 기준일자 , 할인율 AS 최종할인율
FROM (
  SELECT 할인율 , 기준일자
  ,ROW_NUMBER() OVER(PARTITION BY 기준일자 ORDER BY 변경순번 DESC) RNUM
  FROM 상품할인율
  WHERE 기준일자 BETWEEN '20220301' AND '20220331'
  AND 상품번호 = 'R0014' 
)
WHERE RNUM =1
ORDER BY 기준일자;
```

> #### 🍎 정리
- 기준일자별 변경 순번을 내림차순으로 정렬한 뒤 가장 최근 1건을 추출한다.

   

   