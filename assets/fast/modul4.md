# =========================================================
# CHECKLIST MODUL 4 — POSTGRESQL DI DOCKER
# =========================================================

# Masuk folder project
cd ~/docker-lab/postgresql


# ---------------------------------------------------------
# 1. PostgreSQL container running dan healthy
# Checklist: docker compose ps
# ---------------------------------------------------------
docker compose up -d
docker compose ps


# ---------------------------------------------------------
# 2. Init script berhasil — tabel di schema app ada
# Checklist: tabel app.* muncul
# ---------------------------------------------------------
docker exec postgres-db psql -U labuser -d labdb -c "\dt app.*"

# Bukti tambahan: data sample dari init script
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM app.mahasiswa;"


# ---------------------------------------------------------
# 3. psql dari host bisa connect
# Checklist: psql -h localhost -U labuser -d labdb
# ---------------------------------------------------------

# Install psql client jika belum ada
sudo apt install -y postgresql-client

# Connect dari host
psql -h localhost -U labuser -d labdb

# Setelah masuk psql, jalankan:
# \dt app.*
# SELECT * FROM app.mahasiswa;
# \q


# ---------------------------------------------------------
# 4. pgAdmin4 bisa diakses dan connect ke database
# Checklist: http://localhost:5050
# ---------------------------------------------------------

# Buka browser:
# http://localhost:5050
#
# Login:
# Email    : admin@pens.ac.id
# Password : admin123
#
# Add New Server:
# Name     : Lab PostgreSQL
# Host     : db
# Port     : 5432
# Database : labdb
# Username : labuser
# Password : labpass123
#
# Setelah connect:
# Servers → Lab PostgreSQL → Databases → labdb → Schemas → app → Tables


# ---------------------------------------------------------
# 5. Query CRUD berhasil — INSERT, SELECT, UPDATE, DELETE
# ---------------------------------------------------------

# CREATE / INSERT
docker exec postgres-db psql -U labuser -d labdb -c "
INSERT INTO app.mahasiswa (nrp, nama, kelas, kelompok, email)
VALUES ('3122600010', 'Fajar Rizki', 'D', 5, 'fajar@student.pens.ac.id');
"

# READ / SELECT
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT * FROM app.mahasiswa WHERE nrp = '3122600010';
"

# UPDATE
docker exec postgres-db psql -U labuser -d labdb -c "
UPDATE app.mahasiswa
SET email = 'fajar.rizki@student.pens.ac.id'
WHERE nrp = '3122600010';
"

# CEK UPDATE
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT nrp, nama, email FROM app.mahasiswa WHERE nrp = '3122600010';
"

# DELETE
docker exec postgres-db psql -U labuser -d labdb -c "
DELETE FROM app.mahasiswa WHERE nrp = '3122600010';
"

# CEK DELETE
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT * FROM app.mahasiswa WHERE nrp = '3122600010';
"


# ---------------------------------------------------------
# 6. Backup berhasil — file .dump dan .sql ada di ./backup/
# ---------------------------------------------------------

# Backup custom format
docker exec postgres-db pg_dump -U labuser -d labdb -Fc \
  -f /backup/labdb_backup.dump

# Backup SQL plain text
docker exec postgres-db pg_dump -U labuser -d labdb \
  -f /backup/labdb_backup.sql

# Cek file backup di host
ls -la backup/


# ---------------------------------------------------------
# 7. Restore berhasil — data bisa dibaca di labdb_restore
# ---------------------------------------------------------

# Hapus database restore jika sudah ada
docker exec postgres-db psql -U labuser -d postgres -c "
DROP DATABASE IF EXISTS labdb_restore WITH (FORCE);
"

# Buat database restore
docker exec postgres-db psql -U labuser -d postgres -c "
CREATE DATABASE labdb_restore;
"

# Restore dari backup
docker exec postgres-db pg_restore -U labuser -d labdb_restore \
  /backup/labdb_backup.dump

# Verifikasi data hasil restore
docker exec postgres-db psql -U labuser -d labdb_restore -c "
SELECT * FROM app.mahasiswa;
"


# ---------------------------------------------------------
# 8. Data persist — setelah docker compose down + up, data masih ada
# ---------------------------------------------------------

# Cek jumlah data sebelum down
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT COUNT(*) FROM app.mahasiswa;
"

# Down tanpa hapus volume
docker compose down

# Up lagi
docker compose up -d

# Cek data lagi
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT COUNT(*) FROM app.mahasiswa;
"


# ---------------------------------------------------------
# 9. Monitoring query berfungsi — pg_stat_activity, pg_database_size
# ---------------------------------------------------------

# Cek ukuran database
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT pg_database.datname,
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
"

# Cek koneksi aktif
docker exec postgres-db psql -U labuser -d labdb -c "
SELECT pid, usename, datname, client_addr, state, query_start, query
FROM pg_stat_activity
WHERE datname = 'labdb';
"


# ---------------------------------------------------------
# 10. Log PostgreSQL bisa dibaca di /var/log/postgresql/
# ---------------------------------------------------------

# Lihat daftar file log
docker exec postgres-db ls -la /var/log/postgresql/

# Lihat isi log terbaru
docker exec postgres-db sh -c 'cat /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log | tail -30'