# Hafta 4 · Routing, Model Binding ve Dönüş Tipleri

**Okuma süresi:** ~44 dk
**Neden bu konu:** Bir isteğin hangi metoda gideceği (routing), o metoda verinin nasıl ulaşacağı (model binding) ve geriye ne döneceği (dönüş tipleri) — bir web uygulamasının giriş ve çıkış kapıları. Üçü de bootcamp'in Web API projelerinde günlük iş olacak.

---

## Önce Basitçe

Tarayıcıdan bir adrese gittiğinde aslında sunucuya bir kâğıt uzatmış oluyorsun. Üstünde "şu adresi istiyorum, şu bilgileri de yanında getirdim" yazıyor. Sunucunun yapması gereken üç iş var ve bu notun tamamı bu üç işten ibaret.

Birinci iş: **bu kâğıt kime gidecek?** Sunucunun içinde yüzlerce metot var. Hangisinin bu adresle ilgilendiğini bulması lazım. Buna yönlendirme diyoruz. İkinci iş: **kâğıttaki bilgileri içeri geçirmek.** Adresin içinde bir numara, sonunda bir arama kelimesi, belki de kâğıdın arkasında bir sürü alan var. Bunların hepsi düz yazıdır; sunucu bunları kendi diline, yani C# değişkenlerine çevirmek zorundadır. Üçüncü iş: **cevabı hazırlamak.** Bazen bir sayfa, bazen "böyle bir kayıt yok" yazısı, bazen "sen buraya giremezsin" uyarısı, bazen de "şu adrese git" notu.

Bu üç işin can sıkıcı tarafı, üçünün de büyük ölçüde **otomatik** olmasıdır. Otomatik olan şeyi anlamak zordur, çünkü çalışırken görünmez; ancak bozulduğunda kendini gösterir. Adresini yanlış yazdığın bir metoda hiç girilmez ve 404 alırsın. Formdaki bir alanın adı bir harf farklıysa o alan sessizce boş gelir. Yanlış dönüş tipi seçersen tarayıcı sayfa beklerken JSON alır.

Bir de kimsenin ilk günden söylemediği bir şey var: bu otomatiklik **güvenlik tarafında da çalışır**. Sunucu, gönderdiğin her alanı doldurmaya çalışır. Sen forma koymadığın bir alanı biri elle ekleyip gönderirse, sistem onu da doldurur. "Formda yoktu ki" savunması işe yaramaz, çünkü sunucu formu görmez; yalnızca gelen veriyi görür.

> **Ana benzetme:** Bir isteğin yolculuğu, kalabalık bir devlet dairesine dilekçe vermeye benzer. Kapıdaki danışma seni doğru odaya yollar (routing), odadaki görevli dilekçendeki bilgileri kurumun kendi matbu formuna geçirir (model binding), sonunda eline bir cevap kâğıdı tutuşturulur — onay, "böyle bir kayıt yok" ya da "3. kata gidin" (dönüş tipi).

Şimdi bu üç kapıyı tek tek, detayıyla açıyoruz.

---

## Bu Notta Ne Var

1. Convention-based routing
2. Attribute routing
3. Model binding nedir
4. Verinin geldiği üç kaynak: `[FromRoute]`, `[FromQuery]`, `[FromBody]`
5. Over-posting ve korunma
6. `IActionResult` — neden somut tip değil
7. `ActionResult<T>` ve Web API farkı
8. Async her zaman gerekli mi

---

## 1. Convention-Based Routing

> **Benzetme —** Bir sitede adresler tek bir kurala göre yazılır: "Blok / Kat / Daire". Kimse tek tek tabela asmaz, çünkü kural herkes için aynıdır. Yeni bir blok yapıldığında da ayrıca bir şey yapman gerekmez; adres kuralı onu da kapsar. Kargocu da kuralı bildiği için, hiç görmediği bir daireyi bulabilir.

**Basitçe:** Uygulamanın tamamı için tek bir adres kalıbı yazarsın. Gelen her adres bu kalıba göre parçalanır: ilk parça hangi sınıf, ikinci parça hangi metot, üçüncü parça varsa numara. Yeni bir sayfa eklediğinde ayrıca adres tanımlaman gerekmez, kalıp onu da kapsar.

**Teknik olarak: Routing (yönlendirme)** — Gelen URL'in hangi controller ve action'a gideceğini belirleyen mekanizma.

ASP.NET Core MVC'nin varsayılanı **convention-based** (gelenek tabanlı) yönlendirmedir: tek bir şablon yazılır, tüm uygulama ona uyar.

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

Şablonun okunuşu:

| Parça | Anlamı |
|---|---|
| `{controller=Home}` | İlk segment controller adı; yazılmazsa `Home` |
| `{action=Index}` | İkinci segment action adı; yazılmazsa `Index` |
| `{id?}` | Üçüncü segment `id` parametresi; `?` isteğe bağlı demek |

| URL | Controller | Action | id |
|---|---|---|---|
| `/` | Home | Index | — |
| `/Urun` | Urun | Index | — |
| `/Urun/Detay` | Urun | Detay | — |
| `/Urun/Detay/5` | Urun | Detay | 5 |

Tablodaki ilk satır önemlidir: `/` adresinde hiçbir parça yazılmamıştır, ikisi de varsayılana düşer ve `HomeController.Index` çalışır. Uygulamanın "ana sayfası" dediğin şey işte bu varsayılanların sonucudur, ayrıca tanımlanmış bir şey değildir.

**Artısı:** Tek yerden yönetilir, tutarlıdır, yeni controller eklediğinde hiçbir şey yazman gerekmez.
**Eksisi:** URL yapısı klasör mantığına mahkûmdur. `site.com/Urun/Detay/5` yerine `site.com/magaza/urun/5` gibi bir adres istiyorsan yetmez.

> **Benzetme nerede bozulur:** Sitedeki daire numarası fiziksel bir gerçektir, değiştiremezsin. Route şablonu ise senin yazdığın bir metindir; istersen `{controller}/{action}` sırasını bile değiştirebilirsin. Yani burada "kural" doğa kanunu değil, senin tercihindir — ve bir sonraki başlıkta göreceğin gibi, tek tek tabela asma hakkın da saklıdır.

---

## 2. Attribute Routing

> **Benzetme —** Site kuralı "Blok/Kat/Daire" olsa da, zemin kattaki fırıncı kapısına kendi tabelasını asar: "Ahmet Usta Fırın". Müşteri artık "C Blok Kat 0 Daire 2" demez, tabeladaki adı kullanır. Fırıncının tabelası site kuralını ortadan kaldırmaz; sadece o kapı için geçerli olan, daha akılda kalıcı bir adres ekler.

**Basitçe:** Adresi tek bir merkezî kuralda değil, doğrudan metodun üstünde yazarsın. "Bu metot şu adresten çağrılır" dersin. Böylece adres, dosya ve sınıf adlarına bağlı kalmaz; istediğin gibi, insanın okuyup anlayacağı biçimde olur.

**Teknik olarak: Attribute routing** — Yönlendirmeyi doğrudan controller veya metodun üstüne öznitelik olarak yazmak. SEO dostu ve daha kontrollü URL'ler için kullanılır.

```csharp
[Route("magaza")]                         // controller seviyesinde ön ek
public class StoreController : Controller
{
    [Route("urun/{id:int}")]              // action seviyesinde
    public IActionResult ProductDetail(int id)
    {
        return View();
    }
}
```

Sonuç: `ornek.com/magaza/urun/5`

Controller seviyesindeki `[Route("magaza")]` bir **ön ek**tir; altındaki her action'ın yoluna eklenir.

Dikkat edilecek nokta: adres artık `StoreController` ve `ProductDetail` isimlerinden **bağımsız** hâle geldi. Sınıfın adını yarın `ShopController` yapsan bile dışarıdaki adres değişmez. Bu iyi bir şeydir — dışarıya verdiğin adresler, içerideki isimlendirme tercihlerinden etkilenmemelidir.

### Route kısıtları (constraints)

`{id:int}` yazımındaki `:int` bir **kısıttır**: yalnızca sayı kabul edilir. `/magaza/urun/abc` adresi bu route ile eşleşmez ve 404 döner — metoda hiç girilmez.

Buradaki "metoda hiç girilmez" ifadesi önemlidir. Kısıt bir doğrulama değil, bir **eşleşme şartıdır**. Doğrulama metodun içinde olur ve hata mesajı üretirsin; kısıt ise metodun kapısına bile gelmeden isteği eler.

| Kısıt | Anlamı |
|---|---|
| `{id:int}` | Tam sayı |
| `{id:guid}` | GUID |
| `{ad:alpha}` | Yalnızca harf |
| `{fiyat:min(0)}` | Minimum değer |
| `{slug:length(1,50)}` | Uzunluk aralığı |

Kısıt kullanmak, metodun içinde tip kontrolü yazma ihtiyacını ortadan kaldırır.

Birden fazla kısıt zincirlenebilir; aralarına `:` konur:

```csharp
[Route("urun/{id:int:min(1)}")]          // sayı olacak VE 1'den küçük olmayacak
public IActionResult Detay(int id) => View();
```

### HTTP metodu öznitelikleri

```csharp
[HttpGet("{id}")]           // GET   /magaza/urun/5
[HttpPost]                  // POST  /magaza/urun
[HttpPut("{id}")]           // PUT   /magaza/urun/5
[HttpDelete("{id}")]        // DELETE /magaza/urun/5
```

Aynı adres, farklı HTTP metotlarıyla farklı işleri yapabilir. `/magaza/urun/5` adresine `GET` gitmesi "bana bu ürünü göster", `DELETE` gitmesi "bu ürünü sil" demektir. Adres aynıdır, **fiil** farklıdır.

Web API'lerde neredeyse her zaman attribute routing kullanılır; klasik MVC'de convention-based yeterlidir. İkisi aynı projede bir arada kullanılabilir — attribute routing yazılmış bir action, varsayılan şablonu yok sayar.

---

## 3. Model Binding Nedir

> **Benzetme —** Lokantada garson siparişini ağzından duyduğu gibi mutfağa götürmez. "Az pişmiş köfte, yanına pilav, ayran" cümlesini alır ve mutfağın anladığı fişe döker: ürün kodu, adet, pişirme notu. Mutfak senin cümleni okumaz, fişi okur. Garson yanlış kutucuğa işaretlerse mutfak yanlış yemeği çıkarır — ve suç mutfakta değildir.

**Basitçe:** Tarayıcıdan gelen her şey düz yazıdır. Adresteki `5` bile bir sayı değil, "5" harfidir. Model binding, bu düz yazıları senin metodunun beklediği tiplere çeviren ara katmandır. Sen `int id` yazarsın, o gider isteğin içinde `id` arar, bulur, sayıya çevirir, parametreye koyar. Sen bu işi hiç görmezsin.

**Teknik olarak: Model binding** — Gelen HTTP isteğindeki ham veriyi (URL parçaları, form alanları, JSON gövdesi) otomatik olarak C# değişkenlerine ve nesnelerine dönüştüren sistem.

Bu olmasaydı her action'da şöyle bir şey yazman gerekirdi:

```csharp
// Model binding olmasaydı
var idText = Request.Query["id"];
if (!int.TryParse(idText, out int id)) return BadRequest();
```

Model binding sayesinde:

```csharp
public IActionResult Detay(int id) { ... }
```

Çatı, gelen isteğe bakar, `id` adında bir değer bulur, `int`'e çevirir ve parametreye yerleştirir. Çevirememişse `ModelState`'e hata ekler.

**Eşleştirme kuralı:** Form alanının `name` değeri, hedef özelliğin adıyla **birebir aynı** olmalıdır. `name="AdSoyad"` → `t.AdSoyad`. Eşleşmeyen alan `null` veya `default` kalır.

Bu cümlenin sonundaki "`null` veya `default` kalır" kısmı, hata ayıklarken en çok zaman kaybettiren davranıştır. Model binding eşleşmeyen alan için **hata vermez**; sessizce boş bırakır. Ekranda "Ad boş geldi" diye bakarsın, hâlbuki formda değer vardı — sorun ismin bir harfindedir.

```cshtml
<!-- YANLIŞ: name, özellik adıyla aynı değil -->
<input name="adsoyad" />        <!-- model: AdSoyad -->

<!-- DOĞRU: tag helper ismi kendisi üretir, yazım hatası ihtimali kalmaz -->
<input asp-for="AdSoyad" />
```

`asp-for` kullanmanın asıl faydası budur: `name` değerini elle yazmazsın, dolayısıyla yanlış yazamazsın.

> **Benzetme nerede bozulur:** Garson yanlış anladığında sana tekrar sorabilir. Model binding soramaz. Elindeki veriyle ne yapabiliyorsa onu yapar, yapamadığını `ModelState`'e hata olarak yazar ve metodu yine de **çalıştırır**. Yani metodun içine girildiğinde binding'in başarılı olduğu garanti değildir; bu yüzden `ModelState.IsValid` kontrolü vardır.

---

## 4. Verinin Geldiği Üç Kaynak

> **Benzetme —** Bir kargo kolisi düşün. Üç ayrı yerde bilgi taşır: kutunun üstündeki **adres etiketi** (bu koli nereye, hangi kayda ait), etiketin yanına iliştirilmiş **küçük not** ("kapıda ödeme", "kırılacak eşya") ve **kutunun içi** (asıl mal). Kargocu adrese bakarak taşır, nota bakarak nasıl davranacağına karar verir, kutuyu ise yalnızca alıcı açar.

**Basitçe:** Veri sana üç ayrı yoldan gelebilir: adresin içinden, adresin sonundaki soru işaretinden sonra, ya da isteğin gövdesinden. Hangi yoldan geleceğini genelde ASP.NET Core kendisi doğru tahmin eder, ama sen açıkça yazarsan hem kod okunur hem de yanlış tahmin ihtimali biter.

**Teknik olarak:** ASP.NET Core veriyi üç yerden okuyabilir. Genelde kendisi doğru olanı seçer, ama açıkça belirtmek okunabilirliği artırır ve belirsizlikleri ortadan kaldırır.

### 4.1 `[FromRoute]` — URL'in içinden

```csharp
// ornek.com/Urun/Detay/5
public IActionResult Detay([FromRoute] int id)   // id = 5
```

Route şablonundaki `{id}` parçasından okunur. Kaynak belirtilmezse zaten buradan bakılır.

Kolinin **adres etiketi** budur: kaydın kimliğini taşır, adresin bir parçasıdır, paylaşılabilir ve yer imine eklenebilir.

### 4.2 `[FromQuery]` — sorgu parametresinden

```csharp
// ornek.com/Urun/Ara?kelime=telefon&sayfa=2
public IActionResult Ara([FromQuery] string kelime, [FromQuery] int sayfa = 1)
```

`?` işaretinden sonraki `anahtar=değer` çiftlerinden okunur. Arama, filtreleme ve sayfalama için standart yoldur.

Etiketin yanındaki **küçük not** budur: isteğe bağlıdır, olmayabilir, sıralaması önemli değildir. Yukarıdaki `int sayfa = 1` yazımı da bunu söyler — parametre gelmezse 1 kabul edilir.

> **Dikkat:** Query string tarayıcı geçmişine, sunucu loglarına ve referrer başlıklarına düşer. Kişisel veri, token veya şifre buraya konmaz.

### 4.3 `[FromBody]` — istek gövdesinden

```csharp
[HttpPost]
public IActionResult Kaydet([FromBody] Product yeniUrun)
```

İstek gövdesindeki JSON'dan okunur. Web API'lerde ve JavaScript `fetch` çağrılarında kullanılır.

**Kutunun içi** budur: adreste görünmez, logda görünmez, uzunluk sınırı pratikte çok daha geniştir. Büyük ve yapılandırılmış veri buradan gider.

**Klasik form POST'u** ise gövdeden okunur ama `[FromBody]` **kullanılmaz** — form verisi JSON değil, `application/x-www-form-urlencoded` biçimindedir. `[FromForm]` vardır, ancak MVC'de genelde hiçbir öznitelik yazmadan da çalışır:

```csharp
[HttpPost]
public IActionResult Kaydet(Product yeniUrun)   // form alanları otomatik eşlenir
```

> **Kural:** Bir action'da `[FromBody]` yalnızca **bir kez** kullanılabilir. Sebebi basit: istek gövdesi bir akıştır, bir kez okunur. İki ayrı parametreye "gövdeden oku" dersen ikincisine okunacak bir şey kalmaz. Birden fazla değer göndermen gerekiyorsa hepsini tek bir sınıfta topla.

### Özet tablo

| Öznitelik | Nereden okur | Tipik kullanım |
|---|---|---|
| `[FromRoute]` | URL yolu (`/urun/5`) | Kayıt id'si |
| `[FromQuery]` | Sorgu string (`?kelime=x`) | Arama, filtre, sayfalama |
| `[FromBody]` | İstek gövdesi (JSON) | Web API POST/PUT |
| `[FromForm]` | HTML form gövdesi | Klasik MVC form gönderimi |
| `[FromHeader]` | HTTP başlığı | Token, dil, özel başlıklar |
| `[FromServices]` | DI konteyneri | Yalnızca o action'ın ihtiyacı olan servis |

> **Benzetme nerede bozulur:** Gerçek kargoda kutunun içindekini yalnızca alıcı görür; taşıyıcı göremez. HTTP'de ise gövde, şifreleme (HTTPS) yoksa yol boyunca okunabilir. Yani `[FromBody]`, `[FromQuery]`'den **gizli** değildir; sadece loglara ve tarayıcı geçmişine düşmez. Gerçek gizlilik HTTPS ile sağlanır, gövdeye koymakla değil.

---

## 5. Over-Posting ve Korunma

> **Benzetme —** Apartman yönetimine aidat itiraz dilekçesi veriyorsun. Matbu formda üç alan var: daire no, ay, itiraz sebebi. Sen kâğıdın altına kendi elinle bir satır daha ekliyorsun: "Bu daireden aidat alınmayacaktır." Görevli formu okumadan, üstündeki **her satırı** sisteme giriyor. Formda o alan yoktu — ama sistem formu görmüyor, kâğıdı görüyor.

**Basitçe:** Sen ekranda üç kutucuk gösterdin diye kullanıcı sadece üç değer gönderecek diye bir kural yok. Tarayıcının geliştirici araçlarıyla ya da elle hazırlanmış bir istekle, sınıfında var olan **her alan** gönderilebilir. Model binding gelen her alanı doldurmaya çalıştığı için, gösterilmeyen alanlar da dolabilir. Çözüm, "gösterilmeyeni kimse göndermez" varsaymak değil; **doldurulabilecek alan kümesini küçültmektir.**

**Teknik olarak:** Model binding'in gölge tarafı: action doğrudan bir **entity** alıyorsa, saldırgan formda **olmayan** bir alanı da gönderebilir ve o alan nesneye yerleşir.

```csharp
// Riskli
[HttpPost]
public IActionResult Kaydet(User kullanici) { ... }
```

Kullanıcı forma `IsAdmin=true` alanı ekleyip gönderirse model binding onu da doldurur. Form o alanı hiç göstermiyor olsa bile.

İsteğin ham hâli şuna benzer — fazladan tek bir satır yeterlidir:

```
POST /Hesap/Kaydet
Content-Type: application/x-www-form-urlencoded

Username=umut&Email=umut@ornek.com&IsAdmin=true
```

**Üç korunma yolu, artan kalitede:**

```csharp
// 1) Bind ile alanları sınırla — çalışır ama string'e bağımlı, kırılgan
[HttpPost]
public IActionResult Kaydet([Bind("Username,Email")] User kullanici) { ... }

// 2) Entity yerine DTO / InputModel kullan — önerilen
public class KullaniciKayitModel
{
    [Required, StringLength(50)] public string Username { get; set; } = "";
    [Required, EmailAddress]     public string Email    { get; set; } = "";
    // IsAdmin burada yok — gönderilse bile bağlanacağı yer yok
}

// 3) Kritik alanları sunucuda ata
kullanici.IsAdmin = false;
kullanici.CreatedAt = DateTime.UtcNow;
```

İkinci yolun neden en iyisi olduğunu bir cümleyle söylemek gerekirse: `[Bind]` "bu alanları doldurma" der ve listeyi güncellemeyi unutabilirsin; DTO ise o alanı **hiç var etmez**. Var olmayan bir alan doldurulamaz. Güvenliği hatırlamaya değil, tipe bağlamış olursun.

DTO ile birlikte kullanılan tipik akış:

```csharp
[HttpPost]
public async Task<IActionResult> Kaydet(KullaniciKayitModel model)
{
    if (!ModelState.IsValid)
        return View(model);

    var kullanici = new User
    {
        Username  = model.Username,
        Email     = model.Email,
        IsAdmin   = false,              // kullanıcıdan gelmez
        CreatedAt = DateTime.UtcNow     // kullanıcıdan gelmez
    };

    _context.Users.Add(kullanici);
    await _context.SaveChangesAsync();
    return RedirectToAction(nameof(Index));
}
```

**İlke:** Kullanıcıdan gelmemesi gereken veri, kullanıcıdan alınmaz — sunucuda üretilir.

> **Benzetme nerede bozulur:** Apartman görevlisi dikkatli davranıp fazladan satırı fark edebilir. Model binding fark edemez, çünkü onun için "fazladan" diye bir kavram yoktur; yalnızca "eşleşen" ve "eşleşmeyen" alan vardır. Bu yüzden korunma dikkatle değil, yapıyla sağlanır.

---

## 6. `IActionResult` — Neden Somut Tip Değil

> **Benzetme —** Kuruma dilekçe verdin, sonunda eline bir zarf tutuşturuyorlar. Zarfın içinden ne çıkacağı belli değil: onaylanmış belgen, "böyle bir kayıt bulunamadı" yazısı, "önce kimlik ibraz edin" uyarısı ya da "bu işlem 3. katta yapılır" notu. Dördü de farklı kâğıttır ama dördü de **zarftır**. Sen zarfı verirken içinde ne olduğunu söylemek zorunda değilsin.

**Basitçe:** Bir metot her zaman aynı şeyi döndürmez. Kayıt varsa sayfayı, yoksa "bulunamadı"yı, yetkisizse uyarıyı döndürür. Bunların hepsi farklı tiptedir. `IActionResult` bu farklı tiplerin ortak üst başlığıdır: "buradan bir HTTP cevabı çıkacak, ne olduğu duruma göre değişir" demenin yoludur.

**Teknik olarak:** Bir action metodundan doğrudan `Product` dönebilecekken neden `IActionResult` dönüyoruz?

**Çünkü bir isteğin sonucu her zaman başarı değildir.** Aynı metot duruma göre farklı şeyler döndürmek zorunda kalabilir:

```csharp
public IActionResult Detay(int id)
{
    var urun = _repo.Get(id);

    if (urun == null)
        return NotFound();                    // HTTP 404

    if (!KullaniciYetkili())
        return Unauthorized();                // HTTP 401

    if (urun.Silinmis)
        return RedirectToAction("Index");     // HTTP 302

    return View(urun);                        // HTTP 200 + HTML
}
```

`View`, `NotFound`, `Unauthorized`, `RedirectToAction` — hepsi **farklı tiplerdir** ama hepsi `IActionResult` arayüzünden türer. Bu sayede tek bir metot, duruma göre farklı yanıtlar döndürebilir.

**`IActionResult` bir arayüzdür**; "bu metot bir HTTP yanıtı döndürecek, ne olduğu duruma göre değişir" demektir.

Dikkat: `return NotFound();` satırı metodu **bitirmez çünkü özel bir şey yapar** demek değildir; sıradan bir `return`'dür. Geriye bir `NotFoundResult` nesnesi verir; HTTP yanıtına dönüşmesi ise sen döndükten sonra çatının işidir. Yani `NotFound()` "404 gönder" değil, "404 gönderilecek nesneyi üret" demektir.

### Sık kullanılan sonuçlar

| Metot | HTTP kodu | Ne döner |
|---|---|---|
| `View(model)` | 200 | Razor ile üretilmiş HTML |
| `Ok(veri)` | 200 | JSON |
| `RedirectToAction("Index")` | 302 | Yönlendirme |
| `NotFound()` | 404 | Kayıt yok |
| `BadRequest("mesaj")` | 400 | Geçersiz istek |
| `Unauthorized()` | 401 | Kimlik doğrulanmamış |
| `Forbid()` | 403 | Kimlik var ama yetki yok |
| `NoContent()` | 204 | Başarılı, döndürecek veri yok |

> **401 ve 403 farkı:** 401 "kim olduğunu bilmiyorum, giriş yap" demektir; 403 "kim olduğunu biliyorum ama bunu yapamazsın" demektir.

### Asenkron hâli

```csharp
public async Task<IActionResult> Index()
{
    var liste = await _context.Products.ToListAsync();
    return View(liste);
}
```

`Task<IActionResult>` okunuşu: *"Bu metot asenkron çalışacak ve işi bittiğinde geriye bir `IActionResult` verecek."* Hafta 1'in `04-Asenkron-Programlama.md` notundaki `Task` kavramının buradaki karşılığı.

> **Benzetme nerede bozulur:** Kurumdaki zarf gerçekten kapalıdır, içeriği dışarıdan bilinmez. `IActionResult`'ta ise durum kodu zarfın **üstünde** yazılıdır — 404 mü 200 mü, istemci daha içine bakmadan bilir. Dolayısıyla belirsiz olan "ne döndüğü" değil, "hangi C# tipinin döndüğü"dür. Bu ayrım, bir sonraki başlığın tam konusudur.

---

## 7. `ActionResult<T>` ve Web API Farkı

> **Benzetme —** İki koli yan yana duruyor. Birinin etiketinde "içinde bir şey var" yazıyor, diğerinde "içinde 1 adet kitap var". İkisi de taşınır, ikisi de teslim edilir. Ama ikinci koliyi alan kişi açmadan raf ayırabilir, kaç kişi geleceğini planlayabilir, yanlış rafa koyarsa uyarı alır. Etiketin ayrıntılı olması taşımayı değiştirmez; **etrafındaki herkesin işini** değiştirir.

**Basitçe:** `IActionResult` "bir cevap döneceğim" der. `ActionResult<T>` ise "ya bir hata sonucu döneceğim ya da tam olarak şu tipte bir veri" der. İkincisinde metodun imzasına bakan herkes — derleyici, Swagger, API'yi kullanan arkadaşın — ne bekleyeceğini bilir.

**Teknik olarak:** Klasik MVC'de `IActionResult` doğru seçimdir. Ama **Web API** yazarken daha iyi bir seçenek var.

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<Product>> GetProduct(int id)
{
    var urun = await _repo.GetByIdAsync(id);

    if (urun == null)
        return NotFound();        // HTTP 404 dönebilir

    return urun;                  // nesnenin kendisini döndür — arka planda 200 + JSON olur
}
```

**`ActionResult<T>`** iki şeyi birden mümkün kılar: hata durumlarında `NotFound()` gibi sonuçlar döndürmek **ve** başarılı durumda nesnenin kendisini doğrudan döndürmek.

Bunu mümkün kılan şey, `ActionResult<T>` üzerindeki **örtük dönüşüm** (implicit conversion) operatörleridir. `return urun;` yazdığında derleyici bunu sessizce `new ActionResult<Product>(urun)` hâline getirir; `return NotFound();` yazdığında ise `ActionResult` tarafını kullanır. Tek dönüş tipi, iki farklı şekil.

**Neden daha iyi:** Metodun imzası, geriye ne döneceğini **açıkça** söyler. Bunun üç somut faydası var:

1. **Swagger** (API dokümantasyon aracı) yanıtın şemasını otomatik çıkarabilir
2. API'yi kullanan diğer geliştiriciler dokümana bakmadan tipi görür
3. Derleyici yanlış tip döndürmeni engeller

`IActionResult` döndüren bir API metodunda bu bilgilerin hiçbiri yoktur — Swagger "bir şey dönüyor" der, o kadar.

Yine de `IActionResult` kullanmak zorunda kaldığın durumlarda bu bilgiyi elle verebilirsin:

```csharp
[HttpGet("{id}")]
[ProducesResponseType(typeof(Product), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetProduct(int id) { ... }
```

Bu, `ActionResult<T>`'nin bedava verdiği bilgiyi elle yazmaktır. Çalışır, ama iki yerde tutulan bilgi zamanla birbirinden ayrı düşer — imza değişir, öznitelik olduğu gibi kalır.

### Hangisi ne zaman

| Proje tipi | Tercih | Sebep |
|---|---|---|
| **Klasik MVC** (View döndüren) | `IActionResult` | HTML, redirect, hata sayfası — hepsi farklı tip, esneklik gerekir |
| **Web API** (JSON döndüren) | `ActionResult<T>` | Tip güvenliği + Swagger + okunabilirlik |
| İşlem sonucu döndürmeyen | `Task` veya `IActionResult` | — |

---

## 8. Async Her Zaman Gerekli mi

> **Benzetme —** Fırına ekmek söyledin, on dakika sürecek. Tezgâhın önünde dikilip beklemek yerine yan dükkândan alışverişini yapar, sonra dönüp ekmeği alırsın. Ama cebindeki parayı saymak için kimse dışarı çıkmaz — o iş zaten senin elinde, beklemek diye bir şey yok. Beklenecek bir şey yokken "verimli bekleme" düzeneği kurmak, sadece fazladan iştir.

**Basitçe:** `async` beklemeyi hızlandırmaz. Beklerken sunucunun o iş parçacığını başka isteklere ayırmasını sağlar. Dışarıdan bir şey beklemiyorsan bekleyecek bir şey de yoktur; `async` yazmak sadece gereksiz bir makine kurar.

**Teknik olarak:** Kısa cevap: **hayır.** Karar verme kuralı tek bir soruya bakar:

> *"Bu metot, sunucunun dışındaki bir kaynağı bekleyecek mi?"*

**Evet ise (I/O-bound) → async kullan.** Veritabanı sorgusu, dosya okuma, dış API çağrısı, SMS gönderimi:

```csharp
public async Task<IActionResult> SmsGonder(string telefon)
{
    await _smsService.SendAsync(telefon);    // dış kaynak bekleniyor
    return Ok();
}
```

**Hayır ise (CPU-bound) → async kullanma.** Sadece RAM üzerinde hesaplama yapıyorsa:

```csharp
public IActionResult KdvHesapla(decimal fiyat)
{
    decimal kdvli = fiyat * 1.20m;    // veritabanı yok, ağ yok
    return View(kdvli);
}
```

Senkron bir metodu zorla `async` yapmak, arka planda gereksiz bir durum makinesi (state machine) üretir ve performansı **düşürür**. Ayrıntı: `Hafta-01/04-Asenkron-Programlama.md`.

> **Benzetme nerede bozulur:** Fırın örneğinde "sen" tek kişisin ve gerçekten başka iş yaparsın. Sunucuda ise beklerken işi devralan **senin thread'in değildir**; thread havuza geri döner ve başka bir isteğe verilir. Yani kazanç "bir kişinin daha çok iş yapması" değil, "aynı sayıda kişiyle daha çok müşteriye bakılması"dır. Tek bir isteğin süresi async ile kısalmaz — hatta mikro ölçekte biraz uzar.

---

## Tek Bakışta Özet

- **Convention-based routing** tek şablonla tüm uygulamayı yönetir; **attribute routing** URL'i action'ın üstünde tanımlar ve SEO dostu adresler için kullanılır.
- `{id:int}` gibi **route kısıtları** metoda hiç girmeden yanlış tipi eler — doğrulama değil, eşleşme şartıdır.
- **Model binding** ham HTTP verisini C# nesnesine çevirir; `name` değeri özellik adıyla birebir eşleşmelidir. Eşleşmeyen alan hata vermez, sessizce boş kalır.
- Veri üç yerden gelir: **`[FromRoute]`** (URL), **`[FromQuery]`** (sorgu), **`[FromBody]`** (JSON gövde). Form POST'u için öznitelik gerekmez. `[FromBody]` bir action'da yalnızca bir kez kullanılır.
- **Over-posting**: entity'yi doğrudan parametre yapmak, formda olmayan alanların gönderilmesine izin verir. Çözüm DTO — var olmayan alan doldurulamaz.
- **`IActionResult`** bir arayüzdür; aynı metodun duruma göre HTML, 404, redirect döndürebilmesini sağlar.
- **`ActionResult<T>`** Web API'lerde tercih edilir: hem hata sonuçları hem tip güvenliği hem Swagger desteği. Örtük dönüşüm sayesinde `return urun;` yazabilirsin.
- **Async kararı** tek soruya bağlıdır: dışarıdan bir şey bekliyor muyum? Beklemiyorsa senkron yaz.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Routing | URL'in hangi controller/action'a gideceğini belirleme |
| Convention-based routing | Tek şablonla tüm uygulamayı yönlendirme |
| Attribute routing | Yolu action/controller üstünde öznitelikle tanımlama |
| Route constraint | Route parçasına tip veya biçim kısıtı (`{id:int}`) |
| Model binding | HTTP verisini C# parametre ve nesnelerine dönüştürme |
| `[FromRoute]` / `[FromQuery]` / `[FromBody]` | URL yolundan / sorgudan / gövdeden okuma |
| `[FromForm]` / `[FromHeader]` / `[FromServices]` | Form gövdesinden / başlıktan / DI'dan alma |
| `asp-for` | Tag helper; `name` ve `id` değerlerini özellik adından otomatik üretir |
| Over-posting | Formda olmayan alanların gönderilerek veriyi manipüle etmesi |
| `[Bind]` | Model binding'in dolduracağı alanları sınırlama |
| DTO / InputModel | Yalnızca taşınacak alanları içeren, entity'den ayrı sınıf |
| `IActionResult` | Farklı HTTP yanıt tiplerini tek çatıda toplayan arayüz |
| `ActionResult<T>` | Hem hata sonucu hem tipli veri döndürebilen yapı |
| Örtük dönüşüm (implicit conversion) | Derleyicinin bir tipi başka bir tipe sessizce çevirmesi |
| `[ProducesResponseType]` | Yanıt tipini ve durum kodunu belgeye elle bildiren öznitelik |
| `Task<IActionResult>` | Asenkron çalışıp bir HTTP yanıtı döndürecek metot |
| Swagger / OpenAPI | API'yi otomatik belgeleyen ve test ettiren araç |
| I/O-bound / CPU-bound | Dış kaynağı bekleyen / hesaplama yapan iş |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `04-EF-Core-CRUD-ve-Change-Tracking.md`
