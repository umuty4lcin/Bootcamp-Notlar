# Hafta 3 · Perşembe — EF Core Temelleri ve Migration

**Okuma süresi:** ~50 dk
**Neden bu konu:** EF Core, .NET dünyasında veritabanıyla konuşmanın varsayılan yoludur. Bootcamp'te yazacağın her projede karşına çıkar. Sen MvcCv'de EF Core'u **Database First** ile kullandın — `Scaffold-DbContext` çalıştırdın, sınıflar üretildi. Bootcamp'te büyük ihtimalle **Code First** ve migration göreceksin. Bu not iki yolu da anlatır ve hangi durumda hangisinin seçildiğini netleştirir.

---

## Önce Basitçe

C# tarafında `Musteri` diye bir sınıf yazarsın: içinde `Ad`, `Soyad`, `Sehir` vardır. Veritabanı tarafında `Musteriler` diye bir tablo vardır: içinde `Ad`, `Soyad`, `Sehir` kolonları vardır. İkisi aynı şeyi anlatır ama aynı dilde konuşmaz. Biri nesne, diğeri satır. Biri referansla bağlanır, diğeri anahtarla. Bu iki dünya arasında sürekli çeviri yapman gerekir.

Eskiden bu çeviriyi elle yazardın. `SqlCommand` açar, SQL metnini kurar, `SqlDataReader` ile satırları gezer, her kolonu doğru özelliğe atardın. Otuz kolonlu bir tabloda bu iş yüz satır sıkıcı kod demekti. Hepsi de birbirinin kopyası. Ve bir kolon adı değişince otuz yerde birden elle düzeltirdin.

ORM, bu çeviriyi senin yerine yapan katmandır. Sen `context.Musteriler.ToList()` dersin, o SQL'i kurar, sonucu okur, nesneleri doldurur ve sana liste verir. Sen `musteri.Sehir = "Kayseri"` dersin, o değişikliği fark eder ve `SaveChanges()` dediğinde uygun `UPDATE` cümlesini üretir.

İkinci mesele veritabanının şeklini kimin belirlediğidir. İki yol var. **Code First**'te sınıflarını yazarsın, EF Core sana uygun tabloları üretir. **Database First**'te veritabanı zaten vardır, EF Core sana ona uygun sınıfları üretir. Sen MvcCv'de ikincisini yaptın. Bootcamp'te birincisini göreceksin.

Code First'ün kalbi **migration**dır. Sınıfına yeni bir özellik eklediğinde veritabanı bunu bilmez. Migration, "modelin eski hâli ile yeni hâli arasındaki farkı SQL'e çeviren" adımdır. Her migration bir dosyadır, tarihe göre sıralanır ve veritabanında hangilerinin uygulandığı kaydedilir. Böylece takımdaki herkesin veritabanı aynı noktaya gelir. Şimdi detaya iniyoruz.

> **Ana benzetme:** EF Core, iki dil bilen bir **tercümandır**. Sen Türkçe konuşursun (C# nesneleri), veritabanı başka bir dil konuşur (tablolar ve SQL). Tercüman ortada durur, ne dediğini çevirir, cevabı geri çevirir. **Migration** ise bu tercümanın tuttuğu **tadilat defteri**dir: binada yapılan her değişiklik tarih sırasıyla yazılır, hangi sayfaya kadar uygulandığı bilinir. Yeni bir usta geldiğinde defteri baştan okur ve binayı aynı hâle getirir.

---

## Bu Notta Ne Var

1. ORM nedir, hangi problemi çözer
2. EF Core mimarisi: provider, model, change tracker, query pipeline
3. `DbContext` — yaşam döngüsü ve DI'a kayıt
4. Code First akışı
5. Migration dosyasının anatomisi
6. Migration'ı geri alma, silme, birleştirme
7. Üretimde migration
8. Seed data
9. Konfigürasyon: connection string ve design-time factory
10. Database First ile karşılaştırma
11. Üçüncü yol: SQL script'i elle yönetmek
12. Temel CRUD'a kısa bakış

---

## 1. ORM Nedir, Hangi Problemi Çözer

> **Benzetme —** Kargo şirketi. Sen evinden bir paket göndereceksin. Adres bilirsin, alıcıyı bilirsin. Ama uçağın hangi ambara yükleneceğini, hangi barkodun basılacağını, hangi aktarma merkezinden geçeceğini bilmezsin ve bilmek zorunda da değilsin. Kargo şirketi senin "şu paketi şu adrese" cümleni kendi iç diline çevirir. Sen paketle konuşursun, o barkodla.

**Basitçe:** ORM, nesne dünyası ile tablo dünyası arasındaki çeviriyi otomatikleştiren kütüphanedir. Sen sınıflarla çalışırsın, o SQL yazar.

**Teknik olarak:** **ORM (Object-Relational Mapping — nesne-ilişkisel eşleme)** — Nesne yönelimli koddaki sınıfları ilişkisel veritabanındaki tablolara eşleyen ve iki yön arasındaki dönüşümü yöneten katman.

Çözdüğü asıl probleme **impedance mismatch (uyumsuzluk)** denir. İki modelin yapısal olarak birbirine tam oturmamasıdır:

| Nesne dünyası | İlişkisel dünya | Uyumsuzluk |
|---|---|---|
| Kalıtım (`Calisan : Kisi`) | Kalıtım yok | Tek tablo mu, ayrı tablo mu? |
| Referans (`siparis.Musteri`) | Foreign key (`MusteriId`) | Nesne bağlantısı ile anahtar aynı şey değil |
| Koleksiyon (`musteri.Siparisler`) | Ayrı tablo + `JOIN` | İki yönlü ilişkiyi kim tutar? |
| Kimlik: referans eşitliği | Kimlik: birincil anahtar | Aynı satırın iki kopyası aynı nesne mi? |
| `null` | `NULL` (üç değerli mantık) | `NULL = NULL` yanlıştır, `null == null` doğrudur |
| Tip sistemi (`decimal`, `DateTime`) | SQL tipleri (`DECIMAL(18,2)`, `datetime2`) | Hassasiyet ve aralık farkları |

Elle yazıldığında bu çeviri sıkıcıdır: bağlantı aç, komut kur, `SqlDataReader` ile satırları gez, her kolonu tek tek doğru özelliğe ata. Otuz kolonlu bir tabloda yüz satır tekrar eden kod demektir. EF Core'da aynı iş tek ifadeye iner:

```csharp
var liste = await context.Musteriler
    .Where(m => m.Sehir == sehir)
    .ToListAsync();
```

| ORM'in kazandırdığı | ORM'in bedeli |
|---|---|
| Tekrar eden eşleme kodu yok | Ürettiği SQL'i bilmezsen performans sorunu görmezsin |
| Derleme zamanı denetim (kolon adı yazım hatası yakalanır) | Öğrenme eğrisi: change tracker, tracking, lazy loading |
| Parametreli sorgu varsayılan — injection'a kapalı | Karmaşık raporda elle SQL kadar iyi olmayabilir |
| Farklı veritabanlarına taşınabilirlik | Soyutlama sızar: her provider aynı davranmaz |
| Değişiklik takibi ve transaction otomatik | "Sihir" hissi — ne olduğunu anlamak için içini bilmek gerekir |

> ORM "SQL bilmene gerek yok" demek değildir. Tam tersi: ORM'i iyi kullanmak için SQL'i **daha iyi** bilmen gerekir. Çünkü üretilen SQL'i okuyup değerlendirecek olan sensin.

**Bu benzetme şurada bozulur:** Kargo şirketine paketi verip unutabilirsin. EF Core'a sorguyu verip unutamazsın. Yazdığın LINQ ifadesi, arka planda üç tabloyu birleştiren ve on milyon satır tarayan bir SQL'e dönüşmüş olabilir. Kargo şirketi paketini geç teslim ederse sana haber verir; EF Core yavaş SQL üretirse hiçbir şey söylemez, sadece yavaştır.

---

## 2. EF Core Mimarisi

> **Benzetme —** İnşaat şantiyesi. **Proje (model)** kâğıt üzerindeki plandır: hangi oda nerede, hangi duvar nereye. **Şantiye şefi (change tracker)** kimin ne yaptığını not eder: şu duvar yıkıldı, şu pencere takıldı. **Usta (provider)** o bölgenin malzemesiyle ve yöntemiyle çalışır — Ankara'da başka, Kayseri'de başka. **İş akışı (query pipeline)** ise isteğin plandan ustaya kadar hangi elden geçtiğidir.

**Basitçe:** EF Core dört ana parçadan oluşur. Model, hangi sınıfın hangi tabloya karşılık geldiğini bilir. Provider, belirli bir veritabanına özel SQL'i üretir. Change tracker, bellekteki nesnelerin ne olduğunu takip eder. Query pipeline, LINQ ifadeni SQL'e çeviren boru hattıdır.

**Teknik olarak:**

| Parça | Görevi |
|---|---|
| **Model** | Entity'ler, ilişkiler, kolon tipleri ve kısıtların bellekteki haritası. `OnModelCreating` ile şekillenir, bir kez kurulur ve önbelleğe alınır |
| **Provider** | Veritabanına özel katman. SQL Server için `Microsoft.EntityFrameworkCore.SqlServer`, PostgreSQL için `Npgsql.EntityFrameworkCore.PostgreSQL` |
| **Change tracker** | Çekilen her entity'nin ilk hâlini saklar; `SaveChanges` anında farkı hesaplar |
| **Query pipeline** | LINQ ağacını alır, çevrilebilir kısmı SQL'e dönüştürür, sonucu nesneye eşler |

### Query pipeline adım adım

```csharp
var sonuc = await context.Siparisler
    .Where(s => s.MusteriId == 17 && !s.Iptal)
    .Select(s => new { s.SiparisId, s.SiparisTarihi })
    .ToListAsync();
```

Bu satırın arkasında sırayla şunlar olur:

1. **Expression tree oluşur.** LINQ ifadesi kod olarak değil, veri yapısı olarak tutulur.
2. **Ön işleme (preprocessing).** Navigasyon özellikleri genişletilir, alt sorgular düzenlenir.
3. **Çevrilebilirlik kontrolü.** Ağacın her düğümü SQL karşılığı var mı diye incelenir.
4. **SQL üretimi.** Provider, kendi sözdizimiyle metni kurar.
5. **Parametreleme.** Sabit değerler `@__musteriId_0` gibi parametrelere dönüşür.
6. **Çalıştırma ve materyalizasyon.** Satırlar okunur, nesneler oluşturulur, change tracker'a kaydedilir.

Üretilen SQL:

```sql
SELECT [s].[SiparisId], [s].[SiparisTarihi]
FROM [Siparisler] AS [s]
WHERE [s].[MusteriId] = @__musteriId_0 AND [s].[Iptal] = CAST(0 AS bit)
```

> **Çevrilemeyen ifade:** EF Core 3.0'dan itibaren, çevrilemeyen bir ifade **sessizce belleğe düşmez**; hata verir. Eskiden (EF Core 2.x) tüm tabloyu çekip filtreyi bellekte uygulardı ve kimse fark etmezdi. Bugün `InvalidOperationException` alırsın. Bu iyi bir değişikliktir: sorunu üretimde değil, ilk çalıştırmada görürsün.

Üretilen SQL'i görmek için loglamayı aç:

```csharp
optionsBuilder
    .UseSqlServer(connectionString)
    .LogTo(Console.WriteLine, LogLevel.Information)
    .EnableSensitiveDataLogging();     // parametre değerlerini de yazar — SADECE geliştirmede
```

> `EnableSensitiveDataLogging()` parametre değerlerini log'a yazar. Üretimde açık bırakırsan şifre, TC kimlik gibi veriler log dosyasına düşer. Geliştirme ortamıyla sınırla.

**Bu benzetme şurada bozulur:** Şantiye benzetmesinde usta değişse bile bina aynı olur. EF Core'da provider değişince davranış da değişir. SQL Server'da çalışan bir sorgu PostgreSQL'de çevrilemeyebilir; `string` karşılaştırması SQL Server'da büyük/küçük harf duyarsızken PostgreSQL'de duyarlıdır. "EF Core kullanıyorum, veritabanını istediğim zaman değiştiririm" cümlesi pratikte nadiren doğrudur.

---

## 3. `DbContext` — Yaşam Döngüsü ve Kayıt

> **Benzetme —** Banka gişesinde açılan bir işlem dosyası. Gişeye oturursun, memur senin için bir dosya açar; o dosyada yaptığın her hareket biriktirilir. İşin bitince dosya kapanır. Dosyayı günlerce açık tutmazsın, başkasıyla paylaşmazsın ve iki kişi aynı dosyaya aynı anda yazmaz.

**Basitçe:** `DbContext`, veritabanıyla olan bir oturumdur. Kısa ömürlüdür: aç, işini yap, kapat. Uzun süre açık tutulursa bellekte kayıt biriktirir ve yavaşlar.

**Teknik olarak:** `DbContext`, hem bağlantı yönetimini hem change tracker'ı hem de model bilgisini taşıyan sınıftır. Aynı zamanda bir **Unit of Work**tür: birden çok değişikliği toplar, `SaveChanges` ile tek transaction hâlinde uygular. İçindeki her `DbSet<T>`, bir tabloya açılan kapıdır ve aynı zamanda bir **Repository** gibi davranır.

```csharp
public class MagazaDbContext : DbContext
{
    public MagazaDbContext(DbContextOptions<MagazaDbContext> options) : base(options) { }

    public DbSet<Musteri>        Musteriler       => Set<Musteri>();
    public DbSet<Siparis>        Siparisler       => Set<Siparis>();
    public DbSet<SiparisDetay>   SiparisDetaylari => Set<SiparisDetay>();
    public DbSet<Urun>           Urunler          => Set<Urun>();
    public DbSet<Kategori>       Kategoriler      => Set<Kategori>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(MagazaDbContext).Assembly);
    }
}
```

### `OnConfiguring` mi, DI mi

İki yol vardır. `OnConfiguring`, context'in kendi içinde bağlantıyı kurar. DI ile kayıt ise yapılandırmayı dışarıda bırakır.

```csharp
// Yol 1 — OnConfiguring: hızlı ama bağlantı bilgisi sınıfa gömülü
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    if (!optionsBuilder.IsConfigured)
        optionsBuilder.UseSqlServer("Server=.;Database=MagazaDb;Trusted_Connection=True;TrustServerCertificate=True");
}
```

```csharp
// Yol 2 — DI: tercih edilen yol. Program.cs içinde
builder.Services.AddDbContext<MagazaDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Varsayilan")));
```

| Kriter | `OnConfiguring` | DI ile kayıt |
|---|---|---|
| Connection string nerede | Kodun içinde | `appsettings.json` / ortam değişkeni |
| Ortama göre değiştirme | Zor | Kolay (Development / Production) |
| Test edilebilirlik | Zayıf — InMemory'ye çevirmek zor | Kolay — `options` dışarıdan verilir |
| Konsol uygulaması / küçük araç | Yeterli | Gereksiz olabilir |

> `Scaffold-DbContext` ile üretilen context, varsayılan olarak `OnConfiguring` içine connection string'i **gömer** ve bir uyarı satırı bırakır. MvcCv'de bunu görmüşsündür. İlk iş o satırı silip yapılandırmayı `appsettings.json`a taşımaktır; aksi hâlde şifre Git'e girer.

### `AddDbContext` ve Scoped ömür

`AddDbContext` varsayılan olarak **Scoped** ömürle kaydeder. ASP.NET Core'da scope, bir HTTP isteğidir. Yani her istek kendi `DbContext`ini alır, istek bitince context dispose edilir. Bu, "kısa ömürlü olsun" kuralının otomatik uygulanmış hâlidir.

| Ömür | Davranış | Uygun mu |
|---|---|---|
| Scoped | İstek başına bir context | Evet — varsayılan ve doğru seçim |
| Transient | Her enjeksiyonda yeni context | Hayır — aynı istekte iki farklı change tracker olur |
| Singleton | Uygulama boyunca tek context | Kesinlikle hayır — bellek şişer, thread-safe değildir |

> **Kritik uyarı: `DbContext` thread-safe değildir.** Aynı context üzerinde iki işi paralel çalıştıramazsın. Klasik hata, `await` unutmaktır:

```csharp
// KÖTÜ: iki sorgu aynı context üzerinde aynı anda başlıyor
var t1 = context.Musteriler.ToListAsync();
var t2 = context.Urunler.ToListAsync();
await Task.WhenAll(t1, t2);   // InvalidOperationException: second operation started
```

```csharp
// İYİ: sırayla çalıştır
var musteriler = await context.Musteriler.ToListAsync();
var urunler    = await context.Urunler.ToListAsync();

// İYİ (gerçekten paralellik gerekiyorsa): her iş kendi context'ini alır
await using var ctx1 = await factory.CreateDbContextAsync();
await using var ctx2 = await factory.CreateDbContextAsync();
```

`IDbContextFactory<T>`, özellikle Blazor Server ve arka plan servislerinde bu amaçla kullanılır:

```csharp
builder.Services.AddDbContextFactory<MagazaDbContext>(options =>
    options.UseSqlServer(cs));
```

### `DbContextPool`

Context oluşturmak ücretsiz değildir. `AddDbContextPool`, kullanılmış context örneklerini atmak yerine havuzda tutar ve durumunu sıfırlayıp yeniden verir.

```csharp
builder.Services.AddDbContextPool<MagazaDbContext>(options =>
    options.UseSqlServer(cs), poolSize: 128);
```

Yüksek trafikte görünür kazanç sağlar, az trafikli uygulamada fark yaratmaz. İki kuralı var: context'in constructor'ı `DbContextOptions` dışında parametre almamalı, ve context'e kendi alan (field) değerlerini koyup güvenmemelisin — havuzdan gelen örnekte eski değer kalabilir.

**Bu benzetme şurada bozulur:** Banka dosyası benzetmesinde dosya kapanınca her şey biter. `DbContext` dispose edildiğinde ise ondan çektiğin **nesneler yaşamaya devam eder**. Elinde `Musteri` nesnesi kalır ama artık takip edilmez; `musteri.Siparisler` demeye kalkarsan lazy loading çalışmaz ve hata alırsın. Dosya kapandı, elindeki fotokopiler duruyor.

---

## 4. Code First Akışı

> **Benzetme —** Terziye gidip ölçü verirsin, elbise sana göre dikilir. Database First ise hazır giyim mağazasıdır: elbise zaten vardır, sen ona uyarsın. Code First'te ölçü sendedir — modelin gerçeği tanımlar, veritabanı ona uyar.

**Basitçe:** Önce C# sınıflarını yazarsın. Sonra EF Core'a "şu modele uygun veritabanını kur" dersin. Modelde bir şey değişince yeni bir migration üretir, veritabanını o noktaya taşırsın.

**Teknik olarak:** Akış dört adımdır.

**Adım 1 — Entity sınıfları:**

```csharp
public class Kategori
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public ICollection<Urun> Urunler { get; set; } = new List<Urun>();
}

public class Urun
{
    public int Id { get; set; }
    public string Ad { get; set; } = null!;
    public decimal Fiyat { get; set; }
    public int StokAdedi { get; set; }
    public bool Aktif { get; set; } = true;

    public int KategoriId { get; set; }          // foreign key
    public Kategori Kategori { get; set; } = null!;   // navigasyon özelliği
}
```

**Adım 2 — `DbContext`** (yukarıdaki gibi) ve gerekirse konfigürasyon:

```csharp
public class UrunConfiguration : IEntityTypeConfiguration<Urun>
{
    public void Configure(EntityTypeBuilder<Urun> builder)
    {
        builder.ToTable("Urunler");
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Ad).HasMaxLength(200).IsRequired();
        builder.Property(u => u.Fiyat).HasColumnType("decimal(18,2)");
        builder.HasIndex(u => u.Ad);
    }
}
```

**Adım 3 — Migration üretme:**

```
# .NET CLI (her ortamda çalışır)
dotnet ef migrations add IlkOlusturma

# Package Manager Console (Visual Studio)
Add-Migration IlkOlusturma
```

**Adım 4 — Veritabanına uygulama:**

```
dotnet ef database update
# PMC: Update-Database
```

Gerekli paketler:

| Paket | Ne için |
|---|---|
| `Microsoft.EntityFrameworkCore.SqlServer` | SQL Server provider'ı |
| `Microsoft.EntityFrameworkCore.Design` | Migration komutları için (design-time) |
| `Microsoft.EntityFrameworkCore.Tools` | PMC komutları (`Add-Migration` vb.) |
| `dotnet-ef` (global tool) | `dotnet ef` komutu için: `dotnet tool install --global dotnet-ef` |

> Migration adını anlamlı ver. `Migration1`, `Yeni`, `Test` gibi adlar altı ay sonra hiçbir şey ifade etmez. `UrunlereBarkodEklendi`, `SiparisIptalTarihiEklendi` gibi adlar dosya listesini okunur hâle getirir.

**Bu benzetme şurada bozulur:** Terzi benzetmesi tek kişilik çalışmayı varsayar. Takımda beş kişi aynı anda ölçü verirse migration'lar çakışır: iki kişi aynı anda migration üretir, ikisi de aynı snapshot'tan türer ve birleştirme (merge) sırasında çakışan bir `ModelSnapshot` dosyası çıkar. Çözümü 6. bölümde.

---

## 5. Migration Dosyasının Anatomisi

> **Benzetme —** Apartmanın tadilat defteri. Her sayfada bir iş yazılıdır: "3. kata duvar örüldü", tarihiyle birlikte. Her sayfanın arkasında da "bu işi geri almak için ne yapılacağı" not edilmiştir. Bir de en sonda binanın **son hâlinin planı** durur — defteri baştan okumadan binanın şu anki şeklini oradan görürsün.

**Basitçe:** Her migration üç şey üretir: ileri gitme adımları, geri alma adımları ve modelin o andaki tam fotoğrafı.

**Teknik olarak:** `dotnet ef migrations add BarkodEklendi` komutu `Migrations/` klasörüne şu dosyaları koyar:

| Dosya | İçeriği |
|---|---|
| `20260923120000_BarkodEklendi.cs` | `Up` ve `Down` metotları |
| `20260923120000_BarkodEklendi.Designer.cs` | O migration anındaki model meta verisi |
| `MagazaDbContextModelSnapshot.cs` | Modelin **güncel** tam hâli. Migration başına değil, tek dosya |

```csharp
public partial class BarkodEklendi : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Barkod",
            table: "Urunler",
            type: "nvarchar(50)",
            maxLength: 50,
            nullable: true);

        migrationBuilder.CreateIndex(
            name: "IX_Urunler_Barkod",
            table: "Urunler",
            column: "Barkod",
            unique: true,
            filter: "[Barkod] IS NOT NULL");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(name: "IX_Urunler_Barkod", table: "Urunler");
        migrationBuilder.DropColumn(name: "Barkod", table: "Urunler");
    }
}
```

`Down`, `Up`'ın tersidir ve sırası da terstir: `Up` önce kolon ekleyip sonra indeks kurduysa, `Down` önce indeksi düşürüp sonra kolonu siler.

### Snapshot neden kritik

EF Core yeni bir migration üretirken veritabanına **bakmaz**. Karşılaştırdığı iki şey vardır: mevcut model (kodun) ve `ModelSnapshot` dosyası (en son migration'daki hâl). Fark neyse `Up` ona göre yazılır.

Bunun iki sonucu vardır:

- Snapshot dosyasını silersen ya da elle bozarsan, EF Core her şeyi "yeni" sanır ve var olan tabloları yeniden oluşturmaya çalışır.
- Veritabanında elle yaptığın bir değişikliği EF Core görmez. Elle kolon eklersen model ile veritabanı sessizce ayrışır.

### `__EFMigrationsHistory` tablosu

`Update-Database`, uygulanan her migration'ın adını bu tabloya yazar.

```sql
SELECT * FROM __EFMigrationsHistory ORDER BY MigrationId;
```

| MigrationId | ProductVersion |
|---|---|
| 20260901093000_IlkOlusturma | 9.0.0 |
| 20260923120000_BarkodEklendi | 9.0.0 |

EF Core, `Migrations/` klasöründeki dosyalarla bu tabloyu karşılaştırır ve yalnızca eksik olanları uygular. Tablodaki satırı elle silersen migration bir daha uygulanmaya çalışılır ve büyük ihtimalle "column already exists" hatası alırsın.

### Elle SQL ekleme

Migration'a kendi SQL'ini de koyabilirsin. Veri taşıma, view veya stored procedure oluşturma bu yolla yapılır:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(name: "TamAd", table: "Musteriler", nullable: true);

    // Var olan satırlar için veriyi doldur
    migrationBuilder.Sql("UPDATE Musteriler SET TamAd = Ad + ' ' + Soyad;");

    migrationBuilder.Sql(@"
        CREATE VIEW vw_MusteriGenel AS
        SELECT MusteriId, TamAd, Sehir FROM Musteriler;");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("DROP VIEW IF EXISTS vw_MusteriGenel;");
    migrationBuilder.DropColumn(name: "TamAd", table: "Musteriler");
}
```

> **Sıra tuzağı:** Yeni bir kolonu `NOT NULL` yapmak istiyorsan tek adımda olmaz. Önce nullable ekle, sonra `migrationBuilder.Sql` ile doldur, sonra `AlterColumn` ile `nullable: false` yap. Doğrudan `NOT NULL` eklersen mevcut satırlar yüzünden komut hata verir.

**Bu benzetme şurada bozulur:** Tadilat defterinde her işin geri alınabileceği varsayılır. Migration'da `Down` her zaman gerçek bir geri alma değildir. `DropColumn` kolonu geri getirir ama **içindeki veriyi getirmez**. `Down` yapısal olarak tersini yapar, veriyi değil. Bu yüzden üretimde "bir önceki sürüme dönmek" çoğu zaman migration'ı geri almakla değil, yedekten dönmekle çözülür.

---

## 6. Migration'ı Geri Alma, Silme, Birleştirme

> **Benzetme —** Defterdeki yanlış sayfa. İki ihtimal var: sayfa henüz kimseye gösterilmediyse yırtıp atarsın. Ama sayfa fotokopiyle dağıtıldıysa yırtamazsın — üstüne düzeltme sayfası eklersin. Migration'da da ayrım aynıdır: **paylaşıldı mı, paylaşılmadı mı.**

**Basitçe:** Henüz kimseye gitmemiş bir migration'ı silebilirsin. Takıma ya da üretime gitmiş bir migration'ı silemezsin; onun üstüne yeni migration yazarsın.

**Teknik olarak:**

```
# Son migration'ı SİL (veritabanına uygulanmamışsa)
dotnet ef migrations remove
# PMC: Remove-Migration

# Veritabanını belirli bir migration'a geri al
dotnet ef database update IlkOlusturma
# PMC: Update-Database IlkOlusturma

# Tüm migration'ları geri al (veritabanını boşalt)
dotnet ef database update 0

# Uygulanmış migration'ları listele
dotnet ef migrations list
```

Doğru sıra şudur: önce veritabanını geri al, **sonra** migration dosyasını sil.

```
dotnet ef database update BirOncekiMigrationAdi   # 1) veritabanını geri sar
dotnet ef migrations remove                        # 2) dosyayı sil
```

Ters sırada yaparsan `__EFMigrationsHistory` tablosunda artık dosyası olmayan bir kayıt kalır ve EF Core "model ile veritabanı uyuşmuyor" der.

| Durum | Ne yapılır |
|---|---|
| Migration üretildi, henüz uygulanmadı | `migrations remove` — dosya silinir, snapshot geri alınır |
| Uygulandı ama sadece senin makinende | Önce `database update <öncekiAd>`, sonra `migrations remove` |
| Takıma push edildi | **Silme.** Düzeltmeyi yeni bir migration olarak ekle |
| Üretimde çalışıyor | **Kesinlikle silme.** İleri yönlü düzeltme migration'ı yaz |

### Birleştirme (merge) çakışmaları

İki kişi aynı anda migration ürettiğinde çakışan dosya neredeyse her zaman `ModelSnapshot`tur; çünkü tek dosyadır ve ikisi de onu günceller.

Çözüm sırası:

1. `ModelSnapshot` çakışmasını elle çözmeye çalışma.
2. Kendi migration'ını `migrations remove` ile geri al.
3. Arkadaşının migration'ını `pull` ile al.
4. Kendi değişikliğin için migration'ı **yeniden üret**.

Böylece snapshot doğal olarak son hâline gelir. Elle birleştirilen snapshot dosyaları, bir süre sonra "bu kolon neden iki kez ekleniyor" tipi hatalar üretir.

> **Takım kuralı:** Migration dosyaları üretildikten sonra **değiştirilmez**. Uygulanmış bir migration'ın `Up` metodunu düzenlersen, senin makinende çalışan veritabanı ile arkadaşınınki farklılaşır ve kimse farkı göremez. Yanlış varsa yeni migration üret.

**Bu benzetme şurada bozulur:** Defterden sayfa yırtmak ile migration silmek arasında görünmeyen bir fark var: sayfayı yırttığında bina değişmez, ama `migrations remove` **snapshot dosyasını da** geri alır. Yani sadece bir sayfayı değil, binanın planını da eski hâline döndürür. Bu yüzden komut çalışırken kaynak kontrolünün temiz olması önemlidir — neyin değiştiğini `git diff` ile görebilmelisin.

---

## 7. Üretimde Migration

> **Benzetme —** Apartmanda tadilat. Kendi evinde duvarı istediğin gibi yıkarsın. Ama bina yönetimi ortak alanda iş yapacaksa önce proje sunar, onay alır, komşuları haber eder ve iş bittikten sonra da geri dönülemeyeceğini bilir. Üretim veritabanı ortak alandır.

**Basitçe:** Geliştirme makinende `Update-Database` demek yeterlidir. Üretimde ise komut çalıştırmak yerine, uygulanacak SQL'i önce üretip incelemek, sonra kontrollü biçimde çalıştırmak gerekir.

**Teknik olarak:** İki yaygın yaklaşım var.

**1) SQL script üretme (tercih edilen):**

```
# Tüm migration'lar için script
dotnet ef migrations script

# İki migration arası
dotnet ef migrations script IlkOlusturma BarkodEklendi

# Idempotent: hangisi uygulanmışsa atlar, eksik olanı uygular
dotnet ef migrations script --idempotent --output deploy.sql
```

`--idempotent` ile üretilen script, her migration'ı bir kontrol bloğuna sarar:

```sql
IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20260923120000_BarkodEklendi')
BEGIN
    ALTER TABLE [Urunler] ADD [Barkod] nvarchar(50) NULL;
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20260923120000_BarkodEklendi', N'9.0.0');
END;
GO
```

Bu script'i aynı veritabanında iki kez çalıştırsan da sorun çıkmaz. Hangi sunucunun hangi noktada olduğunu bilmediğin durumlarda hayat kurtarır.

**2) `Database.Migrate()` çağırmak:**

```csharp
// Program.cs içinde — kolay ama riskli
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<MagazaDbContext>();
    db.Database.Migrate();
}
```

| Riski | Neden |
|---|---|
| Çoklu örnek (instance) yarışı | Üç sunucu aynı anda açılırsa üçü de migration uygulamaya çalışır; kilit ve hata |
| Yüksek yetki gerekir | Uygulamanın çalışma hesabı `ALTER TABLE` yetkisine sahip olmak zorunda kalır |
| Geri dönüş yok | Hatalı migration uygulanır, uygulama ayağa kalkar, sorun canlıda görülür |
| SQL önceden görülmez | Üretimde ne çalışacağını kimse incelememiştir |
| Uzun süren iş | Büyük tabloda indeks kurulumu uygulamanın açılışını dakikalarca bekletir |

> `Database.Migrate()` küçük tek sunuculu projelerde ve geliştirme ortamında kabul edilebilir. Ciddi üretimde tercih edilen yol, script'i CI/CD hattında ayrı bir adım olarak çalıştırmaktır. Uygulamanın kendisi şema değiştirmemelidir.

> **`EnsureCreated()` ile karıştırma.** `Database.EnsureCreated()` veritabanı yoksa modele göre oluşturur ama **migration geçmişi tutmaz**. Sonradan migration'a geçemezsin. Sadece testlerde ve atılacak veritabanlarında kullanılır. `EnsureCreated` ve `Migrate` aynı projede birlikte kullanılmaz.

### Geri alınamayan değişiklikler

Bazı şema değişiklikleri veri kaybeder ve `Down` bunu telafi edemez. EF Core bunların bir kısmında uyarı basar:

| Değişiklik | Sonuç |
|---|---|
| Kolon silme | Veri gider, `Down` geri getiremez |
| `nvarchar(200)` → `nvarchar(50)` | Uzun değerler kesilir ya da komut hata verir |
| Kolon tipini daraltma (`bigint` → `int`) | Taşan değerlerde hata |
| Kolonu `NOT NULL` yapma | Mevcut `NULL` satırlar varsa hata |
| Kolon adını değiştirme | EF Core bazen "sil + ekle" üretir; veriyi kaybedersin |

Son madde özellikle sinsidir. Kolon adı değişikliğinde üretilen migration'ı **her zaman aç ve oku**. `DropColumn` + `AddColumn` gördüysen bunu elle `RenameColumn` ile değiştir:

```csharp
// EF Core'un ürettiği (veri kaybettirir)
migrationBuilder.DropColumn(name: "Adres", table: "Musteriler");
migrationBuilder.AddColumn<string>(name: "AcikAdres", table: "Musteriler", nullable: true);

// Doğrusu
migrationBuilder.RenameColumn(name: "Adres", table: "Musteriler", newName: "AcikAdres");
```

**Bu benzetme şurada bozulur:** Apartman tadilatında iş bittiğinde görürsün: duvar yerinde mi, değil mi. Veritabanı değişikliğinde ise hata çoğu zaman **o anda görünmez**. `nvarchar(50)`'ye daraltılan kolon o gün sorun çıkarmaz; üç hafta sonra uzun bir adres girilmeye çalışıldığında patlar. Bu yüzden script'i okumak, çalıştırdıktan sonra bakmaktan daha değerlidir.

---

## 8. Seed Data

> **Benzetme —** Yeni açılan bir dükkânın rafına konan ilk ürünler. Dükkân boş açılmaz: kasa fişi numaraları, ödeme tipleri, iller listesi gibi "olmazsa olmaz" kayıtlar en baştan konur. Bunlar satılacak mal değil, dükkânın çalışması için gereken sabit envanterdir.

**Basitçe:** Seed data, uygulamanın çalışması için baştan var olması gereken kayıtlardır: kategoriler, roller, il listesi, ayar satırları. Test verisi değildir.

**Teknik olarak:** İki yol vardır.

**1) `HasData` — migration'a gömülü seeding:**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Kategori>().HasData(
        new Kategori { Id = 1, Ad = "Elektronik" },
        new Kategori { Id = 2, Ad = "Giyim" },
        new Kategori { Id = 3, Ad = "Kitap" }
    );
}
```

`Add-Migration` çalıştırdığında bu veriler migration'ın `Up` metoduna `InsertData` olarak girer:

```csharp
migrationBuilder.InsertData(
    table: "Kategoriler",
    columns: new[] { "Id", "Ad" },
    values: new object[,] { { 1, "Elektronik" }, { 2, "Giyim" }, { 3, "Kitap" } });
```

`HasData`'nın sınırları katıdır:

| Sınır | Açıklama |
|---|---|
| **Birincil anahtar elle verilmeli** | `Id` otomatik üretilse bile burada sabit yazılır |
| Anahtar değişirse sil-ekle üretir | `Id`yi 1'den 10'a çevirirsen migration `DELETE` + `INSERT` yazar |
| Navigasyon özelliği kullanılamaz | İlişkili kaydı `Kategori = k` ile değil, `KategoriId = 1` ile verirsin |
| Dinamik değer olmaz | `DateTime.Now`, `Guid.NewGuid()` yazarsan her migration'da fark görünür ve sonsuz migration üretirsin |
| Veritabanını okuyamaz | "Bu kayıt yoksa ekle" mantığı kurulamaz |
| Kullanıcı verisini ezer | Üretimde biri o satırı değiştirdiyse migration onu geri alabilir |

> `HasData` içinde `DateTime.Now` kullanma. Her `Add-Migration` çalıştırdığında değer değişir, EF Core bunu "veri değişmiş" sayar ve boş yere `UpdateData` üretir. Sabit bir tarih yaz: `new DateTime(2026, 1, 1)`.

**2) Uygulama başlangıcında seeding (esnek yol):**

Karmaşık, koşullu ya da ilişkili veri için `HasData` yetmez. Bu durumda başlangıçta çalışan bir seeder yazılır:

```csharp
public static class DbSeeder
{
    public static async Task SeedAsync(MagazaDbContext context)
    {
        if (await context.Kategoriler.AnyAsync())
            return;                       // zaten dolu, dokunma

        var elektronik = new Kategori { Ad = "Elektronik" };
        context.Kategoriler.Add(elektronik);

        context.Urunler.Add(new Urun
        {
            Ad = "Kulaklık",
            Fiyat = 899.90m,
            StokAdedi = 25,
            Kategori = elektronik          // navigasyon kullanılabilir
        });

        await context.SaveChangesAsync();
    }
}
```

```csharp
// Program.cs
using (var scope = app.Services.CreateScope())
{
    var ctx = scope.ServiceProvider.GetRequiredService<MagazaDbContext>();
    await DbSeeder.SeedAsync(ctx);
}
```

| Kriter | `HasData` | Başlangıç seeder'ı |
|---|---|---|
| Sabit referans verisi (roller, iller) | Uygun | Gereksiz |
| İlişkili / hesaplanmış veri | Uygun değil | Uygun |
| Migration script'ine dâhil | Evet | Hayır |
| Koşullu ("yoksa ekle") | Yapılamaz | Yapılabilir |
| Üretimde kullanıcı verisini ezme riski | Var | Yok (kontrol sende) |

> **Sürüm notu:** EF Core 9 ile `UseSeeding` ve `UseAsyncSeeding` yapılandırma seçenekleri geldi. Başlangıç seeder'ını `DbContext` yapılandırmasının parçası hâline getirir ve `EnsureCreated`/`Migrate` ile birlikte çalışır. `HasData`'nın yerini almaz; ikinci yolu resmîleştirir.

**Bu benzetme şurada bozulur:** Dükkân rafına konan ilk ürünler bir kez konur, sonra dükkân sahibi onları istediği gibi değiştirir. `HasData` böyle davranmaz: o satırlar modelin parçası sayılır. Üretimde biri "Elektronik" kategorisinin adını "Teknoloji" yaptıysa, bir sonraki migration onu `UpdateData` ile geri çevirebilir. `HasData` ile konan veri, veri değil **şemanın parçası** gibi davranır.

---

## 9. Konfigürasyon: Connection String ve Design-Time Factory

> **Benzetme —** Elektrik tesisatı ile sigorta kutusu. Kablolar duvarın içindedir, ama açma-kapama tek bir kutudan yapılır. Ampulü değiştirmek için duvarı delmezsin; kutuya gidersin. Connection string de öyle: koda gömülmez, tek bir yapılandırma noktasında durur.

**Basitçe:** Bağlantı bilgisi `appsettings.json` dosyasında durur, koda yazılmaz. Geliştirme ve üretim için farklı dosyalar kullanılır. Şifre içeren bağlantılar Git'e girmez.

**Teknik olarak:**

```json
{
  "ConnectionStrings": {
    "Varsayilan": "Server=.;Database=MagazaDb;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "Logging": { "LogLevel": { "Default": "Information" } }
}
```

```csharp
builder.Services.AddDbContext<MagazaDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Varsayilan"),
        sql => sql.EnableRetryOnFailure(maxRetryCount: 3)));
```

Yapılandırma kaynakları sırayla okunur; sonraki öncekini ezer:

| Kaynak | Kullanım |
|---|---|
| `appsettings.json` | Ortak ayarlar, gerçek şifre içermez |
| `appsettings.Development.json` | Geliştirici makinesi |
| **User Secrets** | Geliştiricinin kendi şifresi — proje klasöründe değil, kullanıcı profilinde durur |
| Ortam değişkeni | Üretim. `ConnectionStrings__Varsayilan` biçiminde yazılır |
| Key Vault / Secret Manager | Kurumsal üretim ortamı |

### `IDesignTimeDbContextFactory`

`dotnet ef` komutları çalışırken uygulaman **ayağa kalkmaz**. EF Core, `DbContext`i nasıl oluşturacağını bulmak için önce `Program.cs`teki host kurulumunu taklit etmeye çalışır. Bazı projelerde bu başarısız olur — örneğin `DbContext` bir sınıf kütüphanesindeyse, ya da `Program.cs` karmaşıksa. O zaman şu hatayı görürsün:

```
Unable to create a 'DbContext' of type 'MagazaDbContext'.
```

Çözüm, design-time için ayrı bir fabrika yazmaktır:

```csharp
public class MagazaDbContextFactory : IDesignTimeDbContextFactory<MagazaDbContext>
{
    public MagazaDbContext CreateDbContext(string[] args)
    {
        var config = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false)
            .AddJsonFile("appsettings.Development.json", optional: true)
            .Build();

        var options = new DbContextOptionsBuilder<MagazaDbContext>()
            .UseSqlServer(config.GetConnectionString("Varsayilan"))
            .Options;

        return new MagazaDbContext(options);
    }
}
```

Bu sınıf yalnızca `dotnet ef` komutları tarafından kullanılır; uygulama çalışırken devreye girmez.

> Çok projeli çözümde (`Data` katmanı ayrı bir sınıf kütüphanesi) komutlara iki proje de söylenmelidir:
> `dotnet ef migrations add Ad --project src/Magaza.Data --startup-project src/Magaza.Web`

**Bu benzetme şurada bozulur:** Sigorta kutusu benzetmesi tek bir merkez olduğunu söyler. Gerçekte yapılandırma **katmanlıdır** ve hangi değerin kazandığını bilmek gerekir. `appsettings.json`da bir bağlantı yazarsın, ortam değişkeninde başka biri durur, uygulama üçüncüsünü kullanır. "Ayarı değiştirdim ama hiçbir şey olmadı" şikâyetinin sebebi neredeyse her zaman budur: daha öncelikli bir kaynak seninkini eziyordur.

---

## 10. Database First ile Karşılaştırma

> **Benzetme —** Ev yaptırmak ile hazır daire almak. Terziye ölçü verip elbise diktirmek (Code First) ile hazır giyim mağazasından beden seçmek (Database First). İkisi de doğru cevaptır; soru "hangisi daha iyi" değil, **"evi kim inşa ediyor"**dur.

**Basitçe:** Code First'te gerçek kaynağı C# kodudur, veritabanı ona uyar. Database First'te gerçek kaynağı veritabanıdır, kod ona uyar. Sen MvcCv'de ikincisini yaptın.

**Teknik olarak:** MvcCv'de izlediğin akış şuydu:

```
# Var olan veritabanından sınıfları ve DbContext'i üret
Scaffold-DbContext "Server=.;Database=CvDb;Trusted_Connection=True;TrustServerCertificate=True" `
    Microsoft.EntityFrameworkCore.SqlServer `
    -OutputDir Models -Context CvDbContext -Force

# .NET CLI karşılığı
dotnet ef dbcontext scaffold "Server=.;Database=CvDb;Trusted_Connection=True;TrustServerCertificate=True" `
    Microsoft.EntityFrameworkCore.SqlServer --output-dir Models --context CvDbContext --force
```

Komut veritabanını okur, her tablo için bir sınıf ve tüm `DbSet`leri içeren bir `DbContext` üretir. Veritabanında bir kolon eklendiğinde komutu `-Force` ile tekrar çalıştırırsın ve dosyalar **yeniden yazılır**.

### Yan yana karşılaştırma

| Konu | Code First | Database First |
|---|---|---|
| **Veritabanının sahibi kim** | Kod. Model gerçeği tanımlar, şema ondan türer | Veritabanı. Kod ondan türer |
| **Mevcut veritabanı** | Yeni proje için ideal; var olan büyük şemaya uydurmak zahmetli | Doğal seçim. Eski ve büyük şemalarda tek makul yol |
| **Takım çalışması** | Migration dosyaları Git'te izlenir; kim ne değiştirmiş görünür | Değişiklik veritabanında yapılır; Git'te sadece üretilmiş kod görünür |
| **Sürüm takibi** | `__EFMigrationsHistory` ile her ortam aynı noktaya gelir | Ayrı bir şema sürümleme aracı gerekir |
| **Özelleştirme** | Entity sınıfı senindir, istediğini yazarsın | Üretilen dosyaya yazdığın her şey silinir; `partial class` şart |
| **Yeniden üretme** | Söz konusu değil | `-Force` tüm dosyaları ezer; elle düzeltmeler kaybolur |
| **Veritabanı yöneticisi (DBA) varsa** | Sürtüşme çıkar — şemayı kod belirliyor | Uyumlu — DBA şemayı yönetir |
| **Saklı yordam ağırlıklı sistem** | Zorlayıcı | Doğal |
| **Öğrenme** | Migration kavramını öğrenmek gerekir | Tek komut, hızlı başlangıç |

### `partial class` ile özelleştirme

Database First'te üretilen sınıfa doğrudan kod yazamazsın — bir sonraki scaffold'da silinir. Çözüm, `partial` anahtar kelimesidir: üretilen dosya da senin dosyan da aynı sınıfın parçasıdır.

```csharp
// Models/Musteri.cs — ÜRETİLMİŞ DOSYA, elleme
public partial class Musteri
{
    public int MusteriId { get; set; }
    public string Ad { get; set; } = null!;
    public string Soyad { get; set; } = null!;
}
```

```csharp
// Models/Musteri.Ozel.cs — SENİN DOSYAN, scaffold buna dokunmaz
public partial class Musteri
{
    public string TamAd => $"{Ad} {Soyad}";
}
```

Doğrulama (validation) attribute'ları için de aynı sorun vardır; çözümü `MetadataType` ya da ayrı bir ViewModel kullanmaktır.

> **Yeniden scaffold etme problemi:** Database First'in en can sıkıcı tarafı budur. Veritabanına tek bir kolon eklendiğinde bütün model klasörünü yeniden üretirsin. `-Force` kullanmazsan komut "dosya zaten var" der; kullanırsan elle yaptığın her düzeltme gider. `--table Musteriler` ile tek tabloyu üretmek kısmen yardımcı olur ama `DbContext` yine baştan yazılır. Bu yüzden Database First projelerinde **üretilen klasöre hiç dokunmama** disiplini şarttır.

### Hangi durumda hangisi

| Durum | Seçim |
|---|---|
| Sıfırdan yeni proje | Code First |
| Yıllardır çalışan, büyük mevcut veritabanı | Database First |
| Veritabanını başka bir ekip / DBA yönetiyor | Database First |
| Şema sık değişiyor, takım hızlı ilerliyor | Code First |
| Veritabanı başka uygulamalarla paylaşılıyor | Database First ya da elle script |
| Testte InMemory / SQLite kullanılacak | Code First |
| Şema saklı yordam ve view ağırlıklı | Database First |

> Bootcamp'te büyük ihtimalle Code First göreceksin, çünkü öğretmesi ve değerlendirmesi kolaydır. İş hayatında ise ikisiyle de karşılaşırsın. MvcCv deneyimin burada avantaj: Database First'ün nasıl hissettirdiğini zaten biliyorsun.

**Bu benzetme şurada bozulur:** "Ev yaptırmak ile hazır daire almak" ikisinden birini seçtiğini varsayar. Gerçek projelerde ikisi karışır. Code First projesinde bile view'lar, stored procedure'ler ve indeksler çoğu zaman elle SQL olarak yazılıp `migrationBuilder.Sql` ile migration'a konur. Database First projesinde de bazı tablolar elle eklenip `partial` sınıflarla desteklenir. Saf bir yaklaşım nadirdir.

---

## 11. Üçüncü Yol: SQL Script'i Elle Yönetmek

> **Benzetme —** Muhtarlıkta tutulan el yazısı defter. Bilgisayar programı yok, ama defter düzenlidir: her kayıt tarihli, sıralı ve tek elden yazılır. Yavaştır, ama kimse defterin ne dediğini tartışmaz.

**Basitçe:** Ne migration ne scaffold. Veritabanı şemasını numaralandırılmış SQL dosyalarıyla, elle yönetirsin. Kod tarafında EF Core'u sadece sorgulamak için kullanırsın.

**Teknik olarak:** Şema değişikliklerini sıralı dosyalarda tutarsın:

```
db/migrations/
  001_ilk_semalar.sql
  002_urunlere_barkod.sql
  003_siparis_indeksleri.sql
  004_vw_siparis_ozet.sql
```

Hangi dosyanın uygulandığını kendi tuttuğun bir sürüm tablosundan takip edersin — `__EFMigrationsHistory`'nin elle yazılmış hâli.

Bu yaklaşımı otomatikleştiren araçlar vardır: **DbUp**, **Flyway**, **Liquibase**, **RoundhousE**. Hepsi aynı fikri uygular — sıralı script'ler ve bir sürüm tablosu.

| Artı | Eksi |
|---|---|
| Çalışacak SQL'in tamamı görünür ve incelenebilir | Her değişikliği elle yazarsın |
| Veritabanına özel her şey yazılabilir (partition, filtered index, trigger) | Model ile şema arasındaki uyumu sen sağlarsın |
| DBA ile çalışmaya uygun | Yazım hatası çalışma zamanında çıkar |
| Birden çok uygulamanın paylaştığı veritabanı için doğal | Code First'ün hızından vazgeçersin |
| Veri taşıma (data migration) tam kontrolle yazılır | Araç kurulumu ve disiplin gerektirir |

**Nerede seçilir:** Veritabanı birden çok uygulama tarafından paylaşılıyorsa, şirkette DBA onayı gerekiyorsa, ya da şema EF Core'un modelleyemeyeceği özellikler içeriyorsa. Kurumsal ortamlarda bu yaklaşım Code First'ten daha yaygındır.

**Bu benzetme şurada bozulur:** El yazısı defterde hata yaparsan bir sonraki okuyucu fark eder. SQL script'inde hata yaparsan kimse fark etmez — script sessizce yanlış şeyi yapar. Migration'ın sağladığı asıl güvence "SQL yazmaktan kurtulmak" değil, **model ile şemanın ayrışmasını fark ettirmektir**. Elle script yönetiminde o güvenceyi kendin kurmak zorundasın.

---

## 12. Temel CRUD'a Kısa Bakış

> **Benzetme —** Alışveriş sepeti. Ürünleri sepete atarsın, birini çıkarırsın, birinin adedini değiştirirsin. Kasaya gidene kadar hiçbiri gerçekleşmemiştir. Kasa, `SaveChanges()`tır.

**Basitçe:** Ekleme, güncelleme ve silme işlemleri anında veritabanına gitmez. Bellekte biriktirilir, `SaveChanges()` dediğinde hepsi birden uygulanır.

**Teknik olarak:**

```csharp
// CREATE
var urun = new Urun { Ad = "Klavye", Fiyat = 1250.00m, StokAdedi = 40, KategoriId = 1 };
context.Urunler.Add(urun);
await context.SaveChangesAsync();
// urun.Id burada dolmuş olur

// READ
var tumu     = await context.Urunler.ToListAsync();
var tek      = await context.Urunler.FindAsync(5);
var filtreli = await context.Urunler
                    .Where(u => u.Aktif && u.Fiyat < 2000)
                    .OrderBy(u => u.Ad)
                    .ToListAsync();

// UPDATE — takip edilen nesneyi değiştir, Update() çağırmaya gerek yok
var guncellenecek = await context.Urunler.FindAsync(5);
guncellenecek!.Fiyat = 1399.00m;
await context.SaveChangesAsync();

// DELETE
var silinecek = await context.Urunler.FindAsync(7);
context.Urunler.Remove(silinecek!);
await context.SaveChangesAsync();
```

| Metot | Ne yapar |
|---|---|
| `Add` / `AddRange` | Entity'yi `Added` durumuna alır |
| `Update` | Entity'yi `Modified` yapar — **tüm kolonlar** güncellenir |
| `Remove` / `RemoveRange` | Entity'yi `Deleted` durumuna alır |
| `Find` | Önce change tracker'a bakar, yoksa veritabanına gider |
| `SaveChanges` | Bekleyen tüm değişiklikleri tek transaction'da uygular |

> **`Update()` çoğu zaman gereksizdir.** Entity'yi sorguyla çektiysen zaten takip ediliyordur; özelliğini değiştirip `SaveChanges()` demen yeter. `Update()`, yalnızca takip edilmeyen (detached) bir nesneyi geri yazarken gerekir — tipik olarak MVC'de formdan gelen nesnede. Ve `Update()` tüm kolonları yazar; formda olmayan alanları sıfırlayabilir.

> Bu konunun tamamı — change tracker durumları, `AsNoTracking`, over-posting, soft delete — Hafta 4'teki `04-EF-Core-CRUD-ve-Change-Tracking.md` notunda işleniyor. İlişkiler, navigasyon özellikleri ve Fluent API ise yarınki notun konusu.

**Bu benzetme şurada bozulur:** Sepetten bir ürünü çıkarınca sepet küçülür; sen de görürsün. EF Core'da `Remove` çağırdığında nesne listeden kaybolmaz, sadece durumu `Deleted` olur. Aynı `DbContext` üzerinden tekrar sorgularsan onu hâlâ görebilirsin. Sepetin görüntüsü ile içeriği arasındaki fark, EF Core'da değişiklik takibinin nasıl çalıştığını anlamadan çözülmez.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| ORM | Nesneler ile tabloları eşleyen katman |
| Impedance mismatch | Nesne modeli ile ilişkisel modelin yapısal uyumsuzluğu |
| Provider | Belirli bir veritabanı için SQL üreten EF Core bileşeni |
| `DbContext` | Veritabanı oturumu; aynı zamanda Unit of Work |
| `DbSet<T>` | Bir entity tipine açılan sorgu ve değişiklik kapısı |
| Change tracker | Çekilen entity'lerin ilk hâlini tutup farkı hesaplayan bileşen |
| Query pipeline | LINQ ağacını SQL'e çeviren işlem zinciri |
| Scoped ömür | İstek başına bir örnek üretilen DI ömrü |
| `DbContextPool` | Context örneklerini yeniden kullanan havuz |
| Code First | Modelin veritabanını tanımladığı yaklaşım |
| Database First | Veritabanının modeli tanımladığı yaklaşım |
| `Scaffold-DbContext` | Mevcut veritabanından entity ve context üreten komut |
| Migration | Model değişikliğini şema değişikliğine çeviren dosya |
| `ModelSnapshot` | Modelin en güncel hâlini tutan tek dosya |
| `__EFMigrationsHistory` | Uygulanan migration'ların kaydedildiği tablo |
| `HasData` | Migration'a gömülen sabit veri tanımı |
| `IDesignTimeDbContextFactory` | `dotnet ef` komutlarının context üretmesini sağlayan fabrika |
| `partial class` | Aynı sınıfı birden çok dosyaya bölme; scaffold'da özelleştirmenin yolu |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `05-Iliskiler-ve-Fluent-API.md` (Cuma)
