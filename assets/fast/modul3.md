# =========================================================
# CHECKLIST MODUL 3 — WEB SERVICE DOCKER: APACHE & NGINX
# =========================================================


# ---------------------------------------------------------
# 0. Masuk folder project Modul 3
# ---------------------------------------------------------

cd ~/docker-lab/web-service


# ---------------------------------------------------------
# 0.1 Tambahkan DNS lokal
# Wajib agar site1.lab, site2.lab, dan app.lab bisa dikenali
# ---------------------------------------------------------

echo "127.0.0.1 site1.lab site2.lab app.lab" | sudo tee -a /etc/hosts

# Cek hasilnya
cat /etc/hosts | grep lab


# ---------------------------------------------------------
# 1. Self-signed SSL certificate di-generate
# Checklist: file .crt dan .key ada di ./certs/
# ---------------------------------------------------------

mkdir -p certs

openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/C=ID/ST=Jawa Timur/L=Surabaya/O=PENS Lab/CN=*.lab" \
  -addext "subjectAltName=DNS:*.lab,DNS:site1.lab,DNS:site2.lab,DNS:app.lab"

# Cek file certificate
ls -la certs/

# Cek detail certificate
openssl x509 -in certs/server.crt -noout -subject -issuer -dates


# ---------------------------------------------------------
# 2. docker compose up --build -d — 4 service running
# ---------------------------------------------------------

docker compose up --build -d

# Cek 4 service
docker compose ps

# Expected service:
# nginx-proxy
# apache-web
# flask-app
# postgres-db
#
# Status minimal:
# Up
#
# Untuk database:
# healthy


# ---------------------------------------------------------
# 3. curl -k https://site1.lab:8443
# Checklist: menampilkan Site 1 Company
# ---------------------------------------------------------

curl -k https://site1.lab:8443

# Cek lebih singkat
curl -k https://site1.lab:8443 | grep -i "Site 1"

# Expected:
# Site 1 — Company Profile


# ---------------------------------------------------------
# 4. curl -k https://site2.lab:8443
# Checklist: menampilkan Site 2 Blog
# ---------------------------------------------------------

curl -k https://site2.lab:8443

# Cek lebih singkat
curl -k https://site2.lab:8443 | grep -i "Site 2"

# Expected:
# Site 2 — Blog


# ---------------------------------------------------------
# 5. curl -k https://app.lab:8443/api/health
# Checklist: database connected
# ---------------------------------------------------------

curl -k https://app.lab:8443/api/health

# Versi rapi
curl -k https://app.lab:8443/api/health | python3 -m json.tool

# Expected:
# "status": "ok"
# "db_status": "connected"


# ---------------------------------------------------------
# 6. HTTP→HTTPS redirect berfungsi
# Checklist: curl -I http://site1.lab:8080 return 301
# ---------------------------------------------------------

curl -I http://site1.lab:8080

# Cek hanya header awal
curl -I http://site1.lab:8080 | head

# Expected:
# HTTP/1.1 301 Moved Permanently
# Location: https://site1.lab...


# ---------------------------------------------------------
# 7. API CRUD berfungsi — POST dan GET /api/visitors berhasil
# ---------------------------------------------------------

# POST visitor
curl -k -X POST https://app.lab:8443/api/visitors \
  -H "Content-Type: application/json" \
  -d '{"name": "Daffi PENS"}'

# Cek POST dengan status HTTP
curl -k -i -X POST https://app.lab:8443/api/visitors \
  -H "Content-Type: application/json" \
  -d '{"name": "Daffi PENS"}'

# Expected:
# HTTP/1.1 201 CREATED

# GET visitors
curl -k https://app.lab:8443/api/visitors

# Versi rapi
curl -k https://app.lab:8443/api/visitors | python3 -m json.tool

# Expected:
# Data visitor yang tadi di-POST muncul


# ---------------------------------------------------------
# 8. Log per-site terpisah
# Checklist: file log berbeda untuk site1, site2, app
# ---------------------------------------------------------

# Generate traffic dulu agar log terisi
curl -k https://site1.lab:8443 > /dev/null
curl -k https://site2.lab:8443 > /dev/null
curl -k https://app.lab:8443/api/health > /dev/null

# Cek daftar log Nginx
docker exec nginx-proxy ls -la /var/log/nginx

# Expected file Nginx:
# site1-access.log
# site1-error.log
# site2-access.log
# site2-error.log
# app-access.log
# app-error.log

# Cek isi log Nginx site1
docker exec nginx-proxy cat /var/log/nginx/site1-access.log

# Cek isi log Nginx site2
docker exec nginx-proxy cat /var/log/nginx/site2-access.log

# Cek isi log Nginx app
docker exec nginx-proxy cat /var/log/nginx/app-access.log

# Cek daftar log Apache
docker exec apache-web ls -la /var/log/apache2

# Cek isi log Apache site1
docker exec apache-web cat /var/log/apache2/site1-access.log

# Cek isi log Apache site2
docker exec apache-web cat /var/log/apache2/site2-access.log


# ---------------------------------------------------------
# 9. X-Real-IP terlihat di response Flask
# Checklist: field client_ip muncul di response Flask
# ---------------------------------------------------------

curl -k https://app.lab:8443

# Versi rapi
curl -k https://app.lab:8443 | python3 -m json.tool

# Expected response mengandung:
# "client_ip": "..."
# "proto": "https"
#
# Ini membuktikan Nginx meneruskan header X-Real-IP
# dan X-Forwarded-Proto ke Flask.