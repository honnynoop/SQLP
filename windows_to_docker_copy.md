# Windows → Docker 컨테이너 파일 복사 가이드

> **목적**: `C:\oraclesrc\db-sample-schemas-main\human_resources` 폴더를  
> Oracle 23c 컨테이너의 `/tmp/human_resources`로 복사

---

## 방법 1. `docker cp` 명령어 (가장 간단 ✅ 권장)

### PowerShell 또는 CMD에서 실행

```powershell
docker cp `
  "C:\oraclesrc\db-sample-schemas-main\db-sample-schemas-main\human_resources" `
  oracle23c:/tmp/human_resources
```

> CMD(명령 프롬프트)를 사용하는 경우 백틱(`) 없이 한 줄로 입력합니다.

```cmd
docker cp "C:\oraclesrc\db-sample-schemas-main\db-sample-schemas-main\human_resources" oracle23c:/tmp/human_resources
```

### 복사 확인

```bash
# 컨테이너 내부에서 파일 목록 확인
docker exec -it --user oracle oracle23c ls -l /tmp/human_resources
```

**예상 출력:**

```
total 64
-rw-r--r-- 1 oracle oinstall  2847 Mar 12 09:00 hr_code.sql
-rw-r--r-- 1 oracle oinstall  1203 Mar 12 09:00 hr_create.sql
-rw-r--r-- 1 oracle oinstall  4921 Mar 12 09:00 hr_install.sql
-rw-r--r-- 1 oracle oinstall  3287 Mar 12 09:00 hr_popul.sql
...
```

---

## 방법 2. `docker-compose` 볼륨 마운트 (영속적 공유)

컨테이너를 재시작해도 파일이 유지되어야 할 때 적합합니다.

### `docker-compose.yml` 수정

```yaml
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
      # ↓ Windows 경로를 컨테이너 /tmp/human_resources에 마운트
      - "C:/oraclesrc/db-sample-schemas-main/db-sample-schemas-main/human_resources:/tmp/human_resources:ro"

volumes:
  oracle23c-data:
```

> `:ro` — 읽기 전용(read-only) 마운트. 원본 파일 보호 목적.  
> 쓰기가 필요하면 `:ro`를 제거합니다.

### 컨테이너 재시작

```powershell
docker compose down
docker compose up -d
```

### 확인

```bash
docker exec -it --user oracle oracle23c ls -l /tmp/human_resources
```

---

## 방법 3. `docker run` 실행 시 볼륨 마운트

컨테이너를 처음 생성할 때부터 마운트하는 방법입니다.

```powershell
docker run -d `
  --name oracle23c `
  -p 1521:1521 `
  -p 5500:5500 `
  -e ORACLE_PWD=Oracle123! `
  -v oracle23c-data:/opt/oracle/oradata `
  -v "C:/oraclesrc/db-sample-schemas-main/db-sample-schemas-main/human_resources:/tmp/human_resources:ro" `
  container-registry.oracle.com/database/free:latest
```

---

## 방법 비교

| 방법 | 명령어 위치 | 영속성 | 재시작 후 유지 | 추천 상황 |
|------|-------------|--------|----------------|-----------|
| `docker cp` | PowerShell / CMD | ❌ 일회성 | 컨테이너 삭제 시 소멸 | 빠른 테스트, 단발성 작업 |
| `docker-compose` 볼륨 마운트 | docker-compose.yml | ✅ 영속 | 유지됨 | 반복 실습, 파일 자주 수정 |
| `docker run` 볼륨 마운트 | PowerShell / CMD | ✅ 영속 | 유지됨 | compose 미사용 환경 |

---

## 복사 후 HR 스키마 설치

파일 복사가 완료되면 아래 순서로 HR 스키마를 설치합니다.

```bash
# 1. sqlplus 접속
docker exec -it --user oracle oracle23c sqlplus / as sysdba
```

```sql
-- 2. PDB로 전환
ALTER SESSION SET CONTAINER = FREEPDB1;

-- 3. 현재 컨테이너 확인
SHOW CON_NAME;
-- 결과: FREEPDB1

-- 4. HR 스키마 설치 스크립트 실행
@/tmp/human_resources/hr_install.sql
-- 프롬프트: Enter a tablespace for HR [USERS]: → Enter (기본값 사용)

-- 5. 설치 확인
SELECT table_name FROM dba_tables WHERE owner = 'HR' ORDER BY 1;
```

---

## 트러블슈팅

### ❶ `docker cp` 실행 시 "No such container" 오류

```powershell
# 컨테이너 실행 여부 확인
docker ps -a

# 중지된 경우 시작
docker start oracle23c
```

### ❷ 권한 오류 (Permission denied)

```bash
# 컨테이너 내부에서 권한 변경
docker exec -it --user root oracle23c chmod -R 755 /tmp/human_resources
docker exec -it --user root oracle23c chown -R oracle:oinstall /tmp/human_resources
```

### ❸ Windows 경로 구분자 오류 (볼륨 마운트 시)

```yaml
# ❌ 잘못된 예 (백슬래시)
- "C:\oraclesrc\human_resources:/tmp/human_resources"

# ✅ 올바른 예 (슬래시로 변경)
- "C:/oraclesrc/db-sample-schemas-main/db-sample-schemas-main/human_resources:/tmp/human_resources:ro"
```

### ❹ `hr_install.sql` 실행 시 파일 없음 오류

```sql
-- 경로 재확인
HOST ls /tmp/human_resources
-- 또는
!ls /tmp/human_resources
```

---

*작성 기준: Docker Desktop for Windows · Oracle Database 23c Free · 2026년 3월*
