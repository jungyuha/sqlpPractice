출처 : SQLP 기출노트 1권 ( p.38 ~ p.40)
### ✍️ 1,2번 : 실행계획 출력 방법
#### 1) Oracle 실행계획 출력 방법
```sql
SQL > explain plan for
select * from emp where ename =: ename;

SQL > select * from table(dbms_xplan.display(null,null,'typical');
```
#### 2) SQL SERVER 실행계획 출력 방법
```sql
use pubs
go
set showplan_text on
go
select * from table(dbms_xplan.display(null,null,'typical');
go
```
#### 🍋 기출 포인트
1. **⭐️ 오라클에서 'explain plan for' [쿼리] 를 실행하면 실행계획이 PLAN TABLE에 저장된다. ⭐️**
2. **⭐️ 오라클에서 PLAN TABLE에 저장된 정보를 쉽게 포맷팅하여 읽을려면 dbms_xplan.display(opt1,opt2,opt3) 메서드를 사용한다. ⭐️**
2. **⭐️ SQL SERVER에서 'set showplan_text on' 명령어를 실행한 후에 SQL을 실행하면 실행계획을 출력할 수 있다. ⭐️**

### ✍️ 3번 : 실행계획에서 확인할 수 있는 정보, 없는 정보
#### Oracle 예상 실행계획에서 확인할 수 있는 정보
1. 예상 Cardinality
2. 예상 Cost
3. 조건절 정보 ( Predicate Information )

#### Oracle 예상 실행계획의 모양새
```sql
SQL > explain plan for
select * from emp where ename =: ename;

SQL> select * from table(dbms_xplan.display(null,null,'typical');

PLAN_TABLE_OUTPUT
 ----------------------------------------------------------------------------- 
Plan hash value: 4024650034

------------------------------------------------------------------------------ 
|ID | Operation                         | Name | Rows | Bytes | Cost (%CPU)  |
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT                  |      |      |     32|     1     (0)|
|  1|  TABLE ACCESS BY INDEX ROWID      |EMP   |     1|     32|     1     (0)|
|* 2|   INDEX UNIQUE SCAN               |EMP_PK|     1|       |     0     (0)|
------------------------------------------------------------------------------
Predicate Information (identified by operation id):
------------------------------------------------------------------------------
  2 - access("EMPNO"=7900)
```
#### 🍋 기출 포인트
1. **⭐️ 오라클에서 예상 SORT는 예상 실행계획에서 확인할 수 없다.⭐️**

### ✍️ 4번 : AutoTrace에서 확인할 수 있는 정보, 없는 정보
#### Oracle AutoTrace에서 확인할 수 있는 정보
1. 예상 실행계획
2. 실제 디스크에서 읽은 블록 수
3. 실제 기록한 Redo 크기

#### Oracle AutoTrace에서 확인할 수 없는 정보
1. 실제 사용한 CPU Time

#### Oracle AutoTrace의 모양새
```sql
SQL > set autotrace traceonly;

SQL > select * from emp where ename =: ename;

Execution Plan -> 예상 실행계획
 ----------------------------------------------------------------------------- 
Plan hash value: 4024650034

------------------------------------------------------------------------------ 
|ID | Operation                         | Name | Rows | Bytes | Cost (%CPU)  |
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT                  |      |      |     32|     1     (0)|
|  1|  TABLE ACCESS BY INDEX ROWID      |EMP   |     1|     32|     1     (0)|
|* 2|   INDEX UNIQUE SCAN               |EMP_PK|     1|       |     0     (0)|
------------------------------------------------------------------------------
Predicate Information (identified by operation id):
------------------------------------------------------------------------------
  2 - access("EMPNO"=7900)
  
 Statistics
----------------------------------------------------------
 0 recursive calls
 0 db block gets
 0 consistent gets
 0 physical reads -> 실제 디스크에서 읽은 블록 수
 0 redo size -> 실제 기록한 Redo 크기
 519 bytes sent via SQL*Net to client
 523 bytes received via SQL*Net from client
 2 SQL*Net roundtrips to/from client
 0 sorts (memory)
 0 sorts (disk)
 1 rows processed
```
#### 🍋 기출 포인트
1. **⭐️ 오라클에서 실제 사용한 CPU Time는 Autotrace 실행통계에서 확인할 수 없다.⭐️**

### ✍️ 5번 : AutoTrace 옵션
#### 다양한 AutoTrace 옵션
  - [예시]
  ```sql
      SQL> set autotrace on
       SQL> select * from scott.emp where empno=7900;
  ```
- **set autotrace on**
  - SQL을 실행하고 그결과와 함께 실행 계획 및 실행통계를 출력한다.
- **set autotrace on explain**
  - SQL을 실행하고 그결과와 함께 실행 계획을 출력한다.
- **set autotrace on statistics**
  - SQL을 실행하고 그결과와 함께 실행통계를 출력한다.
- **set autotrace traceonly**
  - SQL을 실행하지만 그 결과는 출력하지 않고, 실행계획과 실행통계만을 출력한다.
  - 실행 통계를 보여줘야 하므로 쿼리를 실제 수행한다.
- **set autotrace traceonly explain**
  - SQL을 실행하지않고 실행계획만을 출력한다.
- **set autotrace traceonly statistics**
  - SQL을 실행하지만 그 결과는 출력하지 않고, 실행통계만을 출력한다.
  - 실행 통계를 보여줘야 하므로 쿼리를 실제 수행한다.

### ✍️ 6,7,8번 : 오라클 SQL 트레이스 확인하기
#### 오라클 SQL 트레이스 확인하기 
```sql
SQL > alter session set sql_trace = true
SQL > select * from emp where ename =: ename;
```
#### 오라클 SQL 트레이스에서 확인할 수 있는 정보 
1. 하드파싱 횟수
2. 실제 사용한 CPU Time
3. 실제 디스크에서 읽은 블록 수
#### 오라클 SQL 트레이스에서 확인할 수 없는 정보 
1. 실제 기록한 Redo 크기
#### Oracle SQL 트레이스의 모양새
```sql
SELECT
    EMPNO,
    JOB,
    TO_CHAR(HIREDATE, :"SYS_B_0") HIREDATE,
    SAL
FROM EMP
WHERE
    SAL > :"SYS_B_1"

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      2      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          0          8          0          14
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        5      0.00       0.00          0          8          0          14

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: FIRST_ROWS
Parsing user id: 29

Rows     Row Source Operation
-------  ---------------------------------------------------
     14  TABLE ACCESS FULL EMP (cr=8 pr=0 pw=0 time=47 us)

```
#### 🍋 기출 포인트
1. **⭐️ 오라클 SQL 트레이스 확인하려면 쿼리 수행전 'alter session set sql_trace = true' 명령어를 수행한다. ⭐️**
2. **⭐️ 오라클 SQL 트레이스 파일을 분석해서 리포트 파일로 보고 싶다면 TKProf 유틸리티를 활용한다. ⭐️**
3. **⭐️ 오라클에서 실제 기록한 Redo 크기는 오라클 SQL 트레이스에서 확인할 수 없다.⭐️**

### ✍️ 9번 : Autotrace의 statistics와 SQL 트레이스 항목 비교
#### Oracle AutoTrace의 모양새
```sql
SQL > set autotrace traceonly;

SQL > select * from emp where ename =: ename;

Execution Plan -> 예상 실행계획
 ----------------------------------------------------------------------------- 
Plan hash value: 4024650034

------------------------------------------------------------------------------ 
|ID | Operation                         | Name | Rows | Bytes | Cost (%CPU)  |
------------------------------------------------------------------------------ 
|  0| SELECT STATEMENT                  |      |      |     32|     1     (0)|
|  1|  TABLE ACCESS BY INDEX ROWID      |EMP   |     1|     32|     1     (0)|
|* 2|   INDEX UNIQUE SCAN               |EMP_PK|     1|       |     0     (0)|
------------------------------------------------------------------------------
Predicate Information (identified by operation id):
------------------------------------------------------------------------------
  2 - access("EMPNO"=7900)
  
 Statistics
----------------------------------------------------------
 0 recursive calls
 0 db block gets
 0 consistent gets
 0 physical reads -> 실제 디스크에서 읽은 블록 수
 0 redo size -> 실제 기록한 Redo 크기
 519 bytes sent via SQL*Net to client
 523 bytes received via SQL*Net from client
 2 SQL*Net roundtrips to/from client
 0 sorts (memory)
 0 sorts (disk)
 1 rows processed
```
#### Oracle SQL 트레이스의 모양새
```sql
SELECT
    EMPNO,
    JOB,
    TO_CHAR(HIREDATE, :"SYS_B_0") HIREDATE,
    SAL
FROM EMP
WHERE
    SAL > :"SYS_B_1"

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      2      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          0          8          0          14
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        5      0.00       0.00          0          8          0          14

Misses in library cache during parse: 1
Misses in library cache during execute: 1
Optimizer mode: FIRST_ROWS
Parsing user id: 29

Rows     Row Source Operation
-------  ---------------------------------------------------
     14  TABLE ACCESS FULL EMP (cr=8 pr=0 pw=0 time=47 us)

```
1. **consistent gets는 query와 같다.** 
2. **SQL*Net roundtrips to/from client는 fetch call count와 같다.** 
3. **rows processed는 fetch rows와 같다.** 
4. **recursive calls는 하드파싱 과정에 딕셔너리를 조회하거나 DB저장형 함수에 내장된 SQL을 수행할 때 발생한 Call 횟수이다.** 

### ✍️10,11번 : SGA 메모리에 기록한 오라클 SQL 트레이스 확인하기
```sql
SQL > select /*+ gather_plan_statistics */ count(*) from emp;

SQL > select * from table(dbms_xplan.display_cursor(null,null,'allstats last')));
```
1. **gather_plan_statistics** 힌트를 사용하면 SQL 트레이스 정보를 서버 파일이 아닌 SGA 메모리에 기록한다.
2. SGA 메모리에 저장된 트레이스 정보를 **dbms_xplan.display_cursor** 함수를 이용하면 분석하기 쉬운 형태로 포매팅해준다. 
3. **dbms_xplan.display_cursor** 의 첫번째 인자는 SQL 커서의 ID , 두번째 인자에는 CHILD_NUMBER를 입력한다.
   -  첫번재 인자와 두번째 인자가 모두 null이면 바로 직전에 수행한 SQLd의 커서 ID와 child_number를 내부에서 자동 선택한다.
#### 🍋 기출 포인트
1. **⭐️gather_plan_statistics 힌트를 사용하면 SQL 트레이스 정보를 서버 파일이 아닌 SGA 메모리에 기록한다. ⭐️**
2. **⭐️SGA 메모리에 저장된 트레이스 정보를 dbms_xplan.display_cursor 함수를 이용하면 분석하기 쉬운 형태로 포매팅해준다. ⭐️**
2. **⭐️dbms_xplan.display_cursor 의 첫번째 인자는 SQL 커서의 ID , 두번째 인자에는 CHILD_NUMBER를 입력한다.⭐️**

### ✍️12,13번 : dbms_xplan.display_cursor 함수 관련
1. SGA 메모리에 저장된 트레이스 정보를 보기 위한 **dbms_xplan.display_cursor** 함수를 사용하기 전 사용해야할 힌트 또는 관련된 사항의 목록은 다음과 같다.
   - **gather_plan_statistics** 힌트 사용 
   - statistics_level 파라미터를 all로 설정
   - _rowsource_execution_statistics 파라미터를 true로 설정
#### 🍋 기출 포인트
1. **⭐️monitor 힌트는 실시간 SQL 모니터링을 위해 사용하는 힌트로 dbms_xplan.display_cursor 함수와는 상관이 없다. ⭐️**
