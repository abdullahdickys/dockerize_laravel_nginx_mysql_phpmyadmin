di sini kita kan membuat docker yang berjalan dengan sebuah framework laravel.
Install laravel bisa dengan clone dari repo github atau lewat composer.saat blog ini dibuat saya menggunakan laravel versi 7
composer create-project --prefer-dist laravel/laravel laravel_docker_project
cd laravel_docker_project
mkdir -p nginx/conf.d
mkdir php
setelah kita buat folder dengan perintah di atas kita lanjut untuk membuat beberapa file docker diantaranya 1. docker-compose.yml, 2. Dockerfile, 3. config nginx app.conf, 4. config php.ini
oh ya untuk yang baru belajar docker dan belum mengerti dasar, mungkin link ini membantu sama seperti saya sebelum mengerti dasar docker disini.
docker-compose.yml
touch docker-compose.yml
isi file docker-compose.yml seperti di bawah
NB: jangan copypaste, usahakan ketik.
version: '3'
services:
#PHP Service
app:
build:
context: .
dockerfile: Dockerfile
image: php_service
container_name: app
restart: unless-stopped
tty: true
environment:
SERVICE_NAME: app
SERVICE_TAGS: dev
working_dir: /var/www
volumes:
- ./:/var/www
- ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
networks:
- app-network
#Nginx Service
webserver:
image: nginx:alpine
container_name: webserver
restart: unless-stopped
tty: true
ports:
- "88:80"
- "443:443"
volumes:
- ./:/var/www
- ./nginx/conf.d/:/etc/nginx/conf.d/
networks:
- app-network
#MySQL Service
db:
image: mysql
container_name: db
restart: unless-stopped
tty: true
ports:
- "33061:3306"
environment:
MYSQL_DATABASE: laravel
MYSQL_USER: amrilsyaifa
MYSQL_PASSWORD: qwerty1234
MYSQL_ROOT_PASSWORD: qwerty1234
SERVICE_TAGS: dev
SERVICE_NAME: mysql
networks:
- app-network
#PHPMyAdmin Service
phpmyadmin:
container_name: phpmyadmin
image: phpmyadmin/phpmyadmin
ports:
- "7000:80"
links:
- db:db
environment:
MYSQL_USER: amrilsyaifa
MYSQL_PASSWORD: qwerty1234
MYSQL_ROOT_PASSWORD: qwerty1234
UPLOAD_LIMIT: 3000000000
networks:
- app-network
#Docker Networks
networks:
app-network:
driver: bridge
#Volumes
volumes:
lbdata:
driver: local
2. Dockerfile
touch Dockerfile
isi file Dockerfile seperti di bawah ini
FROM php:7.2-fpm
# Copy composer.lock and composer.json
COPY composer.lock composer.json /var/www/
# Set working directory
WORKDIR /var/www
# Install dependencies
RUN apt-get update && apt-get install -y \
build-essential \
libmcrypt-dev \
mariadb-client \
libpng-dev \
libjpeg62-turbo-dev \
libfreetype6-dev \
locales \
zip \
jpegoptim optipng pngquant gifsicle \
vim \
unzip \
git \
curl
# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*
# Install extensions
RUN docker-php-ext-install pdo_mysql mbstring zip exif pcntl
RUN docker-php-ext-configure gd --with-gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-png-dir=/usr/include/
RUN docker-php-ext-install gd
# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www
# Copy existing application directory contents
COPY . /var/www
# Copy existing application directory permissions
COPY --chown=www:www . /var/www
# Change current user to www
USER www
# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
3. app.conf
touch nginx/conf.d/app.conf
isi app.conf seperti di bawah ini
server {
listen 80;
index index.php index.html;
error_log  /var/log/nginx/error.log;
access_log /var/log/nginx/access.log;
root /var/www/public;
location ~ \.php$ {
try_files $uri =404;
fastcgi_split_path_info ^(.+\.php)(/.+)$;
fastcgi_pass app:9000;
fastcgi_index index.php;
include fastcgi_params;
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
fastcgi_param PATH_INFO $fastcgi_path_info;
}
location / {
try_files $uri $uri/ /index.php?$query_string;
gzip_static on;
}
}
4. php.
touch php/local.ini
isi file local.ini seperti di bawah
upload_max_filesize=40M
post_max_size=40M
max_execution_time = 180
memory_limit = 3000M
Jika sudah kita lanjut untuk build container dengan mengetik perintah di bawah
docker-compose up -d
jika image di dalam docker belum ada maka dia akan download di docker hub. pastikan tidak ada error dan ada keterangan done seperti gambar di bawah.
Image for post
ketik docker ps untuk melihat container yang sedang berjalan
Image for post
sekarang buka http://localhost:88/
booommmmmm!!!!!
Image for post
berhasil,
lanjut buka phpmyadmin http://localhost:7000/
masukkan password, di sini saya menggunakan password yang di define di docker-compose.yml.
sudah ada database dengan nama laravel sesuai yang kita define di atas.
lanjut untuk menghubungkan database kita edit .env laravel
docker-compose exec app vim .env
ubah seperti di bawah
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=amrilsyaifa
DB_PASSWORD=qwerty1234
lakukan clear config dan cache dengan perintah di bawah
docker-compose exec app php artisan config:clear
docker-compose exec app php artisan cache:clear
sebelum melakukan migrate ubah dahulu AppServiceProvider.php
use Illuminate\Support\Facades\Schema;
public function boot()
{
Schema::defaultStringLength(191);
}
lanjut migrate
docker-compose exec app php artisan migrate
boom….!!!

