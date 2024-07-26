
# Panduan Instalasi Nextcloud dan Konfigurasi dengan Nginx di Port 6000 pada Ubuntu Server

## Prasyarat
1. Server Ubuntu yang telah diperbarui.
2. Hak akses root atau sudo.
3. Domain atau subdomain yang dapat diakses.

## Langkah 1: Perbarui Sistem
Pertama, perbarui paket-paket yang ada di sistem Anda:

```bash
sudo apt update && sudo apt upgrade -y
```

## Langkah 2: Instal Dependensi yang Diperlukan
Instal Nginx, MariaDB, PHP, dan modul PHP yang diperlukan:

```bash
sudo apt install nginx mariadb-server php-fpm php-mysql php-gd php-xml php-mbstring php-curl php-zip php-intl php-bcmath php-gmp php-imagick -y
```

## Langkah 3: Konfigurasi MariaDB
Amankan instalasi MariaDB dan buat database untuk Nextcloud:

```bash
sudo mysql_secure_installation
```

Masukkan perintah berikut di prompt MariaDB untuk membuat database dan user:

```sql
sudo mysql -u root -p
CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Langkah 4: Download dan Ekstrak Nextcloud
Unduh dan ekstrak Nextcloud ke direktori web root Nginx:

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-25.0.0.zip
unzip nextcloud-25.0.0.zip
sudo mv nextcloud /var/www/
```

## Langkah 5: Setel Izin Direktori
Setel izin untuk direktori Nextcloud:

```bash
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

## Langkah 6: Konfigurasi PHP-FPM
Edit file konfigurasi PHP-FPM untuk mengatur pengguna dan grup menjadi `www-data`:

```bash
sudo nano /etc/php/7.4/fpm/pool.d/www.conf
```

Ubah baris berikut:

```conf
user = www-data
group = www-data
listen.owner = www-data
listen.group = www-data
```

Restart PHP-FPM untuk menerapkan perubahan:

```bash
sudo systemctl restart php7.4-fpm
```

## Langkah 7: Konfigurasi Nginx
Buat file konfigurasi Nginx untuk Nextcloud:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

Tambahkan konfigurasi berikut:

```nginx
server {
    listen 6000;
    server_name your_domain_or_IP;

    root /var/www/nextcloud;
    client_max_body_size 10G;
    fastcgi_buffers 64 4K;

    gzip off;

    index index.php index.html /index.php$request_uri;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }

    location = /.well-known/caldav {
        return 301 $scheme://$host/remote.php/dav;
    }

    location ~ ^/.well-known/acme-challenge/ {
        allow all;
    }

    location / {
        rewrite ^ /index.php$request_uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
        deny all;
    }

    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
        deny all;
    }

    location ~ \.(?:flv|mp4|mov|m4a)$ {
        mp4;
        mp4_buffer_size 100m;
        mp4_max_buffer_size 1024m;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS on;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }

    location ~ \.(?:css|js|woff|svg|gif)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public";
        access_log off;
    }

    location ~* \.(?:jpg|jpeg|gif|bmp|ico|png|swf)$ {
        access_log off;
    }
}
```

Aktifkan konfigurasi situs dan restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## Langkah 8: Selesaikan Instalasi Nextcloud
Buka browser Anda dan akses Nextcloud menggunakan `http://your_domain_or_IP:6000`. Ikuti panduan instalasi di browser untuk menyelesaikan konfigurasi Nextcloud.

Anda akan diminta untuk:
1. Membuat akun admin.
2. Memasukkan detail database yang telah Anda buat sebelumnya (`nextcloud`, `nextclouduser`, dan `your_password`).
3. Selesai dan login ke Nextcloud.

## Langkah 9: Konfigurasi SSL dengan Let's Encrypt
Untuk mengamankan koneksi Nextcloud dengan HTTPS, Anda dapat menggunakan Let's Encrypt. Instal Certbot dan dapatkan sertifikat SSL:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot certonly --nginx -d your_domain
```

Edit file konfigurasi Nginx untuk menambahkan konfigurasi HTTPS:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

Tambahkan blok server berikut di atas blok `server` yang ada:

```nginx
server {
    listen 443 ssl;
    server_name your_domain_or_IP;

    ssl_certificate /etc/letsencrypt/live/your_domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain/privkey.pem;

    root /var/www/nextcloud;
    client_max_body_size 10G;
    fastcgi_buffers 64 4K;

    gzip off;

    index index.php index.html /index.php$request_uri;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }

    location = /.well-known/caldav {
        return 301 $scheme://$host/remote.php/dav;
    }

    location ~ ^/.well-known/acme-challenge/ {
        allow all;
    }

    location / {
        rewrite ^ /index.php$request_uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
        deny all;
    }

    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
        deny all;
    }

    location ~ \.(?:flv|mp4|mov|m4a)$ {
        mp4;
        mp4_buffer_size 100m;
        mp4_max_buffer_size 1024m;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS on;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }

    location ~ \.(?:css|js|woff|svg|gif)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public";
        access_log off;
    }

    location ~* \.(?:jpg|jpeg|gif|bmp|ico|png|swf)$ {
        access_log off;
    }
}
```

Restart Nginx untuk menerapkan perubahan:

```bash
sudo systemctl restart nginx
```

Sekarang, Nextcloud harus berjalan di port 6000 pada server Ubuntu Anda. Anda dapat mengaksesnya melalui `http://your_domain_or_IP:6000` untuk HTTP dan `https://your_domain` untuk HTTPS.
