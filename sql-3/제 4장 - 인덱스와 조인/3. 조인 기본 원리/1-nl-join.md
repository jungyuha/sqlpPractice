# (1) NL Join

## \[1] Nested Loop Join

### (1) 중첩 루프문 원리

```java
begin
  for outer in (select deptno, empno, rpad(ename, 10) ename from emp) loop    -- outer 루프
    for inner in (select dname from dept where deptno = outer.deptno) loop  -- inner 루프
      dbms_output.put_line(outer.empno||' : '||outer.ename||' : '||inner.dname);
    end loop;
  end loop;
end;
```

### (2) 오라클 사용예제

```sql
-- <1>
SELECT /*+ ordered use_nl(d) */ E.EMPNO, E.ENAME, D.DNAME
  FROM EMP E, DEPT D
 WHERE D.DEPTNO = E.DEPTNO
-- <2>
SELECT /*+ leading(e) use_nl(d) */ E.EMPNO, E.ENAME, D.DNAME
  FROM DEPT D, EMP E
 WEHRE D.DEPTNO = E.DEPTNO
```

* ordered : FROM 절에 나열된 테이블 순서대로 조인
* use\_nl(d) : NL(Nested Loop) JOIN을 이용하여 조인
  * d : EMP 테이블을 OUTER/DRIVING TABLE로 조인

### (3) NL Join 수행 과정 분석

#### 1. NL Join 수행 예제

```sql
SELECT /*+ ordered use_nl(e) */ E.EMPNO, E.ENAME, D.DNAME, E.JOB, E.SAL
FROM DEPT D, EMP E
 WHERE D.DEPTNO = E.DEPTNO ............①
   AND D.LOC = 'SEOUL'.................②
   AND D.GB = '2'......................③
   AND E.SAL >= 1500...................④
 ORDER BY SAL DESC
```

* 사용되는 인덱스 : DEPT\_LOC\_IDX, EMP\_DEPTNO\_IDX
* 조건비교 순서 : ② → ③ → ① → ④
* 조인 과정 ![](https://velog.velcdn.com/images/yooha9621/post/9cebfaf2-9d2d-4a20-b63d-256d06cba117/image.png)
  * 1, 19, 31, 32 : 스캔할 데이터가 더 있는지 확인하는 one-plus 스캔
    * ※one-plus 스캔 : Non Unique Scan
  * NL Join의 첫 번째 부하지점 :: 전체 일량은 dept\_loc\_idx 인덱스를 스캔하는 양
    * 단일 칼럼 인덱스를 '=' 조건으로 스캔(Non Unique Scan)을 비효율 없이 6(=5+1)건
    * 그만큼 테이블 Random 액세스가 발생
  * NL Join의 두 번째 부하지점 :: emp\_deptno\_idx 인덱스를 탐색하는 부분
    * Outer 테이블인 dept를 읽고 나서 조인 액세스가 얼마만큼 발생하느냐에 의해 결정
    * Random 액세스에 해당하며, emp\_deptno\_idx의 높이(height)가 3이면 매 건마다 그만큼의 블록 I/O가 발생하고, 리프 블록을 스캔하면서 추가적인 블록 I/O가 더해진다.
  * NL Join의 세 번째 부하지점 :: emp\_deptno\_idx를 읽고 나서 emp 테이블을 액세스하는 부분.
    * 여기서도 sal >= 1500 조건에 의해 필터링되는 비율이 높다면 emp\_deptno\_idx 인덱스에 sal 칼럼을 추가하는 방안을 고려해야 한다.

### (4) NL Join의 특징

#### 1. 액세스 위주의 조인 방식

* 하나의 레코드를 읽으려고 블록을 통째로 읽는 Random 액세스 방식은 설령 메모리 버퍼에서 빠르게 읽더라도 비효율이 존재
* 따라서 인덱스 구성이 아무리 완벽하더라도 대량의 데이터를 조인할 때 매우 비효율적이다.

#### 2. 조인을 한 레코드씩 순차적으로 진행

* 한 레코드씩 액세스 위주의 조인 방식 때문에 대용량 데이터 처리 시 매우 치명적인 한계가 있다.
* 반대로 조인을 한 레코드씩 순차적으로 진행하기 때문에 아무리 대용량 집합이더라도 매우 극적인 응답 속도를 가진다.
  * 부분범위처리가 가능한 상황에서 그렇다.
* 먼저 액세스되는 테이블의 처리 범위에 의해 전체 일량이 결정된다.

#### 3. 인덱스 구성 전략이 특히 중요하다.

#### 4. NL Join을 고려하는 방식

* OLTP 시스템에서 조인을 튜닝할 때 일차적으로 NL Join부터 고려하는 것이 올바른 순서이다.
* 우선, NL Join 메커니즘을 따라 각 단계의 수행 일량을 분석해 과도한 Random 액세스가 발생하는 지점을 파악한다.
* 조인 순서를 변경해 Random 액세스 발생량을 줄일 수 있는 경우가 있지만, 그렇지 못할 때는 인덱스 칼럼 구성을 변경하거나 다른 인덱스의 사용을 고려해야 한다.
* 여러 가지 방안을 검토한 결과 NL Join이 효과적이지 못하다고 판단될 때 Hash Join이나 Sort Merge Join을 검토한다.
