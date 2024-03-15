# Configuring `autovacuum`

The default configs for vacuuming work fine for some time but afterwards can cause millions of dead tuples to accumulate. Configuring vacuuming is required so it doesn’t only kick in once a month for certain tables. A lack of vacuuming causes the following problems:

- It messes up the stats for query optimization.
- Postgres duplicates rows for reading and writing in order to allow for MVCC. If vacuum isn’t run, hint bits aren’t set which effectively make reads as expensive as writes since the DB doesn’t know how to efficiently lock the row.

## Investigating Vacuuming

- Some queries in this section will return incorrect results if the stats aren’t up-to-date. You can check if the stats are up-to-date using the following query:
    
    ```sql
    SELECT
        datname,
        stats_reset 
    FROM
        pg_stat_database 
    WHERE
        datname = 'postgres';
    ```
    
- Finding the slow queries (these results can be compared with the ones displayed on RDS performance insights):
    
    ```sql
    SELECT
        query,
        round(total_time::NUMERIC, 2) AS total_time,
        calls,
        round(mean_time::NUMERIC, 2) AS mean,
        round( ( 100 * total_time / SUM(total_time::NUMERIC) OVER () )::NUMERIC, 2 ) AS percentage_overall 
    FROM
        pg_stat_statements 
    ORDER BY
        total_time DESC LIMIT 20;
    ```
    
- Finding which queries are causing table scans:
    
    ```sql
    SELECT
        schemaname,
        relname,
        seq_scan,
        seq_tup_read,
        idx_scan,
        seq_tup_read / seq_scan AS AVG 
    FROM
        pg_stat_user_tables 
    WHERE
        seq_scan > 0 
    ORDER BY
        seq_tup_read DESC;
    ```
    
- Checking when each table was last vacuumed and analyzed, and whether they’re correlated to which of our DB reads are slowest:
    
    ```sql
    SELECT
        schemaname,
        relname,
        n_dead_tup,
        n_live_tup,
        round(n_dead_tup::FLOAT / n_live_tup::FLOAT*100) dead_pct,
        autovacuum_count,
        last_vacuum,
        last_autovacuum,
        last_autoanalyze,
        last_analyze 
    FROM
        pg_stat_all_tables 
    WHERE
        n_live_tup > 0;
    ```
    
- To figure out the types of DMLs done on a table, use the following query:
    
    ```sql
    SELECT
        n_tup_ins AS "inserts",
        n_tup_upd AS "updates",
        n_tup_del AS "deletes",
        n_live_tup AS "live_tuples",
        n_dead_tup AS "dead_tuples" 
    FROM
        pg_stat_user_tables 
    WHERE
        schemaname = 'my_schema' 
        AND relname = 'my_table';
    ```
    
- Vacuum can’t run if there are long running queries. Check if that’s the case using:
    
    ```sql
    SELECT
        pid,
        age(backend_xid) AS age_in_xids,
        now () - xact_start AS xact_age,
        now () - query_start AS query_age,
        state,
        query 
    FROM
        pg_stat_activity 
    WHERE
        state != 'idle' 
    ORDER BY 2 DESC
    LIMIT 10;
    ```
    
- Vacuum can’t run if there are uncommitted prepared statements. Check if that’s the case using:
    
    ```sql
    SELECT
        gid,
        prepared,
        owner,
        database,
        transaction 
    FROM
        pg_prepared_xacts 
    ORDER BY
        age(transaction) DESC;
    ```
    
    Similarly, check for unused replication slots:
    
    ```sql
    SELECT
        slot_name,
        slot_type,
        database,
        xmin 
    FROM
        pg_replication_slots 
    ORDER BY
        age(xmin) DESC;
    ```
    
- Here’s how to see which tables are going to be automatically vacuumed:
    
    ```sql
    WITH vbt AS 
    (
        SELECT
            setting AS autovacuum_vacuum_threshold 
        FROM
            pg_settings 
        WHERE
            name = 'autovacuum_vacuum_threshold'
    )
    ,
    vsf AS 
    (
        SELECT
            setting AS autovacuum_vacuum_scale_factor 
        FROM
            pg_settings 
        WHERE
            name = 'autovacuum_vacuum_scale_factor'
    )
    ,
    fma AS 
    (
        SELECT
            setting AS autovacuum_freeze_max_age 
        FROM
            pg_settings 
        WHERE
            name = 'autovacuum_freeze_max_age'
    )
    ,
    sto AS 
    (
        SELECT
            opt_oid,
            split_part(setting, '=', 1) AS param,
            split_part(setting, '=', 2) AS VALUE 
        FROM
            (
                SELECT
                    oid opt_oid,
                    UNNEST(reloptions) setting 
                FROM
                    pg_class
            )
            opt
    )
    SELECT
        '"' || ns.nspname || '"."' || c.relname || '"' AS relation,
        pg_size_pretty(pg_table_size(c.oid)) AS table_size,
        age(relfrozenxid) AS xid_age,
        COALESCE(cfma.VALUE::FLOAT, autovacuum_freeze_max_age::FLOAT) autovacuum_freeze_max_age,
        (
            COALESCE(cvbt.VALUE::FLOAT, autovacuum_vacuum_threshold::FLOAT) + COALESCE(cvsf.VALUE::FLOAT, autovacuum_vacuum_scale_factor::FLOAT) * c.reltuples
        )
        AS autovacuum_vacuum_tuples,
        n_dead_tup AS dead_tuples 
    FROM
        pg_class c 
        JOIN
            pg_namespace ns 
            ON ns.oid = c.relnamespace 
        JOIN
            pg_stat_all_tables stat 
            ON stat.relid = c.oid 
        JOIN
            vbt 
            ON (1 = 1) 
        JOIN
            vsf 
            ON (1 = 1) 
        JOIN
            fma 
            ON (1 = 1) 
        LEFT JOIN
            sto cvbt 
            ON cvbt.param = 'autovacuum_vacuum_threshold' 
            AND c.oid = cvbt.opt_oid 
        LEFT JOIN
            sto cvsf 
            ON cvsf.param = 'autovacuum_vacuum_scale_factor' 
            AND c.oid = cvsf.opt_oid 
        LEFT JOIN
            sto cfma 
            ON cfma.param = 'autovacuum_freeze_max_age' 
            AND c.oid = cfma.opt_oid 
    WHERE
        c.relkind = 'r' 
        AND nspname <> 'pg_catalog' 
        AND 
        (
            age(relfrozenxid) >= COALESCE(cfma.VALUE::FLOAT, autovacuum_freeze_max_age::FLOAT) 
            OR COALESCE(cvbt.VALUE::FLOAT, autovacuum_vacuum_threshold::FLOAT) + COALESCE(cvsf.VALUE::FLOAT, autovacuum_vacuum_scale_factor::FLOAT) * c.reltuples <= n_dead_tup
        )
    ORDER BY
        age(relfrozenxid) DESC LIMIT 50;
    ```
    
- Here’s how to see which tables are currently being automatically vacuumed:
    
    ```sql
    SELECT
        datname,
        usename,
        pid,
        state,
        wait_event,
        CURRENT_TIMESTAMP - xact_start AS xact_runtime,
        query 
    FROM
        pg_stat_activity 
    WHERE
        UPPER(query) LIKE '%VACUUM%' 
    ORDER BY
        xact_start;
    ```
    
- Here’s how to check if the indexes are bigger than their tables (too many indices can cause performance to suffer):
    
    ```sql
    SELECT
        pg_size_pretty(pg_relation_size('pgbench_accounts'));
    SELECT
        pg_size_pretty(pg_indexes_size('pgbench_accounts'));
    ```
    
- Here’s how to check for unused indexes:
    
    ```sql
    SELECT
        * 
    FROM
        pg_stat_user_indexes 
    WHERE
        relname = 'pgbench_accounts' 
    ORDER BY
        idx_scan DESC;
    SELECT
        schemaname,
        relname,
        indexrelname,
        idx_scan 
    FROM
        pg_stat_user_indexes 
    WHERE
        relname = 'pgbench_accounts' 
    ORDER BY
        idx_scan DESC;
    ```
    

## Implementing Vacuuming

- Tune certain tables. For example, if a table receives 100,000 inserts a day, and a negligible number of updates/deletes, then we can do something like
    
    ```sql
    ALTER TABLE my_table SET (autovacuum_freeze_max_age = 100000);
    ```
    
- Part of `autovacuum` is to `autoanalyze`. Consider tuning that parameter separately such as:
    
    ```sql
    ALTER TABLE my_table SET (autovacuum_analyze_scale_factor = 0.02);
    ```
    
    or:
    
    ```sql
    ALTER TABLE my_table SET (
    	autovacuum_analyze_scale_factor = 0,
    	autovacuum_analyze_threshold = 1000000
    );
    ```
    

## Optimizing Vacuuming

- Set `autovacuum` to run at a time when there isn’t peak usage.
- Configure the `autovacuum_work_mem` based on the available memory on the system, and how long vacuuming takes.
- Consider `pg_repack` (no locking!) instead of `VACUUM FULL` (locks the table for hours).
- Similarly, consider `REINDEX` if the bloat is greater than 20% on a table.