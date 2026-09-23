# Hafta 4 · EF Core ile CRUD ve Change Tracking

**Okuma süresi:** ~52 dk
**Neden bu konu:** CRUD'ı yazmak kolaydır; **neden çalıştığını** bilmek zordur. Özellikle güncelleme işleminde EF Core'un arka planda ne yaptığını anlamayan biri, veri kaybettiren hataları fark edemez.

> Hafta 3'te (SQL Server ve EF Core) modelleme ve sorgulama işlenecek. Bu not, MVC tarafından bakıldığında CRUD'ın nasıl çalıştığına odaklanır.

---

## Önce Basitçe

EF Core'u "veritabanına SQL yazan kütüphane" diye düşünürsen bu notun yarısı anlaşılmaz kalır. Daha doğru bir bakış şu: EF Core, senin adına **not tutan bir asistandır**. Veritabanından bir kayıt istediğinde onu getirir, ama getirmekle kalmaz — kenara "bu kaydı şu hâliyle getirdim" diye bir kopyasını da yazar.

Sonra sen o nesnenin bir özelliğini değiştirirsin. `urun.Fiyat = 200;` yazarsın. Bu satır veritabanına gitmez. Hiçbir şey olmaz. Ama asistanın elindeki kopyayla ortadaki nesne artık birbirini tutmuyordur. Asistan bunu görür ve aklında tutar: "bu kaydın fiyatı değişmiş."

Asıl olay `SaveChanges()` dediğinde başlar. Asistan o ana kadar biriktirdiği tüm farkları önüne koyar, her biri için gereken SQL cümlesini yazar ve hepsini **tek seferde** veritabanına gönderir. Bir tanesi bile hata verirse hiçbiri uygulanmaz. Yani ya hepsi olur ya hiçbiri.

Bu mekanizmayı anladığın anda üç şey birden yerine oturur. Birincisi: `Add`, `Update`, `Remove` metotlarının hiçbiri veritabanına yazmaz — sadece asistana not aldırır. İkincisi: veritabanından çektiğin bir nesneyi değiştirdiğinde `Update()` çağırmana gerek yoktur, çünkü asistan zaten o nesneyi tanıyor. Üçüncüsü: formdan gelen bir nesneyi asistan **tanımaz** — onu sen tanıtmak zorundasın, ve tanıtma biçimin veri kaybına yol açabilir.

Bu notun zor kısmı, nesnenin "hangi durumda" olduğunu takip edebilmektir. Çünkü aynı `SaveChanges()` satırı, nesnenin durumuna göre `INSERT` de yazar, `UPDATE` de, `DELETE` de, hiçbir şey de yazmaz.

> **Ana benzetme:** EF Core, elinde kurşun kalemle sipariş defteri tutan bir garsondur. Masada söylediğin her şeyi deftere yazar, siler, değiştirir — mutfağa hiçbir şey gitmez. Ancak "tamam, geçebilirsin" dediğinde (`SaveChanges`) defterdeki her satır mutfağa tek fiş hâlinde iner. O ana kadar yaptığın her değişiklik yalnızca kâğıt üstündedir.

Şimdi bu defterin nasıl tutulduğuna, satır satır bakıyoruz.

---

## Bu Notta Ne Var

1. Change Tracking — EF Core'un kalbi
2. Create (Ekleme)
3. Read (Okuma)
4. Update (Güncelleme) — iki aşamalı akış
5. `Update()` metodu arka planda ne yapar
6. Alternatif: yalnızca değişen alanları güncelleme
7. Delete — Hard delete ve Soft delete
8. `Update()` ne zaman gerekli, ne zaman gereksiz

---

## 1. Change Tracking — EF Core'un Kalbi

> **Benzetme —** Markette alışveriş sepetiyle dolaşıyorsun. Rafdan aldığın her ürün sepete girer, vazgeçtiğini geri koyarsın, elindeki paketi değiştirip başkasını alırsın. Bütün bu süre boyunca **hiçbir şey satın alınmamıştır**. Mağazanın stok sistemi senin sepetinden habersizdir. Alışveriş, ancak kasadan geçtiğinde gerçekleşir — ve kasada ya hepsi ödenir ya da kart geçmez, hiçbiri ödenmez.

**Basitçe:** EF Core, veritabanından çektiğin nesneleri bir kenarda takip eder. Onları değiştirdiğinde fark eder ama hemen bir şey yapmaz. Yaptığın her ekleme, değiştirme ve silme bir **not**tur. Notlar biriktirilir. `SaveChanges()` dediğinde hepsi birden veritabanına gider. O satırı yazmadığın sürece veritabanında hiçbir şey değişmez.

**Teknik olarak: Change Tracking (değişiklik izleme)** — EF Core'un, kendisinden çekilen nesneleri hafızada takip edip üzerlerinde yapılan değişiklikleri fark etmesi.

Bir nesneyi veritabanından çektiğinde EF Core onu bir listeye yazar ve orijinal değerlerinin bir kopyasını saklar. Sonra sen nesnenin bir özelliğini değiştirdiğinde, EF Core bunu **`SaveChanges` çağrıldığında** karşılaştırarak anlar.

Buradaki "orijinal değerlerin kopyası" ifadesi, mekanizmanın can damarıdır. EF Core senin ne değiştirdiğini sihirle bilmez; elinde iki sürüm vardır — çektiği andaki hâl (original values) ve şu andaki hâl (current values). Farkı bulmak için ikisini karşılaştırır. Bu karşılaştırmanın adı **snapshot change tracking**'tir.

Her takip edilen nesnenin bir **durumu (state)** vardır:

| Durum | Anlamı | `SaveChanges` ne üretir |
|---|---|---|
| **Unchanged** | Çekildi, değişmedi | Hiçbir şey |
| **Added** | Yeni eklendi | `INSERT` |
| **Modified** | Değişti | `UPDATE` |
| **Deleted** | Silinmek üzere işaretlendi | `DELETE` |
| **Detached** | EF Core bu nesneyi tanımıyor | Hiçbir şey |

### 1.1 Beş durumu üç benzetmeyle oturtmak

Bu tablo ezberlenmez, oturtulur. Üç ayrı benzetmeyle üç farklı yüzünü görelim; her birinin bir kör noktası var ve o kör noktaları da yazıyorum.

**Birinci benzetme — alışveriş sepeti.**

| Durum | Sepette karşılığı |
|---|---|
| `Unchanged` | Rafdan aldın, sepete koydun, hiç dokunmadın |
| `Added` | Sepete **yeni** bir ürün attın, kasa henüz görmedi |
| `Modified` | Sepetteki paketi açıp içindekini değiştirdin |
| `Deleted` | Sepetten çıkarıp "bunu almıyorum" rafına koydun |
| `Detached` | Ürün hâlâ rafta; sepetine hiç girmedi, kasa varlığından habersiz |

> **Bu benzetme şurada bozulur:** Markette sepete koyduğun ürün rafdan fiziksel olarak eksilir. EF Core'da ise nesneyi "sepete koymak" veritabanından bir şey eksiltmez; veritabanı senin sepetinden tamamen habersizdir ve aynı kaydı başka biri aynı anda değiştirebilir. Sepet benzetmesi eşzamanlılığı (concurrency) anlatamaz.

**İkinci benzetme — otel resepsiyonundaki kayıt defteri.**

Resepsiyonist gün boyunca deftere işler: yeni gelen misafiri yazar, oda değişikliğini kaydeder, çıkanı işaretler. Merkez sisteme aktarım akşam yapılır.

| Durum | Resepsiyonda karşılığı |
|---|---|
| `Unchanged` | Misafir kayıtlı, bugün hiçbir değişiklik olmadı |
| `Added` | Yeni check-in yapıldı, deftere yazıldı, merkeze henüz bildirilmedi |
| `Modified` | Misafirin odası veya telefonu değişti, defterde üzeri düzeltildi |
| `Deleted` | Check-out yapıldı, kayıt kapatılmak üzere işaretlendi |
| `Detached` | Lobiden geçen bir yabancı — defterde hiç yok, resepsiyonist onu tanımıyor |

Bu benzetmenin güçlü yanı `Detached`'ı net göstermesidir. **Detached, "silinmiş" demek değildir. "Bu nesne benim defterimde hiç yok" demektir.** Formdan POST ile gelen nesne tam olarak budur: lobiden içeri girmiş, kimliğini söylemiş ama defterde kaydı olmayan biri.

> **Bu benzetme şurada bozulur:** Resepsiyonist misafiri yüzünden tanır. EF Core nesneyi **referansından** tanır, `Id`'sinden değil. Aynı `Id`'ye sahip ama başka bir `new Product()` nesnesi, EF Core için tamamen yabancıdır. Yani "Id'si aynı, o hâlde tanır" çıkarımı yanlıştır — bu, notun ilerleyen bölümlerindeki `Update()` tartışmasının tam kalbidir.

**Üçüncü benzetme — kurşun kalemle tutulan defter.**

Deftere kalemle yazarsın, silgiyle silersin, üzerini çizersin. Sayfada izler kalır: neyin yeni yazıldığı, neyin üzerinin çizildiği, neyin düzeltildiği bellidir. Sayfayı temize çekip teslim ettiğinde (`SaveChanges`) defter tertemiz olur; tüm satırlar artık "yazılmış" sayılır.

Bu benzetmenin anlattığı şey `SaveChanges` sonrası olandır: **kaydettikten sonra tüm `Added` ve `Modified` nesneler `Unchanged` hâline döner.** Defter temize çekilmiştir, iz kalmaz. Aynı nesneyi tekrar `SaveChanges` etmek ikinci bir `UPDATE` üretmez, çünkü ortada fark kalmamıştır.

> **Bu benzetme şurada bozulur:** Kâğıtta silinen yazının izi gözle görünür. EF Core'da ise "orijinal değer" kopyası bellekte durur ve sen ona bakmadıkça görünmez. Görmek istersen açıkça sormak zorundasın — bunun nasıl yapıldığı 1.3'te.

### 1.2 Durumlar birbirine nasıl dönüşür

**Kritik nokta:** `Add`, `Update`, `Remove` metotları veritabanına **hiçbir şey yazmaz**. Yalnızca durumu işaretlerler. Tek yazma anı `SaveChanges()`'tir ve o da tüm işaretli değişiklikleri **tek bir transaction** içinde gönderir.

```
_context.Products.Add(yeni);        → durum: Added      (veritabanı: değişiklik yok)
urun.Fiyat = 200;                   → durum: Modified   (veritabanı: değişiklik yok)
_context.Products.Remove(eski);     → durum: Deleted    (veritabanı: değişiklik yok)

await _context.SaveChangesAsync();  → INSERT + UPDATE + DELETE, hepsi tek transaction
```

Geçişleri bir arada görmek işi kolaylaştırır:

```
                 Add()                         SaveChanges()
   Detached ───────────────► Added ──────────────────────────► Unchanged
       │                                                            │
       │ Attach() / sorgudan gelme                   özellik değişti │
       │                                                            ▼
       └──────────────────► Unchanged ◄──────────────────────── Modified
                                │                                   ▲
                                │ Remove()                          │
                                ▼                     Update() ──────┘
                             Deleted ──── SaveChanges() ───► Detached
```

Buradan okunacak iki ince nokta var:

1. **`SaveChanges` sonrası `Deleted` nesne `Detached` olur.** Silinen kaydın karşılığı artık veritabanında yoktur, dolayısıyla takip edilecek bir şey de kalmaz.
2. **`Update()` bir nesneyi doğrudan `Modified` yapar** — `Detached` olsun, `Unchanged` olsun fark etmez. Bu, 5. bölümdeki veri kaybı riskinin çıkış noktasıdır.

Bir hata çıkarsa hiçbiri uygulanmaz (rollback). Bu, "ya hep ya hiç" garantisidir.

### 1.3 Durumu kendi gözünle görmek

Change tracking soyut kaldığı sürece kafa karıştırır. En hızlı çözüm, EF Core'a durumları doğrudan sormaktır:

```csharp
var urun = await _context.Products.FindAsync(5);

Console.WriteLine(_context.Entry(urun).State);     // Unchanged

urun.Price = 1600;

Console.WriteLine(_context.Entry(urun).State);     // Modified

// Hangi alan değişti, eski değeri neydi?
foreach (var prop in _context.Entry(urun).Properties)
{
    if (prop.IsModified)
        Console.WriteLine($"{prop.Metadata.Name}: {prop.OriginalValue} -> {prop.CurrentValue}");
}
// Çıktı: Price: 1500,00 -> 1600,00
```

Tüm takip listesini bir kerede dökmek de mümkündür:

```csharp
foreach (var entry in _context.ChangeTracker.Entries())
{
    Console.WriteLine($"{entry.Entity.GetType().Name} -> {entry.State}");
}
```

Bu iki kod parçası, bu notun geri kalanındaki her iddiayı kendi gözünle doğrulamanı sağlar. "`Update()` gerçekten tüm alanları mı işaretliyor?" sorusunun cevabı tahmin değil, çıktıdır.

---

## 2. Create (Ekleme)

> **Benzetme —** Nüfus müdürlüğünde yeni kayıt açtırıyorsun. Önce boş formu alırsın (GET), doldurup memura verirsin (POST). Memur formu inceler; bir alan eksikse formu geri uzatır, tamsa kaydı açar ve seni sıradan çıkarır. Aynı formu ikinci kez vermeye kalkışmaman için de elinden alır — kapıdan çıkarken elinde doldurulmuş form kalmaz.

**Basitçe:** Ekleme iki adımdır. Birinci istekte kullanıcıya boş form gösterilir. İkinci istekte gelen veri kontrol edilir; hatalıysa form geri gösterilir, doğruysa kayıt yapılır ve kullanıcı başka bir sayfaya yönlendirilir. Yönlendirme, kullanıcının F5'e basıp aynı kaydı ikinci kez oluşturmasını engellemek içindir.

**Teknik olarak:** İki adımlı klasik kalıp: GET formu gösterir, POST kaydeder.

```csharp
[HttpGet]
public IActionResult Create()
{
    return View();
}

[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create(Product product)
{
    if (ModelState.IsValid)                       // doğrulama kuralları sağlandı mı?
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
        return RedirectToAction(nameof(Index));   // kayıttan sonra listeye dön
    }
    return View(product);                         // hata varsa aynı sayfaya verilerle dön
}
```

**`ModelState.IsValid`** — Model binding sırasında oluşan hataların (tip uyumsuzluğu) ve model üzerindeki doğrulama özniteliklerinin (`[Required]`, `[StringLength]`) toplu sonucu. Tek satırda tüm alanları kontrol eder.

**`nameof(Index)`** — Metot adını string olarak verir. `"Index"` yazmak yerine bunu kullanmanın sebebi: metodu yeniden adlandırdığında derleyici hatayı yakalar, string sessizce bozulmaz.

**`RedirectToAction`** — POST-Redirect-GET deseni. Kullanıcı F5'e bastığında aynı kayıt ikinci kez eklenmez.

**`[ValidateAntiForgeryToken]`** — Formu gerçekten senin sayfandan mı geldi diye kontrol eder. Başka bir sitenin, kullanıcının oturumunu kullanarak senin POST metoduna istek göndermesini (CSRF) engeller. Tag helper'la yazılmış her `<form>` bu token'ı otomatik ekler; öznitelik de karşı tarafta onu doğrular.

### Ekleme sırasında `Id` ne zaman dolar

Yeni nesneyi oluşturduğunda `Id` değeri `0`'dır — veritabanı henüz bir numara vermemiştir. Bu numara `SaveChanges` sonrasında nesneye **geri yazılır**:

```csharp
var urun = new Product { Name = "Klavye", Price = 750 };

Console.WriteLine(urun.Id);            // 0  — henüz kaydedilmedi

_context.Products.Add(urun);
await _context.SaveChangesAsync();

Console.WriteLine(urun.Id);            // 42 — veritabanının verdiği numara
```

Bu davranış, "kaydettikten sonra detay sayfasına yönlendir" gibi senaryoların temelidir:

```csharp
return RedirectToAction(nameof(Details), new { id = urun.Id });
```

> `Add` ile `AddAsync` arasındaki fark küçüktür ve çoğu zaman önemsizdir: `AddAsync` yalnızca bazı değer üreteçlerinin (örneğin `HiLo`) veritabanına gitmesi gerektiğinde anlam kazanır. Normal `Identity` sütunlarında `Add` yeterlidir; ikisi de veritabanına yazmaz.

---

## 3. Read (Okuma)

> **Benzetme —** Kütüphaneden kaynak alıyorsun. İki ayrı ihtiyaç var: bazen kitabı **ödünç** alırsın — adın deftere yazılır, kitap sende kayıtlıdır, iade etmen beklenir. Bazen de sadece **okuma salonunda** bakarsın; kimse adını yazmaz, kalktığında iz kalmaz. İkincisi kütüphaneye daha az iş çıkarır, ama o kitabın üstünde bir işlem yaptıramazsın.

**Basitçe:** Veri okurken iki soru sorarsın. Birincisi "ne kadarını çekiyorum?" — gereğinden fazla satır ve kolon çekmek boşuna yüktür. İkincisi "bu veriyi sonra değiştirecek miyim?" — sadece ekranda göstereceksen EF Core'un bunu takip etmesine gerek yoktur, takip etmemesini söylersen daha hızlı çalışır.

**Teknik olarak:**

```csharp
// Liste
public async Task<IActionResult> Index()
{
    var products = await _context.Products
                                 .Where(p => !p.IsDeleted)
                                 .ToListAsync();
    return View(products);
}

// Tek kayıt
public async Task<IActionResult> Details(int id)
{
    var product = await _context.Products.FirstOrDefaultAsync(p => p.Id == id);

    if (product == null) return NotFound();

    return View(product);
}
```

**Performans ilkesi:** Yalnızca ihtiyaç duyulan veri çekilir. Bunun iki boyutu var:

- **Satır sayısı** — `Where` ile filtrele, `Skip/Take` ile sayfala
- **Kolon sayısı** — `Select` ile yalnızca gereken alanları projekte et

```csharp
// 40 kolonlu tablodan sadece 3 kolon
var ozet = await _context.Products
    .Select(p => new { p.Id, p.Name, p.Price })
    .ToListAsync();
```

### 3.1 `AsNoTracking()` — basit dille

> **Benzetme —** Garson masaya menüyü getirdiğinde siparişini deftere yazmaz; sadece göstermiştir. Yazmak, ancak bir şey sipariş edeceksen gerekir. Boşuna yazılan her satır, akşam defteri okurken kaybedilen zamandır.

**Basitçe:** EF Core, çektiği her nesnenin bir kopyasını kenara alır ki sonra "değişmiş mi" diye bakabilsin. Sen o nesneyi değiştirmeyeceksen bu kopya tamamen boşunadır. `AsNoTracking()` yazdığında "bunları takip etme, ben sadece ekranda göstereceğim" demiş olursun. Daha az bellek, daha hızlı sorgu.

**Teknik olarak:** **Salt okuma senaryolarında `AsNoTracking()`** kullanmak, EF Core'un takip listesi tutma maliyetini ortadan kaldırır:

```csharp
var liste = await _context.Products.AsNoTracking().ToListAsync();
```

Veriyi yalnızca ekranda göstereceksen bu ücretsiz bir kazançtır. Ama o nesneyi sonradan güncelleyeceksen kullanma — takip edilmediği için değişiklikler fark edilmez.

Sessiz tuzağı şudur:

```csharp
var urun = await _context.Products.AsNoTracking().FirstAsync(p => p.Id == 5);

urun.Price = 1600;
await _context.SaveChangesAsync();     // hiçbir şey olmaz, hata da vermez
```

Kod hata vermez. Sorgu çalışır, `SaveChanges` çalışır, geriye `0` döner ve fiyat değişmez. Çünkü bu nesne `Detached`'tır; takip listesinde olmayan bir nesnenin değişikliği fark edilemez. Hata mesajı olmadığı için bu tür bir bug'ı bulmak uzun sürer — bu yüzden `AsNoTracking()`'i **yalnızca** sonu `View`'a giden salt okuma sorgularına koy.

Kazancın ne kadar olduğu listenin boyutuna bağlıdır: 10 kayıtta fark ölçülemez, 10.000 kayıtta gözle görülür. Kural olarak, `foreach` ile ekrana basılan her liste sorgusu `AsNoTracking()` adayıdır.

> `FindAsync(id)` ile `FirstOrDefaultAsync(p => p.Id == id)` farkı: `FindAsync` önce **hafızadaki takip listesine** bakar, orada bulursa veritabanına hiç gitmez. Birincil anahtarla aramada bu yüzden daha verimlidir.

Bu farkın pratik sonucu: aynı istek içinde bir kaydı iki kez `FindAsync` ile çekersen ikinci çağrı sorgu üretmez ve **aynı nesne referansını** döndürür. `FirstOrDefaultAsync` ise her seferinde veritabanına gider. Öte yandan `FindAsync` yalnızca birincil anahtarla çalışır; başka bir kolona göre arayacaksan seçeneğin yoktur.

---

## 4. Update (Güncelleme) — İki Aşamalı Akış

> **Benzetme —** Bankada hesap bilgini güncelleteceksin. Gişeye gidersin, memur kaydını ekrana getirir ve sana çıktısını verir (GET). Sen kâğıdı alıp masaya oturursun, düzeltirsin, sonra gişeye geri gelirsin (POST). Ama bu sefer **başka bir memur** vardır ve seni tanımaz. Elindeki kâğıtta müşteri numarası yazmıyorsa, hangi hesabı güncelleyeceğini bilemez.

**Basitçe:** Güncelleme tek bir iş gibi görünür ama aslında iki ayrı ziyarettir. İlkinde veriyi çekip kullanıcıya gösterirsin. İkincisinde kullanıcı düzenlenmiş hâlini geri gönderir. Arada sunucu her şeyi unutur — hangi kaydı gösterdiğini bile. Bu yüzden kaydın kimliği, formun içinde gizli bir alanda geri taşınmak zorundadır.

**Teknik olarak:** Güncelleme, MVC'de en çok yanlış anlaşılan işlemdir. Sebebi: **iki bağımsız HTTP isteği** ve iki ayrı veritabanı işlemi içermesi.

```
1) GET  /Product/Edit/5   → kaydı çek, formu doldur, kullanıcıya gönder
                             (HTTP stateless: yanıt gidince sunucu bu nesneyi unutur)
2) POST /Product/Edit     → formdan gelen veriyi al, veritabanına yaz
```

> **Bu benzetme şurada bozulur:** Bankada ikinci memur, ilk memurun aynı şirkette çalıştığını bilir ve gerekirse ona sorabilir. Sunucuda böyle bir imkân yoktur: POST isteğini karşılayan `DbContext`, GET'teki `DbContext`'in varlığından bile habersizdir. İkisi arasında paylaşılan tek şey, kullanıcının tarayıcısında taşınan veridir. "Sunucu hatırlar" varsayımı, bu konudaki hataların büyük kısmının kaynağıdır.

### Aşama 1 — Veriyi bulma (GET)

```csharp
[HttpGet]
public async Task<IActionResult> Edit(int id)
{
    // EF Core: SELECT * FROM Products WHERE Id = 5
    var product = await _context.Products.FindAsync(id);

    if (product == null) return NotFound();

    return View(product);
}
```

Bu aşamada çekilen nesne EF Core tarafından takip edilmeye başlar. **Ama HTTP doğası gereği**, yanıt kullanıcıya gönderildiği an istek biter, `DbContext` bırakılır ve takip sona erer. Bir sonraki POST isteği **yepyeni bir context** ile gelir.

Burayı bir kez daha vurgulamakta fayda var: takip bir dosyada ya da veritabanında tutulmaz. `DbContext` nesnesinin **içinde** tutulur. `DbContext` ise varsayılan olarak istek başına oluşturulur ve istek bitince atılır (scoped yaşam süresi). Yani takip listesi, isteğin ömrü kadar yaşar.

### Aşama 2 — Formda Id'yi taşımak

```cshtml
@model Product

<form asp-action="Edit" method="post">
    @* ID BİLGİSİ GİZLİ OLARAK TUTULMALIDIR *@
    <input type="hidden" asp-for="Id" />

    <div>
        <label>Ürün Adı:</label>
        <input asp-for="Name" />
    </div>

    <div>
        <label>Fiyat:</label>
        <input asp-for="Price" />
    </div>

    <button type="submit">Kaydet</button>
</form>
```

**Gizli `Id` alanı olmazsa POST'ta hangi kaydın güncelleneceği bilinemez.** Bu, güncelleme formlarının en sık atlanan parçasıdır; unutulduğunda `Id = 0` gelir ve ya hata alınır ya da yanlış kayıt güncellenir.

> `type="hidden"` yalnızca ekranda görünmemesini sağlar; kullanıcı tarayıcı araçlarından değerini değiştirebilir. Yetki gerektiren işlemlerde bu id'ye körü körüne güvenilmez — sunucuda "bu kayıt bu kullanıcıya ait mi" kontrolü yapılır.

Pratikte bu kontrol şuna benzer:

```csharp
var urun = await _context.Products.FindAsync(formProduct.Id);
if (urun == null) return NotFound();
if (urun.OwnerId != KullaniciId()) return Forbid();      // başkasının kaydı
```

### Aşama 3 — Kaydetme (POST)

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(Product product)
{
    if (!ModelState.IsValid)
        return View(product);

    _context.Products.Update(product);
    await _context.SaveChangesAsync();

    return RedirectToAction(nameof(Index));
}
```

Bu kod çalışır. Ama bir sonraki bölümün konusu, **neden her zaman istediğin şeyi yapmadığıdır.**

---

## 5. `Update()` Arka Planda Ne Yapar

> **Benzetme —** Kargoya verdiğin iade formunu düşün. Memur, senin doldurduğun kâğıdı alıp sistemdeki kaydın **tamamının** yerine geçirir. Sen sadece "telefon numaram değişti" demek istemiştin; ama formda adres satırı boştu ve sistemdeki adres de boşaldı. Memur kötü niyetli değil — ona "bu kâğıt artık kaydın yeni hâli" denmiş.

**Basitçe:** `Update()` metodu EF Core'a "bu nesnenin **her alanı** değişmiş olabilir, hepsini yaz" der. Formdan gelen nesnede olmayan alanlar boş (`0`, `null`) olduğu için, veritabanındaki gerçek değerlerinin üzerine bu boşluklar yazılır. Görünürde hata yoktur; veri sessizce kaybolur.

**Teknik olarak:** Bu POST metodunda elimize gelen `product` nesnesi, veritabanından çektiğimiz nesne **değildir**. Model binding'in form verilerinden yarattığı **yepyeni** bir nesnedir. EF Core bu nesneyi daha önce hiç görmemiştir — durumu `Detached`'tır.

`_context.Products.Update(product);` satırı çalıştığında üç şey olur:

1. **EF Core takibi başlatır** — nesneyi hafızasına alır.
2. **Durumu `Modified` olarak işaretler** — "bu nesnenin veritabanında bir karşılığı var (Id üzerinden), ve tüm özellikleri değişmiş olabilir" demektir.
3. **`SaveChangesAsync()` sorguyu üretir** — `Modified` işaretli nesne için, **tüm sütunları** güncelleyen bir `UPDATE` yazar.

Burada işin püf noktası şudur: EF Core bu nesneyi hiç görmediği için elinde **orijinal değer kopyası yoktur**. Karşılaştıracak bir şey olmadığında "hangi alan değişti" sorusunun cevabı bulunamaz. Cevabı bulamayınca da en güvenli görünen varsayımı yapar: *hepsi değişmiş olabilir.* Veri kaybı, kötü bir algoritmadan değil, **eksik bilgiden** doğar.

```sql
-- EF Core'un ürettiği varsayılan sorgu (tüm sütunları günceller)
UPDATE [Products]
SET [Name] = 'Yeni Ad', [Price] = 1600.00, [Stock] = 50, [IsDeleted] = 0
WHERE [Id] = 5;
```

**Buradaki risk:** `Stock` alanı formda yoktu. Model binding onu dolduramadı, dolayısıyla varsayılan değerinde (`0`) kaldı. `Update()` tüm sütunları yazdığı için **veritabanındaki gerçek stok değeri sıfırlandı.**

Bu, sessiz veri kaybının klasik biçimidir. Tablo küçükken fark edilmez, kolon eklendikçe tehlikeli hâle gelir.

### 5.1 Tek alan güncellemenin üç yolu — yan yana

"Sadece fiyatı değiştirmek istiyorum" cümlesinin üç ayrı karşılığı vardır ve ürettikleri SQL bambaşkadır.

```csharp
// YOL 1 — Update(): TÜM sütunlar yazılır. Formda olmayan alanlar ezilir.
_context.Products.Update(formProduct);
await _context.SaveChangesAsync();
// UPDATE Products SET Name=..., Price=..., Stock=0, IsDeleted=0 WHERE Id=5

// YOL 2 — Çek ve ata: yalnızca gerçekten değişen alan yazılır. Önerilen.
var db = await _context.Products.FindAsync(formProduct.Id);
db.Price = formProduct.Price;
await _context.SaveChangesAsync();
// UPDATE Products SET Price=... WHERE Id=5

// YOL 3 — Attach + tek alanı işaretle: tek sorgu, tek sütun. İleri seviye.
var stub = new Product { Id = formProduct.Id, Price = formProduct.Price };
_context.Attach(stub);
_context.Entry(stub).Property(p => p.Price).IsModified = true;
await _context.SaveChangesAsync();
// UPDATE Products SET Price=... WHERE Id=5
```

Üçüncü yol, ikinci yolun `SELECT` maliyetini de ortadan kaldırır: `Attach` nesneyi `Unchanged` olarak takibe alır, sonra sen yalnızca bir özelliği `IsModified = true` yaparsın. EF Core da yalnızca onu yazar. Bedeli, elle yönetilen bir yapı olması ve alan eklendikçe unutulmaya açık olmasıdır. Günlük işte 2. yol, sıcak yollarda (hot path) 3. yol tercih edilir.

> **Bu benzetme şurada bozulur:** Kargo memuru senin boş bıraktığın satırı görüp "burası boş kalmış, eskisini koruyayım" diyebilir. EF Core bunu **yapamaz**, çünkü onun için `0` ile "kullanıcı boş bıraktı" arasında hiçbir fark yoktur. `int Stock` alanının `0` değeri, tamamen geçerli bir stok miktarıdır. Niyeti okumak diye bir şey yoktur; yalnızca değer vardır.

### 5.2 Over-posting'in buradaki yüzü

Routing notunda gördüğün **over-posting** sorunu, güncellemede daha tehlikeli hâle gelir. Orada saldırgan olmayan bir alanı **ekliyordu**; burada olmayan bir alan **siliniyor**.

**Basitçe:** Entity'nin tamamını parametre olarak almak iki yönlü açık yaratır. Kullanıcı göndermediği alanları sıfırlatabilir (veri kaybı), göndermemesi gereken alanları gönderebilir (yetki yükseltme). İkisinin de tek bir çözümü var: **formun taşıdığı alanlar için ayrı bir sınıf yaz.**

```csharp
public class ProductEditModel
{
    public int Id { get; set; }
    [Required, StringLength(100)] public string Name  { get; set; } = "";
    [Range(0, 1_000_000)]         public decimal Price { get; set; }
    // Stock yok, IsDeleted yok — ne ezilebilir ne enjekte edilebilir
}

[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(ProductEditModel model)
{
    if (!ModelState.IsValid) return View(model);

    var db = await _context.Products.FindAsync(model.Id);
    if (db == null) return NotFound();

    db.Name  = model.Name;
    db.Price = model.Price;
    // db.Stock ve db.IsDeleted'a dokunulmadı — orijinal değerleri korunur

    await _context.SaveChangesAsync();
    return RedirectToAction(nameof(Index));
}
```

Bu kalıpta `Update()` **hiç geçmez**, çünkü `db` nesnesi takip edilmektedir. Ve `Stock` alanı hem modelde hem kodda yoktur; unutmakla kaybedilecek bir şey kalmamıştır.

---

## 6. Alternatif: Yalnızca Değişen Alanları Güncelleme

> **Benzetme —** Tarif defterindeki bir yemeğin tuz miktarını değiştireceksin. İki yolun var: sayfayı baştan temize çekmek (bu sırada hatırlamadığın satırlar eksik kalır) ya da mevcut sayfayı açıp yalnızca tuz satırını düzeltmek. İkincisi bir hamle fazladır — defteri açman gerekir — ama geri kalan tarifi bozmaz.

**Basitçe:** Önce kaydı veritabanından çek. Böylece elinde **gerçek ve tam** hâli olur. Sonra yalnızca değişmesini istediğin alanları üzerine yaz. Geri kalan alanlara dokunmadığın için onlar olduğu gibi kalır. Bu nesne zaten takip edildiğinden `Update()` yazmana da gerek kalmaz.

**Teknik olarak:** Riski ortadan kaldıran yol: veriyi **önce veritabanından çek**, sonra yalnızca formdan gelen alanları ata.

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(Product formProduct)
{
    // 1. Orijinal veriyi veritabanından çek (bu nesne TAKİP EDİLİYOR)
    var dbProduct = await _context.Products.FindAsync(formProduct.Id);

    if (dbProduct == null) return NotFound();

    // 2. Sadece değişmesini istediğimiz alanları ez
    dbProduct.Name  = formProduct.Name;
    dbProduct.Price = formProduct.Price;
    // dbProduct.Stock alanına hiç dokunmadık — orijinal değeri korunur

    // 3. Update() YAZMAYA GEREK YOK
    //    EF Core dbProduct'ı zaten takip ettiği için değişikliği kendisi algılar
    await _context.SaveChangesAsync();

    return RedirectToAction(nameof(Index));
}
```

Bu yöntemde EF Core yalnızca gerçekten değişen alanlar için `UPDATE` yazar:

```sql
UPDATE [Products] SET [Name] = 'Yeni Ad', [Price] = 1600.00 WHERE [Id] = 5;
```

Daha ilginci: hiçbir alan gerçekten değişmemişse EF Core **hiç sorgu göndermez**. `SaveChanges` geriye `0` döner. Çünkü orijinal kopyayla karşılaştırma yapılmış ve fark bulunamamıştır.

### İki yöntemin karşılaştırması

| | `Update(formNesnesi)` | Çek → alanları ata |
|---|---|---|
| Veritabanı gidiş sayısı | 1 (sadece UPDATE) | 2 (SELECT + UPDATE) |
| Formda olmayan alanlar | **Ezilir** (veri kaybı riski) | Korunur |
| Üretilen SQL | Tüm sütunlar | Yalnızca değişenler |
| `Update()` çağrısı | Gerekli | **Gereksiz** |
| Değişiklik yoksa | Yine de `UPDATE` atar | Hiç sorgu atmaz |
| Güvenlik | Düşük | **Yüksek** |

**Sonuç:** İkinci yöntem bir ekstra sorgu maliyetine karşılık veri bütünlüğünü korur ve genellikle tercih edilir. Formun entity'nin **tüm** alanlarını içerdiğinden emin olduğun küçük tablolarda birinci yöntem de doğrudur — ama bunu bilerek seçmek gerekir.

> **Bu benzetme şurada bozulur:** Tarif defterinde sayfayı açtığın an başkası aynı sayfaya yazamaz. Veritabanında ise `SELECT` ile `UPDATE` arasındaki kısa sürede başka biri aynı kaydı değiştirebilir; senin yazdığın değer onunkini sessizce ezer. Bu soruna **concurrency (eşzamanlılık)** denir ve çözümü ayrı bir konudur (`RowVersion` / `[ConcurrencyCheck]`). Çek-ve-ata yöntemi veri kaybı riskini azaltır, sıfırlamaz.

---

## 7. Delete — Hard Delete ve Soft Delete

> **Benzetme —** Kütüphanede bir kitabı kaybettin. İki seçenek var: kart kataloğundan fişini tamamen yırtıp atmak ya da fişin üstüne "kayıp" damgası vurup çekmecede bırakmak. Fişi atarsan kitabın hiç var olmadığını sanırsın; ama o kitaba atıf yapan on tane kaynak listesi elinde kalır ve hiçbirini çözemezsin. Damga vurursan kitap raflarda görünmez ama geçmiş kayıtlar anlamlı kalır.

**Basitçe:** Bir kaydı iki türlü "silebilirsin". Gerçekten silmek, satırı tablodan yok etmektir — geri dönüşü yoktur ve o kayda bağlı eski kayıtlar sahipsiz kalır. Diğer yol, kaydı silmek yerine "bu artık pasif" diye işaretlemektir; veri durur, sadece listelerde görünmez. Kurumsal projelerde neredeyse her zaman ikincisi tercih edilir.

**Teknik olarak:**

### Hard Delete (fiziksel silme)

```csharp
_context.Products.Remove(product);
await _context.SaveChangesAsync();
```

Kayıt tablodan gerçekten silinir. İlişkisel veritabanlarında bu çoğu zaman **veri bütünlüğünü bozar**: silinen ürüne bağlı eski fatura satırları, sipariş geçmişi veya raporlar yetim kalır. Yabancı anahtar kısıtı varsa silme işlemi zaten hata verir.

Silinecek kaydı önce çekmek zorunda değilsin; elinde yalnızca `Id` varsa bir "stub" nesne yeterlidir:

```csharp
var stub = new Product { Id = id };
_context.Products.Remove(stub);        // Attach + Deleted işaretleme
await _context.SaveChangesAsync();     // DELETE FROM Products WHERE Id = 5
```

Bu, gereksiz bir `SELECT`'ten kurtarır. Karşılığında, kayıt yoksa `DbUpdateConcurrencyException` alırsın — "silmeye çalıştığım satır orada değil" demektir.

### Soft Delete (mantıksal silme)

Kurumsal uygulamalarda tercih edilen yol: kaydı silmek yerine **pasife çekmek**.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Stock { get; set; }
    public decimal Price { get; set; }
    public bool IsDeleted { get; set; }    // soft delete bayrağı
}
```

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Delete(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null) return NotFound();

    // 1. YÖNTEM: Hard Delete (fiziksel silme)
    // _context.Products.Remove(product);

    // 2. YÖNTEM: Soft Delete (durum güncelleme)
    product.IsDeleted = true;

    await _context.SaveChangesAsync();
    return RedirectToAction(nameof(Index));
}
```

Change tracking açısından bu iki satırın farkı dikkat çekicidir. `Remove(product)` nesneyi `Deleted` yapar ve `DELETE` üretir. `product.IsDeleted = true;` ise nesneyi `Modified` yapar ve tek sütunluk bir `UPDATE` üretir. Yani "silme" işleminin arkasında aslında bir güncelleme vardır; kavram silme, işlem güncellemedir.

Listeleme sorgularının bunu dikkate alması gerekir:

```csharp
var products = await _context.Products
                             .Where(p => !p.IsDeleted)
                             .ToListAsync();
```

**Soft delete'in bedeli:** Her sorguya `Where(p => !p.IsDeleted)` eklemeyi unutmak, silinmiş kayıtların ekranda görünmesine yol açar. EF Core'un **global query filter** özelliği bunu tek yerden çözer:

```csharp
// AppDbContext.OnModelCreating içinde
modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
```

Bu satırdan sonra tüm sorgulara filtre otomatik eklenir. Silinmişleri kasten görmek gerektiğinde `IgnoreQueryFilters()` kullanılır.

```csharp
// Yönetim panelinde silinmişleri de göster
var hepsi = await _context.Products.IgnoreQueryFilters().ToListAsync();
```

> **Ek fayda:** Soft delete, "yanlışlıkla sildim" durumunu geri alınabilir hâle getirir ve denetim (audit) gereksinimlerini karşılar. Ek olarak `DeletedAt` ve `DeletedBy` alanları tutulursa kim ne zaman sildi sorusu da cevaplanır.

> **Bu benzetme şurada bozulur:** Kütüphanedeki "kayıp" damgası kitabı raftan kaldırmaz; kitap oradadır, sadece fişinde not vardır. Veritabanında ise `IsDeleted = true` satırın kendisine hiçbir şey yapmaz — satır tüm sorgularda hâlâ görünür durumdadır. Görünmemesini sağlayan şey damga değil, **senin her sorguya eklediğin filtredir**. Filtre yoksa soft delete diye bir şey de yoktur; sadece kullanılmayan bir kolon vardır. Global query filter'ın değeri tam olarak budur: filtreyi hatırlama işini senden alır.

---

## 8. `Update()` Ne Zaman Gerekli

> **Benzetme —** Mahalle bakkalı seni tanıyorsa "deftere yaz" demen yeter, adını söylemene gerek yoktur. Tanımıyorsa önce kendini tanıtman gerekir: "ben 3 numaralı daireden geliyorum". EF Core'da `Update()` tam olarak bu tanıtmadır — tanınan nesnede gereksiz, tanınmayan nesnede şart.

**Basitçe:** Tek bir soru sor: *"Bu nesneyi EF Core bu istek içinde veritabanından kendisi mi çekti?"* Cevap evetse takip ediliyordur, `Update()` gereksizdir. Hayırsa (formdan geldi, `new` ile oluşturdun, `AsNoTracking` ile çektin) EF Core onu tanımıyordur; ya `Update()` ile tanıtırsın ya da veritabanından çekip üzerine yazarsın.

**Teknik olarak:** Bu sorunun cevabı tek bir şeye bağlıdır: **nesne takip ediliyor mu?**

| Durum | `Update()` gerekli mi | Neden |
|---|---|---|
| Nesne veritabanından çekildi, sonra değiştirildi | **Hayır** | EF Core zaten takip ediyor, değişikliği kendisi algılar |
| Nesne formdan (POST) geldi | **Evet** | EF Core bu nesneyi tanımıyor (`Detached`), işaretlemen gerekir |
| Nesne `AsNoTracking()` ile çekildi | **Evet** | Takip kapatıldığı için değişiklik algılanmaz |
| Nesne başka bir context'ten geldi | **Evet** | Bu context onu tanımıyor |

```csharp
// TAKİP EDİLİYOR → Update() gereksiz
var urun = await _context.Products.FindAsync(5);
urun.Stock -= 1;
await _context.SaveChangesAsync();          // UPDATE otomatik üretilir

// TAKİP EDİLMİYOR → Update() gerekli
public async Task<IActionResult> EditProduct(Product formdanGelenUrun)
{
    _context.Products.Update(formdanGelenUrun);   // "bunu güncellenecek olarak işaretle"
    await _context.SaveChangesAsync();
}
```

Tablodaki "gerekli" satırları için bir hatırlatma: `Update()` **gerekli** olması, **doğru seçim** olduğu anlamına gelmez. 5. ve 6. bölümde görüldüğü gibi, tanınmayan bir nesneyi `Update()` ile tanıtmak tüm sütunları yazdırır. Çoğu senaryoda doğru hamle, nesneyi tanıtmak değil, **veritabanından tanınan hâlini çekip** üzerine yazmaktır.

Karar zinciri kısaca şöyledir:

```
Nesne bu context tarafından mı çekildi?
  ├─ Evet → sadece değiştir, SaveChanges yeter
  └─ Hayır → Formun tüm alanları entity'yi kapsıyor mu?
               ├─ Evet → Update() kullanılabilir (bilerek)
               └─ Hayır → FindAsync ile çek, değişen alanları ata (önerilen)
```

---

## Tek Bakışta Özet

- **Change tracking** EF Core'un kalbidir: çektiği nesneleri izler, değişiklikleri `SaveChanges`'te SQL'e çevirir. Karşılaştırma, çekildiği andaki **orijinal değer kopyası** ile yapılır.
- `Add` / `Update` / `Remove` **yazmaz, işaretler**. Tek yazma anı `SaveChanges` ve o da tek transaction.
- Nesne durumları: **Unchanged, Added, Modified, Deleted, Detached**. `Detached` "silinmiş" değil, "tanınmıyor" demektir.
- Durumu tahmin etmek yerine `_context.Entry(nesne).State` ile **gözünle görebilirsin**.
- Güncelleme **iki bağımsız HTTP isteğidir**; arada sunucu nesneyi unutur. Bu yüzden formda **gizli `Id`** şarttır.
- `Update(formNesnesi)` **tüm sütunları** yazar — çünkü elinde orijinal kopya yoktur, karşılaştıracak bir şey bulamaz. Formda olmayan alanlar ezilir.
- Çek-ve-ata yöntemi bunu önler, `Update()` gerektirmez ve değişiklik yoksa hiç sorgu atmaz.
- **Over-posting** güncellemede iki yönlü çalışır: gönderilmeyen alan sıfırlanır, gönderilmemesi gereken alan yazılır. Çözüm entity yerine **edit modeli (DTO)**.
- **Hard delete** veri bütünlüğünü bozabilir; kurumsal uygulamalarda **soft delete** (`IsDeleted`) tercih edilir. Soft delete arka planda bir `UPDATE`'tir ve ancak filtreyle anlam kazanır; global query filter tekrarlı `Where`'i ortadan kaldırır.
- `Update()` yalnızca **takip edilmeyen** nesneler için gereklidir — ve gerekli olması, en iyi seçim olduğu anlamına gelmez.
- Salt okumada `AsNoTracking()` ücretsiz performans kazancıdır; ama o nesneyi güncellemeye kalkarsan **hata bile almadan** hiçbir şey olmaz.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Change tracking | EF Core'un nesnelerdeki değişiklikleri izlemesi |
| Snapshot | Nesnenin çekildiği andaki değerlerinin saklanan kopyası |
| Entity state | Takip edilen nesnenin durumu (Added, Modified, Deleted...) |
| `Detached` | EF Core'un tanımadığı, takip etmediği nesne |
| `ChangeTracker` | Takip edilen tüm nesnelere ve durumlarına erişim noktası |
| `Entry(nesne)` | Tek bir nesnenin durumuna ve alan bilgilerine erişim |
| `Attach` | Var olan bir kaydı `Unchanged` olarak takibe alma |
| `SaveChanges` | İşaretli tüm değişiklikleri tek transaction'da yazma |
| Transaction | Ya hepsi ya hiçbiri garantisi olan işlem paketi |
| Rollback | Hata durumunda tüm değişikliklerin geri alınması |
| `AsNoTracking` | Takip listesine almadan salt okuma |
| `FindAsync` | Birincil anahtarla arama; önce hafızaya bakar |
| Scoped yaşam süresi | `DbContext`'in istek başına oluşturulup istek bitince atılması |
| `ModelState.IsValid` | Model binding ve doğrulama hatalarının toplu kontrolü |
| `nameof` | Metot/özellik adını string olarak veren derleme anı işleci |
| `[ValidateAntiForgeryToken]` | Formun gerçekten kendi sayfandan geldiğini doğrulama (CSRF koruması) |
| Over-posting | Formda olmayan alanların gönderilerek veriyi manipüle etmesi |
| Edit modeli / DTO | Yalnızca düzenlenecek alanları içeren, entity'den ayrı sınıf |
| Hard delete | Kaydı tablodan fiziksel olarak silme |
| Soft delete | Kaydı silmek yerine pasif işaretleme (`IsDeleted`) |
| Global query filter | Tüm sorgulara otomatik eklenen filtre |
| `IgnoreQueryFilters` | Global filtreyi o sorgu için devre dışı bırakma |
| Concurrency | Aynı kaydın eşzamanlı değiştirilmesi sorunu |
| POST-Redirect-GET | Form gönderimi sonrası yönlendirerek tekrarı önleme |
| Projeksiyon (`Select`) | Yalnızca gereken kolonları çekme |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`Add` çağrılınca kayıt eklenir" | `SaveChanges` çağrılana kadar veritabanına hiçbir şey gitmez |
| "`Update()` sadece değişen alanları yazar" | Varsayılan olarak **tüm sütunları** yazar; formda olmayan alanlar ezilir |
| "Değiştirdiğim her nesne için `Update()` çağırmalıyım" | Takip edilen nesnede gereksizdir; `SaveChanges` yeter |
| "`Detached` nesne silinmiş nesnedir" | `Detached`, EF Core'un o nesneyi hiç tanımadığı anlamına gelir |
| "Id'si aynıysa EF Core nesneyi tanır" | Takip referansa bağlıdır; aynı Id'li yeni bir nesne yine `Detached`'tır |
| "Gizli Id alanı güvenlik sağlar" | Yalnızca taşıma içindir; kullanıcı değiştirebilir, sunucuda doğrulanmalı |
| "Hard delete daha temizdir" | İlişkili kayıtları yetim bırakır; kurumsalda soft delete tercih edilir |
| "Soft delete kaydı gizler" | Kaydı gizleyen şey filtredir; `IsDeleted` tek başına hiçbir şey yapmaz |
| "`AsNoTracking` her yerde kullanılmalı" | Güncelleyeceğin nesnelerde kullanılırsa değişiklikler algılanmaz — üstelik hata da vermez |
| "GET'te çektiğim nesne POST'ta hâlâ takipte" | HTTP stateless'tır; POST yeni bir context ile gelir |
| "Çek-ve-ata yöntemi eşzamanlılık sorununu çözer" | Riski azaltır; gerçek çözüm `RowVersion` gibi concurrency kontrolüdür |

---

## Sonraki

→ `05-Action-Sonuclari.md` — `IActionResult` ve `ActionResult<T>`: bir action'ın geriye tam olarak ne döndürdüğü, sonuç tiplerinin hiyerarşisi ve hangi durumda hangisinin seçileceği.

Sonrasında Hafta 4'ün kalan konuları (Identity, filter'lar, view component'ler, Serilog ile loglama) yol haritasındaki sırayla eklenecek.

İlgili okumalar:
- Repository ve Unit of Work desenleri → `03-Projeler/01-MvcCv/02-Alternatif-Yapilar.md`
- Bu kavramların gerçek bir projedeki hâli → `03-Projeler/01-MvcCv/00-Proje-Dokumani.md`
