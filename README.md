# 🛡️ Katmanlı Web Güvenliği Lab

Kasıtlı olarak zafiyetli bir web uygulamasının etrafına kurulmuş, uçtan uca **katmanlı güvenlik (defense-in-depth)** laboratuvarı. Amaç, bir güvenlik mimarisini gerçek bir üretim ortamı gibi kurmak: her OWASP Top 10 (2025) zafiyetini önce **sömürmek**, sonra hem **kod** hem de **WAF** düzeyinde **kapatmak**, ve saldırıları bir **SIEM** üzerinden izlemek.

> ⚠️ **Uyarı:** Bu uygulama eğitim amaçlı *kasıtlı zafiyetlidir*. Yalnızca izole bir yerel ortamda çalıştırın; internete açık bir sunucuda veya üretimde ASLA kullanmayın.

---

## İçindekiler

- [Mimari](#mimari)
- [Kullanılan Bileşenler](#kullanilan-bilesenler)
- [OWASP Top 10 (2025) Kapsamı](#owasp-top-10-2025-kapsami)
- [Saldırılar — Detaylı Anlatım](#saldirilar--detayli-anlatim)
- [STRIDE Tehdit Modeli](#stride-tehdit-modeli)
- [Sızma Testi Raporu](#sizma-testi-raporu)
- [Ekran Görüntüleri](#ekran-goruntuleri)
- [Kurulum](#kurulum)
- [Dizin Yapısı](#dizin-yapisi)

---

## Mimari

Trafik dıştan içe altı katmandan geçer. Her istek önce kenardaki (edge) Caddy'ye gelir; oradan arkadaki uygulamaya iletilir.

```
Internet
   |  HTTPS (TLS)
   v
+-----------------------------------------------+
|  Caddy (edge)                                 |
|  TLS - Guvenlik basliklari (HSTS/CSP/...)     |
|  CrowdSec bouncer - Coraza WAF (OWASP CRS)    |
+------------------------+----------------------+
                         |  (internal)
                         v
             +-------------------------+
             |  Flask app (gunicorn)   |
             |  Kod savunmalari +      |
             |  rate limit             |
             +-----------+-------------+
                         |  loglar (internal)
                         v
            Alloy  ->  Loki  ->  Grafana  (SIEM)
```

**Ağ segmentasyonu:** `edge` (Caddy ↔ dış dünya), `backend` (Caddy ↔ app, internal), `monitor` (log yığını, internal). Uygulama ve log konteynerleri internete doğrudan açık değildir — yalnızca Caddy üzerinden erişilir.

**Temel ilke:** Her zafiyet **iki yerde** kapatıldı — kod (asıl düzeltme) + WAF (edge kalkanı). WAF devre dışı kalsa bile kod savunması tutar.

---

## Kullanılan Bileşenler

### Ters Proxy (Reverse Proxy) — Caddy
Dış dünya ile uygulamanın arasına giren **kapı görevlisi**. Tüm istekler önce ona gelir, o da arkadaki uygulamaya iletir. Uygulamayı doğrudan internete açmazsın; araya bir kontrol noktası koyarsın. Caddy ayrıca TLS sertifikasını otomatik yönetir (HTTPS), güvenlik başlıkları ekler, WAF ve CrowdSec'i barındırır.

### WAF (Web Application Firewall) — Coraza + OWASP CRS
WAF, **web uygulama güvenlik duvarı**. Normal firewall IP/port seviyesinde çalışır; WAF ise HTTP isteğinin *içeriğine* bakar — URL, parametreler, gövde. İçinde saldırı imzası görürse (ör. `' OR '1'='1`, `<script>`, `../../etc/passwd`) isteği keser.
- **Coraza:** açık kaynaklı, ModSecurity uyumlu WAF motoru; Caddy'ye gömülü çalışır.
- **OWASP CRS (Core Rule Set):** Coraza'nın kullandığı hazır kural seti — binlerce bilinen saldırı paternini tanır.
- **İki mod:** `On` (eşleşen isteği engeller → 403) ve `DetectionOnly` (engellemez, sadece loglar). Test ederken DetectionOnly kullanıldı ki payload uygulamaya ulaşsın ve asıl kod savunması sınansın.
- **Neden tek başına yetmez?** WAF imza tabanlıdır; yeni/özel saldırıyı kaçırabilir. O yüzden zafiyetler kodda da kapatıldı (defense in depth).

### CrowdSec — davranışsal IP engelleme
WAF tek isteğe bakar; CrowdSec **davranışa** bakar. Aynı IP 1 dakikada 50 başarısız giriş denerse bu bir saldırı paternidir — CrowdSec o IP'yi engeller (bouncer aracılığıyla). WAF *ne* sorusuna, CrowdSec *kim nasıl davranıyor* sorusuna bakar.

### Oran Sınırlama (Rate Limiting) — Flask-Limiter
Bir rotaya belirli sürede kaç istek atılabileceğini sınırlar (ör. `/login` → 5/dakika). Kaba kuvvet parola denemesini durdurur.

### SIEM — Loki + Alloy + Grafana
Güvenlik olaylarını tek yerde toplayıp izleme sistemi.
- **Alloy:** log toplayıcı/taşıyıcı. Caddy ve WAF loglarını okur, Loki'ye yollar.
- **Loki:** log deposu ve sorgu motoru (LogQL ile sorgulanır).
- **Grafana:** gösterge paneli; Loki'den gelen veriyi grafik ve panel olarak çizer.

Bir SOC analisti tam da bu ekrana bakar: kaç saldırı engellendi, hangi IP, hangi saat.

### Konteyner Sertleştirme (Container Hardening)
"Biri içeri girerse ne kadar zarar verebilir?" sorusuna karşı konteyner daraltılır:
- **non-root (uid 10001):** uygulama root değil, sınırlı kullanıcı.
- **read_only dosya sistemi:** saldırgan dosya yazamaz (backdoor koyamaz); sadece `/tmp` geçici yazılabilir.
- **cap_drop: ALL:** Linux yeteneklerini sıfırlar.
- **no-new-privileges:** yetki yükseltme yolu kapalı.
- **bellek limiti:** tek konteyner tüm sunucuyu tüketemez.

---

## OWASP Top 10 (2025) Kapsamı

| # | Kategori | Saldırı | Savunma | Durum |
|---|---|---|---|---|
| A01 | Broken Access Control (+SSRF) | IDOR, SSRF | Sahiplik kontrolü, URL allowlist | ✅ |
| A02 | Security Misconfiguration | Başlık/debug denetimi | Güvenlik başlıkları + CSP, Server gizli, debug kapalı | ✅ |
| A03 | Software Supply Chain Failures | Bağımlılık riski | Pin'li imajlar, minimal bağımlılık | ◻ Kavramsal |
| A04 | Cryptographic Failures | Düz metin/zayıf saklama | TLS (HSTS), parola hash | ✅ |
| A05 | Injection | SQLi, XSS, SSTI, Command Injection | Parametreli sorgu, escape + CSP, sabit şablon, arg-list | ✅ |
| A06 | Insecure Design | Tasarım eksikliği | Tehdit modelleme, katmanlı savunma, en az yetki | ✅ Tasarımsal |
| A07 | Authentication Failures | Kaba kuvvet, zayıf parola | Rate limit (5/dk), parola politikası | ✅ |
| A08 | Software/Data Integrity Failures | Insecure deserialization (pickle RCE) | JSON'a geçiş | ✅ |
| A09 | Logging & Alerting Failures | Görünürlük eksikliği | Loki/Grafana SIEM, WAF audit | ✅ |
| A10 | Mishandling of Exceptional Conditions | Hata ile bilgi sızması | Global 404/500 handler, fail-closed | ✅ |

---

## Saldırılar — Detaylı Anlatım

Tüm enjeksiyonların ortak mantığı: **kullanıcı girdisinin "veri" olması gerekirken "kod/komut" olarak çalıştırılması.** Çözüm hep aynı refleks: girdiyi veri olarak tut, asla kod gibi yorumlatma.

### SQL Injection (SQLi)
- **Nedir:** Kullanıcı girdisi SQL sorgusuna doğrudan yapıştırılırsa, saldırgan sorgunun mantığını değiştirir.
- **Nasıl çalışır:** `/ara?q=' OR '1'='1` → sorgu `... WHERE baslik = '' OR '1'='1'` olur; `'1'='1'` her zaman doğru, tüm kayıtlar döner.
- **Yama:** Parametreli sorgu — `cursor.execute("... WHERE ad = ?", (q,))`. Girdi sorgu metnine karışmaz, salt *değer* olur.
- **Neden tutar:** Sorgunun iskeleti ile verisi ayrışır; payload düz metin olarak aranır.

### XSS (Cross-Site Scripting)
- **Nedir:** Saldırganın yazdığı `<script>` başka kullanıcının tarayıcısında çalışır (çerez/oturum çalma).
- **Reflected:** `/ara?q=<script>alert(1)</script>` sayfaya olduğu gibi basılırsa çalışır. **Stored:** script veritabanına kaydedilir, sayfayı açan herkeste çalışır.
- **Yama:** Çıktı escape (`markupsafe.escape`) — `<` → `&lt;`, tarayıcı metin görür. Ek olarak **CSP başlığı** inline script'i tarayıcı seviyesinde engeller.
- **Neden tutar:** İki katman — kod escape eder, tarayıcı CSP ile inline script'i reddeder.

### SSTI (Server-Side Template Injection)
- **Nedir:** Girdi şablon motoruna (Jinja2) kod olarak girerse sunucuda çalışır — RCE'ye kadar gider.
- **Nasıl çalışır:** Girdi şablon metnine gömülürse `{{7*7}}` → `49` olur; bu, girdinin *çalıştırıldığının* kanıtıdır.
- **Yama:** Girdiyi şablonun metnine değil **değişkenine** geçirmek — `render_template_string("Selam {{ad}}", ad=ad)`.
- **Neden tutar:** `{{7*7}}` değer olarak basılır, işlenmez; düz metin kalır.

### Komut Enjeksiyonu (OS Command Injection)
- **Nedir:** Girdi işletim sistemi komutuna karışırsa, saldırgan ek komut çalıştırır.
- **Nasıl çalışır:** `os.system("ping " + host)` ve `host=127.0.0.1; id` → `ping 127.0.0.1; id` çalışır.
- **Yama:** `subprocess.run(["ping","-c","1", host])` — argüman **listesi**, shell yok. Ayrıca host regex ile doğrulandı.
- **Neden tutar:** Shell devrede olmayınca `;` özel anlamını yitirir; girdi tek argüman olarak geçer.

### IDOR (Insecure Direct Object Reference)
- **Nedir:** Erişim kontrolü olmadan, sadece `id` değiştirilerek başkasının verisine ulaşmak.
- **Nasıl çalışır:** `/indir/5` senin dosyan; `/indir/6` yaparsan başkasının dosyası iner — çünkü "bu dosya bu kullanıcının mı?" sorulmuyor.
- **Yama:** Sahiplik kontrolü — `dosya.yazar_id == oturum.kullanici_id`; değilse 403.
- **Neden tutar:** Artık id tahmini yetmez; sistem her erişimde yetki sorar.

### SSRF (Server-Side Request Forgery)
- **Nedir:** Sunucuyu, saldırganın istediği bir adrese **istek atan elçi** yapmak. Hedef genelde iç ağ veya bulut metadata servisi.
- **Nasıl çalışır:** "Avatar URL'si gir" özelliğine `http://169.254.169.254/latest/meta-data/` verirsen, sunucu senin yerine oraya gider ve dönen gizli veriyi getirir.
- **Elçi kim?** Kendi uygulama sunucun — URL'yi o çekiyor. Dışarıdan erişemediğin iç kaynaklara sunucu "içeriden" eriştiği için kötüye kullanılır.
- **Yama:** URL allowlist — sadece `http/https`, ve **özel IP bloklarını** (`127.0.0.1`, `169.254.x`, `10.x`, `192.168.x`) reddet.
- **Neden tutar:** İç hedeflere giden istekler gönderilmeden engellenir.

### Insecure Deserialization
- **Nedir:** Güvenilmez veriyi deserialize ederken (ör. Python `pickle`) kod çalışması — doğrudan RCE.
- **Nasıl çalışır:** `pickle.loads(kullanici_verisi)` — pickle, nesneyi kurarken içindeki kodu çalıştırabilir.
- **Yama:** `pickle` yerine `json.loads` — JSON sadece veri taşır, kod çalıştırmaz.
- **Neden tutar:** JSON'da "çalıştırılabilir kod" kavramı yok; en kötü ihtimalle parse hatası alırsın.

---

## STRIDE Tehdit Modeli

| STRIDE | Tehdit | Önlem |
|---|---|---|
| **S**poofing | Oturum/parola ele geçirme, kaba kuvvet | Parola politikası, rate limit, CrowdSec IP ban |
| **T**ampering | SQLi, girdi manipülasyonu | Parametreli sorgular, WAF/CRS, girdi doğrulama |
| **R**epudiation | İşlemi inkar, iz bırakmama | Merkezî loglama (Loki), WAF audit log |
| **I**nformation Disclosure | Stack trace, IDOR, SSRF, header sızıntısı | Genel hata sayfaları, sahiplik kontrolü, SSRF allowlist, Server gizli |
| **D**enial of Service | İstek seli | Rate limit, CrowdSec, konteyner kaynak limitleri |
| **E**levation of Privilege | RCE, komut enjeksiyonu, deserialization | subprocess arg-list, JSON, non-root + read-only FS + cap_drop |

---

## Sızma Testi Raporu

WAF iki modda doğrulandı: `DetectionOnly` (kod savunmasını izole etmek için) ve `On` (edge kalkanı).

| # | Zafiyet | Payload / Yöntem | Beklenen | Sonuç |
|---|---|---|---|---|
| 1 | SQLi | `/ara?q=' OR '1'='1` | Kayıt dökülmez | ✅ |
| 2 | Reflected XSS | `/ara?q=<script>...` | Script çalışmaz (escape + CSP) | ✅ |
| 3 | SSTI | `/selam?ad={{7*7}}` | Düz metin, 49 değil | ✅ |
| 4 | IDOR | `/indir/<başka id>` | 403 | ✅ |
| 5 | Komut enjeksiyonu | `host=127.0.0.1; id` | Reddedilir | ✅ |
| 6 | SSRF | `url=http://169.254.169.254/...` | Engellenir | ✅ |
| 7 | Insecure deserialization | pickle verisi | Hata yakalanır | ✅ |
| 8 | Güvenlik başlıkları (A02) | `curl -I` | HSTS/CSP/X-Frame var, Server yok | ✅ |
| 9 | Hata yönetimi (A10) | olmayan sayfa | Temiz 404, stack trace yok | ✅ |
| 10 | Rate limit (A07) | `/login` POST ×10 | 5 denemeden sonra 429 | ✅ (401×5, 429×5) |

---

## Ekran Görüntüleri

| WAF saldırıyı engelliyor (403) | Grafana SIEM panosu |
| --- | --- |
| ![WAF](screenshots/waf-block.png) | ![SIEM](screenshots/grafana.png) |

| Uygulama (normal akış) | Tüm yığın ayakta (docker compose ps) |
| --- | --- |
| ![Site](screenshots/site.png) | ![Containers](screenshots/docker-ps.png) |

---

## Kurulum

```bash
git clone https://github.com/mahirkau1/guvenlik-lab.git
cd guvenlik-lab
cp .env.ornek .env        # kendi gizli degerlerini gir
docker compose up -d --build
```

---

## Dizin Yapısı

```
guvenlik-lab/
├── app/               # Flask uygulamasi
├── caddy/             # Caddyfile + WAF yapilandirmasi
├── alloy/             # log toplayici
├── grafana/           # datasource + panolar
├── screenshots/       # README gorselleri
├── docker-compose.yml
├── .env.ornek         # ornek env (gercek sir yok)
└── .env               # GIZLI — repoda yok
```

---

## Güvenlik Notu

Gizli anahtarlar (`SECRET_KEY`, API anahtarları, Grafana parolası) `.env` içinde tutulur ve `.gitignore` ile repodan dışlanır. Seed kullanıcı parolaları yalnızca demo amaçlıdır.

---

**Hazırlayan:** Kaan Budak — SOC L1 → Güvenlik Mühendisliği hedefi
