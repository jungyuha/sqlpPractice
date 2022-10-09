문제 출처 : SQLP 자격검정 핵심노트2 P.35
# 문제
야간 배치(Batch) 프로그램에서 수행하는 아래 UPDATE 문을 튜닝하고, 정확히 원하는 실행계획이 나오도록 조인 순서와 방식, 인덱스 힌트를 기술하시오. (인덱스 추가 및 변경은 불가)
## [데이터]
- 고객 = 100만 명
- 미성년자(성인여부 = 'N') = 2%
- 법정대리인을 등록한 미성년자 = 50%
## [인덱스 구성]
- 고객_PK : 고객번호
- 고객_X1 : 고객명
- 고객_X2 : 연락처
- 고객_X3 : 법정대리인_고객번호
## SQL
```sql
SQL >
UPDATE 고객 C
	SET 법정대리인_연락처 =
    	NVL( (SELECT 연락처
				FROM 고객
				WHERE 고객번호 = C.법정대리인_고객번호)
			,C.법정대리인_연락처)
WHERE 성인여부 = 'N'

Execution Plan
-----------------------------------------------------------------
UPDATE STATEMENT Optimizer=ALL_ROWS (Cost=32 Card=14 Bytes=126)
	UPDATE OF '고객'
		TABLE ACCESS (FULL) OF '고객' (TABLE) (Cost=4 Card=14 Bytes=126)
		TABLE ACCESS (BY INDEX ROWID) OF '고객' (TABLE) (Cost=1 Card=1 Bytes=13)
			INDEX (UNIQUE SCAN) OF '고객번호_PK' (INDEX (UNIQUE)) (Cost=0 Card=1)

```