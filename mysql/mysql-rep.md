# MySQL 5.6 Replication Setup

## Region: 10.75  
**Architecture:**  
Master (192.168.0.99) → Forward (10.75.0.18) → Replica (172.17.9.9 Docker)

---

## 📌 Scenario

- **Master DB:** `192.168.0.99`
- **Forward Server:** `10.75.0.18`
- **Replica Server:** `172.17.9.9`
- **MySQL Version:** 5.6 (Docker)
- Each Docker container = One Replica per region

---

# 🧱 Step 1 — Port Forward on 10.75

On server `10.75.0.18`:

## Install socat

```bash
sudo apt install socat -y
Run Forward
sudo socat TCP-LISTEN:3306,fork,reuseaddr TCP:192.168.0.99:3306
Create systemd Service
sudo nano /etc/systemd/system/mysql3306-forward.service
[Unit]
Description=Forward 3306 -> 192.168.0.99:3306
After=network.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:3306,fork,reuseaddr TCP:192.168.0.99:3306
Restart=always

[Install]
WantedBy=multi-user.target
sudo systemctl daemon-reload
sudo systemctl enable mysql3306-forward
sudo systemctl start mysql3306-forward
🔐 Restrict Port 3306 Access

Allow only Replica (172.17.9.9):

sudo iptables -I INPUT 1 -p tcp -s 172.17.9.9 --dport 3306 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
Persist Rules
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
🐳 Step 2 — Setup Docker Replica on 172.17.9.9
Create Data Directory (10TB Storage)
sudo mkdir -p /data-mysql/replicate-10-75
sudo chown -R 999:999 /data-mysql
docker-compose.yml

Path:

/data-mysql/replicate-10-75/docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:5.6
    container_name: replicate-10-75
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 6taK..@213
    volumes:
      - ./data:/var/lib/mysql
      - ./conf:/etc/mysql/conf.d
    ports:
      - "3307:3306"
Replica MySQL Configuration

Path:

/data-mysql/replicate-10-75/conf/my.cnf
[mysqld]
server-id=9001
relay-log=relay-bin
log-bin=mysql-bin
read_only=1
skip-name-resolve=1
Start Container
cd /data-mysql/replicate-10-75
docker compose up -d
📦 Step 3 — Restore Backup
Extract Backup
cd /data-mysql/replicate-10-75/backup
tar -xzf 23-6a1f43e4bbf689311cd7c204c337cf0c.tar.gz
Create Databases
docker exec -it replicate-10-75 mysql -uroot -p'6taK..@213' -e "
CREATE DATABASE IF NOT EXISTS anbar;
CREATE DATABASE IF NOT EXISTS baskool;
CREATE DATABASE IF NOT EXISTS tranzit;
"
Parallel Import (8 Threads)
cd /data-mysql/replicate-10-75/backup/2026/02/23/Mysql

ls backup_*.sql.gz | xargs -P8 -I{} bash -c '
  db=$(echo "{}" | sed -E "s/^backup_(.*)\.sql\.gz$/\1/")
  gunzip -c "{}" | docker exec -i replicate-10-75 mysql -uroot -p"6taK..@213" "$db"
'
🔑 Step 4 — Create Replication User on Master

On 192.168.0.99:

CREATE USER 'repl'@'192.168.0.%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.0.%';
FLUSH PRIVILEGES;
Check Master Status
SHOW MASTER STATUS;

Example:

File: mysql-bin.010857
Position: 383147024
🔗 Step 5 — Connect Replica to Master

On Replica:

docker exec -it replicate-10-75 mysql -uroot -p'6taK..@213'
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST='10.75.0.18',
  MASTER_PORT=3306,
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_LOG_FILE='mysql-bin.010857',
  MASTER_LOG_POS=383147024;

START SLAVE;
✅ Verify Replication Status
docker exec -it replicate-10-75 mysql -uroot -p'6taK..@213' -e "SHOW SLAVE STATUS\G"

Expected:

Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
🧨 Fix "Table doesn't exist" Error

If you see:

Last_SQL_Error: Table 'tranzit.SodurePateDarbeKhoruj' doesn't exist
On Master:
mysqldump -u root -p --single-transaction tranzit SodurePateDarbeKhoruj | gzip > table.sql.gz
On Replica:
gunzip -c table.sql.gz | docker exec -i replicate-10-75 mysql -uroot -p'6taK..@213' tranzit

Then:

START SLAVE SQL_THREAD;
🧪 Final Replication Test
On Master:
INSERT INTO test.repl_check VALUES (1);
On Replica:
SELECT * FROM test.repl_check;
📊 Healthy Replication Indicators
On Master:
SHOW PROCESSLIST;

Should show:

User: repl
Command: Binlog Dump
On Replica:
Slave_IO_State: Waiting for master to send event
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
🏁 Final Result

Region 10.75 Replica is fully operational with:

Docker-based MySQL 5.6

Dedicated 10TB Storage

Port Forward via socat

Firewall Restricted Access

Parallel Restore

Live Replication Active