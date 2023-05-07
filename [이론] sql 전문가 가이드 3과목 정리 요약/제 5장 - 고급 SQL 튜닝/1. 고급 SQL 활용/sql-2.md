# 고급 SQL 활용(2)

**기록 ✍️**

#### author : jung yuha

#### **first Registered : 2022-04-12**

#### last modified : **2023-04-29**

## \[4] 페이징 처리

* CASE문이나 DECODE 함수를 활용하는 기법은 IFELSE 같은 분기조건을 포함한 복잡한 처리절차를 One-SQL로 구현하는 데 반드시 필요하다.

### (1) 일반적인 페이징 처리용 SQL

**관심 종목에 대해 사용자가 입력한 거래일시 이후 거래 데이터를 페이징 처리 방식으로 조회하는 SQL**

```sql
 -- :pgsize : Fetch해 올 데이터 건수
 -- :page : 출력하고자 하는 페이지 번호
SELECT *
FROM  (SELECT ROWNUM NO , 거래일시 , 체결건수 , 체결수량 , 거래대금
            , COUNT(*) OVER () CNT ------------------------- (1)
       FROM  (SELECT 거래일시 , 체결건수 , 체결수량 , 거래대금
              FROM   시간별종목거래
              WHERE  종목코드 = :isu_cd     -- 사용자가 입력한 종목코드
              AND    거래일시 >= :trd_time  -- 사용자가 입력한 거래일자 또는 거래일시
              ORDER BY 거래일시 ---------------------------- (2)
              )
       WHERE  ROWNUM <= :page * :pgsize + 1 ---------------- (3)
       )
WHERE NO BETWEEN (:page - 1) * :pgsize + 1
AND :pgsize * :page;
```

* 첫 페이지만큼은 가장 최적의 수행 속도를 보인다.
* 대부분 업무에서 앞쪽 일부만 조회하므로 표준적인 페이징 처리 구현 패턴에 가장 적당하다.
* **(1)** : '다음' 페이지에 읽을 데이터가 더 있는지 확인하는 용도
  * CNT > :pgsize \* :page => '다음' 페이지에 출력할 데이터가 더 있음. 필요하지 않으면 (3)번 라인에서 +1을 제거하면 됨.
* **(2)** : \[종목코드 + 거래일시] 인덱스가 있으면 Sort 오퍼레이션 생략된다.
  * 없더라도 TOP-N 쿼리 알고리즘 작동으로 SORT 부하 최소화 할 수 있음.
* **(3)** : page = 10, :pgsize = 3 일 때, 거래일시 순으로 31건만 읽음.
* **(4)** : page = 10, :pgsize = 3 일 때, 읽은 31건 중 21\~30번째 데이터 즉, 3 페이지만 리턴함.

### (2) 뒤쪽 페이지까지 자주 조회할 때

* 뒤쪽의 어떤 페이지로 이동하더라도 빠르게 조회되도록 구현하기 위해서는, 해당 페이지 레코드로 바로 찾아가도록 구현해야 한다.
* 뒤쪽 페이지까지 자주 조회할 때만약 사용자가 ‘다음’ 버튼을 계속 클릭해서 뒤쪽으로 많이 이동하는 업무라면 위 쿼리는 비효율적이다.
  * 인덱스 도움을 받아 NOSORT 방식으로 처리하더라도 앞에서 읽었던 레코드들을 계속 반복적으로 액세스해야 하기 때문
* 인덱스마저 없다면 전체 조회 대상 집합을 매번 반복적으로 액세스한다.
* 뒤쪽의 어떤 페이지로 이동하더라도 빠르게 조회되도록 구현해야 한다면 앞쪽 레코드를 스캔하지 않고 해당 페이지 레코드로 바로 찾아가도록 구현해야 한다.

**첫 번째 페이지를 출력하고 나서 ‘다음’ 버튼을 누를 때의 구현 예시**

```sql
SELECT 거래일시 , 체결건수 , 체결수량 , 거래대금
FROM  (SELECT 거래일시 , 체결건수 , 체결수량 , 거래대금
       FROM   시간별종목거래 A
       WHERE  :페이지이동 = 'NEXT'
       AND 종목코드 = :isu_cd
       AND 거래일시 >= :trd_time
       ORDER BY 거래일시 )
WHERE  ROWNUM <= 11 ;
```

* 한 페이지에 10건씩 출력하는 것으로 가정한다.
* 첫 화면에서는 :trd\_time 변수에 사용자가 입력한 거래일자 또는 거래일시를 바인딩한다.
* 사용자가 ‘다음(▶)’ 버튼을 눌렀을 때는 ‘이전’ 페이지에서 출력한 마지막 거래일시를 입력한다.
* COUNT(STOPKEY)는 \[종목코드 + 거래일시] 순으로 정렬된 인덱스를 스캔하다가 11번째 레코드에서 멈추게 됨을 의미한다.
* ORDER BY 절이 사용됐음에도 실행계획에 소트 연산이 전혀 발생하지 않음

**사용자가 ‘이전(◀)’ 버튼을 클릭했을 때의 구현 예시**

```sql
SELECT 거래일시 , 체결건수 , 체결수량 , 거래대금
FROM  (SELECT 거래일시 , 체결건수 , 체결수량 , 거래대금
       FROM   시간별종목거래 A
       WHERE  :페이지이동 = 'PREV'
       AND 종목코드 = :isu_cd
       AND 거래일시 <= :trd_time
       ORDER BY 거래일시 DESC )
WHERE  ROWNUM <= 11
ORDER BY 거래일시 ;
```

* :trd\_time 변수에는 이전 페이지에서 출력한 첫 번째 거래일시를 바인딩한다.
* ‘SORT (ORDER BY)’가 나타났지만, 'COUNT (STOPKEY)’ 바깥 쪽에 위치했으므로 조건절에 의해 선택된 11건에 대해서만 소트 연산을 수행한다.
* 인덱스를 거꾸로 읽었지만 화면에는 오름차순으로 출력되게 하려고 ORDER BY를 한번 더 사용한 것이다. - 옵티마이저 힌트를 사용하면 SQL을 더 간단하게 구사할 수 있지만 인덱스 구성이 변경될 때 결과가 틀려질 위험성이 있다.
* 될 수 있으면 힌트를 이용하지 않고도 같은 방식으로 처리되도록 SQL을 조정하는 것이 바람직하다.

### (3) Union All 활용

* 설명한 방식은 사용자가 어떤 버튼(조회, 다음, 이전)을 눌렀는지에 따라 별도의 SQL을 호출하는 방식
* Union All을 활용하면 하나의 SQL로 처리하는 것도 가능하다.

```sql
SELECT 거래일시, 체결건수, 체결수량, 거래대금 
FROM ( SELECT 거래일시, 체결건수, 체결수량, 거래대금 
            FROM 시간별종목거래 
            WHERE :페이지이동 = 'NEXT' -- 첫 페이지 출력 시에도 'NEXT' 입력 
            AND 종목코드 = :isu_cd AND 거래일시 >= :trd_time 
            ORDER BY 거래일시 ) 
WHERE ROWNUM <= 11 
UNION ALL 
SELECT 거래일시, 체결건수, 체결수량, 거래대금 
FROM ( SELECT 거래일시, 체결건수, 체결수량, 거래대금 
            FROM 시간별종목거래 
            WHERE :페이지이동 = 'PREV' AND 종목코드 = :isu_cd AND 거래일시 <= :trd_time 
            ORDER BY 거래일시 DESC ) 
WHERE ROWNUM <= 11 
ORDER BY 거래일시 ;
```

## \[5] 윈도우 함수 활용

![](https://velog.velcdn.com/images/yooha9621/post/fbedd8ce-ddbf-469f-bf35-5818ee703854/image.png)

* 장비측정 결과를 저장시 일련번호를 1씩 증가시키면서 측정값을 입력하고, 상태코드는 장비상태가 바뀔 때만 저장된다.
* 조회시 상태코드가 NULL이면 가장 최근에 상태코드가 바뀐 레코드 값을 보여주려고 함.

**부분범위처리 방식으로 앞쪽 일부만 보다 멈춘다면 이 쿼리가 최적**

```sql
select 일련번호 , 측정값
     ,(select /*+ index_desc(장비측정 장비측정_idx) */ 상태코드
       from   장비측정
       where  일련번호 <= o.일련번호
       and    상태코드 is not null
       and    rownum <= 1) 상태코드
from   장비측정 o
order by 일련번호
```

**전체결과를 다 읽어야 한다면 윈도우 함수를 이용하는게 가장 쉬움**

```sql
select 일련번호
, 측정값
, last_value(상태코드 ignore nulls) over(order by 일련번호 rows between unbounded preceding and current row) 상태코드
from   장비측정
order by 일련번호
```

* **last\_value(상태코드 ignore nulls) over(order by 일련번호) 상태코드**
  * rows between unbounded preceding and current row 는 디폴트 값이므로 생략 가능하다.

## \[6] With 구문 활용

```sql
with 위험고객카드 as (select 카드.카드번호
                      from   고객 , 카드
                      where  고객.위험고객여부 = 'Y'
                      and    고객.고객번호 = 카드발급.고객번호)
select v.*
from  (select a.카드번호 as 카드번호
		, sum(a.거래금액) as 거래금액
        , null as 현금서비스잔액 
        , null as 해외거래금액
       from   카드거래내역 a , 위험고객카드 b
       where  조건
       group by a.카드번호
       union all
       select a.카드번호 as 카드번호
       		, null as 현금서비스잔액
            , null as 해외거래금액
       from  (select a.카드번호 as 카드번호
       			, sum(a.거래금액) as amt
              from   현금거래내역 a , 위험고객카드 b
              where  조건
              group by a.카드번호
              union all
              select a.카드번호 as 카드번호
              	, sum(a.결재금액) * -1 as amt
              from   현금결재내역 a , 위험고객카드 b
              where  조건
              group by a.카드번호 ) a
       group by a.카드번호
       union all
       select a.카드번호 as 카드번호
       		, null as 현금서비스잔액
            , null as 현금서비스금액
            , sum(a.거래금액) as 해외거래금액
       from   해외거래내역 a , 위험고객카드 b
       where  조건
       group by a.카드번호 ) v ;
```

* 고객 테이블에는 2천만 건 이상, 카드 테이블에는 1억 건 이상의 데이터가 저장된다.

### (1) With 절 처리 방식

* Oracle은 2가지 방식을 모두 지원한다.
  * 실행방식을 상황에 따라 옵티마이저가 결정한다.
  * 필요하다면 사용자가 힌트(materialize, inline)로써 지정 가능하다.
* SQL Server는 Inline 방식만 실행한다.
* With절을 2개 이상 선언할 수 있으며, With절 내에서 다른 With절을 참조 가능

#### 1. Materialize 방식

* 내부적으로 임시 테이블을 생성함으로써 반복 재사용한다.
* 생성된 임시 데이터는 영구적인 오브젝트가 아니어서, With절을 선언한 SQL문이 실행되는 동안만 유지됨
* 배치 프로그램에서 특정 데이터 집합을 반복적으로 사용할 때 활용
* 전체 처리 흐름을 단순화시킬 목적으로 임시 테이블을 자주 활용함
* Materialize 방식의 With 절을 이용하면 명시적으로 오브젝트를 생성하지 않고도 같은 처리 가능

#### 2. Inline 방식

* 물리적으로 임시 테이블을 생성하지 않으며, 참조된 횟수만큼 런타임 시 반복 수행한다.
