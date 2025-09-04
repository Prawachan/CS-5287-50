# CA0 Fraud Detection Demo

This project sets up a simple fraud-detection pipeline using **Kafka**, **MongoDB**, and **Python producers/processors** deployed across three VMs.  
Producers generate simulated ATM/POS transactions → Kafka Broker → Processor → MongoDB.

---

## Cloud & Infrastructure Setup

### Cloud Provider
- **Provider**: Azure
- **Region**: `East US 2`
- **Resource Group**: `rg-ca0-iot`
- **Virtual Network**: `vnet-ca0` (`10.42.0.0/24`)
- **Subnet**: `snet-core` (`10.42.0.0/25`)

### VM Specs

| VM        | OS           | Size             | Private IP | Role                  |
|-----------|--------------|------------------|------------|-----------------------|
| vm-app    | Ubuntu 22.04 | Standard_B2s     | 10.42.0.6  | Producers + Processor |
| vm-broker | Ubuntu 22.04 | Standard_B2s     | 10.42.0.4  | Kafka Broker          |
| vm-db     | Ubuntu 22.04 | Standard_B2s     | 10.42.0.5  | MongoDB Database      |

---

## VM Creation

Run from **Azure Cloud Shell**:

```bash
# vm-app
az vm create \
  --resource-group rg-ca0-iot \
  --name vm-app \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-ca0 \
  --subnet snet-core \
  --admin-username student \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --private-ip-address 10.42.0.6

# vm-broker
az vm create \
  --resource-group rg-ca0-iot \
  --name vm-broker \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-ca0 \
  --subnet snet-core \
  --admin-username student \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --private-ip-address 10.42.0.4

# vm-db
az vm create \
  --resource-group rg-ca0-iot \
  --name vm-db \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --vnet-name vnet-ca0 \
  --subnet snet-core \
  --admin-username student \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --private-ip-address 10.42.0.5
```

---

## Firewall Configuration

Restrict inbound access to **only essential ports**:

```bash
# On each VM
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing

# vm-app (connects to broker + db)
sudo ufw allow from 10.42.0.4 to any port 9092 proto tcp
sudo ufw allow from 10.42.0.5 to any port 27017 proto tcp

# vm-broker
sudo ufw allow from 10.42.0.6 to any port 9092 proto tcp

# vm-db
sudo ufw allow from 10.42.0.6 to any port 27017 proto tcp
```

Verify:
```bash
sudo ufw status verbose
```

---

## Components

| Component  | Image/Version     | Host      | Port   |
|------------|-------------------|-----------|--------|
| Kafka      | Confluent 7.5.0   | vm-broker | 9092   |
| MongoDB    | 7.0.x             | vm-db     | 27017  |
| Producers  | python:3.11-slim  | vm-app    | N/A    |
| Processor  | python:3.11-slim  | vm-app    | N/A    |

---

## Application Setup (on `vm-app`)

1. **Clone code into `~/ca0/app/`**
   ```bash
   cd ~
   mkdir -p ca0/app
   cd ca0/app
   ```

2. **Files included**
   - `producer_atm.py`
   - `producer_pos.py`
   - `processor.py`
   - `requirements.txt`
   - `Dockerfile.app`
   - `compose.yaml`

3. **Sample `compose.yaml`**
   ```yaml
   services:
     producer-atm:
       build: { context: ., dockerfile: Dockerfile.app }
       user: "1000:1000"
       command: ["python","producer_atm.py"]
       environment:
         KAFKA_BOOTSTRAP: "${BROKER_PRIV}:9092"
       restart: unless-stopped

     producer-pos:
       build: { context: ., dockerfile: Dockerfile.app }
       user: "1000:1000"
       command: ["python","producer_pos.py"]
       environment:
         KAFKA_BOOTSTRAP: "${BROKER_PRIV}:9092"
       restart: unless-stopped

     processor:
       build: { context: ., dockerfile: Dockerfile.app }
       user: "1000:1000"
       command: ["python","processor.py"]
       environment:
         KAFKA_BOOTSTRAP: "${BROKER_PRIV}:9092"
         MONGO_URI: "mongodb://10.42.0.5:27017/bankdb"
       restart: unless-stopped
   ```

4. **Build and start**
   ```bash
   docker compose up -d --build
   ```

---

## Verification

### Producers & Processor Logs
```bash
docker compose logs -f processor
```
You should see:
```
Inserted: <uuid> flagged=False
Inserted: <uuid> flagged=True
```

### MongoDB Verification
On `vm-db`:
```bash
mongosh --host 127.0.0.1 --port 27017
use bankdb
db.transactions.find().limit(5).pretty()
```

---

## Deviations From Reference Stack
- Used **Azure VMs** instead of local Docker only
- Explicit **private IP assignments** for predictability
- Restricted firewall rules for security
- Ran containers as non-root (`user: "1000:1000"`)  

---

## Demo Video
For the final demo:
1. Show producer containers starting (`docker compose up -d`)
2. Tail processor logs (`docker compose logs -f processor`)
3. Connect to MongoDB and run:
   ```bash
   db.transactions.find().limit(5).pretty()
   ```
4. Show flagged entries in the output.
