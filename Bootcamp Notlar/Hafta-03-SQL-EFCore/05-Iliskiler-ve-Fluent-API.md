# Hafta 3 · Cuma — İlişkiler, Fluent API ve Data Annotations

**Okuma süresi:** ~75 dk
**Neden bu konu:** EF Core'un ürettiği veritabanı şeması, senin C# sınıflarından çıkarılır. Bu çıkarımın kurallarını bilmezsen, beklediğinden farklı bir tablo alırsın ve farkı ancak üretimde görürsün. Bootcamp'te Code First ile model kuracaksın; MvcCv'de Database First kullandığın için bu taraf sana yeni. İlişkiler, silme davranışları ve eşzamanlılık kontrolü tam da burada tanımlanır.

---

## Önce Basitçe

Elinde birkaç C# sınıfı var: `Musteri`, `Siparis`, `Urun`. Veritabanında ise tablolar, kolonlar, anahtarlar ve kısıtlar olacak. Birinden diğerine geçişi biri yapmak zorunda. EF Core bu işi üstlenir ve sen hiçbir şey söylemesen bile bir şema üretir.

Hiçbir şey söylemediğinde EF Core tahmin yürütür. `Id` adlı bir özellik görürse "bu birincil anahtardır" der. `Siparis` sınıfı içinde `Musteri` tipinde bir özellik görürse "demek ki siparişin bir müşterisi var, aralarında ilişki kuralım" der. Tablonun adını `DbSet` özelliğinin adından alır. Bu tahminlere **convention (varsayılan kural)** denir ve şaşırtıcı derecede iyi çalışır. Basit bir modelde tek satır konfigürasyon yazmadan çalışan bir veritabanı elde edersin.

Ama tahmin her zaman doğru olmaz. Kolon adını Türkçe istiyorsundur, `decimal` alanın kaç basamak olacağını belirtmek istiyorsundur, iki sınıf arasında iki ayrı ilişki vardır ve EF hangisinin hangisi olduğunu bilemez. İşte o noktada araya girip "hayır, şöyle olacak" dersin. Bunu söylemenin iki yolu var.

Birincisi sınıfın üstüne yapıştırılan kısa etiketlerdir: `[Required]`, `[MaxLength(100)]`. Bunlara **Data Annotations** denir. Okunması kolaydır, sınıfa bakan herkes görür. Ama sınırlıdır — karmaşık ilişkileri, silme davranışlarını, index'leri bunlarla tam olarak anlatamazsın.

İkincisi ayrı bir yerde, kod olarak yazdığın tam konfigürasyondur. Buna **Fluent API** denir. Daha uzundur, sınıfa bakınca görünmez, ama her şeyi yapabilir. Ve en önemlisi: üçü çakıştığında Fluent API kazanır. Sıralama nettir — varsayılan kural en altta, etiket ortada, Fluent API en üstte.

Notun geri kalanı bu üç katmanı ve aralarındaki ilişkiyi anlatıyor. İlişki tiplerini (bire-çok, bire-bir, çoka-çok), silindiğinde ne olacağını, kalıtımın tabloya nasıl döküldüğünü, index'leri ve eşzamanlılık kontrolünü tek tek göreceksin. Şimdi detaya iniyoruz.

> **Ana benzetme:** Model konfigürasyonu bir **inşaat ruhsatı** gibidir. İmar yönetmeliği hiçbir şey yazmasan bile kat yüksekliğini, çekme mesafesini belirler — bu convention'dır. Projenin üstüne düştüğün kısa notlar bazı maddeleri değiştirir — bu Data Annotations'tır. Belediyeye teslim ettiğin mühürlü proje dosyası ise her şeyin üstündedir; yönetmelikle de, kenar notuyla da çelişse o geçerlidir — bu da Fluent API'dir.

---

## Bu Notta Ne Var

1. Convention — hiçbir şey yazmazsan ne olur
2. Üç konfigürasyon yolu ve öncelik sırası
3. Data Annotations ve sınırları
4. Fluent API ve `IEntityTypeConfiguration<T>`
5. Bire-çok ilişki (1-N)
6. Bire-bir ve çoka-çok ilişkiler
7. `OnDelete` — silme davranışları
8. Gölge özellik ve owned entity
9. Kalıtım eşlemesi: TPH, TPT, TPC
10. Index, unique constraint ve global query filter
11. Concurrency token ve value converter

---

## 1. Convention — Hiçbir Şey Yazmazsan Ne Olur

> **Benzetme —** Terziye gidip "bir gömlek" dersin, başka hiçbir şey söylemezsin. Terzi seni şöyle bir süzer, standart bedeni seçer, standart kol boyunu keser, standart düğmeyi diker. Çoğu zaman üstüne olur. Ama kolun uzunsa ya da yaka tipi fikrin varsa, söylemek zorundasın. Terzi ölçü almadan da bir gömlek verir sana — sadece senin istediğin gömleği vermez.

**Basitçe:** EF Core, sınıflarına bakarak veritabanını kendi kendine çıkarır. Anahtarın hangisi olduğunu, iki sınıfın birbiriyle ilişkili olduğunu, tablonun adının ne olacağını isim kalıplarından tahmin eder.

**Teknik olarak:** **Convention (varsayılan kural)** — EF Core'un, hiçbir açık konfigürasyon bulamadığında uyguladığı yerleşik kurallar bütünü.

Aşağıdaki sınıflarda tek satır konfigürasyon yok:

```csharp
public class Musteri
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public string? Telefon { get; set; }
    public DateTime KayitTarihi { get; set; }

    public List<Siparis> Siparisler { get; set; } = new();
}

public class Siparis
{
    public int Id { get; set; }
    public DateTime Tarih { get; set; }
    public decimal ToplamTutar { get; set; }

    public int MusteriId { get; set; }
    public Musteri Musteri { get; set; } = null!;
}

public class AppDbContext : DbContext
{
    public DbSet<Musteri> Musteriler => Set<Musteri>();
    public DbSet<Siparis> Siparisler => Set<Siparis>();
}
```

EF Core bundan şu şemayı üretir:

```sql
CREATE TABLE [Musteriler] (
    [Id] int NOT NULL IDENTITY,
    [Ad] nvarchar(max) NOT NULL,
    [Telefon] nvarchar(max) NULL,
    [KayitTarihi] datetime2 NOT NULL,
    CONSTRAINT [PK_Musteriler] PRIMARY KEY ([Id])
);

CREATE TABLE [Siparisler] (
    [Id] int NOT NULL IDENTITY,
    [Tarih] datetime2 NOT NULL,
    [ToplamTutar] decimal(18,2) NOT NULL,
    [MusteriId] int NOT NULL,
    CONSTRAINT [PK_Siparisler] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Siparisler_Musteriler_MusteriId]
        FOREIGN KEY ([MusteriId]) REFERENCES [Musteriler] ([Id]) ON DELETE CASCADE
);

CREATE INDEX [IX_Siparisler_MusteriId] ON [Siparisler] ([MusteriId]);
```

Tek satır yazmadan tablolar, anahtarlar, yabancı anahtar ve index geldi. Kuralları tek tek görelim.

### 1.1 Birincil anahtar bulma

EF Core şu iki isimden birini arar:

| Aranan isim | Örnek |
|---|---|
| `Id` | `Musteri.Id` |
| `<SınıfAdı>Id` | `Musteri.MusteriId` |

Büyük-küçük harf farkı önemsizdir; `ID` de bulunur. İkisi de yoksa EF Core hata verir: anahtarsız bir varlık tipi oluşturamaz.

Anahtar `int`, `long`, `Guid` gibi bir tipse ve veritabanı destekliyorsa, **otomatik artan (IDENTITY)** yapılır. `Guid` için değeri EF Core istemci tarafında üretir.

### 1.2 Yabancı anahtar ve ilişki çıkarımı

EF Core, bir sınıfta başka bir varlık tipine işaret eden özellik gördüğünde ilişki olduğunu varsayar. Buna **navigation property (gezinti özelliği)** denir.

Yabancı anahtar adayı olarak sırayla şu isimleri arar:

| Kalıp | `Siparis` içindeki `Musteri` navigation'ı için |
|---|---|
| `<NavigationAdı><AnahtarAdı>` | `MusteriId` |
| `<HedefSınıfAdı><AnahtarAdı>` | `MusteriId` |
| `<AnahtarAdı>` | `Id` — ama bu zaten PK olduğu için kullanılmaz |

Bulursa onu FK yapar. Bulamazsa **gölge özellik** olarak kendisi bir FK kolonu uydurur (8. bölüm).

```csharp
// FK özelliği yazılmamış hâli — EF Core yine de MusteriId kolonu üretir,
// ama sen C# tarafından o değeri okuyamazsın.
public class Siparis
{
    public int Id { get; set; }
    public Musteri Musteri { get; set; } = null!;
}
```

### 1.3 Tablo ve kolon adlandırma

| Ne | Varsayılan kaynak |
|---|---|
| Tablo adı | `DbSet<T>` özelliğinin adı (`Musteriler`) |
| `DbSet` yoksa | Sınıfın adı (`Musteri`) |
| Kolon adı | Özelliğin adı |
| Şema | Sağlayıcının varsayılanı (SQL Server'da `dbo`) |

### 1.4 Tip ve null eşlemesi

| C# tipi | SQL Server karşılığı |
|---|---|
| `int` | `int NOT NULL` |
| `int?` | `int NULL` |
| `string` (nullable referans tipleri açık, `string`) | `nvarchar(max) NOT NULL` |
| `string?` | `nvarchar(max) NULL` |
| `decimal` | `decimal(18,2)` |
| `DateTime` | `datetime2` |
| `bool` | `bit` |
| `Guid` | `uniqueidentifier` |

> **Uyarı:** `string` için varsayılan `nvarchar(max)`'tır. Bu kolon üzerinde index oluşturamazsın (900 bayt sınırı) ve gereksiz yer kaplar. Her metin alanına uzunluk vermeyi alışkanlık hâline getir.

> **Uyarı:** Nullable referans tipleri (`<Nullable>enable</Nullable>`) açıksa, `string` ile `string?` arasındaki fark doğrudan `NOT NULL` / `NULL` farkına dönüşür. MvcCv'de bu ayar açık; oradaki `T?` tartışmasının kökeni aynı özelliktir.

> **Bu benzetme şurada bozulur:** Terzi standart bedeni verir ama sana "bu standart" der, haberin olur. EF Core convention'ları sessizdir. `decimal(18,2)` kararını alır ve sana söylemez; kuruş hassasiyeti yetmediğinde bunu ancak veri bozulunca fark edersin. Varsayılanın ne olduğunu bilmek senin sorumluluğundadır.

---

## 2. Üç Konfigürasyon Yolu ve Öncelik Sırası

> **Benzetme —** Yolda giderken hız sınırını üç şey söyler. Kanunda yazan genel limit vardır (şehir içi 50). Yol kenarındaki levha bunu değiştirir (30 yazıyorsa 30'dur). Ama kavşakta bir trafik polisi elini kaldırmışsa, levha ne derse desin duracaksın. Üçü aynı şeyi söylerken fark etmez; çeliştiklerinde sıra bellidir.

**Basitçe:** Aynı ayarı üç ayrı yerde söyleyebilirsin. Çeliştiklerinde en açık olan kazanır: Fluent API > Data Annotations > convention.

**Teknik olarak:** EF Core model oluştururken önce convention'ları uygular, sonra Data Annotations'ları okuyup üstüne yazar, en son `OnModelCreating` içindeki Fluent API çağrılarını işler.

```
convention  <  Data Annotations  <  Fluent API
  (en zayıf)                          (en güçlü)
```

Çakışma örneği:

```csharp
[Table("Musteri")]                 // Data Annotations: tablo adı "Musteri" olsun
public class Musteri
{
    public int Id { get; set; }

    [MaxLength(50)]                // Data Annotations: 50 karakter
    public string Ad { get; set; } = null!;
}

// Fluent API — OnModelCreating içinde
modelBuilder.Entity<Musteri>().ToTable("TblMusteriler");   // kazanan bu
modelBuilder.Entity<Musteri>().Property(m => m.Ad).HasMaxLength(200);  // kazanan bu
```

Sonuç: tablo adı `TblMusteriler`, `Ad` kolonu `nvarchar(200)`. Annotation'lar yok sayılmadı — sadece üstlerine yazıldı.

### Hangisini ne zaman kullanmalı

| Durum | Tercih |
|---|---|
| Basit kısıt: zorunluluk, uzunluk, kolon adı | Data Annotations — sınıfa bakan görür |
| İlişki yönü, `OnDelete`, birleşik anahtar | Fluent API — annotation yetmez |
| Index, unique constraint, filtered index | Fluent API |
| Global query filter, value converter, owned entity | Fluent API — annotation karşılığı yok |
| Varlık sınıfı başka bir katmanda, referans veremiyorsun | Fluent API — sınıfa dokunmadan yapılandırır |

Yaygın pratik: doğrulama amaçlı annotation'ları (`[Required]`, `[MaxLength]`) modelde bırak, şema kararlarını Fluent API'ye taşı. İkisini karıştırmak sorun değildir; **aynı ayarı iki yerde yazmak** sorundur, çünkü hangisinin geçerli olduğunu okuyan kişi göremez.

> **Bu benzetme şurada bozulur:** Trafikte polisin işareti anlıktır, levha yerinde kalır. Konfigürasyonda ise Fluent API kalıcıdır ve annotation'ı sessizce etkisizleştirir. Sınıfa bakan biri `[MaxLength(50)]` görür ve 50 sanır; gerçek 200'dür. Bu yüzden "iki yerde yazma" kuralı sadece stil meselesi değil, yanlış okumayı engelleyen bir tedbirdir.

---

## 3. Data Annotations ve Sınırları

> **Benzetme —** İlaç kutusunun üstündeki etikettir. "Günde iki kez", "buzdolabında saklayın" gibi kısa ve yerinde bilgiler yazar. Kutuya bakan herkes görür, prospektüsü açmaya gerek kalmaz. Ama ilacın hangi enzimle etkileştiğini kutuya sığdıramazsın; o bilgi prospektüsün içindedir.

**Basitçe:** Sınıfın ve özelliklerin üstüne yazdığın köşeli parantezli etiketlerdir. Kısa ayarlar için idealdir, karmaşık olanlar için yetmez.

**Teknik olarak:** İki ayrı isim alanından gelirler ve bu ayrım önemlidir:

| İsim alanı | Amacı | EF Core'a etkisi |
|---|---|---|
| `System.ComponentModel.DataAnnotations` | Doğrulama (validation) | Bazıları şemayı da etkiler |
| `System.ComponentModel.DataAnnotations.Schema` | Şema eşlemesi | Sadece şema |

### 3.1 Temel etiketler

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("TblMusteriler", Schema = "satis")]
public class Musteri
{
    [Key]                                   // birincil anahtar (isim kalıba uymuyorsa şart)
    public int MusteriNo { get; set; }

    [Required]                              // NOT NULL
    [MaxLength(100)]                        // nvarchar(100)
    [Column("MusteriAdi")]                  // kolon adı
    public string Ad { get; set; } = null!;

    [MaxLength(20)]
    public string? Telefon { get; set; }

    [Column(TypeName = "date")]             // datetime2 yerine date
    public DateTime KayitTarihi { get; set; }

    [Precision(18, 4)]                      // EF Core 5+ — decimal(18,4)
    public decimal KrediLimiti { get; set; }

    [NotMapped]                             // veritabanına hiç yazılmaz
    public string GorunenAd => $"{Ad} ({Telefon})";

    public List<Siparis> Siparisler { get; set; } = new();
}
```

Etiketlerin ne yaptığı:

| Etiket | Ne yapar |
|---|---|
| `[Key]` | Birincil anahtarı belirtir |
| `[Required]` | Kolonu `NOT NULL` yapar |
| `[MaxLength(n)]` | Maksimum uzunluk — `nvarchar(n)` |
| `[StringLength(n)]` | Aynı etkiyi yapar, ayrıca `MinimumLength` doğrulaması sunar |
| `[Column("ad")]` | Kolon adını değiştirir |
| `[Column(TypeName = "...")]` | Kolonun SQL tipini doğrudan verir |
| `[Table("ad", Schema = "...")]` | Tablo adını ve şemasını değiştirir |
| `[NotMapped]` | Özelliği veya sınıfı eşlemenin dışında bırakır |
| `[Precision(p, s)]` | `decimal` basamak ve ondalık sayısı (EF Core 5+) |
| `[DatabaseGenerated(...)]` | Değerin kim tarafından üretildiğini belirler |
| `[Timestamp]` | `rowversion` eşzamanlılık kolonu (11. bölüm) |
| `[Index(...)]` | Sınıf üstünde index tanımı (EF Core 5+) |

`[DatabaseGenerated]` üç seçenek alır:

```csharp
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]  // INSERT'te DB üretir
public int Id { get; set; }

[DatabaseGenerated(DatabaseGeneratedOption.Computed)]  // INSERT ve UPDATE'te DB üretir
public DateTime SonGuncelleme { get; set; }

[DatabaseGenerated(DatabaseGeneratedOption.None)]      // değeri sen verirsin
public int Kod { get; set; }
```

### 3.2 İlişki etiketleri

`[ForeignKey]` iki yerde yazılabilir ve ikisi de geçerlidir:

```csharp
public class Siparis
{
    public int Id { get; set; }

    public int MusteriKodu { get; set; }     // isim kalıba uymuyor

    [ForeignKey(nameof(MusteriKodu))]        // navigation üstünde: FK özelliğini söyler
    public Musteri Musteri { get; set; } = null!;
}

// Alternatif yazım — FK özelliği üstünde: navigation'ı söyler
public class Siparis2
{
    public int Id { get; set; }

    [ForeignKey(nameof(Musteri))]
    public int MusteriKodu { get; set; }

    public Musteri Musteri { get; set; } = null!;
}
```

`[InverseProperty]`, iki sınıf arasında **birden fazla ilişki** olduğunda hangisinin hangisiyle eşleştiğini söyler. Bu olmadan EF Core hata verir:

```csharp
public class Siparis
{
    public int Id { get; set; }

    public int OlusturanPersonelId { get; set; }
    [InverseProperty(nameof(Personel.OlusturduguSiparisler))]
    public Personel OlusturanPersonel { get; set; } = null!;

    public int? OnaylayanPersonelId { get; set; }
    [InverseProperty(nameof(Personel.OnayladigiSiparisler))]
    public Personel? OnaylayanPersonel { get; set; }
}

public class Personel
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;

    [InverseProperty(nameof(Siparis.OlusturanPersonel))]
    public List<Siparis> OlusturduguSiparisler { get; set; } = new();

    [InverseProperty(nameof(Siparis.OnaylayanPersonel))]
    public List<Siparis> OnayladigiSiparisler { get; set; } = new();
}
```

> **Uyarı:** İki sınıf arasında iki ilişki varsa ve `[InverseProperty]` ya da Fluent API ile eşleştirme yapmazsan, EF Core `InvalidOperationException` fırlatır. Hatanın metni "Unable to determine the relationship represented by navigation..." ile başlar. Bu hatayı gördüğünde ilk bakacağın yer buradır.

### 3.3 Data Annotations neyi yapamaz

| Yapamadığı | Neden |
|---|---|
| Birleşik birincil anahtar | `[Key]` birden fazla özelliğe konunca EF Core sıra bilemez, hata verir |
| `OnDelete` davranışı | Karşılığı yok |
| Filtered index, dahil edilen kolonlu index | Karşılığı yok |
| Global query filter | Karşılığı yok |
| Value converter | Karşılığı yok |
| Owned entity / value object | Karşılığı yok (`[Owned]` var ama yapılandırılamaz) |
| Varsayılan değer (`HasDefaultValueSql`) | Karşılığı yok |
| Kalıtım stratejisi (TPT/TPC) | Karşılığı yok |
| Başka assembly'deki sınıfı yapılandırma | Sınıfa dokunamazsın |

Birleşik anahtar için mecburen Fluent API gerekir:

```csharp
// SiparisDetaylari tablosunda anahtar (SiparisId, UrunId) çiftidir
modelBuilder.Entity<SiparisDetay>()
    .HasKey(sd => new { sd.SiparisId, sd.UrunId });
```

> **Bu benzetme şurada bozulur:** İlaç etiketi eksik bilgi verse de yanlış bilgi vermez. Data Annotations ise bazen yanıltır: `[Required]` hem EF Core'a "NOT NULL" der, hem de ASP.NET Core model doğrulamasına "bu alan boş geçilemez" der. İki ayrı sistem aynı etiketi okur. Birini istediğin hâlde diğerini de almış olursun ve bunu fark etmezsin.

---

## 4. Fluent API ve `IEntityTypeConfiguration<T>`

> **Benzetme —** Büyük bir inşaatta bütün detayı tek pafta üstüne çizmezsin. Elektrik projesi ayrı, tesisat projesi ayrı, her kat ayrı pafta. Hepsi aynı ruhsat dosyasında toplanır ama biri diğerinin üstünü kaplamaz. Usta hangi paftayı arayacağını bilir.

**Basitçe:** `OnModelCreating` metodunun içinde, kod yazarak modelin her ayrıntısını tarif edersin. Model büyüyünce bu metot şişer; o zaman her varlık için ayrı bir konfigürasyon sınıfı açarsın.

**Teknik olarak:** **Fluent API** — `ModelBuilder` üzerinden zincirleme metot çağrılarıyla modeli yapılandırma yöntemi. `DbContext.OnModelCreating(ModelBuilder)` içinde çalışır.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Musteri> Musteriler => Set<Musteri>();
    public DbSet<Siparis> Siparisler => Set<Siparis>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Musteri>(entity =>
        {
            entity.ToTable("TblMusteriler");
            entity.HasKey(m => m.Id);

            entity.Property(m => m.Ad)
                  .HasColumnName("MusteriAdi")
                  .HasMaxLength(100)
                  .IsRequired();

            entity.Property(m => m.Telefon).HasMaxLength(20);

            entity.Property(m => m.KayitTarihi)
                  .HasColumnType("date")
                  .HasDefaultValueSql("GETDATE()");

            entity.HasIndex(m => m.Telefon).IsUnique();
        });

        base.OnModelCreating(modelBuilder);
    }
}
```

> **Uyarı:** Kendi kodunu yazmadan önce ya da sonra `base.OnModelCreating(modelBuilder)` çağırmayı unutma. Identity gibi hazır altyapılar kendi modellerini orada kurar. Unutursan "The entity type 'IdentityUserLogin' requires a primary key" benzeri hatalar alırsın.

### 4.1 Sık kullanılan `Property` metotları

| Metot | Ne yapar |
|---|---|
| `.HasColumnName("...")` | Kolon adı |
| `.HasColumnType("...")` | SQL tipi |
| `.HasMaxLength(n)` | Uzunluk |
| `.IsRequired()` / `.IsRequired(false)` | `NOT NULL` / `NULL` |
| `.HasPrecision(18, 4)` | `decimal` hassasiyeti |
| `.HasDefaultValue(0)` | Sabit varsayılan değer |
| `.HasDefaultValueSql("GETDATE()")` | SQL ifadeli varsayılan |
| `.HasComputedColumnSql("[Adet] * [Fiyat]")` | Hesaplanmış kolon |
| `.ValueGeneratedOnAdd()` | Ekleme sırasında DB üretir |
| `.ValueGeneratedNever()` | Değeri sen verirsin |
| `.IsConcurrencyToken()` | Eşzamanlılık kontrolüne dahil et |
| `.HasConversion<string>()` | Value converter (11. bölüm) |

Hesaplanmış kolon örneği — sipariş detayında satır toplamı:

```csharp
modelBuilder.Entity<SiparisDetay>()
    .Property(sd => sd.SatirToplami)
    .HasComputedColumnSql("[Adet] * [BirimFiyat]", stored: true);
```

`stored: true` değeri veritabanında saklar (SQL Server'da `PERSISTED`), böylece üzerine index koyabilirsin.

### 4.2 Konfigürasyonu ayrı sınıflara bölmek

On beş varlığı olan bir modelde `OnModelCreating` bin satırı geçer. Çözüm `IEntityTypeConfiguration<T>` arayüzüdür.

```csharp
public class MusteriConfiguration : IEntityTypeConfiguration<Musteri>
{
    public void Configure(EntityTypeBuilder<Musteri> builder)
    {
        builder.ToTable("TblMusteriler");

        builder.Property(m => m.Ad)
               .HasMaxLength(100)
               .IsRequired();

        builder.Property(m => m.Telefon).HasMaxLength(20);

        builder.HasIndex(m => m.Telefon)
               .IsUnique()
               .HasDatabaseName("UX_Musteri_Telefon");

        builder.HasMany(m => m.Siparisler)
               .WithOne(s => s.Musteri)
               .HasForeignKey(s => s.MusteriId)
               .OnDelete(DeleteBehavior.Restrict);
    }
}
```

Bu sınıfları `OnModelCreating` içinde tek tek uygulayabilirsin:

```csharp
modelBuilder.ApplyConfiguration(new MusteriConfiguration());
modelBuilder.ApplyConfiguration(new SiparisConfiguration());
```

Ya da hepsini tek satırda tarayabilirsin:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    base.OnModelCreating(modelBuilder);
}
```

`ApplyConfigurationsFromAssembly`, verilen assembly'deki `IEntityTypeConfiguration<T>` uygulayan tüm **public, soyut olmayan** sınıfları bulup uygular. Yeni bir konfigürasyon sınıfı eklediğinde `OnModelCreating`'e dokunman gerekmez.

> **Uyarı:** Sınıfı `internal` yaparsan ya da parametresiz kurucusunu kaldırırsan tarama onu bulamaz. Hata da vermez — konfigürasyon sessizce uygulanmaz. Migration çıktısında beklediğin kolonu göremiyorsan önce erişim belirleyicisine bak.

### 4.3 Aynı ayarı topluca uygulamak

Bütün `decimal` alanlara aynı hassasiyeti vermek gibi işler için model üzerinde gezinebilirsin:

```csharp
foreach (var entityType in modelBuilder.Model.GetEntityTypes())
{
    foreach (var property in entityType.GetProperties())
    {
        if (property.ClrType == typeof(decimal) || property.ClrType == typeof(decimal?))
        {
            property.SetPrecision(18);
            property.SetScale(4);
        }
    }
}
```

EF Core 6'dan itibaren daha temiz bir yolu var — **pre-convention model configuration**:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<decimal>().HavePrecision(18, 4);
    configurationBuilder.Properties<string>().HaveMaxLength(200);
}
```

Bu metot `OnModelCreating`'den **önce** çalışır ve varsayılanları değiştirir. Yani sonradan `OnModelCreating` içinde yazdığın her şey yine üstüne yazabilir.

> **Bu benzetme şurada bozulur:** Paftalar birbirinden bağımsızdır; elektrik projesi tesisatı bozmaz. Konfigürasyon sınıfları ise aynı ilişkiyi iki taraftan yapılandırabilir. `MusteriConfiguration` içinde `HasMany(...).WithOne(...)` yazarsan ve `SiparisConfiguration` içinde `HasOne(...).WithMany(...)` ile aynı ilişkiyi farklı `OnDelete` ile tanımlarsan, son uygulanan kazanır. Bir ilişkiyi tek bir yerden yapılandır.

---

## 5. Bire-Çok İlişki (1-N)

> **Benzetme —** Lokantada bir masaya bir adisyon açılır, o adisyona birden çok satır yazılır: iki çorba, bir pide, üç ayran. Her satır tek bir adisyona aittir; hiçbir satır iki adisyona birden yazılamaz. Adisyonun numarası her satırın üstünde durur — satırın nereye ait olduğunu o numara söyler.

**Basitçe:** Bir müşterinin çok siparişi olur, bir sipariş tek müşteriye aittir. Veritabanında bu, "çok" tarafında duran bir yabancı anahtar kolonuyla kurulur.

**Teknik olarak:** **1-N (one-to-many)** ilişki. İki taraf vardır:

| Taraf | Adı | Örnek | Özelliği |
|---|---|---|---|
| "Bir" tarafı | **Principal (asıl)** | `Musteri` | Anahtarı referans edilir |
| "Çok" tarafı | **Dependent (bağımlı)** | `Siparis` | Yabancı anahtarı taşır |

Yabancı anahtar **her zaman bağımlı tarafta** durur. Tablo şemasında `Siparisler.MusteriId` vardır; `Musteriler` tablosunda sipariş bilgisi yoktur.

### 5.1 Fluent API ile tanımlama

İlişkiyi iki yönden de yazabilirsin; ikisi aynı ilişkiyi kurar:

```csharp
// "Bir" tarafından başlayarak
modelBuilder.Entity<Musteri>()
    .HasMany(m => m.Siparisler)
    .WithOne(s => s.Musteri)
    .HasForeignKey(s => s.MusteriId)
    .OnDelete(DeleteBehavior.Restrict);

// "Çok" tarafından başlayarak — birebir aynı sonuç
modelBuilder.Entity<Siparis>()
    .HasOne(s => s.Musteri)
    .WithMany(m => m.Siparisler)
    .HasForeignKey(s => s.MusteriId)
    .OnDelete(DeleteBehavior.Restrict);
```

Zincirin okunuşu: "Müşterinin çok siparişi **vardır** (`HasMany`), her siparişin de tek müşterisi **vardır** (`WithOne`), bağlantı `MusteriId` üzerinden kurulur."

### 5.2 Zorunlu ve isteğe bağlı ilişki

Farkı, yabancı anahtarın nullable olup olmaması belirler:

```csharp
public class Siparis
{
    public int MusteriId { get; set; }          // zorunlu — her siparişin müşterisi var
    public Musteri Musteri { get; set; } = null!;

    public int? KampanyaId { get; set; }        // isteğe bağlı — kampanyasız da olur
    public Kampanya? Kampanya { get; set; }
}
```

| FK tipi | İlişki | Varsayılan silme davranışı |
|---|---|---|
| `int` | Zorunlu (required) | `Cascade` |
| `int?` | İsteğe bağlı (optional) | `ClientSetNull` |

Fluent API ile açıkça da söylenir:

```csharp
modelBuilder.Entity<Siparis>()
    .HasOne(s => s.Kampanya)
    .WithMany(k => k.Siparisler)
    .HasForeignKey(s => s.KampanyaId)
    .IsRequired(false);
```

### 5.3 Navigation property türleri

| Tür | Nerede | Örnek |
|---|---|---|
| Referans navigation | Bağımlı tarafta | `Siparis.Musteri` |
| Koleksiyon navigation | Asıl tarafta | `Musteri.Siparisler` |
| Çift yönlü | İki tarafta da var | Yukarıdaki ikisi birlikte |
| Tek yönlü | Sadece bir tarafta var | Sadece `Siparis.Musteri` |

Tek yönlü ilişkide karşı tarafı boş bırakırsın:

```csharp
// Musteri sınıfında Siparisler koleksiyonu yok
modelBuilder.Entity<Siparis>()
    .HasOne(s => s.Musteri)
    .WithMany()                       // karşı navigation yok
    .HasForeignKey(s => s.MusteriId);
```

Koleksiyon navigation'ları için tip seçimi:

```csharp
public List<Siparis> Siparisler { get; set; } = new();        // en yaygın
public ICollection<Siparis> Siparisler2 { get; set; } = new List<Siparis>();
public HashSet<Siparis> Siparisler3 { get; set; } = new();    // çok elemanda Contains hızlı
```

> **Uyarı:** Koleksiyonu **mutlaka başlat** (`= new()`). Başlatmazsan `musteri.Siparisler.Add(...)` satırı `NullReferenceException` verir. EF Core veritabanından okurken koleksiyonu kendisi oluşturur, ama sen `new Musteri()` ile nesne ürettiğinde oluşturmaz.

### 5.4 Kendine referans veren ilişki

Kategorilerin alt kategorisi olması gibi durumlarda ilişki aynı tabloya bakar:

```csharp
public class Kategori
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;

    public int? UstKategoriId { get; set; }
    public Kategori? UstKategori { get; set; }
    public List<Kategori> AltKategoriler { get; set; } = new();
}

modelBuilder.Entity<Kategori>()
    .HasOne(k => k.UstKategori)
    .WithMany(k => k.AltKategoriler)
    .HasForeignKey(k => k.UstKategoriId)
    .OnDelete(DeleteBehavior.Restrict);
```

FK'nın `int?` olması şarttır — en üstteki kategorinin üstü yoktur. `Restrict` de gereklidir; kendine referans veren bir tabloda `Cascade` SQL Server tarafından reddedilir.

> **Bu benzetme şurada bozulur:** Adisyon benzetmesinde satır, adisyon olmadan var olamaz. Kodda ise `new Siparis()` yazıp `MusteriId` vermeden nesneyi oluşturabilirsin — C# tarafında hiçbir şey seni durdurmaz. Kısıt ancak `SaveChanges()` anında, veritabanı tarafında devreye girer. C# nesne grafiği ile veritabanı şeması aynı kuralları aynı anda uygulamaz.

---

## 6. Bire-Bir ve Çoka-Çok İlişkiler

> **Benzetme —** Bire-bir ilişki, kişi ve kimlik numarasıdır: her kişinin bir numarası, her numaranın bir kişisi vardır. Çoka-çok ise kursiyer ve kurstur: bir kursiyer birkaç kursa yazılır, bir kursa birkaç kursiyer gelir. İkincisinde ilişkiyi ne kursiyerin dosyasında ne kursun dosyasında tutabilirsin — ayrı bir kayıt defteri açarsın: "kim hangi kursa, ne zaman yazıldı".

**Basitçe:** Bire-bir ilişkide iki tarafın da tek kaydı vardır ve FK'yı hangi tarafa koyacağını sen söylemelisin. Çoka-çokta ise iki tarafı bağlayan üçüncü bir tablo gerekir.

### 6.1 Bire-bir (1-1)

**Teknik olarak:** 1-1 ilişkide EF Core asıl (principal) ve bağımlı (dependent) tarafı kendi başına bulamaz — çünkü ikisi de aday. FK'nın hangi tarafta olacağını `HasForeignKey<T>()` ile açıkça yazman gerekir.

```csharp
public class Musteri
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public MusteriDetay? Detay { get; set; }
}

public class MusteriDetay
{
    public int Id { get; set; }
    public string? VergiNo { get; set; }
    public string? Notlar { get; set; }

    public int MusteriId { get; set; }             // FK burada
    public Musteri Musteri { get; set; } = null!;
}

modelBuilder.Entity<Musteri>()
    .HasOne(m => m.Detay)
    .WithOne(d => d.Musteri)
    .HasForeignKey<MusteriDetay>(d => d.MusteriId);   // generic parametre: bağımlı taraf
```

`HasForeignKey<MusteriDetay>` yazımındaki tip parametresi "bağımlı olan `MusteriDetay`'dır" der. Bunu yazmazsan EF Core şu hatayı verir: "Unable to determine which end of the relationship is the principal end."

EF Core, FK kolonunu **benzersiz (unique)** index ile korur — böylece bir müşterinin iki detayı olamaz:

```sql
CREATE UNIQUE INDEX [IX_MusteriDetaylari_MusteriId]
    ON [MusteriDetaylari] ([MusteriId]);
```

Daha sıkı bir yol: bağımlı tarafın birincil anahtarını doğrudan FK yapmak. Böylece ayrı bir `Id` kolonu bile olmaz.

```csharp
public class MusteriDetay
{
    public int MusteriId { get; set; }              // hem PK hem FK
    public Musteri Musteri { get; set; } = null!;
    public string? VergiNo { get; set; }
}

modelBuilder.Entity<MusteriDetay>().HasKey(d => d.MusteriId);
modelBuilder.Entity<MusteriDetay>()
    .HasOne(d => d.Musteri)
    .WithOne(m => m.Detay)
    .HasForeignKey<MusteriDetay>(d => d.MusteriId);
```

### 6.2 Çoka-çok (N-N) — otomatik ara tablo

EF Core 5'ten itibaren ara tabloyu yazmadan çoka-çok kurabilirsin. İki tarafta koleksiyon olması yeterlidir:

```csharp
public class Urun
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public List<Kategori> Kategoriler { get; set; } = new();
}

public class Kategori
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public List<Urun> Urunler { get; set; } = new();
}
```

EF Core arka planda bir **join entity** üretir ve tabloyu kendisi oluşturur:

```sql
CREATE TABLE [KategoriUrun] (
    [KategorilerId] int NOT NULL,
    [UrunlerId] int NOT NULL,
    CONSTRAINT [PK_KategoriUrun] PRIMARY KEY ([KategorilerId], [UrunlerId]),
    CONSTRAINT [FK_KategoriUrun_Kategoriler_KategorilerId]
        FOREIGN KEY ([KategorilerId]) REFERENCES [Kategoriler] ([Id]) ON DELETE CASCADE,
    CONSTRAINT [FK_KategoriUrun_Urunler_UrunlerId]
        FOREIGN KEY ([UrunlerId]) REFERENCES [Urunler] ([Id]) ON DELETE CASCADE
);
```

Kullanımı doğrudan koleksiyon üzerindendir:

```csharp
var urun = await _db.Urunler
    .Include(u => u.Kategoriler)
    .FirstAsync(u => u.Id == 5);

var kategori = await _db.Kategoriler.FindAsync(3);
urun.Kategoriler.Add(kategori!);          // ara tabloya satır eklenir
await _db.SaveChangesAsync();
```

Ara tablonun adını ve kolonlarını yine de yapılandırabilirsin:

```csharp
modelBuilder.Entity<Urun>()
    .HasMany(u => u.Kategoriler)
    .WithMany(k => k.Urunler)
    .UsingEntity(j => j.ToTable("UrunKategorileri"));
```

### 6.3 Çoka-çok — açık join entity

Ara tabloda **fazladan alan** tutman gerekiyorsa otomatik yol yetmez. Sınıfı kendin yazarsın:

```csharp
public class UrunKategori
{
    public int UrunId { get; set; }
    public Urun Urun { get; set; } = null!;

    public int KategoriId { get; set; }
    public Kategori Kategori { get; set; } = null!;

    public DateTime EklenmeTarihi { get; set; }     // fazladan alan
    public int Sira { get; set; }                   // fazladan alan
}

public class Urun
{
    public int Id { get; set; }
    public List<UrunKategori> UrunKategorileri { get; set; } = new();
}

// Konfigürasyon
modelBuilder.Entity<UrunKategori>()
    .HasKey(uk => new { uk.UrunId, uk.KategoriId });

modelBuilder.Entity<UrunKategori>()
    .HasOne(uk => uk.Urun)
    .WithMany(u => u.UrunKategorileri)
    .HasForeignKey(uk => uk.UrunId);

modelBuilder.Entity<UrunKategori>()
    .HasOne(uk => uk.Kategori)
    .WithMany(k => k.UrunKategorileri)
    .HasForeignKey(uk => uk.KategoriId);
```

EF Core 5+ ile ikisini birleştirip hem doğrudan koleksiyonu hem join entity'yi kullanabilirsin:

```csharp
modelBuilder.Entity<Urun>()
    .HasMany(u => u.Kategoriler)
    .WithMany(k => k.Urunler)
    .UsingEntity<UrunKategori>(
        r => r.HasOne(uk => uk.Kategori).WithMany().HasForeignKey(uk => uk.KategoriId),
        l => l.HasOne(uk => uk.Urun).WithMany().HasForeignKey(uk => uk.UrunId),
        j => j.ToTable("UrunKategorileri"));
```

### 6.4 Hangisini ne zaman

| Durum | Seçim |
|---|---|
| Ara tabloda sadece iki FK var | Otomatik ara tablo — daha az kod |
| Ara tabloda tarih, sıra, adet, fiyat gibi alan var | Açık join entity — zorunlu |
| Ara tablodaki satırları doğrudan sorgulaman gerekiyor | Açık join entity |
| Ara tabloya kendi başına index / kısıt koyacaksın | Açık join entity |
| Sadece "hangi ürün hangi kategoride" sorusu | Otomatik yeterli |

`SiparisDetaylari` tablosu bu ayrımın tipik örneğidir. Sipariş ile ürün arasında çoka-çok gibi görünür ama satırda `Adet` ve `BirimFiyat` vardır. Bu yüzden asla otomatik ara tablo olarak kurulmaz; kendi varlık sınıfıdır:

```csharp
public class SiparisDetay
{
    public int SiparisId { get; set; }
    public Siparis Siparis { get; set; } = null!;

    public int UrunId { get; set; }
    public Urun Urun { get; set; } = null!;

    public int Adet { get; set; }
    public decimal BirimFiyat { get; set; }
    public decimal IndirimOrani { get; set; }
}
```

> **Bu benzetme şurada bozulur:** Kayıt defteri benzetmesinde defteri açıp satırlara bakabilirsin. Otomatik ara tabloda böyle bir sınıf **yoktur** — EF Core onu sadece kendi modelinde tutar. `_db.UrunKategori` diye bir `DbSet` yazamazsın. Ara tablonun satırlarını doğrudan sorgulaman gerektiği an, otomatik yol biter ve açık join entity'ye geçmek zorunda kalırsın. Bu geçiş bir migration ister; baştan doğru seçmek daha ucuzdur.

---

## 7. `OnDelete` — Silme Davranışları

> **Benzetme —** Bir kargo şubesi kapanıyor. O şubeye bağlı paketler ne olacak? Üç seçenek var: paketleri de imha edersin (cascade), şubeyi kapatmayı reddedersin çünkü paket duruyor (restrict), ya da paketlerin üstündeki şube bilgisini silip "merkez" yaparsın (set null). Hangisinin doğru olduğu paketin ne olduğuna bağlıdır — kimse dördüncü bir cevap veremez.

**Basitçe:** Asıl kaydı sildiğinde ona bağlı kayıtlara ne olacağını sen belirlersin. Yanlış seçim ya veri kaybettirir ya da silme işlemini imkânsız hâle getirir.

**Teknik olarak:** `DeleteBehavior` enum'ı, asıl (principal) varlık silindiğinde ya da ilişki koparıldığında bağımlı (dependent) varlıklara ne yapılacağını belirler.

| Değer | Bellekte (takip edilen) | Veritabanında |
|---|---|---|
| `Cascade` | Bağımlılar silinir | `ON DELETE CASCADE` |
| `Restrict` | Bir şey yapılmaz | `ON DELETE NO ACTION` — silme reddedilir |
| `SetNull` | FK `null` yapılır | `ON DELETE SET NULL` |
| `ClientSetNull` | FK `null` yapılır | `NO ACTION` — DB tarafında kısıt yok |
| `ClientCascade` | Bağımlılar silinir | `NO ACTION` — silmeyi EF Core yapar |
| `NoAction` | Bir şey yapılmaz | `NO ACTION` |
| `ClientNoAction` | Bir şey yapılmaz | `NO ACTION` — EF Core kontrol bile etmez |

"Client" ön eki olan seçenekler işi **EF Core'a** bırakır; olmayanlar veritabanına bırakır. Fark kritiktir: `ClientCascade` sadece bağımlı varlıklar **change tracker'a yüklenmişse** çalışır. Yüklenmediyse veritabanı FK hatası fırlatır.

### 7.1 Varsayılanlar

```csharp
// Zorunlu ilişki (int MusteriId) → varsayılan Cascade
// Müşteriyi silersen bütün siparişleri de gider
modelBuilder.Entity<Siparis>()
    .HasOne(s => s.Musteri)
    .WithMany(m => m.Siparisler)
    .HasForeignKey(s => s.MusteriId);
```

```csharp
// İsteğe bağlı ilişki (int? KampanyaId) → varsayılan ClientSetNull
modelBuilder.Entity<Siparis>()
    .HasOne(s => s.Kampanya)
    .WithMany(k => k.Siparisler)
    .HasForeignKey(s => s.KampanyaId);
```

> **Uyarı:** Zorunlu ilişkilerde varsayılanın `Cascade` olduğunu unutma. Hiçbir şey yazmazsan, bir müşteriyi silmek onun tüm siparişlerini, dolayısıyla tüm sipariş detaylarını zincirleme siler. Muhasebe verisinde bu felakettir. Finansal kayıt tutan tablolarda açıkça `Restrict` yaz.

### 7.2 Doğru seçim

| İlişki | Önerilen | Neden |
|---|---|---|
| `Siparis` → `SiparisDetay` | `Cascade` | Detay siparişsiz anlamsızdır |
| `Musteri` → `Siparis` | `Restrict` | Sipariş geçmişi silinmemeli |
| `Kategori` → `Urun` | `Restrict` veya `SetNull` | Ürün kategorisiz de var olabilir |
| `Siparis` → `Kampanya` | `SetNull` | Kampanya biter, sipariş kalır |
| Kendine referans (`Kategori` → `Kategori`) | `Restrict` | SQL Server `Cascade`'i reddeder |

```csharp
modelBuilder.Entity<SiparisDetay>()
    .HasOne(sd => sd.Siparis)
    .WithMany(s => s.Detaylar)
    .HasForeignKey(sd => sd.SiparisId)
    .OnDelete(DeleteBehavior.Cascade);          // sipariş giderse detay da gider

modelBuilder.Entity<SiparisDetay>()
    .HasOne(sd => sd.Urun)
    .WithMany(u => u.SiparisDetaylari)
    .HasForeignKey(sd => sd.UrunId)
    .OnDelete(DeleteBehavior.Restrict);         // ürün satılmışsa silinemez
```

### 7.3 SQL Server'ın çoklu cascade path hatası

Migration uygularken şu hatayı er ya da geç görürsün:

```
Introducing FOREIGN KEY constraint 'FK_SiparisDetaylari_Urunler_UrunId'
on table 'SiparisDetaylari' may cause cycles or multiple cascade paths.
Specify ON DELETE NO ACTION or ON UPDATE NO ACTION, or modify other
FOREIGN KEY constraints.
```

**Sebebi:** SQL Server, aynı tabloya **birden fazla cascade yolundan** ulaşılmasına izin vermez. Örnek:

```
Musteriler ──cascade──> Siparisler ──cascade──> SiparisDetaylari
Urunler    ──cascade──> SiparisDetaylari
```

`SiparisDetaylari` tablosuna iki ayrı zincirle cascade ulaşıyor. SQL Server bunu çözemez ve tabloyu oluşturmayı reddeder. Bu, EF Core'un değil veritabanının kısıtıdır.

**Çözümü:** Zincirlerden birini kır.

```csharp
// Urun → SiparisDetay yolunu Restrict yap; zaten mantıklı olan da bu
modelBuilder.Entity<SiparisDetay>()
    .HasOne(sd => sd.Urun)
    .WithMany(u => u.SiparisDetaylari)
    .HasForeignKey(sd => sd.UrunId)
    .OnDelete(DeleteBehavior.Restrict);
```

Modelde çok sayıda ilişki varsa, cascade'i topluca kapatıp ihtiyaç olan yerlerde açmak yaygın bir yöntemdir:

```csharp
foreach (var fk in modelBuilder.Model.GetEntityTypes()
                                     .SelectMany(e => e.GetForeignKeys()))
{
    fk.DeleteBehavior = DeleteBehavior.Restrict;
}
```

> **Uyarı:** Bu döngüyü `OnModelCreating`'in **sonunda** çalıştır. Başında çalıştırırsan, sonradan yazdığın `OnDelete(DeleteBehavior.Cascade)` çağrıları üstüne yazar ve döngünün bir etkisi kalmaz.

> **Bu benzetme şurada bozulur:** Kargo benzetmesinde şube kapandığında paketler fiziksel olarak oradadır; kararı vermemek diye bir seçenek yoktur. Veritabanında ise `ClientSetNull` gibi bir ara hâl var: EF Core bellekte bir şey yapar, veritabanında hiçbir kısıt kurulmaz. Uygulaman dışından (SSMS'ten, başka bir servisten) silme yapılırsa o kararın hiçbir hükmü kalmaz. Veri bütünlüğünü uygulamaya değil veritabanına yazdırmak her zaman daha güvenlidir.

---

## 8. Gölge Özellik ve Owned Entity

> **Benzetme —** Dilekçeni verdiğinde memur kâğıdın arkasına bir tarih damgası basar. O damga senin yazdığın metnin parçası değildir, dilekçende görünmez — ama dosyada durur ve gerektiğinde okunur. Gölge özellik budur. Owned entity ise başka bir şeydir: kimlik kartının üstündeki adres satırıdır. Ayrı bir belge değildir, kartın kendi parçasıdır; kartı alırsan adres de gelir, kartı yırtarsan adres de gider.

**Basitçe:** Gölge özellik, C# sınıfında olmayan ama veritabanında bulunan kolondur. Owned entity ise ayrı bir sınıf olarak yazdığın ama kendi tablosu olmayan, sahibinin tablosuna gömülen parçadır.

### 8.1 Shadow property (gölge özellik)

**Teknik olarak:** **Shadow property (gölge özellik)** — Varlık sınıfında karşılığı olmayan, yalnızca EF Core modelinde tanımlı kolon. Değerine `EF.Property<T>()` ile erişilir.

```csharp
modelBuilder.Entity<Siparis>().Property<DateTime>("OlusturmaTarihi");
modelBuilder.Entity<Siparis>().Property<string>("OlusturanKullanici").HasMaxLength(50);
```

Okuma ve yazma:

```csharp
// Yazma
var siparis = new Siparis { MusteriId = 3, Tarih = DateTime.Now };
_db.Siparisler.Add(siparis);
_db.Entry(siparis).Property("OlusturmaTarihi").CurrentValue = DateTime.UtcNow;
await _db.SaveChangesAsync();

// Sorguda kullanma
var yeniler = await _db.Siparisler
    .Where(s => EF.Property<DateTime>(s, "OlusturmaTarihi") > DateTime.UtcNow.AddDays(-7))
    .OrderBy(s => EF.Property<DateTime>(s, "OlusturmaTarihi"))
    .ToListAsync();
```

Denetim (audit) alanlarını `SaveChanges` içinde topluca doldurmak yaygın bir kalıptır:

```csharp
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries()
        .Where(e => e.State is EntityState.Added or EntityState.Modified))
    {
        if (entry.Metadata.FindProperty("OlusturmaTarihi") is not null &&
            entry.State == EntityState.Added)
        {
            entry.Property("OlusturmaTarihi").CurrentValue = DateTime.UtcNow;
        }
    }
    return base.SaveChanges();
}
```

En sık karşılaşacağın gölge özellik, **sen yazmadığın FK kolonudur**. 1.2'de gördüğün gibi, navigation yazıp FK özelliği yazmazsan EF Core kolonu gölge olarak üretir.

### 8.2 Owned entity (sahipli varlık) / value object

**Teknik olarak:** **Owned entity type** — Kendi kimliği olmayan, yalnızca sahibi (owner) üzerinden var olan varlık tipi. Varsayılan olarak sahibinin tablosuna gömülür.

```csharp
public class Adres                              // kendi Id'si YOK
{
    public string Il { get; set; } = null!;
    public string Ilce { get; set; } = null!;
    public string AcikAdres { get; set; } = null!;
    public string? PostaKodu { get; set; }
}

public class Musteri
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public Adres FaturaAdresi { get; set; } = null!;
    public Adres? TeslimatAdresi { get; set; }
}
```

```csharp
modelBuilder.Entity<Musteri>().OwnsOne(m => m.FaturaAdresi, a =>
{
    a.Property(p => p.Il).HasColumnName("FaturaIl").HasMaxLength(50);
    a.Property(p => p.Ilce).HasColumnName("FaturaIlce").HasMaxLength(50);
    a.Property(p => p.AcikAdres).HasColumnName("FaturaAcikAdres").HasMaxLength(500);
    a.Property(p => p.PostaKodu).HasColumnName("FaturaPostaKodu").HasMaxLength(10);
});

modelBuilder.Entity<Musteri>().OwnsOne(m => m.TeslimatAdresi, a =>
{
    a.Property(p => p.Il).HasColumnName("TeslimatIl").HasMaxLength(50);
    a.Property(p => p.Ilce).HasColumnName("TeslimatIlce").HasMaxLength(50);
    a.Property(p => p.AcikAdres).HasColumnName("TeslimatAcikAdres").HasMaxLength(500);
    a.Property(p => p.PostaKodu).HasColumnName("TeslimatPostaKodu").HasMaxLength(10);
});
```

Tek tablo, iki adres, ayrı kolonlar:

```sql
CREATE TABLE [Musteriler] (
    [Id] int NOT NULL IDENTITY,
    [Ad] nvarchar(100) NOT NULL,
    [FaturaIl] nvarchar(50) NOT NULL,
    [FaturaIlce] nvarchar(50) NOT NULL,
    [FaturaAcikAdres] nvarchar(500) NOT NULL,
    [FaturaPostaKodu] nvarchar(10) NULL,
    [TeslimatIl] nvarchar(50) NULL,
    ...
);
```

İstersen ayrı tabloya da alabilirsin:

```csharp
modelBuilder.Entity<Musteri>().OwnsOne(m => m.FaturaAdresi,
    a => a.ToTable("MusteriFaturaAdresleri"));
```

Koleksiyon hâli için `OwnsMany` vardır — o zaman ayrı tablo zorunludur:

```csharp
modelBuilder.Entity<Musteri>().OwnsMany(m => m.Adresler, a =>
{
    a.ToTable("MusteriAdresleri");
    a.WithOwner().HasForeignKey("MusteriId");
    a.Property<int>("Id");
    a.HasKey("Id");
});
```

| Owned entity | Normal ilişkili varlık |
|---|---|
| Kendi `Id`'si yok | `Id`'si var |
| `DbSet` tanımlanmaz | `DbSet` olur |
| Sahibiyle birlikte otomatik yüklenir — `Include` gerekmez | `Include` gerekir |
| Sahibi silinince gider | `OnDelete` kararına bağlı |
| Tek başına sorgulanamaz | Sorgulanır |

> **Bu benzetme şurada bozulur:** Kimlik kartındaki adres kartın parçasıdır ama sen onu kalemle değiştirebilirsin. Owned entity'de ise `musteri.FaturaAdresi = null` yazmak `SaveChanges` sırasında beklediğin şeyi yapmayabilir; EF Core sahipli bir referansı `null` yapmayı "gerekli parça kayboldu" olarak yorumlar. Zorunlu owned entity'leri `null` yapmak yerine yeni bir `Adres` nesnesi ata.

---

## 9. Kalıtım Eşlemesi: TPH, TPT, TPC

> **Benzetme —** Nüfus müdürlüğü kayıt tutuyor. Üç yöntem var. Birincisi: tek büyük defter, herkes aynı deftere yazılır, yanına "vatandaş / yabancı / geçici koruma" diye bir sütun konur — o sütun kimin ne olduğunu söyler. İkincisi: bir ana defterde herkesin ortak bilgisi, her tür için ayrı ek defterde o türe özel bilgiler; tam kayıt için iki deftere birden bakarsın. Üçüncüsü: her tür için tamamen ayrı, kendi başına tam defter; ama "tüm kayıtlar" istediğinde üç defteri birleştirmek zorundasın.

**Basitçe:** C#'ta bir taban sınıf ve ondan türeyen sınıflar yazdın. Veritabanında tablo kalıtımı diye bir şey yok. EF Core bunu üç farklı şekilde tabloya dökebilir.

**Teknik olarak:** Üç strateji vardır.

Örnek model:

```csharp
public abstract class Odeme
{
    public int Id { get; set; }
    public int SiparisId { get; set; }
    public decimal Tutar { get; set; }
    public DateTime Tarih { get; set; }
}

public class KrediKartiOdemesi : Odeme
{
    public string KartSonDortHane { get; set; } = null!;
    public int TaksitSayisi { get; set; }
}

public class HavaleOdemesi : Odeme
{
    public string Iban { get; set; } = null!;
    public string BankaAdi { get; set; } = null!;
}
```

### 9.1 TPH — Table Per Hierarchy (varsayılan)

Tek tablo, tüm alanlar, bir **discriminator** kolonu.

```csharp
modelBuilder.Entity<Odeme>()
    .HasDiscriminator<string>("OdemeTipi")
    .HasValue<KrediKartiOdemesi>("KrediKarti")
    .HasValue<HavaleOdemesi>("Havale");
```

```sql
CREATE TABLE [Odemeler] (
    [Id] int NOT NULL IDENTITY,
    [SiparisId] int NOT NULL,
    [Tutar] decimal(18,2) NOT NULL,
    [Tarih] datetime2 NOT NULL,
    [OdemeTipi] nvarchar(max) NOT NULL,      -- discriminator
    [KartSonDortHane] nvarchar(4) NULL,      -- sadece kredi kartında dolu
    [TaksitSayisi] int NULL,
    [Iban] nvarchar(34) NULL,                -- sadece havalede dolu
    [BankaAdi] nvarchar(100) NULL,
    CONSTRAINT [PK_Odemeler] PRIMARY KEY ([Id])
);
```

Türe özel kolonlar **zorunlu olamaz** — çünkü diğer türün satırlarında boş kalacaklar. Bu, TPH'nin en büyük bedelidir.

Discriminator'ı yapılandırmazsan EF Core `Discriminator` adında bir `nvarchar` kolon üretir ve değer olarak sınıf adını yazar. Sınıf adını sonradan değiştirirsen mevcut veriler eşleşmez; bu yüzden değeri açıkça yazmak iyi alışkanlıktır.

### 9.2 TPT — Table Per Type

```csharp
modelBuilder.Entity<Odeme>().UseTptMappingStrategy();
// veya
modelBuilder.Entity<Odeme>().ToTable("Odemeler");
modelBuilder.Entity<KrediKartiOdemesi>().ToTable("KrediKartiOdemeleri");
modelBuilder.Entity<HavaleOdemesi>().ToTable("HavaleOdemeleri");
```

Ortak alanlar `Odemeler`'de, türe özel alanlar kendi tablosunda. Alt tablonun PK'sı aynı zamanda üst tabloya FK'dır. Tam kaydı okumak `JOIN` gerektirir.

### 9.3 TPC — Table Per Concrete Type (EF Core 7+)

```csharp
modelBuilder.Entity<Odeme>().UseTpcMappingStrategy();
```

Her somut tür için kendi başına tam tablo. Taban sınıf için tablo yoktur. `_db.Odemeler.ToList()` çağrısı `UNION ALL` üretir.

> **Uyarı:** TPC'de `IDENTITY` kullanamazsın. İki ayrı tablo bağımsız artarsa `Id` çakışır ve taban tip üzerinden sorgu yapınca aynı `Id`'den iki kayıt görürsün. Çözüm `HiLo` ya da paylaşılan bir `SEQUENCE`'tır: `modelBuilder.HasSequence<int>("OdemeIds")` ve `.UseHiLo("OdemeIds")`.

### 9.4 Karşılaştırma

| | TPH | TPT | TPC |
|---|---|---|---|
| Tablo sayısı | 1 | 1 + tür sayısı | Somut tür sayısı |
| Discriminator | Var | Yok | Yok |
| Türe özel alan `NOT NULL` olabilir mi | Hayır | Evet | Evet |
| Tek tür sorgusu | Hızlı (`WHERE` filtresi) | `JOIN` gerekir | En hızlı |
| Taban tip sorgusu | En hızlı | Çoklu `JOIN` | `UNION ALL` |
| Ekleme performansı | En iyi | İki `INSERT` | İyi |
| Normalizasyon | Kötü — boş kolon çok | İyi | Orta — kolonlar tekrarlanır |
| `IDENTITY` kullanılabilir mi | Evet | Evet | Hayır |
| EF Core sürümü | Hep | Hep | 7+ |
| Varsayılan | Evet | Hayır | Hayır |

**Pratik kural:** Türler birbirine yakınsa ve türe özel alan azsa TPH kullan — varsayılan olmasının sebebi budur ve çoğu senaryoda doğru seçimdir. Türlerin alan kümeleri çok farklıysa ve bu alanların zorunlu olması gerekiyorsa TPC'ye bak. TPT en normalize görünendir ama sorgu maliyeti en yüksek olandır; performans ölçmeden seçme.

> **Bu benzetme şurada bozulur:** Nüfus defterinde "yabancı" sütununu boş bırakmak kâğıtta yer kaplamaz. TPH'de ise her satır her kolonu taşır. Bir milyon havale kaydı, kredi kartına ait `TaksitSayisi` kolonunu da `NULL` olarak taşır. Kolon sayısı arttıkça bu maliyet gerçek olur; on beş alt tipi olan bir hiyerarşide TPH artık iyi fikir değildir.

---

## 10. Index, Unique Constraint ve Global Query Filter

> **Benzetme —** Kütüphanede fihrist kutusu vardır: kitabın nerede olduğunu raflar arasında dolaşmadan söyler. Index budur. Global query filter ise başka bir şeydir — kütüphaneye girerken taktığın renkli gözlüktür. O gözlükle bakarken bazı kitaplar hiç görünmez. Sen "kütüphanede o kitap yok" sanırsın; aslında raftadır, sen göremiyorsundur.

**Basitçe:** Index, arama hızlandırıcıdır. Unique constraint, "bu değer iki kere olmasın" kuralıdır. Global query filter, o tabloya yapılan her sorguya otomatik eklenen bir `WHERE` şartıdır.

### 10.1 Index tanımlama

```csharp
// Tek kolon
modelBuilder.Entity<Musteri>().HasIndex(m => m.Telefon);

// Benzersiz
modelBuilder.Entity<Musteri>()
    .HasIndex(m => m.Email)
    .IsUnique()
    .HasDatabaseName("UX_Musteri_Email");

// Birleşik (kolon sırası önemlidir)
modelBuilder.Entity<Siparis>()
    .HasIndex(s => new { s.MusteriId, s.Tarih });

// Filtered index — sadece aktif kayıtlar indekslenir
modelBuilder.Entity<Urun>()
    .HasIndex(u => u.Barkod)
    .IsUnique()
    .HasFilter("[Silindi] = 0");

// Dahil edilen kolonlar (covering index)
modelBuilder.Entity<Siparis>()
    .HasIndex(s => s.Tarih)
    .IncludeProperties(s => new { s.ToplamTutar, s.MusteriId });
```

Data Annotations karşılığı (EF Core 5+, **sınıf üstünde** yazılır):

```csharp
[Index(nameof(Email), IsUnique = true)]
[Index(nameof(MusteriId), nameof(Tarih), Name = "IX_Musteri_Tarih")]
public class Musteri { ... }
```

> **Uyarı:** Birleşik index'te kolon sırası sorgu planını belirler. `(MusteriId, Tarih)` index'i "şu müşterinin şu tarihteki siparişleri" sorgusunda kullanılır; "şu tarihteki tüm siparişler" sorgusunda kullanılamaz. Sıralamayı en seçici kolondan değil, `WHERE`'de en sık eşitlikle kullanılan kolondan başlat.

> **Uyarı:** `nvarchar(max)` kolona index koyamazsın. Bu yüzden benzersiz olacak metin alanlarına mutlaka `HasMaxLength` ver. `Email` alanına 256, `Barkod` alanına 32 gibi.

Benzersizliği iki kolon üzerinden kurmak, çoka-çok ara tablolarında sık gereken bir kısıttır:

```csharp
modelBuilder.Entity<UrunKategori>()
    .HasIndex(uk => new { uk.UrunId, uk.KategoriId })
    .IsUnique();
```

### 10.2 Global query filter

**Teknik olarak:** **Global query filter** — Bir varlık tipine yapılan tüm LINQ sorgularına otomatik eklenen `WHERE` koşulu. `HasQueryFilter` ile tanımlanır.

En yaygın kullanımı soft delete'tir:

```csharp
public class Urun
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public bool Silindi { get; set; }
}

modelBuilder.Entity<Urun>().HasQueryFilter(u => !u.Silindi);
```

Bundan sonra yazdığın her sorguya şart otomatik eklenir:

```csharp
var urunler = await _db.Urunler.ToListAsync();
// Üretilen SQL: SELECT ... FROM [Urunler] WHERE [Silindi] = CAST(0 AS bit)

var urun = await _db.Urunler.FirstOrDefaultAsync(u => u.Id == 5);
// Silinmiş bir kaydın Id'si olsa bile null döner
```

Çok kiracılı (multi-tenant) uygulamalarda ikinci yaygın kullanım budur:

```csharp
public class AppDbContext : DbContext
{
    private readonly int _firmaId;
    public AppDbContext(DbContextOptions<AppDbContext> opt, ICurrentUser user) : base(opt)
        => _firmaId = user.FirmaId;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Siparis>().HasQueryFilter(s => s.FirmaId == _firmaId);
    }
}
```

Filtreyi bilinçli olarak devre dışı bırakmak için:

```csharp
var silinmisDahilHepsi = await _db.Urunler
    .IgnoreQueryFilters()
    .ToListAsync();
```

### 10.3 Global query filter'ın tuzakları

| Tuzak | Açıklama |
|---|---|
| `IgnoreQueryFilters()` **hepsini** kapatır | Tek bir filtreyi seçip kapatamazsın; tenant filtresi de kalkar |
| `Include` edilen navigation'a da uygulanır | Silinmiş kategorisi olan ürünün `Kategori`'si `null` gelir |
| Zorunlu navigation'da tutarsızlık | `Urun.Kategori` zorunluyken kategori filtrelenmişse EF Core uyarı verir |
| `Find()` önbellekten dönerse filtre uygulanmaz | Change tracker'da duran silinmiş kayıt geri gelebilir |
| Ham SQL'e uygulanmaz | `FromSql` ile yazdığın sorguya şart eklenmez |
| Türetilmiş tipe ayrı filtre yazılamaz | Filtre yalnızca kök varlık tipine tanımlanır |

> **Uyarı:** En tehlikelisi `IgnoreQueryFilters()`'ın toplu davranışıdır. "Silinmiş ürünleri de görmek istiyorum" diye yazdığın satır, çok kiracılı bir uygulamada başka firmanın verisini de açar. Filtreleri ayırmak istiyorsan soft delete'i filtre yerine sorguda açık `Where` ile yönetmeyi düşün.

```csharp
// Tehlikeli: her iki filtre de kalkar
var hepsi = await _db.Siparisler.IgnoreQueryFilters().ToListAsync();

// Güvenli: tenant filtresi korunur, silinmişleri elle katarsın
var hepsi2 = await _db.Siparisler
    .IgnoreQueryFilters()
    .Where(s => s.FirmaId == _firmaId)
    .ToListAsync();
```

> **Bu benzetme şurada bozulur:** Renkli gözlüğü çıkardığında kütüphanenin tamamını görürsün ve bu iyidir. `IgnoreQueryFilters()` ise sadece senin göremediğin kitapları değil, görmeye **yetkin olmayan** kitapları da açar. Gözlük iki işi birden yapıyordu: gizleme ve koruma. Çıkardığında ikisi birden gider.

---

## 11. Concurrency Token ve Value Converter

> **Benzetme —** Banka kuyruğunda aynı hesabın bakiyesini iki memur aynı anda düzenliyor. İkisi de ekranda 1000 TL görüyor. Biri 200 yatırıyor, diğeri 300 çekiyor. İkisi de "benim gördüğüm 1000'di" diye kaydederse, son kaydeden diğerinin işlemini silmiş olur. Çözüm, kaydederken "ben 1000 görmüştüm, hâlâ 1000 mü?" diye sormaktır. Değiştiyse işlemi reddedersin.

**Basitçe:** İki kullanıcı aynı kaydı aynı anda düzenlerse, ikincisinin kaydı birincinin değişikliğini sessizce ezer. Concurrency token bunu yakalar. Value converter ise ayrı bir konu: C#'taki tipi veritabanında başka bir tiple saklamanı sağlar.

### 11.1 Optimistic concurrency

**Teknik olarak:** **Optimistic concurrency (iyimser eşzamanlılık)** — Kaydı kilitlemek yerine, güncellerken "okuduğumdan beri değişmiş mi" diye kontrol eden yaklaşım.

SQL Server'da en pratik yol `rowversion` kolonudur. Her `UPDATE`'te veritabanı bu değeri kendisi artırır.

```csharp
public class Urun
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public decimal Fiyat { get; set; }
    public int StokAdedi { get; set; }

    [Timestamp]                                  // Data Annotations yolu
    public byte[] RowVersion { get; set; } = null!;
}

// Fluent API yolu
modelBuilder.Entity<Urun>()
    .Property(u => u.RowVersion)
    .IsRowVersion();
```

EF Core, `UPDATE` cümlesine bu kolonu `WHERE`'e ekler:

```sql
UPDATE [Urunler] SET [Fiyat] = @p0
WHERE [Id] = @p1 AND [RowVersion] = @p2;
SELECT @@ROWCOUNT;
```

Etkilenen satır sayısı 0 dönerse, kayıt araya girip değişmiştir. EF Core bunu `DbUpdateConcurrencyException` ile bildirir.

```csharp
try
{
    urun.Fiyat = 150m;
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var guncelDegerler = await entry.GetDatabaseValuesAsync();

        if (guncelDegerler is null)
        {
            // Kayıt başkası tarafından silinmiş
            ModelState.AddModelError("", "Bu ürün başka bir kullanıcı tarafından silindi.");
        }
        else
        {
            var dbUrun = (Urun)guncelDegerler.ToObject();
            ModelState.AddModelError("",
                $"Kayıt değişti. Veritabanındaki fiyat: {dbUrun.Fiyat}");

            // "Veritabanı kazansın" stratejisi:
            entry.OriginalValues.SetValues(guncelDegerler);
        }
    }
}
```

Üç çözüm stratejisi vardır:

| Strateji | Ne yapılır |
|---|---|
| Client wins (istemci kazanır) | `entry.OriginalValues.SetValues(dbDegerler)` → tekrar kaydet |
| Store wins (veritabanı kazanır) | `entry.Reload()` → kullanıcının değişikliği atılır |
| Kullanıcıya sor | İki değeri de ekranda göster, seçtir — en doğrusu budur |

`rowversion` kullanmak istemiyorsan tek bir alanı token yapabilirsin:

```csharp
modelBuilder.Entity<Urun>()
    .Property(u => u.StokAdedi)
    .IsConcurrencyToken();
```

Bu durumda `WHERE` şartına `StokAdedi` eklenir. Yalnız o alan değişirse çakışma algılanır; diğer alanlar korunmaz.

> **Uyarı:** `rowversion` kolonu web formlarında gidip gelmelidir. Aksi hâlde POST'a gelen nesnenin `RowVersion`'ı boş olur ve kontrol anlamsızlaşır. Gizli alana koy ve `SetOriginalValue` ile ata:

```csharp
_db.Entry(urun).Property(u => u.RowVersion).OriginalValue = formdanGelenRowVersion;
```

### 11.2 Value converter

**Teknik olarak:** **Value converter (değer dönüştürücü)** — Bir özelliğin C# tipiyle veritabanındaki tipi arasında iki yönlü dönüşüm tanımlar.

Enum'u sayı yerine metin olarak saklamak en yaygın kullanımdır:

```csharp
public enum SiparisDurumu { Beklemede, Hazirlaniyor, Kargoda, TeslimEdildi, Iptal }

public class Siparis
{
    public int Id { get; set; }
    public SiparisDurumu Durum { get; set; }
}

// Kısa yol — enum adını string olarak saklar
modelBuilder.Entity<Siparis>()
    .Property(s => s.Durum)
    .HasConversion<string>()
    .HasMaxLength(20);
```

Veritabanında `2` yerine `Kargoda` yazar. Raporlama yapan biri tabloyu açtığında ne olduğunu anlar; enum'a yeni bir değer eklediğinde eski kayıtların anlamı kaymaz.

Açık dönüşüm yazımı:

```csharp
modelBuilder.Entity<Siparis>()
    .Property(s => s.Durum)
    .HasConversion(
        durum => durum.ToString(),
        metin => (SiparisDurumu)Enum.Parse(typeof(SiparisDurumu), metin));
```

Başka örnekler:

```csharp
// bool → 'E' / 'H'
modelBuilder.Entity<Urun>()
    .Property(u => u.Aktif)
    .HasConversion(v => v ? "E" : "H", v => v == "E")
    .HasMaxLength(1);

// DateTime'ı her zaman UTC olarak okumak
modelBuilder.Entity<Siparis>()
    .Property(s => s.Tarih)
    .HasConversion(
        v => v,
        v => DateTime.SpecifyKind(v, DateTimeKind.Utc));

// Hazır dönüştürücü sınıfları
modelBuilder.Entity<Siparis>()
    .Property(s => s.Durum)
    .HasConversion(new EnumToStringConverter<SiparisDurumu>());
```

EF Core 6+ ile tüm enum'lara tek seferde uygulayabilirsin:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder builder)
{
    builder.Properties<SiparisDurumu>().HaveConversion<string>();
}
```

> **Uyarı:** Dönüştürülen kolon üzerinde yapılan karşılaştırmalar veritabanında **dönüştürülmüş hâliyle** çalışır. `Where(s => s.Durum > SiparisDurumu.Beklemede)` yazarsan, `string` olarak saklanan kolonda alfabetik karşılaştırma yapılır — beklediğin sonucu vermez. Dönüştürülmüş alanlarda sadece eşitlik ve `Contains` güvenlidir.

> **Uyarı:** Bir koleksiyonu ya da nesneyi JSON'a çevirip saklarsan, EF Core değişiklik takibini yapamaz — nesnenin içi değişse bile fark edemez. Bu durumda `ValueComparer` da tanımlaman gerekir:

```csharp
modelBuilder.Entity<Urun>()
    .Property(u => u.Etiketler)
    .HasConversion(
        v => string.Join(',', v),
        v => v.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList(),
        new ValueComparer<List<string>>(
            (a, b) => a!.SequenceEqual(b!),
            v => v.Aggregate(0, (h, s) => HashCode.Combine(h, s.GetHashCode())),
            v => v.ToList()));
```

> **Bu benzetme şurada bozulur:** Bankada memur "bakiye değişmiş" dediğinde işlemi iptal eder ve kimse veri kaybetmez. EF Core'da ise `DbUpdateConcurrencyException` yakalanmazsa istek 500 ile düşer ve kullanıcı formda yazdığı her şeyi kaybeder. Token'ı eklemek işin yarısıdır; ikinci yarısı çakışmayı kullanıcıya anlaşılır biçimde göstermektir.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Convention | EF Core'un açık konfigürasyon yokken uyguladığı varsayılan kural |
| Data Annotations | Sınıf ve özelliklerin üstüne yazılan köşeli parantezli konfigürasyon etiketleri |
| Fluent API | `OnModelCreating` içinde zincirleme metotlarla yapılan konfigürasyon |
| `IEntityTypeConfiguration<T>` | Tek bir varlığın konfigürasyonunu taşıyan ayrı sınıf |
| Navigation property | Bir varlıktan ilişkili varlığa giden özellik |
| Principal (asıl) | İlişkide anahtarı referans edilen taraf |
| Dependent (bağımlı) | İlişkide yabancı anahtarı taşıyan taraf |
| Foreign key (yabancı anahtar) | Bağımlı tarafta duran, asıl tarafın anahtarını işaret eden kolon |
| Join entity | Çoka-çok ilişkideki ara tabloyu temsil eden varlık |
| `DeleteBehavior` | Asıl kayıt silindiğinde bağımlılara ne olacağını belirleyen enum |
| Cascade path | Silmenin zincirleme yayıldığı ilişki yolu |
| Shadow property | Sınıfta karşılığı olmayan, sadece modelde tanımlı kolon |
| Owned entity | Kendi kimliği olmayan, sahibinin tablosuna gömülen varlık tipi |
| Value object | Kimliği değil değeri önemli olan, davranışsız veri nesnesi |
| TPH / TPT / TPC | Kalıtımın tabloya üç farklı dökülme stratejisi |
| Discriminator | TPH'de satırın hangi alt tipe ait olduğunu söyleyen kolon |
| Global query filter | Bir varlığa yapılan tüm sorgulara otomatik eklenen `WHERE` şartı |
| Soft delete | Kaydı silmek yerine bayrakla gizleme |
| Concurrency token | Güncellemede "okuduğumdan beri değişti mi" kontrolünde kullanılan kolon |
| Optimistic concurrency | Kilitlemeden, çakışmayı kayıt anında yakalayan yaklaşım |
| `rowversion` | SQL Server'ın her `UPDATE`'te otomatik artırdığı ikili kolon |
| Value converter | C# tipi ile veritabanı tipi arasında iki yönlü dönüşüm tanımı |
| `ValueComparer` | Dönüştürülmüş değerin değişip değişmediğini anlamak için gereken karşılaştırıcı |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `06-EF-Core-Performans.md` (Cumartesi)
