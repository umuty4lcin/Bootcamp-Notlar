# Hafta 1 · Perşembe — Asenkron Programlama

**Okuma süresi:** ~55 dk
**Neden bu konu:** ASP.NET Core'da yazacağın hemen her controller metodu `async` olacak. Yanlış kullanımı, tek kullanıcıda görünmeyip yük altında uygulamayı kilitleyen türden hatalar üretir — ve bunlar hata ayıklaması en zor problemlerdir.

---

## Önce Basitçe

Bir programın yaptığı işlerin çoğu aslında **beklemektir**. Veritabanına soru sorar, cevabı bekler. Bir siteye istek atar, cevabı bekler. Dosyayı diskten okur, bekler. Bu beklemeler insan ölçeğinde kısa — yüz milisaniye, iki yüz milisaniye — ama bilgisayar ölçeğinde çok uzundur. O süre boyunca işlemci boş durur.

Asıl mesele şu: eskiden bu beklemeyi yapan kişi, işi yapan kişiyle aynıydı. Program veritabanına soruyu sorar ve **aynı eleman** cevap gelene kadar orada dikilirdi. Bir eleman dikiliyorsa başka iş yapamaz. Elli kişi aynı anda siteye girdiğinde, elli elemanın hepsi birer köşede bekliyor olabilir ve yeni gelen kimseye bakılamaz. Site "çok yavaş" değildir aslında; site **boş bekleyerek** tıkanmıştır.

Asenkron programlama bu sorunun çözümüdür. "Soruyu sordum, cevap gelince beni çağırın, ben o arada başka işlere bakıyorum" demenin kod karşılığıdır. Eleman beklemeye geçmez, havuza döner, başka isteklere bakar. Cevap geldiğinde — belki aynı eleman, belki başkası — iş kaldığı yerden devam eder.

Burada çok önemli ve sık yanlış anlaşılan bir nokta var: asenkronluk **hız kazandırmaz**. Veritabanı sorgusu iki yüz milisaniye sürüyorsa, asenkron yazdın diye yüz milisaniye sürmez. Kazandığın şey, o iki yüz milisaniyede sunucunun başka insanlara hizmet edebilmesidir. Yani tek kişinin deneyimi aynı kalır, aynı sunucunun taşıyabileceği kişi sayısı katlanır. Kapasite meselesidir, hız meselesi değil.

C# bu işi `async` ve `await` diye iki kelimeyle yapar ve bu iki kelimenin en güzel tarafı, kodun hâlâ yukarıdan aşağı okunmasıdır. Beklemeli kod ile beklemesiz kod neredeyse aynı görünür. Bu kolaylığın bedeli de var: kod basit göründüğü için insanlar arkada ne olduğunu öğrenmeden kullanıyor ve birkaç klasik hataya düşüyor. Bu notun en uzun bölümü o hatalara ayrıldı — çünkü asenkron programlamada bilmen gereken şey sözdizimi değil, **nerede batacağındır**.

Son olarak şunu baştan bil: asenkron kodun en kötü hataları sessizdir. Kendi bilgisayarında test edersin, çalışır. Beş kişiyle test edersin, çalışır. Beş yüz kişi girdiğinde uygulama yanıt vermemeye başlar, işlemci de boştur, hiçbir hata da yoktur. Bu notta o sessiz hataların hepsini tek tek göreceksin. Şimdi detaya iniyoruz.

> **Ana benzetme:** Asenkron programlama, iyi bir **lokanta garsonu** gibi çalışmaktır. Garson siparişi alır, mutfağa verir ve mutfağın önünde tabağın pişmesini beklemez — diğer masalara bakar, hesap alır, su tazeler. Mutfak "hazır" diye seslendiğinde gelip tabağı götürür. Senkron garson ise siparişi verip mutfağın önünde durur; lokantada yirmi masa boş beklerken o tek tabağa bakmaktadır. Aynı mutfak, aynı yemek süresi — fark sadece garsonun bekleyip beklememesinde.

---

## Bu Notta Ne Var

1. Eşzamanlılık, paralellik ve asenkronluk — üç farklı şey
2. Thread, thread pool ve web sunucusu modeli
3. `Task` nedir
4. `async` / `await` gerçekte ne yapar
5. Asenkronluğun kazandırdığı şey (ve kazandırmadığı)
6. Klasik tuzaklar: deadlock, `async void`, `.Result`
7. `CancellationToken`
8. Birden çok işi birlikte çalıştırma
9. `ValueTask`, `ConfigureAwait`, exception davranışı
10. Adlandırma ve tasarım kuralları

---

## 1. Üç Kavramı Ayır

> **Benzetme —** Evde üç iş var: çamaşır, bulaşık, fırında börek. Tek başınasın ama üçünü de aynı öğlen bitiriyorsun — çamaşırı makineye atıp bulaşığa geçiyor, börek pişerken masayı kuruyorsun. Bu **eşzamanlılık**: işler iç içe ilerliyor, sen hep tek kişisin. Eşin de gelip bulaşığı üstlenirse artık iki kişi **aynı anda** iş yapıyor; bu **paralellik**. Makineyi çalıştırıp başına dikilmemek, "bitince zil çalar" deyip başka işe geçmek ise **asenkronluk**. Üçü farklı şeyler ve üçü de aynı mutfakta olabiliyor.

**Basitçe:** Bu üç kelime günlük konuşmada karışır ama teknik olarak ayrı şeylerdir. Eşzamanlılık, birden çok işi iç içe yürütmektir — tek çekirdekte bile olur. Paralellik, gerçekten aynı anda çalışmaktır ve birden çok çekirdek gerektirir. Asenkronluk ise bir işi başlatıp sonucunu beklemeden devam edebilmektir. `async`/`await` üçüncüsü içindir.

**Teknik olarak:**

| Kavram | Tanım | Örnek |
|---|---|---|
| **Eşzamanlılık (concurrency)** | Birden çok işin **iç içe ilerlemesi**. Aynı anda çalışmaları gerekmez | Bir aşçının fırındaki keki beklerken salata hazırlaması |
| **Paralellik (parallelism)** | Birden çok işin **fiziksel olarak aynı anda** çalışması. Çok çekirdek gerekir | İki aşçının iki ayrı yemeği aynı anda pişirmesi |
| **Asenkronluk (asynchrony)** | Bir işin başlatılıp, **sonucu beklenmeden** kontrolün geri verilmesi | Fırını çalıştırıp önüne oturmamak |

`async`/`await` **asenkronluk** içindir, paralellik için değil. Paralellik için `Parallel.For`, PLINQ veya birden çok `Task` vardır.

### İki farklı iş türü

| Tür | Ne yapar | Doğru yaklaşım |
|---|---|---|
| **I/O-bound** | Bekler: veritabanı, HTTP, dosya, ağ | **`async`/`await`** — bekleyen thread serbest kalır |
| **CPU-bound** | Hesaplar: görüntü işleme, şifreleme, büyük döngü | Paralellik (`Task.Run`, `Parallel`) — thread meşgul olmak *zorunda* |

Bu ayrım kritiktir. CPU-bound işi `async` yapmak hiçbir şey kazandırmaz; sadece kodu karmaşıklaştırır.

Ayrımı ev işleri üzerinden düşün: çamaşır makinesi **I/O-bound**tur — sen çalıştırırsın, makine döner, senin orada durmana gerek yoktur. Hamur yoğurmak ise **CPU-bound**tur — biri onu fiilen yapmak zorundadır, "başlat ve git" diye bir şey yoktur. Hamuru daha hızlı yoğurmanın tek yolu ikinci bir çift el bulmaktır; işte paralellik budur.

```csharp
// I/O-bound: bekleme var, thread serbest bırakılmalı
public async Task<string> SayfayiGetirAsync(string url)
    => await _httpClient.GetStringAsync(url);

// CPU-bound: hesaplama var, birinin fiilen yapması gerekiyor
public long Topla(int n)
{
    long t = 0;
    for (int i = 0; i < n; i++) t += i;
    return t;
}
```

> **Bu benzetme şurada bozulur:** Ev işleri örneğinde sen hep aynı kişisin. Asenkron .NET'te ise `await` sonrası işe **başka bir thread** devam edebilir. Çamaşırı sen atarsın, zili başkası duyar ve makineyi o boşaltır. İşin sırası korunur ama işi yapan kişi değişebilir. Bu ayrıntı, 9. bölümdeki `ConfigureAwait` konusunun temelidir.

---

## 2. Thread ve Thread Pool

> **Benzetme —** Bankada altı vezne var, altı memur. Senkron çalışan bir memur şöyle yapar: müşterinin evrakını alır, arka ofise "bu doğru mu" diye sorar ve cevap gelene kadar **gişeyi kapatıp bekler**. O gişe artık kuyruğa hizmet etmiyordur. Altı müşterinin altısı da arka ofis cevabı bekliyorsa, banka çalışıyor görünür ama hiçbir iş ilerlemez; kuyruk kapıdan taşar. Asenkron memur ise evrakı arka ofise yollar, müşteriye "cevap gelince çağıracağım" der ve sıradaki müşteriye bakar. Aynı altı memur, kat kat fazla kişiye hizmet eder.

**Basitçe:** Programın işini yapan "elemanlar" thread'lerdir ve sayıları sınırlıdır. Her web isteği bu havuzdan bir eleman alır. O eleman beklemeye geçerse havuzdan düşmüş olur. Asenkron kodun yaptığı tek şey, bekleme süresince elemanı havuza iade etmektir. Bütün mesele budur.

**Teknik olarak:** **Thread (iş parçacığı)** — İşletim sisteminin zamanladığı en küçük yürütme birimi. Kendi stack'i vardır (~1 MB) ve oluşturulması pahalıdır.

**Thread pool** — .NET'in yönettiği, yeniden kullanılabilir thread havuzu. Her iş için yeni thread açmak yerine havuzdan alınır, iş bitince geri verilir.

Havuzun boyutu sabit değildir; .NET yük arttıkça yeni thread ekler ama bunu **yavaş** yapar (saniyede bir-iki thread). Yani ani bir yük altında havuz hemen büyümez. Bu yavaşlık bilinçli bir tercihtir: thread eklemek pahalıdır ve gerçek çözüm nadiren "daha çok thread"dir.

### Web sunucusu modeli — asenkronluğun asıl sebebi

ASP.NET Core'da gelen her HTTP isteği thread pool'dan bir thread alır.

**Senkron kodda:** İstek veritabanı sorgusu yaparsa thread, sonuç gelene kadar **bloke** olur. 200 ms boyunca hiçbir iş yapmaz ama havuzdan çıkmıştır. Havuz tükenince yeni istekler kuyruğa girer.

**Asenkron kodda:** `await` noktasında thread **havuza geri döner** ve başka isteklere hizmet eder. Veritabanı cevabı geldiğinde havuzdan (muhtemelen başka) bir thread alınıp kalınan yerden devam edilir.

Sayıyla bakalım. Diyelim havuzda 100 thread var ve her istek 200 ms veritabanı beklemesi içeriyor:

- **Senkron:** Her thread 200 ms boyunca kilitli. Saniyede taşınabilecek istek ≈ 100 / 0.2 = 500. Fazlası kuyrukta bekler.
- **Asenkron:** Thread'ler bekleme boyunca serbest. Aynı 100 thread ile binlerce eşzamanlı isteği taşıyabilirsin; sınır artık thread sayısı değil, veritabanının kapasitesidir.

> Asenkronluk tek bir isteği **hızlandırmaz**. Aynı donanımla **çok daha fazla eşzamanlı isteğe** hizmet etmeni sağlar. Ölçek meselesidir, hız meselesi değil.

**Thread pool starvation (havuz açlığı)** — Havuzdaki tüm thread'lerin bloke olması. Uygulama yanıt vermemeye başlar; CPU kullanımı düşüktür ama istekler ilerlemez. Sebebi neredeyse her zaman async kodun senkron bloke edilmesidir.

Bu tablonun nasıl göründüğünü tanı, çünkü ilk bakışta yanıltıcıdır:

| Belirti | Ne düşünürsün | Gerçek |
|---|---|---|
| Uygulama yanıt vermiyor | "Sunucu yetersiz, CPU yetmiyor" | CPU %5, thread'lerin hepsi boş bekliyor |
| Yanıt süreleri kademeli uzuyor | "Veritabanı yavaşladı" | Kuyruk birikiyor, veritabanı normal |
| Sunucu büyütünce düzelmiyor | "Daha da büyütelim" | Sorun donanımda değil, bloke eden kodda |

> **Bu benzetme şurada bozulur:** Banka örneğinde memur beklerken "boş oturuyor" gibi görünür ama en azından yerinde durur. Thread'de durum daha kötüdür: bloke olan thread yaklaşık 1 MB bellek tutmaya devam eder ve işletim sistemi onu zamanlama listesinde tutar. Yani boş bekleme bedavaya bile değildir. Ayrıca bankada müdür yeni memur çağırabilir; .NET de havuza thread ekler ama saniyede birkaç tane — yük anlık geldiğinde bu yetişmez.

---

## 3. `Task` Nedir

> **Benzetme —** Kuru temizlemeciye montu bırakırsın, sana bir **fiş** verirler. O fiş montun kendisi değildir; "böyle bir iş var, hazır olunca bununla alırsın" belgesidir. Fişi cebine koyup işine gidersin. Önemli olan şu: fiş, arkada bir çalışanın sürekli senin montunla uğraştığı anlamına gelmez. Mont makinede dönerken kimse başında durmaz. Fiş sadece bir kayıttır.

**Basitçe:** `Task`, "henüz bitmemiş bir iş"in elindeki temsilidir. Sonuç hazır olduğunda oradan alırsın. En sık yapılan hata, her `Task`'ın bir eleman (thread) meşgul ettiğini sanmaktır — doğru değildir. Bekleyen bir ağ isteğinde hiç kimse çalışmıyor olabilir; sadece bir kayıt vardır ve işletim sistemi cevap gelince haber verecektir.

**Teknik olarak:** **`Task`** — Gelecekte tamamlanacak bir işin **temsili**. Diğer dillerdeki "promise/future" kavramının karşılığı.

| Tip | Anlamı |
|---|---|
| `Task` | Sonuç döndürmeyen asenkron iş (senkron `void` karşılığı) |
| `Task<T>` | `T` tipinde sonuç döndürecek asenkron iş |
| `ValueTask<T>` | Genelde senkron tamamlanan işler için ayırma yapmayan hafif alternatif |

Bir `Task`'ın durumları: `WaitingToRun` → `Running` → `RanToCompletion` / `Faulted` / `Canceled`.

**Önemli:** `Task` bir thread **değildir**. Bir I/O `Task`'ı beklerken hiçbir thread çalışmıyor olabilir — işletim sistemi tamamlanma bildirimi gönderene kadar sadece bir kayıt vardır. "Her Task bir thread tüketir" yanılgısı yaygındır ve yanlıştır.

Bu yanılgı neden bu kadar yaygın? Çünkü `Task.Run` gerçekten bir thread kullanır. `Task` tipinin iki farklı kullanımı vardır ve ikisi karıştırılır:

```csharp
// 1) I/O Task'ı: thread kullanmaz, işletim sistemi bildirim gönderir
Task<string> t1 = _httpClient.GetStringAsync(url);

// 2) Task.Run: thread pool'dan gerçekten bir thread alır ve orada kod çalıştırır
Task<long> t2 = Task.Run(() => AgirHesap());
```

Birincisine "promise task", ikincisine "delegate task" denir. İkisi de `Task` tipindedir ama maliyetleri taban tabana zıttır.

Hazır bir sonucu `Task` olarak döndürmek gerektiğinde nesne ayırmadan yapabilirsin:

```csharp
public Task<int> HesaplaAsync(bool onbellekteVar)
{
    if (onbellekteVar)
        return Task.FromResult(42);          // yeni iş başlatmaz, hazır Task döner

    return YavasHesaplaAsync();
}
```

> **Bu benzetme şurada bozulur:** Kuru temizleme fişi pasiftir — sen gidip sormazsan hiçbir şey olmaz. `Task` ise kendisi haber verebilir: iş bitince ona bağlanmış devam bloğu (continuation) otomatik çalışır. Ayrıca fişi kaybedersen mont yine de temizlenir; `Task`'ı `await` etmeden bırakırsan iş çalışır ama içinde bir hata olursa **kimse görmez**. 9. bölümdeki "unobserved exception" konusu tam olarak budur.

---

## 4. `async` ve `await` Gerçekte Ne Yapar

> **Benzetme —** Usta bir aşçı tarifini duraklarla yazar: "1) Soğanı kavur. 2) Fırına ver — **zil çalınca** devam. 3) Sosu ekle. 4) Tekrar fırına — **zil çalınca** devam. 5) Servis et." Aşçı ikinci maddeye gelince mutfaktan çıkar, başka bir yemeğe bakar. Zil çaldığında elindeki not sayesinde "üçüncü maddedeydim" der ve devam eder. `await` o "zil çalınca devam" işaretidir. Derleyicinin yaptığı şey ise, düz yazılmış tarifini bu duraklı listeye çevirmektir.

**Basitçe:** `async` kelimesi bir metodu asenkron yapmaz — derleyiciye "bu metodun içinde duraklar olabilir, ona göre düzenle" der. `await` ise gerçek iş yapan kelimedir: "burada bekleme var, bu metodun geri kalanını beklet, elemanı şimdilik serbest bırak" anlamına gelir. Kod yine yukarıdan aşağı okunur; tek fark, aradaki duraklarda başka işlerin yapılmış olmasıdır.

**Teknik olarak:**

**`async`** — Derleyiciye "bu metodun içinde `await` olabilir, gövdeyi bir durum makinesine (state machine) çevir" der. Metodu kendi başına asenkron **yapmaz**.

**`await`** — "Bu iş bitene kadar bu metodun geri kalanını beklet, thread'i şimdilik bırak" der.

### Derleyici ne üretir
`async` bir metot derlendiğinde, gövdesi parçalara bölünmüş bir durum makinesi sınıfına dönüşür. Her `await` bir devam noktasıdır. İş tamamlanınca **continuation** (devam bloğu) çalıştırılır.

```csharp
public async Task<Musteri> GetirAsync(int id)
{
    var musteri = await _repo.BulAsync(id);   // ① burada thread serbest kalır
    var siparis = await _repo.SiparisAsync(id); // ② tekrar serbest kalır
    musteri.Siparisler = siparis;
    return musteri;                            // ③ Task tamamlanır
}
```

Akış: metot ①'e kadar senkron çalışır → `BulAsync` henüz bitmediyse metot **geri döner** ve çağırana yarım kalmış bir `Task` verir → veritabanı cevabı gelince kalan kısım devam eder.

**`await` sırayı bozmaz.** Kod yukarıdan aşağı okunur; sadece arada thread başka işler yapmıştır.

Durum makinesinin somut anlamı şudur: metodun yerel değişkenleri artık stack'te değil, derleyicinin ürettiği bir sınıfın alanlarında tutulur. Çünkü metot ortasından çıkıp geri dönebilmelidir; stack o sırada başka işlerle dolmuş olacaktır. Bu, `async` metodun küçük bir bellek maliyeti olduğu anlamına gelir — gereksiz `async` kullanmamanın sebebi budur (6.3'e bak).

### Her `await` gerçekten bekler mi?

Hayır. `await` edilen iş **zaten tamamlanmışsa** durum makinesi hiç devreye girmez, kod kesintisiz devam eder. Yani `await`, "her zaman durur" değil, "gerekirse durur" demektir.

```csharp
public async Task<Urun> GetirAsync(int id)
{
    if (_onbellek.TryGetValue(id, out var urun))
        return urun;                       // await yok, tamamen senkron akış

    return await _repo.BulAsync(id);       // burada gerçekten duraklayabilir
}
```

Bu davranış, 9. bölümdeki `ValueTask`'ın varlık sebebidir: çoğu zaman senkron biten metotlar için her seferinde `Task` nesnesi üretmek israftır.

> **Bu benzetme şurada bozulur:** Aşçı örneğinde zil çaldığında mutfağa **aynı aşçı** döner. .NET'te ise devam bloğunu havuzdaki herhangi bir thread çalıştırabilir. Bu genelde önemsizdir ama bazı ortamlarda (masaüstü uygulamalarında arayüz thread'i gibi) kritiktir: arayüzü sadece kendi thread'i güncelleyebilir. `ConfigureAwait` tam olarak bu "kim devam etsin" sorusunu ayarlar.

---

## 5. Ne Kazandırır, Ne Kazandırmaz

> **Benzetme —** İki şeritli bir yolu dört şeride çıkardın. Arabaların hızı değişmedi — hâlâ 90 ile gidiyorlar. Ama aynı sürede yoldan geçen araç sayısı ikiye katlandı. Asenkronluk yolu genişletir, arabayı hızlandırmaz. Trafik sıkışıklığı geçer, yolculuk süresi aynı kalır.

**Basitçe:** Asenkron yazmak, tek bir işlemin süresini kısaltmaz. Aynı sunucunun aynı anda kaç kişiye hizmet edebileceğini artırır. Bu ayrımı kaçırırsan, "async yaptım ama sayfa hâlâ 300 ms'de açılıyor" diye şaşırırsın — açılmalı zaten, orası değişmedi.

**Teknik olarak:**

**Kazandırır**
- Aynı sunucuda çok daha fazla eşzamanlı istek
- Masaüstü/mobil uygulamalarda donmayan arayüz
- Kaynakların (thread, bellek) verimli kullanımı

**Kazandırmaz**
- Tek bir işlemin süresini kısaltmaz. 300 ms'lik sorgu yine 300 ms sürer
- CPU-bound işi hızlandırmaz
- Kod karmaşıklığını azaltmaz — aksine bir miktar artırır

Tek bir isteği gerçekten hızlandırmanın tek asenkron yolu, **bağımsız işleri aynı anda başlatmaktır** (8. bölüm). Bu bile asenkronluğun kendi kazancı değildir; eşzamanlılığın kazancıdır.

```csharp
// Sıralı: 200 + 150 = 350 ms
var musteri = await _repo.MusteriAsync(id);
var siparis = await _repo.SiparislerAsync(id);

// Birlikte: max(200, 150) = 200 ms
var mGorev = _repo.MusteriAsync(id);
var sGorev = _repo.SiparislerAsync(id);
await Task.WhenAll(mGorev, sGorev);
```

İkinci örnekte kazanılan süre, `async` sayesinde değil, iki işin **beklemelerinin üst üste binmesi** sayesindedir.

> **Bu benzetme şurada bozulur:** Yol genişletme örneğinde şeritler eklemek her zaman iyi gibi görünür. Asenkronlukta ise genişleyen şerit yükü bir sonraki darboğaza taşır: sunucun artık binlerce isteği aynı anda veritabanına iletebilir ve bu sefer **veritabanı** tıkanır. Asenkronluk darboğazı ortadan kaldırmaz, yerini değiştirir. Bu yüzden yük altında ilk bakacağın yer her zaman sunucu olmayabilir.

---

## 6. Klasik Tuzaklar

> **Benzetme —** Bu bölümdeki hataların hepsi aynı kökten çıkar: **asenkron bir işi zorla senkron bekletmek**. Yani çamaşır makinesini çalıştırıp başına dikilmek. Makine aynı sürede biter, sen hiçbir şey kazanmazsın, üstelik o süre boyunca evdeki diğer işler durur. Kodda bunun adı `.Result` ve `.Wait()`'tir. En kötü hâlinde iş bitmez bile: makinenin bitmesi için senin kalkıp bir düğmeye basman gerekiyorsa ve sen de makinenin bitmesini bekliyorsan, ikiniz sonsuza kadar birbirinizi beklersiniz.

**Basitçe:** Asenkron kod yazmanın kolay kısmı `async` ve `await` yazmaktır. Zor kısmı, zincirin hiçbir yerinde onu senkron koda **geri çevirmemektir**. Aşağıdaki beş hatanın hepsi, bir yerde "ben burada bekleyeyim olsun bitsin" demekten doğar. Bu bölümü ezberle değil, sebebiyle öğren — çünkü bu hatalar test ortamında görünmez, yük altında ortaya çıkar.

**Teknik olarak:** Hataların ortak adı **sync-over-async**tır: asenkron bir metodun sonucunu senkron olarak beklemek.

### 6.1 `.Result` ve `.Wait()` — deadlock üreticisi

```csharp
// YAPMA
var musteri = _service.GetirAsync(id).Result;
```

**Basitçe ne oluyor:** Tek kasiyerli bir market düşün. Kasiyer bir ürünün fiyatını depoya soruyor ve cevabı beklemek için kasada duruyor, sıradakine bakmıyor. Depodaki eleman ise cevabı **bizzat kasiyere** söylemek zorunda; kasiyer meşgul olduğu için sıraya giriyor. Kasiyer cevabı bekliyor, cevap ise kasiyerin boşalmasını bekliyor. İkisi de sonsuza kadar birbirini bekler. Market kapanmaz, kilitlenir. Kodda buna **deadlock** denir.

**Teknik olarak:** Bu satır asenkron işi senkron beklemeye zorlar. Bloke olan thread, işin tamamlanması için gereken devam bloğunun çalışacağı thread'in ta kendisi olabilir — o zaman uygulama **kilitlenir** (deadlock).

Klasik deadlock'un şartı, ortamda bir **`SynchronizationContext`** bulunmasıdır: "devam bloğu şu thread'de çalışmalı" kuralını koyan yapı. Eski ASP.NET (System.Web) ve masaüstü uygulamaları (WinForms, WPF) böyle bir bağlam kullanır.

```csharp
// Masaüstü uygulamasında klasik kilitlenme
private void Button_Click(object sender, EventArgs e)
{
    var veri = VeriGetirAsync().Result;   // UI thread bloke oldu
    label.Text = veri;                    // buraya asla gelinmez
}

private async Task<string> VeriGetirAsync()
{
    await Task.Delay(1000);               // devam bloğu UI thread'ini istiyor
    return "hazır";                       // UI thread bloke, sıra hiç gelmiyor
}
```

ASP.NET Core'da klasik `SynchronizationContext` kaldırıldığı için bu tam deadlock daha nadirdir; ama thread pool açlığı sorunu aynen kalır.

Yani ASP.NET Core'da `.Result` yazmak uygulamayı hemen kilitlemez — daha sinsi bir şey yapar. Her `.Result` çağrısı bir thread'i tam süre boyunca bloke eder. Yük arttıkça havuz tükenir, istekler kuyruğa girer, yanıt süreleri uzar. Hiçbir hata da görmezsin.

**Kural: async, çağrı zincirinin en tepesine kadar gider.** Buna *"async all the way"* denir. Bir yerde senkron bloke edersen kazanılan her şey kaybolur.

```csharp
// Zincirin her halkası async — doğru
public async Task<IActionResult> Getir(int id)
{
    var musteri = await _service.GetirAsync(id);   // .Result yok
    return Ok(musteri);
}

public async Task<Musteri> GetirAsync(int id)
    => await _repo.BulAsync(id);
```

**Peki gerçekten senkron bir yerdeysem?** Bazen elinde async olmayan bir arayüz olur (eski bir kütüphane, bir `Main` metodu, bir constructor). Sırasıyla dene:
1. Metodu `async` yapabiliyor musun? (`Main` bile `async Task Main` olabilir.)
2. Senkron bir alternatifi var mı? (`File.ReadAllText` / `File.ReadAllTextAsync` gibi.)
3. Hiçbiri olmuyorsa `GetAwaiter().GetResult()` kullan — `.Result`'tan tek farkı, hatayı `AggregateException` içine sarmadan olduğu gibi fırlatmasıdır. Yine de bloke eder; son çare olarak bilinçli kullanılır.

```csharp
// Modern: Main bile async olabilir
public static async Task Main(string[] args)
{
    await UygulamaCalistirAsync();
}
```

> Constructor içinde asla asenkron iş bekleme. İhtiyacın varsa bir `static async Task<T> CreateAsync()` fabrika metodu yaz.

### 6.2 `async void`

```csharp
public async void KaydetAsync() { ... }     // YAPMA
```

**Basitçe ne oluyor:** Kargoyu takip numarası almadan göndermek gibidir. Paket yola çıkar ama elinde hiçbir kayıt yoktur: ne bittiğini bilirsin, ne kaybolduğunu. Yolda bir sorun çıkarsa haberin bile olmaz — daha doğrusu, haberin **uygulamanın çökmesiyle** olur.

**Teknik olarak:** Neden yanlış:
- Çağıran taraf **bekleyemez** — iş bitmeden devam eder
- İçindeki exception yakalanamaz; doğrudan uygulamayı çökertir
- Test edilemez

`Task` döndüren bir metotta fırlayan exception `Task` nesnesinin içinde saklanır ve `await` ettiğinde sana ulaşır. `async void`'de ise saklanacak bir `Task` yoktur; exception doğrudan `SynchronizationContext`'e ya da thread pool'a düşer ve süreci sonlandırır.

```csharp
// async void: try/catch bunu YAKALAYAMAZ
try
{
    KaydetVoid();            // metot geri döner, iş hâlâ devam ediyor
}
catch (Exception ex)
{
    // buraya asla gelinmez; exception başka yerde patlar
}

// Task döndüren hâli: yakalanır
try
{
    await KaydetAsync();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Kayıt başarısız");
}
```

**Tek istisna:** UI olay yöneticileri (`button_Click`). Onun dışında her zaman `Task` döndür. UI olay yöneticisinde bile içeriyi `try/catch` ile sarmak zorundasın, çünkü dışarıda kimse yakalayamaz.

### 6.3 Gereksiz `async` sarmalama

```csharp
// Gereksiz — durum makinesi maliyeti ekliyor, hiçbir şey katmıyor
public async Task<int> SayAsync() => await _repo.CountAsync();

// Yeterli
public Task<int> SayAsync() => _repo.CountAsync();
```

**Basitçe ne oluyor:** Kargodan gelen paketi açıp, içindekini yeni bir kutuya koyup öyle teslim etmek gibi. Alıcı aynı şeyi alıyor, sen fazladan bir kutu harcadın.

Sadece `Task`'ı geçiriyorsan `async`/`await` ekleme. (İstisna: `using` bloğu içindeysen `await` etmen gerekir, yoksa kaynak iş bitmeden kapanır.)

```csharp
// BURADA await ŞART — using olmadan connection iş bitmeden kapanır
public async Task<int> SayAsync()
{
    using var baglanti = _fabrika.Olustur();
    return await baglanti.CountAsync();    // await kaldırılırsa bağlantı erken kapanır
}
```

Aynı kural `try/catch` için de geçerlidir: hatayı bu metotta yakalamak istiyorsan `await` etmelisin, yoksa exception henüz fırlamamış olur.

### 6.4 Döngü içinde `await`

```csharp
// Sıralı — 100 istek × 200 ms = 20 saniye
foreach (var id in idler)
    sonuclar.Add(await ApiCagirAsync(id));

// Eşzamanlı — hepsi birlikte, ~200 ms
var gorevler = idler.Select(id => ApiCagirAsync(id));
var sonuclar2 = await Task.WhenAll(gorevler);
```

**Basitçe ne oluyor:** Evde üç çamaşır makinesi varken birini çalıştırıp bitmesini beklemek, sonra ikinciyi, sonra üçüncüyü. Üçünü birden çalıştırabilecekken üç kat süre harcamak.

> Dikkat: EF Core'un `DbContext`'i thread güvenli **değildir**. Aynı context üzerinde `Task.WhenAll` ile paralel sorgu çalıştıramazsın; her iş için ayrı context gerekir.

Ayrıca "hepsini birden başlat" her zaman doğru değildir. Yüz isteği aynı anda bir API'ye yollarsan karşı taraf seni sınırlayabilir (rate limit) ya da bağlantı havuzu tükenebilir. Böyle durumlarda eşzamanlılığı sınırlaman gerekir:

```csharp
// En fazla 5 iş aynı anda
using var kapi = new SemaphoreSlim(5);

var gorevler = idler.Select(async id =>
{
    await kapi.WaitAsync();
    try   { return await ApiCagirAsync(id); }
    finally { kapi.Release(); }
});

var sonuclar3 = await Task.WhenAll(gorevler);
```

### 6.5 `Task.Run` ile I/O sarmalama

```csharp
// YANLIŞ — I/O zaten thread kullanmıyordu, sen bir thread harcadın
var veri = await Task.Run(() => _httpClient.GetStringAsync(url).Result);

// DOĞRU
var veri2 = await _httpClient.GetStringAsync(url);
```

**Basitçe ne oluyor:** Çamaşır makinesi kendi kendine dönerken, başına "makineye baksın" diye bir kişi dikmek. O kişi hiçbir iş yapmıyor, sadece havuzdan eksiliyor.

`Task.Run`, CPU-bound iş içindir — birinin fiilen hesap yapması gerektiğinde. Web sunucusunda ise `Task.Run` kullanmak çoğu zaman yanlıştır: istek zaten bir thread pool thread'inde çalışıyordur, işi başka bir thread pool thread'ine devretmek net kayıptır.

> **Bu benzetme şurada bozulur:** Çamaşır makinesi benzetmesi "beklemek boşunadır" fikrini iyi anlatır ama deadlock'u tam karşılamaz. Gerçek hayatta makinenin başında beklersen makine yine de biter; sen sadece vakit kaybedersin. Kodda ise bazı durumlarda iş **hiç bitmez**, çünkü işi bitirecek olan tam da senin bloke ettiğin kişidir. "Beklemek yavaşlatır" ile "beklemek işi imkânsızlaştırır" arasındaki fark budur ve bu yüzden `.Result` sadece verimsiz değil, tehlikelidir.

---

## 7. `CancellationToken`

> **Benzetme —** Kargo şubesini arayıp "siparişten vazgeçtim, yola çıkmadıysa göndermeyin" dersin. Bu bir **emir** değil, bir **haberdir**: kurye yoldaysa yine de gelir, depodaysa iptal edilir. Kimseyi zorla durduramazsın; sadece "artık gerek yok" sinyalini iletirsin. Önemli olan, bu haberin zincirdeki herkese ulaşmasıdır. Sen şubeye söyleyip şube depoya söylemezse haber boşa gitmiştir.

**Basitçe:** `CancellationToken`, "bu işe artık gerek kalmadı" haberini taşıyan küçük bir nesnedir. Kullanıcı sayfayı kapattığında ya da tarayıcı isteği iptal ettiğinde, sunucunun o istek için veritabanını yormaya devam etmesinin anlamı yoktur. Token'ı metoduna parametre olarak alır ve çağırdığın bütün alt metotlara **geçirirsin**. Geçirmezsen hiçbir işe yaramaz.

**Teknik olarak:** **`CancellationToken`** — Bir işin "artık gerek yok, dur" sinyalini taşıyan yapı. Kullanıcı sayfayı kapattığında, istek zaman aşımına uğradığında veya uygulama kapanırken devreye girer.

```csharp
public async Task<IActionResult> Ara(string q, CancellationToken ct)
{
    var sonuc = await _service.AraAsync(q, ct);
    return Ok(sonuc);
}
```

ASP.NET Core, controller metoduna `CancellationToken` parametresi koyarsan onu **otomatik doldurur** ve istemci bağlantıyı kestiğinde iptal sinyali gönderir. Bu, iptal edilmiş istekler için boşuna veritabanı işi yapmayı önler.

Token'ı çağırdığın alt metotlara **geçirmen** gerekir; yoksa hiçbir işe yaramaz. İptal gerçekleştiğinde `OperationCanceledException` fırlatılır — bu bir hata değil, beklenen akıştır ve genelde loglanmaz.

```csharp
public async Task<List<Urun>> AraAsync(string q, CancellationToken ct)
{
    // Token aşağı geçiyor — EF Core sorguyu gerçekten iptal edebilir
    return await _context.Urunler
        .Where(u => u.Ad.Contains(q))
        .ToListAsync(ct);
}
```

Uzun süren kendi döngülerinde token'ı elle kontrol etmelisin:

```csharp
foreach (var satir in satirlar)
{
    ct.ThrowIfCancellationRequested();   // iptal edildiyse burada çıkar
    Isle(satir);
}
```

Kendi zaman aşımını kurmak istersen `CancellationTokenSource` kullanılır ve istek token'ıyla birleştirilebilir:

```csharp
using var zamanAsimi = CancellationTokenSource.CreateLinkedTokenSource(ct);
zamanAsimi.CancelAfter(TimeSpan.FromSeconds(5));

var sonuc = await _service.AraAsync(q, zamanAsimi.Token);
```

İptali yakalarken hata loglamasından ayırmayı unutma:

```csharp
try
{
    await IsYapAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // Beklenen akış — loglama, hata sayma
    return NoContent();
}
```

> **Bu benzetme şurada bozulur:** Kargo örneğinde "vazgeçtim" demek her zaman güvenlidir; en kötü ihtimalle paket yine gelir. Kodda ise iptal **yarıda kalmış iş** bırakabilir: veritabanına iki kayıttan biri yazılmış, ikincisi yazılmamış olabilir. İptal edilebilir bir işlemin arkasında bir transaction yoksa, iptal veriyi tutarsız bırakır. Yani iptal sadece "durdurma" değil, "nerede durursam güvenli olur" sorusudur.

---

## 8. Birden Çok İşi Birlikte Çalıştırma

> **Benzetme —** Akşam yemeği için üç iş var: pilav, tavuk, salata. Üçünü sırayla yapmak yerine hepsini birden başlatırsın; toplam süre en uzun işin süresi kadar olur. `Task.WhenAll` budur. Bazen de "iki markete de adam yolladım, hangisi önce gelirse onunla yetinirim" dersin — bu da `Task.WhenAny`.

**Basitçe:** Birbirine bağlı olmayan işleri arka arkaya beklemek için sebep yoktur. Hepsini başlat, sonra hepsinin bitmesini bekle. Toplam süre, işlerin toplamı değil, en uzun olanı kadar olur. Ama bu yalnızca işler gerçekten **bağımsızsa** doğrudur; birinin sonucu diğerine giriyorsa sıralı beklemek zorundasın.

**Teknik olarak:**

| Metot | Davranış |
|---|---|
| `Task.WhenAll(...)` | Hepsi bitene kadar bekler. Sonuçları dizi olarak döndürür |
| `Task.WhenAny(...)` | İlk biten yeter — zaman aşımı ve yedekli çağrılarda kullanılır |
| `Task.Delay(ms)` | Asenkron bekleme. **`Thread.Sleep` kullanma** — o thread'i bloke eder |
| `Task.Run(...)` | CPU-bound işi thread pool'a atar. I/O için gereksizdir |

```csharp
var musteriGorev = _repo.MusteriAsync(id);
var siparisGorev = _repo.SiparislerAsync(id);

await Task.WhenAll(musteriGorev, siparisGorev);   // ikisi birlikte

var musteri  = musteriGorev.Result;   // WhenAll sonrası .Result güvenlidir
var siparis  = siparisGorev.Result;
```

Burada `.Result` kullanmak istisnaen güvenlidir çünkü `WhenAll` döndüğünde işler **zaten bitmiştir**; bekleme yoktur. Yine de alışkanlık olarak `await musteriGorev` yazmak daha okunaklıdır ve yanlış örnek teşkil etmez.

`Task.WhenAll` üzerine iki kritik ayrıntı:

**Birincisi, görevleri ne zaman başlattığına dikkat et.** `WhenAll`'a verdiğin liste tembel bir LINQ sorgusuysa, görevler `WhenAll` çağrıldığında değil, liste gezilince başlar — ve bu beklediğinden farklı davranabilir. Güvenli yol, listeyi önce maddeleştirmektir:

```csharp
var gorevler = idler.Select(id => ApiCagirAsync(id)).ToList();  // hepsi ŞİMDİ başladı
var sonuclar = await Task.WhenAll(gorevler);
```

**İkincisi, hata davranışı.**

> `Task.WhenAll` birden çok görev hata verirse yalnızca **ilk** exception'ı fırlatır. Hepsini görmek için dönen `Task`'ın `Exception` özelliğine (`AggregateException`) bakmak gerekir.

```csharp
var hepsi = Task.WhenAll(gorevler);
try
{
    await hepsi;
}
catch
{
    // await sadece ilkini fırlattı; tamamı burada
    foreach (var hata in hepsi.Exception!.InnerExceptions)
        _logger.LogError(hata, "Görev başarısız");
}
```

Ayrıca `WhenAll` hata verse bile **diğer görevler iptal olmaz**; arka planda çalışmaya devam ederler. İptal istiyorsan ortak bir `CancellationToken` geçirmelisin.

`Task.WhenAny`'nin en yaygın kullanımı zaman aşımıdır:

```csharp
var isGorev = UzunIsAsync();
var sure    = Task.Delay(TimeSpan.FromSeconds(3));

var ilkBiten = await Task.WhenAny(isGorev, sure);

if (ilkBiten == sure)
    throw new TimeoutException("İş 3 saniyede bitmedi");

var sonuc = await isGorev;   // hatayı doğru şekilde fırlatması için tekrar await
```

> Not: `WhenAny` kaybeden görevi durdurmaz. Yukarıdaki örnekte `UzunIsAsync` arka planda çalışmaya devam eder. Gerçekten durdurmak için `CancellationToken` gerekir.

> **Bu benzetme şurada bozulur:** Yemek örneğinde üç ocağı birden yakmak hep kazançlıdır. Kodda ise "hepsini birden başlat" bir sınırı zorlayabilir: aynı `DbContext`'i paylaşan iki sorgu çöker, aynı API'ye yüz istek rate limit yer, yüzlerce eşzamanlı bağlantı havuzu tüketir. Mutfakta ocak sayısı bellidir; koda da o sınırı sen koymalısın (6.4'teki `SemaphoreSlim` kalıbı).

---

## 9. Diğer Kavramlar

> **Benzetme —** Lokanta benzetmesine dönelim. Bazı lokantalarda katı bir kural vardır: bir masaya hangi garson baktıysa tabağı da **o** götürmeli. Bu kural müşteri için iyidir ama mutfak hazır olduğunda o garson başka masadaysa tabak bekler. Bazı lokantalarda ise "kim boştaysa götürsün" denir; daha hızlıdır ama masayı tanıyan garson gelmez. `ConfigureAwait` tam olarak bu kuralı ayarlar: devam eden iş, önceki thread'e dönmek zorunda mı, yoksa kim boşsa o mu devam etsin?

**Basitçe:** Bu bölüm, günlük kodda her gün kullanmayacağın ama karşılaşınca ne olduğunu bilmen gereken dört konuyu topluyor: devam bloğunun hangi thread'de çalışacağı, hafif bir `Task` alternatifi, `await` edilmeyen hataların nereye gittiği ve veriyi parça parça asenkron okuma.

**Teknik olarak:**

### `ConfigureAwait(false)`

**`ConfigureAwait(false)`** — "Devam bloğu orijinal bağlama (UI thread'i gibi) dönmek zorunda değil" demektir. Kütüphane kodunda performans ve deadlock güvenliği için önerilir. **ASP.NET Core'da gerekmez** (klasik `SynchronizationContext` yoktur), ama yeniden kullanılabilir kütüphane yazıyorsan alışkanlık edinmek iyidir.

Neden kütüphanede önemli? Çünkü kütüphaneni kimin çağıracağını bilmezsin. Bir WinForms uygulaması çağırırsa, orada "aynı garson" kuralı vardır ve senin kodun gereksiz yere UI thread'ini bekletir — hatta çağıran taraf `.Result` kullanmışsa kilitlenir. `ConfigureAwait(false)` bu bağımlılığı keser.

```csharp
// Kütüphane kodu: bağlama dönmeye ihtiyacı yok
public async Task<string> IndirAsync(string url)
{
    var yanit = await _http.GetAsync(url).ConfigureAwait(false);
    return await yanit.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

| Ortam | `ConfigureAwait(false)` gerekli mi |
|---|---|
| ASP.NET Core uygulama kodu | Hayır — bağlam zaten yok |
| WinForms / WPF olay yöneticisi | Hayır — UI'a dönmen **gerekiyor** |
| WinForms / WPF içindeki yardımcı metotlar | Evet |
| Paylaşılan kütüphane (NuGet paketi) | Evet |

Dikkat: `ConfigureAwait(false)` deadlock'u **çözmez**, sadece bir türünü engeller. `.Result` kullanmaya devam edersen thread pool açlığı sorunu aynen sürer. Doğru çözüm her zaman `.Result`'ı kaldırmaktır.

### `ValueTask<T>`

**`ValueTask<T>`** — Metot çoğu zaman **senkron** tamamlanıyorsa (örneğin cache'ten dönüyorsa), her seferinde bir `Task` nesnesi ayırmamak için kullanılır. Kuralları katıdır: birden çok kez `await` edilemez, saklanamaz. Emin değilsen `Task` kullan.

**Basitçe:** Vestiyerde montun zaten elindeyse fiş kesmenin anlamı yoktur. `ValueTask`, "sonuç hazırsa doğrudan ver, hazır değilse fiş kes" demenin yoludur.

```csharp
public ValueTask<Urun> GetirAsync(int id)
{
    if (_onbellek.TryGetValue(id, out var urun))
        return new ValueTask<Urun>(urun);        // ayırma yok, doğrudan sonuç

    return new ValueTask<Urun>(YavasGetirAsync(id));
}
```

Katı kuralları şunlardır ve ihlal edilirse davranış tanımsızdır:
- Bir `ValueTask` yalnızca **bir kez** `await` edilebilir.
- Alana atanıp saklanamaz, listede tutulamaz.
- `Task.WhenAll`'a doğrudan verilemez — önce `.AsTask()` ile çevirmek gerekir.

Bu yüzden `ValueTask`, ölçüm yaparak "burada gerçekten ayırma maliyeti var" dediğin yerlerde kullanılır. Varsayılanın `Task` olsun.

### Exception davranışı

**Exception davranışı** — `async` bir metotta fırlayan exception `Task` içinde saklanır ve `await` edildiğinde yeniden fırlatılır. `await` etmezsen **hata sessizce kaybolur** ("unobserved exception"). Bu, `async void`'in tehlikesinin de kaynağıdır.

**Basitçe:** Mektup gönderdin, iade geldi ama posta kutunu hiç açmadın. Hata oldu, kimse görmedi.

```csharp
// Hata sessizce kaybolur — Task hiç await edilmiyor
_ = KaydetAsync();          // "fire and forget"

// En azından hatayı gör
_ = KaydetAsync().ContinueWith(
        t => _logger.LogError(t.Exception, "Arka plan işi başarısız"),
        TaskContinuationOptions.OnlyOnFaulted);
```

Bir diğer ayrıntı: `async` metotta fırlayan exception, metot **çağrıldığında** değil, `await` edildiğinde ortaya çıkar. Yani şu kodda `try` bloğu hatayı yakalamaz:

```csharp
Task gorev;
try
{
    gorev = KaydetAsync();      // exception henüz fırlamadı, Task'ın içinde
}
catch { /* buraya gelinmez */ }

await gorev;                    // hata BURADA fırlar, try dışında
```

ASP.NET Core'da "fire and forget" işleri için doğru yol `IHostedService` / `BackgroundService` ya da bir kuyruk kullanmaktır. İstek bitince uygulamanın o işi tamamlayacağına dair hiçbir garanti yoktur.

### `IAsyncEnumerable<T>`

**`IAsyncEnumerable<T>`** — Asenkron akış. Veriyi tümü hazır olmadan, geldikçe işlemeni sağlar. `await foreach` ile gezilir. EF Core'da `AsAsyncEnumerable()` ile büyük sonuç kümelerini belleğe almadan işlemek için kullanılır.

**Basitçe:** `Task<List<T>>` "hepsi hazır olunca tek seferde ver" demektir; `IAsyncEnumerable<T>` ise "geldikçe ver, ben işleyeyim". Bir milyon satırı belleğe almadan işlemenin yoludur.

```csharp
await foreach (var kayit in _context.Loglar.AsAsyncEnumerable())
{
    Isle(kayit);                 // satırlar geldikçe işlenir, hepsi belleğe alınmaz
}

// Kendi akışını üretmek
public async IAsyncEnumerable<string> SatirlariOkuAsync(
    string yol,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var okuyucu = new StreamReader(yol);
    string? satir;
    while ((satir = await okuyucu.ReadLineAsync(ct)) is not null)
        yield return satir;
}
```

> **Bu benzetme şurada bozulur:** Lokanta kuralı benzetmesi `ConfigureAwait`'i iyi anlatır ama bir yanılgıya yol açabilir: "false yazarsam daha hızlı olur" gibi. ASP.NET Core'da dönülecek bir bağlam zaten yoktur, dolayısıyla `ConfigureAwait(false)` orada ölçülebilir bir kazanç sağlamaz — sadece gürültü yapar. Fayda, bağlamı olan ortamlarda çağrılma **ihtimali** olan kütüphane kodundadır.

---

## 10. Kurallar

> **Benzetme —** Her ustanın çırağa verdiği kısa bir liste vardır: "şuraya elini sokma, şu kabloyu önce kes, şunu asla açık bırakma". Kuralların sebebi genelde uzundur ama kuralın kendisi kısadır, çünkü iş üstünde düşünecek vaktin yoktur. Aşağıdaki sekiz madde asenkron programlamanın o listesidir. Sebeplerini yukarıdaki dokuz bölümde okudun; burada sadece hatırlatma var.

**Basitçe:** Bu liste, asenkron kod yazarken ezberden uygulayabileceğin sekiz kuraldır. Bir kod incelemesinde önce bunlara bakılır.

**Teknik olarak:**

1. **Async all the way** — zincirin hiçbir yerinde `.Result` / `.Wait()` yok.
2. **`async void` yazma** — UI olay yöneticileri hariç.
3. **I/O için `async`, CPU için paralellik.** İkisini karıştırma.
4. **Metot adları `Async` ile bitsin** — `GetirAsync`, `KaydetAsync`. Sadece gelenek değil, çağıranı uyarır.
5. **`CancellationToken`'ı parametre olarak al ve aşağı geçir.**
6. **`Thread.Sleep` yerine `await Task.Delay`.**
7. **Bağımsız işleri `Task.WhenAll` ile birlikte çalıştır** — ama paylaşılan `DbContext` ile değil.
8. **Sadece geçiriyorsan `async` ekleme** — gereksiz durum makinesi üretme.

Kod incelemesinde arayacağın kalıplar, kısa bir liste hâlinde:

| Gördüğünde şüphelen | Neden |
|---|---|
| `.Result` / `.Wait()` | Sync-over-async; thread bloke, deadlock riski |
| `async void` | Hata yakalanamaz, beklenemez |
| `Task.Run(() => ... Async(...))` | I/O işini boşuna thread'e atıyor |
| `Thread.Sleep` | Thread'i bloke eder; `Task.Delay` olmalı |
| `foreach` içinde `await` (bağımsız işler) | Sıralı çalışıyor, `WhenAll` olabilir |
| `CancellationToken` alınıp geçirilmemiş | İptal hiç çalışmaz |
| `async` metotta `await` yok | Gereksiz durum makinesi ya da unutulmuş `await` |

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Concurrency / Parallelism / Asynchrony | İç içe ilerleme / fiziksel eşzamanlılık / beklemeden dönme |
| I/O-bound / CPU-bound | Beklemeye dayalı / hesaplamaya dayalı iş |
| Thread | İşletim sisteminin zamanladığı yürütme birimi |
| Thread pool | Yeniden kullanılan thread havuzu |
| Thread pool starvation | Havuzdaki thread'lerin tükenmesi, uygulamanın yanıtsız kalması |
| Task | Gelecekte tamamlanacak işin temsili |
| Promise task / delegate task | I/O bekleyen Task / `Task.Run` ile thread'de çalışan Task |
| State machine | Derleyicinin `async` metottan ürettiği durum makinesi |
| Continuation | `await` sonrası çalışacak devam bloğu |
| SynchronizationContext | "Devam bloğu şu thread'de çalışsın" kuralını koyan yapı |
| Sync-over-async | Asenkron işi senkron beklemek (`.Result`, `.Wait()`) |
| Deadlock | Karşılıklı bekleme sonucu oluşan kilitlenme |
| async all the way | Zincirin tamamının asenkron tutulması ilkesi |
| CancellationToken | İptal sinyalini taşıyan yapı |
| CancellationTokenSource | İptal sinyalini üreten ve zaman aşımı kurmaya yarayan yapı |
| Task.WhenAll / WhenAny | Hepsini bekle / ilk bitenle devam et |
| AggregateException | Birden çok hatayı içinde taşıyan exception |
| SemaphoreSlim | Aynı anda çalışacak iş sayısını sınırlayan kapı |
| ConfigureAwait(false) | Devam bloğunun orijinal bağlama dönmesini gereksiz kılma |
| ValueTask | Senkron tamamlanan işler için hafif Task alternatifi |
| Unobserved exception | `await` edilmediği için fark edilmeyen hata |
| Fire and forget | Başlatılıp sonucu beklenmeyen iş |
| IAsyncEnumerable | Asenkron veri akışı, `await foreach` ile gezilir |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `05-Modern-CSharp-Ozellikleri.md` (Cuma)
