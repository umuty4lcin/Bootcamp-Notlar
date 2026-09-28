# Hafta 2 · Pazar — Hafta Özeti: OOP, SOLID ve Tasarım Kalıpları

**Okuma süresi:** ~30 dk
**Neden bu konu:** Bu hafta altı not okudun ve her biri kendi başına duruyor. Özet, onları tek bir resme oturtur. Bootcamp'te mimari projelere geldiğinde (Proje 10, 11, 12) sana lazım olacak şey ayrı ayrı tanımlar değil, **aralarındaki bağ** olacak: neden kompozisyon kalıtımdan önce gelir, neden DIP olmadan DI eksik kalır, neden middleware aslında bir tasarım kalıbıdır.

---

## Haftanın Tek Cümlesi

Bu haftayı üç basamaklı bir merdiven gibi düşün.

**Birinci basamak — OOP: araçlar.** Sınıf, arayüz, kalıtım, polimorfizm. Bunlar bir marangozun testeresi, rendesi, çekici. Aleti tanımak gerekir ama alet tanımak marangoz yapmaz. Çoğu geliştirici bu basamakta kalır: dört sütunu ezbere sayar, `abstract class` ile `interface` arasındaki farkı bilir, ama "burada hangisini kullanmalıyım" sorusuna gelince duraklar.

**İkinci basamak — SOLID: aletleri kullanma disiplini.** Beş ilke de aslında tek bir soruyu farklı açılardan soruyor: *"Yarın bu kod değişmek zorunda kalırsa, kaç yeri açmam gerekir?"* SRP der ki bir sınıfın değişmesi için tek bir sebep olsun. OCP der ki yeni davranış ekleyeceksen eski kodu açma. LSP der ki alt tip, üst tipin verdiği sözü tutsun. ISP der ki kimse kullanmadığı metodu uygulamak zorunda kalmasın. DIP der ki bağımlılık somuta değil soyuta gitsin. Beşi birden "değişimin maliyetini düşür" diyor.

**Üçüncü basamak — Tasarım kalıpları: disiplinin yerleşmiş çözümleri.** Pattern'ler yeni bir şey icat etmez; aynı problemle defalarca karşılaşmış insanların "bunu şöyle çözdük" demesidir. Strategy aslında OCP'nin uygulanmış hâli. Decorator aslında kompozisyonun uygulanmış hâli. Chain of Responsibility ise ASP.NET Core'un middleware pipeline'ının ta kendisi. Yani pattern öğrenmek, ilkeleri kod hâline getirilmiş görmektir.

Merdivenin önemli özelliği şu: **basamak atlanmaz.** Pattern ezberleyip SOLID bilmeyen biri, her yere Singleton ve Factory sokup kodu daha da karmaşık hâle getirir. SOLID bilip OOP'yi pratikte oturtmamış biri, soyutlamayı yanlış yere koyar. Sırayı koruduğun sürece her basamak bir öncekini anlamlandırır.

> **Ana benzetme:** OOP tezgâhtaki aletler, SOLID ustanın çalışma disiplini, tasarım kalıpları ise atölyede nesilden nesile aktarılan "şu iş şöyle yapılır" bilgisi. Alet olmadan iş olmaz; disiplin olmadan alet zarar verir; aktarılan bilgi olmadan her usta aynı hatayı baştan yapar.

---

## Bu Notta Ne Var

1. OOP dört sütun, pratikte
2. Kalıtım yerine kompozisyon
3. SOLID 1-3: SRP, OCP, LSP
4. SOLID 4-5: ISP, DIP ve Dependency Injection
5. Yaratımsal kalıplar
6. Yapısal ve davranışsal kalıplar
7. Bağlantı haritası
8. Kendini yoklama

---

## 1. OOP Dört Sütun, Pratikte

> **Benzetme —** Bir apartmanın dairesi gibi. Dışarıdan sadece kapı zili ve posta kutusu görünür; içerideki tesisat, kablolar, dolap düzeni komşuyu ilgilendirmez. Kapsülleme budur. Kat planının aynı olması ama her dairenin farklı döşenmesi polimorfizmdir.

**Basitçe:** Dört sütun ezberlenecek tanımlar değil, dört ayrı derde deva. Kapsülleme "içeriyi koru", kalıtım "ortak olanı tekrarlama", polimorfizm "aynı komutla farklı davranış", soyutlama "detayı değil sözleşmeyi göster" demek.

**Notun damıtılmışı:**

- **Erişim belirleyiciler** altı tanedir. `protected internal` = birleşim (aynı assembly **veya** türeyen sınıf), `private protected` = kesişim (aynı assembly **ve** türeyen sınıf). Karıştırılır.
- **Property, field değildir.** Property bir metot çiftidir; field bir bellek alanı. Arayüzde property tanımlanabilir, field tanımlanamaz. Public field kullanma — sonradan doğrulama ekleyemezsin ve binary uyumluluğu bozarsın.
- **`virtual`/`override`** gerçek polimorfizmdir; **`new`** ise method hiding'dir ve referans tipine göre farklı metot çağırır. Bu, sessiz hataların klasik kaynağıdır.
- **`abstract class` vs `interface`:** durum (field) taşıyacaksan, ortak constructor mantığı varsa, "is-a" ilişkisi gerçekse abstract class. Birden çok yetenek, çoklu uygulama, düşük bağ istiyorsan interface. Default interface method'lar geldi ama **durum tutamaz** ve sınıf referansından görünmez — abstract class'ın yerini almaz.
- **Constructor'da sanal metot çağırma.** Taban sınıfın ctor'u çalışırken türeyen sınıfın alanları henüz atanmamıştır; override edilmiş metot yarı kurulmuş nesne üzerinde çalışır.
- **`Equals` ve `GetHashCode` birlikte override edilir.** Biri override edilip diğeri edilmezse `Dictionary` ve `HashSet` sessizce yanlış çalışır.
- **`record`**, değer eşitliğini ve `ToString`'i senin yerine yazar. Bu yüzden DTO'lar için idealdir.
- **`sealed`** kapıyı kapatmaktır ve JIT'in devirtualization yapmasına izin verdiği için ölçülebilir bir performans kazancı da sağlar.

```csharp
// new ile gizleme: referansın TİPİ kararı verir — polimorfizm değil
public class Taban        { public void Yaz() => Console.WriteLine("Taban"); }
public class Turetilmis : Taban { public new void Yaz() => Console.WriteLine("Türetilmiş"); }

Taban t = new Turetilmis();
t.Yaz();                   // "Taban" — çoğu kişi "Türetilmiş" bekler
```

```csharp
// Sözleşme: Equals'ı override ettiysen GetHashCode'u da et
public override bool Equals(object? o) => o is Urun u && u.Kod == Kod;
public override int GetHashCode() => Kod.GetHashCode();
```

---

## 2. Kalıtım Yerine Kompozisyon

> **Benzetme —** Kalıtım evlatlık almaya benzer: aldığın şeyin tüm geçmişini, huyunu ve ileride değişecek hâlini de almış olursun. Kompozisyon ise işe adam almaktır: işi yapsın yeter, beğenmezsen değiştirirsin.

**Basitçe:** Kalıtım en sıkı bağdır. Taban sınıf değiştiğinde tüm türeyenler etkilenir ve bunu derleyici sana söylemez. Kompozisyon aynı yeniden kullanımı çok daha gevşek bağla verir.

**Notun damıtılmışı:**

- **Kırılgan taban sınıf problemi (fragile base class):** Taban sınıfta masum görünen bir değişiklik (bir metodun içinden başka bir metodu çağırmaya başlaması gibi) türeyen sınıfların davranışını sessizce bozar.
- **Karar testi:** "Her X bir Y midir — her zaman, her koşulda?" Cevap "evet ama..." ise kalıtım değil kompozisyon.
- **Derin hiyerarşi** kombinasyon patlaması üretir. İki bağımsız eksende değişen davranış için kalıtım ağacı `2×2=4` sınıf gerektirir; kompozisyon iki ayrı parça yeter.
- **LSP ihlali** en sık şu üç biçimde görülür: alt tip ön koşulu güçlendirir, son koşulu zayıflatır, ya da üst tipte olmayan bir istisna fırlatır. `NotImplementedException` en açık ihlal kokusudur.
- **Davranışı parametre olarak geçirmek** — `Func<>`, delegate ya da küçük bir arayüz — Strategy kalıbının kapısıdır.
- **Extension method** sanal değildir: çalışma anındaki tipe değil, derleme anındaki tipe göre seçilir. Polimorfizm beklersen tuzağa düşersin.
- **Kalıtım hâlâ doğru olduğu yerler:** framework tipleri (`Controller`, `DbContext`, `Exception`), Template Method kalıbı, gerçekten kararlı ve dar hiyerarşiler.

```csharp
// KÖTÜ: kalıtımla yetenek ekleme — kombinasyon patlar
class Rapor { }
class PdfRapor : Rapor { }
class SifreliPdfRapor : PdfRapor { }      // sonra: SikistirilmisSifreliPdfRapor...

// İYİ: kompozisyon — parçaları birleştir
class Rapor(IBicimlendirici bicim, IKoruma koruma) { }
```

---

## 3. SOLID 1-3: SRP, OCP, LSP

> **Benzetme —** Muhtarlık, tapu ve nüfus müdürlüğünün ayrı binalarda olması gibi. Hepsi tek binada olsaydı nüfus kayıt yönetmeliği değiştiğinde tapu işlemleri de kapanırdı. Ayrı olmalarının sebebi kalabalık değil, **değişimin birbirine bulaşmaması**.

**Basitçe:** SRP "değişmek için tek sebep", OCP "yeni davranış eklerken eski kodu açma", LSP "alt tip verdiği sözü tutsun". Üçü birlikte değişimin maliyetini düşürür.

**Notun damıtılmışı:**

- **SRP'nin doğru okunuşu** "sınıf tek iş yapsın" değil, **"sınıf tek bir aktöre karşı sorumlu olsun"**. Muhasebe departmanının isteği değiştiğinde İK'yı ilgilendiren kod değişmemeli.
- **Fat controller** en sık görülen ihlal: controller hem doğrulama, hem iş kuralı, hem veri erişimi, hem e-posta gönderimi yapar.
- **SRP'nin sınırı var.** Her metodu ayrı sınıfa koymak da bir hatadır; okunabilirliği öldürür. Ölçü "bu iki şey birlikte mi değişiyor?" sorusudur.
- **OCP pratikte** `switch` zincirini polimorfizme çevirmektir. Yeni bir tür eklediğinde mevcut `switch`'i açmıyorsan OCP'ye uyuyorsun demektir.
- **OCP'nin bedeli spekülatif soyutlamadır.** "İleride lazım olur" diye kurulan arayüzlerin çoğu lazım olmaz, sadece dolaylılık ekler. Üçüncü tekrarı görmeden soyutlama.
- **LSP** sadece derleme meselesi değil, **davranış** meselesidir. Kod derlenir, test geçer, ama alt tip başka türlü davranır.

```csharp
// KÖTÜ — OCP ihlali: yeni ödeme tipi = bu switch'i aç
decimal Komisyon(Odeme o) => o.Tip switch
{
    "kredi"  => o.Tutar * 0.02m,
    "havale" => 0m,
    _        => throw new NotSupportedException()
};

// İYİ — yeni tip = yeni sınıf, bu dosyaya dokunulmaz
public interface IOdemeYontemi { decimal Komisyon(decimal tutar); }
public class KrediKarti : IOdemeYontemi { public decimal Komisyon(decimal t) => t * 0.02m; }
public class Havale     : IOdemeYontemi { public decimal Komisyon(decimal t) => 0m; }
```

```csharp
// LSP ihlali: alt tip ön koşulu güçlendiriyor
public class Hesap        { public virtual void Cek(decimal t) { /* bakiye yeterse */ } }
public class VadeliHesap : Hesap
{
    public override void Cek(decimal t) => throw new InvalidOperationException("Vadeli hesaptan çekilemez");
}
// Çözüm: kısıtı davranışa taşı — bool CekilebilirMi(decimal t)
```

---

## 4. SOLID 4-5: ISP, DIP ve Dependency Injection

> **Benzetme —** Bir elektrik prizine takılan alet, prizin arkasındaki kabloyu tanımaz. Priz standart bir sözleşmedir; santralin kömürle mi rüzgârla mı çalıştığı aleti ilgilendirmez. Soyutlamanın işlevi budur.

**Basitçe:** ISP "kimse kullanmayacağı metodu uygulamak zorunda kalmasın", DIP "üst seviye modül alt seviyeye değil, ikisi de soyutlamaya bağlansın". DI ise DIP'i hayata geçirmenin en yaygın yolu — ama aynı şey değil.

**Notun damıtılmışı:**

- **Şişman arayüz** belirtisi: uygulayan sınıflardan biri bazı metotları `NotImplementedException` ile geçiyorsa arayüz bölünmelidir.
- **Rol arayüzleri:** `IOkuyucu` / `IYazici` ayrımı gibi. Bir sınıf birden çok rolü üstlenebilir, ama tüketici sadece ihtiyacı olan rolü ister.
- **DIP'in en çok atlanan ayrıntısı:** arayüz, onu *uygulayanın* değil **tüketenin** katmanında tanımlanır. `IUrunRepository` iş katmanında durur; veri erişim katmanı onu uygular. Bağımlılık oku böylece ters döner.
- **DI üç yoldan yapılır:** constructor (varsayılan tercih), property (isteğe bağlı bağımlılık), method (tek seferlik). Constructor injection zorunluluğu tipte belgeler.
- **Composition root:** `new` yazmanın meşru olduğu tek yer, uygulamanın en dış katmanıdır (`Program.cs`). Geri kalan her yerde bağımlılık dışarıdan gelir.
- **Service locator** (`container.Resolve<T>()` çağrısını sınıfın içine koymak) anti-pattern'dir: bağımlılığı gizler, testi zorlaştırır, çalışma anına erteler.
- **DIP ≠ DI.** DIP bir tasarım ilkesidir (bağımlılık yönü), DI bir tekniktir (nesneyi dışarıdan verme). Somut bir sınıfı constructor'dan geçirirsen DI yapmış ama DIP'e uymamış olursun.
- **DI'ın asıl kazancı test edilebilirliktir.** Bağımlılığı dışarıdan verebiliyorsan sahte (fake) bir uygulama koyup sınıfı tek başına test edebilirsin.

```csharp
// DIP: soyutlama tüketicinin katmanında
// --- Business katmanı ---
public interface IUrunRepository { Urun? Getir(int id); }
public class UrunServisi(IUrunRepository repo)
{
    public Urun? Bul(int id) => repo.Getir(id);
}
// --- DataAccess katmanı (Business'a referans verir, tersi değil) ---
public class EfUrunRepository(AppDbContext db) : IUrunRepository
{
    public Urun? Getir(int id) => db.Urunler.Find(id);
}
```

```csharp
// Composition root — new'in meşru olduğu tek yer
builder.Services.AddScoped<IUrunRepository, EfUrunRepository>();
builder.Services.AddScoped<UrunServisi>();
```

> **MvcCv bağlantısı:** Projende controller'lar repository'yi doğrudan `new`liyordu. DIP açısından bakıldığında sorun "fazladan satır" değil: controller (üst seviye) somut repository'ye (alt seviye) bağlı olduğu için veri erişimini değiştirmeden controller'ı test edemezsin.

---

## 5. Yaratımsal Kalıplar

> **Benzetme —** Bir lokantada yemeği sen pişirmezsin, mutfağa sipariş verirsin. Mutfağın hangi tencereyi kullandığı seni ilgilendirmez. Yaratımsal kalıpların tamamı "nesnenin nasıl doğduğunu kullanıcıdan saklama" işidir.

**Basitçe:** Bu aile, `new` çağrısını kimin, nerede ve nasıl yapacağını düzenler. Amaç: kullanan kod, kullandığı şeyin nasıl kurulduğunu bilmesin.

**Notun damıtılmışı:**

- **Tasarım kalıbı hazır kod değildir**, ortak bir dildir. "Burada decorator kullanalım" demek, on cümlelik açıklamayı tek kelimeye indirir.
- **Singleton** klasik olarak `Lazy<T>` ile thread-safe yazılır. Ama modern .NET'te kendi Singleton'ını yazmak nadiren doğrudur: DI konteynerindeki `AddSingleton` aynı işi yapar ve **test edilebilirliği bozmaz**. Klasik Singleton gizli bağımlılıktır — constructor'a bakarak göremezsin.
- **Singleton'a Scoped enjekte etmek** captive dependency üretir; çözüm `IServiceScopeFactory`.
- **Factory Method** ile **Abstract Factory** karıştırılır: Factory Method *bir* ürün üretir ve üretimi alt sınıfa devreder; Abstract Factory *birbirine ait bir ürün ailesini* birlikte üretir.
- **Builder**, çok parametreli constructor derdine çözümdür. Ama C#'ta `record` + nesne başlatıcı + `with` çoğu senaryoda builder'ı gereksiz kılar.
- **.NET'te kalıplar zaten var:** `IServiceProvider` (Abstract Factory + Service Locator), `StringBuilder` (Builder), `IHttpClientFactory` ve `ILoggerFactory` (Factory).

```csharp
// Singleton yazmak yerine konteynere söyle
builder.Services.AddSingleton<IAyarSaglayici, AyarSaglayici>();

// Singleton içinde Scoped kullanman gerekiyorsa
public class ArkaPlanIsi(IServiceScopeFactory scopeFactory)
{
    public void Calistir()
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    }
}
```

---

## 6. Yapısal ve Davranışsal Kalıplar

> **Benzetme —** Yapısal kalıplar binanın parçalarının nasıl birleştiğiyle ilgilenir: hangi oda hangi koridora açılıyor, hangi duvar taşıyıcı. Davranışsal kalıplar ise binanın içinde işin nasıl aktığıyla: evrak hangi masadan hangisine gidiyor, kim kime haber veriyor.

**Basitçe:** Yapısal kalıplar nesneleri birleştirir (Adapter, Decorator, Facade, Proxy). Davranışsal kalıplar nesneler arası iş akışını düzenler (Strategy, Observer, Command, Template Method, Chain of Responsibility).

**Notun damıtılmışı:**

- **Adapter** uyumsuz arayüzü uydurur. Üçüncü parti kütüphaneyi doğrudan çağırmak yerine kendi arayüzünle sarmalamanın adı budur — kütüphaneyi değiştirmek tek sınıfı değiştirmeye iner.
- **Decorator** aynı arayüzü uygulayıp davranış ekler. Loglama, cache, retry katmanları böyle eklenir. DI'da kayıt sırası = çalışma sırasıdır.
- **Facade** karmaşık alt sistemi tek kapıdan sunar. **Unit of Work**'ün facade yönü tam olarak budur: dokuz repository'yi ayrı ayrı enjekte etmek yerine tek bir kapı.
- **Proxy** araya girip erişimi yönetir. EF Core'un lazy loading proxy'leri gerçek bir örnektir — ve N+1'in de kaynağıdır.
- **Strategy**, OCP'nin uygulanmış hâlidir: `switch` yerine arayüz, seçimi DI yapar.
- **Observer**, .NET'te `event` ve `IObservable<T>` olarak zaten dilin içindedir. Abone olup çıkmamak bellek sızıntısı üretir.
- **Template Method** abstract class'ın asıl kullanım yeridir: iskelet sabit, adımlar değişken. Strategy'den farkı, değişkenliğin kalıtımla değil kompozisyonla gelmesi.
- **Chain of Responsibility**, **ASP.NET Core middleware pipeline'ının ta kendisidir.** Her halka isteği ya işler ya bir sonrakine geçirir.
- **Iterator** ise `IEnumerable<T>` ve `yield return` olarak dilin parçası.
- **Pattern hastalığı** gerçek bir risktir: üç satırlık bir `if` için Strategy kurmak kodu iyileştirmez, gizler.

```csharp
// Decorator: log ve cache katmanları, asıl sınıfa dokunmadan
public class LoglayanUrunRepository(IUrunRepository ic, ILogger<LoglayanUrunRepository> log)
    : IUrunRepository
{
    public Urun? Getir(int id)
    {
        log.LogInformation("Ürün isteniyor: {Id}", id);
        return ic.Getir(id);
    }
}
```

```csharp
// Chain of Responsibility = middleware
app.Use(async (ctx, next) =>
{
    // gidiş yönü
    await next();          // zincirin geri kalanı
    // dönüş yönü
});
```

---

## 7. Bağlantı Haritası

Bu haftanın konuları birbirinden bağımsız değil. Bağlar şöyle:

| Bu konu | Şuna açılıyor |
|---|---|
| LSP ihlali | Kalıtım yerine kompozisyon (Not 2) |
| OCP | Strategy kalıbı (Not 6) |
| Kompozisyon + davranışı parametre geçme | Strategy, Decorator (Not 6) |
| DIP | DI (Not 4) → ASP.NET Core konteyneri → `Hafta-04/02-Program-cs-ve-Pipeline.md` |
| DI yaşam süreleri | Captive dependency, Singleton tuzağı (Not 5) |
| Chain of Responsibility | Middleware pipeline → `Hafta-04/02-Program-cs-ve-Pipeline.md` |
| Facade | Unit of Work → `03-Projeler/01-MvcCv/02-Alternatif-Yapilar.md` |
| Proxy | EF Core lazy loading → `Hafta-03/06-EF-Core-Performans.md` |
| Iterator | `IEnumerable`, `yield` → `Hafta-01/02-Koleksiyonlar.md` |
| Template Method | `abstract class`'ın doğru kullanımı (Not 1) |
| Abstract Factory | `IServiceProvider` → DI konteyneri |

**Hafta 1'e geri bakış:** Generic'ler, delegate/`Func`/`Action` ve extension method'lar (`Hafta-01/06-Dil-Altyapisi.md`) bu haftanın alt yapısıydı. Strategy'yi `Func<>` ile yazmak, Decorator'ı generic ile kurmak oradan geliyor.

**Hafta 3'e köprü:** Repository ve Unit of Work, bu haftanın ilkelerinin veri erişimine uygulanmış hâlidir. Hafta 3'te EF Core'un `DbContext`'inin zaten bir Unit of Work, `DbSet<T>`'nin zaten bir Repository olduğunu göreceksin — ve "üstüne bir katman daha koymalı mıyım" sorusuna bu haftanın bilgisiyle cevap vereceksin.

---

## Kendini Yoklama

Aşağıdaki soruları nota bakmadan, yüksek sesle cevaplamayı dene. Takıldığın maddenin notuna dön.

1. `protected internal` ile `private protected` arasındaki fark nedir?
2. `virtual`/`override` ile `new` arasındaki davranış farkını bir örnekle anlatabilir misin?
3. `abstract class` yerine `interface` seçmen gereken üç durum say. Tersi için de üç durum.
4. Default interface method'lar abstract class'ın yerini neden almaz?
5. Constructor içinde sanal metot çağırmak neden tehlikelidir?
6. `Equals` override edip `GetHashCode` etmezsen ne kırılır?
7. "Kırılgan taban sınıf problemi" nedir — somut bir senaryo anlat.
8. Kalıtım mı kompozisyon mu kararını hangi soruyla veriyorsun?
9. SRP'nin "tek sorumluluk" ifadesi neden yanıltıcı? Doğru okunuşu ne?
10. OCP'ye uyduğunu nasıl anlarsın — hangi somut testle?
11. LSP ihlalinin üç biçimini sayabilir misin?
12. DIP ile DI arasındaki fark nedir? Birini yapıp diğerini yapmamak mümkün mü?
13. `IUrunRepository` arayüzü hangi katmanda durmalı, neden?
14. Service locator neden anti-pattern sayılıyor?
15. Factory Method ile Abstract Factory'nin farkı ne?
16. Kendi Singleton'ını yazmak yerine `AddSingleton` kullanmanın kazancı nedir?
17. Decorator'ları DI'da kaydederken sıra neden önemli?
18. ASP.NET Core middleware pipeline'ı hangi tasarım kalıbıdır?
19. Strategy ile Template Method arasındaki fark nedir?
20. "Pattern hastalığı" ne demek — kendi kodunda nasıl fark edersin?

---

## Tek Bakışta Özet

- OOP araçtır, SOLID disiplindir, pattern'ler disiplinin yerleşmiş çözümleridir. Sıra atlanmaz.
- Kapsülleme public field ile olmaz; property kullan.
- `new` ile gizleme polimorfizm değildir ve sessiz hata üretir.
- Durum taşıyacaksan abstract class, yetenek tanımlayacaksan interface.
- `Equals` ve `GetHashCode` birlikte override edilir.
- Kalıtım en sıkı bağdır; "her X gerçekten bir Y mi" testini geçmiyorsa kompozisyon kullan.
- LSP davranış meselesidir; derleyici ihlali yakalamaz.
- SRP "tek iş" değil, **tek değişim sebebi / tek aktör** demektir.
- OCP'ye uymanın somut testi: yeni davranış eklerken eski dosyayı açıyor musun?
- Spekülatif soyutlama OCP'nin değil, karmaşıklığın hizmetindedir.
- ISP: `NotImplementedException` gören arayüz bölünmelidir.
- DIP'te arayüz **tüketenin** katmanında tanımlanır.
- DIP ilke, DI tekniktir. İkisi aynı şey değildir.
- `new` yazmanın meşru yeri composition root'tur.
- Kendi Singleton'ın yerine `AddSingleton`; Singleton içinde Scoped için `IServiceScopeFactory`.
- Factory Method tek ürün, Abstract Factory ürün ailesi.
- Decorator = kompozisyonla davranış ekleme; kayıt sırası çalışma sırasıdır.
- Strategy = OCP'nin kod hâli; Chain of Responsibility = middleware.
- Kalıp, problemi çözmüyorsa kodu sadece gizler.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Kapsülleme | İç durumu dışarıdan saklayıp erişimi kontrollü hâle getirme |
| Method hiding | `new` ile taban sınıf metodunu gizleme — polimorfizm değil |
| Fragile base class | Taban sınıftaki değişikliğin türeyenleri sessizce bozması |
| Kompozisyon | Yeteneği kalıtımla almak yerine bir parça olarak içeride tutma |
| Delegasyon | İşi içerideki parçaya devretme |
| LSP | Alt tipin, üst tipin yerine sorunsuz kullanılabilmesi ilkesi |
| Ön koşul / son koşul | Metodun çalışması için gereken şart / çalıştıktan sonra garanti ettiği durum |
| SRP | Bir sınıfın değişmesi için tek bir sebep (tek aktör) olması |
| OCP | Genişlemeye açık, değişikliğe kapalı olma |
| Spekülatif soyutlama | İleride lazım olur diye kurulan, kullanılmayan soyutlama |
| ISP | Kimsenin kullanmadığı metodu uygulamaya zorlanmaması |
| Rol arayüzü | Tek bir yeteneği tanımlayan dar arayüz |
| DIP | Üst ve alt seviyenin ikisinin de soyutlamaya bağlı olması |
| DI | Bağımlılığın nesneye dışarıdan verilmesi tekniği |
| Composition root | Nesne grafiğinin kurulduğu, `new`'in meşru olduğu tek yer |
| Service locator | Bağımlılığı sınıf içinden konteynerden isteme — anti-pattern |
| Captive dependency | Uzun ömürlü servisin kısa ömürlü servisi esir alması |
| Tasarım kalıbı | Tekrarlayan tasarım problemine yerleşmiş çözüm ve ortak dil |
| Decorator | Aynı arayüzü uygulayıp davranış ekleyen sarmalayıcı |
| Facade | Karmaşık alt sistemi tek ve sade bir arayüzle sunma |
| Chain of Responsibility | İsteğin zincirde sırayla işlendiği davranışsal kalıp |
| YAGNI | "You Aren't Gonna Need It" — gerekmeden yapma ilkesi |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "SRP = sınıf tek iş yapsın" | SRP = sınıfın değişmesi için tek sebep (tek aktör) olsun |
| "`new` ile `override` benzer şeyler" | `new` polimorfizmi kırar; referans tipine göre metot seçilir |
| "Interface her zaman abstract class'tan iyidir" | Durum, ortak ctor mantığı ve gerçek "is-a" varsa abstract class doğrudur |
| "Default interface method geldi, abstract class'a gerek kalmadı" | Durum tutamaz, sınıf referansından görünmez — yerini almaz |
| "Kalıtım kod tekrarını önler, o yüzden iyidir" | Tekrarı kompozisyon da önler; kalıtım bunu en sıkı bağla yapar |
| "LSP'yi derleyici kontrol eder" | Derleyici imzayı kontrol eder, davranışı değil |
| "DIP ile DI aynı şey" | DIP ilke (bağımlılık yönü), DI teknik (nesneyi dışarıdan verme) |
| "Arayüzü uygulayan katman tanımlar" | Arayüz tüketenin katmanında tanımlanır; bağımlılık oku böyle ters döner |
| "Konteynerden `Resolve` çağırmak DI'dır" | O service locator'dır; bağımlılığı gizler |
| "Singleton bir tasarım kalıbı, o hâlde kullanmakta sakınca yok" | Gizli bağımlılık ve testi zorlaştırma sebebiyle çoğu zaman `AddSingleton` tercih edilir |
| "Factory Method ile Abstract Factory aynı" | Biri tek ürün, diğeri birbirine ait ürün ailesi üretir |
| "Ne kadar çok pattern o kadar iyi mimari" | Gereksiz kalıp karmaşıklığı gizler; YAGNI geçerlidir |

---

## Sonraki

→ `../Hafta-03-SQL-EFCore/01-Iliskisel-Modelleme-ve-Index.md` (Hafta 3 · Pazartesi)

Hafta 3, yol haritasındaki en yüksek getirili hafta: bootcamp'teki 20 projenin neredeyse tamamının altında SQL Server ve EF Core var.
