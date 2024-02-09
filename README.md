# django-ubuntu-deploy

## Gerekli Araçların İndirilmesi

* İlk olarak Ubuntu makinemizi güncelleyerek ve yükselterek  işe başlıyoruz:

    ```
    sudo apt-get update
    ```

* Makineyi güncelledikten sonra bir sonraki adım ihtiyacımız olan öğeleri Ubuntu repolarından indirmek olacak:

    ```
    sudo apt install python3-venv python3-dev libpq-dev postgresql postgresql-contrib nginx curl redis uwsgi gcc gunicorn
    ```

## Veritabanı kurulumu

* Şimdi Django uygulamamız için bir veritabanı oluşturalım, aşağıdakileri yazarak etkileşimli bir Postgres oturumuna giriş yapabiliriz:

    ```
    sudo -u postgres psql
    ```

* Projemiz için bir veritabanı oluşturalım ve noktalı virgüllere de dikkat edelim:
  
    ```
    CREATE DATABASE sampledb;
    ```

* Daha sonra projemiz için bir veritabanı kullanıcısı oluşturalım, güvenli bir şifre kullandığımızdan emin olalım:

    ```
    CREATE USER sampleuser WITH PASSWORD 'samplePASSWORD';
    ```

    Django, PostgreSQL veritabanıyla iletişim kurarken sıkıntılar yaşamasın diye bazı ayarlamalar yapmamız gerekiyor.

* Varsayılan karakter kodlamasını django'nun da beklediği gibi `utf8` olarak ayarlıyoruz:
    ```
    ALTER ROLE sampleuser SET client_encoding TO 'utf8';
    ```

* Varsayılan işlem izolasyon düzenini `read committed` olarak ayarlıyoruz:
    ```
    ALTER ROLE sampleuser SET default_transaction_isolation TO 'read committed';
    ```
    Yukarıdaki komutu açıklarsak: Django'nun belirli bir veritabanı yapılandırmasını beklediğini ve aynı zamanda varsayılan işlem izolasyon düzenini "read committed" olarak ayarladığımızı ifade etmektedir. Bu, Django'nun veritabanı işlemleri sırasında belirli bir güvenlik ve tutarlılık seviyesi sağlamak üzere bu yapılandırmayı kullanacağı anlamına gelir.

* Saat dilimini bulunduğumuz bölgeye uygun ayarlıyoruz:
    ```
    ALTER ROLE sampleuser SET timezone TO 'Europe/Istanbul';
    ```

* Oluşturduğumuz kullanıcıya, oluşturduğumuz veritabanını kullanma yetkisi veriyoruz:
    ```
    GRANT ALL PRIVILEGES ON DATABASE sampledb TO sampleuser;
    ```

* İşimiz bittiği için PostgreSQL komut isteminden çıkıyoruz:
    ```
    \q
    ```

    Postgresql sunucumuza dışarıdan `pgadmin` kullanarak erişmek istersek, postgresql servisimizin ayarlarında bazı değişiklikler yapmamız gerekir.

* Bunun için ilk olarak aşağıdaki komutu yazıp `postgresql.conf` dosyamızı düzenliyoruz:

    ```
    sudo nano /etc/postgresql/`{postgresql_sürümünüz}`/main/postgresql.conf
    ```

    Dinlenilen adres kısmına kendi IP adresimizi ya da tamamına izin vermek istiyorsak da `listen_addresses = '*'` şeklinde değiştirebiliriz.

* Ardından `pg_hba.conf` dosyamızı düzenleyelim:

    ```
    sudo nano /etc/postgresql/`{postgresql_sürümünüz}`/main/pg_hba.conf
    ```

    En alt satıra gelip `host    all             all             0.0.0.0/0               md5` bunu ekleyelim.

* Ardından `postgreql` servisimize restart atalım:
  
    ```
    sudo systemctl restart postgresql
    ```

## Projenin Çalışabilirliğini Doğrulamak:

* İlk önce proje dosyalarımızı içinde tutacak bir dizin oluşturalım ve içinde girelim:
  
    ```
    mkdir sample
    ```
    ```
    cd sample
    ```

* Proje dizininde şunu yazarak bir Python sanal ortamı oluşturalım:

    ```
    python3 -m venv venv
    ```

* Sanal ortamı etkinleştirelim:

    ```
    source venv/bin/activate
    ```

* Sanal ortamı etkinleştikten sonra deploy için gerekli olacak `gunicorn`, `uvicorn`, `uwsgi` araçlarını yükleyelim:

    ```
    pip3 install gunicorn uvicorn uwsgi
    ```

* Projeyi `Github` reposundan indirelim:

    ```
    git clone <Repo URL>
    ```

  * Github kullanıcı adı istendikten sonra  şifre de istenecektir ama bu istenen şifre hesap şifreniz değil, `https://github.com/settings/tokens` adresine girip token almanız gerekiyor.

* Ardından aşağıdaki komutu yazıp `.env` dosyası oluşturuyoruz:

    ```
    nano sample-project-backend/backend/.env
    ```

    Aşağıdaki alanları `.env` içine ekleyip, doldurup kaydediyoruz

    ```
    DB_HOST=
    POSTGRES_DB=
    POSTGRES_USER=
    POSTGRES_PASSWORD=
    DJANGO_SECRET_KEY=
    DJANGO_ALLOWED_HOSTS=
    DJANGO_EMAIL_BACKEND=
    DJANGO_EMAIL_HOST=
    DJANGO_EMAIL_HOST_USER=
    DJANGO_EMAIL_HOST_PASSWORD=
    DJANGO_DEFAULT_FROM_EMAIL=
    DEFAULT_FROM_EMAIL=
    DJANGO_EMAIL_PORT=
    DJANGO_EMAIL_USE_TLS=
    SENTRY_DSN=
    SENTRY_TRACES_SAMPLE_RATE=
    CELERY_BROKER=redis://redis:6379/0
    CELERY_BACKEND=redis://redis:6379/0
    DEBUG=
    AWS_ACCESS_KEY_ID=
    AWS_SECRET_ACCESS_KEY=
    AWS_STORAGE_BUCKET_NAME=
    AWS_S3_ENDPOINT_URL=
    AWS_SES_REGION_ENDPOINT=
    ```

    - Yukarıdaki değişken adları genel bir django projesi için düşünülmüştür, projenizin gerekliklerine göre yeniden düzenleme yapmanız elzemdir.

* Ardından `backend` dizinine gidip `requirements.txt` dosyasındaki modülleri yükleyelim:
    ```
    pip3 install -r sample-project-backend/requirements.txt
    ```
    
    debug=1 olarak ayarlanmışsa bunu da ekliyoruz:
    ```
    pip3 install -r sample-project-backend/requirements.dev.txt
    ```

* Daha sonra veritabanı tarafında tabloların oluşması için migrate işlemlerini gerçekleştirelim:

    ```
    python3 manage.py migrate
    ```

* Ardından sistemin sağlık çalışması açısından gerekli olan verileri veritabanına yükleyelim:

    ```
    python3 manage.py loaddata initial_data.json
    ```

* Sonra da yetkili bi kullanıcı oluşturalım:

    ```
    python3 manage.py createsuperuser
    ```

* Statik dosyalarımızı /vol dizinine alacağımız için orayı oluşturup, kullanım iznine açalım:

    ```
    sudo mkdir /vol
    ```

    ```
    sudo chmod -R 777 /vol
    ```

* Statik dosyalarımızı dışarı çıkartalım:

    ```
    python3 manage.py collectstatic
    ```

    Sunucumuzu koruyan bir UFW güvenlik duvarımız olmalıdır. Geliştirme sunucusunu test etmek için kullanacağımız bağlantı noktasına erişime izin vermemiz gerekir.

* Aşağıdaki komutu yazarak 8000 numaralı bağlantı noktası için bir istisna oluşturalım:
  
    ```
    sudo ufw allow 8000
    ```

* Son olarak sunucumuzu geliştirme modunda açıp test edebiliriz:

    ```
    python3 manage.py runserver
    ```
    CTRL+C diyip server'ı durduralım.

    ### Gunicorn'un Projeye Hizmet Verme Yeteneğinin Test Edilmesi

* Gunicorn'un uygulamaya hizmet edebildiğinden emin olmak için test edelim. Bunu proje dizinine girip gunicorn kullanarak projenin WSGI modülünü yükleyerek yapalım:
  
    ```
    gunicorn --bind 0.0.0.0:8000 core.wsgi
    ```

    Doğruladıktan sonra CTRL-C diyip çıkıyoruz ve sanal ortamı aşağıdaki komutla kapatıyoruz.

    ```
    deactivate
    ```

    ### Gunicorn için systemd Soket ve Servis Dosyaları Oluşturma
    Gunicorn'un Django uygulamamızla etkileşime girebildiğini test ettik ancak artık uygulama sunucusunu başlatma ve durdurmanın daha sağlam bir yolunu uygulamalıyız. Bunu başarmak için systemd servis ve soket dosyalarını oluşturacaz.

    Gunicorn soketi açılışta oluşturulacak ve bağlantıları dinleyecektir. Bir bağlantı oluştuğunda, systemd bağlantıyı yönetmek için otomatik olarak Gunicorn işlemini başlatacaktır.

* Gunicorn için sudo ayrıcalıklarına sahip bir systemd soket dosyası oluşturup açarak başlıyoruz:

    ```
    sudo nano /etc/systemd/system/gunicorn.socket
    ```

    Aşağıdaki satırları bu dosyaya ekliyoruz:
    ```
    [Unit]
    Description=gunicorn socket

    [Socket]
    ListenStream=/run/gunicorn.sock

    [Install]
    WantedBy=sockets.target
    ```
    [`Unit`]: Bu bölüm, servis birimini açıklar. "Description" satırı, servisin kısa bir açıklamasını içerir.

    [`Socket`]: Bu bölüm, bir Unix soketi üzerinde dinleme yapan bir systemd soketi (socket) servisini açıklar. Gunicorn'un web uygulamasının gelen bağlantıları dinlemesi için bir soket oluşturulmuştur. "ListenStream" satırı, Gunicorn'un dinlediği soketin dosya yolu ve türünü belirtir.

    [`Install`]: Bu bölüm, servisin ne zaman yüklenmesi gerektiğini belirtir. "WantedBy" satırı, servisin hangi hedef (target) üzerinde çalışması gerektiğini belirtir. Bu durumda, "sockets.target" hedefi üzerinde çalışması isteniyor.

* Ardından, metin düzenleyicinizde Gunicorn için sudo ayrıcalıklarına sahip bir systemd hizmet dosyası oluşturup açalım. Service dosya adı, uzantı dışında soket dosya adıyla eşleşmelidir:

    ```
    sudo nano /etc/systemd/system/gunicorn.service
    ```

* Aşağıdaki satırları bu dosyaya ekleyin:

    ```
    [Unit]
    Description=gunicorn daemon
    Requires=gunicorn.socket
    After=network.target

    [Service]
    User=username
    Group=www-data
    WorkingDirectory=/home/username/sample/sample-project-backend/backend
    ExecStart=/home/username/sample/venv/bin/gunicorn \
            --access-logfile - \
            -k uvicorn.workers.UvicornWorker \
            --workers 3 \
            --bind unix:/run/gunicorn.sock \
            core.asgi:application

    [Install]
    WantedBy=multi-user.target
    ```

    [`Unit`]:
    Description: Servisin kısa bir açıklamasını içerir. Bu durumda, "gunicorn daemon" olarak adlandırılmış bir Gunicorn servisini açıklar.
    Requires: Bu satır, servisin başlamadan önce belirtilen diğer birimlerin (unit) yüklenmiş olması gerektiğini belirtir. Bu durumda, "gunicorn.socket" adlı başka bir systemd soketi servisinin yüklenmiş olması gerekiyor.
    After: Servisin ne zaman başlaması gerektiğini belirtir. "network.target" hedefi üzerine geldikten sonra bu servisin başlaması sağlanır.

    [`Service`]:
    User ve Group: Servisin çalıştırılacağı kullanıcı ve grup bilgilerini belirtir. Bu durumda, kullanıcı "username" ve grup "www-data" olarak ayarlanmış.
    WorkingDirectory: Servisin çalıştırılacağı dizini belirtir. Bu durumda, "/home/username/sample/sample-project-backend/backend" dizininde çalışacak.
    ExecStart: Servisin nasıl başlatılacağını belirtir. Bu durumda, Gunicorn'un sanal ortamındaki bin klasöründeki "gunicorn" betiği kullanılır. Ardından, Uvicorn işçisi (uvicorn.workers.UvicornWorker) ile çalışan, 3 işçi süreci içeren ve "/run/gunicorn.sock" adlı Unix soketi üzerinden dinleyen bir Gunicorn komutu verilmiştir. Ayrıca, Gunicorn'un çalıştırılacağı Python modülü ve uygulaması ("core.asgi:application") belirtilmiştir.

    [`Install`]:
    WantedBy: Servisin hangi hedef (target) üzerinde çalışması gerektiğini belirtir. Bu durumda, "multi-user.target" hedefi üzerinde çalışması isteniyor.

    Bununla systemd hizmet dosyamız tamamlandı. Şimdi kaydedip kapatalım.

    Artık Gunicorn soketini başlatabilir ve etkinleştirebiliriz. Bu, soket dosyasını şimdi /run/gunicorn.sock konumunda ve açılışta oluşturacaktır. Bu sokete bir bağlantı yapıldığında, systemd bunu işlemek için otomatik olarak gunicorn.service'i başlatacaktır:

    ```
    sudo systemctl start gunicorn.socket
    ```
    ```
    sudo systemctl enable gunicorn.socket
    ```

    Başlatılıp başlatılamayacağını öğrenmek için işlemin durumunu kontrol edelim:

    ```
    sudo systemctl status gunicorn.socket
    ```

    Daha sonra, /run dizini içinde gunicorn.sock dosyasının varlığını kontrol edin:

    ```
    file /run/gunicorn.sock
    ```

    Systemctl status komutu bir hata oluştuğunu belirtiyorsa veya gunicorn.sock dosyasını dizinde bulamıyorsanız, bu Gunicorn soketinin doğru şekilde oluşturulamadığının bir göstergesidir. Gunicorn soketinin günlüklerini şunu yazarak kontrol edin:

    ```
    sudo journalctl -u gunicorn.socket
    ```

    Şu anda gunicorn.socket ünitesini yeni başlattıysanız, gunicorn.service henüz aktif olmayacaktır çünkü soket henüz herhangi bir bağlantı almamıştır. Bunu yazarak kontrol edebilirsiniz:

    ```
    sudo systemctl status gunicorn
    ```

    Soket etkinleştirme mekanizmasını test etmek için aşağıdaki komutu yazarak curl aracılığıyla sokete bir bağlantı gönderebilirsiniz:

    ```
    curl --unix-socket /run/gunicorn.sock localhost
    ```

    curl çıktısı veya systemctl status çıktısı bir sorun oluştuğunu gösteriyorsa ek ayrıntılar için günlükleri kontrol edin:
    ```
    sudo journalctl -u gunicorn
    ```

    ``/etc/systemd/system/gunicorn.service`` dosyanızda sorun olup olmadığını kontrol edin. ``/etc/systemd/system/gunicorn.service`` dosyasında değişiklik yaparsanız, hizmet tanımını yeniden okumak için arka plan programını yeniden yükleyin ve şunu yazarak Gunicorn işlemini yeniden başlatın:
    ```
    sudo systemctl daemon-reload
    ```
    ```
    sudo systemctl restart gunicorn
    ```

    ### Nginx'i Gunicorn'a Proxy Pass ile yönlendirmek

    Artık Gunicorn kurulduğuna göre, Nginx'i sürece trafik aktaracak şekilde yapılandırmamız gerekiyor.

    Nginx'in kullanılabilir siteler dizininde yeni bir sunucu bloğu oluşturup açarak başlayalım:

    ```
    sudo nano /etc/nginx/sites-available/core
    ```

    - İçeride yeni bir sunucu bloğu açalım. Bu bloğun normal 80 numaralı bağlantı noktasını dinlemesi gerektiğini ve sunucumuzun alan adına veya IP adresine yanıt vermesi gerektiğini belirterek başlayacağız.
  
    - Daha sonra Nginx'e favicon bulmayla ilgili herhangi bir sorunu görmezden gelmesini söyleyeceğiz. Ayrıca ``/vol/web/static`` dizininizde topladığımız statik varlıkları nerede bulacağını da söyleyeceksiniz. Bu dosyaların tümünün standart URI öneki "/static" olduğundan, bu istekleri eşleştirmek için bir konum bloğu oluşturabiliriz.

    - Son olarak, diğer tüm isteklerle eşleşecek bir ``location / {}`` bloğu oluşturun. Bu konumun içine, Nginx kurulumunda bulunan standart ``proxy_params`` dosyasını dahil edecek ve ardından trafiği doğrudan Gunicorn soketine aktaracaksınız.
  
    ```
    server {
        listen 80;
        server_name 192.168.0.11;

        location = /favicon.ico { access_log off; log_not_found off; }
        location /static/ {
            root /vol/web;
        }

        location / {
            include proxy_params;
            proxy_pass http://unix:/run/gunicorn.sock;
        }
    }
    ```

    - Yukarıda `root /vol/web` yazan yere `settings.py` içinde bulunan statik alanların aşağıdaki gibi olduğu varsayılarak hazırlanmıştır:

        ```
        STATIC_URL = '/static/'
        MEDIA_URL = '/media/'


        STATIC_ROOT = '/vol/web/static'
        MEDIA_ROOT = '/vol/web/media'
        ```

    İşimiz bittiğinde dosyayı kaydedin ve kapatın. Artık dosyayı sitelerin etkin olduğu dizine bağlayarak etkinleştirebilirsiniz:

    ```
    sudo ln -s /etc/nginx/sites-available/core /etc/nginx/sites-enabled
    ```

    Aşağıdakileri yazarak Nginx yapılandırmanızı sözdizimi hatalarına karşı test edelim:

    ```
    sudo nginx -t
    ```

    Herhangi bir hata bildirilmezse devam edelim ve şunu yazarak Nginx'i yeniden başlatalım:

    ```
    sudo systemctl restart nginx
    ```

    Son olarak, güvenlik duvarınızı 80 numaralı bağlantı noktasında normal trafiğe açmamız gerekir. Artık geliştirme sunucusuna erişmemiz gerekmediğinden, 8000 numaralı bağlantı noktasını açma kuralını da kaldırabiliriz:

    ```
    sudo ufw delete allow 8000
    ```

    ```
    sudo ufw allow 'Nginx Full'
    ```

    ### Nginx ve Gunicorn Sorunlarını Giderme

    Bu son adım uygulamamızı göstermezse kurulumumuzda sorun gidermemiz gerekecektir.

    #### Nginx, Django Uygulaması Yerine Varsayılan Sayfayı Gösteriyor

    Nginx, uygulamanıza proxy göndermek yerine varsayılan sayfayı görüntülerse, bu genellikle ``/etc/nginx/sites-available/core`` dosyası içindeki ``server_name``'i sunucumuzun IP adresini veya etki alanı adını gösterecek şekilde ayarlamamız gerektiği anlamına gelir.

    Projenizin sunucu bloğundaki ``server_name``, seçilecek varsayılan sunucu bloğundaki ``server_name``'inden daha spesifik olmalıdır.

    #### Nginx, Django Uygulaması Yerine 502 Hatalı Ağ Geçidi Hatası Görüntülüyor

    502 hatası, Nginx'in isteği başarılı bir şekilde proxy olarak gerçekleştiremediğini gösterir. Çok çeşitli yapılandırma sorunları kendilerini 502 hatasıyla ifade eder, bu nedenle sorunu doğru şekilde gidermek için daha fazla bilgi gerekir.

    Daha fazla bilgi aramak için birincil yer Nginx'in hata günlükleridir. Genel olarak bu size proxy olayı sırasında hangi koşulların sorunlara neden olduğunu söyleyecektir. Aşağıdakileri yazarak Nginx hata günlüklerini takip edin:

    ```
    sudo tail -F /var/log/nginx/error.log
    ```

    Şimdi tarayıcınızda yeni bir hata oluşturmak için başka bir istekte bulunun (sayfayı yenilemeyi deneyin). Günlüğe yazılan yeni bir hata mesajı almalısınız. Mesaja bakarsanız sorunu daraltmanıza yardımcı olacaktır.

    Aşağıdaki iletileri alabiliriz:

    ``unix:/run/gunicorn.sock'a bağlanılamadı (2: Böyle bir dosya veya dizin yok)``

    Bu, Nginx'in ``gunicorn.sock`` dosyasını belirtilen konumda bulamadığını gösterir. ``/etc/nginx/sites-available/core`` dosyasında tanımlanan ``proxy_pass`` konumunu ``gunicorn.socket`` sistemd birimi tarafından oluşturulan ``gunicorn.sock`` dosyasının gerçek konumuyla karşılaştırmalısınız.

    ``connect() to unix:/run/gunicorn.sock failed (13: Permission denied)``

    Bu, Nginx'in izin sorunları nedeniyle Gunicorn soketine bağlanamadığı anlamına gelir. Bu, prosedür bir sudo kullanıcısı yerine kök kullanıcı kullanılarak izlendiğinde meydana gelebilir. Systemd Gunicorn soket dosyasını oluşturabilse de Nginx bu dosyaya erişemiyor.

    ``gunicorn.sock`` dosyasının kök dizini ``(/)`` arasındaki herhangi bir noktada sınırlı izinler varsa bu durum meydana gelebilir. Soket dosyanızın ve onun üst dizinlerinin her birinin izinlerini ve sahiplik değerlerini, soket dosyanızın mutlak yolunu ``namei`` komutuna ileterek inceleyebilirsiniz:

    ```
    namei -l /run/gunicorn.sock
    ```

    ```
    Output
    f: /run/gunicorn.sock
    drwxr-xr-x root root /
    drwxr-xr-x root root run
    srw-rw-rw- root root gunicorn.sock
    ```

    Çıktı, dizin bileşenlerinin her birinin izinlerini görüntüler. İzinlere (birinci sütun), sahipe (ikinci sütun) ve grup sahibine (üçüncü sütun) bakarak, soket dosyasına ne tür erişime izin verildiğini anlayabilirsiniz.

    Yukarıdaki örnekte, soket dosyası ve soket dosyasına giden her dizin, sistemdeki her kullanıcı tarafından okuma ve çalıştırma izinlerine sahiptir (dizinlerin izin sütunu, --- yerine r-x ile biter). Nginx süreci, sokete başarıyla erişebilmelidir.

    Eğer sokete giden dizinlerden herhangi birinin herkes tarafından okuma ve çalıştırma izinlerine sahip olmaması durumunda, Nginx, bu dizinlere erişim sağlayamaz. Bu izinleri sağlamak için ya herkesin bu dizinlere okuma ve çalıştırma izinlerini vermek veya Nginx'in üye olduğu bir grup tarafından bu izinleri almak gereklidir.

    #### Django'da Alınan Hata: "Could not connect to server: Connection refused"

    Web tarayıcısında uygulamanın bazı bölümlerine erişmeye çalıştığınızda Django'dan alabileceğiniz olası bir hata şudur:

    ``OperationalError at /admin/login/
    could not connect to server: Connection refused
    Is the server running on host "localhost" (127.0.0.1) and accepting
    TCP/IP connections on port 5432?``

    Bu, Django'nun Postgres veritabanına bağlanamadığını gösterir. Aşağıdakileri yazarak Postgres örneğinin çalıştığından emin olun:

    ```
    sudo systemctl status postgresql
    ```

    Aktif değilse, şunu yazarak başlatabilir ve önyükleme sırasında otomatik olarak başlamasını sağlayabilirsiniz (önceden yapılandırılmamışsa):


    ```
    sudo systemctl start postgresql
    ```

    ```
    sudo systemctl enable postgresql
    ```

    Hala sorun yaşıyorsanız ``~/sample-project-backend/backend/core/settings.py`` dosyasında tanımlanan veritabanı ayarlarının doğru olduğundan emin olun.

    #### Daha Fazla Sorun Giderme

    Ek sorun giderme işlemleri için günlükler temel nedenleri daraltmaya yardımcı olabilir. Her birini sırayla kontrol edin ve sorunlu alanları belirten mesajları arayın.

    Aşağıdaki günlükler yararlı olabilir:

    Şunu yazarak Nginx işlem günlüklerini kontrol edin:
    ```
    sudo Journalctl -u nginx
    ```
    
    Şunu yazarak Nginx erişim günlüklerini kontrol edin:
    ```
    sudo less /var/log/nginx/access.log
    ```
    Şunu yazarak Nginx hata günlüklerini kontrol edin: 
    ```
    sudo less /var/log/nginx/error.log
    ```
    Gunicorn uygulama günlüklerini şunu yazarak kontrol edin: 
    ```
    sudo Journalctl -u gunicorn
    ```
    Gunicorn soket günlüklerini şunu yazarak kontrol edin: 
    ```
    sudo Journalctl -u gunicorn.socket
    ```

    Yapılandırmanızı veya uygulamanızı güncellerken, değişikliklerinize uyum sağlamak için büyük olasılıkla işlemleri yeniden başlatmanız gerekecektir.

    Django uygulamanızı güncellerseniz, aşağıdakileri yazarak değişiklikleri almak için Gunicorn işlemini yeniden başlatabilirsiniz:

    ```
    sudo systemctl restart gunicorn
    ```

    Gunicorn soketini veya hizmet dosyalarını değiştirirseniz arka plan programını yeniden yükleyin ve şunu yazarak işlemi yeniden başlatın:

    ```
    sudo systemctl daemon-reload
    ```

    ```
    sudo systemctl restart gunicorn.socket gunicorn.service
    ```

    Nginx sunucu bloğu yapılandırmasını değiştirirseniz, yapılandırmayı ve ardından şunu yazarak Nginx'i test edin:

    ```
    sudo nginx -t && sudo systemctl restart nginx
    ```

    Bu dökümantasyon [How To Set Up Django with Postgres, Nginx, and Gunicorn on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-22-04) kaynağı referans alınarak hazırlanmıştır.