# Hafta 3 — Terimler Sözlüğü: SQL Server ve EF Core

**Ne işe yarar:** Bu dosya baştan sona okunmak için değil, **aranmak** için.
Haftanın 8 notundaki sözlükler burada birleştirildi: **131 terim**.
Aynı terim birden çok notta geçtiyse ilk tanımı alındı; "Nerede" sütunu
terimin ayrıntılı anlatıldığı dosyayı gösterir.

> Ctrl+F ile ara. Bir terimi bulamıyorsan başka haftanın sözlüğünde olabilir.

---

| Terim | Tanım | Nerede |
|---|---|---|
| ACID | Atomicity, Consistency, Isolation, Durability | `03-Stored-Procedure-View-Transaction.md` |
| Aggregate fonksiyon | Çok satırdan tek değer üreten fonksiyon | `02-T-SQL-Sorgulama.md` |
| Alias (takma ad) | `SELECT` aşamasında doğan sütun/tablo adı | `02-T-SQL-Sorgulama.md` |
| Anti-join | "Karşılığı olmayanları bul" kalıbı (`NOT EXISTS`, `LEFT JOIN ... IS NULL`) | `02-T-SQL-Sorgulama.md` |
| `APPLY` | Her dış satır için tablo değerli bir ifadeyi çalıştıran operatör | `02-T-SQL-Sorgulama.md` |
| B-tree | Index'in dengeli ağaç yapısı | `01-Iliskisel-Modelleme-ve-Index.md` |
| Batching | Birden çok komutun tek ağ turunda gönderilmesi | `06-EF-Core-Performans.md` |
| Birincil anahtar (PK) | Satırı benzersiz tanımlayan, değişmeyen sütun | `07-Hafta-Ozeti.md` |
| Cartesian explosion | Çoklu koleksiyon `JOIN`'inde satır sayısının çarpılarak artması | `06-EF-Core-Performans.md` |
| Cascade path | Silmenin zincirleme yayıldığı ilişki yolu | `05-Iliskiler-ve-Fluent-API.md` |
| Change tracker | Çekilen entity'lerin ilk hâlini tutup farkı hesaplayan bileşen | `04-EF-Core-Temelleri-ve-Migration.md` |
| Change tracking | EF Core'un nesnelerdeki değişikliği izleyip UPDATE üretmesi | `08-Dapper-Giris.md` |
| Client-side evaluation | Sorgunun bir kısmının veritabanı yerine bellekte çalışması | `06-EF-Core-Performans.md` |
| Clustered index | Yaprağında satırın kendisini tutan, tablo başına tek index | `01-Iliskisel-Modelleme-ve-Index.md` |
| Code First | Modelin veritabanını tanımladığı yaklaşım | `04-EF-Core-Temelleri-ve-Migration.md` |
| Compiled query | Çevrilmiş sorgunun bir temsilciye bağlanıp saklanması | `06-EF-Core-Performans.md` |
| Concurrency token | Güncellemede "okuduğumdan beri değişti mi" kontrolünde kullanılan kolon | `05-Iliskiler-ve-Fluent-API.md` |
| Convention | EF Core'un açık konfigürasyon yokken uyguladığı varsayılan kural | `05-Iliskiler-ve-Fluent-API.md` |
| Covering index | Sorgunun tüm sütunlarını kapsayan index | `01-Iliskisel-Modelleme-ve-Index.md` |
| CTE | `WITH` ile tanımlanan, adı olan geçici sonuç kümesi | `02-T-SQL-Sorgulama.md` |
| Data Annotations | Sınıf ve özelliklerin üstüne yazılan köşeli parantezli konfigürasyon etiketleri | `05-Iliskiler-ve-Fluent-API.md` |
| Database First | Veritabanının modeli tanımladığı yaklaşım | `04-EF-Core-Temelleri-ve-Migration.md` |
| DbContext pooling | `DbContext` nesnelerinin sıfırlanıp yeniden kullanılması | `06-EF-Core-Performans.md` |
| `DbContext` | Veritabanı oturumu; aynı zamanda Unit of Work | `04-EF-Core-Temelleri-ve-Migration.md` |
| `DbContextPool` | Context örneklerini yeniden kullanan havuz | `04-EF-Core-Temelleri-ve-Migration.md` |
| `DbSet<T>` | Bir entity tipine açılan sorgu ve değişiklik kapısı | `04-EF-Core-Temelleri-ve-Migration.md` |
| Deadlock | İki veya daha çok işlemin birbirinin kilidini beklemesi | `03-Stored-Procedure-View-Transaction.md` |
| `DeleteBehavior` | Asıl kayıt silindiğinde bağımlılara ne olacağını belirleyen enum | `05-Iliskiler-ve-Fluent-API.md` |
| Denormalization | Okuma hızı için kasıtlı veri tekrarı | `01-Iliskisel-Modelleme-ve-Index.md` |
| Dependent (bağımlı) | İlişkide yabancı anahtarı taşıyan taraf | `05-Iliskiler-ve-Fluent-API.md` |
| Dirty read | Commit edilmemiş veriyi okuma | `03-Stored-Procedure-View-Transaction.md` |
| Discriminator | TPH'de satırın hangi alt tipe ait olduğunu söyleyen kolon | `05-Iliskiler-ve-Fluent-API.md` |
| Domain | Bir sütunun alabileceği değerler kümesi (tip + kısıt) | `01-Iliskisel-Modelleme-ve-Index.md` |
| DTO | Katmanlar arasında veri taşımak için yazılan sade sınıf | `06-EF-Core-Performans.md` |
| `DynamicParameters` | Koşullu parametre ekleme ve OUTPUT / ReturnValue alma için Dapper sınıfı | `08-Dapper-Giris.md` |
| Eager loading | İlişkili veriyi ana sorguyla birlikte getirme (`Include`) | `06-EF-Core-Performans.md` |
| `__EFMigrationsHistory` | Uygulanan migration'ların kaydedildiği tablo | `04-EF-Core-Temelleri-ve-Migration.md` |
| `ExecuteUpdate` / `ExecuteDelete` | Belleğe çekmeden doğrudan `UPDATE`/`DELETE` çalıştıran metotlar | `06-EF-Core-Performans.md` |
| Execution plan | Sorgunun nasıl çalıştırılacağının haritası | `03-Stored-Procedure-View-Transaction.md` |
| Explicit loading | İlişkili veriyi sonradan, elle yükleme (`Load`) | `06-EF-Core-Performans.md` |
| Fan-out | `JOIN`'in satırları çoğaltması sonucu toplamların şişmesi | `02-T-SQL-Sorgulama.md` |
| Fluent API | `OnModelCreating` içinde zincirleme metotlarla yapılan konfigürasyon | `05-Iliskiler-ve-Fluent-API.md` |
| Foreign key | Başka tablonun anahtarına işaret eden sütun | `01-Iliskisel-Modelleme-ve-Index.md` |
| Foreign key (yabancı anahtar) | Bağımlı tarafta duran, asıl tarafın anahtarını işaret eden kolon | `05-Iliskiler-ve-Fluent-API.md` |
| Frame (`ROWS`/`RANGE`) | Pencere içinde hangi satırların hesaba katılacağı | `02-T-SQL-Sorgulama.md` |
| `FromSql` | Ham SQL ile varlık döndüren sorgu metodu | `06-EF-Core-Performans.md` |
| `GetDbConnection()` | `DbContext`'in kullandığı bağlantıyı döndüren EF Core metodu | `08-Dapper-Giris.md` |
| `GetDbTransaction()` | EF Core transaction'ının altındaki `DbTransaction`'ı döndüren metot | `08-Dapper-Giris.md` |
| Global query filter | Bir varlığa yapılan tüm sorgulara otomatik eklenen `WHERE` şartı | `05-Iliskiler-ve-Fluent-API.md` |
| `HasData` | Migration'a gömülen sabit veri tanımı | `04-EF-Core-Temelleri-ve-Migration.md` |
| `HAVING` | Gruplama sonrasında grupları eleyen yan tümce | `02-T-SQL-Sorgulama.md` |
| `IDbConnection` | ADO.NET'in bağlantı arayüzü; Dapper tüm metotlarını buna ekler | `08-Dapper-Giris.md` |
| Identity resolution | Aynı anahtarlı satırların tek nesneye eşlenmesi | `06-EF-Core-Performans.md` |
| `IDesignTimeDbContextFactory` | `dotnet ef` komutlarının context üretmesini sağlayan fabrika | `04-EF-Core-Temelleri-ve-Migration.md` |
| `IEntityTypeConfiguration<T>` | Tek bir varlığın konfigürasyonunu taşıyan ayrı sınıf | `05-Iliskiler-ve-Fluent-API.md` |
| Impedance mismatch | Nesne modeli ile ilişkisel modelin yapısal uyumsuzluğu | `04-EF-Core-Temelleri-ve-Migration.md` |
| Implicit conversion | SQL Server'ın sessizce yaptığı tip dönüşümü | `01-Iliskisel-Modelleme-ve-Index.md` |
| Indexed view | Üstünde kümelenmiş indeks olan, sonucu diskte tutulan view | `03-Stored-Procedure-View-Transaction.md` |
| Inner join | Yalnızca eşleşen satırları döndüren birleştirme | `02-T-SQL-Sorgulama.md` |
| Isolation level | Eşzamanlı işlemlerin birbirinin verisini ne kadar gördüğünü belirleyen ayar | `07-Hafta-Ozeti.md` |
| Join entity | Çoka-çok ilişkideki ara tabloyu temsil eden varlık | `05-Iliskiler-ve-Fluent-API.md` |
| Junction table | N-N ilişkiyi kuran ara tablo | `01-Iliskisel-Modelleme-ve-Index.md` |
| Kartezyen çarpım | Koşulsuz birleştirmede her satırın her satırla eşleşmesi | `02-T-SQL-Sorgulama.md` |
| Key lookup | Index'ten sonra sütunlar için tabloya gidilmesi | `01-Iliskisel-Modelleme-ve-Index.md` |
| Keyset pagination | `OFFSET` yerine son görülen anahtardan devam eden sayfalama | `02-T-SQL-Sorgulama.md` |
| Korelasyonlu alt sorgu | Dış sorgunun sütununa referans veren, her satır için çalışan alt sorgu | `02-T-SQL-Sorgulama.md` |
| Lazy loading | Navigation'a dokunulduğunda otomatik yükleme | `06-EF-Core-Performans.md` |
| Logical processing order | SQL yan tümcelerinin değerlendirilme sırası | `02-T-SQL-Sorgulama.md` |
| Logical reads | Sorgunun okuduğu 8 KB'lık sayfa sayısı | `01-Iliskisel-Modelleme-ve-Index.md` |
| Mantıksal işlenme sırası | SQL'in yazılış sırasından farklı olan gerçek değerlendirme sırası | `07-Hafta-Ozeti.md` |
| Materialization | Gelen satırların C# nesnesine dönüştürülmesi | `06-EF-Core-Performans.md` |
| Micro-ORM | ORM'in sadece satır-nesne eşleme kısmını yapan ince kütüphane | `08-Dapper-Giris.md` |
| Migration | Model değişikliğini şema değişikliğine çeviren dosya | `04-EF-Core-Temelleri-ve-Migration.md` |
| ModelSnapshot | EF Core'un mevcut modeli hatırladığı, migration farkını ürettiği dosya | `07-Hafta-Ozeti.md` |
| `ModelSnapshot` | Modelin en güncel hâlini tutan tek dosya | `04-EF-Core-Temelleri-ve-Migration.md` |
| N+1 problemi | Bir ana sorgu artı satır başına bir ek sorgu üreten kalıp | `06-EF-Core-Performans.md` |
| Navigation property | Bir varlıktan ilişkili varlığa giden özellik | `05-Iliskiler-ve-Fluent-API.md` |
| Non-clustered index | Anahtar + işaretçi tutan ek index | `01-Iliskisel-Modelleme-ve-Index.md` |
| Non-repeatable read | Aynı satırı iki kez okuyup farklı değer bulma | `03-Stored-Procedure-View-Transaction.md` |
| Normalizasyon | Veri tekrarını ve güncelleme anomalisini gideren tablo ayrıştırma süreci | `07-Hafta-Ozeti.md` |
| Normalization | Tekrarı ve anormallikleri gidermek için tabloyu bölme | `01-Iliskisel-Modelleme-ve-Index.md` |
| Optimistic concurrency | Kilitlemeden, çakışmayı kayıt anında yakalayan yaklaşım | `05-Iliskiler-ve-Fluent-API.md` |
| ORM | Nesneler ile tabloları eşleyen katman | `04-EF-Core-Temelleri-ve-Migration.md` |
| Outer join | Eşleşmeyen tarafı da koruyan birleştirme (`LEFT`/`RIGHT`/`FULL`) | `02-T-SQL-Sorgulama.md` |
| Owned entity | Kendi kimliği olmayan, sahibinin tablosuna gömülen varlık tipi | `05-Iliskiler-ve-Fluent-API.md` |
| Parameter sniffing | Planın ilk çağrının parametre değerine göre üretilmesi | `03-Stored-Procedure-View-Transaction.md` |
| `partial class` | Aynı sınıfı birden çok dosyaya bölme; scaffold'da özelleştirmenin yolu | `04-EF-Core-Temelleri-ve-Migration.md` |
| `PARTITION BY` | Pencereyi gruplara bölen yan tümce | `02-T-SQL-Sorgulama.md` |
| Phantom read | Aynı şartla iki kez sorup farklı satır kümesi bulma | `03-Stored-Procedure-View-Transaction.md` |
| Primary key | Seçilmiş aday anahtar; benzersiz ve `NOT NULL` | `01-Iliskisel-Modelleme-ve-Index.md` |
| Principal (asıl) | İlişkide anahtarı referans edilen taraf | `05-Iliskiler-ve-Fluent-API.md` |
| Projection | `Select` ile sonucu başka bir şekle dönüştürme | `06-EF-Core-Performans.md` |
| Projeksiyon | Yalnızca gereken sütunları DTO'ya seçme | `07-Hafta-Ozeti.md` |
| Provider | Belirli bir veritabanı için SQL üreten EF Core bileşeni | `04-EF-Core-Temelleri-ve-Migration.md` |
| Proxy | Lazy loading için EF Core'un türettiği ara sınıf | `06-EF-Core-Performans.md` |
| Query pipeline | LINQ ağacını SQL'e çeviren işlem zinciri | `04-EF-Core-Temelleri-ve-Migration.md` |
| Query Store | SQL Server'ın sorgu planlarını ve istatistiklerini tuttuğu yapı | `06-EF-Core-Performans.md` |
| Round trip (ağ turu) | Veritabanına gidip gelen her bir sorgu | `06-EF-Core-Performans.md` |
| `rowversion` | SQL Server'ın her `UPDATE`'te otomatik artırdığı ikili kolon | `05-Iliskiler-ve-Fluent-API.md` |
| SARGability | `WHERE` koşulunun index kullanabilecek biçimde yazılmış olması | `07-Hafta-Ozeti.md` |
| SARGable | Koşulun index seek'e çevrilebilir olması | `01-Iliskisel-Modelleme-ve-Index.md` |
| Savepoint | Transaction'ın bir kısmına dönülmesini sağlayan işaret | `03-Stored-Procedure-View-Transaction.md` |
| `Scaffold-DbContext` | Mevcut veritabanından entity ve context üreten komut | `04-EF-Core-Temelleri-ve-Migration.md` |
| Scalar UDF | Tek değer döndüren kullanıcı fonksiyonu | `03-Stored-Procedure-View-Transaction.md` |
| Scoped ömür | İstek başına bir örnek üretilen DI ömrü | `04-EF-Core-Temelleri-ve-Migration.md` |
| Sequential GUID | Rastgele değil artan sırada üretilen GUID | `01-Iliskisel-Modelleme-ve-Index.md` |
| Set operatörü | İki sonucu alt alta birleştiren operatör (`UNION`, `EXCEPT`) | `02-T-SQL-Sorgulama.md` |
| Shadow property | Sınıfta karşılığı olmayan, sadece modelde tanımlı kolon | `05-Iliskiler-ve-Fluent-API.md` |
| Skaler alt sorgu | Tek bir değer döndüren alt sorgu | `02-T-SQL-Sorgulama.md` |
| SNAPSHOT isolation | Kilit yerine satır sürümü okuyan izolasyon seviyesi | `03-Stored-Procedure-View-Transaction.md` |
| Soft delete | Kaydı silmek yerine bayrakla gizleme | `05-Iliskiler-ve-Fluent-API.md` |
| Split query | Her koleksiyonu ayrı sorguda getiren çalışma biçimi | `06-EF-Core-Performans.md` |
| `splitOn` | Satırın nereden itibaren ikinci nesne sayılacağını söyleyen Dapper parametresi | `08-Dapper-Giris.md` |
| SQL injection | Kullanıcı verisinin SQL cümlesine kod olarak sızması | `03-Stored-Procedure-View-Transaction.md` |
| Stored procedure | Veritabanında saklanan, parametre alan T-SQL yordamı | `03-Stored-Procedure-View-Transaction.md` |
| Surrogate key | Sırf kimliklendirme için üretilmiş anlamsız anahtar | `01-Iliskisel-Modelleme-ve-Index.md` |
| Three-valued logic | `TRUE` / `FALSE` / `UNKNOWN` üçlüsüyle çalışan mantık | `01-Iliskisel-Modelleme-ve-Index.md` |
| TPH / TPT / TPC | Kalıtımın tabloya üç farklı dökülme stratejisi | `05-Iliskiler-ve-Fluent-API.md` |
| Tracking / no-tracking | Sorgu sonucunun takibe alınıp alınmaması | `06-EF-Core-Performans.md` |
| Transaction | Bölünemez şekilde yürütülen ifadeler kümesi | `03-Stored-Procedure-View-Transaction.md` |
| `TransactionScope` | Birden çok kaynağı tek transaction'a alan .NET yapısı | `03-Stored-Procedure-View-Transaction.md` |
| Transitive dependency | Sütunun başka bir anahtar dışı sütuna bağlı olması | `01-Iliskisel-Modelleme-ve-Index.md` |
| Trigger | Veri değişikliğinde otomatik çalışan yordam | `03-Stored-Procedure-View-Transaction.md` |
| `UseTransaction` | Dışarıda başlatılmış bir transaction'ı EF Core'a kullandırma metodu | `08-Dapper-Giris.md` |
| Value converter | C# tipi ile veritabanı tipi arasında iki yönlü dönüşüm tanımı | `05-Iliskiler-ve-Fluent-API.md` |
| Value object | Kimliği değil değeri önemli olan, davranışsız veri nesnesi | `05-Iliskiler-ve-Fluent-API.md` |
| `ValueComparer` | Dönüştürülmüş değerin değişip değişmediğini anlamak için gereken karşılaştırıcı | `05-Iliskiler-ve-Fluent-API.md` |
| View | İsimle saklanan `SELECT` ifadesi; veri tutmaz | `03-Stored-Procedure-View-Transaction.md` |
| Window function | Satırı yok etmeden pencere üzerinde hesap yapan fonksiyon | `02-T-SQL-Sorgulama.md` |
| Yabancı anahtar (FK) | Başka tablonun PK'sına işaret eden, referans bütünlüğünü zorlayan sütun | `07-Hafta-Ozeti.md` |
| Özyinelemeli CTE | Kendine referans veren, anchor + recursive parçadan oluşan CTE | `02-T-SQL-Sorgulama.md` |

---

Haftanın özet ve tuzak listesi için: [`00-Hizli-Tekrar.md`](00-Hizli-Tekrar.md)

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
