# \[SQLP실기풀이]6장 고급SQL튜닝 (6) 고급 SQL 활용 59번

문제 링크 : https://velog.io/@yooha9621/SQLP실기문제-고급SQL활용59번

### 1) 기존 쿼리 분석

1. 1시간마다 수집되는 데이터의 비율은 0.1%인데 RSTLOG 쿼리에 조건절 컬럼이 인덱스에 없어 늘 FULL SCAN으로 데이터를 조회한다.
   * 하지만 인덱스를 만든다 가정할 때 인덱스를 10개를 만들어야하는데 이렇게 되면 DML 수행시 성능이 느려지게 된다.
2. 조건절만 다른 동일한 패턴의 쿼리가 너무 잦게 반복적으로 수행되어 DB 콜 횟수도 많고 DML 부하와 파싱 부하가 크다.

* 같은 범위(10만건)의 데이터를 반복 액세스한다.

1. RSTLOG가 지속적으로 쌓이고 있는 와중에 계속 ERRLOG에 INSERT하므로 데이터 정합성이 깨진다. 4.SELECT와 DELETE 사이에 입력된 로그값은 ERRLOG의 입력 대상이 되지 않는다.

### 2) 튜닝 포인트

1. INSERT-SELECT문을 ONE SQL로 구현하여 문장 수준의 일관성을 보장한다.(데이터 정합성 유지)

*   👉

    ```sql
      INSERT INTO ERRLOG
      SELECT TO CHAR(SYSDATE, 'YYYYMMDDHH24') CH_DTH
      , CASE WHEN ST01='E01' THEN 'E01'
        WHEN ST02='E02' THEN 'E02'
        WHEN ST03='E03' THEN 'E03'
        WHEN ST04='E04' THEN 'E04'
        WHEN ST05='E05' THEN 'E05'
        WHEN ST06='E06' THEN 'E06'
        WHEN ST07='E07' THEN 'E07'
        WHEN ST08='E08' THEN 'E08'
        WHEN ST09='E09' THEN 'E09'
        WHEN ST10='E10' THEN 'E10'
        END AS ERR_CD 
      , RNO
        FROM RSTLOG ;
    ```

````
2. 트랜잭션 수준 일관성은 봊장되지 않으므로 INSERT를 시작한 이후 ~ DELETE 까지 Serializable 수준으로 격리성을 상향 조정한다.

## 3) 쿼리1
```sql
SQL> 
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

INSERT INTO ERRLOG
    SELECT
    TO CHAR(SYSDATE, 'YYYYMMDDHH24') CH_DTH
    , CASE WHEN ST01='E01' THEN 'E01'
      WHEN ST02='E02' THEN 'E02'
      WHEN ST03='E03' THEN 'E03'
      WHEN ST04='E04' THEN 'E04'
      WHEN ST05='E05' THEN 'E05'
      WHEN ST06='E06' THEN 'E06'
      WHEN ST07='E07' THEN 'E07'
      WHEN ST08='E08' THEN 'E08'
      WHEN ST09='E09' THEN 'E09'
      WHEN ST10='E10' THEN 'E10'
      END AS ERR_CD 
    , RNO
      FROM RSTLOG 
      WHERE (
      	ST01 = 'A' OR
          ST02 = 'B' OR
          ST03 = 'C' OR
          ST04 = 'D' OR
          ST05 = 'E' OR
          ST06 = 'F' OR
          ST07 = 'G' OR
          ST08 = 'H' OR
          ST09 = 'I' OR
          ST10 = 'J'
      );
      
     DELETE FROM RSTLOG; 
````
