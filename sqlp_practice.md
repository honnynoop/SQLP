# SQLP 실기 기출 유형 문제 & 모범 답안
> 최신 출제 경향(2023~2025) 반영 | 실기 2문제 (각 15점, 총 30점)  
> 출처: 한국데이터산업진흥원 SQLP 실기 출제 유형 기반 재구성

---

## 📌 실기 시험 구조 이해

| 구분 | 내용 |
|------|------|
| 문항 수 | 2문제 (각 15점) |
| 총 배점 | 30점 |
| 합격 기준 | 필기 합산 60점 이상, 과목별 40% 이상 |
| 문제 유형 | SQL 튜닝형 / 성능 트러블슈팅형 |
| 답안 방식 | 힌트 포함 SQL 직접 작성 + 인덱스 설계 + 원인 분석 |

> ⚠️ **감점 주의사항**  
> - 선택(optional) 관계에서 INNER JOIN 사용 시 감점  
> - DATE 컬럼에 문자 리터럴 사용 시 감점  
> - 상충되는 힌트 동시 사용 시 감점 (예: USE_NL + FULL)  
> - 제약 조건 위반 SQL 작성 시 감점

---

## 🔴 실기 문제 1 — SQL 튜닝형 (ERD + 비효율 SQL 개선)

### 업무 설명

온라인 쇼핑몰 시스템에서 **특정 기간 내 주문된 상품의 카테고리별 매출 현황**을 조회한다.  
현재 배치 프로그램이 5분 이상 소요되어 장애 임박 상태이며, 튜닝이 필요하다.

---

### 테이블 구조 (DDL)

```sql
-- 고객 테이블
CREATE TABLE CUSTOMER (
    CUST_ID     VARCHAR2(10)  NOT NULL,  -- 고객ID (PK)
    CUST_NM     VARCHAR2(50)  NOT NULL,
    JOIN_DT     DATE          NOT NULL,
    GRADE       VARCHAR2(2)               -- 고객 등급 (NULL 허용)
);

-- 상품 테이블
CREATE TABLE PRODUCT (
    PROD_ID     VARCHAR2(10)  NOT NULL,  -- 상품ID (PK)
    PROD_NM     VARCHAR2(100) NOT NULL,
    CATEGORY    VARCHAR2(20)  NOT NULL,  -- 카테고리
    UNIT_PRICE  NUMBER(10)    NOT NULL
);

-- 주문 테이블
CREATE TABLE ORDERS (
    ORDER_ID    NUMBER(12)    NOT NULL,  -- 주문번호 (PK)
    CUST_ID     VARCHAR2(10)  NOT NULL,  -- FK → CUSTOMER
    ORDER_DT    DATE          NOT NULL,  -- 주문일시
    STATUS      VARCHAR2(2)   NOT NULL   -- 주문상태: '10'=정상, '99'=취소
);

-- 주문상품 테이블
CREATE TABLE ORDER_ITEM (
    ORDER_ID    NUMBER(12)    NOT NULL,  -- FK → ORDERS (PK 구성)
    ITEM_SEQ    NUMBER(4)     NOT NULL,  -- 순번 (PK 구성)
    PROD_ID     VARCHAR2(10)  NOT NULL,  -- FK → PRODUCT
    QTY         NUMBER(7)     NOT NULL,  -- 주문수량
    SALE_PRICE  NUMBER(10)    NOT NULL   -- 판매가
);
```

---

### 인덱스 현황

```sql
-- CUSTOMER
PK_CUSTOMER   : CUST_ID
IDX_CUST_01   : JOIN_DT

-- PRODUCT
PK_PRODUCT    : PROD_ID
IDX_PROD_01   : CATEGORY

-- ORDERS
PK_ORDERS     : ORDER_ID
IDX_ORD_01    : CUST_ID
IDX_ORD_02    : ORDER_DT + STATUS   ← 복합 인덱스

-- ORDER_ITEM
PK_ORDER_ITEM : ORDER_ID + ITEM_SEQ
IDX_OIT_01    : PROD_ID
```

---

### 데이터 볼륨

| 테이블 | 건수 | 비고 |
|--------|------|------|
| CUSTOMER | 300만 건 | |
| PRODUCT | 50만 건 | |
| ORDERS | 5,000만 건 | 월 평균 200만 건 적재 |
| ORDER_ITEM | 1억 5,000만 건 | 주문당 평균 3건 |

---

### 현재 SQL (비효율, 튜닝 전)

```sql
-- [현재 SQL] 2024년 1월 한 달간 카테고리별 매출 합계 및 주문 건수 조회
SELECT P.CATEGORY,
       COUNT(DISTINCT O.ORDER_ID)  AS ORDER_CNT,
       SUM(OI.QTY * OI.SALE_PRICE) AS TOTAL_SALES
FROM   CUSTOMER C,
       ORDERS O,
       ORDER_ITEM OI,
       PRODUCT P
WHERE  C.CUST_ID    = O.CUST_ID
AND    O.ORDER_ID   = OI.ORDER_ID
AND    OI.PROD_ID   = P.PROD_ID
AND    TO_CHAR(O.ORDER_DT, 'YYYYMM') = '202401'
AND    O.STATUS     = '10'
AND    C.GRADE      IS NOT NULL
GROUP BY P.CATEGORY
ORDER BY TOTAL_SALES DESC;
```

---

### 현재 실행계획 (Execution Plan)

```
--------------------------------------------------------------------
| Id | Operation                     | Name         | Rows  | Cost |
--------------------------------------------------------------------
|  0 | SELECT STATEMENT              |              |       | 9832K|
|  1 |  SORT ORDER BY                |              |   28  | 9832K|
|  2 |   HASH GROUP BY               |              |   28  | 9832K|
|  3 |    HASH JOIN                  |              |  180K | 9820K|
|  4 |     TABLE ACCESS FULL         | PRODUCT      |  500K | 1240 |
|  5 |     HASH JOIN                 |              |  180K | 9810K|
|  6 |      HASH JOIN                |              |   95K | 5020K|
|  7 |       TABLE ACCESS FULL       | CUSTOMER     |  300K | 8500 |← ①
|  8 |       TABLE ACCESS FULL       | ORDERS       |  5000M| 4990K|← ②
|  9 |           FILTER              |              |       |      |
| 10 |            TABLE ACCESS FULL  | ORDER_ITEM   | 1500M | 3800K|← ③
--------------------------------------------------------------------
```

---

### 문제

**(1)** 위 실행계획의 문제점을 **4가지** 이상 기술하시오.

**(2)** 튜닝된 SQL을 **힌트를 포함하여** 작성하시오. (인덱스 추가 없이 기존 인덱스 활용)

**(3)** 추가 인덱스를 설계하여 성능을 더욱 개선하고, 해당 인덱스 DDL과 개선 이유를 기술하시오.

---

## ✅ 모범 답안 — 문제 1

### (1) 실행계획 문제점 분석

#### ① CUSTOMER 테이블 FULL SCAN (불필요한 조인)
- `C.GRADE IS NOT NULL` 조건만으로 CUSTOMER(300만 건) 전체를 스캔하고 있음
- CUSTOMER와의 조인은 GRADE 필터 외 어떤 컬럼도 SELECT절에 사용되지 않음
- **핵심 문제**: 이 조인은 **세미조인(EXISTS) 혹은 조인 제거**로 대체 가능하며, 불필요하게 300만 건의 대용량 테이블을 조인에 포함시키고 있음

#### ② ORDERS 테이블 FULL SCAN + INDEX 미활용
- `TO_CHAR(O.ORDER_DT, 'YYYYMM') = '202401'` 조건에서 **컬럼에 함수 적용** 으로 인해 `IDX_ORD_02(ORDER_DT + STATUS)` 인덱스를 사용하지 못하고 FULL SCAN 발생
- ORDERS 5,000만 건 전체 스캔

#### ③ ORDER_ITEM 테이블 FULL SCAN
- ORDERS와 조인 전에 ORDERS 필터링이 이루어지지 않아 ORDER_ITEM 1억 5,000만 건 전체 스캔
- 선행 테이블(ORDERS)에서 먼저 필터링 후 NL 조인으로 접근했어야 함

#### ④ 조인 순서 비효율
- 가장 선택도가 낮은(결과 건수가 적은) ORDERS의 날짜+상태 필터를 먼저 적용해야 하는데, CUSTOMER → ORDERS → ORDER_ITEM 순으로 대용량 테이블을 먼저 조인
- 드라이빙 테이블이 잘못 선택됨

#### ⑤ COUNT(DISTINCT) 사용으로 인한 SORT 부하
- ORDER_ITEM 단위로 조인 후 GROUP BY + COUNT(DISTINCT ORDER_ID) 수행
- ORDERS 먼저 집계(GROUP BY ORDER_ID) 후 조인하는 방식이 더 효율적

---

### (2) 튜닝된 SQL (기존 인덱스 활용)

```sql
/*
 * 튜닝 포인트:
 * 1. ORDER_DT 범위 조건으로 변경 → IDX_ORD_02(ORDER_DT+STATUS) 인덱스 활용
 * 2. CUSTOMER 조인을 EXISTS 서브쿼리로 변경 (불필요한 대용량 조인 제거)
 * 3. ORDERS → ORDER_ITEM NL 조인 유도 (선행 필터링 후 접근)
 * 4. ORDERS 단위로 먼저 집계 후 PRODUCT 조인 (COUNT DISTINCT 제거)
 */
SELECT /*+ LEADING(O) USE_NL(OI) USE_HASH(P) INDEX(O IDX_ORD_02) */
       P.CATEGORY,
       COUNT(O.ORDER_ID)            AS ORDER_CNT,
       SUM(O.ITEM_SALES)            AS TOTAL_SALES
FROM (
    SELECT /*+ INDEX(O IDX_ORD_02) */
           OI.PROD_ID,
           O.ORDER_ID,
           SUM(OI.QTY * OI.SALE_PRICE) AS ITEM_SALES
    FROM   ORDERS O,
           ORDER_ITEM OI
    WHERE  O.ORDER_DT >= TO_DATE('20240101', 'YYYYMMDD')
    AND    O.ORDER_DT <  TO_DATE('20240201', 'YYYYMMDD')   -- ★ 범위 조건으로 변경
    AND    O.STATUS   = '10'                                -- ★ IDX_ORD_02 복합 인덱스 활용
    AND    OI.ORDER_ID = O.ORDER_ID                        -- ★ NL 조인: ORDERS 먼저 필터링
    AND    EXISTS (                                         -- ★ CUSTOMER 조인 → EXISTS로 변경
               SELECT /*+ NO_UNNEST */ 1
               FROM   CUSTOMER C
               WHERE  C.CUST_ID = O.CUST_ID
               AND    C.GRADE IS NOT NULL
           )
    GROUP BY OI.PROD_ID, O.ORDER_ID
) O,
PRODUCT P
WHERE O.PROD_ID = P.PROD_ID
GROUP BY P.CATEGORY
ORDER BY TOTAL_SALES DESC;
```

---

### (3) 추가 인덱스 설계 및 개선 이유

#### 인덱스 1: ORDERS 테이블 인덱스 재설계

```sql
-- 기존: IDX_ORD_02 : ORDER_DT + STATUS
-- 개선: STATUS를 선두로 바꾸거나, 커버링 인덱스 구성

CREATE INDEX IDX_ORD_03 ON ORDERS (ORDER_DT, STATUS, CUST_ID, ORDER_ID);
```

**개선 이유:**  
- `ORDER_DT` 범위 + `STATUS = '10'` 조건 처리 후 `CUST_ID`, `ORDER_ID`를 인덱스에서 바로 추출 가능 (테이블 접근 최소화)  
- `CUST_ID`가 인덱스에 포함되어 EXISTS 서브쿼리의 CUSTOMER 조인 시 인덱스만으로 조인 가능

#### 인덱스 2: ORDER_ITEM 테이블 인덱스 재설계

```sql
CREATE INDEX IDX_OIT_02 ON ORDER_ITEM (ORDER_ID, PROD_ID, QTY, SALE_PRICE);
```

**개선 이유:**  
- `ORDER_ID`로 NL 조인 시 인덱스 스캔만으로 `PROD_ID`, `QTY`, `SALE_PRICE` 컬럼까지 조회 가능 → **커버링 인덱스(Index Only Scan)** 달성  
- 테이블 랜덤 액세스(TABLE ACCESS BY INDEX ROWID) 제거로 I/O 대폭 감소

#### 인덱스 3: CUSTOMER 선택적 인덱스 (GRADE 조건 최적화)

```sql
CREATE INDEX IDX_CUST_02 ON CUSTOMER (CUST_ID, GRADE);
```

**개선 이유:**  
- EXISTS 서브쿼리에서 `CUST_ID` + `GRADE IS NOT NULL` 조건을 인덱스만으로 처리  
- NULL 값이 많은 경우 인덱스 스캔 범위가 줄어 효율 증대

---

---

## 🔴 실기 문제 2 — 성능 트러블슈팅형 (실행계획 분석 + 개선안)

### 업무 설명

은행 계좌 이체 서비스에서 **특정 고객의 최근 6개월 거래 이력과 잔액 현황**을 조회하는 화면이 있다.  
특정 시간대(오전 9~10시)에 **응답 시간이 30초 이상** 걸리는 장애가 발생하며, 동일 SQL이 다른 시간대엔 0.5초 이내로 응답한다.

---

### 테이블 구조

```sql
CREATE TABLE ACCOUNT (
    ACCT_NO     VARCHAR2(15)  NOT NULL,  -- 계좌번호 (PK)
    CUST_NO     VARCHAR2(10)  NOT NULL,  -- 고객번호
    PROD_CD     VARCHAR2(5)   NOT NULL,  -- 상품코드
    OPEN_DT     DATE          NOT NULL,  -- 개설일
    CLOSE_DT    DATE,                    -- 해지일 (NULL = 유효계좌)
    BAL_AMT     NUMBER(15,2)  NOT NULL   -- 잔액
);

CREATE TABLE TRANS (
    ACCT_NO     VARCHAR2(15)  NOT NULL,  -- 계좌번호 (FK)
    TRANS_DT    DATE          NOT NULL,  -- 거래일시
    TRANS_SEQ   NUMBER(8)     NOT NULL,  -- 거래순번
    TRANS_CD    VARCHAR2(3)   NOT NULL,  -- 거래코드
    TRANS_AMT   NUMBER(15,2)  NOT NULL,  -- 거래금액
    BALANCE     NUMBER(15,2)  NOT NULL   -- 거래 후 잔액
);
```

---

### 인덱스 현황

```sql
PK_ACCOUNT    : ACCT_NO
IDX_ACCT_01   : CUST_NO, PROD_CD
IDX_ACCT_02   : CUST_NO, CLOSE_DT

PK_TRANS      : ACCT_NO + TRANS_DT + TRANS_SEQ
IDX_TRANS_01  : TRANS_DT, ACCT_NO     ← 순서 주의
```

---

### 문제가 되는 SQL

```sql
-- [화면 SQL] 고객번호 :v_cust_no의 유효계좌별 최근 6개월 거래 이력 조회
SELECT A.ACCT_NO,
       A.BAL_AMT,
       T.TRANS_DT,
       T.TRANS_CD,
       T.TRANS_AMT,
       T.BALANCE
FROM   ACCOUNT A,
       TRANS   T
WHERE  A.CUST_NO  = :v_cust_no
AND    A.CLOSE_DT IS NULL
AND    T.ACCT_NO  = A.ACCT_NO
AND    T.TRANS_DT >= ADD_MONTHS(SYSDATE, -6)
ORDER BY A.ACCT_NO, T.TRANS_DT DESC, T.TRANS_SEQ DESC;
```

---

### 현재 실행계획 (Execution Plan)

```
----------------------------------------------------------------------
| Id | Operation                      | Name          | Rows | Cost |
----------------------------------------------------------------------
|  0 | SELECT STATEMENT               |               |      | 89K  |
|  1 |  SORT ORDER BY                 |               | 3.2K | 89K  |
|  2 |   NESTED LOOPS                 |               | 3.2K | 89K  |
|  3 |    NESTED LOOPS                |               |  10  |  55  |
|  4 |     TABLE ACCESS BY INDEX ROWID| ACCOUNT       |  10  |  25  |
|  5 |      INDEX RANGE SCAN          | IDX_ACCT_02   |  10  |   5  |
|  6 |     INDEX RANGE SCAN           | IDX_TRANS_01  | 320  |   3  |← ①
|  7 |    TABLE ACCESS BY INDEX ROWID | TRANS         | 320  | 8890 |← ②
----------------------------------------------------------------------
```

---

### AWR / 트레이스 추가 정보

```
-- 오전 9~10시 장애 발생 시 통계
Consistent Gets  : 4,200,000  (평소: 3,200)
Physical Reads   : 1,800,000  (평소: 0)
Buffer Busy Wait : 38초
Elapsed Time     : 42초

-- TRANS 테이블 통계 정보
총 건수          : 12억 건
TRANS_DT 분포    : 1일 평균 500만 건 적재
최근 6개월 건수  : 약 9억 건
```

---

### 문제

**(1)** 장애의 근본 원인을 **실행계획과 AWR 정보를 근거로** 설명하시오.

**(2)** `IDX_TRANS_01` 인덱스의 컬럼 순서 문제를 설명하고, 올바른 인덱스 구성을 제시하시오.

**(3)** SQL과 인덱스를 개선하여 최적의 성능을 낼 수 있는 최종 튜닝 방안을 기술하시오.

---

## ✅ 모범 답안 — 문제 2

### (1) 장애 근본 원인 분석

#### 원인 1: 인덱스 컬럼 순서 오류로 인한 과도한 랜덤 I/O

`IDX_TRANS_01`은 `(TRANS_DT, ACCT_NO)` 순서로 구성되어 있다.

SQL 조건은 `T.ACCT_NO = A.ACCT_NO` AND `T.TRANS_DT >= ADD_MONTHS(SYSDATE, -6)` 이다.

NL 조인의 이너 루프에서 ACCOUNT의 ACCT_NO로 TRANS를 조회할 때, 인덱스 선두 컬럼이 `TRANS_DT`이므로 **ACCT_NO 조건을 인덱스의 선두 필터로 사용하지 못한다.** 이로 인해 `TRANS_DT >= 6개월전` 범위 전체(최근 6개월 ≈ 9억 건)를 스캔한 후 ACCT_NO 조건으로 필터링하게 되어 과도한 I/O 발생.

#### 원인 2: 오전 9~10시 집중으로 인한 Buffer Busy Wait

- 오전 9~10시는 영업 시작 직후로 동시 접속 폭증 시간대
- 한 고객이 여러 계좌를 가질 경우, 동일 TRANS 블록을 다수의 세션이 동시에 참조/수정
- `Consistent Gets: 4,200,000` vs 평소 `3,200`은 약 **1,300배 차이**
- `Buffer Busy Wait: 38초` → 동일 버퍼 블록을 여러 세션이 경쟁

#### 원인 3: SORT ORDER BY 부하

최근 6개월 거래(고객당 수천~수만 건) 전체를 가져온 후 정렬하여 추가 메모리/디스크 SORT 발생

---

### (2) IDX_TRANS_01 인덱스 컬럼 순서 문제 및 재설계

#### 현재 인덱스의 문제

```
현재: IDX_TRANS_01 = (TRANS_DT, ACCT_NO)
```

- NL 조인에서 `ACCT_NO`가 드라이브 조건(= 조건)임에도 선두 컬럼이 아님
- `TRANS_DT >= 범위조건`이 선두이므로 ACCT_NO는 **인덱스 필터** 조건으로만 작동
- 결과적으로 최근 6개월치 전체 TRANS 레코드를 스캔

#### 개선 인덱스

```sql
-- 기존 인덱스 삭제
DROP INDEX IDX_TRANS_01;

-- 신규 인덱스: ACCT_NO 선두, TRANS_DT 후위
CREATE INDEX IDX_TRANS_02 ON TRANS (ACCT_NO, TRANS_DT DESC, TRANS_SEQ DESC);
```

**인덱스 컬럼 선정 이유:**

| 컬럼 | 위치 | 이유 |
|------|------|------|
| ACCT_NO | 1번 (선두) | NL 이너 루프의 `=` 조건 → 인덱스 액세스 조건 |
| TRANS_DT DESC | 2번 | 범위 조건 `>=` 처리 + ORDER BY 방향 일치로 SORT 제거 |
| TRANS_SEQ DESC | 3번 | ORDER BY 완전 일치 → SORT ORDER BY 오퍼레이션 제거 |

---

### (3) 최종 튜닝 방안

#### 튜닝 SQL

```sql
SELECT /*+ LEADING(A) USE_NL(T) INDEX(A IDX_ACCT_02) INDEX(T IDX_TRANS_02) */
       A.ACCT_NO,
       A.BAL_AMT,
       T.TRANS_DT,
       T.TRANS_CD,
       T.TRANS_AMT,
       T.BALANCE
FROM   ACCOUNT A,
       TRANS   T
WHERE  A.CUST_NO  = :v_cust_no          -- ★ 바인드 변수 유지 (소프트 파싱 활용)
AND    A.CLOSE_DT IS NULL               -- ★ IDX_ACCT_02(CUST_NO, CLOSE_DT) 활용
AND    T.ACCT_NO  = A.ACCT_NO           -- ★ NL 조인: A가 드라이버
AND    T.TRANS_DT >= ADD_MONTHS(TRUNC(SYSDATE, 'DD'), -6)  -- ★ TRUNC로 시분초 제거
ORDER BY A.ACCT_NO, T.TRANS_DT DESC, T.TRANS_SEQ DESC;
-- ★ IDX_TRANS_02(ACCT_NO, TRANS_DT DESC, TRANS_SEQ DESC) → SORT 제거
```

#### 예상 실행계획 (튜닝 후)

```
----------------------------------------------------------------------
| Id | Operation                      | Name          | Rows | Cost |
----------------------------------------------------------------------
|  0 | SELECT STATEMENT               |               |      |  320 |
|  1 |   NESTED LOOPS                 |               | 3.2K |  320 |
|  2 |    TABLE ACCESS BY INDEX ROWID | ACCOUNT       |   10 |   25 |
|  3 |     INDEX RANGE SCAN           | IDX_ACCT_02   |   10 |    5 |
|  4 |    TABLE ACCESS BY INDEX ROWID | TRANS         |  320 |  295 |← 대폭 감소
|  5 |     INDEX RANGE SCAN           | IDX_TRANS_02  |  320 |   20 |← 선두=ACCT_NO
----------------------------------------------------------------------
-- SORT ORDER BY 오퍼레이션 제거 (인덱스 순서와 ORDER BY 일치)
```

#### 추가 튜닝 방안 (페이징 처리)

화면 조회이므로 전체 데이터를 불러올 필요가 없다. 로우넘 페이징을 적용한다:

```sql
SELECT *
FROM (
    SELECT /*+ LEADING(A) USE_NL(T) INDEX(A IDX_ACCT_02) INDEX(T IDX_TRANS_02) */
           A.ACCT_NO, A.BAL_AMT,
           T.TRANS_DT, T.TRANS_CD, T.TRANS_AMT, T.BALANCE,
           ROWNUM AS RN
    FROM   ACCOUNT A, TRANS T
    WHERE  A.CUST_NO  = :v_cust_no
    AND    A.CLOSE_DT IS NULL
    AND    T.ACCT_NO  = A.ACCT_NO
    AND    T.TRANS_DT >= ADD_MONTHS(TRUNC(SYSDATE, 'DD'), -6)
    AND    ROWNUM     <= :end_row      -- ★ ROWNUM 조건으로 조기 종료 유도
    ORDER BY A.ACCT_NO, T.TRANS_DT DESC, T.TRANS_SEQ DESC
)
WHERE RN >= :start_row;
```

#### Buffer Busy Wait 해결 방안

| 방법 | 설명 |
|------|------|
| 인덱스 재설계 | 랜덤 I/O 감소 → 버퍼 경합 자체 감소 |
| 커서 공유 (바인드 변수) | 하드 파싱 방지 → Library Cache 경합 감소 |
| PARTITIONING | TRANS 테이블을 TRANS_DT 기준 RANGE PARTITION 적용 → 파티션 프루닝으로 I/O 감소 |

```sql
-- 파티션 테이블 예시 (TRANS 테이블 재설계)
CREATE TABLE TRANS (
    ACCT_NO     VARCHAR2(15) NOT NULL,
    TRANS_DT    DATE         NOT NULL,
    TRANS_SEQ   NUMBER(8)    NOT NULL,
    TRANS_CD    VARCHAR2(3)  NOT NULL,
    TRANS_AMT   NUMBER(15,2) NOT NULL,
    BALANCE     NUMBER(15,2) NOT NULL
)
PARTITION BY RANGE (TRANS_DT) INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION P_INIT VALUES LESS THAN (DATE '2020-01-01')
);
-- → 월별 파티션 자동 생성, 파티션 프루닝으로 최근 6개월 파티션만 스캔
```

---

## 📋 핵심 튜닝 포인트 요약

### 인덱스 설계 원칙

| 순서 | 원칙 | 이유 |
|------|------|------|
| 1 | `=` 조건 컬럼을 선두에 | 인덱스 액세스 조건 최대화 |
| 2 | 범위 조건 (`BETWEEN`, `>=`) 컬럼을 후위에 | 범위 스캔 최소화 |
| 3 | ORDER BY 컬럼을 인덱스에 포함 | SORT 오퍼레이션 제거 |
| 4 | SELECT 컬럼을 인덱스에 포함 (선택적) | 커버링 인덱스로 테이블 접근 제거 |

### SQL 튜닝 체크리스트

- [ ] 인덱스 컬럼에 함수/형변환 적용 여부 (`TO_CHAR(DATE컬럼)` 등)
- [ ] 불필요한 조인 존재 여부 (EXISTS/IN으로 대체 가능한지)
- [ ] NL 조인 드라이버 테이블의 선택도(선별성) 확인
- [ ] 인덱스 컬럼 순서와 조인/조건 순서 일치 여부
- [ ] SORT 오퍼레이션 제거 가능 여부 (인덱스 ORDER 활용)
- [ ] 페이징 처리로 조기 종료 가능 여부
- [ ] 선택(optional) 관계에서 OUTER JOIN 사용 여부

---

*[참고] 본 문제는 한국데이터산업진흥원 SQLP 실기 출제 기준(SQL 튜닝형, 성능 트러블슈팅형)을 기반으로 재구성하였으며, 실제 시험에서는 보다 복잡한 ERD와 실행계획이 제공됩니다.*
