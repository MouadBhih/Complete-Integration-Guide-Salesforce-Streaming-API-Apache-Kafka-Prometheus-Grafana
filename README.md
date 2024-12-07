# Salesforce Streaming API to Grafana Monitoring - Complete Setup Guide

## Table of Contents
1. Salesforce Configuration
2. Python Environment Setup
3. Kafka Setup
4. Prometheus Setup
5. Grafana Setup
6. Complete System Integration

## 1. Salesforce Configuration

### 1.1. Create Connected App
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
4. Save the Connected App
5. Note down:
   - Consumer Key (client_id)
   - Consumer Secret

### 1.2. Create Platform Event
1. Setup → Platform Events → New Platform Event
2. Configure:
   ```
   Label: Monitoring Event
   Plural Label: Monitoring Events
   Object Name: Monitoring_Event__e
   ```
3. Add Custom Fields:
   ```
   Field Label: Metric Name
   Field Name: Metric_Name__c
   Type: Text(255)
   
   Field Label: Metric Value
   Field Name: Metric_Value__c
   Type: Number(18, 2)
   
   Field Label: Timestamp
   Field Name: Timestamp__c
   Type: DateTime
   ```
4. Save the Platform Event

## 2. Python Environment Setup

### 2.1. Install Python Requirements
```bash
pip install simple-salesforce aiosfstream kafka-python requests
```

### 2.2. Create Project Structure
```
salesforce-monitoring/
├── config/
│   └── settings.json
├── src/
│   ├── __init__.py
│   ├── salesforce_listener.py
│   └── kafka_producer.py
└── requirements.txt
```

### 2.3. Create Configuration File (config/settings.json)
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

### 2.4. Create Main Script (src/salesforce_listener.py)
```python
import json
import asyncio
from aiosfstream import Client
from simple_salesforce import Salesforce
from kafka import KafkaProducer
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SalesforceEventListener:
    def __init__(self, config_path):
        with open(config_path) as f:
            self.config = json.load(f)
        
        self.producer = self.create_kafka_producer()
        self.sf = self.create_salesforce_client()

    def create_kafka_producer(self):
        return KafkaProducer(
            bootstrap_servers=self.config['kafka']['bootstrap_servers'],
            value_serializer=lambda x: json.dumps(x).encode('utf-8')
        )

    def create_salesforce_client(self):
        return Salesforce(
            username=self.config['salesforce']['username'],
            password=self.config['salesforce']['password'],
            security_token=self.config['salesforce']['security_token'],
            consumer_key=self.config['salesforce']['consumer_key'],
            consumer_secret=self.config['salesforce']['consumer_secret'],
            domain=self.config['salesforce']['domain']
        )

    async def process_event(self, message):
        try:
            logger.info(f"Processing event: {message}")
            self.producer.send(
                self.config['kafka']['topic'],
                value={
                    'event_type': 'Monitoring_Event__e',
                    'data': message,
                    'timestamp': message.get('event').get('createdDate')
                }
            )
            logger.info("Event sent to Kafka")
        except Exception as e:
            logger.error(f"Error processing event: {e}")

    async def start_listening(self):
        client = Client(
            self.sf.session_id,
            self.sf.instance_url,
        )

        try:
            await client.open()
            logger.info("Connected to Salesforce Streaming API")
            
            async for message in client.subscribe('/event/Monitoring_Event__e'):
                await self.process_event(message)
                
        except Exception as e:
            logger.error(f"Streaming error: {e}")
        finally:
            await client.close()

def main():
    listener = SalesforceEventListener('config/settings.json')
    asyncio.run(listener.start_listening())

if __name__ == "__main__":
    main()
```

## 3. Kafka Setup

### 3.1. Start Kafka Environment (Windows)
```cmd
# Start Zookeeper
cd C:\kafka
.\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties

# Start Kafka
cd C:\kafka
.\bin\windows\kafka-server-start.bat .\config\server.properties
```

### 3.2. Create Kafka Topic
```cmd
.\bin\windows\kafka-topics.bat --create --topic salesforce_events --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

## 4. Prometheus Setup

### 4.1. Create Prometheus Configuration
Create `prometheus.yml` in C:\prometheus:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['localhost:9308']
```

### 4.2. Create Prometheus Start Script
Create `start-prometheus.bat`:
```batch
cd C:\prometheus
prometheus.exe --config.file=prometheus.yml
```

## 5. Grafana Setup

### 5.1. Install Prometheus Plugin
```cmd
grafana-cli plugins install grafana-prometheus-datasource
```

### 5.2. Configure Data Source
1. Open Grafana (http://localhost:3000)
2. Go to Configuration → Data Sources
3. Add Prometheus data source
4. Configure:
   ```
   Name: Prometheus
   URL: http://localhost:9090
   Access: Browser
   ```

### 5.3. Create Dashboard
1. Create New Dashboard
2. Add Panel
3. Query:
```
rate(kafka_topic_messages_total{topic="salesforce_events"}[5m])
```

### 5.4. Sample Dashboard JSON
```json
{
  "annotations": {
    "list": []
  },
  "editable": true,
  "panels": [
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "showPoints": "never",
            "spanNulls": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "events/sec"
        }
      },
      "title": "Salesforce Events Rate",
      "type": "timeseries"
    }
  ],
  "refresh": "5s",
  "schemaVersion": 30,
  "title": "Salesforce Events Monitor",
  "version": 1
}
```

## 6. Complete System Integration

### 6.1. Start Order
1. Start Kafka and Zookeeper
2. Start Prometheus
3. Start Grafana
4. Run Python script

### 6.2. Verification Steps
1. Verify Kafka Consumer:
```cmd
.\bin\windows\kafka-console-consumer.bat --topic salesforce_events --from-beginning --bootstrap-server localhost:9092
```

2. Check Prometheus Metrics:
- Open http://localhost:9090
- Query: `kafka_topic_messages_total`

3. Verify Grafana Dashboard:
- Open http://localhost:3000
- Check data is flowing in dashboard panels

### 6.3. Error Handling and Logging
The Python script includes logging. Check the console output for:
- Connection status
- Event processing
- Error messages

### 6.4. Monitoring Health
1. Kafka Health:
```cmd
.\bin\windows\kafka-topics.bat --describe --topic salesforce_events --bootstrap-server localhost:9092
```

2. Prometheus Health:
- http://localhost:9090/targets

3. Grafana Health:
- http://localhost:3000/health

## 7. Troubleshooting

### 7.1. Common Issues
1. Salesforce Connection:
- Check credentials
- Verify OAuth settings
- Confirm API version

2. Kafka Issues:
- Verify Zookeeper is running
- Check Kafka logs
- Confirm topic exists

3. Prometheus Issues:
- Check target status
- Verify metrics are being scraped
- Review configuration

4. Grafana Issues:
- Verify data source connection
- Check query syntax
- Review panel configuration

### 7.2. Logging Locations
- Kafka: C:\kafka\logs
- Prometheus: C:\prometheus\data
- Grafana: C:\Program Files\GrafanaLabs\grafana\data\log
