# Hafta 4 · Ek Not — Action Sonuçları: `IActionResult` ve `ActionResult<T>`

**Okuma süresi:** ~45 dk
**Neden bu konu:** Bir controller metodu yazdığın her seferde ilk verdiğin karar "bu metot ne döndürecek?" olur. `void` mü, `IActionResult` mü, `ActionResult<T>` mü, doğrudan `Urun` mü? Bu seçim sadece bir tip meselesi değil: tarayıcıya hangi HTTP durum kodunun gideceğini, Swagger'ın dokümanı doğru üretip üretmeyeceğini ve test yazarken elinin ne kadar rahat olacağını belirler. MvcCv projende her yerde `IActionResult` kullandın; neden ve ne zaman başka bir şey gerektiğini bu not anlatıyor.

---

## Önce Basitçe

Bir controller metodunu, nüfus müdürlüğündeki memur olarak düşün. İnsanlar gelip bir şey ister: "bana ikametgâh belgesi verin", "şu kaydı güncelleyin", "şu listeyi göreyim". Memur her seferinde aynı şeyi vermez. Bazen eline **basılı bir belge** tutuşturur. Bazen "bu iş bizde değil, **üçüncü kata** gidin" der. Bazen "böyle bir kayıt **yok**" der. Bazen "formu **eksik doldurmuşsun**, düzelt de gel" der. Bazen hiçbir şey söylemeden sadece işlemi yapar ve "**tamam**" der.

Bunların hepsi birer **cevap**tır. Ama cevapların *türü* farklıdır: biri kâğıt, biri yönlendirme, biri ret, biri uyarı. Memurun ağzından çıkan şeyin ortak adı "cevap"tır — ama içeriği duruma göre değişir.

ASP.NET Core'da işte bu "cevap" kavramının adı **action result**tır. Controller metodun geriye bir action result döndürür; ASP.NET Core da onu alıp gerçek bir HTTP cevabına çevirir: durum kodu, başlıklar, gövde.

Sorun şu: C# statik tipli bir dildir. Bir metot "bazen belge, bazen yönlendirme, bazen ret" döndüremez — tek bir dönüş tipi yazmak zorundasın. Çözüm, hepsinin ortak üst tipini yazmaktır: **`IActionResult`**. Bu, "buradan bir cevap çıkacak, ama hangi çeşit olduğunu çalışma anında göreceksin" demektir.

Sonra bir ihtiyaç daha doğar. Web API yazarken cevabın içinde **veri** de vardır: bir ürün nesnesi, bir liste. `IActionResult` yazdığında derleyici ve Swagger o verinin ne olduğunu bilemez — çünkü tipte böyle bir bilgi yoktur. İşte bunun için **`ActionResult<T>`** vardır: "bu metot ya bir `T` döndürür ya da bir cevap nesnesi" demenin yolu. Hem esnekliği hem de tip bilgisini aynı anda verir.

> **Ana benzetme:** `IActionResult` = memurun elinden çıkan "bir cevap" — ama ne olduğu kâğıda bakmadan anlaşılmaz. `ActionResult<T>` = üzerinde "bu ya ikametgâh belgesidir ya da bir ret yazısıdır" diye önceden yazan zarf. İkincisi, zarfı açmadan da ne bekleyeceğini bilmeni sağlar.

Şimdi bu kavramların gerçek karşılıklarına, tip hiyerarşisine ve hangi durumda hangisinin seçileceğine inelim.

---

## Bu Notta Ne Var

1. Bir action metodu ne döndürebilir — üç seçenek
2. `IActionResult` nedir, nasıl çalışır
3. Somut sonuç tipleri ve üreten yardımcı metotlar
4. `ActionResult` (sınıf) ile `IActionResult` (arayüz) farkı
5. `ActionResult<T>` — tip bilgisi taşıyan sürüm
6. Doğrudan modeli döndürmek (`Urun`, `List<Urun>`)
7. MVC tarafı: görünüm, yönlendirme, dosya
8. API tarafı: durum kodu üreten metotlar
9. HTTP durum kodları — hangi durumda hangisi
10. Asenkron karşılıkları: `Task<IActionResult>`
11. Karar tablosu — hangi durumda hangisi
12. Yaygın hatalar ve tuzaklar

---

## 1. Bir Action Metodu Ne Döndürebilir

> **Benzetme —** Bir kargo şubesine paket bırakırsın. Şube sana ya paketin kendisini geri verir (yanlış adres), ya bir teslim fişi verir, ya da "bu paketi almıyoruz" der. Şubenin tabelasında ne yazdığına göre ne bekleyeceğini bilirsin: "sadece koli kabul edilir" yazıyorsa koli beklersin, "her türlü işlem" yazıyorsa ne çıkacağı belli değildir.

**Basitçe:** Controller metodunun dönüş tipi, ASP.NET Core'a "senden ne bekleyeyim" der. Üç ana seçenek vardır: somut bir veri tipi, `IActionResult`, ya da ikisini birleştiren `ActionResult<T>`.

**Teknik olarak:** ASP.NET Core bir action metodunun dönüş değerini üç kategoride ele alır.

| Dönüş tipi | Ne demek | Tipik kullanım |
|---|---|---|
| **Somut tip** (`Urun`, `List<Urun>`, `string`) | "Her zaman bu veriyi döndüreceğim" | Hata ihtimali olmayan basit API uçları |
| **`IActionResult`** | "Bir HTTP cevabı döndüreceğim, çeşidi değişebilir" | MVC controller'ları, çok yollu API uçları |
| **`ActionResult<T>`** | "Ya bir `T` ya da bir HTTP cevabı" | Modern Web API — ikisinin birleşimi |

Bunların asenkron karşılıkları da vardır: `Task<Urun>`, `Task<IActionResult>`, `Task<ActionResult<T>>`.

```csharp
// 1) Somut tip — tek yol var
public List<Urun> Listele() => _repo.List();

// 2) IActionResult — birden çok yol var
public IActionResult Getir(int id)
{
    var urun = _repo.Get(id);
    if (urun is null) return NotFound();     // bir çeşit cevap
    return Ok(urun);                          // başka bir çeşit cevap
}

// 3) ActionResult<T> — birden çok yol var AMA veri tipi de belli
public ActionResult<Urun> GetirTipli(int id)
{
    var urun = _repo.Get(id);
    if (urun is null) return NotFound();     // cevap nesnesi
    return urun;                              // doğrudan model — örtük dönüşüm
}
```

Üçüncüsünde `return urun;` yazabilmenin sebebi `ActionResult<T>`'nin **örtük dönüşüm operatörü** (implicit conversion) tanımlamasıdır. Derleyici `Urun`'ü otomatik olarak `ActionResult<Urun>`'e sarar.

---

## 2. `IActionResult` Nedir

> **Benzetme —** Bir lokantada "yemek" dersin. Yemek tek bir şey değildir: çorba da yemektir, pilav da, tatlı da. Garson "size yemek getireceğim" dediğinde ne geleceğini bilmezsin ama bir şeyin geleceğini bilirsin. Ortak sıfat budur: hepsi tabakta gelir, hepsi masaya konur. `IActionResult` de böyle bir ortak sıfattır: "ben HTTP cevabına dönüşebilen bir şeyim."

**Basitçe:** `IActionResult`, "beni çalıştırırsan ortaya bir HTTP cevabı çıkar" sözü veren bir arayüzdür. Tek bir işi vardır ve controller'ından döndürdüğün her cevap tipi bu sözü verir.

**Teknik olarak:** `IActionResult`, `Microsoft.AspNetCore.Mvc` altında tanımlı bir **arayüzdür (interface)**. Tek üyesi vardır:

```csharp
public interface IActionResult
{
    Task ExecuteResultAsync(ActionContext context);
}
```

Akış şöyle işler:

```
İstek gelir
   ↓
Routing → hangi controller, hangi action
   ↓
Model binding → parametreler doldurulur
   ↓
Action metodu çalışır → bir IActionResult döner
   ↓
Framework, o nesnenin ExecuteResultAsync'ini çağırır
   ↓
Durum kodu, başlıklar ve gövde HTTP cevabına yazılır
```

Yani action metodun **cevabı yazmaz**, cevabı *tarif eden bir nesne* döndürür. Asıl yazma işini framework yapar. Bu ayrım önemlidir: metodun `return NotFound();` dediğinde henüz hiçbir şey tarayıcıya gitmemiştir — sadece "404 döneceğiz" kararı bir nesne hâlinde paketlenmiştir.

> Bu ayrımın pratik faydası testte görülür. Bir action metodunu birim testinde çağırıp dönen nesnenin tipine bakabilirsin (`Assert.IsType<NotFoundResult>(sonuc)`) — HTTP sunucusu ayağa kaldırmana gerek kalmaz.

**Bu benzetme şurada bozulur:** Lokantada garson tabağı masaya kendi koyar. Burada ise action metodu tabağı hazırlayıp *mutfak penceresine* bırakır; masaya taşıma işini başka biri (framework) yapar. Bu yüzden `return` sonrasında hâlâ filtreler (action filter, result filter) devreye girebilir ve cevabı değiştirebilir.

---

## 3. Somut Sonuç Tipleri ve Yardımcı Metotlar

> **Benzetme —** Postanede farklı işlemler için farklı matbu formlar vardır: taahhütlü gönderi formu, iadeli taahhütlü formu, havale formu. Memur her seferinde sıfırdan kâğıt yazmaz, raftan doğru formu çeker. `View()`, `Ok()`, `NotFound()` gibi metotlar da o raftır — doğru form nesnesini senin yerine üretirler.

**Basitçe:** `IActionResult`'ı uygulayan onlarca hazır sınıf vardır. Bunları `new ViewResult()` diye elle kurmazsın; `ControllerBase` sınıfındaki yardımcı metotlar (`View()`, `Ok()`, `NotFound()`) senin yerine kurar.

**Teknik olarak:** `Controller` ve `ControllerBase` sınıfları, sonuç nesnelerini üreten kısa metotlar sunar.

| Yardımcı metot | Ürettiği tip | HTTP durumu | Nerede |
|---|---|---|---|
| `View()` / `View(model)` | `ViewResult` | 200 | MVC |
| `PartialView()` | `PartialViewResult` | 200 | MVC |
| `RedirectToAction()` | `RedirectToActionResult` | 302 | MVC |
| `Redirect(url)` | `RedirectResult` | 302 | MVC |
| `Content("metin")` | `ContentResult` | 200 | Her ikisi |
| `File(...)` | `FileResult` | 200 | Her ikisi |
| `Json(nesne)` | `JsonResult` | 200 | Her ikisi |
| `Ok()` / `Ok(nesne)` | `OkResult` / `OkObjectResult` | 200 | API |
| `Created(uri, nesne)` | `CreatedResult` | 201 | API |
| `CreatedAtAction(...)` | `CreatedAtActionResult` | 201 | API |
| `NoContent()` | `NoContentResult` | 204 | API |
| `BadRequest()` / `BadRequest(nesne)` | `BadRequestResult` / `BadRequestObjectResult` | 400 | API |
| `Unauthorized()` | `UnauthorizedResult` | 401 | Her ikisi |
| `Forbid()` | `ForbidResult` | 403 | Her ikisi |
| `NotFound()` / `NotFound(nesne)` | `NotFoundResult` / `NotFoundObjectResult` | 404 | Her ikisi |
| `Conflict()` | `ConflictResult` | 409 | API |
| `StatusCode(500)` | `StatusCodeResult` | serbest | Her ikisi |
| `Problem()` | `ObjectResult` (ProblemDetails) | 500 (varsayılan) | API |

**`XxxResult` ile `XxxObjectResult` farkı:** Parametresiz sürüm sadece durum kodu döndürür, gövde boştur. Nesne alan sürüm gövdeye veriyi de yazar.

```csharp
return NotFound();                     // 404, gövde boş
return NotFound("Ürün bulunamadı");    // 404, gövdede mesaj var
```

**`ObjectResult` — hepsinin atası:** Nesne alan sürümlerin tamamı `ObjectResult`'tan türer. `ObjectResult`, kendisine verilen nesneyi **content negotiation** (içerik uzlaşması) ile serileştirir: istemci `Accept: application/json` dediyse JSON, `application/xml` dediyse ve XML formatter kayıtlıysa XML üretir.

```csharp
// İkisi aynı şeyi yapar
return Ok(urun);
return new ObjectResult(urun) { StatusCode = 200 };
```

> `Json(nesne)` ile `Ok(nesne)` aynı şey değildir. `Json()` **her zaman JSON** üretir, içerik uzlaşmasını atlar. `Ok()` ise istemcinin istediği formatı dikkate alır. API yazıyorsan `Ok()` daha doğrudur; JSON'u zorlamak istediğin özel bir durum varsa `Json()` kullanılır.

---

## 4. `ActionResult` (Sınıf) ile `IActionResult` (Arayüz) Farkı

> **Benzetme —** "Taşıt" bir kavramdır (arayüz); "motorlu taşıt" ise o kavramı karşılayan bir sınıftır — bisiklet taşıttır ama motorlu taşıt değildir. Çoğu zaman "taşıt" demek yeter; ama bir şeyin motorlu olduğunu garanti etmen gereken bir yerde alt kümeyi yazarsın.

**Basitçe:** `IActionResult` arayüz, `ActionResult` ise o arayüzü uygulayan soyut bir sınıftır. Hazır sonuç tiplerinin neredeyse tamamı `ActionResult`'tan türer. Pratikte ikisi arasında fark hissetmezsin — fark, `ActionResult<T>`'yi mümkün kılmasıdır.

**Teknik olarak:** Hiyerarşi şöyledir:

```
IActionResult                 (arayüz)
   └── ActionResult           (soyut sınıf)
          ├── ObjectResult
          │      ├── OkObjectResult
          │      ├── NotFoundObjectResult
          │      └── BadRequestObjectResult
          ├── StatusCodeResult
          │      ├── OkResult
          │      ├── NotFoundResult
          │      └── NoContentResult
          ├── ViewResult
          ├── RedirectResult
          ├── ContentResult
          └── FileResult ...
```

| | `IActionResult` | `ActionResult` |
|---|---|---|
| Ne | Arayüz | Soyut sınıf |
| Uygulaması | `ExecuteResultAsync` | Aynı arayüzü uygular |
| Dönüş tipi olarak | Yaygın, esnek | Nadiren doğrudan yazılır |
| `ActionResult<T>` ile ilişkisi | — | `ActionResult<T>` bunun generic kardeşidir |

Dönüş tipi olarak hangisini yazsan da çalışır. Yaygın alışkanlık arayüzü (`IActionResult`) yazmaktır — "arayüze programla" ilkesine uyar ve `ActionResult`'tan türemeyen özel bir sonuç sınıfı yazarsan da uyumlu kalır.

---

## 5. `ActionResult<T>` — Tip Bilgisi Taşıyan Sürüm

> **Benzetme —** Kargo kutusunun üstünde "kırılacak eşya" yazması ile hiçbir şey yazmaması arasındaki fark. İçerik aynı olabilir; ama etiket varsa herkes — kargocu, alıcı, gümrük — ne beklediğini kutuyu açmadan bilir. `ActionResult<T>` o etikettir: cevabın içinde ne tür bir veri olacağını tipte söyler.

**Basitçe:** `IActionResult` "bir cevap döneceğim" der ama neyin döneceğini söylemez. `ActionResult<T>` "ya bir `T` ya da bir cevap nesnesi döneceğim" der. Bu sayede Swagger doğru dokümanı üretir, derleyici yanlış tipi yakalar, test yazmak kolaylaşır.

**Teknik olarak:** `ActionResult<T>` .NET Core 2.1 ile geldi. İki örtük dönüşüm operatörü tanımlar:

```csharp
public sealed class ActionResult<TValue> : IConvertToActionResult
{
    public static implicit operator ActionResult<TValue>(TValue value);
    public static implicit operator ActionResult<TValue>(ActionResult result);
}
```

Bu yüzden ikisi de derlenir:

```csharp
public ActionResult<Urun> Getir(int id)
{
    var urun = _repo.Get(id);
    if (urun is null)
        return NotFound();          // ActionResult → ActionResult<Urun>

    return urun;                    // Urun → ActionResult<Urun>
}
```

**Neden `IActionResult` yerine bunu kullanmalı (API'de):**

| Konu | `IActionResult` | `ActionResult<Urun>` |
|---|---|---|
| Swagger/OpenAPI dokümanı | "200 döner" — neyin döndüğü belirsiz | `Urun` şeması otomatik çıkar |
| `[ProducesResponseType]` gerekliliği | Şart | Başarı durumu için gerekmez |
| Derleyici kontrolü | Yok — yanlış tip döndürsen de derlenir | `return musteri;` derlenmez |
| Testte kullanım | Cast gerekir | `sonuc.Value` doğrudan erişilir |

```csharp
// IActionResult ile: Swagger'a tipi elle anlatmak gerekir
[ProducesResponseType(typeof(Urun), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public IActionResult Getir(int id) { ... }

// ActionResult<T> ile: başarı tipi zaten belli, sadece hata durumları yazılır
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<Urun> Getir(int id) { ... }
```

**Testte fark:**

```csharp
// IActionResult ile
var sonuc = controller.Getir(1);
var ok = Assert.IsType<OkObjectResult>(sonuc);
var urun = Assert.IsType<Urun>(ok.Value);

// ActionResult<T> ile
var sonuc = controller.Getir(1);
Assert.Equal("Klavye", sonuc.Value.Ad);       // Value zaten Urun tipinde
```

**Bu benzetme şurada bozulur:** Kargo etiketi kutunun dışındadır, içeriği değiştirmez. `ActionResult<T>` ise sadece etiket değil, aynı zamanda bir *kaptır* — `Value` ve `Result` diye iki ayrı alanı vardır. Model döndürdüysen `Value` dolu `Result` boş, `NotFound()` döndürdüysen `Result` dolu `Value` boştur. Test yazarken hangisinin dolu olduğunu kontrol etmen gerekir.

```csharp
var sonuc = controller.Getir(999);
Assert.Null(sonuc.Value);                      // model yok
Assert.IsType<NotFoundResult>(sonuc.Result);   // cevap nesnesi var
```

---

## 6. Doğrudan Modeli Döndürmek

> **Benzetme —** Bir esnafa "kaç para?" dediğinde "elli lira" der — pusula yazmaz, fatura kesmez, sadece cevabı söyler. Tek bir olası cevap varsa fazladan kâğıda gerek yoktur.

**Basitçe:** Metodun her koşulda aynı tipte veri döndürüyorsa, `IActionResult` sarmalamaya gerek yoktur. Doğrudan modeli döndür; ASP.NET Core onu 200 OK olarak paketler.

**Teknik olarak:** `[ApiController]` işaretli bir controller'da somut tip döndürmek geçerlidir:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UrunlerController : ControllerBase
{
    [HttpGet]
    public List<Urun> Listele() => _repo.List();       // her zaman 200 + liste

    [HttpGet("sayi")]
    public int Sayi() => _repo.List().Count;           // her zaman 200 + sayı
}
```

Framework dönen nesneyi `ObjectResult`'a sarar, içerik uzlaşmasını yapar ve 200 döndürür.

**Ne zaman yetmez:** Metodun iki farklı sonuç üretebiliyorsa (bulundu / bulunamadı, geçerli / geçersiz) somut tip yetmez. `null` döndürmek çözüm değildir:

```csharp
// KÖTÜ: kayıt yoksa 204 No Content döner — istemci bunu "boş ürün" sanır
public Urun Getir(int id) => _repo.Get(id);

// İYİ: niyet açık
public ActionResult<Urun> Getir(int id)
{
    var urun = _repo.Get(id);
    return urun is null ? NotFound() : urun;
}
```

> `[ApiController]` işaretli bir controller'da somut tip döndüren bir metot `null` döndürürse ASP.NET Core **204 No Content** üretir, 404 değil. Bu, istemci tarafında "kayıt yok" ile "kayıt boş" ayrımını imkânsız kılar. Kayıt bulunamama ihtimali varsa mutlaka `ActionResult<T>` kullan.

---

## 7. MVC Tarafı: Görünüm, Yönlendirme, Dosya

> **Benzetme —** Bir belediye binasında üç tip cevap alırsın: masaya oturtulup **form doldurtulur** (görünüm), "bu iş yan binada" denip **yönlendirilirsin** (redirect), ya da elinize bir **belge tutuşturulur** (dosya indirme). Üçü de geçerli cevaptır ama sonrasında ne yapacağın değişir.

**Basitçe:** MVC controller'larında cevap genellikle bir HTML sayfasıdır (`View`) ya da başka bir sayfaya yönlendirmedir (`RedirectToAction`). MvcCv projende neredeyse tamamı bu ikisidir.

**Teknik olarak:**

### `View` — görünüm döndürme

```csharp
public IActionResult Index()
{
    var liste = _repo.List();
    return View(liste);                    // Views/Ilgili/Index.cshtml
}

public IActionResult Detay(int id)
{
    return View("OzelGorunum", model);     // görünüm adını elle verme
}
```

`View()` çağrısı görünümü **o anda render etmez**; sadece "şu görünüm şu modelle render edilecek" bilgisini taşıyan bir `ViewResult` üretir. Render, `ExecuteResultAsync` sırasında olur.

### `RedirectToAction` — POST-Redirect-GET kalıbı

```csharp
[HttpPost]
public IActionResult Ekle(Urun urun)
{
    if (!ModelState.IsValid)
        return View(urun);                  // aynı sayfayı hatalarla geri göster

    _repo.Add(urun);
    return RedirectToAction("Index");       // 302 → tarayıcı Index'i GET'ler
}
```

Bu kalıbın adı **POST-Redirect-GET (PRG)**. POST'tan sonra doğrudan `View()` döndürürsen kullanıcı F5'e bastığında tarayıcı POST'u tekrarlar ve kayıt ikinci kez eklenir. `RedirectToAction` bunu engeller.

| Metot | Ne yapar |
|---|---|
| `RedirectToAction("Index")` | Aynı controller'ın Index'ine |
| `RedirectToAction("Index", "Urun")` | Urun controller'ının Index'ine |
| `RedirectToAction("Detay", new { id = 5 })` | Route değeriyle |
| `Redirect("/hakkimda")` | Ham URL'ye |
| `LocalRedirect(url)` | Sadece kendi siteye — açık yönlendirme koruması |
| `RedirectToRoute("rotaAdi", ...)` | İsimlendirilmiş rotaya |

> **Güvenlik notu:** Kullanıcıdan gelen bir URL'ye `Redirect(returnUrl)` ile yönlendirmek **open redirect** açığıdır — saldırgan kullanıcıyı kendi sitesine yollayabilir. Ya `LocalRedirect` kullan ya da `Url.IsLocalUrl(returnUrl)` ile kontrol et. MvcCv'de giriş sonrası yönlendirmede bu kontrol kullanılıyor.

### `PartialView` — parça görünüm

AJAX ile sayfanın bir bölümünü tazelerken kullanılır. Layout uygulanmaz, sadece parçanın HTML'i döner.

```csharp
public IActionResult UrunListesiParcasi()
{
    return PartialView("_UrunListesi", _repo.List());
}
```

### `Content` ve `File`

```csharp
return Content("Merhaba", "text/plain");                  // düz metin
return File(byteDizisi, "application/pdf", "rapor.pdf");  // indirme
return PhysicalFile(@"C:\dosyalar\a.pdf", "application/pdf");
return File(stream, "image/png");                          // stream'den
```

**Bu benzetme şurada bozulur:** Belediyede yönlendirildiğinde yeni binaya *sen* yürürsün. HTTP'de yönlendirmeyi **tarayıcı** yapar ve bu ikinci bir istek demektir — yani `RedirectToAction` sonrası `ViewBag`'e koyduğun her şey kaybolur. Yönlendirmenin diğer ucuna veri taşımak için `TempData` gerekir:

```csharp
TempData["Mesaj"] = "Kayıt eklendi";
return RedirectToAction("Index");
// Index görünümünde: @TempData["Mesaj"]
```

---

## 8. API Tarafı: Durum Kodu Üreten Metotlar

> **Benzetme —** Bir kurumla yazışırken cevap zarfının üstündeki **kaşe** her şeyi söyler: "kabul edildi", "reddedildi", "evrak eksik", "böyle bir dosya yok". Mektubu açmadan bile ne olduğunu anlarsın. HTTP durum kodu o kaşedir.

**Basitçe:** API'de cevabın gövdesi kadar durum kodu da önemlidir. İstemci önce koda bakar, sonra gövdeye. Doğru kodu döndürmek, istemcinin doğru davranmasını sağlar.

**Teknik olarak:** Tipik bir CRUD API'sinin tam hâli:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UrunlerController : ControllerBase
{
    private readonly IUrunRepository _repo;
    public UrunlerController(IUrunRepository repo) => _repo = repo;

    // GET api/urunler
    [HttpGet]
    public ActionResult<IEnumerable<Urun>> Listele()
        => _repo.List().ToList();                            // 200

    // GET api/urunler/5
    [HttpGet("{id:int}")]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<Urun> Getir(int id)
    {
        var urun = _repo.Get(id);
        if (urun is null)
            return NotFound(new { mesaj = $"{id} numaralı ürün yok" });   // 404
        return urun;                                                       // 200
    }

    // POST api/urunler
    [HttpPost]
    public ActionResult<Urun> Ekle(UrunEkleDto dto)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);                    // 400

        var urun = new Urun { Ad = dto.Ad, Fiyat = dto.Fiyat };
        _repo.Add(urun);

        // 201 + Location başlığı: yeni kaynağın adresi
        return CreatedAtAction(nameof(Getir), new { id = urun.Id }, urun);
    }

    // PUT api/urunler/5
    [HttpPut("{id:int}")]
    public IActionResult Guncelle(int id, UrunGuncelleDto dto)
    {
        if (id != dto.Id) return BadRequest("Id uyuşmuyor");  // 400
        if (_repo.Get(id) is null) return NotFound();          // 404

        _repo.Update(dto);
        return NoContent();                                    // 204
    }

    // DELETE api/urunler/5
    [HttpDelete("{id:int}")]
    public IActionResult Sil(int id)
    {
        var urun = _repo.Get(id);
        if (urun is null) return NotFound();                   // 404
        _repo.Delete(urun);
        return NoContent();                                    // 204
    }
}
```

**`CreatedAtAction` neden önemli:** 201 cevabına `Location` başlığı ekler — istemci yeni kaydın adresini öğrenir. `Created("/api/urunler/5", urun)` ile URL'yi elle yazmak da mümkündür ama rota değişirse bozulur; `CreatedAtAction` rota tablosundan üretir.

**`Unauthorized` ile `Forbid` farkı:**

| | Anlamı |
|---|---|
| `Unauthorized()` — 401 | "Kim olduğunu bilmiyorum" — giriş yapılmamış |
| `Forbid()` — 403 | "Kim olduğunu biliyorum ama yetkin yok" |

`Forbid()`, cookie authentication kullanıyorsan kullanıcıyı "erişim reddedildi" sayfasına yönlendirir; `Unauthorized()` ise giriş sayfasına. Karıştırılırsa kullanıcı sonsuz giriş döngüsüne girer.

**`Problem()` ve ProblemDetails:** RFC 7807 standardında hata gövdesi üretir. `[ApiController]` işaretli controller'larda 400 ve üstü cevaplar zaten otomatik olarak bu formata çevrilir.

```csharp
return Problem(
    detail: "Stok yetersiz",
    statusCode: StatusCodes.Status409Conflict,
    title: "İşlem tamamlanamadı");
```

---

## 9. HTTP Durum Kodları — Hangi Durumda Hangisi

> **Benzetme —** Hastanede triyaj renkleri gibi: yeşil, sarı, kırmızı. Renk tek başına hastalığı anlatmaz ama aciliyeti ve nereye gidileceğini söyler. 2xx/4xx/5xx de öyle: sorunun **kimde** olduğunu söyler.

**Basitçe:** 2xx "oldu", 3xx "başka yere bak", 4xx "senin hatan", 5xx "benim hatam". En sık yapılan hata, istemci hatasına 500 döndürmektir.

**Teknik olarak:**

| Kod | Adı | Ne zaman | Yardımcı metot |
|---|---|---|---|
| 200 | OK | Başarılı okuma/güncelleme, gövde var | `Ok(nesne)` |
| 201 | Created | Yeni kayıt oluşturuldu | `CreatedAtAction(...)` |
| 204 | No Content | Başarılı ama gövde yok (silme, güncelleme) | `NoContent()` |
| 301/302 | Redirect | Kalıcı / geçici yönlendirme | `RedirectPermanent` / `Redirect` |
| 400 | Bad Request | Gelen veri geçersiz | `BadRequest(ModelState)` |
| 401 | Unauthorized | Kimlik yok veya geçersiz | `Unauthorized()` |
| 403 | Forbidden | Kimlik var, yetki yok | `Forbid()` |
| 404 | Not Found | Kaynak yok | `NotFound()` |
| 409 | Conflict | Çakışma (mükerrer kayıt, eşzamanlılık) | `Conflict()` |
| 422 | Unprocessable | Sözdizimi doğru, iş kuralı ihlali | `UnprocessableEntity()` |
| 500 | Server Error | Beklenmeyen hata — **senin hatan** | `StatusCode(500)` |

**Kural:** İstemcinin düzeltebileceği her durum 4xx'tir. 500, "kodda bir şey patladı, kullanıcının yapabileceği bir şey yok" demektir. `try/catch` içinde her hatayı 500'e çevirmek, istemciden gerçek sebebi saklar.

```csharp
// KÖTÜ: her şey 500
try { ... } catch (Exception ex) { return StatusCode(500, ex.Message); }

// İYİ: ayrıştır
try
{
    ...
}
catch (KayitBulunamadiException)
{
    return NotFound();
}
catch (IsKuraliException ex)
{
    return UnprocessableEntity(ex.Message);
}
// Beklenmeyenleri hiç yakalama — global exception handler middleware halleder
```

> `ex.Message`'ı istemciye döndürmek bilgi sızdırır: bağlantı dizesi, tablo adı, dosya yolu içerebilir. Üretimde genel bir mesaj döndür, ayrıntıyı logla.

---

## 10. Asenkron Karşılıkları

> **Benzetme —** Memura "bu belgeyi hazırlayın" dersin, memur arşive gider. Sen kapıda bekleyebilirsin (senkron) ya da sıra numaranı alıp oturabilirsin (asenkron). Cevabın *türü* değişmez — sadece bekleme şeklin değişir.

**Basitçe:** Asenkron yaptığında dönüş tipini `Task<...>` ile sararsın, gerisi aynıdır. `IActionResult` → `Task<IActionResult>`, `ActionResult<T>` → `Task<ActionResult<T>>`.

**Teknik olarak:**

```csharp
public async Task<IActionResult> Index()
{
    var liste = await _repo.ListAsync();
    return View(liste);
}

public async Task<ActionResult<Urun>> Getir(int id)
{
    var urun = await _context.Urunler.FindAsync(id);
    if (urun is null) return NotFound();
    return urun;
}

// Veri tabanına gitmiyorsa async'e gerek yok
public IActionResult Hakkimda() => View();
```

| Senkron | Asenkron |
|---|---|
| `IActionResult` | `Task<IActionResult>` |
| `ActionResult<T>` | `Task<ActionResult<T>>` |
| `Urun` | `Task<Urun>` |
| `void` | `Task` |

**`ValueTask<ActionResult<T>>`** de geçerlidir ama controller seviyesinde kazancı ihmal edilebilir; okunabilirlik adına `Task` tercih edilir.

> **Asla `async void` yazma.** Action metodu `async void` olursa framework onu bekleyemez; istisna yakalanamaz ve uygulama çöker. Controller'da `async void` için geçerli hiçbir gerekçe yoktur.

---

## 11. Karar Tablosu — Hangi Durumda Hangisi

> **Benzetme —** Alet çantasından tornavida seçmek gibi. Vidanın başı hangi şekildeyse o tornavidayı alırsın. Yanlış olanı da bir şekilde çevirir ama vidanın başını sıyırır.

**Basitçe:** MVC'de neredeyse her zaman `IActionResult`. API'de tek yol varsa somut tip, birden çok yol varsa `ActionResult<T>`.

**Teknik olarak:**

| Durum | Seçim | Gerekçe |
|---|---|---|
| MVC controller, görünüm döndürüyor | `IActionResult` | Çoğu metot bazen `View` bazen `RedirectToAction` döndürür |
| MVC, sadece tek bir görünüm dönüyor ve koşul yok | `IActionResult` | Yine de; ileride koşul eklenince tip değişmesin |
| API, her koşulda aynı veri | Somut tip (`List<Urun>`) | En sade, en okunaklı |
| API, veri veya hata | `ActionResult<Urun>` | Tip bilgisi + esneklik |
| API, veri yok sadece durum | `IActionResult` | Döndürecek `T` yok (`NoContent`, `NoFound`) |
| Dosya indirme | `IActionResult` | `FileResult` tipi taşınacak veri değil |
| Birim testte dönüş verisine sık bakılıyor | `ActionResult<T>` | `.Value` ile doğrudan erişim |
| Swagger dokümanı önemli | `ActionResult<T>` | Şema otomatik çıkar |

**MvcCv projendeki durum:** Tüm controller'lar MVC tarafında ve `IActionResult` döndürüyor — bu doğru seçim. `ActionResult<T>`'yi bootcamp'te Web API konusuna geldiğinde kullanmaya başlayacaksın.

---

## 12. Yaygın Hatalar ve Tuzaklar

> **Benzetme —** Formu doğru doldurup yanlış gişeye vermek gibi. İçerik doğru, sonuç yanlış. Bu bölümdeki hatalar da öyle: kod çalışır, hata vermez, ama davranış yanlıştır.

**Basitçe:** Bu bölümdeki hataların çoğu derleyiciden geçer ve sessizce yanlış davranır. En tehlikeli hata türü budur.

**Teknik olarak:**

### 1. `null` döndürüp 404 sanmak

```csharp
public Urun Getir(int id) => _repo.Get(id);   // kayıt yoksa 204 döner, 404 değil
```
Çözüm: `ActionResult<Urun>` + `NotFound()`.

### 2. Görünüm döndürürken model tipini karıştırmak

```csharp
public IActionResult Index()
{
    return View(_repo.List());       // List<Urun> gönderiliyor
}
```
Görünümde `@model Urun` yazılıysa çalışma anında `InvalidOperationException` alırsın — derleyici yakalamaz. Görünümün `@model` satırıyla gönderdiğin tipin eşleştiğini kontrol et.

### 3. POST sonrası `View()` döndürüp PRG'yi atlamak

Kullanıcı F5'e bastığında kayıt ikinci kez eklenir. Başarılı POST'tan sonra **her zaman** `RedirectToAction`.

### 4. `RedirectToAction` sonrası `ViewBag` beklemek

Yönlendirme yeni bir istektir; `ViewBag` sıfırlanır. `TempData` kullan.

### 5. Hata durumunda 200 döndürmek

```csharp
// KÖTÜ: durum kodu 200 ama içerik hata
return Ok(new { basarili = false, mesaj = "Bulunamadı" });
```
İstemci `response.ok` kontrolüyle bunu başarı sanır. Durum kodu, gövdedeki bayraktan önce gelir.

### 6. `Json()` ile `Ok()` karıştırmak

`Json()` içerik uzlaşmasını atlar. API'de `Ok()` kullan.

### 7. `Forbid()` yerine `Unauthorized()` kullanmak

Yetkisi olmayan giriş yapmış kullanıcıya 401 dönersen, tarayıcı onu tekrar giriş sayfasına yollar; kullanıcı giriş yapar, yine 401 alır. Sonsuz döngü.

### 8. `[ApiController]` olmadan otomatik 400 beklemek

`[ApiController]` işareti yoksa `ModelState` geçersizken otomatik 400 üretilmez; `if (!ModelState.IsValid) return BadRequest(ModelState);` elle yazılmalıdır.

### 9. `StatusCode(200, nesne)` yazmak

`Ok(nesne)` zaten bunu yapar ve niyeti daha iyi anlatır. `StatusCode()`, standart yardımcısı olmayan kodlar için saklanmalıdır.

### 10. `ActionResult<T>` testinde `.Value`'yu doğrudan okumak

Metot `NotFound()` döndürdüyse `.Value` `null`'dur; `.Result` dolu olur. Testte önce hangisinin dolu olduğuna bak.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Action result | Controller metodunun döndürdüğü, HTTP cevabını tarif eden nesne |
| `IActionResult` | Tüm sonuç tiplerinin uyduğu arayüz; tek üyesi `ExecuteResultAsync` |
| `ActionResult` | `IActionResult`'ı uygulayan soyut sınıf; hazır tiplerin atası |
| `ActionResult<T>` | Ya `T` ya da bir sonuç nesnesi taşıyan generic sarmalayıcı |
| `ObjectResult` | Gövdeye nesne yazan, içerik uzlaşması yapan sonuç tipi |
| `StatusCodeResult` | Sadece durum kodu döndüren, gövdesiz sonuç tipi |
| Content negotiation | İstemcinin `Accept` başlığına göre format (JSON/XML) seçimi |
| Implicit conversion | Derleyicinin otomatik yaptığı tip dönüşümü — `ActionResult<T>`'nin temeli |
| PRG (POST-Redirect-GET) | POST sonrası yönlendirme kalıbı; tekrar gönderimi engeller |
| `TempData` | Bir sonraki isteğe kadar yaşayan geçici veri deposu |
| `ProblemDetails` | RFC 7807 standardında hata gövdesi formatı |
| `CreatedAtAction` | 201 + `Location` başlığı üreten yardımcı metot |
| Open redirect | Kullanıcıyı dış siteye yönlendirmeye izin veren güvenlik açığı |
| `[ProducesResponseType]` | Swagger'a hangi kodda hangi tipin döndüğünü bildiren öznitelik |
| `[ApiController]` | Otomatik 400, kaynak binding ve ProblemDetails davranışını açan öznitelik |

---

## Sık Karıştırılanlar

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

## Sonraki

→ Bu konu Hafta 4'ün ek notudur. Web API ve REST konusunda bootcamp'te tekrar karşına çıkacak; o zaman `ActionResult<T>`, `ProblemDetails` ve `[ProducesResponseType]` günlük kullanımın parçası olacak.

İlgili okumalar:
- `01-MVC-ve-Razor.md` — `View()` ve model geçişi
- `03-Routing-ve-Model-Binding.md` — parametrelerin nasıl dolduğu
- `04-EF-Core-CRUD-ve-Change-Tracking.md` — veriyi nereden aldığın
