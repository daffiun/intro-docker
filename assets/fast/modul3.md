# MODUL 3 — Jawaban Checklist Web Service Docker — Apache & Nginx

> Tujuan file ini: menjalankan dan membuktikan setiap item checklist Modul 3.  
> Project diasumsikan berada di `~/docker-lab/web-service`.

---

## 0. Masuk folder project dan siapkan host lokal

```bash
cd ~/docker-lab/web-service
```

Tambahkan domain lab ke `/etc/hosts` jika belum ada:

```bash
grep -q "site1.lab" /etc/hosts || \
  echo "127.0.0.1 site1.lab site2.lab app.lab" | sudo tee -a /etc/hosts
```

**Bukti berhasil:**

```bash
getent hosts site1.lab site2.lab app.lab
```

---

## 1. Self-signed SSL certificate di-generate — `.crt` dan `.key` ada di `./certs/`

Generate certificate jika belum ada:

```bash
mkdir -p certs

openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/C=ID/ST=Jawa Timur/L=Surabaya/O=PENS Lab/CN=*.lab" \
  -addext "subjectAltName=DNS:*.lab,DNS:site1.lab,DNS:site2.lab,DNS:app.lab"
```

Verifikasi:

```bash
ls -la certs/
openssl x509 -in certs/server.crt -noout -subject -issuer -dates
```

**Bukti berhasil:** folder `certs/` berisi:

- `server.crt`
- `server.key`

---

## 2. `docker compose up --build -d` — 4 service running

```bash
docker compose up --build -d
docker compose ps
```

**Bukti berhasil:** 4 service berjalan, biasanya:

- `nginx-proxy`
- `apache-web`
- `flask-app`
- `postgres-db`

Status harus `running` atau `Up`.

---

## 3. `curl -k https://site1.lab:8443` menampilkan Site 1 Company

```bash
curl -k https://site1.lab:8443
```

**Bukti berhasil:** output HTML memuat teks:

```text
Site 1
Company Profile
site1.lab
```

---

## 4. `curl -k https://site2.lab:8443` menampilkan Site 2 Blog

```bash
curl -k https://site2.lab:8443
```

**Bukti berhasil:** output HTML memuat teks:

```text
Site 2
Blog
site2.lab
```

---

## 5. `curl -k https://app.lab:8443/api/health` — database connected

```bash
curl -k https://app.lab:8443/api/health | python3 -m json.tool
```

**Bukti berhasil:** response JSON memuat:

```json
"db_status": "connected"
```

atau informasi PostgreSQL berhasil dibaca.

---

## 6. HTTP → HTTPS redirect berfungsi — return 301

```bash
curl -I http://site1.lab:8080
```

**Bukti berhasil:** response header memuat:

```text
HTTP/1.1 301 Moved Permanently
Location: https://site1.lab/
```

Jika domain lokal belum resolve, gunakan variasi ini:

```bash
curl -I -H "Host: site1.lab" http://localhost:8080
```

---

## 7. API CRUD berfungsi — POST dan GET `/api/visitors` berhasil

Tambah visitor:

```bash
curl -k -X POST https://app.lab:8443/api/visitors \
  -H "Content-Type: application/json" \
  -d '{"name": "Mahasiswa PENS"}' | python3 -m json.tool
```

Lihat daftar visitor:

```bash
curl -k https://app.lab:8443/api/visitors | python3 -m json.tool
```

**Bukti berhasil:**

- POST mengembalikan data visitor baru atau status sukses.
- GET menampilkan daftar visitor termasuk `Mahasiswa PENS`.

---

## 8. Log per-site terpisah — file log berbeda untuk site1, site2, app

Generate traffic dulu:

```bash
curl -k https://site1.lab:8443 > /dev/null
curl -k https://site2.lab:8443 > /dev/null
curl -k https://app.lab:8443/api/health > /dev/null
```

Cek log Nginx:

```bash
docker exec nginx-proxy sh -c "ls -l /var/log/nginx/*access.log"
docker exec nginx-proxy sh -c "tail -5 /var/log/nginx/site1-access.log"
docker exec nginx-proxy sh -c "tail -5 /var/log/nginx/site2-access.log"
docker exec nginx-proxy sh -c "tail -5 /var/log/nginx/app-access.log"
```

Cek log Apache:

```bash
docker exec apache-web sh -c "ls -l /var/log/apache2/*access.log"
docker exec apache-web sh -c "tail -5 /var/log/apache2/site1-access.log"
docker exec apache-web sh -c "tail -5 /var/log/apache2/site2-access.log"
```

**Bukti berhasil:** ada file log berbeda untuk site1, site2, dan app; setiap file berisi request sesuai host.

---

## 9. `X-Real-IP` terlihat di response Flask

```bash
curl -k https://app.lab:8443/ | python3 -m json.tool
```

**Bukti berhasil:** response JSON dari Flask memuat field seperti:

```json
"client_ip": "..."
```

atau field lain yang menunjukkan IP request yang diteruskan oleh Nginx melalui header `X-Real-IP`.

---

## Ringkasan bukti yang bisa dimasukkan ke laporan

```bash
cd ~/docker-lab/web-service
ls -la certs/
docker compose ps
curl -k https://site1.lab:8443
curl -k https://site2.lab:8443
curl -k https://app.lab:8443/api/health | python3 -m json.tool
curl -I http://site1.lab:8080
curl -k -X POST https://app.lab:8443/api/visitors -H "Content-Type: application/json" -d '{"name": "Mahasiswa PENS"}'
curl -k https://app.lab:8443/api/visitors | python3 -m json.tool
docker exec nginx-proxy sh -c "ls -l /var/log/nginx/*access.log"
docker exec apache-web sh -c "ls -l /var/log/apache2/*access.log"
curl -k https://app.lab:8443/ | python3 -m json.tool
```
