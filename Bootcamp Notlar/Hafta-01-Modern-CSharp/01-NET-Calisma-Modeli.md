# Hafta 1 · Pazartesi — .NET Çalışma Modeli, Bellek ve Tip Sistemi

**Okuma süresi:** ~45 dk
**Neden bu konu:** Bootcamp boyunca karşılaşacağın "bu nesne neden değişti?", "neden `NullReferenceException` aldım?", "uygulama neden bellek şişiriyor?" sorularının cevabı burada. Bu temel olmadan ileriki konular ezber kalır.

---

## Önce Basitçe

Bir program yazdığında aslında iki ayrı dünya vardır. Birincisi senin yazdığın metin: harfler, satırlar, süslü parantezler. İkincisi ise o metnin bilgisayarda gerçekten koşan hâli. Bu ikisi aynı şey değildir. Aradaki mesafeyi kim kapatıyor, bu notun büyük kısmı onu anlatıyor.

.NET dediğimiz şey tek bir program değil. Bir düzenek. İçinde senin yazdığını makinenin anlayacağı hâle çeviren bir tercüman var, programın çalışırken oturduğu bir bina var, o binanın bakımını yapan bir görevli var, bir de "sen zaten kurmuşsun, ben hazır kullanayım" diyebileceğin devasa bir hazır malzeme deposu var. C# ise bu binanın içinde konuştuğun dil. Dili bilmek yetmez; binanın nasıl işlediğini de bilmek gerekir, çünkü kodunun neden şöyle davrandığının cevabı çoğu zaman dilde değil, binada saklıdır.

Programın çalışırken kullandığı hafıza da tek parça değil. İki ayrı yer var: biri küçük, hızlı ve düzenli; öteki büyük, esnek ve zamanla dağınıklaşan. Bir değişkeni başka bir değişkene atadığında ne olduğu tamamen o verinin bu iki yerden hangisinde durduğuna bağlı. Kimi veri kopyalanır, kimi veri paylaşılır. Yeni başlayanların "ben bunu değiştirmemiştim ki" dediği hataların neredeyse hepsi bu tek ayrımdan çıkar.

Dağınıklaşan tarafı da birinin toplaması gerekir. .NET'te bunu sen elinle yapmazsın; arka planda çalışan bir temizlik mekanizması, artık kimsenin kullanmadığı şeyleri bulup atar. Bu çok rahatlatıcıdır ama sihirli değildir. Temizlikçinin göremediği şeyler vardır (dışarıdan açtığın dosya, veritabanı bağlantısı gibi) ve senin "bu lazım olur" diye tuttuğun şeyleri de asla atmaz. Bellek sızıntısı dediğimiz şey, temizlikçinin tembelliği değil, senin bırakmamandır.

Bunların hepsi birbirine bağlı. Derleme zinciri belleği belirler, bellek modeli tip davranışını belirler, tip davranışı da hangi hataları alacağını belirler. Şimdi detaya iniyoruz — her başlıkta önce günlük dille, sonra teknik tanımıyla.

> **Ana benzetme:** .NET'i iyi yönetilen bir apartman gibi düşün. CLR apartman yöneticisidir; kimin nereye yerleşeceğine, kuralların çiğnenip çiğnenmediğine o bakar. Stack dairenin içindeki masa üstüdür — küçük, düzenli, akşam toplanır. Heap ise bodrumdaki ortak depodur: geniştir, herkes bir şey bırakır, zamanla dağılır. GC apartman görevlisidir, sahipsiz kalanı toplar. `IDisposable` ise dışarıdan kiraladığın eşyadır: onu apartman görevlisi değil, sen iade etmek zorundasın.

---

## Bu Notta Ne Var

1. .NET'in ne olduğu ve sürüm karmaşası
2. Kaynak kodun çalışan programa dönüşme zinciri
3. CLR, yönetilen kod ve tip sistemi
4. Bellek: stack, heap, LOH
5. Value type ve reference type ayrımı — C#'ın en kritik ayrımı
6. `string`'in özel durumu ve immutability
7. Boxing / unboxing
8. Garbage Collector'ın çalışma mantığı
9. `IDisposable` ve deterministik temizlik

---

## 1. .NET Nedir

> **Benzetme —** .NET bir yemek tarifi değil, donanımlı bir mutfaktır. İçinde ocak var (runtime), kiler dolusu hazır malzeme var (BCL), bir de tarifleri pişirilebilir hâle getiren usta var (derleyici). C#, bu mutfakta konuştuğun dildir. Aynı mutfakta F# ya da VB.NET konuşan biri de yemek yapabilir; ocak hepsine aynı şekilde çalışır.

**Basitçe:** .NET bir programlama dili değil. Uygulamanı yazman, derlemen ve çalıştırman için gereken her şeyi bir arada veren paketin adı. C# ise o paketin içinde kullandığın dillerden biri. "C# öğreniyorum" demek, aslında hem dili hem de bu paketi öğreniyorum demektir.

**Teknik olarak:** **.NET** — Bir dil değil, bir **geliştirme platformudur**. İçinde şunlar var: çalışma zamanı ortamı (runtime), sınıf kütüphaneleri (BCL), derleyiciler ve araçlar. C#, F# ve VB.NET bu platform üzerinde çalışan dillerdir.

### Sürüm karmaşası — bir kez netleştir

İsimlendirme tarih boyunca iki kez değiştiği için karışık görünür. Bir kez oturt, bir daha düşünme.

| İsim | Yıllar | Durum |
|---|---|---|
| **.NET Framework** (1.0 – 4.8) | 2002–2019 | Sadece Windows. Artık yeni özellik almıyor, sadece güvenlik güncellemesi. Eski kurumsal projelerde hâlâ karşına çıkar |
| **.NET Core** (1.0 – 3.1) | 2016–2019 | Çapraz platform yeniden yazım. İsim artık kullanılmıyor |
| **.NET 5, 6, 7, 8, 9, 10...** | 2020– | Framework ile Core'un birleşimi. "Core" kelimesi düştü. Bugün "**.NET**" denince bu kastedilir |

> `ASP.NET Core` isminde "Core" kelimesinin kalmasının sebebi tarihsel: eski `ASP.NET` (Framework üzerindeki) ile karışmasın diye. Yani ".NET 9 üzerinde ASP.NET Core" ifadesi tuhaf görünse de doğrudur.

**LTS (Long Term Support)** — Çift numaralı sürümler (8, 10...) 3 yıl destek alır. Tek numaralılar (9, 11...) 18 ay. Kurumsal projeler LTS seçer.

Pratikte bu seni şurada ilgilendirir: bir projeye başlarken hedef framework'ü (`<TargetFramework>net10.0</TargetFramework>`) LTS bir sürüme sabitlersen, on sekiz ay sonra "destek bitti" diye acil yükseltme yapmak zorunda kalmazsın.

```xml
<!-- .csproj dosyasının en kritik satırı -->
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

### BCL nedir

Kiler benzetmesinin karşılığı budur: her seferinde sıfırdan liste, dosya okuma, tarih hesabı yazmazsın; hazırı vardır.

**BCL (Base Class Library)** — `System.Collections`, `System.IO`, `System.Linq` gibi, her .NET uygulamasının kullanabildiği hazır tipler kümesi. "Kütüphane kurmadan elimde ne var" sorusunun cevabı.

---

## 2. Kaynak Koddan Çalışan Programa

> **Benzetme —** Elinde Türkçe yazılmış bir yemek tarifi var ve bunu dünyanın her mutfağında uygulatmak istiyorsun. İki adımda yaparsın. Önce tarifi herkesin anladığı ortak bir mutfak diline çevirirsin — artık hangi ülkede olduğunu bilmene gerek yok. Sonra tarif fiilen pişirileceği mutfakta, o mutfağın ocağına göre uyarlanır: gazlı ocak başka, indüksiyon başka. Birinci çeviri bir kez yapılır ve dosyada durur. İkinci uyarlama ise yemek ilk kez pişirileceği an yapılır.

**Basitçe:** Yazdığın C# dosyası doğrudan çalışmaz. Önce ara bir dile çevrilir; bu çeviri, hangi bilgisayarda çalışacağını henüz bilmeyen tarafsız bir hâldir. Sonra program çalışırken, her metot ilk kez kullanıldığı anda o ara dil gerçek makine koduna dönüştürülür. Yani derleme tek seferde bitmez, ikiye bölünür: biri sen `build` derken, öteki program koşarken.

**Teknik olarak:**

```
   C# kaynak kodu (.cs)
          │
          │  Roslyn derleyicisi  (dotnet build)
          ▼
   IL + Metadata  →  paketlenir  →  .dll / .exe   (assembly)
          │
          │  CLR assembly'yi yükler
          ▼
   JIT (Just-In-Time) derleyici — metot ilk çağrıldığında
          ▼
   Makine kodu (x64 / ARM)  →  CPU
```

**IL (Intermediate Language)** — Ara dil. CPU'ya özgü değildir; "hangi işlemcide çalışacağı henüz belli olmayan" makine kodu gibi düşün. `MSIL` veya `CIL` olarak da geçer.

**Assembly** — Derleme çıktısı olan `.dll` veya `.exe`. İçinde IL kodu + **metadata** (tiplerin, metotların, parametrelerin tanımı) + manifest (sürüm, bağımlılıklar) bulunur.

**Metadata** neden önemli: Reflection, IntelliSense, serileştirme, DI konteyneri ve EF Core'un tamamı bu metadata'yı okuyarak çalışır. C#'ta "çalışma anında tipin ne olduğunu sorabilmen" bu sayededir.

```csharp
// Metadata'nın çalışma anında okunabilmesi — reflection
var tip = typeof(Musteri);
Console.WriteLine(tip.FullName);                  // Namespace.Musteri
foreach (var p in tip.GetProperties())
    Console.WriteLine($"{p.PropertyType.Name} {p.Name}");
// Bu bilgi .dll içinde yazılı durur; derleyici oraya koymuştur.
```

**JIT (Just-In-Time)** — IL'i makine koduna, **metot ilk kez çağrıldığı anda** çeviren derleyici. Sonuç önbelleğe alınır, ikinci çağrı doğrudan makine kodunu kullanır.

Bundan çıkan üç pratik sonuç:

1. **İlk istek yavaştır.** Web uygulaman ilk açılışta neden gecikiyor sorusunun cevabı budur (buna *cold start* denir). Çözümleri: **ReadyToRun** (derleme anında kısmi makine kodu üretimi) ve **Native AOT** (tamamen önceden derleme).
2. **JIT çalıştığı donanımı bilir.** Sunucunun desteklediği CPU komut setlerine göre optimize edebilir — statik derlemenin yapamadığı bir şey.
3. **IL geri okunabilir.** Bir `.dll`'i ILSpy/dnSpy ile açıp neredeyse kaynak koda çevirebilirsin. Bu yüzden API anahtarı, şifre gibi bilgiler koda gömülmez.

> **Bu benzetme şurada bozulur:** Tarif örneğinde çeviri bir kez yapılıp bitiyor gibi durur. JIT ise sadece ilk çağrıda değil, sonrasında da devrede kalabilir: **tiered compilation** ile metot önce hızlı ve kaba derlenir, çok çağrıldığı görülürse arka planda yeniden, daha iyi optimize edilerek derlenir. Yani aynı metot programın ömrü boyunca iki farklı makine koduna sahip olabilir.

Üçüncü maddenin pratik karşılığı şudur: bağlantı cümlesini (connection string) koda gömmek, kapıyı kilitleyip anahtarı kapı önündeki paspasın altına koymaktır.

```csharp
// Yanlış — derlenmiş .dll içinden okunabilir
var cs = "Server=.;Database=Shop;User Id=sa;Password=P@ssw0rd!";

// Doğru — yapılandırmadan gelir, dosya deploy ortamında durur
var cs = builder.Configuration.GetConnectionString("Default");
```

---

## 3. CLR ve Yönetilen Kod

> **Benzetme —** CLR apartmanın yöneticisidir. Yeni taşınan olduğunda daireyi o gösterir (bellek ayırma), boşalan daireleri o takip eder (GC), kavga çıkarsa o müdahale eder (exception), kimin hangi daireye girebileceğini o denetler (tip güvenliği), dışarıdan gelen usta binaya girecekse ona da o izin verir (assembly yükleme). Sen sadece oturursun.

**Basitçe:** Kodunun altında, sen farkında olmadan çalışan bir gözetmen var. Bellek ayırmayı, hata yönetimini, tiplerin doğru kullanıldığını o üstleniyor. C/C++ yazan biri bunları elle yapar; sen yapmıyorsun, işte bu yüzden koduna "yönetilen kod" deniyor.

**Teknik olarak:** **CLR (Common Language Runtime)** — .NET'in çalışma zamanı motoru. Sorumlulukları:

| Sorumluluk | Ne yapar |
|---|---|
| Bellek yönetimi | Nesne ayırma ve Garbage Collection |
| Tip güvenliği | Bir `int`'e `string` yazmanı çalışma anında da engeller |
| Exception yönetimi | `try/catch/finally` altyapısı |
| Thread yönetimi | İş parçacıkları ve thread pool |
| Güvenlik | Kod erişim kontrolleri |
| Assembly yükleme | Bağımlılıkların bulunması ve yüklenmesi |

**Yönetilen kod (managed code)** — CLR'ın gözetiminde çalışan kod. Belleği sen ayırmaz, sen serbest bırakmazsın.
**Yönetilmeyen kod (unmanaged code)** — İşletim sistemi API'leri, C/C++ kütüphaneleri, veritabanı sürücülerinin alt katmanları. Bunların açtığı kaynakları GC göremez — `IDisposable` konusunun asıl sebebi budur (bkz. bölüm 9).

Yönetilmeyen tarafa geçişin somut hâli şöyle görünür:

```csharp
using System.Runtime.InteropServices;

// Windows API'sine doğrudan çağrı — bu satırdan sonrası CLR'ın kontrolü dışında
[DllImport("user32.dll")]
static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
```

### CTS ve CLS

**CTS (Common Type System)** — Tüm .NET dillerinin paylaştığı ortak tip tanımı. C#'taki `int`, aslında `System.Int32`'dir; VB.NET'teki `Integer` de aynı tiptir. Diller arası uyumluluk buradan gelir.

```csharp
int a = 5;
System.Int32 b = 5;
Console.WriteLine(a.GetType() == b.GetType());   // True — ikisi aynı tip
// "int" sadece bir takma addır (alias), ayrı bir tip değildir.
```

**CLS (Common Language Specification)** — Tüm .NET dillerinin desteklemesi gereken asgari kurallar kümesi. Örneğin C# büyük/küçük harf duyarlıdır ama VB.NET değildir; bu yüzden yalnızca harf durumuyla ayrılan iki public üye CLS uyumlu değildir.

```csharp
public class Rapor
{
    public void Yazdir() { }
    public void yazdir() { }   // C# derler; CLS uyumlu değildir,
}                              // VB.NET'ten bu sınıf kullanılamaz.
```

---

## 4. Bellek: Stack ve Heap

> **Benzetme —** Bir markette çalıştığını düşün. Kasanın yanındaki dar tezgâh senin stack'indir: üstüne sadece o anki müşterinin ürünlerini koyarsın, iş bitince hepsini bir hamlede süpürürsün, her şey elinin altındadır. Arkadaki depo ise heap'tir: kocamandır, her şey oraya konur, kim ne bıraktı zamanla karışır, arada bir birinin gelip toplaması gerekir. Tezgâhta yer bulmak bir saniye sürer; depoda kutuyu bulmak dakikalar.

**Basitçe:** Programın kullandığı bellek iki bölgeye ayrılıyor. Biri küçük, çok hızlı ve kendi kendini toplayan bölge — metot bitince oradaki her şey otomatik siliniyor. Öteki büyük, esnek ve elle toplanmayan bölge — orayı arka plandaki temizlikçi topluyor. Bir verinin hangi bölgede durduğu, o veriyle çalışırken hızını da, hayatta kalma süresini de belirliyor.

**Teknik olarak:**

|  | **Stack** | **Heap** |
|---|---|---|
| Ne tutar | Metot çerçeveleri, yerel değişkenler, value type'lar, referansların kendisi | Nesnelerin gövdesi (reference type örnekleri) |
| Boyut | Küçük ve sabit — thread başına ~1 MB | Büyük, dinamik olarak büyür |
| Yapı | LIFO (son giren ilk çıkar) | Serbest, parçalanabilir |
| Temizlik | Metot bitince **anında ve otomatik** | **GC** tarafından, zamanı belirsiz |
| Hız | Çok hızlı (tek bir pointer kaydırma) | Daha yavaş (yer bulma + toplama maliyeti) |
| Kapsam | Thread'e özel | Uygulama genelinde paylaşılan |

**Stack frame** — Bir metot çağrıldığında stack'e eklenen blok: parametreler, yerel değişkenler ve dönüş adresi. Metot bitince blok atılır.

```csharp
void Disaridaki()
{
    int x = 10;                 // stack
    var k = new Kisi();         // 'k' referansı stack'te, Kisi nesnesi heap'te
    Icerideki(x);               // yeni bir stack frame eklenir
}                               // metot biter -> frame atılır, x yok olur.
                                // Heap'teki Kisi nesnesi ise hâlâ orada;
                                // ona ulaşan referans kalmadığı için çöp sayılır.
```

Buradan iki tanıdık hata çıkar:
- **`StackOverflowException`** — Sonsuz özyineleme (recursion) stack'i doldurur. Yakalanamaz, süreç anında sonlanır; çünkü exception'ı işlemek için de stack alanı gerekir.
- **`OutOfMemoryException`** — Heap'te yer kalmaz veya çok büyük bir nesne ayrılamaz.

```csharp
int Faktoriyel(int n) => n * Faktoriyel(n - 1);   // taban durum yok
Faktoriyel(5);   // StackOverflowException — try/catch bunu YAKALAYAMAZ
```

**LOH (Large Object Heap)** — 85.000 byte'tan büyük nesneler (büyük diziler, dosya buffer'ları) normal heap yerine buraya konur. Varsayılan olarak **sıkıştırılmaz**, bu yüzden zamanla parçalanır. Büyük dizileri sürekli oluşturup atmak yerine havuzlamak (`ArrayPool`) bu yüzden önerilir.

```csharp
// Her çağrıda LOH'a 100 KB'lık yeni nesne — parçalanma kaynağı
byte[] buffer = new byte[100_000];

// Havuzdan ödünç al, işin bitince geri ver
var havuz = System.Buffers.ArrayPool<byte>.Shared;
byte[] odunc = havuz.Rent(100_000);
try   { /* odunc ile çalış */ }
finally { havuz.Return(odunc); }
```

> **Bu benzetme şurada bozulur:** Tezgâh–depo ayrımı "küçük şeyler tezgâhta, büyük şeyler depoda" gibi bir izlenim verir. Doğru ölçü boyut değil, **ömür ve sahiplik**. Küçücük bir `int` bile bir sınıfın alanıysa heap'te durur; koca bir `struct` yerel değişkense stack'te durur. Belirleyici olan verinin büyüklüğü değil, nereye ait olduğudur.

---

## 5. Value Type ve Reference Type

> **Benzetme —** Arkadaşın senden yemek tarifini istiyor. İki seçeneğin var. Ya tarifin fotokopisini çekip verirsin — o kendi kâğıdına istediğini yazar, senin defterin aynı kalır. Ya da "defter mutfaktaki dolapta, anahtar da şurada" dersin — o anahtarla defterini açar ve içine yazarsa, senin defterin de değişmiş olur. Value type fotokopidir. Reference type anahtardır.

**Basitçe:** Bir değişkeni başka bir değişkene atadığında iki şeyden biri olur. Ya verinin bir kopyası çıkarılır ve artık iki bağımsız veri olur, ya da aynı verinin adresi verilir ve iki isim tek şeyi gösterir. Hangisinin olacağı, tipin türüne bağlıdır ve bunu ezberlemek zorundasın; çünkü C#'ta seni en çok şaşırtacak davranışların kaynağı budur.

**Teknik olarak:** C#'ın en kritik ayrımı. Tek cümlelik tanım:

> **Value type atandığında verinin kendisi kopyalanır; reference type atandığında sadece adresi kopyalanır.**

| | Value type | Reference type |
|---|---|---|
| Anahtar kelime | `struct`, `enum` | `class`, `interface`, `delegate`, `record` (varsayılan) |
| Örnekler | `int`, `double`, `bool`, `char`, `decimal`, `DateTime`, `Guid`, `TimeSpan` | `string`, `object`, diziler, `List<T>`, senin sınıfların |
| Nerede durur | Yerel değişkense stack'te; bir sınıfın alanıysa o nesneyle birlikte heap'te | Gövdesi hep heap'te, referansı stack'te |
| Atama davranışı | Tam kopya | Adres kopyası — iki değişken aynı nesneyi gösterir |
| Varsayılan değer | `0`, `false`, `default(T)` | `null` |
| `null` olabilir mi | Hayır (`int?` gibi nullable hâli hariç) | Evet |
| Eşitlik varsayılanı | Alan alan karşılaştırma | Referans karşılaştırması (aynı nesne mi) |

```csharp
// Value type — kopyalanır
int a = 5;
int b = a;
b = 10;
Console.WriteLine(a);        // 5   — a etkilenmedi

// Reference type — adres kopyalanır
var kisi1 = new Kisi { Ad = "Umut" };
var kisi2 = kisi1;           // aynı nesneyi işaret ediyor
kisi2.Ad = "Ali";
Console.WriteLine(kisi1.Ad); // "Ali"  — tek nesne var, iki isim
```

Aynı ayrım koleksiyonlarda da aynen geçerlidir ve orada daha sinsidir:

```csharp
var liste1 = new List<int> { 1, 2, 3 };
var liste2 = liste1;          // List bir class -> adres kopyalandı
liste2.Add(4);
Console.WriteLine(liste1.Count);   // 4  — tek liste var

// Gerçekten ayrı liste istiyorsan kopyasını çıkar
var liste3 = new List<int>(liste1);
liste3.Add(5);
Console.WriteLine(liste1.Count);   // 4  — bu sefer etkilenmedi
```

### Metoda parametre geçerken

```csharp
void DegistirVeri(Kisi k)     { k.Ad = "Yeni"; }        // dışarıdaki nesne DEĞİŞİR
void DegistirNesne(Kisi k)    { k = new Kisi(); }       // dışarısı ETKİLENMEZ
void DegistirRef(ref Kisi k)  { k = new Kisi(); }       // dışarıdaki değişken DEĞİŞİR
```

Ayrımı şöyle tut: **referansın kendisi de değer olarak geçer.** Yani nesnenin *içini* değiştirirsen dışarısı görür; değişkene *yeni bir nesne atarsan* görmez. `ref` bu ikinci durumu da mümkün kılar.

Benzetmeye dönersek: anahtarı verdiğinde arkadaşın dolaptaki deftere yazabilir (içi değişir, sen görürsün). Ama arkadaşın kendi cebine başka bir dolabın anahtarını koyarsa, senin dolabın yerinde durmaya devam eder. `ref`, ona senin cebine de uzanma izni vermektir.

> Bu, bootcamp'te entity ve DTO karıştırıldığında ortaya çıkan en sinsi hata kaynağıdır. Bir entity'yi servis katmanına gönderip "değiştirmez herhâlde" varsaymak, EF Core'un change tracking'i ile birleşince beklenmedik `UPDATE` sorgularına dönüşür.

> **Bu benzetme şurada bozulur:** Fotokopi örneği "value type kopyalandığı için her zaman güvenlidir" izlenimi verir. Ama kopyalama **yüzeyseldir**: bir `struct` içinde bir `class` alanı varsa, struct kopyalandığında o alanın sadece adresi kopyalanır. Yani value type'ın içinden reference type sızabilir. Bu yüzden `struct`'ların immutable ve sade tutulması önerilir.

### `struct` ne zaman kullanılır

Şartların **hepsi** sağlanıyorsa: küçük (~16 byte altı), mantıksal olarak tek bir değeri temsil ediyor, immutable, ve çok sayıda üretiliyor. Örnek: koordinat, para birimi, tarih aralığı.

Emin değilsen `class` kullan. Büyük bir `struct`'ı sürekli kopyalamanın maliyeti, heap ayırmadan kazandığından fazla olur.

```csharp
// Uygun bir struct: küçük, tek bir değeri temsil ediyor, immutable
public readonly struct Koordinat
{
    public double Enlem  { get; }
    public double Boylam { get; }
    public Koordinat(double enlem, double boylam) => (Enlem, Boylam) = (enlem, boylam);
}
```

`readonly struct` yazmak, derleyiciye "bu tip değişmez" demektir; gereksiz savunma kopyalarını (defensive copy) engeller.

### `record` nedir

**`record`** — C# 9 ile gelen, **değer eşitliğine** sahip reference type. Normal bir `class`'ta iki farklı nesne asla eşit sayılmazken, `record`'da tüm alanları aynıysa eşit sayılır. DTO'lar için doğal seçimdir.

```csharp
public record Musteri(string Ad, string Email);

var m1 = new Musteri("Umut", "u@x.com");
var m2 = new Musteri("Umut", "u@x.com");
Console.WriteLine(m1 == m2);   // True   — class olsaydı False olurdu
```

`record struct` ise value type olan hâlidir.

`record`'un ikinci faydası `with` ifadesidir: mevcut nesneyi bozmadan, tek alanı değişmiş bir kopyasını üretir.

```csharp
var guncel = m1 with { Email = "yeni@x.com" };
Console.WriteLine(m1.Email);      // u@x.com    — orijinal bozulmadı
Console.WriteLine(guncel.Email);  // yeni@x.com
```

---

## 6. `string` — Reference Type Ama Değer Gibi Davranır

> **Benzetme —** `string`, matbaada bastırdığın bir tabela gibidir. Tabelayı astıktan sonra üzerindeki yazıyı düzeltemezsin; "şu harfi değiştireyim" dersen matbaa yeni bir tabela basar, sen de eskisini indirip yenisini asarsın. Dışarıdan bakan "tabela değişti" sanır; oysa o tabela değişmedi, başka bir tabela geldi. Eski tabela da çöpe gider.

**Basitçe:** `string` bir sınıftır, yani normalde paylaşılması gerekir. Ama içeriği asla değiştirilemediği için, paylaşılması hiç fark etmez. Bir metne ekleme yaptığında eski metin olduğu gibi kalır, yepyeni bir metin doğar. Bu yüzden `string` sana kopyalanıyormuş gibi davranır — aslında kopyalanmıyor, sadece hiç değişmiyor.

**Teknik olarak:** `string` bir sınıftır, yani reference type'tır. Buna rağmen:

```csharp
string s1 = "merhaba";
string s2 = s1;
s2 = "günaydın";
Console.WriteLine(s1);   // "merhaba"  — beklenenin aksine değişmedi
```

Sebep: **`string` immutable'dır (değiştirilemez).** Bir string'in içeriği oluşturulduktan sonra asla değişmez. "Değiştirdiğin" her an aslında **yeni bir string nesnesi** doğar ve değişkenin ona bakmaya başlar.

Bunun en net kanıtı, string metotlarının hiçbirinin yerinde değişiklik yapmamasıdır:

```csharp
string ad = "  umut  ";
ad.Trim();                      // hiçbir işe yaramaz — sonuç atılır
Console.WriteLine($"[{ad}]");   // [  umut  ]

ad = ad.Trim();                 // doğrusu: dönen yeni string'i geri ata
Console.WriteLine($"[{ad}]");   // [umut]
```

**Immutability neden tercih edildi:** thread güvenliği (paylaşılan string'i kimse bozamaz), hash'lerin sabit kalması (`Dictionary` anahtarı olabilmesi) ve string interning'in mümkün olması.

**String interning** — Kod içinde birebir aynı string literal'i birden çok kez yazarsan CLR bunları tek bir nesnede toplar. Bellek tasarrufudur ama sadece derleme anında bilinen sabitler için geçerlidir.

```csharp
string a = "merhaba";
string b = "merhaba";
Console.WriteLine(ReferenceEquals(a, b));            // True  — interning

string c = new string(new[] {'m','e','r','h','a','b','a'});
Console.WriteLine(ReferenceEquals(a, c));            // False — çalışma anında üretildi
Console.WriteLine(a == c);                           // True  — == içeriği karşılaştırır
```

Son satır önemli: `string` için `==` operatörü özel olarak **içerik** karşılaştırmasına çevrilmiştir. Diğer reference type'larda `==` referans karşılaştırır.

### Pratik sonucu

```csharp
// Döngüde 10.000 ayrı string nesnesi üretir — her biri çöp olur
string sonuc = "";
for (int i = 0; i < 10_000; i++) sonuc += i;

// Tek bir değiştirilebilir buffer kullanır
var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) sb.Append(i);
string sonuc2 = sb.ToString();
```

**Kural:** Birkaç parçayı birleştiriyorsan `+` veya string interpolation (`$"..."`) yeterlidir; **döngü içindeyse** `StringBuilder`.

Neden birkaç parçada `+` yeterli: derleyici arka planda `string.Concat` çağrısına çevirir ve tek seferde tek nesne üretir. Döngüde ise her tur ayrı bir çağrıdır, derleyici onları birleştiremez.

> **Bu benzetme şurada bozulur:** Tabela örneği, her metin işleminin mutlaka yeni nesne doğurduğunu düşündürür. Modern C#'ta metni **okurken** kopya çıkarmadan çalışman mümkündür: `ReadOnlySpan<char>` ve `string.AsSpan()` mevcut metnin bir penceresini verir, yeni nesne üretmez. Yani "her dokunuş yeni tabela" kuralı yazma için geçerlidir, okuma için değil.

```csharp
string yol = "raporlar/2026/ocak.csv";
ReadOnlySpan<char> uzanti = yol.AsSpan(yol.LastIndexOf('.') + 1);   // ayırma yok
Console.WriteLine(uzanti.SequenceEqual("csv"));   // True
```

---

## 7. Boxing ve Unboxing

> **Benzetme —** Elinde tek bir vida var ve bunu kargoyla göndermen gerekiyor. Vidayı öylece kargoya veremezsin; bir koli bulup içine koyarsın, bant çekersin, etiket yapıştırırsın. Karşı taraf koliyi açıp vidayı çıkarır. Vida hiç değişmedi ama sen koli, bant ve emek harcadın. Bunu tek bir vida için yaparsan sorun değil; on bin vidanın her birini ayrı koliyle göndermeye kalkarsan kargo masrafı vidanın değerini geçer.

**Basitçe:** Sayı, tarih gibi küçük değerler normalde hızlı bölgede durur. Ama onları "her şeyi kabul eden" bir yere koyman gerektiğinde, .NET onları büyük bölgeye taşımak için etraflarına bir sarmalayıcı örer. Bu taşıma bedava değildir. Tek seferlik önemsizdir; döngü içinde on binlerce kez olursa program yavaşlar.

**Teknik olarak:** **Boxing** — Bir value type'ın heap'e taşınıp reference type gibi davranması. `object` veya bir interface'e atandığında olur.
**Unboxing** — Kutudan geri çıkarma. Tip kontrolü + kopyalama içerir.

```csharp
int sayi = 42;
object kutu = sayi;      // boxing:   heap'te yeni nesne + kopyalama
int geri = (int)kutu;    // unboxing: tip kontrolü + kopyalama
```

Kutunun gerçekten bir kopya taşıdığını görmek için:

```csharp
int x = 1;
object kutulu = x;   // kopyalandı
x = 99;
Console.WriteLine(kutulu);   // 1  — kutudaki değer x'ten bağımsız

// Yanlış tiple açmak çalışma anında patlar
object o = 42;
long y = (long)o;            // InvalidCastException — kutuda int var, long değil
long z = (long)(int)o;       // doğrusu: önce doğru tiple aç, sonra dönüştür
```

İkisi de maliyetlidir çünkü heap ayırma ve GC baskısı üretir. Görünmez şekilde gerçekleştiği yerlere dikkat:

| Durum | Boxing olur mu |
|---|---|
| `ArrayList`e `int` eklemek | Evet, her eleman için |
| `List<int>`e `int` eklemek | Hayır — generic sayesinde |
| `object.Equals` çağırmak | Genellikle evet |
| `string.Format("{0}", 5)` | Evet |
| `$"{5}"` (interpolation) | Modern C#'ta çoğunlukla hayır |

**Generic'lerin asıl sebebi budur.** `List<int>` sadece tip güvenliği için değil, **boxing'i ortadan kaldırdığı için** `ArrayList`ten üstündür.

```csharp
// Her Add çağrısında bir boxing — 1 milyon eleman, 1 milyon heap nesnesi
var eski = new System.Collections.ArrayList();
for (int i = 0; i < 1_000_000; i++) eski.Add(i);

// Hiç boxing yok — int'ler doğrudan iç dizide durur
var yeni = new List<int>(capacity: 1_000_000);
for (int i = 0; i < 1_000_000; i++) yeni.Add(i);
```

Gizli boxing'in bootcamp projelerinde en sık görülen hâli, `enum`'ların sözlük anahtarı ya da log parametresi olarak kullanılmasıdır:

```csharp
enum Durum { Bekliyor, Onaylandi }

// Durum bir value type; object parametresine girerken kutulanır
void Logla(string mesaj, object deger) { }
Logla("durum", Durum.Onaylandi);          // boxing

// Generic karşılığında kutulanmaz
void Logla<T>(string mesaj, T deger) { }
Logla("durum", Durum.Onaylandi);          // boxing yok
```

---

## 8. Garbage Collector

> **Benzetme —** Apartman görevlisini düşün. Her kata tek tek çıkıp "bu eşyanın sahibi var mı" diye sormaz; kapının önüne bırakılanlara bakar. En sık baktığı yer giriş kapısının önüdür — oraya bırakılanların çoğu zaten çöptür, bir dakikada toplanır. Bodruma yılda bir iner, orası zahmetlidir çünkü her kutuyu açıp bakması gerekir. Ve en önemlisi: senin daireye tıka basa doldurup kapıyı kilitlediğin eşyalara asla dokunmaz. O eşyalar çöp olsa bile, sahibi varmış gibi görünür.

**Basitçe:** Artık kullanmadığın nesneleri senin silmene gerek yok; arka planda çalışan bir mekanizma onları buluyor ve atıyor. "Artık kullanılmıyor"un ölçüsü şu: o nesneye ulaşan hiçbir yol kalmamış olması. Ama dikkat — bir yerde hâlâ o nesneye tutunan bir şey varsa, sen onu unutmuş olsan bile mekanizma onu çöp saymaz. Bellek sızıntısı tam olarak budur.

**Teknik olarak:** **GC (Garbage Collector)** — Heap'te artık erişilemeyen nesneleri otomatik temizleyen mekanizma. "Erişilemez" demek: kökten (stack değişkenleri, statik alanlar, CPU register'ları) başlayarak takip edildiğinde o nesneye ulaşan hiçbir referans kalmaması.

Dikkat: ölçüt "kullanılmıyor" değil, **"ulaşılamıyor"**. Bu ikisi aynı şey değildir ve tüm sızıntı hikâyesi bu farkta saklıdır.

### Üç aşama

1. **Mark (işaretle)** — Köklerden başlanır, ulaşılabilen her nesne canlı işaretlenir.
2. **Sweep (süpür)** — İşaretlenmeyenler çöp sayılır, alanları serbest bırakılır.
3. **Compact (sıkıştır)** — Kalan nesneler bellekte yan yana toplanır, boşluklar kapanır. (LOH varsayılan olarak sıkıştırılmaz.)

### Nesil (generation) modeli

GC her seferinde tüm heap'i taramaz — bu çok pahalı olurdu. Bunun yerine heap üç nesle bölünür:

| Nesil | İçerik | Toplama |
|---|---|---|
| **Gen 0** | Yeni doğan nesneler | Çok sık, çok ucuz |
| **Gen 1** | Gen 0'ı atlatanlar — tampon bölge | Orta sıklıkta |
| **Gen 2** | Uzun yaşayanlar: statikler, cache, singleton | Nadir ama **pahalı** — tüm heap taranır |

Dayanağı olan gözlem: **nesnelerin çoğu genç ölür.** Bir metot içinde oluşturulan geçici nesne saniyenin binde biri yaşar. Gen 0 toplaması bu yüzden hem sık hem ucuzdur.

**Bir toplama sırasında ne olur:** Yönetilen thread'ler kısa süreliğine durdurulur (*stop-the-world*). Gen 0 için bu mikrosaniyeler sürer; Gen 2 için milisaniyelere çıkabilir. Yüksek trafikli bir API'de gecikme sıçramalarının (latency spike) klasik sebebidir.

Bir nesnenin hangi nesilde olduğunu ve toplama sayılarını okuyabilirsin:

```csharp
var nesne = new byte[1024];
Console.WriteLine(GC.GetGeneration(nesne));   // 0

GC.Collect();                                  // sadece gözlem amaçlı
Console.WriteLine(GC.GetGeneration(nesne));   // 1  — hayatta kaldı, terfi etti

Console.WriteLine($"Gen0: {GC.CollectionCount(0)}, Gen2: {GC.CollectionCount(2)}");
Console.WriteLine($"Toplam yönetilen bellek: {GC.GetTotalMemory(false):N0} byte");
```

> **Bu benzetme şurada bozulur:** Apartman görevlisi örneği "GC arada bir gelir, sıra kimdeyse onu toplar" izlenimi verir. Gerçekte GC'yi tetikleyen bir takvim değil, **tahsis baskısıdır**: Gen 0 bölgesi dolduğunda toplama başlar. Yani hiç nesne üretmeyen bir uygulamada GC hiç çalışmaz; saniyede milyon nesne üretende ise saniyede defalarca çalışır. Toplama zamanını belirleyen sensin, saat değil.

### Pratik sonuç

Kısa ömürlü nesne üretmekten korkma; ucuzdur. Pahalı olan **uzun yaşayan nesneleri gereksiz yere tutmaktır.** ASP.NET Core'da bellek sızıntısı vakalarının neredeyse tamamı şu üç kalıptan çıkar:

1. Hiç temizlenmeyen `static` liste veya sözlük
2. Abone olunup (`+=`) hiç bırakılmayan event handler'lar — yayıncı, aboneyi hayatta tutar
3. Kapatılmayan bağlantı, stream, `HttpClient` yanlış kullanımı

Üçünün de kod hâli:

```csharp
// 1) Sınırsız büyüyen statik — uygulama kapanana kadar hiçbiri toplanmaz
public static class Izleme
{
    public static readonly List<Istek> Gecmis = new();   // sızıntı
}

// 2) Abonelikten çıkılmayan event — 'yayinci' yaşadığı sürece 'this' de yaşar
yayinci.VeriGeldi += Isle;      // ekledin
yayinci.VeriGeldi -= Isle;      // çıkarmayı unutma

// 3) HttpClient'ı her istekte new'lemek soket tüketir; tersi de yanlış değildir:
//    doğrusu IHttpClientFactory ile yönetilen tek örnek kullanmaktır.
```

**Server GC vs Workstation GC** — Sunucu uygulamaları (ASP.NET Core) varsayılan olarak Server GC kullanır: her CPU çekirdeği için ayrı heap ve ayrı GC thread'i. Throughput'u artırır, bellek kullanımını yükseltir.

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

---

## 9. `IDisposable` ve Deterministik Temizlik

> **Benzetme —** Apartman görevlisi sadece binanın içini toplar. Ama sen belediyeden bir alan kiralamışsan, ya da komşudan matkap ödünç almışsan, bunları görevli iade edemez — onun yetkisi dışındadır. O borçları sen kapatmak zorundasın, üstelik zamanında. "Nasıl olsa bir gün birisi hallederim" dersen, matkap sende kalır ve komşu bir daha vermez. Dosya tanıtıcıları ve veritabanı bağlantıları tam olarak bu ödünç matkaptır.

**Basitçe:** Otomatik temizlik sadece .NET'in kendi ürettiği nesneler için geçerli. Programın dışarıdan tuttuğu şeyler — açık bir dosya, bir veritabanı bağlantısı, bir ağ soketi — otomatik temizliğin göremediği kaynaklardır. Bunları "işim bitti" der demez sen bırakmalısın. C# bunun için özel bir söz dizimi verir ve hata çıksa bile bırakmayı garanti eder.

**Teknik olarak:** GC **yönetilen belleği** temizler. Ama şu kaynakları göremez:

- Dosya tanıtıcıları (file handle)
- Veritabanı bağlantıları
- Ağ soketleri
- İşletim sistemi seviyesindeki her şey

Bunlar **yönetilmeyen kaynaklardır** ve ne zaman bırakılacakları belli olmalıdır — GC'nin "bir ara gelirim" yaklaşımı yetmez. Çözüm arayüzü:

**`IDisposable`** — Tek metodu vardır: `Dispose()`. "Bu nesnenin tuttuğu kaynakları şimdi bırak" demektir.

```csharp
using var connection = new SqlConnection(connectionString);
connection.Open();
// blok bitince Dispose() otomatik çağrılır — exception atsa bile
```

`using` bloğu aslında bir `try/finally` yapısına dönüşür; bu yüzden hata olsa bile temizlik garanti edilir.

Derleyicinin ürettiğinin özü şudur:

```csharp
// Sen bunu yazarsın
using (var akis = File.OpenRead("rapor.csv")) { /* oku */ }

// Derleyici bunu üretir
var akis = File.OpenRead("rapor.csv");
try     { /* oku */ }
finally { if (akis != null) akis.Dispose(); }
```

Kendi tipin yönetilmeyen kaynak tutuyorsa arayüzü uygularsın:

```csharp
public sealed class RaporYazici : IDisposable
{
    private readonly StreamWriter _yazici = new("rapor.txt");
    private bool _atildi;

    public void Yaz(string satir) => _yazici.WriteLine(satir);

    public void Dispose()
    {
        if (_atildi) return;      // Dispose iki kez çağrılabilir; güvenli olmalı
        _yazici.Dispose();
        _atildi = true;
    }
}
```

Asenkron kaynaklar için `IAsyncDisposable` vardır; `await using` ile kullanılır.

```csharp
await using var conn = new SqlConnection(cs);
await conn.OpenAsync();
```

**Finalizer (`~ClassName`)** — GC'nin nesneyi toplamadan önce çağırdığı son çare metodu. Ne zaman çalışacağı belirsizdir ve nesnenin ömrünü bir GC turu daha uzatır. Modern C#'ta doğrudan yönetilmeyen kaynak tutmuyorsan yazma.

**Kural:** Bir tip `IDisposable` uyguluyorsa `using` ile kullan. IntelliSense'te `Dispose()` görüyorsan bu bir davettir, öneri değil.

Tek istisna: DI konteynerinden gelen servisleri sen `Dispose` etmezsin. Onların ömrünü konteyner yönetir; scope kapandığında temizliği kendisi yapar.

---

## Tek Bakışta Özet

- **.NET** platform, **C#** dil, **CLR** çalışma zamanı, **BCL** hazır kütüphane. Bugünkü ".NET", Framework ile Core'un birleşmiş hâli.
- Derleme iki aşamalı: kaynak → **IL** (derleme anı), IL → **makine kodu** (JIT, çalışma anı, metot bazında).
- **Assembly** = IL + metadata. Reflection, DI, EF Core, IntelliSense hep metadata'yı okur.
- **Stack** hızlı ve otomatik, **heap** esnek ve GC'ye bağımlı. 85 KB üstü nesneler **LOH**'a gider.
- **Value type kopyalanır, reference type paylaşılır.** C#'taki şaşırtıcı davranışların çoğu bu tek cümleye dayanır.
- **`string` immutable'dır** — döngüde birleştirme yerine `StringBuilder`.
- **Boxing** value type'ı heap'e taşır; generic kullanmak bunu ortadan kaldırır.
- **GC nesil temellidir**: genç nesneler ucuz, Gen 2 pahalı. Sızıntı kaynağı statikler, event'ler ve kapatılmayan kaynaklardır.
- **`IDisposable` + `using`**, GC'nin göremediği yönetilmeyen kaynaklar için deterministik temizliktir.
- Tek cümlelik bağ: **ne zaman kopyalandığını, nerede durduğunu ve kimin temizlediğini** bilirsen, bu haftanın geri kalanı ezber olmaktan çıkar.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| .NET | Runtime, kütüphane ve araçlardan oluşan geliştirme platformu |
| CLR | .NET'in çalışma zamanı motoru — bellek, tip güvenliği, exception, thread |
| BCL | Her .NET uygulamasında hazır gelen temel sınıf kütüphanesi |
| IL / MSIL / CIL | Derleyicinin ürettiği, CPU'dan bağımsız ara dil |
| Assembly | IL + metadata içeren `.dll` / `.exe` |
| Metadata | Tiplerin ve üyelerin çalışma anında okunabilir tanımı |
| JIT | IL'i metot ilk çağrıldığında makine koduna çeviren derleyici |
| Tiered compilation | Sık çağrılan metodun arka planda yeniden, daha iyi derlenmesi |
| AOT | Çalışmadan önce doğrudan makine kodu üretme (cold start çözümü) |
| Cold start | JIT ve yükleme maliyeti yüzünden ilk isteğin yavaş olması |
| Managed code | CLR gözetiminde çalışan, belleği GC tarafından yönetilen kod |
| CTS / CLS | Ortak tip sistemi / diller arası asgari uyumluluk kuralları |
| Stack | Metot çerçeveleri ve yerel değişkenler için LIFO bellek bölgesi |
| Stack frame | Bir metot çağrısının stack'te kapladığı blok |
| Heap | Nesne gövdelerinin tutulduğu, GC tarafından yönetilen bölge |
| LOH | 85 KB üstü nesnelere ayrılmış, sıkıştırılmayan heap bölgesi |
| Value type | Atamada içeriği kopyalanan tip (`struct`, `enum`, `int`...) |
| Reference type | Atamada adresi kopyalanan tip (`class`, `string`, diziler...) |
| record | Değer eşitliğine sahip, DTO'lar için ideal reference type |
| `with` ifadesi | Bir record'un tek alanı değişmiş kopyasını üretme söz dizimi |
| Immutable | Oluşturulduktan sonra içeriği değiştirilemeyen |
| String interning | Aynı string literal'lerinin tek nesnede toplanması |
| Boxing / Unboxing | Value type'ın heap'e taşınması / geri çıkarılması |
| GC | Erişilemez nesneleri otomatik temizleyen mekanizma |
| Generation (Gen 0/1/2) | GC'nin nesneleri yaşlarına göre ayırdığı bölümler |
| Stop-the-world | GC toplaması sırasında thread'lerin kısa süre durdurulması |
| Server GC | Çekirdek başına ayrı heap kullanan, sunucu için varsayılan GC modu |
| IDisposable | Yönetilmeyen kaynakların deterministik bırakılması arayüzü |
| IAsyncDisposable | Aynı işin asenkron karşılığı — `await using` ile kullanılır |
| Finalizer | GC toplamadan önce çağrılan son çare temizlik metodu |
| Memory leak | Artık gerekmeyen nesnelerin referansla hayatta tutulması |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Value type hep stack'te durur" | Bir sınıfın alanı olan value type, o nesneyle birlikte **heap'te** durur |
| "`string` value type'tır" | Reference type'tır; value gibi davranmasının sebebi immutability |
| "GC'yi `GC.Collect()` ile çağırmak performansı artırır" | Neredeyse her zaman zarar verir — GC'nin nesil sezgisini bozar |
| "`Dispose()` nesneyi bellekten siler" | Hayır, yönetilmeyen kaynağı bırakır. Belleği yine GC temizler |
| ".NET Core ile .NET ayrı ürünler" | .NET 5'ten itibaren tek ürün; "Core" ismi bırakıldı |
| "`class` yerine `struct` kullanmak hep daha hızlı" | Büyük struct'larda kopyalama maliyeti kazancı yer |
| "`ad.Trim()` yazınca metin temizlenir" | Yeni string döner; geri atamazsan hiçbir şey değişmez |
| "GC belirli aralıklarla çalışır" | Takvimle değil, tahsis baskısıyla tetiklenir — Gen 0 dolunca |
| "`int` ile `System.Int32` farklı tiplerdir" | Aynı tiptir; `int` yalnızca bir takma addır |
| "Bir nesneyi `null`'a eşitlemek onu anında siler" | Sadece referansı koparır; toplama zamanını GC belirler |

---

## Sonraki

→ `02-Koleksiyonlar.md` (Salı)
