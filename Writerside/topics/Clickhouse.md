# Clickhouse
- a column oriented database designed for real-time analytical queries(OLAP:online analytical processing)
  - To build an analytical report , you need to filter and aggregate data, typically we use GROUP BY for such queries.
    - In row oriented database all values related to row are `physically` stored next to each other , while in column
    based , values from different columns are stored separately and values from same column are `physically` stored 
    next to each other thus filtering data on particular column becomes much faster as values have consecutive address 
    on memory

- Key properties of OLAP 
  - Transactions are not necessary.
  - Tables are wide : large numbers of columns
  - large datasets and high throughput in a single query(up to billions of rows per second per server)
  - Column values are fairly small: numbers and short strings
  - For simple queries, latencies around 50ms are allowed
  - Inserts happen in fairly large batches (> 1000 rows), not by single rows.
## SETUP Clickhouse

### IN Docker 

- Quick setup without persistent storage:[doc](https://hub.docker.com/r/clickhouse/clickhouse-server) 
```Docker
docker run -d -p 18123:8123 -p 19000:9000 --name OneClick --ulimit nofile=262144:262144 clickhouse/clickhouse-server &&
sleep 1 && docker exec -it OneClick clickhouse-client
```

