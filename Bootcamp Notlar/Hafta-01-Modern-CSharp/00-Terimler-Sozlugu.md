# Hafta 1 — Terimler Sözlüğü: Modern C# ve .NET Çalışma Modeli

**Ne işe yarar:** Bu dosya baştan sona okunmak için değil, **aranmak** için.
Haftanın 7 notundaki sözlükler burada birleştirildi: **142 terim**.
Aynı terim birden çok notta geçtiyse ilk tanımı alındı; "Nerede" sütunu
terimin ayrıntılı anlatıldığı dosyayı gösterir.

> Ctrl+F ile ara. Bir terimi bulamıyorsan başka haftanın sözlüğünde olabilir.

---

| Terim | Tanım | Nerede |
|---|---|---|
| .NET | Runtime, kütüphane ve araçlardan oluşan geliştirme platformu | `01-NET-Calisma-Modeli.md` |
| Abstract class | Kısmen uygulanmış, örneklenemeyen taban sınıf | `06-Dil-Altyapisi.md` |
| Action / Func / Predicate | Döndürmeyen / döndüren / `bool` döndüren hazır delegate'ler | `06-Dil-Altyapisi.md` |
| AggregateException | Birden çok hatayı içinde taşıyan exception | `04-Asenkron-Programlama.md` |
| Aggregation | Çok elemandan tek değer üretme (`Sum`, `Count`) | `03-LINQ.md` |
| Amortized O(1) | Çoğu işlem sabit, ara sıra pahalı (List'in büyümesi gibi) | `02-Koleksiyonlar.md` |
| Anonim tip | Derleyicinin ürettiği, yerel kullanımlık isimsiz tip | `03-LINQ.md` |
| AOT | Çalışmadan önce doğrudan makine kodu üretme (cold start çözümü) | `01-NET-Calisma-Modeli.md` |
| Assembly | IL + metadata içeren `.dll` / `.exe` | `01-NET-Calisma-Modeli.md` |
| async all the way | Zincirin tamamının asenkron tutulması ilkesi | `04-Asenkron-Programlama.md` |
| BCL | Her .NET uygulamasında hazır gelen temel sınıf kütüphanesi | `01-NET-Calisma-Modeli.md` |
| Big-O | Maliyetin eleman sayısıyla büyüme hızı | `02-Koleksiyonlar.md` |
| Boxing / Unboxing | Value type'ın heap'e taşınması / geri çıkarılması | `01-NET-Calisma-Modeli.md` |
| Buffering operatör | Tüm veriyi görmesi gereken operatör (`OrderBy`, `GroupBy`) | `03-LINQ.md` |
| Cache locality | Bitişik bellekte duran verinin CPU önbelleğinden hızlı okunması | `02-Koleksiyonlar.md` |
| CancellationToken | İptal sinyalini taşıyan yapı | `04-Asenkron-Programlama.md` |
| CancellationTokenSource | İptal sinyalini üreten ve zaman aşımı kurmaya yarayan yapı | `04-Asenkron-Programlama.md` |
| Capacity / Count | Ayrılmış alan / gerçek eleman sayısı | `02-Koleksiyonlar.md` |
| Closure | Lambda'nın dış kapsamdaki değişkeni yakalaması | `06-Dil-Altyapisi.md` |
| Closure (kapanış) | Lambda'nın dışarıdaki değişkeni yakalaması | `03-LINQ.md` |
| CLR | .NET'in çalışma zamanı motoru — bellek, tip güvenliği, exception, thread | `01-NET-Calisma-Modeli.md` |
| Cold start | JIT ve yükleme maliyeti yüzünden ilk isteğin yavaş olması | `01-NET-Calisma-Modeli.md` |
| Concurrency / Parallelism / Asynchrony | İç içe ilerleme / fiziksel eşzamanlılık / beklemeden dönme | `04-Asenkron-Programlama.md` |
| ConfigureAwait(false) | Devam bloğunun orijinal bağlama dönmesini gereksiz kılma | `04-Asenkron-Programlama.md` |
| Constraint | `T`'nin ne olabileceğini sınırlayan kural | `06-Dil-Altyapisi.md` |
| Continuation | `await` sonrası çalışacak devam bloğu | `04-Asenkron-Programlama.md` |
| Covariance / Contravariance | Generic tiplerde okuma / yazma yönünde tip esnekliği | `06-Dil-Altyapisi.md` |
| CTS / CLS | Ortak tip sistemi / diller arası asgari uyumluluk kuralları | `01-NET-Calisma-Modeli.md` |
| Deadlock | Karşılıklı bekleme sonucu oluşan kilitlenme | `04-Asenkron-Programlama.md` |
| Deconstruction | Bir nesneyi parçalarına ayırarak değişkenlere atama | `05-Modern-CSharp-Ozellikleri.md` |
| Default interface method | Arayüzde gövdeli tanımlanan, isteğe bağlı olarak ezilen üye | `06-Dil-Altyapisi.md` |
| Deferred execution | Sorgunun tanımlandığında değil, sonuç istendiğinde çalışması | `02-Koleksiyonlar.md` |
| Delegate | Metoda işaret eden tip | `06-Dil-Altyapisi.md` |
| Dispose pattern | Yönetilen/yönetilmeyen kaynakları ayıran standart temizlik yapısı | `06-Dil-Altyapisi.md` |
| Event | Dışarıdan yalnızca abone olunabilen kapsüllenmiş delegate | `06-Dil-Altyapisi.md` |
| Exception | Olağandışı durumu temsil eden nesne | `06-Dil-Altyapisi.md` |
| Exception filter (`when`) | Yakalamaya koşul ekleyen, stack'i çözmeden değerlendirilen sözdizimi | `06-Dil-Altyapisi.md` |
| Exhaustiveness | Tüm ihtimallerin kapatıldığının derleyici tarafından denetlenmesi | `05-Modern-CSharp-Ozellikleri.md` |
| Explicit implementation | Arayüz üyesinin yalnızca arayüz üzerinden erişilebilir uygulanması | `06-Dil-Altyapisi.md` |
| Expression tree | Kodun veri yapısı olarak temsili | `02-Koleksiyonlar.md` |
| Expression-bodied member | `=>` ile tek satırlık üye tanımı | `05-Modern-CSharp-Ozellikleri.md` |
| Extension method | `this` parametresiyle tipe eklenmiş gibi görünen statik metot | `06-Dil-Altyapisi.md` |
| FIFO / LIFO | İlk giren ilk çıkar / son giren ilk çıkar | `02-Koleksiyonlar.md` |
| File-scoped namespace | Süslü parantezsiz, dosyanın tamamını kapsayan namespace | `05-Modern-CSharp-Ozellikleri.md` |
| Finalizer | GC toplamadan önce çağrılan son çare temizlik metodu | `01-NET-Calisma-Modeli.md` |
| Fire and forget | Başlatılıp sonucu beklenmeyen iş | `04-Asenkron-Programlama.md` |
| Flow analysis | Derleyicinin, o satıra kadarki kontrollere bakarak null durumunu izlemesi | `05-Modern-CSharp-Ozellikleri.md` |
| GC | Erişilemez nesneleri otomatik temizleyen mekanizma | `01-NET-Calisma-Modeli.md` |
| Generation (Gen 0/1/2) | GC'nin nesneleri yaşlarına göre ayırdığı bölümler | `01-NET-Calisma-Modeli.md` |
| Generic | Tipi kullanım anında belirlenen sınıf/metot | `06-Dil-Altyapisi.md` |
| Global using | Tüm dosyalarda geçerli using bildirimi | `05-Modern-CSharp-Ozellikleri.md` |
| GroupJoin | Sol tarafı koruyan, eşleşmeleri grup olarak veren birleştirme | `03-LINQ.md` |
| Hash collision | İki anahtarın aynı indekse düşmesi | `02-Koleksiyonlar.md` |
| Hash sözleşmesi | Eşit nesnelerin aynı hash'i üretmesi kuralı | `06-Dil-Altyapisi.md` |
| Hash tablosu | Anahtardan adres hesaplayarak O(1) erişim sağlayan yapı | `02-Koleksiyonlar.md` |
| Heap | Nesne gövdelerinin tutulduğu, GC tarafından yönetilen bölge | `01-NET-Calisma-Modeli.md` |
| I/O-bound / CPU-bound | Beklemeye dayalı / hesaplamaya dayalı iş | `04-Asenkron-Programlama.md` |
| IAsyncDisposable | Aynı işin asenkron karşılığı — `await using` ile kullanılır | `01-NET-Calisma-Modeli.md` |
| IAsyncEnumerable | Asenkron veri akışı, `await foreach` ile gezilir | `04-Asenkron-Programlama.md` |
| IDisposable | Yönetilmeyen kaynakların deterministik bırakılması arayüzü | `01-NET-Calisma-Modeli.md` |
| IEnumerable | Yalnızca gezilebilirlik sunan temel koleksiyon arayüzü | `02-Koleksiyonlar.md` |
| IGrouping | Anahtarı olan eleman kümesi — `GroupBy` çıktısı | `03-LINQ.md` |
| IL / MSIL / CIL | Derleyicinin ürettiği, CPU'dan bağımsız ara dil | `01-NET-Calisma-Modeli.md` |
| Immutable | Oluşturulduktan sonra içeriği değiştirilemeyen | `01-NET-Calisma-Modeli.md` |
| Immutable koleksiyon | Değiştirilemeyen, her işlemde yenisini döndüren koleksiyon | `02-Koleksiyonlar.md` |
| Implicit usings | Proje tipine göre otomatik eklenen using kümesi | `05-Modern-CSharp-Ozellikleri.md` |
| init | Yalnızca nesne oluşturulurken atanabilen özellik | `05-Modern-CSharp-Ozellikleri.md` |
| Inner exception | Bir hataya sebep olan alttaki hata | `06-Dil-Altyapisi.md` |
| Interface | "Ne yapabilir" sözleşmesi | `06-Dil-Altyapisi.md` |
| Invariant | Ne covariant ne contravariant olabilen generic tip (`List<T>`) | `06-Dil-Altyapisi.md` |
| IQueryable | Sorguyu ifade ağacı olarak taşıyan, kaynağa çeviren arayüz | `02-Koleksiyonlar.md` |
| JIT | IL'i metot ilk çağrıldığında makine koduna çeviren derleyici | `01-NET-Calisma-Modeli.md` |
| Keyset pagination | `Skip` yerine "son görülen anahtardan sonrası" ile sayfalama | `03-LINQ.md` |
| Koleksiyon ifadesi | `[1, 2, 3]` yazımıyla koleksiyon oluşturma | `05-Modern-CSharp-Ozellikleri.md` |
| Lambda | İsimsiz metot sözdizimi | `06-Dil-Altyapisi.md` |
| Lambda ifadesi | `x => ...` biçiminde yazılan isimsiz metot | `03-LINQ.md` |
| LINQ | Dil içine gömülü sorgulama altyapısı | `03-LINQ.md` |
| Liste deseni | Dizi/liste şekline göre eşleme (`[1, 2, ..]`) | `05-Modern-CSharp-Ozellikleri.md` |
| Local function | Metot içinde tanımlı yardımcı metot | `05-Modern-CSharp-Ozellikleri.md` |
| LOH | 85 KB üstü nesnelere ayrılmış, sıkıştırılmayan heap bölgesi | `01-NET-Calisma-Modeli.md` |
| Managed code | CLR gözetiminde çalışan, belleği GC tarafından yönetilen kod | `01-NET-Calisma-Modeli.md` |
| Materialization | Tembel sorgunun `ToList`/`ToArray` ile belleğe alınması | `02-Koleksiyonlar.md` |
| Memory leak | Artık gerekmeyen nesnelerin referansla hayatta tutulması | `01-NET-Calisma-Modeli.md` |
| Metadata | Tiplerin ve üyelerin çalışma anında okunabilir tanımı | `01-NET-Calisma-Modeli.md` |
| Method / Query syntax | Zincirleme metot çağrısı / SQL benzeri sözdizimi | `03-LINQ.md` |
| Metot grubu dönüşümü | Bir metot adının doğrudan delegate'e atanması | `06-Dil-Altyapisi.md` |
| Multicast delegate | Birden çok metot tutabilen delegate | `06-Dil-Altyapisi.md` |
| Multiple enumeration | Aynı tembel sorgunun birden çok kez gezilip tekrar çalışması | `02-Koleksiyonlar.md` |
| N+1 problemi | Bir sorgu + her satır için ek sorgu üreten kalıp | `03-LINQ.md` |
| nameof | Üye adını string olarak veren derleme anı işleci | `05-Modern-CSharp-Ozellikleri.md` |
| NRT | Nullable reference types — null analizini derlemeye taşıyan özellik | `05-Modern-CSharp-Ozellikleri.md` |
| Null durum özniteliği | `[NotNullWhen]` gibi, metodun null sözleşmesini derleyiciye bildiren işaret | `05-Modern-CSharp-Ozellikleri.md` |
| Null-forgiving (`!`) | Null uyarısını bastıran, çalışma anı etkisi olmayan işleç | `05-Modern-CSharp-Ozellikleri.md` |
| Pattern matching | Tip/şekil/içerik eşlemesi ve parça yakalama | `05-Modern-CSharp-Ozellikleri.md` |
| Primary constructor | Sınıf tanımının yanında parametre bildirimi | `05-Modern-CSharp-Ozellikleri.md` |
| Projection | Elemanı başka bir şekle dönüştürme (`Select`) | `03-LINQ.md` |
| Promise task / delegate task | I/O bekleyen Task / `Task.Run` ile thread'de çalışan Task | `04-Asenkron-Programlama.md` |
| Provider | LINQ sorgusunu belirli bir kaynağa çeviren bileşen | `03-LINQ.md` |
| Quantifier | Varlık/kapsam sorgusu (`Any`, `All`, `Contains`) | `03-LINQ.md` |
| Range / Index | `[2..5]` ve `[^1]` ile dilim ve sondan erişim | `05-Modern-CSharp-Ozellikleri.md` |
| Raw string literal | Üç tırnaklı, kaçışsız çok satırlı metin | `05-Modern-CSharp-Ozellikleri.md` |
| record | Değer eşitliğine sahip, DTO'lar için ideal reference type | `01-NET-Calisma-Modeli.md` |
| record struct | Değer eşitlikli value type | `05-Modern-CSharp-Ozellikleri.md` |
| Reference type | Atamada adresi kopyalanan tip (`class`, `string`, diziler...) | `01-NET-Calisma-Modeli.md` |
| Reified generics | Generic tip bilgisinin çalışma anında da korunması | `06-Dil-Altyapisi.md` |
| required | Nesne oluşturulurken atanması zorunlu üye | `05-Modern-CSharp-Ozellikleri.md` |
| SelectMany | İç içe koleksiyonları tek düzeye indirme | `03-LINQ.md` |
| SemaphoreSlim | Aynı anda çalışacak iş sayısını sınırlayan kapı | `04-Asenkron-Programlama.md` |
| Server GC | Çekirdek başına ayrı heap kullanan, sunucu için varsayılan GC modu | `01-NET-Calisma-Modeli.md` |
| Spread (`..`) | Bir koleksiyonun elemanlarını başka bir koleksiyona açma | `05-Modern-CSharp-Ozellikleri.md` |
| Stack | Metot çerçeveleri ve yerel değişkenler için LIFO bellek bölgesi | `01-NET-Calisma-Modeli.md` |
| Stack frame | Bir metot çağrısının stack'te kapladığı blok | `01-NET-Calisma-Modeli.md` |
| Stack trace | Hatanın hangi çağrı zincirinden geldiğini gösteren döküm | `06-Dil-Altyapisi.md` |
| State machine | Derleyicinin `async` metottan ürettiği durum makinesi | `04-Asenkron-Programlama.md` |
| Static lambda | Dış değişken yakalaması derleyici tarafından engellenen lambda | `06-Dil-Altyapisi.md` |
| Stop-the-world | GC toplaması sırasında thread'lerin kısa süre durdurulması | `01-NET-Calisma-Modeli.md` |
| Streaming operatör | Veriyi tek tek geçiren operatör (`Where`, `Select`) | `03-LINQ.md` |
| String interning | Aynı string literal'lerinin tek nesnede toplanması | `01-NET-Calisma-Modeli.md` |
| Switch expression | Değer döndüren switch | `05-Modern-CSharp-Ozellikleri.md` |
| Sync-over-async | Asenkron işi senkron beklemek (`.Result`, `.Wait()`) | `04-Asenkron-Programlama.md` |
| SynchronizationContext | "Devam bloğu şu thread'de çalışsın" kuralını koyan yapı | `04-Asenkron-Programlama.md` |
| Target-typed new | Sol taraftan tip çıkarımıyla `new()` yazımı | `05-Modern-CSharp-Ozellikleri.md` |
| Task | Gelecekte tamamlanacak işin temsili | `04-Asenkron-Programlama.md` |
| Task.WhenAll / WhenAny | Hepsini bekle / ilk bitenle devam et | `04-Asenkron-Programlama.md` |
| Thread | İşletim sisteminin zamanladığı yürütme birimi | `04-Asenkron-Programlama.md` |
| Thread pool | Yeniden kullanılan thread havuzu | `04-Asenkron-Programlama.md` |
| Thread pool starvation | Havuzdaki thread'lerin tükenmesi, uygulamanın yanıtsız kalması | `04-Asenkron-Programlama.md` |
| Thread-safe koleksiyon | Eşzamanlı erişimde tutarlılığı garanti eden koleksiyon | `02-Koleksiyonlar.md` |
| Tiered compilation | Sık çağrılan metodun arka planda yeniden, daha iyi derlenmesi | `01-NET-Calisma-Modeli.md` |
| ToLookup | Anahtar başına çoklu değer tutan, anında çalışan gruplama | `03-LINQ.md` |
| Top-level statements | `Main` metodu yazmadan program gövdesi | `05-Modern-CSharp-Ozellikleri.md` |
| Try-desen (`TryParse`) | Hata yerine `bool` döndüren, exception maliyetinden kaçınan kalıp | `06-Dil-Altyapisi.md` |
| Type inference | Tipin derleyici tarafından çıkarılması (`var`) | `05-Modern-CSharp-Ozellikleri.md` |
| Type parameter | `<T>` içindeki yer tutucu tip | `06-Dil-Altyapisi.md` |
| Unobserved exception | `await` edilmediği için fark edilmeyen hata | `04-Asenkron-Programlama.md` |
| using declaration | Kapsam sonunda otomatik `Dispose` çağıran bildirim | `06-Dil-Altyapisi.md` |
| Value type | Atamada içeriği kopyalanan tip (`struct`, `enum`, `int`...) | `01-NET-Calisma-Modeli.md` |
| ValueTask | Senkron tamamlanan işler için hafif Task alternatifi | `04-Asenkron-Programlama.md` |
| with expression | Kopya üretip belirli alanları değiştirme (yüzeysel kopya) | `05-Modern-CSharp-Ozellikleri.md` |
| `with` ifadesi | Bir record'un tek alanı değişmiş kopyasını üretme söz dizimi | `01-NET-Calisma-Modeli.md` |
| yield return | Elemanları istendikçe tek tek üreten mekanizma | `02-Koleksiyonlar.md` |
| Özellik deseni | Nesnenin alanlarına göre eşleme (`{ Tutar: > 100 }`) | `05-Modern-CSharp-Ozellikleri.md` |

---

Haftanın özet ve tuzak listesi için: [`00-Hizli-Tekrar.md`](00-Hizli-Tekrar.md)

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
