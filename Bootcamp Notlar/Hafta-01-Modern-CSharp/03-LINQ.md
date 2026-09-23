# Hafta 1 · Çarşamba — LINQ

**Okuma süresi:** ~52 dk
**Neden bu konu:** LINQ, .NET'te veri işlemenin ortak dilidir. Listeyi de veritabanını da aynı sözdizimiyle sorgularsın. Ama tam da bu benzerlik yüzünden, bellekte masum olan bir ifade veritabanında felakete dönüşebilir. Ayrım burada öğrenilir.

---

## Önce Basitçe

Yazdığın her programda aynı dört soruyu sorarsın: "Bunlardan hangileri şu şarta uyuyor?", "Bana sadece şu bilgilerini ver", "Şuna göre sırala", "Kaç tane var?". Bu soruları listeye de sorarsın, veritabanına da, bir dosyadan okuduğun kayıtlara da. LINQ, bu dört soruyu her yere **aynı cümlelerle** sorabilmenin adıdır.

Eskiden her kaynağın kendi dili vardı. Listeyi `for` döngüsüyle tarardın, veritabanına SQL yazardın, XML'e başka bir şey. Üç ayrı dil, üç ayrı hata yapma biçimi. LINQ bunların üstüne tek bir konuşma biçimi koydu. Sen "fiyatı 100'den büyük olanları, adına göre sıralı ver" dersin; arkadaki çevirmen bu cümleyi kaynağın anlayacağı dile döker. Listeyse döngüye, veritabanıysa SQL'e.

Bu kolaylığın bir bedeli var ve notun yarısı o bedeli anlatıyor. Aynı cümleyi iki farklı yere söylediğinde, cümle aynı görünse bile arka planda olanlar tamamen farklı olabilir. Bellekteki listede masum duran bir ifade, veritabanında "önce bütün tabloyu getir, sonra ayıkla" anlamına gelebilir. Bin satırlık tabloda fark etmezsin, on milyon satırlıkta uygulama durur.

İkinci önemli nokta zamanlamadır. LINQ'te bir sorgu yazdığın anda hiçbir şey olmaz. Cümle kurulmuştur ama kimse harekete geçmemiştir. İş, sen sonucu gerçekten istediğin anda yapılır — ve her istediğinde baştan yapılır. Bu davranış, LINQ'te yapılan hataların büyük çoğunluğunun kaynağıdır. Bir kere anladığında ise geri kalan her şey yerine oturur.

Üçüncüsü, LINQ'in operatörleri tek tek küçük ve sıkıcıdır: filtrele, dönüştür, sırala, grupla, say. Güç, bunları arka arkaya dizmekten gelir. Her operatör bir öncekinin çıktısını alır, üstüne kendi işini ekler ve devreder. Tek satırda beş iş yapan bir zincir yazabilirsin; önemli olan o zincirin hangi noktada gerçekten çalıştığını bilmektir.

Bu notta önce dilin kendisini, sonra zamanlama davranışını, ardından operatörleri sırayla göreceksin. En sonda da en çok kafa karıştıran konu var: bellekteki LINQ ile veritabanındaki LINQ'in aynı görünüp farklı davranması. Şimdi detaya iniyoruz.

> **Ana benzetme:** LINQ, elindeki isteği yazdığın bir **dilekçe**dir. Aynı dilekçeyi muhtara da verirsin, bankaya da, kargo şirketine de; metni değişmez. Arada bir memur oturur ve dilekçeni o kurumun formatına çevirir. Dilekçeyi yazmak işi bitirmez — bir yetkiliye teslim edene kadar kâğıt masada durur. Ve aynı dilekçenin üç kopyasını üç ayrı gişeye verirsen, iş üç kez yapılır.

---

## Bu Notta Ne Var

1. LINQ nedir, iki sözdizimi
2. Ertelenmiş çalıştırma (deferred execution)
3. Filtreleme, projeksiyon, sıralama
4. Gruplama ve `GroupBy`'ın gerçek dönüş tipi
5. Birleştirme: `Join`, `GroupJoin`, `SelectMany`
6. Toplama (aggregation) operatörleri
7. Eleman ve niceleyici operatörleri
8. Küme operatörleri ve sayfalama
9. Yürütmeyi zorlayan operatörler
10. LINQ to Objects ve LINQ to Entities farkı

---

## 1. LINQ Nedir

> **Benzetme —** Bir mahallede üç yer var: evindeki dolap, mahalle bakkalı ve ilçedeki büyük depo. Üçünden de "elimde şu listeye uyan ne varsa lazım" diye isteyebilirsin. Dolapta kendin bakarsın, bakkalda bakkala söylersin, depoda bir form doldurursun. LINQ, üç durumda da **senin ağzından çıkan cümlenin aynı kalmasını** sağlar. Cümleyi yerine göre eyleme çeviren kişi, arkadaki sağlayıcıdır.

**Basitçe:** LINQ, veriyi sorgulamanın C#'a gömülmüş hâlidir. Nereden geldiğine bakmadan — bellekteki bir liste, veritabanındaki bir tablo, bir XML dosyası — aynı metot isimleriyle "filtrele, dönüştür, sırala, say" diyebilirsin. Arkada, o kaynağa özel bir çevirmen senin isteğini o kaynağın diline aktarır.

**Teknik olarak:** **LINQ (Language Integrated Query)** — Farklı veri kaynaklarını (nesne koleksiyonları, veritabanı, XML, JSON) **tek bir sorgulama sözdizimiyle** işlemeyi sağlayan dil özelliği.

Sağlayıcıya (provider) göre isimlendirilir:

| Sağlayıcı | Kaynak |
|---|---|
| **LINQ to Objects** | Bellekteki koleksiyonlar (`IEnumerable<T>`) |
| **LINQ to Entities** | EF Core üzerinden veritabanı (`IQueryable<T>`) |
| **LINQ to XML** | XML belgeleri |

**Provider (sağlayıcı)** — Senin yazdığın LINQ ifadesini belirli bir kaynağın anlayacağı eyleme çeviren bileşendir. LINQ to Objects'te bu çeviri basittir: ifade doğrudan bir döngüye dönüşür. LINQ to Entities'te ise ifade önce bir **ağaç** olarak incelenir, sonra SQL metnine çevrilir. 10. bölümdeki farkların kaynağı budur.

### İki sözdizimi

```csharp
// Method syntax (fluent) — yaygın kullanım
var sonuc = urunler
    .Where(u => u.Fiyat > 100)
    .OrderBy(u => u.Ad)
    .Select(u => u.Ad);

// Query syntax (SQL benzeri)
var sonuc2 = from u in urunler
             where u.Fiyat > 100
             orderby u.Ad
             select u.Ad;
```

İkisi de derlemede aynı koda dönüşür. Query syntax bazı `join` ve `group by` senaryolarında daha okunaklıdır; ama tüm operatörlerin karşılığı yoktur (`Count`, `Any` gibi). Pratikte method syntax hâkimdir.

Query syntax'ın tek gerçek üstünlüğü `let` anahtar kelimesidir: ara bir değeri isimlendirip birden çok yerde kullanmanı sağlar.

```csharp
// Query syntax — ara değer bir kez hesaplanıp isimlendiriliyor
var sonuc3 = from u in urunler
             let kdvli = u.Fiyat * 1.20m
             where kdvli > 500
             orderby kdvli descending
             select new { u.Ad, kdvli };

// Method syntax karşılığı daha dolambaçlı
var sonuc4 = urunler
    .Select(u => new { Urun = u, Kdvli = u.Fiyat * 1.20m })
    .Where(x => x.Kdvli > 500)
    .OrderByDescending(x => x.Kdvli)
    .Select(x => new { x.Urun.Ad, x.Kdvli });
```

İki sözdizimi karıştırılabilir de: query syntax'ı parantezle sarıp sonuna method çağırabilirsin.

```csharp
var adet = (from u in urunler where u.Fiyat > 100 select u).Count();
```

**Extension method** — LINQ operatörleri aslında `System.Linq.Enumerable` sınıfındaki genişletme metotlarıdır. `urunler.Where(...)` yazabilmenin sebebi `using System.Linq;` satırıdır. Bu mekanizma Cumartesi notunda anlatılıyor.

**Lambda ifadesi** — `u => u.Fiyat > 100` yazımı, "bana bir ürün ver, sana doğru/yanlış döndüreyim" diyen isimsiz bir metottur. `u` parametrenin adıdır, sen koyarsın; `=>` sağındaki ise gövdedir. LINQ operatörlerinin neredeyse hepsi böyle bir lambda alır.

> **Bu benzetme şurada bozulur:** Dolap, bakkal ve depo örneğinde cümlen aynı kalsa da her yerde **aynı şeyi** alacağını varsaydık. Gerçekte depodaki memur bazı isteklerini "bu formda böyle bir madde yok" diye reddeder. LINQ'te de bellekte çalışan bir ifade veritabanında çevrilemeyebilir. Cümle aynıdır, kabul edilirliği değil.

---

## 2. Ertelenmiş Çalıştırma — LINQ'in En Kritik Davranışı

> **Benzetme —** Bir yemek tarifi yazmakla yemeği pişirmek aynı şey değildir. Tarifi kâğıda dökersin, mutfak hâlâ soğuktur. Ocağı ancak "hadi bunu yapalım" dediğinde yakarsın. Dahası: aynı tarifi üç kez uygularsan üç kez pişirmiş olursun, kâğıt sana hazır yemeği hatırlamaz. Bir de şu var: tarifte "buzdolabındaki domatesler" yazıyorsa, pişirdiğin anda dolapta ne varsa onu kullanırsın — tarifi yazdığın andaki hâlini değil.

**Basitçe:** LINQ sorgusu yazdığın anda hiçbir şey olmaz. Sorgu, "ne yapılacağının yazılı hâli"dir. Asıl iş, sen sonucu istediğin anda — `foreach` ile gezdiğinde ya da `ToList()` dediğinde — yapılır. Ve her istediğinde baştan yapılır.

**Teknik olarak:** **Deferred execution (ertelenmiş çalıştırma)** — Sorgu tanımlandığında **çalışmaz**; sonucu ilk kez istendiğinde çalışır.

```csharp
var sorgu = sayilar.Where(x => x > 5);   // burada HİÇBİR ŞEY olmadı
sayilar.Add(100);                        // listeye ekleme yapıldı
foreach (var s in sorgu) { ... }         // sorgu ŞİMDİ çalışır — 100 de dahil
```

Sorgu bir **tarif**tir, sonuç değil. Tarif her okunduğunda yeniden pişirilir:

```csharp
var pahaliUrunler = urunler.Where(u => u.Fiyat > 1000);

var adet  = pahaliUrunler.Count();   // 1. tarama
var ilk   = pahaliUrunler.First();   // 2. tarama
var liste = pahaliUrunler.ToList();  // 3. tarama
```

Aynı filtre üç kez çalıştı. Veritabanı söz konusuysa bu üç ayrı SQL sorgusu demektir.

**Çözüm:** Sonucu birden çok kez kullanacaksan bir kez maddeleştir (`ToList()`), sonra listeyle çalış.

```csharp
// Doğru kalıp: bir kez çalıştır, sonra listeyle çalış
var pahaliListe = urunler.Where(u => u.Fiyat > 1000).ToList();

var adet2  = pahaliListe.Count;      // artık alan okuması, tarama değil
var ilk2   = pahaliListe[0];
```

### Kapanış (closure) tuzağı

Ertelenmiş çalıştırmanın az bilinen yan etkisi: sorgu içinde kullandığın **değişken**, sorgu çalıştığı andaki değeriyle okunur.

```csharp
int esik = 100;
var sorgu2 = urunler.Where(u => u.Fiyat > esik);

esik = 5000;                     // değişkeni sonradan değiştirdik
var sonuc = sorgu2.ToList();     // filtre 5000'e göre çalışır, 100'e göre değil
```

Lambda, `esik` değişkeninin değerini değil **kendisini** yakalar. Buna **closure (kapanış)** denir. Beklenmedik sonuç aldığında ilk bakılacak yerlerden biridir.

**Streaming vs buffering operatörler**
- *Streaming*: `Where`, `Select`, `Take`, `Skip` — elemanları tek tek geçirir, tümünü belleğe almaz.
- *Buffering*: `OrderBy`, `GroupBy`, `Reverse`, `ToList` — sonucu üretmek için **tüm veriyi görmek zorundadır**. Bu yüzden zincirin ortasındaki bir `OrderBy` tembelliği kırar.

Bunun pratik sonucu şudur: `Take(10)` yazdığın hâlde zincirin başında bir `OrderBy` varsa, on eleman için bile tüm koleksiyon gezilir. Sıralamak için en küçüğü bulmak gerekir, en küçüğü bulmak için de hepsine bakmak gerekir.

```csharp
// Streaming: ilk 10 uygun elemanı bulunca durur
var ilk10 = kayitlar.Where(k => k.Aktif).Take(10);

// Buffering: sıralamak için TÜM koleksiyonu gezer, sonra 10 alır
var ilk10Sirali = kayitlar.Where(k => k.Aktif).OrderBy(k => k.Ad).Take(10);
```

> **Bu benzetme şurada bozulur:** Tarif benzetmesi, sorgunun "her seferinde baştan çalıştığını" iyi anlatır ama bir noktayı kaçırır. Yemek tarifini iki kez uygularsan iki tabak yemeğin olur; LINQ sorgusunu iki kez çalıştırırsan iki ayrı **sonuç kümen** olmaz — aynı kaynağı iki kez taramış olursun. Kaynak arada değiştiyse iki tarama farklı sonuç verir. Tarif değil, **canlı yayın** gibi düşün: her açtığında o anki hâli görürsün.

---

## 3. Filtreleme, Projeksiyon, Sıralama

> **Benzetme —** Hâlden gelen kasa kasa sebzeyi düşün. Önce **ayıklarsın**: çürükleri kenara, sağlamları öne. Sonra **paketlersin**: müşteri bütün domatesi değil, yarım kilo doğranmışını istiyor — elindeki malı onun istediği şekle sokarsın. En son **dizersin**: tezgâha büyükten küçüğe. Ayıklamak `Where`, paketlemek `Select`, dizmek `OrderBy`'dır. Üçünün sırası da önemlidir: önce ayıkla, boşuna paketleme.

**Basitçe:** Bu üçü LINQ'in günlük ekmeğidir. `Where` istemediklerini eler, `Select` kalanları başka bir şekle sokar, `OrderBy` sonucu düzene koyar. Yazacağın sorguların büyük çoğunluğu bu üçünün bir dizilişidir.

**Teknik olarak:** Üçü de streaming operatör ailesindendir (`OrderBy` hariç — o tüm veriyi görmek zorundadır) ve zincirlenebilir.

### `Where` — filtreleme
Şarta uyan elemanları geçirir. Zincirlenebilir; `Where(a).Where(b)` ile `Where(a && b)` aynı sonucu verir.

```csharp
// İki ayrı Where ile tek Where aynı sonucu verir
var a = urunler.Where(u => u.Aktif).Where(u => u.Fiyat > 100);
var b = urunler.Where(u => u.Aktif && u.Fiyat > 100);
```

Ayrı yazmak, şartları koşullu olarak eklemek gerektiğinde işe yarar:

```csharp
// Dinamik filtre kurma — sorgu henüz çalışmadığı için parça parça eklenebilir
IQueryable<Urun> sorgu = context.Urunler;

if (!string.IsNullOrEmpty(aranan))
    sorgu = sorgu.Where(u => u.Ad.Contains(aranan));

if (minFiyat.HasValue)
    sorgu = sorgu.Where(u => u.Fiyat >= minFiyat.Value);

var sonuc = sorgu.ToList();      // TEK sorgu, tüm şartlar birleşmiş hâlde
```

Bu kalıp, ertelenmiş çalıştırmanın en faydalı kullanımıdır: filtreleri topla, en sonda bir kez çalıştır.

`Where`'in indeksli bir aşırı yüklemesi de vardır (yalnızca LINQ to Objects'te):

```csharp
var ciftIndeksliler = liste.Where((eleman, indeks) => indeks % 2 == 0);
```

### `Select` — projeksiyon
**Projeksiyon** — Her elemanı başka bir şeye dönüştürmek. Sayı sayısını değiştirmez, **şeklini** değiştirir.

```csharp
// Sadece gereken alanları çek — EF Core'da tablo genişliğini düşürür
var ozet = urunler.Select(u => new { u.Ad, u.Fiyat });

// DTO'ya dönüştürme
var dtolar = urunler.Select(u => new UrunDto { Ad = u.Ad, Fiyat = u.Fiyat });

// Hesaplanmış alan ekleme
var kdvli = urunler.Select(u => new
{
    u.Ad,
    Net    = u.Fiyat,
    Brut   = Math.Round(u.Fiyat * 1.20m, 2)
});
```

**Anonim tip** — `new { u.Ad, u.Fiyat }` ifadesi derleyicinin ürettiği isimsiz bir tiptir. Metot dışına çıkarılamaz; sadece yerel kullanım içindir. Metot sınırını geçecekse bir DTO sınıfı ya da `record` tanımlamalısın.

> EF Core'da `Select` ile sadece gereken kolonları istemek, en ucuz performans kazancıdır. 40 kolonlu bir tablodan 3 kolon çekmek ile hepsini çekmek arasında ciddi fark vardır.

`Select`'in de indeksli hâli vardır:

```csharp
var numarali = adlar.Select((ad, i) => $"{i + 1}. {ad}");
```

### `OrderBy` / `ThenBy`
```csharp
urunler.OrderBy(u => u.Kategori).ThenByDescending(u => u.Fiyat)
```
İkinci kriter için `OrderBy`'ı tekrar çağırmak **yanlıştır** — birinci sıralamayı iptal eder. İkinci ve sonraki kriterler `ThenBy` ile verilir.

```csharp
// YANLIŞ — Kategori sıralaması tamamen kayboldu, sadece fiyata göre sıralı
urunler.OrderBy(u => u.Kategori).OrderBy(u => u.Fiyat);

// DOĞRU — önce kategori, aynı kategori içinde fiyat
urunler.OrderBy(u => u.Kategori).ThenBy(u => u.Fiyat);
```

LINQ to Objects'te `OrderBy` **kararlıdır (stable)**: sıralama anahtarı eşit olan elemanların kendi aralarındaki sırası bozulmaz. Veritabanında böyle bir garanti **yoktur** — eşitlik durumunda sıra değişebilir. Bu yüzden sayfalamada her zaman benzersiz bir alanı (`Id` gibi) son kriter olarak eklemek iyi alışkanlıktır.

```csharp
.OrderBy(u => u.Kategori).ThenBy(u => u.Id)   // Id ile sıra kesinleşir
```

> **Bu benzetme şurada bozulur:** Sebze örneğinde "önce ayıkla, sonra paketle" mantıklı geliyor ve LINQ to Objects'te sıra gerçekten performansı etkiler. Ama EF Core'da sen ne sırayla yazarsan yaz, SQL'i veritabanının sorgu planlayıcısı düzenler. Orada zincirin sırası okunabilirlik meselesidir, hız meselesi değil.

---

## 4. Gruplama

> **Benzetme —** Kargo şubesine akşam üstü iki yüz koli geliyor. Görevli bunları tek tek okuyup ilçe ilçe ayrılmış rafların önüne koyar: Kadıköy rafı, Üsküdar rafı, Beşiktaş rafı. İş bittiğinde elinde iki yüz koli değil, **üç raf** vardır. Ama her raf boş bir etiket değildir; hem üzerinde ilçe adı yazar hem de içinde o ilçeye ait koliler durur. `GroupBy`'ın döndürdüğü şey tam olarak budur: adı olan bir raf.

**Basitçe:** `GroupBy`, elemanları ortak bir özelliğe göre öbeklere ayırır. Sonuçta elde ettiğin her öbeğin iki yüzü vardır: bir adı (`Key`) ve içindeki elemanlar. Bu iki yüzlülük ilk başta kafa karıştırır; "grup" hem bir etikettir hem de bir listedir.

**Teknik olarak:** **`GroupBy`** — Elemanları bir anahtara göre kümelere ayırır. Dönüş tipi kafa karıştırıcıdır: `IEnumerable<IGrouping<TKey, TElement>>`.

**`IGrouping<TKey,TElement>`** — Hem bir `Key` özelliğine sahip, hem de kendisi o gruptaki elemanların koleksiyonu olan yapı. Yani "anahtarı olan liste". Tanımı şuna benzer:

```csharp
// Kabaca böyle bir şey
public interface IGrouping<out TKey, out TElement> : IEnumerable<TElement>
{
    TKey Key { get; }
}
```

`IEnumerable<TElement>`'ten türediği için `foreach` ile gezilebilir; `Key` özelliği de üstüne eklenmiştir.

```csharp
var kategoriBazli = urunler.GroupBy(u => u.Kategori);

foreach (var grup in kategoriBazli)
{
    Console.WriteLine($"{grup.Key}: {grup.Count()} ürün");
    foreach (var urun in grup)         // grup'un kendisi gezilebilir
        Console.WriteLine($"  - {urun.Ad}");
}
```

Genelde gruplamanın hemen ardından bir özet istenir:

```csharp
var ozet = urunler
    .GroupBy(u => u.Kategori)
    .Select(g => new
    {
        Kategori   = g.Key,
        Adet       = g.Count(),
        Ortalama   = g.Average(u => u.Fiyat),
        EnPahali   = g.Max(u => u.Fiyat)
    });
```

Bu, SQL'deki `GROUP BY ... HAVING` mantığının karşılığıdır. `HAVING` yerine gruplamadan sonra `Where` kullanılır:

```csharp
.GroupBy(u => u.Kategori)
.Where(g => g.Count() > 5)          // HAVING COUNT(*) > 5
```

### Birden çok alana göre gruplama

Anahtar tek bir alan olmak zorunda değildir; anonim tip ya da tuple ile bileşik anahtar kurabilirsin.

```csharp
var yillikOzet = siparisler
    .GroupBy(s => new { s.Yil, s.Kategori })
    .Select(g => new
    {
        g.Key.Yil,
        g.Key.Kategori,
        Toplam = g.Sum(s => s.Tutar)
    });
```

### Eleman seçicili aşırı yükleme

`GroupBy`, gruba hangi elemanların gireceğini de belirlemene izin verir:

```csharp
// Gruba tüm ürün değil, sadece adları girsin
var adlar = urunler.GroupBy(u => u.Kategori, u => u.Ad);

foreach (var grup in adlar)
    Console.WriteLine($"{grup.Key}: {string.Join(", ", grup)}");
```

**`ToLookup`** — `GroupBy`'ın anında çalışan (eager) hâli. Sonucu bir kez üretip tekrar tekrar kullanacaksan tercih edilir.

`ToLookup` ile `ToDictionary` arasındaki fark önemlidir: `ToDictionary` her anahtar için **tek** değer tutar, aynı anahtar iki kez gelirse hata verir. `ToLookup` ise her anahtar için bir **liste** tutar ve olmayan anahtarı sorduğunda hata yerine boş liste döndürür.

```csharp
var lookup = urunler.ToLookup(u => u.Kategori);
var bosGrup = lookup["OlmayanKategori"];    // hata yok, boş koleksiyon döner

var sozluk = urunler.ToDictionary(u => u.Kod);   // Kod benzersiz değilse patlar
```

> **Bu benzetme şurada bozulur:** Kargo rafı benzetmesinde raflar önceden vardır, sen sadece kolileri dağıtırsın. `GroupBy`'da ise gruplar **veriden doğar** — hangi kategoriler varsa o kadar grup oluşur, sen önceden bilmezsin. Ayrıca EF Core'da gruplama veritabanında yapılır ve orada "grubun içindeki tüm elemanlar" her zaman getirilmez; çoğu zaman sadece özet değerler (`Count`, `Sum`) hesaplanır. Rafın içine bakmak isteyeceksen ayrı bir sorgu gerekebilir.

---

## 5. Birleştirme Operatörleri

> **Benzetme —** Elinde iki defter var: birinde sipariş numaraları ve müşteri numaraları yazıyor, diğerinde müşteri numaraları ve adlar. "Hangi siparişi kim verdi" sorusunun cevabı hiçbir defterde tek başına yok. İki defteri yan yana koyup müşteri numarasından eşleştirmen gerekiyor. `Join` tam olarak bu eşleştirmedir. Eşleşmeyen satırı atarsan `Join`, "eşleşmese de sol defteri koru" dersen `GroupJoin` olur.

**Basitçe:** Birleştirme operatörleri, iki ayrı koleksiyonu ortak bir alan üzerinden birbirine bağlar. `Join` sadece eşleşenleri verir, `GroupJoin` soldakini her hâlükârda korur, `SelectMany` ise iç içe listeleri tek listeye indirir.

**Teknik olarak:** Üçü de SQL'deki `JOIN` ailesinin karşılığıdır ama farklı işler görürler.

### `Join` — inner join
İki koleksiyonu eşleşen anahtarlar üzerinden birleştirir. Eşleşmeyen kaydı **atar**.

```csharp
var sonuc = siparisler.Join(
    musteriler,
    s => s.MusteriId,        // dış anahtar
    m => m.Id,               // iç anahtar
    (s, m) => new { s.SiparisNo, m.Ad });
```

Dört parametrenin sırası ilk başta karışır. Okuma sırası şudur: "şu koleksiyonla birleştir, soldan şu anahtarı al, sağdan şu anahtarı al, eşleşince şunu üret."

Query syntax'ta aynı iş daha okunaklıdır:

```csharp
var sonuc2 = from s in siparisler
             join m in musteriler on s.MusteriId equals m.Id
             select new { s.SiparisNo, m.Ad };
```

### `GroupJoin` — left join temeli
Soldaki her eleman için sağdakilerden eşleşen bir **grup** üretir; eşleşme yoksa boş grup gelir. `DefaultIfEmpty` ile birleştirilince SQL'deki `LEFT JOIN` elde edilir.

```csharp
// Her müşteri listede kalır; siparişi yoksa Siparis null olur
var solBirlesim = from m in musteriler
                  join s in siparisler on m.Id equals s.MusteriId into grup
                  from s in grup.DefaultIfEmpty()
                  select new { m.Ad, SiparisNo = s == null ? "-" : s.SiparisNo };
```

`into grup` kısmı `GroupJoin`'i, ardından gelen `from ... DefaultIfEmpty()` ise düzleştirmeyi yapar. Bu iki adımın birleşimi `LEFT JOIN`'in LINQ'teki karşılığıdır.

### `SelectMany` — düzleştirme
İç içe koleksiyonları tek düzeye indirir. `Select` her elemana karşılık bir *liste* döndürürse elinde liste listesi kalır; `SelectMany` bunu tek listeye çevirir.

```csharp
// Her siparişin satırları var; tüm satırları tek listede istiyoruz
var tumSatirlar = siparisler.SelectMany(s => s.Satirlar);
```

Farkı somut görmek için:

```csharp
// Select  -> IEnumerable<List<SiparisSatiri>>  (liste listesi)
var icIce = siparisler.Select(s => s.Satirlar);

// SelectMany -> IEnumerable<SiparisSatiri>     (tek düzey)
var duz = siparisler.SelectMany(s => s.Satirlar);
```

`SelectMany`'nin ikinci bir aşırı yüklemesi, düzleştirirken üst elemanı da yanında taşımanı sağlar:

```csharp
var satirlar = siparisler.SelectMany(
    s => s.Satirlar,
    (s, satir) => new { s.SiparisNo, satir.UrunAd, satir.Adet });
```

> **Not:** EF Core kullanıyorsan çoğu zaman `Join` yazmana gerek yoktur — navigation property'ler üzerinden `Include` veya doğrudan `Select` ile ilişkiye erişmek hem daha okunaklı hem de EF'in doğru SQL üretmesine olanak verir. `Join` daha çok bellekteki iki koleksiyonu birleştirirken kullanılır.

> **Bu benzetme şurada bozulur:** İki defteri yan yana koyma benzetmesi, `Join`'in **her iki tarafı da baştan sona okuduğunu** ima eder. LINQ to Objects'te durum aslında daha akıllıdır: sağdaki koleksiyon önce bir hash tablosuna alınır, sonra sol taraf bir kez gezilir. Ayrıca soldaki bir satır sağda **iki kez** eşleşirse, sonuçta o satır iki kez çıkar — defter benzetmesinde bu beklenmedik gelir ama `JOIN`'in doğal davranışıdır.

---

## 6. Toplama (Aggregation) Operatörleri

> **Benzetme —** Gün sonunda kasiyerin yaptığı iştir bu. Bütün fişleri tek tek okumazsın; "kaç satış oldu", "toplam ne kadar", "en yüksek fiş hangisi" dersin. Çok sayıda kayıttan **tek bir sayı** çıkarmak. Farkı şurada: "en yüksek tutar ne kadardı" ile "en yüksek tutarlı fiş hangisiydi" ayrı sorulardır. Birincisi `Max`, ikincisi `MaxBy`.

**Basitçe:** Toplama operatörleri, bir koleksiyondan tek bir değer üretir. Sayarlar, toplarlar, ortalama alırlar, en büyüğü bulurlar. Hepsi sorguyu **anında çalıştırır** — çünkü cevabı verebilmek için tüm elemanları görmek zorundadırlar.

**Teknik olarak:**

| Operatör | İş |
|---|---|
| `Count()` / `LongCount()` | Eleman sayısı |
| `Sum()` | Toplam |
| `Average()` | Ortalama |
| `Min()` / `Max()` | En küçük / en büyük |
| `MinBy()` / `MaxBy()` | Değeri değil, **o değere sahip elemanı** döndürür (.NET 6+) |
| `Aggregate()` | Özel katlama işlemi — genel amaçlı |

```csharp
// Max, en yüksek fiyatı verir
decimal enYuksek = urunler.Max(u => u.Fiyat);

// MaxBy, en pahalı ürünün kendisini verir
Urun enPahali = urunler.MaxBy(u => u.Fiyat);
```

`Count()`'un şartlı hâli ayrıca vardır ve `Where` yazmaktan kısadır:

```csharp
int aktifSayisi = urunler.Count(u => u.Aktif);      // Where(...).Count() ile aynı
```

### `Aggregate` — genel amaçlı katlama

Diğerleri `Aggregate`'in hazır hâlleridir. Kendi katlama kuralını yazmak istediğinde kullanılır: bir başlangıç değeri verirsin, her eleman için biriken değeri güncellersin.

```csharp
// Toplama işlemini elle yazmak
int toplam = sayilar.Aggregate(0, (birikim, x) => birikim + x);

// Metinleri birleştirmek
string liste = adlar.Aggregate((a, b) => a + ", " + b);
```

> Metin birleştirmek için `Aggregate` yerine `string.Join(", ", adlar)` kullan. `Aggregate` her adımda yeni bir string üretir; uzun listede pahalıdır.

> `Average()` ve `Min()/Max()` boş koleksiyonda `InvalidOperationException` fırlatır. `Sum()` ise 0 döndürür. Boş olabilecek kaynaklarda `Any()` ile kontrol et veya nullable sürümünü kullan.

```csharp
// Boş koleksiyona dayanıklı iki yol
decimal ort1 = urunler.Any() ? urunler.Average(u => u.Fiyat) : 0m;
decimal? ort2 = urunler.Select(u => (decimal?)u.Fiyat).Average();   // boşsa null
```

`Sum()`'ın sessiz tuzağı ise **taşma**dır: `int` toplamı `int` sınırını aşarsa `OverflowException` alırsın (checked bağlamda) ya da sessizce yanlış sayı elde edersin. Büyük toplamlarda `long` veya `decimal`'e çıkmak gerekir.

```csharp
long guvenliToplam = kayitlar.Sum(k => (long)k.Adet);
```

---

## 7. Eleman ve Niceleyici Operatörleri

> **Benzetme —** Nüfus müdürlüğünde iki farklı arama yaparsın. TC kimlik numarasıyla ararsan **tam olarak bir** kişi çıkmalıdır; iki kişi çıkıyorsa sistemde ciddi bir hata var demektir ve memur bunu sana söylemelidir. "Ahmet" diye ararsan yüzlerce kişi çıkar, sen de "ilki işimi görür" dersin. `Single` birinci aramadır, `First` ikincisi. Hangisini yazdığın, koda bakan kişiye **ne beklediğini** anlatır.

**Basitçe:** Bu operatörler koleksiyondan tek bir eleman çeker ya da "var mı yok mu" sorusunu cevaplar. Aralarındaki fark, hiç eleman yokken veya birden fazla varken ne yaptıklarıdır. `OrDefault` ekleri "yoksa hata verme, boş dön" anlamına gelir.

**Teknik olarak:**

| Operatör | Eşleşme yoksa | Birden çok eşleşme varsa |
|---|---|---|
| `First()` | **Exception** | İlkini döndürür |
| `FirstOrDefault()` | `null` / `default` | İlkini döndürür |
| `Single()` | **Exception** | **Exception** |
| `SingleOrDefault()` | `null` / `default` | **Exception** |
| `Last()` / `LastOrDefault()` | Exception / `null` | Sonuncuyu |
| `ElementAt(i)` | Exception | — |

Seçim, **niyetini** belgeler:
- `Single` = "burada tam olarak bir tane olmalı, yoksa veri bozuk demektir"
- `First` = "birden fazla olabilir, ilki işimi görür"

Id ile kayıt ararken `SingleOrDefault` kullanmak, veritabanında mükerrer kayıt olması hâlinde bunu sana **hata olarak** bildirir — sessizce yanlış kaydı işlemektense iyidir. Bedeli: EF Core en fazla iki satır çeker.

```csharp
// Tam olarak bir tane bekliyorum; yoksa null, fazlaysa hata istiyorum
var musteri = context.Musteriler.SingleOrDefault(m => m.Id == id);
if (musteri is null) return NotFound();

// Birden fazla olabilir, en yenisi işimi görür
var sonKayit = loglar.OrderByDescending(l => l.Tarih).FirstOrDefault();
```

**`default` neyi döndürür?** Referans tipinde `null`, değer tipinde tipin sıfır hâli (`int` için `0`, `bool` için `false`). Bu, değer tiplerinde tehlikelidir: `sayilar.FirstOrDefault()` sonucu `0` geldiğinde, "liste boştu" mu yoksa "ilk eleman gerçekten 0'dı" mı ayırt edemezsin. .NET 6 ile gelen aşırı yükleme bu belirsizliği çözer:

```csharp
int ilk = sayilar.FirstOrDefault(-1);   // boşsa -1 döner, 0 ile karışmaz
```

**`Last()` ve `ElementAt()` uyarısı:** İkisi de sıra bilgisine dayanır. `IQueryable` üzerinde `Last()` kullanmak için sorgunun sıralanmış olması gerekir; sırasız bir sorguda EF Core hata verir. Sıralı olmayan bir kaynakta "son eleman" kavramı zaten anlamsızdır.

**Niceleyiciler**

| Operatör | Ne yapar |
|---|---|
| `Any()` | En az bir eleman var mı — bulur bulmaz durur |
| `Any(şart)` | Şarta uyan var mı |
| `All(şart)` | Hepsi uyuyor mu (boş koleksiyonda `true` döner) |
| `Contains(x)` | Belirli eleman var mı |

> `Count() > 0` yerine **`Any()`** kullan. `Count()` tüm koleksiyonu gezer; `Any()` ilk elemanda durur. Veritabanında ise `SELECT COUNT(*)` yerine `EXISTS` üretilir.

`All`'un boş koleksiyonda `true` döndürmesi mantıksal bir kuraldır ("boş kümede her önerme doğrudur") ama pratikte hata kaynağıdır:

```csharp
var bosListe = new List<Urun>();
bool hepsiUcuz = bosListe.All(u => u.Fiyat < 10);   // true — beklediğin bu olmayabilir

// Niyetin "en az bir eleman var ve hepsi ucuz" ise iki şartı birlikte yaz
bool gercektenHepsiUcuz = bosListe.Any() && bosListe.All(u => u.Fiyat < 10);
```

> **Bu benzetme şurada bozulur:** Nüfus müdürlüğü örneği `Single` ile `First`'ün **niyet** farkını iyi anlatır ama maliyet farkını gizler. Memur TC ile arayınca tek kayıt bulup durmaz — `Single`, "başka var mı" diye emin olmak için **ikinci bir eleman aramak zorundadır**. `First` ilkini bulunca durur. Bu yüzden `Single` her zaman biraz daha pahalıdır; bedeli bilerek ödersin, çünkü karşılığında veri bütünlüğü kontrolü alırsın.

---

## 8. Küme İşlemleri ve Sayfalama

> **Benzetme —** Düğün davetiye listesi hazırlıyorsun. Bir listede damadın davetlileri, diğerinde gelinin. İkisini birleştirip aynı kişiyi iki kez yazmamak `Union`. Sadece iki tarafın da tanıdıklarını bulmak `Intersect`. Damatta olup gelinde olmayanlar `Except`. Listeyi olduğu gibi alt alta eklemek, tekrarları umursamamak ise `Concat`. Sayfalama da şu: liste iki yüz kişi, sen davetiyeyi yirmişerli sayfalar hâlinde bastırıyorsun.

**Basitçe:** Küme operatörleri iki koleksiyonu matematiksel küme mantığıyla birleştirir ya da ayırır. Sayfalama ise uzun bir listeyi parçalara bölüp bir seferde bir parçasını göstermektir — `Skip` ile baştakileri atlar, `Take` ile istediğin kadarını alırsın.

**Teknik olarak:**

| Operatör | İş |
|---|---|
| `Distinct()` | Tekrarları atar |
| `DistinctBy(x => x.Alan)` | Belirli alana göre teklileştirir (.NET 6+) |
| `Union` / `Intersect` / `Except` | Birleşim / kesişim / fark |
| `Concat` | Birleştirir, tekrarları **atmaz** |
| `Skip(n)` / `Take(n)` | Atla / al — sayfalamanın temeli |
| `Chunk(n)` | Koleksiyonu n'lik parçalara böler (.NET 6+) |

```csharp
var damat = new[] { "Ali", "Veli", "Ayşe" };
var gelin = new[] { "Ayşe", "Fatma" };

damat.Union(gelin);       // Ali, Veli, Ayşe, Fatma   (tekrar yok)
damat.Concat(gelin);      // Ali, Veli, Ayşe, Ayşe, Fatma  (tekrar var)
damat.Intersect(gelin);   // Ayşe
damat.Except(gelin);      // Ali, Veli
```

**Önemli ayrıntı:** `Distinct`, `Union`, `Intersect` ve `Except` eşitliği `Equals` ve `GetHashCode` üzerinden belirler. Kendi sınıfların için bunları geçersiz kılmadıysan iki ayrı nesne, alanları aynı olsa bile farklı sayılır. `record` tipler bu metotları otomatik ürettiği için bu sorunu yaşamazsın.

```csharp
// class ise: iki nesne aynı verilere sahip olsa da Distinct onları ayrı sayar
var tekilUrunler = urunler.Distinct();              // beklendiği gibi çalışmayabilir

// Niyetin "koda göre tekilleştir" ise açıkça söyle
var tekilKodlar = urunler.DistinctBy(u => u.Kod);   // .NET 6+
```

`Chunk`, toplu işlemlerde çok işe yarar — örneğin bin kaydı yüzerlik paketler hâlinde bir API'ye göndermek:

```csharp
foreach (var paket in kayitlar.Chunk(100))
{
    // paket bir Kayit[] dizisidir, en fazla 100 eleman
    GonderAsync(paket);
}
```

**Sayfalama kalıbı:**

```csharp
int sayfa = 3, boyut = 20;

var sayfaVerisi = urunler
    .OrderBy(u => u.Id)          // sıralama ŞART
    .Skip((sayfa - 1) * boyut)
    .Take(boyut)
    .ToList();
```

`OrderBy` olmadan sayfalama anlamsızdır: veritabanı sıra garantisi vermez, aynı kayıt iki sayfada çıkabilir veya hiç çıkmayabilir.

Gerçek bir sayfalama isteğinde toplam sayıya da ihtiyaç duyarsın. Bunu ayrı bir sorguyla almak gerekir:

```csharp
var temel = context.Urunler.Where(u => u.Aktif);

int toplam = await temel.CountAsync();                 // 1. sorgu
var veri   = await temel.OrderBy(u => u.Id)
                        .Skip((sayfa - 1) * boyut)
                        .Take(boyut)
                        .ToListAsync();                // 2. sorgu
```

`temel` değişkeni henüz çalışmamış bir sorgudur; iki kez kullanmak iki ayrı SQL üretir. Burada bu **istenen** davranıştır — `ToList()` deseydin tüm tabloyu belleğe çekmiş olurdun.

> **Bu benzetme şurada bozulur:** Davetiye listesi örneğinde `Skip(1000)` demek, ilk bin ismin üstünden parmağını hızlıca geçirmek gibi görünür — ucuz. Veritabanında ise `OFFSET 1000` gerçekten bin satır üretip atar. Sayfa numarası büyüdükçe sorgu yavaşlar. Çok derin sayfalama gereken yerlerde `Skip/Take` yerine "son gördüğün Id'den sonrasını getir" kalıbı (keyset pagination) kullanılır:

```csharp
// Derin sayfalamada daha ucuz alternatif
var sonraki = context.Urunler
    .Where(u => u.Id > sonGorulenId)
    .OrderBy(u => u.Id)
    .Take(boyut)
    .ToList();
```

---

## 9. Yürütmeyi Zorlayan Operatörler

> **Benzetme —** Lokantada masaya oturur, garsona sırayla söylersin: çorba, ana yemek, salata, ayran. Her söylediğinde mutfağa koşmaz; siparişi biriktirir. Mutfak, sen "tamam, bu kadar" dediğinde harekete geçer. LINQ'te de operatörleri arka arkaya dizmek sipariş vermektir; `ToList()` demek "tamam, bu kadar, getir" demektir. Ne kadar geç dersen, mutfak o kadar toplu ve verimli çalışır.

**Basitçe:** Bazı operatörler sorguyu inşa etmeye devam eder, bazıları ise "yeter, çalıştır" der. İkincisine **maddeleştirme (materialization)** denir. Hangisinin hangisi olduğunu bilmek, sorgunun nerede gerçekten çalıştığını bilmek demektir.

**Teknik olarak:** Şunlar sorguyu **anında** çalıştırır:

`ToList()` · `ToArray()` · `ToDictionary()` · `ToHashSet()` · `Count()` · `Sum()` · `Average()` · `Min()` · `Max()` · `First()` · `Single()` · `Any()` · `All()` · `Contains()`

Geri kalanı (`Where`, `Select`, `OrderBy`, `GroupBy`, `Join`, `Skip`, `Take`) sorguyu inşa etmeye devam eder.

**Kural:** Sorguyu olabildiğince zincirle, maddeleştirmeyi **en sona** bırak.

Bu kuralın en pahalı ihlali, `ToList()`'i zincirin ortasına koymaktır:

```csharp
// YANLIŞ — tüm tabloyu belleğe çeker, sonra bellekte filtreler
var sonuc = context.Urunler
    .ToList()                        // burada SQL çalıştı: SELECT * FROM Urunler
    .Where(u => u.Fiyat > 1000)      // artık bellekte, LINQ to Objects
    .ToList();

// DOĞRU — filtre SQL'e gider, sadece gereken satırlar gelir
var sonuc2 = context.Urunler
    .Where(u => u.Fiyat > 1000)
    .ToList();                       // SELECT ... WHERE Fiyat > 1000
```

İki kod da aynı sonucu üretir. Birincisi on milyon satırlık tabloda uygulamayı düşürür, ikincisi çalışır. Kod incelemesinde bakacağın ilk şeylerden biri budur.

`AsEnumerable()` de aynı sınırı çizer ama veriyi belleğe **toplamadan** yapar: o noktadan sonrası LINQ to Objects'tir, akış devam eder. SQL'e çevrilemeyen bir işlemi zorunlu olarak bellekte yapman gerekiyorsa, `ToList()` yerine bunu kullanmak daha ucuzdur — ama sınırın nerede olduğunu bilerek.

```csharp
var sonuc3 = context.Urunler
    .Where(u => u.Aktif)             // SQL'de çalışır
    .AsEnumerable()                  // sınır: buradan sonrası bellekte
    .Where(u => OzelKural(u))        // kendi C# metodun, SQL'e çevrilemez
    .ToList();
```

---

## 10. LINQ to Objects ve LINQ to Entities

> **Benzetme —** Aynı cümleyi iki kişiye söylüyorsun. Biri Türkçe bilen bir arkadaşın: ne dersen anlıyor, deyim de kullanabilirsin. Diğeri bir tercüman aracılığıyla konuştuğun yabancı bir muhatap: tercüman cümlelerinin çoğunu aktarıyor ama bazılarında duruyor — "bu deyimin karşılığı yok" diyor. Üstelik iki muhatabın bazı alışkanlıkları farklı: senin "boş" dediğin şeyi biri "yok" sayıyor, diğeri "belirsiz". Cümlen aynı, cevaplar farklı.

**Basitçe:** Aynı LINQ kodu, bellekteki bir liste üzerinde ile veritabanı üzerinde tamamen farklı şeyler yapar. Bellekte C#'ın her özelliğini kullanabilirsin. Veritabanında ise yazdığın her şeyin SQL'e çevrilebilmesi gerekir; çevrilemeyen bir şey yazarsan ya hata alırsın ya da — daha kötüsü — tüm tablo belleğe çekilir.

**Teknik olarak:** Aynı sözdizimi, farklı motor. Fark bilinmezse üretimde sürpriz çıkar.

| | LINQ to Objects | LINQ to Entities (EF Core) |
|---|---|---|
| Kaynak tipi | `IEnumerable<T>` | `IQueryable<T>` |
| Sorgu temsili | Delegate | Expression tree |
| Nerede çalışır | Bellekte | Veritabanında (SQL'e çevrilerek) |
| C# metodu çağırabilir miyim | Evet, her şeyi | **Hayır** — sadece SQL'e çevrilebilenleri |
| Büyük/küçük harf duyarlılığı | C# kurallarına göre | Veritabanı collation'ına göre |

**Expression tree (ifade ağacı)** — Farkın kaynağı budur. `IEnumerable` üzerinde yazdığın lambda, derlenmiş bir **metottur**: çalıştırılır, biter. `IQueryable` üzerinde yazdığın lambda ise derlenmez, **veri yapısı olarak saklanır**: "eşittir karşılaştırması, solunda Fiyat alanı, sağında 1000 sabiti" şeklinde bir ağaç. EF Core bu ağacı gezerek SQL metnini yazar. Bir C# metodunu çeviremiyor olmasının sebebi de budur — metodun içine bakamaz, sadece adını görür.

### Çevrilemeyen ifade sorunu

```csharp
// Kendi metodun SQL'e çevrilemez — çalışma anında hata verir
var sonuc = context.Urunler
    .Where(u => FiyatUygunMu(u))     // ✗ çevrilemez
    .ToList();
```

EF Core bu durumda ya exception fırlatır ya da (eski sürümlerde) sessizce tüm tabloyu belleğe çekip filtreler — ikincisi çok daha tehlikelidir.

Aynı kuralın daha sinsi hâlleri:

```csharp
// Bunlar SQL'e çevrilir
.Where(u => u.Ad.Contains("kalem"))        // LIKE '%kalem%'
.Where(u => u.Ad.StartsWith("A"))          // LIKE 'A%'
.Where(u => u.Tarih.Year == 2025)          // YEAR(Tarih) = 2025

// Bunlar genelde çevrilmez
.Where(u => u.Ad.Normalize() == aranan)
.Where(u => TarihHesapla(u.Tarih) > esik)
.Where(u => u.Etiketler.Any(e => Kontrol(e)))
```

Çözüm üç yoldan biridir: ifadeyi SQL'e çevrilebilir hâle getirmek, işi veritabanı tarafında bir ifadeyle yapmak, ya da `AsEnumerable()` ile sınırı **bilerek** çizip kalan işi bellekte yapmak. Üçüncüsünü seçtiğinde, sınırdan önce mümkün olan tüm filtreyi uygulamış olman gerekir.

### `null` davranış farkı
Bellekte `null` karşılaştırmaları C# kurallarına uyar. SQL'de `NULL = NULL` sonucu `false`'tur (üç değerli mantık). Aynı LINQ ifadesi iki ortamda farklı sonuç verebilir.

```csharp
// Bellekte: Aciklama'sı null olan kayıtlar da gelir (null != "x" -> true)
// SQL'de:   Aciklama IS NULL olan satırlar GELMEZ (NULL <> 'x' -> NULL -> false)
.Where(u => u.Aciklama != "x")

// Niyetini açıkça yaz
.Where(u => u.Aciklama == null || u.Aciklama != "x")
```

### Karşılaşacağın klasik: N+1 problemi

```csharp
var siparisler = context.Siparisler.ToList();       // 1 sorgu
foreach (var s in siparisler)
    Console.WriteLine(s.Musteri.Ad);                 // her satır için 1 sorgu daha
```

100 sipariş = 101 sorgu. Çözümü Hafta 3'te (`Include`, projeksiyon) işlenecek; şimdilik **sebebini** bil: ertelenmiş yükleme + döngü.

> **Bu benzetme şurada bozulur:** Tercüman benzetmesi, çevrilemeyen bir cümlede tercümanın "duracağını" söylüyor. EF Core'un eski sürümlerinde tercüman durmazdı — cümleyi çeviremeyince sessizce "hepsini getir, ben burada ayıklarım" derdi. Bu, hata almaktan çok daha kötüdür: kod çalışır, testler geçer, üretimde veri büyüyünce uygulama durur. EF Core 3.0'dan itibaren bu davranış kaldırıldı ve artık hata fırlatılıyor; ama eski kodla karşılaşırsan bu tuzağı bil.

---

## Tek Bakışta Özet

- LINQ, farklı kaynaklar için tek sorgulama dilidir; method ve query syntax aynı koda derlenir.
- Sorgu bir **tariftir**; ertelenmiş çalışır ve her gezilişte yeniden çalışır.
- Lambda, değişkenin değerini değil kendisini yakalar (closure) — sorgu çalıştığı andaki değer geçerlidir.
- `Select` projeksiyondur — EF Core'da sadece gereken kolonları çekmenin yolu.
- İkinci sıralama kriteri `ThenBy` ile verilir; ikinci `OrderBy` ilkini iptal eder.
- `GroupBy`, `IGrouping` döndürür: anahtarı olan liste.
- `ToLookup` çoklu değere izin verir, `ToDictionary` vermez.
- `SelectMany` iç içe koleksiyonları düzleştirir; `GroupJoin` + `DefaultIfEmpty` left join'dir.
- `Max` değeri, `MaxBy` o değere sahip elemanı verir.
- `First`/`Single`/`OrDefault` seçimi niyet beyanıdır; Id aramasında `SingleOrDefault` veri bütünlüğünü korur.
- Varlık kontrolünde `Count() > 0` değil **`Any()`**. `All()` boş koleksiyonda `true` döner.
- Sayfalamada `OrderBy` olmadan `Skip/Take` güvenilmezdir; benzersiz bir alanı son kriter yap.
- `ToList()` sınırdır: sonrası bellekte çalışır. En sona koy.
- `IQueryable` üzerinde her C# metodu çalışmaz; çevrilemeyen ifade ya hata verir ya da tüm tabloyu çeker.
- `null` karşılaştırmaları bellekte ve SQL'de farklı sonuç verir.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| LINQ | Dil içine gömülü sorgulama altyapısı |
| Provider | LINQ sorgusunu belirli bir kaynağa çeviren bileşen |
| Method / Query syntax | Zincirleme metot çağrısı / SQL benzeri sözdizimi |
| Lambda ifadesi | `x => ...` biçiminde yazılan isimsiz metot |
| Closure (kapanış) | Lambda'nın dışarıdaki değişkeni yakalaması |
| Deferred execution | Sorgunun sonuç istendiğinde çalışması |
| Materialization | Sorgunun çalıştırılıp sonucun koleksiyona alınması |
| Streaming operatör | Veriyi tek tek geçiren operatör (`Where`, `Select`) |
| Buffering operatör | Tüm veriyi görmesi gereken operatör (`OrderBy`, `GroupBy`) |
| Projection | Elemanı başka bir şekle dönüştürme (`Select`) |
| Anonim tip | Derleyicinin ürettiği, yerel kullanımlık isimsiz tip |
| IGrouping | Anahtarı olan eleman kümesi — `GroupBy` çıktısı |
| ToLookup | Anahtar başına çoklu değer tutan, anında çalışan gruplama |
| SelectMany | İç içe koleksiyonları tek düzeye indirme |
| GroupJoin | Sol tarafı koruyan, eşleşmeleri grup olarak veren birleştirme |
| Aggregation | Çok elemandan tek değer üretme (`Sum`, `Count`) |
| Quantifier | Varlık/kapsam sorgusu (`Any`, `All`, `Contains`) |
| Expression tree | Lambda'nın kod yerine veri yapısı olarak saklanmış hâli |
| Keyset pagination | `Skip` yerine "son görülen anahtardan sonrası" ile sayfalama |
| N+1 problemi | Bir sorgu + her satır için ek sorgu üreten kalıp |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Sorgu tanımlandığında çalışır" | Sonucu istendiğinde çalışır — ve her seferinde yeniden |
| "İkinci sıralama için `OrderBy` tekrar yazılır" | `ThenBy` kullanılır; ikinci `OrderBy` ilkini iptal eder |
| "`Count() > 0` ile `Any()` aynıdır" | `Any()` ilk elemanda durur, `Count()` hepsini gezer |
| "`Where` içinde her C# metodu kullanılabilir" | `IQueryable` üzerinde yalnızca SQL'e çevrilebilenler |
| "`FirstOrDefault` her zaman en güvenlisi" | Tek kayıt bekleniyorsa `SingleOrDefault` veri hatasını görünür kılar |
| "`GroupBy` sözlük döndürür" | `IEnumerable<IGrouping<K,T>>` döndürür; sözlük istiyorsan `ToDictionary` |
| "`Concat` ile `Union` aynı şey" | `Union` tekrarları atar, `Concat` atmaz |
| "`All()` boş listede `false` döner" | `true` döner; niyetin farklıysa `Any() && All()` yaz |
| "`ToList()` nereye koyulduğu fark etmez" | Zincirin ortasındaki `ToList()` filtreyi belleğe taşır |
| "Lambda içindeki değişken sorgu yazıldığı andaki değeridir" | Sorgu **çalıştığı** andaki değeri kullanılır (closure) |

---

## Sonraki

→ `04-Asenkron-Programlama.md` (Perşembe)
