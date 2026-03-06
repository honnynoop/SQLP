# 📚 DBMS 인덱스(Index) 완벽 정리

> 데이터베이스 성능 최적화의 핵심 개념인 인덱스의 종류, 작동 원리, 활용법, 주의사항을 정리한 참고 문서

---

## 1. 인덱스(Index)란?

인덱스는 테이블의 특정 컬럼에 대해 **검색 속도를 높이기 위한 별도의 자료구조**이다.  
책의 "목차"나 "색인"과 같은 역할로, 전체 데이터를 스캔(Full Table Scan)하지 않고 원하는 데이터를 빠르게 찾을 수 있도록 돕는다.

```
[ 테이블 ]                    [ 인덱스 ]
┌────┬────────┬──────┐        ┌────────┬──────────┐
│ ID │  Name  │ Age  │  →→→   │  Name  │ Row Ptr  │
├────┼────────┼──────┤        ├────────┼──────────┤
│ 1  │ Charlie│  30  │        │  Alice │  → Row 3 │
│ 2  │  Bob   │  25  │        │  Bob   │  → Row 2 │
│ 3  │  Alice │  28  │        │ Charlie│  → Row 1 │
└────┴────────┴──────┘        └────────┴──────────┘
```

---

## 2. 인덱스의 종류

### 2-1. 구조(자료구조)에 따른 분류

| 종류 | 자료구조 | 지원 DB | 특징 | 적합한 쿼리 |
|------|----------|---------|------|------------|
| **B-Tree 인덱스** | 균형 이진 트리 (Balanced Tree) | MySQL, PostgreSQL, Oracle, MSSQL | 가장 일반적, 범위 검색에 강함 | `=`, `<`, `>`, `BETWEEN`, `LIKE 'abc%'` |
| **Hash 인덱스** | 해시 테이블 | MySQL(Memory), PostgreSQL | 동등 비교에 최적, 범위 검색 불가 | `=` (완전 일치만) |
| **Bitmap 인덱스** | 비트맵 배열 | Oracle, PostgreSQL | 카디널리티 낮은 컬럼에 효율적 | `AND`, `OR`, `NOT` 복합 조건 |
| **Full-Text 인덱스** | 역인덱스 (Inverted Index) | MySQL, PostgreSQL, Elasticsearch | 텍스트 전문 검색 특화 | `MATCH ... AGAINST`, `LIKE '%word%'` |
| **R-Tree 인덱스** | 공간 트리 | MySQL, PostgreSQL (PostGIS) | 공간/지리 데이터 전용 | 좌표 범위, 거리 계산 |
| **GiST / GIN 인덱스** | 범용 검색 트리 / 역인덱스 | PostgreSQL | JSON, 배열, 전문 검색 등 다양한 타입 | JSON 키 검색, 배열 포함 여부 |

---

### 2-2. 논리적 특성에 따른 분류

| 종류 | 설명 | 특징 | 예시 |
|------|------|------|------|
| **클러스터드 인덱스** (Clustered Index) | 테이블 데이터 자체를 인덱스 키 순서로 물리적 정렬 저장 | 테이블당 1개만 가능, 범위 검색 매우 빠름, PK에 자동 생성 | MySQL InnoDB의 PK |
| **논클러스터드 인덱스** (Non-Clustered Index) | 별도 인덱스 구조에 키와 행 포인터 저장 | 테이블당 여러 개 가능, 추가 조회 필요 | 일반 `CREATE INDEX` |
| **유니크 인덱스** (Unique Index) | 인덱스 컬럼의 중복 값을 허용하지 않음 | 데이터 무결성 + 검색 성능 동시 확보 | `UNIQUE` 제약조건 |
| **복합 인덱스** (Composite Index) | 2개 이상의 컬럼을 조합하여 생성 | 컬럼 순서가 중요 (선두 컬럼 원칙) | `INDEX(last_name, first_name)` |
| **커버링 인덱스** (Covering Index) | 쿼리에 필요한 모든 컬럼이 인덱스에 포함 | 테이블 접근 없이 인덱스만으로 쿼리 해결, 매우 빠름 | `SELECT name FROM idx(name)` |
| **함수 기반 인덱스** (Function-Based Index) | 컬럼에 함수를 적용한 값으로 인덱스 생성 | 변환된 값으로 검색 시 사용 | `INDEX(UPPER(email))` |
| **부분 인덱스** (Partial Index) | WHERE 조건을 만족하는 행에만 인덱스 생성 | 인덱스 크기 줄이고 효율 향상 | `WHERE status = 'active'` |

---

## 3. B-Tree 인덱스 작동 원리

B-Tree는 DBMS에서 가장 널리 사용되는 인덱스 구조이다.

```
                    [ Root Node ]
                    |  30  |  70  |
                   /        |      \
        [ 10 | 20 ]   [ 40 | 60 ]   [ 80 | 90 ]
        (Leaf)          (Leaf)         (Leaf)
           ↕               ↕               ↕
       Row Ptr         Row Ptr         Row Ptr
```

### 검색 과정 (값 40 검색 시)
```
① Root 노드 확인 → 40 < 70, 40 > 30 → 중간 자식으로 이동
② 중간 Leaf 노드 [40 | 60] 확인 → 40 발견
③ Row Pointer로 실제 데이터 접근
```

| 특성 | 내용 |
|------|------|
| **시간복잡도** | O(log n) — 데이터 양이 많아져도 탐색 깊이가 완만하게 증가 |
| **균형 유지** | 삽입/삭제 시 자동으로 트리 재균형 (Split & Merge) |
| **범위 검색** | Leaf 노드들이 링크드 리스트로 연결되어 순차 탐색 용이 |
| **정렬** | 항상 정렬된 상태 유지 → ORDER BY 성능 향상 |

---

## 4. 클러스터드 vs 논클러스터드 인덱스 비교

```
[ 클러스터드 인덱스 ]          [ 논클러스터드 인덱스 ]
  인덱스 = 데이터 자체            인덱스  →  데이터 포인터
  ┌───────────────┐               ┌────────┐    ┌──────────┐
  │ 1 | Alice | 28│               │ Alice  │──→ │ 실제 행  │
  │ 2 |  Bob  | 25│               │  Bob   │──→ │ 실제 행  │
  │ 3 |Charlie| 30│               │Charlie │──→ │ 실제 행  │
  └───────────────┘               └────────┘    └──────────┘
    (물리적 정렬 저장)             (별도 구조)
```

| 구분 | 클러스터드 인덱스 | 논클러스터드 인덱스 |
|------|------------------|-------------------|
| **저장 방식** | 데이터 자체가 인덱스 순서로 저장 | 별도 인덱스 구조 + 포인터 |
| **개수 제한** | 테이블당 **1개** | 테이블당 **여러 개** (보통 최대 999개) |
| **검색 속도** | 범위 검색 매우 빠름 | 추가 I/O 필요 |
| **삽입/수정** | 물리적 재정렬 비용 발생 | 상대적으로 빠름 |
| **용량** | 별도 저장 공간 없음 | 추가 저장 공간 필요 |
| **자동 생성** | PK 설정 시 자동 생성 (InnoDB) | 수동 생성 필요 |

---

## 5. 성능 향상을 위한 인덱스 활용 전략

### 5-1. 인덱스 적용이 효과적인 경우

| 상황 | 이유 | 예시 |
|------|------|------|
| **WHERE 절에 자주 사용되는 컬럼** | 검색 조건으로 빠르게 필터링 | `WHERE user_id = 100` |
| **JOIN 조건 컬럼** | JOIN 시 매칭 속도 향상 | `ON a.id = b.user_id` |
| **ORDER BY / GROUP BY 컬럼** | 정렬 연산 생략 가능 | `ORDER BY created_at DESC` |
| **카디널리티(Cardinality)가 높은 컬럼** | 고유값이 많을수록 인덱스 효과 큼 | email, user_id, phone |
| **범위 검색 컬럼** | B-Tree의 범위 탐색 활용 | `BETWEEN`, `>`, `<` |
| **커버링 인덱스 설계** | 테이블 접근 없이 인덱스만으로 처리 | `SELECT id, name FROM idx(id, name)` |

### 5-2. 복합 인덱스 컬럼 순서 전략

```sql
-- 인덱스: (status, created_at, user_id)
-- 선두 컬럼(status)을 반드시 사용해야 인덱스 활용 가능

SELECT * FROM orders
WHERE status = 'ACTIVE'         -- ✅ 인덱스 사용
  AND created_at > '2024-01-01' -- ✅ 인덱스 사용
  AND user_id = 5;              -- ✅ 인덱스 사용

SELECT * FROM orders
WHERE created_at > '2024-01-01' -- ❌ status 없으면 인덱스 미사용
  AND user_id = 5;
```

| 순서 결정 기준 | 설명 |
|--------------|------|
| **선택도(Selectivity) 높은 컬럼 우선** | 더 많은 행을 걸러낼 수 있는 컬럼을 앞에 배치 |
| **등호(=) 조건 컬럼 우선** | 범위 조건보다 동등 조건 컬럼을 앞에 배치 |
| **자주 사용하는 조건 우선** | WHERE 절에 항상 등장하는 컬럼을 선두에 |

### 5-3. 인덱스 활용 쿼리 패턴

```sql
-- ✅ 인덱스 사용 (컬럼 직접 비교)
SELECT * FROM users WHERE email = 'test@example.com';
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 5000;
SELECT * FROM products WHERE name LIKE 'Apple%';  -- 전방 일치

-- ❌ 인덱스 미사용 (인덱스 무력화 패턴)
SELECT * FROM users WHERE UPPER(email) = 'TEST@EXAMPLE.COM';  -- 함수 적용
SELECT * FROM orders WHERE amount + 0 = 1000;                 -- 연산 적용
SELECT * FROM products WHERE name LIKE '%phone%';             -- 후방/양방향 LIKE
SELECT * FROM users WHERE age != 30;                          -- NOT 조건
```

---

## 6. 인덱스 사용 시 주의사항

### 6-1. 인덱스 무력화 패턴 (Index Sargability)

| 안티패턴 | 문제 이유 | 해결 방법 |
|---------|---------|---------|
| `WHERE YEAR(created_at) = 2024` | 함수 적용으로 인덱스 무력화 | `WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'` |
| `WHERE name LIKE '%홍%'` | 후방 일치 → Full Scan | Full-Text Index 사용 |
| `WHERE status != 'DONE'` | NOT 조건 → 대부분 Full Scan | 반대 조건으로 재작성 |
| `WHERE id + 1 = 100` | 컬럼에 연산 적용 | `WHERE id = 99` |
| `WHERE TO_CHAR(date) = '2024'` | 타입 변환 함수 | 날짜 범위로 직접 비교 |
| `NULL` 값 비교 | B-Tree는 NULL을 인덱싱하지 않음 | `IS NOT NULL` + 기본값 설정 |

### 6-2. 인덱스 과다 생성 문제

| 문제 | 영향 |
|------|------|
| **INSERT/UPDATE/DELETE 성능 저하** | 데이터 변경 시 모든 인덱스를 동시에 갱신해야 함 |
| **저장 공간 낭비** | 인덱스 자체가 상당한 디스크 공간 차지 |
| **옵티마이저 혼란** | 인덱스가 너무 많으면 최적 실행 계획 선택이 어려워짐 |
| **메모리 부족** | 인덱스가 버퍼 풀/캐시를 과도하게 점유 |

### 6-3. 인덱스가 오히려 손해인 경우

| 상황 | 이유 |
|------|------|
| **카디널리티가 낮은 컬럼** | gender, boolean 등 — 전체 스캔이 더 빠를 수 있음 |
| **소규모 테이블** | 행 수가 적으면 Full Scan이 오히려 빠름 |
| **자주 UPDATE되는 컬럼** | 인덱스 재정렬 비용이 이득보다 클 수 있음 |
| **SELECT 비율이 낮고 DML이 많은 테이블** | 쓰기 성능 하락이 읽기 이득을 상쇄 |

---

## 7. 실행 계획(EXPLAIN)으로 인덱스 확인

```sql
-- MySQL / MariaDB
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

### EXPLAIN 주요 컬럼 해석 (MySQL 기준)

| 컬럼 | 의미 | 좋은 값 | 나쁜 값 |
|------|------|---------|---------|
| `type` | 접근 방식 | `const`, `ref`, `range` | `ALL` (Full Scan) |
| `key` | 실제 사용된 인덱스 | 인덱스명 표시 | `NULL` (미사용) |
| `rows` | 예상 스캔 행 수 | 적을수록 좋음 | 전체 행 수에 가까우면 위험 |
| `Extra` | 추가 정보 | `Using index` (커버링) | `Using filesort`, `Using temporary` |

---

## 8. 인덱스 생성/관리 SQL

```sql
-- 단일 인덱스 생성
CREATE INDEX idx_users_email ON users(email);

-- 복합 인덱스 생성
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- 유니크 인덱스 생성
CREATE UNIQUE INDEX idx_users_phone ON users(phone);

-- 커버링 인덱스 생성
CREATE INDEX idx_covering ON orders(user_id, status, amount);

-- 부분 인덱스 (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'ACTIVE';

-- 함수 기반 인덱스 (PostgreSQL)
CREATE INDEX idx_upper_email ON users(UPPER(email));

-- 인덱스 삭제
DROP INDEX idx_users_email ON users;        -- MySQL
DROP INDEX idx_users_email;                  -- PostgreSQL

-- 인덱스 목록 확인 (MySQL)
SHOW INDEX FROM users;
```

---

## 9. 인덱스 종합 비교 요약표

| 항목 | B-Tree | Hash | Bitmap | Full-Text | Clustered |
|------|--------|------|--------|-----------|-----------|
| **등호 검색** | ✅ 빠름 | ✅ 매우 빠름 | ✅ 빠름 | ❌ | ✅ 빠름 |
| **범위 검색** | ✅ 빠름 | ❌ 불가 | ✅ 가능 | ❌ | ✅ 매우 빠름 |
| **정렬(ORDER BY)** | ✅ 지원 | ❌ 불가 | ❌ | ❌ | ✅ 지원 |
| **전문 검색** | ❌ | ❌ | ❌ | ✅ 최적 | ❌ |
| **카디널리티** | 높음에 유리 | 높음에 유리 | 낮음에 유리 | 텍스트 | 모든 경우 |
| **DML 성능** | 보통 | 빠름 | 느림 | 느림 | 느림 |
| **공간 효율** | 보통 | 좋음 | 매우 좋음(저카디널리티) | 나쁨 | 없음(데이터 자체) |
| **주요 DB** | 모든 RDBMS | MySQL Memory | Oracle, PG | MySQL, PG | MySQL InnoDB |

---

## 10. 인덱스 설계 체크리스트

```
✅ WHERE 절에 자주 등장하는 컬럼인가?
✅ JOIN 조건에 사용되는 컬럼인가?
✅ 카디널리티가 충분히 높은가? (고유값 비율 > 10~20% 권장)
✅ 복합 인덱스라면 선두 컬럼이 올바른가?
✅ 인덱스 개수가 과도하게 많지 않은가? (테이블당 5개 이내 권장)
✅ DML(INSERT/UPDATE/DELETE)이 매우 빈번한 테이블이 아닌가?
✅ EXPLAIN으로 실제 인덱스 사용 여부를 확인했는가?
✅ 함수/연산을 컬럼에 적용하여 인덱스를 무력화하고 있지 않은가?
✅ 커버링 인덱스 적용으로 추가 최적화가 가능한가?
```

---

> 📌 **핵심 원칙**: 인덱스는 `읽기 성능`을 높이는 대신 `쓰기 성능`과 `저장 공간`을 희생한다.  
> 무조건 많이 만드는 것이 아니라, **쿼리 패턴 분석 후 필요한 곳에 적절히** 설계해야 한다.
