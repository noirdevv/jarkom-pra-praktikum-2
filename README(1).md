# Pra-Praktikum Modul 2

## HTTP Server, DNS Server, dan Reverse Proxy

**Pre-requisite:**

a. GNS3 VM dan GNS3-GUI sudah terinstall.
b. Appliance `netics-pc` / `netics-server` dan `netics-pc-desktop`.

**Cakupan Modul:** HTTP Server (nginx), DNS Server (BIND), Integrasi DNS dengan DHCP, Reverse Proxy Berbasis Path, Reverse Proxy Multi-Backend Lintas Port.

> **Note:** Semua problem dan bagian wajib dikerjakan sebagai bahan persiapan untuk praktikum hari H, pengerjaan laporan mengikuti instruksi yang ada di readme github. IP Prefix disesuaikan dengan pembagian yang sudah diumumkan sebelumnya.

---

## Daftar Isi

- [Pemetaan Node](#pemetaan-node)
- [0. Pre](#0-pre)
- [Bagian A - HTTP Server](#a-bagian-a---http-server)
- [Bagian B - DNS Server](#b-bagian-b---dns-server)
- [Bagian C - Reverse Proxy](#c-bagian-c---reverse-proxy)
- [e. Verifikasi Menyeluruh](#e-verifikasi-menyeluruh)
- [f. Tugas Mandiri Bagian C](#f-tugas-mandiri-bagian-c)

---

## Pemetaan Node

| Node | Appliance | Peran | IP Address |
|------|-----------|-------|------------|
| Router | `netics-pc` | Menyediakan akses internet dari NAT | eth0: Auto (DHCP)<br>eth1: `[IP Prefix].200.1/24` (statis) |
| Sargon, Iberia, Higashi | `netics-pc` | Client uji command line (curl) | Auto (DHCP) |
| Ognisko | `netics-pc-desktop` | Client uji browser (VNC + Chromium) | Auto (DHCP) |
| Aegir | `netics-pc` | DHCP Server | `[IP Prefix].200.10/24` (statis)<br>Gateway: `[IP Prefix].200.1`<br>Pool DHCP: `[IP Prefix].200.100–150`<br>Tambahkan di konfigurasi dhcpd nya: `option router [IP Prefix].200.1` |
| Kazdel | `netics-pc` atau `netics-server` | Web Server (Bagian A) -> Reverse Proxy (Bagian C) | `[IP Prefix].200.11/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| Chernobog | `netics-pc` atau `netics-server` | DNS Server (BIND) | `[IP Prefix].200.12/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| web1 | `netics-pc` atau `netics-server` | Web1, backend HTTP server | `[IP Prefix].200.20/24` (statis), port 8080 (statis)<br>Gateway: `[IP Prefix].200.1` |
| web2 | `netics-pc` atau `netics-server` | Web2, backend HTTP server + aplikasi Python (Flask) | `[IP Prefix].200.21/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |

---

## 0. Pre

1. Konfigurasi Router sesuai dengan tabel pemetaan node di atas, kemudian hubungkan `eth0` router ke NAT.
2. Masuk ke console node Router, seharusnya secara otomatis akan mendapat IP untuk interface `eth0`.
3. Kemudian jalankan perintah berikut:

   ```sh
   apk update
   apk add iptables
   iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0 -s [IP Prefix].0.0/16
   ```

4. Pastikan kalian bisa `ping google.com` dari node router.
5. Di node DHCP Server (Aegir), jalankan:

   ```sh
   echo "nameserver 8.8.8.8" > /etc/resolv.conf
   ```

   Kemudian coba `ping google.com` dari node tersebut.
6. Konfigurasikan DHCP Server seperti pada modul 1 sebelumnya dan sesuai dengan ketentuan pool yang ada dalam tabel dengan menambahkan konfigurasi pada `dhcpd.conf` nya.

   ```
   option router [IP Prefix].200.1
   ```

---

## A. Bagian A - HTTP Server

### a. Tujuan Pembelajaran

1. Memahami HTTP server sebagai salah satu bentuk konkret aplikasi jaringan (*network application*) di Application Layer.
2. Memasang dan menjalankan web server nginx untuk keperluan information sharing sederhana.

### b. Tools yang Dibutuhkan

1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. Nginx.
3. `curl` (di Sargon) dan browser Chromium (di Ognisko).

### Skenario Topologi

Topologi yang dipakai pada bagian ini terdiri dari satu ethernet switch yang menghubungkan:

a. Sargon, Iberia, Higashi, dan Ognisko (semuanya ip config: auto, mendapat IP dari DHCP server).
b. Aegir sebagai DHCP server (`[IP Prefix].200.10/24`, pool `[IP Prefix].200.100–150`).
c. Kazdel sebagai web/HTTP server dengan IP statis `[IP Prefix].200.11/24`.

Dan satu node router (`netics-pc`) yang berfungsi sebagai penyedia internet melalui NAT.

Web server sengaja diberi IP statis (bukan DHCP) karena alamatnya perlu tetap dan dapat diprediksi, baik untuk diakses langsung oleh client, maupun untuk nantinya dipetakan oleh DNS server pada Bagian B.

### c. Problem A.1 - Setup HTTP Server

**Soal:**

Lakukan setup HTTP server/web server menggunakan nginx pada Kazdel dengan IP `[IP Prefix].200.11/24`.

Konvensi direktori yang dipakai sepanjang praktikum ini:

| Direktori | Fungsi |
|-----------|--------|
| `/root/myconfig` | Lokasi file konfigurasi (nginx.conf, start script) |
| `/root/myweb` | Lokasi file web (index.html) |
| `/root/mylogs` | Lokasi pencatatan log |

**Langkah Kerja:**

1. Setup IP address Kazdel ke `[IP Prefix].200.11/24`, dilakukan lewat edit konfigurasi network interface pada node (bukan `ip addr add` sementara), agar IP tetap statis setiap kali node dinyalakan ulang. Pada panel Kazdel interfaces, uncomment dan sesuaikan bagian static config:

   ```
   # Static config for eth0
   auto eth0
   iface eth0 inet static
           address [IP Prefix].200.11
           netmask 255.255.255.0
           gateway [IP Prefix].200.1
   ```

2. Install nginx menggunakan:

   ```sh
   echo "nameserver 8.8.8.8" > /etc/resolv.conf
   apk update
   apk add nginx
   ```

3. Buat struktur direktori:

   ```sh
   mkdir -p /root/myconfig
   mkdir -p /root/myweb
   mkdir -p /root/mylogs
   ```

4. Buat file `index.html` di `/root/myweb`:

   ```html
   <html>
       <head>
           <title>This is My Web</title>
       </head>
       <body>
           <p>
           This is my web
           </p>
       </body>
   </html>
   ```

5. Buat file `nginx.conf` di `/root/myconfig`:

   ```nginx
   user root;
   worker_processes auto;
   worker_cpu_affinity auto;
   pid /tmp/nginx.pid;
   error_log /root/mylogs/error.log;

   events { worker_connections 768; }

   http {
       server {
           listen 80;
           server_name _;

           root /root/myweb;
           index index.html;

           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```

6. Buat file `start_http.sh` di `/root/myconfig`:

   ```sh
   #!/bin/bash
   nginx -c /root/myconfig/nginx.conf -g 'daemon off;'
   ```

7. Berikan izin eksekusi buat file `start_http.sh`:

   ```sh
   chmod +x start_http.sh
   ```

8. Jalankan web server dengan perintah berikut di `/root/myconfig`:

   ```sh
   ./start_http.sh
   ```

**Indikator Keberhasilan:**

1. Verifikasi lewat browser:
   1. Buka console pada Ognisko menggunakan VNC viewer.
   2. Klik kanan pada layar desktop untuk memunculkan menu, lalu pilih *Desktops -> chromium*.
   3. Pada Chromium, ketik alamat URL `http://[IP Prefix].200.11`.
   4. Akan terlihat tampilan web yang bertuliskan "This is my web".

2. Verifikasi lewat command line:
   1. Dari Sargon, gunakan mode *Console Command Line Interface*, lalu jalankan:

      ```sh
      curl -vL http://[IP Prefix].200.11
      ```

   2. Contoh hasil yang diharapkan:

      ```
      * Trying [IP Prefix].200.11:80...
      * Connected to [IP Prefix].200.11 ([IP Prefix].200.11) port 80
      * using HTTP/1.x
      > GET / HTTP/1.1
      > Host: [IP Prefix].200.11
      > User-Agent: curl/8.14.1
      > Accept: */*
      >
      * Request completely sent off
      < HTTP/1.1 200 OK
      < Server: nginx/1.26.3
      < Date: Sat, 27 Sep 2025 13:31:41 GMT
      < Content-Type: text/html
      < Content-Length: 111
      < Last-Modified: Sat, 27 Sep 2025 07:46:19 GMT
      < Connection: keep-alive
      < ETag: "68d7964b-6f"
      < Accept-Ranges: bytes
      <
      <html>
              <head>
                      <title>This is My Web</title>
              </head>
              <body>
                      <p>
                      This is my web
                      </p>
              </body>
      </html>
      * Connection #0 to host [IP Prefix].200.11 left intact
      ```

---

## B. Bagian B - DNS Server

### a. Tujuan Pembelajaran

1. Memahami DNS sebagai mekanisme mapping nama domain ke IP address.
2. Memasang DNS server (BIND) dan mengintegrasikannya dengan DHCP agar seluruh client otomatis mengetahui alamat DNS server.

### b. Tools yang Dibutuhkan

1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. BIND (`named`).
3. DHCP server yang sudah aktif (untuk diintegrasikan dengan opsi DNS).

### c. Problem B.1 - Setup DNS Server

**Soal:**

Pasang DNS server pada Chernobog (IP `[IP Prefix].200.12/24`) untuk domain `netics.my.id`, dan integrasikan dengan DHCP server agar seluruh client otomatis mengetahui alamat DNS server ini.

**Langkah Kerja:**

1. Setup IP address Chernobog ke `[IP Prefix].200.12/24` dan set gateway address nya ke `[IP Prefix].200.1`.
2. Buat struktur direktori:

   ```sh
   mkdir -p /root/dns
   mkdir -p /root/dnsdata
   ```

3. Install bind menggunakan:

   ```sh
   echo "nameserver 8.8.8.8" > /etc/resolv.conf
   apk update
   apk add bind
   ```

4. Buat file `named.conf` di `/root/dns`:

   ```
   options {
       directory "/root/dns";
       listen-on { any; };
       allow-query { any; };
   };

   logging {
       channel default_log {
           file "/root/dns/named.log" versions 3 size 5m;
           severity info;
           print-time yes;
           print-severity yes;
           print-category yes;
       };
       category default { default_log; };
       category queries { default_log; };
   };

   zone "localhost" {
       type master;
       file "db.localhost";
   };

   zone "netics.my.id" {
       type master;
       file "db.netics.my.id";
   };
   ```

5. Buat file `db.localhost`:

   ```
   $TTL 86400
   @ IN SOA localhost. root.localhost. (
       1       ; serial
       3600    ; refresh
       1800    ; retry
       604800  ; expire
       86400 ) ; minimum

     IN NS localhost.
     IN A 127.0.0.1
   ```

6. Buat file `db.netics.my.id`, memetakan `www.netics.my.id` ke IP web server (Kazdel):

   ```
   $TTL 86400
   @ IN SOA ns1.netics.my.id. admin.netics.my.id. (
       1       ; serial
       3600    ; refresh
       1800    ; retry
       604800  ; expire
       86400 ) ; minimum

     IN NS ns1.netics.my.id.
   ns1 IN A [IP Prefix].200.12
   www IN A [IP Prefix].200.11
   ```

7. Buat file `start_dns.sh` di `/root/dns` dan jalankan DNS servernya:

   ```sh
   #!/bin/sh
   named -g -c /root/dns/named.conf
   ```

   Untuk melihat progress/aktivitas dari DNS Server, pantaulah di file `/root/dns/named.log` (gunakan `tmux`), dengan perintah:

   ```sh
   tail -f /root/dns/named.log
   ```

8. Buka Console pada node Chernobog (klik kanan node -> Console) untuk memantau proses `named` berjalan.

**Langkah Kerja, Integrasi dengan DHCP:**

Agar seluruh client otomatis mengetahui alamat DNS server tanpa perlu diset manual satu per satu:

1. Edit file `dhcpd.conf` pada DHCP server yang sudah ada, tambahkan baris:

   ```
   option dns [IP Prefix].200.12 8.8.8.8
   ```

2. Reload kembali DHCP server, lalu restart seluruh client agar setting DNS otomatis ter-set.

**Indikator Keberhasilan:**

1. Di Sargon, cek `/etc/resolv.conf`:

   ```sh
   cat /etc/resolv.conf
   ```

   Pastikan terdapat baris `domain` dan `nameserver` yang mengarah ke DNS server (`[IP Prefix].200.12`).

2. Lakukan `ping` dan `curl` memakai nama domain (bukan IP):

   ```sh
   ping www.netics.my.id
   curl -vL http://www.netics.my.id
   ```

3. Dari Ognisko, akses `http://www.netics.my.id` lewat Firefox.
4. Ulangi pengecekan dari `netics-pc` lain untuk memastikan konsisten di seluruh jaringan.

### d. Problem B.2 - Multi-Zone DNS dan Multi-Domain Web Server

**Soal:**

Perluas DNS server agar melayani dua zona sekaligus (`netics.my.id` dan `netics.org`), dan perluas web server agar melayani nama domain berbeda dengan lokasi index yang relevan: `http://www.netics.my.id/` dan `http://web.netics.org`.

**Langkah Kerja, DNS Server (Chernobog):**

1. Tambahkan definisi zona baru di `named.conf`:

   ```
   zone "netics.org" {
       type master;
       file "db.netics.org";
   };
   ```

2. Buat file `db.netics.org`, dengan record `web` mengarah ke IP web server:

   ```
   $TTL 86400
   @ IN SOA ns1.netics.org. admin.netics.org. (
       1       ; serial
       3600    ; refresh
       1800    ; retry
       604800  ; expire
       86400 ) ; minimum

     IN NS ns1.netics.org.
   ns1 IN A [IP Prefix].200.12
   web IN A [IP Prefix].200.11
   ```

3. Restart node Chernobog (stop lalu start kembali) agar konfigurasi baru terbaca.
4. Sebagai indikator keberhasilan, cek dari Sargon dan yang lain dengan `ping ns1.netics.org`.

**Langkah Kerja, Web Server (Kazdel):**

1. Buat direktori baru untuk konten `web.netics.org`:

   ```sh
   mkdir -p /root/myweb-netics-org
   ```

2. Buat file `index.html` di direktori tersebut:

   ```html
   <html>
   <head>
   <title>Hello</title>
   </head>
   <body>
   <p>
   This is NETICS-ORG web
   </p>
   </body>
   </html>
   ```

3. Ubah blok `http` pada `nginx.conf` agar melayani dua nama domain sekaligus dari root direktori yang berbeda:

   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;

           root /root/myweb;
           index index.html;

           location / {
               try_files $uri $uri/ =404;
           }
       }

       server {
           listen 80;
           server_name web.netics.org;

           root /root/myweb-netics-org;
           index index.html;

           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```

4. Restart node Kazdel (stop lalu start kembali).

**Indikator Keberhasilan:**

1. Dari Sargon dan `netics-pc` lain: `ping web.netics.org` dan `ping www.netics.my.id`, keduanya berhasil resolve (ke `[IP Prefix].200.11`).
2. Dari Iberia (atau client lain):

   ```sh
   curl -vL http://www.netics.my.id
   curl -vL http://web.netics.org
   ```

   Masing-masing dengan konten dan ETag berbeda.
3. Dari Ognisko, cek kedua alamat lewat Firefox (`www.netics.my.id` menampilkan "This is my web", `web.netics.org` menampilkan "This is NETICS-ORG web").

---

## C. Bagian C - Reverse Proxy

### a. Tujuan Pembelajaran

1. Memahami pemisahan fungsi antara reverse proxy dan real server pada arsitektur HTTP server.
2. Mengonfigurasi reverse proxy yang meneruskan request berdasarkan path URL ke backend yang berbeda, termasuk backend yang berjalan di port berbeda.

### b. Tools yang Dibutuhkan

1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. Nginx (reverse proxy dan real server).
3. Python3 dan Flask (untuk backend aplikasi dinamis).

Arsitektur reverse proxy: *internet users* mengakses `www.netics.my.id`, diteruskan oleh reverse proxy ke *real server* (Nginx, Nginx, Python) di belakang layar.

### c. Problem C.1 - Reverse Proxy di Depan Dua Web Server

**Soal:**

Implementasikan reverse proxy di depan dua web server berbeda:

a. Melayani base URL `http://www.netics.my.id`, meneruskan `/web1` ke Web1 (`http://[IP Prefix].200.20:8080`).
b. `/web2` ke Web2 (`http://[IP Prefix].200.21:8080`).

**Langkah Kerja, Web1 (web1):**

1. Setup IP address ke `[IP Prefix].200.20/24`.

   ```
   # Static config for eth0
   auto eth0
   iface eth0 inet static
           address [IP Prefix].200.20
           netmask 255.255.255.0
           gateway [IP Prefix].200.1
   ```

2. Buat struktur direktori (`myconfig`, `myweb`, `mylogs`) seperti Bagian A.
3. Jalankan command yang sama untuk instalasi nginx seperti pada bagian A.
4. Buat `index.html` seperti yang ada pada bagian A tapi berisi teks "Ini WEB-1".
5. Buat `nginx.conf` seperti yang ada pada bagian A, dan modifikasi blok `http`, kali ini nginx listen di port **8080** (bukan 80):

   ```nginx
   http {
       server {
           listen 8080;
           server_name _;

           root /root/myweb;
           index index.html;

           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```

6. Buat `start_http.sh` seperti yang ada di bagian A sebelumnya, dan set start command node.
7. Restart node Web1, verifikasi dari Sargon dan Ognisko:

   ```sh
   curl http://[IP Prefix].200.20:8080
   ```

   Hasil yang diharapkan: halaman dengan teks "Ini WEB-1" (baik via `curl` maupun browser).

**Langkah Kerja, Web2 (web2):**

1. Ulangi langkah yang sama persis seperti Web1, dengan perbedaan:
   - IP address `[IP Prefix].200.21/24`.
   - Isi `index.html`: "Ini WEB2-2".

2. Isi `index.html` di Web2:

   ```html
   <html>
       <head>
           <title>This is My Web</title>
       </head>
       <body>
           <p>
           Ini WEB-2
           </p>
       </body>
   </html>
   ```

3. Isi `nginx.conf` dan `start_http.sh` identik dengan Web1 (hanya root/direktori yang menyesuaikan `/root/myweb` milik Web2 sendiri).
4. Restart node Web2, verifikasi dari Sargon dan Ognisko:

   ```sh
   curl http://[IP Prefix].200.21:8080
   ```

**Langkah Kerja, Reverse Proxy (Kazdel):**

1. IP address tetap `[IP Prefix].200.11/24`, `server_name` tetap `www.netics.my.id`.
2. Ubah `nginx.conf` di `/root/myconfig` menjadi konfigurasi reverse proxy (menggantikan konfigurasi Bagian A/B):

   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;

           location = /web1 {
               proxy_pass http://[IP Prefix].200.20:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }

           location = /web2 {
               proxy_pass http://[IP Prefix].200.21:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
       }
   }
   ```

3. Restart node Kazdel.

**Indikator Keberhasilan:**

1. Dari Sargon:

   ```sh
   curl http://www.netics.my.id/web1
   curl http://www.netics.my.id/web2
   ```

   Masing-masing menghasilkan "Ini WEB-1" dan "Ini WEB-2", padahal keduanya diakses dari satu domain dan port (80) yang sama.
2. Dari Ognisko, cek juga kedua path lewat Firefox.

### d. Problem C.2 - Menambahkan Backend Aplikasi di Port Berbeda

**Soal:**

Terdapat aplikasi kalender berbasis Python yang ditempatkan di Web2, berjalan di port 5000. Perluas reverse proxy agar `/app` diarahkan ke aplikasi ini (`http://[IP Prefix].200.21:5000`), selain `/web1` dan `/web2` yang sudah ada.

**Langkah Kerja, Aplikasi di Web2 (web2):**

1. Buat direktori baru:

   ```sh
   mkdir -p /root/aplikasi
   ```

2. Install python menggunakan:

   ```sh
   echo "nameserver 8.8.8.8" > /etc/resolv.conf
   apk update
   apk add python3
   ```

3. Buat file `program.py` pada direktori `/root/aplikasi`:

   ```python
   import calendar
   from datetime import datetime
   from http.server import BaseHTTPRequestHandler, HTTPServer

   PORT = 5000


   class CalendarRequestHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           # 1. Get current year and month
           now = datetime.now()
           year = now.year
           month = now.month

           # 2. Generate calendar data
           cal = calendar.Calendar(
               firstweekday=6
           )  # 6 = Sunday start. Change to 0 for Monday.
           month_days = cal.monthdayscalendar(year, month)
           month_name = calendar.month_name[month]

           # 3. Build HTML response
           html = f"""<!DOCTYPE html>
           <html lang="en">
           <head>
               <meta charset="UTF-8">
               <meta name="viewport" content="width=device-width, initial-scale=1.0">
               <title>{month_name} {year} - Calendar</title>
               <style>
                   body {{
                       font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
                       background-color: #f4f7f6;
                       display: flex;
                       justify-content: center;
                       align-items: center;
                       height: 100vh;
                       margin: 0;
                   }}
                   .calendar-container {{
                       background: #ffffff;
                       padding: 24px;
                       border-radius: 12px;
                       box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
                       width: 100%;
                       max-width: 500px;
                   }}
                   h2 {{
                       text-align: center;
                       color: #2c3e50;
                       margin-top: 0;
                       margin-bottom: 20px;
                   }}
                   table {{
                       width: 100%;
                       border-collapse: collapse;
                   }}
                   th {{
                       background-color: #3498db;
                       color: white;
                       font-weight: 600;
                       padding: 12px 0;
                       width: 14.28%;
                       border-radius: 4px;
                   }}
                   td {{
                       text-align: center;
                       padding: 16px 0;
                       color: #333;
                       font-size: 16px;
                       font-weight: 500;
                   }}
                   .today {{
                       background-color: #e8f4fd;
                       color: #3498db;
                       border-radius: 50%;
                       font-weight: bold;
                   }}
                   .empty {{
                       color: #ccc;
                   }}
               </style>
           </head>
           <body>
               <div class="calendar-container">
                   <h2>{month_name} {year}</h2>
                   <table>
                       <thead>
                           <tr>
                               <th>Sun</th>
                               <th>Mon</th>
                               <th>Tue</th>
                               <th>Wed</th>
                               <th>Thu</th>
                               <th>Fri</th>
                               <th>Sat</th>
                           </tr>
                       </thead>
                       <tbody>
           """

           # 4. Construct table rows from calendar data
           for week in month_days:
               html += "<tr>"
               for day in week:
                   if day == 0:
                       html += '<td class="empty">&bull;</td>'
                   elif day == now.day:
                       html += f'<td class="today">{day}</td>'
                   else:
                       html += f"<td>{day}</td>"
               html += "</tr>"

           html += """
                       </tbody>
                   </table>
               </div>
           </body>
           </html>
           """

           # 5. Send HTTP Response headers and content
           self.send_response(200)
           self.send_header("Content-type", "text/html")
           self.end_headers()
           self.wfile.write(html.encode("utf-8"))


   def run():
       server_address = ("", PORT)
       httpd = HTTPServer(server_address, CalendarRequestHandler)
       print(f"Server running at http://localhost:{PORT}/")
       try:
           httpd.serve_forever()
       except KeyboardInterrupt:
           print("\nServer stopped.")


   if __name__ == "__main__":
       run()
   ```

4. Jalankan aplikasi:

   ```sh
   python3 /root/aplikasi/program.py
   ```

5. Verifikasi awal dari Ognisko:

   ```sh
   curl http://[IP Prefix].200.21:5000
   ```

   Hasil yang diharapkan: HTML kalender bulan berjalan.

**Langkah Kerja, Reverse Proxy (Kazdel):**

1. Tambahkan satu blok `location` baru pada `nginx.conf`:

   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;

           location = /web1 {
               proxy_pass http://[IP Prefix].200.20:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }

           location = /web2 {
               proxy_pass http://[IP Prefix].200.21:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }

           location = /app {
               proxy_pass http://[IP Prefix].200.21:5000/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
       }
   }
   ```

2. Restart node Kazdel.

**Indikator Keberhasilan:**

1. Dari Sargon:

   ```sh
   curl http://www.netics.my.id/web1
   curl http://www.netics.my.id/web2
   curl http://www.netics.my.id/app
   ```

   `/app` menghasilkan tampilan kalender dari aplikasi Python.
2. Dari Ognisko, cek ketiga path lewat browser.

---

## e. Verifikasi Menyeluruh

Setelah Problem C.1 dan C.2 selesai, bagian ini menguji seluruh komponen (DNS, HTTP server, reverse proxy) sebagai satu kesatuan, dari berbagai titik di jaringan.

| No. | Item Verifikasi | Perintah / Cara | Dari Node |
|-----|-----------------|-----------------|-----------|
| 1 | Client otomatis memiliki alamat DNS server | `cat /etc/resolv.conf` | Sargon |
| 2 | Resolusi nama domain berhasil | `ping www.netics.my.id`, `ping web.netics.org` | Sargon, `netics-pc` lain |
| 3 | Akses langsung ke tiap backend (tanpa proxy) masih berfungsi | `curl http://[IP Prefix].200.20:8080`, `curl http://[IP Prefix].200.21:8080`, `curl http://[IP Prefix].200.21:5000` | Sargon |
| 4 | Reverse proxy meneruskan ke backend yang benar | `curl http://www.netics.my.id/web1`, `/web2`, `/app` | Sargon, Iberia |
| 5 | Multi-domain web server tetap berfungsi | `curl http://www.netics.my.id`, `curl http://web.netics.org` | Sargon |
| 6 | Verifikasi visual lewat browser | Firefox ke seluruh alamat pada baris 2–5 | Ognisko |
| 7 | Header diteruskan dengan benar (Host, X-Real-IP) | `curl -vL` ke tiap path, amati header request/response | Sargon |

---

## f. Tugas Mandiri Bagian C

### Pemetaan Node (dengan Web3)

| Node | Appliance | Peran | IP Address |
|------|-----------|-------|------------|
| Router | `netics-pc` | Menyediakan akses internet dari NAT | eth0: Auto (DHCP)<br>eth1: `[IP Prefix].200.1/24` (statis) |
| Sargon, Iberia, Higashi | `netics-pc` | Client uji command line (curl) | Auto (DHCP) |
| Ognisko | `netics-pc-desktop` | Client uji browser (VNC + Chromium) | Auto (DHCP) |
| Aegir | `netics-pc` | DHCP Server | `[IP Prefix].200.10/24` (statis)<br>Gateway: `[IP Prefix].200.1`<br>Pool DHCP: `[IP Prefix].200.100–150`<br>Tambahkan di konfigurasi dhcpd nya: `option router [IP Prefix].200.1` |
| Kazdel | `netics-pc` atau `netics-server` | Web Server (Bagian A) -> Reverse Proxy (Bagian C) | `[IP Prefix].200.11/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| Chernobog | `netics-pc` atau `netics-server` | DNS Server (BIND) | `[IP Prefix].200.12/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| web1 | `netics-pc` atau `netics-server` | Web1, backend HTTP server | `[IP Prefix].200.20/24` (statis), port 8080 (statis)<br>Gateway: `[IP Prefix].200.1` |
| web2 | `netics-pc` atau `netics-server` | Web2, backend HTTP server + aplikasi Python (Flask) | `[IP Prefix].200.21/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |
| web3 | `netics-pc` atau `netics-server` | Web3, backend HTTP server | `[IP Prefix].200.22/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |

### Soal

1. Konfigurasi mengikuti bagian sebelumnya (Bagian A hingga E).
   **Tambahan:** tunjukkan juga untuk soal 1 topologi yang kamu buat di GNS3.
2. Tambahkan Web3 (backend HTTP baru) dan integrasikan ke reverse proxy lewat path `/web3`.
3. Daftarkan nama domain baru di DNS server (misalnya `app.netics.my.id`) yang mengarah ke reverse proxy, lalu ubah konfigurasi reverse proxy agar juga bisa diakses lewat nama domain baru ini (*name-based virtual host*), selain `www.netics.my.id`.
4. Buktikan dengan `curl` dan capture (`termshark`) pada Kazdel (gunakan `tmux` untuk dapat menjalankan nginx dan termshark bersamaan) bahwa `X-Real-IP` pada request ke Web3 sesuai dengan IP address client yang sesungguhnya mengakses.
