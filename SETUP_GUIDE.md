# 🚀 ClearVue NoSQL BI + Streaming Pipeline – Setup Guide (Windows)
Purpose: Reproducible setup for MongoDB + Kafka streaming + Power BI dashboard
Audience: Group members or evaluators running this project on Windows 10/11
Tech Stack: Python, MongoDB Community Edition, Apache Kafka 3.9, Power BI Desktop 

## Prerequisites
- Python 3.9+
- MongoDB Community Edition
- Java JDK (17 or 21 (LTS) - ⚠️ Do NOT use JDK 25 - Kafka 3.9 is incompatible)
- Kafka (v3.8+)
- Power BI Desktop
- VS Code

💡 JDK Installation Tip:
Download Eclipse Temurin JDK 17 (free):
https://adoptium.net/temurin/releases/?version=17
Install to default location (e.g., C:\Program Files\Eclipse Adoptium\jdk-17.0.12.7-hotspot) 

Remember to clone the repo!!

## Step-by-Step Setup
### 1. Start MongoDB
Press Win + R → type services.msc → Enter.
Find “MongoDB Server (MongoDB)”.
If Status ≠ Running, right-click → Start.
Set Startup type = Automatic.
✅ Verify: Open MongoDB Compass → connect to mongodb://localhost:27017.

### 2. Install Python Dependencies
Open PowerShell in your project root: 
powershell
cd python
pip install pymongo pandas openpyxl kafka-python

### 3. Load Static Data into MongoDB
In the same terminal:
powershell
python import_excel_2.py

✅ Expected: Inserts ~350k records across collections (customers, sales, products, etc.).
✅ Verify in MongoDB Compass: database clearvue_db appears.

### 4. Set JAVA_HOME to JDK 17/21
⚠️ Critical: Kafka 3.9 does not work with JDK 25. 

In every new terminal used for Kafka, run first:
powershell
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.12.7-hotspot"

Replace path if your JDK is installed elsewhere. Confirm with: 
powershell
java -version

### 5. Start ZooKeeper
Open new PowerShell (with JAVA_HOME set):
powershell
cd C:\kafka_2.13-3.9.0
.\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties

✅ Wait for: binding to port 0.0.0.0/0.0.0.0:2181 → leave running.

### 6. Start Kafka Broker
Open another new PowerShell (with JAVA_HOME set):
powershell
cd C:\kafka_2.13-3.9.0
.\bin\windows\kafka-server-start.bat .\config\server.properties

✅ Wait for: [KafkaServer id=0] started → leave running.

### 7. Create Kafka Topic
Open another new PowerShell (with JAVA_HOME set):
powershell
cd C:\kafka_2.13-3.9.0
.\bin\windows\kafka-topics.bat --create --topic sales_stream --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1

✅ Expected: Created topic sales_stream.

### 8. Start Kafka → MongoDB Consumer
Open new PowerShell in your project folder:
powershell
cd python
python kafka_to_mongo.py

✅ Expected: Kafka consumer started. Waiting for messages... → leave running.

### 📄 kafka_to_mongo.py contents (ensure this matches): 
python
from kafka import KafkaConsumer
import json, pymongo

client = pymongo.MongoClient("mongodb://localhost:27017/")
db = client["clearvue_db"]
collection = db["streaming_sales"]

consumer = KafkaConsumer(
    'sales_stream',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

print("Kafka consumer started. Waiting for messages...")
for msg in consumer:
    print(f"Received: {msg.value}")
    collection.insert_one(msg.value)


### 9. Test Streaming
Open new PowerShell (with JAVA_HOME set):
powershell
cd C:\kafka_2.13-3.9.0
.\bin\windows\kafka-console-producer.bat --bootstrap-server localhost:9092 --topic sales_stream

Paste one-line JSON (based on Sales Line.xlsx schema):
json
{"DOC_NUMBER":"STREAM001","INVENTORY_CODE":"TEST123","QUANTITY":2,"UNIT_SELL_PRICE":100.0,"TOTAL_LINE_PRICE":200.0,"LAST_COST":60.0,"TRANS_DATE":"2025-10-13T10:00:00Z"}

Press Enter.

### ✅ Verify:
Consumer terminal prints the message.
MongoDB Compass → clearvue_db → streaming_sales → new document appears.

### 10. Open & Refresh Power BI Dashboard
Open prototype/dashboards/ClearVue_Dashboard.pbix.
To include streaming data:
Home → Get Data → Other → Python script
Paste:
python
import pymongo, pandas as pd
db = pymongo.MongoClient("mongodb://localhost:27017/").clearvue_db
df = pd.DataFrame(list(db.streaming_sales.find()))
if '_id' in df.columns: df = df.drop('_id', axis=1)

Load → add to report.
Refresh after sending new messages to see real-time updates.

## 🛠️ Troubleshooting
NoBrokersAvailable / Connection to node -1 → JDK 25 incompatibility. Use JDK 17/21 and set JAVA_HOME.

MongoDB connection error → Ensure MongoDB service is running (services.msc).

Power BI Python error → Install matplotlib in Power BI’s Python env, or remove unused imports.

Kafka topic creation timeout → Wait 30 sec after Kafka broker starts before creating topic.

## 📚 References
Kafka + Windows: https://kafka.apache.org/quickstart
MongoDB BI Connector (alternative): https://www.mongodb.com/docs/bi-connector/
ClearVue RFP Requirements: RFQ_ClearVue Ltd_Outgoing Comms.pdf