# \[SQLP실기문제-DBCall최소화]40번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-DBCall최소화40번

### 1) 내가 생각한 튜닝 포인트🤔

1. 기존 쿼리는 수납 테이블 update,insert시 Literal Sql로 실시해 하드파싱 부하가 심하다.
   * 👉 Literal SQL 사용을 지운다.
2. 은행입금내역 조회시 '입금일시'컬럼을 변환하여 조회하여 인덱스를 제대로 못태운다.
   * 👉 '입금일시'컬럼으로 인덱스를 제대로 태울 수 있도록 조건절을 수정한다.
3. 수납 조회시 '수납일자'컬럼의 조건절이 숫자로 되어있어 인덱스를 제대로 못태운다.
   * 👉 '수납일자'컬럼의 조건절을 문자열로 바꿔 조회하도록 한다.
4. 은행입금내역에서 고객ID수는 10만건이며 입금일시를 만족하는 입금내역중 1 고객당 약 2만건의 입금내역이 있다.
   * 따라서 기존에는 수납 테이블에 약 2만건의 update가 반복 수행된다.
   * 👉 '수납' 테이블에 한꺼번에 액세스하고 UPDATE 처리할 수 있도록 DBCALL을 최소화한다.
5. 기존 쿼리는 루프 내에서 건건이 커밋한다.
   * 건건이 커밋하는 부분을 삭제하고 한번에 커밋할 수 있도록 한다.

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL >
declare
	1_수납금액 number;
begin
	for c in (select 고객ID, sum(입금액) 입금액
			  from 은행입금내역
			  where to_char(입금일시, 'yyyymmdd') = '20210329'
			  group by 고객ID)
	loop
		begin
          select 수납금액 into l_수납금액
          from 수납
          where 고객ID = c.고객ID
          and 수납일자 = 20210329;
          
          execute immediate
          'update 수납 set 수납금액 = ' || c.입금액 ||
          'where 고객ID = '|| c.고객ID ||
          'and 수납일자 = 20210329';

		  exception
            when no_data_found then
            	execute immediate
                  'insert into 수납(고객ID, 수납일자, 수납금액) values(' 
                  || c. 고객ID || ', 20210329, ' || c. 입금액 || ')';
		end;
		commit;
	end loop;
end;
```

**튜닝 후 쿼리**

```sql
SQL >
BEGIN      
	UPDATE 수납 A
    	SET A.수납금액 = ( SELECT SUM(입금액)
        				FROM 은행입금내역
                        WHERE 입금일시 >= TO_DATE('20210329','YYYYMMDD')
                        AND 입금일시 < TO_DATE('20210330','YYYYMMDD')
                        AND 고객ID = A.고객ID)
    AND A.수납일자 ='20210329'
    AND EXISTS ( SELECT 'X'
        		 FROM 은행입금내역
                 WHERE 입금일시 >= TO_DATE('20210329','YYYYMMDD')
                 AND 입금일시 < TO_DATE('20210330','YYYYMMDD')
                AND 고객ID = A.고객ID);
    
	INSERT INTO 수납 (고객ID , 수납일자 , 수냡금액) 
    SELECT 고객ID , '20210329' , SUM(입금액)
    FROM 은행입금내역 A
    WHERE 입금일시 >= TO_DATE('20210329','YYYYMMDD')
    AND 입금일시 < TO_DATE('20210330','YYYYMMDD')
    AND NOT EXISTS (
      SELECT 'X'
      FROM 수납
      WHERE 수납일자 = '20210329'
      AND 고객ID = A.고객ID
    )
    GROUP BY 고객ID;
    
    COMMIT;
END;
```

**MERGE INTO 문으로 튜닝하기**

```sql
SQL >
MERGE INTO 수납 R
USING ( SELECT 고객ID , sum(입금액) 입금액
		FROM 은행입금내역
        WHERE 입금일시 >= TO_DATE('20210329','YYYYMMDD')
    	AND 입금일시 < TO_DATE('20210330','YYYYMMDD')
        GROUP BY 고객ID) A
ON (R.고객ID = A.고객ID AND R.수납일자 ='20210329')
WHEN MATCHED THEN UPDATE
	SET R.수납금액 = A.입금액
WHEN NOT MATCHED THEN
 	INSERT (고객 ID , 수납일자 , 수납금액) VALUES ( A.고객ID , '20210329',A.입금액);
    
COMMIT;
```

> **🍎 정리**

* '수납' 테이블에 한꺼번에 액세스하고 UPDATE 처리할 수 있도록 DBCALL을 최소화한다.
* 건건이 커밋을 한 번의 커밋으로 바꾼다.
