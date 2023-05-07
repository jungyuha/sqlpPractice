# 6. 문장수준 읽기 일관성

**기록 ✍️**

#### author : jung yuha

#### **first Registered : 2022-04-16**

#### last modified : **2022-04-16**

## 문장수준 읽기 일관성이란 ? <a href="#undefined" id="undefined"></a>

* 단일 SQL문이 수행되는 도중 다른 트랜잭션에 의해 데이터가 변경 되어도 일관성 있는 결과 집합을 리턴 하는 것

### 1. 다른 DBMS에서는 문장수준 읽기 일관성 구현을 어떻게 할까? <a href="#1-dbms" id="1-dbms"></a>

* 로우 Lock을 사용해 변경중인 또는 변경 후 커밋 전의 레코드의 접근을 막아 Dirty Read를 방지한다.
* 읽을 때 Shared Lock을 사용하며, Exclusive Lock 걸린 로우는 읽지 못한다.
* 로우 Lock으로 완벽한 문장수준 읽기 일관성은 보장되지 않는다.
  * 트랜잭션 고립화 수준을 상향 조정하여 테이블 레벨 Lock을 통해 읽기 일관성을 확보하면 동시성 저하 및 교착상태(Deadlock) 발생 가능성이 높아진다.

#### Level 2 (Read Commtted) 수준의 읽기 일관성 구현 <a href="#level-2-read-commtted" id="level-2-read-commtted"></a>

* 레코드를 읽는 순간에만 Shared Lock을 걸고 다음 레코드 이동시 Shared Lock을 해제한다.
* 기본적으로 사용되고있는 격리 수준이다.
* 다른 트랜잭션에 의해 Exclusive Lock 된 Row은 못 읽는다.
* 트랜잭션 진행중에 읽고 지나간 (Shared Lock 이 해제된) Row에 다른 트랜잭션이 변경할 수 있으며 이는 동일 트랜잭션 상에서 동일한 쿼리문에 대해 서로 다른 결과를 조회할 가능성을 초래해 완벽한 문장수준 일ㄹ끼 일관성을 보장하지 않는 점을 알 수 있다.
* 트랜잭션 내에서 같은 SELECT문을 실행하더라도 다른 트랜잭션의 commit 여부에 따라 다른 결과가 출력될 수 있다.

#### Level 3 (Repeatable Read) 수준의 읽기 일관성 구현 <a href="#level-3-repeatable-read" id="level-3-repeatable-read"></a>

* 쿼리가 읽은 레코드는 트랜잭션이 종료 될 때 까지 Shared Lock 이 유지된다.
* 교착 상태가 발생할 수 있다.
* 트랜잭션이 시작되기 전 commit된 결과를 참조한다.
  * 헷갈리지 말 것 !! '쿼리'가 시작되기 전이 아닌 '트랜잭션'이 시작되기 전이다!
  * 트랜잭션 내 같은 쿼리문은 트랜잭션이 시작된 시점의 데이터 스냅샷 정보를 조회하기 때문에 다른 트랜잭션의 commit 여부와 무관하게 항상 같은 결과가 출력된다.
* PHANTOM READ가 발생할 수 있다.
  * 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다가 안 보였다가 하는 현상이 나올 수 있다.

#### Level 4 (Serializable) Table-level Lock 수준의 읽기 일관성 구현 <a href="#level-4-serializable-table-level-lock" id="level-4-serializable-table-level-lock"></a>

* 동시성이 매우 저하된다.
*   아주 조심스럽게 써야 하는 락(Lock) 수준이다.

    > #### ✅ 트랙잭션의 격리성 수준 <a href="#undefined" id="undefined"></a>
    >
    > *
    >   1. READ UNCOMMITTED
    >   2. 변경 내용이 Commit이나 Rollback 여부에 상관 없이 다른 트랜잭션에서 조회할 수 있다.
    > *
    >   1. READ COMMITTED\
    >      \- 데이터를 변경했더라도 Commit이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있다.
    >   2. Oracle DBMS에서 기본적으로 사용되고 있는 격리 수준이다.
    > *
    >   1. REPEATABLE READ\
    >      \- 동일한 트랜잭션 내에서는 동일한 결과를 보여줄 수 있도록 보장한다.
    >   2. Read committed 경우에도 트랜잭션 시작 시점의 commit된 버전의 데이터를 보여준다.
    >   3. Undo 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있도록 보장한다.(Read committed은 commit 되기 전/후 모두의 데이터를 보여준다.)
    > *
    >   1. SERIALIZABLE\
    >      \- 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없다.

### 2. 오라클에서의 완벽한 문장수준 읽기 일관성 구현 <a href="#2" id="2"></a>

* Undo 세그먼트 내 Undo 데이터 활용하여 완벽한 문장수준 읽기 일관성을 보장한다.

## Consistent 모드 블록 읽기 <a href="#consistent" id="consistent"></a>

* 오라클은 _**쿼리**_** 가 시작된 시점 기준**으로 커밋된 데이터만 읽는다.
* 변경이 발생한 블록(SCN3)은 쿼리 블록(SCN2)에서 Current 블록(SCN3)으로부터 CR 블록(SCN2)을 생성해서 읽는다.
* Current 블록 SCN이 _**쿼리**_ SCN 보다 작다면 그냥 읽어도된다.

### 1. Current 블록과 CR 블록 <a href="#1-current-cr" id="1-current-cr"></a>

* 하나의 Current 블록은 여러개의 CR블록을 가진다.
* ( Current 블록 : CR 블록 ) = ( 1 : N )

#### Current 블록 <a href="#current" id="current"></a>

* 최종 상태의 원본 블록으로 한 개 존재한다.
* 특정 노드 Current 블록이 Exclusive 모드로 업그레이드되면 나머지 노드의 Current 블록은 Null 모드로 다운그레이드 된다.
  * RAC 환경에서 Current 블록이 **노드 숫자만큼** 생길 수(캐싱될 수) 있으나, Exclusive 모드의 Current 블록은 항상 한 개이다.
  * Null 모드로 다운그레이드 된 블록은 **다른 노드 혹은 디스크**에서 블록을 다시 읽어야 한다.

#### CR 블록 <a href="#cr" id="cr"></a>

* Current 블록의 복사본이다.
* 여러 버전이 존재한다.

### 2. Current/Consistent 모드 읽기 <a href="#2-currentconsistent" id="2-currentconsistent"></a>

#### Current 모드 <a href="#current" id="current"></a>

* 데이터를 찾아간 바로 그 시점에서의 최종 값을 읽는다.

#### Consistent 모드 <a href="#consistent" id="consistent"></a>

* 쿼리가 시작된 시점을 기준으로 값을 읽는다.
* 이 때 시점은 SCN(System Commit Number) 로 식별한다.
  * v$database.current\_scn\
    : 일관성과 동시성 구현 , Redo Log 정보 순서 식별, 데이터를 복구하는 데에 사용된다.
  * 블록 SCN과는 다른 개념이다.
    * 블록 SCN은 System Change Number의 약자로 블록이 마지막으로 변경된 시점의 정보를 나타낸다.
    * 블록 헤드에 있는 ITL 엔트리의 트랜잭션별 커밋 SCN 과 별개이다.

## Consistent 모드 블록 읽기의 세부원리 <a href="#consistent" id="consistent"></a>

\
[이미지출처](https://jungmina.com/781)

<figure><img src="https://velog.velcdn.com/images/yooha9621/post/906b044c-7def-4375-b8f5-d7d4e635abf3/image.png" alt=""><figcaption></figcaption></figure>

* 오라클에서 모든 쿼리는 SCN 값 확인 후 작업을 시작한다.
  1. 쿼리 SCN을 (스냅샷 SCN) 확보한 후
  2. 확보된 쿼리 SCN 과 읽을 블록의 SCN 비교 판단한다.

### 1) Current 블록 SCN <= 쿼리 SCN 이고, committed 상태 <a href="#1-current-scn-scn-committed" id="1-current-scn-scn-committed"></a>

* Consistent 모드 읽기는 블록 SCN(System Change Number)이 쿼리 SCN(System Comit Number) 보다 작거나 같은 블록만 읽을 수 있다.
* 쿼리가 시작된 이후 블록에 변경이 가해지지 않은 상태로 Current 블록을 바로 읽는다.

### 2) Current 블록 SCN > 쿼리 SCN 이고, committed 상태 <a href="#2-current-scn-scn-committed" id="2-current-scn-scn-committed"></a>

* 쿼리가 시작된 이후 블록이 변경된 후 커밋된 상태로 CR 블록 생성 후 읽어야한다.

#### CR 블록 생성 <a href="#cr" id="cr"></a>

* CR 블록을 읽을 수 있는 과거 버전(쿼리 SCN 보다 낮은 마지막 Committed 시점)으로 되돌려 원본(Current 블록)의 복사본(CR 블록)을 생성한다.
* 블록 헤더의 ITL 슬롯에서 UBA(Undo Block Address)가 가리키는 Undo 블록을 사용하여 읽을 수 있을 때 까지 반복적으로 ITL 슬롯의 UBA가 가리키는 Undo 블록 사용하여 되돌린다.
* 블록 SCN이 쿼리 SCN 보다 같거나 작으면서 커밋되지 않은 내용이 포함되지 않으면 CR 블록을 읽을 수 있다.

#### Snapshot too old <a href="#snapshot-too-old" id="snapshot-too-old"></a>

* 되돌릴 때 필요한 Undo 정보가 다른 트랜잭션에 의해 덮어 씌워지는 경우 생기는 현상이다.
* Delayed 블록 클린아웃 과정에서 관련 트랜잭션 테이블 슬롯이 재사용 되었다면 블록의 정확한 커밋 시점 확인이 불가하다.
* CR 블록 생성에 필요한 Undo 정보가 없는 경우 ORA-1555 에러가 발생한다.

#### CR 블록의 반복적 되돌림 데모 <a href="#cr" id="cr"></a>

* TX1 의 UPDATE 를 반복 수행 함에 따라, TX2 의 consistent gets 가 하나씩 계속 증가하는 경우
  *
    1. CR 블록 생성을 위한 UNDO 블록을 반복 방문한다.
  *
    1. CR블록 내 ITL 변경 내역도 Undo 레코드에 기록된다.
  *
    1. 되돌려질 때 ITL 내 UBA 도 같이 되돌려진다.
  *
    1. 따라서 반복적 되돌림에 UBA를 계속 사용할 수 있게 된다.

#### IMU(In-Memory Undo) <a href="#imuin-memory-undo" id="imuin-memory-undo"></a>

* Undo 데이터를 Undo 세그먼트 대신 Shared Pool 내 미리 할당된 IMU Pool(KTI-Undo)에 저장한다.
* 각 IMU Pool 은 하나의 트랜잭션 전용으로 할당되며 in memory undo latch 로 보호된다.
* IMU Pool이 가득 차면 Undo 데이터를 Undo 세그먼트로 일괄 기록(IMU Flush)후 이후 Undo 데이터는 예전처럼 Undo 세그먼트에 저장된다.
* 작은 트랜잭션을 위한 기능이다.
* Undo 세그먼트 헤더/일반 블록에 대한 경합 감소시킬 수 있다.
* 10g NF에서의 IMU(In-Memory Undo) 관련 파라미터
  * \_in\_memory\_undo : IMU 사용 여부로 (TRUE / FALSE)가 있다.
  * \_imu\_pools : IMU Pool 갯수이다.

### 3) Current 블록이 Active 상태, 즉 갱신이 진행 중인 상태 <a href="#3-current-active" id="3-current-active"></a>

* Delayed 블록 클린아웃 때문에 Active 상태 확인을 위해 Undo 세그먼트 헤더의 트랜잭션 테이블 접근 이 필요하다.
* 레코드 Lock Byte가 있고 ITL 내 커밋 정보 없다면 Active 상태로 추정한다.
* 트랜잭션 테이블로부터 커밋정보를 가져와 블록 클린아웃을 시도한다.
  * 쿼리 SCN 이전 커밋된 블록인 경우 그냥 읽는다.
  * 쿼리 SCN 이후 커밋 혹은 커밋전 블록인 경우 CR 블록 생성 후 읽는다.

### 4) DBA당 CR 개수 제한 <a href="#4-dba-cr" id="4-dba-cr"></a>

* 데이터 블록 한개 당 여섯개 까지 CR Copy 허용한다.
* 관련 파라미터명 : \_db\_block\_max\_cr\_dba
* CR Copy 는 LRU 리스트에서 LRU End 쪽에 위치한다.

#### 문장수준 읽기 일관성이란 ?

* 단일 SQL문이 수행되는 도중 다른 트랜잭션에 의해 데이터가 변경 되어도 일관성 있는 결과 집합을 리턴 하는 것

### 1. 다른 DBMS에서는 문장수준 읽기 일관성 구현을 어떻게 할까? <a href="#1-dbms" id="1-dbms"></a>

* 로우 Lock을 사용해 변경중인 또는 변경 후 커밋 전의 레코드의 접근을 막아 Dirty Read를 방지한다.
* 읽을 때 Shared Lock을 사용하며, Exclusive Lock 걸린 로우는 읽지 못한다.
* 로우 Lock으로 완벽한 문장수준 읽기 일관성은 보장되지 않는다.
  * 트랜잭션 고립화 수준을 상향 조정하여 테이블 레벨 Lock을 통해 읽기 일관성을 확보하면 동시성 저하 및 교착상태(Deadlock) 발생 가능성이 높아진다.

#### Level 2 (Read Commtted) 수준의 읽기 일관성 구현 <a href="#level-2-read-commtted" id="level-2-read-commtted"></a>

* 레코드를 읽는 순간에만 Shared Lock을 걸고 다음 레코드 이동시 Shared Lock을 해제한다.
* 기본적으로 사용되고있는 격리 수준이다.
* 다른 트랜잭션에 의해 Exclusive Lock 된 Row은 못 읽는다.
* 트랜잭션 진행중에 읽고 지나간 (Shared Lock 이 해제된) Row에 다른 트랜잭션이 변경할 수 있으며 이는 동일 트랜잭션 상에서 동일한 쿼리문에 대해 서로 다른 결과를 조회할 가능성을 초래해 완벽한 문장수준 일ㄹ끼 일관성을 보장하지 않는 점을 알 수 있다.
* 트랜잭션 내에서 같은 SELECT문을 실행하더라도 다른 트랜잭션의 commit 여부에 따라 다른 결과가 출력될 수 있다.

#### Level 3 (Repeatable Read) 수준의 읽기 일관성 구현 <a href="#level-3-repeatable-read" id="level-3-repeatable-read"></a>

* 쿼리가 읽은 레코드는 트랜잭션이 종료 될 때 까지 Shared Lock 이 유지된다.
* 교착 상태가 발생할 수 있다.
* 트랜잭션이 시작되기 전 commit된 결과를 참조한다.
  * 헷갈리지 말 것 !! '쿼리'가 시작되기 전이 아닌 '트랜잭션'이 시작되기 전이다!
  * 트랜잭션 내 같은 쿼리문은 트랜잭션이 시작된 시점의 데이터 스냅샷 정보를 조회하기 때문에 다른 트랜잭션의 commit 여부와 무관하게 항상 같은 결과가 출력된다.
* PHANTOM READ가 발생할 수 있다.
  * 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다가 안 보였다가 하는 현상이 나올 수 있다.

#### Level 4 (Serializable) Table-level Lock 수준의 읽기 일관성 구현 <a href="#level-4-serializable-table-level-lock" id="level-4-serializable-table-level-lock"></a>

* 동시성이 매우 저하된다.
*   아주 조심스럽게 써야 하는 락(Lock) 수준이다.

    > #### ✅ 트랙잭션의 격리성 수준 <a href="#undefined" id="undefined"></a>
    >
    > *
    >   1. READ UNCOMMITTED
    >   2. 변경 내용이 Commit이나 Rollback 여부에 상관 없이 다른 트랜잭션에서 조회할 수 있다.
    > *
    >   1. READ COMMITTED\
    >      \- 데이터를 변경했더라도 Commit이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있다.
    >   2. Oracle DBMS에서 기본적으로 사용되고 있는 격리 수준이다.
    > *
    >   1. REPEATABLE READ\
    >      \- 동일한 트랜잭션 내에서는 동일한 결과를 보여줄 수 있도록 보장한다.
    >   2. Read committed 경우에도 트랜잭션 시작 시점의 commit된 버전의 데이터를 보여준다.
    >   3. Undo 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있도록 보장한다.(Read committed은 commit 되기 전/후 모두의 데이터를 보여준다.)
    > *
    >   1. SERIALIZABLE\
    >      \- 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근할 수 없다.

### 2. 오라클에서의 완벽한 문장수준 읽기 일관성 구현 <a href="#2" id="2"></a>

* Undo 세그먼트 내 Undo 데이터 활용하여 완벽한 문장수준 읽기 일관성을 보장한다.

## Consistent 모드 블록 읽기 <a href="#consistent" id="consistent"></a>

* 오라클은 _**쿼리**_** 가 시작된 시점 기준**으로 커밋된 데이터만 읽는다.
* 변경이 발생한 블록(SCN3)은 쿼리 블록(SCN2)에서 Current 블록(SCN3)으로부터 CR 블록(SCN2)을 생성해서 읽는다.
* Current 블록 SCN이 _**쿼리**_ SCN 보다 작다면 그냥 읽어도된다.

### 1. Current 블록과 CR 블록 <a href="#1-current-cr" id="1-current-cr"></a>

* 하나의 Current 블록은 여러개의 CR블록을 가진다.
* ( Current 블록 : CR 블록 ) = ( 1 : N )

#### Current 블록 <a href="#current" id="current"></a>

* 최종 상태의 원본 블록으로 한 개 존재한다.
* 특정 노드 Current 블록이 Exclusive 모드로 업그레이드되면 나머지 노드의 Current 블록은 Null 모드로 다운그레이드 된다.
  * RAC 환경에서 Current 블록이 **노드 숫자만큼** 생길 수(캐싱될 수) 있으나, Exclusive 모드의 Current 블록은 항상 한 개이다.
  * Null 모드로 다운그레이드 된 블록은 **다른 노드 혹은 디스크**에서 블록을 다시 읽어야 한다.

#### CR 블록 <a href="#cr" id="cr"></a>

* Current 블록의 복사본이다.
* 여러 버전이 존재한다.

### 2. Current/Consistent 모드 읽기 <a href="#2-currentconsistent" id="2-currentconsistent"></a>

#### Current 모드 <a href="#current" id="current"></a>

* 데이터를 찾아간 바로 그 시점에서의 최종 값을 읽는다.

#### Consistent 모드 <a href="#consistent" id="consistent"></a>

* 쿼리가 시작된 시점을 기준으로 값을 읽는다.
* 이 때 시점은 SCN(System Commit Number) 로 식별한다.
  * v$database.current\_scn\
    : 일관성과 동시성 구현 , Redo Log 정보 순서 식별, 데이터를 복구하는 데에 사용된다.
  * 블록 SCN과는 다른 개념이다.
    * 블록 SCN은 System Change Number의 약자로 블록이 마지막으로 변경된 시점의 정보를 나타낸다.
    * 블록 헤드에 있는 ITL 엔트리의 트랜잭션별 커밋 SCN 과 별개이다.

## Consistent 모드 블록 읽기의 세부원리 <a href="#consistent" id="consistent"></a>

\
[이미지출처](https://jungmina.com/781)

<figure><img src="https://velog.velcdn.com/images/yooha9621/post/906b044c-7def-4375-b8f5-d7d4e635abf3/image.png" alt=""><figcaption></figcaption></figure>

* 오라클에서 모든 쿼리는 SCN 값 확인 후 작업을 시작한다.
  1. 쿼리 SCN을 (스냅샷 SCN) 확보한 후
  2. 확보된 쿼리 SCN 과 읽을 블록의 SCN 비교 판단한다.

### 1) Current 블록 SCN <= 쿼리 SCN 이고, committed 상태 <a href="#1-current-scn-scn-committed" id="1-current-scn-scn-committed"></a>

* Consistent 모드 읽기는 블록 SCN(System Change Number)이 쿼리 SCN(System Comit Number) 보다 작거나 같은 블록만 읽을 수 있다.
* 쿼리가 시작된 이후 블록에 변경이 가해지지 않은 상태로 Current 블록을 바로 읽는다.

### 2) Current 블록 SCN > 쿼리 SCN 이고, committed 상태 <a href="#2-current-scn-scn-committed" id="2-current-scn-scn-committed"></a>

* 쿼리가 시작된 이후 블록이 변경된 후 커밋된 상태로 CR 블록 생성 후 읽어야한다.

#### CR 블록 생성 <a href="#cr" id="cr"></a>

* CR 블록을 읽을 수 있는 과거 버전(쿼리 SCN 보다 낮은 마지막 Committed 시점)으로 되돌려 원본(Current 블록)의 복사본(CR 블록)을 생성한다.
* 블록 헤더의 ITL 슬롯에서 UBA(Undo Block Address)가 가리키는 Undo 블록을 사용하여 읽을 수 있을 때 까지 반복적으로 ITL 슬롯의 UBA가 가리키는 Undo 블록 사용하여 되돌린다.
* 블록 SCN이 쿼리 SCN 보다 같거나 작으면서 커밋되지 않은 내용이 포함되지 않으면 CR 블록을 읽을 수 있다.

#### Snapshot too old <a href="#snapshot-too-old" id="snapshot-too-old"></a>

* 되돌릴 때 필요한 Undo 정보가 다른 트랜잭션에 의해 덮어 씌워지는 경우 생기는 현상이다.
* Delayed 블록 클린아웃 과정에서 관련 트랜잭션 테이블 슬롯이 재사용 되었다면 블록의 정확한 커밋 시점 확인이 불가하다.
* CR 블록 생성에 필요한 Undo 정보가 없는 경우 ORA-1555 에러가 발생한다.

#### CR 블록의 반복적 되돌림 데모 <a href="#cr" id="cr"></a>

* TX1 의 UPDATE 를 반복 수행 함에 따라, TX2 의 consistent gets 가 하나씩 계속 증가하는 경우
  *
    1. CR 블록 생성을 위한 UNDO 블록을 반복 방문한다.
  *
    1. CR블록 내 ITL 변경 내역도 Undo 레코드에 기록된다.
  *
    1. 되돌려질 때 ITL 내 UBA 도 같이 되돌려진다.
  *
    1. 따라서 반복적 되돌림에 UBA를 계속 사용할 수 있게 된다.

#### IMU(In-Memory Undo) <a href="#imuin-memory-undo" id="imuin-memory-undo"></a>

* Undo 데이터를 Undo 세그먼트 대신 Shared Pool 내 미리 할당된 IMU Pool(KTI-Undo)에 저장한다.
* 각 IMU Pool 은 하나의 트랜잭션 전용으로 할당되며 in memory undo latch 로 보호된다.
* IMU Pool이 가득 차면 Undo 데이터를 Undo 세그먼트로 일괄 기록(IMU Flush)후 이후 Undo 데이터는 예전처럼 Undo 세그먼트에 저장된다.
* 작은 트랜잭션을 위한 기능이다.
* Undo 세그먼트 헤더/일반 블록에 대한 경합 감소시킬 수 있다.
* 10g NF에서의 IMU(In-Memory Undo) 관련 파라미터
  * \_in\_memory\_undo : IMU 사용 여부로 (TRUE / FALSE)가 있다.
  * \_imu\_pools : IMU Pool 갯수이다.

### 3) Current 블록이 Active 상태, 즉 갱신이 진행 중인 상태 <a href="#3-current-active" id="3-current-active"></a>

* Delayed 블록 클린아웃 때문에 Active 상태 확인을 위해 Undo 세그먼트 헤더의 트랜잭션 테이블 접근 이 필요하다.
* 레코드 Lock Byte가 있고 ITL 내 커밋 정보 없다면 Active 상태로 추정한다.
* 트랜잭션 테이블로부터 커밋정보를 가져와 블록 클린아웃을 시도한다.
  * 쿼리 SCN 이전 커밋된 블록인 경우 그냥 읽는다.
  * 쿼리 SCN 이후 커밋 혹은 커밋전 블록인 경우 CR 블록 생성 후 읽는다.

### 4) DBA당 CR 개수 제한 <a href="#4-dba-cr" id="4-dba-cr"></a>

* 데이터 블록 한개 당 여섯개 까지 CR Copy 허용한다.
* 관련 파라미터명 : \_db\_block\_max\_cr\_dba
* CR Copy 는 LRU 리스트에서 LRU End 쪽에 위치한다.

