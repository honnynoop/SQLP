# SQLP 수행구조 — 데이터베이스 아키텍처 & I/O 메커니즘

---

## 목차

1. [Oracle 데이터베이스 전체 아키텍처](#1-oracle-데이터베이스-전체-아키텍처)
2. [SQL 수행 처리 흐름](#2-sql-수행-처리-흐름)
3. [메모리 구조 (SGA / PGA)](#3-메모리-구조-sga--pga)
4. [프로세스 구조](#4-프로세스-구조)
5. [I/O 메커니즘 상세](#5-io-메커니즘-상세)
6. [버퍼 캐시 동작 원리](#6-버퍼-캐시-동작-원리)
7. [Physical I/O vs Logical I/O](#7-physical-io-vs-logical-io)
8. [Redo / Undo 메커니즘](#8-redo--undo-메커니즘)
9. [체크포인트 & 라이터 메커니즘](#9-체크포인트--라이터-메커니즘)
10. [핵심 요점 정리](#10-핵심-요점-정리)

---

## 1. Oracle 데이터베이스 전체 아키텍처

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Oracle Database Instance                        │
│                                                                        │
│  ┌──────────────────────────── SGA ──────────────────────────────────┐ │
│  │  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────────┐ │ │
│  │  │  Shared Pool    │  │ Buffer Cache │  │   Redo Log Buffer    │ │ │
│  │  │ ┌─────────────┐ │  │              │  │                      │ │ │
│  │  │ │Library Cache│ │  │  Data Block  │  │  변경 벡터 임시 저장  │ │ │
│  │  │ │(SQL/PL 캐시)│ │  │   Buffers   │  │                      │ │ │
│  │  │ ├─────────────┤ │  │              │  └──────────────────────┘ │ │
│  │  │ │Data Dict.   │ │  └──────────────┘  ┌──────────────────────┐ │ │
│  │  │ │Cache        │ │  ┌──────────────┐  │   Large Pool /       │ │ │
│  │  │ └─────────────┘ │  │ Java Pool /  │  │   Streams Pool       │ │ │
│  │  └─────────────────┘  │ Fixed SGA   │  └──────────────────────┘ │ │
│  │                        └──────────────┘                           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────── Background Processes ─────────────────────┐ │
│  │   DBWn   LGWR   CKPT   SMON   PMON   ARCn   RECO   ...           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐        ┌──────────────────────┐
│   Data Files    │        │   Redo Log Files      │
│ (.dbf)          │        │  (Online / Archive)   │
├─────────────────┤        └──────────────────────┘
│ Control Files   │
│ Parameter File  │
└─────────────────┘
```

---

## 2. SQL 수행 처리 흐름

### 2.1 전체 흐름 다이어그램

```
User Process (Client)
        │
        │  SQL 전송
        ▼
Server Process (Dedicated / Shared)
        │
        ├─① Parse (파싱)
        │    ├─ Syntax Check   → SQL 문법 검사
        │    ├─ Semantic Check → 객체·권한 존재 여부
        │    └─ Soft Parse / Hard Parse
        │           │
        │           └─ Library Cache 조회
        │                ├─ Hit  → 실행계획 재사용 (Soft Parse)
        │                └─ Miss → Optimizer 호출 (Hard Parse)
        │
        ├─② Bind (바인드)
        │    └─ 바인드 변수 값 치환
        │
        ├─③ Execute (실행)
        │    ├─ Buffer Cache 조회
        │    │    ├─ Hit  → Logical Read (메모리)
        │    │    └─ Miss → Physical Read (디스크 → 캐시 적재)
        │    └─ DML 시 Undo/Redo 생성
        │
        └─④ Fetch (페치)
             └─ 결과 집합 → Client 전송 (SELECT 한정)
```

### 2.2 Parse 단계 상세

| 단계 | 내용 | 비용 |
|------|------|------|
| **Syntax Check** | SQL 문법 오류 검사 | 低 |
| **Semantic Check** | 테이블·컬럼·권한 존재 여부 (Data Dictionary 조회) | 中 |
| **Soft Parse** | Library Cache에서 실행계획 발견 → 재사용 | 低 |
| **Hard Parse** | 실행계획 없음 → Optimizer가 최적 실행계획 생성 | **高** |

> ⚠️ **Hard Parse는 CPU/메모리를 많이 소비** → 바인드 변수 사용으로 Soft Parse 유도가 SQLP 핵심 최적화 전략

---

## 3. 메모리 구조 (SGA / PGA)

### 3.1 SGA (System Global Area) — 인스턴스 공유 메모리

```
┌─────────────────────────────────────────────────────┐
│                        SGA                          │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │            Shared Pool                       │  │
│  │  ┌─────────────────┬──────────────────────┐  │  │
│  │  │  Library Cache  │  Data Dictionary     │  │  │
│  │  │  - SQL Area     │  Cache               │  │  │
│  │  │  - PL/SQL Area  │  - 테이블/컬럼 메타  │  │  │
│  │  │  - Cursor 정보  │  - 권한 정보         │  │  │
│  │  └─────────────────┴──────────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │           Database Buffer Cache               │  │
│  │   Dirty Buffer │ Clean Buffer │ Free Buffer   │  │
│  │   (변경됨)      │ (변경안됨)   │ (비어있음)    │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────┐  ┌───────────────────────┐   │
│  │  Redo Log Buffer │  │     Large Pool        │   │
│  │  (변경 벡터)     │  │ (병렬처리/백업/공유서버)│   │
│  └──────────────────┘  └───────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 3.2 PGA (Program Global Area) — 세션 전용 메모리

| 영역 | 설명 |
|------|------|
| **Sort Area** | ORDER BY, GROUP BY, 해시 조인 시 정렬 공간 |
| **Hash Area** | Hash Join 시 빌드 입력 저장 |
| **Bitmap Merge Area** | 비트맵 인덱스 병합 |
| **Cursor State** | 현재 실행 중인 SQL 커서 상태 |
| **Session Information** | 세션 변수, NLS 설정 등 |

> **PGA가 부족**하면 Temporary Tablespace(디스크)로 Spill → 성능 급락

### 3.3 SGA vs PGA 비교

| 구분 | SGA | PGA |
|------|-----|-----|
| 공유 범위 | **인스턴스 전체** 공유 | **세션 전용** (비공유) |
| 주요 역할 | SQL 캐시, 데이터 캐시 | 정렬, 해시, 세션 정보 |
| 크기 설정 | `sga_target` / `sga_max_size` | `pga_aggregate_target` |
| 접근 보호 | Latch / Mutex 필요 | 필요 없음 |

---

## 4. 프로세스 구조

### 4.1 Background Process 역할

| 프로세스 | 약칭 | 역할 | I/O 관련성 |
|---------|------|------|------------|
| **Database Writer** | DBWn | Dirty Buffer → Data File 기록 | ⭐ 핵심 Write I/O |
| **Log Writer** | LGWR | Redo Log Buffer → Redo Log File | ⭐ 핵심 Write I/O |
| **Checkpoint** | CKPT | 체크포인트 신호 및 Control File 갱신 | SCN 관리 |
| **System Monitor** | SMON | 인스턴스 복구, 임시 세그먼트 정리 | 복구 I/O |
| **Process Monitor** | PMON | 비정상 종료 프로세스 정리 | 롤백 처리 |
| **Archiver** | ARCn | Online Redo Log → Archive Log 복사 | 아카이브 I/O |
| **Recoverer** | RECO | 분산 트랜잭션 복구 | - |

### 4.2 Server Process vs Background Process

```
Client
  │
  └──→ Server Process (포그라운드)
           │  SQL 파싱·실행·페치
           │  Buffer Cache Read
           │  Redo 생성 → Redo Buffer에 기록
           │
           └──→ 필요 시 Background Process 깨움
                    ├─ LGWR : Redo 기록 (commit 시)
                    ├─ DBWn : Dirty 버퍼 기록 (Free 버퍼 부족 시)
                    └─ CKPT : 체크포인트 트리거
```

---

## 5. I/O 메커니즘 상세

### 5.1 I/O 계층 구조

```
SQL 실행
  │
  ▼
┌─────────────────────────────────────────┐
│         Buffer Cache (Logical I/O)      │  ← 1st 탐색
│  Block이 있으면 → 즉시 반환 (메모리 I/O) │
└─────────────┬───────────────────────────┘
              │  Cache Miss
              ▼
┌─────────────────────────────────────────┐
│        OS File System Cache             │  ← 2nd 탐색 (OS 레벨)
│        (OS Buffer Cache)                │
└─────────────┬───────────────────────────┘
              │  OS Cache Miss
              ▼
┌─────────────────────────────────────────┐
│          Physical Disk I/O              │  ← 실제 디스크 읽기
│   HDD: ~5~10ms  /  SSD: ~0.1ms         │
└─────────────────────────────────────────┘
```

### 5.2 Single Block I/O vs Multi Block I/O

| 구분 | Single Block I/O | Multi Block I/O |
|------|-----------------|-----------------|
| **대상** | Index Scan | Full Table Scan |
| **단위** | 1 Block (DB_BLOCK_SIZE) | `DB_FILE_MULTIBLOCK_READ_COUNT` blocks |
| **특성** | Random Access | Sequential Access |
| **I/O 수** | 많음 | 적음 |
| **성능** | 소량 데이터에 유리 | 대량 데이터에 유리 |
| **대기 이벤트** | `db file sequential read` | `db file scattered read` |

```
Single Block I/O (Index)           Multi Block I/O (FTS)
┌───┐                              ┌───┬───┬───┬───┬───┐
│ B │ ← 1번에 1 Block              │ B │ B │ B │ B │ B │ ← 1번에 N Blocks
└───┘                              └───┴───┴───┴───┴───┘
(db file sequential read)          (db file scattered read)
```

### 5.3 I/O 대기 이벤트 목록

| 대기 이벤트 | 발생 원인 | 해결 방향 |
|------------|----------|----------|
| `db file sequential read` | 인덱스 스캔 (단일 블록 읽기) | 인덱스 효율화, 클러스터링 팩터 개선 |
| `db file scattered read` | FTS (멀티 블록 읽기) | 파티셔닝, 인덱스 추가 |
| `db file parallel read` | 병렬 쿼리 | 병렬도 조정 |
| `direct path read` | PGA Spill, 병렬 FTS | PGA 증설 |
| `log file sync` | COMMIT 후 LGWR 완료 대기 | Redo I/O 최소화, 배치 COMMIT |
| `log file parallel write` | LGWR가 Redo 기록 중 | Redo Log 파일을 빠른 디스크로 |
| `free buffer waits` | Free 버퍼 부족 (DBWn 지연) | Buffer Cache 증설, DBWn 튜닝 |

---

## 6. 버퍼 캐시 동작 원리

### 6.1 LRU 알고리즘

```
Most Recently Used (MRU)                    Least Recently Used (LRU)
◄────────────────────────────────────────────────────────────────────►

[새 블록 적재] ──→  [HOT]  [HOT]  [WARM]  [COLD]  [COLD]  ──→ [제거 대상]
                    최근 사용                           오래된 블록
```

- **LRU List** : 사용 빈도에 따라 블록 관리
- **DIRTY List (LRUW)** : 변경된(Dirty) 블록 추적 → DBWn이 기록 대상 선택

### 6.2 Buffer 상태

| 상태 | 설명 |
|------|------|
| **Free** | 비어 있음, 즉시 사용 가능 |
| **Clean** | 디스크와 동일한 내용 (변경 없음) |
| **Dirty** | 변경됨, 아직 디스크에 기록 안 됨 |
| **Pinned** | 현재 Access 중 (고정됨) |

### 6.3 Buffer Cache Miss 처리 흐름

```
1. LRU List 스캔 → Free Buffer 탐색
2. Free Buffer 없음 → Dirty Buffer 중 LRU End 버퍼 선택
3. Dirty Buffer가 있으면 → DBWn 에게 기록 요청 (free buffer waits 발생)
4. DBWn이 Data File에 기록 → 해당 버퍼를 Free로 전환
5. 디스크에서 새 블록 읽어 해당 버퍼에 적재
```

---

## 7. Physical I/O vs Logical I/O

### 7.1 개념 비교

| 구분 | Logical I/O (논리적) | Physical I/O (물리적) |
|------|---------------------|----------------------|
| **정의** | Buffer Cache를 포함한 모든 블록 접근 수 | 실제 디스크에서 읽은 블록 수 |
| **속도** | 빠름 (ns ~ μs 수준) | 느림 (ms 수준) |
| **측정 지표** | `consistent gets` + `db block gets` | `physical reads` |
| **최적화 목표** | Logical I/O 자체를 줄이는 것 | Buffer Cache Hit Rate 높이기 |

### 7.2 Buffer Cache Hit Ratio

```
Buffer Cache Hit Ratio = (Logical I/O - Physical I/O) / Logical I/O × 100

예: Logical I/O = 10,000,  Physical I/O = 500
    Hit Ratio = (10,000 - 500) / 10,000 × 100 = 95%
```

> ⚠️ Hit Ratio가 높아도 **Logical I/O 자체가 많으면** 성능 문제 발생  
> → **진짜 목표는 Logical I/O 수를 줄이는 것**

### 7.3 I/O 효율화 방법

| 방법 | 설명 |
|------|------|
| **인덱스 최적화** | 불필요한 Random I/O 제거 |
| **클러스터링 팩터** | 인덱스와 테이블 데이터 정렬 일치 |
| **파티셔닝** | Partition Pruning으로 I/O 범위 제한 |
| **바인드 변수** | Hard Parse 줄여 Shared Pool 효율화 |
| **배치 처리** | Array Fetch / Bulk Collect로 Round Trip 최소화 |
| **클러스터 테이블** | 자주 조인되는 테이블을 물리적으로 인접 저장 |

---

## 8. Redo / Undo 메커니즘

### 8.1 Redo (재실행 로그)

```
목적: 장애 복구 (Roll Forward)

DML 발생
  │
  ├─ 변경 내용(Redo Vector)을 Redo Log Buffer에 기록
  │
  └─ COMMIT
       │
       └─ LGWR가 Redo Log Buffer → Online Redo Log File 기록
              │  (Log File Sync 대기)
              └─ COMMIT 완료 응답
```

**LGWR가 Redo를 기록하는 시점:**
- COMMIT 발생 시
- Redo Log Buffer가 1/3 이상 찼을 때
- 3초마다 (타이머)
- DBWn이 Dirty Buffer 기록하기 전 (WAL 원칙)

### 8.2 Undo (롤백 세그먼트)

```
목적: 트랜잭션 롤백, 읽기 일관성(CR Block 생성)

DML 발생
  │
  ├─ 변경 이전 값(Before Image)을 Undo Segment에 기록
  │
  ├─ ROLLBACK 시 → Undo를 이용해 원래 값으로 복원
  │
  └─ SELECT (다른 세션) → CR Block 생성 (읽기 일관성)
       └─ 현재 블록의 변경 내용을 Undo로 되돌린 CR(Consistent Read) 블록 생성
```

### 8.3 Redo vs Undo 비교

| 구분 | Redo Log | Undo Segment |
|------|----------|--------------|
| **목적** | 변경 재실행 (복구) | 변경 취소 (롤백 / CR) |
| **저장 내용** | 변경 후 이미지 (After Image) | 변경 전 이미지 (Before Image) |
| **저장 위치** | Redo Log Buffer → Redo Log File | Undo Tablespace |
| **Write 시점** | DML 즉시 & COMMIT 시 강제 | DML 즉시 |
| **관리** | LGWR 담당 | SMON 정리 |

### 8.4 Redo / Undo 흐름 통합 다이어그램

```
                    DML (UPDATE)
                        │
         ┌──────────────┼──────────────┐
         ▼                             ▼
  Undo Segment                  Redo Log Buffer
  (Before Image 저장)           (After Image + Undo 변경 기록)
         │                             │
         │  ROLLBACK                   │  COMMIT
         ▼                             ▼
  원래 데이터 복원             LGWR → Online Redo Log
                              (WAL: Write-Ahead Logging)
```

---

## 9. 체크포인트 & 라이터 메커니즘

### 9.1 Checkpoint 개념

```
목적: 복구 시간 단축 (Redo Log 전체 적용 불필요)

체크포인트 발생
  │
  ├─ CKPT 프로세스가 DBWn에게 신호 → Dirty Buffer 디스크 기록
  ├─ Control File에 체크포인트 SCN 기록
  └─ Data File Header에 SCN 기록

복구 시: 체크포인트 SCN 이후의 Redo만 적용하면 됨
```

### 9.2 체크포인트 발생 조건

| 조건 | 유형 |
|------|------|
| `LOG_CHECKPOINT_INTERVAL` 초과 | 시간 기반 |
| Redo Log File 스위치 발생 | 로그 스위치 |
| `ALTER SYSTEM CHECKPOINT` 명령 | 수동 |
| 테이블스페이스 Offline/Begin Backup | 이벤트 기반 |
| 인스턴스 정상 종료 | 종료 시 |

### 9.3 DBWn (Database Writer) 동작 조건

| 기록 트리거 | 설명 |
|------------|------|
| Free Buffer 부족 | LRU End Dirty 버퍼 기록 |
| Checkpoint 요청 | CKPT 신호 수신 |
| Dirty 버퍼 임계값 도달 | `db_writer_processes` 파라미터 |
| Timeout (3초) | 주기적 점검 |
| Tablespace Offline | 해당 데이터 파일 버퍼 모두 기록 |

---

## 10. 핵심 요점 정리

### 10.1 SQL 수행 단계별 핵심

| 단계 | 핵심 포인트 | 최적화 방향 |
|------|------------|------------|
| **Parse** | Hard Parse는 비용이 매우 크다 | 바인드 변수 사용, Cursor 재사용 |
| **Bind** | 바인드 변수 Peeking 주의 | 히스토그램 + 바인드 변수 관계 파악 |
| **Execute** | Buffer Cache Hit 여부가 성능 결정 | Logical I/O 최소화, 인덱스 최적화 |
| **Fetch** | Array Size 조정으로 Round Trip 감소 | `arraysize` / `fetchsize` 설정 |

### 10.2 I/O 최적화 핵심 원칙

```
① Logical I/O를 줄여라
   → 인덱스, 파티셔닝, 클러스터링으로 접근 블록 수 감소

② Buffer Cache Hit Rate를 높여라
   → SGA 크기 최적화, 자주 사용 블록이 LRU에서 제거되지 않도록

③ Physical I/O가 불가피하면 Sequential로 유도하라
   → Random < Sequential (Scattered Read 유도)

④ COMMIT 빈도를 조절하라
   → LGWR I/O 최소화 (배치 COMMIT, NOLOGGING 활용)

⑤ PGA를 충분히 확보하라
   → Sort/Hash Join Spill 방지 → Temp I/O 제거
```

### 10.3 주요 파라미터 정리

| 파라미터 | 설명 | 권장 설정 |
|---------|------|----------|
| `db_block_size` | 블록 크기 | OLTP: 8KB, DW: 16~32KB |
| `db_cache_size` | Buffer Cache 크기 | 가능한 크게 |
| `shared_pool_size` | Shared Pool 크기 | SQL 종류·수에 따라 조정 |
| `pga_aggregate_target` | PGA 전체 목표 | 물리 메모리의 20% 기준 |
| `db_file_multiblock_read_count` | FTS 시 멀티블록 읽기 수 | 8~128 (I/O 특성에 따라) |
| `log_buffer` | Redo Log Buffer 크기 | 고 DML 환경에서 증설 |
| `undo_retention` | Undo 보존 기간(초) | 장시간 쿼리 시 증가 |

### 10.4 SQLP 시험 빈출 개념 요약

| 개념 | 핵심 설명 |
|------|----------|
| **CR Block** | 읽기 일관성을 위해 Undo로 생성한 Consistent Read 블록 |
| **Current Block** | 가장 최신 변경이 반영된 블록 (DML 대상) |
| **SCN** | System Change Number — 변경 순서를 추적하는 타임스탬프 역할 |
| **WAL** | Write-Ahead Logging — DBWn보다 LGWR가 반드시 먼저 기록 |
| **Latch** | SGA 내 공유 자원 보호하는 경량 잠금 (Spin Lock 방식) |
| **Mutex** | Library Cache 보호에 주로 사용 (Latch 대체/보완) |
| **클러스터링 팩터** | 인덱스 순서와 테이블 데이터 저장 순서 일치 정도 |
| **선택도(Selectivity)** | 인덱스 스캔 효율 → 낮을수록 인덱스 효과적 |

---

> 📌 **최종 핵심 한 줄 요약**  
> SQLP에서 I/O 최적화의 본질은 **"디스크를 덜 읽고, 메모리에서 더 많이 처리하며, 불필요한 Parse와 Lock을 피하는 것"** 이다.

---

*작성 기준: Oracle Database Architecture (12c/19c), SQLP 핵심 이론*
