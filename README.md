# DE_PIPELINE
# Real-Time Data Pipeline with Kafka, Debezium, Flink & StarRocks

A comprehensive Docker Compose setup for building a real-time data pipeline that captures database changes using Debezium, processes them with Apache Flink, and stores them in StarRocks data warehouse.

## 🏗️ Architecture Overview

This pipeline consists of the following components:

```
MySQL/PostgreSQL → Debezium → Kafka → Apache Flink → StarRocks
                                 ↓
                           Schema Registry
                                 ↓
                              Kafdrop UI
```

## 📦 Components

### Core Services
- **Apache Kafka** (7.6.0) - Event streaming platform running in KRaft mode
- **Debezium Connect** (2.7.3) - Change Data Capture (CDC) from MySQL/PostgreSQL
- **Apache Flink** (1.18.0) - Stream processing engine with low-latency optimizations
- **StarRocks** (3.3) - MPP database for analytics and data warehousing

### Supporting Services
- **Schema Registry** (7.6.0) - Avro schema management for Kafka
- **MySQL** (8.1) - Source database with binlog enabled
- **PostgreSQL** (15) - Source database with logical replication
- **Kafdrop** - Web UI for Kafka cluster management
- **Connect Sink** - Additional Kafka Connect instance for sink operations

## 🚀 Quick Start

### Prerequisites
- Docker and Docker Compose installed
- At least 8GB RAM available for containers
- Ports 3306, 5432, 8030, 8040, 8081-8084, 9000, 9020, 9030, 9092 available

### 1. Configuration Setup

Before starting, update the following environment variables:

#### MySQL Configuration
```yaml
MYSQL_ROOT_PASSWORD: your_secure_password
MYSQL_DATABASE: your_database_name
```

#### PostgreSQL Configuration
```yaml
POSTGRES_USER: your_postgres_user
POSTGRES_PASSWORD: your_secure_password
POSTGRES_DB: your_database_name
```

Update the PostgreSQL healthcheck with your credentials:
```yaml
pg_isready -U your_postgres_user -d your_database_name
```

### 2. Create Required Directories

```bash
mkdir -p flink-jars
# Add any additional Flink JARs to this directory
```

### 3. Start the Pipeline

```bash
# Start all services
docker-compose up -d

# Check service health
docker-compose ps

# View logs for specific service
docker-compose logs -f kafka
```

### 4. Verify Setup

Check that all services are healthy:
```bash
# Kafka topics
curl http://localhost:9000

# Debezium Connect
curl http://localhost:8083

# Flink JobManager
curl http://localhost:8081

# StarRocks Frontend
mysql -h127.0.0.1 -P9030 -uroot -e "SHOW FRONTENDS;"
```

## 🔧 Service Configuration

### Kafka Optimizations
- **KRaft Mode**: No Zookeeper dependency
- **Auto Topic Creation**: Enabled with 4 partitions
- **Replication Factor**: Set to 1 for single-node setup

### Flink Optimizations
- **Low Latency**: Buffer timeout set to 0ms
- **Checkpointing**: Every 500ms with AT_LEAST_ONCE mode
- **Memory Management**: 2GB process size with optimized network buffers
- **State Backend**: RocksDB with tuned write buffers

### StarRocks Optimizations
- **Stream Load**: 10s timeout with 4 parallel instances
- **Memory Limits**: 4GB per container
- **Load Performance**: 1GB streaming load capacity

## 📊 Access Points

| Service | URL | Description |
|---------|-----|-------------|
| Kafdrop | http://localhost:9000 | Kafka cluster management UI |
| Flink JobManager | http://localhost:8081 | Flink job management and monitoring |
| Debezium Connect | http://localhost:8083 | CDC connector management |
| Connect Sink | http://localhost:8084 | Sink connector management |
| StarRocks FE | http://localhost:8030 | StarRocks frontend web UI |
| MySQL | localhost:3306 | MySQL database connection |
| PostgreSQL | localhost:5432 | PostgreSQL database connection |

## 🔌 Setting Up Connectors

### MySQL Debezium Connector
```json
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "debezium",
    "database.server.id": "1",
    "database.server.name": "mysql-server",
    "database.include.list": "your_database_name",
    "database.history.kafka.bootstrap.servers": "kafka:29092",
    "database.history.kafka.topic": "dbhistory.mysql"
  }
}
```

### PostgreSQL Debezium Connector
```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "your_postgres_user",
    "database.password": "your_secure_password",
    "database.dbname": "your_database_name",
    "database.server.name": "postgres-server",
    "plugin.name": "pgoutput"
  }
}
```

### Deploy Connectors
```bash
# Deploy MySQL connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @mysql-connector.json

# Deploy PostgreSQL connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @postgres-connector.json
```

## 🎯 Flink Job Development

### Sample Flink SQL Job
```sql
-- Create Kafka source table
CREATE TABLE kafka_source (
  id BIGINT,
  name STRING,
  email STRING,
  created_at TIMESTAMP(3),
  WATERMARK FOR created_at AS created_at - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'your-topic',
  'properties.bootstrap.servers' = 'kafka:29092',
  'format' = 'json'
);

-- Create StarRocks sink table
CREATE TABLE starrocks_sink (
  id BIGINT,
  name STRING,
  email STRING,
  created_at TIMESTAMP(3)
) WITH (
  'connector' = 'starrocks',
  'jdbc-url' = 'jdbc:mysql://starrocks-fe-0:9030',
  'load-url' = 'starrocks-fe-0:8030',
  'database-name' = 'your_database',
  'table-name' = 'your_table',
  'username' = 'root',
  'password' = ''
);

-- Process and insert data
INSERT INTO starrocks_sink
SELECT * FROM kafka_source;
```

## 📈 Monitoring and Troubleshooting

### Health Checks
All services include health checks. Monitor with:
```bash
# Check all service status
docker-compose ps

# Check specific service logs
docker-compose logs -f [service-name]
```

### Common Issues

1. **Memory Issues**: Increase Docker memory allocation to at least 8GB
2. **Port Conflicts**: Ensure all ports are available before starting
3. **Database Permissions**: Verify Debezium user has proper replication permissions
4. **Slow Startup**: Services have dependencies; wait for health checks to pass

### Performance Monitoring
- **Flink**: Monitor job metrics at http://localhost:8081
- **Kafka**: Check topic lag and throughput in Kafdrop
- **StarRocks**: Monitor load performance in FE web UI

## 🛠️ Customization

### Adding Custom Flink JARs
Place JAR files in the `flink-jars` directory before starting:
```bash
cp your-custom-connector.jar flink-jars/
docker-compose restart jobmanager taskmanager
```

### Scaling TaskManagers
```yaml
taskmanager:
  deploy:
    replicas: 3  # Scale to 3 TaskManager instances
```

### Additional Kafka Topics
```bash
# Create topic via Kafdrop UI or CLI
docker exec kafka kafka-topics --create --topic new-topic --partitions 4 --replication-factor 1 --bootstrap-server localhost:9092
```

## 🔒 Security Considerations

- Change default passwords for all databases
- Configure network security if exposing services
- Enable SSL/TLS for production deployments
- Set up proper authentication for Kafka and Schema Registry

## 📚 Additional Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Apache Flink Documentation](https://flink.apache.org/documentation/)
- [StarRocks Documentation](https://docs.starrocks.io/)

## 🤝 Contributing

1. Fork the repository
2. Create feature branch
3. Test changes thoroughly
4. Submit pull request with detailed description

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.
