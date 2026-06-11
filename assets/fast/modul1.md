# 0. Persiapan
ping -c 3 google.com
sudo apt update
docker rm -f web-server web-public ubuntu-test pens-app 2>/dev/null || true

# 1. Verifikasi Docker
docker version
docker info
sudo systemctl status docker
docker ps

# 2. Hello world
docker run hello-world

# 3. Pull image
docker pull nginx
docker pull nginx:1.26
docker pull ubuntu:22.04
docker pull alpine:3.20

# 4. List image
docker images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# 5. Run Nginx background
docker run -d --name web-server nginx
docker ps

# 6. Run Nginx dengan port mapping
docker run -d --name web-public -p 8080:80 nginx
curl http://localhost:8080

# 7. Ubuntu interaktif
docker run -dit --name ubuntu-test ubuntu:22.04 /bin/bash
docker exec -it ubuntu-test /bin/bash

# 8. Log dan stats
curl http://localhost:8080
docker logs web-public
docker stats --no-stream

# 9. Build custom image
mkdir -p ~/docker-lab/custom-web
cd ~/docker-lab/custom-web

cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><title>Docker Lab - PENS</title></head>
<body>
<h1>Docker Lab PENS</h1>
<p>Container berhasil berjalan!</p>
<p>Praktikum: Modul 1 — Instalasi Docker</p>
</body>
</html>
EOF

cat > Dockerfile << 'EOF'
FROM nginx:1.26-alpine
LABEL maintainer="admin@pens.ac.id"
LABEL description="Custom Nginx untuk praktikum Docker PENS"
LABEL version="1.0"
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

docker build -t pens-web:1.0 .
docker images | grep pens-web
docker run -d --name pens-app -p 9090:80 pens-web:1.0
curl http://localhost:9090

# 10. Layer dan disk usage
docker image history nginx:1.26-alpine
docker image history pens-web:1.0
docker system df