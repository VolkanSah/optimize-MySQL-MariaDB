# Several Ways to Optimize MySQL or MariaDB for Performance
###### Update 2024

## Table of Contents
- [1. Tune Your Database Configuration](#1-tune-your-database-configuration)
- [2. Optimize Your Queries](#2-optimize-your-queries)
- [3. Enable Query and Slow Query Logging](#3-enable-query-and-slow-query-logging)
- [4. Use Connection Pooling](#4-use-connection-pooling)
- [5. Monitor and Tune Your System](#5-monitor-and-tune-your-system)
- [Best Practices and Troubleshooting](#best-practices-and-troubleshooting)

## 1. Tune Your Database Configuration
Adjust your database configuration to align with your server hardware and usage patterns. This involves tweaking parameters like buffer sizes, cache sizes, and timeouts.

### Example Configuration Changes
- **Buffer Pool Size**: Increase `innodb_buffer_pool_size` to store more data in memory, reducing disk I/O. For a system with 16GB RAM, setting it to 8GB (50% of RAM) could be effective.
  
  ```shell
  innodb_buffer_pool_size = 8G
  ```

- **Log File Size**: Adjust `innodb_log_file_size` to manage the transaction log size, improving write performance.

  ```shell
  innodb_log_file_size = 512M
  ```

## 2. Optimize Your Queries
Use proper indexing, minimize subqueries and temporary tables, and avoid expensive joins. Utilize the `EXPLAIN` command to analyze and optimize query execution plans.

### Example Query Optimization
```sql
EXPLAIN SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id;
```
This command helps identify if the join operation is using indexes efficiently.

## 3. Enable Query and Slow Query Logging
Identifying resource-intensive queries is crucial. Enable slow query logging to detect queries that take too long to execute, allowing for targeted optimizations.

### Configuration for Slow Query Logging
```shell
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/mysql-slow.log
long_query_time = 2
```
This setup logs queries that take longer than 2 seconds to complete.

## 4. Use Connection Pooling
Reduce the overhead associated with establishing new database connections by implementing connection pooling.

### Implementing Connection Pooling with `mysqlnd_ms`
```php
$hosts = array(
  "mydb1" => array("host" => "host1", "weight" => 2),
  "mydb2" => array("host" => "host2", "weight" => 1)
);
mysqlnd_ms_set_qos($conn, MYSQLND_MS_QOS_CONSISTENCY_EVENTUAL, MYSQLND_MS_QOS_OPTION_GTID, $hosts);
```

## 5. Monitor and Tune Your System
Regularly check performance metrics such as CPU usage, memory usage, and disk I/O. Adjust configurations and queries based on these metrics.

### Tools for Monitoring
- **MyTop**: A real-time console-based tool to monitor queries and performance.
- **PT-Query-Digest**: Analyze slow queries to find potential bottlenecks.

## Best Practices and Troubleshooting
### Best Practices
- Always validate changes in a staging environment before production.
- Regularly update your database system to benefit from performance improvements and security patches.

### Troubleshooting Common Issues
- **High CPU Usage**: Check for inefficient queries and consider increasing `query_cache_size`.
- **Disk I/O Bottlenecks**: Increase `innodb_buffer_pool_size` to reduce disk reads.

These enhancements and explanations aim to provide a thorough guide for tuning and optimizing MySQL and MariaDB, helping to achieve better database performance across various environments.
