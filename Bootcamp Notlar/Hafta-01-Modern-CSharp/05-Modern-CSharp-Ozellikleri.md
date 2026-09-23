# Hafta 1 · Cuma — Modern C# Özellikleri

**Okuma süresi:** ~52 dk
**Neden bu konu:** Bootcamp'te göreceğin kodlar C# 9–12 sözdizimiyle yazılmış olacak. Bu özellikleri bilmemek "kodu okuyamamak" demektir. Ayrıca nullable reference types, `NullReferenceException`'ı derleme anına taşıyan tek pratik araçtır.

---

## Önce Basitçe

Bir dil yıllar içinde değişir. Yirmi yıl önceki bir gazeteyi bugün okuduğunda kelimeler tanıdıktır ama cümle kurma biçimi eskidir; tersine, bugünün gazetesini yirmi yıl önceki birine versen bazı kısaltmalara takılır. C# da böyle bir dil. Yirmi küsur yaşında ve her yıl yeni yazım biçimleri ekleniyor. Eskiler çöpe atılmıyor — eski yazım hâlâ çalışıyor — ama yeni yazım kısa olduğu için insanlar ona geçiyor. Sonuçta ortaya şu durum çıkıyor: aynı işi yapan kodun beş farklı yazılışı var ve sen hepsini okuyabilmek zorundasın.

Bu notta anlatılan şeylerin neredeyse tamamı **kısaltma** ya da **güvenlik ağı**. İki grup gibi düşün. Birinci grup, zaten yapabildiğin bir şeyi daha az yazıyla yapmanı sağlıyor: uzun uzun tip adı yazmak yerine `var`, yirmi satırlık bir veri sınıfı yerine tek satırlık `record`, dosyanın başındaki on satır `using` yerine hiçbir şey. Bunlar programın davranışını değiştirmiyor, sadece gözünü yormuyor. İkinci grup daha ciddi: hata yapmanı engellemeye çalışıyor. Bir değerin boş olabileceğini derleyiciye söyleyebiliyorsun, bir nesnenin kurulduktan sonra değişmemesini garantileyebiliyorsun, bir alanı doldurmayı unutursan program derlenmiyor bile. Bu ikinci grup, hatayı sen fark etmeden önce derleyicinin fark etmesini sağlıyor.

En çok kafa karıştıran konu ikinci gruptan: **null**. Bir değişkenin "hiçbir şeye işaret etmiyor" hâline null denir ve programların en sık çöktüğü yer burasıdır. Eskiden C#, bir değerin boş olup olamayacağını hiç bilmezdi; her şey boş olabilirdi ve sen unutursan program çalışırken patlardı. Modern C#, bu bilgiyi tipin kendisine yazmanı istiyor: "bu alan asla boş kalmayacak" ya da "bu alan boş olabilir, dikkat et". Söyledikten sonra derleyici seni kollamaya başlıyor. Ama burada dürüst olmak gerek: bu kolluk çalışma anında değil, yazma anında. Dışarıdan gelen veri kuralı hâlâ çiğneyebilir.

Bir de kodun **şeklini sorgulama** meselesi var, buna pattern matching diyorlar. Normalde bir nesnenin ne olduğunu anlamak için üst üste `if` yazarsın: tipini kontrol et, sonra içindeki alanı kontrol et, sonra o alanın değerini kontrol et. Modern C# bunların hepsini tek satırda sormanı sağlıyor. Sonuç, on satırlık iç içe kontrolün tek okunur cümleye inmesi. İş kuralları yazarken en çok işine yarayacak şey bu.

Bu notta sürüm sürüm ne geldiğini, her birinin ne işe yaradığını ve — daha önemlisi — nerede yanlış kullanıldığını göreceksin. Ezberlemene gerek yok; hangi yazımı gördüğünde ne düşüneceğini bilmen yeterli. Şimdi detaya iniyoruz.

> **Ana benzetme:** Modern C# özellikleri, eski bir **mutfağın yenilenmesi** gibidir. Ocak da tencere de yerinde duruyor, eski usulle yemek hâlâ pişiyor. Ama artık rondo var, ölçülü kaplar var, fırının kapağında "içerideki sıcak" uyarısı var. Rondoyu kullanmak zorunda değilsin; ancak bu mutfağa girip yemek yapan başkasının tarifini okuyacaksan, rondonun ne olduğunu bilmek zorundasın.

---

## Bu Notta Ne Var

1. C# sürümleri ve dil–platform ilişkisi
2. `var`, hedef tipli `new`, `nameof`
3. Nullable reference types
4. `record` ve değer eşitliği
5. `init`, `required` ve nesne değişmezliği
6. Pattern matching
7. Switch expression
8. String interpolation ve raw string
9. Top-level statements, global using, file-scoped namespace
10. Null işleçleri ve savunmacı yazım
11. Diğer faydalı özellikler

---

## 1. C# Sürümleri

> **Benzetme —** Trafik yönetmeliği her birkaç yılda bir güncellenir. Yeni tabelalar eklenir, bazı işaretlerin anlamı netleşir. Sen o yeni tabelayı kendi sokağına asmak zorunda değilsin; ama başka bir şehre gidip yolda görürsen ne demek olduğunu bilmek zorundasın. Üstelik hangi tabelaların geçerli olduğu, hangi yılın yönetmeliğine tabi olduğuna bağlıdır.

**Basitçe:** C#'ın numaralı sürümleri var ve her sürüm birkaç yeni yazım biçimi getiriyor. Hangi sürümü kullanabileceğini sen tek tek seçmiyorsun; projenin hedeflediği .NET sürümü bunu belirliyor. Yani ".NET 8 kullanıyorum" demek, aynı zamanda "C# 12 yazabilirim" demek.

**Teknik olarak:** Dil sürümü, hedef aldığın .NET sürümüne bağlıdır — `.csproj` dosyasındaki `<TargetFramework>` dil sürümünü de belirler.

| Sürüm | Öne çıkan özellikler |
|---|---|
| **C# 8** | Nullable reference types, switch expression, `using` declaration, `IAsyncEnumerable` |
| **C# 9** | `record`, `init`, top-level statements, hedef tipli `new` |
| **C# 10** | File-scoped namespace, global using, `record struct` |
| **C# 11** | Raw string literals, `required` üyeler, generic attribute'lar |
| **C# 12** | Primary constructor (sınıflarda), koleksiyon ifadeleri, alias any type |

Yeni sözdizimi kullanmak zorunda değilsin; ama başkasının kodunu okumak zorundasın.

Eşleşmeyi somut görmek istersen:

```xml
<!-- .csproj -->
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>   <!-- varsayılan dil sürümü: C# 12 -->
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

Dil sürümünü elle zorlamak mümkündür ama önerilmez:

```xml
<LangVersion>11.0</LangVersion>   <!-- geriye çekmek: nadiren gerekir -->
<LangVersion>preview</LangVersion> <!-- henüz kesinleşmemiş özellikler: üretimde kullanma -->
```

> Bir özelliğin "derleyici özelliği" mi yoksa "çalışma zamanı özelliği" mi olduğu önemlidir. `record`, `init`, pattern matching gibi şeyler derleyicinin ürettiği koddur; eski çalışma zamanlarında bile çoğu zaman derlenir. Buna karşılık `IAsyncEnumerable` gibi yeni tip gerektiren özellikler, çalışma zamanının da o tipi tanımasını ister.

---

## 2. `var`, Hedef Tipli `new`, `nameof`

> **Benzetme —** Kargo kolisinin üstüne etiket yaparsın. Kolinin içinde ne olduğu paketleme anında bellidir; aynı bilgiyi hem koliye hem faturaya hem de etikete tekrar tekrar yazmak zaman kaybıdır. Ama içinde ne olduğu belirsiz bir koli varsa etiket hayat kurtarır. `var` tam olarak bu ölçüdür: bilgi zaten görünüyorsa tekrar yazma, görünmüyorsa yaz.

**Basitçe:** Bu üçü de "aynı şeyi iki kez yazma" derdine çözüm. `var`, tipi derleyicinin anlamasını sağlar. Hedef tipli `new`, sol tarafta yazdığın tipi sağda tekrarlamaktan kurtarır. `nameof` ise bir şeyin adını tırnak içinde elle yazmak yerine derleyiciden alır — böylece adı değiştirdiğinde metin de kendiliğinden değişir.

**Teknik olarak:**

**`var`** — Tipin derleyici tarafından çıkarılması. **Dinamik tip değildir**; derleme anında tip kesinleşir, sonradan değişemez.

```csharp
var liste = new List<Musteri>();     // tip nettir, okunabilir
var x = HesaplaBirSey();             // tip belirsiz — burada açık yazmak daha iyi
```

Ölçü: sağ taraf tipi zaten söylüyorsa `var` kullan; söylemiyorsa açık yaz.

`var`'ın statik olduğunu kanıtlayan en basit örnek:

```csharp
var sayi = 5;
sayi = "beş";        // derleme hatası: sayi artık int'tir, değiştirilemez

dynamic d = 5;
d = "beş";           // sorun yok — dynamic gerçekten dinamiktir, kontrol çalışma anına kalır
```

`var` kullanılamayacağı yerler de vardır; bunlar sınavlık değil ama karşına çıkar:

```csharp
var x;                       // hata: başlangıç değeri yok, tip çıkarılamaz
var y = null;                // hata: null'ın tipi yoktur
var? z = BulabilirsinBelki(); // hata: var zaten nullable bilgisini taşır
// alan (field) tanımında da kullanılamaz — yalnızca yerel değişkenlerde
```

**Hedef tipli `new`** (C# 9) — Sol taraf tipi belirtiyorsa sağda tekrar yazmaya gerek yok.

```csharp
List<Musteri> liste = new();
private readonly Dictionary<string, int> _cache = new();
```

En çok kazandırdığı yer, tip adının uzun olduğu alan tanımlarıdır:

```csharp
// Öncesi
private readonly Dictionary<string, List<SiparisSatiri>> _sepet
    = new Dictionary<string, List<SiparisSatiri>>();

// Sonrası
private readonly Dictionary<string, List<SiparisSatiri>> _sepet = new();
```

Parametre varsayılanlarında ve metot çağrılarında da çalışır:

```csharp
void Kaydet(Ayarlar ayar) { }
Kaydet(new());                          // Ayarlar örneği üretilir

void Yaz(StringBuilder sb = null!) { }
List<Musteri> Bos() => new();           // dönüş tipinden çıkarım
```

> `var` ile hedef tipli `new` birbirinin zıddıdır: `var` tipi **sağdan** okur, `new()` tipi **soldan** okur. İkisini aynı satırda kullanamazsın — `var x = new();` derlenmez, çünkü ortada tipi söyleyen kimse kalmaz.

**`nameof`** — Bir değişkenin, özelliğin veya tipin adını **string olarak** verir. Yeniden adlandırma yapıldığında string de otomatik güncellenir; elle yazılan string ise sessizce bozulur.

```csharp
throw new ArgumentNullException(nameof(musteri));
_logger.LogError("{Alan} boş olamaz", nameof(Musteri.Email));
```

`nameof` tamamen derleme anında çözülür — çalışma anında hiçbir maliyeti yoktur, üretilen IL'de sadece düz bir metin durur:

```csharp
var s = nameof(Musteri.Email);   // derleyici bunu "Email" sabitine çevirir
```

Bunun en kritik kullanıldığı yer, metin hâlindeki üye adlarının kod ile bağını korumaktır:

```csharp
public class Musteri : INotifyPropertyChanged
{
    private string _ad = "";
    public string Ad
    {
        get => _ad;
        set { _ad = value; Bildir(nameof(Ad)); }   // "Ad" yazsaydın, alan adı değişince bağ kopardı
    }
    // ...
}
```

---

## 3. Nullable Reference Types (NRT)

> **Benzetme —** Resmi bir form doldurduğunu düşün. Bazı alanların yanında yıldız vardır: "zorunlu". Boş bırakıp memura verirsen memur formu geri çevirir, içeri bile sokmaz. Yıldızsız alanları boş bırakabilirsin ama sonraki aşamada onları kullanacak kişi "burası boş olabilir" diye önce bakmak zorundadır.
>
> **Bu benzetme şurada bozulur:** Memur sadece elden verilen formu kontrol eder. Posta kutusundan gelen, faksla düşen ya da eski bir arşivden çıkan formu kimse kontrol etmemiştir — zorunlu alan boş olabilir. NRT'de de aynısı olur: senin yazdığın kodu derleyici denetler, ama JSON'dan, veritabanından, reflection'dan gelen veri bu denetimin dışında kalır.

**Basitçe:** Bir değişkenin "boş" olabileceğini tipin yanına koyduğun soru işaretiyle söylüyorsun. `string` dersen "burası asla boş kalmayacak", `string?` dersen "boş olabilir" demiş oluyorsun. Bunu söyledikten sonra derleyici seni takip etmeye başlıyor: boş olabilecek bir şeyi kontrol etmeden kullanırsan uyarı veriyor. Program çalışırken patlamasını beklemek yerine, daha yazarken haberin oluyor.

**Teknik olarak:** C# 8 ile gelen, `NullReferenceException`'ı **derleme anında** yakalamayı hedefleyen özellik.

Açıldığında (`.csproj` içinde `<Nullable>enable</Nullable>` — yeni projelerde varsayılan) reference type'ların anlamı değişir:

| Yazım | Anlamı |
|---|---|
| `string ad` | **null olamaz.** `null` atarsan derleyici uyarır |
| `string? ad` | null olabilir. Kullanmadan önce kontrol etmezsen uyarı alırsın |

```csharp
public class Musteri
{
    public string Ad { get; set; } = string.Empty;   // null olmayacak, başlangıç değeri şart
    public string? Aciklama { get; set; }            // null olabilir
}

void Yazdir(Musteri m)
{
    Console.WriteLine(m.Ad.ToUpper());            // güvenli
    Console.WriteLine(m.Aciklama.ToUpper());      // uyarı: burada null olabilir
    Console.WriteLine(m.Aciklama?.ToUpper());     // güvenli
}
```

**Akış analizi (flow analysis)** — Derleyici sadece tipe değil, o satıra kadar yazdığın kontrollere de bakar. Kontrol ettiğin andan itibaren uyarı kesilir:

```csharp
void Yazdir2(Musteri m)
{
    if (m.Aciklama is null) return;

    Console.WriteLine(m.Aciklama.ToUpper());   // uyarı yok: derleyici null olamayacağını biliyor
}

void Yazdir3(string? metin)
{
    if (string.IsNullOrEmpty(metin))
        return;

    Console.WriteLine(metin.Length);           // uyarı yok: IsNullOrEmpty'nin imzası bunu bildiriyor
}
```

Bu son örnekteki sihir, .NET'in kendi metotlarına eklenmiş **null durum öznitelikleridir**. Kendi yardımcı metotlarına da ekleyebilirsin:

```csharp
using System.Diagnostics.CodeAnalysis;

public static class Dogrula
{
    // "false dönersem, parametre kesinlikle null değildir"
    public static bool BosMu([NotNullWhen(false)] string? deger)
        => string.IsNullOrWhiteSpace(deger);
}

if (!Dogrula.BosMu(ad))
    Console.WriteLine(ad.Length);   // uyarı yok
```

| Öznitelik | Söylediği |
|---|---|
| `[NotNull]` | Metottan çıkışta bu değer null değildir |
| `[NotNullWhen(true/false)]` | Metot şu değeri döndürürse parametre null değildir |
| `[MaybeNull]` | Dönüş değeri, tipi null kabul etmiyor görünse bile null olabilir |
| `[MemberNotNull(nameof(...))]` | Bu metot çalıştıktan sonra şu alan null değildir (başlatma metotları için) |

**Kritik nokta:** Bu bir **derleme anı analizidir**, çalışma anı garantisi değildir. Dışarıdan gelen veri (JSON deserialization, veritabanı, reflection) `null` olmayan bir alana `null` koyabilir. NRT hataları azaltır, ortadan kaldırmaz.

```csharp
public class Ayar { public string Anahtar { get; set; } = string.Empty; }

// JSON'da "anahtar" alanı hiç yoksa, deserializer alanı null bırakabilir.
var ayar = JsonSerializer.Deserialize<Ayar>("{}")!;
Console.WriteLine(ayar.Anahtar.Length);   // derleyici memnun; çalışma anında patlayabilir
```

Bu yüzden **sınırlarda doğrulama** yaparsın: dışarıdan içeri giren her noktada (controller, deserializer, veritabanı okuma) değeri bir kez kontrol et; içeride NRT'ye güven.

**`!` — null-forgiving operatörü** — "Ben biliyorum, burada null değil" der ve uyarıyı susturur. Analizin gerçekten yanıldığı yerlerde kullanılır; uyarıdan kurtulmak için serpiştirilirse özelliğin tüm faydası kaybolur.

```csharp
var musteri = _repo.Bul(id)!;    // gerçekten emin değilsen kullanma
```

Uyarıları proje genelinde hataya çevirmek, ekipçe disiplini korumanın en pratik yoludur:

```xml
<Nullable>enable</Nullable>
<WarningsAsErrors>nullable</WarningsAsErrors>
```

Mevcut büyük bir projeye NRT'yi bir günde açamazsın. Kademeli geçiş için dosya başına açıp kapatabilirsin:

```csharp
#nullable enable
// bu dosyada analiz açık
#nullable restore
```

---

## 4. `record` ve Değer Eşitliği

> **Benzetme —** Cebindeki iki yüz liralık banknotu düşün. Seri numaraları farklıdır ama biri diğerinin yerine geçer; markette "bu benim banknotum değil" diye bir itiraz olmaz, çünkü önemli olan üstündeki değerdir. Kimlik kartı ise böyle değildir: iki kişinin adı ve doğum tarihi aynı olsa bile kartlar birbirinin yerine geçmez, çünkü kart bir **kişiyi** işaret eder. `record` banknottur, `class` kimlik kartıdır.

**Basitçe:** Normal bir sınıfta iki nesne, ancak bellekte aynı nesneyse eşit sayılır — içlerindeki bilgiler birebir aynı olsa bile. `record` bu kuralı tersine çevirir: içindekiler aynıysa eşittir. Veriyi taşımak için kullandığın nesnelerde istediğin davranış budur.

**Teknik olarak:** **`record`** — Veriyi temsil etmek için tasarlanmış, **değer eşitliğine** sahip reference type.

Normal bir `class`'ta iki nesne ancak aynı nesneyse eşittir. `record`'da tüm alanları aynıysa eşittir.

```csharp
public record MusteriDto(string Ad, string Email);

var a = new MusteriDto("Umut", "u@x.com");
var b = new MusteriDto("Umut", "u@x.com");

Console.WriteLine(a == b);          // True
Console.WriteLine(a);               // MusteriDto { Ad = Umut, Email = u@x.com }
```

Aynı veriyi `class` ile yazdığında farkı net görürsün:

```csharp
public class MusteriSinif
{
    public string Ad { get; init; } = "";
    public string Email { get; init; } = "";
}

var c = new MusteriSinif { Ad = "Umut", Email = "u@x.com" };
var d = new MusteriSinif { Ad = "Umut", Email = "u@x.com" };

Console.WriteLine(c == d);                      // False — referanslar farklı
Console.WriteLine(ReferenceEquals(c, d));       // False
Console.WriteLine(c);                           // Namespace.MusteriSinif  (yararsız çıktı)
```

Derleyici `record` için otomatik üretir: `Equals`, `GetHashCode`, `ToString`, deconstructor ve `with` desteği.

Deconstructor sayesinde parçalarına ayırabilirsin:

```csharp
var (ad, email) = a;                 // positional record'lar deconstruct edilebilir
Console.WriteLine(ad);               // Umut
```

**`with` ifadesi** — Mevcut nesnenin bir kopyasını, belirtilen alanları değiştirerek üretir. Orijinal değişmez.

```csharp
var guncel = a with { Email = "yeni@x.com" };
Console.WriteLine(a.Email);          // u@x.com   — orijinal dokunulmadı
Console.WriteLine(guncel.Email);     // yeni@x.com
Console.WriteLine(a == guncel);      // False — bir alan farklı
```

> **`with` yüzeysel (shallow) kopya üretir.** İçinde bir liste varsa, kopya ile orijinal **aynı listeyi** paylaşır. Birine eleman eklersen diğerinde de görünür. Bu, "record değişmezdir" sanısının en sık kırıldığı yerdir.

```csharp
public record Sepet(string Sahip, List<string> Urunler);

var s1 = new Sepet("Umut", new List<string> { "ekmek" });
var s2 = s1 with { Sahip = "Ali" };

s2.Urunler.Add("süt");
Console.WriteLine(s1.Urunler.Count);   // 2 — liste paylaşıldı
```

Gövdeli (nominal) yazım da mümkündür; positional yazım zorunlu değildir:

```csharp
public record Adres
{
    public required string Sehir { get; init; }
    public string? Ilce { get; init; }

    public string Ozet => Ilce is null ? Sehir : $"{Ilce}/{Sehir}";
}
```

Kalıtım da vardır ve eşitlik **tip duyarlıdır**: derleyici gizli bir `EqualityContract` üyesi üretir, bu yüzden farklı tipler alanları aynı olsa bile eşit çıkmaz.

```csharp
public record Kisi(string Ad);
public record Ogrenci(string Ad, string Okul) : Kisi(Ad);

Kisi k = new Kisi("Umut");
Kisi o = new Ogrenci("Umut", "ODTU");
Console.WriteLine(k == o);    // False — tipler farklı
```

**Ne zaman `record`:** DTO, API isteği/yanıtı, konfigürasyon nesnesi, domain value object — yani kimliği değil **içeriği** önemli olan her şey.
**Ne zaman `class`:** Kimliği olan, durumu zamanla değişen varlıklar — EF Core entity'leri gibi.

EF Core entity'lerinde `record` kullanmamanın somut sebebi: EF, nesneyi `Id` üzerinden takip eder ve alanlarını değiştirir. Değer eşitliği bu takibi bozar, `init`-only özellikler de EF'in materyalizasyonunu zorlaştırır.

**`record struct`** — Aynı davranışın value type hâli. Küçük ve değişmez veri parçaları için.

```csharp
public readonly record struct Para(decimal Tutar, string Birim);

var p1 = new Para(100m, "TRY");
var p2 = new Para(100m, "TRY");
Console.WriteLine(p1 == p2);    // True — heap'te yer kaplamadan değer eşitliği
```

`readonly record struct` yazmak alışkanlık hâline getirilmelidir: aksi hâlde struct değiştirilebilir kalır ve kopyalanma davranışı sürpriz üretir.

---

## 5. `init`, `required` ve Değişmezlik

> **Benzetme —** Beton dökülürken şekil verirsin. Kalıbı kurar, demiri yerleştirir, betonu dökersin — bu aşamada her şey ayarlanabilir. Beton donduktan sonra "şu duvarı yarım metre sola alalım" diyemezsin. `init` tam olarak budur: nesne kurulurken ayarlarsın, kurulum bitince taşlaşır. `required` ise kalıbı kurarken "bu demiri koymadan beton dökemezsin" diyen ustadır.

**Basitçe:** Bazı bilgiler bir nesne oluşturulurken verilir ve ondan sonra bir daha değişmemelidir — veritabanı bağlantı adresi gibi. `init` bunu sağlar: sadece kurulum anında yazabilirsin. `required` ise bir adım öteye gider ve o bilgiyi vermeyi unutursan programı derletmez.

**Teknik olarak:**

**`init`** — Özelliğin **yalnızca nesne oluşturulurken** atanabilmesi. Sonrasında salt okunur olur.

```csharp
public class Ayarlar
{
    public string BaglantiCumlesi { get; init; }
    public int ZamanAsimi { get; init; }
}

var a = new Ayarlar { BaglantiCumlesi = "...", ZamanAsimi = 30 };
a.ZamanAsimi = 60;    // derleme hatası
```

`readonly` alanla farkı: `init` nesne başlatıcı sözdizimiyle (`{ ... }`) çalışır, constructor'a parametre eklemek gerekmez.

Üç yazımı yan yana koyunca fark netleşir:

```csharp
public class Ornek
{
    public string A { get; set; }          // her zaman değiştirilebilir
    public string B { get; init; }         // yalnızca nesne başlatıcıda
    public string C { get; }               // yalnızca constructor içinde

    public Ornek(string c) { C = c; }
}

var o = new Ornek("x") { A = "1", B = "2" };
o.A = "3";     // serbest
o.B = "4";     // derleme hatası
```

**`required`** (C# 11) — Nesne oluşturulurken bu üyenin **mutlaka** atanmasını zorunlu kılar.

```csharp
public class Kullanici
{
    public required string Email { get; init; }
    public string? Telefon { get; init; }
}

var k = new Kullanici();                        // Email atanmadı — derleme hatası
var k2 = new Kullanici { Email = "a@b.com" };   // geçerli
```

`required` + `init` birleşimi, constructor yazmadan "eksiksiz ve değişmez nesne" üretmenin en temiz yoludur.

`required`, NRT ile birlikte kullanıldığında bir başka derdi de çözer: artık `= string.Empty;` gibi anlamsız başlangıç değerleri yazmak zorunda kalmazsın.

```csharp
// Öncesi: derleyiciyi susturmak için uydurma varsayılan
public string Ad { get; set; } = string.Empty;

// Sonrası: gerçek niyet — "bunu vermek zorundasın"
public required string Ad { get; init; }
```

Bir constructor'ın `required` üyeleri kendisinin doldurduğunu söylemesi gerekirse:

```csharp
public class Rapor
{
    public required string Baslik { get; init; }

    public Rapor() { }

    [SetsRequiredMembers]                       // System.Diagnostics.CodeAnalysis
    public Rapor(string baslik) => Baslik = baslik;
}

var r = new Rapor("Aylık");    // nesne başlatıcı gerekmiyor: constructor sorumluluğu üstlendi
```

**Primary constructor** (C# 12) — Sınıf adının yanına doğrudan parametre yazılması. Özellikle DI ile servis sınıflarında kod kısaltır.

```csharp
public class MusteriService(IMusteriRepository repo, ILogger<MusteriService> logger)
{
    public Task<Musteri> GetirAsync(int id) => repo.BulAsync(id);
}
```

Kazancı, klasik yazımla yan yana koyunca görülür:

```csharp
// Öncesi — üç satır tekrar
public class MusteriService
{
    private readonly IMusteriRepository _repo;
    private readonly ILogger<MusteriService> _logger;

    public MusteriService(IMusteriRepository repo, ILogger<MusteriService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
}
```

> Primary constructor parametreleri sınıfın her yerinden görünür ama **otomatik olarak `readonly` alan değildir**. Metot içinde parametreye atama yapabilirsin ve bu, gizli bir alanı değiştirir. Değişmezlik istiyorsan `private readonly` alana kendin kopyala.

`record` ile `class` primary constructor'ı arasındaki farkı karıştırma:

```csharp
public record RKisi(string Ad);     // Ad bir public özelliktir
public class CKisi(string Ad);      // Ad yalnızca bir parametredir, dışarıdan görünmez
```

---

## 6. Pattern Matching

> **Benzetme —** Postanenin ayrıştırma bandını düşün. Gelen paketi tek bakışta değerlendirirsin: bu bir zarf mı koli mi, ağırlığı ne, üstünde "kırılacak eşya" yazıyor mu, gideceği il hangisi. Bu soruları tek tek sıraya dizmezsin; pakete bakarken hepsini aynı anda görürsün ve doğru sepete atarsın. Pattern matching, koda bu bakışı kazandırır.

**Basitçe:** Bir değeri sorgularken normalde üst üste `if` yazarsın: önce tipi, sonra null olup olmadığı, sonra içindeki alanın değeri. Pattern matching bu soruların hepsini tek bir ifadede sormanı sağlar. Hem de sorarken içindeki parçayı bir değişkene alır, ayrıca cast yazmana gerek kalmaz.

**Teknik olarak:** **Pattern matching (desen eşleme)** — Bir değerin tipini, şeklini veya içeriğini tek bir ifadede kontrol edip parçalarını yakalama.

Eski yazımla farkı:

```csharp
// Öncesi — üç adım, biri unutulursa çalışma anı hatası
if (nesne != null && nesne is Musteri)
{
    var m = (Musteri)nesne;
    Console.WriteLine(m.Ad);
}

// Sonrası — tek adım
if (nesne is Musteri m)
    Console.WriteLine(m.Ad);
```

### Tip deseni
```csharp
if (nesne is Musteri m)          // hem kontrol eder hem değişkene atar
    Console.WriteLine(m.Ad);
```

### Sabit ve null deseni
```csharp
if (deger is null) { ... }
if (deger is not null) { ... }
if (durum is 1 or 2 or 3) { ... }
```

> `is null` ile `== null` aynı şey değildir: `==` operatörü aşırı yüklenebilir (overload), `is null` yüklenemez. Bir tip `==` davranışını değiştirmişse `is null` yine de gerçek null kontrolü yapar. Bu yüzden kütüphane kodunda `is null` tercih edilir.

### İlişkisel ve mantıksal desenler
```csharp
string Kategori(decimal fiyat) => fiyat switch
{
    < 100            => "Ucuz",
    >= 100 and < 500 => "Orta",
    >= 500           => "Pahalı",
    _                => "Bilinmiyor"
};
```

`and`, `or`, `not` desen birleştiricileridir ve okunabilirliği ciddi artırır:

```csharp
bool GecerliHarf(char c) => c is >= 'a' and <= 'z' or >= 'A' and <= 'Z';
bool CalismaGunu(DayOfWeek g) => g is not (DayOfWeek.Saturday or DayOfWeek.Sunday);
```

### Özellik deseni
```csharp
if (siparis is { Durum: "Onaylandi", Tutar: > 1000 })
    IndirimUygula(siparis);
```

İç içe de girebilir ve yakalanan parçayı adlandırabilirsin:

```csharp
if (siparis is { Musteri: { Sehir: "Ankara" } m, Tutar: > 500 })
    Console.WriteLine(m.Ad);

// C# 10 ile kısaltılmış nokta yazımı
if (siparis is { Musteri.Sehir: "Ankara" })
    KargoyuUcretsizYap(siparis);
```

Özellik deseni aynı zamanda sessiz bir null kontrolüdür: `siparis` null ise desen eşleşmez, `NullReferenceException` atmaz.

```csharp
Musteri? m = null;
Console.WriteLine(m is { Ad.Length: > 0 });   // False — patlamaz
```

### Liste deseni (C# 11)
```csharp
if (dizi is [1, 2, ..])          // ilk iki eleman 1 ve 2, gerisi önemsiz
```

Liste desenlerinin daha kullanışlı biçimleri:

```csharp
string Anlat(int[] d) => d switch
{
    []            => "boş",
    [var tek]     => $"tek eleman: {tek}",
    [var ilk, .., var son] => $"{ilk} ile başlıyor, {son} ile bitiyor",
    _             => "bilinmiyor"
};

// Yakalayarak dilim alma
if (parcalar is [var komut, .. var argumanlar])
    Calistir(komut, argumanlar);
```

### `var` deseni ve atma (discard)
```csharp
// var deseni: her zaman eşleşir, değeri bir isme bağlar — ara hesaplama için
if (Hesapla(x) is var sonuc && sonuc > 10) { ... }

// _ : değeri umursamıyorum
if (nokta is (_, 0)) { ... }    // y ekseni üzerindeki noktalar
```

Pattern matching, iç içe `if` yığınlarını tek okunur ifadeye indirir. Bootcamp'te iş kuralları yazarken sürekli işine yarayacak.

---

## 7. Switch Expression

> **Benzetme —** Eski `switch`, kurumdaki memura soru sormak gibiydi: sen sorarsın, memur önündeki kâğıda bir şey yazar, sen sonra o kâğıdı alıp okursun. Yeni `switch` ise otomat makinesidir — tuşa basarsın, ürün elinde. Aradaki kâğıt, yani geçici değişken, ortadan kalkar.

**Basitçe:** Eski `switch` bir değer üretmezdi; bir değişken tanımlar, her dalda o değişkene atama yapardın. Yeni yazım doğrudan sonucu döndürür. Daha kısa olmasının yanında, derleyici "bütün ihtimalleri kapattın mı" diye de kontrol eder.

**Teknik olarak:** Klasik `switch` bir **ifade** değil, deyimdi — değer döndüremezdi. C# 8 ile geldi:

```csharp
// Eski
string sonuc;
switch (kod)
{
    case 1: sonuc = "Bir"; break;
    case 2: sonuc = "İki"; break;
    default: sonuc = "Bilinmiyor"; break;
}

// Yeni
string sonuc2 = kod switch
{
    1 => "Bir",
    2 => "İki",
    _ => "Bilinmiyor"
};
```

`_` varsayılan durumdur. Hiçbir dal eşleşmez ve `_` yoksa `SwitchExpressionException` fırlar — derleyici bunu genelde uyarı olarak da bildirir.

**`when` koşulu** ile ek şart eklenebilir:

```csharp
var mesaj = musteri switch
{
    { Tip: "VIP" } when musteri.Bakiye > 10000 => "Öncelikli",
    { Tip: "VIP" }                             => "VIP",
    _                                          => "Standart"
};
```

**Sıra önemlidir.** Dallar yukarıdan aşağı denenir; ilk eşleşen kazanır. Genel bir dalı yukarı koyarsan altındakiler ölü kod olur ve derleyici bunu bildirir:

```csharp
var etiket = tutar switch
{
    > 0     => "pozitif",
    > 1000  => "yüksek",   // buraya asla girilmez: üstteki dal zaten yakalıyor
    _       => "diğer"
};
```

Switch expression, pattern matching ile birleşince asıl gücünü gösterir:

```csharp
decimal Kargo(Siparis s) => s switch
{
    { Tutar: >= 500 }                        => 0m,
    { Adres.Sehir: "Ankara" or "İstanbul" }  => 29.90m,
    { Agirlik: > 20 }                        => 149.90m,
    null                                     => throw new ArgumentNullException(nameof(s)),
    _                                        => 49.90m
};
```

Tuple üzerinde eşleme, karar tablolarını koda birebir çevirmenin en temiz yoludur:

```csharp
string Sonuc(bool odendi, bool stokVar) => (odendi, stokVar) switch
{
    (true,  true)  => "Kargoya ver",
    (true,  false) => "Tedarik bekleniyor",
    (false, true)  => "Ödeme bekleniyor",
    (false, false) => "İptal"
};
```

Burada `_` dalı yoktur ve gerekmez: derleyici dört ihtimalin de kapatıldığını görür. Buna **exhaustiveness (bütünlük) kontrolü** denir ve switch expression'ın en değerli tarafıdır — enum'a yeni bir değer eklediğinde, kapatmadığın dal için uyarı alırsın.

> `throw`, switch expression içinde bir dal olabilir çünkü C# 7'den beri `throw` bir **ifadedir**. Aynı şekilde `=> throw new NotSupportedException()` yazımı da geçerlidir.

---

## 8. String Interpolation ve Raw String

> **Benzetme —** Matbaadan aldığın davetiye şablonu gibi: "Sayın ______, ______ tarihinde ______ salonunda..." Boşlukları doldurursun, metnin geri kalanı basılı gelir. String interpolation budur. Raw string ise fotokopi makinesidir — elindeki kâğıtta ne varsa aynen çıkar, tırnak da olsa ters bölü de olsa kimse dokunmaz.

**Basitçe:** Metnin içine değişken koymanın kısa yolu, metnin başına dolar işareti koyup değişkeni süslü parantez içinde yazmaktır. Uzun ve içinde tırnak geçen metinler için de üç tırnaklı yazım var; orada hiçbir karakteri kaçırman gerekmez.

**Teknik olarak:**

**String interpolation** — `$"..."` ile değişkenleri doğrudan metne gömme. `string.Format`'tan hem okunaklı hem (modern C#'ta) daha performanslı.

```csharp
Console.WriteLine($"{musteri.Ad} — {tutar:C2} — {tarih:dd.MM.yyyy}");
```

Formatlar: `:C` para, `:N2` iki ondalık, `:P` yüzde, `:dd.MM.yyyy` tarih.

Hizalama de verilebilir; tablo benzeri çıktılarda işe yarar:

```csharp
Console.WriteLine($"{ad,-20}{tutar,10:N2}");   // ad sola 20 karakter, tutar sağa 10 karakter
```

Süslü parantez basmak istiyorsan iki kez yazarsın:

```csharp
Console.WriteLine($"{{ {ad} }}");   // çıktı: { Umut }
```

> **Kültür tuzağı:** `$"..."` mevcut kültürü kullanır. Türkçe kültürde ondalık ayracı virgüldür. Makineye gidecek metinlerde (SQL, JSON, dosya adı) bu bozulma yaratır; orada `FormattableString.Invariant($"...")` veya `ToString(CultureInfo.InvariantCulture)` kullanılır.

```csharp
var kullaniciyaGoster = $"{1234.5:N2}";                          // 1.234,50  (tr-TR)
var makineyeGonder    = FormattableString.Invariant($"{1234.5}"); // 1234.5
```

`$` ile `@` birlikte kullanılabilir; sıra C# 8'den beri serbesttir:

```csharp
var yol = $@"C:\Kullanicilar\{kullanici}\Belgeler";
```

Loglamada interpolation kullanmak yaygın bir hatadır. Yapılandırılmış log şablonunu bozar:

```csharp
_logger.LogInformation($"Müşteri {id} bulundu");    // yanlış: metin önceden birleşir, alan kaybolur
_logger.LogInformation("Müşteri {Id} bulundu", id); // doğru: Id ayrı bir alan olarak kaydedilir
```

**Raw string literal** (C# 11) — Üç tırnak. Kaçış karakteri gerektirmez, satır sonlarını korur. JSON, SQL ve HTML yazarken çok işe yarar.

```csharp
var sorgu = """
    SELECT Id, Ad, Fiyat
    FROM   Urunler
    WHERE  Kategori = @kategori
    """;
```

Kapanış tırnaklarının girintisi, tüm bloktan kırpılacak boşluğu belirler.

İçinde tırnak geçen metinlerde kazancı belirgindir:

```csharp
// Öncesi — kaçış cehennemi
var json = "{ \"ad\": \"Umut\", \"sehir\": \"Ankara\" }";

// Sonrası
var json2 = """
    { "ad": "Umut", "sehir": "Ankara" }
    """;
```

Metnin kendisinde üç tırnak varsa, çiti dörde çıkarırsın:

```csharp
var ornek = """"
    Şu kod bloğunu anlat: """ ... """
    """";
```

Raw string ile interpolation birleşebilir. Dolar işaretinin sayısı, kaç süslü parantezin değişken sayılacağını belirler:

```csharp
var sehir = "Ankara";

var tek = $"""
    { "sehir": "{sehir}" }
    """;      // dikkat: JSON'un kendi süslü parantezleri değişken sanılır — hata

var ikili = $$"""
    { "sehir": "{{sehir}}" }
    """;      // iki dolar: değişken için iki süslü parantez gerekir, JSON'unki serbest kalır
```

---

## 9. Dosya ve Proje Yapısı Kısaltmaları

> **Benzetme —** Türkiye'de bir zarfa adres yazarken "Dünya, Avrupa, Türkiye" diye başlamazsın; bunlar zaten bellidir, yazmak yer kaybıdır. Apartman girişindeki ortak anahtar da böyledir: her daire kendi kapısı için ayrı anahtar taşır ama dış kapı için herkeste aynı anahtar vardır. Bu bölümdeki üç özellik, "zaten belli olanı yazmama" fikrinin farklı yüzleri.

**Basitçe:** Modern bir C# dosyasını açtığında eskiden gördüğün kalıp satırların çoğu yok: dosya başındaki `using` listesi yok, `Main` metodu yok, namespace'in süslü parantezleri yok. Bunlar kaldırılmadı, görünmez hâle geldi. Derleyici hepsini arkada üretiyor.

**Teknik olarak:**

**Top-level statements** (C# 9) — `Program.cs` içinde `class Program` ve `static void Main` yazmaya gerek kalmaz.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.Run();
```

Modern ASP.NET Core şablonları böyle gelir; `Main` metodunu aramak yerine dosyanın kendisini oku.

Kaybolmadığının kanıtı: `args` değişkeni hâlâ elinin altındadır ve `return` ile çıkış kodu verebilirsin.

```csharp
if (args.Length == 0)
{
    Console.Error.WriteLine("Parametre gerekli");
    return 1;                 // derleyici Main'i int döndürecek şekilde üretir
}
return 0;
```

Kuralları: projede yalnızca **tek bir dosya** top-level statement içerebilir, bu ifadeler dosyanın **başında** olmalı ve `using` satırlarından sonra gelmelidir. Aynı dosyada tip tanımı yapabilirsin ama ifadelerden **sonra** yazman gerekir.

**Global using** (C# 10) — Bir kez tanımlanan `using`, projedeki tüm dosyalarda geçerli olur.

```csharp
// GlobalUsings.cs
global using System.Linq;
global using Microsoft.EntityFrameworkCore;
```

`.csproj` içinde `<ImplicitUsings>enable</ImplicitUsings>` varsa, proje tipine göre yaygın namespace'ler zaten otomatik gelir — bu yüzden yeni projelerde dosya başında `using System;` görmezsin.

İstenmeyen bir örtük using'i `.csproj` içinden kaldırabilir, kendi listeni ekleyebilirsin:

```xml
<ItemGroup>
  <Using Include="MyApp.Domain" />
  <Using Remove="System.Net.Http" />
  <Using Include="System.Console" Static="true" />   <!-- WriteLine("...") doğrudan yazılabilir -->
</ItemGroup>
```

Takma ad (alias) da global olabilir; uzun generic tiplerde okunabilirliği kurtarır:

```csharp
global using SonucSozlugu = System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<int>>;
```

**File-scoped namespace** (C# 10) — Bir seviye girintiden kurtarır.

```csharp
namespace MyApp.Services;       // süslü parantez yok

public class MusteriService { }
```

Bir dosyada yalnızca tek bir file-scoped namespace olabilir ve klasik yazımla karıştırılamaz.

**`file` erişim belirleyicisi** (C# 11) — Bir tipi yalnızca tanımlandığı dosyaya görünür kılar. Kaynak üreticilerinin (source generator) ad çakışmasını önlemek için eklendi ama kendi yardımcı tiplerinde de işine yarar:

```csharp
file sealed class IcHesaplayici { }   // başka dosyadan erişilemez
```

---

## 10. Null İşleçleri

> **Benzetme —** Birinin ev adresini bulmaya çalışıyorsun: müşteri kaydı var mı, kaydında adres var mı, adreste sokak yazılı mı? Zincirin herhangi bir halkası boşsa devam etmenin anlamı yok — `?.` "boşsa uğraşma, eli boş dön" demenin yoludur. `??` ise "bulamazsan şu varsayılanı kullan" der. `!` ise sadece bir arkadaşının "merak etme, adresi var" demesidir: söz sözdür, kapıyı çaldığında ev yine boş olabilir.

**Basitçe:** Bu işleçler null ile uğraşmayı kısaltır. Soru işaretli nokta, sol taraf boşsa ifadeyi keser. Çift soru işareti, boş çıkarsa yedek değeri verir. Ünlem ise hiçbir şey yapmaz — sadece derleyiciyi susturur.

**Teknik olarak:**

| İşleç | Adı | Ne yapar |
|---|---|---|
| `?.` | Null-conditional | Sol taraf `null` ise ifadeyi kesip `null` döndürür |
| `?[]` | Null-conditional indeks | Aynısı indeksleyici için |
| `??` | Null-coalescing | Sol `null` ise sağdakini kullan |
| `??=` | Null-coalescing atama | Sadece `null` ise ata |
| `!` | Null-forgiving | Derleyici uyarısını susturur (çalışma anında etkisi yok) |

```csharp
int? uzunluk = musteri?.Adres?.Sokak?.Length;      // zincirin herhangi biri null ise null
string ad = musteri?.Ad ?? "Bilinmiyor";
_cache ??= new Dictionary<string, string>();
```

**Kısa devre (short-circuit) davranışı:** Zincirin ilk halkası null olduğunda **kalan halkalar hiç çalıştırılmaz**. Metot çağrısı içeriyorsa o metot da çağrılmaz.

```csharp
musteri?.Kaydet();                     // musteri null ise Kaydet hiç çağrılmaz
var ilk = liste?[0];                   // liste null ise indeksleme yapılmaz
var uzun = musteri?.Ad.Length;         // musteri null ise Ad'a da bakılmaz — sonuç int?
```

Son satırdaki dönüş tipine dikkat: `?.` kullanıldığı anda ifade nullable olur. `int` beklerken `int?` almanın sebebi budur.

**`??` zincirlenebilir** ve sağdan sola değerlendirilir:

```csharp
var deger = ayarDosyasi ?? ortamDegiskeni ?? varsayilan ?? "son çare";
```

`??` ile `throw` birleşimi, doğrulama yazmanın en kısa yoludur:

```csharp
public MusteriService(IMusteriRepository repo)
{
    _repo = repo ?? throw new ArgumentNullException(nameof(repo));
}

// C# 11 sonrası daha kısası
ArgumentNullException.ThrowIfNull(repo);
```

**`??=`** en çok tembel başlatmada (lazy initialization) kullanılır:

```csharp
private List<Urun>? _urunler;
public List<Urun> Urunler => _urunler ??= _repo.HepsiniGetir();
```

> Bu yazım **thread-safe değildir**. Aynı anda iki thread girerse liste iki kez oluşturulabilir. Çok thread'li senaryoda `Lazy<T>` kullan.

Event tetiklemede `?.Invoke(...)` neredeyse zorunlu bir kalıptır:

```csharp
public event EventHandler? Degisti;

protected void Bildir()
{
    Degisti?.Invoke(this, EventArgs.Empty);   // abone yoksa null'dır, çağırmak patlatır
}
```

> `?.` çalışma anında `null` kontrolüdür; `!` ise yalnızca derleyiciye verilen bir sözdür. İkisini karıştırma — `!` hiçbir şeyi güvenli hâle getirmez.

Bunu bir örnekle sabitle:

```csharp
string? ad = null;

Console.WriteLine(ad?.Length);   // yazdırır: (boş) — çalışır
Console.WriteLine(ad!.Length);   // NullReferenceException — uyarı susturuldu, hata kalmadı sanma
```

**Nullable value type ile nullable reference type farkı:** `int?` gerçek bir tiptir (`Nullable<int>`), bellekte bir bayrak taşır. `string?` ise derleyici için bir nottur — üretilen IL'de `string`'den farkı yoktur.

```csharp
int? x = null;
Console.WriteLine(x.HasValue);       // False   — gerçek bir üye
Console.WriteLine(x.GetValueOrDefault()); // 0
int y = x ?? -1;                     // -1

// string? üzerinde HasValue yoktur; çünkü ortada ayrı bir tip yoktur
```

---

## 11. Diğer Faydalı Özellikler

> **Benzetme —** Mutfak çekmecesindeki küçük aletler gibi: hiçbiri tek başına yemek pişirmez ama her biri bir işi on saniyede bitirir. Rendeyi bilmeyen bıçakla uğraşır, iş yine olur ama uzun sürer.

**Basitçe:** Bu bölümdekiler tek tek küçük kolaylıklar. Dizi oluşturmayı kısaltır, birden çok değer döndürmeyi mümkün kılar, dizinin sonundan eleman almayı okunur hâle getirir. Hiçbiri yeni bir kavram getirmez; hepsi var olan işi kısaltır.

**Teknik olarak:**

**Koleksiyon ifadeleri** (C# 12)
```csharp
int[] sayilar = [1, 2, 3];
List<string> adlar = ["Umut", "Ali"];
int[] birlesik = [..sayilar, 4, 5];      // spread
```

Aynı yazım pek çok koleksiyon tipinde çalışır ve hedefe göre en uygun kodu derleyici üretir:

```csharp
Span<int> s          = [1, 2, 3];
HashSet<string> kume = ["a", "b"];
List<int> bos        = [];

// İki listeyi birleştirmenin en kısa hâli
List<int> hepsi = [..birinci, ..ikinci];
```

**Tuple ve deconstruction**
```csharp
(string ad, int yas) kisi = ("Umut", 27);
var (ad, yas) = kisi;

// Birden çok değer döndürmek için
(bool basarili, string mesaj) Dogrula(...) => (true, "Tamam");
```

Kullanırken:

```csharp
var (basarili, mesaj) = Dogrula(girdi);
if (!basarili) Console.WriteLine(mesaj);

// İlgilenmediğin parçayı atabilirsin
var (_, sadeceMesaj) = Dogrula(girdi);
```

Tuple'lar değer eşitliğine sahiptir ve alan adları derleme anı bilgisidir:

```csharp
Console.WriteLine((1, "a") == (1, "a"));   // True
```

> Tuple, **iki-üç değerlik geçici** dönüşler içindir. Üçten fazla alan varsa ya da değer dışarıya (API, kütüphane sınırı) çıkıyorsa `record` yaz — isim vermek belgelemedir.

Kendi tipini deconstruct edilebilir yapmak için `Deconstruct` metodu eklersin:

```csharp
public class Nokta
{
    public int X { get; init; }
    public int Y { get; init; }
    public void Deconstruct(out int x, out int y) => (x, y) = (X, Y);
}

var (nx, ny) = new Nokta { X = 3, Y = 4 };
```

**Range ve Index** (C# 8)
```csharp
var son     = dizi[^1];      // sondan birinci
var ilkUc   = dizi[..3];     // 0,1,2
var ortadan = dizi[2..5];    // 2,3,4
```

Birkaç ek kalıp:

```csharp
var sonUc  = dizi[^3..];          // son üç eleman
var kopya  = dizi[..];            // tamamının kopyası
var aralik = 1..4;                // Range bir değerdir, değişkende tutulabilir
var dilim  = dizi[aralik];

// string üzerinde de çalışır
var uzanti = dosyaAdi[dosyaAdi.LastIndexOf('.')..];
```

> Dizide `[1..4]` **yeni bir dizi kopyalar**; `Span<T>` üzerinde ise kopya yapmadan pencere açar. Sıcak döngülerde fark önemlidir.

**Local function** — Metot içinde tanımlı yardımcı metot. Yalnızca o metotta anlamlı olan mantığı sınıf seviyesine taşımaktan kurtarır.

```csharp
public List<int> Filtrele(List<int> girdi, int esik)
{
    return girdi.Where(Uygun).ToList();

    bool Uygun(int x) => x > esik;    // esik'i kapsamdan görür, parametre geçmeye gerek yok
}
```

Lambda'ya göre iki üstünlüğü vardır: özyineleme (recursion) doğal yazılır ve `static` yaparsan yanlışlıkla dış değişken yakalamanı derleyici engeller.

```csharp
static int Faktoryel(int n) => n <= 1 ? 1 : n * Faktoryel(n - 1);   // static local function
```

**Expression-bodied member** — Tek satırlık üyeler için `=>` sözdizimi.
```csharp
public string TamAd => $"{Ad} {Soyad}";
public Task<int> SayAsync() => _repo.CountAsync();
```

Hemen her üye türünde kullanılabilir:

```csharp
public class Sepet
{
    private readonly List<Urun> _urunler = new();

    public Sepet(IEnumerable<Urun> urunler) => _urunler.AddRange(urunler);  // constructor
    public int Adet => _urunler.Count;                                      // özellik
    public Urun this[int i] => _urunler[i];                                 // indeksleyici
    public void Ekle(Urun u) => _urunler.Add(u);                            // metot
    public override string ToString() => $"{Adet} ürün";                    // override
}
```

> `=> ifade` ile `{ return ifade; }` **tamamen aynı** koda derlenir. Aralarında performans farkı yoktur; seçim okunabilirlik meselesidir. Gövde tek satıra sığmıyorsa süslü parantez kullan.

---

## Tek Bakışta Özet

- Dil sürümü hedef framework'e bağlıdır; yazmasan da **okumak** zorundasın.
- `var` dinamik değildir; tip derleme anında kesinleşir. `var` sağdan, `new()` soldan okur.
- **NRT** null hatalarını derleme anına taşır ama garanti vermez; `!` operatörünü serpiştirmek özelliği anlamsızlaştırır. Dış veriyi sınırda doğrula.
- **`record`** = değer eşitliği + `with`. DTO'lar için doğal seçim, entity'ler için değil. `with` yüzeysel kopya üretir.
- **`init` + `required`** = constructor yazmadan eksiksiz ve değişmez nesne.
- **Pattern matching** iç içe `if` yığınlarını tek ifadeye indirir; özellik deseni aynı zamanda sessiz null kontrolüdür.
- **Switch expression** değer döndürür; `_` dalını unutma, ama tuple ile tüm ihtimalleri kapattıysan gerekmez.
- **Raw string** SQL/JSON yazarken kaçış cehennemini bitirir; interpolation'da kültür tuzağına dikkat.
- Modern şablonlarda `Main`, `using` satırları ve namespace parantezleri görünmez — kaybolmadılar, örtük hâle geldiler.
- `?.` çalışma anı kontrolü, `!` sadece derleyiciye verilen söz. `?.` kullandığın anda sonuç nullable olur.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Type inference | Tipin derleyici tarafından çıkarılması (`var`) |
| Target-typed new | Sol taraftan tip çıkarımıyla `new()` yazımı |
| nameof | Üye adını string olarak veren derleme anı işleci |
| NRT | Nullable reference types — null analizini derlemeye taşıyan özellik |
| Flow analysis | Derleyicinin, o satıra kadarki kontrollere bakarak null durumunu izlemesi |
| Null durum özniteliği | `[NotNullWhen]` gibi, metodun null sözleşmesini derleyiciye bildiren işaret |
| Null-forgiving (`!`) | Null uyarısını bastıran, çalışma anı etkisi olmayan işleç |
| record | Değer eşitliğine sahip veri tipi |
| with expression | Kopya üretip belirli alanları değiştirme (yüzeysel kopya) |
| record struct | Değer eşitlikli value type |
| init | Yalnızca nesne oluşturulurken atanabilen özellik |
| required | Nesne oluşturulurken atanması zorunlu üye |
| Primary constructor | Sınıf tanımının yanında parametre bildirimi |
| Pattern matching | Tip/şekil/içerik eşlemesi ve parça yakalama |
| Özellik deseni | Nesnenin alanlarına göre eşleme (`{ Tutar: > 100 }`) |
| Liste deseni | Dizi/liste şekline göre eşleme (`[1, 2, ..]`) |
| Switch expression | Değer döndüren switch |
| Exhaustiveness | Tüm ihtimallerin kapatıldığının derleyici tarafından denetlenmesi |
| Raw string literal | Üç tırnaklı, kaçışsız çok satırlı metin |
| Top-level statements | `Main` metodu yazmadan program gövdesi |
| Global using | Tüm dosyalarda geçerli using bildirimi |
| Implicit usings | Proje tipine göre otomatik eklenen using kümesi |
| File-scoped namespace | Süslü parantezsiz, dosyanın tamamını kapsayan namespace |
| Koleksiyon ifadesi | `[1, 2, 3]` yazımıyla koleksiyon oluşturma |
| Spread (`..`) | Bir koleksiyonun elemanlarını başka bir koleksiyona açma |
| Deconstruction | Bir nesneyi parçalarına ayırarak değişkenlere atama |
| Range / Index | `[2..5]` ve `[^1]` ile dilim ve sondan erişim |
| Local function | Metot içinde tanımlı yardımcı metot |
| Expression-bodied member | `=>` ile tek satırlık üye tanımı |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`var` dinamik tiptir" | Statik tiptir; `dynamic` ile karıştırma |
| "NRT null hatasını tamamen bitirir" | Derleme anı analizidir; dış veriler yine null getirebilir |
| "`record` her yerde `class`'tan iyidir" | Kimliği ve değişen durumu olan entity'ler için `class` doğru |
| "`record` tamamen değişmezdir" | `with` yüzeysel kopyalar; içindeki liste paylaşılır |
| "`init` ile `readonly` aynı şey" | `init` nesne başlatıcıyla çalışır, `readonly` constructor'la |
| "Primary constructor parametreleri readonly alandır" | Değildir; metot içinden değiştirilebilir |
| "`?.` ile `!` benzer işler yapar" | `?.` çalışma anında korur, `!` sadece uyarıyı susturur |
| "`x?.Length` bir `int` döner" | `int?` döner — `?.` kullanıldığı anda ifade nullable olur |
| "`is null` ile `== null` aynı" | `==` aşırı yüklenebilir, `is null` yüklenemez |
| "`??=` thread-safe tembel başlatmadır" | Değildir; iki thread aynı anda başlatabilir, `Lazy<T>` kullan |
| "Top-level statements ile `Main` kaldırıldı" | Derleyici hâlâ üretiyor, sadece sen yazmıyorsun |
| "İnterpolasyon her yerde güvenle kullanılır" | Kültüre bağlıdır; log şablonlarını ve makine formatlarını bozar |

---

## Sonraki

→ `06-Dil-Altyapisi.md` (Cumartesi)
