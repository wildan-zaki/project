<div align="center"><a href="https://rocket.chat/"><img src="https://www.stickermule.com/marketplace/embed_img/10009" style="max-width:100%;"></a></div>

[Sekilas Tentang](#sekilas-tentang) | [Instalasi](#instalasi) | [Konfigurasi](#konfigurasi) | [Otomatisasi](#otomatisasi) | [Cara Pemakaian](#cara-pemakaian) | [Pembahasan](#pembahasan) | [Referensi](#referensi)
:---:|:---:|:---:|:---:|:---:|:---:|:---:

# Sekilas Tentang

<div >Rocket.Chat adalah solusi obrolan chat berbasis open source. aplikasi ini dapat berjalan dengan desktop os, android, ios  atau dengan free trial cloud demo di web.

Rocket.chat merupakan sebuah perusahaan yang menciptakan aplikasi berbasis web dengan nama persis seperti nama perusahaannya yaitu Rocket.chat, yang berdasarkan nilai  open source dan kecintaan akan efisiensi. Perusahaan tersebut didukung oleh beberapa komunitas kontributor dari berbagai belahan dunia yang luar biasa, dengan anggota tim yang berbakat, yang bekerja tanpa lelah untuk menjaga agar kode tetap menjadi standar yang tinggi dan membuat hidup kontributor menjadi lebih mudah, yang dipimpin oleh  Gabriel Engel sebagai CEO nya.

Rocket.chat telah dinobatkan sebagai top project di Rookies Open Source Black Duck of the Year pada tahun 2016. Perusahaan tersebut berbasis di Brazil, namun sebagian besar tim bekerja dari jarak jauh. Pesaing utama dari Rocket.Chat adalah aplikasi seperti Slack dan HipChat.


# Instalasi
Panduan ini dibangun dengan asumsi berikut:
- Sistem operasi: Ubuntu Server 16.04
- Layanan ssh berjalan.
- Layanan Rocket.Chat akan dijalankan pada port 3000

## Prasyarat 
Ref: [[1]](#1)
### VPS (minimal)

- Single core (2 GHz)
- 1 GB RAM
- 30 GB SSD

Ideal untuk penggunaan hingga 200 pengguna dengan 50 akses secara bersamaan. Aktivitas _upload_, _sharing_, dan _bot_ pada level minimum. 

### VPS (recommended)

- Dual core (2 GHz)
- 2 GB RAM
- 40 GB SSD

Dapat mengakomodir penggunaan hingga 500 pengguna dengan 100 akses secara bersamaan. Aktivitas _upload_, _sharing_, dan _bot_ pada level wajar.

## Instalasi Cepat dengan Snaps (Disarankan)
```bash
  sudo snap install rocketchat-server
```

Setelah instalasi selesai, aplikasi langsung aktif pada port 3000 http dan dapat di akses melalui _url_ [http://localhost:3000](http://localhost:3000)

## Instalasi secara Manual

Dependensi:
- Node.js (versi 4.5.0 atau lebih)
- MongoDB
- curl
- graphicsmagick

```bash
# Instal dependensi
sudo apt update
sudo apt install mongodb-server nodejs npm build-essential curl graphicsmagick
sudo npm install -g n
sudo n 4.8.4
# Pastikan mongodb berjalan
sudo service mongodb start

# Download Rocket.Chat
curl -L https://download.rocket.chat/stable -o rocket.chat.tgz

# Deploy Rocket.Chat
tar zxvf rocket.chat.tgz
mv bundle Rocket.Chat && cd Rocket.Chat/programs/server
sudo npm install

# Set Envirenment Variables
cd ../..
export ROOT_URL=http://localhost:3000/
export MONGO_URL=mongodb://localhost:27017/rocketchat
export PORT=3000
export ADMIN_USERNAME=adminusername
export ADMIN_PASS=adminpassword
export ADMIN_EMAIL=adminemail@example.com
node main.js
```

Jika proses _startup_ berhasil, cli menampilkan informasi seperti berikut:
```bash
➔ System ➔ startup
➔ +------------------------------------------+
➔ |              SERVER RUNNING              |
➔ +------------------------------------------+
➔ |                                          |
➔ |  Rocket.Chat Version: 0.59.1             |
➔ |       NodeJS Version: 4.8.4 - x64        |
➔ |             Platform: linux              |
➔ |         Process Port: 3000               |
➔ |             Site URL: http://localhost/  |
➔ |     ReplicaSet OpLog: Disabled           |
➔ |          Commit Hash: 0710c94d6a         |
➔ |        Commit Branch: HEAD               |
➔ |                                          |
➔ +------------------------------------------+
```
Hal ini mengindikasikan bahwa layanan Rocket.Chat sudah berjalan pada port 3000 http dan dapat diakses melalui _url_ [http://localhost:3000](http://localhost:3000)

> Diperlukan wewenang sudo untuk menjalankan layanan pada port _default_ http 80. Hal ini dapat dilakukan dengan perintah berikut:
> ```bash
> sudo ROOT_URL=http://localhost/ \
>      MONGO_URL=mongodb://localhost:27017/rocketchat \
>      PORT=80 \
>      node main.js
> ```

# Konfigurasi

## Penanganan SSL

Rocket.Chat sebagai aplikasi tingkat menengah tidak dapat menangani layanan SSL sendirian. Tugas tersebut dapat diserahkannya kepada web server lain seperti nginx. Metode ini disebut _reverse proxy_.

```bash
sudo apt install nginx
```

### Membuat Self-Signed Certificate

```bash
sudo mkdir /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/nginx/ssl/nginx.key \
     -out /etc/nginx/ssl/nginx.crt
```

### Konfigurasi Nginx untuk penanganan SSL

Sesuaikan file konfigurasi /etc/nginx/sites-enabled/default

```bash
# Upstreams
upstream backend {
    server 127.0.0.1:3000;
}

# http configuration
server {
        listen      80 default_server;
        server_name localhost;
        access_log  off;
        error_log off;
        ## redirect http to https ##
        return      301 https://$server_name$request_uri;
}

# https configuration
server {
        listen  443 ssl;

        root  /var/www/html;
        index   index.html index.htm;

        server_name localhost;
        ssl_certificate   /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;

        location / {
                # try_files $uri $uri/ =404;
                proxy_pass http://backend/;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header Host $http_host;

                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forward-Proto http;
                proxy_set_header X-Nginx-Proxy true;

                proxy_redirect off;
        }
}
```

> Dengan konfigurasi ini, server sudah memiliki dua layanan web server nginx pada port 443 https dan rocketchat pada port 3000. Konfigurasi dapat berfungsi apabila kedua layanan ini berjalan. Karenanya sebelum mengakses _url_ [https://localhost](https://localhost), terlebih dahulu jalankan layanan rocketchat. node ~/Rocket.Chat/main.js

## OAuth

OAuth merupakan metode autentikasi meggunakan akun aplikasi lain. Rocket.Chat mendukung beberapa OAuth seperti GitHub, Facebook, Google. Fitur ini dapat dikonfigurasi pada Menu **[Administration > OAuth](https://localhost:4444/admin/OAuth)**.

![Form OAuth - Rocket](etc/oauth-form.png)

**Client Id** dan **Client Secret** Github dapat diperoleh dari menu [**Settings > Developer settings > OAuth Apps**](https://github.com/settings/developers).

![OAuth - Github](etc/oauth-github.png)

Tombol login dengan akun OAuth akan muncul pada halaman login.

![Form Login OAuth - Rocket](etc/oauth-login.png)
<!--
## Layout

https://localhost:4444/admin/Layout
-->

## Livechat

Fitur Livechat memungkinkan layanan chat diakses dari halaman web.

![LiveChat - Rocket](etc/livechat-demo.png)

Fitur ini dapat diaktifkan dari menu [**Administration > Livechat**](https://localhost:4444/admin/Livechat). Setelah mengaktifkan fitur ini, menu **Livechat Manager** akan ditampilkan pada _side-menu_ untuk mengakomodir pengelolaan Livechat lanjutan di antaranya pengaturan role user pada LiveChat dan departemen.

![LiveChat - Rocket](etc/livechat-menu.png)

Pada menu [**Livechat Manager > Installaltion**](https://localhost:4444/livechat-manager/installation) diberikan script untuk diletakkan pada halaman web.

![LiveChat - Rocket](etc/livechat-installation.png)

<!--#  Maintenance

https://localhost:4444/admin/Logs
-->

# Otomatisasi

## Jalankan sebagai service

Ref: [[2]](#2)

Dengan konfigurasi ini, layanan RocketChat akan berjalan otomatis setiap kali _boot_.

```bash
sudo npm install -g forever forever-service
cd ~/Rocket.Chat
sudo forever-service install -s main.js -e  \
  "ROOT_URL=https://localhost/ \
  MONGO_URL=mongodb://localhost:27017/rocketchat PORT=3000" \
  rocketchat
sudo service rocketchat start
```

# Cara Pemakaian

- Tampilan aplikasi web
<div width="100%">
![Rocket](etc/gambar1.png)
</div>
<center>Gambar 1. fitur login dan daftar baru</center>

<div width="100%">
![Rocket](etc/gambar2.png)
</div>
<center>Gambar 2. daftarkan akun baru apabila belum punya akun rocket.chat</center>

<div width="100%">
![Rocket](etc/gambar3.png)
 </div>
<center>Gambar 3. masukkan nama pengguna yang akan ditampilkan sebagai username</center>
-->
<div width="100%">
![Rocket](etc/gambar4.ng)
<center>Gambar 4. tampilan halaman utama/beranda rocket.chat</center
 -->
 
- Fungsi-fungsi utama
1. Membuat grup & channel
<div width="100%">
![Rocket](etc/gambar5.png)
<center></center>

Membuat grup di Rocket.Chat dapat dilakukan dengan meng-klik tanda “+” yang berada disebelah kolom search. untuk tampilan lebih jelasnya dapat dilihat pada gambar 5 diatas.

Selanjutnya kita bisa memilih ingin membuat grup atau channel dengan cara meng-klik button private channel. jika kita ingin membuat grup (private channel) maka kita pilih button tersebut menjadi warna hijau, seperti pada gambar 6 diatas. Selanjutnya jika kita ingin membuat channel (public channel) maka kita klik button tersebut sehingga warna nya berubah menjadi warna merah, seperti gambar 7 dibawah ini.
-->
<div width="100%">
![Rocket](etc/gambar6.png)
 
 
<center>Gambar 6. Mode Private</center> 
<div width="100%">
 ![Rocket](etc/gambar7.png)
 
<center>Gambar 7. Mode Public</center>  

Selanjutnya kita dapat menambahkan user yang ingin kita ajak bergabung kedalam grup/channel yang telah kita buat tadi dengan cara masukkan username pengguna di invite pengguna.

<div width="100%">
![Rocket](etc/gambar8.png)
<center>Gambar 8. Mention user lain</center>  

<div width="100%">
![Rocket](etc/gambar9.png)

<center>Gambar 9. Mention all user</center>  

Selanjutnya setelah grup/channel berhasil dibuat, maka akan muncul tampilan seperti gambar 8 diatas. Kita juga dapat melakukan mention ke user lain dengan cara ketik (@username).

# Pembahasan
Rocket Chat merupakan Web Chat Server yang dikembangkan dalam javascript dengan menggunakan Meteor fullstack framework. Rocket Chat dapat menjadi solusi yang bagus
untuk komunitas atau perusahaan yang menginginkan chat service mereka dihosting secara private. Sebagai aplikasi chat open source alternatif dari slack, Rocket Chat memiliki kelebihan sebagai berikut:

Dapat dihosting pada private server, sehingga kemanan data chat service lebih terjamin dan dapat di kustomisasi sesuai kebutuhan.
Realtime translation. Rocket Chat menggunakan teknologi machine learning untuk melakukan automatic real time translation ketika user melakukan chat.
Multiplatform. Rocket Chat dapat digunakan di Android, IOS, web based dan desktop based.
Tersedianya banyak plugin, system integration, tema tampilan aplikasi dan powerful API.
Menyediakan banyak skema authentikasi, seperti LDAP Group Sync, two-factor authentication (2-FA), E2E encryption, SSO, dan mendukung banyak skema authentikasi OAuth.
Unlimited users, channels, searches, guests, messages danfiles.

Disamping memiliki banyak kelebihan, Rocket Chat memiliki beberapa kekurangan yang perlu dipertimbangkan antara lain:
Versi cloud dari Rocket Chat masih belum free
Karena versi gratis dari Rocket Chat harus dihosting secara private, pengguna harus memahami tata cara instalasi Rocket Chat dan tata cara maintenance-nya.

# Rocket Chat dan Slack

Aplikasi Chat Service lain yang banyak digunakan saat ini adalah Slack. Salah satu sebab Slack mejadi aplikasi chat favorit karena mudah diintegrasikan dengan banyak tools, akan tetapi Slack sangat terbatas terutama dalam versi gratisnya dan versi berbayarnya sangat mahal. Berikut ini beberapa perbandingan antara Slack dan Rocket Chat:

Aplikasi Rocket Chat dapat di jalankan pada private hosting dan tersedia pula versi cloud-nya walaupun tidak gratis, sedangkan slack tidak dapat dijalankan pada private hosting melaikan hanya tersedia versi cloud-nya saja.
Methode autentikasi yang disediakan Rocket Chat lebih banyak dibandingkan dengan Slack.
Slack lebih mudah di integrasikan dengan berbagai macam tools seperti github, trello dan lain sebagainya. Berbeda dengan Rocket Chat, pengguna perlu melakukan setup sendiri untuk mengintegrasikan Rocket Chat dengan aplikasi atau tools lain.
Slack memiliki fungsi search yang powerfull, pengguna dapat melakukan pencarian medalam terhadap file, potongan kode program (snippet), bahkan slack dapat melakukan pecarian data pada service yang diintegrasikan dengannya, sedangkan Rocket Chat hanya menyediakan fungsi search trivial seperti kebanyakan aplikasi chat lainnya.

# Referensi
1. <a id="1"></a>[https://docs.rocket.chat/](https://docs.rocket.chat/) Rocket.Chat Documentation - Rocket.Chat
2. <a id="2"></a>[https://www.digitalocean.com/community/tutorials/how-to-...](https://www.digitalocean.com/community/tutorials/how-to-install-configure-and-deploy-rocket-chat-on-ubuntu-14-04) How To Install, Configure, and Deploy Rocket.Chat on Ubuntu 14.04 - DigitalOcean
