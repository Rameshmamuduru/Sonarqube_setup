Great question 👍
Let’s do a **PROD-grade SonarQube setup from scratch using Docker images**, the **right way** (the way companies actually do it).

I’ll explain **why** each component is needed and then give you **exact commands**.

---

# 🧱 Production SonarQube Architecture (Docker)

In **PROD**, SonarQube **MUST NOT** use the embedded H2 DB.

**Correct PROD setup:**

```
SonarQube  →  PostgreSQL
   |
 Docker container
```

✅ Separate DB
✅ Persistent volumes
✅ Non-root execution
✅ Proper memory sizing

---

# 📋 Minimum System Requirements (PROD – Small)

For a **small production / team setup**:

| Component | Requirement                         |
| --------- | ----------------------------------- |
| RAM       | **4 GB recommended** (minimum 3 GB) |
| CPU       | 2 vCPU                              |
| Disk      | 20–30 GB                            |
| OS        | Linux (Ubuntu 20.04+)               |
| Docker    | Latest                              |

> ⚠️ With **2 GB RAM**, SonarQube will be unstable in PROD.

---

# 1️⃣ Install Docker & Docker Compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

Verify:

```bash
docker --version
docker-compose --version
```

---

# 2️⃣ Kernel & OS Tuning (VERY IMPORTANT)

SonarQube uses Elasticsearch → requires kernel tuning.

### Set `vm.max_map_count`

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
```

Persist it:

```bash
sudo nano /etc/sysctl.conf
```

Add:

```
vm.max_map_count=524288
fs.file-max=131072
```

Apply:

```bash
sudo sysctl -p
```

---

# 3️⃣ Create Directory Structure

```bash
mkdir -p ~/sonarqube-prod
cd ~/sonarqube-prod
```

---

# 4️⃣ Create Docker Compose (PROD-READY)

Create file:

```bash
nano docker-compose.yml
```

### ✅ **PROD SonarQube + PostgreSQL**

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15
    container_name: sonar-postgres
    restart: always
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - sonar-net

  sonarqube:
    image: sonarqube:9.9-community
    container_name: sonarqube
    restart: always
    depends_on:
      - postgres
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://postgres:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
      SONAR_ES_BOOTSTRAP_CHECKS_DISABLE: "true"
    volumes:
      - sonar-data:/opt/sonarqube/data
      - sonar-extensions:/opt/sonarqube/extensions
      - sonar-logs:/opt/sonarqube/logs
    ulimits:
      nofile:
        soft: 131072
        hard: 131072
      nproc:
        soft: 8192
        hard: 8192
    networks:
      - sonar-net

volumes:
  sonar-data:
  sonar-extensions:
  sonar-logs:
  postgres-data:

networks:
  sonar-net:
```

---

# 5️⃣ Start SonarQube (First Time)

```bash
docker-compose up -d
```

Check logs:

```bash
docker-compose logs -f sonarqube
```

Wait **2–3 minutes**.

---

# 6️⃣ Access SonarQube

Browser:

```
http://<SERVER-IP>:9000
```

Default credentials:

```
Username: admin
Password: admin
```

👉 You’ll be asked to change password immediately.

---

# 7️⃣ Memory Breakdown (Important)

### SonarQube PROD Memory Usage

| Component     | Memory  |
| ------------- | ------- |
| Elasticsearch | ~1.5 GB |
| Web + Compute | ~700 MB |
| PostgreSQL    | ~300 MB |
| OS + Docker   | ~500 MB |

✅ **Total recommended:** **4 GB RAM**

---

# 8️⃣ Production Best Practices

✔ Use **external PostgreSQL** (or managed RDS in AWS)
✔ Do **not run as root** (Docker handles this)
✔ Enable **backups of volumes**
✔ Put SonarQube behind **Nginx reverse proxy**
✔ Enable **HTTPS**
✔ Do not colocate Nexus + SonarQube on 2GB server

---

# 9️⃣ Useful PROD Commands

### Stop

```bash
docker-compose down
```

### Restart

```bash
docker-compose restart
```

### Upgrade SonarQube

```bash
docker-compose down
docker pull sonarqube:9.9-community
docker-compose up -d
```

---

# 🔥 Why Docker is Better than Manual Install

| Manual                   | Docker |
| ------------------------ | ------ |
| Java issues              | ❌      |
| Permission errors        | ❌      |
| Elasticsearch root issue | ❌      |
| Easy upgrades            | ❌      |

Docker → **clean, stable, PROD-ready**

---

## ✅ Next Steps (Recommended)

I can help you with:
1️⃣ **Nginx reverse proxy + SSL**
2️⃣ **SonarQube integration with Jenkins pipeline**
3️⃣ **SonarQube + GitHub/GitLab setup**
4️⃣ **Sizing for real enterprise PROD**

👉 Tell me **what you want next**, and I’ll guide you step by step like a real PROD DevOps setup.
