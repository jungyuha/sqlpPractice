# \[SQLP실기풀이-대량DML튜닝]31번

문제 링크 : https://velog.io/@yooha9621/SQLP실기풀이-대량DML튜닝31번

```sql
DELETE FROM TARGET_T;
COMMIT;
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND */ INTO TARGET_T T1
SELECT /*+ FULL(T2) PARALLEL(T2 4) */ *
FROM SOURCE_T T2;

COMMIT;
ALTER SESSION DISABLE PARALLEL DML;
```

### 1) 해당 문제 튜닝 순서

1. DELETE => TRUNCATE 로 처리
2. 인덱스 제약조건 적용 안되게 설정
   * ALTER TABLE TARGET\_T MODIFY CONSTRAINT TARGET\_T\_PK DISABLE DROP INDEX;
3. 병렬 DML 모드 켬 (세션 범위)
   * ALTER SESSION ENABLE PARALLEL DML;
   * 이 때 **append** 힌트는 없어도 된다.
4. Nologging 모드 켬 (테이블 범위)
   * ALTER TABLE TARTGET\_T NOLOGGING;
5. 제대로 된 병렬 처리를 한다.(짝 맞추기) - INSERT에도 PARALLEL(T2 4) 추가

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL >
DELETE FROM TARGET_T;

COMMIT;

ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND */ INTO TARGET_T T1
SELECT /*+ FULL(T2) PARALLEL(T2 4) */*
FROM SOURCE_T T2;

COMMIT;

ALTER SESSION DISABLE PARALLEL DML;
```

**튜닝 후 쿼리**

```sql
SQL >
TRUNCATE TABLE TARGET_T;

ALTER TABLE TARGET_T MODIFY CONSTRAINT TARGET_T_PK DISABLE DROP INDEX;

ALTER SESSION ENABLE PARALLEL DML;

ALTER TABLE TARGET_T NOLOGGING;

INSERT /*+ PARALLEL(T1 4) */ INTO TARGET_T T1
SELECT /*+ FULL(T2) PARALLEL(T2 4) */ *
FROM SOURCE_T T2;

COMMIT;

ALTER TABLE TARGET_T MODIFY CONSTRAINT TARGET_T_PK ENABLE NOVALIDATE;
ALTER TABLE TARGET_T LOGGING;

ALTER SESSION DISABLE PARALLEL DML;
```

> **✅ 헷갈리는 점**

* nologging 모드 : redo 로그를 생성하지 않게 함
* nologging 모드는 Direct Path Insert 시에만 설정이 가능함 (자동 적용 아님 ! )
* 병렬로 INSERT할 때 자동으로 Direct Path Insert 로 됨
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
