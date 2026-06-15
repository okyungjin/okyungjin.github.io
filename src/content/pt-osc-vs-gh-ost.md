---
title: "온라인 스키마 변경 도구 비교: pt-osc vs gh-ost"
description: 무중단으로 대용량 테이블의 DDL을 적용하는 pt-online-schema-change와 gh-ost의 동작 원리와 차이를 정리하고, Aurora MySQL에서 무엇을 택할지, 그리고 INT→BIGINT 실습까지 다룹니다.
pubDate: 2026-06-16
category: MySQL
tags:
  - mysql
  - online-ddl
  - aurora
---

대용량 테이블에 `ALTER`를 걸어야 하는 순간은 늘 부담스럽습니다. MySQL의 네이티브 Online DDL이 많이 좋아졌지만, 수억 건짜리 테이블의 컬럼 타입을 바꾸는 작업은 여전히 락과 복제 지연을 걱정하게 만듭니다. 이럴 때 쓰는 대표적인 무중단 스키마 변경 도구가 **pt-online-schema-change(pt-osc)** 와 **gh-ost** 입니다.

이 글에서는 두 도구의 동작 원리와 차이를 정리하고, Aurora MySQL 환경에서 무엇을 택해야 하는지, 그리고 `INT → BIGINT` 변경을 직접 돌려보는 실습까지 다룹니다.

## 왜 온라인 스키마 변경 도구가 필요한가

네이티브 `ALTER TABLE`은 대용량 테이블에서 여전히 세 가지 부담이 있습니다.

- **메타데이터 락(MDL)**: 일부 DDL은 작업 중 테이블에 락을 걸어 읽기/쓰기를 막습니다. 트래픽이 몰리는 시간대라면 그대로 장애로 번집니다.
- **복제 지연**: 대형 `ALTER`가 binlog를 통해 복제본에 그대로 전파되며 긴 replication lag을 유발합니다.
- **되돌리기 어려움**: 진행 중인 네이티브 DDL은 중간에 멈추거나 속도를 늦추기 어렵습니다. 한번 시작하면 끝까지 가야 합니다.

그래서 **원본 테이블을 직접 건드리지 않고**, 구조가 같은 빈(shadow) 테이블에 변경을 먼저 적용한 뒤 데이터를 점진적으로 복사하고, 마지막에 원자적으로 교체하는 도구를 씁니다. 그 대표가 pt-osc와 gh-ost입니다.

## 공통 골격: 새 테이블 + 점진 복사 + 원자적 교체

두 도구는 설계 철학이 다르지만 큰 그림은 같습니다.

1. 원본과 같은 구조의 **새 테이블을 생성**하고 여기에 변경(`ALTER`)을 적용
2. 기존 행을 **청크 단위로 점진 복사**
3. 복사가 진행되는 동안 원본에 들어온 변경분을 **새 테이블에 따라잡기**
4. `RENAME`으로 원본과 새 테이블을 **원자적으로 교체**

복사가 도는 동안에도 원본은 정상적으로 읽기/쓰기를 받고, 컷오버(교체) 순간에만 아주 짧게 락이 걸립니다. 갈리는 지점은 **3번 "따라잡기"를 어떻게 구현하느냐**입니다.

## pt-osc: 트리거 기반 동기화

Percona Toolkit의 도구로, 원본 테이블에 **트리거**를 걸어 복사가 진행되는 동안 들어오는 변경을 새 테이블로 실시간 반영합니다.

1. **빈 테이블 생성**: 원본과 같은 구조의 `_new` 테이블을 만들고 `ALTER`를 적용
2. **트리거 설치**: 원본에 `INSERT`/`UPDATE`/`DELETE` 트리거를 걸어 변경분을 새 테이블로 전달
3. **데이터 복사**: 기존 행을 청크 단위로 새 테이블에 복사 (부하를 보며 조절)
4. **원자적 교체**: `RENAME`으로 원본과 새 테이블을 교체하고 트리거 제거

#### 주의할 부담

트리거가 원본 테이블의 **모든 쓰기 경로에 얹혀** 동작합니다. 쓰기 부하가 높으면 트리거 자체가 오버헤드가 되고, 이미 트리거가 걸린 테이블에는 사용할 수 없습니다.

## gh-ost: binlog 기반 동기화

GitHub이 만든 도구로, 트리거 대신 **binlog(ROW 포맷)** 를 읽어 변경을 따라잡습니다. 원본 테이블에 트리거를 걸지 않는 'triggerless' 방식입니다.

1. **빈 테이블 생성**: 구조가 같은 ghost 테이블(`_gho`)을 만들고 `ALTER`를 적용
2. **binlog 구독**: 복제본인 척 binlog 스트림에 붙어 원본의 변경 이벤트를 수신
3. **복사 + 따라잡기**: 기존 행을 복사하면서 binlog로 들어온 변경을 ghost 테이블에 반영
4. **원자적 컷오버**: `RENAME`으로 교체. 진행 중 언제든 일시정지/속도조절 가능

#### 장점

원본의 쓰기 경로에 트리거가 끼지 않아 **부하 영향이 작습니다.** 변경 작업을 실시간으로 throttle/pause 할 수 있고, binlog를 별도 복제본에서 가져오게 분리해 원본 부하를 더 줄일 수도 있습니다.

## 공통점과 차이점

### 공통점

- **같은 골격**: 새 테이블 생성 → `ALTER` 적용 → 점진 복사 → 원자적 `RENAME` 교체
- **무중단**: 복사 중에도 원본은 정상 동작, 컷오버 순간만 짧게 락
- **부하 제어**: 복제 지연·서버 부하를 지표로 보며 복사 속도를 조절(throttle)
- **공통 제약**: 외래 키(FK)와 기존 트리거가 있는 테이블에는 제약이 크고, PK 또는 유니크 키가 반드시 있어야 함

### 차이점

| 항목 | pt-osc | gh-ost |
| --- | --- | --- |
| 변경 동기화 방식 | 트리거로 원본 쓰기를 가로채 반영 | binlog(ROW)를 읽어 반영 |
| 원본 부하 | 쓰기마다 트리거 실행, 부하 증가 | 트리거 없음, 부하 영향 작음 |
| 속도 조절·중단 | 제한적 (트리거는 끌 수 없음) | 실시간 throttle / pause / resume |
| 부하 분리 | 원본 서버에서만 동작 | 복제본 binlog에서 읽도록 분리 가능 |
| 필수 전제 | 특별한 binlog 설정 불필요 | `binlog_format=ROW` 필요 |
| 기존 트리거 테이블 | 사용 불가 | 사용 가능 (트리거 안 씀) |
| 생태계 | Percona Toolkit에 통합, 점검 옵션 다양 | 단일 목적 도구, 운영 관측성 우수 |

한 줄로 요약하면, **pt-osc는 원본에 트리거를 얹는 방식이라 단순하지만 부하에 민감하고, gh-ost는 binlog를 따라가는 방식이라 부하 영향이 작고 통제가 쉽습니다.**

## Aurora MySQL에서의 선택과 주의점

Aurora는 쓰기 부하가 주요 테이블에 몰리는 경우가 많아, 트리거 부하가 없는 **gh-ost가 대체로 더 안전**합니다. 다만 Aurora 특유의 설정 몇 가지를 반드시 짚어야 합니다.

### 가장 주의할 함정: 컷오버가 멈추는 문제

Aurora는 `aurora_enable_repl_bin_log_filtering`이 기본 활성화되어 있습니다. 이 상태에서는 gh-ost의 **컷오버 신호 이벤트가 binlog에서 필터링돼 무한 대기**에 빠집니다. 클러스터 파라미터 그룹에서 이 값을 `0`으로 변경해야 합니다.

```sql
-- 적용 전 현재 값 확인
SHOW VARIABLES LIKE 'aurora_enable_repl_bin_log_filtering';
SHOW VARIABLES LIKE 'binlog_format';   -- ROW 여야 함
```

> 단, Aurora MySQL 3에서는 이 파라미터가 아예 없고 필터링이 항상 켜져 있으니, 작업 전에 클러스터 버전을 먼저 확인해야 합니다.

### 마스터 자동 감지 이슈

Aurora의 reader(복제본)는 일반 MySQL처럼 binlog로 복제하지 않고, writer와 **스토리지를 공유**하는 방식으로 동작합니다. 즉 reader에는 gh-ost가 따라갈 **binlog 스트림이 없고**, binlog는 writer에서만 나옵니다.

그래서 gh-ost를 writer에 직접 붙여야 하며, 이때 `--allow-on-master`가 사실상 필수가 됩니다. 복제본으로 부하를 분리하려면 별도의 binlog 복제 클러스터를 따로 구성해야 합니다.

#### `--allow-on-master`는 왜 필요한가

gh-ost는 기본적으로 **복제본(replica)에 접속**해서 그 복제본의 binlog를 읽고, 실제 변경 적용과 컷오버만 마스터에서 수행하도록 설계돼 있습니다. binlog를 읽는 부하를 운영 마스터에서 떼어내 더 안전하게 진행하려는 의도입니다.

복제본 대신 마스터에 직접 붙어 binlog를 읽게 하려면, 사용자가 의도적으로 허용했다는 표시가 필요한데 그 승인 플래그가 `--allow-on-master`입니다. 없이 마스터로 붙으면 gh-ost가 안전을 위해 실행을 막습니다. 앞서 본 것처럼 Aurora의 reader에는 binlog가 없으므로, Aurora에서는 이 플래그를 거의 항상 쓰게 됩니다.

```bash
# Aurora에서 gh-ost 실행 시 권장 옵션
gh-ost ... --assume-rbr --allow-on-master --execute
```

pt-osc를 쓴다면 binlog 설정 부담은 없지만, 트리거 부하가 Aurora writer 인스턴스에 직접 실리는 점을 감수해야 합니다.

## 의사결정 가이드

**gh-ost를 선택**

- 운영 중 쓰기 부하가 높은 주요 테이블
- 변경 도중 속도 제어나 일시정지가 필요한 경우
- 이미 트리거가 걸린 테이블을 변경해야 하는 경우
- binlog ROW 포맷 사용이 가능한 환경

**pt-osc를 선택**

- binlog ROW 설정을 켤 수 없는 환경
- 여러 인스턴스를 일괄로 다뤄야 하는 경우
- Percona Toolkit 운영에 이미 익숙한 팀
- binlog 접근 권한 확보가 까다로운 경우

두 도구 모두 **FK나 트리거가 걸린 테이블**은 기본적으로 제약이 크니, 이 경우는 별도 검토가 필요합니다. 작업 전 공통 체크리스트는 다음과 같습니다.

1. 대상 테이블에 PK 또는 유니크 키가 있는지 확인
2. FK와 트리거 존재 여부 점검
3. 디스크 여유 확보 (작업 중 테이블이 일시적으로 2배)
4. 컷오버는 트래픽이 낮은 시간대 선택
5. 충분한 사전 리허설

## 직접 해보기: INT → BIGINT 실습

개념만으로는 감이 잘 안 오니, 같은 `ALTER`를 두 도구로 각각 돌려보겠습니다. **반드시 테스트 DB에서만** 진행하고 운영 환경에는 바로 적용하지 않습니다.

### 실습 환경 준비

```bash
# pt-osc (Percona Toolkit)
sudo apt install percona-toolkit   # Ubuntu / Debian
brew install percona-toolkit       # macOS
pt-online-schema-change --version

# gh-ost
brew install gh-ost                # macOS
# Linux는 GitHub Releases에서 바이너리를 받아 PATH에 둡니다
gh-ost --version
```

gh-ost 실습에는 binlog 관련 권한(`REPLICATION SLAVE` 등)도 필요합니다.

### 실습용 테이블과 데이터 만들기

변경에 시간이 걸려야 진행 과정을 관찰할 수 있으므로, 행을 넉넉히 채웁니다.

```sql
CREATE DATABASE osc_lab;
USE osc_lab;

CREATE TABLE members (
  id         INT AUTO_INCREMENT PRIMARY KEY,   -- INT(약 21억)으로 시작
  name       VARCHAR(100),
  email      VARCHAR(200),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 재귀 CTE로 더미 데이터 대량 삽입 (수십만 건)
INSERT INTO members (name, email)
WITH RECURSIVE seq AS (
  SELECT 1 n UNION ALL SELECT n + 1 FROM seq WHERE n < 500000
)
SELECT CONCAT('user', n), CONCAT('user', n, '@test.com') FROM seq;
```

이번 실습의 목표 변경은 PK인 `id`를 `INT`에서 `BIGINT`로 바꾸는 것입니다. INT는 약 21억에서 한계에 닿으므로 오버플로 전에 키 타입을 키우는, 실무에서 흔한 시나리오입니다.

> **왜 컬럼 추가가 아니라 타입 변경인가?** MySQL 8.0에서 `ADD COLUMN`은 대부분 INSTANT로 즉시 끝나 관찰할 게 없습니다. 반면 `INT → BIGINT` 같은 타입 변경은 테이블 전체를 다시 써야 해서, 두 도구가 실제로 복사하며 동작하는 과정을 관찰하기에 알맞습니다.

### pt-osc로 변경하기

먼저 `--dry-run`으로 무엇을 할지 확인하고, 문제없으면 `--execute`로 실제 적용합니다.

```bash
# 1단계: dry-run (실제 변경 없이 계획만 확인)
pt-online-schema-change \
  --alter "MODIFY id BIGINT NOT NULL AUTO_INCREMENT" \
  D=osc_lab,t=members \
  --dry-run

# 2단계: 실제 실행
pt-online-schema-change \
  --alter "MODIFY id BIGINT NOT NULL AUTO_INCREMENT" \
  D=osc_lab,t=members \
  --execute
```

자주 쓰는 옵션으로 `--max-load`(부하가 높으면 잠시 멈춤), `--critical-load`(위험 수준이면 즉시 중단), `--chunk-size`(한 번에 복사할 행 수)가 있습니다.

### pt-osc가 만든 흔적 관찰하기

실행 도중 **다른 세션**을 하나 더 열어 두고 아래를 조회하면 동작이 보입니다.

```sql
-- 작업용 새 테이블(_members_new)이 잠시 생겼다 사라진다
SHOW TABLES LIKE '%members%';

-- 원본에 걸린 트리거 확인 (이게 pt-osc의 동작 원리)
SHOW TRIGGERS FROM osc_lab;

-- 작업 중에도 원본은 정상 동작하는지 확인
SELECT COUNT(*) FROM members;
INSERT INTO members (name, email) VALUES ('live', 'live@test.com');
```

작업 중 INSERT한 `live` 행이 변경 완료 후에도 새 테이블에 그대로 남아 있다면, 트리거가 실시간 변경을 새 테이블로 옮겼다는 증거입니다. 쓰기를 계속 발생시키면서 실행해 보면 트리거가 쓰기마다 함께 도는 부담도 체감할 수 있습니다.

### gh-ost로 같은 변경 해보기

```bash
gh-ost \
  --host=127.0.0.1 --port=3306 \
  --user="osc_user" --password="****" \
  --database="osc_lab" --table="members" \
  --alter="MODIFY id BIGINT NOT NULL AUTO_INCREMENT" \
  --allow-on-master \
  --initially-drop-ghost-table \
  --execute
```

전제 조건은 `binlog_format = ROW`, binlog 읽기 권한, 그리고 원본에 PK 또는 유니크 키가 있어야 한다는 점입니다.

### gh-ost 동작 관찰과 실시간 제어

gh-ost는 실행 중 소켓으로 명령을 받아 속도를 조절하거나 멈출 수 있습니다. 이게 큰 장점이니 꼭 해봅니다.

```bash
# 실행 중 일시정지 (소켓 파일 경로는 실행 로그에 표시됨)
echo throttle | nc -U /tmp/gh-ost.osc_lab.members.sock

# 다시 진행
echo no-throttle | nc -U /tmp/gh-ost.osc_lab.members.sock

# 현재 상태 확인
echo status | nc -U /tmp/gh-ost.osc_lab.members.sock
```

다른 세션에서 `SHOW TABLES LIKE '%members%'`를 조회하면 `_members_gho` ghost 테이블이 채워지는 과정을 볼 수 있습니다. throttle를 걸면 복사가 멈추지만 원본은 계속 정상 동작하고, `SHOW TRIGGERS`로 확인하면 원본에 **트리거가 없다는 점**까지 pt-osc와 비교됩니다.

### 마무리 점검과 정리

```sql
-- id 타입이 BIGINT로 바뀌었는지 확인
DESC members;

-- 변경 전후 건수가 일치하는지, 작업 중 넣은 행이 유실 없이 남았는지 확인
SELECT COUNT(*) FROM members;
```

뒷정리로 남은 작업 테이블(`_new`, `_gho`, `_del` 등)을 삭제하고, 잔여 트리거가 있는지 `SHOW TRIGGERS`로 확인한 뒤, 실습 스키마(`osc_lab`)를 통째로 DROP합니다. 여유가 있다면 **FK나 트리거가 이미 있는 테이블**로 같은 실습을 시도해, 두 도구가 각각 어떻게 막히거나 우회하는지 확인해 보면 좋습니다.

## 정리

- 두 도구 모두 "새 테이블 + 점진 복사 + 원자적 교체"라는 같은 골격을 따릅니다.
- 차이는 **따라잡기 방식**: pt-osc는 트리거, gh-ost는 binlog.
- 트리거 부하가 없고 실시간 제어가 가능한 **gh-ost가 운영 부하 측면에서 유리**하지만, `binlog_format=ROW`가 전제입니다.
- **Aurora MySQL이라면 기본은 gh-ost**, 단 `aurora_enable_repl_bin_log_filtering=0`과 `--allow-on-master`는 반드시 챙겨야 합니다.

## 참고 자료

- [Percona Toolkit: pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [GitHub: gh-ost](https://github.com/github/gh-ost)
- [gh-ost: RDS/Aurora 문서](https://github.com/github/gh-ost/blob/master/doc/rds.md)
- [gh-ost: Requirements and Limitations](https://github.com/github/gh-ost/blob/master/doc/requirements-and-limitations.md)
- [MySQL 8.0 Reference: Online DDL Operations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)
