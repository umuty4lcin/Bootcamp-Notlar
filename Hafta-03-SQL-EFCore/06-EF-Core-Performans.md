# Hafta 3 · Cumartesi — EF Core Performansı

**Okuma süresi:** ~75 dk
**Neden bu konu:** EF Core'da yavaş kod, yanlış kod gibi görünmez. Derlenir, testler geçer, on kayıtlık geliştirme veritabanında anında döner. Fark, tabloda yüz bin satır olduğunda ortaya çıkar. Bootcamp'te yazacağın her listeleme ekranı bu tuzakların birinin üstünden geçer. MvcCv'deki `GenericRepository.List()` metodu da tam olarak bu tuzaklardan birine oturuyor — bu notta onu da açacağız.

---

## Önce Basitçe

EF Core sana bir kolaylık verir: veritabanını C# nesnesi gibi kullanırsın. `_db.Musteriler.ToList()` yazarsın, elinde müşteri listesi olur. Arada SQL yazmamış olursun. Bu kolaylık gerçek ve değerlidir.

Ama bir bedeli var: yazdığın satırın veritabanında kaça mal olduğunu göremezsin. `musteri.Siparisler` yazmak C# tarafında sadece bir özellik okumasıdır. Veritabanı tarafında ise ayrı bir sorgu, ayrı bir gidiş-geliş olabilir. Bunu bir döngünün içine koyarsan yüz sorgu olur. Kod hiç değişmez, maliyet yüz katına çıkar.

Performans meselesi aslında iki basit soruya indirgenir. Birincisi: **kaç kere gidiyorsun?** Veritabanına her gidiş, ağ üzerinden bir tur, sunucuda bir plan, geri dönüşte bir bekleme demektir. Yüz küçük sorgu, bir orta sorgudan neredeyse her zaman yavaştır. İkincisi: **her gidişte ne getiriyorsun?** Sadece müşterinin adı lazımken bütün satırı, bütün kolonlarıyla çekiyorsan, hem ağ hem bellek hem de EF Core'un iç muhasebesi boşuna çalışır.

Üçüncü bir soru daha var ama o ilk ikisinin arkasında saklıdır: **getirdiğin şeyi takip ediyor musun?** EF Core, sana verdiği her nesnenin bir kopyasını kenarda tutar ve "acaba değişti mi" diye izler. Değiştireceksen bu gereklidir. Sadece ekranda göstereceksen tamamen boşa harcanmış iştir — ve büyük listelerde bu israf hissedilir hâle gelir.

Bu notta tek tek bu üç sorunun cevaplarını göreceksin. Takip açık mı kapalı mı, ilişkili veriyi nasıl getiriyorsun, kaç sorgu üretiyorsun, sorguyu nasıl inceliyorsun. Hepsinin ortak noktası şu: **önce gerçekten ne olduğunu gör, sonra düzelt.** Tahmine dayalı optimizasyon çoğu zaman kodu çirkinleştirir ve hiçbir şeyi hızlandırmaz. Şimdi detaya iniyoruz.

> **Ana benzetme:** EF Core kullanmak, **markete alışverişe gitmek** gibidir. İki maliyet vardır: kaç kere gidip geldiğin ve her seferinde arabaya ne attığın. On kalem için on ayrı sefer yapmak, hepsini tek seferde almaktan kat kat pahalıdır. Bir kutu süt lazımken bütün rafı arabaya boşaltmak da ayrı bir israftır. Üstüne bir de her aldığın şeyi eve girerken deftere yazıyorsan — ki EF Core varsayılan olarak bunu yapar — sadece bakıp bırakacağın şeyler için bile defter tutmuş olursun.

---

## Bu Notta Ne Var

1. Ölçmeden başlama — performansın çerçevesi
2. Tracking ve no-tracking
3. Yükleme stratejileri: eager, explicit, lazy
4. N+1 problemi
5. Cartesian explosion ve `AsSplitQuery()`
6. Projeksiyon — en ucuz kazanç
7. Client-side evaluation
8. Toplu işlemler: `ExecuteUpdate` ve `ExecuteDelete`
9. Sorguyu inceleme: `ToQueryString`, loglama, `TagWith`
10. Compiled query ve DbContext pooling
11. Sayfalama, keyset pagination ve ham SQL

---

## 1. Ölçmeden Başlama — Performansın Çerçevesi

> **Benzetme —** Bakkala "bir ekmek" demek için gidersin, bakkal sana bütün ekmek rafını uzatır: "buyur, içinden seçersin". Sen de yüz ekmeği eve taşır, birini alır, kalanını çöpe atarsın. Kimse bunu yapmaz. Ama kodda `ToList()` yazdığında tam olarak bunu yapıyor olabilirsin ve hiçbir şey seni uyarmaz.

**Basitçe:** EF Core'da performans sorunlarının neredeyse hepsi üç başlıktan çıkar: gereğinden çok gidiş, gereğinden çok veri, gereksiz takip. Notun geri kalanı bu üçünün çeşitlemeleridir.

**Teknik olarak:** Bir EF Core sorgusunun maliyeti dört kalemden oluşur:

| Kalem | Neye bağlı |
|---|---|
| Sorgu derleme | LINQ ifadesinin SQL'e çevrilmesi — önbelleklenir |
| Ağ turu (round trip) | Kaç ayrı sorgu gönderildiği |
| Veritabanı işi | Index kullanımı, taranan satır sayısı |
| Materialization | Gelen satırların C# nesnesine dönüştürülmesi ve takibe alınması |

İlk ikisi senin LINQ yazımınla, üçüncüsü şemayla, dördüncüsü de ne kadar veri istediğinle belirlenir.

### 1.1 Gerçek bir örnek: `GenericRepository.List()`

MvcCv projesindeki generic repository'nin listeleme metodu şu:

```csharp
// MvcCv/Repositories/GenericRepository.cs
public List<T> List()
{
    return _table.ToList();
}
```

Tek satır, tertemiz görünüyor. Üretilen SQL de tertemiz:

```sql
SELECT [t].[Id], [t].[Baslik], [t].[Aciklama], [t].[Tarih], ...
FROM [TblDeneyimlerim] AS [t];
```

Bu metotta dört ayrı sorun birlikte duruyor:

| Sorun | Sonuç |
|---|---|
| `WHERE` yok | Tablodaki **her** satır gelir |
| `ORDER BY` yok | Sıra garanti değildir; aynı sorgu farklı sırayla dönebilir |
| Sayfalama yok | 50.000 kayıtta 50.000 nesne belleğe kurulur |
| Tracking açık | 50.000 nesnenin ayrıca snapshot'ı tutulur — bellek iki katına çıkar |

MvcCv bir kişisel CV sitesi; tablolarında on-yirmi satır var, bu yüzden sorun görünmüyor. Ama aynı metodu bir sipariş tablosuna uygularsan uygulama durur. Notun sonunda bu metodun düzeltilmiş hâlini göreceksin.

> **Uyarı:** "Şimdilik az kayıt var" cümlesi, performans borcunun standart açılış cümlesidir. Veri her zaman beklediğinden hızlı büyür ve o gün geldiğinde `List()` metodunu çağıran otuz yer olur.

### 1.2 Önce gör, sonra düzelt

Performans çalışmasının sırası şudur:

1. Sorguyu ve üretilen SQL'i **gör** (9. bölüm).
2. Kaç sorgu üretildiğini **say**.
3. Gereksiz kolon ve satırı **kes** (6. bölüm).
4. Gereksiz takibi **kapat** (2. bölüm).
5. Gerekirse veritabanı tarafına (index) bak.

Bu sırayı atlayıp doğrudan `AsNoTracking()` serpiştirmek, yüz sorgu üreten bir kodu doksan dokuz sorgu üreten bir koda çevirir.

> **Bu benzetme şurada bozulur:** Bakkal örneğinde bütün rafı taşıdığını anlarsın — kolların kopar. Kodda böyle bir geri bildirim yoktur. `ToList()` ile gelen elli bin nesne, geliştirme makinesinde göz kırpması kadar sürer. Acıyı hissetmediğin için de düzeltmeye gerek duymazsın. EF Core'da performans, hissedilen değil **ölçülen** bir şeydir.

---

## 2. Tracking ve No-Tracking

> **Benzetme —** Otelin vestiyerine paltonu bıraktığında görevli fiş yazar, numara verir, defterine işler. Bu iş bir maliyettir ama paltonu geri alacaksan gereklidir. Şimdi düşün: vestiyerden geçip sadece askıdaki paltolara bakacaksan ve hiçbirini almayacaksan, görevlinin her palto için fiş yazması tamamen boşunadır.

**Basitçe:** EF Core, sana verdiği her nesnenin bir kopyasını saklar ve değişip değişmediğini izler. Kaydı güncelleyeceksen bu şart. Sadece ekranda göstereceksen tamamen gereksizdir.

**Teknik olarak:** **Change tracker (değişiklik takipçisi)** — `DbContext`'in, sorgudan dönen her varlık için orijinal değerlerin bir kopyasını (snapshot) tutan ve `SaveChanges` anında farkı hesaplayan bileşeni. Detayları Hafta 4'teki `04-EF-Core-CRUD-ve-Change-Tracking.md` notunda.

Takibin maliyeti üç kalemdir:

| Maliyet | Açıklama |
|---|---|
| Bellek | Her varlık için ikinci bir değer kopyası |
| CPU | `DetectChanges` çağrılarında tüm takip edilen varlıkların taranması |
| Identity resolution | Aynı anahtarlı satırın tek nesneye eşlenmesi için sözlük bakımı |

### 2.1 `AsNoTracking()`

```csharp
// Takipli — varsayılan
var urunler = await _db.Urunler.ToListAsync();

// Takipsiz — sadece okuma
var urunlerSalt = await _db.Urunler
    .AsNoTracking()
    .ToListAsync();
```

Fark, satır sayısı arttıkça belirginleşir. On satırda ölçülmez; on bin satırda hem bellek hem süre gözle görülür şekilde düşer.

```csharp
// Tipik listeleme action'ı — burada tracking'in hiçbir faydası yok
public async Task<IActionResult> Index()
{
    var siparisler = await _db.Siparisler
        .AsNoTracking()
        .Include(s => s.Musteri)
        .OrderByDescending(s => s.Tarih)
        .Take(50)
        .ToListAsync();

    return View(siparisler);
}
```

### 2.2 `AsNoTrackingWithIdentityResolution()`

`AsNoTracking()` kullandığında EF Core aynı satırı iki kez gördüğünde **iki ayrı nesne** üretir. Normalde sorun değildir; ama `Include` ile aynı müşteriyi yüz siparişte tekrar görüyorsan yüz ayrı `Musteri` nesnesi oluşur.

```csharp
var siparisler = await _db.Siparisler
    .AsNoTracking()
    .Include(s => s.Musteri)
    .ToListAsync();

// Aynı müşterinin iki siparişi olsa bile:
bool ayniMi = ReferenceEquals(siparisler[0].Musteri, siparisler[1].Musteri);  // false
```

EF Core 5'ten itibaren ara bir seçenek var — takip yok, ama kimlik çözümlemesi var:

```csharp
var siparisler2 = await _db.Siparisler
    .AsNoTrackingWithIdentityResolution()
    .Include(s => s.Musteri)
    .ToListAsync();

bool ayniMi2 = ReferenceEquals(siparisler2[0].Musteri, siparisler2[1].Musteri);  // true
```

| Seçenek | Takip | Kimlik çözümleme | Bellek |
|---|---|---|---|
| Varsayılan (tracking) | Var | Var | En yüksek |
| `AsNoTrackingWithIdentityResolution()` | Yok | Var | Orta |
| `AsNoTracking()` | Yok | Yok | En düşük |

### 2.3 Global ayar

Uygulamanın büyük bölümü okuma ağırlıklıysa varsayılanı çevirebilirsin:

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseSqlServer(connectionString);
    opt.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

Ya da context içinden:

```csharp
_db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

Bu durumda takip gereken yerlerde tersini yazarsın:

```csharp
var urun = await _db.Urunler
    .AsTracking()                          // global ayar NoTracking olsa bile takip et
    .FirstAsync(u => u.Id == id);

urun.Fiyat = 250m;
await _db.SaveChangesAsync();
```

> **Uyarı:** Global `NoTracking` güçlü bir ayardır ama tehlikelidir. Güncelleme yapan bir kod `AsTracking()` yazmayı unutursa, `SaveChanges()` sessizce hiçbir şey yapmaz. Hata da vermez. Ekip alışkanlığı oturmadan bu ayarı açma; onun yerine listeleme sorgularına tek tek `AsNoTracking()` yaz.

### 2.4 Ne zaman tracking şart

| Durum | Tracking |
|---|---|
| Listeleme, rapor, ekranda gösterme | Gerekmez |
| API'den DTO döndürme | Gerekmez |
| Kaydı çekip alanını değiştirip `SaveChanges` | **Şart** |
| `Remove()` ile silme | **Şart** |
| İlişkili nesne ekleme (`musteri.Siparisler.Add(...)`) | **Şart** |
| `Find()` ile önbellekten okuma beklentisi | **Şart** — `Find` takip edilen nesneye bakar |

```csharp
// YANLIŞ — SaveChanges hiçbir şey yapmaz, hata da vermez
var urun = await _db.Urunler.AsNoTracking().FirstAsync(u => u.Id == 5);
urun.Fiyat = 199m;
await _db.SaveChangesAsync();               // 0 satır etkilendi

// DOĞRU
var urun2 = await _db.Urunler.FirstAsync(u => u.Id == 5);
urun2.Fiyat = 199m;
await _db.SaveChangesAsync();               // UPDATE çalışır
```

> **Bu benzetme şurada bozulur:** Vestiyerde fiş almadan paltonu geri isteyemezsin — görevli seni durdurur. EF Core seni durdurmaz. `AsNoTracking()` ile çektiğin nesneyi değiştirip `SaveChanges()` çağırırsan istisna almazsın; hiçbir şey olmaz. Sessiz başarısızlık, gürültülü hatadan her zaman daha zor bulunur.

---

## 3. Yükleme Stratejileri: Eager, Explicit, Lazy

> **Benzetme —** Kütüphaneden bir kitap istiyorsun ve kitabın bir de ek cilt haritası var. Üç yol var. Birincisi: baştan "kitabı ve haritasını birlikte ver" dersin, görevli ikisini bir seferde getirir. İkincisi: önce kitabı alırsın, haritaya ihtiyacın olursa geri dönüp "haritayı da ver" dersin. Üçüncüsü: kitabı alırsın, haritanın sayfasını açtığın anda görevli arkadan fırlayıp haritayı getirir — sen istemeden, sen fark etmeden.

**Basitçe:** İlişkili veriyi (müşterinin siparişleri gibi) üç farklı şekilde getirebilirsin: baştan birlikte, sonradan elle, ya da dokunduğun anda otomatik.

### 3.1 Eager loading — `Include` / `ThenInclude`

**Teknik olarak:** **Eager loading (istekli yükleme)** — İlişkili veriyi ana sorgunun parçası olarak, aynı gidişte getirme.

```csharp
var musteriler = await _db.Musteriler
    .Include(m => m.Siparisler)
    .ToListAsync();
```

Derine inmek için `ThenInclude` zincirlenir:

```csharp
var musteriler2 = await _db.Musteriler
    .Include(m => m.Siparisler)
        .ThenInclude(s => s.Detaylar)
            .ThenInclude(d => d.Urun)
                .ThenInclude(u => u.Kategori)
    .ToListAsync();
```

EF Core 5'ten itibaren `Include` içinde filtreleme de yapabilirsin:

```csharp
var musteriler3 = await _db.Musteriler
    .Include(m => m.Siparisler
                   .Where(s => s.Tarih >= DateTime.Today.AddMonths(-1))
                   .OrderByDescending(s => s.Tarih)
                   .Take(5))
    .ToListAsync();
```

Filtrelenmiş `Include` içinde yalnızca `Where`, `OrderBy`, `ThenBy`, `Skip` ve `Take` kullanılabilir.

> **Uyarı:** Aynı navigation'a hem filtreli hem filtresiz `Include` yazarsan EF Core istisna fırlatır. Bir navigation için tek bir `Include` tanımı olmalıdır.

### 3.2 Explicit loading — `Entry().Load()`

**Teknik olarak:** **Explicit loading (açık yükleme)** — Ana nesne elindeyken, ilişkili veriyi ayrı bir çağrıyla, bilinçli olarak yükleme.

```csharp
var musteri = await _db.Musteriler.FirstAsync(m => m.Id == 5);

// Koleksiyon navigation
await _db.Entry(musteri).Collection(m => m.Siparisler).LoadAsync();

// Referans navigation
var siparis = musteri.Siparisler.First();
await _db.Entry(siparis).Reference(s => s.Musteri).LoadAsync();
```

`Query()` ile yüklemeden önce filtreleyebilir ya da sadece sayabilirsin:

```csharp
// Sadece son 10 siparişi yükle
await _db.Entry(musteri)
    .Collection(m => m.Siparisler)
    .Query()
    .OrderByDescending(s => s.Tarih)
    .Take(10)
    .LoadAsync();

// Hiç yüklemeden sadece say
int adet = await _db.Entry(musteri)
    .Collection(m => m.Siparisler)
    .Query()
    .CountAsync();
```

Explicit loading, ilişkili veriye **bazen** ihtiyaç duyduğun durumlarda değerlidir: şart sağlanırsa yükle, sağlanmazsa hiç dokunma.

### 3.3 Lazy loading — proxy paketi

**Teknik olarak:** **Lazy loading (tembel yükleme)** — Navigation özelliğine ilk erişildiği anda ilişkili verinin otomatik olarak yüklenmesi.

Üç şart birden gerekir:

```csharp
// 1) Paket: Microsoft.EntityFrameworkCore.Proxies
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseLazyLoadingProxies()
       .UseSqlServer(connectionString));
```

```csharp
// 2) Navigation'lar virtual olmalı
public class Musteri
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public virtual List<Siparis> Siparisler { get; set; } = new();   // virtual
}

public class Siparis
{
    public int Id { get; set; }
    public int MusteriId { get; set; }
    public virtual Musteri Musteri { get; set; } = null!;            // virtual
}
```

```csharp
// 3) Sınıf sealed olmamalı, kurucusu public/protected olmalı
```

Kullanımda hiçbir özel çağrı yoktur — tam da sorun budur:

```csharp
var musteri = await _db.Musteriler.FirstAsync(m => m.Id == 5);
// Buraya kadar tek sorgu

var adet = musteri.Siparisler.Count;
// Bu satır SESSİZCE ikinci bir sorgu çalıştırdı
```

Proxy paketi istemiyorsan `ILazyLoader` enjeksiyonu da vardır, ama nadiren tercih edilir.

### 3.4 Üçünün karşılaştırması

| | Eager (`Include`) | Explicit (`Load`) | Lazy (proxy) |
|---|---|---|---|
| Ne zaman yüklenir | Ana sorguyla birlikte | Sen çağırınca | Özelliğe dokununca |
| Sorgu sayısı | 1 (veya `AsSplitQuery` ile N) | Çağırdığın kadar | Dokunduğun kadar — kontrolsüz |
| Kodda görünür mü | Evet | Evet | **Hayır** |
| N+1 riski | Düşük | Orta | **Çok yüksek** |
| `AsNoTracking` ile çalışır mı | Evet | Hayır — takip gerekir | Hayır |
| Context kapandıktan sonra | Veri elinde | Hata | `ObjectDisposedException` |
| Ek paket | Gerekmez | Gerekmez | Gerekir |
| Sınıfa müdahale | Yok | Yok | `virtual` zorunlu |

### 3.5 Lazy loading neden varsayılan olmamalı

EF Core'da lazy loading varsayılan olarak **kapalıdır** ve bu bilinçli bir karardır.

| Sebep | Açıklama |
|---|---|
| Görünmezlik | Sorguyu tetikleyen satır, sorgu gibi görünmez |
| N+1 üretir | Döngü içinde navigation okumak yüz sorgu demektir (4. bölüm) |
| Serileştirmede patlar | JSON serileştirici tüm navigation'ları gezer, hepsini tetikler |
| Context ömrü | Context kapandıktan sonra erişim `ObjectDisposedException` verir |
| `AsNoTracking` ile çalışmaz | Takip kapalıysa proxy yükleyemez |
| Asenkron değildir | Yükleme senkron olur, thread bloklanır |

```csharp
// Lazy loading + JSON serileştirme = felaket
public async Task<IActionResult> Get(int id)
{
    var musteri = await _db.Musteriler.FindAsync(id);
    return Ok(musteri);
    // Serileştirici Siparisler'e dokunur → sorgu
    // Her siparişin Detaylar'ına dokunur → N sorgu
    // Her detayın Urun'üne dokunur → N*M sorgu
    // Urun.Kategori.Urunler → sonsuz döngü riski
}
```

**Pratik kural:** Varsayılanı eager loading yap. İhtiyacın koşulluysa explicit loading kullan. Lazy loading'i sadece eski bir kod tabanını taşırken ve geçici olarak aç.

> **Bu benzetme şurada bozulur:** Kütüphanede görevli haritayı getirirken sen onu görürsün — koşturduğunu fark edersin. Lazy loading'de hiçbir şey görmezsin. `musteri.Siparisler.Count` satırı, `liste.Count` satırından ayırt edilemez. Kütüphane benzetmesi maliyeti görünür kılıyor; kodda maliyet tamamen görünmezdir. Bütün mesele budur.

---

## 4. N+1 Problemi

> **Benzetme —** Elinde yüz kalemlik bir alışveriş listesi var. Bakkala gidip listeyi uzatmak yerine, her kalem için ayrı ayrı gidip geliyorsun. Yüz kere merdiven, yüz kere sokak, yüz kere kapı. Aldığın mal aynı, harcadığın zaman yüz katı. Üstelik bakkal bunu sana söylemez; her seferinde gülümseyip malı verir.

**Basitçe:** Bir sorguyla listeyi çekersin, sonra listenin her elemanı için ayrı bir sorgu daha çalışır. Yüz elemanlık listede toplam 101 sorgu olur. Kodda bunu gösteren hiçbir işaret yoktur.

**Teknik olarak:** **N+1 problemi** — Bir sorgu (ana liste) artı liste elemanı başına bir sorgu (N adet) üreten erişim kalıbı.

### 4.1 Nasıl oluşur

```csharp
// 1 sorgu: 100 sipariş
var siparisler = await _db.Siparisler.Take(100).ToListAsync();

foreach (var s in siparisler)
{
    // Her tur için 1 sorgu daha — toplam 100 sorgu
    Console.WriteLine($"{s.Id} - {s.Musteri.Ad}");
}
// TOPLAM: 101 sorgu
```

Lazy loading kapalıysa bu kod `NullReferenceException` verir — ki bu aslında **iyi haberdir**, sorunu hemen görürsün. Lazy loading açıksa kod çalışır ve 101 sorgu sessizce gider.

Explicit loading ile de aynı tuzağa düşersin:

```csharp
// Bu da N+1'dir — sadece açıkça yazılmıştır
foreach (var m in musteriler)
{
    await _db.Entry(m).Collection(x => x.Siparisler).LoadAsync();
}
```

Repository katmanı bu kalıbı özellikle kolaylaştırır:

```csharp
// Görünüşte masum — gerçekte N+1
var siparisler = _siparisRepo.List();                     // 1 sorgu
foreach (var s in siparisler)
{
    var musteri = _musteriRepo.Get(s.MusteriId);          // her tur 1 sorgu
    ViewBag.Adlar.Add(musteri.Ad);
}
```

### 4.2 Üretilen SQL

101 sorgu şuna benzer:

```sql
-- 1 kere
SELECT TOP(100) [s].[Id], [s].[Tarih], [s].[MusteriId], [s].[ToplamTutar]
FROM [Siparisler] AS [s];

-- 100 kere, her seferinde farklı @p0 ile
SELECT [m].[Id], [m].[Ad], [m].[Telefon], [m].[KayitTarihi]
FROM [Musteriler] AS [m] WHERE [m].[Id] = @p0;
```

Her sorgu belki 1 ms sürüyor. Ama her biri ayrı bir ağ turu. Veritabanı başka bir sunucudaysa tur başına 2-5 ms gecikme eklenir. 100 sorgu = 300-500 ms, sadece bekleme.

### 4.3 Nasıl fark edilir

Tek güvenilir yol logları açmaktır (9. bölüm):

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseSqlServer(connectionString);
    if (builder.Environment.IsDevelopment())
    {
        opt.LogTo(Console.WriteLine, LogLevel.Information);
        opt.EnableSensitiveDataLogging();
    }
});
```

Bir sayfayı açtığında konsolda arka arkaya onlarca `Executing DbCommand` satırı görüyorsan, N+1 vardır. Aynı SQL metninin sadece parametresi değişerek tekrar etmesi en net işarettir.

### 4.4 Üç çözümü

**Birincisi — eager loading:**

```csharp
var siparisler = await _db.Siparisler
    .Include(s => s.Musteri)
    .Take(100)
    .ToListAsync();
// 1 sorgu (JOIN'li)
```

**İkincisi — projeksiyon (çoğu zaman en iyisi):**

```csharp
var satirlar = await _db.Siparisler
    .Take(100)
    .Select(s => new SiparisSatiriDto
    {
        Id = s.Id,
        Tarih = s.Tarih,
        MusteriAdi = s.Musteri.Ad,             // JOIN üretir, tüm müşteriyi çekmez
        Tutar = s.ToplamTutar
    })
    .ToListAsync();
// 1 sorgu, sadece 4 kolon
```

**Üçüncüsü — toplu ön yükleme (sözlüğe alma):**

`Include` zincirinin çalışmadığı, iki ayrı kaynaktan veri birleştirdiğin durumlarda kullanışlıdır:

```csharp
var siparisler = await _db.Siparisler.Take(100).AsNoTracking().ToListAsync();

var idler = siparisler.Select(s => s.MusteriId).Distinct().ToList();

var musteriler = await _db.Musteriler
    .Where(m => idler.Contains(m.Id))          // tek sorgu, IN (...) üretir
    .AsNoTracking()
    .ToDictionaryAsync(m => m.Id);

foreach (var s in siparisler)
{
    var ad = musteriler[s.MusteriId].Ad;       // bellekten okuma, sorgu yok
}
// TOPLAM: 2 sorgu
```

| Çözüm | Sorgu sayısı | Ne zaman |
|---|---|---|
| `Include` | 1 | Tam varlığa ihtiyaç varsa |
| Projeksiyon | 1 | Sadece birkaç alan lazımsa — en ucuzu |
| Sözlüğe ön yükleme | 2 | `Include` kurulamayan durumlarda |

> **Uyarı:** `Contains` ile geçirdiğin liste çok büyükse (binlerce Id), SQL Server parametre sınırına (2100) takılırsın ve sorgu planı bozulur. Bu durumda listeyi parçalara böl ya da geçici tablo / `JOIN` kurgusuna geç.

> **Bu benzetme şurada bozulur:** Bakkala yüz kere gitmek seni yorar, dolayısıyla ikinci seferde fark edersin. Kodda yorulan sen değilsin, sunucu. Ve sunucu geliştirme ortamında yorulmaz çünkü tabloda on satır var. N+1'in en zalim tarafı, sorunu üreten kişinin onu asla hissetmemesidir.

---

## 5. Cartesian Explosion ve `AsSplitQuery()`

> **Benzetme —** Düğün için iki liste var: on yemek çeşidi ve on müzik parçası. Bunları tek bir kâğıda "her yemek için her parça" diye yazarsan yüz satır olur. Oysa bilgi hâlâ yirmi satırlık. Kâğıdı okuyan kişi de aynı yemeği on kez görür ve "bu kadar çok yemek mi var" diye şaşırır.

**Basitçe:** Birden çok koleksiyonu aynı anda `Include` ettiğinde, veritabanı bunları çarpar. On siparişi ve beş adresi olan bir müşteri için elli satır döner. Bilgi aynı, taşınan veri kat kat fazla.

**Teknik olarak:** **Cartesian explosion (kartezyen patlama)** — Aynı ana varlığa bağlı birden çok koleksiyonun tek `JOIN`'li sorguda birleştirilmesi sonucu satır sayısının çarpılarak artması.

```csharp
var musteriler = await _db.Musteriler
    .Include(m => m.Siparisler)        // müşteri başına 10 sipariş
    .Include(m => m.Adresler)          // müşteri başına 5 adres
    .ToListAsync();
```

Üretilen SQL tek bir sorgudur:

```sql
SELECT [m].[Id], [m].[Ad], [s].[Id], [s].[Tarih], [a].[Id], [a].[Il]
FROM [Musteriler] AS [m]
LEFT JOIN [Siparisler] AS [s] ON [m].[Id] = [s].[MusteriId]
LEFT JOIN [Adresler] AS [a] ON [m].[Id] = [a].[MusteriId]
ORDER BY [m].[Id], [s].[Id];
```

100 müşteri için satır sayısı: `100 × 10 × 5 = 5.000`. Gerçek veri ise `100 + 1.000 + 500 = 1.600` satırlık. Üstelik müşteri bilgileri 50 kez, sipariş bilgileri 5 kez tekrarlanarak ağdan geçer.

Üçüncü bir koleksiyon eklersen çarpım devam eder: `100 × 10 × 5 × 3 = 15.000` satır.

> **Uyarı:** Patlama sadece **koleksiyon** navigation'larında olur. Referans navigation'lar (`Include(s => s.Musteri)`) satır sayısını artırmaz — her satır zaten tek bir müşteriye bağlıdır. İstediğin kadar referans `Include` edebilirsin.

### 5.1 `AsSplitQuery()` çözümü

EF Core 5'ten itibaren her koleksiyonu ayrı bir sorguya bölebilirsin:

```csharp
var musteriler = await _db.Musteriler
    .Include(m => m.Siparisler)
    .Include(m => m.Adresler)
    .AsSplitQuery()
    .ToListAsync();
```

Bu üç ayrı sorgu üretir:

```sql
-- 1) 100 satır
SELECT [m].[Id], [m].[Ad] FROM [Musteriler] AS [m] ORDER BY [m].[Id];

-- 2) 1.000 satır
SELECT [s].[Id], [s].[Tarih], [s].[MusteriId] FROM [Siparisler] AS [s]
INNER JOIN [Musteriler] AS [m] ON [s].[MusteriId] = [m].[Id] ORDER BY [m].[Id];

-- 3) 500 satır
SELECT [a].[Id], [a].[Il], [a].[MusteriId] FROM [Adresler] AS [a]
INNER JOIN [Musteriler] AS [m] ON [a].[MusteriId] = [m].[Id] ORDER BY [m].[Id];
```

5.000 satır yerine 1.600 satır. EF Core, sonuçları bellekte birleştirip nesne grafiğini kurar.

Global olarak da açılabilir:

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString,
        sql => sql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

Global açtıysan tekil sorgu istediğin yerde `AsSingleQuery()` yazarsın.

### 5.2 Bölmenin bedeli

`AsSplitQuery()` bedava değildir:

| Bedel | Açıklama |
|---|---|
| Çoklu ağ turu | 1 yerine N sorgu — her biri ayrı gidiş-geliş |
| Tutarlılık riski | Sorgular arasında veri değişirse eksik/fazla satır gelebilir |
| `First`/`Single` ile sorun | Bölünmüş sorguda `Take(1)` mantığı karmaşıklaşır |
| Küçük veride yavaş | Az satırda tek sorgu her zaman daha hızlıdır |

Tutarlılık riskini kapatmak için açık transaction gerekir:

```csharp
using var tx = await _db.Database.BeginTransactionAsync(IsolationLevel.Snapshot);

var musteriler = await _db.Musteriler
    .Include(m => m.Siparisler)
    .Include(m => m.Adresler)
    .AsSplitQuery()
    .ToListAsync();

await tx.CommitAsync();
```

### 5.3 Ne zaman hangisi

| Durum | Seçim |
|---|---|
| Tek koleksiyon `Include` | Tek sorgu — bölmeye gerek yok |
| Sadece referans navigation'lar | Tek sorgu |
| İki veya daha çok koleksiyon, az satır | Tek sorgu |
| İki veya daha çok koleksiyon, çok satır | `AsSplitQuery()` |
| Koleksiyonlarda geniş kolonlar (`nvarchar(max)`) | `AsSplitQuery()` — tekrarı pahalıdır |
| Sadece birkaç alan lazım | İkisi de değil — **projeksiyon** |

> **Bu benzetme şurada bozulur:** Düğün listesinde çarpımı görürsün — kâğıt yüz satır uzunluğundadır. EF Core'da sonuç sana düzgün bir nesne grafiği olarak gelir: 100 müşteri, her birinde 10 sipariş. Patlamanın hiçbir izi kalmaz. Ağdan 5.000 satır geçtiğini sadece SQL'e bakarak anlayabilirsin. EF Core burada seni korumuyor, sadece pisliği örtüyor.

---

## 6. Projeksiyon — En Ucuz Kazanç

> **Benzetme —** Ankara'daki bir kuruma bir belge göndermen gerekiyor. Belgeyi koca dosya klasörünün içinde, bütün ekleriyle kargoya verebilirsin; ya da o tek sayfayı fotokopi çekip zarfa koyabilirsin. Karşı taraf zaten sadece o sayfayı okuyacak. Kargo ücreti, taşıma süresi ve karşı tarafın dosyayı karıştırma zahmeti — hepsi zarfla ortadan kalkar.

**Basitçe:** Sadece ihtiyacın olan kolonları çek. Tek satırlık bir değişiklikle hem daha az veri taşırsın, hem takip maliyetinden kurtulursun.

**Teknik olarak:** **Projection (projeksiyon)** — `Select` ile sorgu sonucunu varlık nesnesi yerine başka bir şekle dönüştürme. EF Core, `Select` içinde hangi alanlara dokunduğunu görür ve `SELECT` cümlesine **yalnızca onları** koyar.

```csharp
// KÖTÜ — 20 kolonluk varlığın tamamı geliyor, hepsi takip ediliyor
var musteriler = await _db.Musteriler.ToListAsync();
var adlar = musteriler.Select(m => m.Ad).ToList();
```

```csharp
// İYİ — tek kolon geliyor, takip yok
var adlar2 = await _db.Musteriler
    .Select(m => m.Ad)
    .ToListAsync();
```

Üretilen SQL farkı:

```sql
-- Kötü
SELECT [m].[Id], [m].[Ad], [m].[Telefon], [m].[Email], [m].[KayitTarihi],
       [m].[VergiNo], [m].[Notlar], [m].[FaturaAcikAdres], ...
FROM [Musteriler] AS [m];

-- İyi
SELECT [m].[Ad] FROM [Musteriler] AS [m];
```

### 6.1 DTO'ya projeksiyon

**DTO (Data Transfer Object — veri taşıma nesnesi)**, katmanlar arasında taşınacak veriyi tarif eden sade sınıftır.

```csharp
public class SiparisListeDto
{
    public int Id { get; set; }
    public DateTime Tarih { get; set; }
    public string MusteriAdi { get; set; } = null!;
    public decimal Tutar { get; set; }
    public int KalemSayisi { get; set; }
}

var liste = await _db.Siparisler
    .Where(s => s.Tarih >= DateTime.Today.AddDays(-30))
    .OrderByDescending(s => s.Tarih)
    .Select(s => new SiparisListeDto
    {
        Id = s.Id,
        Tarih = s.Tarih,
        MusteriAdi = s.Musteri.Ad,              // JOIN — tüm müşteri gelmez
        Tutar = s.ToplamTutar,
        KalemSayisi = s.Detaylar.Count()        // alt sorgu — detaylar gelmez
    })
    .ToListAsync();
```

Tek sorgu, beş kolon. `Include` yok, takip yok, N+1 yok. Bu kalıp, listeleme ekranlarının standart çözümüdür.

`record` ile daha kısa yazılır:

```csharp
public record SiparisOzet(int Id, DateTime Tarih, string MusteriAdi, decimal Tutar);

var ozet = await _db.Siparisler
    .Select(s => new SiparisOzet(s.Id, s.Tarih, s.Musteri.Ad, s.ToplamTutar))
    .ToListAsync();
```

### 6.2 İç içe projeksiyon

Koleksiyonları da projeksiyon içinde şekillendirebilirsin:

```csharp
var detayli = await _db.Siparisler
    .Where(s => s.Id == id)
    .Select(s => new
    {
        s.Id,
        s.Tarih,
        Musteri = new { s.Musteri.Ad, s.Musteri.Telefon },
        Kalemler = s.Detaylar
            .OrderBy(d => d.Id)
            .Select(d => new { d.Urun.Ad, d.Adet, d.BirimFiyat })
            .ToList()
    })
    .FirstOrDefaultAsync();
```

### 6.3 Projeksiyon takibi otomatik kapatır

Projeksiyonun sessiz faydası budur: sonuç bir varlık tipi değilse **takip edilmez**.

```csharp
// AsNoTracking yazmana gerek yok — DTO takip edilmez
var dtolar = await _db.Urunler
    .Select(u => new UrunDto { Id = u.Id, Ad = u.Ad })
    .ToListAsync();
```

Ama dikkat: projeksiyonun içinde **varlık nesnesi** taşırsan o varlık takip edilir.

```csharp
// Urun varlığı takip EDİLİR — anonim tipin içinde olması fark etmez
var karisik = await _db.Urunler
    .Select(u => new { Urun = u, StokDurumu = u.StokAdedi > 0 })
    .ToListAsync();
```

### 6.4 Projeksiyonun sınırları

| Sınır | Açıklama |
|---|---|
| Güncelleme yapılamaz | DTO'da değişiklik `SaveChanges` ile kaydedilmez |
| Lazy/explicit loading yok | DTO bir varlık değildir, navigation'ı yoktur |
| Karmaşık `Select` çevrilemeyebilir | Metot çağrıları SQL'e dönüşmezse hata alırsın |
| Kod tekrarı | Her ekran için ayrı DTO ve ayrı `Select` yazılır |

Son maddeye karşı, `Select` ifadesini tek yerde tanımlayıp tekrar kullanabilirsin:

```csharp
public static class SiparisProjections
{
    public static readonly Expression<Func<Siparis, SiparisListeDto>> ToListeDto =
        s => new SiparisListeDto
        {
            Id = s.Id,
            Tarih = s.Tarih,
            MusteriAdi = s.Musteri.Ad,
            Tutar = s.ToplamTutar
        };
}

// Kullanımı
var liste2 = await _db.Siparisler.Select(SiparisProjections.ToListeDto).ToListAsync();
```

> **Uyarı:** `Expression<Func<...>>` olarak yazmak şarttır. `Func<...>` yazarsan EF Core onu SQL'e çeviremez; sorgu belleğe düşer ve tüm tablo çekilir. Bu ayrımın detayı Hafta 1'deki `03-LINQ.md` notunda.

> **Bu benzetme şurada bozulur:** Kargoda dosyanın tamamını göndermek sadece pahalıdır, zararlı değildir. Projeksiyon yapmamak ise bazen güvenlik sorunudur: varlığı olduğu gibi API'den döndürürsen `ParolaHash`, `Notlar`, `IcNot` gibi alanlar da JSON'a girer. DTO sadece performans aracı değil, aynı zamanda neyin dışarı çıkacağını belirleyen sınırdır.

---

## 7. Client-Side Evaluation

> **Benzetme —** Bir tercümanla resmi bir görüşmeye gittin. Konuştuğun dilde "kolay gelsin" diye bir deyim var ama karşı dilde tam karşılığı yok. İyi tercüman durur ve "bunu çeviremiyorum, başka türlü söyler misin" der. Kötü tercüman ise susar, cümleyi atlar ve sen karşı tarafın anladığını sanırsın.

**Basitçe:** LINQ'te yazdığın her ifade SQL'e çevrilemez. Çevrilemeyen bir şey yazdığında EF Core hata verir — ve bu iyi bir şeydir.

**Teknik olarak:** **Client-side evaluation (istemci tarafı değerlendirme)** — Sorgunun bir kısmının veritabanında değil, veriyi belleğe çektikten sonra C# tarafında çalıştırılması.

### 7.1 EF Core 3.0 öncesi ve sonrası

EF Core 2.x, çeviremediği ifadeyi sessizce belleğe taşırdı:

```csharp
// EF Core 2.x davranışı: tüm tablo çekilir, filtre bellekte uygulanırdı
var sonuc = _db.Urunler
    .Where(u => TemizleVeKarsilastir(u.Ad, aranan))     // çevrilemez
    .ToList();
```

Kod çalışırdı, testler geçerdi, üretimde tablo büyüyünce uygulama dururdu. EF Core 3.0 bu davranışı kaldırdı. Artık `Where` içinde çevrilemeyen bir ifade `InvalidOperationException` fırlatır:

```
The LINQ expression 'DbSet<Urun>().Where(u => TemizleVeKarsilastir(u.Ad, __aranan_0))'
could not be translated. Either rewrite the query in a form that can be translated,
or switch to client evaluation explicitly by inserting a call to 'AsEnumerable',
'AsAsyncEnumerable', 'ToList', or 'ToListAsync'.
```

Hata metni çözümü de söylüyor: ya çevrilebilir biçimde yeniden yaz, ya da sınırı **bilerek** geç.

### 7.2 Sınırı bilerek geçmek

`AsEnumerable()`, `IQueryable`'ı `IEnumerable`'a çevirir. O noktadan sonrası bellekte çalışır.

```csharp
var sonuc = await _db.Urunler
    .Where(u => u.Aktif && u.KategoriId == 3)      // SQL'de — veritabanı filtreler
    .Select(u => new { u.Id, u.Ad, u.Fiyat })      // SQL'de — sadece 3 kolon
    .AsEnumerable()                                 // SINIR: buradan sonrası bellekte
    .Where(u => TemizleVeKarsilastir(u.Ad, aranan)) // C#'ta
    .ToList();
```

Kural basittir: **sınırı mümkün olduğunca geç koy.** Önce veritabanında filtrele, daralt, sadece gereken kolonları seç; kalan küçük kümeyi belleğe al.

```csharp
// YANLIŞ — sınır en başta, tüm tablo belleğe geliyor
var kotu = _db.Urunler
    .AsEnumerable()
    .Where(u => u.Aktif && u.Fiyat > 100)
    .ToList();

// DOĞRU — sınır en sonda
var iyi = _db.Urunler
    .Where(u => u.Aktif && u.Fiyat > 100)
    .AsEnumerable()
    .Where(u => OzelKural(u))
    .ToList();
```

### 7.3 Hâlâ izin verilen client evaluation

En üst seviyedeki `Select` içinde C# metodu çağırabilirsin — burada veri zaten daraltılmış olarak gelir:

```csharp
var liste = await _db.Urunler
    .Where(u => u.Aktif)
    .Select(u => new
    {
        u.Id,
        FormatliFiyat = u.Fiyat.ToString("C", new CultureInfo("tr-TR"))  // çalışır
    })
    .ToListAsync();
```

Bu, `Where` içinde olsaydı hata verirdi. Fark şudur: `Select`'teki dönüşüm satır sayısını değiştirmez, `Where`'deki filtre değiştirir. Filtreyi bellekte yapmak "önce hepsini getir" anlamına gelir; dönüşümü bellekte yapmak gelen satırları biçimlendirmek demektir.

### 7.4 Sık çevrilemeyenler

| İfade | Durum |
|---|---|
| `string.ToUpper()`, `Contains`, `StartsWith` | Çevrilir |
| `DateTime.Year`, `.Month`, `.Day` | Çevrilir |
| `Math.Abs`, `Math.Round` | Çevrilir |
| `string.Format(...)`, interpolation | Genelde çevrilir (`CONCAT`) |
| Kendi yazdığın metot | **Çevrilmez** |
| `Enum.Parse`, `Convert.ToInt32` | Genelde çevrilmez |
| `DateTime.ToString("dd.MM.yyyy")` | Çevrilmez |
| `GroupBy` sonrası karmaşık `Select` | Duruma göre |
| Value converter uygulanmış alanda büyüklük karşılaştırması | Çevrilir ama **yanlış sonuç** verir |

> **Uyarı:** Son satır özellikle sinsidir. `HasConversion<string>()` ile saklanan bir enum'da `Where(s => s.Durum > SiparisDurumu.Beklemede)` yazarsan, SQL tarafında metinler alfabetik karşılaştırılır. Hata almazsın, yanlış satırlar gelir.

> **Bu benzetme şurada bozulur:** Tercüman sustuğunda en azından ortam sessizleşir, bir şeyin ters gittiğini sezersin. EF Core 2.x'in sessiz client evaluation'ında hiçbir belirti yoktu — sorgu sonucu **doğruydu**, sadece bütün tablo belleğe gelmişti. Doğru sonuç veren yanlış kod, yanlış sonuç veren koddan daha uzun süre fark edilmeden yaşar.

---

## 8. Toplu İşlemler: `ExecuteUpdate` ve `ExecuteDelete`

> **Benzetme —** On bin dosyayı arşivden çıkarmak gerekiyor. Bir yol var: her dosyayı tek tek masaya getirip, üstüne "iptal" damgası vurup geri koymak. Bir yol daha var: arşiv sorumlusuna tek bir yazı yazıp "2019 öncesi bütün dosyaları iptal edin" demek. İkinci yol kat kat hızlıdır. Bedeli şudur: senin masandaki kopyalar hâlâ eski hâlini gösterir, çünkü sen onlara dokunmadın.

**Basitçe:** Binlerce kaydı değiştirmek için hepsini belleğe çekmek zorunda değilsin. EF Core 7'den itibaren doğrudan `UPDATE` ve `DELETE` cümlesi çalıştırabilirsin.

### 8.1 EF Core 7 öncesindeki problem

```csharp
// 2020 öncesi tüm siparişleri silmek
var eskiler = await _db.Siparisler
    .Where(s => s.Tarih < new DateTime(2020, 1, 1))
    .ToListAsync();                        // 50.000 nesne belleğe kuruldu

_db.Siparisler.RemoveRange(eskiler);
await _db.SaveChangesAsync();              // 50.000 DELETE cümlesi
```

Üç ayrı maliyet: 50.000 satırın çekilmesi, 50.000 nesnenin oluşturulup takibe alınması, 50.000 ayrı `DELETE` cümlesi.

### 8.2 `ExecuteDelete` ve `ExecuteUpdate` (EF Core 7+)

```csharp
// Tek DELETE cümlesi — hiçbir satır belleğe gelmez
int silinen = await _db.Siparisler
    .Where(s => s.Tarih < new DateTime(2020, 1, 1))
    .ExecuteDeleteAsync();
```

```sql
DELETE FROM [s] FROM [Siparisler] AS [s] WHERE [s].[Tarih] < '2020-01-01';
```

Güncelleme için `SetProperty` zincirlenir:

```csharp
int guncellenen = await _db.Urunler
    .Where(u => u.KategoriId == 3)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(u => u.Fiyat, u => u.Fiyat * 1.20m)
        .SetProperty(u => u.GuncellemeTarihi, u => DateTime.UtcNow));
```

```sql
UPDATE [u] SET [u].[Fiyat] = [u].[Fiyat] * 1.20,
               [u].[GuncellemeTarihi] = GETUTCDATE()
FROM [Urunler] AS [u] WHERE [u].[KategoriId] = 3;
```

İkisi de etkilenen satır sayısını döndürür.

### 8.3 Dikkat edilecekler

| Davranış | Sonucu |
|---|---|
| `SaveChanges` çağrılmaz | Metot **anında** çalışır; bekleyen diğer değişikliklerle aynı işlemde değildir |
| Change tracker güncellenmez | Bellekte duran nesneler eski değeri göstermeye devam eder |
| `SaveChanges` override'ı çalışmaz | Denetim alanları, soft delete kancaları devreye girmez |
| `SaveChanges` interceptor'ları çalışmaz | Aynı sebeple |
| Global query filter uygulanır | Sorgu bir LINQ sorgusudur; filtre `WHERE`'e eklenir |
| Cascade delete veritabanına bırakılır | FK `Restrict` ise hata alırsın |

```csharp
// TUZAK — bellekteki nesne eski değeri gösterir
var urun = await _db.Urunler.FirstAsync(u => u.Id == 5);
Console.WriteLine(urun.Fiyat);             // 100

await _db.Urunler.Where(u => u.Id == 5)
    .ExecuteUpdateAsync(s => s.SetProperty(u => u.Fiyat, 200m));

Console.WriteLine(urun.Fiyat);             // HÂLÂ 100 — takipçi haberdar değil

await _db.Entry(urun).ReloadAsync();
Console.WriteLine(urun.Fiyat);             // 200
```

Birkaç toplu işlemi atomik yapmak istiyorsan açık transaction kur:

```csharp
using var tx = await _db.Database.BeginTransactionAsync();

await _db.SiparisDetaylari
    .Where(d => d.Siparis.Tarih < esik)
    .ExecuteDeleteAsync();

await _db.Siparisler
    .Where(s => s.Tarih < esik)
    .ExecuteDeleteAsync();

await tx.CommitAsync();
```

> **Uyarı:** Soft delete kullanıyorsan `ExecuteDelete` kaydı **gerçekten** siler. `SaveChanges` override'ında yazdığın "silme yerine bayrak koy" mantığı devreye girmez. Soft delete için `ExecuteUpdate` ile bayrağı set et:

```csharp
await _db.Urunler
    .Where(u => u.KategoriId == 3)
    .ExecuteUpdateAsync(s => s.SetProperty(u => u.Silindi, true));
```

### 8.4 `AddRange` ve `SaveChanges` toplu davranışı

Toplu ekleme için döngüde `Add` yerine `AddRange` kullan:

```csharp
// YAVAŞ — her Add çağrısı değişiklik tespitini tetikleyebilir
foreach (var u in yeniUrunler) _db.Urunler.Add(u);

// HIZLI
_db.Urunler.AddRange(yeniUrunler);
await _db.SaveChangesAsync();
```

Çok büyük kümelerde değişiklik tespitini geçici olarak kapatabilirsin:

```csharp
_db.ChangeTracker.AutoDetectChangesEnabled = false;
try
{
    _db.Urunler.AddRange(yeniUrunler);
    await _db.SaveChangesAsync();
}
finally
{
    _db.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

`SaveChanges` zaten kendi içinde **toplu gönderim (batching)** yapar: komutları tek bir ağ turunda birleştirir. SQL Server sağlayıcısında varsayılan parti boyutu 42'dir ve ayarlanabilir:

```csharp
opt.UseSqlServer(connectionString, sql => sql.MaxBatchSize(100));
```

> **Uyarı:** On binlerce satırlık gerçek toplu ekleme için EF Core en iyi araç değildir. `SqlBulkCopy` ya da bunun üstüne kurulu bir kütüphane, EF Core'dan kat kat hızlıdır. EF Core'un yeri binlerce satıra kadardır.

> **Bu benzetme şurada bozulur:** Arşiv sorumlusuna yazı yazdığında en azından sana bir cevap gelir ve masandaki kopyaların artık geçersiz olduğunu bilirsin. `ExecuteUpdate` sana sadece satır sayısı döndürür; bellekteki nesnelerin bayatladığını hatırlaman gereken tek kişi sensin. Bu yüzden toplu işlemleri, takip edilen nesnelerin olmadığı ayrı bir akışta çalıştırmak en temizidir.

---

## 9. Sorguyu İnceleme: `ToQueryString`, Loglama, `TagWith`

> **Benzetme —** Arabada bir ses var. Tamirciye gidip "garip bir ses geliyor" dersin. İyi tamirci motoru dinlemez; cihazı takar, hata kodlarını okur ve "şu sensör" der. Kötüsü parça değiştirerek dener. Performansta da tahmin yürütmek, parça değiştirerek denemektir.

**Basitçe:** EF Core'un ne ürettiğini görmeden hiçbir şeyi düzeltemezsin. Üç ayrı araç var: sorgunun SQL'ini tek tek görmek, tüm sorguları loglamak, ve sorguya etiket koyup veritabanı tarafında tanımak.

### 9.1 `ToQueryString()`

EF Core 5'ten itibaren bir `IQueryable`'ın üreteceği SQL'i çalıştırmadan görebilirsin:

```csharp
var sorgu = _db.Siparisler
    .Where(s => s.Tarih >= DateTime.Today.AddDays(-7))
    .Include(s => s.Musteri)
    .OrderByDescending(s => s.Tarih);

string sql = sorgu.ToQueryString();
Console.WriteLine(sql);

var sonuc = await sorgu.ToListAsync();
```

Çıktı, parametre bildirimleriyle birlikte gelir ve doğrudan SSMS'e yapıştırıp plan inceleyebilirsin:

```sql
DECLARE @__p_0 datetime2(7) = '2026-09-16T00:00:00.0000000';

SELECT [s].[Id], [s].[Tarih], [s].[MusteriId], [s].[ToplamTutar],
       [m].[Id], [m].[Ad], [m].[Telefon]
FROM [Siparisler] AS [s]
INNER JOIN [Musteriler] AS [m] ON [s].[MusteriId] = [m].[Id]
WHERE [s].[Tarih] >= @__p_0
ORDER BY [s].[Tarih] DESC
```

`ToQueryString()` yalnızca `IQueryable` üzerinde çalışır. `ToList()` çağırdıktan sonra elinde liste vardır, sorgu değil.

### 9.2 Loglama

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseSqlServer(connectionString);

    if (builder.Environment.IsDevelopment())
    {
        opt.LogTo(Console.WriteLine, LogLevel.Information);
        opt.EnableSensitiveDataLogging();
        opt.EnableDetailedErrors();
    }
});
```

Sadece SQL komutlarını görmek istersen kategori filtresi verebilirsin:

```csharp
opt.LogTo(Console.WriteLine,
          new[] { DbLoggerCategory.Database.Command.Name },
          LogLevel.Information);
```

| Ayar | Ne yapar | Üretimde |
|---|---|---|
| `LogTo` | SQL ve olayları loglar | Dikkatli — `LogLevel.Warning` |
| `EnableSensitiveDataLogging()` | Parametre **değerlerini** loga yazar | **Asla açma** |
| `EnableDetailedErrors()` | Materialization hatalarında kolon adı verir | Kapalı tut |

> **Uyarı:** `EnableSensitiveDataLogging()` parametre değerlerini düz metin olarak loga yazar. TC kimlik numarası, e-posta, parola sıfırlama anahtarı — sorguya parametre olarak ne geçiyorsa log dosyasına düşer. Bu ayarı üretimde açmak, KVKK ihlalinin en kolay yoludur. `IsDevelopment()` kontrolünün içine almayı alışkanlık hâline getir.

N+1 avı tam olarak burada yapılır. Bir sayfayı yenile, konsola bak, aynı SQL'in kaç kez tekrarladığını say.

### 9.3 `TagWith`

Üretimde DBA sana "şu sorgu sunucuyu yoruyor" dediğinde, o SQL'in koddaki hangi satırdan geldiğini bulman gerekir. `TagWith` sorgunun başına SQL yorumu ekler:

```csharp
var liste = await _db.Siparisler
    .TagWith("SiparisController.Index - son 30 gun listesi")
    .Where(s => s.Tarih >= DateTime.Today.AddDays(-30))
    .ToListAsync();
```

```sql
-- SiparisController.Index - son 30 gun listesi

SELECT [s].[Id], [s].[Tarih], ... FROM [Siparisler] AS [s]
WHERE [s].[Tarih] >= @__p_0
```

Yorum, SQL Server'ın sorgu deposunda (Query Store) ve profiler çıktısında görünür. Ağır sorgulara etiket koymak, üretim teşhisini dakikalara indirir.

EF Core 6'dan itibaren dosya ve satır numarasını otomatik ekleyen bir varyant da var:

```csharp
var liste2 = await _db.Siparisler
    .TagWithCallSite()
    .ToListAsync();
```

### 9.4 Basit süre ölçümü

```csharp
var sw = Stopwatch.StartNew();
var sonuc = await _db.Siparisler.Include(s => s.Detaylar).ToListAsync();
sw.Stop();
_logger.LogInformation("Sorgu {Ms} ms, {Adet} kayit", sw.ElapsedMilliseconds, sonuc.Count);
```

İki farklı yazımı karşılaştırmak için bu kadarı çoğu zaman yeter. Kesin ölçüm için BenchmarkDotNet kullanılır ama bootcamp seviyesinde `Stopwatch` ve log sayısı yeterlidir.

> **Bu benzetme şurada bozulur:** Tamirci cihazı taktığında araba normal çalışmaya devam eder. `EnableSensitiveDataLogging()` ise ölçüm aracının kendisi bir risk kaynağıdır — açık unutulursa teşhis aracı güvenlik açığına dönüşür. Ölçüm aracını kurarken onu nasıl kapatacağını da planla.

---

## 10. Compiled Query ve DbContext Pooling

> **Benzetme —** Kahvehanede her müşteriye yeni bardak alınmaz; bardak yıkanır, tekrar kullanılır. Bardağı üretmek pahalıdır, yıkamak ucuzdur. DbContext pooling budur. Compiled query ise başka bir tasarruf: aynı dilekçeyi her gün tercümana götürüp yeniden çevirtmek yerine, çevrilmiş hâlini bir kere alıp saklamaktır.

**Basitçe:** İkisi de "aynı işi tekrar tekrar yapma" fikrinden çıkar. Sorgunun SQL'e çevrilmesini bir kez yapıp saklarsın; `DbContext` nesnesinin kurulumunu bir kez yapıp geri dönüştürürsün.

### 10.1 Sorgu önbelleği zaten var

EF Core, her LINQ sorgusunun çevrilmiş hâlini kendi içinde önbelleğe alır. Aynı sorgu ikinci kez çalıştığında çeviri tekrarlanmaz. Yani bu iş **zaten yapılıyor**.

Ama önbellekten faydalanmak için sorgu yapısının aynı kalması gerekir. Değerleri sabit yazarsan her değer için ayrı bir önbellek girdisi oluşur:

```csharp
// KÖTÜ — her id için ayrı sorgu ağacı, önbellek şişer
var sorgu = _db.Urunler.Where(u => u.Id == 5);

// İYİ — parametre olarak geçer, tek önbellek girdisi
int id = 5;
var sorgu2 = _db.Urunler.Where(u => u.Id == id);
```

Pratikte değişken kullandığın sürece bu doğru şekilde çalışır.

### 10.2 Compiled query

**Teknik olarak:** **Compiled query (derlenmiş sorgu)** — LINQ ifadesinin çevrilmiş hâlini bir temsilciye (delegate) bağlayıp önbellek arama adımını da atlayan yöntem.

```csharp
public static class Sorgular
{
    public static readonly Func<AppDbContext, int, Task<Siparis?>> SiparisGetir =
        EF.CompileAsyncQuery((AppDbContext db, int id) =>
            db.Siparisler
              .Include(s => s.Musteri)
              .FirstOrDefault(s => s.Id == id));

    public static readonly Func<AppDbContext, DateTime, IAsyncEnumerable<Siparis>> SonSiparisler =
        EF.CompileAsyncQuery((AppDbContext db, DateTime baslangic) =>
            db.Siparisler
              .AsNoTracking()
              .Where(s => s.Tarih >= baslangic)
              .OrderByDescending(s => s.Tarih));
}

// Kullanımı
var siparis = await Sorgular.SiparisGetir(_db, 42);

await foreach (var s in Sorgular.SonSiparisler(_db, DateTime.Today.AddDays(-7)))
{
    // ...
}
```

Senkron karşılığı `EF.CompileQuery`'dir.

| Artısı | Eksisi |
|---|---|
| Önbellek arama maliyeti kalkar | Sorgu `static readonly` alan olarak yazılır |
| Çok sık çağrılan sorgularda ölçülebilir kazanç | Dinamik sorgu kurulamaz (koşullu `Where` eklenemez) |
| | Kod okunurluğu düşer |
| | Çoğu uygulamada fark hissedilmez |

> **Uyarı:** Compiled query, listenin en sonundaki optimizasyondur. N+1 varken ya da tüm tabloyu çekerken compiled query yazmak, delik teknede kürek hızını artırmaya benzer. Önce 4, 5 ve 6. bölümdeki sorunları çöz.

### 10.3 DbContext pooling

`DbContext` her istekte yeniden kurulur. Kurulum ucuz değildir: iç servisler, değişiklik takipçisi, sorgu önbelleği referansları hazırlanır. Pooling, nesneyi yok etmek yerine sıfırlayıp havuza geri koyar.

```csharp
// Normal
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString));

// Havuzlu
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseSqlServer(connectionString), poolSize: 128);
```

Havuza dönen context'in değişiklik takipçisi temizlenir, ama **senin eklediğin alanlar temizlenmez**. Bu yüzden kısıtları vardır:

| Kısıt | Açıklama |
|---|---|
| Kurucuda yalnızca `DbContextOptions` | Başka servis enjekte edilemez |
| Kendi alanların sıfırlanmaz | `_firmaId` gibi durum tutarsan bir sonraki isteğe sızar |
| `OnConfiguring` içinde durum kurma | Aynı sebeple riskli |

```csharp
// POOLING İLE ÇALIŞMAZ — kurucuda ek servis var
public class AppDbContext : DbContext
{
    private readonly ICurrentUser _user;
    public AppDbContext(DbContextOptions<AppDbContext> opt, ICurrentUser user) : base(opt)
        => _user = user;
}
```

Multi-tenant global query filter kullanan bir uygulamada pooling'i doğrudan kullanamazsın. `IDbContextFactory` ile ayrı bir kurgu gerekir.

> **Uyarı:** Pooling'in kazancı yüksek trafikli uygulamalarda anlamlıdır. Günde birkaç yüz istek alan bir siteye pooling eklemek ölçülebilir bir fark yaratmaz, ama yukarıdaki kısıtları getirir. Gerekmedikçe `AddDbContext` ile kal.

> **Bu benzetme şurada bozulur:** Kahvehanede bardağı yıkayan kişi içinde şeker kalıp kalmadığına bakar. Havuzdaki `DbContext` ise sadece EF Core'un kendi durumunu temizler; senin sınıfa eklediğin bir alan olduğu gibi bir sonraki isteğe geçer. "Temizlendi" sandığın şey aslında yarı temizlenmiştir ve kalan yarısı en kötü hata türünü üretir: bir kullanıcının verisinin başka bir kullanıcıya görünmesi.

---

## 11. Sayfalama, Keyset Pagination ve Ham SQL

> **Benzetme —** Kütüphaneciden "51'inci kitaptan 60'ıncıya kadar olanları ver" diye istiyorsun. Bu isteğin anlamlı olması için rafın **belirli bir sıraya göre dizili** olması şart. Raf rastgele diziliyse, her gelişinde farklı on kitap alırsın; hatta bazı kitapları iki kez görür, bazılarını hiç görmezsin.

**Basitçe:** Sayfalama yapmak için `Skip` ve `Take` yeterli değildir — mutlaka `OrderBy` de gerekir. Ve derin sayfalarda `Skip` yavaşlar; o noktada farklı bir yönteme geçilir.

### 11.1 `Skip` / `Take` ve `OrderBy` şartı

```csharp
// YANLIŞ — sıra garanti değil, aynı kayıt iki sayfada çıkabilir
var sayfa = await _db.Siparisler
    .Skip(50).Take(10)
    .ToListAsync();

// DOĞRU
var sayfa2 = await _db.Siparisler
    .OrderByDescending(s => s.Tarih)
    .ThenBy(s => s.Id)                      // benzersiz alan — eşitlikte belirleyici
    .Skip(50).Take(10)
    .ToListAsync();
```

`ThenBy(s => s.Id)` satırı önemlidir. Aynı tarihe sahip iki sipariş varsa, aralarındaki sıra belirsiz kalır ve sayfa sınırında biri kaybolabilir. Sıralamanın son kriteri her zaman benzersiz bir alan olmalıdır.

Tam sayfalama kalıbı:

```csharp
public async Task<PagedResult<SiparisListeDto>> ListeleAsync(int sayfa, int boyut)
{
    var sorgu = _db.Siparisler
        .AsNoTracking()
        .Where(s => !s.Silindi);

    int toplam = await sorgu.CountAsync();          // 1. sorgu

    var kayitlar = await sorgu
        .OrderByDescending(s => s.Tarih).ThenBy(s => s.Id)
        .Skip((sayfa - 1) * boyut)
        .Take(boyut)
        .Select(s => new SiparisListeDto
        {
            Id = s.Id,
            Tarih = s.Tarih,
            MusteriAdi = s.Musteri.Ad,
            Tutar = s.ToplamTutar
        })
        .ToListAsync();                             // 2. sorgu

    return new PagedResult<SiparisListeDto>(kayitlar, toplam, sayfa, boyut);
}
```

### 11.2 Keyset pagination

`Skip(100000)` yazdığında veritabanı ilk yüz bin satırı **yine de üretir**, sonra atar. Derin sayfalarda maliyet doğrusal artar.

**Keyset pagination (anahtar tabanlı sayfalama)** — "kaç satır atla" yerine "son gördüğüm anahtardan sonrası" mantığıyla çalışır.

```csharp
// İlk sayfa
var ilk = await _db.Siparisler
    .OrderByDescending(s => s.Tarih).ThenByDescending(s => s.Id)
    .Take(20)
    .ToListAsync();

var sonTarih = ilk[^1].Tarih;
var sonId = ilk[^1].Id;

// Sonraki sayfa — Skip yok
var sonraki = await _db.Siparisler
    .Where(s => s.Tarih < sonTarih || (s.Tarih == sonTarih && s.Id < sonId))
    .OrderByDescending(s => s.Tarih).ThenByDescending(s => s.Id)
    .Take(20)
    .ToListAsync();
```

`(Tarih, Id)` üzerinde bir index varsa, veritabanı doğrudan doğru yere atlar. Sayfa numarası arttıkça maliyet artmaz.

| | `Skip`/`Take` | Keyset |
|---|---|---|
| Sayfa numarasına atlama | Var | Yok — sadece ileri/geri |
| Derin sayfa maliyeti | Doğrusal artar | Sabit |
| Toplam sayfa sayısı | Kolay | Ayrı `Count` gerekir |
| Araya kayıt eklenmesi | Kayma olur | Kayma olmaz |
| Uygun olduğu yer | Admin tabloları, az sayfa | Sonsuz kaydırma, API, log listesi |

### 11.3 Ham SQL'e düşmek

EF Core her sorguyu iyi çeviremez. Pencere fonksiyonları, `PIVOT`, karmaşık CTE'ler, hint gerektiren sorgular — bunlarda LINQ ya çeviremez ya da kötü SQL üretir.

```csharp
// Varlık döndüren ham SQL — EF Core 7+ sözdizimi
var urunler = await _db.Urunler
    .FromSql($"SELECT * FROM Urunler WHERE KategoriId = {kategoriId}")
    .AsNoTracking()
    .ToListAsync();
```

`FromSql` **enterpolasyonlu** string alır ve değerleri otomatik olarak parametreye çevirir — SQL injection'a karşı güvenlidir. String birleştirmeyle sorgu kurma:

```csharp
// TEHLİKELİ — SQL injection
var kotu = await _db.Urunler
    .FromSqlRaw("SELECT * FROM Urunler WHERE Ad = '" + arananAd + "'")
    .ToListAsync();

// GÜVENLİ
var iyi = await _db.Urunler
    .FromSqlRaw("SELECT * FROM Urunler WHERE Ad = {0}", arananAd)
    .ToListAsync();
```

Ham sorgunun üstüne LINQ eklemeye devam edebilirsin:

```csharp
var sonuc = await _db.Urunler
    .FromSql($"SELECT * FROM Urunler WHERE KategoriId = {kategoriId}")
    .Where(u => u.Fiyat > 100)              // dış sorgu olarak sarılır
    .OrderBy(u => u.Ad)
    .Take(20)
    .ToListAsync();
```

Varlık olmayan tipler için EF Core 7+ `SqlQuery` sunar:

```csharp
var toplamlar = await _db.Database
    .SqlQuery<decimal>($"SELECT SUM(ToplamTutar) FROM Siparisler WHERE MusteriId = {id}")
    .ToListAsync();
```

Hiç sonuç döndürmeyen komutlar için:

```csharp
int etkilenen = await _db.Database
    .ExecuteSqlAsync($"UPDATE Urunler SET Fiyat = Fiyat * 1.1 WHERE KategoriId = {kategoriId}");
```

| `FromSql` kuralı | Açıklama |
|---|---|
| Varlığın **tüm** kolonları dönmeli | Eksik kolon materialization hatası verir |
| Kolon adları eşleşmeli | `AS` ile yeniden adlandırabilirsin |
| `Include` kullanılabilir | Ama sorgu birleştirilebilir (composable) olmalı |
| Saklı yordamda LINQ eklenemez | `EXEC` sonrası `Where` yazamazsın |
| Parametreler enterpolasyonla geçmeli | String birleştirme = injection |

> **Uyarı:** Ham SQL'e geçmek yenilgi değildir, ama ilk çözüm de değildir. Önce projeksiyonla, `AsSplitQuery` ile, index ekleyerek dene. Ham SQL'in bedeli, şema değiştiğinde derleyicinin seni uyarmamasıdır — kolon adı değişir, kod derlenir, çalışma anında patlar.

### 11.4 `GenericRepository.List()`'in düzeltilmiş hâli

Notun başındaki metoda geri dönelim:

```csharp
// MvcCv'deki hâli
public List<T> List() => _table.ToList();
```

Aynı fikri koruyan ama tuzaklardan arınmış hâli:

```csharp
public async Task<List<T>> ListAsync(
    Expression<Func<T, bool>>? filtre = null,
    Func<IQueryable<T>, IOrderedQueryable<T>>? siralama = null,
    int? sayfa = null,
    int boyut = 50,
    bool takipEt = false)
{
    IQueryable<T> sorgu = _table;

    if (!takipEt) sorgu = sorgu.AsNoTracking();
    if (filtre is not null) sorgu = sorgu.Where(filtre);
    sorgu = siralama is not null ? siralama(sorgu) : sorgu;

    if (sayfa is not null)
        sorgu = sorgu.Skip((sayfa.Value - 1) * boyut).Take(boyut);

    return await sorgu.ToListAsync();
}
```

| Eklenen | Çözdüğü sorun |
|---|---|
| `AsNoTracking()` varsayılan | Gereksiz takip maliyeti |
| `filtre` parametresi | Tüm tablonun çekilmesi |
| `siralama` parametresi | Garanti olmayan satır sırası |
| `sayfa` / `boyut` | Bellekte satır patlaması |
| `async` | Thread'in beklemede bloklanması |

Yine de bu imza projeksiyon yapmaz — varlığın tüm kolonları gelir. Listeleme ekranları için doğru cevap, generic repository değil, o ekrana özel bir `Select` yazan sorgu metodudur.

> **Bu benzetme şurada bozulur:** Kütüphanede raf sabittir; sen kitapları sayarken kimse araya kitap sokmaz. Veritabanında ise sen 3. sayfadayken başka biri kayıt ekler. `Skip(20).Take(10)` ile ikinci sayfada gördüğün kaydı üçüncü sayfada bir daha görürsün. Keyset pagination'ın asıl üstünlüğü hız değil, bu kaymanın hiç olmamasıdır.

---

## Performans Kontrol Listesi

Bir listeleme ya da detay ekranı yazdıktan sonra bu tabloyu sırayla geç.

| # | Kontrol | Nasıl bakılır | Çözüm |
|---|---|---|---|
| 1 | Sorgu SQL'ini gördün mü | `ToQueryString()` / `LogTo` | Görmeden düzeltme |
| 2 | Kaç sorgu üretiliyor | Log sayımı | Tekrarlayan SQL varsa N+1 |
| 3 | `WHERE` var mı | Üretilen SQL | Filtre ekle |
| 4 | Sayfalama var mı | `Skip`/`Take` | Ekle, `OrderBy` ile birlikte |
| 5 | `OrderBy`ın son kriteri benzersiz mi | Sorgu kodu | `ThenBy(x => x.Id)` |
| 6 | Kolonların hepsi lazım mı | `SELECT` listesi | `Select` ile projeksiyon |
| 7 | Takibe ihtiyaç var mı | Kayıt güncellenecek mi | `AsNoTracking()` |
| 8 | Döngü içinde navigation okuyor musun | Kod okuması | `Include` veya projeksiyon |
| 9 | Birden çok koleksiyon `Include` ediyor musun | Sorgu kodu | `AsSplitQuery()` veya projeksiyon |
| 10 | Lazy loading açık mı | `UseLazyLoadingProxies` | Kapat |
| 11 | Toplu silme/güncelleme döngüyle mi | Kod okuması | `ExecuteDelete` / `ExecuteUpdate` |
| 12 | `AsEnumerable()` çok erken mi | Sorgu kodu | Sona kaydır |
| 13 | `WHERE` kolonlarında index var mı | Execution plan | Index ekle |
| 14 | Ağır sorgu etiketli mi | `TagWith` | Ekle |
| 15 | `EnableSensitiveDataLogging` kapalı mı | `Program.cs` | Üretimde kapat |

---

## Tek Bakışta Özet

- EF Core'da yavaş kod hata vermez; sorunu ancak veri büyüyünce görürsün.
- Performans üç soruya iner: kaç kere gidiyorsun, ne getiriyorsun, getirdiğini takip ediyor musun.
- Ölçmeden optimize etme: önce `ToQueryString()` ve `LogTo` ile üretilen SQL'i gör.
- Okuma amaçlı her sorguda `AsNoTracking()` — güncelleme yapacaksan asla.
- `AsNoTracking()` ile aynı kayıt birden çok nesneye dönüşür; gerekirse `AsNoTrackingWithIdentityResolution()`.
- Lazy loading varsayılan olarak kapalıdır ve öyle kalmalı; sorgu üreten satır kodda görünmez.
- N+1, döngü içinde navigation okumaktan doğar; çözümü `Include`, projeksiyon veya sözlüğe ön yükleme.
- Birden çok koleksiyon `Include` etmek satır sayısını çarpar; `AsSplitQuery()` böler, bedeli çoklu ağ turudur.
- Projeksiyon en ucuz kazançtır: az kolon, otomatik no-tracking, N+1'siz tek sorgu.
- EF Core 3+ çeviremediği `Where` ifadesinde hata verir; `AsEnumerable()` ile sınırı **en sona** koyarsın.
- `ExecuteUpdate` / `ExecuteDelete` (EF Core 7+) change tracker'ı, `SaveChanges` override'ını ve interceptor'ları atlar.
- `SaveChanges` zaten toplu gönderim yapar; on binlerce satır için EF Core yerine `SqlBulkCopy` düşün.
- `EnableSensitiveDataLogging()` parametre değerlerini loga yazar — üretimde asla açma.
- `TagWith` ile etiketlenen sorguyu üretim loglarında saniyeler içinde bulursun.
- Compiled query ve pooling listenin en sonundaki optimizasyonlardır; önce N+1 ve projeksiyonu hallet.
- Sayfalamada `OrderBy` şarttır ve son kriteri benzersiz olmalıdır; derin sayfalarda keyset pagination'a geç.
- Ham SQL bir kaçış kapısıdır; bedeli şema değişiminde derleyicinin sessiz kalmasıdır.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Change tracker | Takip edilen varlıkların orijinal değerlerini tutan bileşen |
| Tracking / no-tracking | Sorgu sonucunun takibe alınıp alınmaması |
| Identity resolution | Aynı anahtarlı satırların tek nesneye eşlenmesi |
| Materialization | Gelen satırların C# nesnesine dönüştürülmesi |
| Eager loading | İlişkili veriyi ana sorguyla birlikte getirme (`Include`) |
| Explicit loading | İlişkili veriyi sonradan, elle yükleme (`Load`) |
| Lazy loading | Navigation'a dokunulduğunda otomatik yükleme |
| Proxy | Lazy loading için EF Core'un türettiği ara sınıf |
| N+1 problemi | Bir ana sorgu artı satır başına bir ek sorgu üreten kalıp |
| Cartesian explosion | Çoklu koleksiyon `JOIN`'inde satır sayısının çarpılarak artması |
| Split query | Her koleksiyonu ayrı sorguda getiren çalışma biçimi |
| Projection | `Select` ile sonucu başka bir şekle dönüştürme |
| DTO | Katmanlar arasında veri taşımak için yazılan sade sınıf |
| Client-side evaluation | Sorgunun bir kısmının veritabanı yerine bellekte çalışması |
| Round trip (ağ turu) | Veritabanına gidip gelen her bir sorgu |
| Batching | Birden çok komutun tek ağ turunda gönderilmesi |
| `ExecuteUpdate` / `ExecuteDelete` | Belleğe çekmeden doğrudan `UPDATE`/`DELETE` çalıştıran metotlar |
| Compiled query | Çevrilmiş sorgunun bir temsilciye bağlanıp saklanması |
| DbContext pooling | `DbContext` nesnelerinin sıfırlanıp yeniden kullanılması |
| Keyset pagination | `Skip` yerine son görülen anahtardan devam eden sayfalama |
| `FromSql` | Ham SQL ile varlık döndüren sorgu metodu |
| Query Store | SQL Server'ın sorgu planlarını ve istatistiklerini tuttuğu yapı |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`AsNoTracking()` her yere serpiştirilir, zararı yok" | Güncelleme yapan sorguda `SaveChanges` sessizce hiçbir şey yapmaz |
| "Tracking sadece bellek tüketir" | `DetectChanges` taraması CPU maliyeti de üretir |
| "`Include` her zaman doğru çözümdür" | Sadece birkaç alan lazımsa projeksiyon daha ucuzdur |
| "Lazy loading kolaylık sağlar" | Sorgu üreten satırı kodda görünmez kılar — N+1'in ana kaynağıdır |
| "N+1 sadece lazy loading ile olur" | Explicit loading ve repository çağrıları da aynı kalıbı üretir |
| "Tek sorgu her zaman iyidir" | Çoklu koleksiyon `Include`'unda satır sayısı çarpılır |
| "`AsSplitQuery()` her zaman hızlandırır" | Küçük veride çoklu ağ turu daha yavaştır |
| "Projeksiyon için `AsNoTracking()` de yazmalıyım" | DTO zaten takip edilmez |
| "`Select` içinde varlık taşırsam takip edilmez" | Edilir — anonim tipin içinde olması fark etmez |
| "Client evaluation EF Core'da artık yok" | Üst seviye `Select` içinde hâlâ var; kalkan `Where` içindeki |
| "`AsEnumerable()` zararsızdır" | Erken yazılırsa tüm tabloyu belleğe çeker |
| "`ExecuteDelete` soft delete mantığımı çalıştırır" | `SaveChanges` override'ı devreye girmez; kayıt gerçekten silinir |
| "`ExecuteUpdate` sonrası bellekteki nesne güncellenir" | Güncellenmez; `Reload()` gerekir |
| "`SaveChanges` her kayıt için ayrı gidiş yapar" | Komutları partiler hâlinde birleştirir |
| "Compiled query ciddi hızlanma getirir" | Çoğu uygulamada fark hissedilmez; önce N+1'i çöz |
| "`Skip`/`Take` sayfalama için yeterlidir" | `OrderBy` olmadan sıra garanti değildir |
| "Derin sayfada `Skip` maliyeti sabittir" | Atlanan satırlar yine de üretilir; doğrusal artar |
| "`FromSqlRaw` ile string birleştirmek sorun değil" | SQL injection açığıdır; enterpolasyon veya parametre kullan |

---

## Sonraki

→ `07-Hafta-Ozeti.md` (Pazar)
