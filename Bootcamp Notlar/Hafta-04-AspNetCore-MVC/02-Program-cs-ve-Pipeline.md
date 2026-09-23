# Hafta 4 · `Program.cs`, Middleware Pipeline ve DI Yaşam Döngüleri

**Okuma süresi:** ~48 dk
**Neden bu konu:** Bir ASP.NET Core uygulamasının tamamı bu tek dosyadan başlar. "Ayarımı yazdım ama çalışmıyor" ve "uygulama yük altında kilitleniyor" sorunlarının büyük kısmı burada, üç satırlık bir sıra hatasından çıkar.

---

## Önce Basitçe

Havalimanına gittiğini düşün. Uçağa binmek için tek bir kapıdan geçmezsin; arka arkaya dizilmiş kontrol noktalarından geçersin. Önce güvenlik, sonra check-in, sonra pasaport, sonra kapı görevlisi. Her nokta kendi işini yapar ve seni bir sonrakine yollar. Biri "sen geçemezsin" derse orada durursun; arkadaki noktalar seni hiç görmez.

Bir ASP.NET Core uygulamasına gelen her HTTP isteği de tam olarak böyle ilerler. Tarayıcıdan gelen istek, senin sıraladığın kontrol noktalarından tek tek geçer, en sonunda Controller'a ulaşır. Controller yanıtı üretir ve yanıt aynı noktalardan **geriye doğru** çıkar. Bu dizilime **pipeline**, her bir noktaya **middleware** denir.

Bu noktaların **sırası** kendi başlarına ne yaptıkları kadar önemlidir. Pasaport kontrolünden önce "bu yolcu VIP salona girebilir mi" diye sormanın anlamı yoktur; çünkü henüz kim olduğunu bilmiyorsundur. Aynı şekilde kimlik doğrulamadan önce yetki kontrolü koyarsan, sistem herkesi "tanımıyorum" diye görür ve yetki kuralların sessizce yanlış çalışır. Sıra hatalarının en can sıkıcı tarafı budur: program çöküp sana hata vermez, sadece yanlış davranır.

Havalimanının bir de arka tarafı var: kontrol noktaları açılmadan önce birinin X-ray cihazını, bantları, bilgisayarları kurması gerekir. `Program.cs` dosyasının ilk bölümü tam olarak bu kurulumdur. Uygulamanın ihtiyaç duyacağı her araç — veritabanı bağlantısı, kendi yazdığın servisler, MVC altyapısı — önceden tanıtılır. Buna **Dependency Injection (DI)** deniyor. Kısaca: bir sınıf ihtiyacı olan aracı kendisi imal etmez, "bana şu lazım" der ve çatı onu eline verir.

Tanıttığın her araç için bir de **ne kadar yaşayacağına** karar verirsin. Bazı araçlar her istendiğinde yenilenir, bazıları bir yolcunun işlemi bitene kadar aynı kalır, bazıları da havalimanı açık olduğu sürece tek tanedir. Bu üç seçeneğe Transient, Scoped ve Singleton diyoruz ve yanlış seçim, uygulamanın yük altında tuhaf davranmasının bir numaralı sebebidir.

Özetle `Program.cs` iki soruyu cevaplar: "Elimde hangi araçlar var ve ne kadar yaşıyorlar?" ve "İstek hangi noktalardan, hangi sırayla geçiyor?" Şimdi detaya iniyoruz.

> **Ana benzetme:** `Program.cs` bir havalimanının kurulum ve akış planıdır. Birinci bölümde cihazları ve personeli tanıtırsın (DI), üçüncü bölümde yolcunun geçeceği kontrol noktalarını sıraya dizersin (middleware pipeline). Sıra yanlışsa kimse şikâyet etmez, sadece yanlış kişiler yanlış yerlere girer.

---

## Bu Notta Ne Var

1. `Program.cs`'in üç bölümü
2. Bölüm 1 — Builder ve servis kayıtları, Dependency Injection
3. DI yaşam döngüleri: Transient, Scoped, Singleton ve captive dependency
4. Bölüm 2 — Uygulamanın inşası
5. Bölüm 3 — Middleware pipeline
6. Sıra neden hayati
7. Kendi middleware'ini yazmak

---

## 1. `Program.cs`'in Üç Bölümü

> **Benzetme —** Bir lokanta açmayı düşün. Önce mutfağı kurarsın: ocak, buzdolabı, tedarikçi anlaşmaları. Sonra kapıyı açarsın — o andan sonra "bir fırın daha alalım" diyemezsin, servis başlamıştır. En sonunda müşterinin izleyeceği yolu belirlersin: kapı, vestiyer, karşılama, masa. Üç aşama, hep aynı sırada.

**Basitçe:** Bu dosya üç parçadan oluşur ve parçalar birbirine karışmaz. Önce "neyim var" dersin, sonra "başlıyorum" dersin, sonra "istek şu yollardan geçsin" dersin. Bir satıra baktığında hangi parçada olduğunu anlamak, dosyayı okumanın yarısıdır.

**Teknik olarak:** Eski .NET sürümlerinde `Startup.cs` ve `Program.cs` ayrı dosyalardı. .NET 6'dan itibaren her şey tek dosyada toplandı ve `Main` metodu da örtük hâle geldi (Hafta 1, `05-Modern-CSharp-Ozellikleri.md` — top-level statements).

Dosya her zaman şu üç bölümden oluşur:

```
┌─────────────────────────────────────────────────────┐
│  BÖLÜM 1 — BUILDER: servisleri kaydet               │
│  var builder = WebApplication.CreateBuilder(args);  │
│  builder.Services.Add...                            │
├─────────────────────────────────────────────────────┤
│  BÖLÜM 2 — BUILD: uygulamayı inşa et                │
│  var app = builder.Build();                         │
│  ⚠ bu satırdan sonra YENİ SERVİS EKLENEMEZ          │
├─────────────────────────────────────────────────────┤
│  BÖLÜM 3 — PIPELINE: ara katmanları sırala          │
│  app.Use...                                         │
│  app.Run();                                         │
└─────────────────────────────────────────────────────┘
```

Bu üç bölümü ayırt edebilmek, dosyayı okurken nerede olduğunu bilmeni sağlar: `builder.Services` gördüğün her yer 1. bölüm, `app.Use` gördüğün her yer 3. bölümdür.

Pratik bir ayırt etme kuralı: değişken adına bak. `builder` ile başlıyorsa kurulumdasın, `app` ile başlıyorsa akıştasın.

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();  // 1. bölüm — kurulum
var app = builder.Build();                                  // 2. bölüm — sınır
app.UseAuthentication();                                    // 3. bölüm — akış
```

> **Bu benzetme şurada bozulur:** Lokantada kapıyı açtıktan sonra da mutfağa yeni bir tencere sokabilirsin, kimse engellemez. Kodda `builder.Build()` satırından sonra servis eklemek **mümkün değildir**; konteyner mühürlenir ve denersen çalışma anında hata alırsın. Yani buradaki sınır mecazi değil, gerçek bir kapıdır.

---

## 2. Bölüm 1 — Builder ve Servis Kayıtları

> **Benzetme —** Apartman görevlisini düşün. Ampulün patladığında markete koşup ampul aramazsın; görevliyi ararsın, o depodan uygun ampulü getirir. Sen "bana ışık lazım" dersin, hangi markadan, nereden geldiğiyle uğraşmazsın. Dependency Injection budur: sınıfın ihtiyacını `new` ile kendi üretmez, görevliden ister. Ama görevlinin depoda ne olduğunu bilmesi gerekir — işte `builder.Services` o deponun kayıt defteridir.

**Basitçe:** Uygulama başlamadan önce "şu isteyene şunu ver" diye bir liste doldurursun. Sonra herhangi bir sınıf bir şeye ihtiyaç duyduğunda onu constructor'ında ister, çatı listeye bakıp uygun nesneyi eline verir. Sınıfın kendisi hiçbir şey imal etmez.

**Teknik olarak:** Uygulama ayağa kalkmadan önce ihtiyaç duyduğu her şey burada **DI konteynerine** tanıtılır.

```csharp
// 1. Uygulama oluşturucusunu başlat
var builder = WebApplication.CreateBuilder(args);

// 2. MVC yapısını etkinleştir (Controller ve View'ları aktif eder)
builder.Services.AddControllersWithViews();

// 3. Veritabanı (DbContext) ayarı
//    Bağlantı cümlesi appsettings.json'dan okunur, koda gömülmez
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 4. Kendi yazdığın servisler
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

**`builder.Configuration`** — `appsettings.json`, ortam değişkenleri ve (varsa) user secrets gibi kaynakları birleştiren konfigürasyon nesnesi. `GetConnectionString("DefaultConnection")` ifadesi aslında `Configuration["ConnectionStrings:DefaultConnection"]` için bir kısayoldur.

**Dependency Injection (Bağımlılık Enjeksiyonu)** — Bir sınıfın ihtiyaç duyduğu nesneleri `new` ile kendi içinde oluşturmak yerine **dışarıdan alması** ilkesi. ASP.NET Core'da bu genellikle constructor üzerinden olur:

```csharp
public class OrderController : Controller
{
    private readonly ApplicationDbContext _context;

    // Çatı (framework) bu context'i otomatik sağlar
    public OrderController(ApplicationDbContext context)
    {
        _context = context;
    }
}
```

Kazandırdığı şey somut: yarın MSSQL yerine PostgreSQL'e geçmek istediğinde veya birim test yazmak istediğinde, controller kodunu **hiç değiştirmeden** yalnızca enjekte edilen servisi değiştirirsin.

Farkı iki kod parçasında görmek en kolayı:

```csharp
// DI YOK — sınıf kendi bağımlılığını üretiyor
public class SiparisServisi
{
    private readonly SmtpEpostaGonderici _eposta = new SmtpEpostaGonderici();

    // Bu sınıfı test etmek için gerçek bir SMTP sunucusu gerekir.
    // E-posta sağlayıcısını değiştirmek için bu dosyayı açman gerekir.
}

// DI VAR — sınıf ihtiyacını dışarıdan istiyor
public class SiparisServisi
{
    private readonly IEpostaGonderici _eposta;

    public SiparisServisi(IEpostaGonderici eposta) => _eposta = eposta;

    // Testte sahte (fake) bir gönderici verirsin, hiçbir e-posta gitmez.
    // Sağlayıcı değişince yalnızca Program.cs'teki tek satır değişir.
}
```

Kayıt tarafı da tek satırdır ve değişen tek yer orasıdır:

```csharp
builder.Services.AddScoped<IEpostaGonderici, SmtpEpostaGonderici>();
// Yarın SendGrid'e geçersen:
// builder.Services.AddScoped<IEpostaGonderici, SendGridEpostaGonderici>();
```

Bir servisi kaydetmeyi unutursan uygulama açılışta değil, o servis **ilk kez istendiğinde** patlar ve mesaj nettir:

```
InvalidOperationException: Unable to resolve service for type
'IEpostaGonderici' while attempting to activate 'SiparisServisi'.
```

Bu hatayı gördüğünde bakacağın yer her zaman aynıdır: `Program.cs`'in birinci bölümünde o arayüz için bir `Add...` satırı var mı?

> **Bu benzetme şurada bozulur:** Apartman görevlisinden ampul istediğinde o depoya gider ve *o an* bakar; deponun içindekiler gün içinde değişebilir. DI konteyneri ise **uygulama başlarken** mühürlenir. Çalışma sırasında "şunu da ekleyeyim" diyemezsin. Ayrıca görevli sana yanlış ampul verirse ampulü elinde tutar ve uyarırsın; konteyner ise kaydı olmayan bir tipi hiç üretmez, doğrudan istisna fırlatır. Yani bu görevli esnek değil, katı kurallı bir görevlidir.

---

## 3. DI Yaşam Döngüleri

> **Benzetme —** Yine apartman. Görevliden üç farklı şey istersin. Bir: **çöp poşeti**. Her istediğinde yeni bir tane alırsın, kullanır atarsın, kimseyle paylaşmazsın. İki: **daire anahtarı**. Sen o dairede oturduğun sürece aynı anahtar senindir; ailenin her ferdi aynı anahtarı kullanır ama yan dairedeki komşunun anahtarı bambaşkadır. Üç: **apartmanın ana giriş kapısı**. Binada tek tanedir, herkes aynı kapıyı kullanır, sen taşınsan da o kapı yerinde durur.

**Basitçe:** Bir servisi kaydederken "bu nesne ne kadar yaşasın" sorusuna cevap verirsin. Üç cevap var: her istendiğinde yenisi, her kullanıcı isteği için bir tane, ya da uygulama boyunca tek tane. Yanlış cevap, kodun yazıldığı gün değil, yük bindiği gün sorun çıkarır.

**Teknik olarak:** Bir servisi kaydederken üç seçenekten birini seçersin. Bu seçim performansı ve **veri tutarlılığını** doğrudan etkiler.

| Metot | Ne zaman yeni nesne üretilir | Ömrü | Apartman karşılığı |
|---|---|---|---|
| **`AddTransient`** | Her istendiğinde | Çağrı başına | Çöp poşeti |
| **`AddScoped`** | Her HTTP isteğinde bir kez | İstek başına | Daire anahtarı |
| **`AddSingleton`** | Uygulama başlarken bir kez | Uygulama boyunca | Ana giriş kapısı |

Burada kilit kavram **scope** (kapsam) kelimesidir ve genelde yanlış anlaşılır. Scope, "bir kullanıcı" ya da "bir oturum" demek **değildir**. Scope, **tek bir HTTP isteğinin** başlangıcı ile yanıtın gönderilmesi arasındaki süredir. Aynı kullanıcı sayfayı üç kez yenilerse üç ayrı scope oluşur.

### 3.1 `AddTransient` — her seferinde yepyeni

Aynı HTTP isteği içinde bile, servis her istendiğinde yeni bir nesne üretilir.

**Senaryo:** Şifre sıfırlama için SMS gönderen bir `SmsService` var. Bir istek geldiğinde hem Controller hem de arka plandaki loglama katmanı bu servisi istiyor. `AddTransient` ile **iki ayrı** `SmsService` nesnesi üretilir.

**Ne zaman uygun:** İçinde durum (state) tutmayan, hafif, hızlı çalışıp kapanan yardımcı servisler.

Farkı gözle görmek için servise bir kimlik verip aynı istek içinde iki kez istemek yeterlidir:

```csharp
public class IzSurucu
{
    public Guid Kimlik { get; } = Guid.NewGuid();   // her nesnede farklı
}

public class DemoController : Controller
{
    private readonly IzSurucu _birinci;
    private readonly IzSurucu _ikinci;

    public DemoController(IzSurucu birinci, IzSurucu ikinci)
    {
        _birinci = birinci;
        _ikinci = ikinci;
    }

    public IActionResult Index()
    {
        // AddTransient  -> iki Guid FARKLI
        // AddScoped     -> iki Guid AYNI (aynı istek içindeyiz)
        // AddSingleton  -> iki Guid AYNI (ve sayfayı yenileyince yine aynı)
        return Content($"{_birinci.Kimlik}\n{_ikinci.Kimlik}");
    }
}
```

Bu üç satırı sırayla deneyip sayfayı birkaç kez yenilemek, yaşam döngülerini okumaktan daha iyi öğretir:

```csharp
builder.Services.AddTransient<IzSurucu>();
// builder.Services.AddScoped<IzSurucu>();
// builder.Services.AddSingleton<IzSurucu>();
```

### 3.2 `AddScoped` — istek başına bir tane

Bir kullanıcı butona bastığında istek sunucuya ulaşır ve yanıt dönene kadar geçen süre bir **"scope"** (kapsam) oluşturur. Bu süre içinde servisi kim isterse istesin **aynı nesne** verilir. Başka bir kullanıcının isteği için ayrı bir nesne üretilir.

**Senaryo:** Bir ERP'de sipariş kaydediliyor. Controller `IUnitOfWork`'ü çağırıyor; UnitOfWork siparişi kaydetmek için `OrderRepository`'yi, stok düşmek için `ProductRepository`'yi kullanıyor. `DbContext` **Scoped** olduğu için her iki repository de **aynı veritabanı bağlantısı** üzerinde çalışır. İşlemlerden biri hata verirse diğeri de transaction sayesinde geri alınır (rollback).

`DbContext` yanlışlıkla `Transient` yapılsaydı: sipariş kaydı ayrı, stok düşümü ayrı bağlantı açardı ve **veri bütünlüğü bozulurdu**.

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    public IOrderRepository Orders { get; }
    public IProductRepository Products { get; }

    // Her ikisi de AYNI _context üzerinde çalışır — çünkü Scoped.
    public UnitOfWork(ApplicationDbContext context,
                      IOrderRepository orders,
                      IProductRepository products)
    {
        _context = context;
        Orders = orders;
        Products = products;
    }

    // Tek SaveChanges: ya ikisi de yazılır, ya hiçbiri.
    public Task<int> KaydetAsync() => _context.SaveChangesAsync();
}
```

**Ne zaman uygun:** `DbContext`, repository'ler, Unit of Work — yani bir isteğin tamamı boyunca tutarlı kalması gereken her şey.

### 3.3 `AddSingleton` — uygulama boyunca tek

Uygulama ilk ayağa kalktığında bir kez oluşturulur; uygulama kapanana (veya sunucu yeniden başlatılana) kadar sisteme giren herkese aynı nesne hizmet eder.

**Senaryo:** Bir fintech uygulamasında canlı döviz kurları gerekiyor. Kurlar dakikada bir güncelleniyorsa, sisteme giren 10.000 kullanıcının her biri için kurları veritabanından veya dış API'den tekrar tekrar çekmek sunucuyu felç eder. Bunun yerine bir `ExchangeRateCacheService` yazılır ve `AddSingleton` olarak eklenir: kurlar ilk çağrıldığında hafızaya alınır, sonraki binlerce istek RAM'den okur.

**Ne zaman uygun:** Önbellekleme (cache), `appsettings.json` okuyan konfigürasyon servisleri, tüm sistemin paylaştığı sabit veriler.

Singleton'ın görünmeyen bedeli **thread güvenliğidir**. Aynı nesneyi aynı anda yüzlerce istek kullanır. İçinde paylaşılan bir koleksiyon varsa onu korumak senin işindir:

```csharp
public class KurOnbellegi
{
    // Dikkat: normal Dictionary burada YANLIŞ olurdu.
    // Aynı anda iki istek yazmaya kalkarsa sözlük bozulur.
    private readonly ConcurrentDictionary<string, decimal> _kurlar = new();

    public decimal Getir(string kod) => _kurlar.TryGetValue(kod, out var d) ? d : 0m;
    public void Guncelle(string kod, decimal deger) => _kurlar[kod] = deger;
}
```

```csharp
builder.Services.AddSingleton<KurOnbellegi>();
```

### 3.4 Restoran analojisi

| Yaşam döngüsü | Restoran karşılığı |
|---|---|
| **Transient** | **Kağıt peçete** — her müşteriye, hatta aynı müşteriye her istediğinde yepyeni bir tane verilir, iş bitince atılır |
| **Scoped** | **Garson** — restorana girdiğinde (istek başladığında) masana bir garson atanır; yemeğin bitene kadar su da hesap da aynı garsondan gelir. Yan masanın garsonu başkadır |
| **Singleton** | **Menü (veya müdür)** — restorandaki herkes aynı menüyü okur, herkesin müdürü aynı kişidir. Yeni müşteri gelse de o nesne değişmez |

> **Bu benzetme şurada bozulur:** Restoranda garson masadan masaya gider, aynı garson iki masaya birden bakabilir. Scoped nesne böyle davranmaz: bir scope'un nesnesi **yalnızca o isteğe** aittir, başka bir istek onu asla görmez. Ayrıca gerçek garson mesai bitince eve gider ama işe yarar hâlde kalır; Scoped nesne istek biterken `Dispose` edilir, yani kapatılır. Yanlışlıkla elinde tutup sonra kullanmaya çalışırsan `ObjectDisposedException` alırsın.
>
> İkinci kırılma noktası: menü benzetmesi Singleton'ı fazla masum gösterir. Menüyü herkes **okur**, kimse üstüne yazmaz. Singleton servisine ise istekler aynı anda **yazabilir**. Menü değil, duvara asılmış ortak bir not defteri düşün — herkes aynı anda karalarsa okunmaz hâle gelir. `DbContext`'in asla Singleton olmamasının sebebi tam olarak budur: içinde her istek için değişen bir değişiklik takibi (change tracking) durumu tutar.

### 3.5 Kargo bandı benzetmesiyle üç ömrün yan yana görünümü

> **Benzetme —** Bir kargo şubesini düşün. **Koli bandı** her paket için yeniden kesilir, bir kez kullanılır, atılır (Transient). **Sevk irsaliyesi** bir gönderi işlemi boyunca aynı kâğıttır; paketi tartan da, etiketi basan da, teslim alan da aynı irsaliyeye bakar (Scoped). **Bandın kendisi**, yani şubedeki taşıma bandı, gün boyu tek tanedir ve bütün paketler onun üstünden geçer (Singleton).

**Basitçe:** Aynı işlemde birden çok kişi aynı kâğıda bakması gerekiyorsa Scoped'tır. Kimse kimsenin kâğıdını görmeyecekse Transient'tır. Herkesin tek bir şeyi paylaşması gerekiyorsa Singleton'dır.

> **Bu benzetme şurada bozulur:** Kargo bandı bozulursa şube durur ve herkes fark eder. Singleton bir servis bozuk durum tuttuğunda uygulama durmaz — çalışmaya devam eder, sadece yanlış sonuç üretir. Örneğin Singleton bir servise "son işlem yapan kullanıcı"yı yazarsan, yoğun saatte A kullanıcısı B'nin verisini görebilir ve hiçbir hata kaydı oluşmaz. Bu tür hatalar test ortamında (tek kullanıcı) asla görünmez, sadece canlıda görünür.

### 3.6 Captive dependency tuzağı

> **Benzetme —** Apartman görevlisi (Singleton, bina boyunca aynı kişi) senin daire anahtarını (Scoped, sen oturduğun sürece geçerli) cebine koyup unutuyor. Sen taşınıyorsun, kilit değişiyor, ama görevli hâlâ o eski anahtarı taşıyor ve her seferinde onunla açmaya çalışıyor. Kapı açılmıyor, görevli "ama anahtar bende" diyor. Uzun ömürlü olanın kısa ömürlüyü tutması budur.

**Basitçe:** Uzun yaşayan bir nesne, kısa yaşayan bir nesneyi elinde tutarsa, kısa olanın süresi dolduktan sonra da onu kullanmaya devam eder. Nesne çoktan kapanmıştır. Sonuç: teşhisi zor, rastgele görünen hatalar.

**Teknik olarak:** Uzun ömürlü bir servis, kısa ömürlü bir bağımlılığı tutarsa ona **captive dependency** denir. Örnek: `Singleton` bir servise `Scoped` bir `DbContext` enjekte etmek. Context istek bitince bırakılması gerekirken singleton onu tutmaya devam eder; uygulama bir süre sonra teşhisi zor şekilde bozulur.

```csharp
// YANLIŞ — Singleton, Scoped bir DbContext'i esir alıyor
public class KurOnbellegiHatali
{
    private readonly ApplicationDbContext _context;   // Scoped!

    public KurOnbellegiHatali(ApplicationDbContext context) => _context = context;
    // İlk isteğin context'i sonsuza kadar burada kalır.
    // O istek bitince context Dispose edilir; sonraki çağrılarda
    // ObjectDisposedException ya da eski, bayat veri.
}
```

Doğru çözüm, context'i tutmak yerine **ihtiyaç anında kısa bir scope açmaktır**:

```csharp
// DOĞRU — Singleton, context'i tutmaz; gerektiğinde scope açar
public class KurOnbellegi
{
    private readonly IServiceScopeFactory _scopeFactory;

    public KurOnbellegi(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    public async Task<decimal> KurGetirAsync(string kod)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        return await context.Kurlar
            .Where(k => k.Kod == kod)
            .Select(k => k.Deger)
            .FirstOrDefaultAsync();
        // scope burada kapanır, context düzgünce Dispose edilir
    }
}
```

**Kural:** Bir servisin yaşam döngüsü, bağımlılıklarınınkinden **uzun olamaz**.

Bu kuralı tabloya dökersek:

| Servisin ömrü | Transient alabilir mi | Scoped alabilir mi | Singleton alabilir mi |
|---|---|---|---|
| **Transient** | Evet | Evet | Evet |
| **Scoped** | Evet | Evet | Evet |
| **Singleton** | Dikkat — nesne singleton gibi yaşar | **Hayır** | Evet |

Singleton'ın Transient alması da göründüğü kadar masum değildir: Transient nesne bir kez üretilir ve singleton onu bıraktığı için sonsuza kadar yaşar. Yani "transient" adı yanıltır, davranış singleton olur.

İyi haber: .NET geliştirme ortamında bu hatanın bir kısmını açılışta yakalar. `builder.Build()` çalışırken kapsam doğrulaması yapılır ve şuna benzer bir mesaj alırsın:

```
Cannot consume scoped service 'ApplicationDbContext'
from singleton 'KurOnbellegiHatali'.
```

Bu kontrol geliştirme ortamında varsayılan olarak açıktır. Üretim ortamında kapalı olabilir; hatayı orada görmek istemezsin.

---

## 4. Bölüm 2 — Uygulamanın İnşası

> **Benzetme —** Uçağın kapısının kapanma anı. O ana kadar bavul ekleyebilir, yolcu bindirebilirsin. Kapı kapandıktan sonra "bir valizim daha vardı" demenin faydası yoktur; uçak kalkış hattına girmiştir. `builder.Build()` o kapının kapandığı satırdır.

**Basitçe:** Tek satırlık bir sınır. Bu satırdan önce "neyim var" listesini doldurursun, sonra liste kilitlenir. Sonradan bir şey eklemen gerekiyorsa çare bu satırın üstüne çıkmaktır.

**Teknik olarak:**

```csharp
var app = builder.Build();
```

Tek satır ama bir sınır: **bu satırdan sonra `builder.Services`'e yeni servis eklenemez.** Konteyner kapanmıştır. Yeni bir servis eklemen gerekiyorsa bu satırın üstüne çıkarman gerekir.

Bu satır çalışırken iki iş birden olur. Birincisi, kayıt listesi dondurulur ve gerçek DI konteyneri üretilir. İkincisi, geliştirme ortamındaysan kapsam doğrulaması yapılır — bölüm 3.6'daki captive dependency hatası tam burada yakalanır.

`app` nesnesi üzerinden yine de servis **okuyabilirsin**; eklemek yasaktır, almak değil. Uygulama açılışında bir kereye mahsus iş yapmak için tipik kalıp şudur:

```csharp
var app = builder.Build();

// Açılışta veritabanı migration'larını uygula.
// Dikkat: burada kendi scope'umuzu açıyoruz, çünkü henüz
// bir HTTP isteği yok — yani ortada hazır bir scope da yok.
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    db.Database.Migrate();
}

app.UseHttpsRedirection();
```

> **Bu benzetme şurada bozulur:** Uçağın kapısı kapandığında uçak yine de kalkmayabilir, geri dönebilir. `builder.Build()` ise geri alınamaz; ikinci kez çağıramaz, konteyneri yeniden açamazsın. Ayrıca kapı benzetmesi "her şey bitti" izlenimi verir, oysa asıl iş bu satırdan sonra başlar: pipeline henüz kurulmadı bile.

---

## 5. Bölüm 3 — Middleware Pipeline

> **Benzetme —** Havalimanındaki kontrol noktaları. Terminale girersin, sırayla geçersin: güvenlik, check-in, pasaport, kapı. Her nokta sana bakar, kendi işini yapar ve "geç" der. Sonunda uçağa binersin (Controller). İnerken de aynı koridorlardan geri çıkarsın — ve her nokta dönüşte de sana bir şey yapabilir: çıkışta pasaportuna damga vurulur, bagajın teslim edilir. Yani her nokta seni **iki kez** görür: giderken ve dönerken.

**Basitçe:** Gelen her istek, senin sıraladığın katmanlardan tek tek geçer. Her katman ya işini yapıp isteği bir sonrakine yollar, ya da "burada dur" deyip geri gönderir. Controller bu zincirin en ucundadır, ortası değil.

**Teknik olarak:** **Middleware (ara katman)** — Gelen her HTTP isteğinin sırayla geçtiği katmanlardan her biri.

Pipeline'ı **soğan zarı** gibi düşün: istek en dıştan girer, yazdığın ara katmanlardan geçerek merkeze (Controller'a) ulaşır; Controller yanıtı ürettiğinde yanıt aynı katmanlardan **geriye doğru** çıkarak tarayıcıya ulaşır.

```
   HTTP İsteği
        │
        ▼
┌──────────────────────┐
│ ExceptionHandler     │  hata yakalama — en dışta olmalı
├──────────────────────┤
│ HttpsRedirection     │  http → https
├──────────────────────┤
│ StaticFiles          │  wwwroot: css, js, resim
├──────────────────────┤
│ Routing              │  hangi controller/action?
├──────────────────────┤
│ Authentication       │  sen kimsin?
├──────────────────────┤
│ Authorization        │  bu sayfaya yetkin var mı?
├──────────────────────┤
│ Endpoint (Controller)│  ◄── merkez
└──────────────────────┘
        │
        ▼
    HTTP Yanıtı (aynı yoldan geriye)
```

Tipik bir `Program.cs` pipeline bölümü:

```csharp
// Geliştirme ortamı değilse: hata detayını gizle, HTTPS'i zorunlu kıl
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();   // http isteğini https'e yönlendir
app.UseStaticFiles();        // wwwroot altındaki dosyaları sun
app.UseRouting();            // URL'i çözümle, hangi endpoint olduğunu belirle

app.UseAuthentication();     // kimlik doğrulama: sen kimsin?
app.UseAuthorization();      // yetkilendirme: bunu yapabilir misin?

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();                   // uygulamayı ayağa kaldır ve dinlemeye başla
```

Her katmanın iki şansı olduğunu akılda tutmak, pipeline'ı anlamanın anahtarıdır. Bir katman `next()` çağırmadan önce yazdığı kod **isteğe giderken**, sonra yazdığı kod **yanıt dönerken** çalışır. Üç katmanlık bir zincirin çalışma sırası şöyledir:

```
A girişi  →  B girişi  →  C girişi  →  [Controller]
                                            │
A çıkışı  ←  B çıkışı  ←  C çıkışı  ←───────┘
```

Yani ilk giren, son çıkar. `UseExceptionHandler`'ın en üstte olmasının sebebi budur: en dıştaki katman olduğu için, içeride nerede hata çıkarsa çıksın dönüş yolunda onu yakalayacak konumdadır. Zincirin ortasına koyarsan, kendisinden önceki katmanlarda oluşan hataları göremez.

Bir de yol ayrımı yapabilen biçimleri var. `app.Use` zincire bir halka ekler; `app.Run` zinciri **bitirir** (kendisinden sonrasına geçmez); `app.Map` belirli bir yol için ayrı bir dal açar:

```csharp
// Sadece /saglik adresine gelen istekler bu dala girer
app.Map("/saglik", dal =>
{
    dal.Run(async context => await context.Response.WriteAsync("Calisiyor"));
});
```

> **Bu benzetme şurada bozulur:** Havalimanında kontrol noktaları fiziksel olarak duvarla ayrılmıştır ve sıraları terminalin mimarisiyle sabittir; sen değiştiremezsin. Pipeline'da sırayı **tamamen sen** belirlersin ve yanlış sıralamak serbesttir — derleyici uyarmaz, uygulama açılır. İkincisi, havalimanında bir noktayı atlarsan güvenlik görevlisi seni durdurur. Pipeline'da bir katmanı hiç yazmazsan kimse durdurmaz; o kontrol basitçe **hiç yapılmaz** ve her şey sorunsuz görünür. Eksik güvenlik katmanının belirtisi hata mesajı değil, sessizliktir.

---

## 6. Sıra Neden Hayati

> **Benzetme —** Havalimanında görevli sana "VIP salona girebilirsin" demeden önce pasaportuna bakmak zorundadır. Sırayı ters çevir: görevli önce "bu kişi VIP mi" diye soruyor, ama elinde kimlik yok, kim olduğunu bilmiyor. Cevabı zorunlu olarak "hayır, tanımıyorum" olur. Kimseyi içeri almaz ya da herkesi alır — ikisi de yanlıştır ve ikisi de sessizce olur, çünkü görevli kendi işini kurallara göre yapmıştır. Hata görevlide değil, sıradadır.

**Basitçe:** Pipeline'da her katman, kendinden öncekilerin bıraktığı bilgiyle çalışır. Bir katmanı ihtiyaç duyduğu bilgiden önce koyarsan, o katman elinde boş bilgiyle karar verir. Kod çalışır, ekran açılır, hata mesajı yoktur — sadece sonuç yanlıştır. Bu yüzden sıra, kodun kendisi kadar önemlidir.

Somutlaştıralım: `UseAuthentication()` gelen istekteki çerezi ya da token'ı okur ve "bu istek Ayşe'ye ait" bilgisini `HttpContext.User` içine yazar. `UseAuthorization()` ise `HttpContext.User`'a bakıp "Ayşe bu sayfaya girebilir mi" diye sorar. İkincisini önce koyarsan, baktığı yer henüz **boştur**. Ayşe'yi anonim bir ziyaretçi sanır.

**Teknik olarak:** Pipeline'da **sıra kodun kendisi kadar önemlidir.** Dört klasik hata:

| Hata | Sonuç |
|---|---|
| `UseAuthorization()` **önce**, `UseAuthentication()` sonra | Yetkilendirme çalıştığında kullanıcının kim olduğu henüz belirlenmemiştir; herkes anonim görünür ve `[Authorize]` beklenmedik davranır |
| `UseRouting()` yazılmaması veya sonra yazılması | Hangi controller'a gidileceği bilinmeden yetkilendirme yapılamaz |
| `UseStaticFiles()` çok geç | Her CSS/JS isteği gereksiz yere tüm kimlik ve yetki katmanlarından geçer — boşuna iş |
| `UseExceptionHandler()` en dışta değil | Alt katmanlardaki hatalar yakalanamaz |

**Hatırlatıcı cümle:** *Önce kim olduğunu öğren (Authentication), sonra ne yapabileceğine karar ver (Authorization).* `UseAuthorization` **her zaman** `UseAuthentication`'dan sonra gelir.

Aynı mantığı bir cümlede toplayan sıralama şudur:

```
Hatayı yakala  →  HTTPS'e taşı  →  statik dosyayı hemen ver
               →  nereye gidiyor bul  →  kim olduğunu belirle
               →  yetkisi var mı bak  →  Controller'a ver
```

Her adım, bir öncekinin cevabına muhtaçtır. "Yetkisi var mı" sorusu "kim olduğunu belirle" adımına, o da "nereye gidiyor" adımına dayanır. Sırayı bozmak, cevabı olmayan bir soruyu sormaktır.

`UseStaticFiles()` maddesi biraz farklıdır; orada bir güvenlik açığı değil, **israf** vardır. Bir sayfa açıldığında tarayıcı yirmi otuz ayrı dosya ister: CSS, JS, ikonlar, resimler. Bunlar herkese açık dosyalardır. `UseStaticFiles()`'ı kimlik doğrulamadan önce koyarsan, bu istekler zincire girer girmez cevaplanır ve geri döner; kimlik ve yetki katmanlarını hiç meşgul etmez.

> Bir başka önemli nokta: `UseAuthorization()` tek başına hiçbir şeyi korumaz. O yalnızca `[Authorize]` özniteliklerini işleyen katmandır. `UseAuthentication()` yoksa ve controller'larda `[Authorize]` yoksa, uygulama tamamen açıktır.

Bu son cümle iki ayrı hatayı aynı anda anlatıyor, ayıralım:

```csharp
// 1. Pipeline'da katman var ama controller'da öznitelik yok:
//    sayfa herkese açıktır. Katman işleyecek bir kural bulamaz.
public class RaporController : Controller
{
    public IActionResult Gizli() => View();   // [Authorize] yok — açık
}

// 2. Controller'da öznitelik var ama pipeline'da UseAuthentication yok:
//    kullanıcı hiçbir zaman tanınmaz, herkes kapıda kalır.
[Authorize]
public class RaporController2 : Controller
{
    public IActionResult Gizli() => View();   // kimse giremez
}
```

İkisi de "çalışıyor gibi" görünür. Birincisinde sayfa açılır ve kimse şüphelenmez; ikincisinde herkes giriş sayfasına atılır ve "oturum açıyorum ama geri atıyor" diye saatlerce başka yerde hata aranır.

> **Bu benzetme şurada bozulur:** Havalimanında yanlış sırayla çalışan bir görevli fark edilir; yolcular şikâyet eder, kuyruk tıkanır. Yazılımda sıra hatasının **hiçbir belirtisi yoktur**. Uygulama açılır, sayfalar gelir, loglar temizdir. Yanlış sıranın bedeli ya bir güvenlik denetiminde ya da kötü bir günde ortaya çıkar. Bu yüzden pipeline sırası "test ederiz nasılsa" denecek bir şey değil, ezberlenecek bir şeydir.

---

## 7. Kendi Middleware'ini Yazmak

> **Benzetme —** Havalimanına kendi kontrol noktanı eklemek. Yolcu içeri girerken kronometreyi başlatırsın, çıkarken durdurup "bu yolcu terminalde 12 dakika geçirdi" diye deftere yazarsın. Tek nokta, iki an: giriş ve çıkış. İstersen o noktada "sen geçemezsin" deyip yolcuyu hiç içeri almazsın.

**Basitçe:** Pipeline'a kendi katmanını eklemek birkaç satırlık iştir. `next()` çağrısından önce yazdığın kod istek içeri girerken, sonrası yanıt dışarı çıkarken çalışır. `next()`'i hiç çağırmazsan istek orada kesilir.

**Teknik olarak:** Pipeline'a kendi katmanını eklemek mümkündür — örneğin her isteğin süresini loglamak için:

```csharp
app.Use(async (context, next) =>
{
    var baslangic = DateTime.UtcNow;

    await next();          // sonraki katmana geç (merkeze doğru)

    var sure = DateTime.UtcNow - baslangic;
    Console.WriteLine($"{context.Request.Path} → {sure.TotalMilliseconds} ms");
});
```

`await next()` satırının **öncesi** isteğe giderken, **sonrası** yanıt dönerken çalışır. Soğan zarı benzetmesinin koddaki karşılığı tam olarak budur.

`next()` çağrılmazsa pipeline orada kesilir ve istek Controller'a hiç ulaşmaz — bu bazen istenen davranıştır (örneğin bir IP engelleme katmanı).

```csharp
// Zinciri bilerek kesen bir katman
var yasakliIpler = new HashSet<string> { "203.0.113.7" };

app.Use(async (context, next) =>
{
    var ip = context.Connection.RemoteIpAddress?.ToString();

    if (ip is not null && yasakliIpler.Contains(ip))
    {
        context.Response.StatusCode = 403;
        await context.Response.WriteAsync("Erisim engellendi.");
        return;            // next() YOK — istek burada biter
    }

    await next();
});
```

Katman büyüdüğünde onu `Program.cs` içinde tutmak okunabilirliği bozar. Sınıf hâline getirmenin standart kalıbı şudur:

```csharp
public class IstekSuresiMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<IstekSuresiMiddleware> _logger;

    // RequestDelegate: zincirdeki bir sonraki katman.
    public IstekSuresiMiddleware(RequestDelegate next,
                                 ILogger<IstekSuresiMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sayac = System.Diagnostics.Stopwatch.StartNew();

        await _next(context);      // merkeze doğru

        sayac.Stop();
        _logger.LogInformation("{Yol} {Kod} — {Ms} ms",
            context.Request.Path,
            context.Response.StatusCode,
            sayac.ElapsedMilliseconds);
    }
}
```

```csharp
// Pipeline'a eklemek — sıra yine senin elinde
app.UseMiddleware<IstekSuresiMiddleware>();
```

Bir uyarı: middleware sınıfları uygulama ömrü boyunca **tek örnek** olarak yaşar, yani davranışı Singleton gibidir. Bu yüzden constructor'ına `Scoped` bir servis (`DbContext` gibi) enjekte etmek, bölüm 3.6'daki captive dependency hatasının ta kendisidir. Scoped servise ihtiyacın varsa constructor'dan değil, `InvokeAsync` parametresinden istersin:

```csharp
// DOĞRU — Scoped servis metot parametresi olarak istenir,
// çünkü InvokeAsync her istekte, o isteğin scope'u içinde çalışır.
public async Task InvokeAsync(HttpContext context, ApplicationDbContext db)
{
    await _next(context);
}
```

Yanıt gövdesi yazılmaya başlandıktan sonra başlık (header) ekleyemeyeceğini de unutma. Dönüş yolunda başlık eklemek istiyorsan, bunu `next()` çağrısından **önce** kaydedersin:

```csharp
app.Use(async (context, next) =>
{
    // Yanıt gövdesi yazılmadan hemen önce çalışacak işi kaydet
    context.Response.OnStarting(() =>
    {
        context.Response.Headers["X-Islem-Suresi"] = "olculdu";
        return Task.CompletedTask;
    });

    await next();
});
```

> **Bu benzetme şurada bozulur:** Havalimanında kendi noktanı kursan bile yolcu fiziksel olarak oradan geçmek zorundadır. Middleware'de ise bir üst katman `next()` çağırmazsa senin katmanın **hiç çalışmaz** — kurduğun nokta boş bir koridorda kalır. Ayrıca kronometre benzetmesi tek bir yolcuyu ima eder; gerçekte aynı anda yüzlerce istek aynı middleware nesnesinden geçer. Bu yüzden middleware sınıfının içinde alan (field) olarak durum tutamazsın: bir isteğin verisi diğerininkine karışır. İsteğe ait her şey `HttpContext` içinde taşınır.

---

## Tek Bakışta Özet

- `Program.cs` üç bölümdür: **servis kaydı → build → pipeline**. `builder.Build()` bir sınırdır, sonrasında servis eklenemez. Hangi bölümdesin anlamak için değişkene bak: `builder` kurulum, `app` akış.
- **DI**, bağımlılıkları `new` ile üretmek yerine dışarıdan almaktır; veritabanı değiştirmeyi ve test yazmayı mümkün kılar. Kayıt yoksa hata açılışta değil, servis ilk istendiğinde çıkar.
- **Transient** her seferinde yeni, **Scoped** istek başına bir tane, **Singleton** uygulama boyunca tek.
- **Scope = tek bir HTTP isteği.** Kullanıcı ya da oturum değil. Sayfayı üç kez yenilemek üç scope demektir.
- `DbContext` ve Unit of Work **her zaman Scoped**'tır — veri tutarlılığı buna bağlıdır. Singleton bir `DbContext` hem thread güvenli değildir hem de bayat veri tutar.
- Bir servisin ömrü, bağımlılıklarınınkinden uzun olamaz (**captive dependency**). Singleton bir servisin Scoped'a ihtiyacı varsa `IServiceScopeFactory` ile kısa bir scope açar.
- **Middleware pipeline** soğan zarıdır: istek içeri, yanıt dışarı doğru aynı katmanlardan geçer. Her katman isteği **iki kez** görür: `next()` öncesi ve sonrası.
- **Sıra hayatidir:** her katman kendinden öncekinin bıraktığı bilgiyle çalışır. `UseAuthentication` her zaman `UseAuthorization`'dan önce. Yanlış sıra hata vermez, sessizce yanlış davranır.
- `UseStaticFiles()` erken olmalı — güvenlik için değil, gereksiz iş yapmamak için.
- `UseAuthorization()` tek başına koruma sağlamaz; `[Authorize]` olmadan hiçbir şey kapalı değildir.
- Kendi middleware sınıfın Singleton gibi yaşar: içinde durum tutma, Scoped servisi constructor'dan değil `InvokeAsync` parametresinden iste.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| `WebApplicationBuilder` | Uygulamanın kurulum aşamasını yöneten nesne |
| DI konteyneri | Servisleri kaydedip ihtiyaç duyulduğunda üreten altyapı |
| Dependency Injection | Bağımlılığın `new` ile değil dışarıdan sağlanması ilkesi |
| Constructor injection | Bağımlılığın constructor parametresi olarak istenmesi |
| `AddTransient` | Her istendiğinde yeni nesne üreten kayıt |
| `AddScoped` | HTTP isteği başına tek nesne üreten kayıt |
| `AddSingleton` | Uygulama boyunca tek nesne üreten kayıt |
| Scope (kapsam) | Bir HTTP isteğinin başından yanıt dönene kadarki süre |
| `IServiceScopeFactory` | Elle scope açmayı sağlayan servis; singleton içinden scoped kullanmanın doğru yolu |
| Captive dependency | Uzun ömürlü servisin kısa ömürlü bağımlılığı tutması hatası |
| Kapsam doğrulama | `builder.Build()` sırasında captive dependency'yi yakalayan geliştirme kontrolü |
| `ObjectDisposedException` | Kapatılmış (dispose edilmiş) bir nesneyi kullanmaya çalışınca alınan istisna |
| Thread güvenliği | Aynı nesnenin birden çok istek tarafından aynı anda güvenle kullanılabilmesi |
| Middleware | İsteğin sırayla geçtiği pipeline katmanı |
| Pipeline | Middleware'lerin oluşturduğu, iki yönlü çalışan zincir |
| `next()` | Bir sonraki middleware'e devretme çağrısı |
| `RequestDelegate` | Zincirdeki bir sonraki katmanı temsil eden temsilci (delegate) tipi |
| `HttpContext` | Bir isteğe ait tüm bilginin (istek, yanıt, kullanıcı) taşındığı nesne |
| `app.Use` / `app.Run` / `app.Map` | Zincire halka ekler / zinciri bitirir / belirli yol için dal açar |
| Authentication | Kimlik doğrulama — "sen kimsin?" |
| Authorization | Yetkilendirme — "bunu yapabilir misin?" |
| HSTS | Tarayıcıya siteye hep HTTPS ile gelmesini söyleyen başlık |
| `builder.Configuration` | appsettings, ortam değişkenleri ve secrets'ı birleştiren konfigürasyon |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`UseAuthorization()` sayfayı korur" | Yalnızca `[Authorize]` özniteliklerini işler; öznitelik yoksa hiçbir şey kapalı değildir |
| "Middleware sırası önemli değil, hepsi çalışıyor" | Sıra davranışı belirler; yanlış sıra sessiz güvenlik açığı üretir |
| "Singleton hep daha performanslıdır" | `DbContext` gibi thread güvenli olmayan nesnelerde veri bozulmasına yol açar |
| "Scoped ile Singleton arasında pratik fark yok" | Scoped istek başına izole, Singleton tüm kullanıcılarca paylaşılır |
| "`builder.Build()` sonrası servis eklenebilir" | Eklenemez; konteyner o noktada kapanır |
| "Transient en güvenli seçim, hep onu kullan" | Gereksiz nesne üretimi ve `DbContext` için veri tutarsızlığı demektir |
| "Scope bir kullanıcı oturumudur" | Scope tek bir HTTP isteğidir; aynı kullanıcının her isteği ayrı scope'tur |
| "Middleware Controller'ın içinde çalışır" | Controller pipeline'ın **sonundadır**; middleware ondan önce ve sonra çalışır |
| "Singleton'a Transient vermek güvenlidir" | O Transient nesne singleton'a bağlandığı için uygulama boyunca yaşar; adı yanıltır |
| "Middleware sınıfına `DbContext` enjekte edilir" | Constructor'a değil; `InvokeAsync` parametresine istenir |
| "Hata varsa `UseExceptionHandler` nerede olursa yakalar" | Yalnızca kendisinden **sonra** gelen katmanların hatalarını yakalar; en dışta olmalıdır |

---

## Sonraki

→ `03-Routing-ve-Model-Binding.md`
