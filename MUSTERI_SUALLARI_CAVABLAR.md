# Müştəri Sualları və Texniki Cavablar

Bu sənəd, layihənin texniki arxitekturası, təhlükəsizlik tədbirləri, rəqəmsal transformasiya tələbləri və biznes məntiqi ilə bağlı müştəri tərəfindən verilmiş sualların ətraflı cavablarını əhatə edir.

---

## 1. Ümumi Arxitektura və Sistem Nüvəsi

### 1.1. E-commerce və CRM nüvəsi necə yazılacaq?

Sistem modullu monolit (modular monolith) arxitekturası əsasında qurulacaq. Bu yanaşma ilk mərhələdə inkişaf sürətini təmin edir, daha sonra ehtiyac yarandıqda mikroservislərə parçalanmağa imkan verir.

Nüvə aşağıdakı modullara ayrılacaq:

* **Catalog modulu:** Məhsullar, kateqoriyalar, brendlər, atributlar
* **Order modulu:** Səbət, sifariş, ödəniş axını
* **Account modulu:** İstifadəçi, autentifikasiya, rol idarəetməsi
* **CRM modulu:** Müştəri münasibətləri, kontaktlar, satış axını
* **Inventory modulu:** Anbar, stok hərəkəti
* **Notification modulu:** SMS, email, push bildirişlər
* **Analytics modulu:** Hesabatlar, statistikalar

Hər modul öz daxili API-sinə malik olacaq və digər modullarla yalnız müəyyən edilmiş interfeys üzərindən əlaqə saxlayacaq. Bu, gələcəkdə hər bir modulu ayrıca mikroservisə çevirməyə imkan verir.

### 1.2. Backend və Mobil proqram dili

**Backend:** Python (Django 5.x + Django REST Framework)

Səbəblər:

* Geniş ekosistem və mükəmməl sənədləşmə
* Güclü ORM (SQL injection qoruması daxili olaraq)
* Django admin paneli CRM üçün əla baza
* Sürətli inkişaf, uzun müddətli dəstək
* Azərbaycan bazarında komanda tapmaq asandır

**Mobil tətbiq:** Flutter (Dart)

Səbəblər:

* Tək kod bazası iOS və Android üçün (90%+ kod paylaşımı)
* Google tərəfindən aktiv dəstəklənən stabil framework
* Öz rendering engine-i (Skia) sayəsində hər iki platformada eyni UI və davranış
* Yüksək performans — native koda kompilyasiya edilir (ARM və x86)
* Hot reload ilə sürətli inkişaf dövrü
* Zəngin widget kitabxanası (Material Design və Cupertino)
* Dart dilinin tip təhlükəsizliyi və null safety
* Web və desktop hədəflər də eyni kod bazasından mümkündür (gələcək ehtiyaclar üçün)

### 1.3. Verilənlər bazası və data bütövlüyü

**Əsas DB:** PostgreSQL 16 (Relational)

Səbəblər:

* ACID uyğunluğu tam təmin edir (atomicity, consistency, isolation, durability)
* Güclü foreign key və constraint dəstəyi
* Maliyyə və sifariş əməliyyatları üçün tranzaksiya təminatı
* JSONB sahələri ilə yarı-strukturlaşmış data da saxlanıla bilir
* Full-text search dəstəyi
* Partitioning və replication üçün zəngin alətlər

**Keş və sessiya:** Redis 7

* Sessiya saxlanması, keşləmə, rate limiting sayğacları
* Celery broker kimi asinxron tapşırıqlar üçün

**Axtarış:** Elasticsearch 8 (məhsul axtarışı və filtrlər üçün)

**Data bütövlüyünün qorunması:**

1. **Database-level constraint-lər:** NOT NULL, UNIQUE, CHECK, FOREIGN KEY
2. **Django validators:** Model səviyyəsində validasiya
3. **Transaction istifadəsi:** `@transaction.atomic` kritik əməliyyatlarda
4. **Audit log:** Bütün məlumat dəyişikliklərinin qeydiyyatı (kim, nə vaxt, nə dəyişdi)
5. **Soft delete:** Fiziki silinmə əvəzinə `is_deleted` bayrağı
6. **Backup:** Gündəlik tam, saatlıq incremental ehtiyat nüsxələr (WAL-G)
7. **Replication:** Master-slave konfiqurasiyası, sinxron streaming replication

### 1.4. Server resursları

İlkin mərhələ üçün tövsiyə olunan konfiqurasiya:

**Application Server (Backend):**
* CPU: 4 vCPU
* RAM: 8 GB
* SSD: 100 GB NVMe

**Database Server:**
* CPU: 4 vCPU
* RAM: 16 GB (DB üçün RAM kritik əhəmiyyətlidir)
* SSD: 200 GB NVMe (IOPS yüksək olmalı)

**Cache/Queue Server (Redis):**
* CPU: 2 vCPU
* RAM: 4 GB
* SSD: 20 GB

**Elasticsearch Server:**
* CPU: 2 vCPU
* RAM: 8 GB (minimum 4 GB heap)
* SSD: 100 GB

**CDN və Media Storage:**
* Statik fayl və şəkil saxlanması üçün ayrıca obyekt storage (S3 uyğun)
* CDN üçün Cloudflare istifadə olunur

**GPU:** Bu layihədə GPU tələb olunmur. Əgər gələcəkdə AI əsaslı məhsul tövsiyəsi və ya şəkil analizi əlavə edilərsə, ayrı GPU instansı nəzərdən keçirilə bilər.

**Yük artdıqca horizontal scaling:** Load balancer arxasında bir neçə backend instansı yerləşdirilir.

### 1.5. Testlər və bug düzəlişləri

Bəli, layihə təslim edildikdən sonra müəyyən edilmiş zəmanət dövrü ərzində (adətən 3-6 ay) texniki tapşırıqda (TZ) razılaşdırılmış funksionallıqda aşkar olunan bütün buglar ödənişsiz düzəldilir.

Zəmanət dövrünə daxildir:

* Sistem xətaları (crash, 500 errors)
* Funksional uyğunsuzluqlar (TZ-dan kənarlaşmalar)
* Performans problemləri
* Təhlükəsizlik zəiflikləri

Zəmanətə daxil deyil:

* Yeni funksionallıq əlavə etmək
* Mövcud funksiyaların dəyişdirilməsi
* Üçüncü tərəf servislərin dəyişməsi səbəbindən yaranan problemlər (o şərtlə ki, biz həmin servislə işləyirdik)

### 1.6. Test mühiti və canlı yayım

Üç mühit yaradılacaq:

1. **Development (DEV):** Geliştirmə komandası üçün
2. **Staging (UAT):** Müştəri testi üçün, production-un eyni konfiqurasiyasında
3. **Production (PROD):** Canlı sistem

**Prosedur:**

* Hər yeni özəllik əvvəlcə DEV mühitində yazılır və test olunur
* Qəbul olunduqdan sonra Staging-ə deploy edilir
* Müştəri Staging-də UAT (User Acceptance Testing) aparır
* Təsdiq alındıqdan sonra Production-a çıxarılır
* CI/CD pipeline ilə avtomatlaşdırılmış deploy (GitHub Actions)

**Canlı yayım sonrası:**

* 2 həftəlik gücləndirilmiş dəstək dövrü (hotfix sürəti ilə)
* Monitoring və alerting aktiv
* Rollback planı hər deploy üçün hazırdır
* Testdən sonra aşkar edilən bütün buglar zəmanət çərçivəsində düzəldilir

### 1.7. Təhlükəsizlik və stress testlər

Bəli, aşağıdakı test növləri aparılacaq:

**Təhlükəsizlik testləri:**

* OWASP ZAP ilə avtomatlaşdırılmış pentest
* Burp Suite ilə manual pentest
* Nessus ilə vulnerability scan
* Dependency scan (Snyk, Safety, `flutter pub outdated`, `pub.dev` security advisories)

**Stress və performans testləri:**

* JMeter ilə yük testi və API performans testi aparılır
* Məqsəd: ən azı 1000 paralel istifadəçi rahat xidmət edilməli

**Bot testləri:**

* Rate limiting və captcha effektivliyinin yoxlanması
* Scraping cəhdlərinin qarşısının alınması
* DDoS simulyasiyası (Cloudflare qoruması ilə)

Bütün test nəticələri hesabatlaşdırılıb müştəriyə təqdim olunacaq.

### 1.8. Scaling memarlıq planı

Bəli, ayrıca scaling planı təqdim olunacaq. Qısa icmal:

**Horizontal scaling (üfiqi miqyaslama):**

* Backend application serverləri stateless dizayn edilir
* Nginx load balancer arxasında N sayda instans
* Auto-scaling qaydaları: CPU > 70% və ya response time > 500ms olanda yeni instans əlavə olunur

**Database scaling:**

* Read replicas yaradılır, oxuma sorğuları bura yönləndirilir
* Connection pooling (PgBouncer)
* Yüksək yük üçün Citus genişlənməsi ilə sharding tətbiq ediləcək

**Keş qatı:**

* Redis cluster mode (yük paylaşımı)
* CDN ilə statik məzmunun kənarlaşdırılması
* Application-level keşləmə (məhsul detalları, kateqoriya ağacı)

**Asinxron işlər:**

* Celery + Redis ilə ağır əməliyyatlar fon rejiminə atılır (email göndərmə, hesabat generasiyası, şəkil emalı)
* Sifariş axınında kritik olmayan hissələr queue-yə yazılır

**Monitorinq:**

* Prometheus + Grafana ilə metrika izləmə
* Alert qaydaları CPU, RAM, response time, error rate üçün

### 1.9. Elasticsearch və axtarış

Bəli, məhsul axtarışı və filtrlər üçün **Elasticsearch** istifadə olunacaq.

**Səbəblər:**

* PostgreSQL full-text search milyonlarla məhsul üçün yavaşdır
* Elasticsearch fuzzy matching, synonym, autocomplete dəstəkləyir
* Faceted search (çoxölçülü filtrlər) təbii olaraq dəstəklənir
* Azərbaycan dili üçün xüsusi analyzer yazılacaq (schwa hərfi, sözlərin kökü)

**Tətbiq yanaşması:**

* Django məhsulları PostgreSQL-də saxlayır (əsas məlumat mənbəyi)
* Signal-lar vasitəsilə dəyişikliklər Elasticsearch-ə sinxronlaşdırılır
* Axtarış və filtrlər yalnız Elasticsearch üzərindən gedir
* Bulk reindex gecə avtomatik aparılır (data drift-in qarşısının alınması)

---

## 2. Kiber Təhlükəsizlik Şöbəsinin Tələbləri

### 2.1. Autentifikasiya və sessiya testləri

**Brute-force qoruması**

Tətbiq edilir:

* IP əsaslı rate limiting (dəqiqədə maksimum 5 login cəhdi)
* İstifadəçi əsaslı bloklama (10 uğursuz cəhdən sonra 30 dəqiqə bloklama)
* Cəhdlər arasında ekponensial geri çəkilmə (exponential backoff)
* Captcha avtomatik aktivləşir 3 uğursuz cəhdən sonra
* Bütün cəhdlər `LoginAttempt` cədvəlinə yazılır

**Sessiya tokeninin etibarlılığı**

* Logout zamanı sessiya dərhal silinir (`request.session.flush()`)
* JWT istifadə edilir, logout zamanı refresh token blacklist-ə atılır
* Eyni istifadəçinin paralel sessiyaları izlənilir
* Session timeout: 30 dəqiqə aktivsizlikdən sonra

**JWT imzasının yoxlanması**

* `alg: none` bypass açıq şəkildə qadağandır (`ALGORITHMS = ['HS256']`)
* Secret key minimum 256 bit
* Token payload-da sensitiv məlumat yoxdur
* `exp`, `iat`, `jti` claim-ləri həmişə yoxlanılır

**Şifrə sıfırlama axını**

* Reset token 32 byte kriptoqrafik random (`secrets.token_urlsafe`)
* Token 15 dəqiqə etibarlıdır
* Bir dəfə istifadədən sonra dərhal silinir
* Token istifadə edildikdə digər aktiv sessiyalar invalidate olunur
* Köhnə şifrə ilə eyni ola bilməz (son 5 şifrə yadda saxlanılır)

**MFA bypass qarşısı**

* OTP kodu 5 dəqiqə etibarlıdır
* Maksimum 3 yanlış cəhd, sonra sessiya ləğv olunur
* Hər OTP üçün unikal verification token
* OTP rate limiting (dəqiqədə 1 təkrar göndərmə)

### 2.2. Ödəniş testləri

**Qiymət manipulyasiyası**

Bəzi zəif sistemlərdə səbətə məhsul əlavə edildikdə client tərəfindən məhsulun qiyməti də sorğuda göndərilir və server bu qiyməti qəbul edir. Hücumçu DevTools və ya proxy ilə sorğunu tutub qiyməti dəyişərək məhsulu daha ucuz ala bilər. Bizim sistemdə bu riskin qarşısı belə alınır:

* Client sorğusunda yalnız `product_id` və `quantity` qəbul olunur, qiymət sahəsi ümumiyyətlə emal edilmir
* Server məhsul qiymətini hər dəfə verilənlər bazasından oxuyur
* Sifarişin ümumi məbləği yalnız server tərəfdə hesablanır
* Endirim, kupon və vergi hesablamaları da server tərəfdə aparılır

```python
product = Product.objects.select_for_update().get(id=product_id)
total = product.price * quantity  # Qiymət DB-dən gəlir
```

**Kupon/endirim abuse**

* Hər kuponun istifadə sayğacı (`usage_limit`, `used_count`)
* İstifadəçi başına limit (`per_user_limit`)
* Atomik UPDATE ilə yarış vəziyyətinin qarşısı alınır
* Minimum sifariş məbləği yoxlanılır
* Müddət bitmə tarixi yoxlanılır

**Sifariş modifikasiyası**

* Sifariş `PENDING_PAYMENT` statusundan çıxdıqdan sonra immutable olur
* Status keçidləri yalnız state machine vasitəsilə mümkündür
* Bütün dəyişikliklər audit log-a yazılır

**PCI-DSS uyğunluğu**

* Kart məlumatları **heç vaxt** bizim serverdə saxlanılmır
* Ödəniş provayderinin hosted payment page istifadə olunur
* Yalnız transaction ID və son 4 rəqəm saxlanılır
* CVV heç vaxt log fayllarına, verilənlər bazasına, yaxud hər hansı persistent mühitə yazılmır
* TLS 1.2+ ödəniş endpoint-lərində məcburi

**İkiqat ödəniş (double charge)**

* Hər sifariş üçün unikal `idempotency_key` generasiya olunur
* Ödəniş provayderinə eyni key ilə sorğu göndərilir (idempotent request)
* Database səviyyəsində unique constraint: `(user_id, order_id)` ilə çoxalma qarşısı
* Webhook qəbulunda da idempotency yoxlanılır

### 2.3. OWASP Top 10 Testləri (Pentest)

**SQL Injection**

* Bütün sorğular Django ORM ilə yazılır (parametrlənmiş sorğular)
* Raw SQL istifadə edildikdə yalnız `%s` placeholder istifadə olunur
* String konkatenasiyası ilə sorğu yaradılması qadağandır
* Input sanitization həm client, həm server səviyyəsində

**XSS (Cross-Site Scripting)**

* Django template mühərriki default olaraq bütün dəyişənləri escape edir
* `|safe` filtri yalnız zəruri hallarda və validasiyadan sonra istifadə olunur
* User input HTML-də `|escape`, JavaScript-də `|escapejs` filtri ilə çıxarılır
* Content Security Policy (CSP) header-i təyin olunub
* Məhsul adları, şərhlər üçün markdown-to-safe-html çevirici

**IDOR (Insecure Direct Object Reference)**

* Bütün view-lərdə obyektin istifadəçiyə məxsusluğu yoxlanılır

```python
order = get_object_or_404(Order, id=order_id, user=request.user)
```

* Admin endpoint-lərində `is_staff` yoxlanılır
* Sequential ID-lərin təxmin edilməməsi üçün UUID istifadəsi (həssas URL-lərdə)

**SSRF (Server-Side Request Forgery)**

* Xarici URL-dən şəkil yüklənməsi qadağandır
* Fayl yükləməsi yalnız istifadəçinin öz cihazından mümkündür
* Əgər URL əsaslı yükləmə varsa, domain whitelist tətbiq olunur
* Internal IP aralıqları (10.x, 192.168.x, 127.x, 169.254.x) blok edilir

**XXE (XML External Entity)**

* Sistem XML formatını ümumiyyətlə qəbul etmir
* Əgər lazım olarsa `defusedxml` kitabxanası istifadə edilir
* `lxml.etree` default parametrlərlə istifadə edilmir

**Mass Assignment**

* Serializer-lərdə `fields` açıq şəkildə göstərilir (whitelist)
* `exclude` əvəzinə `fields` istifadə olunur
* `role`, `is_staff`, `is_superuser` kimi sahələr heç vaxt user input-dan qəbul olunmur
* Django forms da eyni prinsipi izləyir

### 2.4. Mobil tətbiq testləri

**APK/IPA reverse engineering**

* API key-lər və secret-lar kod içərisində hardcoded olmayacaq
* Build zamanı `--dart-define-from-file` bayrağı ilə JSON konfiqurasiyadan oxunur (CI/CD pipeline-da secret-lər injection olunur)
* Flutter Dart kodu AOT (Ahead-of-Time) kompilyasiya ilə native ikili fayla çevrilir, bu da reverse engineering-i çətinləşdirir
* Android üçün `--obfuscate --split-debug-info` bayraqları ilə build edilir (Dart simvolları obfuscate olunur)
* iOS üçün bitcode və App Store encryption aktiv
* `flutter_jailbreak_detection` paketi ilə root/jailbreak cihaz aşkarlanması
* Release build-da bütün debug loglar və `print()` çağırışları söndürülür

**SSL Pinning**

* `http_certificate_pinning` paketi ilə sertifikat pinning
* Public key hash-i kod içində əvvəlcədən saxlanılır
* MITM hücumlarına qarşı müdafiə
* Certificate rotation üçün backup pin əlavə olunur
* Pin uyğunsuzluğu halında network sorğuları avtomatik rədd olunur və log yaranır

**Cihazda məlumat saxlanması**

* Kart nömrəsi və CVV cihazda **heç vaxt** saxlanılmır
* JWT token yalnız `flutter_secure_storage` paketi vasitəsilə saxlanılır:
  * Android-də arxada EncryptedSharedPreferences (AES-256-GCM) istifadə olunur
  * iOS-da arxada Keychain (kSecAttrAccessibleAfterFirstUnlock) istifadə olunur
* `local_auth` paketi ilə biometrik autentifikasiya (barmaq izi / FaceID) əlavə qoruma üçün
* Cihazdan çıxış zamanı `flutter_secure_storage.deleteAll()` ilə bütün sensitiv data silinir
* Həssas obyektlər yaddaşda saxlandıqda dərhal `null` edilib zibil yığıcısına buraxılır

**Deeplink manipulation**

* Bütün deeplink-lər `go_router` paketi ilə idarə olunur
* Hər deeplink əsas tətbiqdə autentifikasiya yoxlamasından keçir
* Ödəniş axınında deeplink parametr-lərinə güvənilmir, server təsdiqi tələb olunur
* Kritik əməliyyatlar session context ilə əlaqələndirilir (server-də yoxlanılır)
* Android App Links və iOS Universal Links rəsmi domain doğrulaması ilə
* Custom scheme-lər (məs. `myapp://`) domen sahibliyini yoxlamır və istənilən tətbiq eyni scheme-i qeydiyyatdan keçirib deeplink-i ələ keçirə bilər. Buna görə https əsaslı App Links / Universal Links üstün tutulur, çünki onlar domain doğrulaması tələb edir və yalnız bizim tətbiq tərəfindən açıla bilər

**Clipboard sızması**

* Flutter `TextField` widget-lərində kart məlumatı daxil olan sahələr üçün `enableInteractiveSelection: false` və `contextMenuBuilder` ilə kopya menyu bloklanır
* `autocorrect: false`, `enableSuggestions: false` parametrləri ilə input history söndürülür
* `obscureText: true` PIN və şifrə sahələri üçün
* Background-a keçdikdə həssas ekranlar mask olunur:
  * Android: `flutter_windowmanager` paketi ilə `FLAG_SECURE` aktivləşir (screenshot və task preview qadağan)
  * iOS: `AppLifecycleState.paused` hadisəsində ekran üstünə overlay qoyulur
* Copy-paste hadisəsi log-lanmır (privacy təmini)

### 2.5. Avtorizasiya və Rol əsaslı Giriş Nəzarəti

**Horizontal privilege escalation**

* Hər view-da obyekt sahibliyi yoxlanılır (yuxarıdakı IDOR nümunəsi)
* `queryset = Model.objects.filter(user=self.request.user)` pattern-i
* DRF-də `IsOwnerOrReadOnly` kimi xüsusi permission class-ları

**Vertical privilege escalation**

* Admin endpoint-ləri `IsAdminUser` permission tələb edir
* Django `@staff_member_required` dekoratoru
* Rol yoxlaması həm URL, həm də view səviyyəsində
* Menu elementi gizlətmək kifayət deyil, backend-də də yoxlama aparılır

**Admin panelin gizlədilməsi**

* Admin URL default `/admin/` deyil, xüsusi yoldadır
* IP whitelist tətbiq oluna bilər (yalnız ofis IP-lərdən giriş)
* Admin girişi üçün 2FA məcburidir
* Fail2ban ilə admin bruteforce qoruması

**API endpoint qorunması**

* DRF default permission: `IsAuthenticated`
* Hər endpoint açıq şəkildə permission class elan edir
* Anonymous icazə verilən endpoint-lər nəzarət altındadır (search, homepage)
* Swagger/OpenAPI yalnız developer mühitində açıqdır

### 2.6. API təhlükəsizlik testləri

**Rate limiting**

* DRF throttling (AnonRateThrottle, UserRateThrottle)
* Anonim: saatda 100 sorğu, istifadəçi: saatda 1000 sorğu
* Həssas endpoint-lər üçün (login, OTP) aşağı limit
* Nginx səviyyəsində limit_req ilə ikinci qat
* Cloudflare bot management əlavə qat

**GraphQL introspection**

* Layihədə GraphQL istifadə olunmur (REST API)
* Əgər gələcəkdə əlavə edilsə, production-da introspection söndürüləcək
* Query depth və complexity limit-ləri qoyulacaq

**Versiya fərqləri**

* API versiyalaşdırma URL-də (`/api/v1/`, `/api/v2/`)
* Köhnə versiyalar deprecated olunduqda `Deprecation` header-i göndərilir
* Sunset policy: minimum 6 ay xəbərdarlıq, sonra kapat
* Köhnə versiyalara da təhlükəsizlik patch-ları tətbiq olunur

**Error handling**

* Production-da `DEBUG = False`
* Stack trace heç vaxt client-ə qayıtmır
* Error mesajları generic-dir (məs. "Xəta baş verdi" yerinə "PostgreSQL connection refused at 10.0.0.5:5432" olmamalı)
* Bütün xətalar server tərəfdə log-lanır
* Sensitive məlumatlar (path, DB adı, query) log-a da minimum yazılır

**CORS konfiqurasiyası**

* `Access-Control-Allow-Origin: *` qadağandır
* Yalnız təsdiqlənmiş domain-lər whitelist-də
* Credentials ilə CORS üçün dəqiq origin tələb olunur
* Preflight OPTIONS sorğuları düzgün idarə olunur

### 2.7. İnfrastruktur testləri

**Dependency scan**

* GitHub Dependabot aktiv (avtomatik PR-lar)
* `pip-audit` və `safety` CI pipeline-da icra olunur
* Flutter layihəsində `flutter pub outdated` və `pub.dev` xəbərdarlıqları hər build-da yoxlanılır
* OWASP Dependency-Check Java tərkibli layihələrdə

**Container scan**

* Trivy hər Docker build-dan sonra işə düşür
* CRITICAL və HIGH səviyyəli zəifliklər deploy-u blok edir
* Base image olaraq Google Distroless istifadə olunur (minimum hücum səthi, paket meneceri və shell olmadığı üçün)
* Image-lər müntəzəm yenilənir (həftədə bir)

**Secret scan**

* GitLeaks pre-commit hook kimi konfiqurasiya olunub
* CI pipeline-da ikinci yoxlama
* Tarixi commit-lərdə də yoxlama aparılır
* .env faylları heç vaxt commit edilmir

**TLS konfiqurasiyası**

* SSLLabs A+ rating hədəflənir
* TLS 1.0 və 1.1 söndürülüb, yalnız 1.2 və 1.3 aktiv
* Güclü cipher suite-lər (AEAD algorithms)
* HSTS preload list-ə qeyd olunur
* Certificate auto-renewal (Let's Encrypt + certbot)

**HTTP Security Headers**

| Header | Dəyər |
|--------|-------|
| `Content-Security-Policy` | strict policy with nonce |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | minimum icazələrlə |
| `X-XSS-Protection` | `1; mode=block` |

---

## 3. Rəqəmsal Transformasiya Direktorluğunun Tələbləri

### 3.1. Lisensiyasız komponent istifadə edilməməsi

Bütün istifadə olunan kitabxanalar və komponentlər açıq lisensiyalı (MIT, Apache 2.0, BSD, GPL uyğun) olacaq. Pullu kommersiya komponentinə ehtiyac yarandığı təqdirdə onun siyahısı və qiyməti müştəriyə təqdim olunacaq, lisenziya alınması və ödəniş Avrora tərəfindən həyata keçiriləcək.

### 3.2. Son versiyalı komponentlər

* Komponentin ən son stabil versiyası seçilir (LTS versiyalar üstün tutulur)
* Alpha/beta versiyalar production-da istifadə olunmur
* Versiyalar `requirements.txt` (backend) və `pubspec.yaml` (Flutter) fayllarında pin edilir
* Quarterly review ilə versiyalar yenilənir

### 3.3. Versiya nəzarəti sistemi

* GitHub istifadə olunur
* Branch strategy: GitFlow (main, develop, feature/*, hotfix/*)
* Commit mesajları Conventional Commits formatında
* Pull request məcburidir (heç bir birbaşa main branch-ə push yoxdur)
* Hər pull request daxili komanda standartı çərçivəsində ən azı bir digər developer tərəfindən code review-dan keçir

### 3.4. Source kod təhvili

Publish edilən hər versiya üçün:

* Tagged release GitHub-da yaradılır
* Source code archive (ZIP/TAR) Avrora tərəfindən təyin edilmiş mühitə kopyalanır
* Release notes hazırlanır (nə dəyişdi, hansı buglar düzəldildi)
* Deployment guide əlavə edilir

### 3.5. Texniki sənədləşmə

Hər komponentin texniki sənədi tərtib olunacaq:

| Komponent | Versiya | Lisenziya | Təyinat | Rəsmi sayt |
|-----------|---------|-----------|---------|------------|
| Django | 5.0.x | BSD | Backend framework | djangoproject.com |
| PostgreSQL | 16.x | PostgreSQL | Verilənlər bazası | postgresql.org |
| Redis | 7.x | BSD | Keş və queue | redis.io |
| Flutter | 3.24.x | BSD-3-Clause | Mobil framework | flutter.dev |
| Dart | 3.5.x | BSD-3-Clause | Mobil proqramlaşdırma dili | dart.dev |
| Nginx | 1.25.x | BSD | Web server | nginx.org |

Tam siyahı `DEPENDENCIES.md` faylında saxlanılır və hər release ilə yenilənir.

### 3.6. İctimaiyyətlə əlaqələr və İT təhlükəsizlik təsdiqi

Publish prosesi üçün checklist:

1. Kod review tamamlandı
2. Bütün testlər yaşıldır (unit, integration, security)
3. Pentest hesabatı hazırdır
4. İctimaiyyətlə əlaqələr şöbəsi UI/UX məzmununu təsdiq edib
5. İT təhlükəsizlik şöbəsi security scan nəticələrini qəbul edib
6. Yalnız bütün təsdiqlərdən sonra production deploy edilir

### 3.7. Hosting

Bütün komponentlər Avrora ilə razılaşdırılmış hosting provayderində yerləşdiriləcək. Biz deployment-i həyata keçirəcəyik, lakin server və domen Avroranın nəzarətində qalır. Server ilə bağlı bütün xərclər (hosting, bandwidth, storage, backup, monitoring xidmətləri və s.) Avrora tərəfindən ödənilir.

### 3.8. Domain və IP idarəetməsi

Bu tamamilə Avrora tərəfindən həyata keçiriləcək. Biz yalnız texniki DNS konfiqurasiyası üçün tövsiyələr verəcəyik.

### 3.9. Kod keyfiyyəti və standartları

**Kodlaşdırma standartları:**

* Python: PEP 8, Black formatter, isort import sıralaması
* Dart (Flutter): Effective Dart guidelines, `dart format` və `flutter analyze` məcburidir, `flutter_lints` rəsmi lint paketi istifadə olunur
* Maksimum funksiya uzunluğu: 50 sətir
* Maksimum sinif uzunluğu: 300 sətir
* Cyclomatic complexity maksimum: 10

**Clean Code prinsipləri:**

* Meaningful names (dəyişən və funksiya adları məqsədi açıq göstərir)
* Small functions (bir funksiya bir iş görür — Single Responsibility)
* DRY (Don't Repeat Yourself)
* SOLID prinsipləri
* Kod kommentarları yalnız "niyə" sualına cavab üçündür, "nə" kod özü göstərir

**OWASP secure coding guidelines** tam izlənilir.

### 3.10. Monitorinq və loglama

**Monitoring stack:**

* **Prometheus:** metrika toplama
* **Grafana:** dashboard və vizuallaşdırma
* **Alertmanager:** kritik hadisələr üçün bildiriş
* **Sentry:** application error tracking
* **Uptime Kuma:** uptime monitoring

**Log management:**

* Strukturlaşdırılmış JSON loglar
* **ELK Stack** (Elasticsearch + Logstash + Kibana) ilə log aqreqasiyası və axtarış
* Log-lar Avroranın göstərdiyi mühitdə saxlanılır
* Log retention: minimum 90 gün
* Sensitive məlumatlar log-a yazılmır (maskalanır)

**İzlənən metrikalar:**

* Request rate, error rate, response time (RED metrics)
* CPU, RAM, disk, network (USE metrics)
* Business metrics: sifariş sayı, uğurlu ödənişlər, axtarış sorğuları
* Təhlükəsizlik metriklaları: failed logins, blocked IPs

### 3.11. Test və Keyfiyyət Təminatı

**Unit testlər:**

* pytest (Python backend), `flutter_test` (Flutter mobil)
* Hədəf coverage: minimum 70%, kritik modullarda 90%+
* TDD yanaşması (test first, code later) mümkün olduqda

**Integration testlər:**

* API endpoint-lərinin davranışı
* Verilənlər bazası və xarici servislərlə inteqrasiya
* Django TestCase və pytest-django

**Security testlər:**

* Bandit (Python security linter)
* OWASP ZAP avtomatik scan
* Hər release öncəsi pentest

**Performans testləri:**

* JMeter ilə yük testi
* Hədəf: 95th percentile response time < 500ms
* 1000 concurrent user dəstəyi

**Manual və avtomatik testlər:**

* **Selenium** web avtomatlaşdırma üçün
* **Flutter Integration Tests** (`integration_test` paketi) mobil avtomatlaşdırma üçün
* **Patrol** Flutter üçün E2E test framework (native UI interaksiyaları daxil)
* **Postman/Newman** API testləri
* Manual QA ikmala yaxın son regression testləri üçün

**CI/CD integration:**

* Hər commit-də testlər avtomatik işə düşür
* Test uğursuzluq halında deploy blok olunur
* Nightly full regression test

---

## 4. Kritik Biznes Məntiqi Sualları

### 4.1. Race Condition (yarış vəziyyəti) qarşısı

100-lərlə adam eyni anda sonuncu məhsulu almağa çalışsa, aşağıdakı tədbirlər tətbiq olunur:

**Database-level locking:**

```python
from django.db import transaction

@transaction.atomic
def add_to_cart(user, product_id, quantity):
    # Row-level lock: digər tranzaksiyalar gözləməlidir
    product = Product.objects.select_for_update().get(id=product_id)
    
    if product.qty < quantity:
        raise InsufficientStockError()
    
    product.qty -= quantity
    product.save()
    
    OrderItem.objects.create(user=user, product=product, quantity=quantity)
```

`SELECT ... FOR UPDATE` PostgreSQL-də row-level lock yaradır. Digər eyni anda gələn sorğular birinci tranzaksiya bitənə qədər gözləyir. Bu, stok miqdarının mənfi olmamasını təmin edir.

**Alternativ yanaşma: Optimistic locking**

Hər məhsulun `version` sahəsi var. UPDATE zamanı versiya yoxlanılır, fərqli olarsa UPDATE uğursuz olur və retry edilir.

**Queue əsaslı yanaşma (çox yüksək yük üçün):**

* Flash sale kimi hallarda sifariş Redis queue-yə atılır
* Worker ardıcıl olaraq emal edir
* Yalnız stokda mövcud olanlar qəbul edilir, digərləri rədd edilir

**Redis distributed lock (Redlock):**

* Eyni məhsul üzərində əməliyyat üçün distributed lock
* Lock bir neçə Redis instansında saxlanılır
* Lock sahibi tranzaksiyanı tamamladıqda lock açılır

### 4.2. Transaction Management (tranzaksiya idarəetməsi)

Ödəniş uğurlu olduqda internet qırıldıqda sifarişin itməməsi üçün:

**İki fazalı yanaşma (Two-Phase Commit):**

1. **Faza 1:** Sifariş `PENDING_PAYMENT` statusunda DB-yə yazılır
2. **Faza 2:** Ödəniş provayderinə sorğu göndərilir
3. **Cavab:** Uğur → status `PAID`, uğursuz → status `FAILED`

**Webhook-based approach:**

Ödəniş provayderi transaction bitdikdə webhook göndərir. Müştərinin internet bağlantısından asılı deyil:

```
1. Client → "Pay" tıklanır
2. Backend → PaymentIntent yaradır, DB-yə yazır (PENDING)
3. Client → Provider-ə yönləndirilir
4. Provider → Ödəniş emal edir
5. Provider → Webhook göndərir bizim server-ə (async)
6. Backend → Webhook qəbul edir, DB-də status yeniləyir (PAID)
7. Backend → CRM-ə sinxronlaşdırır
```

**Idempotent webhook:**

* Hər webhook unikal `event_id` ilə gəlir
* Eyni event 2-ci dəfə gəlsə, ikinci dəfə işlənməz (duplicate qarşısı)

**Outbox pattern (CRM sinxronizasiyası üçün):**

1. Sifariş DB-yə yazılanda eyni tranzaksiya daxilində `OutboxEvent` cədvəlinə hadisə yazılır
2. Ayrı bir worker (Celery beat) `OutboxEvent`-ləri oxuyur və CRM-ə göndərir
3. Uğurlu göndərmədən sonra event `processed` işarələnir
4. Uğursuz olarsa retry edilir (exponential backoff ilə)

Bu yanaşma, CRM müvəqqəti olaraq əlçatmaz olsa belə, heç bir sifarişin itməməsini təmin edir.

**Retry mexanizmi:**

```python
@celery_app.task(bind=True, max_retries=5, default_retry_delay=60)
def sync_order_to_crm(self, order_id):
    try:
        order = Order.objects.get(id=order_id)
        crm_client.create_order(order.to_dict())
    except CRMConnectionError as e:
        raise self.retry(exc=e, countdown=2 ** self.request.retries)
```

### 4.3. CRM-də PII şifrələnməsi (Encryption at Rest)

Bəli, bütün şəxsi məlumatlar (PII) şifrələnmiş şəkildə saxlanılacaq.

**Çox qatlı şifrələmə:**

**Qat 1: Disk-level encryption**

* Serverdə Linux LUKS (AES-256-XTS) ilə tam disk şifrələməsi
* Fiziki disk oğurlandığı təqdirdə data oxuna bilməz

**Qat 2: Database-level encryption**

* PostgreSQL **TDE (Transparent Data Encryption)** ilə
* Backup fayllar da avtomatik şifrəli saxlanılır

**Qat 3: Column-level encryption (ən həssas sahələr)**

Django-da `django-encrypted-fields` paketi istifadə olunur:

```python
from encrypted_fields.fields import EncryptedCharField

class Customer(models.Model):
    full_name = models.CharField(max_length=255)
    voen = EncryptedCharField(max_length=50)  # AES-256 şifrəli
    passport_number = EncryptedCharField(max_length=20)
    phone = EncryptedCharField(max_length=20)
```

**Şifrələnmiş sahələr:**

* Telefon nömrəsi
* VOEN / FIN kodu
* Pasport nömrəsi
* Doğum tarixi
* Ünvan detalları
* Bank hesab nömrəsi (əgər saxlanılırsa)

**Şifrələmə açarlarının idarəsi:**

* Master key hosting provayderinin təqdim etdiyi dedicated secret management xidmətində (məsələn HashiCorp Vault) saxlanılır, bu seçim hosting mühiti ilə uzlaşdırılaraq Avrora ilə razılaşdırılacaq
* Key rotation policy: hər 90 gündən bir master key dəyişdirilir
* Data encryption key (DEK) master key ilə şifrələnir (envelope encryption)
* Heç bir şifrələmə açarı kod daxilində saxlanılmır, environment variable-lar vasitəsilə yalnız encrypted reference (key alias) ötürülür

**Axtarış və şifrələnmiş sahələr:**

Şifrələnmiş sahələrdə birbaşa LIKE axtarışı mümkün deyil. Həll:

* **Blind index** texnikası: sahənin HMAC-i yan cədvəldə saxlanılır
* Axtarış zamanı istifadəçi input-unun HMAC-i hesablanır və blind index üzrə axtarılır
* Beləliklə exact match mümkün olur, şifrələmə qorunur

**Məlumat ötürülməsi (Encryption in Transit):**

* Bütün bağlantılar TLS 1.2+ üzərindən
* DB bağlantısı SSL certificate ilə
* Internal servislər arasında mTLS (mutual TLS)

**Audit log:**

* PII sahələrinə hər bir giriş log-lanır
* Kim, nə vaxt, hansı məlumata baxdı — hər şey qeydə alınır
* Log-lar tamper-proof (immutable) saxlanılır

**GDPR/Şəxsi Məlumatlar Haqqında Qanun uyğunluğu:**

* İstifadəçi silmə tələbi (right to be forgotten) dəstəklənir
* Data export funksiyası (portability)
* Razılıq (consent) management

---

## Nəticə

Yuxarıda təsvir edilən yanaşmalar sənaye standartlarına (OWASP, PCI-DSS, ISO 27001 nüvəsi, NIST rəhbər xətləri) tam uyğundur. Layihə müasir təhlükəsizlik tələblərini qarşılayır və Azərbaycan Respublikasının "Şəxsi məlumatlar haqqında" Qanununun tələblərinə cavab verir.

Hər bölmə üzrə ətraflı texniki izah tələb olunarsa, əlavə sənədlər təqdim edilə bilər.
