문제 출처 : SQLP 자격검정 핵심노트2 P.33
# 문제
새벽 시간에는 MYTAB 테이블에 온라인 트랜잭션이 거의 발생하지 않는다.
새벽에 원격 RDS시스템으로부터 1,000만 건 정도의 데이터를 읽어서 INSERT 하는 아래 배치(Batch)프로그램 성능을 개선하세요.
- UPDATE문에서 C1 조건절을 만족하는 데이터는 90% 이상이다.
- MYTAB 테이블의 PK는 (DT+ID)이다.
- 병렬처리는 활용할 수 없다.

## [1] SQL
```sql
SQL >
CREATE TABLE MYTAB_TEMP
AS
SELECT C0 AS ID,C1,C2,C3,C4
FROM
YOURTAB@RDS
WHERE 1=2;

ALTER TABLE MYTAB_TEMP ADD CONSTRAINT MYTAB_TEMP_PK PRIMARY KEY(ID);

DECLARE
V_CNT NUMBER;
BEGIN
	INSERT INTO MYTAB_TEMP
	SELECT C0,C1,C2,C3,C4
	FROM
	YOURTAB@RDS
	WHERE C0 IS NOT NULL
	AND C5 >0;
    
	UPDATE MYTAB_TEMP SET C4 = C4 +1 WHERE C1 TRUNC(SYSDATE);
    
	-- 배치 프로그램을 재실행할 경우를 대비하기 위한 DELETE (보통 0건 삭제)
	DELETE FROM MYTAB WHERE DT = TO_CHAR(SYSDATE 'YYYYMMDD');
    
	INSERT INTO MYTAB (DT,ID,C1,C2,C3,C4)
	SELECT TO_CHAR(SYSDATE,'YYYYMMDD'),A.FROM MYTAB_TEMP A;
    
	V_CNT := SQL%ROWCOUNT;
	INSERT_LOG (SYSDATE,'INSERT MYTAB_TEMP','SUCCESS',V_CNT||'ROWS');
    
	COMMIT;
    
    EXCEPTION
    WHEN dup_val_on_index THEN
    INSERT_LOG(SYSDATE,'INSERT MYTAB_.TEMP','FAIL','중복데이터')
END;

DROP TABLE MYTAB_TEMP;
```