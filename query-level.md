# Optimizing at the Query Level

These tips are for optimizations such as adding indexes.

## Queries

- Limit and sort rows before joining tables. This way, only the required number of rows will get computed instead of every row.
- Try using `ROLLUP` to make `count(*)` faster.
- Create [indexes](https://www.postgresql.org/docs/current/indexes.html).
- Aggregate data using `WITH` before using a `JOIN`. Though this only helps certain queries, it can cut the response time in half.
- Certain queries may benefit from using `AND` instead of `WHERE`.
- Use `EXPLAIN ANALYZE` to figure out where the bottleneck is.
- Use cursor-based pagination instead of offset-based pagination. For example, write `id >  100 LIMIT 10` instead of `OFFSET 100 LIMIT 10`.

## Tables

- Denormalize data such as by writing a server that stores computed results in a new table.
- Use [materialized views](https://www.postgresql.org/docs/current/rules-materializedviews.html). If it isn’t too time consuming to refresh a materialized view when required, then it could be better than writing a server to denormalize the data.
- [Partition](https://www.postgresql.org/docs/current/ddl-partitioning.html) tables. There are three methods for doing this:
    - The first method is to write a server that runs once a day to migrate data to newly created partitions. The server would check if a certain table’s data, such as a user's transactions, have surpassed a certain number of rows. If the number of rows the dataset contains is greater than what’s deemed to be performant, it’ll migrate the dataset to a newly created partition. The problem with this method is that the relevant indexers will effectively be paused due to table locking for hours while migrating the data to a new partition.
    - The second method is to specify partitions for certain datasets in the SQL schema. This isn’t practical as white labelled products won’t be able to benefit from partitioning, and you'll have to keep manually specifying new partitions.
    - The third method is to precreate partitions based on ranges. Let’s consider an example. The transactions table has 100 rows. Each row lists the contract the transaction was from. A contract hash is a number in the range of 1 - 1,000. We’ll create ten partitions for the transactions table. The first partition will hold transactions whose contract hashes are in the range of 1 - 100. We can potentially only do this if the queries only get transactions from a specific contract. This is the only feasible method.