# Hafta 1 · Cumartesi — Generics, Delegate, Extension, Exception, IDisposable

**Okuma süresi:** ~56 dk
**Neden bu konu:** Haftanın kalan yapı taşları. LINQ'in extension method olduğunu, `Action`/`Func`'ın delegate olduğunu, `Repository<T>`'nin generic olduğunu bilmeden bootcamp'teki mimari projeleri okuyamazsın.

---

## Önce Basitçe

Bu notta beş ayrı konu var ama hepsi tek bir soruya cevap veriyor: **kodun parçaları birbirine nasıl bağlanır?** Bir program tek bir dev metottan ibaret olsaydı bunların hiçbirine ihtiyaç olmazdı. Ama gerçek programlar yüzlerce küçük parçadan oluşur ve o parçaların birbirini tanımadan, birbirine bağımlı olmadan çalışması gerekir. Buradaki her özellik, iki parçayı birbirine bağlamanın bir yoludur.

Birinci bağlanma biçimi **tip üzerinden**. Bir kutu yaparsın ama içine ne konacağını şimdiden söylemezsin; kutuyu kullanan kişi söyler. Buna generic deniyor. Bir müşteri listesi ile bir ürün listesi arasındaki tek fark içindekilerdir; listeyi iki kez yazmanın anlamı yok. Bunun eski alternatifi "her şeyi kabul eden kutu" yapmaktı ve o yolda iki bela vardı: kutudan çıkanı her seferinde elle çevirmek zorunda kalırdın ve yanlış çevirirsen program çalışırken patlardı.

İkinci bağlanma biçimi **davranış üzerinden**. Bir metoda "şu işi yap" demek yerine "şunu nasıl yapacağını sana ben söyleyeceğim" diyorsun. Bir listeyi filtreleyen kodu bir kez yazarsın; hangi şarta göre filtreleneceğini çağıran kişi verir. İşte delegate, lambda ve LINQ'in tamamı bu fikrin üstünde duruyor. Bir işi parametre olarak geçirebilmek, ilk duyduğunda garip gelen ama alışınca vazgeçilmez olan bir şeydir.

Üçüncüsü **yazım üzerinden**: extension method. Var olan bir tipe dokunmadan, ona yeni bir metot eklenmiş gibi yazabilmeni sağlar. Gerçekte hiçbir şey eklenmez — derleyici arkada düz bir statik metot çağrısına çevirir — ama kodun okunuşu değişir ve LINQ'in var olma sebebi tam olarak budur.

Dördüncüsü **hata üzerinden**. Bir şey ters gittiğinde ne olacağı da bir bağlantı biçimidir: hatayı burada mı çözeceksin, yoksa üst kata mı bırakacaksın? Bu notun en çok pratik fayda üretecek kısmı burası, çünkü hata yönetiminde yapılan yanlışlar sessizdir — program çalışmaya devam eder ama artık neyin neden bozulduğunu kimse bilmez.

Beşincisi **kaynak üzerinden**. Programın kullandığı bazı şeyler senin belleğinde değildir: bir dosya, bir veritabanı bağlantısı, bir ağ soketi. Bunları bırakmayı unutursan bellek dolmaz ama bağlantı biter ve uygulama durur. Bunun için ayrı bir sözleşme var.

Bu beş konu bootcamp boyunca her gün karşına çıkacak. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bu notu bir **usta–çırak atölyesi** gibi düşün. Generic, atölyedeki standart kasalardır: hepsi aynı ölçüdedir, üstündeki etiket içindekini söyler. Delegate, ustanın çırağa verdiği vekâlettir: "bu işi benim yerime sen yap, nasıl yapacağını şu kâğıda yazdım". Extension method, alete sonradan takılan aparattır: alete dokunmaz, ama kullanırken aletin parçası gibi durur. Exception, tezgâhın üstündeki sigortadır. `IDisposable` ise gün sonunda ödünç aldığın takımı yerine bırakma sorumluluğudur.

---

## Bu Notta Ne Var

1. Generics — tip parametreleri ve kısıtlar
2. Delegate, `Func`, `Action`, `Predicate`
3. Lambda ifadeleri ve closure
4. Extension method
5. Exception ve exception yönetimi
6. `IDisposable` ve `using`
7. `object` sınıfının metotları ve eşitlik
8. Interface'in dil düzeyindeki rolü

---

## 1. Generics

> **Benzetme —** Markete gittiğinde alışverişini tek bir naylon poşete doldurabilirsin: yumurta, deterjan, domates hepsi bir arada. Poşet her şeyi kabul eder — kolay yoldur. Eve gelince bedelini ödersin: elini sokup yoklarsın, yanlış tuttuğunda yumurta kırılır. Alternatifi, üstünde ne yazdığı belli bölmeli kasalardır. Kasa da standarttır, aynı kalıptan çıkmıştır; farkı, üstündeki etiketin baştan belli olmasıdır. `object` poşettir, generic kasadır.

**Basitçe:** Aynı işi yapan ama farklı tiplerle çalışan kodu iki kez yazmak istemezsin. Generic, "bu sınıf bir liste tutar, neyin listesi olduğunu kullanan söyler" demenin yoludur. Kazancın sadece daha az yazı değil: tip baştan belli olduğu için derleyici yanlış kullanımı yakalar ve çalışma anında dönüştürme maliyeti kalmaz.

**Teknik olarak:** **Generic (jenerik)** — Tipi, sınıfın veya metodun tanımında sabitlemeyip **kullanım anında** belirlemeye yarayan mekanizma. `<T>` içindeki `T` bir yer tutucudur.

### Neden var

Generic'ten önce "her tiple çalışan" kod yazmanın tek yolu `object` kullanmaktı:

```csharp
// Eski yol — iki sorun birden
public class Kutu
{
    public object Deger { get; set; }
}
var k = new Kutu { Deger = 5 };
int sayi = (int)k.Deger;      // 1) boxing maliyeti  2) yanlış tipe cast → çalışma anı hatası
```

Generic bu iki sorunu da çözer:

```csharp
public class Kutu<T>
{
    public T Deger { get; set; }
}
var k = new Kutu<int> { Deger = 5 };
int sayi = k.Deger;           // boxing yok, cast yok, tip güvenli
```

Kazanımlar: **tip güvenliği** (hatalar derleme anında), **performans** (boxing yok), **kod tekrarının azalması**.

**Boxing (kutulama)** — Bir value type'ın (`int`, `struct`) heap üzerinde bir nesneye sarılması. `object` kullanan her yerde sessizce olur ve her seferinde bellek ayırma demektir.

```csharp
var eski = new System.Collections.ArrayList();
for (int i = 0; i < 1_000_000; i++) eski.Add(i);   // bir milyon boxing

var yeni = new List<int>();
for (int i = 0; i < 1_000_000; i++) yeni.Add(i);   // sıfır boxing
```

> .NET'te generic'ler çalışma anında da gerçek tiplerini korur (*reified generics*). Java'daki gibi silinmezler; bu yüzden `typeof(List<int>)` ile `typeof(List<string>)` farklı tiplerdir ve reflection ile tip bilgisine ulaşabilirsin.

Bunun doğrudan bir sonucu var: çalışma zamanı, her value type için ayrı makine kodu üretir (özelleştirme), reference type'lar ise paylaşılan tek bir kodu kullanır.

```csharp
Console.WriteLine(typeof(List<int>) == typeof(List<string>));   // False
Console.WriteLine(typeof(List<>).Name);                          // List`1  — açık generic tip
```

### Generic metotlar

Sadece sınıflar değil, metotlar da generic olabilir. Çoğu zaman tip parametresini yazmana bile gerek kalmaz — derleyici argümandan çıkarır.

```csharp
public static T SonuncusuOlmayan<T>(IList<T> liste) => liste[^2];

var x = SonuncusuOlmayan(new[] { 1, 2, 3 });        // T = int, yazmadın
var y = SonuncusuOlmayan<string>(new[] { "a", "b" }); // açıkça da yazabilirsin
```

Birden çok tip parametresi de olabilir:

```csharp
public static Dictionary<TKey, TValue> Birlestir<TKey, TValue>(
    IEnumerable<TKey> anahtarlar, Func<TKey, TValue> uret) where TKey : notnull
    => anahtarlar.ToDictionary(a => a, uret);
```

### Generic kısıtlar (constraints)

`T`'nin ne olabileceğini sınırlar — ve sınırladığın ölçüde `T` üzerinde işlem yapabilirsin.

| Kısıt | Anlamı |
|---|---|
| `where T : class` | Reference type olmalı |
| `where T : struct` | Value type olmalı |
| `where T : new()` | Parametresiz constructor'ı olmalı |
| `where T : IEntity` | Belirli bir arayüzü uygulamalı |
| `where T : BaseEntity` | Belirli bir sınıftan türemeli |
| `where T : IComparable<T>` | Karşılaştırılabilir olmalı |
| `where T : notnull` | Null olamayan bir tip olmalı |
| `where T : unmanaged` | İçinde referans barındırmayan value type olmalı |
| `where T : U` | Başka bir tip parametresinden türemeli |

Bootcamp'te en çok göreceğin generic yapı, generic repository'dir:

```csharp
public interface IRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Delete(T entity);
}
```

Kısıt olmasaydı `T`'nin bir `Id` özelliği olduğunu varsayamazdın. **Kısıt, `T` hakkında derleyiciye verdiğin bilgidir.**

Kısıtın neyi açtığını somut görmek için:

```csharp
// Kısıtsız — T hakkında hiçbir şey bilmiyorsun
public T Bul<T>(List<T> liste, int id)
{
    // liste.First(x => x.Id == id);   // derlenmez: T'nin Id'si olduğunu kimse söylemedi
    throw new NotSupportedException();
}

// Kısıtlı — artık Id'ye erişebilirsin
public T? Bul2<T>(List<T> liste, int id) where T : BaseEntity
    => liste.FirstOrDefault(x => x.Id == id);
```

Birden çok kısıt aynı anda verilebilir; sıralaması sabittir (önce `class`/`struct`, sonra taban sınıf ve arayüzler, en sonda `new()`):

```csharp
public class Fabrika<T> where T : BaseEntity, IValidatable, new()
{
    public T Uret()
    {
        var ornek = new T();      // new() kısıtı olmasaydı bu satır derlenmezdi
        ornek.Dogrula();
        return ornek;
    }
}
```

**`default(T)` tuzağı:** Kısıt yoksa `T` hem `int` hem `string` olabilir; ikisinin varsayılanı farklıdır.

```csharp
public static T? BulVeyaVarsayilan<T>(List<T> liste, Func<T, bool> kosul)
{
    foreach (var x in liste) if (kosul(x)) return x;
    return default;      // T = int ise 0, T = string ise null
}
```

### Varyans — kısa not
**Covariance (`out T`)** — `IEnumerable<Kopek>` bir `IEnumerable<Hayvan>` yerine kullanılabilir (sadece okuma yönünde).
**Contravariance (`in T`)** — Ters yönde, sadece yazma/tüketme yönünde geçerlidir.
Şimdilik "`IEnumerable<T>` neden esnek de `List<T>` neden değil" sorusunun cevabı olarak bilmen yeterli.

Neden `List<T>` esnek değil, tek örnekte:

```csharp
List<Kopek> kopekler = new();
// List<Hayvan> hayvanlar = kopekler;   // izin verilseydi...
// hayvanlar.Add(new Kedi());           // ...köpek listesine kedi girerdi

IEnumerable<Hayvan> okunur = kopekler;  // güvenli: IEnumerable'a ekleme yapılamaz
```

Kural basit: **okunan** yer covariant (`out`), **yazılan** yer contravariant (`in`) olabilir; ikisini birden yapan tip (`List<T>`) invariant kalmak zorundadır.

```csharp
public interface IUretici<out T> { T Uret(); }       // sadece döndürür
public interface ITuketici<in T> { void Tuket(T x); } // sadece alır

ITuketici<Hayvan> h = new HayvanTuketici();
ITuketici<Kopek> k = h;    // geçerli: hayvan tüketebilen, köpeği de tüketir
```

---

## 2. Delegate, `Func`, `Action`, `Predicate`

> **Benzetme —** Noterden vekâlet vermek gibidir. Vekâletnamede kimin, hangi konuda, ne yapabileceği yazar: "tapu işlemleri için", "araç satışı için". Vekâleti alan kişi senin yerine gider ve o işi yapar. Delegate de böyle bir belgedir: hangi parametreleri alıp ne döndürecek bir metodu temsil edebileceği baştan yazılıdır. Yanlış konuda vekâlet kullanılamaz — imza tutmuyorsa noter kabul etmez.

**Basitçe:** Bir metodun kendisini bir değişkende tutabilir, başka bir metoda parametre olarak verebilirsin. "Şu listeyi filtrele" demek yerine "şu listeyi, sana verdiğim şarta göre filtrele" diyebilmenin yolu budur. Filtrelemeyi yapan kod bir kere yazılır; şartı her seferinde çağıran belirler.

**Teknik olarak:** **Delegate** — Bir **metoda işaret eden tip**. Metodu değişkende tutmayı, parametre olarak geçirmeyi ve döndürmeyi sağlar. "Tip güvenli fonksiyon işaretçisi" olarak düşün.

```csharp
public delegate int Hesaplayici(int a, int b);

Hesaplayici topla = (a, b) => a + b;
int sonuc = topla(3, 4);      // 7
```

Delegate bir **tiptir**; bu yüzden metot parametresi, dönüş değeri ve alan olabilir:

```csharp
int Uygula(int a, int b, Hesaplayici islem) => islem(a, b);

Console.WriteLine(Uygula(6, 3, (x, y) => x - y));   // 3
Console.WriteLine(Uygula(6, 3, (x, y) => x * y));   // 18
```

Pratikte kendi delegate tipini tanımlamana genelde gerek yoktur; .NET hazırlarını sunar:

| Tip | İmza | Kullanım |
|---|---|---|
| **`Action`** | Parametre alabilir, **değer döndürmez** | `Action<string>` — bir string alır, `void` |
| **`Func`** | Parametre alabilir, **değer döndürür**. Son tip parametresi dönüş tipidir | `Func<int, int, string>` — iki `int` alır, `string` döner |
| **`Predicate<T>`** | `T` alır, `bool` döner | Filtreleme koşulları |

```csharp
Action<string> yazdir   = mesaj => Console.WriteLine(mesaj);
Func<int, int, int> carp = (a, b) => a * b;
Predicate<Urun> pahaliMi = u => u.Fiyat > 1000;
```

`Func`'ta son tip parametresinin dönüş tipi olduğunu unutma — en sık yapılan okuma hatası budur:

```csharp
Func<string>                 uret;    // parametre yok, string döner
Func<int, string>            cevir;   // int alır, string döner
Func<int, string, bool>      dogrula; // int ve string alır, bool döner
Action                       tetikle; // parametre yok, bir şey döndürmez
Action<int, string>          kaydet;  // iki parametre, dönüş yok
```

**LINQ'in tamamı bunun üstünde durur.** `Where(u => u.Fiyat > 100)` çağrısında parametre bir `Func<Urun, bool>`tur.

Bunu kendin yazarak gör — `Where`'in özü on satırdır:

```csharp
public static IEnumerable<T> BenimWhere<T>(this IEnumerable<T> kaynak, Func<T, bool> kosul)
{
    foreach (var eleman in kaynak)
        if (kosul(eleman))
            yield return eleman;
}

var ucuzlar = urunler.BenimWhere(u => u.Fiyat < 100);
```

> `Predicate<T>` ile `Func<T, bool>` aynı imzaya sahiptir ama **farklı tiplerdir**; birbirine doğrudan atanamazlar. LINQ `Func<T, bool>` kullanır, `List<T>.FindAll` ise `Predicate<T>`. Yeni kodda `Func<T, bool>` tercih edilir.

Bir delegate'e isimli metot da atanabilir; lambda zorunlu değildir:

```csharp
static bool CiftMi(int x) => x % 2 == 0;

Func<int, bool> f = CiftMi;               // metot grubu dönüşümü
var ciftler = sayilar.Where(CiftMi);      // doğrudan metot adı
```

**Multicast delegate** — Bir delegate birden çok metodu tutabilir (`+=` ile eklenir). **Event** mekanizmasının temeli budur.

```csharp
Action<string> bildir = m => Console.WriteLine($"Konsol: {m}");
bildir += m => File.AppendAllText("log.txt", m);

bildir("kaydedildi");    // ikisi de çalışır, eklenme sırasıyla
```

> Multicast bir `Func` çağırdığında **yalnızca son metodun dönüş değeri** elde edilir; diğerlerininki sessizce atılır. Bu yüzden multicast, dönüş değeri olmayan `Action` ile kullanılır.

**Event** — Delegate'in kapsüllenmiş hâli. Dışarıdan yalnızca abone olunabilir (`+=`) veya çıkılabilir (`-=`); tetiklemek sınıfın kendi işidir.

```csharp
public class Siparis
{
    public event EventHandler<string>? DurumDegisti;

    public void Onayla()
    {
        // ... iş mantığı
        DurumDegisti?.Invoke(this, "Onaylandi");   // tetiklemeyi yalnızca sınıf yapabilir
    }
}

var s = new Siparis();
s.DurumDegisti += (gonderen, durum) => Console.WriteLine(durum);
// s.DurumDegisti = null;      // derleme hatası: dışarıdan atanamaz
// s.DurumDegisti.Invoke(...); // derleme hatası: dışarıdan tetiklenemez
```

Alan olarak tanımlanmış düz bir `Action` ile `event` arasındaki fark tam olarak budur: `Action` alanını dışarıdan biri `null`'layabilir veya tetikleyebilir, `event` ise edemez.

> Event'lerde bellek sızıntısının sebebi: abone olan nesne, yayıncı tarafından referansla tutulur. `-=` ile abonelikten çıkılmazsa abone hiç GC'lenmez.

Bu yüzden abonelikten çıkabilmek için lambda'yı bir değişkende tutman gerekir:

```csharp
EventHandler<string> el = (g, d) => Console.WriteLine(d);
s.DurumDegisti += el;
s.DurumDegisti -= el;      // aynı referans olduğu için çıkarılabilir

s.DurumDegisti += (g, d) => Console.WriteLine(d);
s.DurumDegisti -= (g, d) => Console.WriteLine(d);   // çıkmaz: bu farklı bir lambda nesnesi
```

---

## 3. Lambda İfadeleri ve Closure

> **Benzetme —** Emlakçıya not bırakırsın: "5000 liradan ucuz, iki artı bir daire çıkarsa beni ara." Notta bir kural var ve o kuralın içinde senin bugün söylediğin rakam yazılı. Sen evine gidersin, aradan bir ay geçer, sen o rakamı unutursun — ama not hâlâ emlakçının masasında durur ve içindeki 5000 rakamı hâlâ geçerlidir. Lambda bu nottur; notun içindeki rakamı taşıması ise closure'dır.

**Basitçe:** Lambda, adı olmayan küçük bir metottur. Tek satırlık bir kural yazmak için ayrı bir metot tanımlamak zahmetlidir; lambda o kuralı yazdığın yere gömmeni sağlar. Closure ise şudur: lambda, yazıldığı yerdeki değişkenleri hatırlar ve onları kendisiyle birlikte taşır.

**Teknik olarak:** **Lambda** — İsimsiz metot yazma sözdizimi: `(parametreler) => gövde`.

```csharp
u => u.Fiyat > 100                              // tek parametre, tek ifade
(a, b) => a + b                                 // iki parametre
(x) => { var y = x * 2; return y + 1; }         // blok gövde
() => Console.WriteLine("selam")                // parametresiz
```

Parametre tipleri gerekirse açıkça yazılabilir; kullanılmayan parametre `_` ile atılabilir:

```csharp
Func<int, int, int> f = (int a, int b) => a + b;
Action<object?, EventArgs> h = (_, _) => Console.WriteLine("olay");
Func<int, int> g = static x => x * 2;    // static lambda: dış değişken yakalayamaz (C# 9)
```

**Closure (kapanış)** — Lambda'nın, tanımlandığı kapsamdaki değişkenleri **yakalaması**.

```csharp
int esik = 100;
var pahalilar = urunler.Where(u => u.Fiyat > esik);   // esik yakalandı
```

Derleyici burada gizli bir sınıf üretir ve `esik` değişkenini o sınıfın alanı yapar. Sonucu: değişken, metot bitse bile yaşamaya devam eder.

Kritik ayrıntı: **değer değil, değişkenin kendisi yakalanır.** Yakalamadan sonra değeri değiştirirsen lambda yeni değeri görür.

```csharp
int sayac = 0;
Action artir = () => sayac++;

artir();
artir();
Console.WriteLine(sayac);    // 2 — lambda dışarıdaki değişkeni gerçekten değiştirdi
```

Bu, lambda'nın yakaladığı yerel değişkeni **heap'e taşıması** demektir. Sıcak döngülerde her yineleme için yeni bir gizli nesne üretilebilir; performansa duyarlı kodda `static` lambda kullanmak veya değeri parametreyle geçmek bunu önler.

**Klasik tuzak — döngü değişkeni yakalama:**

```csharp
var isler = new List<Action>();
for (int i = 0; i < 3; i++)
    isler.Add(() => Console.WriteLine(i));    // i'nin KENDİSİ yakalanır

foreach (var is_ in isler) is_();             // 3, 3, 3 yazar — 0,1,2 değil
```

Çözüm: döngü içinde yerel bir kopya oluşturmak. (`foreach` değişkeni C# 5'ten beri her turda yeni sayılır, ama `for` için sorun devam eder.)

```csharp
var isler2 = new List<Action>();
for (int i = 0; i < 3; i++)
{
    int kopya = i;                            // her turda yeni değişken
    isler2.Add(() => Console.WriteLine(kopya));
}
foreach (var is_ in isler2) is_();            // 0, 1, 2
```

Aynı tuzak asenkron kodda daha sinsi görünür:

```csharp
var gorevler = new List<Task>();
for (int i = 0; i < 3; i++)
    gorevler.Add(Task.Run(() => Console.WriteLine(i)));   // sonuç öngörülemez
```

> Yakalanan değişken bir nesneye referans tutuyorsa, o nesne lambda yaşadığı sürece GC'lenemez. Uzun ömürlü bir event handler'ın içinde büyük bir nesneyi yakalamak, sessiz bir bellek sızıntısıdır.

**Expression tree farkı:** Aynı lambda, hedef tipe göre iki farklı şeye derlenebilir.

```csharp
Func<Urun, bool> derlenmis = u => u.Fiyat > 100;          // çalıştırılabilir kod
Expression<Func<Urun, bool>> agac = u => u.Fiyat > 100;   // kodun veri hâli
```

`IQueryable` (EF Core) ikincisini ister: sorguyu çalıştırmaz, **okur** ve SQL'e çevirir. `IEnumerable` ise birincisini alır ve bellekte çalıştırır. LINQ'te "bu sorgu neden veritabanına gitmedi" sorusunun cevabı çoğu zaman buradadır.

---

## 4. Extension Method

> **Benzetme —** Eski bir apartmana dış cepheden asansör takılır. İçeriden bakınca sanki binanın kendi asansörüymüş gibi kullanırsın — kapıya basar, çıkarsın. Ama gerçekte asansör binanın içinde değildir: ortak alanlara, kazan dairesine, kilitli bölümlere erişemez. Ve eğer binanın zaten kendi asansörü varsa kimse dış cepheyi kullanmaz.
>
> **Bu benzetme şurada bozulur:** Dış asansör en azından fiziksel olarak binaya bağlanır. Extension method tipe hiçbir şekilde bağlanmaz; derleyici çağrıyı yazarken senin için bir çeviri yapar, o kadar. Tipin kendisi o metodun varlığından habersizdir.

**Basitçe:** Başkasının yazdığı bir tipe (`string`, `DateTime`, bir kütüphane sınıfı) yeni bir metot eklemek istersin ama kaynağına dokunamazsın. Extension method, o metodu ayrı bir yerde yazıp, kullanırken tipin kendi metoduymuş gibi çağırmanı sağlar. Yazım kolaylığıdır; tipe gerçekten bir şey eklenmez.

**Teknik olarak:** **Extension method (genişletme metodu)** — Var olan bir tipe, kaynak kodunu değiştirmeden yeni metot ekliyormuş gibi görünmeyi sağlayan sözdizimi kolaylığı.

Kuralları:
1. `static` bir sınıf içinde olmalı
2. Metot `static` olmalı
3. İlk parametre `this` ile başlamalı

```csharp
public static class StringExtensions
{
    public static bool DoluMu(this string? deger)
        => !string.IsNullOrWhiteSpace(deger);
}

// Kullanım — sanki string'in kendi metoduymuş gibi
if (musteri.Ad.DoluMu()) { ... }
```

Gerçekte olan: derleyici bunu `StringExtensions.DoluMu(musteri.Ad)` çağrısına çevirir. Yani tipe hiçbir şey **eklenmez**; sadece yazım kolaylaşır.

Bu çevirinin doğrudan bir sonucu vardır: **extension method null üzerinde de çağrılabilir.** Normal bir metot `NullReferenceException` atardı; extension method ise sadece `null` bir argüman alır.

```csharp
string? bos = null;
Console.WriteLine(bos.DoluMu());     // False — patlamaz, çünkü aslında statik çağrı

// bos.ToUpper();                    // bu patlar: gerçek üye
```

**Görünürlük kuralı:** Extension method'u kullanabilmek için sınıfının namespace'ini `using` ile içeri almalısın. Metot "kaybolduğunda" ilk bakılacak yer burasıdır.

**Neden önemli:** `System.Linq.Enumerable` sınıfı, `IEnumerable<T>` üzerinde tanımlı onlarca extension method içerir. `Where`, `Select`, `OrderBy` — hepsi budur. `using System.Linq;` satırını silersen LINQ metotları kaybolur; işte sebebi.

ASP.NET Core'da da her yerde karşına çıkar: `services.AddControllers()`, `app.UseAuthentication()` — hepsi extension method'dur.

Kendi projende en çok kullanacağın kalıp, servis kayıtlarını toparlamaktır:

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddUygulamaServisleri(this IServiceCollection services)
    {
        services.AddScoped<IMusteriRepository, MusteriRepository>();
        services.AddScoped<IMusteriService, MusteriService>();
        return services;              // zincirleme için kendini döndür
    }
}

// Program.cs
builder.Services.AddUygulamaServisleri().AddControllers();
```

Zincirlemenin sırrı bu `return services;` satırıdır: extension method kendi aldığı nesneyi geri döndürürse, arka arkaya yazılabilir. LINQ'in `Where(...).OrderBy(...).Select(...)` akışı da tamamen bu fikirdir.

Generic ve kısıtlı extension method da yazılabilir:

```csharp
public static class EnumerableExtensions
{
    public static IEnumerable<IReadOnlyList<T>> Parcala<T>(this IEnumerable<T> kaynak, int boyut)
    {
        var tampon = new List<T>(boyut);
        foreach (var x in kaynak)
        {
            tampon.Add(x);
            if (tampon.Count == boyut) { yield return tampon; tampon = new List<T>(boyut); }
        }
        if (tampon.Count > 0) yield return tampon;
    }
}

foreach (var grup in Enumerable.Range(1, 10).Parcala(3))
    Console.WriteLine(string.Join(",", grup));   // 1,2,3 / 4,5,6 / 7,8,9 / 10
```

> **Sınır:** Extension method özel (`private`) üyelere erişemez ve aynı isimde gerçek bir üye varsa **gerçek üye kazanır**.

İkinci kuralın pratik sonucu: bir kütüphane, senin extension method'unla aynı adda gerçek bir metot eklerse, senin kodun sessizce **başka bir metodu** çağırmaya başlar. Derleme hatası olmaz, davranış değişir. Bu yüzden extension method adlarını yaygın isimlerden (`Add`, `Get`, `Parse`) kaçınarak seç.

Aşırı kullanımın bedeli de vardır: her tipe her şeyi eklemek, IntelliSense listesini şişirir ve nesnenin gerçekten ne yapabildiğini gizler. Ölçü şu: **tipin kendi sorumluluğu değilse** extension olsun; sorumluluğuysa tipin içine yaz.

---

## 5. Exception ve Exception Yönetimi

> **Benzetme —** Evin elektrik tesisatındaki sigortayı düşün. Bir yerde kaçak olunca sigorta atar ve akım kesilir — ev yanmasın diye. Sigortanın attığını görürsün, hangi hat olduğunu kutuda okursun, gider o prizi kontrol edersin. Şimdi birinin, sigorta atınca kutuya gidip **hangi hat olduğunu gösteren etiketi sildiğini** düşün. Akım geri gelir, ev çalışır, ama bir daha arızanın nerede olduğunu kimse bulamaz. `throw ex` yazmak tam olarak budur.

**Basitçe:** Bir şeyler ters gittiğinde program bir "hata nesnesi" fırlatır ve o noktadan sonra normal akış durur. Sen bu nesneyi yakalayıp bir şey yapabilirsin — loglayabilir, kullanıcıya mesaj gösterebilir, ya da çözemiyorsan üst kata bırakabilirsin. En önemli kural: çözemeyeceğin hatayı yakalama, ve yakaladığında hatanın nereden geldiği bilgisini kaybetme.

**Teknik olarak:** **Exception** — Programın normal akışını bozan, olağandışı durumu temsil eden nesne. Tüm exception'lar `System.Exception`'dan türer.

### Hiyerarşi
```
Exception
├── SystemException
│   ├── NullReferenceException
│   ├── InvalidOperationException
│   ├── ArgumentException
│   │   ├── ArgumentNullException
│   │   └── ArgumentOutOfRangeException
│   ├── IndexOutOfRangeException
│   └── FormatException
└── (kendi özel exception'ların)
```

Her exception nesnesinin taşıdığı temel bilgiler:

| Üye | Ne taşır |
|---|---|
| `Message` | İnsan tarafından okunabilir açıklama |
| `StackTrace` | Hatanın hangi çağrı zincirinden geldiği |
| `InnerException` | Bu hataya sebep olan alttaki hata (varsa) |
| `Data` | Ek anahtar/değer bilgisi eklemek için sözlük |
| `Source` | Hatayı üreten assembly adı |

### `try` / `catch` / `finally`

```csharp
try
{
    var veri = await _api.GetirAsync(id);
}
catch (HttpRequestException ex)          // özel olandan
{
    _logger.LogWarning(ex, "API erişilemedi: {Id}", id);
    throw;                               // yeniden fırlat
}
catch (Exception ex)                     // genele doğru
{
    _logger.LogError(ex, "Beklenmeyen hata");
    throw;
}
finally
{
    // hata olsun olmasın çalışır — temizlik yeri
}
```

**Sıralama kuralı:** `catch` blokları **özelden genele** yazılır. `catch (Exception)` en üstteyse alttakiler hiç çalışmaz (derleyici bunu zaten engeller).

### `throw` ve `throw ex` farkı

```csharp
catch (Exception ex)
{
    throw;        // orijinal stack trace korunur
    throw ex;     // stack trace SIFIRLANIR — hatanın nerede doğduğu kaybolur
}
```

Bu, hata ayıklamayı imkânsızlaştıran, çok yaygın ve sessiz bir hatadır.

Farkı görmek için üç katmanlı bir çağrı düşün: `Controller → Service → Repository`. Hata repository'de doğar.

```
throw;      → StackTrace: Repository.Bul → Service.Getir → Controller.Get
throw ex;   → StackTrace: Service.Getir → Controller.Get        (Repository kayıp)
```

Hatayı zenginleştirmek istiyorsan sarmalarsın — ama orijinali içeride tutarsın:

```csharp
catch (SqlException ex)
{
    throw new VeriErisimException($"Müşteri {id} okunamadı", ex);   // inner exception korunur
}
```

Stack trace'i kaybetmeden bir exception'ı saklayıp sonra fırlatman gerekiyorsa (asenkron kodda olur):

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo.Capture(ex).Throw();   // orijinal trace korunarak yeniden fırlatılır
```

### İyi pratikler

| Kural | Gerekçe |
|---|---|
| Exception'ı **akış kontrolü** için kullanma | Pahalıdır (stack unwinding); `if` ile çözülebiliyorsa `if` kullan |
| Boş `catch { }` yazma | Hatayı yutar, sorun görünmez hâle gelir — en tehlikelisi |
| Yakalayamayacağın exception'ı yakalama | Çözemeyeceksen üst katmana bırak |
| Anlamlı özel exception tipleri tanımla | `MusteriBulunamadiException`, üst katmanın doğru tepki vermesini sağlar |
| Merkezî hata yönetimi kur | ASP.NET Core'da exception middleware — her controller'da `try/catch` yerine |
| Kullanıcıya stack trace gösterme | Güvenlik açığı; loga yaz, kullanıcıya sade mesaj ver |
| `catch (Exception)` yalnızca en dış katmanda | İç katmanlarda her şeyi yakalamak, teşhisi imkânsızlaştırır |

Özel exception tipi yazarken üç constructor'ı da vermek adettendir:

```csharp
public class MusteriBulunamadiException : Exception
{
    public int MusteriId { get; }

    public MusteriBulunamadiException(int id)
        : base($"Müşteri bulunamadı: {id}") => MusteriId = id;

    public MusteriBulunamadiException(string mesaj, Exception ic)
        : base(mesaj, ic) { }
}
```

**`when` filtresi** — Yakalama koşulu ekler; koşul sağlanmazsa exception hiç yakalanmamış gibi devam eder.

```csharp
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return null;
}
```

`when` ile `catch` içinde `if` yazmak aynı şey değildir. `when` koşulu **stack daha çözülmeden** değerlendirilir; yakalamazsan hata olduğu yerde kalır, stack trace bozulmaz. `catch` içinde `if` yazıp `throw;` dersen stack zaten çözülmüştür.

```csharp
// Yalnızca loglayıp akışı hiç bozmayan hile
catch (Exception ex) when (Logla(ex)) { }    // Logla her zaman false döner → yakalanmaz
```

**Exception maliyeti:** Bir exception fırlatmak, normal bir metot çağrısından binlerce kat pahalıdır. Doğrulama (validation) için exception değil, sonuç nesnesi veya `TryParse` benzeri desen kullanılır.

```csharp
// Pahalı ve yanlış
try { var sayi = int.Parse(girdi); } catch (FormatException) { sayi = 0; }

// Doğru
if (!int.TryParse(girdi, out var sayi)) sayi = 0;
```

**`finally` ne zaman çalışmaz:** Süreç `Environment.Exit` ile sonlandırılırsa, `StackOverflowException` oluşursa veya süreç dışarıdan öldürülürse `finally` çalışmaz. Kritik temizlikte buna güvenme.

---

## 6. `IDisposable` ve `using`

> **Benzetme —** Kütüphaneden kitap ödünç alırsın. Odanı temizleyen biri olabilir — yerdeki kâğıtları toplar, çöpü atar. Ama o kişi kütüphane kartını bilmez; masandaki kitabı görür, "kitap işte" der, bırakır. Kitabı iade etmek senin işindir. İade etmezsen odan tertemiz kalır ama kütüphanede o kitaptan isteyen başkası bulamaz. Bellek odandır, bağlantı kitaptır.

**Basitçe:** Programın kullandığı bazı şeyler bellekte değildir: açtığın dosya, veritabanı bağlantısı, ağ soketi. Bunları kim ne zaman kapatacak? Otomatik çöp toplayıcı belleği bilir ama bu kaynakları bilmez. Bu yüzden "işim bitti, bırakıyorum" demenin standart bir yolu var: `Dispose`. Ve unutmamak için `using` yazarsın.

**Teknik olarak:** **`IDisposable`** — Yönetilmeyen kaynakların (dosya, bağlantı, soket) **belirli bir anda** bırakılmasını sağlayan arayüz. Tek üyesi: `Dispose()`.

GC yalnızca yönetilen belleği bilir. Bir `SqlConnection` kapatılmazsa bağlantı havuzu tükenir ve uygulama "connection pool exhausted" hatası verir — bellek boldur ama bağlantı yoktur.

```csharp
// using deyimi (klasik)
using (var conn = new SqlConnection(cs))
{
    conn.Open();
}   // burada Dispose() çağrılır

// using bildirimi (C# 8) — kapsam sonunda otomatik
using var conn2 = new SqlConnection(cs);
```

Derleyici bunu `try/finally`'ye çevirir; bu yüzden exception atsa bile `Dispose()` çalışır.

Açığı görmek istersen, `using`'in tam karşılığı şudur:

```csharp
// Bu:
using (var conn = new SqlConnection(cs)) { conn.Open(); }

// Buna derlenir:
var conn = new SqlConnection(cs);
try { conn.Open(); }
finally { if (conn is not null) ((IDisposable)conn).Dispose(); }
```

Birden çok kaynak iç içe yazılmak zorunda değildir:

```csharp
using var dosya = File.OpenRead("veri.csv");
using var okuyucu = new StreamReader(dosya);
// kapsam sonunda ters sırada dispose edilirler: önce okuyucu, sonra dosya
```

**`IAsyncDisposable`** — Asenkron temizlik gerektiren kaynaklar için. `await using` ile kullanılır.

```csharp
await using var conn = new SqlConnection(cs);
await conn.OpenAsync();
// kapsam sonunda DisposeAsync() beklenir — thread bloklanmaz
```

Bir tip hem `IDisposable` hem `IAsyncDisposable` uyguluyorsa, asenkron bağlamda `await using` tercih edilir.

**Dispose pattern** — Hem yönetilen hem yönetilmeyen kaynak tutan sınıflarda uygulanan standart yapı (`Dispose(bool disposing)` + `GC.SuppressFinalize`). Kendi kodunda doğrudan yönetilmeyen kaynak tutmuyorsan gerekmez — ki bootcamp boyunca büyük ihtimalle tutmayacaksın.

Yine de bir kez görmen iyi olur:

```csharp
public class Kaynak : IDisposable
{
    private bool _atildi;
    private StreamReader? _okuyucu;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);    // finalizer varsa çalıştırma, iş bitti
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_atildi) return;          // iki kez çağrılmaya dayanıklı olmalı
        if (disposing)
        {
            _okuyucu?.Dispose();      // yönetilen kaynakları bırak
        }
        // yönetilmeyen handle'lar burada kapatılır
        _atildi = true;
    }
}
```

İki kural buradan çıkar: `Dispose()` **birden çok kez çağrılmaya dayanıklı** olmalı, ve dispose edilmiş bir nesne kullanıldığında `ObjectDisposedException` atmalıdır.

```csharp
private void KontrolEt()
{
    ObjectDisposedException.ThrowIf(_atildi, this);
}
```

**En sık yapılan iki hata:**

```csharp
// 1) HttpClient'ı her istekte using'e almak — soket tükenmesine yol açar
using var http = new HttpClient();      // yanlış: HttpClient uzun ömürlü olmalı
// doğrusu: IHttpClientFactory ile DI'dan al

// 2) using içinde dönen IEnumerable'ı dışarı vermek
IEnumerable<string> Satirlar(string yol)
{
    using var sr = new StreamReader(yol);
    while (sr.ReadLine() is { } satir) yield return satir;
}   // burada dikkat: tüketici foreach'i yarıda bırakırsa dispose yine çalışır,
    // ama hiç foreach yapılmazsa dosya hiç açılmaz — ertelenmiş çalıştırma
```

> ASP.NET Core'da DI konteynerine kayıtlı `IDisposable` servisleri, kapsam (scope) bitince konteyner **otomatik** dispose eder. Elle çağırmaya çalışmak hataya yol açar. Bu konu Hafta 4'te işlenecek.

---

## 7. `object` Sınıfı ve Eşitlik

> **Benzetme —** Kütüphanedeki raf düzenini düşün. Kitap, konu numarasına göre bir rafa konur; aradığında da aynı numaraya gidilir. `GetHashCode` raf numarasını üreten kuraldır, `Equals` ise rafa varınca "aradığım kitap bu mu" sorusudur. Kitabı koyarken bir kural, ararken başka bir kural kullanırsan kitap kütüphanededir ama asla bulunamaz. Kimse de sana "kayıp" demez — sadece yok görünür.

**Basitçe:** C#'ta her nesne, birkaç temel yeteneği doğuştan taşır: metin hâline gelmek, başka bir nesneyle karşılaştırılmak, bir sayı üretmek, kendi tipini söylemek. Bunları değiştirmek istiyorsan bir kural var: eşitliği değiştirdiysen, hash üretimini de değiştirmek zorundasın. Yoksa sözlük ve küme yapıları sessizce yanlış çalışır.

**Teknik olarak:** Her tip `System.Object`'ten türer. Devraldığı metotlar:

| Metot | İş |
|---|---|
| `ToString()` | Metin gösterimi. Override edilmezse tip adını basar |
| `Equals(object)` | Eşitlik. Varsayılan: reference type'ta referans, value type'ta alan karşılaştırması |
| `GetHashCode()` | Hash değeri. `Dictionary` ve `HashSet` bunu kullanır |
| `GetType()` | Çalışma anındaki gerçek tip |

**Altın kural:** `Equals` override ediliyorsa `GetHashCode` da edilmelidir. Aksi hâlde nesne bir `Dictionary`'ye konur ama bir daha bulunamaz — sessiz ve teşhisi zor bir hata.

Hatayı çalışırken gör:

```csharp
public class Kod
{
    public string Deger { get; init; } = "";
    public override bool Equals(object? o) => o is Kod k && k.Deger == Deger;
    // GetHashCode override edilmedi
}

var kume = new HashSet<Kod> { new Kod { Deger = "A" } };
Console.WriteLine(kume.Contains(new Kod { Deger = "A" }));   // büyük ihtimalle False
```

Sebep: iki nesne farklı hash ürettiği için farklı kovalara düşer; `Equals` hiç çağrılmaz bile.

Doğrusu:

```csharp
public sealed class Kod2 : IEquatable<Kod2>
{
    public string Deger { get; init; } = "";

    public bool Equals(Kod2? o) => o is not null && o.Deger == Deger;
    public override bool Equals(object? o) => Equals(o as Kod2);
    public override int GetHashCode() => HashCode.Combine(Deger);
    public override string ToString() => $"Kod({Deger})";

    public static bool operator ==(Kod2? a, Kod2? b) => Equals(a, b);
    public static bool operator !=(Kod2? a, Kod2? b) => !Equals(a, b);
}
```

`HashCode.Combine` birden çok alanı birleştirmenin standart yoludur; elle `hash * 31 + x` yazmaya gerek yoktur.

**Hash sözleşmesi** üç maddedir:

1. İki nesne eşitse hash'leri **mutlaka** aynı olmalı.
2. Hash'leri aynı olan iki nesne eşit olmak zorunda **değildir** (çakışma normaldir).
3. Nesne bir sözlükte dururken hash'i **değişmemelidir**.

Üçüncü madde, hash'i değiştirilebilir (mutable) alanlardan üretmenin neden tehlikeli olduğunu açıklar:

```csharp
var sozluk = new Dictionary<Kod2, int>();
var anahtar = new Kod2 { Deger = "A" };
sozluk[anahtar] = 1;
// anahtar.Deger değiştirilebilseydi, kayıt bulunamaz hâle gelirdi
```

**`==` ile `Equals` farkı:** `==` bir operatördür ve **derleme anındaki** tipe göre çözülür; `Equals` sanaldır ve **çalışma anındaki** tipe göre çalışır. `object` olarak tutulan iki string'i `==` ile karşılaştırmak referans karşılaştırmasına düşebilir.

```csharp
object a = new string("abc".ToCharArray());
object b = new string("abc".ToCharArray());

Console.WriteLine(a == b);              // False — object == object: referans karşılaştırması
Console.WriteLine(a.Equals(b));         // True  — string.Equals çalışır
Console.WriteLine(Equals(a, b));        // True  — statik Equals sanal çağrıya gider
```

Üç eşitlik biçimini ayırt et:

| Yazım | Ne yapar |
|---|---|
| `ReferenceEquals(a, b)` | Aynı nesne mi — her zaman referans karşılaştırması |
| `a.Equals(b)` | Sanal çağrı; `a` null ise patlar |
| `Equals(a, b)` | Statik yardımcı; null'a dayanıklı, sonra sanal çağrı yapar |
| `a == b` | Derleme anındaki tipe göre seçilen operatör |

`record` kullanmanın cazibesi burada: bu üçlünün (`Equals`, `GetHashCode`, `ToString`) tutarlı hâlini derleyici üretir.

`ToString()` override etmek küçük ama çok kazandıran bir alışkanlıktır: log kayıtlarında, hata ayıklayıcıda ve test çıktılarında nesnenin ne olduğunu görürsün. Override edilmezse yalnızca tip adını okursun.

---

## 8. Interface'in Dil Düzeyindeki Rolü

> **Benzetme —** Bir iş ilanı, kişiyi tarif etmez; **yapabileceklerini** tarif eder. "B sınıfı ehliyetli, hafta sonu çalışabilen" der. Kimin geldiği önemli değildir, şartları karşılaması yeterlidir. İşveren ilanı yazarken hangi kişinin geleceğini bilmez ve bilmek zorunda da değildir. Interface bu ilandır: kodun geri kalanı "şunu yapabilen biri" ister, kimin geldiğiyle ilgilenmez.

**Basitçe:** Interface, bir tipin hangi işleri yapabileceğini listeleyen ama nasıl yaptığını hiç söylemeyen bir sözleşmedir. Kodun bir sınıfa değil bu sözleşmeye bağlanması, o sınıfı sonradan değiştirebilmeni sağlar — test ederken sahtesini koyabilir, üretimde gerçeğini kullanabilirsin.

**Teknik olarak:** **Interface (arayüz)** — Bir tipin **ne yapabileceğini** tanımlayan, nasıl yaptığını söylemeyen sözleşme.

| | `abstract class` | `interface` |
|---|---|---|
| Çoklu kalıtım | Hayır, tek taban sınıf | **Evet**, birden çok arayüz |
| Alan (field) tutabilir mi | Evet | Hayır |
| Constructor | Var | Yok |
| Ortak davranış paylaşımı | Doğal yeri | C# 8'den beri varsayılan gövde mümkün ama istisnai |
| İfade ettiği ilişki | "**-dır**" (Kedi bir Hayvan**dır**) | "**-ebilir**" (Dosya kaydedil**ebilir**) |

Interface'in asıl değeri **bağımlılığı tersine çevirmektir**: sınıfın somut bir sınıfa değil, bir sözleşmeye bağlı olması. Test ederken sahte (mock) uygulama geçebilmenin, DI konteynerinin çalışabilmesinin ve katmanların birbirinden ayrılabilmesinin sebebi budur.

Farkı tek örnekte gör:

```csharp
// Bağımlı — test edilemez, değiştirilemez
public class SiparisService
{
    private readonly SqlMusteriRepository _repo = new();   // somut sınıfa çivilendi
}

// Bağımsız — dışarıdan verilir
public class SiparisService2(IMusteriRepository repo)
{
    public Task<Musteri?> BulAsync(int id) => repo.GetByIdAsync(id);
}

// Testte sahte uygulama
public class SahteRepo : IMusteriRepository
{
    public Task<Musteri?> GetByIdAsync(int id) => Task.FromResult<Musteri?>(new Musteri { Id = id });
    // ...
}
```

**Açık (explicit) uygulama** — İki arayüz aynı adda üye istiyorsa ya da bir üyeyi sınıfın genel yüzünden gizlemek istiyorsan kullanılır:

```csharp
public class Cift : IYazici, IOkuyucu
{
    void IYazici.Calistir()  => Console.WriteLine("yaz");
    void IOkuyucu.Calistir() => Console.WriteLine("oku");
}

var c = new Cift();
// c.Calistir();                 // derlenmez: sınıfın kendi üyesi yok
((IYazici)c).Calistir();         // yaz
```

**Varsayılan arayüz metodu** (C# 8) — Arayüze gövdeli bir metot eklenebilir. Amacı yeni özellik eklemek değil, **mevcut uygulamaları bozmadan** arayüze üye ekleyebilmektir.

```csharp
public interface ILog
{
    void Yaz(string seviye, string mesaj);

    void Bilgi(string mesaj) => Yaz("INFO", mesaj);   // uygulayan sınıflar yazmak zorunda değil
}
```

> Varsayılan metotlar sınıfın kendi üyesi olmaz; yalnızca arayüz referansı üzerinden çağrılabilir. Bu yüzden "C# artık çoklu kalıtımı destekliyor" demek yanlıştır — durum (state) hâlâ paylaşılamaz.

**Interface mi, abstract class mı** sorusunun pratik ölçüsü: paylaşılacak olan **durum ve ortak kod** varsa abstract class, yalnızca **sözleşme** varsa interface. İkisi birlikte de kullanılır — arayüz sözleşmeyi, abstract sınıf ortak iskeleti verir.

```csharp
public interface IRapor { string Uret(); }

public abstract class RaporTemel : IRapor
{
    protected abstract string Govde();
    public string Uret() => $"{Baslik()}\n{Govde()}";     // ortak iskelet
    protected virtual string Baslik() => "Rapor";
}
```

Arayüzleri küçük tutmak da bir kuraldır: bir arayüzde yirmi üye varsa, onu uygulayan her sınıf yirmisini de yazmak zorunda kalır ve çoğunu `NotImplementedException` ile geçer. Bu, Hafta 2'de göreceğin **Interface Segregation** ilkesidir.

Bu konu Hafta 2'de SOLID'in D maddesi (Dependency Inversion) olarak derinleşecek.

---

## Tek Bakışta Özet

- **Generic** tip güvenliği + boxing'siz performans sağlar; **kısıt**, `T` hakkında derleyiciye verilen bilgidir.
- .NET generic'leri çalışma anında silinmez; `List<int>` ile `List<string>` gerçekten farklı tiplerdir.
- **Delegate** metoda işaret eden tiptir; `Action` döndürmez, `Func` döndürür, `Predicate` `bool` döner. `Func`'ta **son** tip parametresi dönüş tipidir.
- **Event**, delegate'in kapsüllenmiş hâlidir: dışarıdan yalnızca abone olunur; `-=` unutulursa bellek sızar.
- **Lambda** isimsiz metottur; **closure** dış değişkenin **kendisini** yakalar ve ömrünü uzatır. `for` döngüsünde kopya al.
- `Func<T,bool>` çalıştırılabilir koddur, `Expression<Func<T,bool>>` kodun veri hâlidir — EF Core ikincisini SQL'e çevirir.
- **Extension method** tipe bir şey eklemez, çağrıyı statik metoda çevirir. LINQ'in tamamı budur; bu yüzden `null` üzerinde de çağrılabilir.
- **`throw ex` stack trace'i siler; `throw` korur.** Zenginleştireceksen orijinali `InnerException` olarak taşı.
- Boş `catch` bloğu ve akış kontrolü amaçlı exception, en pahalı iki alışkanlıktır. Doğrulamada `TryParse` desenini kullan.
- **`IDisposable` + `using`**, GC'nin göremediği kaynaklar için deterministik temizliktir; `using` bir `try/finally`'dir.
- `Equals` override ediyorsan `GetHashCode`'u da et — ve hash'i değişmeyen alanlardan üret.
- **Interface** "-ebilir" ilişkisidir ve bağımlılığı tersine çevirmenin aracıdır; küçük tutulur.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Generic | Tipi kullanım anında belirlenen sınıf/metot |
| Type parameter | `<T>` içindeki yer tutucu tip |
| Constraint | `T`'nin ne olabileceğini sınırlayan kural |
| Boxing / Unboxing | Value type'ın heap'e sarılması / geri çıkarılması |
| Reified generics | Generic tip bilgisinin çalışma anında da korunması |
| Covariance / Contravariance | Generic tiplerde okuma / yazma yönünde tip esnekliği |
| Invariant | Ne covariant ne contravariant olabilen generic tip (`List<T>`) |
| Delegate | Metoda işaret eden tip |
| Multicast delegate | Birden çok metot tutabilen delegate |
| Event | Dışarıdan yalnızca abone olunabilen kapsüllenmiş delegate |
| Action / Func / Predicate | Döndürmeyen / döndüren / `bool` döndüren hazır delegate'ler |
| Metot grubu dönüşümü | Bir metot adının doğrudan delegate'e atanması |
| Lambda | İsimsiz metot sözdizimi |
| Closure | Lambda'nın dış kapsamdaki değişkeni yakalaması |
| Static lambda | Dış değişken yakalaması derleyici tarafından engellenen lambda |
| Expression tree | Kodun çalıştırılabilir hâli yerine veri hâli (`Expression<Func<...>>`) |
| Extension method | `this` parametresiyle tipe eklenmiş gibi görünen statik metot |
| Exception | Olağandışı durumu temsil eden nesne |
| Stack trace | Hatanın hangi çağrı zincirinden geldiğini gösteren döküm |
| Inner exception | Bir hataya sebep olan alttaki hata |
| Exception filter (`when`) | Yakalamaya koşul ekleyen, stack'i çözmeden değerlendirilen sözdizimi |
| Try-desen (`TryParse`) | Hata yerine `bool` döndüren, exception maliyetinden kaçınan kalıp |
| IDisposable | Deterministik kaynak temizliği arayüzü |
| IAsyncDisposable | Asenkron temizlik arayüzü (`await using`) |
| using declaration | Kapsam sonunda otomatik `Dispose` çağıran bildirim |
| Dispose pattern | Yönetilen/yönetilmeyen kaynakları ayıran standart temizlik yapısı |
| Hash sözleşmesi | Eşit nesnelerin aynı hash'i üretmesi kuralı |
| Interface | "Ne yapabilir" sözleşmesi |
| Explicit implementation | Arayüz üyesinin yalnızca arayüz üzerinden erişilebilir uygulanması |
| Default interface method | Arayüzde gövdeli tanımlanan, isteğe bağlı olarak ezilen üye |
| Abstract class | Kısmen uygulanmış, örneklenemeyen taban sınıf |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Generic sadece kod tekrarını azaltır" | Asıl kazanç tip güvenliği ve boxing'in ortadan kalkması |
| "Generic tip bilgisi çalışma anında silinir" | .NET'te silinmez; Java ile karıştırılıyor |
| "`List<Kopek>` bir `List<Hayvan>`dır" | Değildir; yazma yönü güvenliği bozardı. `IEnumerable<T>` covariant'tır |
| "`Predicate<T>` ile `Func<T,bool>` aynı tiptir" | İmzaları aynı, tipleri farklı; birbirine atanamaz |
| "`Func<int, string>` iki değer döndürür" | Son tip parametresi dönüş tipidir: `int` alır, `string` döner |
| "Lambda değişkenin değerini kopyalar" | Değişkenin kendisini yakalar; sonradan değişirse lambda yeni değeri görür |
| "`for` döngüsünde lambda yakalaması güvenlidir" | Değildir; `foreach` güvenli, `for` için kopya almalısın |
| "Extension method tipe metot ekler" | Eklemez; derleyici çağrıyı statik metoda çevirir |
| "Extension method `null` üzerinde patlar" | Patlamaz; aslında statik bir çağrıdır, `null` argüman olarak geçer |
| "`throw ex` ile `throw` aynı" | `throw ex` stack trace'i sıfırlar |
| "`catch` içinde `if` ile `when` aynı şey" | `when` stack çözülmeden değerlendirilir; `catch`+`if` çözüldükten sonra |
| "`finally` her durumda çalışır" | Neredeyse — süreç `Environment.Exit` ile sonlanırsa çalışmaz |
| "`Dispose()` nesneyi bellekten siler" | Kaynağı bırakır; belleği yine GC temizler |
| "Her `IDisposable` nesne `using` ile sarılmalı" | `HttpClient` gibi uzun ömürlü olması gerekenler istisnadır |
| "`Equals` yeterli, `GetHashCode` süs" | Hash yanlışsa nesne sözlükte bulunamaz — sessiz hata |
| "Interface ile abstract class birbirinin yerine geçer" | Ortak **durum** paylaşılacaksa abstract class, sadece sözleşme ise interface |
| "Varsayılan arayüz metotları çoklu kalıtım demektir" | Değildir; durum (field) hâlâ paylaşılamaz |

---

## Sonraki

→ `07-Hafta-Ozeti.md` (Pazar)
