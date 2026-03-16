# Oracle 23c on Docker — 아키텍처 실습 가이드

> **환경**: Oracle Database 23c Free · Docker · Windows/Linux  
> **목적**: 설치 → 접속 → HR 스키마 구성 → SQL 실습 → 실행계획 분석까지 체계적 학습

---

## 목차

1. [Oracle 아키텍처 개요](#1-oracle-아키텍처-개요)
2. [Docker 환경 설치](#2-docker-환경-설치)
3. [컨테이너 접속 및 CDB/PDB 구조 확인](#3-컨테이너-접속-및-cdbpdb-구조-확인)
4. [HR 스키마 설치](#4-hr-스키마-설치)
5. [기본 SQL 실습](#5-기본-sql-실습)
6. [실행계획 분석 (EXPLAIN PLAN / TKPROF)](#6-실행계획-분석-explain-plan--tkprof)
7. [SQL Trace 전체 흐름](#7-sql-trace-전체-흐름)
8. [TKPROF 결과 해석](#8-tkprof-결과-해석)
9. [자주 쓰는 관리 명령어 치트시트](#9-자주-쓰는-관리-명령어-치트시트)

---

## 1. Oracle 아키텍처 개요

### 1-1. CDB / PDB 구조

```
┌─────────────────────────────────────────────┐
│          CDB (Container Database)           │
│  ┌───────────┐   ┌────────────────────────┐ │
│  │  CDB$ROOT │   │      FREEPDB1 (PDB)    │ │
│  │  (공통관리) │   │  ┌──────────────────┐ │ │
│  └───────────┘   │  │  HR Schema       │ │ │
│  ┌───────────┐   │  │  (employees 등)  │ │ │
│  │  PDB$SEED │   │  └──────────────────┘ │ │
│  │  (복제원본) │   └────────────────────────┘ │
│  └───────────┘                               │
└─────────────────────────────────────────────┘
```

| 구성 요소 | 설명 |
|-----------|------|
| **CDB$ROOT** | 공통 메타데이터·딕셔너리 관리, SYS/SYSTEM 계정 소유 |
| **PDB$SEED** | 새 PDB 생성 시 복제되는 읽기전용 템플릿 |
| **FREEPDB1** | 실습용 플러그형 데이터베이스 (사용자 객체 생성 가능) |

### 1-2. 메모리 구조 (SGA / PGA)

```
┌────────────────── SGA (공유 메모리) ──────────────────┐
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │  Shared Pool │  │ Database     │  │ Redo Log   │  │
│  │ (SQL 캐시/   │  │ Buffer Cache │  │ Buffer     │  │
│  │  딕셔너리)   │  │ (데이터 블록) │  │            │  │
│  └──────────────┘  └──────────────┘  └────────────┘  │
└───────────────────────────────────────────────────────┘
┌────── PGA (세션별 메모리) ──────┐
│  Sort Area / Hash Area / Stack  │
└─────────────────────────────────┘
```

### 1-3. 프로세스 구조

| 프로세스 | 역할 |
|----------|------|
| **DBWn** | Dirty Buffer → 디스크 기록 |
| **LGWR** | Redo Buffer → Redo Log 파일 기록 |
| **CKPT** | 체크포인트 발생 시 SCN 갱신 |
| **SMON** | 인스턴스 복구, 임시 세그먼트 정리 |
| **PMON** | 실패한 프로세스 정리, 리소스 해제 |
| **ARCn** | 아카이브 로그 생성 (ARCHIVELOG 모드) |

---

## 2. Docker 환경 설치

### 2-1. docker run 방식 (빠른 시작)

```bash
docker run -d \
  --name oracle23c \
  -p 1521:1521 \
  -p 5500:5500 \
  -e ORACLE_PWD=Oracle123! \
  -v oracle23c-data:/opt/oracle/oradata \
  container-registry.oracle.com/database/free:latest
```

| 옵션 | 설명 |
|------|------|
| `-p 1521:1521` | SQL*Plus / JDBC 리스너 포트 |
| `-p 5500:5500` | Enterprise Manager Express |
| `-e ORACLE_PWD` | SYS / SYSTEM / PDBADMIN 초기 패스워드 |
| `-v oracle23c-data` | 데이터 파일 볼륨 영속화 |

> **초기화 확인**: `docker logs -f oracle23c` 에서 `DATABASE IS READY TO USE!` 메시지 대기

---

### 2-2. docker-compose 방식 (권장)

```yaml
# docker-compose.yml
version: '3.8'
services:
  oracle23c:
    image: container-registry.oracle.com/database/free:latest
    container_name: oracle23c
    ports:
      - "1521:1521"
      - "5500:5500"
    environment:
      ORACLE_PWD: Oracle123!
    volumes:
      - oracle23c-data:/opt/oracle/oradata

volumes:
  oracle23c-data:
```

```bash
# 실행
docker compose up -d

# 로그 확인
docker compose logs -f oracle23c

# 중지
docker compose down
```

---

## 3. 컨테이너 접속 및 CDB/PDB 구조 확인

### 3-1. 컨테이너 진입 → sqlplus 접속

```bash
# OS 레벨 접속 (oracle 유저)
docker exec -it --user oracle oracle23c bash

# sqlplus 인증 (OS 인증 — 패스워드 불필요)
sqlplus / as sysdba
```

### 3-2. CDB 상태 확인

```sql
-- 현재 컨테이너 확인
SHOW CON_NAME;
-- 결과: CDB$ROOT

-- 전체 PDB 목록 확인
SHOW PDBS;
/*
    CON_ID CON_NAME   OPEN MODE  RESTRICTED
---------- ---------- ---------- ----------
         2 PDB$SEED   READ ONLY  NO
         3 FREEPDB1   READ WRITE NO
*/

-- DB 버전 확인
SELECT version FROM v$instance;
```

### 3-3. PDB 전환 (FREEPDB1)

```sql
-- 세션을 FREEPDB1로 전환
ALTER SESSION SET CONTAINER = FREEPDB1;

-- 전환 확인
SHOW CON_NAME;
-- 결과: FREEPDB1
```

> ⚠️ **주의**: HR 스키마는 CDB$ROOT가 아닌 **FREEPDB1**에 설치해야 합니다.

---

## 4. HR 스키마 설치

### 4-1. HR 스크립트 실행

```sql
-- FREEPDB1 컨테이너에서 실행
ALTER SESSION SET CONTAINER = FREEPDB1;

@/tmp/human_resources/hr_install.sql
-- 프롬프트: Enter a tablespace for HR [USERS]: → 그냥 Enter (기본값 USERS 사용)
```

### 4-2. 설치 확인

```sql
-- HR 소유 테이블 목록
SELECT table_name FROM dba_tables WHERE owner = 'HR' ORDER BY 1;

/*
TABLE_NAME
-----------
COUNTRIES
DEPARTMENTS
EMPLOYEES
JOB_HISTORY
JOBS
LOCATIONS
REGIONS
*/
```

### 4-3. HR 스키마 ERD 요약

```
REGIONS ──< COUNTRIES ──< LOCATIONS ──< DEPARTMENTS ──< EMPLOYEES
                                                              │
                                              JOBS ───────────┤
                                                              │
                                         JOB_HISTORY <────────┘
```

| 테이블 | 주요 컬럼 | 설명 |
|--------|-----------|------|
| `EMPLOYEES` | employee_id, first_name, last_name, salary, department_id, job_id | 직원 정보 (107건) |
| `DEPARTMENTS` | department_id, department_name, manager_id, location_id | 부서 (27건) |
| `JOBS` | job_id, job_title, min_salary, max_salary | 직무 (19건) |
| `LOCATIONS` | location_id, city, country_id | 근무지 |
| `COUNTRIES` | country_id, country_name, region_id | 국가 |
| `REGIONS` | region_id, region_name | 지역 |
| `JOB_HISTORY` | employee_id, start_date, end_date, job_id | 직무 이력 |

---

## 5. 기본 SQL 실습

### 5-1. 기본 조회

```sql
-- 특정 직원 조회
SELECT * FROM hr.employees WHERE employee_id = 100;

-- 급여 상위 5명
SELECT employee_id, first_name, last_name, salary
FROM hr.employees
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;
```

### 5-2. 조인 (부서별 평균 급여)

```sql
SELECT d.department_name,
       COUNT(e.employee_id) AS headcount,
       ROUND(AVG(e.salary), 0) AS avg_salary
FROM hr.employees e
JOIN hr.departments d ON e.department_id = d.department_id
GROUP BY d.department_name
ORDER BY avg_salary DESC;
```

### 5-3. 윈도우 함수 (급여 순위)

```sql
SELECT employee_id,
       first_name,
       salary,
       RANK()       OVER (ORDER BY salary DESC) AS overall_rank,
       DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
FROM hr.employees;
```

### 5-4. 계층 쿼리 (조직도)

```sql
SELECT LEVEL,
       LPAD(' ', (LEVEL-1)*4) || first_name || ' ' || last_name AS emp_name,
       job_id
FROM hr.employees
START WITH manager_id IS NULL          -- 최상위 (Steven King)
CONNECT BY PRIOR employee_id = manager_id
ORDER SIBLINGS BY last_name;
```

---

## 6. 실행계획 분석 (EXPLAIN PLAN / TKPROF)

### 6-1. DBMS_XPLAN — 가장 간편한 방법

```sql
-- GATHER_PLAN_STATISTICS 힌트로 실제 통계 수집
SELECT /*+ GATHER_PLAN_STATISTICS */ *
FROM hr.employees
WHERE employee_id = 100;

-- 직전 실행된 SQL의 실행계획 출력
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

**출력 예시 해석**:

```
Plan hash value: 1833546154
-------------------------------------------------------------------------------------
| Id | Operation                   | Name        | Starts | E-Rows | A-Rows | Buffers|
-------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT            |             |      1 |        |      1 |      4 |
|  1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |      1 |      1 |      1 |      4 |
|* 2 |   INDEX UNIQUE SCAN         | EMP_EMP_ID_PK|      1 |      1 |      1 |      3 |
-------------------------------------------------------------------------------------
Predicate Information:
   2 - access("EMPLOYEE_ID"=100)
```

| 컬럼 | 의미 |
|------|------|
| `E-Rows` | 옵티마이저가 예측한 행 수 |
| `A-Rows` | 실제 반환된 행 수 |
| `Buffers` | 논리적 I/O (버퍼 캐시 읽기 횟수) |
| `*` 표시 | 필터/접근 조건 적용 |

---

### 6-2. SQL Trace + TKPROF — 상세 성능 분석

#### 단계 ①: Trace 활성화 및 SQL 실행

```sql
-- FREEPDB1 컨테이너로 전환
ALTER SESSION SET CONTAINER = FREEPDB1;

-- SQL Trace 시작
ALTER SESSION SET sql_trace = true;

-- 분석할 쿼리 실행
SELECT * FROM hr.employees WHERE employee_id = 100;

-- SQL Trace 종료
ALTER SESSION SET sql_trace = false;
```

#### 단계 ②: Trace 파일 경로 확인

```sql
SELECT value
FROM v$diag_info
WHERE name = 'Default Trace File';

-- 예) /opt/oracle/diag/rdbms/free/FREE/trace/FREE_ora_36257.trc
```

#### 단계 ③: TKPROF로 리포트 생성

```bash
# Docker 컨테이너 내부에서 실행
tkprof \
  /opt/oracle/diag/rdbms/free/FREE/trace/FREE_ora_36257.trc \
  /tmp/hr_trace.txt \
  sys=no \
  waits=yes \
  sort=exeela
```

| 옵션 | 설명 |
|------|------|
| `sys=no` | SYS 내부 재귀 SQL 제외 |
| `waits=yes` | 대기 이벤트 정보 포함 |
| `sort=exeela` | elapsed time (실행 경과 시간) 내림차순 정렬 |

#### 단계 ④: 결과 확인

```bash
cat /tmp/hr_trace.txt
```

---

## 7. SQL Trace 전체 흐름

```
┌──────────────────────────────────────────────────────────────┐
│                     SQL 실행 흐름                             │
│                                                              │
│  ① ALTER SESSION SET sql_trace = true                        │
│         │                                                    │
│         ▼                                                    │
│  ② SQL 실행 (SELECT / DML)                                   │
│         │                                                    │
│         ▼                                                    │
│  ③ Oracle → .trc 파일에 Parse/Execute/Fetch 정보 기록        │
│         │                                                    │
│         ▼                                                    │
│  ④ ALTER SESSION SET sql_trace = false                       │
│         │                                                    │
│         ▼                                                    │
│  ⑤ SELECT value FROM v$diag_info → trace 파일 경로 확인      │
│         │                                                    │
│         ▼                                                    │
│  ⑥ tkprof [trc파일] [출력파일] sys=no waits=yes sort=exeela  │
│         │                                                    │
│         ▼                                                    │
│  ⑦ cat [출력파일] → 성능 분석                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. TKPROF 결과 해석

### 8-1. 주요 컬럼 의미

```
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        2      0.00       0.00          0          1          0           0
Execute      2      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          0          2          0           1
```

| 컬럼 | 설명 |
|------|------|
| `call` | Parse(파싱) / Execute(실행) / Fetch(결과 인출) 단계 |
| `count` | 해당 단계가 호출된 횟수 |
| `cpu` | CPU 사용 시간(초) |
| `elapsed` | 실제 경과 시간(초) — 대기 포함 |
| `disk` | 물리적 디스크 I/O 횟수 |
| `query` | 일관성 읽기 버퍼 수 (Consistent Read) |
| `current` | 현재 모드 버퍼 수 (주로 DML) |
| `rows` | 처리된 행 수 |

### 8-2. 성능 분석 포인트

| 지표 | 문제 신호 | 해결 방향 |
|------|-----------|-----------|
| `disk` 값이 큼 | 물리 I/O 과다 → 버퍼 캐시 미스 | 인덱스 최적화, 버퍼 캐시 증설 |
| `elapsed >> cpu` | 대기 이벤트 발생 | `waits=yes` 결과에서 대기 원인 파악 |
| `E-Rows ≠ A-Rows` | 통계 정보 부정확 | `DBMS_STATS.GATHER_TABLE_STATS` 실행 |
| `Misses in library cache` 많음 | 하드 파싱 빈발 | 바인드 변수 사용, Shared Pool 크기 확인 |

### 8-3. 실습 결과 요약 (예시)

```
OVERALL TOTALS FOR ALL NON-RECURSIVE STATEMENTS
Parse  : 2회 / cpu 0.00s / elapsed 0.00s / disk 0
Execute: 2회 / cpu 0.00s / elapsed 0.00s
Fetch  : 2회 / query 2블록 / rows 1

Misses in library cache during parse: 1  ← 최초 실행 시 하드 파싱 1회
```

> **해석**: employee_id = 100 단건 조회이므로 disk=0 (버퍼 캐시 히트),  
> query=2 (인덱스 블록 + 테이블 블록), rows=1 → **최적 실행**

---

## 9. 자주 쓰는 관리 명령어 치트시트

### 9-1. 컨테이너 관리

```bash
# 컨테이너 상태 확인
docker ps -a

# Oracle 컨테이너 시작 / 중지
docker start oracle23c
docker stop oracle23c

# oracle 유저로 bash 진입
docker exec -it --user oracle oracle23c bash

# sqlplus 바로 실행
docker exec -it --user oracle oracle23c sqlplus / as sysdba
```

### 9-2. DB 관리 (sqlplus)

```sql
-- PDB 시작/중지
ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;
ALTER PLUGGABLE DATABASE FREEPDB1 CLOSE;

-- 재시작 후 자동 오픈 설정
ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;

-- 현재 세션 정보
SELECT sys_context('USERENV','CON_NAME') AS con_name,
       sys_context('USERENV','SESSION_USER') AS session_user
FROM dual;

-- 세션 목록
SELECT sid, serial#, username, status, machine
FROM v$session
WHERE type = 'USER';
```

### 9-3. 권한 관리

```sql
-- HR 계정 잠금 해제 (FREEPDB1에서)
ALTER SESSION SET CONTAINER = FREEPDB1;
ALTER USER hr IDENTIFIED BY hr ACCOUNT UNLOCK;

-- 기본 권한 부여
GRANT CONNECT, RESOURCE TO hr;
GRANT CREATE VIEW TO hr;

-- HR 계정으로 접속 테스트
CONNECT hr/hr@localhost:1521/FREEPDB1
```

### 9-4. 트레이스 파일 위치

```bash
# 트레이스 디렉터리
ls /opt/oracle/diag/rdbms/free/FREE/trace/

# 최근 수정된 trc 파일 찾기
ls -lt /opt/oracle/diag/rdbms/free/FREE/trace/*.trc | head -5

# TKPROF 일괄 처리
for f in /opt/oracle/diag/rdbms/free/FREE/trace/*.trc; do
  tkprof "$f" "/tmp/$(basename "$f" .trc).txt" sys=no waits=yes sort=exeela
done
```

---

## 참고: 접속 정보 요약

| 항목 | 값 |
|------|----|
| Host | localhost |
| Port | 1521 |
| Service (CDB) | FREE |
| Service (PDB) | FREEPDB1 |
| SYS 패스워드 | Oracle123! |
| HR 패스워드 | hr (설치 후 변경 가능) |
| EM Express URL | https://localhost:5500/em |

---

*작성 기준: Oracle Database 23c Free · Docker · 2026년 3월*
