# \[SQLP실기풀이-대량DML튜닝]34번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대량DML튜닝34번

### 1) 내가 생각한 튜닝 포인트🤔

1. 기존 쿼리문에 ROW Lock 경합이 자주 발생하는 이유
   * 매 업데이트시 전체 데이터의 80%가 업데이트된다.
2. 기존 쿼리에서는 상품재고 테이블의 품질유지일을 수정하기 위해 상품재고 테이블을 같은 조건으로 한번 더 액세스한다. 이 과정은 불필요한 액세스이므로 다른 방법으로 대체한다.
   * 👉 다른 방법 : UPDATE시 자체 조건절에 수정이 필요없는 ROW는 제외해서 조회한다.
3. 업데이트를 하지 않아도 될 부분도 같이 업데이트 되므로 이 부분도 업데이트가 필요한 부분만 처리가 될 수 있도록 하겠다.
   * 👉 EXISTS 구문을 이용한다.
   * ⭐️ 존재 여부만을 판단하므로 nl\_sj (nl세미조인)과 unnest 힌트를 넣어준다.

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL >
UPDATE 상품재고 T
	SET T.품질유지일 = 
    NVL((SELECT TRUNC(SYSDATE) - TO_DATE(MAX(A. 변경일자), 'YYYYMMDD')
		FROM 상품재고이력 A, 상품재고 B
		WHERE A. 상품번호 = B. 상품번호
		AND B. 업체코드 = 'Z'
		AND B. 가용재고량 = 0
		AND NVL(B.가상재고수량, 0) 〈= 0
		AND A.상품번호 = T.상품번호
		GROUP BY A.상품번호 )
        , T.품질유지일)
WHERE T.업체코드 = 'z'
AND T.가용재고량 = 0
AND NVL(T. 가상재고수량, 0) <= 0;
```

**튜닝 후 쿼리**

```sql
SQL >
UPDATE 상품재고 T
	SET T.품절유지일 =
    	(SELECT TRUNC(SYSDATE) - TO_DATE(MAX(변경일자),'YYYYMMDD') 품질유지일
        	FROM 상품재고이력 A
  			WHERE A.상품번호 = T.상품번호)
WHERE T.업체코드 = 'z'
AND T.가용재고량 = 0
AND NVL(T. 가상재고수량, 0) <= 0
AND EXISTS (
	SELECT /*+nl_sj unnest*/ 'X'
	FROM 상품재고이력
    WHERE 상품번호 = T.상품번호
);
```

> **🍎 정리**

* EXISTS문으로 NL\_SJ 조인을 이용해 2번 액세스하는 걸 1번 액세스로 줄여 update하였다.
