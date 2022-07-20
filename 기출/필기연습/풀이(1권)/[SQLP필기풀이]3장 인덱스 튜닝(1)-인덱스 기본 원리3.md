### ✍️ 27번 : Index Range Scan 가능하도록 재작성
#### Index Range Scan 가능하도록 재작성
1. 
```sql
--전
  SELECT *
  FROM 사원
  WHERE 월급여 * 12 >= 36000000;
--후
  SELECT *
  FROM 사원
  WHERE = 36000000 / 12;
```
2. 
```sql
--전
SELECT *
FROM 주문
WHERE NVL(주문수량, 0) 100;
--후
SELECT *
FROM 주문
WHERE 주문수량 >= 100;
```
3. 
```sql
--전
SELECT *
FROM 주문
WHERE TO_CHAR(일시, 'YYYYMMDD') = idt; 
--후
SELECT *
FROM 주문
WHERE 일시 >= TO_DATE(:dt,'YYYYMMDD')
AND 일시 < TO_DATE(:dt,'YYYYMMDD') + 1;
```
4. 
```sql
--전
SELECT *
FROM 주문
WHERE FLOOR(할인율) :dcrt;
--후 
SELECT *
FROM 주문
WHERE 할인율 < CEIL(:dcrt);
```
5.
```sql
--전
SELECT *
FROM 업체
WHERE SUBSTR(업체명, 1,2) = '대한';
--후
SELECT *
FROM 업체
WHERE 업체명 LIKE '대한%';
```
### ✍️ 28번 : Index Range Scan 가능하도록 재작성
![](https://velog.velcdn.com/images/yooha9621/post/3442fda4-629f-4e3b-91d2-39a3a87d3eed/image.png)

```sql
select 주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where 주문상태코드 <> 3
and 주문일자 between :dt1 and :dt2;
```
#### SQL Index Range Scan 가능하도록 재작성
1. In-List 이용
```sql
select 주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where 주문상태코드 in (0, 1, 2, 4, 5)
and 주문일자 between dt1 and dt2;
```
2. NL 조인
```sql
select /*+ unnest(Osubq) leading(주문상태esubq) use_n1 (주문) */
주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where 주문상태코드 in (select /*+ qb_name(subq) */ 주문상태코드
					from 주문상태
                    where 주문상태코드 <> 3 )
and 주문일자 between dt1 and dt2;
```
3. NL 조인
```sql
select /*+ ordered use_nl(b) */
주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from (select 주문상태코드
		from 주문상태
		where 주문상태코드 ◇ 3) a, 주문 b
where b. 주문상태코드 = a. 주문상태코드
and b. 주문일자 between :dt1 and :dt2 ; 
```
4. or expand
```sql
select /*+ use_concat */
주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where (주문상태코드 < 3 or 주문상태코드 > 3)
and 주문일자 between :dt1 and :dt2;
```
5. union all
```sql
select 주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where 주문상태코드 〈3
and 주문일자 between :dt1 and :dt2
union all
select 주문번호, 주문일시, 고객ID, 총주문금액, 처리상태
from 주문
where 주문상태코드 >3
and 주문일자 between :dt1 and :dt2;
```

### ✍️ 29번 : 특정 인덱스로 Range Scan 가능하도록 재작성
#### 월말계좌상태_PK 인덱스로 Range Scan 가능하도록 재작성
```sql
[인덱스 구성 ]
월말계좌상태_PK : 계좌번호 + 계좌일련번호 + 기준년월
월말계좌상태_X1 : 기준년월 + 상태구분코드
--전
UPDATE 월별계좌상태 SET 상태구분코드 = '87'
WHERE 상태구분코드 '01'
AND 기준년월 = :BASE_DT
AND 계좌번호 || 계좌일련번호 IN
  (SELECT 계좌번호 || 계좌일련번호
  FROM 계좌원장
  WHERE 개설일자 LIKE :STD_YM || '%');
  
--후
UPDATE 월별계좌상태 SET 상태구분코드 = '87'
WHERE 상태구분코드 '01'
AND 기준년월 = :BASE_DT
  AND (계좌번호, 계좌일련번호) IN
(SELECT 계좌번호, 계좌일련번호
FROM 계좌원장
WHERE 개설일자 LIKE STD_YM || '%')
```

### ✍️ 30번 : Index Range Scan 가능하도록 재작성
#### Index Range Scan 가능하도록 재작성
```sql
[ 데이터 타입 ]
지수구분코드 VARCHAR2(1)
지수업종코드 VARCHAR2(3)

[인덱스 구성 ]
일별지수업종별거래_PK : 지수구분코드 + 지수업종코드 + 거래일자
일별지수업종별거래_X1 : 거래일자

--전
SELECT 거래일자
, SUM(DECODE(지수구분코드, '1', 지수종가, 0)) KOSPI200_IDX
, SUM(DECODE(지수구분코드, '1', 누적거래량, 0)) KOSPI200_IDX_TRDVOL
, SUM(DECODE(지수구분코드, '2', 지수종가, 0)) KOSDAQ_IDX
, SUM(DECODE(지수구분코드, '2', 지수종가, 0)) KOSDAQ_IDX
FROM 일별지수업종별거래 A
WHERE 거래일자 BETWEEN :startDd AND :endDd
AND 지수구분코드 || 지수업종코드 IN ('1001', '2003')
GROUP BY 거래일자;
--후
SELECT 거래일자
, SUM(DECODE(지수구분코드, '1', 지수종가, 0)) KOSPI200_IDX
, SUM(DECODE(지수구분코드, '1', 누적거래량, 0)) KOSPI200_IDX_TRDVOL
, SUM(DECODE(지수구분코드, '2', 지수종가, 0)) KOSDAQ_IDX
, SUM(DECODE(지수구분코드, '2', 지수종가, 0)) KOSDAQ_IDX
FROM 일별지수업종별거래 A
WHERE 거래일자 BETWEEN :startDd AND :endDd
AND (지수구분코드 , 지수업종코드) IN (('1','001'), ('2','003'))
GROUP BY 거래일자;
```

### ✍️ 31번 : 인덱스 컬럼이 가공된 SQL 튜닝
#### 인덱스 컬럼이 가공된 SQL 튜닝
```sql
[인덱스 구성 ]
주문_PK : 주문일자 + 주문번호

--튜닝전
SELECT NVL((MAX(주문번호 + 1), 1)
FROM 주문
WHERE 주문일자 = :주문일자 ;

--튜닝후
SELECT NVL(MAX(주문번호),0)+1
FROM 주문
WHERE 주문일자 = :주문일자 ;
```