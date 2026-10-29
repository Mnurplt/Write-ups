# Write-up: Blind SQL Injection with Conditional Errors (Oracle)

**Lab:** PortSwigger Web Security Academy — Blind SQL injection with conditional errors
**Veritabanı:** Oracle
**Hedef:** `administrator` kullanıcısının parolasını karakter karakter çıkarıp giriş yapmak

---

## Giriş noktası

Her istekte gönderilen `TrackingId` cookie'si, arka planda bir SQL sorgusuna ekleniyordu. Tek tırnak (`'`) ekleyince uygulama hata verdi — klasik sözdizimsel bozulma sinyali, yani bu parametre sorguya doğrudan karışıyordu.

## Bağlamı doğrulama

Oracle'da her `SELECT` bir `FROM` gerektirdiği için, test payload'larını `FROM dual` ile kapattım. `dual` ile çalışan bir payload hata vermeden döndüğünde, bunun Oracle olduğundan ve sorgu bağlamının doğru kurulduğundan emin oldum.

## Asıl payload

```
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

Bu payload'ın mantığı aslında şuna dayanıyor: `CASE` ifadesinin kendisi bir şeyi test etmiyor (`1=1` zaten her zaman doğru). Asıl test, `WHERE username='administrator'` kısmında gizli:

- `administrator` kullanıcısı varsa → `WHERE` 1 satır bırakıyor → `CASE` o satır için çalışıyor → `TO_CHAR(1/0)` tetikleniyor → **hata**
- Kullanıcı yoksa → `WHERE` 0 satır bırakıyor → `CASE` hiç çalıştırılmıyor → **hata yok**

Yani `WHERE` kapı, `CASE` ise kapıdan geçeni infilak ettiren tuzak.

Bu payload'ı denediğimde gerçekten hata aldım — `administrator` kullanıcısının var olduğu doğrulandı.

## Karakter çıkarma

Sıradaki adım, `SUBSTR()` ile parolayı pozisyon pozisyon test etmekti:

```
SUBSTR(password,N,1)='X'
```

Payload'ı Burp Intruder'a taşıdım, pozisyon numarasını sabit tutup karakteri (`X`) payload pozisyonu olarak işaretledim:

```
TrackingId=xyz'||(SELECT CASE WHEN (SUBSTR(password,1,1)='§a§') THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

**Payload listesi:** `a-z` ve `0-9` (36 karakter) — Intruder'da "Simple list" olarak verdim.

**Sinyal:** Doğru karakter denendiğinde status code `500`'e, response length `4195`'ten `2555`'e düşüyordu. Bu fark o kadar netti ki, Grep-Match bile kurmadan sadece Length sütununa bakarak doğru karakteri anında görebiliyordum.

Her pozisyonda doğru karakteri bulduktan sonra, `SUBSTR(password,N,1)` içindeki `N`'i bir artırıp taramayı yeniden başlattım. Bu şekilde parolayı şu sıraya göre çıkardım:

```
bicn57c3i11yzco... (devamı)
```

## Karşılaşılan sorun

İlerlerken bir noktada hangi pozisyonu test ettiğimi şaşırdım — Burp'ün otomatik attack numarası (13. attack gibi) ile benim takip ettiğim `SUBSTR` pozisyon numarası birbirine karışmıştı. Bunun sebebi basitti: Burp her "Start attack" tıklandığında sayacı otomatik artırıyor, bu da pozisyon numarasıyla hiç örtüşmüyor. Positions sekmesine gidip payload'daki gerçek `SUBSTR(password,X,1)` değerine bakarak, şu ana kadar bulduğum parola dizisiyle karşılaştırıp doğru yerde olduğumu teyit ettim ve devam ettim.

**Ders:** Attack sayacına değil, her zaman payload'ın kendi içindeki pozisyon değerine güvenmek gerekiyor — özellikle çok sayıda ardışık tarama yapılan uzun bir blind SQLi oturumunda karışmak çok kolay.

## Sonuç

Tüm karakterleri tek tek çıkardıktan sonra `administrator` / çıkarılan parola ile giriş yaptım ve lab "solved" durumuna geçti.

---

## Genel çıkarımlar

- **`1=1` yanıltıcı görünse de aslında hiçbir şey test etmiyor** — asıl mantık her zaman `WHERE` cümlesinde.
- **Length sütunu, bazı durumlarda Grep-Match'ten bile daha hızlı bir sinyal olabiliyor** — status code + length kombinasyonu burada çok net bir ayrım sağladı.
- **Uzun, çok adımlı blind SQLi'lerde takip sistemi şart.** Pozisyonu elle takip ederken (attack sayacı değil, payload'daki gerçek değer) bir yerde karışmak kolay — düzenli olarak "şu ana kadar bulduğum parola + şu an test ettiğim pozisyon" diye kendine checkpoint koymak işe yarıyor.
- **Oracle'a özgü `dual` + `SUBSTR` + `TO_CHAR(1/0)` üçlüsü**, bu tarz conditional-error senaryolarının temel iskeleti — MySQL'de `extractvalue`/`SLEEP`, MSSQL'de farklı bir hata tetikleyici kullanılırdı ama mantık (WHERE = kapı, hata tetikleyici = tuzak) aynı kalıyor.
