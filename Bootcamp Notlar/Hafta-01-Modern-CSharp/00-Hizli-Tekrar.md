# Hafta 1 — Hızlı Tekrar: Modern C# ve .NET Çalışma Modeli

**Okuma süresi:** ~11 dk

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

1. **.NET Çalışma Modeli, Bellek ve Tip Sistemi** — `01-NET-Calisma-Modeli.md` · Pazartesi
2. **Koleksiyonlar ve Koleksiyon Arayüzleri** — `02-Koleksiyonlar.md` · Salı
3. **LINQ** — `03-LINQ.md` · Çarşamba
4. **Asenkron Programlama** — `04-Asenkron-Programlama.md` · Perşembe
5. **Modern C# Özellikleri** — `05-Modern-CSharp-Ozellikleri.md` · Cuma
6. **Generics, Delegate, Extension, Exception, IDisposable** — `06-Dil-Altyapisi.md` · Cumartesi
7. **Haftalık Tekrar Özeti** — `07-Hafta-Ozeti.md` · Pazar

---

## 1. .NET Çalışma Modeli, Bellek ve Tip Sistemi

*Kaynak: [`01-NET-Calisma-Modeli.md`](01-NET-Calisma-Modeli.md) · Pazartesi*

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

### Sık karıştırılanlar

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


---

## 2. Koleksiyonlar ve Koleksiyon Arayüzleri

*Kaynak: [`02-Koleksiyonlar.md`](02-Koleksiyonlar.md) · Salı*

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

### Sık karıştırılanlar

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


---

## 3. LINQ

*Kaynak: [`03-LINQ.md`](03-LINQ.md) · Çarşamba*

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

### Sık karıştırılanlar

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


---

## 4. Asenkron Programlama

*Kaynak: [`04-Asenkron-Programlama.md`](04-Asenkron-Programlama.md) · Perşembe*

- Eşzamanlılık ≠ paralellik ≠ asenkronluk. `async` **asenkronluk** içindir.
- `await`, thread'i havuza geri verir; asenkronluk hız değil **ölçek** kazandırır.
- `Task` bir iş temsilidir, thread değildir. `Task.Run` ise gerçekten thread kullanır.
- `async`, metodu asenkron yapmaz; derleyiciye durum makinesi ürettirir. İşi yapan `await`'tir.
- `await` edilen iş zaten bitmişse duraklama olmaz, akış senkron devam eder.
- `.Result` / `.Wait()` deadlock ve thread pool açlığı üretir — buna sync-over-async denir.
- Deadlock'un şartı bir `SynchronizationContext`'tir; ASP.NET Core'da yoktur ama havuz açlığı vardır.
- `async void` yakalanamayan hata üretir; UI dışında kullanılmaz.
- Döngü içinde `await` işleri sıraya sokar; bağımsızsa `Task.WhenAll` — ama eşzamanlılığı sınırla.
- `CancellationToken` gereksiz işi keser — alınmalı ve **geçirilmeli**.
- `WhenAll` yalnızca ilk hatayı fırlatır; hepsi `AggregateException` içindedir.
- `ConfigureAwait(false)` kütüphane kodu içindir; ASP.NET Core'da gerekmez ve `.Result`'ı güvenli hâle getirmez.
- `await` edilmeyen `Task`'taki exception sessizce kaybolur.
- `IAsyncEnumerable<T>`, büyük sonuçları belleğe almadan geldikçe işlemeni sağlar.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`async` kodu hızlandırır" | Tek işlemi hızlandırmaz; eşzamanlı kapasiteyi artırır |
| "Her `Task` bir thread kullanır" | I/O `Task`'ları beklerken hiçbir thread çalışmaz |
| "`await` thread'i bloke eder" | Tam tersi — thread'i serbest bırakır |
| "`async` kelimesi metodu asenkron yapar" | Yapmaz; işi `await` yapar, `async` sadece derleyiciye izin verir |
| "`await` her zaman duraklar" | İş zaten bittiyse duraklamaz, senkron devam eder |
| "`Task.Run` her şeyi hızlandırır" | I/O işinde gereksiz thread harcar; CPU-bound için vardır |
| "ASP.NET Core'da deadlock olmaz, `.Result` güvenli" | Deadlock riski azalır ama thread pool açlığı devam eder |
| "`ConfigureAwait(false)` her yerde şart" | ASP.NET Core'da gerekmez; kütüphane kodunda anlamlıdır |
| "`ConfigureAwait(false)` yazarsam `.Result` güvenli olur" | Olmaz; thread yine bloke olur, havuz yine tükenir |
| "`CancellationToken` parametresi koymak yeter" | Aşağıya geçirilmezse hiçbir şey iptal edilmez |
| "`Task.WhenAny` diğer işleri durdurur" | Durdurmaz; arka planda çalışmaya devam ederler |
| "`WhenAll` tüm hataları fırlatır" | `await` yalnızca ilkini verir; tamamı `Exception` içindedir |
| "`ValueTask` her yerde daha iyidir" | Kuralları katıdır; varsayılan `Task` olmalı |
| "`async void` da `Task` gibi çalışır" | Beklenemez, hatası yakalanamaz, testi yazılamaz |

---


---

## 5. Modern C# Özellikleri

*Kaynak: [`05-Modern-CSharp-Ozellikleri.md`](05-Modern-CSharp-Ozellikleri.md) · Cuma*

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

### Sık karıştırılanlar

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


---

## 6. Generics, Delegate, Extension, Exception, IDisposable

*Kaynak: [`06-Dil-Altyapisi.md`](06-Dil-Altyapisi.md) · Cumartesi*

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

### Sık karıştırılanlar

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


---

## 7. Haftalık Tekrar Özeti

*Kaynak: [`07-Hafta-Ozeti.md`](07-Hafta-Ozeti.md) · Pazar*

> Bu notun ayrı bir özet bölümü yok — kendisi zaten haftanın
> damıtılmış hâli. Doğrudan dosyayı aç.


---

## Sonraki

Bulanık kalan madde varsa yukarıdaki kaynak satırından dosya adını al ve
sadece o bölümü oku. Haftanın tamamını yeniden okumana gerek yok.

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
