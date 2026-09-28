# Hafta 3 · Pazar — Hafta Özeti: SQL Server ve EF Core

**Okuma süresi:** ~32 dk
**Neden bu konu:** Bu hafta altı not okudun ve her biri kendi başına duruyordu. Özet onları tek resme oturtur. Bootcamp'teki 20 projenin neredeyse tamamının altında SQL Server ve EF Core var; oraya gittiğinde sana lazım olacak şey ayrı ayrı tanımlar değil, **aralarındaki bağ** olacak: neden index bilmeden EF Core sorgusu yazılmaz, neden `SaveChanges` aslında bir transaction'dır, neden `Include` yazmak N+1'i çözerken cartesian explosion üretebilir.

---

## Haftanın Tek Cümlesi

Bu haftayı üç katlı bir bina gibi düşün.

**Zemin — veritabanı işin temeli.** Uygulama kodu değişir, framework değişir, hatta dil değişir; veri kalır. Yanlış kurulmuş bir şema seni yıllarca takip eder, çünkü içinde veri birikmiştir ve artık düzeltmenin maliyeti taşımanın maliyetinden yüksektir. Bu yüzden hafta, C# ile değil tabloyla başladı: birincil anahtar, yabancı anahtar, normalizasyon, veri tipi, `NULL`. Bunlar "ileri konu" değil, binanın temeli.

**Birinci kat — SQL, veritabanının ana dili.** Veritabanı senin C#'ını anlamaz. Onunla konuşmanın tek yolu SQL'dir. `SELECT`'in mantıksal işlenme sırası, `JOIN`'in satır çoğaltması, `WHERE` ile `HAVING`'in farkı, window function'lar, transaction ve isolation level — bunların hepsi "ORM kullanacağım, gerekmez" denip atlanan ama atlandığı anda geri dönüp vuran konular. Bir sorgu yavaşladığında ORM sana sebebi söylemez; sebep SQL tarafındadır ve orada okunur.

**İkinci kat — EF Core, bir tercüman.** EF Core senin LINQ ifadeni alır, SQL'e çevirir, gelen satırları nesneye doldurur ve değişiklikleri takip edip geri yazar. Yaptığı iş gerçekten değerlidir: tekrarlayan kodu yok eder, tip güvenliği verir, migration ile şemayı sürüm kontrolüne sokar. Ama bir tercümandır. Ve **tercümanın ne dediğini anlamadan kullanmak pahalıya patlar.** `Include` yazarsın, arkada altı tablonun kartezyen çarpımı gelir. Döngü içinde bir navigation property okursun, bin sorgu gider. `AsNoTracking` yazmazsın, her okuma bellekte kopya tutar. Bunların hiçbiri hata vermez; sadece yavaşlar, sonra veriyle birlikte büyür.

Katların arasındaki bağ tek yönlü değil. Index'in ne olduğunu bildiğin için `YEAR(SiparisTarihi) == 2026` yazmazsın; transaction'ı bildiğin için `SaveChanges`'in neden tek çağrıda toplanması gerektiğini anlarsın; `NOT IN` + `NULL` tuzağını bildiğin için LINQ'te `!Contains` yazarken durursun. SQL bilgisi EF Core'u yavaşlatmaz, güvenilir kılar.

Haftanın tek cümlesi şu: **EF Core'u iyi kullanmanın yolu EF Core'u değil, altındaki veritabanını iyi bilmekten geçer.**

> **Ana benzetme:** Veritabanı, yabancı bir ülkedeki tapu dairesi. SQL o dairenin resmî dili. EF Core ise yanına aldığın tercüman. Tercümanla işin çok daha hızlı yürür — ama memura ne söylediğini anlamadan kâğıdı imzalarsan, dairenin kapısından çıkınca neyi kabul ettiğini öğrenirsin.

---

## Bu Notta Ne Var

1. İlişkisel modelleme ve index
2. T-SQL sorgulama
3. Stored procedure, view, transaction
4. EF Core temelleri ve migration
5. İlişkiler ve Fluent API
6. EF Core performans
7. LINQ'ten SQL'e köprü tablosu
8. Bağlantı haritası

---

## 1. İlişkisel Modelleme ve Index

> **Benzetme —** Şema kurmak apartman projesi çizmektir. Kolonun yerini yanlış koyarsan, bina bittikten sonra düzeltemezsin; içinde insanlar oturuyordur. Index ise binanın zil panosu: her daireye tek tek bakmak yerine isimden kata gidersin. Panoyu güncel tutmanın da bir maliyeti vardır — her taşınmada tabelayı değiştirirsin.

**Basitçe:** Tablolar veriyi tutar, anahtarlar onları birbirine bağlar, kısıtlar yanlış veriyi baştan engeller, index'ler ise aramayı hızlandırır. Dördü de şema kurulurken düşünülür; sonradan eklemek her zaman daha pahalıdır.

**Notun damıtılmışı:**

- **Birincil anahtar yapay olsun.** `IDENTITY` ya da sequential GUID. Doğal anahtarı (TC kimlik, e-posta, ürün kodu) PK yapma — değişebilir; onu `UNIQUE` constraint ile koru.
- **Yabancı anahtar referans bütünlüğünü garanti eder ama index'ini yaratmaz.** SQL Server FK sütununa otomatik index koymaz. `SiparisDetaylari.SiparisId` gibi her FK sütununa elle index yazmak gerekir; yoksa join ve cascade delete tablo taraması yapar.
- **`ON DELETE CASCADE` sadece gerçek sahiplik ilişkisinde meşrudur.** `Siparis → SiparisDetay` sahipliktir, silinebilir. `Kategori → Urun` sahiplik değildir; orada `Restrict` doğrudur.
- **Veri tipi kararı geri dönüşü zordur:** para `DECIMAL` (asla `float`), kullanıcı metni `NVARCHAR`, tarih `DATETIME2`.
- **`NULL` "bilinmiyor" demektir**, sıfır ya da boş string değil. `NULL = NULL` doğru değildir; `IS NULL` yazılır. Ve `NOT IN` + `NULL` sorguyu sessizce boşaltır — yerine `NOT EXISTS`.
- **Clustered index tablo başına bir tanedir** ve satırın kendisini tutar; non-clustered index işaretçi tutar. Bu yüzden non-clustered index'ten sonra key lookup olur, `INCLUDE` de tam olarak onu bitirmek içindir.
- **Composite index soldan sağa kullanılır:** önce eşitlik sütunları, sonra aralık sütunları. `(MusteriId, SiparisTarihi)` index'i `MusteriId` ile başlayan sorguları kapsar, tek başına `SiparisTarihi` sorgusunu kapsamaz.
- **Her index bir yazma maliyetidir.** Düşük seçicilikli sütuna (`Durum` gibi üç değerli), küçük tabloya ve ağır yazılan log tablosuna index koyma.
- **SARGability:** sütunu fonksiyona sokarsan index ölür. Ölçü birimi süre değil, `logical reads`.

```sql
-- PK yapay, doğal anahtar UNIQUE, FK açık, tutar CHECK ile korunuyor
CREATE TABLE Musteriler (
    MusteriId INT IDENTITY(1,1) PRIMARY KEY,
    Eposta    NVARCHAR(256) NOT NULL UNIQUE,     -- doğal anahtar, PK değil
    AdSoyad   NVARCHAR(150) NOT NULL
);

CREATE TABLE Siparisler (
    SiparisId     INT IDENTITY(1,1) PRIMARY KEY,
    MusteriId     INT NOT NULL,
    SiparisTarihi DATETIME2 NOT NULL,
    ToplamTutar   DECIMAL(18,2) NOT NULL CHECK (ToplamTutar >= 0),
    CONSTRAINT FK_Siparis_Musteri FOREIGN KEY (MusteriId)
        REFERENCES Musteriler(MusteriId) ON DELETE NO ACTION
);

-- FK index'i OTOMATİK gelmez, elle yazılır
CREATE INDEX IX_Siparisler_MusteriId_Tarih
    ON Siparisler (MusteriId, SiparisTarihi)
    INCLUDE (ToplamTutar);          -- covering: key lookup biter
```

```sql
-- SARGability: solda fonksiyon = index ölür
-- KÖTÜ
SELECT * FROM Siparisler WHERE YEAR(SiparisTarihi) = 2026;

-- İYİ: aralık yaz, sütuna dokunma
SELECT * FROM Siparisler
WHERE SiparisTarihi >= '2026-01-01' AND SiparisTarihi < '2027-01-01';
```

---

## 2. T-SQL Sorgulama

> **Benzetme —** `SELECT` yazmak, tapu dairesinde memura dilekçe vermek gibi. Dilekçeyi sen yukarıdan aşağı yazarsın ama memur onu kendi sırasıyla okur: önce hangi dosya dolabı (`FROM`), sonra hangi klasörler (`WHERE`), sonra gruplama, en sonda "bana şu sütunları göster" (`SELECT`). Bu sırayı bilmezsen "ama ben yukarıda yazmıştım" dersin, memur "ben oraya daha gelmedim" der.

**Basitçe:** SQL'in yazılış sırası ile çalışma sırası farklıdır. Sorgu yazarken karşılaştığın "alias burada neden çalışmıyor", "`LEFT JOIN` neden `INNER`'a döndü", "toplamlar neden şişti" sorularının tamamı bu tek gerçeğe bağlanır.

**Notun damıtılmışı:**

- **Mantıksal işlenme sırası:** `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`. Alias `SELECT`'te doğar; bu yüzden `WHERE`, `GROUP BY` ve `HAVING`'de kullanılamaz, `ORDER BY`'da kullanılabilir.
- **`LEFT JOIN` tuzağı:** sağ tablonun sütununa `WHERE` koşulu koyarsan sorgu sessizce `INNER JOIN`'e döner. Koşul `ON`'a taşınmalıdır.
- **`JOIN` satır çoğaltır.** İki detay tablosunu aynı sorguda `SUM`'larsan fan-out hatası alırsın; her toplamı kendi alt sorgusunda hesapla.
- **`WHERE` satır eler, `HAVING` grup eler.** Satır seviyesinde elenebilecek bir koşulu `HAVING`'e bırakmak gereksiz iş yaptırır.
- **Toplama fonksiyonları `NULL`'u atlar:** `COUNT(*)` satır, `COUNT(kolon)` `NULL` olmayanları sayar. Hiç satır yoksa `COUNT` 0, `SUM` `NULL` döner.
- **Varlık kontrolünde `EXISTS`, yoklukta `NOT EXISTS`.** `NOT IN` + alt sorgu, listede tek bir `NULL` varsa sonucu boşaltır.
- **`GROUP BY` satırları yok eder, `OVER` satırları korur.** `RANK` eşitlikten sonra atlar, `DENSE_RANK` atlamaz, `ROW_NUMBER` eşitliği umursamaz. Window function `WHERE`'de kullanılamaz — CTE'ye sarıp dıştan filtrelenir.
- **Sayfalamada `ORDER BY` ve benzersiz tie-breaker şarttır;** derin sayfalarda `OFFSET` yerine keyset. Tekrar beklemiyorsan `UNION ALL` yaz; `MERGE` güçlü ama riskli, ayrı `UPDATE` + `INSERT` çoğu zaman daha güvenli.

```sql
-- LEFT JOIN tuzağı
-- KÖTÜ: hiç siparişi olmayan müşteri kayboldu, LEFT anlamsızlaştı
SELECT m.AdSoyad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId
WHERE s.SiparisTarihi >= '2026-01-01';

-- İYİ: koşul ON'a taşındı, siparişsiz müşteri NULL ile geliyor
SELECT m.AdSoyad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s
       ON s.MusteriId = m.MusteriId
      AND s.SiparisTarihi >= '2026-01-01';
```

```sql
-- Window function: gruplamadan sıralama. Müşteri başına en son 3 sipariş.
WITH Sirali AS (
    SELECT s.SiparisId, s.MusteriId, s.SiparisTarihi, s.ToplamTutar,
           ROW_NUMBER() OVER (PARTITION BY s.MusteriId
                              ORDER BY s.SiparisTarihi DESC, s.SiparisId DESC) AS Sira
    FROM Siparisler s
)
SELECT * FROM Sirali WHERE Sira <= 3;   -- window WHERE'de kullanılamaz, dıştan filtrelenir
```

---
## 3. Stored Procedure, View, Transaction

> **Benzetme —** View, muhtarlıktaki hazır dilekçe örneği: her seferinde baştan yazmazsın, ama içi doldurulunca yine aynı işlem yapılır. Stored procedure, kurumun kendi memurunun yaptığı işlem — sen sadece "şu numarayı işle" dersin. Transaction ise banka havalesi: ya para bir hesaptan çıkıp diğerine girer, ya da hiçbir şey olmaz. Arada takılı kalması diye bir seçenek yoktur.

**Basitçe:** View kaydedilmiş bir sorgudur, stored procedure kaydedilmiş bir iştir, transaction ise birden çok işlemi tek bir "ya hep ya hiç" paketine sarar. Üçü de veritabanının kendi tarafında yaşar; EF Core bunların yerini almaz, üstüne biner.

**Notun damıtılmışı:**

- **View veri tutmaz**, kaydedilmiş sorgudur. Indexed view tutar ama her yazmada güncellenir. `GROUP BY`, `DISTINCT`, `UNION` içeren view güncellenemez; `WITH CHECK OPTION` olmadan filtre dışına yazma engellenmez.
- **Stored procedure üç kanaldan bilgi verir:** sonuç kümesi, `OUTPUT` parametresi ve `RETURN`. `RETURN` sadece durum kodudur, veri taşımaz.
- **Parameter sniffing:** plan ilk çağrının parametre değerine göre üretilip cache'lenir; "bazen hızlı bazen yavaş" şikâyetinin imzası budur.
- **SQL injection'ı önleyen şey stored procedure değil parametredir.** SP içinde string birleştirirsen açık yine açıktır. Dinamik SQL'de değer `sp_executesql` parametresiyle, tablo/kolon adı gibi **yapı** ise beyaz liste ile gelir.
- **ACID:** atomik, tutarlı, yalıtılmış, kalıcı. `SET XACT_ABORT ON;` yazmazsan hata sonrası yarım commit mümkündür.
- **İç içe transaction yoktur.** İçteki `COMMIT` sayaç azaltır, içteki `ROLLBACK` her şeyi geri alır; gerçek iç içelik için savepoint.
- **Isolation sıkılaştıkça anomali azalır, bekleme artar;** `SNAPSHOT`/RCSI bu dengenin dışındadır. `NOLOCK` sadece kirli okuma değil, atlanan ve tekrarlanan satır da üretir. Deadlock'un birinci sebebi tablolara farklı sıralarda dokunmaktır; çözüm sabit kilit sırası ve kısa transaction.
- **`SaveChanges` zaten tek transaction'dır.** `BeginTransaction` yalnızca çok adımlı işler ve araya giren ham SQL için gerekir.

```sql
-- Stored procedure + transaction + hata yönetimi
CREATE OR ALTER PROCEDURE dbo.SiparisOlustur
    @MusteriId INT, @UrunId INT, @Adet INT, @YeniSiparisId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;              -- hata olursa yarım commit kalmasın
    BEGIN TRY
        BEGIN TRANSACTION;

        DECLARE @Fiyat DECIMAL(18,2) = (SELECT BirimFiyat FROM Urunler WHERE UrunId = @UrunId);

        INSERT INTO Siparisler (MusteriId, SiparisTarihi, ToplamTutar)
        VALUES (@MusteriId, SYSUTCDATETIME(), @Fiyat * @Adet);
        SET @YeniSiparisId = SCOPE_IDENTITY();

        INSERT INTO SiparisDetaylari (SiparisId, UrunId, Adet, BirimFiyat)
        VALUES (@YeniSiparisId, @UrunId, @Adet, @Fiyat);

        UPDATE Urunler SET StokAdedi = StokAdedi - @Adet WHERE UrunId = @UrunId;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
        THROW;                      -- orijinal hatayı olduğu gibi yukarı at
    END CATCH
END
```

```csharp
// EF Core tarafı: çok adımlı iş için açık transaction
await using var tx = await db.Database.BeginTransactionAsync();
try
{
    db.Siparisler.Add(siparis);
    await db.SaveChangesAsync();                      // 1. adım
    await db.Database.ExecuteSqlAsync(               // 2. adım: ham SQL
        $"UPDATE Urunler SET StokAdedi = StokAdedi - {adet} WHERE UrunId = {urunId}");
    await tx.CommitAsync();
}
catch { await tx.RollbackAsync(); throw; }
```

> Tek bir `SaveChanges` için `BeginTransaction` yazmana gerek yok — EF Core o çağrıyı zaten tek transaction içinde gönderir.

---

## 4. EF Core Temelleri ve Migration

> **Benzetme —** Migration, tapu dairesindeki işlem defteri. Binayı her değiştirdiğinde deftere bir satır düşersin: "3. kata oda eklendi". Defter sayesinde herkes aynı binayı görür ve yeni gelen sıfırdan değil, satırları sırayla uygulayarak bugüne gelir. Defteri kimse geriye dönüp silmez; yanlış yazdıysan üstüne düzeltme satırı yazarsın.

**Basitçe:** EF Core, nesne dünyası ile tablo dünyası arasındaki çeviriyi yapar. `DbContext` bu çevirinin oturumudur; migration ise şema değişikliklerini koddan veritabanına taşıyan sürüm defteridir.

**Notun damıtılmışı:**

- **EF Core dört parçadır:** model (metadata), provider (SQL Server, PostgreSQL…), change tracker, query pipeline.
- **`DbContext` kısa ömürlüdür ve thread-safe değildir.** Paralel iş için ayrı context gerekir; `AddDbContext` Scoped kaydeder, yani ASP.NET Core'da istek başına bir context.
- **Migration üç dosya üretir:** `Up`/`Down` içeren migration, designer dosyası ve tek bir `ModelSnapshot`. EF Core migration üretirken **veritabanına bakmaz**; modeli snapshot ile karşılaştırır. "Veritabanını elle değiştirdim, migration görmedi" şikâyetinin sebebi budur.
- **Uygulanan migration'lar `__EFMigrationsHistory` tablosunda tutulur.** Paylaşılmamış migration silinebilir (`Remove-Migration`); paylaşılmış olan silinmez, üstüne yeni migration yazılır.
- **Üretimde `dotnet ef migrations script --idempotent`** tercih edilir. `Database.Migrate()` uygulama açılışında çalışır ve çok örnekli dağıtımda yarış üretir.
- **`EnsureCreated()` migration geçmişi tutmaz** ve `Migrate()` ile birlikte kullanılmaz. İkisini karıştırmak klasik bir çıkmazdır.
- **Code First'te gerçeğin kaynağı kod, Database First'te veritabanıdır.** Seçim teknik değil sahiplik sorusudur: şemayı kim yönetiyor? Database First'te üretilen dosyaya yazma, `partial class` ile genişlet.

```csharp
// DbContext: DbSet'ler + konfigürasyon
public class MagazaDbContext(DbContextOptions<MagazaDbContext> options) : DbContext(options)
{
    public DbSet<Musteri> Musteriler => Set<Musteri>();
    public DbSet<Siparis> Siparisler => Set<Siparis>();
    public DbSet<SiparisDetay> SiparisDetaylari => Set<SiparisDetay>();
    public DbSet<Urun> Urunler => Set<Urun>();
    public DbSet<Kategori> Kategoriler => Set<Kategori>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.ApplyConfigurationsFromAssembly(typeof(MagazaDbContext).Assembly);

        mb.Entity<Kategori>().HasData(          // sabit referans verisi
            new Kategori { KategoriId = 1, Ad = "Elektronik" },
            new Kategori { KategoriId = 2, Ad = "Kitap" });
    }
}
```

```
# Migration üretme ve üretime alma
dotnet ef migrations add SiparisDetayEklendi
dotnet ef migrations script --idempotent -o deploy/sema.sql
```

> `Database.Migrate()` uygulama başlangıcına koymak kolaydır ama üretimde birden çok örnek aynı anda açılırsa aynı migration'ı birlikte uygulamaya çalışır. Script yolu bu yüzden tercih edilir.

---
## 5. İlişkiler ve Fluent API

> **Benzetme —** Terziye gidersin, ölçü vermezsen sana standart beden diker. Standart beden de bir şeydir — üstüne olur, ama kolu uzun gelir. Convention standart bedendir: hiçbir şey yazmazsan EF Core yine bir şema üretir. Fluent API ise ölçü vermektir; terzinin neyi varsayacağını bilmezsen ölçü vermen gerektiğini de bilemezsin.

**Basitçe:** EF Core, entity sınıflarına bakıp bir şema tahmin eder. Bu tahminin kuralları nettir ve öğrenilebilir. Tahmin yetmediğinde Data Annotations ya da Fluent API ile devreye girersin; Fluent API her şeyi yapabilir, Data Annotations yapamaz.

**Notun damıtılmışı:**

- **Yabancı anahtar her zaman bağımlı (çok) tarafta durur.** Yazmazsan EF Core gölge kolon olarak üretir — veritabanında vardır, C# sınıfında yoktur.
- **Öncelik sırası nettir:** convention < Data Annotations < Fluent API. Aynı ayarı iki yerde yazma.
- **`IEntityTypeConfiguration<T>` + `ApplyConfigurationsFromAssembly`** büyük modelde tek doğru düzendir; konfigürasyon sınıfını `public` yapmayı unutma, yoksa taranmaz.
- **1-1 ilişkide bağımlı tarafı sen söylemek zorundasın:** `HasForeignKey<T>()`. EF Core hangi tarafın bağımlı olduğunu tahmin edemez.
- **Çoka-çokta ara tabloda fazladan alan varsa** (`Adet`, `BirimFiyat`) otomatik yol biter; açık join entity yazılır. `SiparisDetay` tam olarak budur — `Siparis` ile `Urun` arasındaki N-N ilişkinin veri taşıyan hâli.
- **Zorunlu ilişkinin varsayılan silme davranışı `Cascade`'tir.** Finansal tablolarda bunu açıkça `Restrict` yaz. SQL Server çoklu cascade yolunu reddeder; zincirlerden birini `Restrict`'e çevirerek kırarsın.
- **`IgnoreQueryFilters()` bütün filtreleri kapatır.** Çok kiracılı uygulamada bu bir güvenlik açığıdır, performans ayarı değil.
- **`[Timestamp]` / `IsRowVersion()` kayıp güncellemeyi yakalar;** `rowversion` değerinin formda gidip gelmesi şarttır. Value converter ile saklanan enum'da sıralama ve büyüklük karşılaştırması güvenilir değildir.

```csharp
// Fluent API: ayrı konfigürasyon sınıfı (public olmalı)
public class SiparisConfig : IEntityTypeConfiguration<Siparis>
{
    public void Configure(EntityTypeBuilder<Siparis> b)
    {
        b.ToTable("Siparisler");
        b.HasKey(s => s.SiparisId);
        b.Property(s => s.ToplamTutar).HasPrecision(18, 2);

        b.HasOne(s => s.Musteri)
         .WithMany(m => m.Siparisler)
         .HasForeignKey(s => s.MusteriId)
         .OnDelete(DeleteBehavior.Restrict);      // müşteri silinince sipariş uçmasın

        b.HasMany(s => s.Detaylar)
         .WithOne(d => d.Siparis)
         .HasForeignKey(d => d.SiparisId)
         .OnDelete(DeleteBehavior.Cascade);       // gerçek sahiplik: detay siparişe aittir

        b.HasIndex(s => new { s.MusteriId, s.SiparisTarihi });
        b.Property(s => s.Version).IsRowVersion();
    }
}
```

```csharp
// N-N + veri: açık join entity. SiparisDetaylari bir ara tablo DEĞİL, bir varlık.
public class SiparisDetay
{
    public int SiparisId { get; set; }
    public Siparis Siparis { get; set; } = null!;
    public int UrunId { get; set; }
    public Urun Urun { get; set; } = null!;
    public int Adet { get; set; }                 // ara tabloda veri var → otomatik yol biter
    public decimal BirimFiyat { get; set; }
}

// Birleşik anahtar: Data Annotations yapamaz, Fluent API yapar
b.HasKey(d => new { d.SiparisId, d.UrunId });
```

---

## 6. EF Core Performans

> **Benzetme —** Markete gidip listendeki her ürün için ayrı ayrı çıkıp girmek gibi. Yirmi ürün, yirmi sefer. N+1 problemi tam olarak budur. `Include` ise "hepsini tek seferde al" demektir — ama bu sefer kasada iki ayrı reyonun ürünlerini çaprazlamalı olarak fişe basarlar; cartesian explosion da odur.

**Basitçe:** EF Core'da yavaş kod hata vermez. Derlenir, testi geçer, on kayıtla çalışır. Sorunu ancak veri büyüyünce görürsün — ve o zaman düzeltmek en pahalı andır.

**Notun damıtılmışı:**

- **Performans üç soruya iner:** kaç kere veritabanına gidiyorsun, ne getiriyorsun, getirdiğini takip ediyor musun.
- **Okuma amaçlı her sorguda `AsNoTracking()`;** güncelleme yapacaksan asla. `AsNoTracking()` ile aynı kayıt birden çok nesneye dönüşebilir — gerekirse `AsNoTrackingWithIdentityResolution()`.
- **N+1 döngü içinde navigation okumaktan doğar.** Çözüm: `Include`, projeksiyon ya da sözlüğe ön yükleme. Lazy loading varsayılan olarak kapalıdır ve öyle kalmalı — sorgu üreten satır kodda görünmediği için N+1'in en sinsi kaynağıdır.
- **Birden çok koleksiyon `Include` etmek satır sayısını çarpar.** `AsSplitQuery()` böler; bedeli çoklu ağ turu ve tek transaction garantisinin kaybıdır.
- **Projeksiyon en ucuz kazançtır:** az kolon, otomatik no-tracking, `Include`siz tek sorgu.
- **`ExecuteUpdate` / `ExecuteDelete` (EF Core 7+)** tek SQL ifadesi üretir ama change tracker'ı, `SaveChanges` override'ını ve interceptor'ları atlar.
- **`EnableSensitiveDataLogging()` üretimde asla açılmaz;** `TagWith` etiketi sorguyu loglarda saniyeler içinde buldurur.
- **Compiled query ve DbContext pooling listenin en sonundadır.** Önce N+1 ve projeksiyonu hallet. Derin sayfalarda `OFFSET` yerine keyset pagination.

```csharp
// N+1: döngü içinde navigation — 1 + N sorgu
var siparisler = await db.Siparisler.ToListAsync();
foreach (var s in siparisler)
    Console.WriteLine(s.Musteri.AdSoyad);        // her tur ayrı SELECT

// Çözüm 1: Include
var ile = await db.Siparisler.Include(s => s.Musteri).AsNoTracking().ToListAsync();

// Çözüm 2 (daha iyi): projeksiyon — sadece gereken kolonlar, tek sorgu, takip yok
var ozet = await db.Siparisler
    .Where(s => s.SiparisTarihi >= baslangic)
    .Select(s => new SiparisOzetDto(
        s.SiparisId,
        s.Musteri.AdSoyad,
        s.SiparisTarihi,
        s.Detaylar.Sum(d => d.Adet * d.BirimFiyat)))
    .ToListAsync();
```

```csharp
// Cartesian explosion: iki koleksiyon Include = satırlar çarpılır
var patlak = db.Siparisler
    .Include(s => s.Detaylar).Include(s => s.Odemeler);   // 10 detay x 3 ödeme = 30 satır

var bolunmus = db.Siparisler
    .Include(s => s.Detaylar).Include(s => s.Odemeler)
    .AsSplitQuery();                                      // üç ayrı sorgu, çarpım yok
```

```csharp
// Toplu güncelleme: change tracker'a uğramadan tek UPDATE (EF Core 7+)
await db.Urunler.Where(u => u.KategoriId == 2)
    .ExecuteUpdateAsync(set => set.SetProperty(u => u.BirimFiyat, u => u.BirimFiyat * 1.10m));

var sql = sorgu.ToQueryString();                              // üretilen SQL
var etiketli = db.Siparisler.TagWith("Panel > Aylık ciro");   // logda bulmak için
```

---
## 7. LINQ'ten SQL'e

Bu hafta iki dil öğrendin ve ikisi aynı şeyi söylüyor. Soldaki LINQ'i yazdığında EF Core sağdakini üretir; bir sorgu yavaşladığında ilk işin sol sütundan sağ sütuna geçip SQL'i okumaktır.

| İş | LINQ | Üretilen T-SQL |
|---|---|---|
| Filtre | `.Where(s => s.MusteriId == 5)` | `WHERE MusteriId = 5` |
| Inner join (navigation) | `.Include(s => s.Musteri)` | `INNER JOIN Musteriler m ON m.MusteriId = s.MusteriId` (zorunlu ilişki) |
| Left join (opsiyonel ilişki) | `.Include(u => u.Kategori)` (nullable FK) | `LEFT JOIN Kategoriler k ON k.KategoriId = u.KategoriId` |
| Gruplama | `.GroupBy(s => s.MusteriId).Select(g => new { g.Key, Ciro = g.Sum(x => x.ToplamTutar) })` | `SELECT MusteriId, SUM(ToplamTutar) ... GROUP BY MusteriId` |
| Grup filtresi | `.GroupBy(...).Where(g => g.Count() > 3)` | `... GROUP BY ... HAVING COUNT(*) > 3` |
| Sayfalama | `.OrderBy(s => s.SiparisTarihi).ThenBy(s => s.SiparisId).Skip(20).Take(10)` | `ORDER BY ... OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` |
| Varlık kontrolü | `.Where(m => m.Siparisler.Any())` | `WHERE EXISTS (SELECT 1 FROM Siparisler ...)` |
| Yokluk kontrolü | `.Where(m => !m.Siparisler.Any())` | `WHERE NOT EXISTS (SELECT 1 FROM Siparisler ...)` |
| Liste içinde | `.Where(u => idler.Contains(u.UrunId))` | `WHERE UrunId IN (...)` |
| Sıralama | `.OrderByDescending(s => s.ToplamTutar).ThenBy(s => s.SiparisId)` | `ORDER BY ToplamTutar DESC, SiparisId` |
| Projeksiyon | `.Select(s => new Dto(s.SiparisId, s.Musteri.AdSoyad))` | `SELECT s.SiparisId, m.AdSoyad FROM ... JOIN ...` |

Dikkat edilecek iki yer:

- **`!Contains` ile `NOT IN`:** T-SQL'deki `NULL` tuzağı burada da geçerlidir. Nullable bir alanda yokluk kontrolü yapıyorsan `!Any()` (yani `NOT EXISTS`) daha güvenlidir.
- **`Skip`/`Take` `OrderBy` olmadan** anlamsızdır ve EF Core uyarı üretir; son sıralama kriteri benzersiz olmalı. `Contains` ile üretilen `IN` listesi büyüdükçe plan cache şişer.

---

## 8. Bağlantı Haritası

Bu haftanın konuları birbirinden bağımsız değil. Bağlar şöyle:

| Bu konu | Şuna açılıyor |
|---|---|
| Index tasarımı | SARGability → EF Core sorgu performansı (Not 6) |
| `NOT IN` + `NULL` tuzağı | LINQ'te `!Contains` yerine `!Any()` (Not 7) |
| `LEFT JOIN` + `WHERE` tuzağı | Opsiyonel navigation `Include`'u (Not 5) |
| Transaction ve ACID | `SaveChanges` tek transaction'dır (Not 3, Not 4) |
| Isolation level, deadlock | Uzun tutulan `DbContext` → kilit süresi (Not 4) |
| Concurrency token / `rowversion` | Kayıp güncelleme → `DbUpdateConcurrencyException` (Not 5) |
| İlişkiler ve navigation property | `Include` → N+1 ve cartesian explosion (Not 6) |
| `OnDelete` davranışı | FK constraint ve cascade zinciri (Not 1) |
| Fluent API konfigürasyonu | Migration'ın ürettiği DDL (Not 4) |
| Deferred execution | `IQueryable` ile `IEnumerable` sınırı (Not 6) |
| Stored procedure ve view | `FromSql`, keyless entity (Not 3) |
| Sayfalama (`OFFSET/FETCH`) | Keyset pagination → derin sayfa performansı (Not 2, Not 6) |

**Hafta 1'e geri bakış:** `Hafta-01/03-LINQ.md`'deki deferred execution ve `IQueryable` / `IEnumerable` ayrımı bu haftanın altyapısıydı. EF Core'un LINQ'i SQL'e çevirebilmesinin sebebi, `IQueryable`'ın lambda'yı derlenmiş kod olarak değil **expression tree** olarak taşımasıdır. `AsEnumerable()` yazdığın an bu ağaç kapanır, gerisi bellekte çalışır — Not 6'daki client-side evaluation tuzağı oradan gelir.

**Hafta 2'ye geri bakış:** `Hafta-02/04-SOLID-ISP-DIP-ve-DI.md`'deki DIP ve repository tartışması artık somutlaştı. `DbContext` zaten bir Unit of Work, `DbSet<T>` zaten bir Repository. Üstüne bir katman daha koymalı mısın? Cevap DIP'ten gelir: küçük projede hayır, gereksiz dolaylılık; çok sağlayıcılı ya da ağır test edilen projede evet — ama arayüz **tüketenin** katmanında durur ve `IQueryable` döndürmez, döndürürse soyutlama sızar.

**Hafta 4'e köprü:** `Hafta-04/04-EF-Core-CRUD-ve-Change-Tracking.md`'de bu haftanın bilgisi MVC controller'ının içine girer: change tracker durumları (`Added`, `Modified`, `Unchanged`, `Deleted`), disconnected senaryoda `Update` ile `Attach` farkı, form'dan gelen veriyle güncellemenin doğru yolu. Bu haftada "tracking nedir", orada "tracking ile nasıl CRUD yazılır" sorusunu cevaplayacaksın.

---

## Kendini Yoklama

Aşağıdaki soruları nota bakmadan, yüksek sesle cevaplamayı dene. Takıldığın maddenin notuna dön.

1. Doğal anahtarı neden birincil anahtar yapmıyoruz? Onu nasıl koruyoruz?
2. Yabancı anahtar tanımlamak index yaratır mı? Yaratmıyorsa ne yapman gerekir?
3. `Siparis → SiparisDetay` ile `Kategori → Urun` ilişkilerinde silme davranışı neden farklı olmalı?
4. `NULL = NULL` neden `TRUE` değil? Bu davranış hangi sorgu kalıbını sessizce bozar?
5. Composite index `(MusteriId, SiparisTarihi)` hangi sorguları kapsar, hangilerini kapsamaz?
6. SARGable olmayan bir `WHERE` örneği yaz ve SARGable hâline çevir.
7. `SELECT`'in mantıksal işlenme sırası nedir? Alias'ı neden `WHERE`'de kullanamazsın?
8. `LEFT JOIN` yazdığın bir sorgu neden `INNER JOIN` gibi davranmaya başlar?
9. `WHERE` ile `HAVING` arasındaki fark nedir? Hangisini önce kullanmalısın?
10. `RANK`, `DENSE_RANK` ve `ROW_NUMBER` farkını bir eşitlik senaryosuyla anlat.
11. Window function'ı neden doğrudan `WHERE`'de kullanamazsın? Nasıl filtrelersin?
12. SQL injection'ı gerçekte ne önler — stored procedure mü, parametre mi?
13. Parameter sniffing nedir, hangi şikâyet olarak kendini gösterir?
14. İç içe transaction gerçekten iç içe midir? İçteki `ROLLBACK` ne yapar?
15. EF Core migration üretirken veritabanına bakar mı? Bakmıyorsa neye bakar?
16. `EnsureCreated()` ile `Migrate()` neden birlikte kullanılmaz?
17. Yabancı anahtar hangi tarafta durur? Yazmazsan EF Core ne yapar?
18. `SiparisDetaylari` neden otomatik ara tablo olamaz?
19. N+1 nasıl doğar? Üç farklı çözümünü say.
20. Bir sorgu üretimde yavaşladı. İlk üç adımın ne olur?

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Birincil anahtar (PK) | Satırı benzersiz tanımlayan, değişmeyen sütun |
| Yabancı anahtar (FK) | Başka tablonun PK'sına işaret eden, referans bütünlüğünü zorlayan sütun |
| Normalizasyon | Veri tekrarını ve güncelleme anomalisini gideren tablo ayrıştırma süreci |
| Clustered index | Satırın kendisini sıralı tutan, tablo başına tek olan index |
| Covering index | Sorgunun ihtiyacı olan tüm sütunları taşıyan, key lookup gerektirmeyen index |
| SARGability | `WHERE` koşulunun index kullanabilecek biçimde yazılmış olması |
| Mantıksal işlenme sırası | SQL'in yazılış sırasından farklı olan gerçek değerlendirme sırası |
| CTE | `WITH` ile tanımlanan, okunaklılık sağlayan adlandırılmış geçici sonuç kümesi |
| Window function | Satırları yok etmeden grup üzerinden hesap yapan `OVER` fonksiyonu |
| Keyset pagination | `OFFSET` yerine son anahtardan devam ederek yapılan sayfalama |
| View | Kaydedilmiş sorgu; veri tutmaz (indexed view hariç) |
| Stored procedure | Veritabanında saklanan, parametre alan iş birimi |
| Parameter sniffing | Planın ilk çağrının parametre değerine göre üretilip cache'lenmesi |
| ACID | Atomiklik, tutarlılık, yalıtım, kalıcılık |
| Isolation level | Eşzamanlı işlemlerin birbirinin verisini ne kadar gördüğünü belirleyen ayar |
| Deadlock | İki işlemin birbirinin kilidini karşılıklı beklemesi |
| ORM | Nesne dünyası ile ilişkisel dünya arasında eşleme yapan katman |
| Change tracker | EF Core'un yüklenen nesnelerin durumunu izleyen bileşeni |
| Migration | Model değişikliğini veritabanı şemasına taşıyan sürümlü betik |
| ModelSnapshot | EF Core'un mevcut modeli hatırladığı, migration farkını ürettiği dosya |
| Concurrency token | Kayıp güncellemeyi yakalayan sürüm alanı (`rowversion`) |
| N+1 problemi | Bir liste sorgusunun ardından her satır için ayrı sorgu atılması |
| Cartesian explosion | Birden çok koleksiyon `Include`'unda satır sayısının çarpılması |
| Projeksiyon | Yalnızca gereken sütunları DTO'ya seçme |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `08-Dapper-Giris.md` (ek not) — EF Core'un tercüman olduğu yerde Dapper "SQL'i ben yazarım, sen sadece nesneye doldur" der. Ne zaman hangisinin doğru olduğunu görmek için kısa bir bakış.

→ `../Hafta-04-AspNetCore-MVC/01-MVC-ve-Razor.md` (Hafta 4 · Pazartesi)

Hafta 4'te bu haftanın veri katmanı bir web uygulamasının içine girer: MVC, Razor, routing, model binding ve change tracking ile CRUD.
