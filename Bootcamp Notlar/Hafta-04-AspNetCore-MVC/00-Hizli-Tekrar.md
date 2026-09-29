# Hafta 4 — Hızlı Tekrar: ASP.NET Core MVC

**Okuma süresi:** ~10 dk

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

1. **Hafta 4 · MVC Mimarisi ve Razor Görünüm Motoru** — `01-MVC-ve-Razor.md`
2. **Hafta 4 · `Program.cs`, Middleware Pipeline ve DI Yaşam Döngüleri** — `02-Program-cs-ve-Pipeline.md`
3. **Hafta 4 · Routing, Model Binding ve Dönüş Tipleri** — `03-Routing-ve-Model-Binding.md`
4. **Hafta 4 · EF Core ile CRUD ve Change Tracking** — `04-EF-Core-CRUD-ve-Change-Tracking.md`
5. **Action Sonuçları: `IActionResult` ve `ActionResult<T>`** — `05-Action-Sonuclari.md` · Ek Not

---

## 1. Hafta 4 · MVC Mimarisi ve Razor Görünüm Motoru

*Kaynak: [`01-MVC-ve-Razor.md`](01-MVC-ve-Razor.md)*

- **Model** veri + iş mantığı, **View** sadece gösterim, **Controller** trafik yönetimi. Controller'ın şişmesi yanlış katman işaretidir.
- **Razor** = C# + HTML. Tek kural `@`. Bastığı her değeri HTML-encode eder (XSS koruması). Hesap yapıyorsan parantez kullan: `@(adet * fiyat)`.
- Veri taşımada üç yol: **strongly-typed model** (tip güvenli, asıl veri için), **ViewBag** ve **ViewData** (tip güvensiz, küçük yan bilgiler için). `ViewBag` = `ViewData`'nın dynamic hâli. Üçünün de ömrü tek istektir.
- **Entity ≠ ViewModel.** Entity veritabanını, ViewModel ekranı temsil eder. Ayrım hem güvenlik hem esneklik sağlar; bedeli iki sınıfı senkron tutmaktır.
- **Tag helper** (`asp-for`) bugünün yöntemi; **Html helper** (`@Html.TextBoxFor`) eski kodda karşına çıkar. Tag helper'lar `_ViewImports.cshtml`'deki `@addTagHelper` satırına bağlıdır.
- **Partial view** HTML tekrarı içindir; veri gerekiyorsa **ViewComponent**.
- Katman ayrımının somut kazancı: aynı anda çalışabilmek ve iş mantığını sunucu ayağa kaldırmadan test edebilmek.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "ViewBag ile ViewData farklı şeyler" | Aynı veriyi taşırlar; ViewBag, ViewData'nın `dynamic` sarmalayıcısıdır |
| "`@model` ve `@Model` aynı" | Küçük m tipi tanımlar, büyük M nesneye erişir |
| "Entity'yi view'a göndermek pratiktir" | Küçük projede çalışır; güvenlik ve esneklik bedeli vardır |
| "Tag helper Html helper'ın yerine geçti, eskisi çalışmaz" | İkisi de çalışır; tag helper tercih edilendir |
| "Partial view kendi verisini çekebilir" | Çekemez — o ViewComponent'in işidir |
| "Razor XSS'e karşı korumasız" | `@` ile basılan her değer otomatik encode edilir |
| "`@adet * fiyat` çarpımı basar" | Basmaz; yalnızca `adet` C# sayılır. Parantez gerekir: `@(adet * fiyat)` |
| "Razor kodu tarayıcıda çalışır" | Sunucuda çalışır; tarayıcıya yalnızca üretilmiş HTML gider |
| "ViewBag ile bir sonraki isteğe veri taşınır" | Taşınmaz; ömrü tek istektir. O iş `TempData` veya session işidir |

---


---

## 2. Hafta 4 · `Program.cs`, Middleware Pipeline ve DI Yaşam Döngüleri

*Kaynak: [`02-Program-cs-ve-Pipeline.md`](02-Program-cs-ve-Pipeline.md)*

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

### Sık karıştırılanlar

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


---

## 3. Hafta 4 · Routing, Model Binding ve Dönüş Tipleri

*Kaynak: [`03-Routing-ve-Model-Binding.md`](03-Routing-ve-Model-Binding.md)*

- **Convention-based routing** tek şablonla tüm uygulamayı yönetir; **attribute routing** URL'i action'ın üstünde tanımlar ve SEO dostu adresler için kullanılır.
- `{id:int}` gibi **route kısıtları** metoda hiç girmeden yanlış tipi eler — doğrulama değil, eşleşme şartıdır.
- **Model binding** ham HTTP verisini C# nesnesine çevirir; `name` değeri özellik adıyla birebir eşleşmelidir. Eşleşmeyen alan hata vermez, sessizce boş kalır.
- Veri üç yerden gelir: **`[FromRoute]`** (URL), **`[FromQuery]`** (sorgu), **`[FromBody]`** (JSON gövde). Form POST'u için öznitelik gerekmez. `[FromBody]` bir action'da yalnızca bir kez kullanılır.
- **Over-posting**: entity'yi doğrudan parametre yapmak, formda olmayan alanların gönderilmesine izin verir. Çözüm DTO — var olmayan alan doldurulamaz.
- **`IActionResult`** bir arayüzdür; aynı metodun duruma göre HTML, 404, redirect döndürebilmesini sağlar.
- **`ActionResult<T>`** Web API'lerde tercih edilir: hem hata sonuçları hem tip güvenliği hem Swagger desteği. Örtük dönüşüm sayesinde `return urun;` yazabilirsin.
- **Async kararı** tek soruya bağlıdır: dışarıdan bir şey bekliyor muyum? Beklemiyorsa senkron yaz.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Form POST'u için `[FromBody]` gerekir" | Form verisi JSON değildir; `[FromBody]` bozar. Öznitelik yazmamak veya `[FromForm]` doğrudur |
| "`IActionResult` her yerde en iyisidir" | Web API'de `ActionResult<T>` tip güvenliği ve Swagger desteği verir |
| "401 ile 403 aynı şey" | 401 "giriş yapmamışsın", 403 "yetkin yok" demektir |
| "Attribute routing convention'ın yerine geçer" | İkisi bir arada kullanılabilir; attribute yazılan action varsayılanı yok sayar |
| "Her metodu async yapmak performansı artırır" | CPU-bound işte gereksiz durum makinesi üretir, performansı düşürür |
| "Model binding her alanı güvenle doldurur" | Formda olmayan alanlar da doldurulabilir — over-posting riski |
| "Model binding eşleşmeyen alan için hata verir" | Hata vermez; alanı `null`/`default` bırakır ve metot yine çalışır |
| "Route kısıtı bir doğrulamadır" | Kısıt eşleşme şartıdır; sağlanmazsa metoda hiç girilmez, 404 döner |
| "`[FromBody]` gönderilen veriyi gizler" | Gövde de düz metindir; gizlilik HTTPS ile sağlanır |
| "`NotFound()` çağrıldığı anda 404 gönderilir" | Bir sonuç nesnesi üretir; yanıtı çatı, metot döndükten sonra yazar |

---


---

## 4. Hafta 4 · EF Core ile CRUD ve Change Tracking

*Kaynak: [`04-EF-Core-CRUD-ve-Change-Tracking.md`](04-EF-Core-CRUD-ve-Change-Tracking.md)*

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

### Sık karıştırılanlar

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


---

## 5. Action Sonuçları: `IActionResult` ve `ActionResult<T>`

*Kaynak: [`05-Action-Sonuclari.md`](05-Action-Sonuclari.md) · Ek Not*

- Action metodu HTTP cevabını **yazmaz**, cevabı **tarif eden bir nesne** döndürür; yazma işini framework yapar.
- `IActionResult` bir **arayüzdür**: "bir HTTP cevabı döneceğim, çeşidi değişebilir".
- `ActionResult` o arayüzü uygulayan **soyut sınıftır**; hazır sonuç tiplerinin çoğu ondan türer.
- `ActionResult<T>` = "ya bir `T` ya da bir cevap nesnesi". Örtük dönüşüm sayesinde `return urun;` de `return NotFound();` de derlenir.
- `Value` ve `Result` `ActionResult<T>`'nin iki ayrı alanıdır; biri doluysa diğeri boştur.
- MVC'de pratikte hep `IActionResult`; API'de tek yol varsa somut tip, çok yol varsa `ActionResult<T>`.
- `XxxResult` sadece durum kodu, `XxxObjectResult` durum kodu + gövde döndürür.
- `Ok()` içerik uzlaşması yapar, `Json()` her zaman JSON üretir.
- 401 "kim olduğunu bilmiyorum", 403 "biliyorum ama yetkin yok".
- POST sonrası `RedirectToAction` (PRG); yönlendirmenin ötesine veri `TempData` ile taşınır.
- Somut tip döndüren bir metot `null` döndürürse 404 değil **204** üretir.
- Asenkronda sadece `Task<...>` sarmalı değişir; `async void` asla.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`IActionResult` ile `ActionResult` farklı davranır" | Davranış aynı; biri arayüz biri soyut sınıf. Fark `ActionResult<T>`'yi mümkün kılmasıdır |
| "`ActionResult<T>` sadece süs" | Swagger dokümanı, derleyici kontrolü ve test kolaylığı sağlar |
| "`return null;` 404 döndürür" | 204 No Content döndürür; 404 için `NotFound()` gerekir |
| "`Json()` ile `Ok()` aynı" | `Json()` içerik uzlaşmasını atlar, her zaman JSON üretir |
| "Hata varsa gövdeye `basarili=false` yazmak yeter" | Durum kodu da değişmeli; istemci önce koda bakar |
| "401 ve 403 aynı şey" | 401 kimlik yok, 403 kimlik var yetki yok |
| "`RedirectToAction` sonrası `ViewBag` taşınır" | Taşınmaz — yeni istek. `TempData` gerekir |
| "`View()` çağrısı sayfayı render eder" | Etmez; sadece render tarifini taşıyan `ViewResult` üretir |
| "Her hatayı `try/catch` ile 500'e çevirmek güvenli" | İstemci hatasını 500 yapmak yanlıştır; 4xx ayrıştırılmalı |
| "`async void` action bazen kullanılabilir" | Asla; istisna yakalanamaz, uygulama çöker |

---


---

## Sonraki

Bulanık kalan madde varsa yukarıdaki kaynak satırından dosya adını al ve
sadece o bölümü oku. Haftanın tamamını yeniden okumana gerek yok.

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
