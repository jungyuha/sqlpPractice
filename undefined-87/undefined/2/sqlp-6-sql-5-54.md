# \[SQLP실기풀이]6장 고급SQL튜닝(5) 대용량 배치 프로그램 튜닝 54번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대용량배치프로그램튜닝54번

### 1) 기존 쿼리 분석

기존 쿼리의 순서는 다음과 같다.

1. 상품기본이력임시 테이블 병렬 FULL SCAN 후 TQ10000 테이블큐를 통해 브로드캐스팅
2. 상품기본 테이블 병렬 FULL SCAN 후 브로드캐스팅으로 받은 상품기본이력임시 테이블과 해시 조인
3. 코드 상세 테이블 병렬 FULL SCAN 후 TQ10001 테이블큐를 통해 브로드캐스팅
4. (2)번에서 만들어진 조인 결과 데이터를 TQ10002 테이블큐를 통해 브로드캐스팅
5. 브로드캐스팅으로 받은 (3)번 데이터와 (4)번 데이터 해시조인
6. (5)번 데이터를 TQ10003 테이블큐를 통해 QC로 전송

### 2) 내가 생각한 튜닝 포인트🤔

**테이블큐를 통해 모든 데이터가 브로드캐스팅 방식으로 재분배 되는데 이는 굉장히 부하가 클 것으로 예상된다.(그러니깐 저런 오류가 났겄지;; 😱 )**

1. 상품기본이력임시 테이블 OUTER 테이블로 설정 및 병렬 FULL SCAN

* 👉 **FULL(A)**

1. 상품기본 테이블을 INNER 테이블로 설정 및 병렬 FULL SCAN 후 상품 기본의 가짓수가 어떻게 될지 모르기 때문에 브로드캐스팅말고 파티셔닝 조인을 한다. 이 때 ,INNER 테이블인 상품기본 테이블을 파티셔닝 한다.(실행계획에서 데이터가 훨씬 많으므로)
   * 👉 **FULL(B) pq\_distribute(b,NONE,PARTITION)**
2. 상품기본 테이블과 상품기본이력임시 테이블을 해시조인한다.
3. (3)번에서 만들어진 조인 결과 데이터를 OUTER 테이블로 설정
4. 코드상세 테이블을 INNER 테이블로 설정 및 병렬 FULL SCAN 후 브로드캐스팅한다.
   * 👉 **FULL(C) pq\_distribute(C,NONE,BRADCAST)**
5. (3)번 데이터와 (5)번 데이터를 해시조인한다.
   * 👉 코드 상세 테이블은 데이터가 적은 게 명확하므로 swap\_join\_inputs으로 hash bulid input을 명시해준다. **SWAP\_JOIN\_INPUTS(C)**
6. 상품상세 테이블을 INNER 테이블로 설정 및 병렬 FULL SCAN 후 (6)번 데이터와 조인하기 위해 해시 파티셔닝을 진행한다. 이 때 (6)번 데이터의 양이 현저히 적으므로 build input을 명시한다.
   * 👉 **FULL(D) pq\_distribute(D,HASH,HASH) NO\_SWAP\_JOIN\_INPUTS(D)**
7. 병렬도를 16으로 바꾼다.
   * 👉 **PARALLEL(A , 16) PARALLEL(B,16) PARALLEL(C,16) PARALLEL(D,16)**

### 2) 변경한 쿼리

**변경 전 쿼리**

```sql
SQL> INSERT /*+ APPEND */ INTO 상품기본이력(...)
2> SELECT /*+ PARALLEL(A , 32) PARALLEL(B,32) PARALLEL(C,32) PARALLEL(D,32)
PQ_DISTRIBUTE(B,NONE,BROADCAST)
*/ ....
3> FROM 상품기본이력임시 a, 상품기본 b, 코드상세 c, 상품상세 d
4> WHERE a. 상품번호= b. 상품번호
5> AND
6) /
```

**변경 후 쿼리**

```sql
SQL> INSERT /*+ APPEND */ INTO 상품기본이력(...)
2> SELECT /*+ PARALLEL(A , 16) PARALLEL(B,16) PARALLEL(C,16) PARALLEL(D,16)
FULL(A) FULL(B) FULL(C) FULL(D)
pq_distribute(b,NONE,PARTITION) pq_distribute(D,HASH,HASH)
SWAP_JOIN_INPUT(C) NO_SWAP_JOIN_INPUTS(D)
*/ ....
3> FROM 상품기본이력임시 a, 상품기본 b, 코드상세 c, 상품상세 d
4> WHERE a. 상품번호= b. 상품번호
5> AND
6) /
```
