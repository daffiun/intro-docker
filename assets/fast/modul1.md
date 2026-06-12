# MODUL 1 — Jawaban Checklist Docker dan Instalasi

> Tujuan file ini: menjalankan dan membuktikan setiap item checklist Modul 1.  
> Jalankan di Ubuntu/WSL yang sudah menjadi environment praktikum.

---

## 0. Cek environment dasar

```bash
ping -c 3 google.com
docker --version
```

**Bukti berhasil:** koneksi internet ada dan perintah `docker --version` menampilkan versi Docker.

---

## 1. Docker Engine terinstal — `docker version` menampilkan Client dan Server

```bash
docker version
```

**Bukti berhasil:** output memiliki dua bagian:

- `Client: Docker Engine`
- `Server: Docker Engine`

Jika bagian `Server` tidak muncul, Docker daemon belum berjalan.

---

## 2. Docker Daemon running — status `active (running)`

```bash
sudo systemctl status docker --no-pager
```

**Bukti berhasil:** ada teks:

```text
Active: active (running)
```

Jika belum aktif:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 3. User non-root bisa menjalankan Docker tanpa `sudo`

```bash
docker ps
```

**Bukti berhasil:** tidak muncul error `permission denied`.

Jika masih error permission:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

---

## 4. `docker run hello-world` berhasil

```bash
docker run --rm hello-world
```

**Bukti berhasil:** output menampilkan:

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## 5. `docker pull nginx` berhasil download image

```bash
docker pull nginx
```

**Bukti berhasil:** output berakhir dengan pesan seperti:

```text
Status: Downloaded newer image for nginx:latest
```

atau:

```text
Status: Image is up to date for nginx:latest
```

---

## 6. `docker images` menampilkan image lokal

```bash
docker images
```

**Bukti berhasil:** daftar image tampil, minimal ada:

```text
nginx
hello-world
```

---

## 7. Container Nginx berjalan di background

Bersihkan container lama jika ada:

```bash
docker rm -f web-public web-server 2>/dev/null || true
```

Jalankan Nginx detached:

```bash
docker run -d --name web-public -p 8080:80 nginx
docker ps --filter "name=web-public"
```

**Bukti berhasil:** kolom `STATUS` menampilkan `Up`.

---

## 8. Port mapping berfungsi — akses `http://localhost:8080`

```bash
curl -I http://localhost:8080
```

**Bukti berhasil:** muncul response HTTP, biasanya:

```text
HTTP/1.1 200 OK
Server: nginx
```

Bukti browser: buka `http://localhost:8080`, halaman default Nginx tampil.

---

## 9. Container interaktif Ubuntu bisa dijalankan

```bash
docker rm -f ubuntu-test 2>/dev/null || true

docker run -d --name ubuntu-test ubuntu:22.04 sleep 3600

docker exec -it ubuntu-test bash -lc "cat /etc/os-release && echo 'exec berhasil dari dalam container'"
```

**Bukti berhasil:** output menampilkan informasi Ubuntu dan teks:

```text
exec berhasil dari dalam container
```

---

## 10. `docker logs` menampilkan log container Nginx

Generate request dulu agar log ada:

```bash
curl -s http://localhost:8080 > /dev/null
docker logs --tail 20 web-public
```

**Bukti berhasil:** log request HTTP muncul, misalnya ada `GET /`.

---

## 11. `docker stats` menampilkan resource usage container

```bash
docker stats --no-stream web-public
```

**Bukti berhasil:** muncul kolom:

```text
CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT
```

---

## 12. Dockerfile berhasil di-build

Masuk/buat folder project:

```bash
mkdir -p ~/docker-lab/custom-web
cd ~/docker-lab/custom-web
```

Buat `index.html`:

```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Docker Lab PENS</title>
</head>
<body>
  <h1>Docker Lab PENS</h1>
  <p>Custom image berhasil berjalan.</p>
</body>
</html>
EOF
```

Buat `Dockerfile`:

```bash
cat > Dockerfile << 'EOF'
FROM nginx:1.26-alpine
LABEL description="Custom Nginx untuk praktikum Docker PENS"
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

Build image:

```bash
docker build -t pens-web:1.0 .
```

**Bukti berhasil:** build selesai tanpa error dan muncul:

```text
Successfully tagged pens-web:1.0
```

---

## 13. Custom image `pens-web:1.0` berjalan di port 9090

```bash
docker rm -f pens-app 2>/dev/null || true

docker run -d --name pens-app -p 9090:80 pens-web:1.0

curl http://localhost:9090
```

**Bukti berhasil:** output HTML memuat teks:

```text
Docker Lab PENS
Custom image berhasil berjalan.
```

Bukti browser: buka `http://localhost:9090`.

---

## 14. `docker system df` menampilkan disk usage Docker

```bash
docker system df
```

**Bukti berhasil:** muncul ringkasan:

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images
Containers
Local Volumes
Build Cache
```

---

## Ringkasan bukti yang bisa dimasukkan ke laporan

```bash
docker version
sudo systemctl status docker --no-pager
docker run --rm hello-world
docker images
docker ps
curl -I http://localhost:8080
docker exec -it ubuntu-test bash -lc "cat /etc/os-release"
docker logs --tail 20 web-public
docker stats --no-stream web-public
cd ~/docker-lab/custom-web && docker build -t pens-web:1.0 .
curl http://localhost:9090
docker system df
```
