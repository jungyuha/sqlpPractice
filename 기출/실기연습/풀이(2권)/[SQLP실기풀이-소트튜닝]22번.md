문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-소트튜닝22번

# 문제

장비 구분 코드가 'A001' 인 장비의 최종 상태코드,변경일자,변경순번을 출력하는 최적 SQL을 작성하시오.(단 , DBMS는 오라클이며 , 12c 이후 버전을 사용하고 있다.)

### ⭐️ 쿼리 작성 포인트 
1. 인덱스 정렬을 사용해 TOP 1개만 추출하는 부분범위 처리를 활용한다.

##  튜닝한 SQL문
   
```sql
SELECT /*+ LEADING(P) USE_NL(A) */ 장비번호 
  FROM 장비 P , 상태변경이력 A
  WHERE A.장비번호 = P.장비번호
  AND (P.변경일자 , P.변경순번) = (
  	SELECT 변경일자 ,변경순번
    FROM ( SELECT 변경일자 , 변경순번
        FROM 상태변경이력
        WHERE 장비번호 = P.장비번호
        ORDER BY 변경일자 DESC ,변경순번 DESC)
    WHERE ROWNUM <= 1)
  AND P.장비구분코드 = 'A001';
```
> #### 🍎 정리
- 장비구분코드 = 'A001'을 만족하는 **_상품'들'_**의 각각 최종 변경 정보를 가지고 와야 했다.
- 상태변경이력의 키를 이용하여 (장비번호 + 변경일자 + 변경순번) 인라인뷰 안에서 부분범위처리를 해 각 상품별의 최종 변경 정보를 1개씩 가져왔다.
- 근데 나는 /*+ LEADING(P) USE_NL(A) */ 힌트를 썼는데 안 써도 되는걸까?

##  비슷한 해법을 가진 문제

https://velog.io/@yooha9621/SQLP실기풀이-날짜57번
https://velog.io/@yooha9621/SQLP실기풀이-날짜58번


   

   