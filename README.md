
# Centralized Logging for E-Commerce Microservices using the ELK Stack

### 🧭 Objective:

To monitor logs from different microservices (Node.js, Python, Java) deployed via Docker, centralize them using the **ELK Stack** (Elasticsearch, Logstash, Kibana), and gain insights from the logs (errors, performance issues, usage patterns).

---

## 🔧 Tools Used:

| Tool           | Purpose                             |
| -------------- | ----------------------------------- |
| Elasticsearch  | Stores & indexes logs for searching |
| Logstash       | Ingests, processes & forwards logs  |
| Kibana         | Visualizes log data                 |
| Filebeat       | Lightweight log shipper on services |
| Docker         | Containerize and deploy services    |
| Docker Compose | Manage multi-container setup        |

---

## 🏗 Project Structure:

```
elk-ecommerce-logs/
├── docker-compose.yml
├── nginx-service/
│   └── logs/access.log
├── product-service/
│   └── app.log
├── order-service/
│   └── app.log
├── logstash/
│   └── logstash.conf
└── kibana/
```

---

## 🔥 Step-by-Step Breakdown of ELK Stack Concepts:

---

### **Step 1: Install and Run ELK Stack (via Docker Compose)**

**Concept**: Run Elasticsearch, Logstash, and Kibana using Docker.

📄 `docker-compose.yml`

```yaml

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.9
    volumes:  # Mount Log Files into Logstash Container
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./nginx-service:/usr/share/logstash/pipeline/nginx-service
      - ./product-service:/usr/share/logstash/pipeline/product-service
      - ./order-service:/usr/share/logstash/pipeline/order-service
    ports:
      - "5000:5000"
#This ensures Logstash can read your files inside the container.

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    ports:
      - "5601:5601"
```

```bash
sudo apt  install docker-compose
docker-compose up -d
# watch the logs
docker logs -f elk-ecommerce-logs_logstash_1

docker-compose down
```

---

### **Step 2: Understand and Configure Elasticsearch**

**Concept**: Elasticsearch is a NoSQL search engine. It indexes logs sent from Logstash or Beats.

✔️ Example:

```bash
curl http://localhost:9200
```
Returns cluster metadata confirming Elasticsearch is live.
### Check Data in Elasticsearch
```
curl http://localhost:9200/_cat/indices?v
```
---

### **Step 3: Understand and Configure Logstash**

**Concept**: Logstash collects, parses, and transforms logs before indexing them in Elasticsearch.

📄 `logstash/logstash.conf`

```conf
input {
  file {
    path => "/usr/share/logstash/pipeline/nginx-service/logs/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain
    type => "nginx"
  }

  file {
    path => "/usr/share/logstash/pipeline/product-service/app.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json
    type => "product"
  }

  file {
    path => "/usr/share/logstash/pipeline/order-service/app.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json
    type => "order"
  }
}

filter {
  if [type] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    mutate {
      add_field => { "log_type" => "nginx-access" }
    }
  } else if [type] == "product" or [type] == "order" {
    mutate {
      add_field => { "log_type" => "json-app-log" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "ecommerce-logs-%{+YYYY.MM.dd}"
  }
}

```

This config listens on port `5000` for logs, parses them using **Grok**, and sends them to **Elasticsearch**.

---

### **Step 4: Understand and Use Filebeat (Optional)**

**Concept**: Filebeat is a lightweight log shipper that forwards logs to Logstash or Elasticsearch.

Use if your microservices are not sending logs directly.

```bash
filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/nginx/*.log

output.logstash:
  hosts: ["localhost:5000"]
```

---

### **Step 5: Generate Logs from Services**

**Concept**: Applications generate logs (access, error, custom events). You centralize these using Logstash/Filebeat.

📄 `nginx-service/logs/access.log`

```log
127.0.0.1 - - [20/May/2025:10:00:01 +0000] "GET /products HTTP/1.1" 200 532 "-" "Mozilla/5.0"
127.0.0.1 - - [20/May/2025:10:00:03 +0000] "POST /orders HTTP/1.1" 201 128 "-" "Mozilla/5.0"
127.0.0.1 - - [20/May/2025:10:00:05 +0000] "GET /inventory HTTP/1.1" 200 256 "-" "Mozilla/5.0"
127.0.0.1 - - [20/May/2025:10:00:07 +0000] "GET /products?id=10 HTTP/1.1" 404 62 "-" "Mozilla/5.0"
127.0.0.1 - - [20/May/2025:10:00:09 +0000] "GET /cart HTTP/1.1" 500 91 "-" "Mozilla/5.0"

```

📄 `product-service/app.log`

```json
{"timestamp":"2025-05-20T10:00:01Z","level":"info","service":"product-service","message":"Fetched all products","status":200}
{"timestamp":"2025-05-20T10:00:04Z","level":"info","service":"product-service","message":"Fetched product ID 10","status":200}
{"timestamp":"2025-05-20T10:00:06Z","level":"warn","service":"product-service","message":"Product ID 999 not found","status":404}
{"timestamp":"2025-05-20T10:00:07Z","level":"error","service":"product-service","message":"Database connection timeout","status":500}
{"timestamp":"2025-05-20T10:00:08Z","level":"info","service":"product-service","message":"Added new product","status":201}

```

📄 `order-service/app.log`
```json
{"timestamp":"2025-05-20T10:00:03Z","level":"info","service":"order-service","message":"Order 123 created","status":201}
{"timestamp":"2025-05-20T10:00:05Z","level":"info","service":"order-service","message":"Order 124 created","status":201}
{"timestamp":"2025-05-20T10:00:06Z","level":"warn","service":"order-service","message":"Payment service delayed","status":202}
{"timestamp":"2025-05-20T10:00:07Z","level":"error","service":"order-service","message":"Failed to reserve stock","status":500}
{"timestamp":"2025-05-20T10:00:09Z","level":"info","service":"order-service","message":"Order 125 created","status":201}
```

---

### **Step 6: Explore Logs in Kibana**

**Concept**: Kibana visualizes logs from Elasticsearch.

* Open: [http://localhost:5601](http://localhost:5601)
* Go to **Discover**
* Select index pattern: `ecommerce-logs-*`
* Analyze logs: Search for 404 errors, slow requests, etc.
* View logs from NGINX, product-service, and order-service
Filter by service: nginx, product-service, or order-service


Example query:

```
status: 500
```

---

### **Step 7: Create Dashboards in Kibana**

**Concept**: Visual dashboards help monitor service health and patterns.

Create:

* Pie chart: `status` code distribution
* Line chart: `response time` trends
* Data table: Top URLs by hits

---

### **Step 8: Add Alerting (Optional Advanced)**

Use **Watcher** in Kibana (requires basic license) to set alerts.

Example:

* Alert if > 10 errors in 5 minutes
* Alert if response time > 2s

---

## 📈 Final Result:

You now have:

✅ Centralized logs from multiple services
✅ Real-time log parsing and indexing
✅ Searchable logs with filters
✅ Dashboards for observability

---

## 🧠 Summary of ELK Concepts:

| Concept                  | Tool          | Role                                           |
| ------------------------ | ------------- | ---------------------------------------------- |
| Storage & Index          | Elasticsearch | Search engine for logs                         |
| Ingestion & Parsing      | Logstash      | Pipeline to collect and transform logs         |
| Visualization            | Kibana        | UI for logs and dashboards                     |
| Lightweight Log Shipping | Filebeat      | Sends logs from apps to Logstash/Elasticsearch |

---


## ✅ Enhanced ELK Workflow Summary (by Infrastructure + Log Types)

| Infrastructure Component                | Logs Generated                             | Recommended Pipeline                                                                   | Reason                                                              |
| --------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| 🐳 Docker (Containers)                  | JSON stdout logs via Docker logging driver | `Filebeat → Elasticsearch`                                                             | Logs are already structured; no parsing needed                      |
| 🌐 NGINX / Apache Web Server            | Access & error logs (plain text)           | `Filebeat → Logstash → Elasticsearch`                                                  | Needs parsing with Grok to extract IPs, URLs, methods, status codes |
| 🖥 Linux Servers (Syslog, Auth)         | `/var/log/syslog`, `/var/log/auth.log`     | `Filebeat → Logstash → Elasticsearch`                                                  | Parsing needed + routing/filtering by log type                      |
| ☁️ Cloud (AWS CloudTrail, Azure Logs)   | JSON logs (structured)                     | `Filebeat → Elasticsearch` or `Logstash → Elasticsearch`                               | Use Filebeat if possible; else parse raw logs via Logstash          |
| 🐍 Node.js / Python / Java apps         | Custom logs in JSON or plain text          | If JSON: `Filebeat → Elasticsearch`<br>If plain: `Filebeat → Logstash → Elasticsearch` | Use Logstash only if app doesn't log in structured JSON             |
| 🛠 Kubernetes (Kubelet, container logs) | Structured container logs                  | `Filebeat → Elasticsearch`                                                             | Filebeat Kubernetes module handles parsing                          |
| 🔁 Kafka Queue (Log events)             | Streaming log events (JSON or raw)         | `Logstash → Elasticsearch`                                                             | Filebeat doesn’t ingest from Kafka directly — Logstash does         |
| 📦 Application with compliance concerns | PII or sensitive data in logs              | `Filebeat → Logstash (masking/filtering) → Elasticsearch`                              | Mask secrets/emails/IPs in Logstash before storing                  |


---

Would you like a GitHub-ready version of this project or a script to auto-generate these files?
