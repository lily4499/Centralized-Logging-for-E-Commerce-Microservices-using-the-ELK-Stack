
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
docker-compose down

docker-compose up -d
# watch the logs
docker logs -f elk-ecommerce-logs_logstash_1
```
![image](https://github.com/user-attachments/assets/f7fc966d-6cf0-4e08-993f-3ec969317734)

---

### **Step 2: Understand and Configure Elasticsearch**

**Concept**: Elasticsearch is a NoSQL search engine. It indexes logs sent from Logstash or Beats.

✔️ Example:

```bash
curl http://localhost:9200
```
![image](https://github.com/user-attachments/assets/e3d2d291-4877-4fa1-a7de-0ab4e2c63a17)

Returns cluster metadata confirming Elasticsearch is live.
![image](https://github.com/user-attachments/assets/7ea036db-51ab-49d6-9d19-354b79423beb)


### Check Data in Elasticsearch
```
curl http://localhost:9200/_cat/indices?v
```
![image](https://github.com/user-attachments/assets/39520ed2-7499-4c9e-82d8-2a14542af338)

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
      add_field => {
        "log_type" => "nginx-access"
        "service"  => "nginx"
      }
    }
  } else if [type] == "product" {
    mutate {
      add_field => {
        "log_type" => "json-app-log"
        "service"  => "product-service"
      }
    }
  } else if [type] == "order" {
    mutate {
      add_field => {
        "log_type" => "json-app-log"
        "service"  => "order-service"
      }
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

Here’s a **complete step-by-step guide** to play around with your logs using **Kibana’s Discover tab**, once your logs are flowing into **Elasticsearch** via Filebeat + Logstash.

---

## 🔍 Step-by-Step: Analyze Logs in Kibana

### ✅ Step 1: Open Kibana

In your browser, go to:

```
http://localhost:5601
```

If you're running Docker in WSL or a remote host, replace `localhost` with the appropriate IP.

---

### ✅ Step 2: Create an Index Pattern

1. Click on **“Stack Management”** (⚙️ gear icon in the sidebar)

2. Under **“Kibana” → “Data Views”**, click **Create data view**

3. Enter this as the pattern:

   ```
   ecommerce-logs-*
   ```

4. For **timestamp field**, choose:

   ```
   @timestamp
   ```

   (If missing, select `timestamp` or skip timestamp selection)

5. Click **Create data view**
![image](https://github.com/user-attachments/assets/cbf968ed-cddf-4be6-92c0-4442cf687989)

---

### ✅ Step 3: Go to Discover

1. Click the **“Discover”** tab in the left sidebar.
2. In the top-left dropdown, select:

   ```
   ecommerce-logs-*
   ```
3. You’ll now see logs ingested from:

   * `nginx-service`
   * `product-service`
   * `order-service`
![image](https://github.com/user-attachments/assets/0dad95b7-45c5-4aee-8f1c-d0f40f1cdd8d)


---

### ✅ Step 4: Explore Logs by Service

To filter logs by service:

#### 🟢 NGINX logs:

```
service: "nginx"
```

#### 🟡 Product logs:

```
service: "product-service"
```

#### 🔵 Order logs:

```
service: "order-service"
```

⏩ You can also add filters by clicking the **field name** (e.g., `service`) in the left panel, then clicking a value like `nginx`.
![image](https://github.com/user-attachments/assets/390743d8-b03e-45a2-aba6-a30d84625829)

---

### ✅ Step 5: Search Specific Patterns

In the top search bar:

| What You Want to Find                | Example Query                            |
| ------------------------------------ | ---------------------------------------- |
| All 404 errors                       | `status:404`                             |
| All 500 server errors                | `status:500`                             |
| Logs from product-service only       | `service:"product-service"`              |
| 500 errors from order service        | `service:"order-service" AND status:500` |
| Logs containing “timeout”            | `message:*timeout*`                      |
| Logs with slow responses (if parsed) | `response_time_ms:[1000 TO *]`           |

![image](https://github.com/user-attachments/assets/5f9c2506-3d57-44a7-8b3c-5385a29f1557)

---

### ✅ Step 6: Adjust Time Range

In the top-right corner:

* Click the **time filter** (e.g., “Last 15 minutes”)
* Select a broader range like:

  * “Last 1 hour”
  * “Last 24 hours”
  * “Today”

This helps ensure logs are visible depending on your ingestion timing.

---

### ✅ Step 7: Customize Table Columns

1. In the left sidebar (Fields), click the **➕ icon** next to fields you want to display:

   * `service`
   * `status`
   * `message`
   * `timestamp`
   * `clientip` (from nginx logs)
2. They will appear in your results table for easier analysis.
![image](https://github.com/user-attachments/assets/1abf4b58-b032-451c-b218-7eee9c10e63b)

---

### ✅ Step 8: Save Your View (Optional)

* Click the **Save** icon at the top
* Give your search a name like:

  ```
  All 500 Errors by Service
  ```

You can now reuse it any time.
![image](https://github.com/user-attachments/assets/1ab594c0-6672-4f6a-9bfb-37bfd331b530)

---


### **Step 7: Create Dashboards in Kibana**

**Concept**: Visual dashboards help monitor service health and patterns.

 **create a Kibana dashboard** that visually summarizes your logs from NGINX, product-service, and order-service.

---

## 📊 Step-by-Step: Build a Dashboard in Kibana

### ✅ Step 1: Open the Dashboard Section

1. In Kibana, go to **Dashboard** (📊 icon in the sidebar)
2. Click **Create Dashboard**
3. Click **“+ Create new visualization”**

---

## 🔧 Now create these useful visualizations:

---

### 📌 1. **Pie Chart: Errors by Service**

> Shows how many `500` or `404` errors each service produces.

1. Select **Pie**
2. Data view: `ecommerce-logs-*`
3. **Filter (optional):**

   ```kql
   status: (404 or 500)
   ```
4. **Buckets:**

   * Aggregation: Terms
   * Field: `service.keyword`
5. Click **Save** → Name: `Errors by Service`
6. 
![image](https://github.com/user-attachments/assets/f34caaa6-746f-4e90-98d8-f50c495328b9)

---

### 📌 2. **Bar Chart: Top Requested URLs (NGINX)**

> See the most frequently accessed endpoints.

1. Visualization type: **Bar**
2. Data view: `ecommerce-logs-*`
3. **Filter (optional):**

   ```kql
   service: "nginx"
   ```
4. **X-axis:**

   * Aggregation: Terms
   * Field: `request.keyword`
5. **Y-axis:**

   * Aggregation: Count
6. Click **Save** → Name: `Top NGINX URLs`

---

### 📌 3. **Line Chart: 500 Errors Over Time**

1. Type: **Line**
2. Filter:

   ```kql
   status: 500
   ```
3. **X-axis:**

   * Date Histogram on `@timestamp`
4. **Y-axis:**

   * Count
5. Split Series (optional): `service.keyword`
6. Save as `500 Errors Over Time`
7. 
![image](https://github.com/user-attachments/assets/83f3781e-40eb-4e05-884b-7dd7ba65bacf)

---

### 📌 4. **Metric: Total Logs Ingested**

1. Type: **Metric**
2. Aggregation: `Count`
3. Save as: `Total Logs`

---

### 📌 5. **Data Table: Recent 404/500 Errors**

1. Type: **Data Table**
2. Filter:

   ```kql
   status: (404 or 500)
   ```
3. Columns:

   * `@timestamp`
   * `service`
   * `status`
   * `message` (or `request`)
4. Save as: `Recent Errors Table`

---

### ✅ Step 2: Add Visualizations to Dashboard

1. Go back to your dashboard
2. Click **Add from library**
3. Select all saved visualizations (e.g., `Errors by Service`, `Top URLs`, etc.)
4. Drag and resize as needed
5. Click **Save Dashboard** — name it something like:

   ```
   E-Commerce Log Dashboard
   ```

---

### 🧪 Final Touch: Filter by service or time

* Use the filter bar at the top:

  * `service: "product-service"`
  * `status: 500`
* Change the time range (top-right) to match log activity.

---

Would you like me to generate Kibana `.ndjson` export files so you can import these prebuilt visualizations and dashboard instantly?


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

## ✅ When to Use Only `Logstash → Elasticsearch`

### 🧠 You can skip Filebeat if:

| Condition                                                  | Explanation                                                            |
| ---------------------------------------------------------- | ---------------------------------------------------------------------- |
| 🗃 You already know where the logs are (e.g., local files) | Logstash can read files directly using its `file` input plugin         |
| 🔄 You want fewer moving parts                             | Filebeat is optional; Logstash alone can ingest, parse, and send to ES |
| 📊 You need to parse, transform, and enrich all logs       | Filebeat has limited processing — Logstash is more powerful            |
| 📦 You're not collecting logs from multiple remote sources | Filebeat shines when deployed to edge machines or containers           |

---



## ✅ Pros of `Logstash → Elasticsearch` Only

| Benefit                   | Why It Matters                          |
| ------------------------- | --------------------------------------- |
| 💡 Simpler architecture   | Fewer components, easier to maintain    |
| ⚙️ Full control           | Logstash can handle parsing and routing |
| 📂 Direct access to files | You don’t need agents like Filebeat     |

---

## ❗ Limitations (When You Might Need Filebeat)

| Limitation                                           | When Filebeat is better                            |
| ---------------------------------------------------- | -------------------------------------------------- |
| 📡 You need to collect logs from many remote servers | Filebeat is lightweight and installable everywhere |
| 🐳 You want to ship logs from Docker/K8s containers  | Filebeat has Kubernetes/Docker modules             |
| 📈 You want lightweight log shipping                 | Logstash is heavier than Filebeat                  |



---
---
# To send your log files (`access.log`, `app.log`) to **Elasticsearch** via **Filebeat → Logstash → Elasticsearch**, follow this step-by-step guide.

---

## ✅ Project Goal

You want to:

```
[access.log / app.log] → Filebeat → Logstash → Elasticsearch → Kibana
```

---

## 🧱 Directory Structure You Have

```
elk-ecommerce-logs/
├── docker-compose.yml
├── nginx-service/logs/access.log
├── product-service/app.log
├── order-service/app.log
├── logstash/logstash.conf
├── filebeat/filebeat.yml     👈 You’ll add this
└── kibana/
```

---

## ✅ STEP 1: Create `filebeat.yml`

📄 `elk-ecommerce-logs/filebeat/filebeat.yml`

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /usr/share/filebeat/logs/nginx/access.log
    fields:
      service: nginx
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /usr/share/filebeat/logs/product/app.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      service: product-service
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /usr/share/filebeat/logs/order/app.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      service: order-service
    fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]
```

---

## ✅ STEP 2: Update `logstash.conf`

📄 `elk-ecommerce-logs/logstash/logstash.conf`

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [service] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
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

---

## ✅ STEP 3: Update `docker-compose.yml`

```yaml
services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.9
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.9
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./nginx-service/logs:/usr/share/filebeat/logs/nginx
      - ./product-service:/usr/share/filebeat/logs/product
      - ./order-service:/usr/share/filebeat/logs/order
    depends_on:
      - logstash
```

---

## ✅ STEP 4: Start Everything

```bash
docker-compose down -v
docker-compose up -d
```

Watch Filebeat logs:

```bash
docker logs -f <filebeat_container_name>
```

---

## ✅ STEP 5: View Logs in Kibana

1. Open: [http://localhost:5601](http://localhost:5601)
2. Go to **Discover**
3. Create index pattern: `ecommerce-logs-*`
4. Filter by `service: nginx`, `product-service`, or `order-service`

---




