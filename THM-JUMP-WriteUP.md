# Privilege Escalation Chain: FTP'den Root'a

> **Not:** Bu test [Try Hack ME/ JUMP Machine] üzerinde gerçekleştirilmiştir.

## Özet

Bu yazıda, açık bir FTP servisinden başlayarak dört farklı kullanıcı hesabı üzerinden root yetkisine ulaştığımız bir privilege escalation zincirini adım adım inceliyoruz:

```
recon_user → dev_user → monitor_user → ops_usr → root
```

Zincirin her halkası, farklı ama birbiriyle ilişkili bir izin/konfigürasyon hatasından besleniyor: denetimsiz otomasyon, yanlış dosya sahiplikleri, PATH hijacking ve son olarak klasik bir GTFOBins zafiyeti.

---

## 1. Initial Recon & Foothold (Anonymous FTP → recon_user)

```bash
nmap -sV <target_IP>
```

Tarama sonucunda **21 (FTP)** ve **22 (SSH)** portlarının açık olduğu tespit edildi. Henüz geçerli bir kimlik bilgisi bulunmadığından brute-force ilk aşamada tercih edilmedi. FTP servisine **anonymous** olarak erişim sağlandı ve dizin incelemesinde bir `README.txt` dosyası bulundu:

> *"All recon jobs must be placed in incoming/. Files are processed automatically on arrival. Invalid formats are ignored."*

Mekanizmayı doğrulamak için içeriği `echo 'successful'` olan bir `.sh` dosyası `incoming/` dizinine yüklendi ve arka plandaki servisin bunu otomatik işlediği gözlemlendi. Ardından bir Bash reverse shell payload'u yüklenip yerel makinede `ncat` listener başlatıldı. Otomasyon dosyayı tetikleyince **recon_user** olarak ilk erişim elde edildi.

**Root cause:** FTP servisinin anonim yazma erişimine açık olması ve `incoming/` dizinine yüklenen dosyaların **hiçbir doğrulama yapılmadan doğrudan çalıştırılması**. Otomasyonun, kaynağı doğrulanmamış girdiyi güvenilir kabul etmesi bu zincirin temel nedeni.

---

## 2. Privilege Escalation to dev_user

```bash
find / -type f -writable -user dev_user -not -path "/proc/*" 2>/dev/null
systemctl list-timers
```

`/opt/scripts/backup.sh` dosyasının `dev_user` yetkileriyle düzenli çalışan bir cron job/timer olduğu ve bu dosyaya yazma iznimizin bulunduğu tespit edildi. Script içine reverse shell payload'u enjekte edilerek **dev_user** yetkisi elde edildi.

**Root cause:** Bir kullanıcıya ait olup düzenli olarak çalıştırılan bir dosyanın, **başka bir kullanıcı tarafından yazılabilir** olması. Bir servis/scheduled job çalıştırılıyorsa, o dosyaya (doğrudan ya da dolaylı yoldan) başka hiçbir hesabın yazma erişimi olmamalı — aksi halde dosyanın sahibi olan hesabın yetkisi devralınabilir.

---

## 3. Privilege Escalation to monitor_user (PATH Hijacking)

```bash
ps -u monitor_user
```

`monitor_user` tarafından periyodik çalıştırılan bir healthcheck servisi tespit edildi. Servisin çağırdığı komutlar incelendiğinde, bağımlılıklardan birinin **tam yol (absolute path) belirtilmeden, yalnızca komut adıyla** çağrıldığı görüldü. Bu durum klasik bir **PATH hijacking** zafiyeti oluşturuyordu: yazılabilir bir dizin `$PATH` içinde öncelikliyse, gerçek binary yerine kötü amaçlı bir dosya o isimle oraya yerleştirilip çalıştırılabilir. Bu yöntemle payload tetiklenerek **monitor_user** yetkisine geçildi.

**Root cause:** healthcheck servisinin çağırdığı komutun **mutlak yol yerine yalnızca isimle** çağrılması ve `$PATH` içinde yazılabilir bir dizinin bulunması. Servisler her zaman bağımlılıklarını tam yol ile çağırmalı.

---

## 4. Privilege Escalation to ops_usr

```bash
sudo -l
cat /usr/local/bin/deploy.sh
```

`ops_usr`'ın `/usr/local/bin/deploy.sh` dosyasını sudo ile çalıştırma yetkisi olduğu görüldü. Dosya incelendiğinde, script'in `/opt/app/deploy_helper.sh` dosyasını çağırdığı ve bu dosyanın sahibinin `monitor_user` (bizim önceki adımda ele geçirdiğimiz kullanıcı) olduğu tespit edildi. `deploy_helper.sh` üzerine payload yüklenip:

```bash
sudo -u ops_usr /usr/local/bin/deploy.sh
```

komutu çalıştırılarak **ops_usr** hesabına erişim sağlandı.

**Root cause:** `ops_usr`'ın sudo yetkisiyle çalıştırdığı bir script'in, **başka bir kullanıcıya ait ve o kullanıcı tarafından yazılabilir** bir yardımcı dosyayı (`deploy_helper.sh`) çağırması. Bir sudo yetkisi zincirindeki tüm ara dosyaların sahiplik/yazma izinleri de o zincirin güvenlik sınırının parçasıdır.

---

## 5. Privilege Escalation to Root via `less` (GTFOBins)

```bash
sudo -l
# (Örnek çıktı)
# User ops_usr may run the following commands on target:
#     (root) NOPASSWD: /usr/bin/less
```

`ops_usr` yetkisiyle çalıştırılan `sudo -l` çıktısında, `less` binary'sinin **şifresiz (NOPASSWD)** olarak root yetkisiyle çalıştırılabildiği görüldü. [GTFOBins](https://gtfobins.github.io/gtfobins/less/) üzerinde `less` için tanımlı shell escape tekniği kullanılarak root shell elde edildi:

```bash
sudo less /etc/hosts
# less içerisindeyken:
!/bin/bash
```

**Root cause:** `less` gibi, GTFOBins'te bilinen bir shell-escape zafiyeti barındıran bir binary'nin **sudo ile şifresiz çalıştırılmasına izin verilmesi**. Sudo yetkileri dağıtılırken binary'nin GTFOBins listesinde olup olmadığı mutlaka kontrol edilmeli.

---

## Sonuç / Lessons Learned

| # | Kullanıcı | Zafiyet | Önlem |
|---|-----------|---------|-------|
| 1 | recon_user | Anonymous FTP + doğrulanmamış otomatik çalıştırma | Anonim yazma kapatılmalı, gelen dosyalar sandboxlanmalı/doğrulanmalı |
| 2 | dev_user | Başka kullanıcıya ait scheduled job dosyasının yazılabilir olması | Cron/timer dosyaları yalnızca sahibi tarafından yazılabilir olmalı |
| 3 | monitor_user | PATH hijacking (komutun mutlak yolla çağrılmaması) | Servisler bağımlılıklarını her zaman tam yol ile çağırmalı |
| 4 | ops_usr | Sudo zincirindeki ara dosyanın başka kullanıcı tarafından yazılabilir olması | Sudo ile çalıştırılan script'lerin çağırdığı tüm dosyalar da izin denetimine dahil edilmeli |
| 5 | root | GTFOBins'te bilinen bir binary'ye NOPASSWD sudo izni | Sudoers girişleri GTFOBins'e karşı düzenli denetlenmeli |

Bu zincir, tek başına küçük görünen izin hatalarının bir araya geldiğinde nasıl tam bir sistem ele geçirmeye dönüşebileceğini gösteriyor. Her aşamadaki ortak tema aynı: **bir kullanıcının kontrol ettiği bir kaynağın, başka bir kullanıcının/servisin güven sınırı içine sızması.**
