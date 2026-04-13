To set up **SASL/SCRAM** (Salted Challenge Response Authentication Mechanism) in Kafka 4.x, you need to configure the broker to challenge clients for credentials and then store those credentials in your cluster metadata.

Since you are using **KRaft mode**, you no longer use ZooKeeper to store users; everything is handled via the Kafka administrative tools.

---

### **1. Generate the SCRAM Credentials**
Before restarting the server, you need to create a user and a password. We will use the `kafka-configs.sh` tool to add a user (e.g., `admin`) to the cluster metadata.

Run this command (ensure your Kafka service is running):
```bash
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
--alter --add-config 'SCRAM-SHA-512=[password=admin-password]' \
--entity-type users --entity-name admin
```

---

### **2. Configure the Broker (`server.properties`)**
You need to tell Kafka to listen for SASL connections and define which mechanism to use. Open your config file:
`vi config/server.properties`

Add or update the following settings:

```ini
# 1. Define the listeners (using SASL_PLAINTEXT for internal/non-SSL testing)
listeners=PLAINTEXT://:9092,CONTROLLER://:9093,SASL_PLAINTEXT://:9094
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SASL_PLAINTEXT:SASL_PLAINTEXT
advertised.listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093,SASL_PLAINTEXT://localhost:9094

# 2. Enable SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# 3. Configure the JAAS file (tells Kafka how to authenticate itself)
listener.name.sasl_plaintext.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required;
```

---

### **3. Create a Client Config File**
To connect to a SASL-enabled Kafka, your producers and consumers need a configuration file (let's call it `client-auth.conf`) to provide their credentials.

`vi config/client-auth.conf`

Paste the following:
```ini
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-password";
```

---

### **4. Restart and Test**
1. **Restart Kafka:**
   ```bash
   sudo systemctl restart kafka
   ```

2. **Create test topic
   ```bash
   bin/kafka-topics.sh --create --topic test-sasl --bootstrap-server localhost:9092
   ```

3. **Test with a Producer:**
   Notice we now point to port **9094** and include the `--command-config` flag.
   ```bash
   bin/kafka-console-producer.sh --bootstrap-server localhost:9094 \
   --topic test-sasl \
   --command-config config/client-auth.conf
   ```

4. **Test with a Consumer:**
   ```bash
   bin/kafka-console-consumer.sh --bootstrap-server localhost:9094 \
   --topic test-sasl --from-beginning \
   --command-config config/client-auth.conf
   ```

---

### **Summary of Steps**
| Step | Action |
| :--- | :--- |
| **User Creation** | Use `kafka-configs.sh` to add user to metadata. |
| **Broker Config** | Set `sasl.enabled.mechanisms` and JAAS config in `server.properties`. |
| **Client Config** | Create a `.conf` file with the username/password. |
| **Access** | Use the `--producer.config` or `--consumer.config` flags in commands. |

Are you planning to wrap this in SSL/TLS as well, or is SASL over a plain connection sufficient for your current environment?