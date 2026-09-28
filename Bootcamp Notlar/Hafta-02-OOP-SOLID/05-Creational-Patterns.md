# Hafta 2 · Cuma — Yaratımsal Tasarım Kalıpları

**Okuma süresi:** ~50 dk
**Neden bu konu:** Yaratımsal kalıplar, "bu nesneyi kim, nerede, nasıl üretecek?" sorusunun cevaplarıdır. Bootcamp'te göreceğin her DI konteyneri, her `IServiceProvider` çağrısı, her `HttpClientFactory` kullanımı aslında bu kalıpların .NET'e gömülmüş hâlidir. Kalıpları bilmeden kullanırsın, bilerek kullanınca nerede durman gerektiğini de bilirsin. Özellikle Singleton, en çok bilinen ve en çok yanlış kullanılan kalıptır; bu not onu doğru yerine oturtur.

---

## Önce Basitçe

Yazılım yazarken aynı problemleri tekrar tekrar görürsün. "Bu sınıfın uygulamada tek bir örneği olsun", "hangi sınıfı üreteceğime çalışma anında karar vereyim", "bu nesnenin on beş parametresi var, constructor'ı okunmaz hâle geldi". Bu problemler sana özel değil. Dünyanın her yerindeki geliştiriciler aynı duvarlara çarptı ve zamanla her birinin bilinen, denenmiş çözümleri oluştu. Tasarım kalıbı dediğimiz şey bu: tekrar eden bir probleme verilen, adı konmuş çözüm.

Buradaki kritik kelime "adı konmuş". Kalıpların asıl değeri kodun kendisinde değil, isminde. Bir toplantıda "burada bir factory kullanalım" dediğinde karşındaki on dakikalık açıklamayı dinlemeden ne demek istediğini anlar. Kalıplar, geliştiriciler arasındaki ortak dildir.

Yaratımsal kalıplar bu ortak dilin ilk bölümü. Hepsi tek bir soruyla uğraşır: nesne nasıl doğacak? Normalde `new Musteri()` yazarsın ve iş biter. Ama bazı durumlarda `new` yazmak seni köşeye sıkıştırır. Hangi sınıfı yaratacağın çalışma anında belli oluyorsa, nesnenin kurulumu karmaşıksa, ya da o nesneden uygulamada sadece bir tane olması gerekiyorsa, düz bir `new` yetmez. Yaratımsal kalıplar bu "yetmez" durumlarının her birine bir cevap verir.

Son olarak şunu bil: .NET'i her gün kullanırken bu kalıpların çoğunu zaten kullanıyorsun. `StringBuilder` bir Builder'dır. `IServiceProvider` bir Factory'dir. `ILoggerFactory` adını bile saklamıyor. Notun sonunda bu yerleşik karşılıkları tek tek göreceksin; o bölüm kalıpların soyut değil, elinin altında olduğunu gösterecek. Şimdi detaya iniyoruz.

> **Ana benzetme:** Tasarım kalıpları, usta bir marangozun kafasındaki **birleşim teknikleridir**. Zıvana, lamba-zıvana, kırlangıç kuyruğu... Marangoz her masa yaptığında bunları sıfırdan icat etmez; hangi yükte hangi birleşimin tutacağını bilir ve ismiyle söyler. Çırağına "burayı kırlangıç kuyruğu yap" der, çırak anlar. Ama aynı marangoz, iki tahtayı birleştirmek için kırlangıç kuyruğu açmaz — oraya bir vida yeter. Kalıp bilmek, tekniği bilmek kadar nerede kullanılmayacağını bilmektir.

---

## Bu Notta Ne Var

1. Tasarım kalıbı nedir, ne değildir
2. GoF sınıflandırması: üç aile
3. Singleton — klasik uygulama ve thread-safety
4. Singleton neden çoğu zaman anti-pattern
5. Factory Method
6. Abstract Factory ve ürün ailesi
7. Builder ve çok parametreli constructor problemi
8. Prototype — kopyalama, shallow ve deep copy
9. .NET'te bu kalıpların yerleşik karşılıkları
10. Hangi problemde hangi yaratımsal kalıp

---

## 1. Tasarım Kalıbı Nedir, Ne Değildir

> **Benzetme —** Bir terziye gittiğinde "bana ceket dikin" dersin, terzi sana "klasik kesim mi, blazer mi, kruvaze mi?" diye sorar. Bunlar hazır ceketler değildir; dikim yaklaşımlarının isimleridir. Terzi kruvazeyi biliyorsa her kruvazeyi sıfırdan tasarlamaz, bilinen yaklaşımı senin ölçülerine uygular. Kalıp da böyledir: hazır ürün değil, bilinen bir yaklaşımın adıdır.

**Basitçe:** Tasarım kalıbı, sık karşılaşılan bir tasarım problemine verilen, üzerinde uzlaşılmış ve isimlendirilmiş bir çözüm şeklidir. Kopyalayıp yapıştıracağın bir kod parçası değildir; kendi kodunda uygulayacağın bir fikirdir.

**Teknik olarak:** **Design pattern (tasarım kalıbı)** — Belirli bir bağlamda tekrar eden bir tasarım problemine uygulanabilecek, genel ve yeniden kullanılabilir çözüm şablonu.

### Kalıp ne DEĞİLDİR

**Hazır kod kütüphanesi değildir.** NuGet'ten "Singleton paketi" indiremezsin. Kalıp, senin sınıflarını nasıl düzenleyeceğinin tarifidir; aynı kalıbın iki projedeki kodu birbirine benzemeyebilir.

**Algoritma değildir.** Algoritma bir hesabın adım adım nasıl yapılacağını söyler; kalıp ise sınıfların birbirine nasıl bakacağını söyler. Quicksort bir algoritmadır, Strategy bir kalıptır.

**Bu benzetme şurada bozulur:** Terzi örneğinde yaklaşımı seçen terzidir ve müşteri sonucu görür. Yazılımda ise kalıbı seçen sen, sonucu görense seni takip eden geliştiricidir — çoğu zaman altı ay sonraki sen. Terzi yanlış kesimde kumaşı bozar; sen yanlış kalıpta on dosyayı birbirine bağlarsın ve bunu ancak değiştirmen gerektiğinde fark edersin.

---

## 2. GoF Sınıflandırması: Üç Aile

> **Benzetme —** Bir hastanede üç ana bölüm vardır: doğumhane, ameliyathane düzeni ve poliklinik işleyişi. Doğumhanede yeni hayat başlar, ameliyathanede parçalar bir araya getirilir, poliklinikte ise günlük akış yönetilir. Tasarım kalıpları da tam olarak bu üç iş için ayrılmıştır: nesne doğurmak, nesneleri birleştirmek, nesneler arası akışı yönetmek.

**Basitçe:** Klasik 23 tasarım kalıbı üç gruba ayrılır. Yaratımsal kalıplar nesne üretimiyle, yapısal kalıplar nesnelerin nasıl birleştirileceğiyle, davranışsal kalıplar nesnelerin nasıl haberleşeceğiyle ilgilenir.

**Teknik olarak:** **GoF (Gang of Four)** — 1994'te *Design Patterns: Elements of Reusable Object-Oriented Software* kitabını yazan dört yazara verilen ad. Kitap 23 kalıbı üç kategoride tanımlar; bugün "klasik kalıplar" denince bunlar anlaşılır.

| Aile | Soru | Kalıplar |
|---|---|---|
| **Creational (Yaratımsal)** | Nesne nasıl üretilecek? | Singleton, Factory Method, Abstract Factory, Builder, Prototype |
| **Structural (Yapısal)** | Nesneler nasıl birleştirilecek? | Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight |
| **Behavioral (Davranışsal)** | Nesneler nasıl konuşacak? | Strategy, Observer, Command, Template Method, Chain of Responsibility, Iterator, State, Mediator, Visitor, Memento, Interpreter |

Bu not yaratımsal aileyi anlatıyor. Yapısal ve davranışsal aile yarınki notta.

**Bu benzetme şurada bozulur:** Hastanede bölümler fiziksel olarak ayrıdır; bir hasta aynı anda iki yerde olamaz. Kalıplarda ise sınırlar geçişkendir. Abstract Factory yaratımsaldır ama ürettiği nesneler yapısal bir ilişki kurar. MediatR kütüphanesi hem Mediator hem Command kalıbını taşır. Aile ayrımı düşünmeyi kolaylaştırmak içindir, keskin bir sınır değildir.

---

## 3. Singleton — Tek Örnek Garantisi

> **Benzetme —** Bir ilçede tek bir tapu dairesi vardır. İkinci bir tapu dairesi açılamaz, çünkü tapu kayıtları tek merkezde tutulmak zorundadır. İki daire olsaydı aynı parsel iki kişiye satılabilirdi. Herkes aynı kapıya gider, aynı defteri kullanır. Singleton bunu yapar: bu sınıftan ikinci bir örnek üretilemez, isteyen herkes aynı örneği alır.

**Basitçe:** Singleton, bir sınıftan uygulama boyunca yalnızca tek bir nesne oluşmasını garanti eder ve o nesneye herkesin ulaşabileceği bir kapı açar.

**Teknik olarak:** **Singleton (tekil)** — Sınıfın yalnızca bir örneğinin olmasını sağlayan ve bu örneğe global erişim noktası sunan yaratımsal kalıp. İki iddiası vardır: *tek örnek* ve *global erişim*. İkinci iddia, kalıbın başını derde sokan taraftır.

### Klasik uygulama (thread-safe değil)

```csharp
public class Ayarlar
{
    private static Ayarlar _ornek;

    // constructor private — dışarıdan new yapılamaz
    private Ayarlar()
    {
        BaglantiMetni = "Server=.;Database=MvcCv;Trusted_Connection=True;";
    }

    public string BaglantiMetni { get; }

    public static Ayarlar Ornek
    {
        get
        {
            if (_ornek == null)          // iki iş parçacığı aynı anda buraya girerse
                _ornek = new Ayarlar();  // iki ayrı nesne üretilir
            return _ornek;
        }
    }
}
```

Constructor'ın `private` olması kalıbın kalbidir. Dışarıdan `new Ayarlar()` yazılamaz, tek yol `Ayarlar.Ornek` özelliğidir.

Bu kod tek iş parçacıklı bir konsol uygulamasında çalışır. Bir web uygulamasında çalışmaz. İki istek aynı anda `if (_ornek == null)` satırına girerse ikisi de `null` görür, ikisi de `new` yapar. "Tek örnek" garantisi daha ilk saniyede kırılır.

### Kilitli uygulama (double-checked locking)

```csharp
public class Ayarlar
{
    private static Ayarlar _ornek;
    private static readonly object _kilit = new object();

    private Ayarlar() { }

    public static Ayarlar Ornek
    {
        get
        {
            if (_ornek == null)                 // 1. kontrol: kilit almadan hızlı çıkış
            {
                lock (_kilit)
                {
                    if (_ornek == null)         // 2. kontrol: kilidi bekleyen için
                        _ornek = new Ayarlar();
                }
            }
            return _ornek;
        }
    }
}
```

Buna **double-checked locking (çift kontrollü kilitleme)** denir. Dıştaki `if` performans içindir: nesne bir kez üretildikten sonra hiçbir çağrı kilit almaz. İçteki `if` doğruluk içindir: kilidi bekleyen ikinci iş parçacığı, kilit serbest kaldığında nesnenin artık üretilmiş olduğunu görür.

Bu kod doğrudur ama kimse artık böyle yazmaz. Sebebi bir sonraki başlık.

### Doğru yol: `Lazy<T>`

```csharp
public sealed class Ayarlar
{
    private static readonly Lazy<Ayarlar> _lazy =
        new Lazy<Ayarlar>(() => new Ayarlar());

    private Ayarlar()
    {
        BaglantiMetni = "Server=.;Database=MvcCv;Trusted_Connection=True;";
    }

    public static Ayarlar Ornek => _lazy.Value;

    public string BaglantiMetni { get; }
}
```

`Lazy<T>`, .NET'in gecikmeli ve iş parçacığı güvenli başlatma tipidir. Varsayılan modu `LazyThreadSafetyMode.ExecutionAndPublication`'dır: fabrika metodu yalnızca bir kez çalışır, diğer iş parçacıkları bekler. Kilidi sen yazmazsın, hata yapma ihtimalin kalmaz. `sealed` de kasıtlıdır: kimse bu sınıftan türetip tek örnek garantisini delemez.

### Daha da basit yol: `static readonly` alan

```csharp
public sealed class Ayarlar
{
    public static readonly Ayarlar Ornek = new Ayarlar();
    private Ayarlar() { }
}
```

| Yöntem | Thread-safe | Gerçekten tembel | Kod miktarı |
|---|---|---|---|
| Düz `if (_ornek == null)` | Hayır | Evet | Az |
| Double-checked locking | Evet | Evet | Çok |
| `Lazy<T>` | Evet | Evet | Az |
| `static readonly` alan | Evet | Kısmen | En az |

> **Performans notu:** Singleton nesnesi uygulama boyunca yaşar. İçinde tuttuğun her şey de yaşar. Singleton içinde büyüyen bir `List<T>` ya da `Dictionary<K,V>` tutuyorsan, bu bellek hiçbir zaman geri verilmez. Sınırsız büyüyen bir önbellek, en sık görülen bellek sızıntısı sebeplerinden biridir; `MemoryCache` gibi sınırı ve süre politikası olan yapılar kullan.

**Bu benzetme şurada bozulur:** Tapu dairesi benzetmesi "tek olmak zorunda" fikrini iyi anlatır ama yanıltıcı bir taraf var. Tapu dairesi tektir çünkü *gerçekten* öyle olmak zorundadır — kayıt bütünlüğü bunu gerektirir. Kodda ise çoğu Singleton "tek olmak zorunda olduğu için" değil, "her yerden kolayca erişebileyim diye" yazılır. Yani gerçek sebep kolaylık, iddia edilen sebep bütünlük. Bir sonraki bölüm tam olarak bu farkı anlatıyor.

---

## 4. Singleton Neden Çoğu Zaman Anti-Pattern

> **Benzetme —** Apartmanın girişine, herkesin kullanabileceği ortak bir defter koyduğunu düşün. Başta pratiktir: kim ne ödemiş, hepsi orada. Sonra biri sayfayı yırtar, biri kurşun kalemle yazıp siler, biri gece yarısı girip rakam değiştirir. Kimin ne zaman ne yazdığı belli değildir. Defterin tek olması sorun değil; **kilitsiz ve sahipsiz** olması sorundur. Singleton'ın kötü şöhreti buradan gelir.

**Basitçe:** Singleton'ın "tek örnek" tarafı genelde masumdur. Zararlı olan "global erişim" tarafıdır. Kodun her yerinden görünen, kim tarafından kullanıldığı belli olmayan bir nesne, bağımlılıkları gizler ve testi imkânsızlaştırır.

**Teknik olarak:** Singleton üç ayrı SOLID ilkesini aynı anda zorlar.

**Single Responsibility ihlali:** Sınıf hem kendi işini yapar, hem de kendi yaşam döngüsünü yönetir. İki sorumluluk, tek sınıf.

**Dependency Inversion ihlali:** Kullanan kod `KurServisi.Ornek` yazdığında somut sınıfa bağımlı olur. Arayüze değil, sınıfa. Değiştirmek için kullanan her satırı bulman gerekir.

**Open/Closed ihlali:** Davranışı değiştirmek için sınıfın kendisini açmak zorundasın; dışarıdan farklı bir uygulama veremezsin.

### Gizli bağımlılık problemi

```csharp
// KÖTÜ — bu sınıfın neye ihtiyacı olduğu imzasından anlaşılmaz
public class SiparisServisi
{
    public void Olustur(Siparis s)
    {
        var kur    = KurServisi.Ornek.Cevir(s.Tutar, "USD");
        var ayar   = Ayarlar.Ornek.BaglantiMetni;
        Loglayici.Ornek.Yaz($"Sipariş {s.Id}");
        // ...
    }
}
```

Bu sınıfın constructor'ına bakan biri "hiçbir bağımlılığı yok" sanır. Oysa üç ayrı servise bağımlıdır ve bunlar metot gövdesine gizlenmiştir. Bağımlılıklar imzada görünmüyorsa, sınıfın maliyetini okumadan anlayamazsın.

```csharp
// İYİ — bağımlılıklar constructor'da, görünür ve değiştirilebilir
public class SiparisServisi
{
    private readonly IKurServisi _kur;
    private readonly ILogger<SiparisServisi> _logger;

    public SiparisServisi(IKurServisi kur, ILogger<SiparisServisi> logger)
    {
        _kur = kur;
        _logger = logger;
    }

    public void Olustur(Siparis s)
    {
        var tutar = _kur.Cevir(s.Tutar, "USD");
        _logger.LogInformation("Sipariş {Id}", s.Id);
    }
}
```

İkinci sürümde constructor sınıfın ihtiyaç listesidir. Testte sahte bir `IKurServisi` verirsin, iş biter.

### Test edilebilirlik sorunu

Singleton'ı testte değiştiremezsin. Değiştiremediğin için de gerçek nesneyi kullanmak zorunda kalırsın.

```csharp
// Bu testi yazmanın kolay bir yolu yok
[Fact]
public void Doviz_Cevrimi_Dogru_Hesaplanir()
{
    var servis = new SiparisServisi();
    servis.Olustur(new Siparis { Tutar = 100 });
    // KurServisi.Ornek gerçek servistir. Ağ çağrısı yapar.
    // Kuru sabitleyemezsin, sonucu doğrulayamazsın.
}
```

Daha kötüsü, Singleton **durum taşıdığı** için testler birbirini etkiler. Bir test Singleton'ın içindeki sayacı artırır, diğer test o kirli durumu miras alır. Testler tek tek çalışınca geçer, hep birlikte çalışınca rastgele patlar. Bu, ekip içinde en çok zaman yakan hata sınıfıdır.

### DI konteynerindeki `AddSingleton` ile farkı

Burası çok karıştırılır. `AddSingleton`, GoF'un Singleton kalıbı **değildir**.

```csharp
// Program.cs
builder.Services.AddSingleton<IKurServisi, KurServisi>();
```

| | GoF Singleton | `AddSingleton` |
|---|---|---|
| Tekliği kim sağlar | Sınıfın kendisi | DI konteyneri |
| Constructor | `private` | `public` |
| Erişim | `KurServisi.Ornek` (global) | Constructor enjeksiyonu |
| Arayüzle kullanım | Zorlanır | Doğal (`IKurServisi`) |
| Testte değiştirme | Neredeyse imkânsız | Kayıt değiştirilir, biter |
| Bağımlılık görünürlüğü | Gizli | İmzada açık |

`AddSingleton` ile kaydedilen sınıf, sıradan bir sınıftır. `public` constructor'ı vardır, arayüzü uygular, testte `new KurServisi()` diyerek doğrudan kurabilirsin. Tekil olması sınıfın kendi kararı değil, **konteynerin yaşam süresi politikasıdır**. Yarın `AddScoped` yaparsan sınıfın tek satırı değişmez.

Pratik kural şudur: .NET 6 sonrası bir uygulamada "bundan tek tane olsun" ihtiyacın varsa cevap `AddSingleton`'dır, GoF Singleton değil.

> **Kritik tuzak:** `AddSingleton` ile kaydettiğin sınıfın içine `AddScoped` bir servis enjekte edemezsin. Buna **captive dependency (tutsak bağımlılık)** denir: singleton nesne, scoped nesneyi ilk isteğin ömrü boyunca değil, uygulama ömrü boyunca tutar. `DbContext` scoped'dur; bir singleton'a enjekte edersen ilk isteğin `DbContext`'i sonsuza kadar yaşar ve ikinci istekte bozuk davranır. .NET, `ValidateScopes` açıkken (Development ortamında varsayılan) bunu başlangıçta hata olarak verir. Singleton'ın scoped bir servise ihtiyacı varsa `IServiceScopeFactory` enjekte edip her kullanımda kendi scope'unu açmalısın.

### Ne zaman KULLANMA

- DI konteyneri varken. "Tek tane olsun" ihtiyacının cevabı `AddSingleton`'dır.
- Nesne **durum tutuyorsa**. Değişen ortak durum, testleri birbirine bağlar ve eşzamanlılık hatası üretir.
- Sırf her yerden kolay erişmek için. Kolaylık, gizli bağımlılığın bedelini ödemeye değmez.

Elle yazılmış Singleton yalnızca şu üçü birlikte sağlanıyorsa kabul edilebilir: nesne durumsuz ya da değişmez, tek olması gerçekten zorunlu, ve DI konteynerine erişilemiyor (çok erken başlangıç kodu gibi).

**Bu benzetme şurada bozulur:** Apartman defteri benzetmesinde kötülüğün kaynağı "ortak olması" gibi görünür. Oysa ortaklık sorun değildir; DI konteynerinin singleton'ı da ortaktır ve sorun çıkarmaz. Fark şudur: konteyner nesnesinde kimin kullandığı constructor'lardan izlenebilir, ömrü tek yerden yönetilir ve testte değiştirilebilir. Defter kötüdür çünkü **sahipsizdir**, ortak olduğu için değil.

---

## 5. Factory Method — Üretimi Alt Sınıfa Devretmek

> **Benzetme —** Nöbetçi eczane sistemini düşün. Sen "bana nöbetçi eczaneyi söyle" dersin; hangi eczanenin nöbetçi olduğunu sen bilmezsin, bilmek de zorunda değilsin. Takvimi tutan taraf karar verir, sana bir eczane adresi döner. Sen adresle işini görürsün. Factory Method budur: "bana uygun olanı ver" dersin, hangisinin uygun olduğuna üretici taraf karar verir.

**Basitçe:** Factory Method, nesne üretme işini `new` yazmak yerine bir metoda taşır. Böylece hangi somut sınıfın üretileceğine karar veren yer tek noktada toplanır ve kullanan kod o karardan habersiz kalır.

**Teknik olarak:** **Factory Method (fabrika metodu)** — Nesne üretimi için bir arayüz tanımlar, hangi sınıfın örnekleneceğine alt sınıfların (veya bir üretici bileşenin) karar vermesini sağlar. Amaç, kullanan kodu somut tiplerden ayırmaktır.

### Problem: dağılmış `new` çağrıları

```csharp
// KÖTÜ — üretim kararı her kullanım yerine dağılmış
public class BildirimServisi
{
    public void Gonder(string kanal, string mesaj, string hedef)
    {
        if (kanal == "email")
        {
            var g = new EmailGonderici("smtp.sirket.com", 587);
            g.Gonder(hedef, mesaj);
        }
        else if (kanal == "sms")
        {
            var g = new SmsGonderici("api-key-123");
            g.Gonder(hedef, mesaj);
        }
        else if (kanal == "push")
        {
            var g = new PushGonderici();
            g.Gonder(hedef, mesaj);
        }
    }
}
```

Yeni bir kanal geldiğinde bu metodu açman gerekir. SMTP portu değiştiğinde aynı `new` çağrısını kaç dosyada yazdıysan hepsini bulman gerekir. Üretim bilgisi ile kullanım bilgisi iç içe geçmiştir.

### Çözüm: üretimi bir fabrikaya al

```csharp
public interface IBildirimGonderici
{
    Task GonderAsync(string hedef, string mesaj);
}

public class EmailGonderici : IBildirimGonderici
{
    private readonly string _sunucu;
    public EmailGonderici(string sunucu) => _sunucu = sunucu;
    public Task GonderAsync(string hedef, string mesaj) => Task.CompletedTask;
}

public class SmsGonderici : IBildirimGonderici
{
    private readonly string _apiAnahtari;
    public SmsGonderici(string apiAnahtari) => _apiAnahtari = apiAnahtari;
    public Task GonderAsync(string hedef, string mesaj) => Task.CompletedTask;
}

public interface IBildirimFabrikasi
{
    IBildirimGonderici Olustur(string kanal);
}

public class BildirimFabrikasi : IBildirimFabrikasi
{
    private readonly IConfiguration _config;
    public BildirimFabrikasi(IConfiguration config) => _config = config;

    public IBildirimGonderici Olustur(string kanal) => kanal switch
    {
        "email" => new EmailGonderici(_config["Smtp:Sunucu"]),
        "sms"   => new SmsGonderici(_config["Sms:ApiKey"]),
        "push"  => new PushGonderici(),
        _ => throw new NotSupportedException($"Bilinmeyen kanal: {kanal}")
    };
}
```

Kullanan taraf artık somut sınıfları görmez:

```csharp
public class BildirimServisi
{
    private readonly IBildirimFabrikasi _fabrika;
    public BildirimServisi(IBildirimFabrikasi fabrika) => _fabrika = fabrika;

    public Task GonderAsync(string kanal, string hedef, string mesaj)
        => _fabrika.Olustur(kanal).GonderAsync(hedef, mesaj);
}
```

`switch` kaybolmadı, tek bir yere taşındı. Bu kalıbın vaadi de zaten budur: kararı yok etmek değil, **tek noktaya toplamak**.

### .NET'te pratik yol: DI ile anahtarlı seçim

.NET 8 ile gelen `keyed services`, fabrika yazmadan aynı işi yapar:

```csharp
// Program.cs
builder.Services.AddKeyedScoped<IBildirimGonderici, EmailGonderici>("email");
builder.Services.AddKeyedScoped<IBildirimGonderici, SmsGonderici>("sms");

// Kullanım
public class BildirimServisi
{
    private readonly IServiceProvider _sp;
    public BildirimServisi(IServiceProvider sp) => _sp = sp;

    public Task GonderAsync(string kanal, string hedef, string mesaj)
    {
        var gonderici = _sp.GetRequiredKeyedService<IBildirimGonderici>(kanal);
        return gonderici.GonderAsync(hedef, mesaj);
    }
}
```

### Ne zaman KULLANMA

- Tek bir somut sınıf varsa ve yenisi gelmeyecekse. `IKullaniciFabrikasi` yazıp içinde tek `new Kullanici()` döndürmek katıksız fazlalıktır.
- Sınıfın kurulumu basitse ve DI konteyneri zaten üretebiliyorsa. Konteynerin kendisi bir fabrikadır; onun üstüne ikinci bir fabrika yazma.
- Karar, üretim anında değil kullanım anında değişiyorsa. O zaman aradığın Factory değil **Strategy** kalıbıdır.

**Bu benzetme şurada bozulur:** Nöbetçi eczane örneğinde sana dönen eczane bir "ürün" değil, bir adres. Factory Method'da ise dönen şey gerçekten yeni üretilmiş bir nesnedir ve ömrü senin elindedir. Eczaneyi kapatmak sana düşmez; fabrikanın döndürdüğü `IDisposable` bir nesneyi kapatmak sana düşer. Fabrikadan `HttpClient` veya `DbConnection` gibi kaynak tutan bir şey alıyorsan, o kaynağı kimin dispose edeceği açıkça kararlaştırılmalıdır.

---

## 6. Abstract Factory — Ürün Ailesini Birlikte Üretmek

> **Benzetme —** Bir mutfak tadilatı yaptırıyorsun. Tezgâh, dolap kapağı, çekmece kulpu ve kaplama ayrı ayrı seçilebilir gibi görünür ama seçilemez: hepsinin aynı seriden olması gerekir. Beyaz seriden tezgâh, ceviz seriden kulp alırsan mutfak olmaz. Bu yüzden firmaya "beyaz seri" dersin, firma o serinin tüm parçalarını uyumlu şekilde verir. Abstract Factory tam olarak bunu yapar: parçaları tek tek değil, **seri olarak** seçersin.

**Basitçe:** Abstract Factory, birbiriyle uyumlu olması gereken bir grup nesneyi birlikte üreten fabrikadır. Sen aileyi seçersin, fabrika o ailenin tüm üyelerini verir.

**Teknik olarak:** **Abstract Factory (soyut fabrika)** — Birbiriyle ilişkili veya birbirine bağımlı nesne ailelerini, somut sınıflarını belirtmeden üretmek için bir arayüz sunan kalıp.

### Factory Method'dan farkı — net ayrım

Bu iki kalıp en çok karıştırılan çifttir. Ayrım aslında tek cümleyle yapılır:

| | Factory Method | Abstract Factory |
|---|---|---|
| Kaç tür ürün üretir | **Bir** tür (`IBildirimGonderici`) | **Birden çok** tür (`IBaglanti` + `IKomut` + `IParametre`) |
| Ne seçersin | O türün hangi varyantı | Hangi ürün **ailesi** |
| Arayüzde kaç metot | Genelde bir (`Olustur`) | Ürün türü sayısı kadar |
| Ana derdi | Somut tipten ayrılmak | Ürünlerin **birbiriyle uyumlu** kalması |
| Tipik uygulama | Bir abstract/virtual metot | Birden çok metotlu fabrika arayüzü |

Kısaca: **Factory Method bir ürün üretir, Abstract Factory bir ürün ailesi üretir.** Abstract Factory'nin her metodu aslında birer Factory Method'dur; yani Abstract Factory, Factory Method'ların bir arayüz altında toplanmış hâlidir.

### Ürün ailesi kavramı

**Ürün ailesi (product family)** — Birlikte kullanılması gereken, birbirini varsayan nesneler kümesi. SQL Server için açılmış bir bağlantıya Oracle komutu veremezsin; bunlar farklı ailelerdendir.

```csharp
// Ürün arayüzleri — ailenin üye türleri
public interface IVeriBaglantisi { void Ac(); }
public interface IVeriKomutu    { void Calistir(string sql); }
public interface ISayfalayici   { string SayfaSql(string sql, int sayfa, int boyut); }

// Fabrika arayüzü — ailenin tamamını üretir
public interface IVeritabaniFabrikasi
{
    IVeriBaglantisi BaglantiOlustur();
    IVeriKomutu     KomutOlustur();
    ISayfalayici    SayfalayiciOlustur();
}
```

SQL Server ailesi:

```csharp
public class SqlServerFabrikasi : IVeritabaniFabrikasi
{
    private readonly string _cs;
    public SqlServerFabrikasi(string cs) => _cs = cs;

    public IVeriBaglantisi BaglantiOlustur() => new SqlServerBaglantisi(_cs);
    public IVeriKomutu     KomutOlustur()    => new SqlServerKomutu();
    public ISayfalayici    SayfalayiciOlustur() => new SqlServerSayfalayici();
}

public class SqlServerSayfalayici : ISayfalayici
{
    public string SayfaSql(string sql, int sayfa, int boyut)
        => $"{sql} OFFSET {(sayfa - 1) * boyut} ROWS FETCH NEXT {boyut} ROWS ONLY";
}
```

PostgreSQL ailesi:

```csharp
public class PostgreFabrikasi : IVeritabaniFabrikasi
{
    private readonly string _cs;
    public PostgreFabrikasi(string cs) => _cs = cs;

    public IVeriBaglantisi BaglantiOlustur() => new PostgreBaglantisi(_cs);
    public IVeriKomutu     KomutOlustur()    => new PostgreKomutu();
    public ISayfalayici    SayfalayiciOlustur() => new PostgreSayfalayici();
}

public class PostgreSayfalayici : ISayfalayici
{
    public string SayfaSql(string sql, int sayfa, int boyut)
        => $"{sql} LIMIT {boyut} OFFSET {(sayfa - 1) * boyut}";
}
```

Kullanan kod tek bir fabrika seçer, gerisi otomatik uyumludur:

```csharp
public class RaporServisi
{
    private readonly IVeritabaniFabrikasi _fabrika;
    public RaporServisi(IVeritabaniFabrikasi fabrika) => _fabrika = fabrika;

    public void Calistir(string sql, int sayfa)
    {
        using var baglanti = _fabrika.BaglantiOlustur();
        var komut     = _fabrika.KomutOlustur();
        var sayfalama = _fabrika.SayfalayiciOlustur();

        baglanti.Ac();
        komut.Calistir(sayfalama.SayfaSql(sql, sayfa, 50));
    }
}
```

Buradaki asıl kazanç şudur: SQL Server bağlantısıyla PostgreSQL sayfalayıcısını yan yana getirmen **mümkün değil**. Fabrika onları ayrı ayrı vermiyor, aile olarak veriyor. Uyumsuzluk derleme anında değil, tasarım gereği engellenmiş oluyor.

Aileyi değiştirmek tek satırdır:

```csharp
// Program.cs
builder.Services.AddSingleton<IVeritabaniFabrikasi>(sp =>
    builder.Configuration["Db:Saglayici"] == "postgres"
        ? new PostgreFabrikasi(baglantiMetni)
        : new SqlServerFabrikasi(baglantiMetni));
```

### Ne zaman KULLANMA

- Tek bir ailen varsa ve ikincisi görünürde yoksa. "İleride belki başka veritabanı kullanırız" cümlesi tek başına yeterli sebep değildir; YAGNI.
- Ürünler birbirinden bağımsızsa. Uyumluluk zorunluluğu yoksa Abstract Factory'nin varlık sebebi kalmaz, ayrı ayrı Factory Method'lar daha basittir.
- EF Core kullanıyorsan. EF Core zaten sağlayıcı soyutlamasını kendi içinde yapar; üstüne bir katman daha koymak çoğu projede boşa emektir.

**Bu benzetme şurada bozulur:** Mutfak serisinde parçaları karıştırmak sadece çirkin durur, mutfak yine de çalışır. Yazılımda ise aileleri karıştırmak genelde çalışma anında patlar: SQL Server bağlantısına PostgreSQL sözdizimi gönderdiğinde derleme hatası almazsın, kullanıcı ekranında hata alırsın. Estetik bir tercih değil, doğruluk meselesidir.

---

## 7. Builder — Karmaşık Kurulumu Adım Adım Yapmak

> **Benzetme —** Bir dönerciye gidip "bir dürüm" dersin. Usta sana tek seferde sormaz; sırayla gider: "Etli mi tavuk mu? Acılı olsun mu? Turşu? Soğan? Sos?" Her cevap dürümün üstüne bir şey ekler. Sen "hazır" dediğinde usta dürümü sarar ve verir. Sarılmadan önce hiçbir şey kesinleşmemiştir, sarıldıktan sonra da değişmez. Builder budur: parçaları sırayla söylersin, sonunda "tamam" dersin, nesne o anda doğar.

**Basitçe:** Builder, çok parçalı bir nesneyi tek hamlede değil, adım adım kurmanı sağlar. Hangi parçayı verip hangisini atladığın okunaklı kalır.

**Teknik olarak:** **Builder (kurucu)** — Karmaşık bir nesnenin kurulum sürecini, nesnenin kendisinden ayıran kalıp. Aynı kurulum süreci farklı temsiller üretebilir.

### Problem: telescoping constructor

```csharp
// KÖTÜ — "telescoping constructor" (teleskopik constructor)
public class Rapor
{
    public Rapor(string baslik, DateTime tarih, string altBaslik)
        : this(baslik, tarih, altBaslik, false) { }
    public Rapor(string baslik, DateTime tarih, string altBaslik, bool grafikli)
        : this(baslik, tarih, altBaslik, grafikli, "A4") { }
    public Rapor(string baslik, DateTime tarih, string altBaslik,
                 bool grafikli, string kagitBoyu) { /* ... */ }
}
```

Kullanımı okunmazdır:

```csharp
var r = new Rapor("Aylık Satış", DateTime.Today, null, true, "A4");
// null neydi? true neydi? Çağrı yerinde hiçbir ipucu yok.
```

C#'ın adlandırılmış parametreleri bu problemin bir kısmını çözer:

```csharp
var r = new Rapor("Aylık Satış", grafikli: true, kagitBoyu: "A4");
```

Ama parametre sayısı arttıkça, bazı kombinasyonlar geçersiz hâle geldikçe ve kurulum sırasında doğrulama gerektikçe constructor yetmemeye başlar.

### Fluent Builder

```csharp
public class Rapor
{
    public string Baslik { get; init; }
    public string AltBaslik { get; init; }
    public DateTime Tarih { get; init; }
    public bool Grafikli { get; init; }
    public string KagitBoyu { get; init; }
    public List<string> Kolonlar { get; init; } = new();
}

public class RaporBuilder
{
    private string _baslik = "";
    private string _altBaslik;
    private DateTime _tarih = DateTime.Today;
    private bool _grafikli;
    private string _kagitBoyu = "A4";
    private readonly List<string> _kolonlar = new();

    public RaporBuilder Baslik(string deger)      { _baslik = deger; return this; }
    public RaporBuilder AltBaslik(string deger)   { _altBaslik = deger; return this; }
    public RaporBuilder Tarih(DateTime deger)     { _tarih = deger; return this; }
    public RaporBuilder GrafikEkle()              { _grafikli = true; return this; }
    public RaporBuilder KagitBoyu(string deger)   { _kagitBoyu = deger; return this; }
    public RaporBuilder Kolon(string ad)          { _kolonlar.Add(ad); return this; }

    public Rapor Build()
    {
        if (string.IsNullOrWhiteSpace(_baslik))
            throw new InvalidOperationException("Rapor başlığı zorunludur.");
        if (_kolonlar.Count == 0)
            throw new InvalidOperationException("En az bir kolon gerekir.");

        return new Rapor
        {
            Baslik = _baslik,
            AltBaslik = _altBaslik,
            Tarih = _tarih,
            Grafikli = _grafikli,
            KagitBoyu = _kagitBoyu,
            Kolonlar = _kolonlar.ToList()   // kopya: builder sonradan değişse de rapor etkilenmez
        };
    }
}
```

Kullanımı kendi kendini anlatır:

```csharp
var rapor = new RaporBuilder()
    .Baslik("Aylık Satış")
    .Tarih(new DateTime(2026, 9, 1))
    .Kolon("Ürün").Kolon("Adet").Kolon("Tutar")
    .GrafikEkle()
    .KagitBoyu("A3")
    .Build();
```

İki teknik ayrıntı önemlidir. Birincisi, her metodun `return this` yapması **fluent interface (akıcı arayüz)** denen zincirlemeyi mümkün kılar. İkincisi, doğrulamanın `Build()` içinde olması, nesnenin **asla geçersiz bir durumda doğmamasını** garanti eder. Builder yarım kalabilir, ürün yarım kalamaz.

### Modern alternatif: `record` + `with`

C# 9 ile gelen `record` ve `with` ifadesi, birçok Builder ihtiyacını ortadan kaldırır.

```csharp
public record RaporAyarlari(
    string Baslik,
    DateTime Tarih,
    string AltBaslik = null,
    bool Grafikli = false,
    string KagitBoyu = "A4");

// Temel yapılandırma
var temel = new RaporAyarlari("Aylık Satış", DateTime.Today);

// Türetilmiş yapılandırmalar — orijinal değişmez
var grafikli = temel with { Grafikli = true };
var a3Grafikli = grafikli with { KagitBoyu = "A3" };
```

`with` ifadesi, mevcut kaydın kopyasını alır ve sadece belirttiğin alanları değiştirir. Orijinal nesneye dokunulmaz. Bu, Builder'ın "adım adım kur" vaadinin çok büyük kısmını üç satırda verir.

| | Fluent Builder | `record` + `with` |
|---|---|---|
| Kod miktarı | Ürün başına ayrı builder sınıfı | Sıfır ek sınıf |
| Doğrulama | `Build()` içinde merkezî | Constructor'da veya `init` içinde |
| Koşullu kurulum | Doğal (`if` ile metot çağırırsın) | Zincirde zorlanır |
| Aynı builder'dan çok ürün | Mümkün | Her `with` yeni nesne |
| Kademeli/koşullu doğrulama | Güçlü | Sınırlı |

### Ne zaman Builder gereksiz

- **Parametre sayısı azsa** (dört-beş altı). Adlandırılmış parametreler ve varsayılan değerler yeter.
- **Nesne değişmez ve basitse.** `record` + `with` daha az kodla aynı işi yapar.
- **Kurulum adımları arasında bağımlılık yoksa.** Builder'ın asıl gücü "A verildiyse B zorunlu" gibi kuralları taşıyabilmesidir; böyle kurallar yoksa fazlalıktır.
- **DI konteyneri nesneyi zaten kuruyorsa.** `IOptions<T>` ve `appsettings.json` bağlaması, yapılandırma nesneleri için hazır bir kurulum yoludur.

> **Sık yapılan hata:** Builder'ın `Build()` metodunda koleksiyonu kopyalamazsan, aynı builder'dan iki ürün ürettiğinde ikisi de **aynı liste nesnesini** paylaşır. Birine kolon eklediğinde diğeri de değişir. Yukarıdaki kodda `_kolonlar.ToList()` yazmamın sebebi budur. Bu hata testlerde çok geç fark edilir.

**Bu benzetme şurada bozulur:** Dönercide ustaya "acılı" dedikten sonra vazgeçebilirsin, usta acıyı koymaz. Builder'da da metot çağırmaktan vazgeçebilirsin ama çoğu builder **geri alma** sunmaz; `GrafikEkle()` dedikten sonra `GrafikKaldir()` yoksa yeni bir builder kurman gerekir. Builder ileri doğru çalışır, geri doğru değil. Geri alma ihtiyacın varsa ya builder'a `Reset()` koyarsın ya da `record` + `with` yaklaşımına geçersin.

---

## 8. Prototype — Yeniden Kurmak Yerine Kopyalamak

> **Benzetme —** Muhtarlıkta bir ikametgâh belgesi alırsın. İkinci bir nüsha lazım olduğunda memur bütün bilgileri baştan yazmaz; belgeyi fotokopiye koyar. Kopya, aslın o anki hâlini taşır. Ama fotokopide bir incelik vardır: belgenin üstünde bir dosya numarası yazıyorsa, kopyadaki numara da **aynı dosyayı** işaret eder. Kopya ayrı bir kâğıttır, ama işaret ettiği dosya tektir. Shallow copy ile deep copy arasındaki fark tam olarak budur.

**Basitçe:** Prototype, yeni bir nesneyi sıfırdan kurmak yerine mevcut bir nesneyi kopyalayarak üretir. Kurulum pahalıysa veya mevcut nesnenin durumunu korumak gerekiyorsa işe yarar.

**Teknik olarak:** **Prototype (prototip)** — Üretilecek nesnelerin türünü, örnek alınan bir prototip nesne üzerinden belirleyen ve yeni nesneleri o prototipi kopyalayarak üreten kalıp.

### `ICloneable` ve neden sorunlu

```csharp
public class Basvuru : ICloneable
{
    public int Id { get; set; }
    public string AdSoyad { get; set; }
    public List<string> Yetenekler { get; set; } = new();

    public object Clone() => MemberwiseClone();   // shallow copy
}
```

`ICloneable` .NET'in en eski arayüzlerinden biridir ve Microsoft bugün kullanılmasını **önermez**. Sebebi tek bir eksikliktir: arayüz, kopyanın shallow mı deep mi olduğunu söylemez. `Clone()` çağıran kişi ne alacağını bilemez. Dönüş tipinin `object` olması da ayrı bir sıkıntıdır; her çağrıda cast yazarsın.

### Shallow copy vs deep copy

**Shallow copy (sığ kopya)** — Nesnenin alanları birebir kopyalanır. Referans tipli alanlar için **referans** kopyalanır, işaret edilen nesne kopyalanmaz.

```csharp
var asil = new Basvuru { Id = 1, AdSoyad = "Umut", Yetenekler = { "C#", "SQL" } };
var kopya = (Basvuru)asil.Clone();

kopya.AdSoyad = "Ali";              // string değişmez tiptir, asıl etkilenmez
Console.WriteLine(asil.AdSoyad);    // "Umut" — sorun yok

kopya.Yetenekler.Add("EF Core");    // AYNI listeye eklendi
Console.WriteLine(asil.Yetenekler.Count);   // 3 — asıl da değişti
```

Liste paylaşıldığı için kopyaya yapılan ekleme asıla da yansıdı. `MemberwiseClone` referansı kopyalar, listeyi kopyalamaz.

**Deep copy (derin kopya)** — Referans tipli alanlar da kopyalanır; kopya ile asıl hiçbir nesneyi paylaşmaz.

```csharp
public class Basvuru
{
    public int Id { get; set; }
    public string AdSoyad { get; set; }
    public List<string> Yetenekler { get; set; } = new();
    public Adres Adres { get; set; }

    public Basvuru DerinKopya() => new Basvuru
    {
        Id = Id,
        AdSoyad = AdSoyad,
        Yetenekler = new List<string>(Yetenekler),          // yeni liste
        Adres = Adres is null ? null : new Adres            // yeni adres nesnesi
        {
            Il = Adres.Il,
            Ilce = Adres.Ilce
        }
    };
}
```

Elle yazılan deep copy güvenilirdir ama sınıf büyüdükçe bakımı zorlaşır; yeni bir alan eklediğinde kopyalama metodunu güncellemeyi unutursan sessiz bir hata doğar.

| | Shallow copy | Deep copy |
|---|---|---|
| Değer tipli alanlar | Kopyalanır | Kopyalanır |
| `string` alanlar | Referans kopyalanır (değişmez olduğu için sorunsuz) | Aynı |
| Koleksiyon / nesne alanlar | **Paylaşılır** | Kopyalanır |
| Maliyet | Ucuz | Nesne ağacı kadar pahalı |
| Tipik yol | `MemberwiseClone()` | Elle yazma veya serileştirme |

### .NET'te Prototype neden az kullanılır

- **DI konteyneri var.** Nesne kurulumu pahalıysa çözüm kopyalamak değil, konteynerin uygun yaşam süresiyle yönetmesidir.
- **`record` ve `with` işin çoğunu yapar.** Yapılandırma benzeri nesneler için ayrı bir Prototype altyapısına gerek kalmaz.
- **`ICloneable` önerilmiyor.** Kalıbın .NET'teki resmî arayüzü zaten tavsiye dışıdır; herkes kendi `DerinKopya()` metodunu yazar ve kalıp adı anılmaz.
- **Serileştirme kolay bir kaçış yolu sunar.** JSON'a çevirip geri okumak pratik bir deep copy yöntemidir, ama pahalıdır ve her tip için çalışmaz.

> **Uyarı:** Serileştirmeyle kopyalama, yalnızca serileştirilebilir alanları taşır. `private` alanlar, olay (event) abonelikleri, dosya tutamakları ve döngüsel referanslar ya kaybolur ya hata verir. EF Core varlıklarında bunu yaparsan navigation property'ler yüzünden sonsuz döngüye girebilirsin. Kolay göründüğü için seçilir, sonra sebebi bulunamayan hatalar üretir.

### Ne zaman KULLANMA

- Nesnenin kurulumu ucuzsa. `new` yazmak kopyalamaktan hızlıdır.
- Nesne değişmezse (immutable). Kopyalamaya gerek yok, aynı örneği paylaş.
- EF Core varlıkları için. Takip edilen bir varlığı kopyalamak, değişiklik izleyicisini şaşırtır; kopyalanan nesnenin `Id`'sini sıfırlamayı unutursan güncelleme yerine ekleme olur.

**Bu benzetme şurada bozulur:** Fotokopi benzetmesinde kopya ile asıl arasında hiçbir canlı bağ yoktur — kâğıtlar ayrıdır. Shallow copy'de ise bağ vardır ve görünmezdir. Asıl tehlike de budur: nesne ekranda iki ayrı nesne gibi durur, aynı listeyi paylaştığını ancak birinde yaptığın değişiklik diğerinde göründüğünde anlarsın. Fotokopi yanlış çıkarsa hemen görürsün; sığ kopya yanlışı haftalar sonra görünür.

---

## 9. .NET'te Bu Kalıpların Yerleşik Karşılıkları

> **Benzetme —** Yeni bir eve taşındığında elektrik tesisatını sen çekmezsin; duvarda prizler hazırdır. Prizlerin arkasında bir standart, bir hesap, bir mühendislik vardır ama sen sadece fişi takarsın. .NET'in içindeki kalıplar da böyledir: mühendislik yapılmış, priz duvara konmuştur. Senin işin doğru prizi tanımaktır.

**Basitçe:** Bu notta öğrendiğin kalıpların çoğu .NET'in içine zaten gömülüdür. Yeniden yazmana gerek yoktur; tanıman yeter.

**Teknik olarak:** Aşağıdaki tipler, GoF kalıplarının framework içindeki somut uygulamalarıdır.

| .NET tipi | Kalıp | Ne yapar |
|---|---|---|
| `IServiceProvider` / `IServiceCollection` | Abstract Factory + Service Locator | Kayıtlı tiplerden nesne üretir, bağımlılıkları çözer |
| `StringBuilder` | Builder | Metni adım adım kurar, `ToString()` ile ürünü verir |
| `WebApplicationBuilder` | Builder | Uygulamayı adım adım yapılandırır, `Build()` ile üretir |
| `ModelBuilder` (EF Core Fluent API) | Builder | Model yapılandırmasını zincirleme kurar |
| `IHttpClientFactory` | Factory + Object Pool | `HttpClient` üretir, alttaki handler'ları havuzlar |
| `ILoggerFactory` | Factory Method | Kategoriye göre `ILogger` örneği üretir |
| `DbProviderFactory` | Abstract Factory | Bağlantı, komut, parametre ailesini birlikte üretir |
| `IDbContextFactory<T>` | Factory | Kendi ömrünü sen yönetesin diye `DbContext` üretir |
| `Lazy<T>` | Lazy Initialization | Nesneyi ilk erişimde, iş parçacığı güvenli üretir |
| `ArrayPool<T>` / `ObjectPool<T>` | Object Pool | Pahalı nesneleri yeniden kullanır |

### `IServiceProvider` — konteynerin kendisi bir fabrikadır

```csharp
// Kayıt — hangi arayüz hangi sınıfa, hangi ömürle
builder.Services.AddSingleton<IKurServisi, KurServisi>();
builder.Services.AddScoped<IBasvuruRepository, BasvuruRepository>();
builder.Services.AddTransient<IEmailGonderici, SmtpGonderici>();

// Çözümleme — konteyner constructor'a bakar, bağımlılıkları kendi kurar
public class BasvuruController : Controller
{
    private readonly IBasvuruRepository _repo;
    public BasvuruController(IBasvuruRepository repo) => _repo = repo;
}
```

MvcCv'deki `GenericRepository` kaydını `AddScoped` ile yaptığında, aslında "bu ürünün fabrikası budur, her istek için bir tane üret" demiş oluyorsun. `new BasvuruRepository(new AppDbContext(...))` yazmaman, fabrikanın işini yapmasındandır.

> **Uyarı:** `IServiceProvider`'ı sınıflarına enjekte edip `GetRequiredService<T>()` çağırmak, DI'yi **Service Locator** anti-pattern'ine çevirir. Bağımlılıklar tekrar gizlenir ve Singleton'ın bütün dertleri geri gelir. Konteyneri doğrudan kullanmak yalnızca fabrika sınıflarında, `BackgroundService` içinde ve middleware gibi altyapı noktalarında kabul edilebilir.

### `IHttpClientFactory` — fabrika + havuz

```csharp
// Program.cs — adlandırılmış istemci
builder.Services.AddHttpClient("kur", c =>
{
    c.BaseAddress = new Uri("https://api.exchangerate.host/");
    c.Timeout = TimeSpan.FromSeconds(10);
});

public class KurServisi : IKurServisi
{
    private readonly IHttpClientFactory _fabrika;
    public KurServisi(IHttpClientFactory fabrika) => _fabrika = fabrika;

    public async Task<decimal> GetirAsync(string kod)
    {
        var client = _fabrika.CreateClient("kur");   // fabrika metodu
        var cevap = await client.GetFromJsonAsync<KurCevabi>($"latest?base={kod}");
        return cevap.Rates["TRY"];
    }
}
```

Bu fabrikanın varlık sebebi sadece nesne üretmek değildir. `HttpClient`'ı her istekte `new` ile üretirsen **socket exhaustion (soket tükenmesi)** yaşarsın: kapatılan bağlantılar `TIME_WAIT` durumunda beklediği için port havuzu biter. Tek bir `static HttpClient` kullanırsan bu sefer DNS değişikliklerini kaçırırsın. `IHttpClientFactory` alttaki `HttpMessageHandler`'ları havuzlar ve belirli aralıkla tazeler; iki problemi birden çözer.

### `ILoggerFactory` — adı kalıbı söyleyen tip

```csharp
public class SiparisServisi
{
    private readonly ILogger<SiparisServisi> _logger;

    // Konteyner arka planda ILoggerFactory.CreateLogger("...SiparisServisi") çağırır
    public SiparisServisi(ILogger<SiparisServisi> logger) => _logger = logger;
}
```

`ILogger<T>` enjekte ettiğinde konteyner senin için fabrikayı çağırır ve kategorisi tip adı olan bir logger üretir. Kategori, log satırlarını filtrelemenin anahtarıdır.

**Bu benzetme şurada bozulur:** Priz benzetmesinde duvardaki priz tektir ve alternatifi yoktur. .NET'te ise aynı kalıbın birkaç yerleşik karşılığı olabilir ve hangisini seçeceğin duruma bağlıdır. `AddDbContext` mi `AddDbContextFactory` mi, `AddHttpClient` mi typed client mı — priz var ama doğru prizi seçmek yine sana düşüyor.

---

## 10. Hangi Problemde Hangi Yaratımsal Kalıp

> **Benzetme —** Oto tamircisinde ustanın önünde bir alet dolabı vardır. Hepsini kullanmaz; sese, titreşime, arızanın tarifine bakar ve doğru aleti seçer. Yanlış alet işi görmez, bazen vidayı sıyırır. Kalıp seçimi de arızayı doğru tarif etmekle başlar.

**Basitçe:** Kalıp seçmeden önce problemi bir cümleyle yaz. Cümle doğruysa kalıp kendini gösterir.

**Teknik olarak:** Karar tablosu.

| Problemin | Kalıp | .NET karşılığı |
|---|---|---|
| "Bundan uygulamada tek tane olmalı" | Singleton | `AddSingleton` (GoF Singleton değil) |
| "Hangi sınıfı üreteceğime çalışma anında karar vereceğim" | Factory Method | Fabrika sınıfı, keyed services |
| "Birbiriyle uyumlu olması gereken bir grup nesne üreteceğim" | Abstract Factory | `DbProviderFactory` |
| "Constructor'ım okunmaz hâle geldi, kurulum çok adımlı" | Builder | `StringBuilder`, `WebApplicationBuilder` |
| "Var olan nesnenin bir kopyasıyla devam edeceğim" | Prototype | `record` + `with` |
| "Nesne pahalı, gerçekten lazım olana kadar kurulmasın" | Lazy Initialization | `Lazy<T>` |
| "Pahalı nesneleri yeniden kullanmak istiyorum" | Object Pool | `ArrayPool<T>`, `IHttpClientFactory` |

### Önce sor: kalıba gerçekten ihtiyaç var mı

Üç soru yeter: Problem **şu an** var mı, yoksa "ileride olabilir" mi? .NET bunu **zaten** yapıyor mu? Kalıbı kaldırınca kod anlaşılmaz mı oluyor, yoksa sadeleşiyor mu? Üçüncü sorunun cevabı "sadeleşiyor" ise kalıp gereksizdir.

> **En sık hata:** DI konteynerinin zaten yaptığı işi tekrar etmek. `IKullaniciServisiFactory` yazıp içinde `new KullaniciServisi(_repo, _logger)` döndürmek, konteynerin tam olarak yaptığı iştir. Fabrika yazmanın anlamı, **konteynerin bilemeyeceği bir karar** varsa (çalışma anındaki bir değere göre seçim, isteğe özel bir parametre) doğar.

**Bu benzetme şurada bozulur:** Usta aleti seçerken yanılırsa hemen anlar, vida dönmez. Kalıpta ise yanlış seçim hemen görünmez; kod çalışır, testler geçer. Bedel altı ay sonra, bir değişiklik yapman gerektiğinde ortaya çıkar. Bu yüzden kalıp seçiminde "işe yarıyor mu" yetersiz bir ölçüttür; "değiştirmesi kolay mı" diye sormak gerekir.

---

## Tek Bakışta Özet

- Tasarım kalıbı hazır kod değil, tekrar eden probleme verilmiş **adı konmuş** çözümdür; asıl değeri ortak dil olmasıdır.
- Singleton iki şey vaat eder: tek örnek ve global erişim. Zarar veren ikincisidir.
- Singleton'ı elle yazacaksan `Lazy<T>` veya `static readonly` alan kullan; double-checked locking'i sen yazma.
- .NET'te "tek tane olsun" ihtiyacının cevabı `AddSingleton`'dır, GoF Singleton değil.
- `AddSingleton` ile kaydedilen sınıfa `AddScoped` servis enjekte etme — captive dependency oluşur; `IServiceScopeFactory` kullan.
- Factory Method `switch`'i yok etmez, **tek noktaya toplar**; kullanan kodu somut tipten ayırır.
- Abstract Factory bir ürün **ailesi** üretir; tek fark budur. Yeni aile eklemek ucuzdur, aileye yeni ürün türü eklemek tüm fabrikaları açmayı gerektirir.
- Builder, çok parametreli constructor'ın ve geçersiz nesne doğmasının çözümüdür; doğrulama `Build()` içinde olmalıdır.
- `record` + `with`, Builder ihtiyacının büyük kısmını sıfır ek sınıfla karşılar ama shallow kopya yapar.
- Prototype'ta `MemberwiseClone` sığ kopyadır: koleksiyonlar paylaşılır. `ICloneable` artık önerilmiyor.
- `IServiceProvider`, `StringBuilder`, `IHttpClientFactory`, `ILoggerFactory`, `ModelBuilder` — hepsi yerleşik kalıp uygulamalarıdır.
- Kalıp seçmeden önce sor: problem şu an var mı, .NET zaten yapıyor mu, kalıbı kaldırınca kod sadeleşiyor mu.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Singleton | Sınıfın tek örneğini garanti eden ve global erişim sunan kalıp |
| Double-checked locking | Kilit almadan önce ve aldıktan sonra iki kez kontrol eden tekil üretim tekniği |
| `Lazy<T>` | Gecikmeli ve iş parçacığı güvenli başlatma sağlayan .NET tipi |
| Captive dependency | Uzun ömürlü servisin kısa ömürlü servisi tutsak etmesi |
| Service Locator | Konteyneri sınıfa enjekte edip tip çözme; bağımlılığı gizlediği için anti-pattern |
| Factory Method | Nesne üretimini bir metoda devreden, somut tipten ayıran kalıp |
| Abstract Factory | Birbiriyle uyumlu ürün ailesini birlikte üreten kalıp |
| Ürün ailesi | Birlikte kullanılması zorunlu, birbirini varsayan nesneler kümesi |
| Telescoping constructor | Parametre sayısı artarak çoğalan constructor aşırı yüklemeleri |
| YAGNI | "You Aren't Gonna Need It" — henüz olmayan ihtiyaç için kod yazmama ilkesi |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`AddSingleton` GoF Singleton kalıbıdır" | Değildir; tekliği sınıf değil konteyner sağlar, sınıf sıradan kalır |
| "Singleton'a `DbContext` enjekte edilebilir" | Edilemez; captive dependency olur, `IServiceScopeFactory` ile scope açılır |
| "Factory Method ile Abstract Factory aynı şey" | İlki bir ürün, ikincisi bir ürün ailesi üretir |
| "Abstract Factory'ye yeni ürün eklemek kolaydır" | Tüm somut fabrikaları açmayı gerektirir; kolay olan yeni **aile** eklemektir |
| "Builder her çok parametreli sınıf için gerekir" | Adlandırılmış parametreler ve `record` + `with` çoğu durumda yeter |
| "`record` + `with` deep copy yapar" | Shallow kopya yapar; içindeki liste yine paylaşılır |
| "`MemberwiseClone` nesneyi tam kopyalar" | Referans tipli alanlar paylaşılır; liste ortaktır |
| "`ICloneable` kullanılması gereken standart arayüzdür" | Kopyanın derinliğini belirtmediği için önerilmez |
| "`new HttpClient()` her istekte güvenlidir" | Soket tükenmesine yol açar; `IHttpClientFactory` kullanılır |

---

## Sonraki

→ `06-Structural-ve-Behavioral-Patterns.md` (Cumartesi)
