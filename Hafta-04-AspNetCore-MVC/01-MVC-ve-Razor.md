# Hafta 4 · MVC Mimarisi ve Razor Görünüm Motoru

**Okuma süresi:** ~40 dk
**Neden bu konu:** Bootcamp'in 20 projesinin en az 15'i bu üç harfin üzerine kuruluyor. "Controller ne yapar" sorusuna verdiğin cevabın netliği, ileride katmanlı mimariyi anlamanı doğrudan belirliyor.

> **Kaynak notu:** Bu hafta 4 notları, harici eğitim materyalinden (17 Eylül 2026) derlendi. Hafta 1'de işlenen konular (value/reference tip, LINQ, async) burada tekrarlanmıyor, ilgili nota yönlendiriliyor.

---

## Önce Basitçe

Bir lokantaya gittiğini düşün. İçeri girdiğinde üç ayrı iş yapan üç ayrı grup insan var. Garson seni karşılar, ne istediğini sorar, siparişi mutfağa iletir ve tabağı sana getirir. Mutfak yemeği pişirir; salonu, masa düzenini, senin kim olduğunu bilmez, sadece "iki porsiyon mercimek" duyar ve onu yapar. Bir de masaya konan tabak, sunum, peçete var: yemeği göze hoş gösteren kısım. Tabak yemeği pişirmez, sadece gösterir.

MVC dediğimiz şey tam olarak bu iş bölümü. Mutfak **Model**, tabak **View**, garson **Controller**. Uygulamaya bir istek geldiğinde garson (Controller) onu karşılar, işi kimin yapacağına karar verir, mutfaktan (Model) sonucu alır ve uygun tabakta (View) sana sunar. Garson kendisi yemek pişirmez. Pişirmeye kalkarsa lokanta bir süre sonra kilitlenir.

Bu ayrımın tek amacı düzen değil. Asıl amaç, yarın bir şey değiştiğinde yalnızca **tek bir yeri** değiştirmen. Sunum değişecekse tabağa dokunursun, mutfağa girmezsin. Yemek tarifi değişecekse mutfağa girersin, tabaklarla uğraşmazsın.

Peki HTML tarafında veriyi nasıl gösteriyoruz? Burada **Razor** devreye girer. Razor, HTML'in içine "buraya şu veri gelecek" diye not düşmeni sağlayan bir şablon sistemidir. Tıpkı bir davetiye şablonu gibi: metnin çoğu sabittir, sadece isim kısmı boş bırakılır ve her davetli için doldurulur. Razor'daki `@` işareti "burası boşluk, buraya veri gelecek" demenin yoludur.

Notun geri kalanı bu iki fikrin ayrıntısı: üç parçanın sınırları nerede biter, Razor'a veriyi hangi yollarla taşırsın, hangi yol neden daha güvenlidir. Şimdi detaya iniyoruz.

> **Ana benzetme:** MVC bir lokantanın iş bölümüdür — mutfak (Model) pişirir, tabak (View) sunar, garson (Controller) ikisini birbirine bağlar. Razor ise tabağın üstündeki "isim kartı": şablon sabittir, boşluğu her müşteri için veri doldurur.

---

## Bu Notta Ne Var

1. MVC'nin üç parçası ve sorumluluk sınırları
2. Razor motoru ve `@` sözdizimi
3. Controller'dan View'a veri taşımanın üç yolu
4. Entity ve ViewModel ayrımı
5. Tag Helper ve Html Helper
6. Partial View
7. Katmanların birbirini nasıl görmemesi gerektiği

---

## 1. MVC'nin Üç Parçası

> **Benzetme —** Lokantada mutfak, tabak ve garson. Mutfak yemeği yapar ama masanın nerede olduğunu bilmez. Tabak yemeği gösterir ama pişirmeyi bilmez. Garson ikisini birbirine bağlar; kendisi ne pişirir ne de sunumu tasarlar, sadece doğru işi doğru yere iletir.

**Basitçe:** Uygulamayı üç işe bölersin. Biri veriyi ve kuralları tutar, biri ekranı çizer, biri de gelen isteği karşılayıp doğru kişiye havale eder. Her parça kendi işini bilir, diğerinin işine karışmaz.

**Teknik olarak:** **MVC (Model–View–Controller)** — Uygulamayı üç sorumluluğa bölen mimari desen. Amaç kodun yönetilebilir, test edilebilir ve sürdürülebilir olması.

| Parça | Sorumluluğu | Bilmemesi gereken |
|---|---|---|
| **Model** | Veri ve **iş mantığı**. Veritabanı işlemleri, kurallar, hesaplamalar | HTTP, HTML, hangi ekranda gösterileceği |
| **View** | Kullanıcı arayüzü. HTML, CSS, JS + Razor | Veritabanı, iş kuralları |
| **Controller** | Gelen HTTP isteğini karşılar, ilgili Model'i çağırır, uygun View'ı döndürür | Veritabanı ayrıntıları (ideal durumda) |

Controller'ı **trafik polisi** gibi düşün: kendisi iş yapmaz, işi kimin yapacağına karar verir ve sonucu kime göstereceğini seçer. Controller'ın şişmesi (bir action'ın 100 satır olması) neredeyse her zaman "iş mantığı yanlış katmanda" demektir.

**View'ın görevi sadece veriyi göstermektir; iş mantığı içermemelidir.** Bu cümle basit görünür ama uygulamada en çok ihlal edilen kuraldır. View içinde fiyat hesaplayan bir `@(urun.Fiyat * 1.20m)` ifadesi, KDV oranı değiştiğinde bütün view'ları taramanı gerektirir.

Aynı hesabı doğru yere koymak şöyle görünür:

```csharp
// Model / ViewModel tarafı — kural burada yaşar
public class UrunGoruntuleme
{
    public string Ad { get; set; }
    public decimal FiyatKdvHaric { get; set; }
    public decimal KdvOrani { get; set; } = 0.20m;

    // Kural tek yerde. Oran değişirse tek satır değişir.
    public decimal FiyatKdvDahil => FiyatKdvHaric * (1 + KdvOrani);
}
```

```cshtml
@* View sadece hazır sonucu basar, hesap yapmaz *@
<p>KDV dahil: @Model.FiyatKdvDahil.ToString("N2") ₺</p>
```

> **Bu benzetme şurada bozulur:** Lokantada garson mutfağa giremez, fiziksel bir duvar vardır. Kodda böyle bir duvar yoktur. Controller'ın içine pekâlâ SQL sorgusu yazabilirsin, derleyici seni durdurmaz ve program çalışır. MVC'nin sınırları **dil tarafından zorlanmaz**, senin disiplinin tarafından korunur. Bu yüzden "çalışıyor" ile "doğru yerde" aynı şey değildir.

---

## 2. Razor Motoru

> **Benzetme —** Matbaadan aldığın düğün davetiyesini düşün. Metnin tamamı baskılıdır: tarih, salon adı, süslemeler. Sadece "Sayın ..........." kısmı boş bırakılmıştır ve her davetli için elle doldurulur. Razor bu davetiyenin dijital hâlidir: HTML'in çoğu sabittir, `@` ile işaretlediğin yerler her istekte veriyle doldurulur.

**Basitçe:** Razor, HTML dosyasının içine "burada C# çalışsın" diyebilmeni sağlar. Bunu tek bir işaretle yaparsın: `@`. Gerisi normal HTML'dir.

**Teknik olarak:** **Razor** — HTML'in içine doğrudan C# yazmanı sağlayan şablonlama (templating) motoru. Dosya uzantısı `.cshtml` — açılımı **C# + HTML**.

Razor'ın tek temel kuralı: **`@` işareti.** Derleyici HTML içinde `@` gördüğünde sonrasını C# kodu olarak yorumlar.

```cshtml
<h2>@Model.Baslik</h2>                    @* tek ifade *@
<p>Toplam: @(adet * fiyat) TL</p>         @* parantezli ifade *@

@if (Model.Fiyat > 1000)                  @* kontrol yapısı *@
{
    <p style="color:red">Kargo bedava!</p>
}

@foreach (var u in Model)                 @* döngü *@
{
    <li>@u.Ad</li>
}

@* Bu bir Razor yorumudur, tarayıcıya gitmez *@
<!-- Bu bir HTML yorumudur, kaynak kodda görünür -->
```

Tek ifade ile parantezli ifade arasındaki fark, Razor'ın ifadenin nerede bittiğini nasıl anladığıdır. `@Model.Ad` yazdığında Razor nokta ve harf zincirini takip eder, boşlukta durur. Ama `@adet * fiyat` yazarsan Razor yalnızca `adet`i C# sayar, geri kalanını düz metin basar. Hesap yapıyorsan parantez şart:

```cshtml
@{ var adet = 3; var fiyat = 250m; }

<p>Yanlış: @adet * fiyat</p>       @* ekrana "3 * fiyat" yazar *@
<p>Doğru:  @(adet * fiyat)</p>     @* ekrana "750" yazar *@
```

Birden fazla satır C# çalıştırman gerekiyorsa **kod bloğu** kullanırsın:

```cshtml
@{
    var indirimli = Model.Fiyat * 0.9m;
    var etiket = indirimli < 500 ? "Uygun" : "Standart";
}

<p>@etiket — @indirimli.ToString("N2") ₺</p>
```

Bir de metinle kodun birbirine karıştığı durum var. Razor `@` sonrası e-posta adresi gibi bir şey görürse çoğu zaman doğru tahmin eder, ama tahmin ettirmemek en iyisidir. Ekrana düz `@` basmak istersen çift yazarsın:

```cshtml
<p>İletişim: destek@@sirket.com</p>   @* ekranda: destek@sirket.com *@
```

**Önemli güvenlik davranışı:** Razor `@` ile bastığı her değeri **HTML-encode eder**. Veritabanındaki metinde `<script>alert(1)</script>` olsa bile çalışmaz, düz metin olarak görünür. Bu, XSS saldırılarına karşı varsayılan korumadır. Bilinçli olarak ham HTML basmak istersen `@Html.Raw(...)` kullanılır — ve o an koruma kapanır, bu yüzden yalnızca kaynağına güvendiğin içerikte kullanılır.

```cshtml
@{ var kullaniciYorumu = "<script>alert('ele geçirildi')</script>"; }

<p>@kullaniciYorumu</p>
@* Ekrana metin olarak yazılır, script ÇALIŞMAZ. Razor şuna çevirir:
   &lt;script&gt;alert('ele geçirildi')&lt;/script&gt; *@

<p>@Html.Raw(kullaniciYorumu)</p>
@* Koruma kapalı. Bu script GERÇEKTEN çalışır.
   Kullanıcıdan gelen veride asla böyle yapma. *@
```

> **Bu benzetme şurada bozulur:** Davetiyeyi bir kez doldurup zarfa koyarsın, iş biter. Razor şablonu ise her istekte yeniden doldurulur ve sonuç **sunucuda** üretilip tarayıcıya gönderilir. Yani `@` içindeki C# kodu kullanıcının bilgisayarında değil, senin sunucunda çalışır. Tarayıcıya giden şey sadece sonuçtur — kullanıcı `@if` bloğunu kaynak kodda göremez. Bu ayrımı kaçırırsan "neden JavaScript değişkenim C# tarafında görünmüyor" sorusuna takılırsın.

---

## 3. Controller'dan View'a Veri Taşıma

> **Benzetme —** Mutfaktan salona yemek taşımanın üç yolu var. Birincisi, siparişe göre hazırlanmış kapaklı tabak: içinde ne olduğu yazılıdır, yanlış masaya gitmesi zordur. İkincisi, garsonun eline tutuşturulan üstü açık bir kase: hızlıdır ama ne olduğu belli değildir, yolda karışabilir. Üçüncüsü de aynı kasenin üstüne elle yazılmış bir etiket yapıştırılmış hâli. Kapaklı tabak asıl yemek içindir; kaseler tuzluk biberlik taşımak için.

**Basitçe:** Controller'ın bulduğu veriyi ekrana ulaştırmanın üç yolu var. Biri tip güvenli ve asıl veri için; diğer ikisi pratik ama kontrolsüz, küçük yan bilgiler için. Hangisini seçtiğin, kodun yarın nasıl bozulacağını belirler.

**Teknik olarak:** Üç yol var. Hangisini seçtiğin bir kalite göstergesidir.

### 3.1 Strongly-Typed Model (önerilen)

View'a **tek bir tip** gönderilir ve view'ın en üstünde `@model` ile tanımlanır.

```csharp
// Controller
public IActionResult Detay()
{
    var urun = new Product { Id = 1, Name = "Mekanik Klavye", Price = 1500.50m };
    return View(urun);
}
```

```cshtml
@* View — Detay.cshtml *@
@model Product

<h2>@Model.Name</h2>
<p>@Model.Price ₺</p>
```

Dikkat: **`@model`** (küçük m) tipi tanımlar, **`@Model`** (büyük M) o tipteki nesneye erişir.

**Neden en güvenlisi:** Derleyici kontrolü. `@Model.Nmae` yazarsan proje derlenmez. IntelliSense çalışır, yeniden adlandırma (rename) güvenlidir.

Liste göndermek de aynı mantıkla çalışır; sadece tip değişir:

```csharp
public IActionResult Index()
{
    var urunler = _context.Products.ToList();
    return View(urunler);
}
```

```cshtml
@model List<Product>

<table class="table">
    <thead><tr><th>Ad</th><th>Fiyat</th></tr></thead>
    <tbody>
    @foreach (var u in Model)
    {
        <tr>
            <td>@u.Name</td>
            <td>@u.Price.ToString("N2") ₺</td>
        </tr>
    }
    </tbody>
</table>
```

### 3.2 ViewBag

`dynamic` bir nesnedir; istediğin adı takarsın.

```csharp
ViewBag.SayfaBasligi = "Ürün Detay Sayfası";
```

```cshtml
<title>@ViewBag.SayfaBasligi</title>
```

**Bedeli:** Derleme anında kontrol edilmez. `ViewBag.SayfaBasligi` yerine `ViewBag.SayfaBaslıgı` yazarsan hata almazsın — sessizce `null` gelir ve ekranda boşluk görürsün. Teşhisi sinir bozucudur.

Türkçe karakterli isimlerde bu tuzak özellikle sinsidir: `ı` ile `i`, `ğ` ile `g` gözle ayırt edilmez ama derleyici için iki ayrı anahtardır.

**Ne zaman uygun:** Modelin parçası olmayan, küçük yan bilgiler — sayfa başlığı, bir uyarı mesajı, açılır listeyi dolduracak seçenekler.

### 3.3 ViewData

Aynı işi sözlük (dictionary) sözdizimiyle yapar. `ViewBag`, aslında `ViewData`'nın `dynamic` sarmalayıcısıdır — ikisi **aynı veriyi** taşır.

```csharp
ViewData["Mesaj"] = "Merhaba";
```

```cshtml
<p>@ViewData["Mesaj"]</p>
```

`ViewData` tip dönüşümü ister (`(int)ViewData["Adet"]`), `ViewBag` istemez. Anahtar adı string olduğu için yazım hatası riski aynıdır.

İkisinin aynı depoya yazdığını şu örnek gösterir:

```csharp
ViewData["Baslik"] = "Ürünler";
var ayni = ViewBag.Baslik;       // "Ürünler" — aynı veriyi okur

ViewBag.Adet = 5;
var yine = ViewData["Adet"];     // 5 (object olarak) — yine aynı depo
```

### Karşılaştırma

| | Strongly-typed Model | ViewBag | ViewData |
|---|---|---|---|
| Tip güvenliği | **Var** | Yok | Yok |
| IntelliSense | **Var** | Yok | Yok |
| Yazım hatası | Derlemede yakalanır | Çalışma anında `null` | Çalışma anında `null` |
| Ömür | O istek | O istek | O istek |
| Kullanım yeri | Sayfanın asıl verisi | Küçük yan bilgiler | Küçük yan bilgiler |

> **Kural:** Sayfanın asıl verisi her zaman model ile gider. `ViewBag`, modele sığmayan tek tük bilgiler içindir. Bir sayfada üç dört `ViewBag` görüyorsan, muhtemelen bir ViewModel yazman gerekiyordur.

> **Bu benzetme şurada bozulur:** Lokantada kapaklı tabak da kase de mutfaktan salona **fiziksel olarak** taşınır. Burada taşınan bir şey yok: Controller ile View aynı istek içinde, aynı bellekte çalışır. `ViewBag` bir kutu değil, aynı istek için ayrılmış bir sözlüğe yazmaktır. Bu yüzden ömrü tek istektir — yanıt gidince o sözlük silinir. Bir sonraki istekte oradan bir şey okumaya çalışırsan `null` alırsın. İstekler arası taşıma gerekiyorsa `TempData` ya da session konuşulur, `ViewBag` değil.

---

## 4. Entity ve ViewModel Ayrımı

> **Benzetme —** Nüfus müdürlüğündeki dosyanla, bir siteye üye olurken doldurduğun formu düşün. Devletin dosyasında her şey var: kimlik numaran, anne kızlık soyadın, adres geçmişin. Siteye üye olurken bunların hiçbirini vermezsin; sadece ad ve e-posta yazarsın. İkisi de "seni" temsil eder ama biri arşiv kaydıdır, diğeri o işe özel bir özettir. Entity arşiv dosyası, ViewModel o ekrana özel formdur.

**Basitçe:** Veritabanındaki sınıfı doğrudan ekrana göndermek, arşiv dosyanın tamamını masaya bırakmak gibidir. Ekranın ihtiyacı olan alanları içeren ayrı bir küçük sınıf yazarsın; fazlası ne görünür ne de yanlışlıkla sızar.

**Teknik olarak:** **Entity** — Veritabanı tablosunu temsil eden sınıf. EF Core'un tanıdığı, tabloya birebir karşılık gelen nesne.
**ViewModel (veya DTO — Data Transfer Object)** — Yalnızca belirli bir ekranın ihtiyacı olan veriyi taşıyan sınıf.

Küçük projelerde entity'yi doğrudan view'a göndermek işe yarar. Kurumsal projelerde yapılmaz. İki sebebi var:

**Güvenlik.** Entity'de kullanıcının görmemesi gereken alanlar olabilir:

```csharp
// Veritabanı varlığı (Entity)
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; }   // View'a asla gitmemeli
    public DateTime CreatedAt { get; set; }
}

// Sadece ekran için oluşturulmuş model
public class UserProfileViewModel
{
    public string Username { get; set; }
    // PasswordHash burada yok — güvende
}
```

Entity'yi doğrudan gönderdiğinde view onu basmasa bile veri **belleğe gelmiş** olur; ayrıca aynı entity bir API'den JSON olarak dönerse alan olduğu gibi dışarı sızar.

Dönüştürme işi çoğu zaman tek bir `Select` ile yapılır:

```csharp
public IActionResult Profil(int id)
{
    var model = _context.Users
        .Where(u => u.Id == id)
        .Select(u => new UserProfileViewModel
        {
            Username = u.Username
            // Sadece ihtiyaç duyulan alan seçilir.
            // Bonus: EF Core bunu SQL'e çevirirken de
            // yalnızca bu sütunu SELECT eder.
        })
        .FirstOrDefault();

    if (model is null) return NotFound();
    return View(model);
}
```

**Esneklik.** Bir ekran çoğu zaman tek tabloya karşılık gelmez. "Sipariş detay" sayfası müşteri bilgisi, sipariş satırları ve kargo durumunu birlikte ister. Bunları tek bir ViewModel'de birleştirirsin:

```csharp
public class SiparisDetayViewModel
{
    public string MusteriAdi { get; set; }
    public List<SiparisSatirViewModel> Satirlar { get; set; }
    public string KargoDurumu { get; set; }
    public decimal ToplamTutar { get; set; }
}
```

> Bu ayrım aynı zamanda **over-posting** saldırısına karşı da korur: form yalnızca ViewModel'deki alanları doldurabilir, entity'nin tamamını değil. Ayrıntı: `03-Routing-ve-Model-Binding.md`.

> **Bu benzetme şurada bozulur:** Nüfus dosyası ile üyelik formu iki ayrı kâğıttır; birini değiştirmek diğerini etkilemez. ViewModel'de ise dönüşümü **sen** yazarsın ve iki sınıf sessizce birbirinden uzaklaşabilir. Entity'ye yeni bir alan eklediğinde ViewModel bunu kendiliğinden öğrenmez; ekran eski kalır ve "veriyi kaydettim ama görünmüyor" dersin. Ayrımın bedeli budur: iki sınıfı senkron tutmak senin işin.

---

## 5. Tag Helper ve Html Helper

> **Benzetme —** Bir mağazada fiyat etiketi yazmanın iki yolu. Eski usul: kasadaki görevliye "şu ürüne şu fiyatı yaz" dersin, o gider yazar. Yeni usul: etiketin kendisinin üstünde hazır boşluklar vardır, ürün kodunu yazarsın, fiyat kendiliğinden dolar. İkisi de aynı etiketi üretir; ikincisi rafta duran etikete benzediği için bakan herkes ne olduğunu anlar.

**Basitçe:** Aynı HTML'i üretmenin iki yazım biçimi var. Biri C# metodu çağırır, biri normal HTML etiketi gibi görünür. Bugün ikincisini kullanıyoruz, çünkü HTML'e benzeyen şey HTML editöründe de, insan gözünde de daha okunaklı.

**Teknik olarak:** Aynı işi yapan iki kuşak sözdizimi var.

**Html Helper** — Eski ASP.NET MVC'den gelen, metot çağrısıyla HTML üreten yapılar.

```cshtml
@Html.ActionLink("Detaya Git", "Detail", "Product", new { id = Model.Id }, new { @class = "btn btn-primary" })
```

**Tag Helper** — ASP.NET Core ile gelen, HTML etiketine benzeyen ama arka planda C# çalıştıran yapılar.

```cshtml
<a asp-controller="Product" asp-action="Detail" asp-route-id="@Model.Id" class="btn btn-primary">
    Detaya Git
</a>
```

| | Html Helper | Tag Helper |
|---|---|---|
| Görünüm | C# metot çağrısı | Normal HTML etiketi |
| Tasarımcı dostu | Hayır | **Evet** — HTML editörleri tanır |
| IntelliSense | Kısmen | **Tam** |
| Bugünkü tercih | Eski kodda karşına çıkar | **Yeni kodda bu kullanılır** |

Sık kullanılan tag helper'lar:

```cshtml
<form asp-controller="Urun" asp-action="Kaydet" method="post">
    <label asp-for="Ad"></label>
    <input asp-for="Ad" class="form-control" />
    <span asp-validation-for="Ad" class="text-danger"></span>
    <button type="submit">Kaydet</button>
</form>
```

`asp-for="Ad"` tek başına üç iş yapar: `name="Ad"`, `id="Ad"` ve mevcut değeri `value` olarak basar. Ayrıca modeldeki doğrulama özniteliklerini (`[Required]` gibi) HTML `data-val-*` özniteliklerine çevirir.

Yukarıdaki `<input asp-for="Ad" />` satırının tarayıcıya giden hâli kabaca şudur:

```html
<input class="form-control" type="text" id="Ad" name="Ad" value="Mekanik Klavye"
       data-val="true" data-val-required="Ad alanı zorunludur." />
```

Bir açılır liste de aynı mantıkla bağlanır:

```cshtml
<select asp-for="KategoriId"
        asp-items="@(new SelectList(ViewBag.Kategoriler, "Id", "Ad"))">
    <option value="">-- Kategori seç --</option>
</select>
```

**Tag helper'ların etkinleşmesi** `_ViewImports.cshtml` dosyasındaki şu satıra bağlıdır:

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

Bu satır yoksa `asp-*` öznitelikleri işlenmez, tarayıcıya olduğu gibi gider ve hiçbir şey çalışmaz. Form Tag Helper'ın antiforgery token'ı otomatik eklemesini sağlayan da budur.

Bu hatanın belirtisi tipiktir: sayfa açılır, form görünür, ama gönder dediğinde hiçbir yere gitmez veya `action` boştur. Tarayıcıda "kaynağı görüntüle" dediğinde `asp-action="Kaydet"` yazısını olduğu gibi görüyorsan, eksik olan bu satırdır.

> **Bu benzetme şurada bozulur:** Mağaza etiketi rafta dururken de bir etikettir. Tag helper ise tarayıcıya **hiç ulaşmaz**; sunucuda işlenir, yerine düz HTML konur ve kullanıcı `asp-for` diye bir şey görmez. Yani HTML'e benziyor olması bir kolaylıktır, gerçeği değildir. Bu yüzden bir tag helper'ı JavaScript ile çalışma anında değiştiremezsin — o iş bitmiş, sayfa çoktan üretilmiştir.

---

## 6. Partial View

> **Benzetme —** Bir tarif defterinde "beşamel sos" tarifini her yemeğin altına baştan yazmazsın. Bir kez yazar, sonra "beşamel için sayfa 12'ye bak" dersin. Sos tarifi değişirse tek bir sayfayı düzeltirsin, defterin tamamını taramazsın. Partial view, tarif defterindeki o ortak sayfadır.

**Basitçe:** Sayfanın birden çok yerde tekrarlanan HTML parçasını ayrı bir dosyaya alırsın ve gerektiği yerde çağırırsın. Değişiklik gerektiğinde tek dosyaya dokunursun.

**Teknik olarak:** **Partial View (parçalı görünüm)** — Sayfanın tekrar eden bir parçasını ayrı bir `.cshtml` dosyasına çıkarmak.

```cshtml
<div>
    <h3>Stok Durumu</h3>
    <partial name="_StockSummaryPartial" model="Model.StockDetails" />
</div>
```

Ne zaman işe yarar: menüler, ürün kartı, yorum bloğu, tablo satırı — aynı HTML'in birden çok yerde geçtiği her durum.

Partial'ın kendisi de tipli olabilir; olması da tercih edilir:

```cshtml
@* Views/Shared/_UrunKarti.cshtml *@
@model UrunKartiViewModel

<div class="card">
    <h5 class="card-title">@Model.Ad</h5>
    <p class="card-text">@Model.Fiyat.ToString("N2") ₺</p>
    <a asp-action="Detail" asp-route-id="@Model.Id" class="btn btn-sm">İncele</a>
</div>
```

```cshtml
@* Listeleme sayfası — aynı kart, her ürün için *@
@model List<UrunKartiViewModel>

<div class="row">
@foreach (var urun in Model)
{
    <partial name="_UrunKarti" model="urun" />
}
</div>
```

**Adlandırma geleneği:** Partial view dosyaları alt çizgiyle başlar (`_UrunKarti.cshtml`). Bu, "bu dosya doğrudan bir URL'e yanıt olarak döndürülmez" demektir.

### Partial View mı, ViewComponent mı?

| | Partial View | ViewComponent |
|---|---|---|
| Kendi verisini çekebilir mi | **Hayır** — veriyi dışarıdan alır | **Evet** — kendi C# sınıfı var |
| DI alabilir mi | Hayır | Evet |
| Async çalışabilir mi | Sınırlı | Evet |
| Ne zaman | Aynı HTML'i tekrar etmek | Bağımsız, veri gerektiren bölüm |

Kural basit: **Veri gerekiyorsa ViewComponent, sadece HTML tekrarıysa partial view.**

Sepetteki ürün sayısını her sayfanın üst köşesinde gösteren bir bileşen, tipik bir ViewComponent işidir — çünkü veriyi kendisi çekmesi gerekir:

```csharp
public class SepetOzetiViewComponent : ViewComponent
{
    private readonly ISepetServisi _sepet;

    // ViewComponent DI alabilir; partial view alamaz.
    public SepetOzetiViewComponent(ISepetServisi sepet) => _sepet = sepet;

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var adet = await _sepet.UrunSayisiAsync(User.Identity.Name);
        return View(adet);   // Views/Shared/Components/SepetOzeti/Default.cshtml
    }
}
```

```cshtml
@* Layout içinde tek satır — her sayfa bunu çağırır *@
<vc:sepet-ozeti></vc:sepet-ozeti>
```

> **Bu benzetme şurada bozulur:** Tarif defterinde "sayfa 12'ye bak" dediğinde okuyucu gidip bakar; sayfa 12 kendi başına anlamlı bir tariftir. Partial view ise **kendi başına bir sayfa değildir** ve kendi verisini de getiremez. Ona ne verirsen onu basar; ihtiyacı olan veriyi sen taşımak zorundasın. Verinin kendisini de getirmesini istiyorsan defterden çıkıp aşçıyı çağırman gerekir — o da ViewComponent'tir.

---

## 7. Bu Ayrım Ne Kazandırır

> **Benzetme —** Bir inşaatta elektrikçi, sıhhi tesisatçı ve boyacı aynı anda çalışabilir; çünkü kimin hangi duvara, hangi borulara dokunacağı bellidir. Sınırlar silinirse boyacı su borusunu deler ve herkes durur. Katman ayrımı, aynı binada aynı anda çalışabilmenin şartıdır.

**Basitçe:** Bu ayrımın asıl kazancı teoride güzel durması değil, ekibin aynı anda birbirini bozmadan çalışabilmesidir. Tasarımcı ekranı değiştirirken, sen iş mantığını değiştirebilirsin.

**Teknik olarak:** MVC'nin asıl faydası, **farklı insanların farklı dosyalarda çalışabilmesi**:

- Tasarımcı / front-end geliştirici → `.cshtml` dosyaları
- Back-end geliştirici → Controller ve Model
- İkisi de aynı anda, birbirini bozmadan

Bu ayrım korunmazsa (view'da SQL sorgusu, controller'da HTML üretimi) MVC'nin adı kalır, faydası kalmaz.

İkinci kazanç test edilebilirliktir. İş mantığı Model tarafındaysa, onu HTTP'siz ve tarayıcısız test edebilirsin:

```csharp
// İş kuralı View'da değil, sınıfın içinde olduğu için
// tek satırlık bir testle doğrulanabilir.
[Fact]
public void KdvDahilFiyat_YuzdeYirmiEkler()
{
    var urun = new UrunGoruntuleme { FiyatKdvHaric = 100m };
    Assert.Equal(120m, urun.FiyatKdvDahil);
}
```

Aynı kural bir `.cshtml` dosyasının içinde yazılmış olsaydı, bunu test etmek için sunucu ayağa kaldırıp sayfayı istemen ve üretilen HTML metnini ayrıştırman gerekirdi. "Doğru katman" sorusunun somut karşılığı budur.

> **Bu benzetme şurada bozulur:** İnşaatta duvar gerçekten vardır; boyacı su borusuna çekiçle vurursa herkes duyar. Yazılımda ihlaller sessizdir. View içine konmuş bir veritabanı sorgusu çalışır, sayfa açılır, kimse bir şey fark etmez — bedeli aylar sonra, o sorguyu değiştirmek gerektiğinde ortaya çıkar. Bu yüzden katman disiplini "ceza korkusuyla" değil, alışkanlıkla korunur.

---

## Tek Bakışta Özet

- **Model** veri + iş mantığı, **View** sadece gösterim, **Controller** trafik yönetimi. Controller'ın şişmesi yanlış katman işaretidir.
- **Razor** = C# + HTML. Tek kural `@`. Bastığı her değeri HTML-encode eder (XSS koruması). Hesap yapıyorsan parantez kullan: `@(adet * fiyat)`.
- Veri taşımada üç yol: **strongly-typed model** (tip güvenli, asıl veri için), **ViewBag** ve **ViewData** (tip güvensiz, küçük yan bilgiler için). `ViewBag` = `ViewData`'nın dynamic hâli. Üçünün de ömrü tek istektir.
- **Entity ≠ ViewModel.** Entity veritabanını, ViewModel ekranı temsil eder. Ayrım hem güvenlik hem esneklik sağlar; bedeli iki sınıfı senkron tutmaktır.
- **Tag helper** (`asp-for`) bugünün yöntemi; **Html helper** (`@Html.TextBoxFor`) eski kodda karşına çıkar. Tag helper'lar `_ViewImports.cshtml`'deki `@addTagHelper` satırına bağlıdır.
- **Partial view** HTML tekrarı içindir; veri gerekiyorsa **ViewComponent**.
- Katman ayrımının somut kazancı: aynı anda çalışabilmek ve iş mantığını sunucu ayağa kaldırmadan test edebilmek.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| MVC | Veri, görünüm ve akış kontrolünü ayıran mimari desen |
| Razor | HTML içinde C# yazmayı sağlayan şablon motoru (`.cshtml`) |
| `@model` / `@Model` | View'ın tipini tanımlar / o tipteki nesneye erişir |
| Razor kod bloğu | `@{ ... }` — view içinde birden çok satır C# çalıştırma |
| Strongly-typed view | Tek bir tip üzerinden veri alan, tip güvenli view |
| ViewBag | Controller'dan view'a veri taşıyan `dynamic` nesne |
| ViewData | Aynı işi yapan sözlük yapısı; ViewBag'in alt katmanı |
| Entity | Veritabanı tablosunu temsil eden sınıf |
| ViewModel / DTO | Yalnızca bir ekranın ihtiyacı olan veriyi taşıyan sınıf |
| Over-posting | Formdan, gönderilmemesi gereken alanların da gönderilmesi saldırısı |
| Tag Helper | HTML etiketi gibi görünen, sunucuda çalışan yardımcı (`asp-for`) |
| Html Helper | Metot çağrısıyla HTML üreten eski nesil yardımcı |
| `_ViewImports.cshtml` | Tüm view'lara ortak `@using` / `@addTagHelper` satırlarını taşıyan dosya |
| Partial View | Tekrar eden HTML parçasının ayrı dosyaya çıkarılması |
| ViewComponent | Kendi verisini çekebilen, DI alabilen bağımsız görünüm bileşeni |
| HTML encoding | Metindeki HTML karakterlerinin zararsız hâle getirilmesi |
| `@Html.Raw` | Encoding'i devre dışı bırakıp ham HTML basma (dikkatli kullanılır) |
| XSS | Sayfaya kod enjekte edip başka kullanıcının tarayıcısında çalıştırma saldırısı |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `02-Program-cs-ve-Pipeline.md`
