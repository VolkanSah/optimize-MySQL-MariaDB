
# Several Ways to Optimize MySQL/MariaDB for Performance
###### Update 2024

## Table of Contents
- [1. Tune Your Database Configuration](#1-tune-your-database-configuration)
- [2. Optimize Your Queries](#2-optimize-your-queries)
- [3. Enable Query and Slow Query Logging](#3-enable-query-and-slow-query-logging)
- [4. Use Connection Pooling](#4-use-connection-pooling)
- [5. Monitor and Tune Your System](#5-monitor-and-tune-your-system)
- [Best Practices and Troubleshooting](#best-practices-and-troubleshooting)
- [Example Configuration](#example-configuration)
  - [16 GB of Server RAM](#16-GB-of-Server-RAM)
  - [30 GB of Server RAM](#30-GB-of-Server-RAM)
  - [64 GB of Server RAM](#64-GB-of-Server-RAM)

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

- **Thread Cache Size**: Increase `thread_cache_size` to improve performance by reusing threads.
```shell
  thread_cache_size = 16
```

- **Query Cache**: Configure the query cache to speed up repetitive queries.
```shell
# DEPRECATED: Query Cache removed in MySQL 8.0+
# Still available in MariaDB but NOT recommended due to performance overhead
# Use application-level caching (Redis, Memcached) or ProxySQL instead
  query_cache_size = 128M
  query_cache_type = 1
```

- **Table Open Cache**: Increase `table_open_cache` to optimize the number of open tables.
```shell
  table_open_cache = 4096
```

### System-Specific Configuration File Locations
- **Ubuntu/Debian**: `/etc/mysql/my.cnf`
- **CentOS**: `/etc/my.cnf`
- **Arch Linux**: `/etc/mysql/my.cnf`
- **Other Linux**: Varies, commonly `/etc/mysql/my.cnf` or `/etc/my.cnf`

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
Add the following to your MySQL configuration file:

```shell
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
```

### System-Specific Log File Locations
- **Ubuntu/Debian**: `/var/log/mysql/mysql-slow.log`
- **CentOS**: `/var/log/mysqld-slow.log`
- **Arch Linux**: `/var/log/mysql/mysql-slow.log`
- **Other Linux**: Varies, commonly `/var/log/mysql-slow.log`

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

### Deprecated Features
- **Query Cache (MySQL 8.0+)**: Completely removed. Use Redis/Memcached or ProxySQL instead.
- **Query Cache (MariaDB)**: Still supported but causes performance issues on multi-core systems. Disable with `query_cache_type = 0`.

## Example Configuration
##### 16 GB of Server RAM
An example MySQL/MariaDB configuration file optimized for a system with 16GB of RAM:

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 8G
innodb_log_file_size = 512M
thread_cache_size = 16
# WARNING: Query Cache deprecated - MySQL 8.0+ removed it entirely
# MariaDB still supports it but causes scalability issues on multi-core systems
# Recommended: Disable query_cache or use application-level caching instead
#query_cache_size = 128M
#query_cache_type = 1
table_open_cache = 4096

# Slow query log settings
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2

# Connection settings
max_connections = 500
max_user_connections = 50

# Replication settings
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = mixed

# Additional settings
log_error = /var/log/mysql/error.log
expire_logs_days = 10
max_binlog_size = 100M
```
##### 30 GB of Server RAM
An example MySQL/MariaDB configuration file optimized for a system with 30GB of RAM:

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 24G
innodb_log_file_size = 1G
thread_cache_size = 32
# WARNING: Query Cache deprecated - MySQL 8.0+ removed it entirely
# MariaDB still supports it but causes scalability issues on multi-core systems
# Recommended: Disable query_cache or use application-level caching instead
# query_cache_size = 256M
# query_cache_type = 1
table_open_cache = 8192

# Slow query log settings
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2

# Connection settings
max_connections = 1000
max_user_connections = 100

# Replication settings
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = mixed

# Additional settings
log_error = /var/log/mysql/error.log
expire_logs_days = 10
max_binlog_size = 100M
```

##### 64 GB of Server RAM
An example MySQL/MariaDB configuration file optimized for a system with 64GB of RAM:

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 48G
innodb_log_file_size = 2G
thread_cache_size = 64
# WARNING: Query Cache deprecated - MySQL 8.0+ removed it entirely
# MariaDB still supports it but causes scalability issues on multi-core systems
# Recommended: Disable query_cache or use application-level caching instead
# query_cache_size = 512M
# query_cache_type = 1
table_open_cache = 16384

# Slow query log settings
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2

# Connection settings
max_connections = 2000
max_user_connections = 200

# Replication settings
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = mixed

# Additional settings
log_error = /var/log/mysql/error.log
expire_logs_days = 10
max_binlog_size = 100M
```
These enhancements and explanations aim to provide a thorough guide for tuning and optimizing MySQL and MariaDB, helping to achieve better database performance across various environments.
### Credits
**Volkan Sah**
