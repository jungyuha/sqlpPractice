# \[SQLP실기문제-대량DML튜닝]31번

문제 출처 : SQLP 자격검정 핵심노트2 P.33

## 문제

온라인 트랜잭션이 없는 아간에 대량 데이터를 일괄 INSERT하는 아래 배치(Batch) 프로그램 성능을 개선하세요. （단，TARGET\_T 테이블에 PK인덱스(TARGET\_T\_PK만 존재하며，다른 인덱스는 없는 상태임)

### \[1] SQL

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
