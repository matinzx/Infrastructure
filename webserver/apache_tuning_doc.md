# 📘 راهنمای کامل تنظیم و تیونینگ **Apache HTTP Server (httpd)**

این داکیومنت یک مجموعهٔ کامل از راهکارهای تیونینگ و بهینه‌سازی **Apache HTTPD** برای محیط‌های Production است. هدف، افزایش عملکرد، کاهش مصرف منابع، بهبود مقیاس‌پذیری و رسیدن به پایداری بالا زیر بار ترافیک سنگین است.

این راهنما مشابه داکیومنت تیونینگ Tomcat طراحی شده و شامل:
- تنظیمات بخش‌های اصلی Apache
- محاسبهٔ علمی پارامترها براساس CPU/RAM
- تنظیمات MPM
- بهینه‌سازی Kernel و Network
- روش‌های Benchmark و تست فشار

می‌باشد.

---

# 1) ⚙️ انتخاب و تنظیم MPM مناسب
Apache از چند مدل پردازش (MPM) پشتیبانی می‌کند:
- **prefork** (قدیمی، بدون thread)
- **worker** (thread + process)
- **event** (بهترین برای ترافیک بالا)

✔ برای Production مدرن همیشه MPM **event** توصیه می‌شود.

### 📌 فعال‌سازی MPM Event
فایل:
```
/etc/apache2/mods-enabled/mpm_event.conf
```
یا
```
/etc/httpd/conf.modules.d/00-mpm.conf
```

### 📌 کانفیگ نهایی (پیشنهادی برای سرور 16GB RAM / 16 Core)
```apache
<IfModule mpm_event_module>
    StartServers              4
    ServerLimit               32
    MinSpareThreads           75
    MaxSpareThreads          250
    ThreadsPerChild           25
    MaxRequestWorkers        800
    MaxConnectionsPerChild  5000
</IfModule>
```

---

# 2) 🧮 روش محاسبهٔ پارامترهای MPM بر اساس منابع

## 📌 محاسبه MaxRequestWorkers
فرمول استاندارد:
```
MaxRequestWorkers = CPU cores × 50
```
سرور شما:
```
16 × 50 = 800   ← مقدار صحیح
```

## 📌 محاسبه ThreadsPerChild
برای Event MPM:
```
ThreadsPerChild = 25  (مناسب برای ترافیک پایدار)
```

## 📌 محاسبه ServerLimit
```
ServerLimit = MaxRequestWorkers / ThreadsPerChild
800 / 25 = 32
```
این مقدار دقیقاً در کانفیگ بالا استفاده شده است.

---

# 3) ⚙️ بهینه‌سازی KeepAlive، Timeout و Buffer ها
این بخش تأثیر زیادی روی RPS (Requests Per Second) دارد.

### 📌 تنظیمات پیشنهادی
```
KeepAlive On
MaxKeepAliveRequests 200
KeepAliveTimeout 2
Timeout 30
ServerTokens Prod
ServerSignature Off
```

### 🔍 توضیح:
- **KeepAliveTimeout=2** → جلوگیری از اشغال زیاد کانکشن‌ها
- **ServerTokens/Signature Off** → افزایش امنیت

---

# 4) ⚙️ فعال‌سازی GZIP / Brotli برای کاهش حجم پاسخ
فشرده‌سازی تأثیر مستقیم روی سرعت دارد.

### 📌 فعال‌سازی Gzip
```
AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css application/json application/javascript text/javascript
```

### 📌 فعال‌سازی Brotli (در صورت نیاز)
```
LoadModule brotli_module modules/mod_brotli.so
AddOutputFilterByType BROTLI_COMPRESS text/html text/plain text/css application/json application/javascript
```

---

# 5) ⚙️ تنظیمات Cache برای سرعت بیشتر
```
<IfModule mod_expires.c>
  ExpiresActive On
  ExpiresDefault "access plus 1 month"
  ExpiresByType text/css "access plus 1 month"
  ExpiresByType application/javascript "access plus 1 month"
</IfModule>
```

---

# 6) ⚙️ بهینه‌سازی سطح سیستم‌عامل (Linux Tuning)

### 📌 sysctl.conf
```
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 4096
fs.file-max = 1000000
```

### 📌 limits.conf
```
apache soft nofile 200000
apache hard nofile 200000
```
(اگر یوزر شما httpd یا www-data است، همان را جایگزین کنید)

---

# 7) ⚙️ غیرفعال‌سازی Swap برای Performance و Stability
```
swapoff -a
systemctl mask dev-sdaX.swap
```

---

# 8) 🧪 تست فشار و بنچمارک
استفاده از ابزارهای:
- **ab**
- **hey**
- **h2load**

### 📌 نمونه تست
```
ab -n 20000 -c 500 http://yourserver/
```

### 📌 تحلیل نتایج
- RPS بالا → خوب
- Latency پایین → خوب
- Failed requests → باید صفر باشد

---