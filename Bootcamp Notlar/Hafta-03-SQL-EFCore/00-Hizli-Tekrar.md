# Hafta 3 — Hızlı Tekrar: SQL Server ve EF Core

**Okuma süresi:** ~17 dk

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

1. **İlişkisel Modelleme ve Index** — `01-Iliskisel-Modelleme-ve-Index.md` · Pazartesi
2. **T-SQL Sorgulama** — `02-T-SQL-Sorgulama.md` · Salı
3. **Stored Procedure, View, Transaction** — `03-Stored-Procedure-View-Transaction.md` · Çarşamba
4. **EF Core Temelleri ve Migration** — `04-EF-Core-Temelleri-ve-Migration.md` · Perşembe
5. **İlişkiler, Fluent API ve Data Annotations** — `05-Iliskiler-ve-Fluent-API.md` · Cuma
6. **EF Core Performansı** — `06-EF-Core-Performans.md` · Cumartesi
7. **Hafta Özeti: SQL Server ve EF Core** — `07-Hafta-Ozeti.md` · Pazar
8. **Dapper'a Giriş** — `08-Dapper-Giris.md` · Ek Not

---

## 1. İlişkisel Modelleme ve Index

*Kaynak: [`01-Iliskisel-Modelleme-ve-Index.md`](01-Iliskisel-Modelleme-ve-Index.md) · Pazartesi*

- PK yapay (`IDENTITY` / sequential GUID) olsun; doğal anahtarı `UNIQUE` ile koru.
- FK referans bütünlüğünü garanti eder ama **index'ini yaratmaz** — FK sütunlarına elle index koy.
- `ON DELETE CASCADE` sadece gerçek sahiplik ilişkisinde (`Siparis → SiparisDetay`) meşrudur.
- 1NF: hücreye liste tıkıştırma. 2NF: anahtarın yarısına bağlı sütun olmasın. 3NF: sütun sütuna bağlı olmasın.
- Denormalizasyon son çaredir; önce ölç, sonra index dene, sonra sorguyu düzelt.
- Para `DECIMAL`, asla `float`. Kullanıcı metni `NVARCHAR`. Tarih `DATETIME2`, `DATETIME` değil.
- `NULL` "bilinmiyor" demektir; `NULL = NULL` doğru değildir, `IS NULL` yazılır.
- `NOT IN` + `NULL` sorguyu sessizce boşaltır; `NOT EXISTS` yaz.
- Clustered index tablo başına bir tanedir ve satırın kendisini tutar; non-clustered işaretçi tutar.
- Composite index soldan sağa kullanılır: önce eşitlik sütunları, sonra aralık sütunları.
- `INCLUDE` key lookup'ı bitirir; ağacı şişirmez çünkü sadece yaprakta durur.
- Düşük seçicilikli sütuna, küçük tabloya ve ağır yazılan log tablosuna index koyma.
- Sütunu fonksiyona sokarsan index ölür; `YEAR(Tarih) = 2026` yerine aralık yaz.
- Ölçü birimi `logical reads`tır; süre değil. Index'ten önce ve sonra karşılaştır.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Doğal anahtar varsa PK yapılır" | PK yapay olur, doğal anahtar `UNIQUE` ile korunur |
| "GUID her yerde daha güvenli, hep onu kullan" | `NEWID()` clustered index'i parçalar; bedelini her index'te ödersin |
| "FK koyunca index de gelir" | PK tarafında gelir, FK sütununda **gelmez** |
| "`CASCADE` pratik, her FK'ya koyayım" | Zincirleme silme binlerce kaydı sessizce yok eder |
| "`JOIN` yavaştır, baştan denormalize edeyim" | Çoğu yavaşlık index eksikliğindendir; önce ölç |
| "`float` ondalık sayı tutar, para için uygundur" | `float` yaklaşıktır; para `DECIMAL` ile tutulur |
| "`NULL = NULL` doğrudur" | `UNKNOWN` döner; `IS NULL` yazılır |
| "`CHECK` kısıtı `NULL`'u da engeller" | `UNKNOWN` sonucu kabul edilir; `NOT NULL` ayrı yazılır |
| "Index ne kadar çoksa o kadar iyi" | Her index yazmayı yavaşlatır; az ve doğru olanı seç |
| "Composite index'te sütun sırası fark etmez" | Soldan sağa kullanılır; ortadaki sütuna tek başına dayanılmaz |
| "`WHERE YEAR(Tarih) = 2026` index kullanır" | Sütun fonksiyonda olduğu için index kullanılmaz |
| "Sorgu süresi (ms) iyi bir ölçüdür" | Makine yüküne göre değişir; `logical reads` ölç |

---


---

## 2. T-SQL Sorgulama

*Kaynak: [`02-T-SQL-Sorgulama.md`](02-T-SQL-Sorgulama.md) · Salı*

- Mantıksal sıra `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`'dır; alias `SELECT`'te doğar.
- Bu yüzden alias'ı `WHERE`, `GROUP BY` ve `HAVING`'de kullanamazsın; `ORDER BY`'da kullanabilirsin.
- `LEFT JOIN` yazıp sağ tablonun sütununa `WHERE` koşulu koyarsan sorgu `INNER JOIN`'e döner; koşulu `ON`'a taşı.
- `JOIN` satırları çoğaltır; iki detay tablosunu birlikte `SUM`'larsan fan-out hatası alırsın.
- `WHERE` satır eler, `HAVING` grup eler. Elenebilecek satırı `HAVING`'e bırakma.
- `COUNT(*)` satır sayar, `COUNT(kolon)` `NULL` olmayanları sayar, `AVG` `NULL`'ları paydaya almaz.
- Hiç satır yoksa `COUNT` 0, `SUM` `NULL` döner.
- Korelasyonlu alt sorgu her dış satır için çalışır; büyük veride `JOIN` veya `APPLY` tercih et.
- Varlık kontrolünde `EXISTS`, yokluk kontrolünde `NOT EXISTS` yaz.
- `NOT IN` + alt sorgu, listede tek bir `NULL` varsa sessizce boş sonuç döndürür.
- CTE sonucu saklamaz; okunaklılık sağlar, her kullanımda yeniden hesaplanabilir.
- Özyinelemeli CTE anchor + `UNION ALL` + recursive parçadan oluşur; `MAXRECURSION` varsayılanı 100'dür.
- `GROUP BY` satırları yok eder, `OVER` satırları korur.
- `RANK` eşitlikten sonra atlar, `DENSE_RANK` atlamaz, `ROW_NUMBER` eşitliği umursamaz.
- Window function `WHERE`'de kullanılamaz; CTE'ye sarıp dıştan filtrelersin.
- Sayfalamada `ORDER BY` ve tie-breaker şarttır; derin sayfada `OFFSET` yerine keyset kullan.
- Tekrar beklemiyorsan `UNION ALL` yaz; `UNION` bedava yavaşlıktır.
- `MERGE` güçlü ama riskli; ayrı `UPDATE` + `INSERT` çoğu zaman daha güvenlidir.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`SELECT` ilk çalışır, çünkü ilk yazılır" | `FROM` ilk çalışır; `SELECT` beşinci sıradadır |
| "`SELECT`'te verdiğim alias'ı `WHERE`'de kullanabilirim" | Kullanamazsın; `ORDER BY`'da kullanabilirsin |
| "`LEFT JOIN` yazdım, sol tablo her zaman korunur" | `WHERE`'de sağ tabloya koşul koyarsan korunmaz |
| "`ON` ile `WHERE` aynı şeydir" | `INNER JOIN`'de evet, `LEFT JOIN`'de hayır |
| "`WHERE` ve `HAVING` yer değiştirebilir" | `HAVING` gruptan sonra çalışır; gereksiz iş yaptırır |
| "`COUNT(*)` ile `COUNT(kolon)` aynı sayıyı verir" | `COUNT(kolon)` `NULL` olanları saymaz |
| "`SUM` hiç satır yoksa 0 döner" | `NULL` döner; `ISNULL(SUM(...), 0)` yazılır |
| "`NOT IN` ile `NOT EXISTS` aynıdır" | Alt sorguda `NULL` varsa `NOT IN` boş sonuç döndürür |
| "CTE sonucu bir kez hesaplanıp saklanır" | Saklanmaz; her kullanımda yeniden hesaplanabilir |
| "`RANK` ile `DENSE_RANK` aynı şey" | `RANK` eşitlikten sonra atlar, `DENSE_RANK` atlamaz |
| "Window function `WHERE`'de filtrelenebilir" | `SELECT` aşamasında doğar; CTE'ye sarıp dıştan filtrelenir |
| "`UNION` ile `UNION ALL` arasında fark yok" | `UNION` tekrarları ayıklar ve bunun maliyeti vardır |

---


---

## 3. Stored Procedure, View, Transaction

*Kaynak: [`03-Stored-Procedure-View-Transaction.md`](03-Stored-Procedure-View-Transaction.md) · Çarşamba*

- View kaydedilmiş bir sorgudur, veri tutmaz; indexed view tutar ama yazma maliyeti getirir.
- `GROUP BY`, `DISTINCT`, `UNION` içeren view güncellenemez; `WITH CHECK OPTION` olmadan filtre dışına yazma engellenmez.
- Stored procedure üç kanaldan bilgi verir: sonuç kümesi, `OUTPUT` parametresi ve `RETURN` (sadece durum kodu).
- Plan ilk parametre değerine göre üretilir (parameter sniffing); "bazen hızlı bazen yavaş" bunun imzasıdır.
- Injection'ı önleyen şey stored procedure değil **parametredir**; dinamik SQL'de değer `sp_executesql`, yapı beyaz liste.
- Scalar UDF satır başına çalışır ve planda görünmez; `JOIN` ya da inline TVF + `CROSS APPLY` kullan.
- Trigger ifade başına bir kez çalışır, satır başına değil; `inserted`/`deleted` küme olarak işlenmeli.
- ACID: atomik, tutarlı, yalıtılmış, kalıcı. Transaction ya tamamen olur ya hiç olmaz.
- `SET XACT_ABORT ON;` yazmazsan hata sonrası yarım commit mümkündür.
- İç içe transaction yoktur; içteki `COMMIT` sayaç azaltır, içteki `ROLLBACK` her şeyi öldürür.
- Isolation sıkılaştıkça anomali azalır, bekleme artar; `SNAPSHOT` ve `RCSI` bu dengenin dışındadır.
- `NOLOCK` sadece kirli okuma değil, atlanan ve tekrarlanan satır da üretir.
- Deadlock'un birinci sebebi tablolara farklı sıralarda dokunmaktır; çözüm sabit kilit sırası ve kısa transaction.
- `SaveChanges` zaten tek transaction'dır; `BeginTransaction` yalnızca çok adımlı işler ve ham SQL için gerekir.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "View veriyi kendi içinde tutar" | Tutmaz; sorgulandığında asıl tablolara gidilir (indexed view istisna) |
| "Her view güncellenebilir" | `GROUP BY`, `DISTINCT`, `UNION`, toplama varsa güncellenemez |
| "Stored procedure kullanırsam injection'a kapalıyım" | İçinde birleştirilmiş dinamik SQL varsa açıktır; koruyan şey parametredir |
| "`RETURN` ile SP'den veri döndürülür" | `RETURN` sadece `INT` durum kodu taşır; veri `SELECT` veya `OUTPUT` ile döner |
| "`@@IDENTITY` yeni eklenen kaydın Id'sidir" | Trigger varsa yanlış değeri verir; `SCOPE_IDENTITY()` kullan |
| "Aynı SP her zaman aynı hızda çalışır" | Plan ilk parametre değerine göre üretilir; dağılım dengesizse hız değişir |
| "Trigger her satır için bir kez çalışır" | İfade başına bir kez çalışır; `inserted` çok satırlı olabilir |
| "Hata olursa transaction otomatik geri alınır" | `XACT_ABORT OFF` iken çoğu hata sadece o ifadeyi iptal eder |
| "İç içe `BEGIN TRANSACTION` iç içe transaction açar" | Sadece `@@TRANCOUNT` artar; içteki `ROLLBACK` hepsini geri alır |
| "`NOLOCK` sadece biraz eski veri gösterir" | Var olan satırı atlayabilir, aynı satırı iki kez sayabilir |
| "Deadlock ile blocking aynı şeydir" | Blocking ilerler, deadlock ilerlemez; SQL Server birini kurban seçer |
| "`SaveChanges` için transaction açmam gerekir" | `SaveChanges` zaten kendi transaction'ını kullanır |

---


---

## 4. EF Core Temelleri ve Migration

*Kaynak: [`04-EF-Core-Temelleri-ve-Migration.md`](04-EF-Core-Temelleri-ve-Migration.md) · Perşembe*

- ORM, nesne dünyası ile tablo dünyası arasındaki impedance mismatch'i çözer; SQL bilmeni gereksiz kılmaz.
- EF Core dört parçadır: model, provider, change tracker, query pipeline.
- `DbContext` kısa ömürlüdür ve **thread-safe değildir**; paralel iş için ayrı context gerekir.
- `AddDbContext` Scoped kaydeder — ASP.NET Core'da istek başına bir context demektir.
- Code First akışı: entity → `DbContext` → `Add-Migration` → `Update-Database`.
- Migration üç dosya üretir: `Up`/`Down`, designer ve tek bir `ModelSnapshot`.
- EF Core migration üretirken veritabanına bakmaz; modeli snapshot ile karşılaştırır.
- Uygulanan migration'lar `__EFMigrationsHistory` tablosunda tutulur.
- Paylaşılmamış migration silinebilir; paylaşılmış olan silinmez, üstüne yeni migration yazılır.
- Üretimde `dotnet ef migrations script --idempotent` tercih edilir; `Database.Migrate()` risklidir.
- `EnsureCreated()` migration geçmişi tutmaz; `Migrate()` ile birlikte kullanılmaz.
- `HasData` sabit referans verisi içindir; anahtarı elle verilir, dinamik değer kabul etmez.
- Code First'te gerçeğin kaynağı kod, Database First'te veritabanıdır; seçim "kim sahibi" sorusuna bağlıdır.
- Database First'te üretilen dosyaya yazma; `partial class` ile ayrı dosyada genişlet.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "ORM kullanırsam SQL bilmeme gerek kalmaz" | Üretilen SQL'i değerlendirecek olan sensin; SQL daha da gerekir |
| "`DbContext`i singleton yapıp her yerde kullanırım" | Thread-safe değildir ve bellekte kayıt biriktirir; Scoped olmalı |
| "EF Core migration üretirken veritabanına bakar" | Bakmaz; modeli `ModelSnapshot` ile karşılaştırır |
| "`Down` metodu her şeyi geri alır" | Yapıyı geri alır, **veriyi geri getirmez** |
| "Uygulanmış bir migration'ı düzenleyebilirim" | Düzenlenmez; ortamlar sessizce ayrışır. Yeni migration yaz |
| "`Database.Migrate()` üretimde en pratik yol" | Çoklu sunucuda yarışır, yüksek yetki ister, SQL önceden görülmez |
| "`EnsureCreated()` ile başlayıp sonra migration'a geçerim" | Geçemezsin; `EnsureCreated` geçmiş tutmaz |
| "`HasData` ile her türlü veriyi ekleyebilirim" | Anahtar elle verilir, dinamik değer ve navigasyon kullanılamaz |
| "Kolon adını değiştirmek zararsızdır" | EF Core sil+ekle üretebilir; migration'ı açıp `RenameColumn` yapmalısın |
| "Database First eski ve yanlış bir yöntemdir" | Mevcut veritabanı ve DBA'lı ortamlarda doğru seçimdir |
| "Scaffold ile üretilen sınıfa istediğimi yazarım" | Bir sonraki `-Force` ile silinir; `partial` dosya kullan |
| "Migration varsa elle SQL yazmaya gerek kalmaz" | View, SP, veri taşıma için `migrationBuilder.Sql` yazılır |

---


---

## 5. İlişkiler, Fluent API ve Data Annotations

*Kaynak: [`05-Iliskiler-ve-Fluent-API.md`](05-Iliskiler-ve-Fluent-API.md) · Cuma*

- EF Core hiçbir konfigürasyon olmadan da bir şema üretir; kuralları bilmezsen ürettiğini bilemezsin.
- Anahtar `Id` veya `<SınıfAdı>Id` isminden bulunur; bulunamazsa hata alırsın.
- Yabancı anahtar **her zaman** bağımlı (çok) tarafta durur; yazmazsan gölge kolon olarak üretilir.
- Öncelik sırası nettir: convention < Data Annotations < Fluent API. Aynı ayarı iki yerde yazma.
- Data Annotations birleşik anahtar, `OnDelete`, query filter, value converter ve owned entity yapamaz.
- `IEntityTypeConfiguration<T>` + `ApplyConfigurationsFromAssembly`, büyük modelde tek doğru düzendir; sınıfı `public` yapmayı unutma.
- 1-1 ilişkide bağımlı tarafı `HasForeignKey<T>()` ile sen söylemek zorundasın.
- Çoka-çokta ara tabloda fazladan alan varsa otomatik yol biter, açık join entity yazılır.
- Zorunlu ilişkinin varsayılan silme davranışı `Cascade`'tir — finansal tablolarda açıkça `Restrict` yaz.
- SQL Server çoklu cascade yolunu reddeder; zincirlerden birini `Restrict`'e çevirerek kırarsın.
- Owned entity'nin kendi `Id`'si ve `DbSet`'i yoktur; sahibiyle birlikte `Include`'suz gelir.
- TPH varsayılandır ve çoğu zaman doğrudur; bedeli türe özel alanların zorunlu olamamasıdır.
- `nvarchar(max)` kolona index konulamaz — benzersiz olacak metin alanlarına `HasMaxLength` ver.
- `IgnoreQueryFilters()` bütün filtreleri kapatır; çok kiracılı uygulamada bu bir güvenlik açığıdır.
- `[Timestamp]` / `IsRowVersion()` kayıp güncellemeyi yakalar; `rowversion` formda gidip gelmelidir.
- Value converter ile saklanan enum'da sıralama ve büyüklük karşılaştırması güvenilir değildir.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Data Annotations ile Fluent API aynı şeyi yapar" | Fluent API daha geniştir ve çakışmada o kazanır |
| "`[Required]` sadece doğrulama içindir" | Aynı zamanda kolonu `NOT NULL` yapar |
| "Yabancı anahtar asıl tarafta durur" | Her zaman bağımlı (çok) tarafta durur |
| "1-1 ilişkide EF Core tarafları kendi bulur" | Bulamaz; `HasForeignKey<T>()` ile söylemen gerekir |
| "Çoka-çok için mutlaka ara sınıf yazılır" | EF Core 5+ otomatik üretir; ek alan yoksa gerekmez |
| "Silme davranışının varsayılanı yoktur" | Zorunlu ilişkide `Cascade`, isteğe bağlıda `ClientSetNull` |
| "Cascade hatası EF Core hatasıdır" | SQL Server'ın kısıtıdır; çoklu cascade yolunu kabul etmez |
| "`ClientSetNull` veritabanında `SET NULL` kurar" | Kurmaz; davranış sadece EF Core tarafındadır |
| "Owned entity için `Include` yazılmalı" | Sahibiyle birlikte otomatik yüklenir |
| "TPT daha modern, TPH eski yöntemdir" | TPH varsayılandır ve sorgu maliyeti en düşük olandır |
| "TPC'de `Id` otomatik artabilir" | `IDENTITY` kullanılamaz; sequence veya HiLo gerekir |
| "`HasIndex` sadece hız içindir" | `IsUnique()` ile veri bütünlüğü kısıtı da kurar |
| "Query filter `Find()`'a da uygulanır" | Kayıt change tracker'da bulunursa uygulanmaz |
| "`IgnoreQueryFilters()` tek filtreyi kapatır" | O varlıktaki **tüm** filtreleri kapatır |
| "Concurrency token eklemek yeterli" | `DbUpdateConcurrencyException`'ı yakalamazsan istek hata ile düşer |
| "`HasConversion<string>()` sorguyu etkilemez" | Karşılaştırmalar dönüştürülmüş tip üzerinde çalışır |

---


---

## 6. EF Core Performansı

*Kaynak: [`06-EF-Core-Performans.md`](06-EF-Core-Performans.md) · Cumartesi*

- EF Core'da yavaş kod hata vermez; sorunu ancak veri büyüyünce görürsün.
- Performans üç soruya iner: kaç kere gidiyorsun, ne getiriyorsun, getirdiğini takip ediyor musun.
- Ölçmeden optimize etme: önce `ToQueryString()` ve `LogTo` ile üretilen SQL'i gör.
- Okuma amaçlı her sorguda `AsNoTracking()` — güncelleme yapacaksan asla.
- `AsNoTracking()` ile aynı kayıt birden çok nesneye dönüşür; gerekirse `AsNoTrackingWithIdentityResolution()`.
- Lazy loading varsayılan olarak kapalıdır ve öyle kalmalı; sorgu üreten satır kodda görünmez.
- N+1, döngü içinde navigation okumaktan doğar; çözümü `Include`, projeksiyon veya sözlüğe ön yükleme.
- Birden çok koleksiyon `Include` etmek satır sayısını çarpar; `AsSplitQuery()` böler, bedeli çoklu ağ turudur.
- Projeksiyon en ucuz kazançtır: az kolon, otomatik no-tracking, N+1'siz tek sorgu.
- EF Core 3+ çeviremediği `Where` ifadesinde hata verir; `AsEnumerable()` ile sınırı **en sona** koyarsın.
- `ExecuteUpdate` / `ExecuteDelete` (EF Core 7+) change tracker'ı, `SaveChanges` override'ını ve interceptor'ları atlar.
- `SaveChanges` zaten toplu gönderim yapar; on binlerce satır için EF Core yerine `SqlBulkCopy` düşün.
- `EnableSensitiveDataLogging()` parametre değerlerini loga yazar — üretimde asla açma.
- `TagWith` ile etiketlenen sorguyu üretim loglarında saniyeler içinde bulursun.
- Compiled query ve pooling listenin en sonundaki optimizasyonlardır; önce N+1 ve projeksiyonu hallet.
- Sayfalamada `OrderBy` şarttır ve son kriteri benzersiz olmalıdır; derin sayfalarda keyset pagination'a geç.
- Ham SQL bir kaçış kapısıdır; bedeli şema değişiminde derleyicinin sessiz kalmasıdır.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "`AsNoTracking()` her yere serpiştirilir, zararı yok" | Güncelleme yapan sorguda `SaveChanges` sessizce hiçbir şey yapmaz |
| "Tracking sadece bellek tüketir" | `DetectChanges` taraması CPU maliyeti de üretir |
| "`Include` her zaman doğru çözümdür" | Sadece birkaç alan lazımsa projeksiyon daha ucuzdur |
| "Lazy loading kolaylık sağlar" | Sorgu üreten satırı kodda görünmez kılar — N+1'in ana kaynağıdır |
| "N+1 sadece lazy loading ile olur" | Explicit loading ve repository çağrıları da aynı kalıbı üretir |
| "Tek sorgu her zaman iyidir" | Çoklu koleksiyon `Include`'unda satır sayısı çarpılır |
| "`AsSplitQuery()` her zaman hızlandırır" | Küçük veride çoklu ağ turu daha yavaştır |
| "Projeksiyon için `AsNoTracking()` de yazmalıyım" | DTO zaten takip edilmez |
| "`Select` içinde varlık taşırsam takip edilmez" | Edilir — anonim tipin içinde olması fark etmez |
| "Client evaluation EF Core'da artık yok" | Üst seviye `Select` içinde hâlâ var; kalkan `Where` içindeki |
| "`AsEnumerable()` zararsızdır" | Erken yazılırsa tüm tabloyu belleğe çeker |
| "`ExecuteDelete` soft delete mantığımı çalıştırır" | `SaveChanges` override'ı devreye girmez; kayıt gerçekten silinir |
| "`ExecuteUpdate` sonrası bellekteki nesne güncellenir" | Güncellenmez; `Reload()` gerekir |
| "`SaveChanges` her kayıt için ayrı gidiş yapar" | Komutları partiler hâlinde birleştirir |
| "Compiled query ciddi hızlanma getirir" | Çoğu uygulamada fark hissedilmez; önce N+1'i çöz |
| "`Skip`/`Take` sayfalama için yeterlidir" | `OrderBy` olmadan sıra garanti değildir |
| "Derin sayfada `Skip` maliyeti sabittir" | Atlanan satırlar yine de üretilir; doğrusal artar |
| "`FromSqlRaw` ile string birleştirmek sorun değil" | SQL injection açığıdır; enterpolasyon veya parametre kullan |

---


---

## 7. Hafta Özeti: SQL Server ve EF Core

*Kaynak: [`07-Hafta-Ozeti.md`](07-Hafta-Ozeti.md) · Pazar*

- Veritabanı işin temelidir; şema hatası koddan çok daha uzun yaşar.
- PK yapay olsun, doğal anahtar `UNIQUE` ile korunsun.
- FK index'ini kendin yazarsın; SQL Server yaratmaz.
- `Cascade` sadece gerçek sahiplikte; finansal tabloda `Restrict`.
- Para `DECIMAL`, tarih `DATETIME2`, kullanıcı metni `NVARCHAR`.
- `NULL` bilinmiyordur; `NOT IN` yerine `NOT EXISTS`.
- Composite index soldan sağa kullanılır; sütunu fonksiyona sokarsan index ölür.
- Mantıksal sıra `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`.
- `LEFT JOIN`'in koşulu `ON`'a yazılır, `WHERE`'e değil.
- `GROUP BY` satırları yok eder, `OVER` korur.
- Sayfalamada `ORDER BY` ve benzersiz tie-breaker şart; derin sayfada keyset.
- Injection'ı parametre önler, stored procedure değil.
- İç içe transaction yoktur; `SaveChanges` zaten tek transaction'dır.
- Migration veritabanına değil `ModelSnapshot`'a bakar; geçmiş `__EFMigrationsHistory`'de durur.
- Üretimde idempotent script; `Database.Migrate()` riskli.
- Öncelik: convention < Data Annotations < Fluent API. Aynı ayarı iki yerde yazma.
- Ara tabloda veri varsa açık join entity yazılır.
- Okumada `AsNoTracking()`, güncellemede asla.
- N+1 döngü içindeki navigation'dan doğar; en temiz çözüm projeksiyondur. Çoklu koleksiyon `Include` satırları çarpar, `AsSplitQuery()` böler.
- EF Core'u iyi kullanmanın yolu altındaki SQL'i okuyabilmekten geçer.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "FK tanımlarsam index de gelir" | SQL Server FK sütununa index yaratmaz; elle eklenir |
| "`LEFT JOIN` yazdım, sol tablonun tüm satırları gelir" | Sağ tablonun sütununa `WHERE` koyarsan `INNER JOIN`'e döner |
| "`WHERE` ve `HAVING` aynı işi yapar" | `WHERE` satır eler, `HAVING` grup eler; sıraları farklıdır |
| "`NOT IN` ile `NOT EXISTS` aynı" | Listede tek `NULL` varsa `NOT IN` sonucu boşaltır |
| "Stored procedure kullanırsam SQL injection olmaz" | İçinde string birleştiriyorsan açık devam eder; koruyan şey parametredir |
| "İç içe transaction açabilirim" | İç `COMMIT` sayaç azaltır, iç `ROLLBACK` her şeyi geri alır |
| "EF Core migration üretirken veritabanına bakar" | `ModelSnapshot` ile modeli karşılaştırır; veritabanını görmez |
| "`EnsureCreated()` ile başlarım, sonra migration eklerim" | `EnsureCreated()` geçmiş tutmaz; `Migrate()` ile birlikte kullanılmaz |
| "Data Annotations Fluent API'nin kısa hâlidir" | `OnDelete`, query filter, value converter, owned entity yapamaz |
| "`Include` yazmak her zaman iyileştirir" | Çoklu koleksiyonda satırları çarpar; projeksiyon çoğu zaman daha iyidir |
| "`AsNoTracking()` her yerde iyidir" | Güncelleyeceğin sorguda kullanırsan değişiklikler kaydedilmez |
| "EF Core yavaşsa çözüm compiled query'dir" | Önce N+1, projeksiyon ve index; compiled query listenin sonundadır |

---


---

## 8. Dapper'a Giriş

*Kaynak: [`08-Dapper-Giris.md`](08-Dapper-Giris.md) · Ek Not*

- Dapper bir micro-ORM'dir: SQL'i sen yazarsın, o sadece satırları nesneye eşler.
- Kapalı bağlantı verirsen Dapper açıp kapatır; sen açtıysan kapatmak sana düşer.
- `Query` çok satır, `QueryFirstOrDefault` sıfır ya da bir satır, `QuerySingle` tam bir satır, `Execute` etkilenen satır sayısı, `ExecuteScalar` tek hücre.
- Parametre güvenliğinin kaynağı karakter temizleme değil, verinin komuttan ayrı kanalda gitmesidir.
- `IN @Idler` parantezsiz yazılır; tablo adı, sütun adı ve sıralama yönü parametre olamaz.
- Eşleşmeyen sütun sessizce yok sayılır; bu yüzden `SELECT *` yerine sütunları açıkça yaz.
- `splitOn` ikinci nesnenin ilk sütununu gösterir; 1-N ilişkide satırları sözlükle topla.
- EF Core yazmada, Dapper ağır okumada kazanır — bir seçim değil, iş bölümü.
- Birlikte kullanırken `GetDbConnection()` ve `GetDbTransaction()` ile aynı bağlantı ve transaction paylaşılır.
- Dapper'da generic repository kurma; her varlık için ayrı ve açık repository yaz.

---

### Sık karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Dapper, EF Core'un alternatifidir; birini seçersin" | Aynı projede birlikte kullanılırlar; yaygın dağılım EF Core yazma, Dapper ağır okuma |
| "`QueryFirstOrDefault` ile `QuerySingleOrDefault` aynı şey" | `Single` ikinci satırı görünce istisna fırlatır; `First` sessizce ilkini alır |
| "Parametre kullanmak girdideki tehlikeli karakterleri temizler" | Hiçbir şey temizlenmez; veri komut metninin dışında gittiği için yorumlanmaz |
| "`IN (@Idler)` diye yazılır" | Parantezsiz `IN @Idler` yazılır; Dapper listeyi kendisi açar |
| "`ORDER BY @Sutun` ile dinamik sıralama yapılır" | Sütun adı parametre olamaz; beyaz listeyle doğrulayıp SQL'e yerleştirirsin |
| "Sütun adı tutmazsa Dapper hata verir" | Sessizce yok sayar; property varsayılan değerinde kalır |
| "1-N ilişkide Dapper nesneleri kendisi gruplar" | Gruplamaz; her JOIN satırı için callback çalışır, toplamayı sözlükle sen yaparsın |
| "EF Core ile Dapper'ı birlikte kullanınca transaction ortak olur" | `GetDbTransaction()` ile açıkça geçirmezsen Dapper transaction dışında çalışır |

---


---

## Sonraki

Bulanık kalan madde varsa yukarıdaki kaynak satırından dosya adını al ve
sadece o bölümü oku. Haftanın tamamını yeniden okumana gerek yok.

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
