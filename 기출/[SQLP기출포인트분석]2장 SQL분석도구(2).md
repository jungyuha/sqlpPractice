출처 : SQLP 기출노트 1권 ( p.40 ~ p.)
### ✍️ 13번 : dbms_xplan.display_cursor 트레이스 정보
#### 오라클 dbms_xplan.display_cursor 트레이스에서 확인할 수 있는 정보 
1. **Starts** : 각 오퍼레이션 단계별 실행 횟수
2. **E-Rows** : 옵티마이저가 예상한 Rows
3. **A-Rows** : 각 오퍼레이션 단계에서 읽거나 갱신한 로우 수
   - 오라클 sql 트레이스에서의 **rows** 항목과 같다.
4. **A-Times** : 각 오퍼레이션 단계별 소요시간
   - 오라클 sql 트레이스에서의 **time** 항목과 같다.
5. **Buffers** : 캐시에서 읽은 버퍼 블록 수
   - 오라클 sql 트레이스에서의 **cr(=query),current** 항목과 같다.
6. **Reads** : 디스크에서 읽은 블록수
   - 오라클 sql 트레이스에서의 **pr** 항목과 같다.
#### Oracle dbms_xplan.display_cursor 트레이스의 모양새
```sql
-----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name                   | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |
-----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |                        |      1 |        |   1989 |00:00:04.96 |    9280 |    897 |
|   1 |  NESTED LOOPS OUTER                   |                        |      1 |   2125 |   1989 |00:00:04.96 |    9280 |    897 |
|   2 |   NESTED LOOPS OUTER                  |                        |      1 |   2125 |   1989 |00:00:04.93 |    9271 |    895 |
|   3 |    NESTED LOOPS OUTER                 |                        |      1 |   2125 |   1989 |00:00:00.03 |    5732 |      0 |
|   4 |     COLLECTION ITERATOR PICKLER FETCH |                        |      1 |   1989 |   1989 |00:00:00.01 |       0 |      0 |
|*  5 |     TABLE ACCESS BY INDEX ROWID       | TABLE1                 |   1989 |      1 |   1178 |00:00:00.03 |    5732 |      0 |
|*  6 |      INDEX RANGE SCAN                 | IDX_TABLE1             |   1989 |      2 |   2197 |00:00:00.02 |    3545 |      0 |
|   7 |    TABLE ACCESS BY INDEX ROWID        | TABLE2                 |   1989 |      1 |   1178 |00:00:03.26 |    3539 |    895 |
|*  8 |     INDEX UNIQUE SCAN                 | IDX_TABLE2_PK          |   1989 |      1 |   1178 |00:00:03.25 |    2359 |    895 |
|   9 |   TABLE ACCESS BY INDEX ROWID         | TABLE3                 |   1989 |      1 |      0 |00:00:00.03 |       9 |      2 |
|* 10 |    INDEX UNIQUE SCAN                  | IDX_TABLE3_PK          |   1989 |      1 |      0 |00:00:00.03 |       9 |      2 |
-----------------------------------------------------------------------------------------------------------------------------------
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
#### 🍋 기출 포인트
1. **⭐️ 오라클에서 'explain plan for' [쿼리] 를 실행하면 실행계획이 PLAN TABLE에 저장된다. ⭐️**

### ✍️ 14,15번 : SQL SERVER에서 트레이스 확인하기
#### SQL SERVER에서 트레이스 설정 옵션
```sql
use pubs
go
set statistics profiles on
set statistics io on
set statistics time on
go
select * from dbo.employee
go
```
1. **set statistics profiles on** 
- 각 쿼리가 일반 결과 집합을 반환하고 그 뒤에는 쿼리 실행 프로필을 보여 주는 추가 결과 집합을 반환한다. 출력에는 다양한 연산자에서 **처리한 행수** 및 **연산자의 실행 횟수**에 대한 정보도 포함한다.
1. **set statistics io on** 
- Transact-SQL 문이 실행되고 나서 해당 문에서 만들어진 **디스크 동작 양**에 대한 정보를 표시한다.
1. **set statistics time on** 
- 각 Transact-SQL 문을 **구문 분석**, **컴파일** 및 **실행**하는 데 사용한 시간을 밀리초(0.001초)단위로 표시한다.


#### 🍋 기출 포인트
1. **⭐️ SQL SERVER에서 트레이스 옵션 설정 ⭐️**
- 쿼리 실행 프로필을 보여 주는 추가 결과 집합을 보려면 **set statistics profiles on** 옵션을 쓴다.
2. **⭐️ SQL SERVER에서 트레이스 옵션에서 확인할 수 있는 것들 ⭐️**
- 구문 분석 시간 , 컴파일 시간 , 논리적/물리적 읽기 수 , 각 오퍼레이션 단계별 실행 횟수
3. **⭐️ SQL SERVER에서 트레이스 옵션에서 확인할 수 없는 것 ⭐️**
- Fetch call 횟수

### ✍️ 16번 : 대기이벤트가 나타나는 시점
#### 대기 이벤트가 나타나는 시점
1. **프로세스가 CPU를 OS에 반환 후 수면 상태로 진입할 때**
2. **프로세스가 필요로 하는 특정 리소스가 다른 프로세스에 의해 사용 중일 때**
3. **⭐️ 프로세스가 할 일이 없을 때 ⭐️**

#### 🍋 기출 포인트
1. **프로세스가 공유 메모리에서 래치를 획득할 때마다(버퍼캐시 , 라이브러리 캐시 ..)
대기 이벤트가 일어나는 건 아니다.**
	- 획득 과정 중 경합이 발생할 때에만 대기 이벤트가 발생한다.

### ✍️ 17번 : 하드 파싱과 가장 관련이 있는 대기 이벤트
#### 하드 파싱시 새로운 커서를 shared pool에서 할당하는 과정에서 대기 이벤트가 발생한다.
1. **latch : shared pool**
#### 아래는 그닥 하드 파싱과 관련이 없는 대기 이벤트이다.
1. **Library cache lock**
- 주로 SQL 수행 도중 DDL을 수행할 때 나타난다.
-[참고](http://wiki.gurubee.net/pages/viewpage.action?pageId=3902139)
2. **free buffer waits**
- 서버 프로세스가 버퍼 캐시에서 Free Buffer를 찾지 못해 DBWR에게 공간을 확보해 달라고 신호를 보낸 후 대기할 때 나타난다.
3. **log file sync**
- 커밋 명령을 전송받은 서버 프로세스가 LGWR에게 로그 버퍼를 로그 파일에 기록해 달라고 신호를 보낸 후 대기할 때 나타난다.

#### 🍋 기출 포인트
1. **⭐️ 하드 파싱시 새로운 커서를 shared pool에서 할당하는 과정에서 latch : shared pool 대기 이벤트가 발생한다. ⭐️**
2. **⭐️ latch : shared pool 대기 이벤트는 하드 파싱을 동시에 심하게 일으킬 때 주로 나타난다. ⭐️**

### ✍️ 18,19번 : 응답 시간 분석(대기 이벤트 기반) 성능 관리 방법론
#### 대기 이벤트를 기반으로 세션 또는 시스템 전체에 발생하는 병목 현상과 그 원인을 찾아 문제를 해결하는 방법 과정을 '대기 이벤트 기반' 또는 '응답 시간 분석' 성능관리 방법론이라고 한다.
1. **Response Time = Service Time + Wait Time
또는
Response Time = CPU Time + Queue Time**


#### 🍋 기출 포인트
1. **⭐️ Service = Cpu , wait = Queue ⭐️**

### ✍️ 20,21번 : 오라클 AWR
#### 
1. **응답 시간 분석 방법론의 단점을 보완하기 위해 Ratio 기반 성능 분석 방법론을 도입했다.=> 틀림 ** 
   - 오라클은 전통적으로 Ratio 기반 성능 분석 방법론을 사용했다.
   - 이에 응답 시간 분석 방법론을 더해 Statpack을 개발하였다.
2. **오라클 9i버전 까지 사용하던 성능관리 패키지 Statpack을 업그레이드해서 개발하였다. => 딩동댕 ** 
3. **AWR을 활용해 효과적으로 문제점을 찾아내려면 성능 이슈가 발생했던 시점을 전후해 가능한한 긴 스냅샷 구간(하루 정도)에 대한 리포트를 출력해서 분석해야 한다. => 틀림**
   - 성능 이슈를 AWR을 통해 해결하려면 peak 시간대 , 장애 발생 시점을 전후에 가능한한 짧은 스냅샷 구간을 선택한다.
4. **성능 자료를 수집해서 AWR에 저장할 때 'dba_hist_'로 시작하는 각종 뷰를 이용한다. => 틀림 **
   - Statpack은 SQL로 딕셔너리를 조회해서 성능 정보를 수집해 시스템에 부하가 많았다. 그래서 스냅샷 주기를 짧게 설정하기가 곤란했다.
   - AWR은 뷰를 조회하지 않고 DMA 방식으로 SGA 메모리를 직접 액세스해서 성능 정보를 수집한다.
   - AWR 보고서에 출력되는 항목들은 `dba_hist_`로 시작하는 각종 뷰를 이용해 사용자가 직접 조회할 수 있다.

#### 🍋 기출 포인트
1. **⭐️ Ratio => statpack => AWR 순이다. ⭐️**
2. **⭐️ AWR 보고서 요약에는 SQL 통계가 들어가지 않는다. ⭐️**

### ✍️ 22번 : 파싱과 관련있는 트레이스 항목
1. **Soft Parse %** 
- 실행계획이 라이브러리 패시에서 찾아져서 하드파싱을 일으키지 않고 SQL을 수행한 비율
2. **Execute to Parse %** 
- Parse Call 없이 곧바로 SQL을 수행한 비율로 커서를 애플리케이션에서 캐싱한 체 반복 수행한 비율이다.
3. **parse CPU to Parse Elapsd %** 
- 파싱 총 소요 시간 중 CPU Time이 차지한 비율이다.파싱에 소요된 시간 중 실제 일을 수행한 시간 비율을 말하며 이 값이 낮으면 파싱 도중 대기가 많이 발생했음을 의미한다.

### ✍️ 23,24번 : ASH
1. **Ratio 기반 성능 분석 방법론과 시스템 레벨 대기 이벤트 분석 방법론의 한계를 극복하기 위해 세션 레벨 실시간 모니터링 기능을 ASH 이라고 한다.(오라클 10g부터) %** 
- 실행계획이 라이브러리 패시에서 찾아져서 하드파싱을 일으키지 않고 SQL을 수행한 비율
2. **ASH에 저장된 정보 중 오래되면 AWR 즉, 디스크에 기록한다.** 
3. **ASH에서 과거 세션 히스토리 정보는 dba_his_active_sess_history 뷰를 통해 조회할 수 있다. %** 

#### 🍋 기출 포인트
1. **⭐️ dba_his_active_sess_history 뷰는 '현재' 발생하고있는 장애 원인을 분석하기엔 마땅치 않다. ⭐️**