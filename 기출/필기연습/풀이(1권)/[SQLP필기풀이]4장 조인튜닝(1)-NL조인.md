
### ✍️ 1번 : 알맞은 오라클 힌트 기술
#### FROM 절에 나열한 주문, 고객, 결제방식 순으로 조인하되 고객 테이블과는 NL 조인, 결제방식과는 해시 조인하도록 유도하고자 할 때 빈칸에 들어갈 오라클 힌트를 기술하시오.
```sql
select /*+ ? */
ㅇ. 주문번호, o, 고객번호, c.고객명, c. 전화번호, 0. 주문금액, t.결제방식명
from 주문 0, 고객 c, 결제방식 t
where o. 주문일자 >= trunc(sysdate)
and c. 고객번호 = 0. 고객번호
and t. 결제방식코드 = o. 결제방식코드
```
1. **ordered use_nl(c) use_hash(t)** 👉 ⭕️
1. **leading(o c t) use_nl(c) use_hash(t)** 👉 ⭕️
#### 🍋 기출 포인트
1. **FROM 절에 테이블을 나열한 순으로 조인하고자 할 때 ordered 힌트를 사용한다.**
1. **NL 조인으로 유도할 때 use_n1 힌트를 사용하며, 해시 조인으로 유도할 때 use_hash 힌트를 사용한다.**

### ✍️ 2번 : 알맞은 SQL Server 힌트 기술
#### 주문 테이블을 기준으로 고객 테이블과 NL 조인하도록 유도하고자 할 때 빈칸에 들어갈 SQL Server 힌트를 기술하시오.
```sql
select o.주문번호, p. 고객번호, c.고객명, c.전화번호, o. 주문금액
from 주문 o, 고객 c
where o. 주문일자 >= trunc(sysdate)
and c. 고객번호 = o.고객번호
option ( ? )
```
1. **force order, loop join** 👉 ⭕️
#### 🍋 기출 포인트
1. **FROM 절에 테이블을 나열한 순으로 조인하고자 할 때 force order 힌트를 사용한다.**
1. **NL 조인으로 유도할 때 loop join 힌트를 사용한다.**

### ✍️ 3번 : 알맞은 SQL Server 힌트 기술
#### FROM 절에 나열한 주문, 고객, 결제방식 순으로 조인하되, 고객 테이블과는 NL 조인, 결제방식과는 해시 조인하도록 아래 쿼리를 재작성하고 SQL Server 힌트를 기술하시오.
```sql
select o.주문번호, o.고객번호, c.고객명, c.전화번호, o. 주문금액, t. 결제방식명
from 주문 o, 고객 c, 결제방식 t
where 0. 주문일자 >= trunc(sysdate)
and c. 고객번호 = 0. 고객번호
and t. 결제방식코드 = o. 결제방식코드
```
1. 👉 ⭕️
```sql
select o. 주문번호, 0.고객번호, c. 고객명, c, 전화번호, o, 주문금액, t. 결제방식명
from 주문 o
inner loop join 고객 c on (c.고객번호= o.고객번호)
inner hash join 결제방식 t on (t.결제방식코드 = 0.결제방식코드)
where 0. 주문일자 > trunc(sysdate);
```
#### 🍋 기출 포인트
1. **FROM 절에 테이블을 나열한 순으로 조인하고자 할 때 force order 힌트를 사용한다.**
1. **NL 조인으로 유도할 때 inner loop join 구문을 사용한다.**
1. **해시 조인으로 유도할 때 inner hash join 구문을 사용한다.**

### ✍️ 4번 : SQL을 힌트에 따른 조건절 비교 순서
#### 아래 SQL을 힌트에 지시한 대로 수행할 때, 조건절 비교 순서를 올바르게 나열
```sql
[인덱스 구성]
사원_PK : 사원번호
사원X1 : 입사일자
고객_PK : 고객번호
고객_X1 : 관리사원번호

select /*+ ordered use_nl(c) index(e) index(c) */
e.사원번호, e.사원명, e. 입사일자
c.고객번호, c.고객명, c. 전화번호, c.최종주문금액
from 사원 e, 고객 c
where c.관리사원번호 = e.사원번호	--가
and e. 입사일자 > '19960101' --나
and e. 부서코드 = '2123'	--다
and c. 최종주문금액 > 20000	--라
```
1. **나 -> 다 -> 가 -> 라** 👉 ⭕️
#### 🍋 기출 포인트
1. **사원 테이블은 인덱스를 통해서 액세스한다.입사일자와 부서코드에 조건절이 있는데, 부서코드에는 인덱스가 없으므로 입사일자 조건('나'조건)으로 사원X1 인덱스를 스캔해서 얻은 ROWID로 테이블을 액세스한 후 부서코드 조건('다' 조건)을 필터링한다.**
1. **고객 테이블도 인덱스를 통해서 액세스한다.관리사원번호와 최종주문금액에 조건절이 있는데, 최종주문금액에는 인덱스가 없으므로 관리사원번호 조인 조건('가' 조건)으로 고객_X1 인덱스를 스캔해서 얻은 ROWID로 테이블을 액세스한후 최종주문금액 조건('라' 조건)을 필터링한다.**
#### 🍒 문제 해설
1. **고객 테이블 최종주문금액에 인덱스가 있다면, 나 → 다 → 라 → 가 순으로 진행할 수도 있다.어느 쪽이 유리한지는 관리사원번호 조건과 최종주문금액 조건의 데이터 분포에 따라 결정된다.** 

### ✍️ 5번 : 
#### 아래 SQL index 힌트에 인덱스명이나 컬럼명을 지정하지 않았으므로 여러 가지 액세스 경로가 가능하다. 보기는 각 액세스 경로에서 사용하게 될 인덱스를 나열한 것인데, 가장 가능성이 작은 것을 고르시오.
```sql
[인덱스 구성]
사원_PK : 사원번호
사원_X1 : 입사일자
사원_X2 : 부서코드
고객_PX : 고객번호
고객_X1 : 관리사원번호
고객_X2 : 최종주문금액

select /*+ ordered use_nl(c) index(e) index(c) */
e.사원번호, e.사원명, e. 입사일자, c.고객번호, c.고객명, c. 전화번호, c.최종주문금액
from 사원 e, 고객 c
where c. 관리사원번호 = e.사원번호
and e. 입사일자 >= '19960101'
and e. 부서코드 = '2123'
and c. 최종주문금액 >= 20000
```
1. **② 사원_X1, 고객_X1** 👉 ⭕️
1. **③ 사원_X1, 고객_X2** 👉 ⭕️
1. **④ 사원_X2, 고객_X1** 👉 ⭕️
1. **① 사원_PK, 고객_X1** 👉 ❌

#### 🍒 문제 해설
1. **ordered use_nl(c) 힌트를 지정했으므로 사원을 기준으로 고객과 NL 조인한다.**
1. **사원 테이블은 Index(e) 힌트를 지정했으므로 인덱스를 통해서 액세스한다. 
입사일자와 부서코드에 조건절이 있는데, 두 컬럼 모두 인덱스가 있으므로 입사일자 조건('나' 조건으로사원_X1 인덱스를 스캔할 수도 있고, 부서코드 조건으로 사원_X2 인덱스를 스캔할 수도 있다.** 
1. **고객 테이블도 Index(c) 힌트를 지정했으므로 인덱스를 통해서 액세스한다. 관리사원번호와 최종주문금액에 조건절이 있는데, 두 컬럼 모두 인덱스가 있으므로 관리사원번호 조인 조건
('가' 조건)으로 고객X1 인덱스를 스캔할 수도 있고, 최종주문금액 조건('라' 조건)으로 고객X2 인덱스를 스캔할 수도 있다.**

### ✍️ 6번 : NL 조인의 특징
#### NL 조인의 특징
1. **① 랜덤 액세스 위주의 조인 방식이다.** 👉 ⭕️
1. **② 조인할 대상 레코드가 아무리 많아도 ArraySize에 해당하는 최초 N건을 빠르게 출력할 수 있다.** 👉 ⭕️
1. **③ 인덱스 유무, 인덱스 구성에 의해 성능이 크게 달라진다.** 👉 ⭕️
1. **④ 부분범위 처리가 가능한 상황에서만 효과적인 조인 방식이다.** 👉 ❌ 
#### 🍋 기출 포인트
1. **NL 조인은 소량 데이터를 주로 처리하거나 부분범위 처리가 가능한 온라인 트랜잭션 처리(OLTP) 시스템에 적합한 조인 방식이다.**
1. **부분범위 처리가 불가능한 상황이더라도 소량 데이터를 조인할 때는 NL 조인이 가장 유리하다.**
1. ****
#### 🍒 문제 해설
1. **NL 조인은 랜덤 액세스 위주의 조인 방식인데 이는 인덱스 구성이 아무리 완벽해도 대량 데이터 조인할 때 NL 조인이 불리한 이유다.**
1. **조인을 한 레코드씩 순차적으로 진행한다.따라서 부분범위 처리가 가능하다면 조인할 대상 레코드가 아무리 많아도 빠른 응답 속도를 낼 수 있다.** 
1. **순차적으로 진행하므로 먼저 액세스되는 테이블 처리 범위에 의해 전체 일량이 결정된다.**
1. **조인 컬럼에 대한 인덱스가 있느냐 없느냐, 있다면 컬럼이 어떻게 구성됐느냐에
따라 조인 효율이 크게 달라진다.**


### ✍️ 7번 : 가장 효과적인 SOL 튜닝 방안
####  SOL에 대한 튜닝 방안으로 가장 효과적인 것
```sql
* 한 달 간 거래 건수는 평균 20만 건
select p. 상품코드, p. 상품가격, t.거래일자, t.거래수량, t.거래금액
from 상품 p. 거래 t
where p. 상품분류코드 = 'KT6'
and p. 상품가격 between 100000 and 200000
and t. 상품코드 = p. 상품코드
and t. 거래일자 between '20210101' and '20210131·
```
![](https://velog.velcdn.com/images/yooha9621/post/707a1af8-9d17-4c03-9666-42c1d9345ead/image.png)
1. **③ 상품_X01 인덱스에 컬럼을 추가한다.** 👉 ⭕️
#### 🍋 기출 포인트
1. **상품_X01 인덱스는 (1) 상품분류코드가 선두 컬럼이고 상품가격은 포함하지 않거나, (2) 상품가격이 선두 컬럼이고 상품분류코드는 포함하지 않는다. 상품에 대한 조건절이 2개인 상황에서 인덱스에서 9,185건을 얻었는데 테이블 액세스 후에 69건으로 줄었기 때문이다. **
