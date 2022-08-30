문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-대용량배치프로그램튜닝53번

## 1) 내가 생각한 튜닝 포인트🤔
1. 만들고자 하는 쿼리의 순서는 다음과 같다.
   - 👉 (1)(Q1 , 00) 고객 C 병렬 FULL SCAN  => (Q1 , 01) 로 브로드 캐스팅
   - 👉 (2)(Q1 , 01) 주문 O 병렬 FULL SCAN + (1)에서 브로드 캐스팅으로 넘어온 고객 C 데이터와 해시 조인 => (Q1 , 03) 로 해시 분배
   - 👉 (3)(Q1 , 02) 배송 D 병렬 FULL SCAN  => (Q1 , 03) 로 해시 분배
   - 👉 (4)(Q1 , 03) (2)와 (3)에서 넘어온 데이터 해시 조인
       
## 2) 변경한 쿼리
#### 변경 전 쿼리
```sql
SQL > SELECT /*+ PARALLEL(4) */ *
	  FROM 고객 C, 주문 0, 배송 D
	  WHERE 0. 고객번호 = C.고객번호
	  AND D. 고객번호 = 0. 고객번호
	  AND D. 주문일자 = 0. 주문일자
	  AND D. 주문순번 = 0. 주문순번 ;
```
#### 변경 후 쿼리
```sql
-- outer 테이블(고객 C) broadcast
SQL > SELECT /*+ leading (c o d)
		full(c) full(o) full(d)
        parallel(c 4) parallel(o 4) parallel(d 4)
        use_hash(o) use_hash(d) no_swap_join_inputs
        pq_distribute(o broadcast none) 
        pq_distribute(d hash hash) */ *
	  FROM 고객 C, 주문 0, 배송 D
	  WHERE 0. 고객번호 = C.고객번호
	  AND D. 고객번호 = 0. 고객번호
	  AND D. 주문일자 = 0. 주문일자
	  AND D. 주문순번 = 0. 주문순번 ;
```
> ### ✅ 병렬 조인 종류 1 :  Full Partition Wise 조인 
   - 둘 다 같은 기준으로 파티셔닝된 경우이다.
   - 상호배타적 Partition-Pair 형성으로 각 서버 프로세스가 하나씩 독립적으로 조인을 수행한다.
   - Infra-operation parallelism
   -  다른 병렬 조인은 두 개의 서버집합이 필요한 반면, 여기서는 하나의 서버 집합만 필요하다.
   - Full Partition Wise 조인은 파티션 기반 Granule이므로 서버 프로세스 개수는 파티션 개수 이하로 제한한다.
   - 파티션 방식(Method)은 어떤 것이든 상관없다.
      - Range, 리스트, 해시 상관없이 두 테이블 조인 컬럼에 대해 같은 방식, 같은 기준으로 파티셔닝 돼 있다면 서로 방해받지 않고 Partition-Pair 끼리 독립적 병렬 조인이 가능하다.
   - 조인 방식도 어떤 것이든 선택 가능하다.
      - NL 조인, 소트 머지 조인, 해시 조인 등
   - 실행계획
      - (실행계획의 Hash 조인 위의 단계에서) "PX PARTITION RAGE ALL"  또는 "PX PARTITION RAGE INTERATOR" 가 표시된다. 
      ![](https://velog.velcdn.com/images/yooha9621/post/5bf4ef12-bbfa-44ec-b43b-dcc3279990f6/image.png)      
      
> ### ✅ 병렬 조인 종류 2 :  Partial Partition Wise 조인
   - 둘 중 하나만 파티셔닝된 경우이다.
   - 다른 한쪽 테이블을 같은 기준으로 동적으로 파티셔닝하고 나서 각 Partition-Pair를 독립적으로 병렬 조인한다.
   - 둘 다 파티셔닝이되었지만 파티션 기준이 서로 다른 경우에도 이 방식으로 조인이 가능하다.
   - 데이터를 동적으로 파티셔닝하기 위해 데이터 재분배가 선행되어야 한다.
      - 따라서 Inter-operation parallelism을 위해 두 개의 서버 집합이 필요하다.
   - 실행계획
      - 해시조인 아래쪽 두 테이블 중 어느 한쪽에 "PARTITION (KEY)" 또는 "PART (KEY)" 라고 표시되어 있으면, Partition Wise 조인이다.
      ![](https://velog.velcdn.com/images/yooha9621/post/a62909ec-b3c0-4d00-a279-526cce6fccad/image.png)

> ### ✅ 병렬 조인 종류 3 :  둘 다 파티셔닝되지 않은 경우
   - 동적 파티셔닝이 필요하다.
   - 조인 컬럼에 대해 어느 한쪽도 파티셔닝되지 않은 상황이면 아래 두개의 방식 중 하나를 사용한다.
	- 1) 양쪽 테이블을 동적으로 파티셔닝하고서 Full Partition Wise 조인 
	- 2) 한쪽 테이블을 Broadcast 한 뒤에 조인

병렬조인 종류 출처 : http://bysql.net/w201101/15525
   
> ### ✅ 병렬 조인에서 파티션 분배 방식
- #### PQ_DISTRIBUTE( Inner, none, none ) : 
    - Full-Partition Wise 조인으로 유도할 때 사용한다. 다연히, 양쪽 테이블 모두 조인 컬럼에 대해 같은 기준으로 파티셔닝( equi-partitioning ) 돼 있을 때만 작동한다.
- #### PQ_DISTRIBUTE( Inner, partition, none )
   - Partial-Partition Wise 조인으로 유도할 때 사용하며, outer 테이블을 inner 테이블 파티션기준에 따라 파티셔닝하라는 뜻이다.
당연히, inner 테이블이 조인 키 컬럼에 대해 파티셔닝 돼 있을 때만 작동한다.
- #### PQ_DISTRIBUTE( Inner, none, partition )
   - Partial-Partition Wise 조인으로 유도할 때 사용하며, inner 테이블을 outer 테이블 파티션 기준에 따라 파티셔닝 하라는 뜻이다.
- #### PQ_DISTRIBUTE( Inner, hash, hash )
   - 조인 키 컬럼을 해시 함수에 적용하고 거기서 반환된 값을 기준으로 양쪽 테이블을 동적으로 파티셔닝 하라는 뜻이다.
- #### PQ_DISTRIBUTE( inner, broadcast, none )
   - outer 테이블을 Boardcast 하라는 뜻이다.
- #### PQ_DISTRIBUTE( inner, none, broadcast )
   - inner 테이블을 Broadcast 하라는 뜻이다.