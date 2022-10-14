# 5. Undo 세그먼트

**기록 ✍️**

#### author : jung yuha

#### **first Registered : 2022-04-15**

#### last modified : **2022-04-15**

## Undo 세그먼트 <a href="#undo" id="undo"></a>

\
[이미지출처](https://redkite777.tistory.com/entry/%EC%98%A4%EB%9D%BC%ED%81%B43-1%EB%8B%A8%EA%B3%84-Undo-%EC%84%B8%EA%B7%B8%EB%A8%BC%ED%8A%B8)

<figure><img src="https://velog.velcdn.com/images/yooha9621/post/c66094f6-2556-4f98-8041-cc0e65903f29/image.png" alt=""><figcaption></figcaption></figure>

*   트랜잭션이 발생시킨 테이블과 인덱스에 대한 변경사항들이 Undo 레코드 단위로 Undo 세그먼트(블록)에 기록된다.

    > **⭐️ Redo와 헷갈리지 말 것 !!**
    >
    > * Redo는 데이터베이스의 변경 사항을 로깅하는 부분
    > * Undo는 각 트랜잭션이 가한 변경 사항을 로깅하는 부분

## Undo 세그먼트 특징 <a href="#undo" id="undo"></a>

### 1. 일반 세그먼트와 동일하다. <a href="#1" id="1"></a>

* Extend 단위로 확장된다.
* 버퍼 캐시에 데이터를 캐싱한다.
* 변경사항을 Redo 로깅한다.

### 2. 트랜잭션 별로 Undo 세그먼트가 할당된다. <a href="#2-undo" id="2-undo"></a>

* 변경 사항이 Undo 레코드 단위로 기록된다.
* 복수 트랜잭션이 한 Undo 세그먼트를 공유할 수 있다.
  * (트랜잭션 : Undo 세그먼트) = (N : 1)

### 3. 오라클 버전에 따른 관리 <a href="#3" id="3"></a>

#### 1) \~8i 까지 <a href="#1-8i" id="1-8i"></a>

* Rollback 세그먼트가 수동 관리되었다.
* 세그먼트가 꽉차면 에러가 발생했다구 ..!🤭

#### 2) 9i 부터 \~ <a href="#2-9i" id="2-9i"></a>

**AUM(Automatic Undo Management) 도입**

> ✅ AUM(Automatic Undo Management)
>
> * 1 Undo 세그먼트, 1 트랜잭션 목표로 자동 관리한다.(1:1)
> * Undo 세그먼트 부족 시 가장 적게 사용되는 Undo 세그먼트를 할당한다.
> * Undo 세그먼트 확장 불가 시 다른 Undo 세그먼트로 부터 Free Undo Space를 회수한다. (Dynamic Extent Transfer)
> * Undo Tablespace 내 Free Undo Space 가 소진 되면 에러를 발생시킨다.

## Undo 목적 <a href="#undo" id="undo"></a>

### 1. Transaction Rollback <a href="#1-transaction-rollback" id="1-transaction-rollback"></a>

* 트랜잭션 Rollback 시 Undo 데이터를 사용한다.

### 2. Transaction Recovery <a href="#2-transaction-recovery" id="2-transaction-recovery"></a>

* 인스턴스 복구(Instance Recovery)시 Undo 데이터를 통해 트랜잭션을 Roll Forward 후 Commit 안된 트랜잭션을 Rollback 한다.

### 3. Read Consistency <a href="#3-read-consistency" id="3-read-consistency"></a>

* 읽기 일관성을 위해 Undo 데이터를 사용한다.
  * Oracle 이외의 다른 DBMS는 Lock 을 통해 읽기 일관성을 구현한다.

## Undo 세그먼트 구성 <a href="#undo" id="undo"></a>

\
[이미지출처](https://artdap.tistory.com/entry/%EC%96%B8%EB%91%90-%EB%9E%80)

<figure><img src="https://velog.velcdn.com/images/yooha9621/post/ce84c281-87c1-41e2-96b6-b8fddf86cfb2/image.gif" alt=""><figcaption></figcaption></figure>

### 1. Undo 헤더 (Initial Extent) <a href="#1-undo-initial-extent" id="1-undo-initial-extent"></a>

#### Transaction Table 슬롯의 구성 <a href="#transaction-table" id="transaction-table"></a>

**- 1. Transaction ID**

* USN+ Slot# + Wrap#
  * USN은 Undo Segment Number의 약자

**- 2. Transaction Status**

* 트랜잭션의 상태정보는 COMMITTED, ACTIVE가 있다.
* 트랜잭션 시작
  * 슬롯을 할당 받고, Status 를 ACTIVE 로 변경한다.
  * 슬롯을 못 얻으면 undo segment tx slot 대기이벤트가 발생한다.
* 트랜잭션 종료
  * Status를 COMMITTED 로 변경한다.
  *   Commit SCN을 저장한다.

      > **✅ Commit SCN**
      >
      > * System Change Number의 약자이다.
      > * Database의 commit된 version을 나타낸다.

**- 3. Last UBA**

* Last Undo Block Address의 약자이다.
* 마지막으로 추가한 undo 레코드의 위치를 가리킨다.
* Undo 레코드 체인을 유지하는 일종의 포인터이다.

### 2. Undo 레코드 <a href="#2-undo" id="2-undo"></a>

* 트랜잭션의 변경사항은 Undo 블록에 Undo 레코드로서 순차적으로 하나씩 기록된다.
* Last UBA(Undo Block Address) 정보로 마지막 Undo 레코드를 확인할 수 있다.
* Undo 레코드는 체인 형태로 연결 되며, Rollback 시 체인을 거슬러 올라가며 작업을 수행한다.
* v$transaction.used\_urec는 생성된 Undo 레코드의 갯수를 나타낸다.
* v$transaction.used\_ublk는 생성된 Undo 블록수를 나타낸다.

#### 1) INSERT시 생성되는 Undo 레코드의 개수 <a href="#1-insert-undo" id="1-insert-undo"></a>

* 추가된 레코드의 ROWID를 기록한다.
* 인덱스가 없는 테이블인 경우 v$transaction.used\_urec가 '1' 증가한다.
* 인덱스가 있는 테이블인 경우 v$transaction.used\_urec '2' 증가한다.

#### 2) UPDATE시 생성되는 Undo 레코드의 개수 <a href="#2-update-undo" id="2-update-undo"></a>

* 변경된 컬럼에 대한 Before image를 기록한다.
* 인덱스가 없는 테이블인 경우 v$transaction.used\_urec가 '1' 증가한다.
* 인덱스가 있는 테이블인 경우 v$transaction.used\_urec가 '3' 증가한다.
  * update시 인덱스에서는 내부적으로 Delete와 Insert 작업이 일어난다.

#### 3) DELETE시 생성되는 Undo 레코드의 개수 <a href="#3-delete-undo" id="3-delete-undo"></a>

* 삭제된 ROW의 모든 컬럼에 대한 Before image를 기록한다.
* 인덱스가 없는 테이블인 경우 v$transaction.used\_urec가 '1' 증가한다.
* 인덱스가 있는 테이블인 경우 v$transaction.used\_urec가 '2' 증가한다.

### 3. Undo 세그먼트 재사용 <a href="#3-undo" id="3-undo"></a>

* 커밋 된 순서대로 트랜잭션 슬롯이 순차적으로 재사용된다.
* 커밋 안된 Active 상태의 Undo 블록 및 트랜잭션 슬롯은 재사용할 수 없다.
* 트랜잭션이 커밋되어 상태 정보가 committed이 된 바로 그 시점의 커밋 SCN이 저장된 트랜잭션 슬롯이 재사용된다.

### 4. Undo Retention (undo\_retention) <a href="#4-undo-retention-undo_retention" id="4-undo-retention-undo_retention"></a>

* 완료된 트랜잭션의 Undo 데이터를 지정된 시간만큼 "가급적" 유지한다.
* unexpired와 expired 값을 기준으로 구분된다.
* Undo Extent 필요시 expired 상태의 Extent를 먼저 활용하며 필요시 unexpired 상태의 Extent 도 활용한다.
* guarantee 옵션을 사용하는 경우 unexpired 상태의 Extent는 활용할 수 없다.
  * 사용예시 :: alter tablespace undotbs1 retention guarantee;
* Automatic Undo Retention Tuning
  * 시스템 상황에 따라 tuned\_undo\_retention 값을 자동 계산한다.
  * undo\_retention이 최소값이 되며 가급적 지켜 지도록 Undo Extent를 관리한다.

## 블록 헤더 ITL 슬롯 (24 Byte) <a href="#itl-24-byte" id="itl-24-byte"></a>

![](https://velog.velcdn.com/images/yooha9621/post/456b42f9-8527-4b76-b6b4-96624c118173/image.png)\
[이미지출처](http://www.proligence.com/itl\_waits\_demystified.html)

* 테이블/인덱스의 블록 헤더에 위치한다.
  * UNDO 블록에 있는 게 아니라 블록 헤더에 위치하는 것!!)
* Interested Transaction List의 약자이다.

### 1. 블록 헤더 ITL 슬롯의 구성 <a href="#1-itl" id="1-itl"></a>

* ITL 슬롯 번호
* Transaction ID
* UBA : CR 블록을 생성할때 사용된다.
* Locking 정보
* 커밋 Flag
* 커밋 SCN (트랜잭션이 커밋 된 경우에 존재하는 값이다.)

### 2. 레코드 갱신시의 ITL 슬롯 <a href="#2-itl" id="2-itl"></a>

* 레코드 갱신시 블록 헤더의 ITL 슬롯을 확보 후 트랜잭션 ID 및 트랜잭션을 Active 상태로 먼저 기록해야한다.
* ITL 슬롯이 확보될 때 까지 트랜잭션은 대기이벤트와 함께 Blocking 된다.
  * 이 때 발생하는 대기 이벤트 : enq: TX - allocate ITL entry
* ITL 슬롯 부족을 극복하기 위해 INITTRANS, MAXTRANS, PCTFREE 파라미터를 활용한다.
  * 10g 부터 INITTRANS 는 최소2, MAXTRANS 는 255로 고정된다.
  * 블록에 여유공간(PCTFREE)이 없는경우엔 추가 슬롯 생성이 불가능하다.

## Lock Byte <a href="#lock-byte" id="lock-byte"></a>

* Row-level Lock , 즉 ! 로우 단위로 Lock Byte가 쓰인다.
* 레코드가 저장되는 로우 헤더에 Lock Byte를 할당해, 트랜잭션의 ITL 슬롯 번호를 기록해 둔다.
* 로우 Lock은 로우 단위 Lock과 트랜잭션 Lock(TX Lock)의 조합이다.

### 1. 레코드 갱신 순서 <a href="#1" id="1"></a>

*
  1. 레코드의 헤더의 Lock Byte를 확인한다.
*
  1. 해당 로우를 갱신 중인 트랜잭션 상태를 확인하고 액세스 가능 여부를 결정한다.
  2. 갱신 중인 트랜잭션 상태가 활성화 상태(TRUE)인 경우 다음 단계로 진행한다.
  3. 갱신 중인 트랜잭션 상태가 FALSE 인 경우 레코드를 갱신한다.
*
  1. 블록 헤더에 있는 ITL 슬롯의 트랜잭션ID를 확인한다.
*
  1. Transaction Table 슬롯에서 트랜잭션 상태를 확인한다.(블록 단위)
  2. Transaction Table 슬롯에서의 트랜잭션 상태가 Status COMMITTED 인 경우 레코드를 갱신한다.
  3. Transaction Table 슬롯에서의 트랜잭션 상태가 ACTIVE 인 경우 대기한다.

### 2. DBMS 별 레코드 정보 관리 <a href="#2-dbms" id="2-dbms"></a>

* 다른 DBMS 는 Lock 매니저로 갱신 중인 레코드 정보를 관리한다.
* 위와 같은 이유로 유한한 리소스 문제로인한 로우 → 블럭 → 테이블 레벨로 Lock 에스컬레이션이 발생할 수 있으며 이는 급격한 동시성 저하를 유발한다.
* 오라클은 레코드 속성을 보고 관리하기 때문에 별도의 리소스가 없다.
