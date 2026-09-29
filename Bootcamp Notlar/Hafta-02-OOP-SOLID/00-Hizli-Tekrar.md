# Hafta 2 — Hızlı Tekrar: OOP, SOLID ve Tasarım Kalıpları

**Okuma süresi:** ~13 dk

**Ne işe yarar:** Bu dosya yeni bir şey öğretmez. Haftanın notlarındaki
`Tek Bakışta Özet` ve `Sık Karıştırılanlar` bölümlerinin tek yerde toplanmış
hâlidir — zaten okuduğun şeyin hatırlatıcısı.

**Nasıl kullanılır:**

1. Baştan sona oku. Her maddede kendine sor: *"bunu birine anlatabilir miyim?"*
2. Cevap "hayır" olan maddenin üstündeki dosya adını not et.
3. Sadece o konunun tam nottaki bölümüne dön. Notu baştan okuma.

> Tekrar okumak, bildiğini sanmakla bilmeyi karıştırmanın en kolay yoludur.
> Bir maddeyi görünce "biliyorum" diye geçme — önce kapalı gözle anlatmayı dene,
> sonra satıra bak. Aradaki fark, gerçekten ne bildiğindir.

Terim aradığında bu dosya değil, yanındaki [`00-Terimler-Sozlugu.md`](00-Terimler-Sozlugu.md) açılır.

---

## Bu Dosyada Ne Var

1. **OOP Dört Sütun, Pratikte** — `01-OOP-Pratikte.md` · Pazartesi
2. **Kalıtım Yerine Kompozisyon** — `02-Kompozisyon-ve-Kalitim.md` · Salı
3. **SOLID 1-3: SRP, OCP, LSP** — `03-SOLID-SRP-OCP-LSP.md` · Çarşamba
4. **SOLID 4-5: ISP, DIP ve Dependency Injection** — `04-SOLID-ISP-DIP-ve-DI.md` · Perşembe
5. **Yaratımsal Tasarım Kalıpları** — `05-Creational-Patterns.md` · Cuma
6. **Yapısal ve Davranışsal Kalıplar** — `06-Structural-ve-Behavioral-Patterns.md` · Cumartesi
7. **Hafta Özeti: OOP, SOLID ve Tasarım Kalıpları** — `07-Hafta-Ozeti.md` · Pazar

---

## 1. OOP Dört Sütun, Pratikte

*Kaynak: [`01-OOP-Pratikte.md`](01-OOP-Pratikte.md) · Pazartesi*

- Dört sütun ezberlenecek tanım değil; her biri somut bir bakım sorununun çözümüdür.
- Kapsülleme veriyi saklamaz, veriye giden yolu tek kapıya indirir — kural tek yerde kalır.
- Public field yazma; auto-property bugün aynı, yarın farklıdır. Türetilmiş değeri saklama, computed property ile hesapla; içinde sorgu olan property zaten bir metottur.
- Constructor içinde sanal üye çağırma — türeyenin alanları henüz atanmamıştır.
- `override` nesnenin tipine, `new` değişkenin tipine bakar; `new` çoğu zaman sessiz hatadır.
- Arayüz yapabilirlik bildirir, soyut sınıf kimlik ve ortak altyapı verir; default interface method sürüm aracıdır, tasarım aracı değil.
- `Equals` ezersen `GetHashCode` da ez; hash'i **değişmeyen** alanlardan üret, `HashCode.Combine` kullan.
- `class` referans eşitliği, `record` değer eşitliği yapar; `record` ayrıca `with`, `ToString` ve deconstruct getirir.
- `sealed` şüphedeyken doğru varsayılandır: sözleşmeyi netleştirir, sonradan kaldırmak kimseyi bozmaz.
- Tip başına `switch` çoğalıyorsa davranışı tipe taşı; davranış işleyen tarafa aitse switch kalsın.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Kapsülleme güvenlik sağlar" | Bakım kolaylığı sağlar; `private` alan yansıma ile okunur |
| "`protected internal` dar, `private protected` geniş" | Tam tersi — `private` geçen daha dardır |
| "`new` ile metodu ezmiş olurum" | Ezmezsin, gizlersin; taban tip üzerinden çağrıda eski metot çalışır |
| "Constructor da devralınır" | Devralınmaz; `base(...)` ile çağrılır |
| "Taban constructor'dan sanal metot çağırmak güvenli" | Türeyenin alanları henüz atanmamıştır — `NullReferenceException` |
| "Default interface method geldi, soyut sınıfa gerek kalmadı" | Arayüz hâlâ durum tutamaz, ctor'ı yoktur, default metot sınıf referansından görünmez |
| "`Equals` ezmek yeterli" | `GetHashCode` da ezilmeli; yoksa sözlük ve küme davranışı bozulur |
| "`sealed` esnekliği öldürür, yazmamak daha güvenli" | `sealed` eklemek zor, kaldırmak kolaydır; şüphedeyken yazmak daha az risklidir |

---


---

## 2. Kalıtım Yerine Kompozisyon

*Kaynak: [`02-Kompozisyon-ve-Kalitim.md`](02-Kompozisyon-ve-Kalitim.md) · Salı*

- Kalıtım, iki sınıf arasında kurulabilecek en sıkı bağdır; kompozisyonun bağı tek satırlık bir sözleşmedir.
- Kırılgan taban sınıf: taban sınıftaki masum bir değişiklik türeyenleri sessizce bozar; derleyici yakalamaz.
- "is-a" cümlesinin doğruluğu yetmez; yerine geçebilme, yüzey uyumu ve ilişkinin ömrü de sorulur.
- İlişki çalışma zamanında değişiyorsa kalıtım kuramazsın; roller ve durumlar kompozisyonla tutulur.
- LSP: alt tip, üst tipin sözünü daraltamaz. `NotSupportedException` fırlatan `override` açık ihlaldir.
- LSP ihlalinde önce taban tipin fazla geniş söz verip vermediğine bak — suçlu genelde odur.
- Kısıtı taban sınıfa **soru olarak** taşı (`CekilebilirMi`), çağıran kod dürüst bir yüzeyle konuşsun.
- Değişen tek davranış parçasını sınıf değil parametre yap: `Func<>`, `Action<>` ya da küçük arayüz.
- `sealed` şüphedeyken doğru varsayılandır; kaldırmak kolay, eklemek yıkıcıdır. Devirtualization ve ucuz tip kontrolü yan kazançtır.
- Extension method sanal değildir; polimorfik davranış için kullanma.
- Büyük arayüzü yeteneklere böl; ölçüt boyut değil, birlikte değişme.
- Kalıtım framework tiplerinde, template method'da ve gerçek soyut hiyerarşilerde hâlâ doğru araçtır.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Kod tekrarını önlemek için kalıtım kurulur" | Tekrar için kompozisyon ya da yardımcı metot; kalıtım bir kimlik beyanıdır |
| "is-a cümlesi doğruysa kalıtım doğrudur" | Yerine geçebilme, yüzey uyumu ve ilişkinin ömrü de sağlanmalı |
| "Kare bir dikdörtgendir, kalıtım doğaldır" | Değiştirilebilir bir dikdörtgende kare LSP'yi bozar; ortak arayüz kullan |
| "`NotSupportedException` fırlatmak geçerli bir çözümdür" | Yüzeyin yanlış olduğunun işaretidir; arayüzü böl |
| "`sealed` esnekliği öldürür" | Arayüz, kompozisyon ve extension method açık kalır; kapanan tek kapı kalıtımdır |
| "Extension method ile polimorfik davranış yazılabilir" | Uzantılar statik tipe göre çözülür; sanal değildir |
| "Derin hiyerarşi iyi tasarım göstergesidir" | Üçten derin hiyerarşi bakım maliyeti üretir, değer üretmez |
| "Kalıtım artık kullanılmamalı" | Framework tipleri, template method ve gerçek soyut hiyerarşilerde doğru araçtır |
| "Default interface method çoklu kalıtım getirdi" | Durum tutamaz, ctor'ı yoktur, sınıf referansından görünmez |

---


---

## 3. SOLID 1-3: SRP, OCP, LSP

*Kaynak: [`03-SOLID-SRP-OCP-LSP.md`](03-SOLID-SRP-OCP-LSP.md) · Çarşamba*

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

### Sık karıştırılanlar

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


---

## 4. SOLID 4-5: ISP, DIP ve Dependency Injection

*Kaynak: [`04-SOLID-ISP-DIP-ve-DI.md`](04-SOLID-ISP-DIP-ve-DI.md) · Perşembe*

- ISP, arayüzü **kullananı** korur: kimse kullanmadığı metoda bağlı kalmamalı.
- Şişman arayüzün belirtisi boş gövde ve `NotSupportedException`'dır; aynı anda LSP ihlalidir.
- Rol arayüzü tek bir yeteneği tanımlar; bir sınıf birden fazla rolü üstlenebilir.
- En pratik bölme okuma/yazma ayrımıdır: `IUrunOkuyucu` verirsen o kod veri silemez.
- ISP de abartılır: her metoda ayrı arayüz açmak bölme değil gürültüdür.
- DIP: üst seviye alt seviyeye değil, **ikisi de soyutlamaya** bağlanır.
- Soyutlama **tüketicinin** katmanında tanımlanır; arayüzü sağlayıcının yanına koyarsan ok tersine dönmez.
- İyi bir arayüz arkasındaki teknolojiyi sızdırmaz — `SqlDataReader` döndüren arayüz kötü arayüzdür.
- DI'ın üç yolu: constructor (varsayılan), property (gerçekten opsiyonel olan), method (çağrı başına değişen).
- Constructor parametre sayısı bedava bir SRP ölçüsüdür; dörtten fazlaysa sınıfa tekrar bak.
- `new`, composition root'ta toplanır: `Program.cs` ya da `Main`. DTO ve varlık `new`'lemek serbesttir.
- Service locator bağımlılığı gizler, hatayı çalışma anına erteler ve kötü tasarımın acısını yok eder.
- DIP ile DI aynı şey değildir: DIP "neye bağlısın", DI "nasıl teslim edildi" sorusudur.
- Somut tipi constructor'dan almak DI'dır ama DIP değildir — en sık görülen yarım uygulama budur.
- Konteyner sihir değil: bir sözlük, bir özyinelemeli çözücü ve biraz reflection.
- ASP.NET Core'da üç yaşam süresi vardır; singleton'a scoped enjekte etmek captive dependency üretir.
- DI'ın asıl kazancı testtir: bağımlılık dışarıdan geliyorsa yerine sahtesini koyabilirsin.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "DIP ile DI aynı şeydir" | DIP tasarım ilkesi, DI teslim tekniğidir |
| "Constructor'dan alıyorsam DIP'e uyuyorum" | Somut tip alıyorsan DI var, DIP yok |
| "DI demek konteyner demektir" | Konteyner sadece kolaylıktır; elle DI da DI'dır |
| "Arayüz veri erişim projesinde durur" | Arayüz tüketicinin katmanında tanımlanır |
| "Her sınıfın arayüzü olmalı" | Dış dünyaya dokunan ve değişmesi muhtemel olanların |
| "`IServiceProvider` enjekte etmek DI'dır" | Service locator'dır; bağımlılığı gizler |
| "`new` yazmak yasaktır" | DTO, varlık ve koleksiyon her yerde `new`'lenir |
| "ISP arayüzü uygulayanı korur" | Asıl koruduğu arayüzü **kullanan** taraftır |
| "Singleton en performanslı seçenektir" | Scoped servisi içine alırsa captive dependency üretir |
| "`AddDbContext` singleton yapılabilir" | Scoped olmalı; paylaşılan `DbContext` eşzamanlılıkta patlar |
| "Test için mutlaka Moq gerekir" | Arayüzü uygulayan küçük bir sınıf çoğu zaman yeter |
| "Birim testi geçtiyse kod doğrudur" | Sahteler varsayımını doğrular, gerçek altyapıyı değil |

---


---

## 5. Yaratımsal Tasarım Kalıpları

*Kaynak: [`05-Creational-Patterns.md`](05-Creational-Patterns.md) · Cuma*

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

### Sık karıştırılanlar

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


---

## 6. Yapısal ve Davranışsal Kalıplar

*Kaynak: [`06-Structural-ve-Behavioral-Patterns.md`](06-Structural-ve-Behavioral-Patterns.md) · Cumartesi*

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

### Sık karıştırılanlar

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


---

## 7. Hafta Özeti: OOP, SOLID ve Tasarım Kalıpları

*Kaynak: [`07-Hafta-Ozeti.md`](07-Hafta-Ozeti.md) · Pazar*

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

### Sık karıştırılanlar

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


---

## Sonraki

Bulanık kalan madde varsa yukarıdaki kaynak satırından dosya adını al ve
sadece o bölümü oku. Haftanın tamamını yeniden okumana gerek yok.

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
