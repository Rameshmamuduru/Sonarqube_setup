#  REAL-TIME PROD SETUP

**SonarQube + PostgreSQL + Jenkins Integration**

---

# 🧱 ARCHITECTURE (PROD)

```
Developer
   ↓
GitHub
   ↓
Jenkins (Master + Agent)
   ↓
SonarQube (PostgreSQL DB)
   ↓
Quality Gate
   ↓
Deploy / Artifact
```

---

## PART 1️⃣ Server & Prerequisites (Fresh VM)

### Minimum Requirements

| Component | Requirement             |
| --------- | ----------------------- |
| OS        | Ubuntu 20.04 / 22.04    |
| RAM       | 4 GB (8 GB recommended) |
| Java      | Java 17                 |
| DB        | PostgreSQL              |
| Port      | 9000                    |

---

## PART 2️⃣ Install Java 17 (MANDATORY)

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

✅ SonarQube **will NOT work** without Java 17 in PROD.

---

## PART 3️⃣ Install PostgreSQL (PROD DB)

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Create SonarQube Database

```bash
sudo -i -u postgres
psql
```

```sql
CREATE DATABASE sonarqube;
CREATE USER sonar WITH ENCRYPTED PASSWORD 'sonar123';
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
exit
```

---

## PART 4️⃣ Kernel & System Tuning (VERY IMPORTANT)

# WHY?
    Sonarqube uses elastc search, so linux kurnal parameters like vm_max_map_counts and file limites can be increased for stable poduction operations.

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```

Persist:

```bash
sudo nano /etc/sysctl.conf
```

Add:

```
vm.max_map_count=524288
fs.file-max=131072
```

---

## PART 5️⃣ Install SonarQube (PROD WAY)

```bash
cd /opt
  sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.1.0.118079.zip
sudo unzip sonarqube-*.zip
sudo mv sonarqube-* sonarqube
```
https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.1.0.118079.zip

### Create Dedicated User

```bash
sudo useradd sonar
sudo chown -R sonar:sonar /opt/sonarqube
```

---

## PART 6️⃣ Configure SonarQube with PostgreSQL

Edit config:

```bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Add / update:

```
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube

sonar.web.port=9000
```

---

## PART 7️⃣ Run SonarQube as a Service (PROD)

Create service file:

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

Paste:

```
[Unit]
Description=SonarQube service
After=network.target

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
Restart=always
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

Enable & start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
sudo systemctl status sonarqube
```

---

## PART 8️⃣ Access SonarQube

```
http://<server-ip>:9000
```

Default login:

* Username: `admin`
* Password: `admin` (change immediately)

---

## PART 9️⃣ Jenkins Integration (PROD)

### 1️⃣ Install SonarQube Plugin

* Jenkins → Manage Jenkins → Plugins
* Install **SonarQube Scanner**

---

### 2️⃣ Generate SonarQube Token

* SonarQube → My Account → Security
* Generate token → Copy

---

### 3️⃣ Add Token in Jenkins

* Jenkins → Credentials
* Add **Secret Text**
* Paste token

---

### 4️⃣ Configure SonarQube Server

* Jenkins → Manage Jenkins → Configure System
* SonarQube Servers:

  * Name: `SonarQube`
  * URL: `http://<sonarqube-ip>:9000`
  * Credentials: token

---

### 5️⃣ Configure Sonar Scanner

* Jenkins → Global Tool Configuration
* SonarQube Scanner:

  * Name: `SonarScanner`
  * Install automatically ✅

---

## PART 🔟 Jenkins Pipeline (PROD-READY)

### Maven Project Pipeline

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
```

---

## ✅ THIS IS 100% PROD-READY

✔ External DB (PostgreSQL)
✔ Service-based SonarQube
✔ Secure token auth
✔ Quality Gate enforcement
✔ Jenkins CI integration

