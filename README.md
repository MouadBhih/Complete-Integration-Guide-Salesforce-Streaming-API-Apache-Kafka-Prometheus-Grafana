# Complete Integration Guide: Salesforce Streaming API → Apache Kafka → Prometheus → Grafana
### Monitoring Solution with Event Streaming, Message Brokering, and Metrics Visualization
---

[Rest of the documentation remains the same...]
## Table of Contents
1. System Requirements
2. Component Installation
3. Salesforce Configuration
4. Python Setup
5. Kafka Setup
6. Kafka Exporter Setup
7. Prometheus Setup
8. Grafana Setup
9. Integration and Testing
10. Monitoring and Maintenance
11. Troubleshooting

## 1. System Requirements

### 1.1. Hardware Requirements
- CPU: 2+ cores recommended
- RAM: 8GB minimum
- Disk Space: 20GB free space
- Network: Stable internet connection

### 1.2. Software Prerequisites
- Windows 10/11
- Administrative privileges
- Java 11 or higher installed
- Python 3.8 or higher
- Web browser (Chrome/Firefox recommended)

## 2. Component Installation

### 2.1. Java Installation (if not installed)
1. Download OpenJDK 11:
   - Visit https://adoptium.net/
   - Download Windows MSI installer
2. Install:
   - Run the MSI installer
   - Select "Add to PATH" option
3. Verify:
```cmd
java -version
```

### 2.2. Python Installation
1. Download Python:
   - Visit https://www.python.org/downloads/
   - Download latest Python 3.x Windows installer
2. Install:
   - Run installer
   - Check "Add Python to PATH"
   - Choose "Customize installation"
   - Select all optional features
3. Verify:
```cmd
python --version
pip --version
```

## 3. Salesforce Configuration

### 3.1. Create Connected App
1. Login to Salesforce Setup
2. Navigate to App Manager → New Connected App
3. Configure:
```
Connected App Name: Kafka Integration
API Name: Kafka_Integration
Contact Email: [your email]
Enable OAuth Settings: Yes
Callback URL: http://localhost:8080/callback
Selected OAuth Scopes:
- Access and manage your data (api)
- Perform requests at any time (refresh_token)
- Manage user data via APIs (api)
```
4. Save and note credentials:
   - Consumer Key
   - Consumer Secret
   - Security Token

### 3.2. Create Platform Event
1. Setup → Platform Events → New Platform Event
2. Configure event:
```
Label: Monitoring Event
Plural Label: Monitoring Events
Object Name: Monitoring_Event__e
```
3. Add fields:
```
Field 1:
- Label: Metric Name
- API Name: Metric_Name__c
- Type: Text(255)

Field 2:
- Label: Metric Value
- API Name: Metric_Value__c
- Type: Number(18, 2)

Field 3:
- Label: Timestamp
- API Name: Timestamp__c
- Type: DateTime
```

## 4. Python Setup

### 4.1. Create Project Structure
```
salesforce-monitoring/
├── config/
│   └── settings.json
├── src/
│   ├── __init__.py
│   ├── salesforce_listener.py
│   └── kafka_producer.py
├── logs/
└── requirements.txt
```

### 4.2. Install Dependencies
```cmd
pip install simple-salesforce aiosfstream kafka-python requests
```

### 4.3. Configuration Files
settings.json:
```json
{
    "salesforce": {
        "username": "your_username",
        "password": "your_password",
        "security_token": "your_security_token",
        "consumer_key": "your_consumer_key",
        "consumer_secret": "your_consumer_secret",
        "domain": "login"
    },
    "kafka": {
        "bootstrap_servers": ["localhost:9092"],
        "topic": "salesforce_events"
    }
}
```

## 5. Kafka Setup

### 5.1. Download and Install Kafka
1. Download Kafka:
   - Visit https://kafka.apache.org/downloads
   - Download latest binary for Windows
2. Extract to `C:\kafka`

### 5.2. Configure Kafka
1. Edit `C:\kafka\config\server.properties`:
```properties
listeners=PLAINTEXT://localhost:9092
log.dirs=C:/kafka/logs
```

2. Edit `C:\kafka\config\zookeeper.properties`:
```properties
dataDir=C:/kafka/zookeeper-data
```

### 5.3. Create Start Scripts
create-kafka-start.bat:
```batch
@echo off
start "Zookeeper" cmd /k "C:\kafka\bin\windows\zookeeper-server-start.bat C:\kafka\config\zookeeper.properties"
timeout /t 10
start "Kafka" cmd /k "C:\kafka\bin\windows\kafka-server-start.bat C:\kafka\config\server.properties"
```

## 6. Kafka Exporter Setup

### 6.1. Install Kafka Exporter
1. Download:
   - Visit https://github.com/danielqsj/kafka_exporter/releases
   - Download latest Windows release
2. Extract to `C:\kafka_exporter`

### 6.2. Configure Kafka Exporter
Create `start-exporter.bat`:
```batch
@echo off
cd C:\kafka_exporter
kafka_exporter.exe --kafka.server=localhost:9092
```

## 7. Prometheus Setup

### 7.1. Install Prometheus
1. Download:
   - Visit https://prometheus.io/download/
   - Download latest Windows build
2. Extract to `C:\prometheus`

### 7.2. Configure Prometheus
Create `C:\prometheus\prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['localhost:9308']
    metrics_path: '/metrics'
```

### 7.3. Create Start Script
Create `start-prometheus.bat`:
```batch
@echo off
cd C:\prometheus
prometheus.exe --config.file=prometheus.yml
```

## 8. Grafana Setup

### 8.1. Install Grafana
1. Download:
   - Visit https://grafana.com/grafana/download
   - Download Windows installer (.msi)
2. Install:
   - Run MSI installer
   - Complete installation wizard

### 8.2. Install Plugins
```cmd
grafana-cli plugins install grafana-prometheus-datasource
```

### 8.3. Configure Grafana
1. Start Grafana service:
```cmd
net start Grafana
```

2. Access web interface:
   - Open http://localhost:3000
   - Default login: admin/admin
   - Change password when prompted

### 8.4. Add Data Source
1. Configuration → Data Sources
2. Add Prometheus
3. Configure:
```
Name: Prometheus
URL: http://localhost:9090
Access: Browser
```

## 9. Integration and Testing

### 9.1. Start Order
Create `start-all.bat`:
```batch
@echo off
echo Starting Kafka...
start "Kafka" cmd /k "C:\kafka\bin\windows\zookeeper-server-start.bat C:\kafka\config\zookeeper.properties"
timeout /t 10
start "Kafka" cmd /k "C:\kafka\bin\windows\kafka-server-start.bat C:\kafka\config\server.properties"
timeout /t 10

echo Starting Kafka Exporter...
start "Kafka Exporter" cmd /k "C:\kafka_exporter\start-exporter.bat"
timeout /t 5

echo Starting Prometheus...
start "Prometheus" cmd /k "C:\prometheus\start-prometheus.bat"
timeout /t 5

echo Starting Grafana...
net start Grafana

echo Starting Python Script...
python src/salesforce_listener.py
```

### 9.2. Verification Steps
1. Check Kafka:
```cmd
C:\kafka\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
```

2. Verify Prometheus:
- Open http://localhost:9090/targets
- Check if Kafka target is up

3. Test Grafana:
- Open http://localhost:3000
- Verify Prometheus data source

## 10. Monitoring and Maintenance

### 10.1. Key Metrics to Monitor
1. Kafka Metrics:
```promql
# Message Rate
rate(kafka_topic_partition_current_offset{topic="salesforce_events"}[5m])

# Consumer Lag
kafka_consumergroup_lag{topic="salesforce_events"}

# Partition Count
kafka_topic_partitions{topic="salesforce_events"}
```

### 10.2. Log Locations
```
Kafka: C:\kafka\logs
Prometheus: C:\prometheus\data
Grafana: C:\Program Files\GrafanaLabs\grafana\data\log
Python: logs/salesforce_listener.log
```

## 11. Troubleshooting

### 11.1. Common Issues

#### Kafka Won't Start
1. Check Java installation
2. Verify ports 2181 and 9092 are free:
```cmd
netstat -ano | findstr :2181
netstat -ano | findstr :9092
```

#### Prometheus Errors
1. Check configuration file syntax
2. Verify Kafka Exporter is running
3. Check port 9090 availability

#### Grafana Issues
1. Service status:
```cmd
sc query Grafana
```
2. Check logs:
```cmd
type "C:\Program Files\GrafanaLabs\grafana\data\log\grafana.log"
```

### 11.2. Recovery Procedures
1. Full System Restart:
```cmd
net stop Grafana
taskkill /F /IM kafka-exporter.exe
taskkill /F /IM prometheus.exe
taskkill /F /IM java.exe
```
Then run `start-all.bat`

### 11.3. Support Information
- Kafka Documentation: https://kafka.apache.org/documentation/
- Prometheus Documentation: https://prometheus.io/docs/
- Grafana Documentation: https://grafana.com/docs/
- Salesforce API Documentation: https://developer.salesforce.com/docs
