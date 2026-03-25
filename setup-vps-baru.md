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

---
## SETUP DOMAIN

Ya terus kalau semua itu udah bener tapi domain masih bengong gak mau kebuka, berarti kita masuk fase klasik: **“gue yakin semua udah bener, tapi ternyata ada yang kelewat”**.

Cek satu-satu, jangan sok yakin dulu:

### 1. Test langsung IP

Coba buka:

```
http://43.159.34.85
```

* Kalau ini **gak kebuka** → masalah di server (nginx lo bohong soal “udah nyala”)
* Kalau **kebuka** → lanjut

### 2. DNS resolve

Di lokal:

```
ping vqf.my.id
```

atau:

```
dig vqf.my.id +short
```

Harus keluar:

```
43.159.34.85
```

* Kalau belum → DNS belum propagate / salah record
* Kalau udah → lanjut

### 3. Nginx config (ini yang sering bikin malu)

Lo udah set `server_name` belum?

Contoh minimal:

```nginx
server {
    listen 80;
    server_name vqf.my.id www.vqf.my.id;

    root /var/www/html;
    index index.html;
}
```

Terus:

```
sudo nginx -t
sudo systemctl reload nginx
```

Kalau `server_name` gak cocok:

> nginx bakal fallback ke default config → domain lo di-ignore kayak mantan.

### 4. Firewall layer lain

Lo bilang port kebuka, tapi:

* Cloud provider (Tencent? AWS? dll) ada **security group**
* Cek juga:

```
sudo ufw status
```

### 5. Coba curl dari luar

```
curl -I http://vqf.my.id
```

Kalau timeout:

> berarti masih kehalang di network, bukan nginx

---

Kesimpulan cepat:

* IP bisa diakses → DNS atau nginx config
* IP gak bisa diakses → server / firewall

Kalau semua ini udah lo cek dan masih gagal, berarti semesta lagi ngajarin lo satu hal:
**“DevOps itu bukan soal bener, tapi soal sabar ngelacak kebodohan kecil.”**


Config lo… surprisingly… gak jelek. Langka sih.

Tapi ada beberapa titik rawan yang biasanya bikin orang stuck sambil nyalahin DNS:

---

### 🔍 1. Pastikan config ini KEPAKE

Kadang orang bikin config cakep tapi… gak di-load. Classic.

Cek:

```bash
ls /etc/nginx/sites-enabled/
```

Harus ada file yang isinya config lo (biasanya symlink dari `sites-available`).

Kalau belum:

```bash
sudo ln -s /etc/nginx/sites-available/namafile /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Kalau gak, nginx lo literally ignore config ini.

---

### 🔍 2. Default server nyolong request lo

Kalau ada config default kayak:

```nginx
server {
    listen 80 default_server;
    ...
}
```

Dia bisa “nangkep” semua request duluan.

Solusi:

* Hapus / disable default config:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

---

### 🔍 3. Folder `/var/www/app` beneran ada?

Ini sering banget:

```bash
ls /var/www/app
```

Harus ada `index.html`.

Kalau gak:

> ya jelas 404, bukan karena nginx jahat, tapi karena lo nyuruh dia cari sesuatu yang gak ada.

---

### 🔍 4. Permission (silent killer)

Kalau permission salah, nginx gak bisa baca file:

```bash
sudo chown -R www-data:www-data /var/www/app
sudo chmod -R 755 /var/www/app
```

---

### 🔍 5. Test langsung dari server (biar gak drama DNS)

Di VPS lo:

```bash
curl -H "Host: vqf.my.id" http://127.0.0.1
```

* Kalau ini **berhasil** → nginx config OK → masalah DNS / network
* Kalau **gagal** → config lo masih salah

---

### 🔍 6. DNS double check (iya, lagi)

Kadang orang typo 1 digit terus denial:

```bash
dig vqf.my.id +short
```

Harus:

```
43.159.34.85
```

---

### Kesimpulan jujur:

Setup lo udah 80% bener.
Sisanya biasanya salah di:

* config gak di-enable
* default server ganggu
* atau folder kosong (ini paling nyakitin)

Kalau masih gak jalan setelah ini, berarti masalahnya bukan teknis lagi…
tapi ego manusia yang terlalu cepat bilang “udah bener kok”.


Nah ini nih… klasik banget. Bukan error “misterius”, ini error **lo kelupaan satu hal obvious**.

### 💥 Intinya:

```
NXDOMAIN looking up A for www.vqf.my.id
```

Artinya:

> **`www.vqf.my.id` itu gak ada di DNS. Kosong. Nol. Nihil.**

Lo cuma bikin:

```
vqf.my.id → 43.159.34.85
```

Tapi:

```
www.vqf.my.id → ??? (tidak diketahui alam semesta)
```

---

## 🔧 Solusi (simple banget, serius)

Balik ke DNS, tambah salah satu:

### Opsi 1 (paling umum)

```
Type: A
Name: www
Value: 43.159.34.85
```

---

### Opsi 2 (lebih “rapih”)

```
Type: CNAME
Name: www
Value: vqf.my.id
```

---

## 🤦 Kenapa ini kejadian?

Karena lo tadi di nginx tulis:

```nginx
server_name vqf.my.id www.vqf.my.id;
```

Terus pas certbot:

> dia coba verify **DUA domain itu**

Satu ada → lolos
Satu gak ada → gagal total

Certbot itu gak kompromi. Satu gagal, semua ikut tenggelam.

---

## 🚀 Setelah nambah DNS:

Tunggu bentar (biasanya cepet, gak perlu drama 48 jam), lalu:

```bash
sudo certbot --nginx
```

---

## 🔥 Shortcut kalau lo males ngurus www

Kalau lo tipe orang yang hidupnya minimalis:

```bash
sudo certbot --nginx -d vqf.my.id
```

Jadi cuma 1 domain.

---

## Kesimpulan pahit:

Lo gak salah konfigurasi server.
Lo cuma lupa bahwa:

> subdomain itu bukan “otomatis ada karena feeling”.

DNS itu literal banget. Gak ada → ya gak ada.

Sekarang tinggal tambahin `www`, beres. Internet berhenti nge-judge lo.

---

## 🧱 1. Install certbot + nginx plugin

Di Ubuntu:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

---

## 🔍 2. Pastikan domain udah resolve

Ini WAJIB. Jangan sok lanjut kalau ini belum:

```bash
dig vqf.my.id +short
```

Harus keluar:

```
43.159.34.85
```

Kalau belum → balik ke DNS, jangan maksa. Certbot bukan dukun.

---

## ⚙️ 3. Jalankan certbot

Simple mode (yang paling waras):

```bash
sudo certbot --nginx
```

Nanti dia bakal:

* detect config nginx lo
* nanya domain → pilih `vqf.my.id` + `www.vqf.my.id`
* auto generate SSL
* auto edit config nginx

Lo tinggal pilih:

* redirect HTTP → HTTPS → **pilih YES** (biar gak setengah-setengah hidup lo)

---

## 🧪 4. Test hasilnya

Buka:

```
https://vqf.my.id
```

Kalau berhasil:

> selamat, sekarang traffic lo terenkripsi, walaupun isinya cuma index.html kosong.

---

## 🔄 5. Auto renew (biar gak panik 3 bulan lagi)

Certbot biasanya udah setup cron, tapi ya kita gak percaya sistem tanpa bukti:

```bash
sudo certbot renew --dry-run
```

Kalau sukses → aman.

---

## ⚠️ Problem yang sering muncul

Biar lo gak pura-pura bingung nanti:

### ❌ Port 80 gak bisa diakses

Certbot butuh HTTP challenge.
Kalau fail:

> berarti firewall / security group masih bohong

---

### ❌ Nginx config gak ke-detect

Biasanya karena:

* `server_name` gak sesuai
* config belum di-enable

---

### ❌ Domain belum propagate

Error klasik:

> "Could not resolve host"

Ya… karena DNS lo belum nyampe.

---

## 🧠 Insight biar lo naik level dikit

Certbot itu basically:

1. prove ke CA kalau lo punya domain
2. dapet cert
3. pasang ke nginx

Bukan magic. Cuma automation yang nyelamatin lo dari ribet.

---

Nah, akhirnya jelas.
Bukan “fail2ban error”, tapi emang:

> **file filternya gak ada, jadi jail lo gak mungkin exist.**

Lo lagi nyuruh satpam nangkep maling… tapi lo gak kasih dia definisi maling itu apa.

---

## 🔧 Fix langsung (gak pake drama)

### 1. Bikin filter `nginx-badbots`

```bash
sudo nano /etc/fail2ban/filter.d/nginx-badbots.conf
```

Isi ini:

```ini
[Definition]
failregex = <HOST> -.*"(GET|POST).*HTTP.*" 403
ignoreregex =
```

Ini artinya:

> kalau ada IP kena 403 berulang → dianggap bot aneh → ban

Simple, tapi cukup buat mulai.

---

## ⚙️ 2. Daftarin jail-nya

Edit:

```bash
sudo nano /etc/fail2ban/jail.local
```

Tambahin:

```ini
[nginx-badbots]
enabled = true
port = http,https
filter = nginx-badbots
logpath = /var/log/nginx/access.log
maxretry = 5
bantime = 1h
```

---

## 🔄 3. Restart (ini wajib, bukan opsional)

```bash
sudo systemctl restart fail2ban
```

---

## 🧪 4. Cek lagi

```bash
sudo fail2ban-client status
```

Harusnya sekarang muncul:

```
Jail list: nginx-badbots
```

---

## 🧠 Real talk dikit

Filter yang barusan lo bikin itu:

* basic banget
* belum “pinter”
* tapi cukup buat ngeban scanner receh

Kalau lo liat log nanti banyak request kayak:

* `/wp-login.php`
* `/phpmyadmin`
* `/vendor/phpunit`

Itu semua bot internet yang lagi “ngetok pintu random”.

Dan sekarang:

> lo gak cuma buka pintu… tapi mulai milih siapa yang boleh masuk

---

## ⚠️ Bonus biar gak jadi jebakan batman

Pastikan file ini ada:

```bash
ls /var/log/nginx/access.log
```

Kalau gak ada / kosong:

> fail2ban lo bakal kerja keras… untuk tidak melakukan apa-apa

---

## Kesimpulan

Masalah lo kemarin:

* bukan config salah
* bukan command salah

Tapi:

> lo pakai fitur yang belum pernah lo install (atau bikin)

Sekarang udah ada → jail bisa hidup → server lo mulai punya “refleks”.

Pelan-pelan, dari server polos ke server yang agak susah diganggu.


---

Akhirnya ada yang mikirin swap juga. Jarang-jarang orang inget sebelum server nge-lag terus panik.

Gue tambahin ke summary lo, biar lengkap dan gak keliatan kayak setup setengah jadi:

---

# 🧱 INFRASTRUCTURE SUMMARY (VPS SETUP)

## 🌐 Domain & Web

* Domain: `vqf.my.id`
* DNS:

  * `A @ → 43.159.34.85`
  * `A www → 43.159.34.85`
* Web server: **Nginx**
* Root: `/var/www/app`
* HTTPS: pakai Certbot (Let’s Encrypt)
* Auto redirect HTTP → HTTPS ✔️

---

## 🔐 Security Layer

### 🔥 Firewall (UFW)

* Allow:

  * `80` (HTTP)
  * `443` (HTTPS)
  * `51820/udp` (WireGuard)
* SSH:

  * hanya dari VPN (`10.0.0.0/24`)
  * public access → blocked

---

### 🚫 Fail2ban

* Proteksi basic nginx
* Custom filter `nginx-badbots`
* Ban IP abusive
* Log:

  ```
  /var/log/nginx/access.log
  ```

---

## 🔑 VPN (WireGuard)

### Server

* Interface: `wg0`
* Network: `10.0.0.0/24`
* IP: `10.0.0.1`
* Port: `51820`
* NAT via `eth0`

### Client (Android)

* App: WireGuard
* IP: `10.0.0.2`
* Endpoint:

  ```
  vqf.my.id:51820
  ```
* AllowedIPs:

  ```
  0.0.0.0/0
  ```

---

## 🔐 Access Control

* SSH hanya via VPN ✔️
* Password login disabled ✔️
* Root login disabled ✔️
* SFTP:

  * optional
  * hanya via VPN

---

## 🚀 Deployment

* Utama: Git (`git pull`)
* SSH key untuk repo access
* SFTP hanya fallback

---

## 📊 Monitoring & Logs

* Nginx:

  ```
  /var/log/nginx/access.log
  ```
* Fail2ban:

  ```
  /var/log/fail2ban.log
  ```
* WireGuard:

  ```
  sudo wg
  ```

---

## ⚙️ System Settings

### IP Forwarding

```
net.ipv4.ip_forward=1
```

### Logrotate

* aktif untuk nginx ✔️

---

## 💾 Swap Configuration

### Yang lo lakukan:

```bash
# Matikan swap lama
sudo swapoff /swap.img

# Hapus swap lama
sudo rm /swap.img

# Buat swap baru 4GB
sudo fallocate -l 4G /swap.img

# Permission aman
sudo chmod 600 /swap.img

# Format & aktifkan
sudo mkswap /swap.img
sudo swapon /swap.img
```

### ⚠️ Tambahan penting (ini yang sering kelupaan)

Biar permanen setelah reboot:

```bash
sudo nano /etc/fstab
```

Tambah:

```
/swap.img none swap sw 0 0
```

Kalau gak:

> tiap reboot swap lo hilang, terus lo bingung kenapa RAM tiba-tiba sekarat

---

## 🧠 Architecture

```
Internet
   ↓
[Firewall]
   ↓
[Nginx + HTTPS]
   ↓
[App]

Private:
Client → WireGuard → VPS → Internal Access
```

---

## 💀 FINAL STATE

Server lo sekarang:

* Web live ✔️
* HTTPS ✔️
* VPN ✔️
* Secure SSH ✔️
* Git deploy ✔️
* Swap memory ✔️

Status:

> “udah layak dipakai, gak cuma buat pamer setup di Discord”

---

Ini udah solid banget buat personal / small production.
Kalau nanti lo ngerasa pengen “naik level” lagi… biasanya penyakitnya mulai:

> monitoring, automation, sama paranoia security 😌



