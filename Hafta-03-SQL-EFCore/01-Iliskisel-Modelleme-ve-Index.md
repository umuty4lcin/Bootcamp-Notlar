# Hafta 3 · Pazartesi — İlişkisel Modelleme ve Index

**Okuma süresi:** ~55 dk
**Neden bu konu:** Bir web uygulamasının performans sorunlarının büyük çoğunluğu C# tarafında değil, veritabanı tarafında doğar. Yanlış kurulmuş bir tablo yapısını sonradan düzeltmek, yanlış yazılmış bir metodu düzeltmekten kat kat pahalıdır. MvcCv'de EF Core'un ürettiği tabloları kullandın; bu hafta o tabloların neden öyle olduğunu ve nerede yanlış olabileceğini göreceksin. Bootcamp'te ORM'e geçmeden önce altındaki modeli bilmen isteniyor.

---

## Önce Basitçe

Bir veritabanı, aslında düzenli tutulmuş defterler topluluğudur. Her defterin bir konusu vardır: biri müşterileri yazar, biri siparişleri, biri ürünleri. Her defterin sayfalarında satırlar vardır ve her satır tek bir kaydı anlatır. Sütunlar ise o kaydın hangi bilgilerinin tutulacağını belirler. Bu kadar basit başlar.

İşin zor tarafı defterler arasındaki bağdır. Sipariş defterine müşterinin adını, adresini, telefonunu tekrar tekrar yazarsan, müşteri taşındığında yüz satırı tek tek düzeltmen gerekir; birini atlarsan defterinde iki farklı adres olur. İlişkisel model bunu şöyle çözer: müşteriyi bir kez yaz, ona bir numara ver, sipariş defterine sadece o numarayı düş.

Bu "tek yerde dursun" fikrinin adı normalizasyondur. Kuralları korkutucu isimler taşır — birinci normal form, ikinci normal form — ama söyledikleri sağduyuludur: bir hücreye birden fazla şey tıkıştırma, bir satırda konusu o satır olmayan bilgi tutma, bilgiyi iki yerde tekrarlama.

İkinci büyük konu hız. Yüz satırlık bir defterde aradığını baştan sona okuyarak bulursun; on milyon satırlıkta bu mümkün değildir. Kütüphanedeki kitapları düşün: rafları tek tek gezmek yerine fihristten bakarsın. Index tam olarak o fihristtir. Ama her fihristin bir bedeli vardır — kitap eklediğinde fihristi de güncellemen gerekir. Bu yüzden index koymak her zaman iyi bir fikir değildir.

Üçüncüsü, index'i koymak yetmez; sorgunun onu kullanabilmesi gerekir. Fihrist yazar adına göre dizilmişse "adı içinde 'mehmet' geçen yazarlar" diye ararsan işe yaramaz. SQL'de de bir sütunu fonksiyonun içine soktuğun anda o sütunun index'i devre dışı kalır.

Notta önce modeli kuracağız: tablo, anahtar, ilişki, normalizasyon, veri tipi, kısıt. Sonra hıza geçeceğiz: index nasıl çalışır, ne zaman konmaz. Şimdi detaya iniyoruz.

> **Ana benzetme:** Veritabanı bir **tapu dairesidir**. Her kayıt tek bir defterde, tek bir yerde durur; başka defterler ona parsel numarasıyla atıf yapar. Kimse aynı tapuyu iki deftere yazmaz — yazsa hangisinin geçerli olduğu tartışma konusu olur. Dairenin girişindeki fihrist ise index'tir: parsel numarasını bilirsen dosyaya saniyede ulaşırsın, bilmezsen arşivi baştan sona taramak zorunda kalırsın.

---

## Bu Notta Ne Var

1. İlişkisel model: tablo, satır, sütun, domain
2. Birincil anahtar (PK) ve anahtar seçimi
3. Yabancı anahtar (FK) ve referans bütünlüğü
4. İlişki türleri: 1-1, 1-N, N-N
5. Normalizasyon: 1NF, 2NF, 3NF
6. Denormalizasyon ne zaman meşrudur
7. Veri tipleri ve doğru seçim
8. NULL ve üç değerli mantık
9. Constraint'ler: UNIQUE, CHECK, DEFAULT
10. Index neden hızlandırır
11. Index tasarımı: composite, covering, maliyet
12. SARGability ve ölçme

---

## 1. İlişkisel Model: Tablo, Satır, Sütun, Domain

> **Benzetme —** Bir muhtarlığın arka odasındaki dolabı düşün. Her çekmecede tek konulu bir defter var: nüfus defteri, ikametgâh defteri, emlak defteri. Her defterin sayfalarında aynı başlıklar tekrar eder — ad, soyad, doğum yılı. Bir sayfada bir kişi vardır, iki kişi yazılmaz. Başlıkların altına yazılabilecek şeyler de bellidir: doğum yılı sütununa "Kayseri" yazamazsın. İlişkisel modelin söylediği şey tam olarak budur.

**Basitçe:** İlişkisel model, veriyi tablolarda tutar. Her tablo tek bir konuyu anlatır. Her satır o konudan bir örnektir. Her sütun o örneğin bir özelliğidir ve o sütuna ne tür değerler konabileceği önceden bellidir.

**Teknik olarak:** **İlişkisel model (relational model)** — Veriyi, satır ve sütunlardan oluşan **ilişkiler** (tablolar) hâlinde tutan, aralarındaki bağları değerler üzerinden kuran veri modeli. 1970'te E. F. Codd tanımladı; bugün SQL Server, PostgreSQL, MySQL, Oracle hepsi bu modeli uygular.

| Resmî terim | Günlük karşılığı |
|---|---|
| Relation (tablo) | Defter |
| Tuple (satır) | Sayfa / kayıt |
| Attribute (sütun) | Başlık |
| Domain | "Buraya ne yazılabilir" |
| Cardinality / Degree | Satır sayısı / sütun sayısı |

**Domain (değer alanı)** pratikte iki şeye dönüşür: **veri tipi** ve **kısıt**. `BirimFiyat DECIMAL(18,2)` "buraya sayı gelir", `CHECK (BirimFiyat >= 0)` "üstelik negatif olamaz" demektir.

Modelin üç temel kuralı vardır:

- **Satır sırası anlamsızdır.** `ORDER BY` yazmadığın bir `SELECT`'in dönüş sırasına asla güvenme.
- **Sütun sırası anlamsızdır.** `SELECT *` yerine sütun adı yazmanın sebeplerinden biri budur.
- **Her satır benzersiz olmalıdır.** Bunu sağlayan şey birincil anahtardır.

### Bu notun örnek şeması

Bu hafta boyunca aynı beş tabloyu kullanacağız. Küçük bir e-ticaret veritabanı:

```sql
-- Kategoriler: kendi kendine referans veren ağaç yapısı
CREATE TABLE Kategoriler (
    KategoriId    INT           IDENTITY(1,1) NOT NULL,
    Ad            NVARCHAR(100) NOT NULL,
    UstKategoriId INT           NULL,
    CONSTRAINT PK_Kategoriler PRIMARY KEY (KategoriId),
    CONSTRAINT FK_Kategoriler_Ust FOREIGN KEY (UstKategoriId)
        REFERENCES Kategoriler(KategoriId)
);

CREATE TABLE Urunler (
    UrunId     INT           IDENTITY(1,1) NOT NULL,
    Ad         NVARCHAR(200) NOT NULL,
    KategoriId INT           NOT NULL,
    BirimFiyat DECIMAL(18,2) NOT NULL,
    StokAdedi  INT           NOT NULL CONSTRAINT DF_Urunler_Stok  DEFAULT (0),
    Aktif      BIT           NOT NULL CONSTRAINT DF_Urunler_Aktif DEFAULT (1),
    CONSTRAINT PK_Urunler PRIMARY KEY (UrunId),
    CONSTRAINT FK_Urunler_Kategoriler FOREIGN KEY (KategoriId)
        REFERENCES Kategoriler(KategoriId),
    CONSTRAINT CK_Urunler_Fiyat CHECK (BirimFiyat >= 0)
);

CREATE TABLE Musteriler (
    MusteriId   INT           IDENTITY(1,1) NOT NULL,
    Ad          NVARCHAR(100) NOT NULL,
    Soyad       NVARCHAR(100) NOT NULL,
    Eposta      NVARCHAR(256) NOT NULL,
    Sehir       NVARCHAR(80)  NULL,
    KayitTarihi DATETIME2(3)  NOT NULL
        CONSTRAINT DF_Musteriler_Kayit DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_Musteriler PRIMARY KEY (MusteriId),
    CONSTRAINT UQ_Musteriler_Eposta UNIQUE (Eposta)
);

CREATE TABLE Siparisler (
    SiparisId     INT          IDENTITY(1,1) NOT NULL,
    MusteriId     INT          NOT NULL,
    SiparisTarihi DATETIME2(3) NOT NULL
        CONSTRAINT DF_Siparisler_Tarih DEFAULT (SYSUTCDATETIME()),
    Durum         TINYINT      NOT NULL
        CONSTRAINT DF_Siparisler_Durum DEFAULT (0),   -- 0:Yeni 1:Hazir 2:Kargo 3:Teslim 9:Iptal
    CONSTRAINT PK_Siparisler PRIMARY KEY (SiparisId),
    CONSTRAINT FK_Siparisler_Musteriler FOREIGN KEY (MusteriId)
        REFERENCES Musteriler(MusteriId)
);

CREATE TABLE SiparisDetaylari (
    SiparisDetayId BIGINT       IDENTITY(1,1) NOT NULL,
    SiparisId      INT          NOT NULL,
    UrunId         INT          NOT NULL,
    Adet           INT          NOT NULL,
    BirimFiyat     DECIMAL(18,2) NOT NULL,
    IskontoOrani   DECIMAL(5,4) NOT NULL
        CONSTRAINT DF_SD_Iskonto DEFAULT (0),
    CONSTRAINT PK_SiparisDetaylari PRIMARY KEY (SiparisDetayId),
    CONSTRAINT FK_SD_Siparisler FOREIGN KEY (SiparisId)
        REFERENCES Siparisler(SiparisId) ON DELETE CASCADE,
    CONSTRAINT FK_SD_Urunler FOREIGN KEY (UrunId)
        REFERENCES Urunler(UrunId),
    CONSTRAINT UQ_SD_Siparis_Urun UNIQUE (SiparisId, UrunId),
    CONSTRAINT CK_SD_Adet CHECK (Adet > 0)
);
```

Şimdilik dikkat edilecek üç şey var: `SiparisDetaylari` tablosunda `BirimFiyat` tekrar yazılmış (sebebi 6. bölümde), `Kategoriler` kendine referans veriyor (4. bölüm) ve `CASCADE` yalnızca tek bir FK'da (3. bölüm).

> **Bu benzetme şurada bozulur:** Muhtarlık defterinde sayfalar fiziksel olarak sıralıdır — 12. sayfa 11'den sonra gelir. Tabloda böyle bir garanti yoktur. SQL Server verileri diskte clustered index'in sırasına göre tutar ama sorgu sonucunda bu sırayı korumak zorunda değildir. Paralel çalışan bir plan satırları karışık döndürebilir. Sıra istiyorsan `ORDER BY` yazacaksın.

---

## 2. Birincil Anahtar (PK) ve Anahtar Seçimi

> **Benzetme —** Nüfus müdürlüğünde bir kişiyi tarif etmenin iki yolu var. Ya "Kayseri'de oturan, 1994 doğumlu Ahmet Yılmaz" dersin — uzun, birden fazla kişiye uyabilir ve kişi taşınırsa tarif bozulur. Ya da TC kimlik numarasını verirsin: on bir hane, tek kişi, ömür boyu değişmez. Birincil anahtar ikinci yoldur. Kaydı tarif etmez, sadece işaret eder.

**Basitçe:** Birincil anahtar, bir satırı tek başına ve kesin olarak belirleyen sütundur. Aynı değer iki satırda olamaz, boş bırakılamaz ve değişmemesi beklenir.

**Teknik olarak:** **Primary key (birincil anahtar)** — Tablodaki her satırı benzersiz şekilde tanımlayan sütun veya sütun kümesi. SQL Server'da `PRIMARY KEY` tanımlamak otomatik olarak iki şey yapar: sütunu `NOT NULL` yapar ve varsayılan olarak **clustered** benzersiz bir index oluşturur.

```sql
CONSTRAINT PK_Musteriler   PRIMARY KEY (MusteriId)                -- tek sütunlu
CONSTRAINT PK_UrunEtiket   PRIMARY KEY (UrunId, EtiketId)         -- bileşik (composite)
CONSTRAINT PK_Loglar       PRIMARY KEY NONCLUSTERED (LogId)       -- clustered istemiyorsan
```

### Aday anahtar, doğal anahtar, yapay anahtar

| Kavram | Ne demek | Örnek |
|---|---|---|
| **Candidate key (aday anahtar)** | Satırı benzersiz belirleyebilen sütun kümesi | `MusteriId`, `Eposta` |
| **Primary key** | Aday anahtarlardan seçilmiş olan | `MusteriId` |
| **Alternate key** | Seçilmeyen aday anahtar — `UNIQUE` ile korunur | `Eposta` |
| **Natural key (doğal anahtar)** | Gerçek dünyadan gelen, anlamı olan anahtar | TC kimlik no, ISBN, plaka |
| **Surrogate key (yapay anahtar)** | Sırf kimliklendirmek için üretilmiş anlamsız değer | `IDENTITY`, GUID |

Doğal anahtar cazip görünür — zaten elinde olan bir değeri kullanırsın. Pratikte üç yerde kırılır.

- **Değişir.** Plaka değişir, e-posta değişir, vergi numarası şirket birleşmesinde değişir. PK değiştiği anda ona referans veren bütün FK değerlerini de değiştirmen gerekir.
- **Bazen yoktur.** Yabancı müşterinin TC kimlik numarası yoktur; anahtarı `NULL` bırakamazsın.
- **Geniştir.** `NVARCHAR(50)` bir PK her FK sütununda ve her index'te 50 karakter yer kaplar. `INT` 4 bayttır.

```sql
-- Kötü: doğal anahtar PK -> e-posta değişince ona bakan tüm FK'ları güncellemen gerekir
CREATE TABLE Musteriler_Kotu (
    Eposta NVARCHAR(256) NOT NULL PRIMARY KEY,
    Ad     NVARCHAR(100) NOT NULL
);

-- İyi: yapay anahtar PK, doğal anahtar UNIQUE ile korunur
CREATE TABLE Musteriler_Iyi (
    MusteriId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    Eposta    NVARCHAR(256) NOT NULL UNIQUE,
    Ad        NVARCHAR(100) NOT NULL
);
```

Kural şu: **PK yapay olsun, doğal anahtar `UNIQUE` kısıtıyla korunsun.**

### `IDENTITY` vs GUID

```sql
-- IDENTITY: veritabanı sayar
UrunId INT IDENTITY(1,1) NOT NULL       -- 1'den başla, 1'er artır

-- GUID: 16 baytlık benzersiz değer
UrunId UNIQUEIDENTIFIER NOT NULL DEFAULT (NEWID())
```

| Ölçüt | `IDENTITY` (int/bigint) | `UNIQUEIDENTIFIER` (GUID) |
|---|---|---|
| Boyut | 4 / 8 bayt | 16 bayt |
| Kim üretir | Veritabanı (INSERT anında) | Uygulama da üretebilir |
| Tahmin edilebilirlik | Yüksek — `/Urun/5` görünce `/Urun/6` denenir | Düşük |
| Clustered index davranışı | Artan — sona ekler, bölünme yok | `NEWID()` rastgele — sayfa bölünmesi yaratır |
| Birleştirme (merge/replikasyon) | Çakışır | Çakışmaz |
| Okunabilirlik | Kolay (`15`) | Zor (`a3f2...`) |

GUID'in asıl sorunu boyut değil **rastgeleliktir**. Artan bir `IDENTITY` her yeni satırı sonuncu sayfaya ekler. `NEWID()` ile üretilmiş GUID araya girer; sayfa doluysa SQL Server sayfayı ikiye böler (**page split**) — hem yazma maliyeti hem **fragmentation (parçalanma)**. Çözüm **sequential GUID**'dir: artan sırada GUID üreten yöntem.

```sql
-- SQL Server tarafında: NEWSEQUENTIALID() sadece DEFAULT içinde kullanılabilir
CREATE TABLE Faturalar (
    FaturaId UNIQUEIDENTIFIER NOT NULL
        CONSTRAINT DF_Faturalar_Id DEFAULT (NEWSEQUENTIALID()),
    CONSTRAINT PK_Faturalar PRIMARY KEY CLUSTERED (FaturaId)
);
```

```csharp
public class Fatura
{
    public Guid FaturaId { get; set; }   // EF Core INSERT anında sıralı GUID üretir
}

var id = Guid.CreateVersion7();   // .NET 9: zaman sıralı UUIDv7
```

> **.NET 9 notu:** `Guid.CreateVersion7()` zaman damgası öncelikli GUID üretir; sıralı olduğu için `NEWID()` gibi parçalanma yaratmaz. .NET 9 öncesinde EF Core'un `SequentialGuidValueGenerator` sınıfı aynı işi yapar.

**Seçim kuralı:** Tek veritabanı ve id'nin dışarı sızması sorun değilse `INT IDENTITY`. Dağıtık sistem, offline üretilen kayıt veya URL'de tahmin edilmesi sakıncalı id ise sequential GUID. "Her ihtimale karşı GUID" gerekçesi zayıftır; bedelini her index'te ödersin.

> **Bu benzetme şurada bozulur:** TC kimlik numarası hem benzersizdir hem de değişmez — ideal doğal anahtar gibi durur. Ama veritabanında onu PK yapmak yine de kötü fikirdir, çünkü **kişisel veridir**. FK olarak her tabloya yayılır, log'lara düşer, URL'de görünür. Yapay anahtar bu yayılmayı engeller: kimlik numarası tek tabloda, şifreli kolonda kalır.

---
## 3. Yabancı Anahtar ve Referans Bütünlüğü

> **Benzetme —** Apartman yönetiminde aidat defteri tutuyorsun. Her aidat kaydının yanına daire numarası yazıyorsun. Ama apartmanda 14 numaralı daire yok. Defterde duran o kayıt kimin? Kimse bilmiyor. Yabancı anahtar, "yazdığın daire numarası gerçekten var mı" diye kontrol eden yönetici kuralıdır. Olmayan daireye aidat yazdırmaz.

**Basitçe:** Yabancı anahtar, bir tablodaki sütunun başka bir tablodaki anahtara işaret ettiğini söyler. Veritabanı bu işaretin boşa çıkmasına izin vermez.

**Teknik olarak:** **Foreign key (yabancı anahtar)** — Bir tablonun sütun(lar)ının, başka bir tablonun PK veya UNIQUE anahtarına referans verdiğini bildiren kısıt. Sağladığı garantiye **referential integrity (referans bütünlüğü)** denir.

```sql
-- Siparisler.MusteriId, Musteriler.MusteriId'ye işaret eder
CONSTRAINT FK_Siparisler_Musteriler FOREIGN KEY (MusteriId)
    REFERENCES Musteriler(MusteriId)
```

FK iki yönde koruma sağlar:

```sql
INSERT INTO Siparisler (MusteriId) VALUES (99999);   -- olmayan müşteriye sipariş
DELETE FROM Musteriler WHERE MusteriId = 3;          -- siparişi olan müşteriyi silme
-- İkisi de: Msg 547 ... conflicted with the FOREIGN KEY constraint
```

### Silme ve güncelleme davranışları

`ON DELETE` / `ON UPDATE` ile ana kayıt silindiğinde/güncellendiğinde ne olacağını belirlersin.

| Davranış | Ne yapar | Ne zaman |
|---|---|---|
| `NO ACTION` (varsayılan) | Bağlı kayıt varsa işlemi **reddeder** | Neredeyse her yerde |
| `CASCADE` | Bağlı kayıtları da siler/günceller | Sadece gerçek sahiplik ilişkisinde |
| `SET NULL` | FK sütununu `NULL` yapar (sütun nullable olmalı) | "Kategorisiz ürün" gibi anlamlı durumlarda |
| `SET DEFAULT` | FK sütununa `DEFAULT` değerini yazar | Nadir; "Tanımsız" kaydı varsa |

```sql
-- Meşru CASCADE: sipariş silinirse detayı anlamsız kalır (gerçek sahiplik)
CONSTRAINT FK_SD_Siparisler FOREIGN KEY (SiparisId)
    REFERENCES Siparisler(SiparisId) ON DELETE CASCADE

-- Tehlikeli CASCADE: müşteri silinince tüm siparişleri de silinir
CONSTRAINT FK_Siparisler_Musteriler FOREIGN KEY (MusteriId)
    REFERENCES Musteriler(MusteriId) ON DELETE CASCADE   -- YAZMA
```

> **Tuzak — zincirleme silme.** `Musteriler → Siparisler → SiparisDetaylari` zincirinde her adımda `CASCADE` varsa tek bir `DELETE FROM Musteriler WHERE MusteriId = 3` binlerce kaydı sessizce siler ve geri alınmaz. Ticari kayıtlarda silmek yerine `Aktif BIT` alanıyla **soft delete** yapmak daha doğrudur.

> **Tuzak — çoklu cascade yolu.** Aynı tabloya birden fazla cascade yolu çıkıyorsa SQL Server `CREATE TABLE`'ı reddeder: *"may cause cycles or multiple cascade paths"*. `Siparisler`de hem `GonderenAdresId` hem `FaturaAdresId` aynı `Adresler` tablosuna cascade ile bağlanamaz.

EF Core'un varsayılanı FK'nın nullable olup olmamasına bakar: zorunlu ilişkide (`NOT NULL`) `Cascade`, opsiyonel ilişkide `ClientSetNull`. `OnDelete(DeleteBehavior.Restrict)` ile ezip veritabanına `NO ACTION` yazdırırsın. MvcCv'de Database First çalıştığın için bu ayarlar veritabanından okunuyordu; Code First'te sen yazacaksın.

> **Bu benzetme şurada bozulur:** Apartman yöneticisi 14 numaralı daireyi arar, bulamaz, kaydı reddeder. Veritabanında ise FK'nın **index'i yoksa** bu kontrol pahalıya patlar. SQL Server PK tarafında index'i otomatik yaratır ama **FK sütununda yaratmaz**. `Siparisler.MusteriId` üzerinde index yoksa, bir müşteriyi silmeye kalktığında SQL Server tüm `Siparisler` tablosunu tarar. FK sütunlarına elle index koymak neredeyse her zaman doğrudur.

---

## 4. İlişki Türleri: 1-1, 1-N, N-N

> **Benzetme —** Bir okulda üç tür bağ vardır. Her öğrencinin bir karnesi vardır, her karne bir öğrenciye aittir: bire bir. Bir sınıfta çok öğrenci vardır, her öğrenci tek sınıftadır: bire çok. Bir öğrenci çok kulübe yazılır, bir kulüpte çok öğrenci vardır: çoka çok. Üçüncüsünü deftere yazmanın tek yolu ayrı bir kulüp kayıt defteri tutmaktır.

**Basitçe:** İki tablo arasındaki bağ üç şekilden biridir. İlk ikisi tek bir FK sütunuyla kurulur. Üçüncüsü için araya üçüncü bir tablo koymak zorundasın.

### 1-N (bire çok) — en yaygın

FK, **çok** olan tarafa konur. Bir müşterinin çok siparişi olur, bir sipariş tek müşteriye aittir.

```sql
-- FK "çok" tarafta: Siparisler tablosunda
MusteriId INT NOT NULL,
CONSTRAINT FK_Siparisler_Musteriler FOREIGN KEY (MusteriId)
    REFERENCES Musteriler(MusteriId)
```

```csharp
public class Musteri
{
    public int MusteriId { get; set; }
    public ICollection<Siparis> Siparisler { get; set; } = new List<Siparis>();
}

public class Siparis
{
    public int MusteriId { get; set; }             // FK
    public Musteri Musteri { get; set; } = null!;  // navigation property
}
```

### 1-1 (bire bir)

İki tarafta da tek kayıt olur. Uygulaması: bağımlı tablonun PK'sı aynı zamanda FK'dır.

```sql
CREATE TABLE MusteriDetaylari (
    MusteriId   INT NOT NULL,              -- hem PK hem FK
    VergiNo     NVARCHAR(20)  NULL,
    Adres       NVARCHAR(500) NULL,
    CONSTRAINT PK_MusteriDetaylari PRIMARY KEY (MusteriId),
    CONSTRAINT FK_MD_Musteriler FOREIGN KEY (MusteriId)
        REFERENCES Musteriler(MusteriId) ON DELETE CASCADE
);
```

1-1 iki durumda meşrudur: sütunların bir kısmı nadiren kullanılıp tabloyu şişiriyorsa, ya da farklı güvenlik kuralına tabiyse. Bunun dışında sütunları aynı tabloda tut.

### N-N (çoka çok) — ara tablo şart

İlişkisel modelde N-N doğrudan kurulamaz. Araya **junction table (ara tablo / bağlantı tablosu)** koyarsın. Bir ürünün çok etiketi, bir etiketin çok ürünü olsun istiyorsan, `Urunler` ve `Etiketler` yanına üçüncü bir tablo açarsın:

```sql
-- Ara tablo: PK, iki FK'nın birleşimi
CREATE TABLE UrunEtiketleri (
    UrunId   INT NOT NULL,
    EtiketId INT NOT NULL,
    CONSTRAINT PK_UrunEtiketleri PRIMARY KEY (UrunId, EtiketId),
    CONSTRAINT FK_UE_Urunler   FOREIGN KEY (UrunId)   REFERENCES Urunler(UrunId),
    CONSTRAINT FK_UE_Etiketler FOREIGN KEY (EtiketId) REFERENCES Etiketler(EtiketId)
);
```

Bileşik PK burada iki iş birden yapar: satırı benzersiz kılar ve "aynı ürüne aynı etiket iki kez eklenemez" kuralını bedavaya getirir.

`SiparisDetaylari` da bir N-N ara tablosudur (`Siparisler` ↔ `Urunler`) — ama **payload'lı**: `Adet`, `BirimFiyat`, `IskontoOrani` taşır. Ara tabloya ek sütun geldiği anda kendi yapay anahtarını vermek işleri kolaylaştırır; `SiparisDetayId` bu yüzden var.

```csharp
// EF Core 5+ : payload'sız N-N için ara sınıf yazmana gerek yok
modelBuilder.Entity<Urun>()
    .HasMany(u => u.Etiketler)
    .WithMany(e => e.Urunler)
    .UsingEntity(j => j.ToTable("UrunEtiketleri"));
```

### Self-referencing (kendine referans)

`Kategoriler.UstKategoriId` aynı tablonun PK'sına işaret eder. Ağaç böyle kurulur: kök kategorilerde `UstKategoriId IS NULL`, alt kategorilerde üst kategorinin id'si durur.

```sql
INSERT INTO Kategoriler (Ad, UstKategoriId) VALUES ('Elektronik', NULL);  -- KategoriId = 1
INSERT INTO Kategoriler (Ad, UstKategoriId) VALUES ('Telefon',       1);  -- KategoriId = 2
INSERT INTO Kategoriler (Ad, UstKategoriId) VALUES ('Akıllı Telefon', 2); -- KategoriId = 3
```

Bu ağacı sorgulamak özyinelemeli CTE ister — yarınki notun konusu.

---
## 5. Normalizasyon: 1NF, 2NF, 3NF

> **Benzetme —** Terzinin sipariş defterini düşün. Bir sayfaya müşterinin adını, telefonunu, adresini, ölçülerini, sipariş ettiği üç parçayı ve kumaş fiyatlarını hep birlikte yazıyor. Müşteri ikinci kez geldiğinde adresi yine yazıyor. Taşınınca eski sayfalardaki adres yanlış kalıyor. Kumaş fiyatı değişince hangi sayfalarda güncelleyeceğini bilemiyor. Normalizasyon, bu tek defteri üç deftere bölmektir: müşteri defteri, sipariş defteri, kumaş defteri.

**Basitçe:** Normalizasyon, aynı bilginin birden fazla yerde tutulmasını engelleyerek tabloları bölme işidir. Amacı yer kazanmak değil, **tutarsızlığı imkânsız kılmaktır**.

**Teknik olarak:** **Normalization (normalleştirme)** — Tabloları, veri tekrarını ve güncelleme anormalliklerini ortadan kaldıracak biçimde parçalara ayırma süreci. Normal formlar birikimlidir: 3NF'de olan bir tablo 2NF ve 1NF'dedir.

Çözdüğü üç sorunun adı vardır: **insert anomaly** (siparişi olmayan ürünü sisteme giremezsin), **update anomaly** (40 satırdaki adresin 39'unu güncelleyip birini unutursun), **delete anomaly** (son siparişi silince kategorinin bilgisi de yok olur).

### 1NF — Her hücrede tek değer

**Kural:** Her sütun bölünemez (atomik) tek bir değer tutar. Tekrar eden sütun grubu olmaz.

Önce (1NF değil):

| SiparisId | Musteri | Urunler |
|---|---|---|
| 1 | Ahmet Yılmaz | Klavye, Mouse, Monitör |
| 2 | Ayşe Demir | Laptop |

`Urunler` hücresine üç ürün tıkıştırılmış. "Mouse alan kaç kişi var" sorusuna `LIKE '%Mouse%'` ile cevap aramak zorundasın — hem yavaş hem yanlış (`Mousepad` de eşleşir).

Sonra (1NF):

| SiparisId | MusteriId | UrunId | Adet |
|---|---|---|---|
| 1 | 7 | 101 | 1 |
| 1 | 7 | 102 | 1 |
| 1 | 7 | 103 | 1 |
| 2 | 9 | 104 | 1 |

`Urun1`, `Urun2`, `Urun3` diye üç sütun açmak da 1NF ihlalidir — tekrar eden sütun grubudur ve dördüncü ürün geldiğinde şemayı değiştirmen gerekir. Şemamızdaki `SiparisDetaylari` tablosu 1NF'in doğru uygulanışıdır.

### 2NF — Kısmi bağımlılık olmaz

**Kural:** 1NF'de olacak, **ve** anahtar dışı her sütun **bileşik anahtarın tamamına** bağlı olacak. Sadece bir parçasına bağlı sütun olmayacak. (Bileşik PK yoksa tablo zaten 2NF'dedir.)

Önce (2NF değil) — PK `(SiparisId, UrunId)`:

| SiparisId | UrunId | Adet | UrunAdi | SiparisTarihi |
|---|---|---|---|---|
| 1 | 101 | 2 | Klavye | 2026-03-01 |
| 1 | 102 | 1 | Mouse | 2026-03-01 |
| 2 | 101 | 5 | Klavye | 2026-03-04 |

`UrunAdi` sadece `UrunId`'ye, `SiparisTarihi` sadece `SiparisId`'ye bağlı. İkisi de **kısmi bağımlılık (partial dependency)**. Klavye'nin adı değişirse iki satırı da güncellemen gerekir.

Sonra (2NF) — üç tabloya bölünür:

```sql
Siparisler        : SiparisId (PK), MusteriId, SiparisTarihi   -- SiparisId'ye bağlı olanlar
Urunler           : UrunId (PK), Ad, KategoriId, BirimFiyat    -- UrunId'ye bağlı olanlar
SiparisDetaylari  : SiparisId, UrunId, Adet                    -- ikisine birden bağlı olan
```

`Adet` gerçekten ikisine birden bağlıdır: hangi siparişte hangi üründen kaç tane.

### 3NF — Geçişli bağımlılık olmaz

**Kural:** 2NF'de olacak, **ve** anahtar dışı bir sütun başka bir anahtar dışı sütuna bağlı olmayacak.

Önce (3NF değil):

| UrunId | Ad | KategoriId | KategoriAdi |
|---|---|---|---|
| 101 | Klavye | 5 | Bilgisayar Aksesuarı |
| 102 | Mouse | 5 | Bilgisayar Aksesuarı |
| 104 | Laptop | 6 | Bilgisayar |

`KategoriAdi`, `UrunId`'ye değil `KategoriId`'ye bağlı. `UrunId → KategoriId → KategoriAdi` zinciri **geçişli bağımlılıktır (transitive dependency)**. Kategori adı değişirse o kategorideki bütün ürün satırlarını güncellemen gerekir.

Sonra (3NF): `Urunler(UrunId, Ad, KategoriId, BirimFiyat)` ve `Kategoriler(KategoriId, Ad)`. Kategori adı tek yerde durur, `JOIN` ile alırsın.

3NF'in ötesinde BCNF, 4NF, 5NF de vardır ama pratikte 3NF yeterlidir.

> **Bu benzetme şurada bozulur:** Terzi defterlerini böldü, tutarsızlık bitti. Ama artık "Ahmet'in ne sipariş ettiğini" görmek için üç deftere birden bakması gerekiyor. Normalizasyon tutarlılığı garanti eder ama **okuma maliyetini artırır**. Her bölme bir `JOIN` demektir. Bu yüzden bir sonraki bölüm var.

---

## 6. Denormalizasyon Ne Zaman Meşrudur

> **Benzetme —** Lokanta mutfağında tuz, karabiber ve pul biber merkezî bir kilerde durur — tek yerde, düzenli. Ama her ocağın yanında da küçük birer kap vardır. Aşçı her tutam tuz için kilere gitmez. Kopya vardır, evet; ama bilinçlidir, kontrollüdür ve akşam servis bitince kiler esas alınarak doldurulur. Denormalizasyon budur: kural tanımazlık değil, ölçülmüş bir taviz.

**Basitçe:** Denormalizasyon, okuma hızı için bilerek veri tekrarı yapmaktır. Ama önce ölçersin, sonra tekrar edersin.

**Teknik olarak:** **Denormalization (denormalleştirme)** — 3NF'e uyan bir şemada, belirli bir okuma senaryosunu hızlandırmak için kasıtlı olarak tekrar eden veri bulundurmak. Bedeli: tutarlılığı artık veritabanı değil, **sen** garanti edersin.

Üç meşru gerekçe vardır.

**1) Tarihsel doğruluk (en güçlü gerekçe).** Şemamızdaki `SiparisDetaylari.BirimFiyat` bunun örneğidir.

```sql
-- Yanlış: fiyatı JOIN ile Urunler'den al
SELECT sd.Adet * u.BirimFiyat AS SatirToplam
FROM SiparisDetaylari sd
JOIN Urunler u ON u.UrunId = sd.UrunId;
-- Ürünün fiyatı bugün 500 ise, geçen yılki siparişin tutarı da 500 görünür

-- Doğru: sipariş anındaki fiyat detay satırında saklanır
SELECT sd.Adet * sd.BirimFiyat * (1 - sd.IskontoOrani) AS SatirToplam
FROM SiparisDetaylari sd;
```

Bu aslında tekrar değildir: `Urunler.BirimFiyat` "şu anki fiyat", `SiparisDetaylari.BirimFiyat` "o gün satılan fiyat". Farklı iki olgu, aynı isimli iki sütun.

**2) Hesaplanmış toplamlar (aggregate).** Milyonlarca detay satırı üzerinden her seferinde `SUM` almak pahalıysa toplamı sipariş başlığında tutarsın (`ALTER TABLE Siparisler ADD ToplamTutar DECIMAL(18,2) NULL;`). Tutarlılığı korumanın en temiz yolu **computed column**'dur; tanım tek yerdedir ve veritabanı garanti eder:

```sql
ALTER TABLE SiparisDetaylari
ADD SatirToplam AS (Adet * BirimFiyat * (1 - IskontoOrani)) PERSISTED;
-- PERSISTED değeri diskte saklar, index koyabilirsin; olmadan her okumada hesaplanır
```

**3) Raporlama tabloları.** Gece çalışan bir iş günlük satış özetini ayrı tabloya yazar; rapor ekranı canlı tabloları hiç görmez. Veri bayattır ama rapor milisaniyede açılır.

| Denormalizasyon türü | Tutarlılığı kim sağlar | Risk |
|---|---|---|
| Tarihsel kopya (`BirimFiyat`) | Kimse — zaten farklı olgu | Yok |
| Computed column `PERSISTED` | Veritabanı | Yok (aynı satır içi) |
| Uygulama kodunun güncellediği toplam | Sen | Yüksek |
| Gece çalışan rapor tablosu | Zamanlanmış iş | Bayatlık |

> **Sıralama kuralı:** Önce 3NF'te tasarla, yavaşlığı **ölç**, index dene, sorguyu düzelt. Denormalizasyon en son çaredir. "Nasılsa JOIN yavaş" diye baştan denormalize edilmiş şemalar çoğu zaman index eksikliğinden yavaştır.

> **Bu benzetme şurada bozulur:** Aşçının ocak yanındaki kabı boşalınca fark eder, gider doldurur. Veritabanındaki denormalize sütun **sessizce bozulur**. `Siparisler.ToplamTutar` ile detayların `SUM`'ı tutmadığında kimse uyarı almaz; aylar sonra muhasebe fark eder. Denormalize her sütun için bir doğrulama sorgusu yazıp periyodik çalıştırmak şart.

---

## 7. Veri Tipleri ve Doğru Seçim

> **Benzetme —** Markette poşet seçmek gibi. Tek limon için koca koli almazsın; on kilo patatesi de naylon poşete koymazsın. Ama asıl mesele boyut değil **cinstir**: dondurulmuş balığı kâğıt poşete koyarsan poşet dağılır. Veri tipinde de aynı: `float` kabının içinde para taşırsan kuruşlar sızar.

**Basitçe:** Her sütun için hem yeterince büyük hem de gereksiz büyük olmayan, ve en önemlisi doğru cinsten bir tip seçersin.

### Sayısal tipler

| Tip | Boyut | Aralık | Ne zaman |
|---|---|---|---|
| `TINYINT` | 1 bayt | 0 – 255 | Durum kodu, yaş |
| `SMALLINT` | 2 bayt | ±32 bin | Yıl, küçük sayaç |
| `INT` | 4 bayt | ±2,1 milyar | Varsayılan PK/FK |
| `BIGINT` | 8 bayt | ±9,2 kentilyon | Log, event, yüksek hacimli detay tablosu |
| `DECIMAL(p,s)` | 5–17 bayt | Kesin ondalık | **Para, oran, miktar** |
| `FLOAT` / `REAL` | 8 / 4 bayt | Yaklaşık ondalık | Bilimsel ölçüm, koordinat |

**Para için asla `float` kullanma.** `float` ikili kayan noktalıdır; `0.1` sayısını tam temsil edemez.

```sql
DECLARE @f FLOAT = 0.1, @g FLOAT = 0.2;
SELECT CASE WHEN @f + @g = 0.3 THEN 'eşit' ELSE 'EŞİT DEĞİL' END;   -- EŞİT DEĞİL

DECLARE @d DECIMAL(18,2) = 0.1, @e DECIMAL(18,2) = 0.2;
SELECT CASE WHEN @d + @e = 0.3 THEN 'eşit' ELSE 'EŞİT DEĞİL' END;   -- eşit
```

`DECIMAL(18,2)`: toplam 18 hane, 2'si ondalık — para için standart. Kuruşun altı gerekiyorsa `DECIMAL(18,4)`. `MONEY` tipi de vardır ama 4 hane sabittir ve bölmede yuvarlama sürprizleri yapar.

```csharp
public decimal BirimFiyat { get; set; }   // C# karşılığı decimal, double değil

// EF Core'da hassasiyeti belirt, yoksa uyarı verir ve varsayılana düşer
modelBuilder.Entity<Urun>().Property(u => u.BirimFiyat).HasPrecision(18, 2);
```

### Metin tipleri

| Tip | Karakter | Bayt/karakter | Ne zaman |
|---|---|---|---|
| `VARCHAR(n)` | Tek bayt kodlama | 1 | Kod, kısaltma: `'TR'`, `'ORD-2026-001'` |
| `NVARCHAR(n)` | Unicode (UTF-16) | 2 | **Kullanıcı metni** — ad, adres, açıklama |
| `CHAR(n)` / `NCHAR(n)` | Sabit uzunluk | 1 / 2 | Gerçekten sabit: `CHAR(2)` ülke kodu |
| `(MAX)` biçimleri | 2 GB'a kadar | — | Uzun metin; index'lenemez |

Türkçe karakterler için `NVARCHAR` kullan: `VARCHAR` collation'a bağlıdır, yanlış collation'da "ş" ve "ğ" bozulur. `Ad VARCHAR(50)` kötü, `Ad NVARCHAR(100)` iyi.

> **`(MAX)` tuzağı:** `NVARCHAR(MAX)` bir sütunu index'in anahtarına koyamazsın. Ayrıca gereksiz `(MAX)` kullanımı SQL Server'ın bellek tahminini bozar. "Ne olur ne olmaz" diye `(MAX)` yazma; gerçekçi bir üst sınır koy.

### Tarih ve mantıksal tipler

| Tip | Boyut | Hassasiyet | Not |
|---|---|---|---|
| `DATE` | 3 bayt | Gün | Doğum tarihi, fatura günü |
| `DATETIME` | 8 bayt | ~3,33 ms | **Eski tip — yeni kodda kullanma** |
| `DATETIME2(n)` | 6–8 bayt | 100 ns'ye kadar | Varsayılan seçim |
| `DATETIMEOFFSET` | 10 bayt | + saat dilimi | Çok bölgeli uygulama |
| `BIT` | 1 bit (8'i 1 bayta) | 0/1/NULL | Mantıksal bayrak |

`DATETIME` üç sebeple eskidir: aralığı 1753'te başlar, 3,33 milisaniyeye yuvarlar ve 8 bayt sabittir. `DATETIME2(3)` aynı hassasiyeti 7 baytta verir.

```sql
DECLARE @dt  DATETIME     = '2026-03-01 23:59:59.999';
SELECT @dt;   -- 2026-03-02 00:00:00.000   -- gün değişti!

DECLARE @dt2 DATETIME2(3) = '2026-03-01 23:59:59.999';
SELECT @dt2;  -- 2026-03-01 23:59:59.999   -- olduğu gibi
```

Tarihi **UTC** yaz, görüntülerken çevir: `DEFAULT (SYSUTCDATETIME())`. `GETDATE()` sunucunun yerel saatidir; sunucu taşınırsa veri kayar.

`BIT` üç değer alabilir: `0`, `1` ve `NULL`. `NOT NULL` + `DEFAULT (0)` yazmazsan "belki" durumu oluşur.

---
## 8. NULL ve Üç Değerli Mantık

> **Benzetme —** Nöbetçi eczaneye gidip "sizde şu ilaç var mı" diye soruyorsun. Üç cevap mümkün: "var", "yok" ve "bilmiyorum, sistem kapalı". Üçüncü cevap "yok" değildir. İki eczaneye de sorup ikisinden de "bilmiyorum" alırsan, "ikisinde de aynı durum var" diyemezsin. İki bilinmeyen birbirine eşit değildir.

**Basitçe:** `NULL`, "değer yok" değil **"değer bilinmiyor"** demektir. Bu yüzden `NULL` ile yapılan karşılaştırmalar ne doğru ne yanlış; belirsiz sonuç verir.

**Teknik olarak:** SQL, iki değerli değil **üç değerli mantık (three-valued logic)** kullanır: `TRUE`, `FALSE`, `UNKNOWN`. `WHERE` yan tümcesi yalnızca `TRUE` olan satırları döndürür. `UNKNOWN` olanlar elenir.

```sql
SELECT CASE WHEN NULL = NULL THEN 'eşit' ELSE 'eşit değil / bilinmiyor' END;
-- 'eşit değil / bilinmiyor'  -- karşılaştırma UNKNOWN döner

-- Doğrusu
WHERE Sehir IS NULL
WHERE Sehir IS NOT NULL
```

| İfade | Sonuç |
|---|---|
| `NULL = NULL` | `UNKNOWN` |
| `NULL <> NULL` | `UNKNOWN` |
| `NULL + 5` | `NULL` |
| `'Abc' + NULL` | `NULL` |
| `TRUE AND UNKNOWN` | `UNKNOWN` |
| `TRUE OR UNKNOWN` | `TRUE` |
| `FALSE AND UNKNOWN` | `FALSE` |
| `NOT UNKNOWN` | `UNKNOWN` |

En sık yapılan hata, `NOT IN` ile `NULL` içeren bir alt sorgu kullanmaktır: `x NOT IN (1, 2, NULL)` ifadesi `x<>1 AND x<>2 AND UNKNOWN` demektir ve sorgu hiçbir satır döndürmez. Güvenli yazım `NOT EXISTS`'tir — yarınki notta ayrıntısı var.

`NULL`'u değere çevirmenin yolları (ayrıntısı yarınki notta):

```sql
SELECT ISNULL(Sehir, 'Belirtilmemiş')       FROM Musteriler;  -- SQL Server'a özel, 2 argüman
SELECT COALESCE(Sehir, Ilce, 'Bilinmiyor')  FROM Musteriler;  -- standart, N argüman
SELECT NULLIF(StokAdedi, 0)                 FROM Urunler;     -- 0 ise NULL yap
```

**Toplama fonksiyonları `NULL`'u yok sayar** — `COUNT(*)` hariç. `COUNT(*)` tüm satırları, `COUNT(Sehir)` yalnızca `Sehir`'i `NULL` olmayan satırları sayar; `AVG` ise `NULL`'ları paydaya bile almaz. Yarınki notta bu ayrımı tekrar göreceksin.

> **Tasarım notu:** Sütunu gerçekten bilinmeyebilir değilse `NOT NULL` yaz ve `DEFAULT` ver. Nullable sütun her sorguda bir "acaba" ekler. `Musteriler.Sehir` nullable çünkü kayıt sırasında sorulmayabilir; `Musteriler.Eposta` `NOT NULL` çünkü onsuz kayıt anlamsız.

> **Bu benzetme şurada bozulur:** Eczane örneğinde "bilmiyorum" iki kez gelse bile ikisini eşit saymıyoruz — SQL de öyle. Ama `UNIQUE` kısıtı bu kuralı çiğner: SQL Server bir `UNIQUE` sütunda **yalnızca tek bir `NULL`'a** izin verir. İki `NULL`'u burada "aynı" sayar. Bu davranış standart dışıdır (PostgreSQL çoklu `NULL`'a izin verir) ve SQL Server'a geçerken sürpriz olur. Çözüm: filtrelenmiş index (`WHERE sutun IS NOT NULL`).

---

## 9. Constraint'ler: UNIQUE, CHECK, DEFAULT

> **Benzetme —** İnşaatta kolonu yerine koyduktan sonra "acaba dik mi" diye gözle bakmazsın; şakülle ölçersin ve şakül yalan söylemez. Kısıtlar veritabanının şakülüdür. Kuralı uygulama koduna yazarsan, yarın başka bir uygulama aynı veritabanına bağlandığında kural yok sayılır. Tabloya yazarsan kimse kaçamaz.

**Basitçe:** Kısıtlar, veritabanının kendisinin dayattığı kurallardır. Uygulama unutabilir, kısıt unutmaz.

**Teknik olarak:** **Constraint (kısıt)** — Tablodaki verinin uyması gereken, veritabanı motoru tarafından zorlanan kural. Beş türü vardır: `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `DEFAULT`.

```sql
-- UNIQUE: aynı e-posta iki kez kaydedilemez
CONSTRAINT UQ_Musteriler_Eposta UNIQUE (Eposta)

-- Bileşik UNIQUE: aynı siparişte aynı ürün iki satır olamaz
CONSTRAINT UQ_SD_Siparis_Urun UNIQUE (SiparisId, UrunId)

-- CHECK: değer aralığı
CONSTRAINT CK_Urunler_Fiyat CHECK (BirimFiyat >= 0)
CONSTRAINT CK_SD_Adet       CHECK (Adet > 0)
CONSTRAINT CK_SD_Iskonto    CHECK (IskontoOrani >= 0 AND IskontoOrani < 1)

-- CHECK: durum kodu listesi ve sütunlar arası kural (aynı satır içinde)
CONSTRAINT CK_Siparisler_Durum  CHECK (Durum IN (0, 1, 2, 3, 9))
CONSTRAINT CK_Siparisler_Teslim CHECK (TeslimTarihi IS NULL OR TeslimTarihi >= SiparisTarihi)

-- DEFAULT: değer verilmezse ne olsun
CONSTRAINT DF_Urunler_Aktif DEFAULT (1) FOR Aktif
```

| Kısıt | Ne garanti eder | Index yaratır mı |
|---|---|---|
| `PRIMARY KEY` | Benzersiz + `NOT NULL` | Evet (varsayılan clustered) |
| `UNIQUE` | Benzersiz (tek `NULL`'a izin verir) | Evet (nonclustered) |
| `FOREIGN KEY` | Referans bütünlüğü | **Hayır** — elle koyarsın |
| `CHECK` | İfadenin `FALSE` olmaması | Hayır |
| `DEFAULT` | Değer verilmediğinde varsayılan | Hayır |

> **`CHECK` ve `NULL`:** `CHECK`, ifadesi `FALSE` olduğunda reddeder. `UNKNOWN` olduğunda **kabul eder**. `CHECK (Adet > 0)` kısıtı `Adet = NULL` satırını geçirir. `NULL` istemiyorsan sütunu `NOT NULL` yapacaksın; `CHECK` bunu yapmaz.

### Kısıt adını kendin ver

`CONSTRAINT` anahtar kelimesini yazmazsan SQL Server `CK__Urunler__BirimF__5629CD9C` gibi rastgele bir ad üretir; bu ad her veritabanında farklı olur ve migration'da `DROP CONSTRAINT` yazmayı imkânsızlaştırır. `PK_`, `FK_`, `UQ_`, `CK_`, `DF_`, `IX_` ön ekleri yaygın standarttır. EF Core'da `HasDatabaseName(...)` ve `HasCheckConstraint(...)` ile aynı adları sen verirsin.

> **Uygulama doğrulaması kısıtın yerini tutmaz.** FluentValidation veya `[Range]` attribute'u kullanıcıya nazik hata mesajı gösterir — bu iyidir. Ama veriyi bozulmaktan koruyan şey tablodaki `CHECK`'tir. İkisi birlikte yazılır: biri kullanıcı deneyimi, diğeri veri bütünlüğü içindir.

---

## 10. Index Neden Hızlandırır

> **Benzetme —** Kütüphanede 200 bin kitap var ve sen "Yaban" adlı romanı arıyorsun. Raf raf gezersen akşam olur. Girişteki fihrist dolabına gidersin: çekmeceler alfabetik, "Y" çekmecesini açarsın, karttaki raf numarasını okur, doğrudan o rafa gidersin. Fihrist kitabın kendisi değildir — sadece adı ve yeri yazar. Index de tam olarak budur.

**Basitçe:** Index, bir sütunun sıralanmış kopyasını ve her değerin satırın nerede olduğunu tutan yardımcı yapıdır. Sıralı olduğu için aranan değeri baştan sona taramadan bulur.

**Teknik olarak:** SQL Server index'leri **B-tree (dengeli ağaç)** yapısındadır. Üç katmanı vardır: kök sayfa, ara sayfalar, yaprak sayfalar. Her sayfa 8 KB'dır.

Sade anlatım: Kök sayfa "1–1000 arası şu sayfada, 1001–2000 arası şu sayfada" der. O sayfaya gidersin, o da aralığı daha da daraltır. 10 milyon satırlık tabloda ağaç tipik olarak 3–4 seviyedir; yani **3–4 sayfa okumayla** aradığını bulursun. Index'siz alternatif **table scan**'dir: 10 milyon satır, satır başına 200 bayt ise yaklaşık 250 bin sayfa okuması. Fark 4 ile 250.000 arasındadır.

```sql
SELECT * FROM Siparisler WHERE MusteriId = 4200;   -- index yoksa tüm tablo taranır

CREATE NONCLUSTERED INDEX IX_Siparisler_MusteriId ON Siparisler (MusteriId);
-- artık ağaçta 3-4 sıçrama
```

### Clustered vs Non-clustered

| | **Clustered** | **Non-clustered** |
|---|---|---|
| Ne tutar | **Satırın kendisi** yapraktadır | Yaprakta anahtar + satıra işaretçi |
| Kaç tane olur | Tablo başına **1** | Tablo başına çok (pratikte 5–8) |
| Analoji | Kitabın sayfa sırası | Kitabın sonundaki dizin |
| Sıra | Tablonun fiziksel sırasını belirler | Tabloyu etkilemez |
| Varsayılan | `PRIMARY KEY` ile gelir | `CREATE INDEX` ile yaratılır |

```sql
CONSTRAINT PK_Siparisler PRIMARY KEY CLUSTERED (SiparisId)   -- fiziksel düzen, PK ile gelir
CREATE NONCLUSTERED INDEX IX_Siparisler_Tarih ON Siparisler (SiparisTarihi);  -- ek fihrist
```

Clustered index'i olmayan tabloya **heap** denir; üretim tablolarında heap istemezsin.

**Key lookup:** Non-clustered index aradığını bulur, ama `SELECT *` yazdıysan geri kalan sütunlar için clustered index'e gitmesi gerekir — her satır için bir ek okuma. Çok satırda SQL Server "bu kadar lookup yerine tabloyu tararım" der ve index'i **kullanmaz**. Çözümü bir sonraki bölümdeki covering index'tir.

> **Bu benzetme şurada bozulur:** Kütüphane fihristi kitap ekledikçe elle güncellenir ve arada bir güncellenmese de kütüphane çalışmaya devam eder. Veritabanında öyle değil: her `INSERT`, `UPDATE`, `DELETE` **bütün ilgili index'leri aynı işlem içinde** günceller. Beş index'i olan bir tabloya satır eklemek, tek index'i olana eklemekten belirgin biçimde pahalıdır. Index bedavaya gelmez, okuma hızını yazma hızından satın alırsın.

---
## 11. Index Tasarımı: Composite, Covering, Maliyet

> **Benzetme —** Telefon rehberi soyada göre, aynı soyadlar içinde ada göre dizilidir. "Yılmaz" ailesini saniyede bulursun. "Adı Ahmet olan herkes" dersen rehber işe yaramaz, baştan sona okursun. Sıra tesadüf değildir: **önce daraltan** ölçüt başa yazılır. Composite index'te sütun sırası tam olarak bu rehber sırasıdır.

**Basitçe:** Birden çok sütunlu index yazarken sıra kritiktir. Index, soldan başlayarak kullanılabilir; ortadaki bir sütuna tek başına dayanamazsın.

**Teknik olarak:** **Composite index (bileşik index)** — Birden fazla sütun üzerine kurulan index. Anahtar, sütunların soldan sağa birleşimidir. Buna **leftmost prefix (en soldaki önek)** kuralı denir.

```sql
CREATE NONCLUSTERED INDEX IX_Siparisler_Musteri_Tarih
    ON Siparisler (MusteriId, SiparisTarihi);
```

Bu index şunlarda kullanılır:

| Sorgu koşulu | Index kullanılır mı |
|---|---|
| `WHERE MusteriId = 5` | Evet (sol önek) |
| `WHERE MusteriId = 5 AND SiparisTarihi >= '2026-01-01'` | Evet (tam kullanım) |
| `WHERE SiparisTarihi >= '2026-01-01'` | **Hayır** — sol sütun yok, en iyi ihtimalle tarama |
| `ORDER BY MusteriId, SiparisTarihi` | Evet — sıralama bedavaya gelir |

**Sıralama kuralı:** Önce **eşitlik** (`=`) ile filtrelenen sütunlar, sonra **aralık** (`>`, `<`, `BETWEEN`) ile filtrelenenler, en sonda sadece sıralama için gerekenler.

```sql
-- Sorgu: WHERE Durum = 2 AND SiparisTarihi >= @bas
CREATE INDEX IX_Durum_Tarih ON Siparisler (Durum, SiparisTarihi);   -- doğru: eşitlik önce
CREATE INDEX IX_Tarih_Durum ON Siparisler (SiparisTarihi, Durum);   -- yanlış: aralık önce
```

### Covering index ve `INCLUDE`

**Covering index (kapsayan index)** — Sorgunun ihtiyaç duyduğu **tüm** sütunları içeren index. Key lookup gerekmez; SQL Server tabloya hiç gitmez. Plan'da bunun adı **Index Seek** + hiçbir lookup'tır.

```sql
-- Sorgu: müşterinin siparişlerini tarih ve durumla listele
SELECT SiparisId, SiparisTarihi, Durum
FROM Siparisler
WHERE MusteriId = 4200;

-- Anahtar sadece MusteriId; diğer sütunlar yaprağa taşınır
CREATE NONCLUSTERED INDEX IX_Siparisler_Musteri_Covering
    ON Siparisler (MusteriId)
    INCLUDE (SiparisTarihi, Durum);
```

`INCLUDE` ile eklenen sütunlar **yalnızca yaprak seviyede** durur; ağacı şişirmez, arama derinliğini artırmaz. Kural: filtrelenen/sıralanan sütunlar anahtara, sadece `SELECT`'te geçenler `INCLUDE`'a.

```sql
CREATE NONCLUSTERED INDEX IX_Siparisler_Aktif ON Siparisler (SiparisTarihi)
    WHERE Durum IN (0, 1, 2);     -- filtrelenmiş: iptal ve teslim edilenler index'te yok
```

Filtrelenmiş index hem küçüktür hem bakımı ucuzdur; "aktif kayıtlar" tipi sorgularda çok işe yarar.

### Yazma maliyeti ve ne zaman index KOYMA

Her index bir kopya demektir: yazdığın her satır her index'e de yazılır, güncellediğin sütun kaç index'te geçiyorsa o kadar yerde güncellenir.

**Şu durumlarda index koyma:**

- **Küçük tablolarda.** Birkaç yüz satırlık `Kategoriler` tablosunda tarama zaten tek sayfa okumasıdır.
- **Düşük seçicilikte (low selectivity).** `Aktif BIT` sütununda satırların %95'i `1` ise index işe yaramaz; SQL Server yine tarar. Kabaca: bir değer tablonun %5–10'undan fazlasını döndürüyorsa index seek yerine scan seçilir.
- **Ağır yazma, seyrek okuma yapılan tablolarda.** Log ve audit tablolarına gereksiz index koymak yazma hızını düşürür.
- **Mevcut bir index'in soluna eklenerek kapsanabilecek durumlarda.** `(MusteriId)` index'i varken `(MusteriId, Durum)` yaratırsan ilkini silebilirsin — ikincisi onun işini de görür. Tersi doğru değildir.
- **Deneme amaçlı.** "Belki hızlanır" diye index eklemek yerine ölç, sonra ekle, sonra tekrar ölç.

Hangi index'in işe yaradığını `sys.dm_db_index_usage_stats` görünümünden okursun: `user_updates` yüksek ama `user_seeks` ve `user_scans` sıfır olan bir index'in bedeli vardır, faydası yoktur.

> **Bu benzetme şurada bozulur:** Telefon rehberinde birden çok rehber tutmak sadece raf yeri kaplar. Veritabanında ise her ek index **yazma işlemini yavaşlatır** ve kilitlenme (blocking) süresini uzatır. Ayrıca SQL Server'ın sorgu iyileştiricisi çok sayıda benzer index arasında yanlış olanı seçebilir. Beş iyi index, yirmi ortalama index'ten iyidir.

---

## 12. SARGability ve Ölçme

> **Benzetme —** Postanede kargolar takip numarasına göre dizili. "Numarası 12345 olan koli" dersen görevli doğru rafa gider. Ama "numarasının son üç hanesi 345 olan koli" dersen, görevli bütün kolileri tek tek eline almak zorunda. Diziliş numaranın **tamamına** göredir; numarayı sen değiştirdiğin anda diziliş işe yaramaz. SQL'de sütunu fonksiyona sokmak tam olarak budur.

**Basitçe:** Bir sütunu `WHERE` içinde olduğu gibi bırakırsan index kullanılır. Fonksiyona sokar, hesaplar veya tipini değiştirirsen index ölür.

**Teknik olarak:** **SARGable (Search ARGument able)** — Bir arama koşulunun index seek'e çevrilebilir olması. Sütun koşulun bir tarafında **yalnız** duruyorsa SARGable'dır.

```sql
-- SARGable DEĞİL: sütun fonksiyonun içinde
WHERE YEAR(SiparisTarihi) = 2026
WHERE CONVERT(DATE, SiparisTarihi) = '2026-03-01'
WHERE UPPER(Eposta) = 'AHMET@ORNEK.COM'
WHERE SiparisId + 0 = 500
WHERE ISNULL(Sehir, '') = 'Kayseri'
WHERE Eposta LIKE '%ornek.com'          -- baştaki % index'i bitirir

-- SARGable: sütun yalnız, hesap sağ tarafta
WHERE SiparisTarihi >= '2026-01-01' AND SiparisTarihi < '2027-01-01'
WHERE SiparisTarihi >= '2026-03-01' AND SiparisTarihi < '2026-03-02'
WHERE Eposta = 'ahmet@ornek.com'        -- collation case-insensitive ise UPPER gereksiz
WHERE SiparisId = 500
WHERE (Sehir = 'Kayseri' OR Sehir IS NULL)
WHERE Eposta LIKE 'ahmet%'              -- sondaki % sorun değil
```

Tarih aralığında `BETWEEN` yerine `>= ... < ...` yaz: `BETWEEN '2026-03-01' AND '2026-03-31'` yazarsan 31 Mart'ın 00:00'dan sonraki kayıtları dışarıda kalır.

**Gizli tip dönüşümü (implicit conversion)** da SARGability'yi bozar ve gözle görülmez: sütun `VARCHAR(20)` iken parametre `NVARCHAR` gelirse SQL Server sütunu çevirir ve index'i kullanamaz. Execution plan'da `CONVERT_IMPLICIT` görüyorsan sebebi budur. EF Core'da C# `string` her zaman `NVARCHAR` gönderir; çözüm `IsUnicode(false)` veya `HasColumnType("varchar(20)")` ile tipleri eşitlemektir.

### Ölçme: `SET STATISTICS IO ON`

Tahmin etmek yerine ölçersin. SSMS'te sorgu penceresinde:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT SiparisId, SiparisTarihi FROM Siparisler WHERE MusteriId = 4200;
```

Messages sekmesinde şöyle bir çıktı gelir:

```
Table 'Siparisler'. Scan count 1, logical reads 4, physical reads 0, read-ahead reads 0.
 SQL Server Execution Times: CPU time = 0 ms, elapsed time = 0 ms.
```

| Alan | Anlamı |
|---|---|
| **logical reads** | Okunan 8 KB'lık sayfa sayısı — **asıl ölçü budur**, düşürmeye çalışırsın |
| physical reads | Diskten okunan sayfa; ilk çalıştırmada yüksek, sonra 0 |
| scan count | Tablonun kaç kez gezildiği; yüksekse döngüsel bir plan var |
| elapsed time | Duvar saati süresi; makine yüküne göre dalgalanır, güvenilmez |

`logical reads` makineden makineye değişmez. Index eklemeden önce ve sonra bu sayıyı karşılaştırırsan iyileştirmenin gerçek olup olmadığını görürsün: tipik olarak `2847` (table scan) → `4` (index seek).

### Execution plan'a kısa bakış

SSMS'te `Ctrl+M` (Include Actual Execution Plan) ile sorguyu çalıştırırsın. Plan sağdan sola okunur. Bakılacak beş şey:

| Plan operatörü | Anlamı | Durum |
|---|---|---|
| **Index Seek** | Ağaçta doğrudan gidiş | İstenen |
| **Index Scan** | Index'in tamamı okunuyor | Kabul edilebilir, ama bak |
| **Clustered Index Scan / Table Scan** | Tüm tablo okunuyor | Büyük tabloda kırmızı bayrak |
| **Key Lookup** | Index bulundu, sütunlar için tabloya gidiliyor | `INCLUDE` ile çözülür |
| **Sort** (pahalı) | Sıralama bellekte/tempdb'de yapılıyor | Uygun index sıralamayı bedavaya verir |

Ok kalınlıkları akan satır sayısını gösterir. **Tahmini ile gerçek satır sayısı arasındaki büyük fark** istatistiklerin bayatladığını gösterir; `UPDATE STATISTICS Siparisler;` veya `EXEC sp_updatestats;` ile tazelenir.

SSMS plan üzerinde yeşil "Missing Index" önerisi de gösterir; bu bir başlangıç noktasıdır, emir değil — genellikle gereğinden fazla sütunu `INCLUDE`'a koyar ve mevcut index'leri dikkate almaz.

> **Bu benzetme şurada bozulur:** Postane görevlisi "son üç hane" araması yapınca yavaş olduğunu fark eder ve sana söyler. SQL Server hiçbir şey söylemez — sorgu çalışır, doğru sonucu döndürür, sadece yavaştır. Test verisinde 500 satır varken fark edilmez; üretimde 5 milyon satırda uygulama durur. Bu yüzden `logical reads` ölçümünü **gerçekçi hacimli** veriyle yapman gerekir.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Domain | Bir sütunun alabileceği değerler kümesi (tip + kısıt) |
| Primary key | Seçilmiş aday anahtar; benzersiz ve `NOT NULL` |
| Surrogate key | Sırf kimliklendirme için üretilmiş anlamsız anahtar |
| Sequential GUID | Rastgele değil artan sırada üretilen GUID |
| Foreign key | Başka tablonun anahtarına işaret eden sütun |
| Junction table | N-N ilişkiyi kuran ara tablo |
| Normalization | Tekrarı ve anormallikleri gidermek için tabloyu bölme |
| Transitive dependency | Sütunun başka bir anahtar dışı sütuna bağlı olması |
| Denormalization | Okuma hızı için kasıtlı veri tekrarı |
| Three-valued logic | `TRUE` / `FALSE` / `UNKNOWN` üçlüsüyle çalışan mantık |
| B-tree | Index'in dengeli ağaç yapısı |
| Clustered index | Yaprağında satırın kendisini tutan, tablo başına tek index |
| Non-clustered index | Anahtar + işaretçi tutan ek index |
| Covering index | Sorgunun tüm sütunlarını kapsayan index |
| Key lookup | Index'ten sonra sütunlar için tabloya gidilmesi |
| SARGable | Koşulun index seek'e çevrilebilir olması |
| Implicit conversion | SQL Server'ın sessizce yaptığı tip dönüşümü |
| Logical reads | Sorgunun okuduğu 8 KB'lık sayfa sayısı |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `02-T-SQL-Sorgulama.md` (Salı)
