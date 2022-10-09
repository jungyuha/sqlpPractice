# \[SQLP실기문제-대량DML튜닝]34번

문제 출처 : SQLP 자격검정 핵심노트2 P.36

## 문제

1시간 주기 Batch 프로그램에서 수행하는 아래 SQL에 의해 Row Lock 경합이 자주 발생하여 UPDATE 건수는 8만여 건이다. SQL을 튜닝하고, 원하는 실행계획이 나오도록 힌트를 기술하세요.

### \[데이터]

* 상품재고 : 101,382 rows
* 상품재고이력 : 88,370,445 rows

### \[인덱스 구성]

* 상품재고\_PK : 상품번호
* 상품재고이력 PK : 상품번호 + 변경일자 + 변경순번

### SQL

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
		AND A. 상품번호 = T. 상품번호
		GROUP BY A.상품번호 )
        , T.품질유지일)
WHERE T.업체코드 = 'z'
AND T.가용재고량 = 0
AND NVL(T. 가상재고수량, 0) <= 0;
```
