# Bitwarden Self‑Hosted


# فهرست مطالب

* [معرفی](#معرفی)
* [معماری و مؤلفه‌ها](#معماری-و-مؤلفهها)
* [پیش‌نیازها](#پیشنیازها)
* [پورت‌ها و دسترسی شبکه](#پورتها-و-دسترسی-شبکه)
* [روش ۱: استقرار پایدار (Standard)](#روش-۱-استقرار-پایدار-standard)

  * [دریافت Installation ID/Key](#دریافت-installation-idkey)
  * [آماده‌سازی سیستم](#آمادهسازی-سیستم)
  * [پیکربندی SSL و دامنه](#پیکربندی-ssl-و-دامنه)
  * [پیکربندی SMTP و Admin](#پیکربندی-smtp-و-admin)
  * [Start/Stop/Update](#startstopupdate)
* [روش ۲: استقرار Unified (تک‌کانتینره – بتا)](#روش-۲-استقرار-unified-تککانتینره--بتا)

  * [env نمونه](#env-نمونه)
  * [Docker Compose نمونه](#docker-compose-نمونه)
* [Reverse Proxy (اختیاری)](#reverse-proxy-اختیاری)

  * [NGINX](#nginx)
  * [HAProxy](#haproxy)
* [پشتیبان‌گیری و بازیابی](#پشتیبانگیری-و-بازیابی)
* [عیب‌یابی](#عیبیابی)
* [امنیت و سخت‌سازی](#امنیت-و-سختسازی)
* [منابع رسمی](#منابع-رسمی)
* [مجوز](#مجوز)

---

## معرفی

**Bitwarden** یک مدیر رمز عبور متن‌باز است که امکان **استقرار سلف-هاست** روی سرور لینوکسی را فراهم می‌کند. این مخزن یک راهنمای مرحله‌به‌مرحله برای دو دیپلوی رسمی ارائه می‌کند:

* **Standard (پایدار/Enterprise‑grade)**: چند کانتینر Docker + پایگاه‌داده SQL Server.
* **Unified (تک‌کانتینره/بتا)**: یک ایمیج واحد با پشتیبانی از PostgreSQL/MySQL/SQLite/MSSQL (برای محیط‌های شخصی/آزمایشی).

> **توصیه**: برای محیط سازمانی/پروداکشن از **Standard** استفاده کنید؛ Unified تا زمان خروج از بتا برای Enterprise توصیه نمی‌شود.

---

## معماری و مؤلفه‌ها

* **Web Vault**, **API**, **Identity**, **Notifications**, **Admin Portal**, **Database**
* ذخایر پایدار روی دیسک در مسیر `./bwdata` (Standard) یا Volume کانتینر (Unified)
* ایمیل (SMTP) برای فعال‌سازی حساب‌ها و ورود به پنل ادمین ضروری است.

---

## پیش‌نیازها

* سیستم‌عامل: **Ubuntu/Debian/RHEL** (نمونه‌ها با Ubuntu نوشته شده‌اند)
* دامنه: `<bitwarden.<domain>>` که به IP سرور اشاره کند
* **Docker Engine** و **Docker Compose Plugin** نصب شده
* منابع پیشنهادی Standard: حداقل 2GB RAM / 12GB Disk (پیشنهادی 4GB/25GB)
* باز بودن پورت‌های **80** و **443** در فایروال/Load Balancer

> برای نصب Docker از راهنمای رسمی Docker استفاده کنید.

---

## پورت‌ها و دسترسی شبکه

| مؤلفه | پورت پیش‌فرض | توضیح                                                      |
| ----- | ------------ | ---------------------------------------------------------- |
| HTTP  | 80/tcp       | برای صدور/تجدید SSL با Let’s Encrypt (اگر استفاده می‌کنید) |
| HTTPS | 443/tcp      | دسترسی نهایی کاربران                                       |
| Admin | `/admin`     | روی همان دامنه؛ لینک ورود ایمیلی نیاز دارد                 |

**UFW (غیرمخرب):**

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

---

## روش ۱: استقرار پایدار (Standard)

### دریافت Installation ID/Key

از صفحهٔ زیر **Installation ID** و **Installation Key** را دریافت کنید و برای نصب آماده داشته باشید:

* [https://bitwarden.com/host](https://bitwarden.com/host)

### آماده‌سازی سیستم

کاربر اختصاصی و مسیر نصب بسازید و سپس اسکریپت رسمی را دانلود کنید:

```bash
sudo adduser bitwarden
sudo passwd bitwarden
sudo groupadd docker || true
sudo usermod -aG docker bitwarden
sudo mkdir -p /opt/bitwarden
sudo chmod -R 700 /opt/bitwarden
sudo chown -R bitwarden:bitwarden /opt/bitwarden

sudo -iu bitwarden
cd /opt/bitwarden

# دریافت اسکریپت رسمی نصب
curl -Lso bitwarden.sh "https://func.bitwarden.com/api/dl/?app=self-host&platform=linux" && chmod 700 bitwarden.sh

# شروع نصب تعاملی
./bitwarden.sh install
```

در حین نصب:

* `Domain name`: مقدار `<bitwarden.<domain>>`
* **SSL**: اگر اینترنت دارید، `Let's Encrypt = y` و ایمیل ادمین را وارد کنید
* `Installation ID/Key`: مقادیر دریافتی از `bitwarden.com/host`

پس از نصب:

```bash
# راه‌اندازی سرویس‌ها
./bitwarden.sh start

# وضعیت کانتینرها
docker ps
```

### پیکربندی SSL و دامنه

* اگر Let’s Encrypt را فعال کنید، صدور/تجديد گواهی به‌صورت خودکار انجام می‌شود.
* در صورت استفاده از Reverse Proxy، پورت‌های داخلی و گواهی را می‌توانید در لایه‌ی LB مدیریت کنید. تغییر پورت‌ها از طریق `./bwdata/config.yml` و سپس `rebuild` است.

### پیکربندی SMTP و Admin

برای ایمیل و پنل ادمین نیاز به SMTP دارید:

```bash
vim ./bwdata/env/global.override.env
```

مقادیر زیر را اضافه/ویرایش کنید:

```
globalSettings__mail__smtp__host=<smtp.server.example>
globalSettings__mail__smtp__port=<587 یا 465>
globalSettings__mail__smtp__ssl=<true|false>
globalSettings__mail__smtp__username=<smtp-user>
globalSettings__mail__smtp__password=<smtp-pass>
adminSettings__admins=<admin1@example.com,admin2@example.com>
```

تست SMTP:

```bash
./bitwarden.sh checksmtp
```

> توجه: هر ایمیل در `adminSettings__admins` می‌تواند با رفتن به `https://<bitwarden.<domain>>/admin` لینک ورود یک‌بارمصرف دریافت کند.

### Start/Stop/Update

```bash
./bitwarden.sh start
./bitwarden.sh restart
./bitwarden.sh stop
./bitwarden.sh update       # به‌روزرسانی سرویس‌ها
./bitwarden.sh updateself   # به‌روزرسانی اسکریپت نصب
./bitwarden.sh compresslogs # فشرده‌سازی لاگ‌ها
```

---

## روش ۲: استقرار Unified (تک‌کانتینره – بتا)

> مناسب تست/مصرف شخصی؛ برای Enterprise تا پایان بتا توصیه نمی‌شود.

ساخت دایرکتوری و فایل‌های موردنیاز:

```bash
sudo -iu <user>
mkdir -p ~/bitwarden-unified && cd ~/bitwarden-unified
```

### env نمونه

```bash
vim settings.env
```

نمونهٔ محتوا:

```
BW_DOMAIN=<bitwarden.example.com>
BW_DB_PROVIDER=<sqlserver|postgresql|sqlite|mysql|mariadb>
BW_DB_SERVER=<db.host:port>           # برای sqlite لازم نیست
BW_DB_DATABASE=<bwdb>
BW_DB_USERNAME=<bwuser>
BW_DB_PASSWORD=<supersecret>
BW_INSTALLATION_ID=<your_id_from_host_page>
BW_INSTALLATION_KEY=<your_key_from_host_page>
# فقط برای sqlite (اختیاری):
# BW_DB_FILE=/etc/bitwarden/bw.sqlite
```

### Docker Compose نمونه

```bash
vim compose.yml
```

محتوا:

```yaml
services:
  bitwarden:
    image: ghcr.io/bitwarden/self-host:beta
    env_file:
      - ./settings.env
    volumes:
      - ./bwdata:/etc/bitwarden
    ports:
      - "80:8080"   # HTTP
      # اگر SSL را در خود کانتینر فعال می‌کنید:
      # - "443:8443"
    restart: always
```

راه‌اندازی:

```bash
docker compose up -d
docker ps
```

دسترسی وب: `https://<bitwarden.<domain>>`

---

## Reverse Proxy (اختیاری)

### NGINX

```bash
sudo vim /etc/nginx/sites-available/bitwarden.conf
```

محتوا (نمونه TLS termination روی LB):

```nginx
server {
    listen 80;
    server_name <bitwarden.<domain>>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <bitwarden.<domain>>;

    ssl_certificate     </etc/ssl/<fullchain.pem>>;
    ssl_certificate_key </etc/ssl/<privkey.pem>>;

    location / {
        proxy_pass http://127.0.0.1:8080; # در Standard/Unified با توجه به مپ پورت تنظیم کنید
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

*فعال‌سازی:*

```bash
sudo ln -s /etc/nginx/sites-available/bitwarden.conf /etc/nginx/sites-enabled/bitwarden.conf
sudo nginx -t && sudo systemctl reload nginx
```

### HAProxy

```bash
sudo vim /etc/haproxy/haproxy.cfg
```

پیاده‌سازی ساده‌ی TLS termination و فوروارد به سرویس داخلی:

```
frontend fe_bitwarden
    bind *:443 ssl crt </etc/ssl/<bundle.pem>>
    mode http
    option forwardfor
    default_backend be_bitwarden

backend be_bitwarden
    mode http
    server bw1 127.0.0.1:8080 check
```

*ریلود:*

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg && sudo systemctl reload haproxy
```

---

## پشتیبان‌گیری و بازیابی

> همیشه قبل از Update بک‌آپ بگیرید.

### Standard

* مسیرهای مهم: `/opt/bitwarden/bwdata` و گواهی‌ها (اگر محلی)

```bash
sudo tar -czf /var/backups/bitwarden_bwdata_$(date +%F).tar.gz -C /opt/bitwarden bwdata
```

*بازیابی:*

```bash
sudo systemctl stop docker || true
sudo tar -xzf </path/to/backup.tar.gz> -C /opt/bitwarden
# سپس
sudo systemctl start docker
sudo -iu bitwarden
cd /opt/bitwarden
./bitwarden.sh rebuild && ./bitwarden.sh start
```

### Unified

* مسیر Volume پروژه (مثلاً `~/bitwarden-unified/bwdata`) + دیتابیس خارجی (Postgres/MySQL/…)

```bash
tar -czf ~/bw_unified_backup_$(date +%F).tar.gz -C ~/bitwarden-unified bwdata
```

---

## عیب‌یابی

* وضعیت سرویس‌ها: `docker ps`, `docker logs <container>`
* لاگ NGINX/HAProxy را بررسی کنید.
* تست SMTP: در Standard با `./bitwarden.sh checksmtp`
* اگر پورت‌ها مشغول‌اند: `ss -tulpn | grep -E ":(80|443)"`
* خطاهای گواهی: زمان سیستم/فایروال/دسترسی پورت 80 برای Let’s Encrypt را چک کنید.

---

## امنیت و سخت‌سازی

* اطمینان از فعال بودن **HTTPS** و هدِرهای امنیتی روی LB/وب‌سرور
* محدودسازی دسترسی Admin (`/admin`) از طریق IP allowlist روی LB (در صورت امکان)
* فعال‌سازی **2FA** برای ادمین‌ها و کاربران
* به‌روزرسانی منظم ایمیج‌ها/اسکریپت (`./bitwarden.sh update` / Pull جدید Unified)
* پایش سلامت گواهی‌ها و اعتبار DNS

---

## منابع رسمی

* مستندات Self‑Hosting (Standard): [https://bitwarden.com/help/install-on-premises-linux/](https://bitwarden.com/help/install-on-premises-linux/)
* Self‑Hosted Installation ID/Key: [https://bitwarden.com/host](https://bitwarden.com/host)
* Self‑Hosted Admin: [https://bitwarden.com/help/self-host-admin/](https://bitwarden.com/help/self-host-admin/)
* Unified (Beta) README: [https://github.com/bitwarden/server/tree/main/self-host](https://github.com/bitwarden/server/tree/main/self-host)
* Docker Engine Install: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

> همیشه به آخرین مستندات رسمی مراجعه کنید؛ ممکن است نسخه‌ها/پارامترها تغییر کنند.

---

## مجوز

این راهنما تحت مجوز **MIT** در مخزن شما قابل استفاده است. در صورت استفاده، نام و لینک مخزن خود را اضافه کنید.
