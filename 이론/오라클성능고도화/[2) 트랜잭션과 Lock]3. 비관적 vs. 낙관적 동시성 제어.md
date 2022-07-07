# 비관적 vs. 낙관적 동시성 제어
- 오라클은 완벽한 문장수준의 읽기 일관성은 보장하나 트랜잭션에 대해서는 기본적으로 보장하지 않는다.
## 1. 비관적 동시성 제어
- 사용자들이 같은 데이터를 동시에 수정할 것을 염두에 두며 동시성이 저해될 것이라 가정하고 트랜잭션을 제어한다.
- 한 사용자가 데이터를 읽는 시점에 Lock을 걸고 조회/갱신이 완료될 때까지 유지한다.
### [예시 1]
- 우수고객 대상으로 적립포인트를 추가한다고 가정할 때 다양한 실적정보, 산출공식에 의한 적립포인트 계산중 다른 트랜잭션이 고객 레코드 변경을 시도한다면 select for update문을 사용하여 Lock을 걸어 데이터가 잘못 갱신되는 문제를 방지한다.
```sql
-- select for update문을 사용
select 적립포인트, 방문횟수, 최근방문일시, 구매실적 from 고객 where 고객번호 = :cust_num for update nowait ;
-- 새로운 적립포인트 계산
update 고객 set 적립포인트 = :적립포인트 where 고객번호 = :cust_num;
```
### wait 또는 nowait 옵션
- 두 옵션 모두 동시성 저해를 방지한다.
  - 가끔 SQLP 객관식에서 'wait 옵션은 동시성을 저해한다.'라는 어감으로 헷갈리게 선지를 내곤 하는데.. 단순히 'wait'이라고해서 무조건 동시성이 저하되는 건 아님을 짚고 넘어가야한다.
  - 즉, 두 옵션 모두 동시성 저해를 방지한다.(두 옵션 모두 동시성 저해에 유익하다는 말씀 😎)
- for update nowait 옵션
  - 대기없이 Exception(ORA-00054)을 던진다.
  - for update wait 3 : 3초 대기 후 Exception(ORA-30006)을 던진다.
## 2. 낙관적 동시성 제어
- 사용자들이 같은 데이터를 동시에 수정하지 않을 것이라 가정하고 데이터를 읽을때 Lock을 설정하지 않은 채 트랜잭션을 제어한다.
- ⭐️ 데이터 수정 시점에 앞서 읽은 데이터의 변경여부 반드시 체크해야한다.
- Lock이 유지되는 시간이 매우 짧아져 동시성을 높이는데 유리하다.
- 하지만 '동시성 제어 없는 낙관적 프로그래밍'이 우려된다.
### [예시 1]
- 다음은 온라인 쇼핑몰 주문처리 중 읽은 데이터 컬럼 전체를 where 조건으로 update 구현하는 트랜잭션 구현 사례이다.
```sql
insert into 주문
select :상품코드, :고객ID, :주문일시, :상점번호, ...
from 상품
where 상품코드 = :상품코드
and 가격 = :가격 ; -- 주문을 시작한 시점 가격

if sql%rowcount = 0 then alert('상품가격이 변경되었습니다.');
end if;
```
### [예시 2]
```sql
select 적립포인트, 방문횟수, 최근방문일시, 구매실적 into :a, :b, :c, :d
from 고객 where 고객번호 = :cust_num;

-- 새로운 적립포인트 계산
update 고객
set 적립포인트 = :적립포인트
where 고객번호 = :cust_num
and 적립포인트 = :a
and 방문횟수 = :b
and 최근방문일시 = :c
and 구매실적 = :d ;

if sql%rowcount = 0 then alert('다른 사용자에 의해 변경되었습니다.');
end if;
```
- 다음은 데이터를 변경하기 직전에 데이터의 변경여부를 따지고자 고객번호를 제외한 4개의 컬럼을 참조하여 4개의 컬럼을 읽은 쿼리이다.하지만 select문에서 읽은 컬럼들이 많은 경우 update문에 조건절을 일일이 기술하는 번거로움이 생긴다.
### [예시 3] 최종 변경일시로 비교하기
- update 대상 테이블에 최종변경일시를 관리하여 조건절에 사용시 간단하게 레코드 갱신여부 확인하는 방법이 있다.
-for update 사용으로 동시성이 저하되는 것을 예방한다.
```sql
-- 새로운 적립포인트 계산
select 고객번호
from 고객
where 고객번호 = :cust_num
and 변경일시 = :mod_dt
for update nowait ; -- 다른 트랜젝션에 의해 설정된 Lock 때문에 동시성이 저하되는 것을 예방한다.

-- 최종 변경일시가 앞서 읽은 값과 같은지 비교
update 고객 set 적립포인트 = :적립포인트, 변경일시 = SYSDATE
where 고객번호 = :cust_num
and 변경일시 = :mod_dt ; 
```
### [예시 4] ora_rowscn을 활용
- 오라클 10g부터 제공하는 Pseudo 컬럼 ora_rowscn을 활용한다.
- Timestamp를 오라클이 직접 관리해 주므로 쉽고 완벽하게 동시성 제어가 가능하다.
- ora_rowscn 이라는 Pseudo 컬럼을 이용하면 특정 레코드가 변경된 후 커밋된 시점을 추적할 수 있다.
- 따로 변경일시 컬럼을 만들지 않고도 동시성 제어가 가능 하다.
```sql
 select 적립포인트, 방문횟수, 최근방문일시, 구매실적, ora_rowscn into :a, :b, :c, :d, :rowscn
from 고객
where 고객번호 = :cust_num;

-- 새로운 적립포인트 계산
update 고객
set 적립포인트 = :적립포인트, 변경일시 = SYSDATE
where 고객번호 = :cust_num
and ora_rowscn = :rowscn ;

if sql%rowcount = 0 then alert('다른 사용자에 의해 변경되었습니다.');
end if;end if;
```
- ora_rowscn을 사용하기 위해서 테이블 생성시 ROWDEPENDENCIES 옵션을 사용해야 한다.
```sql
create table t
ROWDEPENDENCIES
nologging
as
select * from scott.emp;
```
- 기본값은 NOROWDEPENDENCIES인데, 이때는 블록단위로 ora_rowscn이 변경된다. 
### ora_rowscn을 활용시 주의할 점
- ora_rowscn이 변경일시 컬럼을 대체할 수는 없다.즉 동시성 제어 용도로만 사용해야 한다.
- ora_rowscn은 영구히 저장되는 값이지만 이를 시간정보로 변환하는 데 사용되는 매핑정보 테이블(sys.smon_scn_time)의 보관주기는 5일이므로 5일 넘게 지난 ora_rowscn 정보의 Timestamp 값 (SCN_TO_TIMESTAMP(ORA_ROWSCN))을 구하려 하면 에러(ORA-08181)가 발생한다.
- 이는 즉 scn_to_timestamp(ora_rowscn) 으로 변경 후 커밋된 시점을 추적할 수도 있다는 말이다.