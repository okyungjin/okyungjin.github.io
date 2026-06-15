---
title: PostgreSQL에서의 Lock
description: MVCC 기반 동작부터 테이블 락 모드, 충돌 매트릭스, 락 모니터링, 데드락 처리까지 PostgreSQL의 잠금 메커니즘을 정리합니다.
pubDate: 2025-02-18
category: PostgreSQL
tags:
  - postgresql
  - db/lock
---

## MVCC와 PostgreSQL의 기본 동작

### MVCC (Multi-Version Concurrency Control)
PostgreSQL은 **행마다 여러 버전**(Tuple Version)을 유지하여 동시에 여러 트랜잭션이 읽고 쓰기를 수행할 수 있게 합니다.

- 단순 SELECT 시 **공유 락(Shared Lock) 없이** 스냅샷을 통해 일관성 있는 데이터 조회 가능
- 읽기 작업이 쓰기 작업을 차단하지 않으며, 쓰기 작업도 읽기 작업을 차단하지 않음
- 각 트랜잭션은 시작 시점의 데이터베이스 스냅샷을 기준으로 데이터를 조회

### Row-Level Lock (행 수준 락)
- **UPDATE**, **DELETE** 실행 시 변경 대상 행에 대해 **Row Exclusive Lock** 획득
- 다른 트랜잭션이 동일한 행을 동시에 UPDATE/DELETE하려고 하면 충돌 발생
- 이 경우 나중에 시도한 트랜잭션은 먼저 획득한 트랜잭션이 커밋 또는 롤백될 때까지 **대기**
- 여러 트랜잭션이 순환 대기하면 **데드락(Deadlock)** 발생 가능

---

## Lock 모드

PostgreSQL은 다양한 수준의 테이블 락을 제공하여 동시성을 제어합니다. 각 락 모드는 특정 작업에 최적화되어 있으며, 다른 락 모드와의 호환성이 다릅니다.

### Access Share Lock
- 단순 SELECT 쿼리 실행 시 획득되는 가장 약한 수준의 락입니다.
- 대부분의 다른 락 모드와 공존할 수 있습니다.
```sql
SELECT * FROM users;
```

### Row Share Lock
- `SELECT ... FOR SHARE` 구문이나 FK 체크, 트리거 등에서 사용됩니다.
- 다른 트랜잭션의 데이터 읽기를 허용하면서 행 수준 공유 락을 획득합니다.
```sql
SELECT * FROM users WHERE id = 1 FOR SHARE;
```

### Row Exclusive Lock
- 일반적인 DML 작업(`INSERT`, `UPDATE`, `DELETE`)에서 획득되는 가장 흔한 락 모드입니다.
- 여러 트랜잭션이 동시에 서로 다른 행을 수정할 수 있습니다.
```sql
UPDATE users SET name = 'John' WHERE id = 1;
```

### Share Update Exclusive
- `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` 등에서 사용됩니다.
- 자기 자신과도 충돌하여 한 번에 하나의 작업만 실행 가능합니다.
```sql
VACUUM users;
```

### Share Lock
- 비동시 모드의 `CREATE INDEX` 실행 시 획득됩니다.
- 여러 SHARE 락은 동시에 획득 가능하지만, 데이터 변경 작업은 차단됩니다.
```sql
CREATE INDEX idx_name ON users(name);
```

### Share Row Exclusive Lock
- `CREATE TRIGGER`나 일부 `ALTER TABLE` 작업에서 사용됩니다.
- SHARE와 ROW EXCLUSIVE의 특성을 결합한 락입니다.
```sql
CREATE TRIGGER trigger_name ...;
```

### Exclusive Lock
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` 등에서 사용됩니다.
- 테이블에 대한 배타적 접근이 필요하지만, ACCESS SHARE 락(읽기)은 허용합니다.
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY my_mv;
```

### Access Exclusive Lock
- **`DROP TABLE`, `TRUNCATE`, `REINDEX`, `ALTER TABLE` 등 DDL 수행 시 획득되는 가장 강력한 테이블 락입니다.**
- 모든 접근을 차단합니다.
```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
```

## Lock 충돌 매트릭스

아래 표는 각 락 모드 간의 충돌 관계를 나타냅니다.

![Lock 충돌 매트릭스](/images/postgresql_lock.png)

행(왼쪽)의 락을 보유한 트랜잭션이 있을 때, 열(위쪽)의 락을 다른 트랜잭션이 획득할 수 있는지 여부를 나타냅니다.

- ACCESS SHARE와 ROW EXCLUSIVE는 호환 가능 → SELECT와 UPDATE가 동시 실행 가능
- ROW EXCLUSIVE와 SHARE는 충돌 (X) → UPDATE 중에는 CREATE INDEX 대기
- ACCESS EXCLUSIVE는 모든 락과 충돌 (X) → ALTER TABLE 중에는 모든 접근 차단

---

## Lock 모니터링

PostgreSQL은 락 관련 상태를 모니터링할 수 있는 시스템 뷰를 제공합니다.

### pg_locks 뷰
현재 어떤 세션이 어떤 객체(테이블/페이지/튜플)에 어떤 락 모드를 가지고 있는지 확인할 수 있습니다.

```sql
-- 현재 획득되지 않은 락 조회 (대기 중인 락)
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE NOT granted;
```

### pg_stat_activity와 블로킹 세션 확인
특정 세션을 블로킹하고 있는 다른 세션을 확인할 수 있습니다.

```sql
-- 활성 세션과 해당 세션을 블로킹하는 PID 조회
SELECT pid, pg_blocking_pids(pid) AS blocking_pids
FROM pg_stat_activity
WHERE state = 'active';
```

### 락을 대기 중인 쿼리 상세 조회
락을 획득하지 못해 대기 중인 쿼리와 해당 세션 정보를 함께 조회합니다.

```sql
-- 락 대기 중인 세션의 상세 정보
SELECT
    a.pid,
    a.usename,
    a.query,
    l.locktype,
    l.relation::regclass,
    l.mode
FROM pg_stat_activity a
    INNER JOIN pg_locks l ON a.pid = l.pid
WHERE l.granted IS FALSE;
```

---

## Deadlock (교착 상태)

### Deadlock이란?
두 개 이상의 트랜잭션이 서로가 보유한 락을 기다리면서 무한 대기 상태에 빠지는 현상입니다.

- 트랜잭션 A: 행 1을 잠그고 행 2를 기다림
- 트랜잭션 B: 행 2를 잠그고 행 1을 기다림
- 결과: 두 트랜잭션 모두 진행 불가

### PostgreSQL의 Deadlock 처리
- PostgreSQL은 **데드락 자동 감지** 기능을 제공 (`deadlock_timeout` 설정, 기본값 1초)
- 감지 시 한 트랜잭션을 자동으로 종료시키고 **ERROR** 반환
- 종료된 트랜잭션은 롤백되며, 다른 트랜잭션은 진행 가능

### Deadlock 원인 파악
- PostgreSQL 로그에서 `deadlock detected` 메시지 확인
- 로그에는 데드락에 연루된 프로세스, 쿼리, 락 정보가 기록됨
- `log_lock_waits` 설정을 활성화하면 락 대기 상황도 로깅 가능

### Deadlock 방지 전략
1. **일관된 락 획득 순서**: 모든 트랜잭션이 동일한 순서로 테이블/행에 접근
2. **트랜잭션 최소화**: 트랜잭션 길이를 짧게 유지하여 락 보유 시간 단축
3. **적절한 인덱스 사용**: 불필요한 락 범위 축소
4. **재시도 로직 구현**: 애플리케이션 레벨에서 데드락 발생 시 재시도
