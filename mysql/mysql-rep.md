MySQL 5.6 Replication Setup
Region: 10.75
Architecture: Master (192.168.0.99) → Forward (10.75.0.18) → Replica (172.17.9.9 Docker)
📌 سناریو

Master DB: 192.168.0.99

Forward Server: 10.75.0.18

Replica Server: 172.17.9.9

MySQL Version: 5.6 (Docker)

هر کانتینر = یک Replica برای هر منطقه

🧱 مرحله 1 — Port Forward روی 10.75

روی سرور 10.75.0.18:

نصب socat
sudo apt install socat -y
اجرای Forward
sudo socat TCP-LISTEN:3306,fork,reuseaddr TCP:192.168.0.99:3306
تبدیل به سرویس systemd
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
🔐 محدود کردن دسترسی پورت 3306

فقط اجازه به Replica (172.17.9.9):

sudo iptables -I INPUT 1 -p tcp -s 172.17.9.9 --dport 3306 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP

Persist:

sudo apt install iptables-persistent -y
sudo netfilter-persistent save
🐳 مرحله 2 — راه‌اندازی Docker Replica روی 172.17.9.9
ساخت مسیر دیتا روی 10TB
sudo mkdir -p /data-mysql/replicate-10-75
sudo chown -R 999:999 /data-mysql
docker-compose.yml

مسیر:

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
تنظیمات MySQL Replica

مسیر:

/data-mysql/replicate-10-75/conf/my.cnf
[mysqld]
server-id=9001
relay-log=relay-bin
log-bin=mysql-bin
read_only=1
skip-name-resolve=1
اجرای کانتینر
cd /data-mysql/replicate-10-75
docker compose up -d
📦 مرحله 3 — Restore Backup
Extract
cd /data-mysql/replicate-10-75/backup
tar -xzf 23-6a1f43e4bbf689311cd7c204c337cf0c.tar.gz
ساخت دیتابیس‌ها
docker exec -it replicate-10-75 mysql -uroot -p'6taK..@213' -e "
CREATE DATABASE IF NOT EXISTS anbar;
CREATE DATABASE IF NOT EXISTS baskool;
CREATE DATABASE IF NOT EXISTS tranzit;
"
Import موازی (8 Thread)
cd /data-mysql/replicate-10-75/backup/2026/02/23/Mysql

ls backup_*.sql.gz | xargs -P8 -I{} bash -c '
  db=$(echo "{}" | sed -E "s/^backup_(.*)\.sql\.gz$/\1/")
  gunzip -c "{}" | docker exec -i replicate-10-75 mysql -uroot -p"6taK..@213" "$db"
'
🔑 مرحله 4 — ساخت User Replication روی Master (192.168.0.99)
CREATE USER 'repl'@'192.168.0.%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.0.%';
FLUSH PRIVILEGES;
🔍 بررسی Master
SHOW MASTER STATUS;

مثال:

mysql-bin.010857
Position: 383147024
🔗 مرحله 5 — اتصال Replica به Master

روی Replica:

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
✅ بررسی وضعیت Replication
docker exec -it replicate-10-75 mysql -uroot -p'6taK..@213' -e "SHOW SLAVE STATUS\G"

باید:

Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
🧨 رفع خطای Table doesn't exist

اگر دیدی:

Last_SQL_Error: Table 'tranzit.SodurePateDarbeKhoruj' doesn't exist
راه درست:

روی Master:

mysqldump -u root -p --single-transaction tranzit SodurePateDarbeKhoruj | gzip > table.sql.gz

روی Replica:

gunzip -c table.sql.gz | docker exec -i replicate-10-75 mysql -uroot -p'6taK..@213' tranzit

بعد:

START SLAVE SQL_THREAD;
🧪 تست نهایی Replication

روی Master:

INSERT INTO test.repl_check VALUES (1);

روی Replica:

SELECT * FROM test.repl_check;
📊 وضعیت سالم Replication

روی Master:

SHOW PROCESSLIST;

باید:

User: repl
Command: Binlog Dump

روی Replica:

Slave_IO_State: Waiting for master to send event
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
🏁 نتیجه نهایی

Replica منطقه 10.75 با ویژگی‌های زیر عملیاتی شد:

Docker-based MySQL 5.6

Dedicated 10TB Storage

Port Forward via socat

Firewall restricted access

Parallel Restore

Live Replication active