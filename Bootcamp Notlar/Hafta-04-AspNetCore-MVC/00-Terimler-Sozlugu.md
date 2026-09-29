# Hafta 4 — Terimler Sözlüğü: ASP.NET Core MVC

**Ne işe yarar:** Bu dosya baştan sona okunmak için değil, **aranmak** için.
Haftanın 5 notundaki sözlükler burada birleştirildi: **94 terim**.
Aynı terim birden çok notta geçtiyse ilk tanımı alındı; "Nerede" sütunu
terimin ayrıntılı anlatıldığı dosyayı gösterir.

> Ctrl+F ile ara. Bir terimi bulamıyorsan başka haftanın sözlüğünde olabilir.

---

| Terim | Tanım | Nerede |
|---|---|---|
| `@Html.Raw` | Encoding'i devre dışı bırakıp ham HTML basma (dikkatli kullanılır) | `01-MVC-ve-Razor.md` |
| `@model` / `@Model` | View'ın tipini tanımlar / o tipteki nesneye erişir | `01-MVC-ve-Razor.md` |
| `[ApiController]` | Otomatik 400, kaynak binding ve ProblemDetails davranışını açan öznitelik | `05-Action-Sonuclari.md` |
| `[Bind]` | Model binding'in dolduracağı alanları sınırlama | `03-Routing-ve-Model-Binding.md` |
| `[FromForm]` / `[FromHeader]` / `[FromServices]` | Form gövdesinden / başlıktan / DI'dan alma | `03-Routing-ve-Model-Binding.md` |
| `[FromRoute]` / `[FromQuery]` / `[FromBody]` | URL yolundan / sorgudan / gövdeden okuma | `03-Routing-ve-Model-Binding.md` |
| `[ProducesResponseType]` | Yanıt tipini ve durum kodunu belgeye elle bildiren öznitelik | `03-Routing-ve-Model-Binding.md` |
| `[ValidateAntiForgeryToken]` | Formun gerçekten kendi sayfandan geldiğini doğrulama (CSRF koruması) | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Action result | Controller metodunun döndürdüğü, HTTP cevabını tarif eden nesne | `05-Action-Sonuclari.md` |
| `ActionResult<T>` | Hem hata sonucu hem tipli veri döndürebilen yapı | `03-Routing-ve-Model-Binding.md` |
| `ActionResult` | `IActionResult`'ı uygulayan soyut sınıf; hazır tiplerin atası | `05-Action-Sonuclari.md` |
| `AddScoped` | HTTP isteği başına tek nesne üreten kayıt | `02-Program-cs-ve-Pipeline.md` |
| `AddSingleton` | Uygulama boyunca tek nesne üreten kayıt | `02-Program-cs-ve-Pipeline.md` |
| `AddTransient` | Her istendiğinde yeni nesne üreten kayıt | `02-Program-cs-ve-Pipeline.md` |
| `app.Use` / `app.Run` / `app.Map` | Zincire halka ekler / zinciri bitirir / belirli yol için dal açar | `02-Program-cs-ve-Pipeline.md` |
| `AsNoTracking` | Takip listesine almadan salt okuma | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `asp-for` | Tag helper; `name` ve `id` değerlerini özellik adından otomatik üretir | `03-Routing-ve-Model-Binding.md` |
| `Attach` | Var olan bir kaydı `Unchanged` olarak takibe alma | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Attribute routing | Yolu action/controller üstünde öznitelikle tanımlama | `03-Routing-ve-Model-Binding.md` |
| Authentication | Kimlik doğrulama — "sen kimsin?" | `02-Program-cs-ve-Pipeline.md` |
| Authorization | Yetkilendirme — "bunu yapabilir misin?" | `02-Program-cs-ve-Pipeline.md` |
| `builder.Configuration` | appsettings, ortam değişkenleri ve secrets'ı birleştiren konfigürasyon | `02-Program-cs-ve-Pipeline.md` |
| Captive dependency | Uzun ömürlü servisin kısa ömürlü bağımlılığı tutması hatası | `02-Program-cs-ve-Pipeline.md` |
| Change tracking | EF Core'un nesnelerdeki değişiklikleri izlemesi | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `ChangeTracker` | Takip edilen tüm nesnelere ve durumlarına erişim noktası | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Concurrency | Aynı kaydın eşzamanlı değiştirilmesi sorunu | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Constructor injection | Bağımlılığın constructor parametresi olarak istenmesi | `02-Program-cs-ve-Pipeline.md` |
| Content negotiation | İstemcinin `Accept` başlığına göre format (JSON/XML) seçimi | `05-Action-Sonuclari.md` |
| Convention-based routing | Tek şablonla tüm uygulamayı yönlendirme | `03-Routing-ve-Model-Binding.md` |
| `CreatedAtAction` | 201 + `Location` başlığı üreten yardımcı metot | `05-Action-Sonuclari.md` |
| Dependency Injection | Bağımlılığın `new` ile değil dışarıdan sağlanması ilkesi | `02-Program-cs-ve-Pipeline.md` |
| `Detached` | EF Core'un tanımadığı, takip etmediği nesne | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| DI konteyneri | Servisleri kaydedip ihtiyaç duyulduğunda üreten altyapı | `02-Program-cs-ve-Pipeline.md` |
| DTO / InputModel | Yalnızca taşınacak alanları içeren, entity'den ayrı sınıf | `03-Routing-ve-Model-Binding.md` |
| Edit modeli / DTO | Yalnızca düzenlenecek alanları içeren, entity'den ayrı sınıf | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Entity | Veritabanı tablosunu temsil eden sınıf | `01-MVC-ve-Razor.md` |
| Entity state | Takip edilen nesnenin durumu (Added, Modified, Deleted...) | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `Entry(nesne)` | Tek bir nesnenin durumuna ve alan bilgilerine erişim | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `FindAsync` | Birincil anahtarla arama; önce hafızaya bakar | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Global query filter | Tüm sorgulara otomatik eklenen filtre | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Hard delete | Kaydı tablodan fiziksel olarak silme | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| HSTS | Tarayıcıya siteye hep HTTPS ile gelmesini söyleyen başlık | `02-Program-cs-ve-Pipeline.md` |
| HTML encoding | Metindeki HTML karakterlerinin zararsız hâle getirilmesi | `01-MVC-ve-Razor.md` |
| Html Helper | Metot çağrısıyla HTML üreten eski nesil yardımcı | `01-MVC-ve-Razor.md` |
| `HttpContext` | Bir isteğe ait tüm bilginin (istek, yanıt, kullanıcı) taşındığı nesne | `02-Program-cs-ve-Pipeline.md` |
| I/O-bound / CPU-bound | Dış kaynağı bekleyen / hesaplama yapan iş | `03-Routing-ve-Model-Binding.md` |
| `IActionResult` | Farklı HTTP yanıt tiplerini tek çatıda toplayan arayüz | `03-Routing-ve-Model-Binding.md` |
| `IgnoreQueryFilters` | Global filtreyi o sorgu için devre dışı bırakma | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Implicit conversion | Derleyicinin otomatik yaptığı tip dönüşümü — `ActionResult<T>`'nin temeli | `05-Action-Sonuclari.md` |
| `IServiceScopeFactory` | Elle scope açmayı sağlayan servis; singleton içinden scoped kullanmanın doğru yolu | `02-Program-cs-ve-Pipeline.md` |
| Kapsam doğrulama | `builder.Build()` sırasında captive dependency'yi yakalayan geliştirme kontrolü | `02-Program-cs-ve-Pipeline.md` |
| Middleware | İsteğin sırayla geçtiği pipeline katmanı | `02-Program-cs-ve-Pipeline.md` |
| Model binding | HTTP verisini C# parametre ve nesnelerine dönüştürme | `03-Routing-ve-Model-Binding.md` |
| `ModelState.IsValid` | Model binding ve doğrulama hatalarının toplu kontrolü | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| MVC | Veri, görünüm ve akış kontrolünü ayıran mimari desen | `01-MVC-ve-Razor.md` |
| `nameof` | Metot/özellik adını string olarak veren derleme anı işleci | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `next()` | Bir sonraki middleware'e devretme çağrısı | `02-Program-cs-ve-Pipeline.md` |
| `ObjectDisposedException` | Kapatılmış (dispose edilmiş) bir nesneyi kullanmaya çalışınca alınan istisna | `02-Program-cs-ve-Pipeline.md` |
| `ObjectResult` | Gövdeye nesne yazan, içerik uzlaşması yapan sonuç tipi | `05-Action-Sonuclari.md` |
| Open redirect | Kullanıcıyı dış siteye yönlendirmeye izin veren güvenlik açığı | `05-Action-Sonuclari.md` |
| Over-posting | Formdan, gönderilmemesi gereken alanların da gönderilmesi saldırısı | `01-MVC-ve-Razor.md` |
| Partial View | Tekrar eden HTML parçasının ayrı dosyaya çıkarılması | `01-MVC-ve-Razor.md` |
| Pipeline | Middleware'lerin oluşturduğu, iki yönlü çalışan zincir | `02-Program-cs-ve-Pipeline.md` |
| POST-Redirect-GET | Form gönderimi sonrası yönlendirerek tekrarı önleme | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| PRG (POST-Redirect-GET) | POST sonrası yönlendirme kalıbı; tekrar gönderimi engeller | `05-Action-Sonuclari.md` |
| `ProblemDetails` | RFC 7807 standardında hata gövdesi formatı | `05-Action-Sonuclari.md` |
| Projeksiyon (`Select`) | Yalnızca gereken kolonları çekme | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Razor | HTML içinde C# yazmayı sağlayan şablon motoru (`.cshtml`) | `01-MVC-ve-Razor.md` |
| Razor kod bloğu | `@{ ... }` — view içinde birden çok satır C# çalıştırma | `01-MVC-ve-Razor.md` |
| `RequestDelegate` | Zincirdeki bir sonraki katmanı temsil eden temsilci (delegate) tipi | `02-Program-cs-ve-Pipeline.md` |
| Rollback | Hata durumunda tüm değişikliklerin geri alınması | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Route constraint | Route parçasına tip veya biçim kısıtı (`{id:int}`) | `03-Routing-ve-Model-Binding.md` |
| Routing | URL'in hangi controller/action'a gideceğini belirleme | `03-Routing-ve-Model-Binding.md` |
| `SaveChanges` | İşaretli tüm değişiklikleri tek transaction'da yazma | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Scope (kapsam) | Bir HTTP isteğinin başından yanıt dönene kadarki süre | `02-Program-cs-ve-Pipeline.md` |
| Scoped yaşam süresi | `DbContext`'in istek başına oluşturulup istek bitince atılması | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Snapshot | Nesnenin çekildiği andaki değerlerinin saklanan kopyası | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| Soft delete | Kaydı silmek yerine pasif işaretleme (`IsDeleted`) | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| `StatusCodeResult` | Sadece durum kodu döndüren, gövdesiz sonuç tipi | `05-Action-Sonuclari.md` |
| Strongly-typed view | Tek bir tip üzerinden veri alan, tip güvenli view | `01-MVC-ve-Razor.md` |
| Swagger / OpenAPI | API'yi otomatik belgeleyen ve test ettiren araç | `03-Routing-ve-Model-Binding.md` |
| Tag Helper | HTML etiketi gibi görünen, sunucuda çalışan yardımcı (`asp-for`) | `01-MVC-ve-Razor.md` |
| `Task<IActionResult>` | Asenkron çalışıp bir HTTP yanıtı döndürecek metot | `03-Routing-ve-Model-Binding.md` |
| `TempData` | Bir sonraki isteğe kadar yaşayan geçici veri deposu | `05-Action-Sonuclari.md` |
| Thread güvenliği | Aynı nesnenin birden çok istek tarafından aynı anda güvenle kullanılabilmesi | `02-Program-cs-ve-Pipeline.md` |
| Transaction | Ya hepsi ya hiçbiri garantisi olan işlem paketi | `04-EF-Core-CRUD-ve-Change-Tracking.md` |
| ViewBag | Controller'dan view'a veri taşıyan `dynamic` nesne | `01-MVC-ve-Razor.md` |
| ViewComponent | Kendi verisini çekebilen, DI alabilen bağımsız görünüm bileşeni | `01-MVC-ve-Razor.md` |
| ViewData | Aynı işi yapan sözlük yapısı; ViewBag'in alt katmanı | `01-MVC-ve-Razor.md` |
| `_ViewImports.cshtml` | Tüm view'lara ortak `@using` / `@addTagHelper` satırlarını taşıyan dosya | `01-MVC-ve-Razor.md` |
| ViewModel / DTO | Yalnızca bir ekranın ihtiyacı olan veriyi taşıyan sınıf | `01-MVC-ve-Razor.md` |
| `WebApplicationBuilder` | Uygulamanın kurulum aşamasını yöneten nesne | `02-Program-cs-ve-Pipeline.md` |
| XSS | Sayfaya kod enjekte edip başka kullanıcının tarayıcısında çalıştırma saldırısı | `01-MVC-ve-Razor.md` |
| Örtük dönüşüm (implicit conversion) | Derleyicinin bir tipi başka bir tipe sessizce çevirmesi | `03-Routing-ve-Model-Binding.md` |

---

Haftanın özet ve tuzak listesi için: [`00-Hizli-Tekrar.md`](00-Hizli-Tekrar.md)

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
