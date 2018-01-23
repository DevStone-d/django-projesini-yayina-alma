# Django Projesini Yayına Alma

[YouTube kanalımdaki](https://www.youtube.com/channel/UC5bnqsX71d-eKCHhNvM2U6g) **[Django Uygulamasını Yayına Alma](https://www.youtube.com/watch?v=uwVmWS1yJ1k&list=PLPrHLaayVkhny4WRNp05C1qRl1Aq3Wswh)** adlı video ders serisi için dökümantasyon. 

İçindekiler

- [Linux Kullanıcı Oluşturma](#linux-kullanıcı-oluşturma)
- [Python3, PostgreSQL ve Nginx bileşenlerinin yüklenmesi](python3-postgresql-ve-nginx-bileşenlerinin-yüklenmesi)
- [PostgreSQL Yapılandırması](#postgresql-yapılandırması)
- [Proje için Python Sanal Ortamın (Virtual Environment) ve Bağımlılıkların Yüklenmesi](#proje-için-python-sanal-ortamın-virtual-environment-ve-bağımlılıkların-yüklenmesi)
- [Django Uygulamasıyla ilgili Yayın Ortamı (Production Environment) Ayarları](#django-uygulamasıyla-ilgili-yayın-ortamı-production-environment-ayarları)
- [Projeyi Django'nun Geliştirme Sunucusuyla Test Etme](#projeyi-djangonun-geliştirme-sunucusuyla-test-etme)
- [Gunicorn](#gunicorn)
- [Nginx Yapılandırması](#nginx-yapılandırması)

Kullanılan teknolojiler : **Ubuntu 16.04 üzerinde Nginx, Gunicorn, PostgreSQL**

Dersler YouTube'da ücretsiz olarak yayımlanmaktadır.

## Linux Kullanıcı Oluşturma

1. Kullanıcı Oluşturma

```
   adduser kullanici_adi
```
   
2. Oluşturulan yeni kullanıcıya sudo (yönetici) yetkisi verme

```
   usermod -aG sudo kullanici_adi
```

### Python3, PostgreSQL ve Nginx bileşenlerinin yüklenmesi

```
  sudo apt-get update
  sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx
```

### PostgreSQL Yapılandırması

1. İnteraktif PostgreSQL komut istemcisine bağlanma

```
  sudo -u postgres psql
```

2. Projenin kullanacağı veritabanını oluşturma

```sql
  CREATE DATABASE proje_adi;
```

3. Projenin kullanacağı veritabanı kullanıcısını oluşturma

```sql
  CREATE USER kullanici_adi WITH PASSWORD 'parola123';
```

4. Varsayılan karakter kodlaması (default character encoding), işlem (transaction) ve bölgesel zaman ile ilgili ayarların yapılması

```sql
  ALTER ROLE kullanici_adi SET client_encoding TO 'utf8';
  ALTER ROLE kullanici_adi SET default_transaction_isolation TO 'read committed';
  ALTER ROLE kullanici_adi SET timezone TO 'Europe/Istanbul';
```

5. Oluşturulan kullanıcıyı veritabanına yönetici erişimi izni verme

```sql
  GRANT ALL PRIVILEGES ON DATABASE proje_adi TO kullanici_adi;
```

6. PostgreSQL komut istemcisinden çıkmak

```sql
  \q
```

### [Proje için Python Sanal Ortamın (Virtual Environment) ve Bağımlılıkların Yüklenmesi](#virtualenv)

1. Python3 için pip güncellemesi ve `virtualenv` paketinin kurulması

```
  sudo -H pip3 install --upgrade pip
  sudo -H pip3 install virtualenv
```

**Not:** *"locale.Error: unsupported locale setting"* hatası için çözüm:

```
  export LC_ALL="en_US.UTF-8"
```

2. Örnek bir Django projesi GitHub'dan kopyalanıyor.

```
  git clone https://github.com/barissaslan/eventhub.git
```

3. Proje dizinine geçme ve `venv` adlı Sanal Python Ortamının oluşturulması

```
  cd eventhub
  virtualenv venv
```

4. Sanal Ortamın aktif edilmesi

```
  source venv/bin/activate
```

5. Proje bağımlılıklarının yüklenmesi

```
  pip install -r requirements.txt
```

6. Gunicorn ve PostgreSQL adaptörünün yüklenmesi

```
  pip install gunicorn psycopg2
```

### [Django Uygulamasıyla ilgili Yayın Ortamı (Production Environment) Ayarları](#production)

#### `setting.py` Dosyasındaki Ayarlar

1. Django projesinin `setting.py` dosyasındaki DEBUG ifadesinin `False` olarak ayarlanması.

```python
  DEBUG = False # Production ortamında DEBUG kesinlikle kapatılmalıdır ki saldırganlar site hakkında detaylı hata raporlarına ulaşamasınlar. 
```

2. Django projesinin `setting.py` dosyasındaki ALLOWED_HOSTS (İzin verilmiş sunucular) ataması

```python
  ALLOWED_HOSTS = ['alan_adiniz_veya_IP_adresiniz', 'ikinci_alan_adiniz_veya_IP_adresiniz', . . .]
  # Örnek:
  ALLOWED_HOSTS = ['alanadi.com', '192.168.1.10'] # IP adresi örnek amaçlı yazılmış olup private adrestir.
```

3. Django projesinin veritabanı ayarlarını SQLite'dan PostgreSQL'e çevirme.

    **Not:** Aşağıdaki `'NAME'`, `'USER'`, `'PASSWORD'` alanları değişken olup, atanan değerler yukarıda ki PostgreSQL Yapılandırması'nda verilen takma adlardan alınmıştır.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'proje_adi',
        'USER': 'kullanici_adi',
        'PASSWORD': 'parola123',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```

4. STATICFILES_DIR Değişkeninin Kaldırılması ve STATIS_ROOT değişkeninin düzenlenmesi

    **Not:** Yayın Ortamında static dosyalar STATIC_ROOT değişkeninin gösterdiği klasör referans alınarak servis edileceği için STATICFILES_DIR'ın bir anlamı olmadığından yorum satırı haline getiriyoruz veya siliyoruz. Projemizde STATIC_ROOT değişkeni "static" klasörünü göstermelidir.
    
```python
    # STATICFILES_DIRS = [
    #    os.path.join(BASE_DIR, "static"),
    # ]
    
    STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

5. Veritabanı şemasını oluşturma ve statik dosyaları toplama

```
   python manage.py makemigrations
   python manage.py migrate
   python manage.py createsuperuser
   python manage.py collectstatic
```

### [Projeyi Django'nun Geliştirme Sunucusuyla Test Etme](#dev-server)

1. Test edilecek 8000 portunu aktif etme

```
   sudo ufw allow 8000
```

2. Geliştirme sunucusunu çalıştırma

```
   # 0.0.0.0 -> sunucunun IP adresi. (sunucu kendi IP adresini bildiğinden 0.0.0.0 yazılabilir)
   python manage.py runserver 0.0.0.0:8000
```

3. Tarayıcıdan test etme

```
   http://alan_adi_veya_IP:8000
```


### [Gunicorn](#gunicorn)

1. Projeyi Gunicorn ile Çalıştırarak Gunicorn'u Test Etmek 

```
   # Sanal ortam aktif olmalı ve manage.py dizininde çalıştırılfmalı. eventhub.wsgi -> proje klasörünün adı.
   gunicorn --bind 0.0.0.0:8000 eventhub.wsgi
```

Tarayıcıdan test etme: http://alan_adi_veya_IP:8000

Gunicorn sunucusunu durduma: `CTRL + C`

2. Gunicorn `systemd` Servis Dosyası Oluşturma

```
   # Sanal ortamdan çıkılmalı
   deactivate
   
   # Servis dosyası oluşturma
   sudo nano /etc/systemd/system/gunicorn.service
```

gunicorn.service dosya içeriği şu şekilde olmalıdır:

```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=baris
Group=www-data
WorkingDirectory=/home/baris/eventhub
ExecStart=/home/baris/eventhub/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/baris/eventhub/eventhub.sock eventhub.wsgi:application

[Install]
WantedBy=multi-user.target
```

**Not:** Sanal ortamın adı `venv`, projenin adı `eventhub`, kullanıcı adı `baris` olarak varsayılmıştır.
**Not 2:** Dosyayı kaydetmek için `CTRL + X` yaptıktan sonra `y` harfine basıp, `enter` tuşuna basılmalı.

3. Gunicorn servisini aktif etme:

```
   sudo systemctl start gunicorn
   sudo systemctl enable gunicorn
```

4. Gunicorn servisinin doğru bir şekilde çalışıp çalışmadığını test etme:

```
   sudo systemctl status gunicorn
   # Yeşil renkte "active (running)" yazıyorsa başarılı bir şekilde çalışıyor demektir.
```

```
   ls /home/baris/eventhub
   # ls çıktısında "proje_adı.sock" adlı (.sock) uzantılı bir soket dosyası görülüyorsa gunicorn başarılı bir şekilde yapılandırılmıştır.
```

### [Nginx Yapılandırması](#nginx)

1. Nginx sunucu bloğu açma:

```
   sudo nano /etc/nginx/sites-available/eventhub
```

   **Dosyanın içinde bulunması gerekenler:**

```
   server {
       listen 80;
       server_name alan_adi_veya_IP;
       root /home/baris/eventhub;

       location /static/ {
       }

       location /media/ {
       }

       location / {
          include proxy_params;
          proxy_pass http://unix:/home/baris/eventhub/eventhub.sock;
       }
   }
```

**Not:** Dosyayı kaydetmek için `CTRL + X` yaptıktan sonra `y` harfine basıp, `enter` tuşuna basılmalı.

2. Nginx dosyasını aktif etmek için dosyayı **`sites-enabled`** dizinine link olarak verme

```
   sudo ln -s /etc/nginx/sites-available/eventhub /etc/nginx/sites-enabled
```

3. Nginx yapılandırma dosyalarında *syntax* hatası olup olmadığını kontrol etme

```
   sudo nginx -t
```

4. Hata yoksa Nginx sunucusunu yeniden başlatma

```
   sudo systemctl restart nginx
```

5. Kullanılmayacak olan 8000 portunu kapatma ve Nginx kurallarını aktif etme

```
   sudo ufw delete allow 8000
   sudo ufw allow 'Nginx Full
```









