# Hafta 1 · Pazar — Haftalık Tekrar Özeti

**Okuma süresi:** ~25 dk
**Nasıl kullanılır:** Haftanın altı notunu tek tek okumak yerine bu sayfayı oku. Bir madde sana yabancı geldiyse ilgili nota dön — sadece o bölüme. Bu sayfa hafta sonlarında ve bootcamp başlamadan önce tekrar tekrar okunmak için var.

---

## Haftanın Tek Cümlesi

> C#'ta karşılaşacağın davranışların çoğu üç şeyden çıkar: **verinin kopyalanıp mı paylaşıldığı**, **işin ne zaman çalıştığı**, ve **thread'in beklerken serbest olup olmadığı.**

Bu hafta yeni bir dil öğrenmedin. Zaten yazdığın dilin **neden öyle davrandığını** öğrendin. Aradaki fark şudur: kod çalışınca sevinmekle, kod çalışmayınca nereye bakacağını bilmek arasındaki fark.

Altı notun tamamı aslında üç soruya çıkıyor. Birincisi: bu veri elden ele geçerken **kopyalanıyor mu, yoksa aynı şey mi paylaşılıyor?** Value ve reference type ayrımı, `string`'in neden tuhaf davrandığı, bir metoda liste gönderince neden dışarısının etkilendiği — hepsi bu tek sorunun cevabı. İkincisi: **bu iş ne zaman çalışıyor?** LINQ sorgusunun yazıldığında değil gezildiğinde çalışması, `yield return`'ün tembelliği, JIT'in metodu ilk çağrıldığında derlemesi, GC'nin ne zaman devreye girdiği — hepsi zamanlama sorusu. Üçüncüsü: **beklerken kim meşgul?** `async`/`await`'in bütün varlık sebebi bu; `.Result`'ın neden yasak olduğu da.

Geri kalan her şey bu üç eksenin etrafına takılıyor. Koleksiyon seçimi, "hangi işlemi çok yapacağım" sorusunun cevabı. Generic, delegate, extension method: dilin altyapısı, yani LINQ'in ve framework'lerin üstüne kurulduğu zemin. Modern C# özellikleri ise daha az yazıp daha çok anlatmanın yolları — yeni kavram değil, aynı işin kısa hâli.

Bir şeye dikkat et: bu haftanın konularının hiçbiri "Hafta 1 konusu" değil. EF Core'un beklenmedik bir `UPDATE` atması, bir API'nin yük altında tıkanması, bir sorgunun on binlerce satır çekmesi — bunların hepsi bootcamp boyunca karşına çıkacak ve hepsinin açıklaması bu haftadaki notlarda. Sondaki bağlantı haritası tam olarak bunu gösteriyor.

> **Ana benzetme:** Bu hafta araba sürmeyi değil, **kaputun altını** öğrendin. Sürmeyi zaten biliyordun; direksiyonu kırınca araba dönüyordu. Şimdi motorun nasıl çalıştığını, yakıtın nereden gittiğini ve o kırmızı lambanın neden yandığını biliyorsun. Yolda kaldığında fark burada ortaya çıkar: sürücü bekler, kaputu bilen bakar.

---

## 1. Çalışma Modeli ve Bellek  →  `01-NET-Calisma-Modeli.md`

> **Benzetme —** Bir marangoz atölyesi düşün. Tezgâhın üstü küçüktür, elinin altındadır, iş bitince süpürülür — orada sadece o an kullandığın şeyler durur. Arkadaki depo ise geniştir, istediğin kadar malzeme koyarsın, ama dağınıklaşır ve arada biri girip kullanılmayanları toplaması gerekir. Tezgâh **stack**'tir, depo **heap**'tir, toplayan kişi **GC**'dir.

**Basitçe:** Programın çalışırken veriyi iki yere koyar. Biri küçük, hızlı ve otomatik temizlenen yer; diğeri büyük, esnek ama temizlenmesi için birinin uğraşması gereken yer. Bir veriyi başkasına verdiğinde ya fotokopisini verirsin (o değiştirse sen etkilenmezsin) ya da evin anahtarını verirsin (o içeriyi değiştirirse sen de görürsün). C#'ın en çok şaşırtan davranışlarının çoğu, elindekinin fotokopi mi anahtar mı olduğunu karıştırmaktan çıkar.

> Bu benzetme şurada bozulur: anahtarı verdiğin kişi *başka bir eve* taşınırsa, senin evin değişmez. Metoda gönderdiğin nesnenin içini değiştirmekle, o değişkene yepyeni bir nesne atamak farklı şeylerdir — ikincisi dışarıyı etkilemez.

**Teknik olarak:**

- .NET **platform**, C# **dil**, CLR **runtime**, BCL **hazır kütüphane**. Bugünkü ".NET", Framework + Core birleşimi.
- Derleme iki aşamalı: kaynak → **IL** (derleme anı) → **makine kodu** (JIT, metot ilk çağrıldığında).
- **Assembly** = IL + metadata. Reflection, DI, EF Core ve IntelliSense bu metadata'yı okur.
- **Stack** hızlı, otomatik, thread'e özel. **Heap** esnek, GC'ye bağımlı. 85 KB üstü nesneler **LOH**'a gider ve sıkıştırılmaz.
- **Value type kopyalanır, reference type paylaşılır.** Nesnenin *içini* değiştirirsen dışarısı görür; değişkene *yeni nesne atarsan* görmez.
- **`string` immutable'dır.** Döngüde birleştirme → `StringBuilder`.
- **Boxing** value type'ı heap'e taşır; generic bunu ortadan kaldırır.
- **GC nesil temellidir**: Gen 0 ucuz ve sık, Gen 2 pahalı ve nadir. Sızıntı kaynakları: statik koleksiyonlar, bırakılmayan event abonelikleri, kapatılmayan kaynaklar.
- **`IDisposable` + `using`** = GC'nin göremediği kaynaklar için deterministik temizlik.

---

## 2. Koleksiyonlar  →  `02-Koleksiyonlar.md`

> **Benzetme —** Elinde iki defter var. Biri sıradan bir alışveriş listesi: baştan sona okursun, aradığını bulana kadar satırları tek tek geçersin. Diğeri bir telefon rehberi: harfe göre ayrılmış, aradığın ismi doğrudan açtığın sayfada bulursun. İki defter de aynı bilgiyi tutabilir. Fark, **aramanın kaç saniye sürdüğündedir.** Liste ile sözlük arasındaki fark tam olarak budur.

**Basitçe:** Koleksiyon seçmek bir zevk meselesi değil, bir hız kararıdır. Önce kendine "bu veriyle en çok ne yapacağım" diye sor: sırayla gezecek misin, sırasına göre mi ulaşacaksın, yoksa sürekli "bu içinde var mı" diye mi arayacaksın. Cevabın hangisiyse ona uygun kabı seç. En pahalı hata, listenin içinde döngüyle arama yapmaktır: eleman sayısı arttıkça süre katlanarak büyür.

**Teknik olarak:**

- Koleksiyon seçimi bir **performans kararıdır**. Soru: hangi işlemi çok yapacağım?
- `List<T>` içeride dizidir → indeks O(1), arama ve ortaya ekleme O(n).
- `Dictionary` / `HashSet` hash tabanlıdır → O(1). Anahtar tipinde `GetHashCode` ve `Equals` tutarlı olmalı, hash'i etkileyen alan sonradan değişmemeli.
- **Liste içinde arama yapan döngü O(n²) tuzağıdır** → `HashSet`'e çevir.
- Arayüz kuralı: **parametrede en az yetenekli** (`IEnumerable<T>`), **dönüşte en kullanışlı** (`IReadOnlyList<T>`).
- **`IEnumerable` bellekte, `IQueryable` veritabanında** çalışır. `AsEnumerable()` / `ToList()` bu sınırı geçer.
- `yield return` tembel üretimdir; koleksiyon her gezilişte yeniden üretilir.

---

## 3. LINQ  →  `03-LINQ.md`

> **Benzetme —** Lokantada masaya oturup sipariş fişini doldurmak yemeği pişirmez. Fiş elinde durduğu sürece mutfak soğuktur. Ocak, sen fişi garsona uzattığın anda yanar. Dahası: aynı fişi üç kez mutfağa gönderirsen üç porsiyon pişer, kimse "bunu az önce yapmıştık" demez.

**Basitçe:** LINQ sorgusu yazmak, yapılacak işin tarifini yazmaktır — işin kendisi değil. İş, sen sonucu gerçekten istediğin anda yapılır: `foreach` ile gezdiğinde ya da `ToList()` dediğinde. Ve her istediğinde baştan yapılır. Bir de şu var: aynı sorgu bellekteki bir listeye yazıldığında masumdur, veritabanına yazıldığında bir SQL cümlesine çevrilir. Çevrilemeyen bir şey yazarsan ya hata alırsın ya da fark etmeden bütün tabloyu belleğe çekersin.

> Bu benzetme şurada bozulur: lokantada fişi verdiğinde yemek bir kez gelir ve tabakta durur. LINQ'te ise sonucu "tabağa koymak" ayrı bir iştir — `ToList()` dediğinde koyarsın. Koymazsan her bakışında yemek yeniden pişer.

**Teknik olarak:**

- Sorgu bir **tariftir**: ertelenmiş çalışır ve **her gezilişte yeniden** çalışır. Çok kez kullanacaksan bir kez maddeleştir.
- `Select` = projeksiyon. EF Core'da sadece gereken kolonları çekmenin yolu.
- `GroupBy` → `IEnumerable<IGrouping<K,T>>`: anahtarı olan liste.
- `SelectMany` iç içe koleksiyonları düzleştirir.
- `First` / `Single` / `OrDefault` seçimi bir **niyet beyanıdır**. Id ile arama → `SingleOrDefault`.
- Varlık kontrolünde `Count() > 0` değil **`Any()`**.
- Sayfalamada `OrderBy` olmadan `Skip/Take` güvenilmezdir.
- `ToList()` sınırdır → **en sona** koy.
- `IQueryable` üzerinde her C# metodu çalışmaz; çevrilemeyen ifade ya hata verir ya da tüm tabloyu belleğe çeker.
- **N+1 problemi:** 1 sorgu + her satır için ek sorgu. Sebebi ertelenmiş yükleme + döngü.

---

## 4. Asenkron Programlama  →  `04-Asenkron-Programlama.md`

> **Benzetme —** Lokantada bir garson düşün. Siparişi alır, mutfağa verir ve **mutfağın önünde beklemez** — gider başka masaların siparişini alır. Yemek hazır olunca haber gelir, o da götürür. Tek garsonla on masaya bakılmasının sebebi budur. Kötü senaryo: garson mutfağın önünde durup yemeğin pişmesini seyrederse, dokuz masa boş bekler. `.Result` yazmak tam olarak budur.

**Basitçe:** `async`/`await` kodunu hızlandırmaz. Beklerken çalışanı serbest bırakır. Veritabanı cevabı, dosya okuması, başka bir servise gidilen istek — bunlar beklenirken thread hiçbir iş yapmıyordur. `await` o thread'i havuza geri verir, başkası kullanır. Kazanç tek kullanıcıda görünmez; yüz kullanıcı aynı anda geldiğinde görünür.

> Bu benzetme şurada bozulur: garson tek kişidir, `await` sonrasında işe devam eden ise havuzdaki **başka** bir thread olabilir. Kodun akışı kaldığı yerden devam eder ama aynı thread'in devam ettiğini varsayma.

**Teknik olarak:**

- Eşzamanlılık ≠ paralellik ≠ asenkronluk. `async` **asenkronluk** içindir.
- **I/O-bound → `async`. CPU-bound → paralellik.**
- `await` thread'i havuza geri verir. Kazanç **hız değil ölçektir**.
- `Task` bir iş temsilidir, **thread değildir**.
- **`.Result` / `.Wait()` yasak** → deadlock ve thread pool açlığı.
- **`async void` yazma** (UI olay yöneticileri hariç): beklenemez, hatası yakalanamaz.
- Döngü içinde `await` işleri sıraya sokar. Bağımsızsa `Task.WhenAll` — ama paylaşılan `DbContext` ile değil.
- `CancellationToken` alınmalı **ve aşağı geçirilmeli**.
- `await` edilmeyen `Task`'taki exception sessizce kaybolur.

---

## 5. Modern C#  →  `05-Modern-CSharp-Ozellikleri.md`

> **Benzetme —** Eskiden her dilekçeyi baştan sona elle yazardın: hitap, adres, tarih, imza. Sonra matbu formlar çıktı — sabit kısımlar zaten basılı, sen sadece doldurman gereken yeri dolduruyorsun. Dilekçenin anlamı değişmedi, yazma zahmeti azaldı. Modern C# özelliklerinin çoğu bu matbu formlardır.

**Basitçe:** Bu bölümdeki özelliklerin hiçbiri yeni bir kavram getirmiyor. Hepsi, zaten yazdığın şeyi daha kısa ve daha az hata yapılır biçimde yazmanın yolu. `record` eşitliği elle yazmaktan kurtarır, `init` nesneyi kurulduktan sonra kilitler, pattern matching iç içe `if` yığınını tek ifadeye indirir. Bir tanesi diğerlerinden farklı: nullable reference types, bir yazım kolaylığı değil, derleyiciye "burası boş olabilir" diye anlattığın bir uyarı sistemidir.

**Teknik olarak:**

- `var` statik tiptir, `dynamic` değildir.
- **NRT** null hatalarını derleme anına taşır ama garanti vermez. `!` operatörünü uyarı susturmak için serpiştirme.
- **`record`** = değer eşitliği + `with`. DTO'lar için doğal, entity'ler için değil.
- **`init` + `required`** = constructor yazmadan eksiksiz ve değişmez nesne.
- **Pattern matching** iç içe `if` yığınlarını tek ifadeye indirir; **switch expression** değer döndürür.
- **Raw string** (`"""`) SQL/JSON yazarken kaçış cehennemini bitirir.
- Modern şablonlarda `Main`, `using` satırları ve namespace parantezleri **örtük** hâle geldi, kaybolmadı.
- `?.` çalışma anı kontrolüdür; `!` sadece derleyiciye verilen sözdür.

---

## 6. Dil Altyapısı  →  `06-Dil-Altyapisi.md`

> **Benzetme —** Bir evde musluğu açarsın, su gelir. Duvarın arkasında borular, vanalar ve bir kolon vardır ama sen onları görmezsin. Tesisatçı ise tam olarak orayı bilir; tıkanıklığı musluğa bakarak değil, borunun nereden geçtiğini bilerek çözer. Generic, delegate, closure ve extension method, C#'ın duvar arkasındaki tesisatıdır. LINQ dahil kullandığın hazır şeylerin çoğu bu boruların üstünde durur.

**Basitçe:** Bu bölüm, framework'lerin nasıl mümkün olduğunu anlatıyor. Generic sayesinde aynı kod her tiple çalışır. Delegate sayesinde bir metodu değişken gibi taşıyıp başka bir metoda parametre verebilirsin — lambda yazdığın her yerde olan budur. Extension method sayesinde var olan bir tipe dokunmadan ona metot eklenmiş gibi yazarsın; LINQ'in tamamı bu numaradan ibarettir. Exception yönetimi de buraya girer, çünkü hatanın nereden geldiğini kaybetmemek bu haftanın en ucuz kazancıdır.

> Bu benzetme şurada bozulur: extension method tipe gerçekten bir şey **eklemez**. Su borusu gibi kalıcı bir bağlantı kurulmaz; derleyici senin çağrını arka planda statik bir metot çağrısına çevirir, o kadar.

**Teknik olarak:**

- **Generic** = tip güvenliği + boxing'siz performans. **Kısıt**, `T` hakkında derleyiciye verdiğin bilgidir.
- **Delegate** metoda işaret eden tiptir. `Action` döndürmez, `Func` döndürür, `Predicate` `bool` döner.
- **Closure** dış değişkeni yakalar ve ömrünü uzatır — `for` döngüsü değişkeni klasik tuzaktır.
- **Extension method** tipe bir şey eklemez; çağrıyı statik metoda çevirir. **LINQ'in tamamı budur.**
- **`throw ex` stack trace'i siler, `throw` korur.**
- Boş `catch` bloğu ve akış kontrolü amaçlı exception: en pahalı iki alışkanlık.
- `Equals` override ediyorsan `GetHashCode`'u da et.
- **Interface** "-ebilir" ilişkisidir; bağımlılığı tersine çevirmenin aracıdır → Hafta 2'nin kapısı.

---

## Bağlantı Haritası — Bu Hafta Nereye Bağlanıyor

> **Benzetme —** Metro haritasındaki aktarma istasyonları gibi. Bu haftanın konuları, ileride bineceğin bütün hatların geçtiği duraklardır. Şimdi "burada niye duruyoruz" diye düşünürsün; Hafta 3'te aktarma yaparken durağı tanıdığına sevinirsin.

**Basitçe:** Aşağıdaki tablo, bu haftanın hangi konusunun ilerideki hangi haftada karşına çıkacağını gösteriyor. Amacı şu: bugün soyut gelen bir madde, ileride somut bir hatanın açıklaması olacak. Bir konuyu "bu ne işime yarayacak" diye geçiştirmeden önce sağ sütuna bak.

| Bu haftaki konu | Nerede karşına çıkacak |
|---|---|
| Value vs reference type | Hafta 3 — EF Core change tracking'in beklenmedik `UPDATE`'leri |
| `IEnumerable` / `IQueryable` | Hafta 3 — EF Core performansı, N+1 problemi |
| LINQ projeksiyon (`Select`) | Hafta 3 ve 5 — DTO'ya dönüştürme, gereksiz kolon çekmeme |
| `async`/`await` | Hafta 4 ve 5 — her controller metodu, her repository çağrısı |
| `IDisposable` | Hafta 4 — DI servis yaşam döngüleri, `DbContext` yönetimi |
| Interface ve DIP | Hafta 2 — SOLID'in D maddesi; Hafta 4 — DI konteyneri |
| Generic + kısıt | Hafta 5 — `IRepository<T>`, generic servis katmanı |
| `record`, `init`, `required` | Hafta 5 — DTO tasarımı, API sözleşmeleri |
| Delegate / lambda | Bootcamp boyunca — LINQ, middleware, event, konfigürasyon |
| Exception yönetimi | Hafta 4 — global exception middleware |

---

## Kendini Yoklama — Cevaplayabiliyor musun

Not: Bu bir ödev değil, **tekrar filtresidir**. Bir soruda takılırsan ilgili nota dön.

1. Bir metoda `List<T>` gönderip içine eleman eklersen dışarıdaki liste etkilenir mi? Ya listeye yeni bir nesne atarsan?
2. `string` reference type olduğu hâlde neden değer gibi davranır?
3. Gen 2 toplaması neden Gen 0'dan pahalıdır?
4. `Dictionary` anahtarı olarak kullandığın nesnenin bir alanını sonradan değiştirirsen ne olur?
5. `context.Urunler.AsEnumerable().Where(...)` ile `context.Urunler.Where(...)` arasında veritabanı açısından ne fark var?
6. Aynı LINQ sorgusunu üç kez `foreach`'lersen kaç kez çalışır?
7. `Count() > 0` yerine neden `Any()`?
8. `await` sırasında thread ne yapıyor?
9. `.Result` neden tehlikeli, ASP.NET Core'da bile?
10. `throw` ile `throw ex` arasındaki fark neyi kaybettiriyor?
11. LINQ'in `Where` metodu teknik olarak nedir, hangi mekanizmayla `IEnumerable`'a bağlanıyor?
12. `record` ile `class` arasında eşitlik açısından ne fark var, hangisini nerede kullanırsın?

---

## Sonraki Hafta

**Hafta 2 — OOP olgunluğu, SOLID, Design Patterns.**
Bu haftanın son maddesi (interface ve bağımlılık) doğrudan oraya bağlanıyor. Hafta 2 biraz daha soyut geçecek; kod yazmaktan çok "bu kodu neden böyle bölüyoruz" sorusunun cevabı olacak. Bootcamp'in 10, 11 ve 12 numaralı mimari projelerinin tamamı o haftaya dayanıyor.
