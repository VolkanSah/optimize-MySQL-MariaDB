# Several Ways to Optimize MySQL/MariaDB Server for Performance
###### Update 2025/2026

## Table of Contents
- [1. Tune Your Database Configuration](#1-tune-your-database-configuration)
- [2. Optimize Your Queries](#2-optimize-your-queries)
- [3. Enable Query and Slow Query Logging](#3-enable-query-and-slow-query-logging)
- [4. Use Connection Pooling](#4-use-connection-pooling)
- [5. Monitor and Tune Your System](#5-monitor-and-tune-your-system)
- [Best Practices and Troubleshooting](#best-practices-and-troubleshooting)
- [Example Configuration](#example-configuration)
  - [16 GB of Server RAM](#16-gb-of-server-ram)
  - [30 GB of Server RAM](#30-gb-of-server-ram)
  - [64 GB of Server RAM](#64-gb-of-server-ram)
- [Support](#your-support)

## 1. Tune Your Database Configuration
Adjust your database configuration to align with your server hardware and usage patterns. This involves tweaking parameters like buffer sizes, cache sizes, and timeouts.

### Example Configuration Changes

- **Buffer Pool Size**: The most critical InnoDB setting. Increase `innodb_buffer_pool_size` to store more data and indexes in memory, drastically reducing disk I/O. For a system with 16GB RAM, setting it to 8GB (50-70% of RAM) is recommended for dedicated database servers.
  
```shell
innodb_buffer_pool_size = 8G
```

> **MySQL 8.0+ Alternative**: Use `innodb_dedicated_server = ON` to let MySQL automatically calculate optimal values for buffer pool size, log file size, and flush method based on available RAM. This is ideal for dedicated database servers.

```shell
innodb_dedicated_server = ON
```

- **Log File Size**: Adjust `innodb_log_file_size` to manage the transaction log size. Larger log files improve write performance by reducing checkpoint frequency but increase crash recovery time. Modern systems benefit from larger values (512M-2G).

```shell
innodb_log_file_size = 512M
```

- **Thread Cache Size**: Increase `thread_cache_size` to improve connection performance by reusing threads instead of creating new ones for each connection. Monitor `Threads_created` status to tune this value.

```shell
thread_cache_size = 16
```

- **Query Cache**: ⚠️ **DEPRECATED AND REMOVED**
```shell
# DEPRECATED: Query Cache removed in MySQL 8.0+ (causes severe scalability issues)
# Still available in MariaDB but NOT recommended due to performance overhead on multi-core systems
# Use application-level caching (Redis, Memcached) or ProxySQL instead
# query_cache_size = 128M
# query_cache_type = 0
```

Modern alternatives:
  - **Application-level caching**: Redis, Memcached
  - **Query result caching**: ProxySQL
  - **Properly sized InnoDB Buffer Pool**: More effective than query cache

- **Table Open Cache**: Increase `table_open_cache` to optimize the number of open table instances. This reduces table opening overhead for frequently accessed tables. Calculate based on `max_connections * N` where N is the average number of tables per query.

```shell
table_open_cache = 4096
```

- **Parallel Processing (MySQL 8.0.27+)**: Enable parallel query execution for specific operations like index creation and table scans.

```shell
innodb_parallel_read_threads = 4
```

### System-Specific Configuration File Locations
- **Ubuntu/Debian**: `/etc/mysql/my.cnf` or `/etc/mysql/mysql.conf.d/mysqld.cnf`
- **CentOS/RHEL**: `/etc/my.cnf`
- **Arch Linux**: `/etc/mysql/my.cnf`
- **Other Linux**: Commonly `/etc/mysql/my.cnf` or `/etc/my.cnf`

## 2. Optimize Your Queries
Query optimization is crucial for performance. Use proper indexing, minimize subqueries and temporary tables, and avoid expensive joins.

### Best Practices for Query Optimization
- **Use EXPLAIN/EXPLAIN ANALYZE**: Analyze query execution plans to identify bottlenecks
- **Add appropriate indexes**: Cover WHERE, JOIN, and ORDER BY columns
- **Avoid SELECT ***: Only fetch columns you need
- **Use prepared statements**: Reduces parsing overhead and prevents SQL injection
- **Limit result sets**: Use LIMIT clauses when fetching large datasets
- **Optimize JOIN operations**: Join on indexed columns, use appropriate JOIN types

### Example Query Optimization
```sql
-- Analyze query execution plan
EXPLAIN SELECT * FROM orders 
JOIN customers ON orders.customer_id = customers.id 
WHERE orders.status = 'pending';

-- Better: Use EXPLAIN ANALYZE (MySQL 8.0.18+) for actual execution stats
EXPLAIN ANALYZE SELECT o.id, o.total, c.name 
FROM orders o
JOIN customers c ON o.customer_id = c.id 
WHERE o.status = 'pending'
LIMIT 100;
```

Check if:
- Indexes are being used (`key` column should not be NULL)
- `type` is efficient (avoid `ALL` full table scans)
- `rows` examined is reasonable
- No temporary tables or filesorts for large datasets

## 3. Enable Query and Slow Query Logging
Identifying resource-intensive queries is crucial for performance tuning. Enable slow query logging to detect queries that exceed your performance threshold.

### Configuration for Slow Query Logging
Add the following to your MySQL configuration file:

```shell
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = 1
```

**Parameters explained:**
- `slow_query_log`: Enables slow query logging
- `long_query_time`: Logs queries taking longer than N seconds (adjust based on your needs)
- `log_queries_not_using_indexes`: Catches potentially problematic queries even if they're fast

### System-Specific Log File Locations
- **Ubuntu/Debian**: `/var/log/mysql/mysql-slow.log`
- **CentOS/RHEL**: `/var/log/mysqld-slow.log`
- **Arch Linux**: `/var/log/mysql/mysql-slow.log`
- **Other Linux**: Commonly `/var/log/mysql-slow.log`

### Analyzing Slow Query Logs
Use tools like `pt-query-digest` from Percona Toolkit:
```bash
pt-query-digest /var/log/mysql/mysql-slow.log
```

## 4. Use Connection Pooling
Connection pooling reduces overhead associated with establishing new database connections by reusing existing connections. This is critical for high-traffic applications.

### Popular Connection Pooling Solutions
- **Application-level**: Most frameworks have built-in pooling (Django, Spring, Node.js)
- **ProxySQL**: Advanced proxy with connection pooling, query caching, and load balancing
- **PgBouncer**: Lightweight connection pooler (PostgreSQL-focused but concept applies)

### PHP Example with PDO
```php
// Configure connection pool
$options = [
    PDO::ATTR_PERSISTENT => true,  // Enable persistent connections
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
];

$pdo = new PDO($dsn, $user, $pass, $options);
```

### ProxySQL Example
ProxySQL provides advanced connection pooling with query routing and caching capabilities:
```sql
-- Configure connection pool in ProxySQL
INSERT INTO mysql_servers(hostgroup_id, hostname, port) 
VALUES (0, '127.0.0.1', 3306);

UPDATE global_variables 
SET variable_value='200' 
WHERE variable_name='mysql-max_connections';
```

## 5. Monitor and Tune Your System
Regular monitoring is essential to maintain optimal performance. Track key metrics and adjust configurations based on real-world usage patterns.

### Key Performance Metrics to Monitor
- **Query Performance**: Slow query count, average query time
- **Connection Usage**: Current connections, max connections reached
- **Buffer Pool**: Hit rate (should be >95%), pages read/written
- **InnoDB Metrics**: Row operations, buffer pool efficiency
- **System Resources**: CPU usage, memory usage, disk I/O wait

### Essential Monitoring Tools

**Performance Schema (MySQL 5.6+)**
Built-in performance monitoring with minimal overhead:
```sql
-- Enable Performance Schema
UPDATE performance_schema.setup_instruments 
SET ENABLED = 'YES', TIMED = 'YES';

-- Find slowest queries
SELECT * FROM performance_schema.events_statements_summary_by_digest 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

**MySQLTuner**
Automated configuration analysis and recommendations:
```bash
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
perl mysqltuner.pl
```

**pt-query-digest (Percona Toolkit)**
Analyze slow query logs and find bottlenecks:
```bash
pt-query-digest /var/log/mysql/mysql-slow.log
```

**mytop**
Real-time console-based MySQL monitoring (like top for MySQL):
```bash
mytop -u root -p
```

**Prometheus + Grafana**
Modern monitoring stack with MySQL exporter for production environments.

## Best Practices and Troubleshooting

### Best Practices
- **Test in staging first**: Always validate configuration changes in a non-production environment before deploying
- **Keep systems updated**: Regularly update MySQL/MariaDB for performance improvements, bug fixes, and security patches
- **Monitor before and after**: Establish baseline metrics before making changes to measure impact
- **Incremental tuning**: Change one parameter at a time to understand its effect
- **Document changes**: Keep track of configuration modifications and their impact
- **Regular backups**: Always maintain current backups before making system changes
- **Capacity planning**: Monitor growth trends and plan for scaling needs

### Troubleshooting Common Issues

**High CPU Usage**
- Check for missing indexes with `EXPLAIN` on slow queries
- Look for full table scans in slow query log
- Review `Threads_running` - high values indicate query bottlenecks
- Consider horizontal scaling or read replicas for high read loads

**Memory Issues**
- Monitor `innodb_buffer_pool_size` - shouldn't exceed 70-80% of RAM
- Check for memory leaks in application connections
- Review `max_connections` - each connection consumes memory
- Monitor `tmp_table_size` and `max_heap_table_size` for temporary table usage

**Disk I/O Bottlenecks**
- Increase `innodb_buffer_pool_size` to cache more data in memory
- Use SSDs for better I/O performance
- Check `innodb_flush_log_at_trx_commit` setting (1=safe, 2=faster)
- Consider enabling `innodb_doublewrite = 0` on reliable storage (SSDs)
- Monitor slow query log for queries causing high disk reads

**Connection Problems**
- Increase `max_connections` if reaching limits
- Implement connection pooling in application
- Check for connection leaks in application code
- Review `wait_timeout` and `interactive_timeout` settings

**Replication Lag**
- Check network latency between master and replica
- Review `slave_parallel_workers` for parallel replication (MySQL 5.7+)
- Monitor binary log size and replication position
- Consider using GTIDs for easier replication management

### Deprecated Features

**Query Cache (MySQL 8.0+)**
- Completely removed due to severe scalability issues on multi-core systems
- **Replacement**: Use Redis, Memcached, or ProxySQL for query result caching

**Query Cache (MariaDB)**
- Still available but causes performance degradation under load
- **Recommendation**: Disable with `query_cache_type = 0`

**expire_logs_days (MySQL 8.0.11+)**
- Deprecated in MySQL 8.0.11, removed in MySQL 8.2.0
- **Replacement**: Use `binlog_expire_logs_seconds` (value in seconds)
  ```shell
  # Old (deprecated)
  expire_logs_days = 10
  
  # New (recommended)
  binlog_expire_logs_seconds = 864000  # 10 days in seconds
  ```

**binlog_format (MySQL 8.0.34+)**
- Deprecated, will be removed in future versions
- **Recommendation**: Use ROW format exclusively for all new setups
  ```shell
  binlog_format = ROW  # Most reliable for replication
  ```

## Example Configuration

These configurations are starting points. Monitor your specific workload and adjust accordingly.

### 16 GB of Server RAM
Optimized for a dedicated database server with 16GB RAM. Allocates ~8GB to InnoDB buffer pool (50% of RAM).

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 8G              # 50% of RAM for dedicated DB server
innodb_buffer_pool_instances = 8          # Split buffer pool (1 instance per GB)
innodb_log_file_size = 512M               # Larger = better write performance
innodb_log_buffer_size = 16M              # Transaction log buffer
innodb_flush_log_at_trx_commit = 2        # 0=fastest, 1=safest, 2=balanced
innodb_flush_method = O_DIRECT            # Avoid double buffering

# Thread and connection settings
thread_cache_size = 16                    # Reuse threads for new connections
max_connections = 500                     # Adjust based on application needs
max_user_connections = 50                 # Per-user connection limit

# Table and query settings
table_open_cache = 4096                   # Cache for open tables
table_definition_cache = 2048             # Cache for table definitions
open_files_limit = 8192                   # OS-level file descriptor limit

# Query cache - DEPRECATED
# WARNING: Query Cache removed in MySQL 8.0+
# MariaDB still supports it but NOT recommended
# Use Redis/Memcached or application-level caching instead
#query_cache_type = 0
#query_cache_size = 0

# Temporary tables
tmp_table_size = 64M                      # Max size of internal memory tables
max_heap_table_size = 64M                 # Max size of memory tables

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2                       # Log queries taking longer than 2 seconds
log_queries_not_using_indexes = 1         # Log queries without indexes

# Binary logging and replication
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW                       # Recommended format for replication
max_binlog_size = 100M
sync_binlog = 1                           # 0=faster, 1=safest

# Binary log expiration
# DEPRECATED: expire_logs_days removed in MySQL 8.2.0
# Use binlog_expire_logs_seconds instead
binlog_expire_logs_seconds = 864000       # 10 days (10 * 24 * 60 * 60)
# expire_logs_days = 10                   # Old parameter (deprecated)

# Error logging
log_error = /var/log/mysql/error.log

# MySQL 8.0+ auto-configuration (optional - comment out manual settings above if using this)
# innodb_dedicated_server = ON            # Auto-configure based on available RAM

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4
```

### 30 GB of Server RAM
Optimized for a dedicated database server with 30GB RAM. Allocates ~24GB to InnoDB buffer pool (80% of RAM).

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 24G             # 80% of RAM for dedicated DB server
innodb_buffer_pool_instances = 16         # Multiple instances for better concurrency
innodb_log_file_size = 1G                 # Larger logs for heavy write workloads
innodb_log_buffer_size = 32M              # Larger log buffer
innodb_flush_log_at_trx_commit = 2        # Balanced performance/safety
innodb_flush_method = O_DIRECT            # Avoid OS cache duplication
innodb_io_capacity = 2000                 # SSD: 2000-5000, HDD: 200
innodb_io_capacity_max = 4000             # Max I/O for background tasks

# Thread and connection settings
thread_cache_size = 32                    # Higher thread cache for more connections
max_connections = 1000                    # Support more concurrent connections
max_user_connections = 100                # Per-user limit

# Table and query settings
table_open_cache = 8192                   # More cached tables
table_definition_cache = 4096             # More cached definitions
open_files_limit = 16384                  # Higher file limit

# Query cache - DEPRECATED
# WARNING: Query Cache removed in MySQL 8.0+
# MariaDB: causes scalability issues, disable it
#query_cache_type = 0
#query_cache_size = 0

# Temporary tables
tmp_table_size = 128M                     # Larger temporary tables
max_heap_table_size = 128M                # Match tmp_table_size

# Parallel processing (MySQL 8.0.27+)
innodb_parallel_read_threads = 4          # Parallel query execution

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = 1

# Binary logging and replication
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
max_binlog_size = 100M
sync_binlog = 1

# Binary log expiration
binlog_expire_logs_seconds = 864000       # 10 days

# Error logging
log_error = /var/log/mysql/error.log

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4
```

### 64 GB of Server RAM
Optimized for a high-performance dedicated database server with 64GB RAM. Allocates ~48GB to InnoDB buffer pool (75% of RAM).

```ini
[mysqld]
# General settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql

# Performance settings
innodb_buffer_pool_size = 48G             # 75% of RAM (leave room for OS/connections)
innodb_buffer_pool_instances = 32         # More instances for high concurrency
innodb_log_file_size = 2G                 # Large logs for write-heavy workloads
innodb_log_buffer_size = 64M              # Large log buffer
innodb_flush_log_at_trx_commit = 2        # Balanced durability
innodb_flush_method = O_DIRECT            # Direct I/O to avoid double buffering
innodb_io_capacity = 5000                 # High for enterprise SSDs
innodb_io_capacity_max = 10000            # Maximum I/O throughput
innodb_read_io_threads = 8                # More read threads
innodb_write_io_threads = 8               # More write threads

# Thread and connection settings
thread_cache_size = 64                    # Large thread cache
max_connections = 2000                    # High concurrency support
max_user_connections = 200                # Per-user connection limit
thread_handling = pool-of-threads         # Thread pool (MariaDB) or MySQL Enterprise

# Table and query settings
table_open_cache = 16384                  # Large table cache
table_definition_cache = 8192             # Large definition cache
open_files_limit = 32768                  # Very high file limit

# Query cache - DEPRECATED
# WARNING: Query Cache removed in MySQL 8.0+
# Use application-level caching (Redis, Memcached) instead
#query_cache_type = 0
#query_cache_size = 0

# Temporary tables
tmp_table_size = 256M                     # Large temporary tables
max_heap_table_size = 256M                # Match tmp_table_size

# Parallel processing (MySQL 8.0.27+)
innodb_parallel_read_threads = 8          # More parallel threads for large scans

# Sort and join buffers
sort_buffer_size = 4M                     # Per-connection sort buffer
join_buffer_size = 4M                     # Per-connection join buffer
read_rnd_buffer_size = 4M                 # Random read buffer

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1                       # Stricter threshold for high-performance system
log_queries_not_using_indexes = 1

# Binary logging and replication
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW                       # Safest and most reliable
max_binlog_size = 100M
sync_binlog = 1
binlog_cache_size = 4M                    # Binary log cache per connection

# Binary log expiration
binlog_expire_logs_seconds = 604800       # 7 days (shorter for high-volume systems)

# Replication (if using)
relay_log = /var/log/mysql/mysql-relay-bin
relay_log_recovery = 1                    # Automatic relay log recovery
slave_parallel_workers = 8                # Parallel replication threads (MySQL 5.7+)
slave_parallel_type = LOGICAL_CLOCK       # MySQL 5.7+ parallel replication

# Error logging
log_error = /var/log/mysql/error.log

# Performance Schema (for monitoring)
performance_schema = ON
performance-schema-instrument = 'stage/%=ON'
performance-schema-consumer-events-stages-current = ON

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4
```

---

### Configuration Notes

**Adjusting for Your Workload:**
- **Read-heavy**: Increase `innodb_buffer_pool_size`, add read replicas
- **Write-heavy**: Increase `innodb_log_file_size`, consider `innodb_flush_log_at_trx_commit = 2`
- **Mixed workload**: Use balanced settings from examples above
- **SSD storage**: Increase `innodb_io_capacity` (2000-5000+)
- **HDD storage**: Keep `innodb_io_capacity` lower (200-500)

**MySQL 8.0+ Simplification:**
For dedicated database servers, consider using:
```ini
innodb_dedicated_server = ON
```
This automatically configures:
- `innodb_buffer_pool_size`
- `innodb_log_file_size`
- `innodb_flush_method`

**Testing Configuration Changes:**
1. Backup current configuration
2. Apply changes to staging environment
3. Run realistic load tests
4. Monitor for 24-48 hours
5. Deploy to production during maintenance window
6. Monitor closely for first few days

---

These enhancements and explanations provide a comprehensive guide for tuning and optimizing MySQL and MariaDB, helping achieve better database performance across various environments and workloads.

## Your Support

Found this useful?

- ⭐ Star this repository
- 🐛 Report issues
- 💡 Suggest improvements
- 💖 [Sponsor development](https://github.com/sponsors/volkansah)

---

**Stay secure. Stay paranoid. 🔒**


### Other Stuff

##### Security Guides:
- [Security Headers — Complete Implementation Guide](https://github.com/VolkanSah/Security-Headers)
- [Securing FastAPI Applications](https://github.com/VolkanSah/Securing-FastAPI-Applications)
- [ModSecurity Webserver Protection Guide](https://github.com/VolkanSah/ModSecurity-Webserver-Protection-Guide)
- [GPT Security Best Practices](https://github.com/VolkanSah/GPT-Security-Best-Practices)
- [WPScan – WordPress Security Scanner Guide](https://github.com/VolkanSah/WordPress-Security-Scanner-advanced-use)


Thank you for your support! ❤️



##### Credits
> Copyright S. Volkan Kücükbudak
> 
> Updated on 05.12.2025
