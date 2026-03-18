akhirnya nyentuh dunia server juga ya… selamat, sekarang elo resmi punya sesuatu yang bisa salah konfigurasi 1000 cara berbeda 👍

tenang, gw bikinin step yang waras dan ga ribet. asumsi: VPS fresh, OS Ubuntu 22.04/24.04.

---

# 1️⃣ Update dulu (biar ga hidup di masa lalu)

```bash
sudo apt update && sudo apt upgrade -y
```

---

# 2️⃣ Setup user (jangan pake root kayak barbar)

```bash
adduser vicky
usermod -aG sudo vicky
```

login ulang pake user itu:

```bash
su - vicky
```

---

# 3️⃣ SSH basic security (opsional tapi penting banget)

Edit:

```bash
sudo nano /etc/ssh/sshd_config
```

ubah:

```
PermitRootLogin no
PasswordAuthentication no
```

restart:

```bash
sudo systemctl restart ssh
```

⚠️ jangan matiin password auth sebelum lo setup SSH key, kecuali lo pengen terkunci dari server sendiri (plot twist paling menyedihkan).

---

# 4️⃣ Firewall (UFW)

ini bagian yang lo mau

```bash
sudo apt install ufw -y
```

default rules:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

allow yang penting:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
```

aktifin:

```bash
sudo ufw enable
```

cek:

```bash
sudo ufw status
```

---

# 5️⃣ Install Nginx

```bash
sudo apt install nginx -y
```

cek:

```bash
sudo systemctl status nginx
```

browser:

```
http://IP_VPS_LO
```

kalau muncul halaman nginx = hidup, congrats, ga sia-sia bayar VPS.

---

# 6️⃣ Install MySQL

```bash
sudo apt install mysql-server -y
```

secure install:

```bash
sudo mysql_secure_installation
```

jawaban aman:

* VALIDATE PASSWORD → yes
* level → medium
* remove anonymous → yes
* disallow root login remote → yes
* remove test db → yes

---

# 7️⃣ Setup MySQL user (jangan pake root buat app)

```bash
sudo mysql
```

```sql
CREATE DATABASE app_db;
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'password_lo_yang_ga_lemah';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

# 8️⃣ (Optional tapi disarankan) Allow MySQL remote? JANGAN DULU

default MySQL cuma listen di:

```
127.0.0.1
```

ini BAGUS. jangan dibuka ke publik kecuali lo ngerti konsekuensinya.

---

# 9️⃣ Struktur basic Nginx (buat app nanti)

```bash
sudo mkdir -p /var/www/app
sudo chown -R $USER:$USER /var/www/app
```

config:

```bash
sudo nano /etc/nginx/sites-available/app
```

isi:

```nginx
server {
    listen 80;
    server_name domain_lo.com;

    root /var/www/app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

aktifin:

```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

# 10️⃣ Checklist akhir (biar ga sok yakin tapi zonk)

* [x] Firewall aktif
* [x] SSH aman
* [x] Nginx jalan
* [x] MySQL jalan
* [x] Ga pake root buat semuanya (ini penting, serius)

---

# Bonus (biar ga jadi korban berikutnya)

kalau mau next level:

* fail2ban (anti brute force)
* certbot (SSL gratis)
* monitoring (htop, netdata)
* backup (ini yang sering orang skip lalu nangis)

---

intinya sekarang VPS lo udah:

* ga terlalu telanjang di internet
* siap jadi backend API / web
* ga langsung jadi ladang crypto miner orang lain

lumayan lah, buat langkah pertama elo ga bikin dosa besar 😌
