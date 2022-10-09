# \[SQLP실기풀이-DBCall최소화]53번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-DBCall최소화40번

### 1) 내가 생각한 튜닝 포인트🤔

1. 기존 쿼리는 결과 로우수 만큼 get\_username 함수가 실행된다.
   * 👉 결과의 사원ID 종류 수만큼 get\_username 함수가 실행될 수 있도록 group by절을 추가한다.

### 2) 튜닝한 쿼리

**튜닝 전 쿼리**

```sql
SQL >
select distinct get_username ( 사원ID )
from 휴가기록
where 휴가일자 >= add_months(sysdate,-1);
```

**튜닝 후 쿼리**

```sql
SQL >
select distinct get_username ( 사원ID )
from 휴가기록
where 휴가일자 >= add_months(sysdate,-1)
group by 사원ID;
```

> **🍎 정리**

* 결과의 사원ID 종류 수만큼 get\_username 함수가 실행될 수 있도록 group by절을 추가한다.
