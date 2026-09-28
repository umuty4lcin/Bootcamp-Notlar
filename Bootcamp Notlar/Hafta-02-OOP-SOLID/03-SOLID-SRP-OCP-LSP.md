# Hafta 2 · Çarşamba — SOLID 1-3: SRP, OCP, LSP

**Okuma süresi:** ~50 dk
**Neden bu konu:** SOLID, "iyi kod" tartışmasının en çok tekrarlanan beş maddesi. Bootcamp'te her mimari kararın gerekçesi olarak karşına çıkacak, iş görüşmelerinde sorulacak. Ama asıl değeri ezberde değil: bir sınıfa bakıp "bu neden değişecek, değiştiğinde nereyi kıracak" diye sorabilmekte. Bu not o soruyu sormayı öğretiyor — ve ilkeleri abartmanın kodu nasıl bozduğunu da gösteriyor.

---

## Önce Basitçe

Bir programı ilk yazdığında her şey kolaydır. Dosya boştur, kural yoktur, aklındaki şeyi yazarsın ve çalışır. Sorun ikinci ayda başlar. Müşteri bir şey ister, sen bir yeri değiştirirsin, alakasız bir yer bozulur. Üçüncü ayda bir dosya bin satır olmuştur ve açmaya üşenirsin. Altıncı ayda o dosyaya dokunmak "riskli" sayılır; herkes etrafından dolaşır.

SOLID, bu çürümeyi yavaşlatmak için yazılmış beş kuraldır. Hiçbiri "programın daha hızlı çalışsın" demez. Hepsi tek bir şeyi hedefler: **değişikliğin maliyetini düşürmek.** Yani yarın bir şey istendiğinde, sadece bir yeri açıp değiştirip kapatabilmek. Bugün yazdığın kodun yarınki hâlini düşünmek.

Bu notta beş kuralın ilk üçü var. Birincisi (SRP) "bir sınıf bir işten sorumlu olsun" der — ama bu cümle neredeyse herkes tarafından yanlış anlaşılır, notun en uzun bölümü onu düzeltmekle geçiyor. İkincisi (OCP) "yeni bir durum eklerken eski kodu kurcalamak zorunda kalma" der. Üçüncüsü (LSP) "bir sınıfın yerine alt sınıfını koyduğunda program şaşırmasın" der.

Şunu baştan söylemek gerek: bu kurallar birer **yön tarifi**dir, kanun değil. Her birini sonuna kadar uygulamaya kalkan biri, üç sınıflık işi otuz sınıfa dağıtır ve kimsenin okuyamadığı bir yapı kurar. Bu hata o kadar yaygın ki adı var: **over-engineering (aşırı mühendislik)**. Notta her ilkenin sonunda "burası nereye kadar" bölümü var, atlamamanı öneririm.

Kuralların çıkış noktası ortak: kod okunmak ve değiştirilmek için vardır, çalışmak zaten asgari şart. Bir yazılımın ömrünün büyük kısmı bakım aşamasında geçer ve altı ay sonra o kodu okuyan kişi, yazan sen olsan bile yabancıdır. Sırayla gideceğiz: önce SOLID'in ne olduğu ve ne zaman zarar verdiği, sonra üç ilke ayrı ayrı — her biri için kötü kod ve iyi kod yan yana. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bir apartman düşün. SRP, her dairenin kendi sayacının olmasıdır — komşu klimasını açınca senin faturan artmaz. OCP, binaya yeni bir daire eklerken mevcut dairelerin duvarını yıkmak zorunda kalmamandır. LSP ise şudur: asansöre "asansör" yazıyorsa içine binen herkes onun yukarı çıkacağını varsayar; yük asansörü de olsa bu varsayımı bozmamalıdır.

---

## Bu Notta Ne Var

1. SOLID nedir, neyi çözer, ne zaman zarar verir
2. SRP: "tek sorumluluk" cümlesinin yanlış okunuşu
3. SRP pratiği: God class ve fat controller
4. SRP'de ayrıştırma nasıl yapılır
5. SRP'nin sınırı: ne kadar bölmek fazla
6. OCP: değişime kapalı, genişlemeye açık
7. OCP pratiği: switch zincirinden polimorfizme
8. OCP'nin bedeli: spekülatif soyutlama
9. LSP: alt tip ikame ilkesi
10. LSP ihlalinin biçimleri: ön koşul, son koşul, istisna

---

## 1. SOLID Nedir, Neyi Çözer

> **Benzetme —** Bir binanın elektrik tesisatını düşün. Kötü tesisatta her şey tek hatta bağlıdır: mutfakta ütü fişini takarsın, salondaki lamba söner. İyi tesisatta her devre ayrıdır, her devrenin kendi sigortası vardır. İkisi de aynı ampulü yakar. Fark, bir şey bozulduğunda ortaya çıkar: birinde tüm binanın elektriği gider, diğerinde tek bir sigorta atar. SOLID, kodun sigorta kutusunu düzenleme kılavuzudur.

**Basitçe:** SOLID, nesne yönelimli kod yazarken işine yarayan beş ilkenin baş harflerinden oluşur. Hiçbiri "böyle yazmazsan çalışmaz" demez; hepsi "böyle yazarsan değiştirmesi kolay olur" der.

**Teknik olarak:** **SOLID** — Robert C. Martin'in derlediği, nesne yönelimli tasarımda bağımlılık yönetimi ve değişime dayanıklılık için beş ilke.

| Harf | Açılımı | Tek cümlede |
|---|---|---|
| **S** | Single Responsibility Principle | Bir sınıfın değişmesi için tek bir sebep olsun |
| **O** | Open/Closed Principle | Genişlemeye açık, değişikliğe kapalı olsun |
| **L** | Liskov Substitution Principle | Alt tip, üst tipin yerine sorunsuz geçebilsin |
| **I** | Interface Segregation Principle | Kimse kullanmadığı metodu uygulamak zorunda kalmasın |
| **D** | Dependency Inversion Principle | Somuta değil soyuta bağlan |

Bu notta ilk üçü var; ISP ve DIP yarınki notta.

### Ortak amaç: değişimin maliyeti

Beş ilkenin hepsi aynı soruya cevap verir: **"Yarın bir şey değişirse kaç dosyaya dokunmam gerekir?"**

İyi tasarımda cevap "bir". Kötü tasarımda cevap "bilmiyorum, deneyip göreceğim".

Bu soruya iki kavram eşlik eder:

**Coupling (bağlılık)** — İki kod parçasının birbirine ne kadar yapışık olduğu. Yüksek bağlılık, birini değiştirince diğerinin bozulması demektir. Hedef: **gevşek bağlılık (loose coupling)**.

**Cohesion (uyum)** — Bir sınıfın içindeki parçaların ne kadar aynı işe hizmet ettiği. Düşük uyum, sınıfın "her şeyden biraz" yapması demektir. Hedef: **yüksek uyum (high cohesion)**.

SOLID'in beş maddesi, bu iki cümlenin ayrıntılandırılmış hâlidir: bağlılığı düşür, uyumu yükselt.

### SOLID ne zaman zarar verir

Bu bölüm notun en önemli kısmı olabilir, o yüzden sona bırakmıyorum.

İlkeleri "ne kadar çok o kadar iyi" diye uygularsan şunu elde edersin:

```csharp
// Üç satırlık işin on sınıfa dağıtılmış hâli
public interface IKdvHesaplayiciFactory { IKdvHesaplayici Olustur(UrunTipi tip); }
public interface IKdvHesaplayici { decimal Hesapla(decimal tutar); }
public interface IKdvOranSaglayici { decimal OranGetir(UrunTipi tip); }
public interface IKdvOranSaglayiciFactory { IKdvOranSaglayici Olustur(); }
// ... ve iki somut uygulama daha
```

Bu kodun yaptığı iş tek satırdır: `decimal kdvli = tutar * 1.20m;`

KDV oranı on yılda bir değişir. Buna karşı üç arayüz ve iki fabrika kurmak korunma değil israftır. Her soyutlama okuyucudan bir zihinsel sıçrama ister: "bu arayüzün gerçek uygulaması nerede?" Bu sıçramaların bedeli vardır.

> **Pratik kural:** Soyutlamayı **ihtiyaç ortaya çıktığında** ekle, ihtimal üzerine değil. İkinci uygulama gerçekten gelirse soyutlarsın. "Belki ileride Oracle'a geçeriz" cümlesiyle kurulan katmanların çoğu hiç kullanılmaz.

Bunun bir adı var: **YAGNI (You Aren't Gonna Need It)** — İhtiyacın olmayacak. SOLID ile YAGNI birbirini dengeler. SOLID "değişime hazırlan" der, YAGNI "hayal ettiğin değişime değil" der.

> **Bu benzetme şurada bozulur:** Elektrik tesisatı benzetmesinde her devreyi ayırmak hep iyidir; maliyeti sadece biraz kablo ve bir sigortadır. Kodda ise her ayrım bir **arayüz, bir dosya, bir dolaylılık katmanı** demektir. Yüz devrelik bir sigorta kutusu, iki odalı bir dairede yönetilemez hâle gelir. Kodda "ayırmanın" bedeli tesisattakinden çok daha yüksektir.

---

## 2. SRP: "Tek Sorumluluk" Cümlesinin Yanlış Okunuşu

> **Benzetme —** Bir mahalle muhtarını düşün. Muhtar ikametgâh verir, nüfus kaydı çıkarır, mahalle sorunlarını belediyeye iletir. Şimdi biri çıkıp "muhtar tek işten sorumlu olsun" dese, muhtarlığı üçe bölmek gerekirdi — saçma olurdu. Doğru soru şu: **muhtarın görevleri kimin talebiyle değişir?** İkametgâh formatı Nüfus Müdürlüğü'nün kararıyla değişir, mahalle sorunları belediyenin kararıyla. İki farklı makam, iki farklı değişim kaynağı. SRP tam olarak bunu söyler: kimin talebiyle değiştiğine bak, kaç iş yaptığına değil.

**Basitçe:** SRP'nin "bir sınıf bir iş yapsın" diye okunması yanlıştır. Doğrusu: bir sınıfın **değişmesi için tek bir sebep** olmalıdır. Aynı sebeple değişen işler bir arada durabilir; farklı sebeplerle değişen işler ayrılmalıdır.

**Teknik olarak:** **SRP (Single Responsibility Principle — Tek Sorumluluk İlkesi)** — Bir sınıfın değişmesi için **tek bir sebebi** olmalıdır. Robert Martin'in sonradan yaptığı netleştirmeyle: bir modül **tek bir aktöre** karşı sorumlu olmalıdır.

**Aktör (actor)** — Değişikliği talep eden taraf: muhasebe departmanı, hukuk, pazarlama, operasyon, sistem yöneticisi. İnsan grubu ya da rol.

### Yanlış okuma neye yol açar

"Bir sınıf bir iş yapsın" diye okuyan kişi şu tuzağa düşer:

```csharp
// SRP'nin yanlış anlaşılmış hâli — metot başına sınıf
public class SiparisToplamHesaplayici { public decimal Hesapla(Siparis s) { /* ... */ } }
public class SiparisKdvHesaplayici    { public decimal Hesapla(Siparis s) { /* ... */ } }
public class SiparisIndirimHesaplayici{ public decimal Hesapla(Siparis s) { /* ... */ } }
public class SiparisKargoHesaplayici  { public decimal Hesapla(Siparis s) { /* ... */ } }
```

Dört sınıf da aynı sebeple değişir: **fiyatlandırma kuralları değiştiğinde.** Aynı aktör hepsini birden değiştirir. Bunları ayırmak, tek bir değişikliği dört dosyaya yaymaktır — SRP'nin tam tersi.

Doğrusu:

```csharp
// Aynı aktöre hizmet eden hesaplamalar bir arada
public class SiparisFiyatlandirma
{
    public decimal AraToplam(Siparis s) => s.Kalemler.Sum(k => k.BirimFiyat * k.Adet);
    public decimal Indirim(Siparis s)   => /* kampanya kuralları */ 0m;
    public decimal Kdv(decimal matrah)  => matrah * 0.20m;
    public decimal Kargo(Siparis s)     => s.Kalemler.Sum(k => k.Adet) > 5 ? 0m : 49.90m;

    public decimal GenelToplam(Siparis s)
    {
        var matrah = AraToplam(s) - Indirim(s);
        return matrah + Kdv(matrah) + Kargo(s);
    }
}
```

Tek sınıf, dört metot, tek değişim sebebi. SRP'ye uygun.

### Doğru soru: "kim ister, ne değişir"

Bir sınıfın SRP'ye uyup uymadığını anlamak için şunları sor:

| Soru | Uyumsuzluk işareti |
|---|---|
| Bu sınıfı kim değiştirmek ister? | Cevapta birden fazla departman varsa |
| Şu iki metot hiç birlikte değişir mi? | Asla değişmiyorsa |
| Sınıfın adında "ve" var mı? | `KullaniciKaydetVeMailGonder` |
| Sınıfın adı çok genel mi? | `Manager`, `Helper`, `Utility`, `Processor` |
| Sınıfın `using` listesi ne kadar uzun? | Veritabanı + SMTP + PDF + HTTP bir aradaysa |

> **Uyarı:** `Manager`, `Helper`, `Service`, `Utils` gibi isimler tek başına suç değildir; ama bu isimler genelde "buraya ne koyacağımı bilemedim" demektir. Bir sınıfa isim koymakta zorlanıyorsan, sebebi çoğunlukla sınıfın birden fazla iş yapmasıdır.

### Aynı sebeple değişen kod bir arada dursun

SRP'nin az söylenen ikinci yarısı budur. İlke sadece "ayır" demez, "aynı sebeple değişeni **birleştir**" de der. Buna **common closure (ortak kapanma)** denir: birlikte değişen şeyler birlikte dursun.

Örnek: TC kimlik numarası kuralı. Uzunluk kontrolü bir validator'da, biçimlendirme bir `StringHelper`'da, algoritma doğrulaması serviste duruyorsa, kural değiştiğinde üç dosya açman gerekir. Üçü de aynı sebeple değişir; öyleyse tek bir `TcKimlikNo` tipinde toplanmalıdır.

> **Bu benzetme şurada bozulur:** Muhtar benzetmesi "aktör" fikrini iyi anlatır ama bir şeyi atlar. Muhtarlıkta görevler resmî olarak tanımlıdır, kim neyi isteyebilir bellidir. Yazılımda aktörler çoğu zaman **görünmezdir**; kodun yüzünde "bu metodu pazarlama ister" yazmaz. Aktörü keşfetmek için geçmişe bakman gerekir: bu dosya son bir yılda hangi taleplerle değişmiş? Git geçmişi, SRP analizinin en iyi aracıdır.

---

## 3. SRP Pratiği: God Class ve Fat Controller

> **Benzetme —** Küçük bir kasabadaki tek tamirciyi düşün. Araba tamir eder, çamaşır makinesi söker, anahtar çoğaltır, yaz gelince klima takar. İş yürür, ta ki adam hastalanana kadar. O gün kasabada hiçbir şey tamir edilemez. Dahası: adam klima bilgisini güncellemek için kursa gitse, o hafta araba tamiri de durur. Tek kişide toplanan yetkinlik, tek noktadan kırılır.

**Basitçe:** Her şeyi yapan sınıfa **God class (tanrı sınıf)** denir. Kodun her yerinden çağrılır, her şeyi bilir, kimse dokunmaya cesaret edemez. MVC'de aynı şeyin controller hâline **fat controller (şişman controller)** denir.

**Teknik olarak:** **God class** — Çok fazla sorumluluk biriktirmiş, yüzlerce/binlerce satırlık, yüksek bağlılığa sahip sınıf. Belirtileri: çok sayıda alan (field), birbiriyle ilgisiz metotlar, uzun `using` listesi, değiştirmesi korkutucu.

### Kötü kod: fat controller

Bu, MVC projelerinde en sık gördüğün hâl:

```csharp
// KÖTÜ — controller her şeyi yapıyor
public class SiparisController : Controller
{
    private readonly MvcDbContext _context = new MvcDbContext();

    [HttpPost]
    public IActionResult Olustur(SiparisViewModel model)
    {
        // 1) Doğrulama
        if (model.Kalemler == null || !model.Kalemler.Any())
            return View(model);
        if (model.Kalemler.Any(k => k.Adet <= 0))
            return View(model);

        // 2) Stok kontrolü — veri erişimi
        foreach (var k in model.Kalemler)
        {
            var urun = _context.Urunler.Find(k.UrunId);
            if (urun == null || urun.Stok < k.Adet)
            {
                ModelState.AddModelError("", "Stok yetersiz");
                return View(model);
            }
        }

        // 3) Fiyat hesabı — iş kuralı
        decimal toplam = 0;
        foreach (var k in model.Kalemler)
        {
            var urun = _context.Urunler.Find(k.UrunId);
            var satir = urun!.Fiyat * k.Adet;
            if (k.Adet >= 10) satir *= 0.90m;      // toptan indirimi
            toplam += satir;
        }
        toplam *= 1.20m;                            // KDV
        if (toplam < 500) toplam += 49.90m;         // kargo

        // 4) Kayıt + 5) stok düşme — veri erişimi
        var siparis = new Siparis { Toplam = toplam, Tarih = DateTime.Now };
        _context.Siparisler.Add(siparis);
        foreach (var k in model.Kalemler)
            _context.Urunler.Find(k.UrunId)!.Stok -= k.Adet;
        _context.SaveChanges();

        // 6) E-posta — altyapı
        var smtp = new SmtpClient("smtp.sirket.com", 587)
        {
            Credentials = new NetworkCredential("no-reply@sirket.com", "P@ssw0rd")
        };
        smtp.Send("no-reply@sirket.com", model.Email, "Siparişiniz alındı",
                  $"Sipariş no: {siparis.Id}");

        // 7) Loglama — altyapı
        System.IO.File.AppendAllText("C:\\loglar\\siparis.txt", $"{siparis.Id}\n");

        return RedirectToAction("Basarili");
    }
}
```

Bu metodun değişme sebeplerini sayalım:

| Değişiklik | Aktör |
|---|---|
| Toptan indirim oranı %10 → %15 | Ticaret ekibi |
| KDV oranı değişikliği | Mevzuat / muhasebe |
| Kargo bedava eşiği 500 → 750 | Pazarlama |
| SMTP sunucusu değişti | Sistem yönetimi |
| Log dosya yerine veritabanına | Altyapı ekibi |
| Stok kontrolü rezervasyonlu olsun | Operasyon |

Altı farklı aktör, tek metot. SRP'nin altı ayrı ihlali. Ve her değişiklik aynı dosyayı açmayı gerektiriyor — yani her değişiklik diğerlerini bozma riski taşıyor.

> **Güvenlik notu:** Yukarıdaki kodda SMTP parolası kaynak koda gömülü. Bu, SRP'den bağımsız ayrı bir hatadır ama fat controller'larda çok sık görülür: her şey aynı yerde olunca sır da oraya sızar. Yapılandırma değerleri `appsettings.json` ve `IConfiguration` üzerinden, parolalar ise **user secrets** veya ortam değişkeninden okunur.

### İyi kod: sorumlulukları ayır

```csharp
// İYİ — controller sadece HTTP işini yapıyor
public class SiparisController : Controller
{
    private readonly ISiparisServisi _siparisServisi;

    public SiparisController(ISiparisServisi siparisServisi)
        => _siparisServisi = siparisServisi;

    [HttpPost]
    public async Task<IActionResult> Olustur(SiparisViewModel model)
    {
        if (!ModelState.IsValid) return View(model);

        var sonuc = await _siparisServisi.OlusturAsync(model.ToKomut());

        if (!sonuc.Basarili)
        {
            ModelState.AddModelError("", sonuc.Hata!);
            return View(model);
        }

        return RedirectToAction("Basarili", new { id = sonuc.SiparisId });
    }
}
```

Controller'ın tek sorumluluğu kaldı: **HTTP isteğini karşıla, sonucu HTTP cevabına çevir.** Model geçerli mi, hangi view, hangi redirect — hepsi bu.

İş kuralı servise taşındı:

```csharp
// C# 12 primary constructor — bağımlılıklar tek satırda
public class SiparisServisi(
    IStokKontrolu stok,
    ISiparisFiyatlandirma fiyat,
    ISiparisDeposu depo,
    IBildirimGonderici bildirim,
    ILogger<SiparisServisi> log) : ISiparisServisi
{
    public async Task<SiparisSonucu> OlusturAsync(SiparisKomutu komut)
    {
        var stokSonucu = await stok.KontrolEtAsync(komut.Kalemler);
        if (!stokSonucu.Yeterli)
            return SiparisSonucu.Basarisiz($"Stok yetersiz: {stokSonucu.EksikUrunAdi}");

        var toplam = fiyat.GenelToplam(komut);
        var siparis = await depo.KaydetAsync(komut, toplam);

        await stok.DusAsync(komut.Kalemler);
        await bildirim.SiparisAlindiAsync(komut.Email, siparis.Id, toplam);

        log.LogInformation("Siparis {SiparisId} olusturuldu", siparis.Id);
        return SiparisSonucu.Basarili(siparis.Id);
    }
}
```

Şimdi tabloya dön:

| Değişiklik | Dokunulacak dosya |
|---|---|
| İndirim oranı | `SiparisFiyatlandirma` |
| KDV oranı | `SiparisFiyatlandirma` |
| SMTP ayarı | `EpostaBildirimGonderici` |
| Log hedefi | Yapılandırma (kod değil) |
| Stok kuralı | `StokKontrolu` |

Her değişiklik tek dosya. SRP'nin vaat ettiği tam olarak bu.

> **MvcCv notu:** Kendi projendeki controller'lara bu gözle bak. `GenericRepository` iyi bir başlangıç — veri erişimini controller'dan çıkarmışsın. Ama iş kuralı hâlâ controller'daysa (hesaplama, koşullu kayıt, e-posta), araya bir servis katmanı girmesi gerekiyor demektir. Repository "veriyi getir/kaydet" katmanıdır, "ne zaman kaydedilir" kararını vermez.

---

## 4. SRP'de Ayrıştırma Nasıl Yapılır

> **Benzetme —** Karışık bir çekmeceyi toplamayı düşün. İki yöntem var. Birincisi: her eşyaya ayrı kutu al — yirmi kutu, hepsi yarı boş, aradığını yine bulamazsın. İkincisi: eşyaları kullanım anına göre grupla — "dikiş" kutusu, "kırtasiye" kutusu, "ilaç" kutusu. İkinci yöntem doğrudur, çünkü insan eşyayı **türüne göre değil ihtiyacına göre** arar.

**Basitçe:** Ayrıştırırken "bu metot hangi kategoriye girer" diye sorma. "Bu metot kimin talebiyle değişir, hangi metotlarla birlikte değişir" diye sor.

**Teknik olarak:** Ayrıştırmanın pratik adımları şöyle işler.

### Adım 1: Değişim sebeplerini yaz

Metodun içindeki her bloğun yanına, kim tarafından değiştirileceğini yaz. Aynı etiketi alan bloklar bir araya gider.

### Adım 2: Katman sınırlarını tanı

Çoğu .NET projesinde ayrım doğal olarak şu hatlardan geçer:

| Katman | Sorumluluk | Bilmemesi gereken |
|---|---|---|
| **Controller / Endpoint** | HTTP: model bağlama, durum kodu, view seçimi | İş kuralı, SQL |
| **Application / Service** | İş akışının sırası, işlem (transaction) sınırı | HTTP, view |
| **Domain** | İş kuralları, doğrulama, hesaplama | Veritabanı, HTTP |
| **Infrastructure** | Veritabanı, SMTP, dosya, dış servis | İş kuralı |

En sık yapılan hata, iş kuralının infrastructure'a (repository'ye) ya da controller'a sızmasıdır.

### Adım 3: Veri yapısını ve davranışı ayrı düşün

Varlık sınıfının kendi kaydını yapması, kendi PDF'ini üretmesi ve kendi e-postasını atması klasik bir SRP ihlalidir. Üç ayrı sebeple değişir: şema, şablon, altyapı.

```csharp
// KÖTÜ — varlık hem veri hem kalıcılık hem sunum taşıyor
public class Fatura
{
    public decimal Tutar { get; set; }

    public void VeritabaninaKaydet() { /* SQL */ }     // kalıcılık
    public string PdfOlustur() { /* şablon */ }        // sunum
    public void MailAt(string adres) { /* SMTP */ }    // altyapı
}

// İYİ — kalıcılık ve sunum dışarı çıkar, iş kuralı varlıkta kalır
public class Fatura
{
    public decimal Tutar { get; private set; }
    public FaturaDurumu Durum { get; private set; }

    public void Kesinlestir()
    {
        if (Durum != FaturaDurumu.Taslak)
            throw new InvalidOperationException("Sadece taslak fatura kesinleştirilebilir.");
        Durum = FaturaDurumu.Kesin;
    }
}

public interface IFaturaDeposu    { Task KaydetAsync(Fatura fatura); }
public interface IFaturaYazdirici { byte[] PdfUret(Fatura fatura); }
```

`Kesinlestir` metodunun sınıfta **kalmasına** dikkat et: o bir iş kuralıdır, faturanın kendi doğasına aittir. SRP "sınıfta hiç metot olmasın" demez; "yabancı sorumluluk taşımasın" der.

> **Bu benzetme şurada bozulur:** Çekmece benzetmesinde eşyalar birbirinden bağımsızdır; makası kırtasiye kutusuna koyduğunda dikiş kutusu etkilenmez. Kodda ise parçalar birbirine **referans verir**. Bir metodu başka sınıfa taşıdığında, onu kullanan her yer o sınıfı tanımak zorunda kalır. Yani ayırmak, yeni bir bağımlılık doğurur. Bu yüzden "ayırmak her zaman iyidir" cümlesi kodda geçerli değildir.

---

## 5. SRP'nin Sınırı: Ne Kadar Bölmek Fazla

> **Benzetme —** Bir terzi düşün. Ceketi parçalara ayırarak diker: ön beden, arka beden, kol, yaka, astar. Bu doğru bir bölmedir, her parça ayrı kalıba sahiptir. Şimdi biri çıkıp "daha da bölelim" dese — her dikiş arasını ayrı parça saysak — ortaya ceket değil yamalı bohça çıkar. Parça sayısı arttıkça birleştirme maliyeti artar. Doğru bölme, parçanın **kendi başına anlamlı** olduğu yerde durur.

**Basitçe:** Çok bölmek de bir hatadır. Her sınıfın tek metodu olduğu, o metodun da üç satır olduğu bir kod tabanında hiçbir şey okunamaz. İş akışını takip etmek için yirmi dosya açman gerekir.

**Teknik olarak:** Aşırı bölmenin belirtileri ve maliyetleri.

| Belirti | Sonuç |
|---|---|
| Sınıfların çoğunun tek metodu var | Akışı izlemek için dosya dosya gezmek gerekir |
| Arayüzlerin tek uygulaması var ve ikincisi asla gelmedi | Boşuna dolaylılık |
| Bir isteği anlamak için 8+ dosya açılıyor | Bilişsel yük |
| `IXYuklemeServisiFactoryProvider` gibi isimler | Soyutlamanın soyutlaması |
| Her sınıf sadece bir sonrakine delege ediyor | Katman değil, boru hattı |

### Fazla bölünmüş kod

```csharp
// AŞIRI BÖLÜNMÜŞ — her adım ayrı sınıf, hiçbiri kendi başına anlamlı değil
public class KullaniciAdiAlici { public string Al(KayitFormu f) => f.KullaniciAdi.Trim(); }
public class KullaniciAdiKucukHarfYapici { public string Yap(string s) => s.ToLowerInvariant(); }
public class KullaniciAdiUzunlukKontrolcusu { public bool Kontrol(string s) => s.Length >= 3; }
public class KullaniciAdiKarakterKontrolcusu { public bool Kontrol(string s) => s.All(char.IsLetterOrDigit); }
public class KullaniciAdiDogrulayiciKoordinatoru { /* dört bağımlılık, dört satır iş */ }
```

Bu kodun tamamı şudur:

Bu kodun tamamı şudur:

```csharp
// YETERİNCE BÖLÜNMÜŞ — tek bir kavram, tek bir sınıf
public static class KullaniciAdi
{
    public static bool Gecerli(string? girdi, out string normalize)
    {
        normalize = (girdi ?? string.Empty).Trim().ToLowerInvariant();
        return normalize.Length >= 3
            && normalize.Length <= 20
            && normalize.All(char.IsLetterOrDigit);
    }
}
```

Beş sınıf yerine bir sınıf. Kural değiştiğinde açılacak dosya sayısı: bir. Bu da SRP'ye uygundur — çünkü beş parçanın hepsi **aynı sebeple** değişir: kullanıcı adı kuralı değiştiğinde.

### Durma noktasını bulmak

Şu üç testi uygula:

1. **İsim testi:** Sınıfa "ve" içermeyen, doğal bir isim verebiliyor musun?
2. **Birlikte değişme testi:** İki parça son bir yılda hep birlikte mi değişmiş? Öyleyse ayırma, birleştir.
3. **Tek başına anlam testi:** Bu sınıf ne işe yarar sorusuna tek cümlede cevap verebiliyor musun?

> **Pratik kural:** Bölmeye **acı hissettiğinde** başla. Dosya büyüdüğü için değil, bir değişikliği yaparken alakasız kodu okumak zorunda kaldığın için böl.

---

## 6. OCP: Değişime Kapalı, Genişlemeye Açık

> **Benzetme —** Prizi düşün. Duvardaki priz yıllardır aynıdır; ne şarj aleti taktın diye değişir, ne süpürge. Yeni bir cihaz çıktığında duvarı kırıp tesisatı yenilemezsin, cihaz prize uyar. Priz **değişikliğe kapalıdır**; sisteme yeni cihaz eklemek ise **açıktır**. Anlaşma noktası, priz deliğinin şeklidir — yani arayüz.

**Basitçe:** Yeni bir durum eklemek gerektiğinde, çalışan eski koda dokunmak zorunda kalmamalısın. Yeni durumu yeni bir sınıf olarak ekleyip sisteme tanıtabilmelisin.

**Teknik olarak:** **OCP (Open/Closed Principle — Açık/Kapalı İlkesi)** — Yazılım varlıkları (sınıf, modül, fonksiyon) **genişlemeye açık, değişikliğe kapalı** olmalıdır. Yani davranış eklenebilmeli, ama mevcut kaynak kod değiştirilmeden.

Neden "kapalı"? Çünkü çalışan koda her dokunuş bir risktir. Test edilmiş, üretimde aylardır sorunsuz çalışan bir metoda yeni bir `if` eklediğinde, eski davranışı bozmadığını ancak test ederek bilebilirsin. Yeni bir sınıf eklemek ise eski kodu hiç riske atmaz.

### Nasıl mümkün olur

Cevap **polimorfizm**dir. Değişken olan kısmı bir soyutlamanın arkasına koyarsın; çağıran taraf soyutlamayı bilir, somut uygulamaları bilmez.

```csharp
// Soyutlama: ödeme yöntemi
public interface IOdemeYontemi
{
    string Ad { get; }
    Task<OdemeSonucu> TahsilEtAsync(decimal tutar, OdemeBilgisi bilgi);
}
```

Yeni bir ödeme yöntemi geldiğinde yeni bir sınıf yazarsın. `IOdemeYontemi` arayüzü değişmez, onu kullanan servis değişmez.

### Abstract'a programlama

**"Abstract'a programlama" (program to an abstraction)** — Kodun, somut bir sınıfın adını değil, soyut bir tipin adını kullanması.

```csharp
// Somuta programlama — genişlemeye kapalı
public class Raporlayici
{
    public void Uret(Rapor r)
    {
        var pdf = new PdfYazici();      // somut tipe çivilenmiş
        pdf.Yaz(r);
    }
}

// Soyuta programlama — genişlemeye açık
public class Raporlayici(IRaporYazici yazici)
{
    public void Uret(Rapor r) => yazici.Yaz(r);
}
```

İkinci versiyonda Excel yazıcısı eklemek için `Raporlayici` sınıfına hiç dokunmazsın.

> **Not:** Soyutlama sadece arayüzle olmaz. `abstract class`, `delegate`, hatta bir `Func<T, TResult>` parametresi de soyutlamadır. En hafif olanı seç: tek bir davranış değişiyorsa `Func` yeterlidir, arayüz açmaya gerek yoktur.

```csharp
// En hafif soyutlama — delegate parametresi
public decimal Hesapla(Siparis s, Func<decimal, decimal> indirimKurali)
    => indirimKurali(s.AraToplam);
```

> **Bu benzetme şurada bozulur:** Priz benzetmesi arayüzün sabitliğini iyi anlatır ama sanki bu sabitlik bedavaymış gibi gösterir. Gerçekte priz standardı da değişir — Avrupa tipi, İngiliz tipi, USB-C. Kodda da arayüzü bir kere doğru tasarlayıp sonsuza dek dokunmamak nadirdir. OCP, arayüzün **hiç değişmeyeceğini** değil, **sık değişmeyeceğini** varsayar. Yanlış çizilmiş bir soyutlama, her yeni gereksinimde arayüzü değiştirmeyi gerektirir — o zaman OCP'nin hiçbir faydası kalmaz.

---

## 7. OCP Pratiği: switch Zincirinden Polimorfizme

> **Benzetme —** Nöbetçi eczane listesi düşün. Kötü yöntem: eczacının kapısına "pazartesi Ahmet, salı Mehmet, çarşamba Ayşe..." diye bir liste asmak. Yeni eczane açıldığında listeyi söküp yeniden yazman gerekir. İyi yöntem: her eczaneye "nöbet günün şu" diye bir tabela vermek ve merkezden "bugün nöbetçi kim" diye sormak. Yeni eczane geldiğinde kendi tabelasını asar, merkezdeki sistem değişmez.

**Basitçe:** Büyüyen bir `switch` ya da `if/else if` zinciri, OCP ihlalinin en görünür işaretidir. Her yeni durum o zincire bir dal eklemeyi gerektirir — yani çalışan koda dokunmayı.

**Teknik olarak:** Bu kalıba **conditional complexity (koşul karmaşası)** denir ve çözümü genelde **Strategy pattern**'dir.

### Kötü kod: büyüyen switch

```csharp
// KÖTÜ — her yeni ödeme yöntemi bu metodu değiştirmeyi gerektirir
public class OdemeIslemcisi
{
    public OdemeSonucu Tahsil(string yontem, decimal tutar, OdemeBilgisi bilgi)
    {
        switch (yontem)
        {
            case "KrediKarti":
                if (bilgi.KartNo.Length != 16) return OdemeSonucu.Hata("Kart no hatalı");
                var komisyon = tutar * 0.018m;
                // banka servisine git
                return OdemeSonucu.Ok(tutar + komisyon);

            case "Havale":
                if (string.IsNullOrEmpty(bilgi.Iban)) return OdemeSonucu.Hata("IBAN gerekli");
                return OdemeSonucu.Beklemede();

            case "KapidaOdeme":
                if (tutar > 5000) return OdemeSonucu.Hata("Kapıda ödeme limiti aşıldı");
                return OdemeSonucu.Ok(tutar + 15m);

            default:
                throw new NotSupportedException($"Bilinmeyen ödeme yöntemi: {yontem}");
        }
    }
}
```

Sorunlar tek tek:

- Yeni yöntem (mobil ödeme, kripto, taksitli kart) eklemek bu metodu **değiştirmeyi** gerektirir.
- Metot büyüdükçe test etmek zorlaşır; her yeni dal tüm metodu yeniden test etmeyi gerektirir.
- Aynı `switch` genelde tek yerde kalmaz. Yakında "ödeme yöntemine göre ikon göster", "ödeme yöntemine göre iade et" diye ikinci ve üçüncü `switch` doğar. Biri güncellenip diğeri unutulur.
- `string` ile eşleşme yazım hatasına açıktır. `"KapidaOdeme"` ile `"Kapidaodeme"` derlenir, çalışmaz.

### İyi kod: strategy

```csharp
// Soyutlama
public interface IOdemeYontemi
{
    string Kod { get; }
    OdemeSonucu Tahsil(decimal tutar, OdemeBilgisi bilgi);
}

// Her yöntem kendi kuralını ve kendi bağımlılığını bilir
public class KrediKartiOdeme(IBankaServisi banka) : IOdemeYontemi
{
    public string Kod => "KrediKarti";

    public OdemeSonucu Tahsil(decimal tutar, OdemeBilgisi bilgi)
    {
        if (bilgi.KartNo?.Length != 16)
            return OdemeSonucu.Hata("Kart numarası 16 hane olmalı");

        var komisyon = tutar * 0.018m;
        return banka.Cek(bilgi.KartNo, tutar + komisyon)
            ? OdemeSonucu.Ok(tutar + komisyon)
            : OdemeSonucu.Hata("Banka işlemi reddetti");
    }
}

public class HavaleOdeme : IOdemeYontemi
{
    public string Kod => "Havale";
    public OdemeSonucu Tahsil(decimal tutar, OdemeBilgisi bilgi)
        => string.IsNullOrWhiteSpace(bilgi.Iban)
            ? OdemeSonucu.Hata("IBAN gerekli")
            : OdemeSonucu.Beklemede();
}

public class KapidaOdeme : IOdemeYontemi
{
    public string Kod => "KapidaOdeme";
    public OdemeSonucu Tahsil(decimal tutar, OdemeBilgisi bilgi)
        => tutar > 5000m
            ? OdemeSonucu.Hata("Kapıda ödeme limiti 5.000 TL")
            : OdemeSonucu.Ok(tutar + 15m);
}
```

Çağıran taraf artık hiçbir yöntemi bilmez:

```csharp
public class OdemeIslemcisi
{
    private readonly IReadOnlyDictionary<string, IOdemeYontemi> _yontemler;

    // Tüm uygulamalar konteynerden gelir, sözlüğe dönüştürülür
    public OdemeIslemcisi(IEnumerable<IOdemeYontemi> yontemler)
        => _yontemler = yontemler.ToDictionary(y => y.Kod);

    public OdemeSonucu Tahsil(string kod, decimal tutar, OdemeBilgisi bilgi)
        => _yontemler.TryGetValue(kod, out var yontem)
            ? yontem.Tahsil(tutar, bilgi)
            : OdemeSonucu.Hata($"Desteklenmeyen ödeme yöntemi: {kod}");
}
```

Yeni yöntem eklemek artık şu kadar:

```csharp
// Yeni sınıf — mevcut hiçbir dosya değişmedi
public class MobilOdeme : IOdemeYontemi
{
    public string Kod => "Mobil";
    public OdemeSonucu Tahsil(decimal tutar, OdemeBilgisi bilgi)
        => tutar > 750m ? OdemeSonucu.Hata("Mobil ödeme limiti 750 TL") : OdemeSonucu.Ok(tutar);
}

// Kayıt (Program.cs)
builder.Services.AddScoped<IOdemeYontemi, MobilOdeme>();
```

`OdemeIslemcisi` değişmedi, diğer üç yöntem değişmedi. OCP'nin vaadi bu.

### Hangi switch kalabilir

Her `switch` kötü değildir. Şu durumlarda olduğu gibi bırak:

| Durum | Neden kalabilir |
|---|---|
| Seçenek kümesi kapalı ve sabit (haftanın günleri, para birimi kodu) | Yeni dal gelmeyecek |
| `switch` sadece bir **eşleme** yapıyor (enum → metin) | Davranış değil veri |
| Tek yerde, üç dal, her dal bir satır | Soyutlamanın maliyeti daha yüksek |

> **Ayırt edici soru:** `switch`'in her dalı **farklı bir iş yapıyorsa** (farklı servis çağırıyor, farklı kural uyguluyor) strategy'ye çıkar. Her dal **aynı işin farklı değerini** üretiyorsa bırak.

> **Bu benzetme şurada bozulur:** Nöbetçi eczane benzetmesinde her eczane bağımsızdır ve merkez sadece listeyi tutar. Kodda ise strategy'lerin çoğu zaman **ortak bir şeye** ihtiyacı olur: aynı log, aynı işlem kaydı, aynı hata biçimi. Bu ortaklığı her sınıfa kopyalarsan, OCP'yi kazanıp DRY'ı kaybedersin. Çözüm genelde soyut bir temel sınıf (`abstract class OdemeYontemiTabani`) ya da bir sarmalayıcıdır (decorator).

---

## 8. OCP'nin Bedeli: Spekülatif Soyutlama

> **Benzetme —** Yeni bir ev yaptırıyorsun. Müteahhit "ileride üç kat daha çıkarsın diye temeli on katlık atalım" diyor. İhtimal varsa mantıklı. Ama sen tek katlı yazlık yaptırıyorsan, on katlık temel parayı çöpe atmaktır — hem de hiç kullanmayacağın bir esneklik için.

**Basitçe:** Her yeri soyutlamak OCP uygulamak değildir. Gerçekleşmeyecek değişikliklere karşı kurulan esneklik, sadece karmaşa üretir.

**Teknik olarak:** **Speculative generality (spekülatif genellik)** — Henüz var olmayan bir ihtiyaç için kurulan soyutlama. Martin Fowler'ın kod kokuları listesinde yer alır.

### Gereksiz soyutlama

```csharp
// GEREKSİZ — tek uygulaması var, ikincisi hiç gelmeyecek
public interface ITarihSaglayici { DateTime SimdiKi { get; } }
public interface IGuidUretici { Guid Uret(); }
public interface IStringBirlestirici { string Birlestir(params string[] parcalar); }
public interface IMatematikServisi { decimal Yuvarla(decimal deger, int basamak); }
```

`IStringBirlestirici` ve `IMatematikServisi` saçmadır: `string.Join` ve `Math.Round` zaten var, değişmeyecek, test edilmesi gerekmez.

Ama `ITarihSaglayici` **saçma değildir**. Sebebi soyutlama sevdası değil, testtir: `DateTime.Now` testi imkânsız kılar. .NET 8 ile bunun standart karşılığı geldi.

```csharp
// .NET 8+ — kendi arayüzünü açmana gerek yok, TimeProvider var
public class AboneligKontrolu(TimeProvider saat)
{
    public bool SuresiDoldu(Abonelik a) => a.BitisTarihi < saat.GetUtcNow();
}

// Testte sahte saat verilir
var sahteSaat = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));
var kontrol = new AboneligKontrolu(sahteSaat);
```

> **Not:** `TimeProvider` .NET 8 ile `System` ad alanına geldi. `FakeTimeProvider` ise `Microsoft.Extensions.TimeProvider.Testing` paketindedir. Daha eski hedeflerde kendi `ITarihSaglayici` arayüzünü açman normaldir.

### Ne zaman soyutla

| Soyutla | Soyutlama |
|---|---|
| Dış dünyaya dokunan şey (DB, ağ, dosya, saat, rastgele) | Saf hesaplama |
| Gerçekten iki veya daha fazla uygulama var | "Belki ileride" |
| Test için yerine sahte koyman gerekiyor | Test edilmesi anlamsız kod |
| İş kuralı sık değişiyor ve çeşitleniyor | Mevzuatla sabitlenmiş tek kural |

**Rule of three (üç kuralı)** — Aynı kalıbı üçüncü kez yazdığında soyutla. İkinci kez tesadüf olabilir, üçüncüsü desen demektir.

### Soyutlamanın gizli maliyetleri

- **Gezinme maliyeti:** `IUrunServisi.Getir` üzerinde "Go to Definition" seni arayüze götürür, gerçek koda değil.
- **Yanlış soyutlama maliyeti:** Kötü çizilmiş bir arayüzü sökmek, hiç soyutlamamaktan pahalıdır. Sandi Metz'in sözüyle: tekrar, yanlış soyutlamadan ucuzdur.
- **Anlam kaybı:** Beş katman delege eden bir zincirde asıl işin nerede yapıldığı kaybolur.

> **Pratik kural:** Soyutlamayı **geriye dönük** ekle. Önce somut yaz, çalıştır. İkinci uygulama gerçekten geldiğinde, elindeki iki somut sınıfın ortak yanına bakarak arayüzü çıkar. Böyle çizilen arayüz, hayal ederek çizilenden neredeyse her zaman daha doğru olur.

> **Bu benzetme şurada bozulur:** Ev temeli benzetmesi, sonradan eklemenin imkânsız olduğunu ima eder — beton döküldükten sonra temel güçlendirilemez. Kod öyle değildir. Kodda soyutlamayı **sonradan eklemek** genellikle kolaydır; modern IDE'ler "Extract Interface" ile bunu saniyede yapar. Bu yüzden yazılımda "ihtimale karşı şimdi yapalım" argümanı inşaattakinden çok daha zayıftır.

---

## 9. LSP: Alt Tip İkame İlkesi

> **Benzetme —** Bir kargo şirketi düşün. "Kargo görevlisi" tanımı şudur: paketi alır, adrese götürür, imza alır. Şimdi yeni bir görevli tipi geliyor: "ekspres görevli". Daha hızlı götürüyor, iyi. Ama bir kural koymuş: "5 kilodan ağır paket almam" ve "imza almam, kapıya bırakırım". Şirketin sistemi tüm görevlileri aynı kabul ediyordu; şimdi her yere "bu ekspres mi" kontrolü eklemek gerekiyor. Ekspres görevli, görevli tanımının yerine **geçemiyor**. LSP ihlali budur.

**Basitçe:** Bir üst tipin beklendiği yere alt tipini koyduğunda, program aynı şekilde çalışmaya devam etmelidir. Çağıran taraf "acaba hangi alt tip geldi" diye kontrol etmek zorunda kalmamalıdır.

**Teknik olarak:** **LSP (Liskov Substitution Principle — Liskov Yerine Geçme İlkesi)** — `S`, `T`'nin alt tipiyse, `T` tipindeki nesneler programın doğruluğunu bozmadan `S` tipindeki nesnelerle değiştirilebilmelidir. Barbara Liskov, 1987.

Pratik hâli: **alt sınıf, üst sınıfın sözleşmesini daraltamaz, genişletebilir.**

### İhlalin en tanınmış hâli: kare-dikdörtgen

```csharp
// KÖTÜ — matematikte kare dikdörtgendir, kodda değildir
public class Dikdortgen
{
    public virtual int Genislik { get; set; }
    public virtual int Yukseklik { get; set; }
    public int Alan => Genislik * Yukseklik;
}

public class Kare : Dikdortgen
{
    public override int Genislik
    {
        get => base.Genislik;
        set { base.Genislik = value; base.Yukseklik = value; }   // sessizce diğerini de değiştiriyor
    }

    public override int Yukseklik
    {
        get => base.Yukseklik;
        set { base.Genislik = value; base.Yukseklik = value; }
    }
}
```

Çağıran taraf şu varsayımla yazılmıştır:

```csharp
void GenisligiIkiyeKatla(Dikdortgen d)
{
    var eskiYukseklik = d.Yukseklik;
    d.Genislik = d.Genislik * 2;

    // Dikdörtgen için doğru, Kare için YANLIŞ
    Debug.Assert(d.Yukseklik == eskiYukseklik);
}
```

`Kare` geldiğinde yükseklik de değişir, varsayım çöker. Kod derlenir; test ya patlar ya da daha kötüsü patlamaz, sessizce yanlış sonuç üretir. Çözüm mirası kaldırmaktır:

```csharp
// İYİ — ikisi de ayrı, ortak davranış arayüzle
public interface IAlanHesaplanabilir { int Alan { get; } }

public sealed class Dikdortgen(int genislik, int yukseklik) : IAlanHesaplanabilir
{
    public int Genislik { get; } = genislik;
    public int Yukseklik { get; } = yukseklik;
    public int Alan => Genislik * Yukseklik;
}

public sealed class Kare(int kenar) : IAlanHesaplanabilir
{
    public int Kenar { get; } = kenar;
    public int Alan => Kenar * Kenar;
}
```

Dikkat: nesneler **değişmez (immutable)** hâle geldi. Setter olmayınca "genişliği değiştirince yükseklik ne olur" sorusu da ortadan kalkar. LSP ihlallerinin büyük kısmı değişebilir durumdan (mutable state) doğar.

> **Ders:** Gerçek dünyadaki "bir X'tir" ilişkisi kodda kalıtım gerektirmez. Kalıtım, **davranış sözleşmesi** ilişkisidir; sınıflandırma ilişkisi değildir. Penguen kuştur ama `Kus.Uc()` metodu varsa penguen `Kus`'tan türememeli.

> **Bu benzetme şurada bozulur:** Kargo görevlisi benzetmesi, ihlalin hep görünür olduğunu ima eder — ekspres görevli kuralını açıkça söylüyor. Kodda LSP ihlalleri genelde **sessizdir**. Alt sınıf hiçbir uyarı vermez, derleyici şikâyet etmez, hatta çoğu test geçer. İhlal ancak belirli bir girdi kombinasyonunda ortaya çıkar. Bu yüzden LSP, SOLID'in en zor fark edilen maddesidir.

---

## 10. LSP İhlalinin Biçimleri

> **Benzetme —** Bir oto tamircisinin kapısında "her marka araç" yazıyor. İçeri girince "dizel bakmıyoruz" diyor. Tabela bir sözleşmedir; tamirci onu daraltmıştır. İkinci tamirci "her marka" diyor, gerçekten de bakıyor, üstelik yıkama da yapıyor — sözleşmeyi genişletmiş, kimse şikâyetçi değil. Sözleşmeyi **daraltmak** ihlaldir, **genişletmek** değildir.

**Basitçe:** LSP ihlali dört biçimde gelir: alt sınıf daha katı giriş şartı koyar, daha zayıf çıktı garantisi verir, beklenmedik istisna fırlatır, ya da metodu hiç uygulamaz.

**Teknik olarak:** Sözleşme (contract) üç parçadan oluşur.

| Parça | Anlamı | Alt sınıf ne yapabilir |
|---|---|---|
| **Ön koşul (precondition)** | Metot çağrılmadan önce doğru olması gerekenler | Sadece **gevşetebilir** |
| **Son koşul (postcondition)** | Metot bittikten sonra garanti edilenler | Sadece **güçlendirebilir** |
| **Değişmez (invariant)** | Nesnenin ömrü boyunca doğru kalan | Korumak zorunda |

Kısaca: **girişte daha hoşgörülü, çıkışta daha cömert ol.**

### Biçim 1: Ön koşulu güçlendirmek

```csharp
// KÖTÜ — alt sınıf daha katı giriş şartı koyuyor
public class Kullanici
{
    public virtual void SifreBelirle(string sifre)
    {
        if (sifre.Length < 6) throw new ArgumentException("En az 6 karakter");
        // kaydet
    }
}

public class YoneticiKullanici : Kullanici
{
    public override void SifreBelirle(string sifre)
    {
        // Ön koşul GÜÇLENDİ: artık 12 karakter gerekiyor
        if (sifre.Length < 12) throw new ArgumentException("En az 12 karakter");
        // kaydet
    }
}
```

`Kullanici` bekleyen bir kod 8 karakterlik parola verir, çalışacağını varsayar. `YoneticiKullanici` geldiğinde patlar. Çağıran taraf hiçbir şey yanlış yapmadı.

```csharp
// İYİ — kural kalıtıma değil, enjekte edilen politikaya taşındı
public class Kullanici(ISifrePolitikasi politika)
{
    public void SifreBelirle(string sifre)
    {
        var sonuc = politika.Dogrula(sifre);
        if (!sonuc.Gecerli) throw new ArgumentException(sonuc.Mesaj);
        // kaydet
    }
}

public interface ISifrePolitikasi { DogrulamaSonucu Dogrula(string sifre); }
public class StandartSifrePolitikasi : ISifrePolitikasi { /* 6 karakter */ }
public class YoneticiSifrePolitikasi : ISifrePolitikasi { /* 12 karakter + simge */ }
```

Artık her `Kullanici` aynı şekilde davranır; farklılık kalıtımda değil, enjekte edilen politikada.

### Biçim 2: Son koşulu zayıflatmak

```csharp
// KÖTÜ — üst sınıf "asla null dönmez" diyor, alt sınıf null dönüyor
public class UrunDeposu
{
    // Son koşul: her zaman bir liste döner, boş olabilir ama null olmaz
    public virtual IReadOnlyList<Urun> Listele() => new List<Urun>();
}

public class OnbellekliUrunDeposu : UrunDeposu
{
    public override IReadOnlyList<Urun> Listele()
    {
        if (!_onbellekDolu) return null!;   // son koşul ZAYIFLADI
        return _onbellek;
    }
}
```

Çağıran kod `foreach (var u in depo.Listele())` yazmıştır. Doğru yazmıştır. Yine de `NullReferenceException` alır.

```csharp
// İYİ — son koşul korunuyor: her durumda liste döner
public override IReadOnlyList<Urun> Listele()
{
    if (!_onbellekDolu) Doldur();
    return _onbellek;
}
```

Aynı kategoride diğer örnekler: üst sınıf "sıralı döner" der alt sınıf sırasız döndürür; üst sınıf "hep kaydeder" der alt sınıf bazen kaydetmez.

### Biçim 3: Beklenmedik istisna fırlatmak

```csharp
// KÖTÜ — üst tip dosya yazmayı vaat ediyor, alt tip reddediyor
public abstract class RaporDeposu
{
    public abstract void Kaydet(Rapor rapor);
}

public class SaltOkunurRaporDeposu : RaporDeposu
{
    public override void Kaydet(Rapor rapor)
        => throw new InvalidOperationException("Bu depo salt okunur.");
}
```

`RaporDeposu` bekleyen hiçbir kod bu istisnayı beklemez. `try/catch` yazması gereken yer, tip sisteminden bunu öğrenemez.

```csharp
// İYİ — yazma yeteneği ayrı bir sözleşme
public interface IRaporOkuyucu { Rapor Getir(int id); }
public interface IRaporYazici  { void Kaydet(Rapor rapor); }

public class DosyaRaporDeposu : IRaporOkuyucu, IRaporYazici { /* ikisini de yapar */ }
public class ArsivRaporDeposu : IRaporOkuyucu { /* sadece okur */ }
```

Artık "yazamaz" bilgisi tipte görünüyor. Yazmaya çalışan kod derlenmiyor bile — çalışma zamanında patlamaktan iyidir. Bu aynı zamanda yarın göreceğin **ISP**'nin de konusudur.

### Biçim 4: NotImplementedException kokusu

```csharp
// KÖTÜ — arayüz zorladı, sınıf uygulayamadı
public class SadeceEklemeliKoleksiyon<T> : ICollection<T>
{
    public void Add(T item) { /* çalışır */ }

    public bool Remove(T item)   => throw new NotImplementedException();
    public void Clear()          => throw new NotImplementedException();
    public bool Contains(T item) => throw new NotImplementedException();
}
```

`NotImplementedException` gövdesi, "bu sınıf bu arayüzü karşılamıyor" demenin kodla yazılmış hâlidir. Üç olasılık vardır:

1. **Yanlış arayüz seçilmiş.** Sınıfın ihtiyacı `ICollection<T>` değil, daha dar bir sözleşme.
2. **Arayüz fazla şişman.** Bölünmesi gerekiyor (ISP).
3. **Kalıtım yanlış kurulmuş.** Kompozisyon daha uygun.

```csharp
// İYİ — sadece yapabildiğini vaat et
public interface IEkleyici<T> { void Ekle(T item); }

public class SadeceEklemeliKoleksiyon<T> : IEkleyici<T>, IEnumerable<T>
{
    private readonly List<T> _liste = new();
    public void Ekle(T item) => _liste.Add(item);
    public IEnumerator<T> GetEnumerator() => _liste.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

Bunun .NET içinde bilinen bir örneği var: `Array`, `IList<T>` uygular ama `Add` çağırırsan `NotSupportedException` alırsın — dizinin boyutu sabittir. İlk sürümlerden kalma bir tasarım borcudur; bugün aynı ihtiyaç `IReadOnlyList<T>` ile karşılanır.

> **Tasarımda işine yarayacak sonuç:** Bir tip "bir şeyi yapamıyorsa", bunu çalışma zamanında istisnayla söylemek yerine, o sözleşmeyi hiç üstlenmemesi daha iyidir. Derleyicinin yakalayabildiği hata, kullanıcının yakaladığından ucuzdur.

> **Bu benzetme şurada bozulur:** Tamirci benzetmesinde tabelayı okuyan insandır, esneklik gösterebilir. Kodda çağıran taraf esneyemez: derlenmiş varsayımlarla çalışır. Ayrıca gerçek hayatta "dizel bakmıyoruz" bilgisi kapıda öğrenilir; kodda bu bilgi **ancak o senaryo üretimde çalıştığında** ortaya çıkar. LSP ihlalinin bedeli bu yüzden geç ödenir.

---

## Tek Bakışta Özet

- SOLID performans için değil, **değişikliğin maliyetini düşürmek** için vardır.
- SRP "bir sınıf bir iş yapsın" değil, "**değişmek için tek sebebi olsun**" demektir.
- Sorulacak doğru soru: bu kodu **kim** değiştirmek ister? Birden çok aktör varsa ayır.
- SRP ayırmayı da birleştirmeyi de söyler: aynı sebeple değişen kod bir arada dursun.
- Fat controller, SRP ihlalinin en sık görülen hâlidir: doğrulama + kural + veri + altyapı tek metotta.
- Controller'ın tek işi HTTP'dir; iş kuralı servise, veri erişimi repository'ye ait.
- Aşırı bölmek de ihlaldir: tek metotluk sınıflar, kullanılmayan arayüzler, delege zincirleri.
- OCP: yeni davranış **yeni sınıf** olarak eklenmeli, çalışan kod kurcalanmamalı.
- Büyüyen `switch`/`if` zinciri OCP ihlalinin görünür işaretidir; çözümü strategy'dir.
- Her dal **farklı iş** yapıyorsa strategy'ye çıkar; sadece **değer eşliyorsa** `switch` kalsın.
- Spekülatif soyutlama kod kokusudur. Soyutlamayı ihtimale göre değil ihtiyaca göre ekle.
- Yanlış soyutlama, tekrardan pahalıdır. Önce somut yaz, ikinci uygulama gelince çıkar.
- LSP: alt tip, üst tipin yerine geçtiğinde çağıran taraf bunu fark etmemeli.
- Alt sınıf ön koşulu **gevşetebilir**, son koşulu **güçlendirebilir**; tersi ihlaldir.
- `NotImplementedException` / `NotSupportedException` gövdesi, yanlış sözleşme işaretidir.
- Çağıran tarafta `is`/`as` ile alt tip ayıklaması görüyorsan polimorfizm çalışmıyor demektir.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| SOLID | Nesne yönelimli tasarımın beş ilkesi |
| SRP | Tek sorumluluk: değişmek için tek sebep |
| Aktör (actor) | Değişikliği talep eden taraf; departman ya da rol |
| OCP | Açık/kapalı: genişlemeye açık, değişikliğe kapalı |
| LSP | Alt tipin üst tipin yerine sorunsuz geçebilmesi |
| Coupling (bağlılık) | İki kod parçasının birbirine yapışıklığı |
| Cohesion (uyum) | Bir sınıf içindeki parçaların aynı işe hizmet etme derecesi |
| God class | Çok fazla sorumluluk biriktirmiş dev sınıf |
| Fat controller | İş kuralı ve altyapı taşıyan şişman controller |
| Polimorfizm | Aynı çağrının tipe göre farklı davranması |
| Strategy pattern | Değişken davranışı ayrı sınıflara taşıyan tasarım deseni |
| Ön koşul (precondition) | Metot çağrılmadan önce sağlanması gerekenler |
| Son koşul (postcondition) | Metot bittikten sonra garanti edilenler |
| Değişmez (invariant) | Nesne ömrü boyunca doğru kalan kural |
| Speculative generality | Var olmayan ihtiyaç için kurulan soyutlama |
| YAGNI | "İhtiyacın olmayacak" — erken soyutlamaya karşı kural |
| Rule of three | Aynı kalıbı üçüncü kez yazınca soyutla |
| Over-engineering | Problemden büyük çözüm kurma |
| Immutable | Oluşturulduktan sonra değişmeyen nesne |
| TimeProvider | .NET 8 ile gelen, saati soyutlayan standart tip |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "SRP: bir sınıf bir iş yapar" | Bir sınıfın değişmek için tek sebebi olur; birden çok metot olabilir |
| "Her metot ayrı sınıfa çıkarsa SRP'ye uyulur" | Aynı sebeple değişen metotları ayırmak SRP'nin tersidir |
| "SRP sadece ayırmayı söyler" | Aynı sebeple değişen kodu birleştirmeyi de söyler |
| "Repository iş kuralını da tutabilir" | Repository veri erişimidir; kural servis/domain katmanına aittir |
| "OCP için her sınıfın arayüzü olmalı" | Tek uygulaması olan arayüz genelde gereksiz dolaylılıktır |
| "Her `switch` OCP ihlalidir" | Sabit küme üzerinde değer eşleyen `switch` sorunsuzdur |
| "Soyutlama sonradan eklenemez, baştan düşünülmeli" | Tersi doğrudur: önce somut yaz, desen belirince çıkar |
| "Tekrar her zaman kötüdür" | Yanlış soyutlama tekrardan pahalıdır |
| "Matematikte kare dikdörtgense kodda da öyledir" | Kalıtım sınıflandırma değil davranış sözleşmesi ilişkisidir |
| "`NotImplementedException` geçici bir çözümdür" | Kalıcı olduğunda yanlış arayüz seçildiğinin kanıtıdır |
| "LSP sadece `class` kalıtımıyla ilgilidir" | Arayüz uygulamaları için de geçerlidir |
| "Alt sınıf doğrulamayı sıkılaştırabilir" | Ön koşulu güçlendirmek LSP ihlalidir |
| "SOLID'e ne kadar çok uyarsan o kadar iyi" | Aşırısı over-engineering üretir; YAGNI dengeler |

---

## Sonraki

→ `04-SOLID-ISP-DIP-ve-DI.md` (Perşembe)
