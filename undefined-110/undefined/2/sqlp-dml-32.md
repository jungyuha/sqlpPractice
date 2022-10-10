# \[SQLP실기풀이-대량DML튜닝]32번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대량DML튜닝32번

### 1) 내가 생각한 튜닝 포인트🤔

1. 기존 쿼리문엔 대량 DML 이전에 PK 제약 조건을 먼저 걸었다.
   * 👉 대량 DML 처리 이후 제약 조건에 안맞는 데이터가 있는지 확인 과정을 추가한다. (즉, 따로 제약 조건을 추가할 필요는 없다.)
2. MYTAB\_TEMP테이블에 대량 INSERT를 하고 굳이 다시 대량 UPDATE를 했다.
   * 👉 대량 INSERT시 UPDATE된 정보로 넣어주어 대량 UPDATE 과정을 생략하도록 하겠다.
3. 대량 INSERT시 NOLOGGING 모드로 처리하겠다.
   * 👉 ALTER TABLE T NOLOGGING ;
   * 처리 후 , ALTER TABLE T LOGGING ;

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

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
    
	UPDATE MYTAB_TEMP SET C4 = C4 +1 WHERE C1 < TRUNC(SYSDATE);
    
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

**튜닝 후 쿼리**

```sql
SQL >
-- CTAS문에서 애초에 조건절이 적용된 데이터로 테이블을 생성한다.
CREATE TABLE MYTAB_TEMP
NOLOGGING
AS
SELECT C0 AS ID,C1,C2,C3
,CASE WHEN C1 < TRUNC(SYSDATE) THEN C4 +1 ELSE C4 END AS C4
FROM YOURTAB@RDS
WHERE C0 IS NOT NULL
AND C5 >0;

DECLARE
V_CNT NUMBER;
BEGIN
    -- 중복값 확인 (제약 조건을 두지 않았으므로)
    SELECT COUNT(*) INTO V_CNT
    FROM (
      SELECT ID
      FROM MYTAB_TEMP
      GROUP BY ID
      HAVING COUNT(*) > 1
      );
    
    IF V_CNT > 0 THEN 
    	INSERT_LOG(SYSDATE,'INSERT MYTAB_.TEMP','FAIL','중복데이터');
    ELSE
    	-- 배치 프로그램을 재실행할 경우를 대비하기 위한 DELETE (보통 0건 삭제)
		DELETE FROM MYTAB WHERE DT = TO_CHAR(SYSDATE 'YYYYMMDD');
        
        INSERT INTO MYTAB (DT,ID,C1,C2,C3,C4)
		SELECT TO_CHAR(SYSDATE,'YYYYMMDD'),A.FROM MYTAB_TEMP A;
    
        V_CNT := SQL%ROWCOUNT;
        INSERT_LOG (SYSDATE,'INSERT MYTAB_TEMP','SUCCESS',V_CNT||'ROWS');
```

> **✅ 헷갈리거나 놓쳤던 점**

* NOLOGGING 모드 설정없이 CTAS 절에서는 바로 NOLOGGING 구문을 써도 된다.
* MYTAB 테이블에는 Direct Path Insert 기능과 nologging 모드를 사용하지 않도록 한다. (온라인 트랜잭션이 발생할 수도 있으므로)
* nologging 모드 : redo 로그를 생성하지 않게 함
* nologging 모드는 Direct Path Insert 시에만 설정이 가능함 (자동 적용 아님 ! )
  * 즉 , nologging 모드는 옵션일뿐 자동으로 적용되는 건 아님

> **✅ DDL 구문**

* 제약조건 변경
  * 제약조건 삭제 + 인덱스 삭제
    * ```sql
       ALTER TABLE '테이블명' MODIFY CONSTRAINT '제약조건명' DISABLE DROP INDEX;
      ```
  * 제약조건 다시 살리기 + 인덱스에 따로 검증 필요 없음
    * ```sql
       ALTER TABLE '테이블명' MODIFY CONSTRAINT '제약조건명' ENABLE NOVALIDATE;
      ```
* 병렬 DML 모드
  * ```sql
     ALTER SESSION ENABLE PARALLEL DML; -- DML 모드 ON
      ALTER SESSION DISABLE PARALLEL DML; -- DML 모드 OFF
    ```
  *
*   NOLOGGING 모드

    * ```sql
       ALTER TABLE '테이블명' NOLOGGING; -- 로깅 해제
       ALTER TABLE '테이블명' LOGGING; -- 로깅 설정
      ```

    ```
    ```
* 인덱스 삭제
  * ```sql
     DROP INDEX [인덱스명];
    ```
*   PK인덱스 삭제

    * ```sql
      ```

    ALTER TABLE \[테이블명] DROP PRIMARY KEY CASCADE; DROP INDEX \[인덱스명];

    ```
    ```
