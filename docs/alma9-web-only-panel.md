# Vesta AlmaLinux 9 Web-Only Panel (Hestia Model)

Bu doküman, Vesta arayüzünü koruyarak AlmaLinux 9 / Rocky 9 / RHEL 9 üzerinde modernleştirilmiş, **web-only** bir panel çekirdeğine geçiş hedefini özetler.

## Amaç

* 2025 sonrası kullanılabilir, hafif ve güvenli bir Vesta türevi oluşturmak.
* Orijinal Vesta UI'ını koruyup yalnızca web barındırmaya odaklanmak.
* HestiaCP'nin modern yaklaşımını (SSL, firewall, güvenlik) AlmaLinux 9 ekosistemine uyarlamak.

## Desteklenen Servisler

* `nginx`
* `php-fpm`
* `mariadb`
* `firewalld`

> Apache, exim, dovecot, bind, vsftpd gibi servisler **kurulmayacak**.

## Aktif Modüller

* Web (domain, vhost, PHP)
* SSL (ACME v2, Let's Encrypt)
* Log (domain ve panel logları)
* Backup (domain bazlı tar.gz)
* Firewall (Vesta/Hestia tarzı UI → firewalld backend)

## Devre Dışı Modüller

* DNS
* Mail
* FTP

İlgili eski scriptler `bin-disabled/` ve `func-disabled/` dizinlerine taşınarak devre dışı bırakılacaktır.

## Mimari Katmanlar

1. **OS & Servis Katmanı:** AlmaLinux 9 / Rocky 9 / RHEL 9 üzerinde gerekli servislerin kurulumu ve yönetimi.
2. **Panel Çekirdeği:** `/usr/local/vesta/` altında `bin/`, `func/`, `web/`, `conf/`, `data/`, `log/`, `backup/`, `tmp/` dizinleri.
3. **Web Sunucu Katmanı:** `nginx + php-fpm` şablonları, domain başına kullanıcı modeli.
4. **Panel UI Katmanı:** Vesta UI'nın korunması, DNS/Mail/FTP menülerinin gizlenmesi, firewalld backend'ine bağlanan yeni firewall menüsü.

## Kullanıcı ve Dizın Modeli

Her domain kendi Linux kullanıcısı ile izole edilir. Örnek dizin yapısı:

```
/home/USER/web/DOMAIN/public_html
/home/USER/web/DOMAIN/logs
/home/USER/web/DOMAIN/ssl
```

Bu model güvenlik, yedekleme ve PHP-FPM havuz yönetimini kolaylaştırır.

## Panel Vhost Örneği (nginx)

Panel varsayılan olarak 8083 portunu dinler ve `/usr/local/vesta/web` kökünü kullanır.

```
server {
    listen 8083;
    server_name _;

    root /usr/local/vesta/web;
    index index.php index.html;

    access_log /var/log/nginx/vesta-access.log;
    error_log  /var/log/nginx/vesta-error.log;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
    }

    location ~* \.(jpg|jpeg|gif|png|css|js|ico)$ {
        expires max;
        log_not_found off;
    }
}
```

## Firewall Backend (firewalld)

* Yeni backend scripti: `/usr/local/vesta/func/firewalld-backend.sh`.
* Panel UI'daki firewall menüsü firewalld ile konuşur.
* Örnek komutlar:

```
v-add-firewall-rule PROTO PORT [IP] ACTION
v-list-firewall
v-delete-firewall-rule ...

# Port açma
firewall-cmd --permanent --add-port=8083/tcp

# IP bazlı düşürme
firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=1.2.3.4 drop"
firewall-cmd --reload
```

## SSL / Let's Encrypt

* ACME v2 desteği ve HTTP-01 varsayılan doğrulama.
* Sertifika dosyaları: `/home/USER/web/DOMAIN/ssl/` altında `cert.pem`, `key.pem`, `ca.pem`.
* Planlanan CLI komutları: `v-add-letsencrypt-domain`, `v-update-ssl`.

## Backup

* Domain bazlı `tar.gz` yedekler.
* İçerik: `public_html`, `logs` (opsiyonel), `ssl` (opsiyonel), domain metadata.
* Komutlar: `v-backup-domain`, `v-restore-domain`.

## Kurulum Akışı (Özet)

1. OS doğrulaması (Alma 9 / Rocky 9 / RHEL 9).
2. Servis kurulumu: nginx, php-fpm, mariadb, firewalld.
3. `/usr/local/vesta` dizin yapısının hazırlanması.
4. Mail/DNS/FTP modüllerinin devre dışı bırakılması.
5. Firewall'un firewalld backend ile güncellenmesi.
6. Panel vhost'unun nginx'e eklenmesi (8083).
7. Firewalld'da SSH/HTTP/HTTPS/Panel portlarının açılması.
8. Admin kullanıcısının oluşturulması ve ilk login testleri.

## Yol Haritası

* AlmaLinux 9 installer tamamlanması.
* Vesta UI'da DNS/Mail/FTP menülerinin temizlenmesi.
* Firewalld backend için rule-id bazlı delete/list desteği.
* Hestia'dan ACME v2 SSL motorunun port edilmesi.
* Domain bazlı backup/restore komutlarının eklenmesi.
* PHP-FPM pool'larının kullanıcı bazlı yönetimi.
* Gelişmiş log görüntüleme (UI filtresi).

---

Bu doküman, Vesta'nın sade arayüzünü modern servislerle birleştirmek için hedeflenen çekirdek mimariyi özetler. Katkılar (issue/PR) özellikle AlmaLinux 9 testleri, nginx/PHP-FPM tuning, firewalld ve ACME geliştirmeleri için teşvik edilir.
