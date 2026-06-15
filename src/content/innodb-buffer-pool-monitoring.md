---
title: InnoDB 버퍼 풀 적재 내용 확인
description: information_schema의 innodb_cached_indexes를 활용해 InnoDB 버퍼 풀에 어떤 인덱스가 얼마나 적재되었는지, 적재 비율은 어떤지 조회하는 방법을 정리합니다.
pubDate: 2025-03-12
tags:
  - mysql
  - innodb
---

## InnoDB 버퍼 풀 적재 현황 조회 쿼리

- `innodb_cached_indexes` 테이블은 각 인덱스별로 버퍼 풀에 캐시된 페이지 개수(n_cached_pages)를 보여줍니다.
- index_id만 제공되므로 `innodb_indexes`, `innodb_tables`와 조인해서 실제 테이블명과 인덱스명을 확인할 수 있습니다.
- MySQL 5.6의 innodb_buffer_page는 버퍼 풀을 직접 스캔해서 성능 영향이 있었지만, 8.0의 `innodb_cached_indexes`는 집계 정보만 제공해 조회 부담이 적습니다.

```sql
SELECT it.name AS table_name
	, ii.name AS index_name
	, ici.n_cached_pages
FROM information_schema.innodb_cached_indexes ici
	INNER JOIN information_schema.innodb_indexes ii ON ici.index_id = ii.index_id
	INNER JOIN information_schema.innodb_tables it ON ii.table_id = it.table_id
WHERE it.name = CONCAT('데이터베이스명', '/', '테이블명');
```


## InnoDB 버퍼 풀 적재 비율 확인 쿼리

- `information_schema.innodb_cached_indexes` 테이블로 현재 버퍼 풀에 캐시된 페이지 수를 구하고, `information_schema.tables`의 데이터 크기로 전체 페이지 수를 추산하여 적재 비율을 확인할 수 있습니다.
- MySQL은 인덱스별 전체 페이지 개수를 직접 제공하지 않습니다. `information_schema.tables`의 data_length와 index_length를 합산하고, 페이지 크기(`@@innodb_page_size`, 기본 16KB)로 나누어 전체 페이지 수를 계산합니다.

```sql
SELECT
    -- 버퍼 풀에 적재된 총 페이지 수
    (SELECT SUM(ici.n_cached_pages)
     FROM information_schema.innodb_tables it
     INNER JOIN information_schema.innodb_indexes ii ON ii.table_id = it.table_id
     INNER JOIN information_schema.innodb_cached_indexes ici ON ici.index_id = ii.index_id
     WHERE it.name = CONCAT('데이터베이스명', '/', '테이블명')) AS cached_pages

    -- 테이블의 전체 예상 페이지 수
    , ROUND((t.data_length + t.index_length - t.data_free) / @@innodb_page_size) AS total_pages

    -- 적재 비율 (%)
    , ROUND(
        (SELECT SUM(ici.n_cached_pages)
         FROM information_schema.innodb_tables it
         INNER JOIN information_schema.innodb_indexes ii ON ii.table_id = it.table_id
         INNER JOIN information_schema.innodb_cached_indexes ici ON ici.index_id = ii.index_id
         WHERE it.name = CONCAT('데이터베이스명', '/', '테이블명'))
        /
        ((t.data_length + t.index_length - t.data_free) / @@innodb_page_size) * 100
    , 2) AS cached_ratio_percent
FROM information_schema.tables t
WHERE t.table_schema = '데이터베이스명'
  AND t.table_name = '테이블명';
```
