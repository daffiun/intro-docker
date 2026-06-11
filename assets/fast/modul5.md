# Masuk folder project
cd ~/docker-lab/logging

# Jalankan stack dari awal
docker compose down -v 2>/dev/null
docker compose up --build -d

# Cek semua service
docker compose ps

# Cek Fluent Bit error
docker compose logs fluent-bit 2>&1 | grep -i "error\|warn\|pgsql"

# Cek tabel logs.fluentbit
docker exec postgres-db psql -U labuser -d labdb -c "\dt logs.*"

# Generate log
sleep 30
for i in $(seq 1 20); do curl -s http://localhost:8080 > /dev/null; done
for i in $(seq 1 10); do curl -s http://localhost:5000/ > /dev/null; done
sleep 10

# Total log masuk
docker exec postgres-db psql -U labuser -d labdb -c "SELECT COUNT(*) FROM logs.fluentbit;"

# Raw data
docker exec postgres-db psql -U labuser -d labdb -c "SELECT tag, time, data FROM logs.fluentbit LIMIT 3;"

# Recent logs
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM logs.recent_logs LIMIT 10;"

# Structured logs
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM logs.structured_logs LIMIT 10;"

# Distribusi per tag
docker exec postgres-db psql -U labuser -d labdb -c "SELECT tag, COUNT(*) AS total FROM logs.fluentbit GROUP BY tag ORDER BY total DESC;"

# Distribusi per level
docker exec postgres-db psql -U labuser -d labdb -c "SELECT log_level, COUNT(*) AS total FROM logs.structured_logs GROUP BY log_level ORDER BY total DESC;"

# Error summary
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM logs.error_summary;"

# API stats
curl -s http://localhost:5000/api/logs/stats | python3 -m json.tool

# API search error
curl -s "http://localhost:5000/api/logs/search?q=error&limit=5" | python3 -m json.tool

# API raw logs
curl -s "http://localhost:5000/api/logs/raw?limit=10" | python3 -m json.tool