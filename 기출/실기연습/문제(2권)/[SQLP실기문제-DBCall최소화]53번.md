문제 출처 : SQLP 자격검정 핵심노트2 P.45
# 문제
튜닝 SQL을 작성하시오.

## [데이터]
- 휴가기록 : 600개 블록에 5만여 건을 저장한 소형 테이블
- GET_USERNAME : 입력받은 사원ID로 사원 테이블을 조회해서 사원명을 반환하는 함수

## SQL
```sql
SQL> set autotrace traceonly;
SQL> set timing on;
SQL> select distinct get_username ( ID )
     from 휴가기록
     where 휴가일자 >= add_months(sysdate,-1);
```