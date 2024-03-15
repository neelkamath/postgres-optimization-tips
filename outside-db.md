# Optimizing Outside the DB

These tips are for optimizations done outside of Postgres but are strongly related to the DB such as using Redis to cache query results.

## Caching

- Cache query results outside the DB using something like Redis.

## Connections

- Reduce the number of DB connections needed. For example, an app running on Fargate will reuse the same few DB connections across API calls whereas Lambda will have many occasionally used DB connections.

## Queries

- Parallelize executing independent queries at the application level.
- Break queries into independent ones, and join their results at the application level.

## DBs

- Use replicas to split the load across multilple DBMSs.
- Migrate at least some of the data to a different type of DB such as Neo4j or MongoDB.