# SonarQube_Installation_Steps
Before we dive in, make sure you have:

- ✅ An **AWS account** (free tier works for testing)
- ✅ A **key pair** created in EC2 for SSH access
- ✅ Basic knowledge of **Linux commands**
- ✅ A machine with an SSH client (Terminal on Mac/Linux, PuTTY or Windows Terminal on Windows)

For this tutorial, we'll use:
- **Instance type:** `t3.medium` (minimum 2 vCPUs, 4 GB RAM — SonarQube needs this)
- **OS:** Ubuntu 22.04 LTS
- **SonarQube version:** Community Edition (free)

> ⚠️ *SonarQube will NOT run on t2.micro — it needs at least 4GB RAM.*

---

## 🟡 SECTION 2: Launch an EC2 Instance

### Step 1 — Log into AWS Console

Head over to [console.aws.amazon.com](https://console.aws.amazon.com) and log in.

### Step 2 — Go to EC2

In the search bar, type **EC2** and click on it. Then click **"Launch Instance"**.

### Step 3 — Configure the Instance

Fill in the following:

| Setting | Value |
|---|---|
| Name | `sonarqube-server` |
| AMI | Ubuntu Server 22.04 LTS |
| Instance Type | `t3.medium` |
| Key Pair | Select your existing key pair |
| Storage | 20 GB (gp3) |

### Step 4 — Configure Security Group

Create a new security group with these **inbound rules**:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |
| Custom TCP | 9000 | 0.0.0.0/0 (or your IP) |

> 💡 *Port 9000 is SonarQube's default web port.*

Click **"Launch Instance"** and wait for it to go into the **Running** state.

---

## 🟡 SECTION 3: Connect to the Instance 
Once your instance is running, grab its **Public IPv4 address** from the EC2 dashboard.

Open your terminal and SSH in:

```bash
ssh -i /path/to/your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

> Replace `/path/to/your-key.pem` with your actual key file path.

If you get a permissions error, fix it with:

```bash
chmod 400 /path/to/your-key.pem
```

You should now be inside the EC2 instance. 

---

## 🟡 SECTION 4: Install Java 
SonarQube requires **Java 17**. Let's install it.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
```

Verify the installation:

```bash
java -version
```

You should see something like:
```
openjdk version "17.x.x" ...
```

---

## 🟡 SECTION 5: Configure System Settings 

**[SCREEN SHARE – Terminal]**

SonarQube needs some system-level tweaks. Let's set them.

### Increase Virtual Memory

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
```

To make these **permanent** across reboots:

```bash
sudo nano /etc/sysctl.conf
```

Add these lines at the bottom:

```
vm.max_map_count=524288
fs.file-max=131072
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

### Set Open File Limits

```bash
sudo nano /etc/security/limits.conf
```

Add:

```
sonarqube   -   nofile   131072
sonarqube   -   nproc    8192
```

---

## 🟡 SECTION 6: Install PostgreSQL 
SonarQube uses a database. We'll use **PostgreSQL**.

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

Now, create a database and user for SonarQube:

```bash
sudo -u postgres psql
```

Inside the PostgreSQL shell, run:

```sql
CREATE USER sonarqube WITH ENCRYPTED PASSWORD 'sonarpass';
CREATE DATABASE sonarqube OWNER sonarqube;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonarqube;
\q
```

> 💡 *You can change `sonarpass` to a stronger password — just remember it for later.*

---

## 🟡 SECTION 7: Download and Install SonarQube 
### Download SonarQube

```bash
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
sudo apt install unzip -y
sudo unzip sonarqube-10.4.1.88267.zip
sudo mv sonarqube-10.4.1.88267 sonarqube
```

### Create a Dedicated User

```bash
sudo adduser --system --no-create-home --group --disabled-login sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube
```

### Configure SonarQube

```bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Find and update these lines:

```properties
sonar.jdbc.username=sonarqube
sonar.jdbc.password=sonarpass
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

Save and exit.

---

## 🟡 SECTION 8: Create a Systemd Service 
Let's make SonarQube run as a **system service** so it starts automatically.

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

Paste the following:

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

Save and exit, then enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
```

Check the status:

```bash
sudo systemctl status sonarqube
```

You should see **"active (running)"**.

---

## 🟡 SECTION 9: Access SonarQube in the Browser 
Open your browser and navigate to:

```
http://<YOUR_EC2_PUBLIC_IP>:9000
```

> ⏳ *It may take 1–2 minutes to fully start. If you see a loading screen, just wait.*

You'll be greeted with the SonarQube login page!

**Default credentials:**
- **Username:** `admin`
- **Password:** `admin`

You'll be prompted to **change your password** on first login. Do that!

🎉 **Congratulations — SonarQube is live on AWS!**

---

## 🟡 SECTION 10: Quick Tour of SonarQube 
Let me give you a super quick tour:

- **Projects** — where your scanned codebases appear
- **Issues** — bugs, vulnerabilities, code smells
- **Rules** — the quality rules SonarQube enforces
- **Administration** — manage users, tokens, plugins

From here, you can connect it to **Jenkins, GitHub Actions, GitLab CI**, and more for automated code scanning.

---

