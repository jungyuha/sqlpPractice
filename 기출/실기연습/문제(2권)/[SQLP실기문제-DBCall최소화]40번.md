문제 출처 : SQLP 자격검정 핵심노트2 P.38
# 문제
아래 프로그램의 문제점을 기술하고, 튜닝 SQL을 작성하시오.

## [ERD 구성]
- 수납 테이블

|수납 |
|---|
|# 수납일자|
|# 고객ID|
|* 수납금액|

- 은행입금내역 테이블

|은행입금내역 |
|---|
|# 입금일시|
|# 고객ID|
|# 은행코드|
|* 입금액|

## [데이터]
- 수납 데이터 보관기간은 10년
- 은행입금내역의 보관기간은 일주일
- 은행입금내역 테이블에서 입금일시 조건을 만족하는 데이터는 20만 건
- 은행입금내역을 고객ID로 GROUP BY 한 결과는 10만 건

## [인덱스 구성]
- 수납_PK : 수납일자 + 고객ID
- 은행입금내역_PK : 입금일시 + 고객ID + 은행코드

## SQL
```sql
SQL >
declare
	1_수납금액 number;
begin
	for c in (select 고객ID, sum(입금액) 입금액
			  from 은행입금내역
			  where to_char(입금일시, 'yyyymmdd') = '20210329'
			  group by 고객ID)
	loop
		begin
          select 수납금액 into l_수납금액
          from 수납
          where 고객ID = c.고객ID
          and 수납일자 = 20210329;
          
          execute immediate
          'update 수납 set 수납금액 = ' || c.입금액 ||
          'where 고객ID = '|| c.고객ID ||
          'and 수납일자 = 20210329';

		  exception
            when no_data_found then
            	execute immediate
                  'insert into 수납(고객ID, 수납일자, 수납금액) values(' 
                  || c. 고객ID || ', 20210329, ' || c. 입금액 || ')';
		end;
		commit;
	end loop;
end;
```