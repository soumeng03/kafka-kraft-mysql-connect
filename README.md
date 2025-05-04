# kafka-kraft-mysql-connect
Simplifying Streaming Architectures: Apache Kafka, KRaft, Kafka Connect and MySQL on Kubernetes

# Deploy Kafka in KRaft Mode with Kafka Connect and MySQL Sink on Minikube

This guide walks you through deploying Apache Kafka in KRaft mode, using Kafka Connect to sink messages into a MySQL database, all within a Minikube environment.

## 🧰 Prerequisites

- Minikube (latest)
- kubectl
- Docker
- Kafka Connect image (e.g., Bitnami)
- MySQL JDBC driver (included in Kafka Connect image or added manually)

---

## 🚀 Step 1: Start Minikube

```bash
minikube start --cpus=4 --memory=8g
```

---

## 🏗️ Step 2: Deploy MySQL

```bash
kubectl apply -f 02-mysql.yaml
```

This will:
- Deploy MySQL with database `kafka_connect_db`
- Set root password to `password`
- Expose it on internal Kubernetes service `mysql:3306`

---

## 🧱 Step 3: Deploy Kafka in KRaft Mode

```bash
kubectl apply -f 01-kafka-kraft.yaml
```

This deploys:
- Kafka running in KRaft (no ZooKeeper)
- A single broker setup
- Exposed via NodePort on `30092`

---

## 🔌 Step 4: Deploy Kafka Connect

```bash
kubectl apply -f 03-kafka-connect.yaml
```

Ensure:
- Kafka Connect has network access to Kafka (`kafka:9092`) and MySQL (`mysql:3306`)
- MySQL JDBC driver is present in the image or mounted

---

## 🔗 Step 5: Register the JDBC Sink Connector

Prepare the JSON config (`mysql-sink-config.json`):

```json
{
  "name": "mysql-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "mysql-sink-topic",
    "connection.url": "jdbc:mysql://mysql:3306/kafka_connect_db",
    "connection.user": "root",
    "connection.password": "password",
    "insert.mode": "insert",
    "auto.create": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}
```

Register the connector:

```bash
curl -X POST -H "Content-Type: application/json" --data @mysql-sink-config.json http://localhost:8083/connectors
```

---

## 📊 Step 6: Check Connector Status

```bash
curl http://localhost:8083/connectors/mysql-sink/status
```

---

## 📥 Step 7: Produce a Sample Kafka Message

Access the Kafka pod:

```bash
kubectl exec -it kafka-0 -- bash
```

Produce a message:

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic mysql-sink-topic
```

Paste:

```json
{"id": 1, "name": "Alice"}
```

Exit with `Ctrl+D`.

---

## 🧾 Step 8: Verify Data in MySQL

Port forward MySQL:

```bash
kubectl port-forward svc/mysql 3306:3306
```

Connect with your preferred SQL client or:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -ppassword kafka_connect_db
```

Run:

```sql
SELECT * FROM mysql_sink_topic;
```

You should see your inserted data.

---

## 🧹 Optional: Clean Up

Delete topic:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic mysql-sink-topic
```

Delete connector:

```bash
curl -X DELETE http://localhost:8083/connectors/mysql-sink
```

---

## ✅ Summary

You’ve successfully:
- Deployed Kafka in KRaft mode
- Set up Kafka Connect
- Integrated a MySQL sink
- Produced and verified Kafka messages

This is a solid base for production-grade streaming pipelines using Kubernetes-native deployments.
