---
author: ["Paul Wong"]
title: "PostgreSQL Tricks"
date: "2019-08-12"
description: "PostgreSQL Tips and Tricks"
summary: "This post has some useful PostgreSQL tips and tricks"
tags: ["postgresql", "sql", "snippet"]
categories: ["postgresql", "snippet"]
ShowToc: true
TocOpen: true
---

### Delete database while in use

```sql
UPDATE
  pg_database
SET
  datallowconn = 'false'
WHERE
  datname = 'my-db';
SELECT
  pg_terminate_backend(pg_stat_activity.pid)
FROM
  pg_stat_activity
WHERE
  pg_stat_activity.datname = 'my-db';
DROP
  DATABASE "my-db";
```

### Delete all tables like

```sql
SELECT
  'DROP TABLE ' || table_name || ';'
FROM
  INFORMATION_SCHEMA.TABLES
WHERE
  TABLE_NAME ILIKE '%PWONG%';
```

### Find all tables with the given column name

#### One Column

```sql
select
  table_name
from
  information_schema.columns
where
  column_name like '%user_id%'
  and table_name ! ~ '[0-9]';
```

#### Mulitple Columns

```sql
select
  table_name
from
  information_schema.columns
where
  column_name = 'building_id'
  and table_name in (
    select
      table_name
    from
      information_schema.columns
    where
      column_name = 'portfolio_id'
  );
```

### Kill a query

#### Kindly

```sql
SELECT
  pg_cancel_backend(< pid of the process >);
```

#### With predjudice

```sql
SELECT
  pg_terminate_backend(< pid of the process >);
```

### List long running queries

```sql
SELECT
  pid,
  NOW() - pg_stat_activity.query_start AS duration,
  query,
  state
FROM
  pg_stat_activity
WHERE
  (
    NOW() - pg_stat_activity.query_start
  ) > INTERVAL '5 minutes';
```

### Look for blocking tables

```sql
rollback;
select
  *
from
  public.pg_stat_activity
where
  pid in (
    select
      pid
    from
      pg_locks
    where
      relation = 'file_infos'::regclass
  );
```

### Look for blocking jobs

```sql
ROLLBACK;
SELECT
  *
FROM
  public.pg_stat_activity;
SELECT
  blocking.pid AS blocking_pid,
  statblocking.current_query AS blocking_query,
  statblocking.backend_start AS blocking_start,
  statblocking.client_addr AS blocking_ip,
  blocked.pid AS blocked_pid,
  statblocked.current_query AS blocked_query
FROM
  pg_locks AS blocked
  JOIN public.pg_stat_activity AS statblocked ON (statblocked.procpid = locked.pid) LEFT
  JOIN pg_locks AS blocking ON (
    blocked.pid ! = blocking.pid
    AND CASE WHEN blocked.locktype = 'transactionid' THEN blocked.transactionid = blocking.transactionid WHEN blocked.locktype = 'relation' THEN blocked.relation = blocking.relation WHEN blocked.locktype = 'page' THEN blocked.relation = blocking.relation
    AND blocked.page = blocking.page WHEN blocked.locktype = 'tuple' THEN blocked.relation = blocking.relation
    AND blocked.tuple = blocking.tuple
    AND blocked.page = blocking.page WHEN blocked.locktype = 'virtualxid' THEN blocked.virtualxid = blocking.virtualxid WHEN blocked.locktype = 'object' THEN blocked.classid = blocking.classid
    AND blocked.objid = blocking.objid WHEN blocked.locktype = 'advisory' THEN blocked.database = blocking.database
    AND blocked.classid = blocking.classid
    AND blocked.objid = blocking.objid
    AND blocked.objsubid = blocking.objsubid ELSE FALSE END
  ) LEFT
  JOIN public.pg_stat_activity AS statblocking ON (
    statblocking.procpid = blocking.pid
  )
WHERE
  NOT blocked.granted; ROLLBACK;

SELECT
  *
FROM
  public.pg_stat_activity;
SELECT
  blocking.pid AS blocking_pid,
  statblocking.current_query AS blocking_query,
  statblocking.backend_start AS blocking_start,
  statblocking.client_addr AS blocking_ip,
  blocked.pid AS blocked_pid,
  statblocked.current_query AS blocked_query
FROM
  pg_locks AS blocked
  JOIN public.pg_stat_activity AS statblocked ON (statblocked.procpid = locked.pid) LEFT
  JOIN pg_locks AS blocking ON (
    blocked.pid ! = blocking.pid
    AND CASE WHEN blocked.locktype = 'transactionid' THEN blocked.transactionid = blocking.transactionid WHEN blocked.locktype = 'relation' THEN blocked.relation = blocking.relation WHEN blocked.locktype = 'page' THEN blocked.relation = blocking.relation
    AND blocked.page = blocking.page WHEN blocked.locktype = 'tuple' THEN blocked.relation = blocking.relation
    AND blocked.tuple = blocking.tuple
    AND blocked.page = blocking.page WHEN blocked.locktype = 'virtualxid' THEN blocked.virtualxid = blocking.virtualxid WHEN blocked.locktype = 'object' THEN blocked.classid = blocking.classid
    AND blocked.objid = blocking.objid WHEN blocked.locktype = 'advisory' THEN blocked.database = blocking.database
    AND blocked.classid = blocking.classid
    AND blocked.objid = blocking.objid
    AND blocked.objsubid = blocking.objsubid ELSE FALSE END
  ) LEFT
  JOIN public.pg_stat_activity AS statblocking ON (
    statblocking.procpid = blocking.pid
  )
WHERE
  NOT blocked.granted;
```

### Check for bloat

#### Simple

```sql
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  (
    n_dead_tup / n_live_tup::REAL * 100
  )::INTEGER AS bloat
FROM
  pg_stat_user_tables
WHERE
  n_live_tup > 0
```

#### Another

```sql
SELECT
  CURRENT_DATABASE(),
  schemaname,
  tablename,

  /*reltuples::bigint, relpages::bigint, otta,*/
  ROUND(
    (
      CASE WHEN otta = 0 THEN 0.0 ELSE sml.relpages::FLOAT / otta END
    )::NUMERIC,
    1
  ) AS tbloat,
  CASE WHEN relpages < otta THEN 0 ELSE bs *(sml.relpages - otta)::BIGINT END AS wastedbytes,
  iname,

  /*ituples::bigint, ipages::bigint, iotta,*/
  ROUND(
    (
      CASE WHEN iotta = 0
      OR ipages = 0 THEN 0.0 ELSE ipages::FLOAT / iotta END
    )::NUMERIC,
    1
  ) AS ibloat,
  CASE WHEN ipages < iotta THEN 0 ELSE bs *(ipages - iotta) END AS wastedibytes
FROM
  (
    SELECT
      schemaname,
      tablename,
      cc.reltuples,
      cc.relpages,
      bs,
      CEIL(
        (
          cc.reltuples *(
            (
              datahdr + ma - (
                CASE WHEN datahdr % ma = 0 THEN ma ELSE datahdr % ma END
              )
            )+ nullhdr2 + 4
          )
        )/(bs - 20::FLOAT)
      ) AS otta,
      COALESCE(c2.relname, '?') AS iname,
      COALESCE(c2.reltuples, 0) AS ituples,
      COALESCE(c2.relpages, 0) AS ipages,
      COALESCE(
        CEIL(
          (
            c2.reltuples *(datahdr - 12)
          )/(bs - 20::FLOAT)
        ),
        0
      ) AS iotta -- very rough approximation, assumes all cols
    FROM
      (
        SELECT
          ma,
          bs,
          schemaname,
          tablename,
          (
            datawidth +(
              hdr + ma -(
                CASE WHEN hdr % ma = 0 THEN ma ELSE hdr % ma END
              )
            )
          )::NUMERIC AS datahdr,
          (
            maxfracsum *(
              nullhdr + ma -(
                CASE WHEN nullhdr % ma = 0 THEN ma ELSE nullhdr % ma END
              )
            )
          ) AS nullhdr2
        FROM
          (
            SELECT
              schemaname,
              tablename,
              hdr,
              ma,
              bs,
              SUM(
                (1 - null_frac)* avg_width
              ) AS datawidth,
              MAX(null_frac) AS maxfracsum,
              hdr +(
                SELECT
                  1 + COUNT(*)/ 8
                FROM
                  pg_stats s2
                WHERE
                  null_frac <> 0
                  AND s2.schemaname = s.schemaname
                  AND s2.tablename = s.tablename
              ) AS nullhdr
            FROM
              pg_stats s,
              (
                SELECT
                  (
                    SELECT
                      CURRENT_SETTING('block_size')::NUMERIC
                  ) AS bs,
                  CASE WHEN SUBSTRING(v, 12,3) IN ('8.0', '8.1', '8.2') THEN 27 ELSE 23 END AS hdr,
                  CASE WHEN v ~ 'mingw32' THEN 8 ELSE 4 END AS ma
                FROM
                  (
                    SELECT
                      VERSION() AS v
                  ) AS foo
              ) AS constants
            GROUP BY
              1,2,
              3,4,
              5
          ) AS foo
      ) AS rs
      JOIN pg_class cc ON cc.relname = rs.tablename
      JOIN pg_namespace nn ON cc.relnamespace = nn.oid
      AND nn.nspname = rs.schemaname
      AND nn.nspname <> 'information_schema'
      LEFT JOIN pg_index i ON indrelid = cc.oid
      LEFT JOIN pg_class c2 ON c2.oid = i.indexrelid
  ) AS sml
ORDER BY
  wastedbytes DESC;
```

#### Yet Another One

```sql
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  TRUNC(
    100 * n_dead_tup /(n_live_tup + 1)
  )::FLOAT "ratio%",
  last_autovacuum
FROM
  pg_stat_all_tables
WHERE
  schemaname IN ('public', 'booking_service')
ORDER BY
  n_dead_tup / (
    n_live_tup * CURRENT_SETTING(
      'autovacuum_vacuum_scale_factor'
    )::FLOAT8 + CURRENT_SETTING('autovacuum_vacuum_threshold')::FLOAT8
  ) DESC
LIMIT
  10;
```

### List table sizes

```sql
\dt+
```

### Function to search all columns in a database

```sql
CREATE
OR REPLACE FUNCTION search_columns(
  needle text, haystack_tables name[] default '{}',
  haystack_schema name[] default '{}'
) RETURNS table(
  schemaname text, tablename text, columnname text,
  rowctid text
) AS $$ begin FOR schemaname,
tablename,
columnname IN
SELECT
  c.table_schema,
  c.table_name,
  c.column_name
FROM
  information_schema.columns c
  JOIN information_schema.tables t ON (
    t.table_name = c.table_name
    AND t.table_schema = c.table_schema
  )
  JOIN information_schema.table_privileges p ON (
    t.table_name = p.table_name
    AND t.table_schema = p.table_schema
    AND p.privilege_type = 'SELECT'
  )
  JOIN information_schema.schemata s ON (s.schema_name = t.table_schema)
WHERE
  (
    c.table_name = ANY(haystack_tables)
    OR haystack_tables = '{}'
  )
  AND (
    c.table_schema = ANY(haystack_schema)
    OR haystack_schema = '{}'
  )
  AND t.table_type = 'BASE TABLE' LOOP FOR rowctid IN EXECUTE format(
    'SELECT ctid FROM %I.%I WHERE cast(%I as text)=%L',
    schemaname, tablename, columnname,
    needle
  ) LOOP -- uncomment next line to get some progress report
  -- RAISE NOTICE 'hit in %.%', schemaname, tablename;
  RETURN NEXT; END LOOP; END LOOP; END; $$ language plpgsql;
```

#### Use

```sql
select
  *
from
  search_columns('where_does_this_show_up');
```

### Schema sizes

```sql
SELECT
  schema_name,
  pg_size_pretty (
    sum(table_size)::bigint
  ),
  (
    sum(table_size) / pg_database_size(
      current_database()
    )
  ) * 100
FROM
  (
    SELECT
      pg_catalog.pg_namespace.nspname as schema_name,
      pg_relation_size(pg_catalog.pg_class.oid) as table_size
    FROM
      pg_catalog.pg_class
      JOIN pg_catalog.pg_namespace ON relnamespace = pg_catalog.pg_namespace.oid
  ) t
GROUP BY
  schema_name
ORDER BY
  schema_name;
```

### Show rights on objects

#### s == sequence, t == table

```sql
SELECT
  relname,
  relacl
FROM
  pg_class
WHERE
  relkind = 'S' -- c,t,S,i,m,r,v
  AND relacl IS NOT NULL
  AND relnamespace IN (
    SELECT
      OID
    FROM
      pg_namespace
    WHERE
      nspname NOT LIKE 'pg_%'
      AND nspname != 'information_schema'
      --AND relname like 'customer_customer%'
  );
```

### Number of open connections

```sql
SELECT
  *
FROM
  (
    SELECT
      COUNT(*) used
    FROM
      pg_stat_activity
  ) q1,
  (
    SELECT
      setting::INT res_for_super
    FROM
      pg_settings
    WHERE
      NAME = $$superuser_reserved_connections$$
  ) q2,
  (
    SELECT
      setting::INT max_conn
    FROM
      pg_settings
    WHERE
      NAME = $$max_connections$$
  ) q3;
```

### Check date gaps

```sql
WITH (
  SELECT
    COUNT(*) CNT,
    UTC_AIRING_DATE AS D,
    ROW_NUMBER() OVER(
      ORDER BY
        UTC_AIRING_DATE
    ) i
  FROM
    events
  GROUP BY
    UTC_AIRING_DATE
) AS T
SELECT
  'events' AS table_name,
  MIN(D) FROM_DATE,
  MAX(D) TO_DATE,
  MIN(CNT),
  MAX(CNT),
  AVG(CNT),
  'UTC_AIRING_DATE' AS SCAN
FROM
  T
GROUP BY
  DATEADD(DAY, - i, D)
ORDER BY
  FROM_DATE
```

### Overlapping Date Windows

```sql
CREATE TABLE el (
  id INT GENERATED ALWAYS AS IDENTITY,
  member_id INT,
  plan_id INT,
  start_date DATE,
  end_date DATE
)


INSERT INTO el (
  member_id, plan_id, start_date, end_date
)
VALUES
  (1, 1, '2024-01-01', '2024-03-31'),
  (2, 1, '2024-01-01', '2024-03-31'),
  (2, 1, '2024-01-01', '2024-03-31'),
  (3, 1, '2024-04-01', '2024-06-30'),
  (3, 1, '2024-05-01', '2024-06-30'),
  (3, 1, '2024-01-01', '2024-03-31'),
  (4, 1, '2024-01-01', '2024-03-31'),
  (4, 1, '2024-01-01', '2024-06-30'),
  (5, 1, '2024-01-01', '2024-03-31');


SELECT
  el1.member_id,
  el1.start_date,
  el1.end_date,
  (
    CASE WHEN EXISTS (
      SELECT
        *
      FROM
        el el2
      WHERE
        el1.id <> el2.id
        AND el1.member_id = el2.member_id
        AND el1.start_date = el2.start_date
        AND el1.end_date = el2.end_date
    ) THEN TRUE ELSE FALSE END
  ) AS is_duplicate,
  (
    CASE WHEN EXISTS (
      SELECT
        *
      FROM
        el el2
      WHERE
        el1.id <> el2.id
        AND el1.member_id = el2.member_id
        AND (el1.start_date, el1.end_date) OVERLAPS (el2.start_date, el2.end_date)
    ) THEN TRUE ELSE FALSE END
  ) AS has_overlap
FROM
  el el1
ORDER BY
  el1.member_id
```

Output
| member_id | start_date | end_date | is_duplicate | has_overlap |
| --- | --- | --- | --- | --- |
| 1 | 2024-01-01 | 2024-03-31 | false | false |
| 2 | 2024-01-01 | 2024-03-31 | true | true |
| 2 | 2024-01-01 | 2024-03-31 | true | true |
| 3 | 2024-04-01 | 2024-06-30 | false | true |
| 3 | 2024-05-01 | 2024-06-30 | false | true |
| 3 | 2024-01-01 | 2024-03-31 | false | false |
| 4 | 2024-01-01 | 2024-03-31 | false | true |
| 4 | 2024-01-01 | 2024-06-30 | false | true |
| 5 | 2024-01-01 | 2024-03-31 | false | false |

### Cool Functions

- pg_size_pretty
