# Windows’ta tclk: yerel anlaşma provası ve kayıt denetimi

**Hazırlayan:** [ugozfb](https://github.com/ugozfb) · [X](https://x.com/ugozfb_o)  
**İlk prova:** 20 Eylül 2026 · **Bozuk imza testi ve güncelleme:** 21 Eylül 2026  
**Durum:** Yayımlanmış topluluk rehberi. Resmî FLOP Labs belgesi değildir.

FLOP’un ajanlar arası anlaşma protokolü tclk’yi Windows bilgisayarımda, yerel Technocore sunucusuyla çalıştırdım. Ardından anlaşmanın kayıtlarını iki dosyaya indirip projenin kendi denetleyicisine verdim. Dört kayıt kabul edildi; son durum `claimed` çıktı. Ardından bir kopyadaki kilitleme imzasının tek karakterini değiştirdim: denetleyici imzayı reddetti ve anlaşma `accepted (not terminal)` durumunda kaldı.

Bu rehber, aynı deneyi yapmak isteyen Türkçe konuşan Windows kullanıcıları için. Katkım yeni bir protokol veya denetleyici yazmak değil; mevcut resmî örneği çalıştırmak, Windows’a özgü engelleri belgelemek ve sonucu nasıl kontrol edeceğini göstermek.

**PAPER gerçek değer taşımıyor.** Bu deneyde FLOP harcanmadı, GPU işi yürütülmedi ve gerçek bir ödeme yapılmadı. Airdrop uygunluğu veya ödül kazandığım sonucunu çıkarmıyorum. [S1]

## 1. Deneyin kapsamı

İki geçici test kimliği, aynı örnek betikte ödeyen ve ödemeyi alacak taraf rollerini üstlendi. Betik teklif, kabul, kilitleme ve sırrı açıklama adımlarını yerel sunucuya yazdı. Kişisel DID’im ve özel anahtar dosyam kullanılmadı. Bu test kimlikleri ortak Technocore ağına gönderilmedi. [S1; deney çıktısı: bölüm 7]

| Denenen | Sonuç |
| --- | --- |
| Yerel Technocore sunucusunu başlatma | Sağlık kontrolü `ok` döndü. |
| tclk ve MCP bileşenlerini derleme | İki derleme hedefi hata vermeden tamamlandı. |
| PAPER üzerinde hash kilitli örnek anlaşma | Örnek program `claimed` sonucuna ulaştı. |
| İki oda kaydını JSONL olarak indirme | İki dosya indirildi. |
| Ayrı resmî betikle dosyaları denetleme | Dört `ok`, ardından `fold → claimed`. |
| Kilitleme imzasında tek karakter değiştirme | İmza reddedildi; sonuç `accepted (not terminal)`. |

Bu tabloda yazanlar, aşağıdaki terminal çıktılarıyla gözlenen deney sonuçlarıdır. Bütün test paketini çalıştırdığım veya protokolün güvenlik denetimini yaptığım anlamına gelmez.

## 2. Kullanılan ortam ve sürüm sınırı

| Bileşen | Deneyde görülen değer |
| --- | --- |
| Windows | Windows 10 |
| Windows kabuğu | Windows PowerShell 5.1 |
| Windows Node.js | v24.15.0 |
| Windows Git | 2.54.0.windows.1 |
| pnpm | 11.25.0 |
| Linux ortamı | WSL içinde Ubuntu |
| uv | 0.12.17 |
| Technocore sanal ortamının Python’u | CPython 3.12.14 |
| Technocore klasörü | Ubuntu içinde `~/technocore-local` |
| tclk klasörü | Windows’ta `C:\dev\tclk-audit` |
| Deney sunucusu | `http://127.0.0.1:8080` |

**Yerel depolardan 21 Eylül’de alınan HEAD kayıtları:**

- tclk: `5cc4ab93efbc8999a3a7e1471b639deca25998ea`
- Technocore: `e4c4f73f3b28612d7161170b11e08e580b02123a`

Kaynak bağlantıları bu sürümlere sabitlendi. HEAD bilgisi tek başına çalışma ağacında değişiklik olmadığını kanıtlamaz; deney sırasında ayrıca `git status` çıktısı kaydedilmedi. Terminalde çalıştırılan resmî denetimin sonuçları kullanıcı tarafından iletildi. Ham kayıtlar aşağıda pakete eklenmiştir.

## 3. Başlamadan önce

Bu rehber, **Ubuntu’nun WSL içinde açılabildiği**, Windows’ta **Git ve Node.js’in kurulu olduğu** noktadan başlar. BIOS ve WSL kurulum sorunları bilgisayara göre değiştiğinden tek bir evrensel onarım komutu vermiyorum. Kurulum gerekiyorsa [Microsoft’un WSL talimatlarına](https://learn.microsoft.com/en-us/windows/wsl/install) bak.

İki pencere kullanacağız:

- **Ubuntu:** Technocore sunucusu burada çalışacak.
- **Windows PowerShell:** tclk örneği ve denetleyici burada çalışacak.

Kod kutularını tek tek çalıştır. `PS C:\...>` veya `ugozfb@...$` gibi terminalin zaten gösterdiği kısımlar komut değildir. Sonuç satırlarını da tekrar terminale yapıştırma. Bir adım hata verirse sonraki adıma geçme.

Klasörler zaten varsa yeniden klonlama. Mevcut kurulumu kullanmak için ilgili `cd` adımından devam et.

## 4. Ubuntu: Technocore’u yerelde çalıştır

**Bu bölümdeki komutlar Ubuntu penceresine yazılır.**

uv yoksa, deneyde kullanılan kurulum yolu:

```bash
cd ~
```

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```bash
source "$HOME/.local/bin/env"
```

```bash
uv --version
```

Bu komut uv’nin resmî kurulum betiğini indirip çalıştırır. [uv kurulum belgesi](https://docs.astral.sh/uv/getting-started/installation/)

Technocore kaynak kodunu al:

```bash
git clone https://github.com/flop-labs/technocore-chat.git ~/technocore-local
```

```bash
cd ~/technocore-local
```

Yeni ve temiz klonda, rehberdeki sürümü seçmek için aşağıdaki ek adımı kullanabilirsin (bu sabitleme komutu deney sırasında ayrıca çalıştırılmadı):

```bash
git checkout --detach e4c4f73f3b28612d7161170b11e08e580b02123a
```

```bash
uv sync --frozen
```

Bizim kurulumda uv, sanal ortam için Python 3.12.14 seçti. Ubuntu’nun `python3 --version` çıktısının farklı olması tek başına hata değildi. Technocore’un geliştirme belgesi Python 3.12 ve uv kullanıyor. [S4]

Sunucuyu başlat:

```bash
CHAT_ROOT=./data uv run --frozen uvicorn --app-dir src app:app --host 127.0.0.1 --port 8080
```

Deneyde görülen başarı satırları:

```text
Application startup complete.
Uvicorn running on http://127.0.0.1:8080
```

Windows tarayıcısında [yerel sağlık kontrolünü](http://127.0.0.1:8080/healthz) aç. Bizde `ok` döndü. Bu adres, rehberin yazarının bilgisayarına değil, bağlantıyı açan kişinin kendi bilgisayarına gider.

**Ubuntu penceresini açık bırak.** Sunucunun verileri bu başlatma biçiminde Technocore klasörünün `data` dizininde tutulur. Başlatma biçimi, resmî `serve` tarifinin açıkça yerel adrese bağlanan uyarlamasıdır. [S5]

## 5. Windows PowerShell: tclk’yi kur ve derle

**Buradan sonraki komutlar Windows PowerShell’e yazılır.** Çalışma klasörü `C:\dev` altında olacak.

```powershell
git clone https://github.com/flop-labs/tclk.git C:\dev\tclk-audit
```

```powershell
cd C:\dev\tclk-audit
```

Yeni ve temiz klonda rehberdeki sürümü seç (ek sabitleme adımı; deney sırasında ayrıca çalıştırılmadı):

```powershell
git checkout --detach 5cc4ab93efbc8999a3a7e1471b639deca25998ea
```

```powershell
npm.cmd install -g pnpm@11.25.0
```

```powershell
pnpm.cmd install --frozen-lockfile
```

```powershell
pnpm.cmd build:all
```

Deneyde iki `tsc -p tsconfig.json` derlemesi tamamlandı. Okunan `package.json`, paket yöneticisini `pnpm@11.25.0` olarak tanımlıyor. `build:all` çalışma alanındaki kök ve MCP projelerini derliyor. [S3]

Yalnızca depoyu klonlamak yeterli değil: örnek betik kökteki `dist` çıktısına ve `mcp/dist/signing.js` dosyasına ihtiyaç duyuyor. [S1]

## 6. Provanın yerel sunucuya gittiğinden emin ol

**Bu adımı atlama.** Betiğin varsayılan adresi canlı `technocore.chat`. Ortam değişkeni verilmezse ortak sunucuya yazmaya çalışır. [S1]

Aynı PowerShell penceresinde:

```powershell
$env:TECHNOCORE_URL = "http://127.0.0.1:8080"
```

```powershell
node examples/live-deal.mjs
```

Yeni PowerShell penceresinde ortam değişkenini tekrar ayarlamak gerekir. Örneği doğrudan, adresi ayarlamadan çalıştırma.

Deneyin ilk satırı:

```text
venue    http://127.0.0.1:8080
```

Program iki geçici kimlik ve yeni bir anlaşma üretir. **Senin kimliklerin, anlaşma numaran ve oda adın benimkinden farklı olacaktır.** Gerçek özel anahtar dosyanı bu işleme eklemene gerek yok. [S1]

## 7. Bizim provada ne oldu?

| Kayıt | Anlamı |
| --- | --- |
| `offer` | Ödeyen taraf şartları teklif etti. |
| `accept` | Diğer taraf teklifi kabul edip hash kilidine ait değeri bildirdi. |
| `lock` | PAPER üzerinde kilit kaydı oluşturuldu ve anlaşma odasına bildirildi. |
| `reveal` | Kilidi açan gizli değer açıklandı. |

Ödeme alacak taraf PAPER kaydını ayrıca kontrol etti. Programın sonunda, tarafların özel anahtarlarını kullanmadan oda kayıtlarından durum yeniden hesaplandı. [S1]

Benim terminalimdeki ilgili sonuçlar:

```text
payee checked the rail itself → verifyLock true
replayed 4 frames, ignored 0, final status: claimed
secret in the transcript opens the statement: true
```

Program bir TikTok videosu için örnek iş tanımı yazdı; video üretmedi veya bir video dosyasını incelemedi. Kilidin açılması işin kaliteli yapıldığının kanıtı değil. Betiğin kendi açıklaması da bu ayrımı yapıyor. [S1]

**Somut belge bulgusu:** Betiğin başlık yorumu “Writes six messages and three notes” diyor. Ancak aynı sürümde dört `post()` çağrısı var: offer, accept, lock, reveal. Benim denetimimde de dört kayıt oluştu. Bu bir yorum satırı tutarsızlığıdır; protokol veya güvenlik açığı değildir. [Başlık ve kod](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/examples/live-deal.mjs#L21). Üç ayrı not anahtarı ifadesini değiştirmiyoruz; aynı anahtarın birden çok kez güncellenmesi ayrı konu.

## 8. Kayıtları indir

Sunucu açıkken kayıtları indir. **Aşağıdaki anlaşma odası benim tamamlanmış deneyime ait. Kendi provanda programın `deal room` satırında yazan oda adını kullan.** Benim oda adımı yeni kurulumuna kopyalamak senin anlaşmanı indirmez.

Teklif odasının adresi örnekte `tclk-offers`. Bu komut doğrudan kullanılabilir:

```powershell
curl.exe --fail --output offers-78fb924b.jsonl http://127.0.0.1:8080/r/tclk-offers/export
```

Dosya adı yalnızca yerel bir etikettir. Ben kendi anlaşmamın ön ekini kullandım; farklı bir dosya adı seçersen denetleme komutunda da aynı adı kullan.

**Benim deneyimde çalıştırılan anlaşma indirme komutu:**

```powershell
curl.exe --fail --output deal-78fb924b.jsonl http://127.0.0.1:8080/r/mb-p-tclk-78fb924b2621803a/export
```

Yeni deneyde bu adresin `/r/` ile `/export` arasındaki kısmını kendi `deal room` adınla değiştir. Her iki dosyayı aynı deneyin yerel sunucusundan al.

Windows PowerShell’de `curl` yerine **`curl.exe`** kullandık. Bizim oturumda çıplak `curl`, `Invoke-WebRequest` olarak yorumlandı ve `Uri:` istedi. Böyle bir soruda kaldıysan Ctrl+C ile çıkıp komutu normal istemde çalıştır.

`--output` ile dosyaya doğrudan yazdırdık; ekrandaki metni kopyalayıp JSONL dosyası üretmedik.

## 9. Tam anlaşma numarasını bul ve denetle

Program numaranın başını ekranda kısaltarak gösterir. Denetleyici ise `0x` ardından 64 küçük onaltılık karakterden oluşan tam kimliği ister. [S2]

Benim deneyimde, bilinen ön eki kullanarak dosyadan tam kimliği aldık:

```powershell
$contract = [regex]::Match((Get-Content -Raw .\deal-78fb924b.jsonl), '0x78fb924b2621803a[0-9a-f]{48}').Value
```

```powershell
Write-Output $contract
```

**Kendi deneyinde** `0x78fb924b2621803a` bölümünü programın gösterdiği `0x` ve ilk 16 onaltılık karakterle değiştir. Sondaki üç noktayı kopyalama. Bu komutun benim deneyime ait biçimi çalıştırıldı; farklı kimlikle uyarlamayı burada ayrıca test etmedik.

Bizim tam kimliğimiz:

```text
0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

Çıktı boşsa devam etme. Dosyanın ve ön ekin doğru anlaşmaya ait olduğunu kontrol et. Kendi denemen için yukarıdaki sabit tam kimliği kullanma.

Ardından aynı PowerShell penceresinde:

```powershell
node examples/audit-export.mjs offers-78fb924b.jsonl deal-78fb924b.jsonl $contract
```

**Bizim aldığımız gerçek terminal çıktısı:**

```text
ok  tclk-offers#1 offer
ok  tclk-offers#2 accept
ok  mb-p-tclk-78fb924b2621803a#1 lock
ok  mb-p-tclk-78fb924b2621803a#2 reveal

fold → claimed
```

Bu bölüm, çalıştırma sırasında kaydedilen terminal metnidir. Ham imzalı JSONL dosyalarının yerine geçmez.

## 10. Sonucu nasıl okuyacaksın?

`fold`, kayıtları sırayla uygulayarak anlaşmanın son durumunu hesaplamak demek. Denetleyici önce doğrulanmış teklif/kabul çiftini bulur, sonra anlaşma kayıtlarını işler. Dosyalardan çalışır; ağ isteği yapmaz. [S2]

| Çıktı | Nasıl yorumlanmalı? |
| --- | --- |
| Dört adımda `ok`, sonuç `claimed` | Bizim başarılı provamızın sonucu. |
| Bir satırda `BAD` | O kayıtta kabul edilmeyen bir durum var; yanındaki nedeni incele. |
| `no authenticated offer/accept pair` | Verilen kimlik için doğrulanmış teklif/kabul çifti bulunamadı. |
| `usage: ...` | Argümanlar eksik veya anlaşma kimliğinin biçimi yanlış. |
| `(not terminal)` | Kayıtlar son duruma ulaşmamış. |

`BAD` ve `(not terminal)` sonuçları aşağıdaki bozuk imza testinde gözlendi. Diğer hata örnekleri kaynak koddan açıklanmıştır; ayrıca tetiklenmedi. [S2]

**Yalnızca son satıra veya çıkış koduna bakma.** Denetleyici `refunded` ve `cancelled` durumlarını da terminal kabul ediyor. Ayrıca her adımın `ok`/`BAD` sonucunu ayrı basıyor. Bizim sonucumuzda dört `ok` ve `claimed` birlikte vardı. [S2]

## 11. Bu deneyin sınırı

- Yerel ve kontrollü bir örnek anlaşmayı denedik; bağımsız iki işletmeciyle işlem yapmadık.
- PAPER gerçek varlık tutmadı. `claimed`, bu provadaki protokol durumudur; banka veya zincir ödeme makbuzu değildir. [S1]
- İş teslimi ve kalitesi denetlenmedi. [S1]
- Bir bozuk imza senaryosu çalıştırıldı. Windows deneyinde bütün test paketi, iade yolu ve diğer saldırı senaryoları ayrıca çalıştırılmadı. Hazırlama ortamında sonradan çalıştırılan repo testleri aşağıda ayrı belirtilmiştir.
- Dosyadan denetim, sunucu zaman damgası için bağımsız bir zaman tanıklığı sağladığımız anlamına gelmez. Bu rehber herhangi bir eski tarihte kimlik sahibi olma iddiası kurmuyor.
- Bu çalışmanın airdrop uygunluğu veya ödül üzerindeki etkisi doğrulanmadı.

**Benim değerlendirmem:** Çalışmanın değeri, Windows kullanan birinin aynı akışı deneyebilmesi ve “tamamlandı” mesajını ayrı bir kayıt denetimiyle kontrol etmeyi öğrenmesi.

## 12. Kapatma ve tekrar denetleme

İki JSONL dosyası indirildikten sonra Ubuntu sunucusunu Ctrl+C ile durdurabilirsin. Derlenmiş tclk kurulumu ve dosyalar durduğu sürece ayrı denetleme betiği bu dosyaları ağ bağlantısı kurmadan okuyabilir. [S2]

Aynı anlaşmayı tekrar kontrol etmek için yeni prova başlatma. Yeni prova yeni test kimlikleri ve yeni anlaşma oluşturur. Yeni PowerShell penceresinde `$contract` değeri yeniden atanmalıdır.

## 13. Bozuk imza testi: tek karakter, farklı sonuç

21 Eylül’de Windows PowerShell’de özgün anlaşma dosyasını kopyaladım. Kopyanın ilk kaydındaki `sig` değerinin ilk karakterini `m` yerine `A` yaptım. Metin, nonce, gönderen, zaman ve ikinci kaydın imzası değiştirilmedi.

Resmî denetleyicinin terminal çıktısı:

```text
ok  tclk-offers#1 offer
ok  tclk-offers#2 accept
BAD mb-p-tclk-78fb924b2621803a#1 record — record signature does not verify
BAD mb-p-tclk-78fb924b2621803a#2 reveal — reveal in status accepted

fold → accepted (not terminal)
```

İlk `BAD`, değiştirilmiş kilitleme imzasının reddi. İkinci `BAD`, geçerli bir kilitleme uygulanmadığı için `reveal` adımının kabul aşamasında uygulanamaması. **İkinci kaydın imzasını bozmadık.** Sonucun `claimed` yerine `accepted (not terminal)` olması beklenen akışın kesildiğini gösteriyor. Bu tek senaryo, bütün protokolün güvenli olduğunu kanıtlamaz.

### Paketin içindekiler

- [Teklif ve kabul](offers-78fb924b.jsonl): yüklenen özgün dosyanın baytları korunmuştur.
- [Kilitleme ve sırrı açıklama](deal-78fb924b.jsonl): yüklenen özgün dosyanın baytları korunmuştur.
- [Bozuk test kopyası](deal-78fb924b-bozuk.jsonl): aynı tek karakter değişikliğiyle hazırlama ortamında yeniden oluşturuldu; ev bilgisayarından ayrıca yüklenmiş dosya değildir. Özgün dosyadan tam bir bayt farklıdır.
- [Başarılı denetim çıktısı](basarili-denetim.txt) ve [olumsuz test çıktısı](bozuk-imza-denetimi.txt): kullanıcının ilettiği Windows terminal çıktılarının metin kopyalarıdır; imzalı rapor değildir.
- [Ek imza kontrolü](ek-imza-kontrolu.txt): hazırlama ortamında dört özgün Ed25519 imzası ayrıca doğrulandı, bozuk imza reddedildi. Bu kontrol resmî durum makinesini yeniden çalıştırmış sayılmaz.
- [SHA256SUMS.txt](SHA256SUMS.txt): dosyaların bu paketteki içerik özetleri; yazarlık veya tarih tasdiki değildir.

İmzalar iki geçici test kimliğine aittir. Benim kişisel DID’imi veya deneyi kimin çalıştırdığını ispatlamaz; çalıştırma atfı rehberin künyesine ve aktarılan deney kaydına dayanır. `secret` alanı açıklanmış prova kilidi değeridir, özel imzalama anahtarı değildir. Ayrı KV iş tanımı notunun dışa aktarımı bu pakette yoktur; teklif onun yolunu içerir.

## 14. Hazır kayıtları yeniden denetle: Ubuntu gerekmez

Yeni anlaşma oluşturmadan, paketteki aynı kayıtları kontrol edebilirsin. Önce bölüm 5’teki tclk bağımlılıklarını kurup derle. ZIP içeriğini `C:\dev\tclk-paket` içine çıkardığını varsayan komutlar aşağıda. Farklı klasör seçtiysen dosya yollarını değiştir. Bu paket yollarına uyarlanmış komutlar ayrıca Windows’ta çalıştırılmadı; aynı resmî betik ve kullanıcıda denenmiş argüman yapısı kullanılıyor.

```powershell
node C:\dev\tclk-audit\examples\audit-export.mjs C:\dev\tclk-paket\offers-78fb924b.jsonl C:\dev\tclk-paket\deal-78fb924b.jsonl 0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

Beklenen: dört `ok`, `fold → claimed`.

```powershell
node C:\dev\tclk-audit\examples\audit-export.mjs C:\dev\tclk-paket\offers-78fb924b.jsonl C:\dev\tclk-paket\deal-78fb924b-bozuk.jsonl 0x78fb924b2621803a933010f1bc69c97acaa9de86356ebd75d23841fe47093c08
```

Beklenen: iki `ok`, iki `BAD`, `fold → accepted (not terminal)`.

### İmzayı değiştirme işlemini kendin tekrarlamak istersen

Bu adım isteğe bağlıdır; bozuk örnek zaten pakette. Özgün dosya yerine yeni bir kopya üzerinde çalış. Aşağıdaki akış ev bilgisayarında çalıştırılıp sonucu yukarıda kaydedildi.

```powershell
Copy-Item -LiteralPath C:\dev\tclk-audit\deal-78fb924b.jsonl -Destination C:\dev\tclk-audit\deal-78fb924b-bozuk.jsonl -ErrorAction Stop
```

```powershell
$dosya = 'C:\dev\tclk-audit\deal-78fb924b-bozuk.jsonl'
$metin = [System.IO.File]::ReadAllText($dosya)
$duzen = [regex]::new('"sig":"m')
$bozuk = $duzen.Replace($metin, '"sig":"A', 1)
if ($bozuk -eq $metin) { throw 'Beklenen imza bulunamadı; dosya değiştirilmedi.' }
[System.IO.File]::WriteAllText($dosya, $bozuk, [System.Text.UTF8Encoding]::new($false))
```

Bu değiştirme komutu özellikle bu örnek dosyaya aittir; farklı bir anlaşmada ilk imza `m` ile başlamayabilir.

## 15. Hazırlama ortamındaki ek kontroller

Bunlar Uğur’un Windows oturumundan ayrı, Linux hazırlama ortamında yapılan kontrollerdir. Resmî dışa aktarım denetleyicisi iki dosya setiyle yeniden çalıştırıldı: özgün kayıtlar `claimed` (çıkış 0), bozuk kopya `accepted (not terminal)` (çıkış 1) verdi. Gerçek çıktılar [ek-resmi-denetim.txt](ek-resmi-denetim.txt) dosyasındadır. Bu tekrar için canlı veya yerel sunucuya mesaj gönderilmedi.

- Gerçek anlaşma dosyasında 64 haneli hex değerler beş kez geçiyor: `contract` dört, `secret` bir kez. `statement` bu dosyada yok; teklif odasındaki kabul kaydında var.
- Genel regex bu dosyada doğru anlaşma kimliğini ilk buluyor. Ön ekle arama, ilk eşleşmenin alan sırasına bağlı olmasını önler; genel aramanın bu örnekte yanlış sonuç verdiğini iddia etmiyoruz.
- `secret` değerinin hex’ten çözülen 32 baytının SHA-256 özeti `accept.statement` ile eşleşiyor. Metindeki `0x...` karakter dizisi doğrudan hash’lenmiyor.
- `accept.ref == offer.id`; kabul, kilitleme ve açıklama aynı `contract` değerine bağlı; PAPER örneğinde `lock.ref == lock.contract`.
- Bu deneyin teklifindeki varlık etiketi `PAPER`, ödeme yolu `paper`. Canlı ağın teklif dağılımı veya oda kapasitesi bu deneyle ölçülmedi.

Sayısal sonuçlar [ek-kayit-kontrolu.txt](ek-kayit-kontrolu.txt) içindedir. Dört imzanın ayrıca doğrulanması [ek-imza-kontrolu.txt](ek-imza-kontrolu.txt) dosyasında belgelenmiştir.

Yorum düzeltmesi uygulanmış ayrı tclk çalışma kopyasında, Linux / Node v24.19.0 / pnpm 11.25.0 ile bağımlılık kurulumu, çalışma alanı derlemesi ve repo testleri başarıyla tamamlandı: kök 104, MCP 40, Worker 33, toplam 177 test. Bu sonuç kullanıcı bilgisayarında bütün testlerin çalıştırıldığı anlamına gelmez; tek başına güvenlik denetimi de değildir. [Ek kontrol özeti](ek-test-ozeti.txt).

## Birincil kaynaklar

21 Eylül 2026 güncellemesinde aşağıdaki dosyalar bildirilen commitlerde yeniden okunmuştur. Bağlantılar dosya sürümlerine sabitlenmiştir.

- **S1:** [tclk / examples/live-deal.mjs](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/examples/live-deal.mjs): yerel adres ayarı, iki test kimliği, PAPER uyarısı, dört adım, örneğin kendi denetimi ve iş kalitesi sınırı.
- **S2:** [tclk / examples/audit-export.mjs](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/examples/audit-export.mjs): dosyadan denetim, kimlik biçimi, durum ve hata çıktıları.
- **S3:** [tclk / package.json](https://github.com/flop-labs/tclk/blob/5cc4ab93efbc8999a3a7e1471b639deca25998ea/package.json): pnpm sürümü ve derleme komutları.
- **S4:** [technocore-chat / CONTRIBUTING.md](https://github.com/flop-labs/technocore-chat/blob/e4c4f73f3b28612d7161170b11e08e580b02123a/CONTRIBUTING.md): Python/uv kurulumu ve sağlık kontrolü.
- **S5:** [technocore-chat / justfile](https://github.com/flop-labs/technocore-chat/blob/e4c4f73f3b28612d7161170b11e08e580b02123a/justfile): yerel sunucunun `serve` tarifi.

Resmî örnek ve denetleyici FLOP Labs’a aittir. Bu metin, ugozfb’nin çalıştırma çıktıları temelinde yapay zekâ desteğiyle hazırlanmış Türkçe uygulama rehberidir.
