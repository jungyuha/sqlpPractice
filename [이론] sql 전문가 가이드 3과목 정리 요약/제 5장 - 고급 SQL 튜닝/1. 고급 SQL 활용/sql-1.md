# 고급 SQL 활용(1)

**기록 ✍️**

#### author : jung yuha

#### **first Registered : 2022-04-12**

#### last modified : **2023-04-28**



## \[1] CASE문 활용

* CASE문이나 DECODE 함수를 활용하는 기법은 IFELSE 같은 분기조건을 포함한 복잡한 처리절차를 One-SQL로 구현하는 데 반드시 필요하다.

### (1) CASE문 적용

![](https://velog.velcdn.com/images/yooha9621/post/c9b8195e-3aae-4176-868e-692da8a6d0cf/image.png)

* 왼쪽에 있는 월별납입방법별집계 테이블을 읽어 오른쪽 월요금납부실적과 같은 형태로 가공

**CASE문 적용 전**

```sql
INSERT INTO 월별요금납부실적 (고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷) 
  SELECT K.고객번호, '200903' 납입월 , A.납입금액 지로 , B.납입금액 자동이체 ,
  C.납입금액 신용카드 , D.납입금액 핸드폰 , E.납입금액 인터넷
  FROM 고객 K ,
  (SELECT 고객번호, 납입금액 FROM 월별납입방법별집계 WHERE 납입월 = '200903' AND 납입방법코드 = 'A') A ,
  (SELECT 고객번호, 납입금액 FROM 월별납입방법별집계 WHERE 납입월 = '200903' AND 납입방법코드 = 'B') B ,
  (SELECT 고객번호, 납입금액 FROM 월별납입방법별집계 WHERE 납입월 = '200903' AND 납입방법코드 = 'C') C ,
  (SELECT 고객번호, 납입금액 FROM 월별납입방법별집계 WHERE 납입월 = '200903' AND 납입방법코드 = 'D') D ,
  (SELECT 고객번호, 납입금액 FROM 월별납입방법별집계 WHERE 납입월 = '200903' AND 납입방법코드 = 'E') E 
  WHERE A.고객번호(+) = K.고객번호 AND B.고객번호(+) = K.고객번호 AND C.고객번호(+) = K.고객번호 
  AND D.고객번호(+) = K.고객번호 AND E.고객번호(+) = K.고객번호 
  AND NVL(A.납입금액,0)+NVL(B.납입금액,0)+NVL(C.납입금액,0)+NVL(D.납입금액,0)+NVL(E.납입금액,0) > 0 ;
```

* One-SQL로 작성하는 자체가 중요한 것이 아니라 어떻게 I/O 효율을 달성할 지 중요하다.
* 동일 레코드를 반복 액세스하지 않고 얼마만큼 블록 액세스 양을 최소화할 수 있느냐에 달렸다.

**CASE문 적용 후**

```sql
INSERT INTO 월별요금납부실적 (고객번호, 납입월, 지로, 자동이체, 신용카드, 핸드폰, 인터넷) 
  SELECT 고객번호, 납입월 , NVL(SUM(CASE WHEN 납입방법코드 = 'A' THEN 납입금액 END), 0) 지로 , 
  NVL(SUM(CASE WHEN 납입방법코드 = 'B' THEN 납입금액 END), 0) 자동이체 , 
  NVL(SUM(CASE WHEN 납입방법코드 = 'C' THEN 납입금액 END), 0) 신용카드 , 
  NVL(SUM(CASE WHEN 납입방법코드 = 'D' THEN 납입금액 END), 0) 핸드폰 , 
  NVL(SUM(CASE WHEN 납입방법코드 = 'E' TJEM 납입금액 END), 0) 인터넷 
  FROM 월별납입방법별집계 WHERE 납입월 = '200903' GROUP BY 고객번호, 납입월 ;
```

## \[2] 데이터 복제 기법 활용

* CASE문이나 DECODE 함수를 활용하는 기법은 IFELSE 같은 분기조건을 포함한 복잡한 처리절차를 One-SQL로 구현하는 데 반드시 필요하다.

### (1) 전통적으로 많이 쓰던 방식

```sql
create table copy_t ( no number, no2 varchar2(2) ); 
insert into copy_t select rownum, lpad(rownum, 2, '0')
from big_table where rownum <= 31; 

alter table copy_t add constraint copy_t_pk primary key(no); 
create unique index copy_t_no2_idx on copy_t(no2); 

select * from emp a, copy_t b where b.no <= 2; 
```

* 복제용 테이블(copy\_t)을 미리 만들어두고 이를 활용
* 이 테이블과 조인절 없이 조인(Cross Join)하면 카티션 곱이 발생해 데이터가 2배로 복제된다.
  * 3배로 복제하려면 no <= 3 조건으로 바꿔주면 된다.

```sql
SQL> select rownum no from dual connect by level <= 2;
SQL> select * from emp a, (select rownum no from dual connect by level <= 2) b;
```

* Oracle 9i부터는 dual 테이블을 사용한다.
* dual 테이블에 start with절 없는 connect by 구문을 사용하면 두 레코드를 가진 집합이 자동으로 만들어진다.

**connect by 절 : 3개의 구문으로 구성**

* WHERE : 데이터를 가져온 뒤 마지막으로 조건절에 맞게 정리
* START WITH : 어떤 데이터로 계층구조를 지정하는지 지정한다.
  * 가장 처음에 데이터를 거르는 플랜을 타게 되고, 따라서 이 컬럼에는 인덱스가 걸려있어야 성능을 보장받음
* CONNECT BY : 각 행들의 연결 관계를 설정한다.
  * CONNECT BY 절의 결과에는 LEVEL 이라는 컬럼이 있으며, 이는 계층의 깊이를 의미함.

### (2) 데이터 복제 기법 예시

**카드상품분류와 고객등급 기준으로 거래실적을 집계하면서 소계까지 한번에 구하는 방법 예시**

```sql
select a.카드상품분류 ,(case when b.no = 1 then a.고객등급 else '소계' end) as 고객등급 , sum(a.거래금액) as 거래금액
from  (select 카드.카드상품분류 as 카드상품분류
            , 고객.고객등급 as 고객등급
            , sum(거래금액) as 거래금액
       from   카드월실적 , 카드 , 고객
       where  실적년월 = '201008'
       and    카드.카드번호 = 카드월실적.카드번호 and    고객,고객번호 = 카드.고객번호
       group by 카드.카드상품분류, 고객.고객등급) a , copy_t b
where  b.no <= 2
group by a.카드상품분류, b.no, (case when b.no = 1 then a.고객등급 else '소계' end)
```

## \[3] Union All을 활용한 M:M 관계의 조인

* M:M 관계의 조인을 해결하거나 Full Outer Join을 대체하는 용도로 Union All을 활용한다.

**월별로 각 상품의 계획 대비 판매 실적을 집계**

![](https://velog.velcdn.com/images/yooha9621/post/4e6715db-5101-4851-9551-cdeff0ea93af/image.png)

* 상품과 연월을 기준으로 볼 때 두 테이블은 M:M 관계이므로 조인하면 카티션 곱이 발생

**상품, 연월 기준으로 group by를 먼저 수행하고 나면 두 집합은 1:1 관계가 되므로 Full Outer Join을 통해 원하는 결과집합을 얻을 수 있다.**

```sql
select nvl(a.상품, b.상품) as 상품
     , nvl(a.계획연월, b.판매연월) as 연월
     , nvl(계획수량, 0) 계획수량
     , nvl(판매수량, 0) 판매수량
from  (select 상품 , 계획연월 , sum(계획수량) 계획수량
       from   부서별판매계획
       where  계획연월 between '200901' and '200903'
       group by 상품, 계획연월 ) a
       full outer join
      (select 상품 , 판매연월 , sum(판매수량) 판매수량
       from   채널별판매실적
       where  판매연월 between '200901' and '200903'
       group by 상품, 판매연월 ) b
       on     a.상품 = b.상품
       and    a.계획연월 = b.판매연월;
       
-----------------------------------------------------------------------------------------
| Id  | Operation              | Name           | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |                |    10 |   400 |    10  (40)| 00:00:01 |
|   1 |  SORT ORDER BY         |                |    10 |   400 |    10  (40)| 00:00:01 |
|   2 |   VIEW                 | VW_FOJ_0       |    10 |   400 |     9  (34)| 00:00:01 |
|*  3 |    HASH JOIN FULL OUTER|                |    10 |   400 |     9  (34)| 00:00:01 |
|   4 |     VIEW               |                |     9 |   180 |     4  (25)| 00:00:01 |
|   5 |      HASH GROUP BY     |                |     9 |   180 |     4  (25)| 00:00:01 |
|*  6 |       TABLE ACCESS FULL| CHANNEL_SELL   |     9 |   180 |     3   (0)| 00:00:01 |
|   7 |     VIEW               |                |    10 |   200 |     4  (25)| 00:00:01 |
|   8 |      HASH GROUP BY     |                |    10 |   200 |     4  (25)| 00:00:01 |
|*  9 |       TABLE ACCESS FULL| DEPT_SELL_PLAN |    10 |   200 |     3   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   3 - access("A"."PROD"="B"."PROD" AND "A"."YYYYMM"="B"."YYYYMM")
   6 - filter("YYYYMM">='200901' AND "YYYYMM"<='200903')
   9 - filter("YYYYMM">='200901' AND "YYYYMM"<='200903')
```

**Union All을 이용하면 M:M 관계의 조인이나 Full Outer Join을 쉽게 해결**

```sql
select 상품, 연월, nvl(sum(계획수량), 0) as 계획수량, nvl(sum(실적수량), 0) as 실적수량
from  (select 상품 , 계획연월 as 연월 , 계획수량 , to_number(null) as 실적수량
       from    부서별판매계획
       where   계획연월 between '200901' and '200903'
       union all
       select  상품 , 판매연월 as 연월 , to_number(null) as 계획수량 , 판매수량
       from    채널별판매실적
       where   판매연월 between '200901' and '200903'
       ) a
group by 상품, 연월;
```
