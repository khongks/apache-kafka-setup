# Apache Kafka 4.x on RHEL: End-to-End Setup Guide

This guide covers the installation of Java 17, the configuration of Kafka in **KRaft mode** (no ZooKeeper), and the automation of the service using `systemd`.

## 1. Prerequisites: Java 17 Installation
Kafka 4.x requires Java 17 or 21. On RHEL, use `dnf` to install and `alternatives` to set the version.

```bash
# Install OpenJDK 17
sudo dnf install java-17-openjdk-devel -y

# Set Java 17 as the default
sudo alternatives --config java
# (Select the number corresponding to Java 17)

# Verify
java -version
```

---

## 2. Kafka Installation & KRaft Configuration
Using **Kafka 4.x**, the configuration files are located directly in the `config/` directory.

### Download and Extract
```bash
cd /root
wget https://archive.apache.org/dist/kafka/4.2.0/kafka_2.13-4.2.0.tgz
tar -xzf kafka_2.13-4.2.0.tgz
cd kafka_2.13-4.2.0
```

### Format Storage (The KRaft Way)
Generate a unique Cluster ID and format the storage directory using the `--standalone` flag for a single-node setup.

```bash
# Generate ID
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

# Format Storage using the new Kafka 4.x pathing
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
```

---

## 3. Background Automation (Systemd)
Create a service file to ensure Kafka starts on boot and runs in the background.

**File Path:** `/etc/systemd/system/kafka.service`

```ini
[Unit]
Description=Apache Kafka Server
After=network.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/kafka_2.13-4.2.0
ExecStart=/root/kafka_2.13-4.2.0/bin/kafka-server-start.sh /root/kafka_2.13-4.2.0/config/server.properties
ExecStop=/root/kafka_2.13-4.2.0/bin/kafka-server-stop.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Activate Service
```bash
sudo systemctl daemon-reload
sudo systemctl start kafka
sudo systemctl enable kafka
```

---

## 4. Verification & Testing
Test the end-to-end data flow using the built-in console scripts.

### Create a Topic
```bash
bin/kafka-topics.sh --create --topic test-events --bootstrap-server localhost:9092
```

### Produce and Consume
1. **Producer:** `bin/kafka-console-producer.sh --topic test-events --bootstrap-server localhost:9092`
2. **Consumer:** `bin/kafka-console-consumer.sh --topic test-events --from-beginning --bootstrap-server localhost:9092`

---

## 5. Maintenance Commands
| Task | Command |
| :--- | :--- |
| **Check Status** | `systemctl status kafka` |
| **View Logs** | `journalctl -u kafka -f` |
| **Stop Service** | `systemctl stop kafka` |
| **List Topics** | `bin/kafka-topics.sh --list --bootstrap-server localhost:9092` |

---

### Firewall Note
If accessing from a remote machine, open port 9092:
```bash
sudo firewall-cmd --add-port=9092/tcp --permanent
sudo firewall-cmd --reload
```