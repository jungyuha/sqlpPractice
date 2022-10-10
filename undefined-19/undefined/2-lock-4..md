# 동시성 구현 사례
## [사례 1] (현재의 MAX값 +1)로 PK 설정하는 경우 
- 데이터가 삽입되는 시점에 실시간으로 현재의 MAX 값을 취해 1만큼 증가시킨 값을 이용한다면 두 개의 트랜젝션이 동시에 같은 값을 읽었을 경우 insert 하려는 순간 PK 제약 위배하는 경우가 생긴다.
- Exception 처리를 통해 동시성 제어를 해야한다.
## [사례 2] MAX 값을 관리하는 별도의 채번 테이블에서 값을 가져오는 방식
### 1. 채번 테이블 생성 및 채번 함수 정의
```sql
-- 채번 테이블 생성
create table seq_tab ( gubun varchar2(1), seq number, constraint pk_seq_tab primary key(gubun, seq) );
organization index;

-- 채번 함수 정의
create or replace function seq_nextval(l_gubun number) return number
as
pragma autonomous_transaction; -- 메인 트랜젝션에 영향을 주지 않고 서브 트랜젝션만 따로 커밋
l_new_seq seq_tab.seq%type;

begin
update seq_tab set seq = seq + 1 where gubun = l_gubun;
select seq into l_new_seq from seq_tab where gubun = l_gubun;
commit;
return l_new_seq;
end;
```
#### pragma autonomous_transaction 옵션을 사용하지 않은 경우
- 메인 트랜젝션의 insert 구문 이후에 롤백하는 경우 앞의 update문까지 이미 커밋된 상태로 되어 데이터 일관성이 깨진다.
- 그렇다고 seq_nextval 함수에서 커밋을 안하면 메인 트랜젝션이 종료될 때까지 채번 테이블에 Lock이 걸린 상태가 되어 성능저하가 초래된다.
- SQL Server에서는 지원하지 않는 기능이며 Savepoint 활용을 권고한다.
### 2. 앞서 정의한 테이블 및 함수를 사용한 트랜젝션 예시
```sql
begin
  update tab1 set col1 = :x where col2 = :y ;
  insert into tab2 values (seq_nextval(123), :x, :y, :z);
  
  loop
     -- do anything ...
  end loop;
  
  commit;
  
  exception
    when others then
    rollback;
end;               
```
## [사례3] 선분이력 정합성 유지
- 선분이력모델은 여러 장점이 있지만 잘못하면 데이터 정합성이 쉽게 깨질 수 있는 단점이 존재하기에 정확히 핸들링하는 방법을 알아야 한다.
### 1. 기본 최종 선분이력을 끊고 새로운 이력 레코드를 추가하는 전형적인 처리 예시
- 신규등록 건이면 ②번 update문에서 실패하고 ③번에서 한 건이 insert 된다.
- 다음 이미지와 같이 히스토리가 쌓인다고 생각하면 쉽다.
![](https://velog.velcdn.com/images/yooha9621/post/6aa6a838-8d21-4a45-964c-7df5f33b951b/image.png)

```sql
declare
cur_dt varchar2(14);

begin
  select 고객ID from 부가서비스이력 where 고객ID = 1 and 종료일시 = to_date('99991231235959', 'yyyymmddhh24miss')
  for update nowait ;

select 고객ID from 고객 where 고객ID = 1 for update nowait ;

cur_dt := to_char(sysdate, 'yyyymmddhh24miss') ; -- ①

update 부가서비스이력 -- ②
set 종료일시 = to_date(:cur_dt, 'yyyymmddhh24miss') - 1/24/60/60
where 고객ID = 1 and 부가서비스ID = 'A' and 종료일시 = to_date('99991231235959', 'yyyymmddhh24miss') ;

insert into 부가서비스이력(고객ID, 부가서비스ID, 시작일시, 종료일시) -- ③
values (1, 'A', to_date(:cur_dt, 'yyyymmddhh24miss'), to_date('99991231235959', 'yyyymmddhh24miss')) ;


commit; -- ④
end;
```
#### select for update문이 없다면?
- 첫 번째 트랜젝션이 ①을 수행하고 ②로 진입하기 직전에 두 번째 트랜젝션이 동일 이력에 대해 ①~④를 먼저 진행해 버린다면 선분이력이 깨진다.
  - 따라서 트랜잭션이 순차적으로 진행할 수 있도록 직렬화 장치가 필요하며 이는 select for update문을 이용해 해당 레코드에 Lock을 설정하여 구현한다.
- 부가서비스이력 테이블에만 select for update로 Lock을 거는 경우
  - 기존에 부가서비스이력이 전혀 없던 고객인 경우 Lock이 걸리지 않아  동시에 두 개 트랜젝션이 ③번 insert문으로 진입하여 시작일시는 다르면서 종료일시는 같은 두 개의 이력 레코드가 생성된다.
  -  즉 , 상위엔티티인 고객 테이블에 select for update로 Lock을 걸어 완벽하게 동시성을 제어해야 한다.
  - 다른 상위엔티티인 부가서비스 테이블에 Lock을 걸 수도 있지만,여러 사용자가 동시에 접근할 가능성이 크기 때문에 동시성이 나빠질 수 있으므로 고객 테이블에 Lock을 설정하는 게 가장 적절하다.