# Hafta 1 · Salı — Koleksiyonlar ve Koleksiyon Arayüzleri

**Okuma süresi:** ~42 dk
**Neden bu konu:** Yazacağın her uygulama veri listeleriyle çalışır. Yanlış koleksiyon seçimi, kodun doğru ama gereksiz yavaş olmasına yol açar — ve `IEnumerable`/`IQueryable` ayrımını bilmeden EF Core'da performans sorunlarını çözemezsin.

---

## Önce Basitçe

Bir programın yaptığı işin büyük kısmı "bir sürü şeyi bir arada tutmak ve içinden bir şey bulmak"tır. Müşteriler, siparişler, ürünler, log kayıtları. Bunları bir yere koyman gerekir. C# sana bu iş için tek bir kap vermez; onlarca farklı kap verir ve hangisini seçtiğin, programının hızını belirler.

Kaplar arasındaki fark kapasiteleri değil, **davranışları**. Bazısında araya bir şey sıkıştırmak kolaydır ama bir şey bulmak zordur. Bazısında bulmak anlıktır ama sıra diye bir şey yoktur. Bazısı sadece en son koyduğunu geri verir. Doğru kabı seçmek için kabın özelliklerini değil, **senin ne yapacağını** bilmen gerekir: çok mu ekleyeceksin, çok mu arayacaksın, sıra önemli mi, tekrar olabilir mi.

Bu seçimin ne kadar önemli olduğunu ölçen bir dil var: bir işlemin, eleman sayısı arttıkça ne kadar yavaşladığını anlatan basit bir gösterim. Yüz elemanda hiçbir seçim yanlış değildir. Bir milyon elemanda yanlış seçim, saniyeler süren bir ekranın dakikalar sürmesi demektir. Bootcamp projelerinde karşına çıkacak "neden bu sayfa geç açılıyor" sorusunun cevabı çoğu zaman bir satırlık bir koleksiyon seçimidir.

Bir de kapların dış kapağı var: arayüzler. Bir metoda "bana bir liste ver" demekle "bana gezilebilir bir şey ver" demek aynı şey değil. İkincisi çok daha esnektir; çağıran taraf ne verirse versin çalışır. Yazılım tasarımında iyi alışkanlık, gerektiğinden fazlasını istememektir.

En sonda, bu haftanın en çok kafa karıştıran konusu var: bazı koleksiyonlar aslında koleksiyon değil, **henüz çalışmamış bir sorgudur**. Bellekte duran bir liste ile veritabanına gidecek bir plan aynı arayüzü paylaşır ama tamamen farklı davranır. Bu farkı kaçırmak, EF Core'da tüm tabloyu belleğe çekmenin klasik sebebidir. Şimdi detaya iniyoruz.

> **Ana benzetme:** Koleksiyon seçmek, eşyayı hangi kaba koyacağını seçmektir. Raf sıralıdır ve numarayla bakarsın. Emanet dolabı fişteki numaradan anında bulunur ama sıra diye bir şey yoktur. Kuyruk ilk geleni ilk alır, tabak destesi ise en üsttekini. Hiçbir kap "en iyisi" değildir; yanlış olan, bakkal poşetine kitaplık muamelesi yapmaktır.

---

## Bu Notta Ne Var

1. Koleksiyon nedir, dizi ile farkı
2. Karmaşıklık (Big-O) sezgisi — seçim yapabilmek için gereken asgari
3. `List<T>` ve iç mekanizması
4. `Dictionary<TKey,TValue>` ve hash tabloları
5. `HashSet<T>`, `Queue<T>`, `Stack<T>`, `LinkedList<T>`
6. Koleksiyon arayüzleri hiyerarşisi
7. `IEnumerable` ve `IQueryable` — en kritik ayrım
8. `yield return` ve tembel üretim
9. Salt okunur ve değişmez koleksiyonlar
10. Hangi durumda hangisi

---

## 1. Dizi ve Koleksiyon

> **Benzetme —** Dizi, sinema salonundaki koltuk sırasıdır: koltuklar yan yana çakılıdır, numaralıdır, "14 numara nerede" diye aramazsın, doğrudan gidersin. Ama salona bir koltuk daha eklemek istersen mümkün değildir — yeni bir salon yaptırıp herkesi taşımak gerekir. Koleksiyon ise lokantanın masalarıdır: kalabalık gelince yan masayı çekip eklersin, boşalınca kaldırırsın.

**Basitçe:** Dizi, en baştan kaç eleman tutacağını söylediğin ve bir daha büyütemediğin en ilkel kaptır. Karşılığında çok hızlıdır, çünkü elemanlar bellekte yan yana durur. Koleksiyon ise büyüyüp küçülebilen kaplardır; arka planda senin yerine dizi yönetirler. Günlük kodda neredeyse hep koleksiyon kullanırsın, dizi ise boyutu gerçekten sabit olan yerlerde kalır.

**Teknik olarak:** **Dizi (array)** — Sabit boyutlu, bellekte **bitişik** duran eleman kümesi. `int[] sayilar = new int[10];`

Bitişik olması iki şey getirir: indeksle erişim anlıktır (adres = başlangıç + indeks × eleman boyutu) ve CPU önbelleği (cache) çok verimli çalışır. Bedeli ise boyutun sabit olmasıdır — büyütmek için yeni dizi açıp kopyalamak gerekir.

```csharp
int[] sayilar = new int[3];      // her eleman 0 ile başlar
sayilar[0] = 10;
// sayilar[3] = 40;              // IndexOutOfRangeException

// Büyütmek diye bir şey yok; yeni dizi açılır ve kopyalanır
Array.Resize(ref sayilar, 5);    // aslında yeni dizi üretip referansı değiştirir
```

**Koleksiyon** — Boyutu çalışma anında değişebilen, ekleme/silme/arama davranışlarını kapsülleyen veri yapıları. `System.Collections.Generic` altındadırlar.

> `System.Collections` (generic olmayan, `ArrayList` / `Hashtable`) sadece eski kodda karşına çıkar. Boxing yapar ve tip güvenli değildir. Yeni kodda kullanılmaz.

Farkın kod hâli:

```csharp
// Generic olmayan — her şeyi kabul eder, hatayı çalışma anına erteler
var eski = new System.Collections.ArrayList();
eski.Add(1);
eski.Add("bir");                 // derlenir!
int x = (int)eski[1];            // InvalidCastException — çalışma anında patlar

// Generic — hata derleme anında yakalanır
var yeni = new List<int>();
yeni.Add(1);
// yeni.Add("bir");              // derlenmez
```

---

## 2. Big-O Sezgisi

> **Benzetme —** Telefon rehberinde birini arıyorsun. Sayfaları baştan tek tek çevirirsen, rehber kalınlaştıkça süren orantılı olarak uzar. Ortadan açıp "aradığım harf önde mi arkada mı" diye ilerlersen, rehber iki katına çıktığında süren sadece bir adım uzar. Kişinin sayfa numarasını zaten biliyorsan, rehberin kalınlığı seni hiç ilgilendirmez. Big-O, bu üç durumun adını koymaktan ibarettir.

**Basitçe:** Big-O bir sürenin kendisi değil, **sürenin nasıl büyüdüğüdür**. "Bu işlem 3 milisaniye sürüyor" demez; "eleman sayısını on katına çıkarırsan süre de on katına çıkar" ya da "hiç değişmez" der. Küçük veride hepsi aynı görünür; asıl fark veri büyüdükçe ortaya çıkar. Bu yüzden koleksiyon seçerken bugünkü veriye değil, yarınki veriye bakarsın.

**Teknik olarak:** **Big-O** — Bir işlemin maliyetinin, eleman sayısı (n) büyüdükçe nasıl arttığını gösteren gösterim. Kesin süre değil, **büyüme hızıdır**.

| Gösterim | Anlamı | Örnek |
|---|---|---|
| **O(1)** | Eleman sayısından bağımsız, sabit | `Dictionary`'de anahtarla erişim, `List`'te indeksle erişim |
| **O(log n)** | Her adımda arama alanı yarılanır | Sıralı dizide ikili arama, `SortedDictionary` |
| **O(n)** | Eleman sayısıyla orantılı | Listede tek tek arama, `foreach` |
| **O(n log n)** | Verimli sıralama algoritmaları | `List.Sort()`, `OrderBy` |
| **O(n²)** | İç içe iki döngü | Liste içinde liste araması |

Pratik anlamı: 10 elemanda hepsi hızlıdır. 1.000.000 elemanda O(1) ile O(n) arasındaki fark, "anında" ile "kullanıcı sekmeyi kapatır" farkıdır.

Kabaca bir büyüklük tablosu, sezgiyi oturtmak için:

| n | O(log n) | O(n) | O(n log n) | O(n²) |
|---|---|---|---|---|
| 100 | ~7 | 100 | ~700 | 10.000 |
| 10.000 | ~13 | 10.000 | ~130.000 | 100.000.000 |
| 1.000.000 | ~20 | 1.000.000 | ~20.000.000 | 10¹² (pratikte imkânsız) |

**En sık yapılan hata:** `List<T>` içinde `Contains` veya `Any` ile arama yapmak O(n)'dir. Bunu bir döngünün içine koyarsan O(n²) olur. Aynı işi `HashSet<T>` ile yapmak O(1)'dir.

```csharp
// O(n²) — 10.000 elemanda ~50 milyon karşılaştırma
foreach (var id in gelenIdler)
    if (mevcutIdler.Contains(id))   // mevcutIdler bir List<int> ise
        ...

// O(n) — aynı iş, HashSet ile
var kume = new HashSet<int>(mevcutIdler);
foreach (var id in gelenIdler)
    if (kume.Contains(id))
        ...
```

Aynı tuzağın daha gizli hâli iç içe LINQ'tir; `Any` içeride tekrar tekrar gezer:

```csharp
// O(n * m) — her sipariş için tüm müşteri listesi taranır
var eslesen = siparisler.Where(s => musteriler.Any(m => m.Id == s.MusteriId)).ToList();

// Önce sözlüğe çevir, sonra tek geçişte eşle
var musteriIdleri = musteriler.Select(m => m.Id).ToHashSet();
var eslesen2 = siparisler.Where(s => musteriIdleri.Contains(s.MusteriId)).ToList();
```

> **Bu benzetme şurada bozulur:** Rehber örneği, küçük sabitlerin önemsiz olduğunu düşündürür. Big-O gerçekten de sabitleri yok sayar — ama gerçek makinede sabitler vardır. 20 elemanlık bir `List` üzerinde doğrusal arama, `Dictionary` kurma maliyetinden genellikle daha hızlıdır, çünkü liste bellekte bitişiktir ve CPU önbelleğine sığar. Big-O "hangi eğri" der, "hangi nokta" demez. Küçük n'de ölçüm, teoriyi yener.

---

## 3. `List<T>`

> **Benzetme —** `List<T>`, duvara monte uzun bir kitap rafıdır. Kitabı sona eklemek bir saniye sürer. Ama araya sıkıştırmak istersen, sağdaki bütün kitapları birer boy kaydırman gerekir. Raf dolduğunda da iki katı uzunlukta yeni bir raf takıp bütün kitapları taşırsın — o gün bir zahmet, sonrası yine rahat.

**Basitçe:** `List<T>` günlük hayatta en çok kullanacağın koleksiyon. İçinde aslında sıradan bir dizi var; dolunca büyütüp kopyalıyor, sen bunu fark etmiyorsun. Sona eklemek ve numarayla erişmek çok hızlı. Ortaya eklemek, silmek ve içinde arama yapmak ise eleman sayısıyla orantılı olarak yavaşlıyor.

**Teknik olarak:** **`List<T>`** — En sık kullanılan koleksiyon. İçeride aslında bir **dizi** tutar; dolduğunda kapasitesini iki katına çıkarıp yeni diziye kopyalar.

| İşlem | Maliyet |
|---|---|
| İndeksle erişim `list[5]` | **O(1)** |
| Sona ekleme `Add` | Ortalama **O(1)** (ara sıra büyüme maliyeti) |
| Başa/ortaya ekleme `Insert(0, x)` | **O(n)** — sonraki tüm elemanlar kaydırılır |
| Silme `Remove` / `RemoveAt` | **O(n)** |
| Arama `Contains` / `IndexOf` | **O(n)** |

**Capacity vs Count** — `Count` içindeki eleman sayısı, `Capacity` ayrılmış dizi boyutudur. Kaç eleman ekleyeceğini biliyorsan baştan vermek yeniden boyutlandırma maliyetini ortadan kaldırır:

```csharp
var liste = new List<Urun>(capacity: 1000);
```

Büyümenin gerçekten olduğunu gözleyebilirsin:

```csharp
var l = new List<int>();
int onceki = l.Capacity;
for (int i = 0; i < 40; i++)
{
    l.Add(i);
    if (l.Capacity != onceki)
    {
        Console.WriteLine($"Count={l.Count}  Capacity {onceki} -> {l.Capacity}");
        onceki = l.Capacity;
    }
}
// 0 -> 4 -> 8 -> 16 -> 32 ...  her seferinde iç dizi yeniden ayrılır ve kopyalanır
```

Buradaki "ortalama O(1)" ifadesinin adı **amortized O(1)**'dir: tek tek bakıldığında bazı `Add` çağrıları pahalıdır, ama pahalı çağrılar seyrekleştiği için eleman başına ortalama maliyet sabit kalır.

> **Dikkat:** Bir koleksiyonu `foreach` ile gezerken içinden eleman silmek `InvalidOperationException` fırlatır. Çözüm: `for` döngüsünü sondan başa çevirmek veya `RemoveAll(...)` kullanmak.

```csharp
// Patlar
foreach (var u in urunler)
    if (u.Pasif) urunler.Remove(u);          // InvalidOperationException

// Doğru — sondan başa
for (int i = urunler.Count - 1; i >= 0; i--)
    if (urunler[i].Pasif) urunler.RemoveAt(i);

// Daha temizi — tek çağrı, tek geçiş
urunler.RemoveAll(u => u.Pasif);
```

Sondan başa gitmenin sebebi: baştan silersen kaydırma yüzünden bir sonraki elemanı atlarsın.

---

## 4. `Dictionary<TKey, TValue>`

> **Benzetme —** Otogardaki emanet dolaplarını düşün. Elindeki fişte bir numara yazıyor ve sen dolapları tek tek gezmiyorsun; numaradan doğrudan hangi dolap olduğunu biliyor, oraya gidiyorsun. Bin dolap da olsa on dolap da olsa süren aynı. Dictionary'nin yaptığı tam olarak budur: anahtardan bir "dolap numarası" hesaplar ve hiç arama yapmaz.

**Basitçe:** Dictionary, "bir kimlikten bir bilgiye" gitmek için kullanılır. Ürün koduyla stok adedi, kullanıcı id'siyle kullanıcı bilgisi gibi. Farkı şu: içinde arama yapmaz, hesaplama yapar. Bu yüzden içinde bir eleman mı, bir milyon eleman mı olduğu erişim süresini değiştirmez. Karşılığında sıra garantisi vermez ve anahtar tipinden bazı sorumluluklar bekler.

**Teknik olarak:** **`Dictionary<TKey,TValue>`** — Anahtar–değer çiftleri tutan, anahtarla **O(1)** erişim sağlayan koleksiyon. Arkasındaki yapı bir **hash tablosudur**.

### Nasıl O(1) olabiliyor
Anahtarın `GetHashCode()` metodu bir sayı üretir; bu sayı içerideki dizi indeksine dönüştürülür. Yani arama yapılmaz, **adres hesaplanır**.

**Hash collision (çakışma)** — İki farklı anahtarın aynı indekse düşmesi. Dictionary bunu zincirleme ile çözer; çakışma çoksa performans O(n)'e doğru bozulur.

**Kritik kural:** Bir tipi Dictionary anahtarı olarak kullanacaksan `GetHashCode()` ve `Equals()` **birlikte ve tutarlı** override edilmelidir. Ayrıca anahtar nesnesinin hash'ini etkileyen alanları sözlüğe konduktan sonra **değiştirilmemelidir** — değişirse o eleman bir daha bulunamaz.

```csharp
var stoklar = new Dictionary<string, int>();
stoklar["ABC-123"] = 50;

// Anahtar yoksa exception atar
int adet = stoklar["YOK-999"];              // KeyNotFoundException

// Güvenli okuma
if (stoklar.TryGetValue("YOK-999", out int deger)) { ... }

// Varsa güncelle, yoksa ekle
stoklar["ABC-123"] = stoklar.GetValueOrDefault("ABC-123") + 10;
```

**`TryGetValue`** iki iş yapar (varlık kontrolü + okuma) ve hash'i **bir kez** hesaplar. `ContainsKey` sonra `[]` yazmak hash'i iki kez hesaplar. Küçük ama ücretsiz bir kazanç.

Kritik kuralın bozulduğunda ne olduğunu görmek öğreticidir:

```csharp
public class Urun                 // GetHashCode/Equals override edilmemiş
{
    public string Kod { get; set; } = "";
}

var d = new Dictionary<Urun, int>();
d[new Urun { Kod = "A" }] = 1;
Console.WriteLine(d.ContainsKey(new Urun { Kod = "A" }));   // False — referans eşitliği

// record kullanmak bu sorunu baştan çözer: değer eşitliği + uyumlu hash hazır gelir
public record UrunKey(string Kod);
var d2 = new Dictionary<UrunKey, int>();
d2[new UrunKey("A")] = 1;
Console.WriteLine(d2.ContainsKey(new UrunKey("A")));        // True
```

`[]` ile `Add` arasındaki fark da sık tökezletir: `[]` varsa üzerine yazar, `Add` varsa `ArgumentException` atar.

```csharp
stoklar["ABC-123"] = 90;          // sessizce günceller
stoklar.Add("ABC-123", 90);       // ArgumentException — anahtar zaten var
stoklar.TryAdd("ABC-123", 90);    // false döner, patlamaz
```

> **Bu benzetme şurada bozulur:** Emanet dolabı örneği, her anahtarın kendine ait boş bir dolabı olduğunu düşündürür. Gerçekte dolap sayısı anahtar sayısından azdır ve iki fiş aynı dolaba düşebilir — çakışma budur. Dictionary o dolabın içinde küçük bir liste tutar ve orada tek tek bakar. Yani O(1), "hiç arama yok" değil, "aranacak yer çok küçük" demektir. Kötü bir `GetHashCode` herkesi tek dolaba doldurursa O(1) sessizce O(n)'e döner.

### Akrabaları
- **`SortedDictionary<K,V>`** — Anahtarlar sıralı tutulur. Erişim O(log n), gezerken sıralı gelir.
- **`ConcurrentDictionary<K,V>`** — Thread güvenli. Çoklu thread'in aynı anda yazdığı senaryolarda (cache) gerekir.

```csharp
// ConcurrentDictionary'de tipik cache kalıbı — anahtar yoksa üretir, varsa mevcudu verir
var cache = new System.Collections.Concurrent.ConcurrentDictionary<int, Urun>();
var urun = cache.GetOrAdd(42, id => VeritabanindanGetir(id));
```

---

## 5. Diğer Koleksiyonlar

> **Benzetme —** Mutfak çekmecesini aç. İçinde kepçe, maşa, süzgeç, rende var. Hiçbiri "en iyi alet" değil; her biri tek bir işi çok iyi yapmak için üretilmiş. Süzgeçle çorba karıştırmaya çalışırsan alet suçlu olmaz. Aşağıdaki koleksiyonlar da öyle: her biri belirli bir soruya cevap vermek için var.

**Basitçe:** `List` ve `Dictionary` işlerin çoğunu görür. Ama bazı ihtiyaçlar o kadar sık tekrar eder ki, .NET onlar için hazır kaplar koymuştur: tekrar istemiyorsan, sırayla işleyeceksen, en son ekleneni önce alacaksan. Bunları bilmenin faydası hız değil, niyetini koda yazabilmektir: `Queue` gören biri "bu sırayla işlenecek" diye okur.

**Teknik olarak:**

**`HashSet<T>`** — Tekrarsız eleman kümesi. `Add`, `Contains`, `Remove` hepsi **O(1)**. Sıra garantisi yoktur. Küme işlemleri hazır gelir: `UnionWith`, `IntersectWith`, `ExceptWith`.
*Ne zaman:* "Bu değeri daha önce gördüm mü?" sorusunu çok kez soracaksan.

```csharp
var a = new HashSet<int> { 1, 2, 3, 4 };
var b = new HashSet<int> { 3, 4, 5 };

Console.WriteLine(a.Add(2));      // False — zaten var, sessizce reddeder
a.IntersectWith(b);               // a = {3, 4}
```

**`Queue<T>`** — FIFO (ilk giren ilk çıkar). `Enqueue` ekler, `Dequeue` alır.
*Ne zaman:* İş kuyruğu, sırayla işlenecek mesajlar, genişlik öncelikli gezinme.

```csharp
var kuyruk = new Queue<string>();
kuyruk.Enqueue("ilk"); kuyruk.Enqueue("ikinci");
Console.WriteLine(kuyruk.Peek());      // "ilk"  — bakar, çıkarmaz
Console.WriteLine(kuyruk.Dequeue());   // "ilk"  — çıkarır
kuyruk.TryDequeue(out var s);          // boşsa patlamaz, false döner
```

**`Stack<T>`** — LIFO (son giren ilk çıkar). `Push` ekler, `Pop` alır.
*Ne zaman:* Geri alma (undo) geçmişi, ifade ayrıştırma, derinlik öncelikli gezinme.

```csharp
var geriAl = new Stack<string>();
geriAl.Push("metin yazıldı");
geriAl.Push("renk değişti");
Console.WriteLine(geriAl.Pop());       // "renk değişti" — en son yapılan ilk geri alınır
```

**`LinkedList<T>`** — Çift yönlü bağlı liste. Elinde düğüm varsa araya ekleme/silme **O(1)**; ama indeksle erişim **O(n)** ve bellekte dağınık durduğu için CPU önbelleği açısından verimsizdir.
*Ne zaman:* Nadiren. Pratikte `List<T>` neredeyse her zaman daha hızlıdır.

**`SortedList<K,V>`** — Sıralı tutar, bellek açısından `SortedDictionary`'den ekonomik, ekleme daha pahalı.

**`PriorityQueue<TElement,TPriority>`** — .NET 6 ile geldi. Sıraya değil önceliğe göre çıkarır; en düşük öncelik değeri önce gelir.
*Ne zaman:* İş sıralaması, zamanlayıcı, en kısa yol algoritmaları.

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("normal is", 5);
pq.Enqueue("acil is", 1);
Console.WriteLine(pq.Dequeue());       // "acil is" — küçük sayı = yüksek öncelik
```

---

## 6. Koleksiyon Arayüzleri Hiyerarşisi

> **Benzetme —** İşe eleman alırken hangi belgeyi şart koştuğun önemlidir. "Ehliyeti olsun" dersen aday havuzun geniştir. "Ağır vasıta ehliyeti, SRC belgesi ve beş yıl deneyim olsun" dersen aday sayısı düşer. Şoför koltuğuna oturtmayacaksan ağır vasıta ehliyeti istemek, sadece kendi seçeneklerini daraltmaktır. Metot parametreleri de böyledir: gerektiğinden fazlasını istemek, kendini kısıtlamaktır.

**Basitçe:** Koleksiyonların arayüzleri bir merdiven gibidir. En altta "sadece gezebilirsin" var, üstüne "sayısını da öğrenebilirsin" biniyor, en üstte "numarayla da erişebilirsin" var. Bir metot yazarken bu merdivenin en alt basamağını istemeye çalış: o zaman çağıran taraf elindeki her şeyi verebilir. Veri döndürürken ise tersi geçerli — karşı tarafa kullanışlı bir şey ver.

**Teknik olarak:**

```
IEnumerable<T>              ← gezilebilir. Tek yetenek: foreach
      ▲
ICollection<T>              ← + Count, Add, Remove, Contains
      ▲
   IList<T>                 ← + indeksle erişim, Insert, RemoveAt
```

Yan dallar: `IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `IDictionary<K,V>`, `ISet<T>`.

| Arayüz | Ne garanti eder | Ne zaman parametre tipi olarak seçilir |
|---|---|---|
| `IEnumerable<T>` | Sadece gezilebilirlik | Sen sadece okuyup gezeceksen — **en esnek seçim** |
| `ICollection<T>` | Sayı + ekleme/çıkarma | Eleman sayısına bakacak veya ekleyeceksen |
| `IList<T>` | Sıra + indeksle erişim | Konum önemliyse |
| `IReadOnlyList<T>` | Sadece okuma + indeks | Dışarıya veri verirken, değiştirilmesini istemiyorsan |

**Genel ilke:** *Parametre olarak en az yetenekli arayüzü iste, dönüş değeri olarak en kullanışlısını ver.* Metodun sadece gezecekse `IEnumerable<T>` iste — o zaman çağıran taraf `List`, dizi, `HashSet` ya da bir LINQ sorgusu geçebilir.

```csharp
// Dar parametre — sadece List kabul eder
decimal Topla(List<Urun> urunler) => urunler.Sum(u => u.Fiyat);

// Geniş parametre — dizi, HashSet, LINQ sorgusu, hepsi geçer
decimal Topla(IEnumerable<Urun> urunler) => urunler.Sum(u => u.Fiyat);

Topla(new[] { u1, u2 });                       // dizi
Topla(urunSeti);                               // HashSet
Topla(tumUrunler.Where(u => u.Aktif));         // LINQ sorgusu
```

Bir uyarı: `IEnumerable<T>` alan metotta koleksiyonu **birden fazla kez gezme**. Kaynak tembel bir sorguysa her geziş yeniden çalışır.

```csharp
// Kötü — kaynak iki kez gezilir
decimal Ortalama(IEnumerable<Urun> urunler) => urunler.Sum(u => u.Fiyat) / urunler.Count();

// İyi — bir kez maddeleştir, sonra kullan
decimal Ortalama2(IEnumerable<Urun> urunler)
{
    var liste = urunler as IReadOnlyList<Urun> ?? urunler.ToList();
    return liste.Count == 0 ? 0 : liste.Sum(u => u.Fiyat) / liste.Count;
}
```

---

## 7. `IEnumerable` ve `IQueryable` — En Kritik Ayrım

> **Benzetme —** Marketten 1000 liranın üzerindeki ürünleri almak istiyorsun. İki yol var. Birincisi: markete listeyi verirsin, market kendi deposunda ayıklar ve sana sadece o ürünleri getirir. İkincisi: bütün depoyu kamyonla evine taşıttırırsın, salonda tek tek bakıp 1000 liranın altındakileri geri gönderirsin. İkisi de sonuçta doğru ürünleri verir. Farkı, kamyonu ve salonunu görene kadar anlamazsın.

**Basitçe:** Aynı görünen iki şey var. Biri, elindeki verinin üzerinde çalışır — veri zaten bellekte, filtreleme senin programında olur. Öteki, henüz alınmamış verinin üzerinde çalışır — filtreleme veritabanına anlatılır ve orada yapılır. Yazarken ikisi de aynı şekilde yazılır. Bu yüzden yanlışlıkla birinciye geçmek çok kolaydır, ve geçtiğin an tüm tablo evine taşınır.

**Teknik olarak:** Bu ayrım Hafta 3'te EF Core'da başına iş açacak konudur; şimdiden oturt.

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Nerede çalışır | **Bellekte** (uygulama tarafında) | **Kaynakta** (veritabanında) |
| Sorgu neyle temsil edilir | Delegate (derlenmiş metot) | **Expression tree** (kod, veri olarak) |
| Filtre ne zaman uygulanır | Veri belleğe geldikten sonra | SQL'e çevrilip veritabanında |
| Tipik kullanım | Listeler, diziler, LINQ to Objects | EF Core, LINQ to SQL |

**Expression tree** — Kodun kendisinin veri yapısı olarak temsil edilmesi. EF Core, `Where(x => x.Fiyat > 100)` ifadesini çalıştırmaz; onu **okuyup** `WHERE Fiyat > 100` SQL'ine çevirir. `IQueryable`'ın sırrı budur.

İkisinin imzasındaki fark, tüm hikâyeyi anlatır:

```csharp
// LINQ to Objects — lambda derlenmiş bir metottur, çalıştırılır
IEnumerable<T> Where<T>(this IEnumerable<T> src, Func<T, bool> predicate);

// LINQ to Entities — lambda bir veri yapısıdır, okunur ve çevrilir
IQueryable<T>  Where<T>(this IQueryable<T> src, Expression<Func<T, bool>> predicate);
```

`Expression<Func<...>>`'ın gerçekten veri olduğunu görebilirsin:

```csharp
System.Linq.Expressions.Expression<Func<Urun, bool>> ifade = u => u.Fiyat > 1000;
Console.WriteLine(ifade.Body);         // (u.Fiyat > 1000)
Console.WriteLine(ifade.Body.NodeType);// GreaterThan
// EF Core tam olarak bunu okuyup SQL üretir.
```

```csharp
// IQueryable — filtreleme veritabanında olur, 5 satır gelir
var pahalilar = context.Urunler
    .Where(u => u.Fiyat > 1000)
    .ToList();
// SQL: SELECT * FROM Urunler WHERE Fiyat > 1000

// IEnumerable'a düşürüldü — TÜM tablo belleğe çekilir, sonra filtrelenir
var pahalilar2 = context.Urunler
    .AsEnumerable()
    .Where(u => u.Fiyat > 1000)
    .ToList();
// SQL: SELECT * FROM Urunler        ← 2 milyon satır
```

İkinci kod **doğru sonucu üretir** ama üretim ortamında uygulamayı düşürür. Bootcamp projelerinde en sık rastlanan performans hatası budur.

**`AsEnumerable()` / `ToList()` bir sınırdır.** Bu noktadan sonraki her LINQ çağrısı bellekte çalışır. Bu yüzden `ToList()`'i sorgunun **en sonuna** koy.

Sınırı istemeden geçmenin üç klasik yolu:

```csharp
// 1) Sayfalama ToList'ten SONRA yapılırsa tüm tablo gelir
var sayfa = context.Urunler.ToList().Skip(100).Take(20);        // yanlış
var sayfa2 = context.Urunler.Skip(100).Take(20).ToList();       // doğru

// 2) Değişkeni IEnumerable olarak tutmak sessizce tipi düşürür
IEnumerable<Urun> sorgu = context.Urunler;
var sonuc = sorgu.Where(u => u.Aktif).ToList();                 // filtre bellekte

// 3) SQL'e çevrilemeyen bir C# metodu çağırmak
var hatali = context.Urunler.Where(u => KendiKontrolum(u)).ToList();
// EF Core bunu çeviremez -> çalışma anında hata verir
```

Sorgunun gerçekten neye çevrildiğini merak ediyorsan EF Core sana gösterir:

```csharp
var sorgu = context.Urunler.Where(u => u.Fiyat > 1000);
Console.WriteLine(sorgu.ToQueryString());   // üretilecek SQL'i yazdırır
```

> **Bu benzetme şurada bozulur:** Market örneği, "her zaman markette ayıklatmak daha iyi" izlenimi verir. Her zaman değil. Veritabanının yapamayacağı ya da çok pahalıya yapacağı işler vardır — karmaşık metin işleme, kendi yazdığın hesaplama metotları gibi. Böyle durumlarda doğru yöntem, **önce veritabanında olabildiğince daraltmak**, sonra bilinçli olarak `AsEnumerable()` deyip kalan küçük kümeyi bellekte işlemektir. Hata, `AsEnumerable()` kullanmak değil; onu daraltmadan önce kullanmaktır.

---

## 8. Tembel Üretim ve `yield return`

> **Benzetme —** Musluk ile kova arasındaki fark. Kova, suyu önce doldurup sonra taşımandır: hepsi hazırdır ama yeri kaplar ve dolmasını beklemen gerekir. Musluk ise sen istedikçe akar; bir bardak istiyorsan bir bardak akar, vazgeçersen kapatırsın. `yield return` musluktur. Bir de şu var: musluğu ikinci kez açtığında su baştan akar, birinci seferki suyu hatırlamaz.

**Basitçe:** Bir metot normalde bütün sonucu hazırlayıp öyle döner. `yield return` ile yazdığında ise sonuçları tek tek, istendikçe üretir. Bu, milyonlarca satırlık veriyi belleğe sığdırmadan işleyebilmeni sağlar. Bedeli şu: ürettiği şey bir liste değil, bir üretim talimatıdır — her gezişte baştan çalışır.

**Teknik olarak:** **Deferred execution (ertelenmiş çalıştırma)** — LINQ sorgusu tanımlandığında değil, **sonucu istendiğinde** çalışır. Bu yarın detaylı işlenecek; koleksiyon tarafındaki karşılığı `yield return`'dür.

**`yield return`** — Bir metodun elemanları **tek tek, istendikçe** üretmesini sağlar. Tüm listeyi bellekte kurmaz.

```csharp
IEnumerable<int> SonsuzSayilar()
{
    int i = 0;
    while (true)
        yield return i++;      // her istendiğinde bir sonrakini üretir
}

var ilkOn = SonsuzSayilar().Take(10).ToList();   // sadece 10 üretilir
```

Metodun gövdesinin ne zaman çalıştığını görmek, kavramı oturtur:

```csharp
IEnumerable<int> Uret()
{
    Console.WriteLine("basladi");
    yield return 1;
    Console.WriteLine("ara");
    yield return 2;
}

var q = Uret();                 // hiçbir şey yazılmaz — gövde henüz çalışmadı
foreach (var x in q)            // "basladi" burada yazılır
    Console.WriteLine(x);
```

Faydası: milyonlarca satırlık bir dosyayı satır satır işlerken belleğe tamamını almazsın.
Bedeli: koleksiyon **her gezildiğinde yeniden üretilir**. Aynı sorguyu üç kez `foreach`'lersen üç kez çalışır.

```csharp
// Tüm dosyayı belleğe alır — 2 GB'lık log dosyasında çöker
string[] tumSatirlar = File.ReadAllLines("buyuk.log");

// Satır satır akıtır — bellek sabit kalır
IEnumerable<string> akan = File.ReadLines("buyuk.log");
var hatalar = akan.Where(s => s.Contains("ERROR")).Take(100).ToList();
```

Kendi tembel dönüştürücünü yazmak da aynı kalıptadır:

```csharp
IEnumerable<Kayit> Ayristir(string yol)
{
    foreach (var satir in File.ReadLines(yol))     // kaynak da tembel
    {
        if (string.IsNullOrWhiteSpace(satir)) continue;
        yield return Kayit.Parse(satir);           // tek tek üretilir
    }
}
```

Bu kalıbın en sık düşülen tuzağı, tembel sonucu değişken olarak tutup birden çok kez kullanmaktır:

```csharp
var pahalilar = urunler.Where(u => u.Fiyat > 1000);   // henüz çalışmadı
Console.WriteLine(pahalilar.Count());                 // 1. geziş
Console.WriteLine(pahalilar.First().Ad);              // 2. geziş — filtre tekrar çalıştı
var liste = pahalilar.ToList();                       // 3. geziş
// Birden fazla kullanacaksan baştan ToList() de, sonra listeyi kullan.
```

---

## 9. Salt Okunur ve Değişmez Koleksiyonlar

> **Benzetme —** Vitrine konmuş bir ürün ile satın alıp eve götürdüğün ürün arasındaki fark. Vitrindekine dokunamazsın, ama mağaza görevlisi istediği an onu değiştirebilir; sen sadece bakma hakkına sahipsin, ürünün sabit kalacağına dair bir garantin yok. Eve götürdüğün ürün ise gerçekten senindir, kimse ona dokunamaz.

**Basitçe:** "Değiştirilemez" diye görünen üç ayrı şey var ve aralarında ciddi fark var. Birincisi sadece *senin* değiştirmeni engeller, altındaki veri başkası tarafından değişebilir. İkincisi de aynı kapıya çıkar. Üçüncüsü ise gerçek garantidir: veri hiç değişmez, "değiştirmek" istediğinde sana yeni bir kopya verilir. Sınıfının içindeki listeyi dışarıya verirken bu farkı bilmek önemli.

**Teknik olarak:** Karıştırılan üç kavram:

| Tip | Ne sağlar |
|---|---|
| **`IReadOnlyList<T>`** | Sadece arayüz kısıtı. Alttaki `List` başka biri tarafından değiştirilebilir — **görünüm**, garanti değil |
| **`ReadOnlyCollection<T>`** | Sarmalayıcı. Yine alttaki liste değişirse yansır |
| **`ImmutableList<T>`** | Gerçek değişmezlik. Her "değişiklik" yeni bir koleksiyon döndürür. Thread güvenlidir |

Sınıfının içindeki listeyi dışarıya `List<T>` olarak vermek, dışarıdakinin senin verini bozmasına izin vermektir. Kapsülleme için en az `IReadOnlyList<T>` döndür.

Üçünün farkı kodda net görünür:

```csharp
var kaynak = new List<int> { 1, 2, 3 };

IReadOnlyList<int> gorunum = kaynak;          // sadece arayüz kısıtı
kaynak.Add(4);
Console.WriteLine(gorunum.Count);             // 4  — alttaki değişti, yansıdı

var sarmal = kaynak.AsReadOnly();             // ReadOnlyCollection<int>
kaynak.Add(5);
Console.WriteLine(sarmal.Count);              // 5  — yine yansıdı

var degismez = System.Collections.Immutable.ImmutableList.CreateRange(kaynak);
kaynak.Add(6);
Console.WriteLine(degismez.Count);            // 5  — anlık görüntü, etkilenmez

var yeni = degismez.Add(99);                  // yeni koleksiyon döner
Console.WriteLine(degismez.Count);            // 5  — orijinal hâlâ aynı
Console.WriteLine(yeni.Count);                // 6
```

Kapsülleme kalıbının doğru hâli:

```csharp
public class Siparis
{
    private readonly List<Kalem> _kalemler = new();

    // Dışarıya yazma yetkisi vermez; ekleme kuralları sınıfın içinde kalır
    public IReadOnlyList<Kalem> Kalemler => _kalemler;

    public void KalemEkle(Kalem k)
    {
        if (k.Adet <= 0) throw new ArgumentException("Adet pozitif olmalı");
        _kalemler.Add(k);
    }
}
```

> **Bu benzetme şurada bozulur:** Vitrin örneği, `IReadOnlyList` döndürmenin işe yaramadığını düşündürebilir. Yaramaz değil — niyeti belgeler ve kazara yapılan değişikliği derleme anında engeller. Ama kötü niyetli ya da dikkatsiz bir kullanıcı `(List<T>)gorunum` diye cast edip yazabilir. Yani `IReadOnlyList` bir sözleşmedir, kilit değildir. Gerçek kilidi `ImmutableList` verir; bedeli her değişiklikte yeni koleksiyon üretmektir.

---

## 10. Hangi Durumda Hangisi

> **Benzetme —** Alet çantası açıkken hangisini alacağına karar verirken alete değil, vidaya bakarsın. Yıldız mı düz mü, kaç numara. Koleksiyon seçerken de koleksiyonun özelliklerini değil, kendi işini tarif etmen gerekir: ne yapacağım, en çok hangi işlemi yapacağım.

**Basitçe:** Aşağıdaki tablo bir ezber listesi değil, bir soru listesi. Kendine "bu veriyle en çok ne yapacağım" diye sor; cevabı tablodan bul. Kararsız kaldığında `List<T>` ile başla, darboğaz gördüğünde değiştir — ama darboğazı ölçerek gör, tahmin ederek değil.

**Teknik olarak:**

| İhtiyaç | Seçim |
|---|---|
| Sıralı liste, indeksle erişim | `List<T>` |
| Anahtarla hızlı erişim | `Dictionary<K,V>` |
| Tekrarsızlık ve "var mı" kontrolü | `HashSet<T>` |
| Sırayla işlenecek iş kuyruğu | `Queue<T>` |
| Geri alma / son eklenen önce | `Stack<T>` |
| Önem sırasına göre işlenecek işler | `PriorityQueue<T,TPriority>` |
| Anahtarlar hep sıralı gelsin | `SortedDictionary<K,V>` |
| Çoklu thread'den yazılıyor | `ConcurrentDictionary<K,V>` |
| Sabit boyut, maksimum performans | `T[]` (dizi) |
| Metot parametresi, sadece gezeceksin | `IEnumerable<T>` |
| Dışarıya veri döndürüyorsun | `IReadOnlyList<T>` |
| Paylaşılan, hiç değişmemesi gereken veri | `ImmutableList<T>` / `ImmutableArray<T>` |
| Veritabanı sorgusu kuruyorsun | `IQueryable<T>` — `ToList()`'i sona bırak |

---

## Tek Bakışta Özet

- Koleksiyon seçimi bir performans kararıdır; **hangi işlemi çok yapacağını** sor.
- `List<T>` içeride dizidir: indeks O(1), arama ve ortaya ekleme O(n).
- `Dictionary` ve `HashSet` hash tabanlıdır: erişim O(1). Anahtar tipinde `GetHashCode`/`Equals` tutarlı olmalı.
- Listede arama yapan döngü O(n²) tuzağıdır; `HashSet`'e çevir.
- Arayüzlerde kural: **parametrede en az yetenekli, dönüşte en kullanışlı**.
- `IEnumerable` bellekte, `IQueryable` veritabanında çalışır. `AsEnumerable()`/`ToList()` bu sınırı geçer.
- `yield return` tembel üretim sağlar; koleksiyon her gezilişte yeniden üretilir.
- `IReadOnlyList` bir görünümdür, `ImmutableList` gerçek garantidir.
- Tembel bir sorguyu birden çok kez gezme; bir kez `ToList()` de, sonra listeyi kullan.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Big-O | Maliyetin eleman sayısıyla büyüme hızı |
| Amortized O(1) | Çoğu işlem sabit, ara sıra pahalı (List'in büyümesi gibi) |
| Capacity / Count | Ayrılmış alan / gerçek eleman sayısı |
| Hash tablosu | Anahtardan adres hesaplayarak O(1) erişim sağlayan yapı |
| Hash collision | İki anahtarın aynı indekse düşmesi |
| FIFO / LIFO | İlk giren ilk çıkar / son giren ilk çıkar |
| IEnumerable | Yalnızca gezilebilirlik sunan temel koleksiyon arayüzü |
| IQueryable | Sorguyu ifade ağacı olarak taşıyan, kaynağa çeviren arayüz |
| Expression tree | Kodun veri yapısı olarak temsili |
| Deferred execution | Sorgunun tanımlandığında değil, sonuç istendiğinde çalışması |
| Materialization | Tembel sorgunun `ToList`/`ToArray` ile belleğe alınması |
| Multiple enumeration | Aynı tembel sorgunun birden çok kez gezilip tekrar çalışması |
| yield return | Elemanları istendikçe tek tek üreten mekanizma |
| Immutable koleksiyon | Değiştirilemeyen, her işlemde yenisini döndüren koleksiyon |
| Thread-safe koleksiyon | Eşzamanlı erişimde tutarlılığı garanti eden koleksiyon |
| Cache locality | Bitişik bellekte duran verinin CPU önbelleğinden hızlı okunması |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`List.Contains` hızlıdır" | O(n)'dir. Sık arama yapacaksan `HashSet` kullan |
| "`IEnumerable` döndürmek her zaman iyidir" | Çağıran birden çok kez gezerse sorgu her seferinde yeniden çalışır |
| "`ToList()` sorguyu hızlandırır" | Sadece sonucu belleğe alır; erken çağrılırsa tüm tabloyu çeker |
| "`LinkedList` ekleme/silmede daha hızlı" | Teorik olarak evet, pratikte önbellek dostu olmadığı için genelde daha yavaş |
| "`Dictionary` sırayı korur" | Garanti etmez. Sıra gerekiyorsa `SortedDictionary` veya liste kullan |
| "O(1) her zaman O(n)'den hızlıdır" | Küçük n'de sabitler baskındır; 20 elemanda `List` taraması genelde daha hızlı |
| "`IReadOnlyList` verimi değiştirilemez yapar" | Sadece arayüz kısıtıdır; alttaki liste değişirse yansır |
| "`dict.Add` ile `dict[key] = x` aynı şey" | `Add` mevcut anahtarda exception atar, `[]` sessizce üzerine yazar |
| "`foreach` içinde silmek sadece yavaştır" | Yavaş değil, hatalıdır — `InvalidOperationException` fırlatır |

---

## Sonraki

→ `03-LINQ.md` (Çarşamba)
