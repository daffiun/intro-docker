# MODUL 2 — Jawaban Checklist Docker Service, Volume & Mount Point

> Tujuan file ini: menjalankan dan membuktikan setiap item checklist Modul 2.  
> Jalankan di Ubuntu/WSL yang sudah memiliki Docker dan Docker Compose plugin.

---

## 0. Persiapan folder kerja

```bash
mkdir -p ~/docker-lab
cd ~/docker-lab
```

---

## 1. User-defined bridge `lab-net` — dua container bisa ping by nama

Bersihkan resource lama:

```bash
docker rm -f server-a server-b 2>/dev/null || true
docker network rm lab-net 2>/dev/null || true
```

Buat network dan jalankan dua container:

```bash
docker network create --driver bridge --subnet 172.20.0.0/16 lab-net

docker run -d --name server-a --network lab-net nginx:alpine
docker run -d --name server-b --network lab-net nginx:alpine
```

Tes DNS antar container:

```bash
docker exec server-a ping -c 3 server-b
docker exec server-b ping -c 3 server-a
```

**Bukti berhasil:** ping menghasilkan `0% packet loss`.

---

## 2. Default bridge — container tidak bisa resolve nama

Bersihkan container lama:

```bash
docker rm -f server-c server-d 2>/dev/null || true
```

Jalankan dua container di default bridge:

```bash
docker run -d --name server-c nginx:alpine
docker run -d --name server-d nginx:alpine
```

Tes ping nama container:

```bash
docker exec server-c ping -c 3 server-d
```

**Bukti berhasil:** perintah gagal dengan pesan seperti:

```text
bad address 'server-d'
```

atau DNS resolution failed. Itu menunjukkan default bridge tidak otomatis resolve nama container.

Bersihkan:

```bash
docker rm -f server-c server-d
```

---

## 3. Named volume `data-vol` — data persist setelah container dihapus

Buat volume dan tulis data:

```bash
docker volume rm data-vol 2>/dev/null || true
docker volume create data-vol

docker run --name writer -v data-vol:/app/data alpine:3.20 \
  sh -c "date > /app/data/log.txt && cat /app/data/log.txt"
```

Hapus container penulis:

```bash
docker rm writer
```

Baca data dari container lain:

```bash
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
```

**Bukti berhasil:** timestamp masih terbaca walaupun container `writer` sudah dihapus.

---

## 4. Volume backup/restore berhasil

Backup volume:

```bash
cd ~/docker-lab

docker run --rm \
  -v data-vol:/source:ro \
  -v "$(pwd)":/backup \
  alpine:3.20 tar czf /backup/data-vol-backup.tar.gz -C /source .
```

Restore ke volume baru:

```bash
docker volume rm data-vol-restored 2>/dev/null || true
docker volume create data-vol-restored

docker run --rm \
  -v data-vol-restored:/target \
  -v "$(pwd)":/backup:ro \
  alpine:3.20 tar xzf /backup/data-vol-backup.tar.gz -C /target
```

Verifikasi:

```bash
ls -lh ~/docker-lab/data-vol-backup.tar.gz
docker run --rm -v data-vol-restored:/data alpine:3.20 cat /data/log.txt
```

**Bukti berhasil:** file backup `.tar.gz` ada dan data dari volume restore sama-sama terbaca.

---

## 5. Bind mount — edit file host langsung terlihat di container

Buat folder dan file HTML:

```bash
mkdir -p ~/docker-lab/web-dev/html
cd ~/docker-lab/web-dev

cat > html/index.html << 'EOF'
<html>
<body>
  <h1>Hello dari Bind Mount!</h1>
  <p>Timestamp: VERSI-1</p>
</body>
</html>
EOF
```

Jalankan Nginx dengan bind mount:

```bash
docker rm -f dev-server 2>/dev/null || true

docker run -d --name dev-server \
  -p 8080:80 \
  -v "$(pwd)/html:/usr/share/nginx/html:ro" \
  nginx:alpine
```

Cek versi awal:

```bash
curl http://localhost:8080
```

Edit file di host:

```bash
sed -i 's/VERSI-1/VERSI-2 (diedit live)/' html/index.html
curl http://localhost:8080
```

**Bukti berhasil:** output kedua langsung berubah menjadi `VERSI-2 (diedit live)` tanpa restart container.

Bersihkan:

```bash
docker rm -f dev-server
```

---

## 6. tmpfs — data hilang setelah container restart

Jalankan container dengan tmpfs:

```bash
docker rm -f tmpfs-demo 2>/dev/null || true

docker run -d --name tmpfs-demo \
  --tmpfs /app/cache:size=64m \
  alpine:3.20 sleep 3600
```

Tulis file ke tmpfs:

```bash
docker exec tmpfs-demo sh -c "echo 'secret-data' > /app/cache/token.txt"
docker exec tmpfs-demo cat /app/cache/token.txt
```

Restart container:

```bash
docker stop tmpfs-demo
docker start tmpfs-demo
docker exec tmpfs-demo sh -c "ls /app/cache/token.txt || echo 'file hilang setelah restart'"
```

**Bukti berhasil:** setelah restart muncul pesan bahwa file hilang.

Bersihkan:

```bash
docker rm -f tmpfs-demo
```

---

## 7. `docker compose up --build -d` — 3 service running

Masuk ke project Compose:

```bash
cd ~/docker-lab/compose-app
```

Jalankan stack:

```bash
docker compose up --build -d
docker compose ps
```

**Bukti berhasil:** 3 service/container berjalan:

- `lab-web`
- `lab-app`
- `lab-db`

Status harus `running` atau `Up`, dan database harus `healthy`.

---

## 8. `http://localhost:8080` menampilkan halaman web

```bash
curl http://localhost:8080
```

**Bukti berhasil:** output HTML memuat teks halaman web, misalnya:

```text
Docker Compose Lab
Nginx → Flask → PostgreSQL
```

Bukti browser: buka `http://localhost:8080`.

---

## 9. `http://localhost:8080/api/health` menampilkan database connected

```bash
curl http://localhost:8080/api/health | python3 -m json.tool
```

**Bukti berhasil:** response JSON memuat:

```json
"db_status": "connected"
```

atau informasi database PostgreSQL berhasil dibaca.

---

## 10. `docker compose down` berhasil cleanup

```bash
cd ~/docker-lab/compose-app

docker compose down
docker compose ps
```

**Bukti berhasil:** container stack berhenti/terhapus dan `docker compose ps` tidak lagi menampilkan container running.

> Catatan: jangan pakai `docker compose down -v` kecuali memang ingin menghapus volume database.

---

## Ringkasan bukti yang bisa dimasukkan ke laporan

```bash
docker network inspect lab-net
docker exec server-a ping -c 3 server-b
docker exec server-c ping -c 3 server-d
docker volume ls
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
ls -lh ~/docker-lab/data-vol-backup.tar.gz
curl http://localhost:8080
curl http://localhost:8080/api/health | python3 -m json.tool
cd ~/docker-lab/compose-app && docker compose ps
```
