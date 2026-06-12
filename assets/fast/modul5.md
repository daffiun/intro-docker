# MODUL 5 — Jawaban Checklist Logging Service Docker dengan PostgreSQL

> Tujuan file ini: menjalankan dan membuktikan setiap item checklist Modul 5.  
> Project diasumsikan berada di `~/docker-lab/logging`.

---

## 0. Masuk folder project dan jalankan stack dari awal

```bash
cd ~/docker-lab/logging
```

Jika ingin mulai bersih dari awal praktikum logging:

```bash
docker compose down -v 2>/dev/null || true
docker compose up --build -d
```

Tunggu log terkumpul dan buat traffic:

```bash
sleep 30

for i in $(seq 1 20); do curl -s http://localhost:8080 > /dev/null; done
for i in $(seq 1 10); do curl -s http://localhost:5000/ > /dev/null; done

sleep 10
```

> Catatan: `down -v` menghapus volume PostgreSQL. Gunakan hanya untuk reset lab, bukan untuk mempertahankan data lama.

---

## 1. PostgreSQL running + tabel `logs.fluentbit` ada

```bash
docker compose ps postgres-db

docker exec postgres-db psql -U labuser -d labdb -c "\dt logs.*"
```

**Bukti berhasil:** output menampilkan tabel:

```text
logs.fluentbit
```

---

## 2. Fluent Bit running tanpa error pgsql

```bash
docker compose ps fluent-bit
docker compose logs fluent-bit --tail=80
```

Cek error fatal:

```bash
docker compose logs fluent-bit 2>&1 | \
  grep -iE "error|failed|relation .*does not exist|connection refused" || \
  echo "Tidak ada error fatal Fluent Bit/PostgreSQL"
```

**Bukti berhasil:**

- `fluent-bit` berstatus `Up`.
- Tidak ada error seperti `relation "logs.fluentbit" does not exist`, `connection refused`, atau gagal insert PostgreSQL.

---

## 3. 3 log producer running — nginx, flask, generator

```bash
docker compose ps nginx-web flask-app log-generator
```

**Bukti berhasil:** ketiga service berikut berstatus `Up`:

- `nginx-web`
- `flask-app`
- `log-generator`

---

## 4. Log masuk ke PostgreSQL — `COUNT(*) > 0`

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT COUNT(*) AS total_logs FROM logs.fluentbit;"
```

**Bukti berhasil:** `total_logs` lebih dari `0`.

Jika masih `0`, generate traffic ulang:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8080 > /dev/null; done
for i in $(seq 1 10); do curl -s http://localhost:5000/ > /dev/null; done
sleep 10
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT COUNT(*) AS total_logs FROM logs.fluentbit;"
```

---

## 5. Raw data terlihat — `SELECT tag, time, data FROM logs.fluentbit LIMIT 3`

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT tag, time, LEFT(data::text, 180) AS data_preview
 FROM logs.fluentbit
 ORDER BY time DESC
 LIMIT 3;"
```

**Bukti berhasil:** output menampilkan kolom `tag`, `time`, dan `data_preview`.

---

## 6. View `recent_logs` menampilkan log terbaru

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT * FROM logs.recent_logs LIMIT 10;"
```

**Bukti berhasil:** view menampilkan daftar log terbaru dengan kolom seperti `time`, `tag`, `container`, dan `log_preview`.

---

## 7. View `structured_logs` menampilkan parsed level dan message

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT received_at, tag, container_name, log_level, message
 FROM logs.structured_logs
 LIMIT 10;"
```

**Bukti berhasil:** output menampilkan kolom:

- `log_level`
- `message`
- `container_name`

Jika kosong, tunggu log-generator menghasilkan log JSON:

```bash
sleep 30
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT received_at, tag, container_name, log_level, message
 FROM logs.structured_logs
 LIMIT 10;"
```

---

## 8. Distribusi per tag — 3 tag berbeda

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT tag, COUNT(*) AS total
 FROM logs.fluentbit
 GROUP BY tag
 ORDER BY tag;"
```

**Bukti berhasil:** output menampilkan 3 tag:

```text
docker.flask
docker.generator
docker.nginx
```

Jika `docker.nginx` belum muncul, generate traffic ke Nginx:

```bash
for i in $(seq 1 20); do curl -s http://localhost:8080 > /dev/null; done
sleep 10
```

---

## 9. View `error_summary` menampilkan summary error/warn/critical

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT * FROM logs.error_summary;"
```

**Bukti berhasil:** output menampilkan summary level `WARN`, `ERROR`, atau `CRITICAL`.

Jika kosong, tunggu generator karena level error dibuat secara acak:

```bash
sleep 60
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT * FROM logs.error_summary;"
```

---

## 10. API `/api/logs/stats` menampilkan statistik JSON

```bash
curl -s http://localhost:5000/api/logs/stats | python3 -m json.tool
```

**Bukti berhasil:** response JSON memuat statistik seperti:

```json
"total_logs_all_time": ...
"last_hour_by_tag": ...
"last_hour_by_level": ...
```

---

## 11. API `/api/logs/search?q=error` menampilkan hasil

```bash
curl -s "http://localhost:5000/api/logs/search?q=error&limit=5" | python3 -m json.tool
```

**Bukti berhasil:** response JSON menampilkan hasil pencarian log yang mengandung keyword `error`.

Jika hasil kosong karena belum ada log error, gunakan pencarian keyword umum:

```bash
curl -s "http://localhost:5000/api/logs/search?q=User&limit=5" | python3 -m json.tool
```

---

## 12. API `/api/logs/raw` menampilkan log termasuk Nginx plain text

```bash
curl -s "http://localhost:5000/api/logs/raw?limit=10" | python3 -m json.tool
```

**Bukti berhasil:** response JSON menampilkan raw log. Untuk membuktikan Nginx plain text ada, jalankan juga:

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT time, LEFT(data->>'log', 180) AS nginx_plain_text
 FROM logs.fluentbit
 WHERE tag = 'docker.nginx'
 ORDER BY time DESC
 LIMIT 5;"
```

**Bukti berhasil:** output Nginx berupa access log plain text, bukan JSON structured log.

---

## Ringkasan bukti yang bisa dimasukkan ke laporan

```bash
cd ~/docker-lab/logging
docker compose ps
docker compose logs fluent-bit --tail=80
docker exec postgres-db psql -U labuser -d labdb -c "\dt logs.*"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT COUNT(*) AS total_logs FROM logs.fluentbit;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT tag, time, LEFT(data::text, 180) AS data_preview FROM logs.fluentbit ORDER BY time DESC LIMIT 3;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM logs.recent_logs LIMIT 10;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT received_at, tag, container_name, log_level, message FROM logs.structured_logs LIMIT 10;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT tag, COUNT(*) AS total FROM logs.fluentbit GROUP BY tag ORDER BY tag;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM logs.error_summary;"
curl -s http://localhost:5000/api/logs/stats | python3 -m json.tool
curl -s "http://localhost:5000/api/logs/search?q=error&limit=5" | python3 -m json.tool
curl -s "http://localhost:5000/api/logs/raw?limit=10" | python3 -m json.tool
```
