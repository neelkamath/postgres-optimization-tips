# Optimizing at the DB Level

These tips are for optimizations such as tuning Postgres configs.

## Configs

- [Configure `autovacuum`](autovacuum.md)
- Checking cache hit ratio:
    
    ```sql
    SELECT 
    	(
    		sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read))
    	) AS ratio
    FROM pg_statio_user_tables;
    ```
    
- Optimize shared buffers. For example:
    
    ```sql
    SHOW shared_buffers;
    
    ALTER SYSTEM SET shared_buffers = '1GB';
    ```
    

## Connections

- Reuse DB connections. For example, don’t reconnect to the DB across Lambda invocations, and indexer spins.
- Use a proxy to pool DB connections such as RDS Proxy for environments with a massive number of DB connections such as a popular API running on Lambda.

## Hardware

- Increase CPU allocation.
- Change the CPU type. For example, use one with a bigger L2 cache.
- Increase memory allocation.
- Change the memory type. For example, DDR4 RAM is faster than DDR3 RAM.
- Change the storage type. For example, NVMe drives have better IOPS than hard disks.
- Ensure networking is fast. For example, a DB hosted in the US will respond faster than one in Europe if the app is hosted in the US.

## Software

- Use the latest Postgres release as newer versions come with performance optimizations.