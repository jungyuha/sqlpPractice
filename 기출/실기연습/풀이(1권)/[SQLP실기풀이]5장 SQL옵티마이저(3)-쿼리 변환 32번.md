문제 링크 : https://velog.io/@yooha9621/SQLP실기문제5장-SQL옵티마이저3-쿼리-변환-32번

# 정리
> #### 🍎 정리
- 고객이 드라이빙 테이블이다.
- 고객 테이블은 풀스캔 처리한다. 👉 `/*+ full(c) */`
- 거래 테이블은 index fast 풀스캔 처리한다.👉 `/*+ index_ffs(거래 거래_PK) */`
   - 거래 테이블을 찌를 때는 데이터가 없는지를 봐야 하므로 거래 테이블 내 모든 데이터를 살펴봐야하며
   - 인덱스 구성은 고객_PK : 고객번호 , 거래_PK : 고객번호 + 거래일시 이므로 거래_PK를 사용하고 테이블까지 갈 필요가 없기 때문이다.
   - 인덱스 힌트 사용법 : index (테이블alias 인덱스명)
- 해시 안티 조인 실행 👉 `/*+ hash_aj */`

# [SQL 답안]
```sql
select /*+ full(c) */ count(*)
from 고객 C
where c. 가입일시 < trunc(add_months(sysdate, -1))
and not exists (
  select /*+ UNNEST hash_aj */ 'x'
  from 거래
  where 고객번호 = c.고객번호
  and rownum <= 1
  )
```
