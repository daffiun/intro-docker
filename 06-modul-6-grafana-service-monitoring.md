# Modul 6: Grafana Service dan Monitoring Resource
1.  docker compose ps — 9 service running

![Screenshot 01](assets/modul-06/screenshot-01.png)

2.  Prometheus Targets — semua status UP

![Screenshot 02](assets/modul-06/screenshot-02.png)

3.  Prometheus query browser — PromQL CPU usage

![Screenshot 03](assets/modul-06/screenshot-03.png)

4.  curl localhost:5000/metrics — Flask Prometheus metrics

![Screenshot 04](assets/modul-06/screenshot-04.png)

5.  Grafana login — halaman utama setelah login

![Screenshot 05](assets/modul-06/screenshot-05.png)

6.  Data sources — Prometheus dan PostgreSQL keduanya OK

![Screenshot 06](assets/modul-06/screenshot-06.png)

![Screenshot 07](assets/modul-06/screenshot-07.png)

7.  Dashboard Docker Host Overview — keseluruhan

![Screenshot 08](assets/modul-06/screenshot-08.png)

8.  Dashboard Docker Host Overview — gauge CPU/Memory saat stress test (lonjakan terlihat)

![Screenshot 09](assets/modul-06/screenshot-09.png)

lonjakan di ujung warna ungu

9.  Dashboard Container Metrics — CPU per container

![Screenshot 10](assets/modul-06/screenshot-10.png)

10. Dashboard Container Metrics — Memory per container

![Screenshot 11](assets/modul-06/screenshot-11.png)

11. Dashboard Log Analytics — log volume time-series

![Screenshot 12](assets/modul-06/screenshot-12.png)

12. Dashboard Log Analytics — pie chart level distribution

![Screenshot 13](assets/modul-06/screenshot-13.png)

13. Custom panel yang dibuat — Flask HTTP Request

![Screenshot 14](assets/modul-06/screenshot-14.png)

14. Alerting rules — daftar alert yang dikonfigurasi

![Screenshot 15](assets/modul-06/screenshot-15.png)

15. Post Test

1.  Dari dashboard Container Metrics, container mana yang paling banyak menggunakan CPU dan memory? Mengapa?

Berdasarkan dashboard Container Metrics, penggunaan CPU terbesar terlihat pada /restricted sekitar 3.69%, diikuti oleh /docker sekitar 3.16%. Label /restricted dan /docker muncul karena cAdvisor pada environment WSL2/Docker Desktop membaca penggunaan resource berdasarkan Linux cgroup path, bukan langsung nama container Docker Compose. Nilai /docker cukup tinggi karena mencerminkan aktivitas container runtime Docker secara keseluruhan, termasuk Prometheus, Grafana, PostgreSQL, Flask App, Fluent Bit, dan container lain yang sedang berjalan. Oleh karena itu, penggunaan CPU terbesar terlihat pada grup proses Docker dibanding service individual.

![Screenshot 16](assets/modul-06/screenshot-16.png)

![Screenshot 17](assets/modul-06/screenshot-17.png)

2.  Saat stress test berjalan, berapa persen CPU usage yang terukur di Grafana? Bandingkan dengan output top atau htop di host.

Saat stress test dijalankan menggunakan stress --cpu 2 --timeout 60, penggunaan CPU pada dashboard Grafana meningkat menjadi sekitar 34.7%. Pada saat yang sama, output top/htop pada host menunjukkan penggunaan CPU sekitar 100%.
Nilai antara Grafana dan top tidak selalu identik karena Grafana memperoleh data dari Prometheus menggunakan proses scraping dan fungsi rate() dalam interval waktu tertentu, sedangkan top/htop menampilkan kondisi CPU secara lebih real-time. Namun keduanya tetap menunjukkan pola yang sama, yaitu kenaikan penggunaan CPU saat stress test dijalankan.

![Screenshot 18](assets/modul-06/screenshot-18.png)

![Screenshot 19](assets/modul-06/screenshot-19.png)

3.  Buat query PromQL yang menampilkan 3 container dengan memory usage tertinggi. Tunjukkan query dan hasilnya.

topk(3, sum by (id) (container_memory_usage_bytes{id!="/"}) / 1024 / 1024)

![Screenshot 20](assets/modul-06/screenshot-20.png)

4.  Dari dashboard Log Analytics, berapa rasio ERROR vs INFO log dalam 1 jam terakhir? Apakah ini normal untuk aplikasi production?

![Screenshot 21](assets/modul-06/screenshot-21.png)

Berdasarkan dashboard Log Analytics pada panel Log Level Distribution, jumlah log INFO jauh lebih banyak dibanding log ERROR. Dari hasil pengamatan, log INFO sekitar 1260, sedangkan log ERROR sekitar 162. Dengan demikian rasio ERROR terhadap INFO adalah sekitar 12.9%.
Untuk aplikasi production, rasio error sebesar 12.9% tergolong cukup tinggi dan biasanya tidak dianggap normal, karena log ERROR seharusnya jauh lebih sedikit dibanding log INFO. Namun pada praktikum ini kondisi tersebut masih wajar karena sistem menggunakan log-generator untuk mensimulasikan berbagai level log, termasuk ERROR dan CRITICAL, agar dapat digunakan untuk pengujian monitoring dan analisis log.

5.  Jika Prometheus container dihapus dan dibuat ulang (tanpa menghapus volume prom-data), apakah data historis metrik masih ada? Buktikan.

Sebelum menghapus

![Screenshot 22](assets/modul-06/screenshot-22.png)

setelah itu menjalankan docker rm -f prometheus untuk menghapus container prometheus saja. Dilanjutkan dengan docker compose up -d prometheus untuk menghidupkan container lagi dan sleep 20 untuk menunggu.

![Screenshot 23](assets/modul-06/screenshot-23.png)

data historis metrik masih ada setelah container Prometheus dihapus dan dibuat ulang, selama volume prom-data tidak dihapus. Hal ini dibuktikan dengan grafik Prometheus yang masih menampilkan data metrik sebelum container Prometheus dihapus. Penyebabnya adalah Prometheus menyimpan time-series database di Docker volume prom-data, bukan hanya di filesystem container. Data historis baru hilang jika volume ikut dihapus, misalnya menggunakan docker compose down -v.
