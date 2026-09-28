# Hafta 3 · Ek Not — Dapper'a Giriş

**Okuma süresi:** ~20 dk
**Neden bu konu:** Yol haritasında Hafta 3 Pazar satırında "Dapper'a giriş" yazıyor ve bootcamp'te Proje 7 için lazım olacak. Hafta boyunca EF Core'u öğrendin; Dapper aynı işi bambaşka bir felsefeyle yapıyor. Gerçek projelerde ikisi genellikle yan yana durur: CRUD'u EF Core, ağır raporlama sorgularını Dapper yazar. Bu notu bitirdiğinde "hangi sorguyu hangisiyle yazayım" sorusuna cevap verebiliyor olacaksın.

---

## Önce Basitçe

EF Core'u bir tercüman gibi düşün. Sen LINQ yazarsın, o bunu SQL'e çevirir, veritabanına gönderir, gelen satırları tekrar C# nesnelerine dönüştürür. Bunu yaparken çok iş üstlenir: hangi nesnenin değiştiğini takip eder, ilişkileri kurar, migration üretir, farklı veritabanları için farklı SQL yazar. Bu kolaylığın bedeli, araya kalın bir katman girmesidir.

Dapper tam ters yerden başlar. Der ki: SQL'i zaten biliyorsun, en iyi sorguyu sen yazarsın; benim işim o sorgunun sonucunu senin sınıflarına doldurmak. Tercümanlık yapmaz, sadece **gelen satırları nesneye eşler**. Ona bu yüzden micro-ORM deniyor — ORM'in sadece eşleme parçasını yapıyor.

Pratikte anlamı şu: Dapper öğrenmek bir öğleden sonranı alır. Toplam yedi sekiz metot var, hepsi `IDbConnection` üzerine takılmış birer genişletme metodu. `DbContext`, `OnModelCreating`, migration klasörü — hiçbiri yok. Bedeli de net: Dapper senin için hiçbir şey hatırlamaz. Nesneyi değiştirdin diye `UPDATE` üretmez, tablo değişince migration yazmaz, veritabanını değiştirince sorgularını taşımaz.

Bu yüzden "Dapper mı EF Core mu" çoğu zaman yanlış sorudur. Doğru soru şu: **bu ekranda hangisi?** Müşteri kaydı düzenleyen bir ekranda EF Core'un change tracking'i sana hediyedir. Dokuz tabloyu birleştiren bir yönetim paneli raporunda ise o SQL'i elinle yazmak hem daha hızlı hem daha okunaklı olur. Şimdi detaya iniyoruz.

> **Ana benzetme:** EF Core paket tur gibidir — uçak, otel, transfer, rehber dahil; rahat ama rotayı sen belirlemezsin. Dapper kendi aracınla yola çıkmaktır: navigasyonu sen kurarsın, yolu sen seçersin. Dapper'ın tek yaptığı, getirdiklerini düzgünce bagaja yerleştirmektir.

---

## Bu Notta Ne Var

1. Micro-ORM nedir, Dapper'ın felsefesi
2. Kurulum ve bağlantı yönetimi
3. Temel API
4. Parametreler ve SQL injection
5. Eşleme ve çoklu eşleme
6. EF Core ile karşılaştırma
7. İkisini birlikte kullanmak
8. Dapper'ın yapmadıkları
9. Repository pattern ile Dapper
10. Ne zaman Dapper kullanma

---

## 1. Micro-ORM Nedir, Dapper'ın Felsefesi

> **Benzetme —** Terziye gitmekle hazır giyim mağazasına gitmek arasındaki fark gibi. Hazır giyimde beden söylersin, gerisi hallolur; ama omuz genişse geniş kalır. Terzide kumaşı sen seçersin, ölçüyü sen verirsin, terzi sadece diker. Dapper terzidir: kumaş senin SQL'in, dikiş ise satırları nesneye çevirme işi.

**Basitçe:** Tam ORM senin yerine SQL yazar. Micro-ORM yazmaz, sadece sonucu nesneye çevirir. Tüm fark bu cümlede.

**Teknik olarak:** Bir ORM (Object-Relational Mapper, nesne-ilişkisel eşleyici) iki dünya arasındaki uyumsuzluğu kapatır: nesne dünyasında referanslar, ilişkisel dünyada yabancı anahtarlar vardır. Tam ORM bu uyumsuzluğun tamamını üstlenir; micro-ORM sadece en mekanik kısmını — `IDataReader`'dan çıkan sütunları bir sınıfın property'lerine yazma kısmını.

Dapper'ın kendi tanımı şudur: `IDbConnection` üzerine eklenmiş bir genişletme metodu (extension method) kümesi. Kendi bağlantı tipi yok, kendi sorgu dili yok. ADO.NET'in üstüne ince bir tabaka.

```csharp
const string sql = "SELECT Id, Ad, Email FROM Musteriler WHERE Sehir = @Sehir";

// Elle ADO.NET
using var cmd = new SqlCommand(sql, conn);
cmd.Parameters.AddWithValue("@Sehir", "Kayseri");
using var reader = cmd.ExecuteReader();
var liste = new List<Musteri>();
while (reader.Read())
    liste.Add(new Musteri { Id = reader.GetInt32(0), Ad = reader.GetString(1), Email = reader.GetString(2) });

// Dapper — aynı iş, aradaki gürültü yok
var liste2 = conn.Query<Musteri>(sql, new { Sehir = "Kayseri" }).ToList();
```

Hızının sırrı: ilk çağrıda, dönen sütunlara bakarak nesneyi dolduran bir metot **üretir** ve önbelleğe alır. Sonraki çağrılarda reflection maliyeti yoktur; elle yazılmış `reader.GetInt32(0)` koduyla neredeyse aynı hızda çalışır.

**Bu benzetme şurada bozulur:** Terzi ölçünü alır, Dapper almaz. Sınıfını hiç tanımaz — hangi tabloya ait olduğunu, hangi alanın anahtar olduğunu bilmez. Çalışma anında gelen sütun adına bakıp aynı isimli property'yi doldurur; kalıp tutmazsa sessizce boş bırakır.

---

## 2. Kurulum ve Bağlantı Yönetimi

> **Benzetme —** Prize takılan çoklu priz gibi. Duvardaki tesisata dokunmaz, prizi değiştirmez; sadece var olanın üstüne takılıp onu kullanışlı hâle getirir. Dapper'ın `IDbConnection`'a yaptığı budur.

**Basitçe:** Tek bir NuGet paketi kurarsın, sonra elindeki her `SqlConnection` nesnesinde yeni metotlar belirir. Başka kurulum yok.

**Teknik olarak:** `dotnet add package Dapper` yeter. `using Dapper;` yazdığın anda `IDbConnection` uygulayan her nesnede Dapper metotları görünür — `SqlConnection`, `NpgsqlConnection`, `SqliteConnection` fark etmez; Dapper somut tipe değil arayüze bağlıdır.

```csharp
using System.Data;
using Microsoft.Data.SqlClient;
using Dapper;

using IDbConnection conn = new SqlConnection(cs);
var musteriler = conn.Query<Musteri>("SELECT Id, Ad, Email, Sehir FROM Musteriler");
```

Bağlantıyı açman gerekmez: Dapper kapalı bir bağlantı verirsen kendisi açar ve işi bitince kapatır. Açık verirsen dokunmaz, açık bırakır. Kural şu — bağlantıyı sen açtıysan kapatmak da senin işin.

> `using` yazmayı unutma. `Dispose` edilmeyen bağlantı havuza (connection pool) dönmez; yüksek trafikte havuz tükenir ve "Timeout expired... prior to obtaining a connection from the pool" hatası alırsın. Bu hatanın kaynağı neredeyse her zaman kapatılmamış bir bağlantıdır.

### DI ile kayıt

ASP.NET Core'da `IDbConnection` doğrudan kaydedilebilir — ama **scoped veya transient** olarak, asla singleton değil:

```csharp
builder.Services.AddScoped<IDbConnection>(_ =>
    new SqlConnection(builder.Configuration.GetConnectionString("Varsayilan")!));
```

Daha temizi, bağlantı yerine bir fabrika enjekte etmektir; her metot kendi bağlantısını açıp kapatır:

```csharp
public interface IBaglantiFabrikasi { IDbConnection Olustur(); }

public sealed class SqlBaglantiFabrikasi(IConfiguration cfg) : IBaglantiFabrikasi
{
    private readonly string _cs = cfg.GetConnectionString("Varsayilan")!;
    public IDbConnection Olustur() => new SqlConnection(_cs);
}

builder.Services.AddSingleton<IBaglantiFabrikasi, SqlBaglantiFabrikasi>();
```

Fabrika durumsuzdur, singleton olması sorun değil; ürettiği bağlantı kullanıldığı yerde `using` ile kapatılır.

**Bu benzetme şurada bozulur:** Çoklu priz elektriği çoğaltır, Dapper bağlantıyı çoğaltmaz. Bir `SqlConnection` aynı anda tek aktif komut taşır. Aynı bağlantıda iki sorguyu paralel `await` etmeye kalkarsan "A second operation was started on this connection" hatası alırsın. Paralellik istiyorsan ayrı bağlantı açacaksın.

---

## 3. Temel API

> **Benzetme —** Nöbetçi eczanedeki reçete penceresi gibi. Reçeteyi uzatırsın; kimi zaman bir kutu ilaç alırsın, kimi zaman poşet dolusu, kimi zaman "bu ilaç yok" cevabı, kimi zaman sadece "tamam, işlendi". Dapper'ın metotları da bu dört beklentiye karşılık gelir.

**Basitçe:** Öğreneceğin metotlar "kaç satır bekliyorum" ve "sonuç lazım mı" sorularına göre ayrışır.

**Teknik olarak:**

| Metot | Ne döner | Ne zaman |
|---|---|---|
| `Query<T>` | `IEnumerable<T>` | Çok satır beklenir |
| `QueryAsync<T>` | `Task<IEnumerable<T>>` | Aynısının asenkron hâli |
| `QueryFirstOrDefault<T>` | `T?` | Sıfır ya da bir satır; yoksa `null` |
| `QueryFirst<T>` | `T` | En az bir satır olmalı; yoksa istisna |
| `QuerySingleOrDefault<T>` | `T?` | Ya sıfır ya tam bir satır; iki satırda istisna |
| `QuerySingle<T>` | `T` | Tam bir satır olmalı |
| `Execute` | `int` (etkilenen satır) | INSERT / UPDATE / DELETE |
| `ExecuteScalar<T>` | `T?` | Tek hücre: COUNT, SUM, SCOPE_IDENTITY |
| `QueryMultiple` | `GridReader` | Tek gidişte birden çok result set |

```csharp
var urunler = await conn.QueryAsync<Urun>(
    "SELECT Id, Ad, Fiyat FROM Urunler WHERE KategoriId = @Kid", new { Kid = 3 });

var musteri = await conn.QueryFirstOrDefaultAsync<Musteri>(
    "SELECT Id, Ad, Email FROM Musteriler WHERE Email = @Email", new { Email = "umut@ornek.com" });

int aktif = await conn.ExecuteScalarAsync<int>(
    "SELECT COUNT(*) FROM Siparisler WHERE Durum = @Durum", new { Durum = "Hazirlaniyor" });
```

`Execute` etkilenen satır sayısını döner; bu sayı işlemin gerçekten yapılıp yapılmadığını anlamanın en basit yoludur. Yeni kaydın kimliğini almak için `ExecuteScalar<int>` ile `SCOPE_IDENTITY()` kullanılır — `@@IDENTITY` kullanma, o tetikleyicilerin ürettiği kimliği de görür.

```csharp
int etkilenen = await conn.ExecuteAsync(
    "UPDATE Urunler SET Fiyat = @Fiyat WHERE Id = @Id", new { Fiyat = 249.90m, Id = 17 });

int yeniId = await conn.ExecuteScalarAsync<int>(
    """
    INSERT INTO Musteriler (Ad, Email, Sehir, KayitTarihi)
    VALUES (@Ad, @Email, @Sehir, SYSUTCDATETIME());
    SELECT CAST(SCOPE_IDENTITY() AS int);
    """,
    new { Ad = "Ayşe Yılmaz", Email = "ayse@ornek.com", Sehir = "Kayseri" });
```

### QueryMultiple

Sipariş detay sayfasında siparişin kendisi, kalemleri ve müşterisi lazımdır. Üç ayrı `Query` üç gidiş-dönüş demektir; `QueryMultiple` üçünü tek komutta gönderir.

```csharp
const string sql = """
    SELECT * FROM Siparisler       WHERE Id = @Id;
    SELECT * FROM SiparisDetaylari WHERE SiparisId = @Id;
    SELECT m.* FROM Musteriler m JOIN Siparisler s ON s.MusteriId = m.Id WHERE s.Id = @Id;
    """;

using var grid = await conn.QueryMultipleAsync(sql, new { Id = 100 });
var siparis  = await grid.ReadFirstOrDefaultAsync<Siparis>();
var detaylar = (await grid.ReadAsync<SiparisDetay>()).ToList();
var musteri2 = await grid.ReadFirstOrDefaultAsync<Musteri>();
```

> `GridReader`'dan okuma **sırası** SQL'deki sorgu sırasıyla aynı olmak zorundadır. Sırayı karıştırırsan eşleme sessizce yanlış çalışır.

**Bu benzetme şurada bozulur:** Eczanede yanlış kutu verseler fark edersin. Dapper'da `QueryFirstOrDefault` yerine yanlışlıkla `QuerySingleOrDefault` seçmek — ya da tersi — hiçbir uyarı üretmez. `First` birden çok satır gelse de ilkini alıp devam eder; `Single` ikinci satırı görünce istisna fırlatır. "Bu sorgu en fazla bir satır dönmeli" diyorsan `Single` kullan, sessiz veri hatalarını yakalar.

---

## 4. Parametreler ve SQL Injection

> **Benzetme —** Bankada gişe camındaki para çekmecesi gibi. Müşteri elini içeri sokmaz; evrakı çekmeceye koyar, çekmece döner, içerideki memur onu **evrak olarak** alır. Parametre de budur: veri, komutun içine karışmadan ayrı bir kanaldan gider.

**Basitçe:** Sorgunun içine `+` ile değer yapıştırırsan kullanıcı sana komut yazabilir. Parametre verirsen yazamaz, çünkü verdiği şey hiçbir zaman komut olarak okunmaz.

**Teknik olarak:** En yaygın yol anonim nesnedir; property adları SQL'deki `@Ad` yer tutucularıyla eşleşir. Dapper bunları `SqlParameter` koleksiyonuna çevirir.

```csharp
var sonuc = conn.Query<Musteri>(
    "SELECT Id, Ad, Sehir FROM Musteriler WHERE Sehir = @Sehir AND KayitTarihi >= @Bas",
    new { Sehir = "Kayseri", Bas = new DateTime(2026, 1, 1) });
```

### Neden güvenli

```csharp
// TEHLİKELİ — string birleştirme
string sehir = Request.Query["sehir"];
var liste = conn.Query<Musteri>("SELECT * FROM Musteriler WHERE Sehir = '" + sehir + "'");
```

Kullanıcı `sehir` olarak `Kayseri'; DROP TABLE Siparisler; --` gönderirse sunucuya giden metin şu olur:

```sql
SELECT * FROM Musteriler WHERE Sehir = 'Kayseri'; DROP TABLE Siparisler; --'
```

Veritabanı bunu iki ayrı komut olarak görür ve ikisini de çalıştırır. Aynı girdi parametreyle gönderildiğinde ise giden komut şudur:

```sql
exec sp_executesql
  N'SELECT * FROM Musteriler WHERE Sehir = @Sehir',
  N'@Sehir nvarchar(100)',
  @Sehir = N'Kayseri''; DROP TABLE Siparisler; --'
```

Komut metni sabittir; kullanıcının yazdığı her şey `@Sehir` adlı bir **değerdir**. İçindeki tırnak, noktalı virgül, `DROP` kelimesi — hepsi düz metin. Güvenliğin kaynağı "tehlikeli karakterleri temizlemek" değil, **veriyi komuttan ayrı bir kanalda taşımaktır**.

> Ek kazanç: parametreli sorgu tek bir yürütme planı üretir ve önbellekte kalır. String birleştirmede her farklı değer yeni bir plan demektir. Güvenlik ve performans burada aynı yöne bakar.

### DynamicParameters

Parametreler koşula göre değişiyorsa ya da `OUTPUT` parametresi lazımsa kullanılır.

```csharp
var p = new DynamicParameters();
p.Add("@Sehir", sehir);
var sql = "SELECT Id, Ad FROM Musteriler WHERE Sehir = @Sehir";

if (minTutar is not null)
{
    p.Add("@MinTutar", minTutar, DbType.Decimal);
    sql += " AND Id IN (SELECT MusteriId FROM Siparisler GROUP BY MusteriId"
         + " HAVING SUM(ToplamTutar) >= @MinTutar)";
}

var liste = await conn.QueryAsync<Musteri>(sql, p);
```

### IN listesi

Listeyi doğrudan parametre olarak verirsin; Dapper `IN @Idler` ifadesini `IN (@Idler1, @Idler2, ...)` hâline kendisi açar. SQL'de **parantez yazılmaz**.

```csharp
var idler = new[] { 3, 7, 12, 40 };
var urunler = await conn.QueryAsync<Urun>(
    "SELECT Id, Ad, Fiyat FROM Urunler WHERE Id IN @Idler", new { Idler = idler });
```

Boş liste verirsen Dapper `IN (SELECT 1 WHERE 1=0)` üretir; hata yok, sıfır satır döner. SQL Server'ın parametre sınırı 2100'dür; listen bunu aşacaksa Table-Valued Parameter kullan.

### Stored procedure

Hafta 3'ün üçüncü notunda yazdığın prosedürleri çağırmak tek parametrelik farktır:

```csharp
var rapor = await conn.QueryAsync<AylikCiro>(
    "sp_AylikCiroRaporu", new { Yil = 2026, Ay = 9 },
    commandType: CommandType.StoredProcedure);

var p2 = new DynamicParameters();
p2.Add("@MusteriId", 42);
p2.Add("@ToplamTutar", dbType: DbType.Decimal, direction: ParameterDirection.Output);
await conn.ExecuteAsync("sp_MusteriCiroHesapla", p2, commandType: CommandType.StoredProcedure);
decimal toplam = p2.Get<decimal>("@ToplamTutar");
```

**Bu benzetme şurada bozulur:** Gişe çekmecesi her şeyi taşır; parametre taşımaz. **Tablo adı, sütun adı ve sıralama yönü parametre olamaz** — bunlar komutun yapısıdır, verisi değil. `ORDER BY @Sutun` yazarsan hata almazsın ama sıralama çalışmaz. Dinamik sıralamada değeri beyaz listeden geçir:

```csharp
string[] izinli = { "Ad", "Fiyat", "StokAdedi" };
string sutun = izinli.Contains(istenenSutun) ? istenenSutun : "Ad";
string yon   = istenenYon == "desc" ? "DESC" : "ASC";
var sql2 = $"SELECT Id, Ad FROM Urunler ORDER BY {sutun} {yon}";
```

---

## 5. Eşleme ve Çoklu Eşleme

> **Benzetme —** Postanedeki posta kutuları gibi. Kutuların üstünde isim yazar, mektupların üstünde de isim yazar. Görevli mektubu okumaz, sadece isimleri karşılaştırıp doğru kutuya atar. İsim tutmuyorsa mektup ortada kalır.

**Basitçe:** Dapper dönen sütun adına bakar ve aynı adlı property'yi doldurur. Ad tutmuyorsa o property varsayılan değerinde kalır — ve hata alınmaz.

**Teknik olarak:** Eşleme büyük-küçük harf duyarsızdır: `ad`, `Ad`, `AD` aynı property'yi bulur. Eşleşmeyen sütun sessizce yok sayılır. Şema adı ile C# adı farklıysa en basit çözüm `AS` ile takma ad vermektir:

```sql
SELECT m.MusteriID AS Id, m.MusteriAdi AS Ad, m.EPosta AS Email
FROM Musteriler m
```

Veritabanı `musteri_adi` gibi snake_case kullanıyorsa her sorguya `AS` yazmak yerine tek bir global ayar açarsın:

```csharp
// Program.cs — uygulama başlarken bir kez
Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true;
```

Bundan sonra `kayit_tarihi` sütunu `KayitTarihi` property'sine kendiliğinden eşlenir. Ayar globaldir; aynı uygulamada iki farklı isimlendirme şeması varsa ikisini birden memnun edemezsin.

### splitOn ile çoklu eşleme

Bir `JOIN` tek düz satır döndürür, ama sen iki ayrı nesne istersin. `splitOn`, Dapper'a "satırı buradan itibaren ikinci nesne say" der.

```csharp
const string sql = """
    SELECT s.Id, s.Tarih, s.ToplamTutar,
           m.Id, m.Ad, m.Email
    FROM Siparisler s
    JOIN Musteriler m ON m.Id = s.MusteriId
    WHERE s.Tarih >= @Bas
    """;

var siparisler = await conn.QueryAsync<Siparis, Musteri, Siparis>(
    sql,
    (siparis, musteri) => { siparis.Musteri = musteri; return siparis; },
    new { Bas = new DateTime(2026, 9, 1) },
    splitOn: "Id");
```

Üç kural: tip parametreleri sırayla yazılır, en sonda dönüş tipi durur; `splitOn` ikinci nesnenin **ilk** sütununun adıdır (varsayılanı `"Id"`); bölünme noktasını `SELECT`'teki sütun sırası belirler.

### 1-N ilişki

JOIN, bir siparişin üç kalemi varsa üç satır döndürür; sipariş üç kez tekrar eder. Bunları tek nesnede toplamak sana düşer:

```csharp
const string sql2 = """
    SELECT s.Id, s.Tarih, s.ToplamTutar,
           d.Id, d.SiparisId, d.UrunId, d.Adet, d.BirimFiyat
    FROM Siparisler s
    JOIN SiparisDetaylari d ON d.SiparisId = s.Id
    WHERE s.MusteriId = @MusteriId
    """;

var sozluk = new Dictionary<int, Siparis>();

await conn.QueryAsync<Siparis, SiparisDetay, Siparis>(
    sql2,
    (siparis, detay) =>
    {
        if (!sozluk.TryGetValue(siparis.Id, out var mevcut))
        {
            mevcut = siparis;
            mevcut.Detaylar = new List<SiparisDetay>();
            sozluk.Add(mevcut.Id, mevcut);
        }
        mevcut.Detaylar.Add(detay);
        return mevcut;
    },
    new { MusteriId = 42 },
    splitOn: "Id");

var sonuc2 = sozluk.Values.ToList();
```

Üç seviye de mümkündür (`splitOn: "Id,Id"`), ama kod hızla çirkinleşir. Daha derin ilişkilerde ya `QueryMultiple` ile parça parça çekersin ya da o sorguyu EF Core'a bırakırsın.

**Bu benzetme şurada bozulur:** Postacı yanlış kutuya atarsa mektup sahibi er geç anlar. Dapper'da eşleşmeyen sütun **hiçbir iz bırakmaz**: `Fiyat` sütununu `Tutar` property'sine eşlemeyi unutursan uygulama çalışır, hata vermez, sadece her yerde sıfır fiyat görürsün. `SELECT *` yerine sütunları açıkça yazmak bu yüzden alışkanlık değil, savunma tedbiridir.

---

## 6. EF Core ile Karşılaştırma

> **Benzetme —** Otomatik vitesli araba ile manuel vites gibi. Otomatikte trafikte yorulmazsın ama rampada hangi viteste olduğuna araba karar verir. Manuelde her şeyi sen yaparsın; şehir içinde yorucu, ama zor bir yokuşta doğru vitesi sen bilirsin.

**Basitçe:** EF Core işini kolaylaştırır, Dapper kontrolü sana verir. Kolaylık CRUD'da kazandırır, kontrol ağır sorgularda.

**Teknik olarak:**

| Ölçüt | EF Core | Dapper |
|---|---|---|
| Change tracking | Var — nesneyi değiştir, `SaveChanges` yeter | Yok — `UPDATE` cümlesini sen yazarsın |
| Migration | Var — `Add-Migration` / `Update-Database` | Yok — DDL senin ya da DbUp / FluentMigrator gibi bir aracın işi |
| Sorgu dili | LINQ, derleme anında tip güvenli | Düz SQL metni, hatası çalışma anında çıkar |
| Öğrenme eğrisi | Dik ve uzun (DbContext, Fluent API, tracking, yaşam süresi) | Çok kısa — bir öğleden sonra |
| Performans | İyi; tracking ve materyalizasyon maliyeti var | Raw ADO.NET'e çok yakın |
| SQL üzerinde kontrol | Dolaylı — üretilen SQL'i loglayıp incelersin | Tam — ne yazarsan o gider |
| Bakım maliyeti (CRUD) | Düşük — model değişir, sorgular çoğunlukla dayanır | Yüksek — sütun eklenince ilgili tüm SQL elden geçer |
| Bakım maliyeti (raporlama) | Yüksek — karmaşık LINQ okunmaz hâle gelir | Düşük — SQL'i DBA da okur, profiler'da aynen görürsün |
| Veritabanı bağımsızlığı | Büyük ölçüde var (sağlayıcıyı değiştirirsin) | Yok — SQL lehçesine bağlısın |
| İlişki yönetimi | Navigation property, `Include`, lazy loading | Elle `JOIN` ve `splitOn` |

**EF Core'un kazandığı yerler:** kayıt ekleme, düzenleme, silme ekranları; domain modelinin merkezde olduğu iş mantığı; şemanın kod tarafından yönetildiği projeler; birden çok varlığın tek `SaveChanges` ile tutarlı kaydedilmesi.

**Dapper'ın kazandığı yerler:** çok tablolu raporlama ve dashboard sorguları; pencere fonksiyonu, CTE, `PIVOT` gibi LINQ'e zor çevrilen T-SQL özellikleri; stored procedure ağırlıklı eski veritabanları; milisaniyenin konuştuğu okuma uç noktaları.

> "Hangisi daha hızlı" yanıltıcı bir sorudur. Tek satır okumada fark mikrosaniyelerdedir ve ağ gecikmesinin yanında kaybolur. Fark, yüzlerce satır materyalize ederken ve change tracking açıkken belirginleşir. EF Core tarafında `AsNoTracking()` tek başına farkın büyük kısmını kapatır.

**Bu benzetme şurada bozulur:** Arabada tek vites kutusu vardır; projende iki tane olabilir. Yukarıdaki tablo bir seçim listesi değil, bir iş bölümü tablosudur — sonraki başlık tam olarak bunu anlatıyor.

---

## 7. İkisini Birlikte Kullanmak

> **Benzetme —** Bir lokantada mutfak ile kasa gibi. Mutfak yemeği hazırlar, kasa hesabı keser; ikisi ayrı iş yapar ama aynı gün sonu kasasını paylaşır. Hesap tutmuyorsa sorun, birinin diğerinden habersiz yazmasındandır.

**Basitçe:** Aynı projede EF Core ile yazarsın, Dapper ile ağır raporları okursun. Yeter ki ikisi aynı bağlantıyı ve aynı transaction'ı paylaşsın.

**Teknik olarak:** Gerçek dünyada en yaygın kullanım budur; bootcamp projelerinde de büyük ihtimalle böyle yapacaksın. En basit hâli `DbContext`'in bağlantısını ödünç almaktır — `DbContext` zaten bir `DbConnection` tutar.

```csharp
public sealed class RaporServisi(MagazaDbContext context)
{
    public async Task<IReadOnlyList<UrunCiro>> EnCokSatanlar(int yil, int adet)
    {
        var conn = context.Database.GetDbConnection();   // aynı bağlantı

        const string sql = """
            SELECT TOP (@Adet)
                   u.Ad                       AS UrunAdi,
                   k.Ad                       AS KategoriAdi,
                   SUM(d.Adet * d.BirimFiyat) AS ToplamCiro
            FROM SiparisDetaylari d
            JOIN Urunler     u ON u.Id = d.UrunId
            JOIN Kategoriler k ON k.Id = u.KategoriId
            JOIN Siparisler  s ON s.Id = d.SiparisId
            WHERE YEAR(s.Tarih) = @Yil
            GROUP BY u.Ad, k.Ad
            ORDER BY ToplamCiro DESC
            """;

        return (await conn.QueryAsync<UrunCiro>(sql, new { Yil = yil, Adet = adet })).ToList();
    }
}
```

Bu bağlantıyı `using` ile sarma — sahibi `DbContext`'tir, kapatmak senin işin değil.

### Aynı transaction'ı paylaşmak

Asıl kritik nokta bu. EF Core ile bir sipariş yazıp aynı işlem içinde Dapper ile stok düşeceksen ikisi **aynı transaction** üzerinde olmalı. Aksi hâlde biri geri alınır, diğerinin yazdığı kalır.

```csharp
public async Task SiparisOlustur(Siparis siparis, CancellationToken ct)
{
    await using var tx = await context.Database.BeginTransactionAsync(ct);

    // 1) Yazma tarafı: EF Core
    context.Siparisler.Add(siparis);
    await context.SaveChangesAsync(ct);

    // 2) Toplu stok düşümü: Dapper — AYNI bağlantı, AYNI transaction
    var conn = context.Database.GetDbConnection();
    var dbTx = tx.GetDbTransaction();          // Microsoft.EntityFrameworkCore.Storage

    await conn.ExecuteAsync(
        """
        UPDATE u SET u.StokAdedi = u.StokAdedi - d.Adet
        FROM Urunler u
        JOIN SiparisDetaylari d ON d.UrunId = u.Id
        WHERE d.SiparisId = @SiparisId
        """,
        new { SiparisId = siparis.Id },
        transaction: dbTx);                    // bunu unutma

    await tx.CommitAsync(ct);
}
```

`transaction: dbTx` geçmezsen komut transaction dışında çalışır ve büyük ihtimalle transaction'ın tuttuğu kilitlerde bekleyip zaman aşımına düşer — sessiz bir tutarsızlık değil ama teşhisi zor bir kilitlenme.

Ters yön de mümkün: transaction'ı sen `DbConnection` üzerinde başlattıysan `DbContext`'e onu kullanmasını söylersin.

```csharp
await using var conn2 = new SqlConnection(cs);
await conn2.OpenAsync(ct);
await using var tx2 = (SqlTransaction)await conn2.BeginTransactionAsync(ct);

await conn2.ExecuteAsync("UPDATE Urunler SET Fiyat = Fiyat * 1.10", transaction: tx2);

context.Database.SetDbConnection(conn2);
await context.Database.UseTransactionAsync(tx2, ct);   // EF Core artık bu tx'i kullanır

context.Kategoriler.Add(new Kategori { Ad = "Kampanya" });
await context.SaveChangesAsync(ct);
await tx2.CommitAsync(ct);
```

> Pratik kural: **tek yazma yolu olsun.** Aynı tabloya hem EF Core hem Dapper yazarsa, EF Core'un bellekte takip ettiği nesneler gerçeği yansıtmaz hâle gelir. Dapper'ı öncelikle okuma için kullan; yazman gerekiyorsa `context.ChangeTracker.Clear()` ya da `AsNoTracking()` ile beklentini netleştir.

**Bu benzetme şurada bozulur:** Lokantada mutfak ile kasa aynı anda çalışır; burada çalışamazlar. Tek `DbConnection` üzerinde aynı anda tek aktif komut olabilir — EF Core sorgusunu `await` etmeden Dapper sorgusunu başlatırsan hata alırsın. Paylaşmak sırayla kullanmaktır, eşzamanlı kullanmak değil.

---

## 8. Dapper'ın Yapmadıkları

> **Benzetme —** Oto tamircisinden ödünç alınan anahtar takımı gibi. Cıvatayı söker, çok da iyi söker; ama arabanın bakım geçmişini tutmaz, "şu parça yıprandı" demez, bir sonraki servis tarihini hatırlatmaz.

**Basitçe:** Dapper eksik değil, dar kapsamlı. Yapmadıkları listesi onun tasarım kararıdır, kusuru değil.

**Teknik olarak:**

| Yok olan | Sonucu |
|---|---|
| Migration | Şema değişikliklerini elle ya da DbUp / FluentMigrator gibi bir araçla yönetirsin |
| Change tracking | Nesneyi değiştirmek hiçbir şey yapmaz; `UPDATE` cümlesini sen yazarsın |
| Identity map | Aynı satırı iki kez çekersen iki ayrı nesne alırsın; `==` ile eşit değillerdir |
| Navigation property doldurma | İlişkileri `JOIN` ve `splitOn` ile elle kurarsın |
| Lazy loading | İlişkili veri kendiliğinden gelmez; sorgusunu sen yazarsın |
| Unit of Work | Transaction'ı açmak, commit ve rollback etmek senin sorumluluğun |
| Veritabanı bağımsızlığı | `TOP`, `OFFSET/FETCH`, `SYSUTCDATETIME()` gibi ifadeler SQL Server'a özeldir |
| Derleme anında sorgu kontrolü | SQL bir string'dir; yazım hatası ancak çalışma anında ortaya çıkar |

Son maddenin pratik karşılığı önemli: LINQ'te sütun adını yanlış yazarsan proje derlenmez, Dapper'da o satır çalışana kadar bilmezsin. Bu yüzden Dapper kullanan projelerde veritabanına gerçekten giden entegrasyon testleri daha çok işe yarar.

---

## 9. Repository Pattern ile Dapper

> **Benzetme —** Muhtarlıktaki tek pencere gibi. İçeride hangi defter, hangi dolap — vatandaşı ilgilendirmez. Pencereye "ikametgâh lazım" der, belgeyi alır. Repository de üst katmana sadece niyeti sorar, SQL'i içeride tutar.

**Basitçe:** SQL cümlelerini controller'lara dağıtmazsın, tek bir sınıfta toplarsın. MvcCv'deki `GenericRepository` ile aynı fikir; sadece altında EF Core yerine Dapper var.

**Teknik olarak:** Küçük ama tam bir örnek:

```csharp
public interface IMusteriRepository
{
    Task<Musteri?> GetirAsync(int id, CancellationToken ct = default);
    Task<int> EkleAsync(Musteri m, CancellationToken ct = default);
    Task<bool> GuncelleAsync(Musteri m, CancellationToken ct = default);
}

public sealed class MusteriRepository(IBaglantiFabrikasi fabrika) : IMusteriRepository
{
    private const string Sutunlar = "Id, Ad, Email, Sehir, KayitTarihi";

    public async Task<Musteri?> GetirAsync(int id, CancellationToken ct = default)
    {
        using var conn = fabrika.Olustur();
        return await conn.QuerySingleOrDefaultAsync<Musteri>(new CommandDefinition(
            $"SELECT {Sutunlar} FROM Musteriler WHERE Id = @Id",
            new { Id = id }, cancellationToken: ct));
    }

    public async Task<int> EkleAsync(Musteri m, CancellationToken ct = default)
    {
        using var conn = fabrika.Olustur();
        return await conn.ExecuteScalarAsync<int>(new CommandDefinition(
            """
            INSERT INTO Musteriler (Ad, Email, Sehir, KayitTarihi)
            VALUES (@Ad, @Email, @Sehir, SYSUTCDATETIME());
            SELECT CAST(SCOPE_IDENTITY() AS int);
            """, m, cancellationToken: ct));
    }

    public async Task<bool> GuncelleAsync(Musteri m, CancellationToken ct = default)
    {
        using var conn = fabrika.Olustur();
        var n = await conn.ExecuteAsync(new CommandDefinition(
            "UPDATE Musteriler SET Ad = @Ad, Email = @Email, Sehir = @Sehir WHERE Id = @Id",
            m, cancellationToken: ct));
        return n > 0;
    }
}
```

Üç ayrıntıya dikkat et. `CommandDefinition` kullanmanın sebebi `CancellationToken` geçebilmek — Dapper'ın kısa imzalarında iptal jetonu yoktur. `Ekle` ve `Guncelle` doğrudan `Musteri` nesnesini alır; Dapper property adlarını `@Ad`, `@Email` yer tutucularıyla eşler, anonim nesne kurmaya gerek kalmaz. Sütun listesi tek sabittedir; şemaya sütun eklendiğinde tek yer değişir.

> Dapper'da **generic repository** genellikle iyi fikir değildir. EF Core'da `DbSet<T>` sayesinde generic CRUD doğal gelir; Dapper'da her tipin SQL'i farklıdır ve generic yapmaya çalışmak seni tablo adını string olarak üretmeye iter — hem kırılgan hem güvensiz. Her varlık için ayrı, açık repository yaz.

---

## 10. Ne Zaman Dapper Kullanma

> **Benzetme —** Şehir içinde kamyonetle dolaşmak gibi. Kamyonet kötü araç değil, yük taşırken üstüne yok. Ama pazara ekmek almaya kamyonetle gidilmez.

**Basitçe:** Dapper'ın kazandığı yer bellidir. O yerin dışında kullanmak, kazandırdığından fazlasını kaybettirir.

**Teknik olarak:**

- **Ağırlıklı olarak CRUD yazıyorsan.** `SaveChanges` ile hallolan işi elle `UPDATE` yazarak yapmak boş emektir; change tracking'in çözdüğü "hangi alan değişti" sorusunu sen çözersin.
- **Şema sık değişiyorsa.** Migration olmadığı için her sütun değişikliği hem DDL hem de dağınık SQL cümleleri demektir.
- **Karmaşık domain modelin varsa.** Derin nesne grafikleri ve çift yönlü ilişkiler elle eşlenince `splitOn` cehennemine dönüşür.
- **Takım SQL'e hâkim değilse.** Elle yazılmış kötü SQL, EF Core'un ürettiği ortalama SQL'den yavaş olur.
- **Veritabanı bağımsızlığı bir gereksinimse.** Müşteriye göre SQL Server ya da PostgreSQL kurulacaksa her sorguyu iki lehçede gözden geçirirsin.
- **"Daha hızlı olsun" diye ölçmeden geçiyorsan.** Yavaşlığın kaynağı çoğu zaman ORM değil, eksik index veya N+1 sorgudur. Önce ölç: `AsNoTracking()`, projeksiyon ve doğru index farkı çoğu zaman kapatır.

> Karar cümlesi: Dapper'ı varsayılan veri erişim katmanın yapma; **belirli sorgular için bilinçli olarak seç.** "Bu sorgunun SQL'ini elimle yazmak istiyorum" diyebiliyorsan Dapper doğru araçtır.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| ORM | Nesne-ilişkisel eşleyici: nesne dünyası ile tablo dünyasını bağlayan katman |
| Micro-ORM | ORM'in sadece satır-nesne eşleme kısmını yapan ince kütüphane |
| `IDbConnection` | ADO.NET'in bağlantı arayüzü; Dapper tüm metotlarını buna ekler |
| SQL injection | Kullanıcı girdisinin komut olarak yorumlanmasıyla oluşan güvenlik açığı |
| `DynamicParameters` | Koşullu parametre ekleme ve OUTPUT / ReturnValue alma için Dapper sınıfı |
| `splitOn` | Satırın nereden itibaren ikinci nesne sayılacağını söyleyen Dapper parametresi |
| Change tracking | EF Core'un nesnelerdeki değişikliği izleyip UPDATE üretmesi |
| `GetDbConnection()` | `DbContext`'in kullandığı bağlantıyı döndüren EF Core metodu |
| `GetDbTransaction()` | EF Core transaction'ının altındaki `DbTransaction`'ı döndüren metot |
| `UseTransaction` | Dışarıda başlatılmış bir transaction'ı EF Core'a kullandırma metodu |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `../Hafta-04-AspNetCore-MVC/01-MVC-ve-Razor.md` (Hafta 4 · Pazartesi)

Hafta 3 burada kapanıyor. Hafta 4'te veriyi ekrana taşıyorsun: MVC'nin istek hattı, Razor sözdizimi ve view'lara model geçirme.
