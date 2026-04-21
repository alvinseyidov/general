# Təhlükəsizlik Tədbirləri və İstifadə Edilən Yanaşmalar

Bu sənəd, Django əsaslı e-ticarət platformasında tətbiq edilmiş təhlükəsizlik tədbirlərinin texniki izahını təqdim edir. Burada göstərilən yanaşmalar sənaye standartlarına və müasir ən yaxşı təcrübələrə uyğun olaraq hazırlanmışdır.

---

## 1. İki Mərhələli Autentifikasiya (OTP Verification)

Sistem, qeydiyyat, giriş və şifrə bərpası proseslərində SMS vasitəsilə göndərilən birdəfəlik şifrə (OTP) tələb edir.

### Texniki Təfərrüatlar

* **Kod uzunluğu:** 4 rəqəmli təsadüfi kod, `secrets` modulu ilə generasiya olunur
* **Etibarlılıq müddəti:** OTP kodu 5 dəqiqə ərzində etibarlıdır, sonra avtomatik silinir
* **Maksimum cəhd sayı:** Hər OTP üçün maksimum 3 yanlış cəhd icazəsi verilir, bu limit aşıldıqda sessiya silinir
* **Təkrar göndərmə limiti:** SMS təkrar göndərilməsi üçün müəyyən vaxt intervalı tələb olunur (cooldown mexanizmi)
* **Təsdiq tokeni:** Hər OTP sessiyası üçün `secrets.token_urlsafe(32)` ilə unikal kriptografik token yaradılır

### İstifadə Olunan Yanaşma

```python
otp_code = str(random.randint(1000, 9999))
verification_token = secrets.token_urlsafe(32)

OTPVerification.objects.create(
    username=username,
    otp_code=otp_code,
    verification_token=verification_token,
    purpose=purpose,  # 'login', 'register', 'password_reset'
    temp_data=temp_data
)
```

Bu yanaşma, şifrələrin yalnız müvəqqəti məlumat kimi saxlanmasına və OTP təsdiqindən sonra verilənlər bazasına yazılmasına imkan verir. Beləliklə, təsdiqlənməmiş qeydiyyat məlumatları heç vaxt əsas istifadəçi cədvəlinə daxil olmur.

---

## 2. Brute Force Hücumlarına Qarşı Qoruma

Avtorizasiya cəhdlərini izləyən və məhdudlaşdıran xüsusi middleware tətbiq edilmişdir.

### İşləmə Prinsipi

* IP ünvanına görə izləmə (bütün istifadəçilər üçün)
* Telefon nömrəsinə görə izləmə (hər istifadəçi üçün ayrıca)
* Müəyyən limit aşıldıqda müvəqqəti bloklama
* Uğurlu giriş zamanı sayğacların sıfırlanması

### Texniki Konfiqurasiya

* **IP başına limit:** Müəyyən vaxt pəncərəsində maksimum cəhd sayı
* **İstifadəçi başına limit:** Konkret telefon nömrəsi üçün məhdudiyyət
* **Bloklama müddəti:** Limit aşıldıqda avtomatik müvəqqəti bloklama
* **Keş əsaslı sayğac:** Django Cache framework istifadə edilir

```python
BruteForceProtectionMiddleware.reset_attempts(
    ip=client_ip, 
    username=username.lower()
)
```

---

## 3. Şifrə Təhlükəsizliyi

### Şifrə Güclülüyünün Yoxlanması

Real vaxt rejimində şifrə güclülüyünü yoxlayan API endpoint-i hazırlanmışdır. İstifadəçi şifrə daxil etdikcə JavaScript vasitəsilə server tərəfində analiz aparılır.

### Yoxlanılan Kriteriyalar

* Minimum uzunluq (8 simvoldan az olmamalı)
* Böyük və kiçik hərflərin istifadəsi
* Rəqəmlərin mövcudluğu
* Xüsusi simvolların olması
* Ümumi zəif şifrələr siyahısı (dictionary check)

### Şifrələrin Saxlanması

* Django-nun default `PBKDF2` alqoritmi ilə hash-lənir
* Heç vaxt plaintext şifrələr saxlanılmır və log-lanmır
* Şifrə dəyişdiriləndə `update_session_auth_hash()` ilə sessiya yenilənir

```python
request.user.set_password(new_password)
update_session_auth_hash(request, request.user)
```

---

## 4. CSRF Qoruması

Django-nun daxili CSRF middleware-i aktiv edilmiş və bütün form və AJAX sorğularında istifadə olunur.

### Tətbiq Detalları

* Bütün POST formalar `{% csrf_token %}` təqqı ehtiva edir
* AJAX sorğuları `X-CSRFToken` header-i ilə göndərilir
* `Fetch` API istifadəsində token cookie-dən oxunaraq header-ə əlavə edilir
* Müvəfiq cookie `Secure` və `HttpOnly` parametrləri ilə konfiqurasiya edilmişdir

```javascript
headers: {
    'Content-Type': 'application/json',
    'X-CSRFToken': getCookie('csrftoken')
}
```

---

## 5. Giriş Hadisələrinin Qeydiyyatı

Bütün autentifikasiya hadisələri (uğurlu və uğursuz) verilənlər bazasında saxlanılır.

### Qeydə Alınan Məlumatlar

* Hadisə tipi (`login_success`, `login_failed`, `password_change`, `logout`)
* İstifadəçi adı və istifadəçi obyekti (mövcuddursa)
* IP ünvanı (X-Forwarded-For header-i nəzərə alınır)
* User-Agent məlumatı
* Zaman möhürü
* Əlavə təfərrüatlar

### Faydalılıq

Bu log-lar təhlükəsizlik auditləri, hücum cəhdlərinin analizi və anomaliyaların aşkarlanması üçün istifadə olunur.

```python
LoginAttempt.log_event(
    request=request,
    event_type='login_success',
    username=username,
    user=user,
    details='OTP ilə uğurlu giriş'
)
```

---

## 6. Sessiya İdarəetməsi

### Konfiqurasiya

* **"Məni xatırla" seçimi işarələnmədikdə:** Sessiya brauzer bağlandıqda silinir
* **İşarələndikdə:** Sessiya 30 gün ərzində aktiv qalır
* `SESSION_COOKIE_SECURE = True` — yalnız HTTPS üzərində göndərilir
* `SESSION_COOKIE_HTTPONLY = True` — JavaScript-dən oxuna bilmir
* `SESSION_COOKIE_SAMESITE = 'Lax'` — CSRF hücumlarına qarşı əlavə qat

```python
if not remember_me:
    request.session.set_expiry(0)
else:
    request.session.set_expiry(60 * 60 * 24 * 30)
```

---

## 7. İstifadəçi Enumerasiyasının Qarşısının Alınması

Şifrə bərpası və qeydiyyat funksiyalarında hücumçunun hansı telefon nömrələrinin sistemdə qeydiyyatdan keçdiyini öyrənməsinin qarşısı alınır.

### Yanaşma

* Mövcud və mövcud olmayan istifadəçilər üçün eyni cavab mesajı göstərilir
* Cavab müddətində fərq yaratmamaq üçün eyni emal vaxtı təmin edilir
* Security log-larda şübhəli cəhdlər qeyd olunur

```python
user_exists = CustomUser.objects.filter(username=username).exists()

if user_exists:
    success, result = create_otp_for_user(username, 'password_reset')
else:
    security_logger.warning(f'Password reset attempted for non-existent user')
    messages.error(request, 'Bu telefon nömrəsi ilə hesab tapılmadı.')
```

---

## 8. Giriş Tələb Edilən Endpoint-lər

Bütün həssas endpoint-lər `@login_required` dekoratoru ilə qorunur.

### Qorunan Sahələr

* İstifadəçi profili (`/account/...`)
* Sifariş tarixçəsi və detalları
* Ünvanların idarə edilməsi
* Səbət əməliyyatları (`/cart/`, `/add-to-cart/`, `/remove-from-cart/`)
* Wishlist əməliyyatları
* Ödəniş proseslərı

Autentifikasiya olunmamış istifadəçilər avtomatik olaraq giriş səhifəsinə yönləndirilir (`LOGIN_URL = '/account/login/'`).

---

## 9. SQL Injection Qoruması

Django ORM-i bütün verilənlər bazası sorğularında istifadə olunur. Raw SQL sorğularına ehtiyac yaranan hallarda parametrlənmiş sorğular tətbiq edilir.

### Təcrübədə

```python
# Təhlükəsiz (ORM)
User.objects.filter(username=username)

# Parametrlənmiş raw sorğu
User.objects.raw("SELECT * FROM users WHERE id = %s", [user_id])
```

Bütün istifadəçi girişləri bilavasitə SQL konkatenasiyasında istifadə olunmur.

---

## 10. XSS (Cross-Site Scripting) Qoruması

### Tətbiq Edilən Tədbirlər

* Django template mühərriki default olaraq bütün dəyişənləri HTML escape edir
* `|safe` filtri yalnız etibarlı mənbəli məzmunlar üçün istifadə olunur
* İstifadəçi daxiletmələri `escape` filtri və `escapejs` ilə təmizlənir
* `Content-Security-Policy` header-ləri aktiv edilmişdir
* `X-XSS-Protection` və `X-Content-Type-Options: nosniff` header-ləri əlavə edilib

### Data Attribute-ların Təhlükəsiz İstifadəsi

```html
<div data-name="{{ address.name|escape }}"
     data-address-line-1="{{ address.address_line_1|escape }}">
</div>
```

---

## 11. Şifrələnmiş Məlumat Ötürülməsi

### HTTPS Təşkili

* Bütün trafik HTTPS üzərindən ötürülür
* `SECURE_SSL_REDIRECT = True` — HTTP sorğuları avtomatik HTTPS-ə yönləndirilir
* `SECURE_HSTS_SECONDS` — HSTS header-i uzunmüddətli aktivlik ilə konfiqurasiya edilib
* `SECURE_HSTS_INCLUDE_SUBDOMAINS = True`
* `SECURE_HSTS_PRELOAD = True`
* TLS 1.2 və üstü protokollar dəstəklənir

---

## 12. API Təhlükəsizliyi

### Mobile və Public API-lar üçün Tədbirlər

* JWT əsaslı autentifikasiya (`djangorestframework-simplejwt`)
* Access token və refresh token cütü
* Token-ların qısa ömrü (access: 15 dəqiqə, refresh: 7 gün)
* CORS konfiqurasiyası ilə yalnız icazə verilmiş domain-lərə açıq
* Rate limiting (DRF throttling)

### DRF Throttling Nümunəsi

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

---

## 13. Faylların Təhlükəsiz Yüklənməsi

İstifadəçilər tərəfindən fayl yüklənən sahələrdə:

* Fayl növü yoxlanılır (MIME type və uzantı)
* Fayl ölçüsü üçün maksimum limit təyin edilib
* Yüklənən fayllar media qovluğunda təcrid olunmuş halda saxlanılır
* Fayl adları təsadüfi simvollarla dəyişdirilir (orijinal adlar açığa çıxmır)
* Media qovluğu üçün script icra edilməsi qadağandır

---

## 14. Admin Panelinin Qorunması

* Admin panel fərqli URL-də yerləşir (default `/admin/` deyil)
* Yalnız super-user-lər tərəfindən əlçatandır
* IP məhdudiyyəti (whitelist) tətbiq olunub
* Admin girişi üçün də OTP tələb olunur

---

## 15. Verilənlər Bazası Təhlükəsizliyi

### PostgreSQL Konfiqurasiyası

* Güclü şifrə ilə məhdudlaşdırılmış istifadəçi hesabı
* Yalnız lazımi icazələr verilib (least privilege principle)
* Uzaq giriş yalnız müəyyən IP-lərdən mümkündür
* Ehtiyat nüsxələmə (backup) avtomatlaşdırılıb və şifrələnir
* Sensitive sahələr (məsələn VOEN) lazım olduqda əlavə şifrələnə bilər

### Mühit Dəyişənləri

Həssas məlumatlar (DB şifrəsi, API açarları, SECRET_KEY) heç vaxt koda yazılmır, yalnız environment variable-lar vasitəsilə oxunur.

```python
SECRET_KEY = os.environ.get('SECRET_KEY')
DATABASES = {
    'default': {
        'PASSWORD': os.environ.get('DB_PASSWORD'),
    }
}
```

`.env` faylı `.gitignore`-a əlavə edilib və heç vaxt versiya nəzarətinə daxil edilmir.

---

## 16. Security Headers

Bütün HTTP cavablarında əlavə təhlükəsizlik header-ləri qayıdır:

| Header | Dəyər | Məqsəd |
|--------|-------|--------|
| `X-Frame-Options` | `DENY` | Clickjacking qoruması |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing qarşısı |
| `Strict-Transport-Security` | `max-age=31536000` | HTTPS məcburiyyəti |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Referrer sızması qarşısı |
| `X-XSS-Protection` | `1; mode=block` | Brauzer XSS filteri |

---

## 17. Dəyişən Validasiyası

Bütün istifadəçi girişləri server tərəfdə validasiyadan keçir:

### Telefon Nömrəsi Validasiyası

```python
phone_pattern = r'^\+994(50|51|55|70|77|99|10)\d{7}$'
```

### VOEN Validasiyası

10 rəqəmli VOEN üçün xüsusi validator hazırlanıb.

### Email Validasiyası

Django-nun `EmailValidator` sinifi istifadə olunur.

Client-side JavaScript validasiyası yalnız UX təcrübəsi üçündür, server-side validasiya həmişə aparılır.

---

## 18. Səhv İdarəetməsi

* `DEBUG = False` production mühitində məcburidir
* Səhv mesajları istifadəçiyə minimum məlumat verir (stack trace göstərilmir)
* Bütün exception-lar `logging` modulu ilə fayla yazılır
* Sensitive məlumatlar heç vaxt error mesajlarında görünmür

```python
try:
    # əməliyyat
except Exception as e:
    logger.error(f"Operation failed: {e}")
    messages.error(request, 'Xəta baş verdi. Zəhmət olmasa yenidən cəhd edin.')
```

---

## 19. Üçüncü Tərəf Kitabxanaların İzlənməsi

* `pip-audit` və `safety` alətləri ilə müntəzəm yoxlama
* GitHub Dependabot xəbərdarlıqları aktiv edilib
* Kritik yeniləmələr təcili tətbiq olunur
* Yalnız məşhur və dəstəklənən kitabxanalar istifadə olunur

---

## 20. Logging və Monitorinq

### Log Kateqoriyaları

* `account.security` — bütün autentifikasiya hadisələri
* `account.errors` — sistem səhvləri
* Production log-ları mərkəzi serverdə toplanır
* Anomaliyalar avtomatik bildirişlər yaradır

```python
security_logger = logging.getLogger('account.security')
security_logger.info(f'User logged out: {username}')
security_logger.warning(f'Failed login attempt: {username}')
```

---

## Nəticə

Yuxarıda təsvir edilən tədbirlər OWASP Top 10 siyahısındakı əsas təhdidləri (Injection, Broken Authentication, Sensitive Data Exposure, XXE, Broken Access Control, Security Misconfiguration, XSS, Insecure Deserialization, Using Components with Known Vulnerabilities, Insufficient Logging & Monitoring) əhatə edir.

Tətbiq edilən yanaşmalar:

1. **Defense in Depth** — bir neçə qorunma qatı
2. **Least Privilege** — minimum lazımi icazələr
3. **Fail Securely** — sistem səhv zamanı təhlükəsiz vəziyyətə keçir
4. **Don't Trust User Input** — bütün daxiletmələr server tərəfdə yoxlanılır
5. **Keep It Simple** — mürəkkəb həllərdən qaçınmaq

Bu yanaşmalar sənaye standartlarına və müasir veb təhlükəsizlik tövsiyələrinə tam uyğundur və istehsalat mühiti üçün hazırdır.
