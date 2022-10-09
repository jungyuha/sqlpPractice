# \[SQLP실기풀이-DBCall최소화]39번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-DBCall최소화39번

### 1) 내가 생각한 튜닝 포인트🤔

1. 회원사코드가 LIKE '8%'인 회원의 계좌번호 마다 '일별예수금잔고'에 DELETE를 반복수행한다.
   * 👉 '일별예수금잔고'에 한꺼번에 액세스하고 DELETE 처리할 수 있도록 한다.

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL >
DECLARE
	V_CNT NUMBER DEFAULT 0;
	V_BASE_DT VARCHAR2(8);
BEGIN
	FOR C IN (
		SELECT B.계좌번호
        FROM 고객 A, 계좌 B
		WHERE A. 회원사코드 LIKE '8%'
		AND B.고객번호 = A. 고객번호)
	LOOP	
    	V_BASE_DT := GET_BASE_DT( C.계좌번호 );
        
        DELETE FROM 일별예수금잔고
        WHERE 기준일자 = V_BASE_DT
        AND 계좌번호 = C.계좌번호;
        
		V_CNT := V_CNT + SQL%ROWCOUNT;
        
        COMMIT;
	END LOOP;
    DBMS_OUTPUT.PUT_LINE(V_CNT {{ '건이 삭제되었습니다.');
END;
```

**튜닝 후 쿼리**

```sql
SQL >
DECLARE
	V_CNT NUMBER DEFAULT 0;
BEGIN      
        DELETE FROM 일별예수금잔고
        WHERE (계좌번호 ,기준일자) IN (
        	SELECT B.계좌번호 , GET_BASE_DT( B.계좌번호 )
        	FROM 고객 A, 계좌 B
			WHERE A. 회원사코드 LIKE '8%'
			AND B.고객번호 = A. 고객번호);
        
		V_CNT := V_CNT + SQL%ROWCOUNT;
        
        COMMIT;
        
    DBMS_OUTPUT.PUT_LINE(V_CNT {{ '건이 삭제되었습니다.');
END;
```

> **🍎 정리**

* 한번의 데이터베이스 CALL로 원하는 데이터를 모두 삭제할 수 있게 튜닝한다.
