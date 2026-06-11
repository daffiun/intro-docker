# =========================================================
# CHECKLIST MODUL 2 — DOCKER SERVICE, VOLUME & MOUNT POINT
# =========================================================


# ---------------------------------------------------------
# 1. User-defined bridge lab-net — dua container bisa ping by nama
# ---------------------------------------------------------

docker network create --driver bridge --subnet 172.20.0.0/16 lab-net

docker run -d --name server-a --network lab-net nginx:alpine
docker run -d --name server-b --network lab-net nginx:alpine

docker network ls
docker network inspect lab-net

docker exec server-a ping -c 3 server-b
docker exec server-b ping -c 3 server-a


# ---------------------------------------------------------
# 2. Default bridge — container tidak bisa resolve nama
# ---------------------------------------------------------

docker run -d --name server-c nginx:alpine
docker run -d --name server-d nginx:alpine

docker exec server-c ping -c 3 server-d || true

# Expected:
# ping: bad address 'server-d'
# Artinya default bridge tidak otomatis resolve nama container.


# Cleanup test network
docker rm -f server-a server-b server-c server-d


# ---------------------------------------------------------
# 3. Named volume data-vol — data persist setelah container dihapus
# ---------------------------------------------------------

docker volume create data-vol
docker volume ls
docker volume inspect data-vol

docker run -d --name writer \
  -v data-vol:/app/data \
  alpine:3.20 sh -c "while true; do date >> /app/data/log.txt; sleep 5; done"

sleep 15

docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt

docker rm -f writer

# Buktikan data tetap ada walaupun container writer sudah dihapus
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt


# ---------------------------------------------------------
# 4. Volume backup/restore berhasil
# ---------------------------------------------------------

# Backup volume data-vol ke file tar.gz
docker run --rm \
  -v data-vol:/source:ro \
  -v $(pwd):/backup \
  alpine:3.20 tar czf /backup/data-vol-backup.tar.gz -C /source .

ls -lh data-vol-backup.tar.gz

# Restore ke volume baru
docker volume create data-vol-restored

docker run --rm \
  -v data-vol-restored:/target \
  -v $(pwd):/backup:ro \
  alpine:3.20 tar xzf /backup/data-vol-backup.tar.gz -C /target

# Verifikasi hasil restore
docker run --rm -v data-vol-restored:/data alpine:3.20 cat /data/log.txt


# ---------------------------------------------------------
# 5. Bind mount — edit file di host langsung terlihat di container
# ---------------------------------------------------------

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

docker run -d --name dev-server \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine

# Cek versi awal
curl http://localhost:8080

# Edit file di host
sed -i 's/VERSI-1/VERSI-2 (diedit live)/' html/index.html

# Cek lagi tanpa restart container
curl http://localhost:8080

# Cleanup bind mount demo
docker rm -f dev-server


# ---------------------------------------------------------
# 6. tmpfs — data hilang setelah container restart
# ---------------------------------------------------------

docker run -d --name tmpfs-demo \
  --tmpfs /app/cache:size=64m \
  alpine:3.20 sh -c "echo 'secret-data' > /app/cache/token.txt && sleep 3600"

# Data masih ada sebelum restart
docker exec tmpfs-demo cat /app/cache/token.txt

# Restart container
docker stop tmpfs-demo
docker start tmpfs-demo

# Data harus hilang
docker exec tmpfs-demo cat /app/cache/token.txt || true

# Expected:
# cat: can't open '/app/cache/token.txt': No such file or directory

docker rm -f tmpfs-demo


# ---------------------------------------------------------
# 7. docker compose up --build -d — 3 service running
# ---------------------------------------------------------

cd ~/docker-lab/compose-app

docker compose up --build -d

docker compose ps

# Expected service:
# lab-web
# lab-app
# lab-db
#
# Status minimal:
# Up
#
# Untuk database:
# healthy


# ---------------------------------------------------------
# 8. http://localhost:8080 menampilkan halaman web
# ---------------------------------------------------------

curl http://localhost:8080

# Expected:
# HTML halaman Docker Compose Lab tampil.
# Untuk screenshot, buka browser:
# http://localhost:8080


# ---------------------------------------------------------
# 9. http://localhost:8080/api/health menampilkan database connected
# ---------------------------------------------------------

curl http://localhost:8080/api/health

# Versi rapi
curl http://localhost:8080/api/health | python3 -m json.tool

# Expected:
# "status": "ok"
# "db_status": "connected"


# ---------------------------------------------------------
# 10. docker compose down berhasil cleanup
# ---------------------------------------------------------

docker compose down

docker compose ps

# Expected:
# Tidak ada container compose yang running.