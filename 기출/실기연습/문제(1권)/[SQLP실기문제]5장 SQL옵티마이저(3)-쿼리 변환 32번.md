문제 : 1권 p.235

# 문제
아래 SQL 튜닝하되, 원하는 방식으로 실행되도록(=절대 다른 방식으로 실행되지 못하도록)
힌트를 정확히 기술하고, 필요시 인덱스 구성 방안도 제시하시오.
# [데이터]
고객 : 100만 건
거래 : 1,000만 건
# [인덱스 구성]
고객_PK : 고객번호
거래_PK : 고객번호 + 거래일시

# [SQL]
```sql
select count(*)
from 고객 C
where c. 가입일시 < trunc(add_months(sysdate, -1))
and not exists (
  select 'x'
  from 거래
  where 고객번호 = c.고객번호
  and rownum <= 1
  )
```

# [기존 실행계획]
![](https://velog.velcdn.com/images/yooha9621/post/74e5b26d-70d8-43c9-af1c-2e642921282f/image.png)
