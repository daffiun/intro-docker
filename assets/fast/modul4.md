# MODUL 4 — Jawaban Checklist Database Service Docker — PostgreSQL

> Tujuan file ini: menjalankan dan membuktikan setiap item checklist Modul 4.  
> Project diasumsikan berada di `~/docker-lab/postgresql`.

---

## 0. Masuk folder project

```bash
cd ~/docker-lab/postgresql
```

Jalankan stack jika belum berjalan:

```bash
docker compose up -d
```

---

## 1. PostgreSQL container running dan healthy — `docker compose ps`

```bash
docker compose ps
```

**Bukti berhasil:** service `db` / container `postgres-db` berstatus `running` dan `healthy`.

Jika belum healthy:

```bash
docker compose logs db --tail 50
```

---

## 2. Init script berhasil — tabel di schema `app` ada

```bash
docker exec postgres-db psql -U labuser -d labdb -c "\dn"
docker exec postgres-db psql -U labuser -d labdb -c "\dt app.*"
```

**Bukti berhasil:** schema `app` ada dan tabel berikut tampil:

- `app.mahasiswa`
- `app.matakuliah`
- `app.nilai`
- `app.activity_log`

---

## 3. `psql` dari host bisa connect

Jika `psql` belum ada di host:

```bash
sudo apt install -y postgresql-client
```

Tes koneksi dari host:

```bash
PGPASSWORD=labpass123 psql -h localhost -U labuser -d labdb -c "SELECT current_database(), current_user;"
```

**Bukti berhasil:** output menampilkan:

```text
current_database | current_user
labdb            | labuser
```

---

## 4. pgAdmin4 bisa diakses dan connect ke database

Buka browser:

```text
http://localhost:5050
```

Login pgAdmin:

```text
Email    : admin@pens.ac.id
Password : admin123
```

Tambahkan server baru:

```text
Name     : Lab PostgreSQL
Host     : db
Port     : 5432
Database : labdb
Username : labuser
Password : labpass123
```

**Bukti berhasil:** pgAdmin menampilkan database `labdb`, schema `app`, dan tabel `mahasiswa`, `matakuliah`, `nilai`, `activity_log`.

---

## 5. Query CRUD berhasil — INSERT, SELECT, UPDATE, DELETE

Jalankan satu blok CRUD:

```bash
docker exec -i postgres-db psql -U labuser -d labdb << 'SQLEOF'
-- CREATE / INSERT
INSERT INTO app.mahasiswa (nrp, nama, kelas, kelompok, email)
VALUES ('3122600099', 'Test Checklist', 'D', 9, 'test.checklist@student.pens.ac.id')
ON CONFLICT (nrp) DO UPDATE
SET nama = EXCLUDED.nama,
    kelas = EXCLUDED.kelas,
    kelompok = EXCLUDED.kelompok,
    email = EXCLUDED.email;

-- READ / SELECT
SELECT id, nrp, nama, kelas, kelompok, email
FROM app.mahasiswa
WHERE nrp = '3122600099';

-- UPDATE
UPDATE app.mahasiswa
SET nama = 'Test Checklist Updated'
WHERE nrp = '3122600099';

SELECT id, nrp, nama, kelas, kelompok, email
FROM app.mahasiswa
WHERE nrp = '3122600099';

-- DELETE
DELETE FROM app.mahasiswa
WHERE nrp = '3122600099';

SELECT COUNT(*) AS sisa_data_test
FROM app.mahasiswa
WHERE nrp = '3122600099';
SQLEOF
```

**Bukti berhasil:**

- INSERT/SELECT menampilkan data `3122600099`.
- UPDATE mengubah nama menjadi `Test Checklist Updated`.
- DELETE membuat `sisa_data_test = 0`.

---

## 6. Backup berhasil — file `.dump` dan `.sql` ada di `./backup/`

```bash
mkdir -p backup

docker exec postgres-db pg_dump -U labuser -d labdb -Fc \
  -f /backup/labdb_backup.dump

docker exec postgres-db pg_dump -U labuser -d labdb \
  -f /backup/labdb_backup.sql

ls -lh backup/
```

**Bukti berhasil:** folder `backup/` berisi:

- `labdb_backup.dump`
- `labdb_backup.sql`

Ukuran file tidak boleh `0 bytes`.

---

## 7. Restore berhasil — data bisa dibaca di database `labdb_restore`

Buat ulang database restore:

```bash
docker exec postgres-db psql -U labuser -d postgres -c "DROP DATABASE IF EXISTS labdb_restore;"
docker exec postgres-db psql -U labuser -d postgres -c "CREATE DATABASE labdb_restore;"
```

Restore:

```bash
docker exec postgres-db pg_restore -U labuser -d labdb_restore \
  /backup/labdb_backup.dump
```

Verifikasi:

```bash
docker exec postgres-db psql -U labuser -d labdb_restore -c "\dt app.*"
docker exec postgres-db psql -U labuser -d labdb_restore -c "SELECT * FROM app.mahasiswa LIMIT 5;"
```

**Bukti berhasil:** tabel `app.*` ada di `labdb_restore` dan data mahasiswa bisa dibaca.

---

## 8. Data persist — setelah `docker compose down` + `up`, data masih ada

Tambahkan data persist-test:

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"INSERT INTO app.mahasiswa (nrp, nama, kelas, kelompok, email)
 VALUES ('3122600088', 'Persist Test', 'A', 8, 'persist@student.pens.ac.id')
 ON CONFLICT (nrp) DO UPDATE SET nama = EXCLUDED.nama;"
```

Restart stack tanpa menghapus volume:

```bash
docker compose down
docker compose up -d
sleep 10
docker compose ps
```

Cek data:

```bash
docker exec postgres-db psql -U labuser -d labdb -c \
"SELECT nrp, nama FROM app.mahasiswa WHERE nrp = '3122600088';"
```

**Bukti berhasil:** data `Persist Test` masih ada setelah `down` + `up`.

> Jangan gunakan `docker compose down -v` untuk bukti persistensi, karena flag `-v` memang menghapus volume.

---

## 9. Monitoring query berfungsi — `pg_stat_activity`, `pg_database_size`

```bash
docker exec -i postgres-db psql -U labuser -d labdb << 'SQLEOF'
SELECT pg_database.datname,
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

SELECT pid, usename, datname, client_addr, state, query_start, query
FROM pg_stat_activity
WHERE datname = 'labdb';

SELECT relname,
       seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch,
       n_tup_ins, n_tup_upd, n_tup_del
FROM pg_stat_user_tables
WHERE schemaname = 'app';
SQLEOF
```

**Bukti berhasil:** query menghasilkan tabel statistik database, koneksi aktif, dan statistik tabel.

---

## 10. Log PostgreSQL bisa dibaca di `/var/log/postgresql/`

```bash
docker exec postgres-db ls -lah /var/log/postgresql/
docker exec postgres-db sh -c "tail -30 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log"
```

**Bukti berhasil:** file log PostgreSQL ada dan bisa dibaca. Jika nama file tanggal berbeda, cek dulu dengan:

```bash
docker exec postgres-db find /var/log/postgresql -type f -maxdepth 1 -print
```

---

## Ringkasan bukti yang bisa dimasukkan ke laporan

```bash
cd ~/docker-lab/postgresql
docker compose ps
docker exec postgres-db psql -U labuser -d labdb -c "\dt app.*"
PGPASSWORD=labpass123 psql -h localhost -U labuser -d labdb -c "SELECT current_database(), current_user;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT * FROM app.mahasiswa LIMIT 5;"
ls -lh backup/
docker exec postgres-db psql -U labuser -d labdb_restore -c "SELECT * FROM app.mahasiswa LIMIT 5;"
docker exec postgres-db psql -U labuser -d labdb -c "SELECT pg_size_pretty(pg_database_size('labdb'));"
docker exec postgres-db ls -lah /var/log/postgresql/
```
