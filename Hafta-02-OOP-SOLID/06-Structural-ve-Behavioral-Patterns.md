# Hafta 2 · Cumartesi — Yapısal ve Davranışsal Kalıplar

**Okuma süresi:** ~50 dk
**Neden bu konu:** ASP.NET Core'un middleware pipeline'ı bir Chain of Responsibility, `IEnumerable` bir Iterator, DI ile kaydettiğin her decorator bir Decorator'dır. Bu kalıpları bilmek framework'ü ezberlemek yerine anlamanı sağlar. Bootcamp'te "burada bir strategy kullanalım" dendiğinde ne istendiğini anlaman, kodu okumandan önce gelir.

---

## Önce Basitçe

Dünkü not nesnelerin nasıl doğduğuyla ilgiliydi. Bu not doğduktan sonrasıyla ilgili: nesneler birbirine nasıl bağlanacak ve birbiriyle nasıl konuşacak.

Yapısal kalıplar bağlantıyla ilgilenir. Elinde bir nesne vardır ama arayüzü sana uymaz; arada bir çevirici gerekir. Ya da mevcut bir sınıfa dokunmadan üstüne yeni bir davranış eklemen gerekir. Ya da arkada on tane sınıf vardır ve kullanan kişinin onuyla da uğraşması saçmadır; tek bir kapı açarsın. Bunların hepsi "parçaları nasıl bir araya getiririm" sorusunun cevaplarıdır.

Davranışsal kalıplar ise konuşmayla ilgilenir. Bir iş için beş farklı yol varsa, hangisinin seçileceğini kim bilecek? Bir şey değiştiğinde onunla ilgilenen üç ayrı yerin haberi nasıl olacak? Bir işlemi yapıp geri alabilmek için o işlemi nasıl saklamak gerekir? Bir isteğin birkaç durakta sırayla işlenmesi gerekiyorsa o zinciri nasıl kurarsın? Bunlar akış ve sorumluluk soruları.

Bu iki ailenin pratikte en çok göreceğin tarafı, .NET'in içinde zaten kullanılıyor olmaları. Middleware yazarken Chain of Responsibility kullanırsın, `foreach` yazarken Iterator, `ILogger` enjekte ederken çoğu zaman bir Decorator zinciri. Kalıpları öğrenmek yeni bir şey öğrenmek değil; her gün kullandığın şeylerin adını öğrenmek. Adını bilince ne zaman kendin yazman gerektiğini de bilirsin. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bir hastanenin işleyişini düşün. Yapısal kalıplar **binanın planıdır**: hangi kapı nereye açılır, danışma nerede durur, hangi bölüm hangisinin önünde yer alır. Davranışsal kalıplar ise **işleyiş yönergesidir**: hasta hangi sırayla hangi duraktan geçer, sonuç çıkınca kime haber verilir, aynı şikâyet için hangi protokol uygulanır. Bina doğru kurulsa bile yönerge yanlışsa hastane çalışmaz; ikisi ayrı işlerdir ve ikisi de gerekir.

---

## Bu Notta Ne Var

1. Adapter — uyumsuz arayüzü uydurmak
2. Decorator — dokunmadan davranış eklemek
3. Facade — karmaşık alt sistemi tek kapıdan sunmak
4. Proxy — araya girip erişimi yönetmek
5. Composite — parça ile bütünü aynı görmek
6. Strategy — algoritmayı dışarıdan vermek
7. Observer — değişikliği ilgilenenlere duyurmak
8. Command — işlemi nesneleştirmek
9. Template Method — iskeleti sabitleyip adımları değiştirmek
10. Chain of Responsibility — zincirde sırayla işlemek
11. Iterator — gezinmeyi koleksiyondan ayırmak
12. Kalıp seçimi ve pattern hastalığı

---

## 1. Adapter — Uyumsuz Arayüzü Uydurmak

> **Benzetme —** Yurt dışından bir cihaz getirdin, fişi üç bacaklı ve İngiliz tipi. Duvardaki prize girmiyor. Cihazı da değiştiremezsin, prizi de sökemezsin. Araya bir priz adaptörü koyarsın: bir tarafı İngiliz fişini kabul eder, öbür tarafı Türk prizine girer. Adaptör elektriği değiştirmez, sadece **şekli** çevirir.

**Basitçe:** Adapter, elindeki sınıfın arayüzünü, senin kodunun beklediği arayüze çevirir. İki tarafa da dokunmadan aralarına girer.

**Teknik olarak:** **Adapter (uyarlayıcı)** — Bir sınıfın arayüzünü, istemcinin beklediği başka bir arayüze dönüştüren yapısal kalıp. Uyumsuzluk yüzünden birlikte çalışamayan sınıfların birlikte çalışmasını sağlar.

### Üçüncü parti kütüphane sarmalama

Bir SMS sağlayıcısının kütüphanesini projene ekledin. Arayüzü şöyle:

```csharp
// Üçüncü parti — sen yazmadın, değiştiremezsin
public class NetGsmClient
{
    public NetGsmResult SendMessage(string phoneNumber, string text, string header)
    {
        // ...
        return new NetGsmResult { Code = "00", Description = "OK" };
    }
}
```

Senin uygulamanın beklediği arayüz ise şu:

```csharp
public interface IBildirimGonderici
{
    Task<bool> GonderAsync(string hedef, string mesaj);
}
```

İkisi uyuşmuyor: biri senkron, diğeri asenkron; biri `bool`, diğeri `NetGsmResult` döndürüyor; birinin fazladan `header` parametresi var. Adapter araya girer:

```csharp
public class NetGsmAdapter : IBildirimGonderici
{
    private readonly NetGsmClient _client;
    private readonly string _baslik;

    public NetGsmAdapter(NetGsmClient client, IConfiguration config)
    {
        _client = client;
        _baslik = config["NetGsm:Baslik"];
    }

    public Task<bool> GonderAsync(string hedef, string mesaj)
    {
        var sonuc = _client.SendMessage(NumarayiDuzelt(hedef), mesaj, _baslik);
        return Task.FromResult(sonuc.Code == "00");
    }

    private static string NumarayiDuzelt(string numara)
        => numara.StartsWith("0") ? "9" + numara : numara;
}
```

Uygulaman artık `NetGsmClient` adını hiç görmez. Kayıt tek satırdır:

```csharp
builder.Services.AddScoped<IBildirimGonderici, NetGsmAdapter>();
```

Bu, SOLID'in **Dependency Inversion** ilkesinin pratikteki adıdır: kodun üçüncü parti tipe değil, kendi arayüzüne bağımlı olur.

### Ne zaman KULLANMA

- Arayüzü sen kontrol ediyorsan. Kendi sınıfının arayüzünü düzelt, araya adapter koyma.
- Uyarlamanın yanına iş mantığı sıkıştırmak istiyorsan. Adapter sadece çevirir; doğrulama, yeniden deneme, loglama girerse orası artık adapter değil, karışık bir sınıftır.
- Tek kullanımlık bir çağrı içinse. Üç satırlık dönüşüm için sınıf açmak fazlalıktır.

**Bu benzetme şurada bozulur:** Priz adaptörü elektriği değiştirmez; voltaj aynı kalır. Yazılımda ise adapter bazen **anlam** da çevirmek zorunda kalır. `NetGsmResult.Code == "00"` ifadesini `true`'ya çevirirken aslında bir yorum yapıyorsun. Sağlayıcı yarın `"0"` dönmeye başlarsa adapter sessizce yanlış cevap verir. Adaptörün ince yeri, şekli değil anlamı çevirdiği yerlerdir.

---

## 2. Decorator — Dokunmadan Davranış Eklemek

> **Benzetme —** Kışın dışarı çıkarken üstüne kazak, onun üstüne mont, onun üstüne yağmurluk giyersin. Her katman bir işe yarar: kazak ısıtır, mont rüzgâr tutar, yağmurluk su geçirmez. Hiçbiri diğerini değiştirmez, üstüne geçer. İstediğini çıkarır, sırasını değiştirirsin. Dışarıdan bakan hâlâ "giyinmiş bir insan" görür.

**Basitçe:** Decorator, mevcut bir sınıfın koduna dokunmadan onun etrafını sarar ve davranış ekler. Sarılan nesne de saran nesne de aynı arayüzü uyguladığı için kullanan taraf farkı görmez.

**Teknik olarak:** **Decorator (dekoratör)** — Bir nesneye, aynı arayüzü uygulayan başka bir nesne içine alınarak, çalışma anında sorumluluk ekleyen yapısal kalıp. Kalıtıma esnek bir alternatiftir.

Bir repository'ye kalıtımla önbellek eklersen, sonra loglama da isteyince `OnbellekliLoglayanRepository` yazman gerekir; üçüncü özellikte sınıf sayısı patlar. Decorator bunu katmanlara böler.

```csharp
public interface IBasvuruRepository
{
    Task<Basvuru> GetirAsync(int id);
}

// Çekirdek uygulama — asıl işi yapar
public class BasvuruRepository : IBasvuruRepository
{
    private readonly AppDbContext _db;
    public BasvuruRepository(AppDbContext db) => _db = db;

    public Task<Basvuru> GetirAsync(int id)
        => _db.Basvurular.FirstOrDefaultAsync(b => b.Id == id);
}
```

Loglama katmanı:

```csharp
public class LoglayanBasvuruRepository : IBasvuruRepository
{
    private readonly IBasvuruRepository _ic;           // sarılan nesne
    private readonly ILogger<LoglayanBasvuruRepository> _logger;

    public LoglayanBasvuruRepository(IBasvuruRepository ic,
                                     ILogger<LoglayanBasvuruRepository> logger)
        => (_ic, _logger) = (ic, logger);

    public async Task<Basvuru> GetirAsync(int id)
    {
        var sayac = Stopwatch.StartNew();
        var sonuc = await _ic.GetirAsync(id);          // işi içeridekine devret
        _logger.LogInformation("Basvuru {Id} {Ms} ms sürdü", id, sayac.ElapsedMilliseconds);
        return sonuc;
    }
}
```

Önbellek katmanı:

```csharp
public class OnbellekliBasvuruRepository : IBasvuruRepository
{
    private readonly IBasvuruRepository _ic;
    private readonly IMemoryCache _cache;

    public OnbellekliBasvuruRepository(IBasvuruRepository ic, IMemoryCache cache)
        => (_ic, _cache) = (ic, cache);

    public async Task<Basvuru> GetirAsync(int id)
    {
        if (_cache.TryGetValue($"basvuru:{id}", out Basvuru mevcut))
            return mevcut;                              // içerideki hiç çağrılmaz

        var sonuc = await _ic.GetirAsync(id);
        _cache.Set($"basvuru:{id}", sonuc, TimeSpan.FromMinutes(5));
        return sonuc;
    }
}
```

### DI ile decorator kaydı

.NET'in yerleşik konteynerinde decorator kaydı için hazır bir metot yoktur; fabrika delegesiyle elle kurarsın:

```csharp
// Program.cs
builder.Services.AddScoped<BasvuruRepository>();          // çekirdek, somut tip

builder.Services.AddScoped<IBasvuruRepository>(sp =>
{
    IBasvuruRepository katman = sp.GetRequiredService<BasvuruRepository>();
    katman = new LoglayanBasvuruRepository(
        katman, sp.GetRequiredService<ILogger<LoglayanBasvuruRepository>>());
    katman = new OnbellekliBasvuruRepository(
        katman, sp.GetRequiredService<IMemoryCache>());
    return katman;
});
```

Kayıt sırası çalışma sırasını belirler. Yukarıdaki kurulumda en dışta önbellek vardır: önbellekte veri varsa loglama **hiç çalışmaz**. Logları her çağrıda görmek istiyorsan loglamayı en dışa alırsın. Sıra bir ayrıntı değil, tasarım kararıdır.

`Scrutor` paketi bu kurulumu tek satıra indirir:

```csharp
builder.Services.AddScoped<IBasvuruRepository, BasvuruRepository>();
builder.Services.Decorate<IBasvuruRepository, LoglayanBasvuruRepository>();
builder.Services.Decorate<IBasvuruRepository, OnbellekliBasvuruRepository>();
```

> **Dikkat:** Decorator ile Proxy kodu neredeyse aynı görünür; ikisi de aynı arayüzü uygular ve içeriye devreder. Fark **niyettedir**. Decorator davranışı *zenginleştirir* (log ekler, önbellek ekler). Proxy erişimi *yönetir* (izin verir, geciktirir, uzağa bağlanır). Aynı kodu iki farklı sebeple yazabilirsin; adı niyetine göre koyarsın.

### Ne zaman KULLANMA

- Tek bir ek davranış varsa ve başkası gelmeyecekse. Doğrudan sınıfın içine yaz.
- Katman sayısı arttıkça hata ayıklama zorlaşır. Beş katmanlı bir zincirde hatanın hangi katmandan geldiğini bulmak yorucudur; üç-dört katmanı aşma.
- Katmanlar birbirinin iç davranışına bağımlıysa. Decorator'ın koşulu, katmanların **birbirinden habersiz** olmasıdır; habersizlik bozulduysa kalıp yanlış seçilmiştir.

**Bu benzetme şurada bozulur:** Mont giymek kazağı görünmez yapar ama işlevini sürdürür. Önbellek katmanında ise dıştaki katman içerideki katmanı **hiç çağırmayabilir**. Yani bu montu giydiğinde kazak bazen hiç ısıtmaz. Decorator'ın devretmeme hakkı vardır ve bu, giysi benzetmesinin karşılamadığı en önemli davranıştır.

---

## 3. Facade — Karmaşık Alt Sistemi Tek Kapıdan Sunmak

> **Benzetme —** Tapu işlemi için gittiğinde arkada emlak vergisi, belediye harcı, döner sermaye, kadastro kaydı ve imza işlemleri vardır. Sen bunların hepsini ayrı ayrı gezmezsin; danışmadaki tek bir memura gidersin, o seni doğru sırayla yönlendirir ve evrakı toplar. Danışma yeni bir daire kurmaz, var olanları senin adına koordine eder. Facade tam olarak o danışmadır.

**Basitçe:** Facade, birbirine bağlı birkaç sınıfın önüne tek ve basit bir arayüz koyar. Kullanan taraf arkadaki karmaşayı görmez.

**Teknik olarak:** **Facade (cephe)** — Bir alt sistemdeki arayüz kümesine, kullanımı kolaylaştıran birleşik bir arayüz sağlayan yapısal kalıp.

### Problem: aynı akış her yerde tekrar ediyor

Bir başvuru alma akışı dosya kaydetme, veritabanına yazma, e-posta ve bildirim adımlarından oluşuyorsa, bu sıralamayı çağıran her yerde (web formu, mobil API, toplu içe aktarma) tekrar yazarsın. Bir adım unutulursa kimse fark etmez.

### Çözüm: tek kapı

```csharp
public interface IBasvuruFacade
{
    Task<BasvuruSonucu> BasvuruAlAsync(BasvuruGirdi girdi);
}

public class BasvuruFacade : IBasvuruFacade
{
    private readonly IDosyaServisi _dosya;
    private readonly IEpostaServisi _eposta;
    private readonly IBildirimServisi _bildirim;
    private readonly AppDbContext _db;

    public BasvuruFacade(IDosyaServisi dosya, IEpostaServisi eposta,
                         IBildirimServisi bildirim, AppDbContext db)
        => (_dosya, _eposta, _bildirim, _db) = (dosya, eposta, bildirim, db);

    public async Task<BasvuruSonucu> BasvuruAlAsync(BasvuruGirdi girdi)
    {
        var yol = await _dosya.KaydetAsync(girdi.Ozgecmis);
        var basvuru = new Basvuru { AdSoyad = girdi.AdSoyad, OzgecmisYolu = yol };

        _db.Basvurular.Add(basvuru);
        await _db.SaveChangesAsync();

        await _eposta.GonderAsync(girdi.Eposta, "Başvurunuz alındı");
        await _bildirim.YoneticiyeHaberVerAsync(basvuru.Id);

        return new BasvuruSonucu(basvuru.Id, true);
    }
}
```

### Unit of Work'ün facade yönü

MvcCv'deki `GenericRepository` yapısının üstünde bir **Unit of Work** varsa, o sınıf iki iş birden yapar. Birincisi kendi kalıbıdır: birden çok repository'nin değişikliklerini tek transaction'da kaydeder. İkincisi ise facade yönüdür: çağıran koda tek bir kapı açar.

```csharp
public interface IUnitOfWork : IDisposable
{
    IGenericRepository<Basvuru> Basvurular { get; }
    IGenericRepository<Ilan> Ilanlar { get; }
    Task<int> KaydetAsync();
}
```

Kullanan kod `AppDbContext`'i, `DbSet`'leri, `SaveChanges` çağrısını ve transaction yönetimini görmez. Tek bir nesne üzerinden iş yapar. EF Core'un `DbContext`'i zaten hem Unit of Work hem Repository olduğu için bu katman çoğu projede tartışmalıdır; ama facade tarafı, ekip içinde ortak bir giriş noktası sağlamak açısından savunulabilir.

> **Uyarı:** Facade bir **delege eden** sınıftır. İş mantığı yazmaya başladığın anda facade olmaktan çıkar, "God object" olmaya başlar. Sınıfın 300 satırı geçtiyse ve içinde `if` yoğunluğu arttıysa, facade değil, bölünmesi gereken bir servis yazmışsındır.

### Ne zaman KULLANMA

- Alt sistem zaten basitse. İki servisi sırayla çağırmak için facade açmak fazlalıktır.
- Facade'ı tek çağıran varsa. Tek çağıran için soyutlama, sadece bir dosya daha demektir.
- Alt sistemin tamamını gizlemek gerekmiyorsa. Facade alt sistemi **kapatmaz**; ihtiyacı olan doğrudan içeri erişebilmelidir.

**Bu benzetme şurada bozulur:** Tapu danışmasında memur yanlış yönlendirirse sen hatayı yerinde görür ve itiraz edersin. Facade'da ise arkadaki bir servis hata verdiğinde çağıran taraf çoğu zaman sadece "işlem başarısız" bilgisini alır; hangi adımda ne olduğu kaybolur. Facade yazarken hata bilgisini yutmamak, hangi adımın patladığını dışarı taşıyabilmek ayrı bir tasarım işidir.

---

## 4. Proxy — Araya Girip Erişimi Yönetmek

> **Benzetme —** Bir avukat, müvekkili adına konuşur. Karşı taraf avukatla konuşurken aslında müvekkille konuşmaktadır; avukat aynı yetkiyle cevap verir. Ama her şeyi doğrudan iletmez: bazı soruları "müvekkilime sormam gerek" diye erteler, bazılarını yetkisi yok diye reddeder. Dışarıdan arayüz aynıdır, arada erişimi yöneten biri vardır.

**Basitçe:** Proxy, gerçek nesnenin yerine geçer ve ona erişimi kontrol eder. Aynı arayüzü uygular, çağrıyı ya iletir ya geciktirir ya da hiç iletmez.

**Teknik olarak:** **Proxy (vekil)** — Başka bir nesneye erişimi denetlemek için onun yerine geçen, aynı arayüzü uygulayan yapısal kalıp.

Yaygın türleri:

| Tür | Ne yapar |
|---|---|
| **Virtual proxy** | Pahalı nesneyi gerçekten gerekene kadar kurmaz (lazy loading) |
| **Protection proxy** | Erişimi yetkiye göre denetler |
| **Remote proxy** | Uzaktaki nesneyi yerel gibi gösterir (HTTP istemcisi) |
| **Caching proxy** | Sonucu saklar, tekrarında ağa/veritabanına gitmez |

### EF Core lazy loading proxy'leri — gerçek örnek

EF Core'un lazy loading özelliği, doğrudan bu kalıbın uygulamasıdır. `Microsoft.EntityFrameworkCore.Proxies` paketi, çalışma anında varlık sınıfından türeyen bir proxy sınıfı üretir; navigation property'ye ilk erişildiğinde veritabanına gider.

```csharp
// Kurulum
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseLazyLoadingProxies()
     .UseSqlServer(cs));

// Varlık — navigation property'ler virtual OLMAK ZORUNDA
public class Ilan
{
    public int Id { get; set; }
    public string Baslik { get; set; }
    public virtual ICollection<Basvuru> Basvurular { get; set; }   // virtual şart
}
```

```csharp
var ilan = await db.Ilanlar.FirstAsync(i => i.Id == 5);
// Buraya kadar tek sorgu: sadece Ilan çekildi.

foreach (var b in ilan.Basvurular)   // BU SATIRDA ikinci sorgu çalışır
    Console.WriteLine(b.AdSoyad);
```

Proxy'nin `virtual` şartının sebebi budur: EF Core, `Basvurular` özelliğini override ederek araya girer. `virtual` değilse override edemez ve lazy loading sessizce çalışmaz.

> **Performans tuzağı:** Lazy loading, **N+1 sorgu problemi**nin en yaygın sebebidir. 50 ilanı listeleyip her birinin başvuru sayısını yazdırırsan 1 + 50 = 51 sorgu çalışır. Kod masum görünür; fatura veritabanına çıkar. Doğru yol, ihtiyacı önceden bildirmektir:
>
> ```csharp
> var ilanlar = await db.Ilanlar
>     .Include(i => i.Basvurular)     // eager loading — tek sorguda getirir
>     .ToListAsync();
> ```
>
> Bu yüzden birçok ekip lazy loading'i hiç açmaz. `Include` ile açıkça belirtmek, sessiz sorgulardan daha güvenlidir.

### Ne zaman KULLANMA

- Yetkilendirme için. ASP.NET Core'da bunun yeri `[Authorize]` ve policy altyapısıdır; proxy yazmak tekerleği yeniden icat etmektir.
- Yalnızca davranış eklemek istiyorsan. Erişimi kontrol etmiyorsan yazdığın şey Decorator'dır; adını doğru koy.
- Lazy loading için, veri erişim katmanında. EF Core'un `Include`'u daha açık ve daha ucuzdur.

**Bu benzetme şurada bozulur:** Avukatla konuştuğunda avukatla konuştuğunu bilirsin. Proxy'de ise bilmezsin — tip aynı görünür. EF Core proxy'si tam olarak burada ısırır: elindeki nesne `Ilan` sanırsın, aslında `IlanProxy` türündedir. `GetType().Name` çağırırsan beklediğin adı görmezsin, `Equals` karşılaştırmaları şaşırtabilir ve serileştirmede fazladan alanlar çıkar. Görünmezlik kalıbın gücü ve tuzağıdır.

---

## 5. Composite — Parça ile Bütünü Aynı Görmek

> **Benzetme —** Bir şirketin organizasyon şemasını düşün. Genel müdürün altında departmanlar, departmanların altında ekipler, ekiplerin altında kişiler vardır. "Bu birimin toplam maaş gideri nedir?" sorusunu hem tek bir kişiye hem koca bir departmana sorabilirsin. Kişi kendi maaşını söyler, departman altındakilere sorup toplar. Soran taraf ikisini ayırt etmez.

**Basitçe:** Composite, tek bir nesneyle bir nesne grubunu aynı arayüz üzerinden kullanmanı sağlar. Ağaç yapılarının kalıbıdır.

**Teknik olarak:** **Composite (bileşik)** — Nesneleri ağaç yapısında birleştirerek parça-bütün hiyerarşisi kuran ve istemcinin tekil nesne ile bileşik nesneyi aynı biçimde kullanmasını sağlayan yapısal kalıp.

```csharp
public interface IMenuOgesi
{
    string Baslik { get; }
    bool Gorunur(ClaimsPrincipal kullanici);
}

// Yaprak (leaf) — alt öğesi yok
public class MenuLinki : IMenuOgesi
{
    public string Baslik { get; init; }
    public string Url { get; init; }
    public string GerekliRol { get; init; }

    public bool Gorunur(ClaimsPrincipal k)
        => GerekliRol is null || k.IsInRole(GerekliRol);
}

// Bileşik (composite) — alt öğeleri var
public class MenuGrubu : IMenuOgesi
{
    public string Baslik { get; init; }
    public List<IMenuOgesi> Alt { get; } = new();

    // Grup, en az bir alt öğesi görünürse görünür
    public bool Gorunur(ClaimsPrincipal k) => Alt.Any(o => o.Gorunur(k));
}
```

### Ne zaman KULLANMA

- Hiyerarşi yoksa. Düz bir liste için Composite fazlalıktır.
- Yaprak ile bileşiğin davranışı gerçekten farklıysa. Ortak arayüze zorlarsın, yaprakta `Ekle()` metodu `NotSupportedException` fırlatır; bu, kalıbın yanlış yerde olduğunun işaretidir.
- Veritabanından gelen derin ağaçlarda dikkatsizce. Özyinelemeli gezinme her düğümde sorgu çalıştırırsa performans çöker; ağacı tek sorguda çekip bellekte kur.

**Bu benzetme şurada bozulur:** Organizasyon şemasında hiyerarşi sonludur ve döngü olamaz; kimse kendi yöneticisinin yöneticisi olamaz. Kodda ise bir düğümü yanlışlıkla kendi altına eklemek mümkündür ve özyinelemeli gezinme sonsuz döngüye girer. Ağaç kurarken döngü kontrolü yapmak, benzetmenin hiç düşündürmediği bir zorunluluktur.

---

## 6. Strategy — Algoritmayı Dışarıdan Vermek

> **Benzetme —** Kayseri'den Ankara'ya gideceksin. Otobüs, uçak ya da kendi arabanla gidebilirsin. "Gitmek" işi aynı; yöntem değişir. Sen sabah kalkıp "bugün hangisi" diye karar verirsin, yolculuğun tanımını değiştirmezsin. Strategy, yöntemi yolculuktan ayırmaktır.

**Basitçe:** Strategy, aynı işi yapan farklı yolları ayrı sınıflara koyar ve hangisinin kullanılacağını dışarıya bırakır. Yeni bir yol eklemek, mevcut kodu açmayı gerektirmez.

**Teknik olarak:** **Strategy (strateji)** — Bir algoritma ailesini tanımlayan, her birini ayrı sınıfa alan ve birbirinin yerine geçebilir kılan davranışsal kalıp.

### Problem: büyüyen `switch`

```csharp
// KÖTÜ — her yeni indirim türü bu metodu açmayı gerektirir
public decimal IndirimHesapla(Siparis s, string tur)
{
    switch (tur)
    {
        case "yok":      return 0;
        case "yuzde10":  return s.Tutar * 0.10m;
        case "sadakat":  return s.Musteri.Puan > 1000 ? s.Tutar * 0.15m : 0;
        case "kampanya": return s.Tutar > 500 ? 50 : 0;
        default: throw new ArgumentException(nameof(tur));
    }
}
```

Bu metot **Open/Closed** ilkesini ihlal eder: davranışı genişletmek için sınıfı açman gerekir. Ayrıca her indirim türünün mantığı aynı metoda sıkışır ve tek tek test edilemez.

### Çözüm: her yol bir sınıf

```csharp
public interface IIndirimStratejisi
{
    string Kod { get; }
    decimal Hesapla(Siparis siparis);
}

public class IndirimYok : IIndirimStratejisi
{
    public string Kod => "yok";
    public decimal Hesapla(Siparis s) => 0m;
}

public class SadakatIndirimi : IIndirimStratejisi
{
    public string Kod => "sadakat";
    public decimal Hesapla(Siparis s)
        => s.Musteri.Puan > 1000 ? s.Tutar * 0.15m : 0m;
}

public class KampanyaIndirimi : IIndirimStratejisi
{
    public string Kod => "kampanya";
    public decimal Hesapla(Siparis s) => s.Tutar > 500 ? 50m : 0m;
}
```

Kullanan sınıf hiçbir türü tanımaz:

```csharp
public class FiyatHesaplayici
{
    private readonly IIndirimStratejisi _strateji;
    public FiyatHesaplayici(IIndirimStratejisi strateji) => _strateji = strateji;

    public decimal Odenecek(Siparis s) => s.Tutar - _strateji.Hesapla(s);
}
```

### DI ile strateji seçimi

Strateji çalışma anında belli oluyorsa, tüm uygulamaları enjekte edip koda göre seç:

```csharp
// Program.cs — hepsi aynı arayüzle kaydedilir
builder.Services.AddScoped<IIndirimStratejisi, IndirimYok>();
builder.Services.AddScoped<IIndirimStratejisi, SadakatIndirimi>();
builder.Services.AddScoped<IIndirimStratejisi, KampanyaIndirimi>();

public class SiparisServisi
{
    private readonly IReadOnlyDictionary<string, IIndirimStratejisi> _stratejiler;

    // Aynı arayüzden çok kayıt varsa IEnumerable<T> ile hepsi gelir
    public SiparisServisi(IEnumerable<IIndirimStratejisi> stratejiler)
        => _stratejiler = stratejiler.ToDictionary(s => s.Kod);

    public decimal Odenecek(Siparis s, string indirimKodu)
    {
        if (!_stratejiler.TryGetValue(indirimKodu, out var strateji))
            strateji = _stratejiler["yok"];
        return s.Tutar - strateji.Hesapla(s);
    }
}
```

`switch` yine kayboldu mu? Hayır — sözlüğe dönüştü. Fark şu: yeni bir indirim eklemek artık **yeni bir sınıf yazıp kaydetmek**tir; mevcut hiçbir dosyaya dokunmazsın. Kazanç budur.

.NET 8 ve sonrasında `AddKeyedScoped<IIndirimStratejisi, SadakatIndirimi>("sadakat")` ile kaydedip `GetRequiredKeyedService<IIndirimStratejisi>("sadakat")` ile çözmek daha doğrudan bir yoldur.

### Ne zaman KULLANMA

- Seçenek sayısı iki ve sabitse. Bir `if` daha okunaklıdır.
- Stratejiler birbirinin durumuna bağımlıysa. Strategy'nin şartı, uygulamaların birbirinden bağımsız olmasıdır.
- `switch` gerçekten değişmiyorsa. Beş yıldır aynı üç durum varsa, üç sınıf yazmak kodu iyileştirmez.

**Bu benzetme şurada bozulur:** Otobüs, uçak ve araba yolculuğu sonuçta seni aynı yere götürür; maliyet ve süre değişir. Stratejilerde ise **sonuç da farklı olabilir**: iki indirim stratejisi iki farklı tutar üretir. Yani seçim bir konfor tercihi değil, iş kuralı kararıdır. Bu yüzden stratejiyi seçen yerin (sözlük, keyed service, yapılandırma) iş kurallarına göre açıkça belgelenmiş olması gerekir.

---

## 7. Observer — Değişikliği İlgilenenlere Duyurmak

> **Benzetme —** Nöbetçi eczane listesine abone olursun; her gece hangi eczanenin nöbetçi olduğu sana bildirilir. Listeyi tutan kurum senin kim olduğunu, kaç kişi abone olduğunu umursamaz. Sen aboneliği istediğin zaman bırakırsın, duyuru yine yapılır. Bilgi tek yerden çıkar, ilgilenen herkese gider.

**Basitçe:** Observer, bir nesnede bir şey değiştiğinde bunu ilgilenen diğer nesnelere haber vermenin yoludur. Haber veren, kimin dinlediğini bilmek zorunda değildir.

**Teknik olarak:** **Observer (gözlemci)** — Bir nesnenin durum değişikliğini, ona bağlı nesnelere otomatik bildiren; yayıncı ile abone arasındaki bağı gevşek tutan davranışsal kalıp.

### .NET event mekanizması — dile gömülü Observer

```csharp
public class Stok
{
    public int Adet { get; private set; }

    // Yayıncı tarafı
    public event EventHandler<StokAzaldiEventArgs> StokAzaldi;

    public void Dus(int miktar)
    {
        Adet -= miktar;
        if (Adet < 10)
            StokAzaldi?.Invoke(this, new StokAzaldiEventArgs(Adet));
    }
}

public class StokAzaldiEventArgs(int kalan) : EventArgs
{
    public int KalanAdet { get; } = kalan;   // C# 12 primary constructor
}
```

Abone tarafı:

```csharp
var stok = new Stok();

stok.StokAzaldi += (gonderen, e) => Console.WriteLine($"Uyarı: {e.KalanAdet} adet kaldı");
stok.StokAzaldi += (g, e) => { /* tedarikçiye otomatik sipariş aç */ };

stok.Dus(95);   // iki abone de sırayla çağrılır
```

`?.Invoke` yazımı kasıtlıdır: hiç abone yoksa `StokAzaldi` alanı `null`'dır ve doğrudan çağırmak `NullReferenceException` verir.

> **Bellek sızıntısı uyarısı:** Olaya abone olan nesne, yayıncı tarafından **referansla tutulur**. Aboneliği `-=` ile bırakmazsan, abone nesne artık kullanılmasa bile çöp toplayıcı onu toplayamaz. Uzun ömürlü bir yayıncıya (singleton, static) kısa ömürlü nesneler abone oluyorsa bu, ders kitaplarına girmiş bir sızıntı kaynağıdır. Aboneliği `IDisposable` içinde bırakmak alışkanlık hâline gelmeli.

.NET aynı kalıbın arayüz biçimini de sunar: `IObservable<T>` ve `IObserver<T>`. Farkı, `Subscribe` çağrısının bir `IDisposable` döndürmesi ve "hata" ile "tamamlandı" durumlarının da modele girmesidir. Rx.NET bu arayüzlerin üstüne kurulur.

| | `event` | `IObservable<T>` |
|---|---|---|
| Aboneliği bırakma | `-=` ile, referansı saklaman gerekir | `Dispose()` ile, doğal |
| Hata ve bitiş bildirimi | Yok | `OnError`, `OnCompleted` |
| Zincirleme/filtreleme | Yok | Rx ile güçlü (`Where`, `Throttle`) |
| Tipik kullanım | Sınıf içi bildirimler, UI | Akış işleme, Rx.NET |

### Ne zaman KULLANMA

- Sıra önemliyse. Olay abonelerinin çağrılma sırasına güvenemezsin; sıralı akış gerekiyorsa Chain of Responsibility kullan.
- Abonenin hata vermesi yayıncıyı etkilememeli diyorsan, kendin sarmalamalısın; `event` bunu yapmaz, bir abonenin fırlattığı hata diğerlerine ulaşılmasını engeller.
- Uygulamalar arası haberleşme için. Süreçler arası olay dağıtımı message broker işidir (RabbitMQ, Azure Service Bus), in-process event değil.

**Bu benzetme şurada bozulur:** Nöbetçi eczane duyurusu tek yönlüdür; sen duyuruyu alınca kurumun işleyişini etkilemezsin. .NET olaylarında ise abone **aynı iş parçacığında ve senkron** çalışır. Bir abone on saniye sürerse yayıncı on saniye bekler; abone hata fırlatırsa yayıncının akışı kırılır. Duyuru pasif bir bilgilendirme değil, doğrudan bir metot çağrısıdır.

---

## 8. Command — İşlemi Nesneleştirmek

> **Benzetme —** Lokantada garson siparişini bir fişe yazar ve mutfağa asar. Fiş, "yapılacak iş"in kâğıda dökülmüş hâlidir. Sırada bekleyebilir, başka bir aşçıya verilebilir, iptal edilebilir, gün sonunda ne yapıldığını görmek için okunabilir. Garson yemeği pişirmeyi bilmez; fişi yazar, gerisi mutfağın işidir.

**Basitçe:** Command, bir işlemi metot çağrısı olarak değil, **nesne** olarak temsil eder. İşlem artık saklanabilir, kuyruğa alınabilir, geri alınabilir ve loglanabilir.

**Teknik olarak:** **Command (komut)** — Bir isteği, gerekli tüm bilgisiyle birlikte nesneye dönüştüren; böylece isteklerin parametreleştirilmesini, kuyruklanmasını, loglanmasını ve geri alınmasını sağlayan davranışsal kalıp.

```csharp
public interface IKomut
{
    void Calistir();
    void GeriAl();
}

public class BasvuruSilKomutu : IKomut
{
    private readonly AppDbContext _db;
    private readonly int _id;
    private Basvuru _yedek;          // geri alma için gereken durum

    public BasvuruSilKomutu(AppDbContext db, int id) => (_db, _id) = (db, id);

    public void Calistir()
    {
        _yedek = _db.Basvurular.Find(_id);
        _db.Basvurular.Remove(_yedek);
        _db.SaveChanges();
    }

    public void GeriAl()
    {
        if (_yedek is null) return;
        _yedek.Id = 0;                        // yeni kayıt olarak eklensin
        _db.Basvurular.Add(_yedek);
        _db.SaveChanges();
    }
}
```

### Undo yığını

Komutları bir yığında tutarsan geri alma neredeyse bedavaya gelir:

```csharp
public class KomutYoneticisi
{
    private readonly Stack<IKomut> _gecmis = new();

    public void Calistir(IKomut komut)
    {
        komut.Calistir();
        _gecmis.Push(komut);
    }

    public void SonIslemiGeriAl()
    {
        if (_gecmis.Count == 0) return;
        _gecmis.Pop().GeriAl();
    }
}
```

Metin editörlerindeki Ctrl+Z, tam olarak budur: her düzenleme bir komut nesnesidir, yığında durur.

### MediatR'a köprü

Bootcamp'te Web API ve CQRS konularına geldiğinde **MediatR** kütüphanesini göreceksin. MediatR, Command kalıbının .NET'teki en yaygın uygulamasıdır: her istek bir nesnedir, her isteğin bir işleyicisi vardır.

```csharp
// İstek — bir komut nesnesi
public record BasvuruOlusturCommand(string AdSoyad, string Eposta) : IRequest<int>;

// İşleyici — komutu çalıştıran taraf
public class BasvuruOlusturHandler : IRequestHandler<BasvuruOlusturCommand, int>
{
    public async Task<int> Handle(BasvuruOlusturCommand istek, CancellationToken ct)
    {
        var basvuru = new Basvuru { AdSoyad = istek.AdSoyad, Eposta = istek.Eposta };
        _db.Basvurular.Add(basvuru);
        await _db.SaveChangesAsync(ct);
        return basvuru.Id;
    }
}

// Controller — sadece komutu gönderir, nasıl işlendiğini bilmez
public async Task<IActionResult> Olustur(BasvuruOlusturCommand komut)
    => Ok(await _mediator.Send(komut));
```

Controller ile iş mantığı arasındaki tek bağ komut nesnesinin kendisidir. İşleyiciyi değiştirebilir, araya doğrulama ve loglama davranışları (`IPipelineBehavior`) ekleyebilirsin — ki o davranışlar da Decorator ve Chain of Responsibility karışımıdır.

### Ne zaman KULLANMA

- Basit CRUD işlemlerinde. `_repo.Ekle(x)` çağrısını komut nesnesine sarmak, dosya sayısını üçe katlar ve hiçbir şey kazandırmaz.
- Geri alma, kuyruklama veya denetim kaydı ihtiyacı yoksa. Kalıbın değeri bu üç şeyden gelir; yoksa fazlalıktır.
- Geri alma mantığı gerçekten karmaşıksa dikkatli ol. Veritabanına yazılmış, e-postası gönderilmiş bir işlemi "geri almak" çoğu zaman mümkün değildir; telafi işlemi (compensating action) tasarlamak gerekir.

**Bu benzetme şurada bozulur:** Fişi yırtarak siparişi iptal edersin ve hiçbir şey olmamış olur. Yazılımda ise `GeriAl()` çoğu zaman "hiç olmamış gibi yapmak" değildir: kayıt yeni bir `Id` ile geri gelir, gönderilen e-posta geri alınamaz, ödeme iadesi ayrı bir işlemdir. Undo, işlemi silmek değil **ters işlemi yapmaktır** ve her işlemin tersi yoktur.

---

## 9. Template Method — İskeleti Sabitleyip Adımları Değiştirmek

> **Benzetme —** Her kurumda evrak akışı aynıdır: evrak gelir, kaydedilir, incelenir, karara bağlanır, arşivlenir. Sıra değişmez. Ama "inceleme" adımı tapu dairesinde başka, nüfus müdürlüğünde başkadır. Akış merkezden gelir, adımın içini her birim kendi doldurur.

**Basitçe:** Template Method, bir işin adımlarının sırasını taban sınıfta sabitler, adımların bazılarının içini alt sınıflara bıraktırır. Sıra değiştirilemez, içerik değiştirilebilir.

**Teknik olarak:** **Template Method (şablon metot)** — Bir algoritmanın iskeletini taban sınıfta tanımlayan, bazı adımları alt sınıflara bırakan davranışsal kalıp. Abstract class'ın asıl varlık sebebi budur.

```csharp
public abstract class RaporUretici
{
    // Şablon metot — sıra burada sabit, override EDİLEMEZ
    public byte[] Uret(int raporId)
    {
        var veri = VeriGetir(raporId);          // her rapor kendi verisini çeker
        Dogrula(veri);                          // ortak
        var icerik = Bicimlendir(veri);         // her rapor kendi biçimi
        return Paketle(icerik);                 // ortak
    }

    protected abstract IReadOnlyList<RaporSatiri> VeriGetir(int raporId);
    protected abstract string Bicimlendir(IReadOnlyList<RaporSatiri> veri);

    // Ortak adımlar — alt sınıf dokunmaz
    private void Dogrula(IReadOnlyList<RaporSatiri> veri)
    {
        if (veri.Count == 0)
            throw new InvalidOperationException("Rapor için veri bulunamadı.");
    }

    private byte[] Paketle(string icerik) => Encoding.UTF8.GetBytes(icerik);

    // Hook — isteyen alt sınıf değiştirir, istemeyen dokunmaz
    protected virtual string Basligi() => "Rapor";
}

public class AylikSatisRaporu : RaporUretici
{
    private readonly AppDbContext _db;
    public AylikSatisRaporu(AppDbContext db) => _db = db;

    protected override IReadOnlyList<RaporSatiri> VeriGetir(int id)
        => _db.Satislar.Where(s => s.DonemId == id)
                       .Select(s => new RaporSatiri(s.UrunAdi, s.Tutar)).ToList();

    protected override string Bicimlendir(IReadOnlyList<RaporSatiri> veri)
        => string.Join("\n", veri.Select(v => $"{v.Ad};{v.Tutar}"));

    protected override string Basligi() => "Aylık Satış";
}
```

Üç erişim belirleyicisi kasıtlıdır: `public Uret` dışarıya açıktır ve **virtual değildir** (sıra kilitli), `protected abstract` adımlar alt sınıf için zorunludur, `protected virtual` hook'lar isteğe bağlıdır.

### Strategy ile farkı

| | Template Method | Strategy |
|---|---|---|
| Mekanizma | Kalıtım (derleme anında sabit) | Kompozisyon (çalışma anında değişir) |
| Değişen | Algoritmanın **bazı adımları** | Algoritmanın **tamamı** |
| Değiştirme zamanı | Alt sınıf yazarak | Nesne vererek |
| Esneklik | Düşük | Yüksek |
| Ortak kodu paylaşma | Doğal | Ayrı bir taban sınıf gerekir |

Kural şudur: adımların **sırası ortak**, içeriği farklıysa Template Method. Algoritmanın tamamı farklıysa Strategy.

> **Kalıtım uyarısı:** Template Method, kalıtımın hâlâ haklı olduğu az sayıda yerden biridir. Bunun dışında "kod paylaşmak için" taban sınıf yazmak kırılgan hiyerarşiler üretir. Taban sınıfa eklediğin her yeni `abstract` metot, tüm alt sınıfları derleme hatasıyla açmanı gerektirir; bu, kalıbın en somut bedelidir.

### Ne zaman KULLANMA

- Değişen adım bir taneyse. Bir `Func<T>` parametresi almak daha basittir.
- Alt sınıf sayısı hızla artıyorsa. Her varyasyon için sınıf açmak yerine Strategy'ye geç.
- Adımların sırası da değişiyorsa. Sıra sabit değilse kalıbın temel varsayımı düşer.

**Bu benzetme şurada bozulur:** Evrak akışında bir birim "bu adımı atlıyorum" diyemez. Kodda ise alt sınıf `Bicimlendir` içinde hiçbir şey yapmayıp boş string döndürebilir; akış çalışır ama sonuç anlamsızdır. Taban sınıf sırayı garanti eder, adımın **doğru doldurulduğunu** garanti etmez. Bu yüzden adımların sözleşmesini yorumla değil, dönüş tipiyle ve doğrulamayla sıkılaştırmak gerekir.

---

## 10. Chain of Responsibility — Zincirde Sırayla İşlemek

> **Benzetme —** Hastaneye girdiğinde önce güvenlik seni kontrol eder, sonra danışma yönlendirir, sonra kayıt masası dosya açar, sonra hemşire ateşini ölçer, en sonunda doktora çıkarsın. Her durak ya kendi işini yapıp seni bir sonrakine gönderir ya da "sen yanlış yerdesin" deyip geri çevirir. Güvenlik seni geri çevirirse doktor senden haberdar bile olmaz.

**Basitçe:** Chain of Responsibility, bir isteği sırayla birkaç işleyiciden geçirir. Her işleyici ya isteği işleyip sonlandırır ya da bir sonrakine devreder.

**Teknik olarak:** **Chain of Responsibility (sorumluluk zinciri)** — İsteği gönderen ile alan arasındaki bağı, isteği bir işleyici zincirinden geçirerek gevşeten davranışsal kalıp. Hangi halkanın işleyeceği çalışma anında belli olur.

### ASP.NET Core middleware pipeline'ı tam olarak budur

Bu, kalıbın .NET'teki en önemli uygulamasıdır. `Program.cs`'te yazdığın her `app.Use...` satırı zincire bir halka ekler.

```csharp
var app = builder.Build();

app.UseExceptionHandler("/Home/Error");   // 1. halka
app.UseHttpsRedirection();                // 2. halka
app.UseStaticFiles();                     // 3. halka
app.UseRouting();                         // 4. halka
app.UseAuthentication();                  // 5. halka
app.UseAuthorization();                   // 6. halka
app.MapControllerRoute(                   // zincirin sonu: endpoint
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

Her middleware iki şeyden birini yapar: işini yapıp `next`'i çağırır (zincir devam eder) ya da çağırmaz (**short-circuit** — zincir orada kesilir). `UseStaticFiles`, istek bir dosyaya denk geldiğinde dosyayı döner ve `next`'i çağırmaz; bu yüzden `wwwroot` isteklerinde yetkilendirme hiç çalışmaz.

Kendi middleware'ini yazdığında zincirin bir halkasını elle kurmuş olursun:

```csharp
public class IstekSuresiMiddleware
{
    private readonly RequestDelegate _next;            // zincirdeki bir sonraki halka
    private readonly ILogger<IstekSuresiMiddleware> _logger;

    public IstekSuresiMiddleware(RequestDelegate next,
                                 ILogger<IstekSuresiMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sayac = Stopwatch.StartNew();

        await _next(context);                          // devret — zincir devam eder

        _logger.LogInformation("{Yol} {Ms} ms",
            context.Request.Path, sayac.ElapsedMilliseconds);
    }
}
```

`await _next(context)` satırının **öncesi** istek yolunda, **sonrası** cevap yolunda çalışır. Zincir ileri gider, en sondan geri döner; yani her middleware isteği hem girişte hem çıkışta görür. Bu iki yönlü akış, sıradan bir Chain of Responsibility'den fazlasıdır ve `Program.cs` notunda ayrıntısıyla anlatılıyor.

> **Sıra kritiktir:** `UseAuthorization`'ı `UseAuthentication`'dan önce yazarsan yetkilendirme, kullanıcının kim olduğunu henüz bilmeyen bir bağlamda çalışır ve her istek yetkisiz sayılır. `UseRouting` olmadan endpoint bilgisi oluşmaz. Zincirdeki sıra bir stil tercihi değil, doğruluk şartıdır. Ayrıntılar için: `Hafta-04-AspNetCore-MVC/02-Program-cs-ve-Pipeline.md`.

### Ne zaman KULLANMA

- Tek bir işleyici varsa. Zincir kurmak, doğrudan çağırmaktan daha karmaşıktır.
- Hangi halkanın işleyeceği baştan belliyse. O zaman aradığın Strategy'dir, zincir değil.
- Zincir uzun ve loglanmıyorsa. Bir isteğin neden cevapsız kaldığını bulmak için halkaları tek tek gezmek gerekir.

**Bu benzetme şurada bozulur:** Hastanede her durak seni gördüğünü bilir ve kayıt tutar. Middleware zincirinde ise bir halka `next`'i çağırmadığında sonraki halkalar **isteğin varlığından bile haberdar olmaz**. Beklediğin bir log satırı hiç yazılmıyorsa, sorun senin middleware'inde değil, ondan önce zinciri kesen bir halkada olabilir. Bu, pipeline hatalarında en sık gözden kaçan noktadır.

---

## 11. Iterator — Gezinmeyi Koleksiyondan Ayırmak

> **Benzetme —** Kütüphanede kitapları raftan almak için kütüphanecinin yöntemini bilmen gerekmez. "Sıradaki" dersin, o sana sıradakini verir; "bitti" derse durursun. Kitapların rafta nasıl dizildiği, hangi sistemle bulunduğu seni ilgilendirmez.

**Basitçe:** Iterator, bir koleksiyonu, içinin nasıl tutulduğunu bilmeden baştan sona gezmeni sağlar. C#'ta `IEnumerable` ve `foreach` bunun dile gömülmüş hâlidir.

**Teknik olarak:** **Iterator (yineleyici)** — Bir topluluğun elemanlarına, iç yapısını açığa çıkarmadan sırayla erişmeyi sağlayan davranışsal kalıp.

```csharp
// foreach yazdığında derleyicinin yaptığı şey (özet)
IEnumerator<Basvuru> gezgin = basvurular.GetEnumerator();
while (gezgin.MoveNext())
{
    var b = gezgin.Current;
    // ...
}
```

`List<T>`, `Dictionary<K,V>`, `HashSet<T>` ve EF Core'un `IQueryable<T>`'ı tamamen farklı iç yapılara sahiptir; hepsini aynı `foreach` ile gezersin. Kalıbın kazancı budur.

### `yield return` — bedava iterator

```csharp
public IEnumerable<Basvuru> AktifBasvurular(IEnumerable<Basvuru> hepsi)
{
    foreach (var b in hepsi)
    {
        if (b.Aktif)
            yield return b;        // durumu koruyan bir iterator'a derlenir
    }
}
```

`yield return` yazdığında derleyici senin için bir durum makinesi üretir. Metot çağrıldığında gövdesi çalışmaz; ilk `MoveNext()` ile başlar, her `yield return`'de durur ve bir sonraki istekte kaldığı yerden devam eder. LINQ'in ertelenmiş çalıştırma davranışının altında bu mekanizma vardır.

### Ne zaman KULLANMA

- Elinde zaten `List<T>` varsa ve tamamı belleğe alınmışsa. `IEnumerable` döndürmek okuyucuyu "bu tembel mi?" diye düşündürür; `IReadOnlyList<T>` niyetini daha net söyler.
- Koleksiyonu birden çok kez gezeceksen. Her geziş `yield` gövdesini yeniden çalıştırır; veritabanına gidiyorsa her seferinde yeni sorgu demektir.
- Kendi `IEnumerator<T>` sınıfını elle yazmak için. `yield return` neredeyse her durumda yeterlidir.

**Bu benzetme şurada bozulur:** Kütüphaneci sana kitapları verirken raftaki düzen değişmez. Kodda ise geziyorken koleksiyonu değiştirmek `InvalidOperationException` ile sonuçlanır ("Collection was modified"). Gezerken silmen gerekiyorsa ya önce `ToList()` ile kopyasını alırsın ya da geriye doğru `for` döngüsüyle gezersin. Kütüphanede kitap almak masum bir eylemdir; gezerken koleksiyona dokunmak değildir.

---

## 12. Kalıp Seçimi ve Pattern Hastalığı

> **Benzetme —** Bir oto tamircisinde ustanın duvarında elli alet asılıdır ama çoğu gün beşini kullanır. Çırak ise yeni öğrendiği alet neyse her arızada onu dener. Aleti çok bilmek ustalık değildir; hangisini **kullanmayacağını** bilmek ustalıktır.

**Basitçe:** Kalıplar problem çözmek içindir. Problem yoksa kalıp, çözüm değil yeni bir problemdir.

**Teknik olarak:** Önce karar tablosu, sonra uyarı.

### Hangi problemde hangi kalıp

| Problemin | Kalıp | .NET'te karşılığı |
|---|---|---|
| "Üçüncü parti kütüphanenin arayüzü bana uymuyor" | Adapter | Kendi arayüzünü sarmalayan sınıf |
| "Sınıfa dokunmadan loglama/önbellek eklemek istiyorum" | Decorator | Scrutor `Decorate`, MediatR `IPipelineBehavior` |
| "Beş servisi doğru sırayla çağırmak her yerde tekrar ediyor" | Facade | Uygulama servisi, Unit of Work |
| "Erişimi denetlemek, geciktirmek veya uzağa bağlamak istiyorum" | Proxy | EF Core lazy loading proxy'leri |
| "Parça ile bütünü aynı şekilde işlemek istiyorum" | Composite | Menü, kategori, dosya ağacı |
| "Kodumda büyüyen bir `switch` var" | Strategy | `IEnumerable<T>` enjeksiyonu, keyed services |
| "Bir şey değişince birkaç yerin haberi olmalı" | Observer | `event`, `IObservable<T>` |
| "İşlemi saklamak, kuyruklamak, geri almak istiyorum" | Command | MediatR `IRequest`, `Stack<IKomut>` |
| "Akış aynı, adımlar farklı" | Template Method | `abstract` taban sınıf |
| "İstek birkaç duraktan sırayla geçmeli" | Chain of Responsibility | ASP.NET Core middleware |
| "Koleksiyonu iç yapısını bilmeden gezmek istiyorum" | Iterator | `IEnumerable<T>`, `yield return` |

### Pattern hastalığı

Kalıpları yeni öğrenen herkes aynı dönemden geçer: elindeki her problem bir kalıp arar. Bunun belirtileri bellidir.

**Belirti 1 — Tek uygulamalı arayüz.** `IKullaniciServisi` arayüzünü yazdın, tek uygulaması `KullaniciServisi`. Test için sahtesini bile kullanmıyorsun. Bu arayüz soyutlama değil, fazladan bir dosya.

**Belirti 2 — Fabrikanın fabrikası.** `IRepositoryFactoryProvider` gibi isimler gördüğünde, birileri DI konteynerinin yaptığı işi üç katman üstten yeniden yazmıştır.

**Belirti 3 — Sekiz katmanlı decorator zinciri.** Her katman ayrı ayrı mantıklıdır; birleşince hatanın nereden geldiği bulunamaz hâle gelir.

**Belirti 4 — İki satırlık iş için strateji.** İki seçenek arasında bir `if` yeterken üç sınıf, bir arayüz ve bir kayıt satırı yazmak.

### YAGNI

**YAGNI (You Aren't Gonna Need It)** — İhtiyaç ortaya çıkmadan ona hazırlık yapma. "İleride belki başka veritabanı kullanırız" cümlesi bugün üç katman yazmanın gerekçesi değildir. İhtiyaç geldiğinde eklemek, gelmeyen ihtiyaç için yazılmış kodu sürdürmekten daha ucuzdur.

> **Karşı uç da vardır:** Pattern hastalığından korkup hiç soyutlama yapmamak da bir hatadır. 800 satırlık bir controller metodu, "sade kod" değil bakımı imkânsız koddur. Ölçü şudur: kalıp bir **acıya** cevap veriyorsa yerindedir. Acı yoksa bekle; acı geldiğinde kalıp kendini gösterir.

**Bu benzetme şurada bozulur:** Usta yanlış aleti seçtiğinde vida dönmez, hatayı hemen görür. Kodda yanlış kalıp seçimi anında görünmez; kod çalışır, testler geçer, kimse şikâyet etmez. Bedel altı ay sonra bir değişiklik istendiğinde ödenir. Bu yüzden "çalışıyor mu" yetersiz bir ölçüttür; doğru soru "değiştirmesi kolay mı"dır.

---

## Tek Bakışta Özet

- Yapısal kalıplar nesneleri **birleştirir**, davranışsal kalıplar nesneleri **konuşturur**.
- Decorator, sınıfa dokunmadan davranış ekler; sarılan ve saran aynı arayüzü uygular.
- Decorator'da **kayıt sırası çalışma sırasıdır**; en dıştaki katman içeridekini hiç çağırmayabilir.
- EF Core lazy loading, çalışma anında üretilen proxy sınıflarıyla çalışır; bu yüzden navigation property'ler `virtual` olmalıdır.
- Lazy loading N+1 sorgu probleminin en yaygın sebebidir; `Include` ile açıkça belirtmek daha güvenlidir.
- Strategy `switch`'i sınıflara böler; yeni seçenek eklemek mevcut kodu açmayı gerektirmez.
- `event` senkron çalışır: bir abone yavaşsa yayıncı bekler, hata fırlatırsa akış kırılır.
- Command işlemi nesneleştirir; undo, kuyruklama ve denetim kaydı buradan gelir. MediatR bunun yaygın uygulamasıdır.
- Template Method akışı sabitler, adımları alt sınıfa bırakır; `abstract class`'ın asıl kullanım yeridir.
- ASP.NET Core middleware pipeline'ı bir Chain of Responsibility'dir; `next` çağrılmazsa zincir kesilir (short-circuit).
- Middleware sırası doğruluk şartıdır: `UseAuthentication` her zaman `UseAuthorization`'dan önce gelir.
- Kalıp bir acıya cevap veriyorsa yerindedir; acı yoksa YAGNI geçerlidir.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Adapter | Uyumsuz bir arayüzü beklenen arayüze çeviren kalıp |
| Decorator | Sınıfı sararak çalışma anında davranış ekleyen kalıp |
| Facade | Karmaşık alt sistemin önüne tek arayüz koyan kalıp |
| Proxy | Gerçek nesnenin yerine geçip erişimi denetleyen kalıp |
| N+1 problemi | Bir sorgu + her satır için ek sorgu üreten kalıp |
| Composite | Parça ile bütünü aynı arayüzden kullandıran kalıp |
| Strategy | Algoritmayı ayrı sınıfa alıp dışarıdan verdiren kalıp |
| Observer | Durum değişikliğini abonelere bildiren kalıp |
| Command | İşlemi nesne olarak temsil eden kalıp |
| Template Method | Akışı taban sınıfta sabitleyip adımları alt sınıfa bırakan kalıp |
| Chain of Responsibility | İsteği işleyici zincirinden sırayla geçiren kalıp |
| Iterator | Koleksiyonu iç yapısını açmadan gezdiren kalıp |
| `yield return` | Derleyicinin durum makinesi ürettiği tembel üretim sözdizimi |
| YAGNI | Henüz olmayan ihtiyaç için kod yazmama ilkesi |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Decorator ile Proxy aynı kalıptır" | Kodları benzer, niyetleri farklı: biri davranış ekler, diğeri erişimi yönetir |
| "Decorator kayıt sırası önemsizdir" | Sıra çalışma sırasıdır; önbellek en dıştaysa loglama hiç çalışmayabilir |
| "Facade tüm alt sistemi kapatır" | Kapatmaz; ihtiyacı olan doğrudan alt sisteme erişebilmelidir |
| "Lazy loading her zaman performans kazandırır" | Çoğu zaman N+1 üretir; `Include` ile eager loading daha öngörülebilirdir |
| "EF Core lazy loading `virtual` olmadan da çalışır" | Çalışmaz; proxy override edemediği için sessizce devre dışı kalır |
| "Strategy ile Template Method aynı işi yapar" | Strategy algoritmanın tamamını, Template Method bazı adımlarını değiştirir |
| "`event` asenkron çalışır" | Senkron çalışır; aboneler yayıncının iş parçacığında sırayla çağrılır |
| "Olay aboneliğini bırakmamak zararsızdır" | Yayıncı aboneyi referansla tutar; bellek sızıntısı olur |
| "Middleware sırası stil tercihidir" | Doğruluk şartıdır; yanlış sırada yetkilendirme ve routing bozulur |

---

## Sonraki

→ `07-Hafta-Ozeti.md` (Pazar)
