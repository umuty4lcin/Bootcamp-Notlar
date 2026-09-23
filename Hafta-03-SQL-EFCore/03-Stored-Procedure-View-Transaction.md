# Hafta 3 · Çarşamba — Stored Procedure, View, Transaction

**Okuma süresi:** ~55 dk
**Neden bu konu:** Buraya kadar veritabanına hep "dışarıdan" sorgu gönderdin. Bu notta veritabanının kendi içinde yaşayan kodu — view, stored procedure, fonksiyon, trigger — ve birden çok işlemi tek bir bütün hâline getiren transaction'ı göreceksin. Para transferi, stok düşme, sipariş oluşturma gibi işlerin doğru çalışması buradaki kavramlara bağlıdır. Bootcamp'te "bu kayıt neden yarım kaldı" ya da "uygulama neden kilitlendi" sorusunun cevabı hep bu nottaki konulardan çıkar.

---

## Önce Basitçe

Şimdiye kadar veritabanını bir depo gibi kullandın: bir şey sordun, cevabını aldın. Ama depoların da kendi düzeni, kendi memurları, kendi kuralları olur. Bu notta o düzeni tanıyacaksın.

İlk kavram **view**. Düşün ki her sabah aynı uzun sorguyu yazıyorsun: şu üç tabloyu birleştir, şu filtreyi uygula, şu kolonları getir. Bunu bir kere yazıp ona bir isim verebilsen ve bir daha o ismi kullanabilsen? İşte view bu. Kaydedilmiş bir sorgudur. Veri tutmaz, sadece "bakış açısı" tutar. Ona sorduğunda arkadaki asıl tablolara gider, taze veriyi getirir.

İkinci kavram **stored procedure**. View kaydedilmiş bir sorguysa, stored procedure kaydedilmiş bir **iş**tir. İçinde birden fazla adım olabilir: önce kontrol et, sonra ekle, sonra güncelle, hata olursa geri al. Parametre alır, sonuç döndürür, veritabanının içinde yaşar. Uygulaman ona "şu işi yap" der, nasıl yaptığına karışmaz.

Üçüncü kavram **transaction**. Bir banka havalesi düşün: senin hesabından para çıkacak, karşı hesaba girecek. İki ayrı işlem ama tek bir olay. Birincisi olup ikincisi olmazsa para buharlaşmış olur. Transaction, "bu adımların hepsi olacak ya da hiçbiri olmayacak" demenin yoludur. Yarısı olmaz. Ya tamamı ya hiçbiri.

Bu notta önce veritabanı içinde yaşayan kod nesnelerini — view, procedure, fonksiyon, trigger — tek tek göreceksin. Sonra transaction'ın anlamını, izolasyon seviyelerini ve kilitleri. En sonda da bunların EF Core tarafında nasıl göründüğünü. Şimdi detaya iniyoruz.

> **Ana benzetme:** Veritabanı bir **tapu dairesi**dir. **View**, memurun masasındaki hazır form — "şu mahallenin boş parselleri" diye baktığında her seferinde arşivden taze bakılır, ama sorgunun kendisi kaydedilmiştir. **Stored procedure**, "devir işlemi" adıyla tanımlanmış resmi prosedürdür: adımları bellidir, evrak ister, sonuç verir. **Transaction**, devir işleminin bütünüdür — eski sahibin adı silinip yeni sahibin adı yazılana kadar işlem bitmemiştir, yarıda kalırsa tapu eski hâline döner. **Kilit** ise o parselin dosyasının bir memurun elindeyken başkasına verilmemesidir.

---

## Bu Notta Ne Var

1. View — kaydedilmiş sorgu
2. Stored procedure — kaydedilmiş iş
3. Hata yönetimi: `TRY...CATCH`, `THROW`, `RAISERROR`
4. Plan cache ve parameter sniffing
5. SQL injection ve parametreli sorgu
6. Fonksiyonlar: scalar ve table-valued
7. Trigger — kendiliğinden çalışan kod
8. Transaction ve ACID
9. Isolation level'lar ve okuma anomalileri
10. Kilitler ve deadlock
11. EF Core tarafında SP ve transaction

---

## 1. View — Kaydedilmiş Sorgu

> **Benzetme —** Nöbetçi eczane listesi düşün. Eczane listesi diye ayrı bir defter tutulmaz; ilçedeki bütün eczanelerin defteri vardır, bir de "bugün nöbetçi olanlar" diye bir bakış. O bakışa her sorduğunda taze cevap gelir, çünkü arkasında hep aynı asıl defter durur. Bakışın kendisi veri tutmaz, sadece "nereye nasıl bakılacağını" tutar.

**Basitçe:** View, bir sorguya isim vermektir. Uzun ve tekrar eden bir `SELECT` yazdıysan onu view olarak kaydedersin, sonra o view'e sanki bir tabloymuş gibi sorarsın. İçinde veri yoktur; sorduğun anda arkadaki tablolara gidilir.

**Teknik olarak:** **View (görünüm)** — Veritabanında isimle saklanan bir `SELECT` ifadesidir. Sorgulandığında tanımındaki sorgu çalışır, sonuç üretilir. Fiziksel olarak veri tutmaz (indexed view hariç, aşağıda).

```sql
CREATE VIEW vw_SiparisOzet
AS
SELECT
    s.SiparisId,
    s.SiparisTarihi,
    m.MusteriId,
    m.Ad + ' ' + m.Soyad AS MusteriAdi,
    SUM(sd.BirimFiyat * sd.Adet) AS ToplamTutar,
    COUNT(*)                     AS KalemSayisi
FROM Siparisler s
JOIN Musteriler m       ON m.MusteriId = s.MusteriId
JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
GROUP BY s.SiparisId, s.SiparisTarihi, m.MusteriId, m.Ad, m.Soyad;
```

Kullanımı tablo gibidir:

```sql
SELECT * FROM vw_SiparisOzet WHERE ToplamTutar > 5000 ORDER BY SiparisTarihi DESC;
```

### Güncellenebilir view sınırları

View'e `INSERT`/`UPDATE`/`DELETE` yapmak **bazen** mümkündür. SQL Server, view'i tek bir tabloya net biçimde geri izleyebiliyorsa yazma işlemini geçirir.

Yazma **yapılamaz** eğer view'de şunlardan biri varsa:

| Engel | Neden |
|---|---|
| `GROUP BY`, `HAVING`, `DISTINCT` | Satırlar birleşmiş; hangi asıl satır güncellenecek belli değil |
| Toplama fonksiyonu (`SUM`, `COUNT`) | Hesaplanmış değer geri yazılamaz |
| `UNION` / `UNION ALL` | Satır hangi tablodan geldi belirsiz |
| Birden çok tabloya yayılan güncelleme | Tek `UPDATE` iki tabloya yazamaz |
| `TOP` ile `ORDER BY` birlikte (bazı durumlar) | Belirsiz satır kümesi |

> **Uyarı:** `WITH CHECK OPTION` olmadan güncellenebilir bir view'e, o view'in filtresine uymayan satır yazabilirsin. `WHERE Aktif = 1` filtreli bir view üzerinden `Aktif = 0` yaparsan satır view'den kaybolur ama güncelleme başarılı sayılır. `WITH CHECK OPTION` eklersen bu işlem hata verir.

### Indexed view (materialized view)

Normal view veri tutmaz. **Indexed view**, üstüne `UNIQUE CLUSTERED INDEX` koyulmuş view'dir; sonucu fiziksel olarak diskte tutar ve asıl tablolar değiştikçe SQL Server onu otomatik günceller.

Kuralları katıdır: `WITH SCHEMABINDING` zorunludur (şema kilitlenir, altındaki kolonu değiştiremezsin), `COUNT(*)` yerine `COUNT_BIG(*)` gerekir, `OUTER JOIN` ve alt sorgu kullanılamaz.

### View üstüne view zinciri tuzağı

View'ler birbirinin üstüne yazılabilir. Ve tam da bu yüzden tehlikelidir.

```sql
CREATE VIEW vw_A AS SELECT * FROM Siparisler WHERE Iptal = 0;
CREATE VIEW vw_B AS SELECT * FROM vw_A WHERE SiparisTarihi >= '2026-01-01';
CREATE VIEW vw_C AS SELECT vw_B.*, m.Sehir FROM vw_B JOIN Musteriler m ON m.MusteriId = vw_B.MusteriId;
```

`vw_C`'ye baktığında ekranda basit bir sorgu görürsün. Gerçekte çalışan şey üç katmanın açılmış hâlidir. Sorun şurada:

Pratik kural: **view zinciri iki katmanı geçmesin**. Üçüncü katmana ihtiyaç duyuyorsan muhtemelen o iş bir stored procedure ya da düzgün bir rapor tablosu olmalı.

**Bu benzetme şurada bozulur:** Nöbetçi eczane listesi benzetmesi view'in "taze veri" tarafını iyi anlatır ama maliyeti gizler. Gerçek hayatta listeye bakmak bedavadır. Veritabanında ise view'e her bakış, arkadaki sorgunun **yeniden çalışması** demektir. Beş kişi aynı anda `vw_SiparisOzet`'e bakarsa o ağır `GROUP BY` beş kez çalışır. View, yazma kolaylığıdır; okuma hızı garantisi değildir.

---

## 2. Stored Procedure — Kaydedilmiş İş

> **Benzetme —** Oto tamircisine gidip "periyodik bakım" dersin. Ustaya yağı boşalt, filtreyi değiştir, buji kontrol et, fren balatasına bak diye tek tek anlatmazsın. "Periyodik bakım" adıyla tanımlanmış, sırası belli bir iş vardır. Sen sadece aracın plakasını ve kilometresini verirsin — parametre. Usta işi yapar, sonunda fatura verir — dönüş değeri.

**Basitçe:** Stored procedure, veritabanının içinde saklanan bir yordamdır. Parametre alır, içinde birden fazla SQL adımı çalıştırır, sonuç kümesi ya da değer döndürür. Uygulaman ona sadece adıyla ve parametreleriyle seslenir.

**Teknik olarak:** **Stored procedure (saklı yordam)** — Veritabanı sunucusunda derlenmiş ve isimle saklanan T-SQL kod bloğu. View'den farkı: içinde akış kontrolü (`IF`, `WHILE`), değişken, hata yönetimi ve veri değiştirme işlemleri olabilir.

```sql
CREATE PROCEDURE sp_MusteriSiparisleri
    @MusteriId   INT,
    @BaslangicTarihi DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        s.SiparisId,
        s.SiparisTarihi,
        SUM(sd.BirimFiyat * sd.Adet) AS Tutar
    FROM Siparisler s
    JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
    WHERE s.MusteriId = @MusteriId
      AND (@BaslangicTarihi IS NULL OR s.SiparisTarihi >= @BaslangicTarihi)
    GROUP BY s.SiparisId, s.SiparisTarihi
    ORDER BY s.SiparisTarihi DESC;
END
```

### Parametreler: giriş ve çıkış

T-SQL'de parametre varsayılan olarak **giriş** (input) parametresidir. Çıkış parametresi için `OUTPUT` yazılır.

```sql
CREATE PROCEDURE sp_SiparisEkle
    @MusteriId   INT,
    @UrunId      INT,
    @Adet        INT,
    @YeniSiparisId INT OUTPUT          -- çıkış parametresi
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Fiyat DECIMAL(18,2);
    SELECT @Fiyat = Fiyat FROM Urunler WHERE UrunId = @UrunId;

    INSERT INTO Siparisler (MusteriId, SiparisTarihi, Iptal)
    VALUES (@MusteriId, GETDATE(), 0);

    SET @YeniSiparisId = SCOPE_IDENTITY();   -- yeni üretilen kimlik

    INSERT INTO SiparisDetaylari (SiparisId, UrunId, Adet, BirimFiyat)
    VALUES (@YeniSiparisId, @UrunId, @Adet, @Fiyat);
END
```

Çağırırken `OUTPUT` kelimesi **iki yerde** de yazılmalıdır, yoksa değer geri gelmez:

```sql
DECLARE @Id INT;
EXEC sp_SiparisEkle @MusteriId = 17, @UrunId = 5, @Adet = 2, @YeniSiparisId = @Id OUTPUT;
SELECT @Id AS OlusanSiparisId;
```

> **Kimlik alma tuzağı:** `@@IDENTITY` yerine `SCOPE_IDENTITY()` kullan. `@@IDENTITY` oturumdaki **son** kimliği verir; tabloda bir trigger varsa ve o trigger başka bir tabloya `INSERT` yapıyorsa `@@IDENTITY` sana o trigger'ın ürettiği değeri döndürür. `SCOPE_IDENTITY()` sadece kendi kapsamına bakar. En güvenlisi ise `OUTPUT INSERTED.SiparisId` cümlesidir.

### Dönüş değeri (`RETURN`) ile sonuç kümesi farkı

Bir stored procedure üç ayrı kanaldan bilgi verebilir. Karıştırılır:

| Kanal | Ne taşır | Sınır |
|---|---|---|
| Sonuç kümesi (`SELECT`) | Satırlar, tablo şeklinde | Asıl veri kanalı |
| `RETURN` | Tek bir `INT` | Sadece durum kodu içindir |

> `RETURN` ile veri döndürmeye çalışma. Sadece `INT` alır ve anlamı "durum kodu"dur. Fiyat, adet, kimlik gibi değerler `OUTPUT` parametresiyle döner. Modern yaklaşımda `RETURN` kodu yerine hata fırlatmak (`THROW`) tercih edilir.

### SP'nin avantajları ve dezavantajları

| Avantaj | Dezavantaj |
|---|---|
| Ağ trafiği az — tek çağrı, çok adım | Kod iki yere bölünür; sürüm takibi zorlaşır |
| Yetkilendirme tablo yerine SP'ye verilebilir | Git'te izlemek ve code review yapmak zahmetli |
| Plan tekrar kullanılır | Veritabanına bağımlılık artar, taşıması zor |
| İş mantığı tek yerden değişir (derleme gerektirmez) | Test yazmak ve hata ayıklamak zor |
| Karmaşık toplu işlemde çok hızlı | İş mantığı katmanı ile SP arasında bilgi dağılır |

**Nerede kullanılır:** Toplu veri işleme, raporlama, çok adımlı ve veri yoğun işler, eski sistemlerle entegrasyon. **Nerede kullanılmaz:** Basit CRUD. `SELECT * FROM Urunler WHERE UrunId = @Id` için SP yazmak sadece bakım yükü üretir; EF Core zaten bunu parametreli ve güvenli şekilde yapar.

**Bu benzetme şurada bozulur:** Oto tamirci benzetmesinde usta her araca aynı bakımı uygular. Stored procedure ise **ilk gelen araca göre** bir plan yapar ve sonrakilere de onu uygular. Bir sonraki bölümün konusu tam olarak budur: usta, ilk gördüğü Şahin'e göre hazırladığı tezgâhı sıradaki TIR'a da uygulamaya kalkarsa ortalık karışır.

---

## 3. Hata Yönetimi: `TRY...CATCH`, `THROW`, `RAISERROR`

> **Benzetme —** Elektrik tesisatındaki sigorta kutusu. Bir yerde kısa devre olduğunda tüm ev kararmaz; ilgili sigorta atar, sen kutuya gider, hangisinin attığına bakar, sebebini anlar ve düzeltirsin. Sigorta iki iş yapar: hasarı sınırlar ve **nerede olduğunu söyler**. Hata yönetimi de öyledir — hem işlemi durdurur hem de neyin olduğunu kaydeder.

**Basitçe:** SQL Server'da bir hata olduğunda varsayılan davranış her zaman "her şeyi durdur" değildir. Bazı hatalar sadece o ifadeyi iptal eder, kodun kalanı çalışmaya devam eder. Bunu kontrol etmek için hatayı `TRY...CATCH` ile yakalar, kendi mesajını `THROW` ile fırlatırsın.

**Teknik olarak:**

```sql
CREATE PROCEDURE sp_SiparisOlustur
    @MusteriId INT,
    @UrunId    INT,
    @Adet      INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        IF NOT EXISTS (SELECT 1 FROM Musteriler WHERE MusteriId = @MusteriId)
            THROW 50001, 'Müşteri bulunamadı.', 1;

        DECLARE @Stok INT;
        SELECT @Stok = StokAdedi FROM Urunler WITH (UPDLOCK) WHERE UrunId = @UrunId;

        IF @Stok IS NULL
            THROW 50002, 'Ürün bulunamadı.', 1;

        IF @Stok < @Adet
            THROW 50003, 'Yetersiz stok.', 1;

        UPDATE Urunler SET StokAdedi = StokAdedi - @Adet WHERE UrunId = @UrunId;

        INSERT INTO Siparisler (MusteriId, SiparisTarihi, Iptal)
        VALUES (@MusteriId, GETDATE(), 0);

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        THROW;   -- hatayı olduğu gibi çağırana ilet
    END CATCH
END
```

### `THROW` ve `RAISERROR` farkı

`RAISERROR` eskidir, `THROW` (SQL Server 2012 ile geldi) yenisidir ve tercih edilendir.

| Özellik | `THROW` | `RAISERROR` |
|---|---|---|
| Parametreli mesaj (`%s`, `%d`) | Yok | Var |
| Hata numarası | 50000 veya üstü olmalı | Serbest (özel mesaj için 50000+) |
| Orijinal hatayı olduğu gibi iletme | `THROW;` ile evet | Hayır, mesajı yeniden kurar |
| Severity | Hep 16 | Ayarlanabilir |
| Toplu işi (batch) durdurur | Evet | Severity'ye bağlı |
| Önceki satırın `;` ile bitmesi | Zorunlu | Zorunlu değil |

> **Yutulmuş hata tuzağı:** `CATCH` bloğunda sadece loglayıp `THROW;` yazmazsan, çağıran taraf işlemin **başarılı olduğunu sanır**. Uygulama hata görmez, kullanıcıya "kaydedildi" der, kayıt yoktur. `CATCH` ya hatayı yeniden fırlatmalı ya da çağırana açık bir başarısızlık sinyali döndürmelidir.

**Bu benzetme şurada bozulur:** Sigorta benzetmesi hataların hepsinin aynı biçimde "attığını" ima eder. SQL Server'da öyle değildir. Derleme hataları (olmayan tablo adı gibi) `TRY...CATCH` tarafından **yakalanamaz** — hata, blok çalışmaya başlamadan önce ortaya çıkar. Aynı şekilde severity 20 ve üstü hatalar bağlantıyı düşürür, `CATCH` çalışmaz. `TRY...CATCH` çalışma zamanı hatalarının çoğunu yakalar, hepsini değil.

---

## 4. Plan Cache ve Parameter Sniffing

> **Benzetme —** Terzi, ilk gelen müşteriye göre bir kalıp çıkarır ve o kalıbı rafa kaldırır. Sonraki müşterilerde "zaten kalıbım var" deyip onu kullanır. İlk müşteri ince yapılıysa ve sıradaki adam iri yarıysa, kalıp uymaz. Terzinin yaptığı yanlış değil — kalıp çıkarmak zamandır, her seferinde yapmak israftır. Sorun, **ilk müşterinin herkesi temsil ettiği varsayımıdır**.

**Basitçe:** SQL Server bir sorguyu ilk çalıştırdığında nasıl çalıştıracağına dair bir plan hazırlar ve saklar. Aynı sorgu tekrar geldiğinde planı yeniden yapmaz, saklanmışı kullanır. Bu genelde iyidir. Ama plan, **ilk gelen parametre değerine göre** yapılmıştır. O değer sıra dışıysa, plan sonraki çağrılar için kötü olabilir.

**Teknik olarak:** **Execution plan (yürütme planı)** — Sorgunun hangi indeksle, hangi sırayla, hangi birleştirme yöntemiyle çalışacağının haritası. **Plan cache**, bu planların saklandığı bellek alanıdır. **Parameter sniffing (parametre koklama)**, optimizer'ın plan üretirken ilk çağrının parametre değerine bakmasıdır.

```sql
CREATE PROCEDURE sp_SehirMusterileri
    @Sehir NVARCHAR(50)
AS
BEGIN
    SELECT MusteriId, Ad, Soyad, KayitTarihi
    FROM Musteriler
    WHERE Sehir = @Sehir;
END
```

Diyelim `Musteriler` tablosunda 2 milyon satır var; 1,4 milyonu İstanbul, 800'ü Kayseri.

- İlk çağrı `@Sehir = 'Kayseri'` ile gelirse: optimizer "az satır dönecek" der, indeks araması (index seek) + key lookup planı yapar. Kayseri için mükemmeldir.
- Sonraki çağrı `@Sehir = 'İstanbul'` gelir. Aynı plan kullanılır: 1,4 milyon satır için tek tek key lookup yapılır. Felaket.
- Sıra tersine dönseydi: İstanbul için tablo taraması (scan) planlanırdı; Kayseri'nin 800 satırı için de tüm tablo taranırdı. Yine kötü, ama daha az kötü.

### Belirtileri ve çözümleri

| Çözüm | Nasıl | Ne zaman |
|---|---|---|
| `OPTION (RECOMPILE)` | Her çağrıda yeni plan üretir | Değer dağılımı çok dengesiz, sorgu seyrek çağrılıyor |
| `OPTIMIZE FOR (@p = 'değer')` | Sabit bir değere göre plan yapar | Tipik değeri biliyorsan |
| `OPTIMIZE FOR UNKNOWN` | Ortalama istatistiğe göre plan yapar | Hiçbir değer tipik değilse |
| Yerel değişkene kopyalama | Sniffing'i devre dışı bırakır (eski yöntem) | Kaçın; niyeti gizler |
| `sp_recompile` / plan temizleme | Planı bir kereliğine atar | Acil müdahale |
| SP'yi ikiye bölmek | Küçük ve büyük küme için ayrı SP | Dağılım iki uçluysa, en temiz çözüm |

**Bu benzetme şurada bozulur:** Terzi kalıbı raftan alırken müşteriye bakar, uymayacağını anlar. SQL Server bakmaz. Plan cache'teki plan, parametre değerine bakılmaksızın uygulanır. Optimizer yalnızca plan **üretirken** değere bakar, **kullanırken** değil. Farkın tamamı buradan çıkar.

---

## 5. SQL Injection ve Parametreli Sorgu

> **Benzetme —** Postanede havale formu doldurursun. Formda "alıcı adı" kutusu vardır, "tutar" kutusu vardır. Sen kutuları doldurursun, memur kutuların **içeriğine** göre işlem yapar. Şimdi düşün ki form yok; memura düz bir cümle yazıp veriyorsun: "Ahmet'e 500 lira gönder." Biri cümleye "Ahmet'e 500 lira gönder, ayrıca kasayı boşalt" yazarsa memur ikisini de yapar. Çünkü cümlede veri ile emir aynı yerde duruyor. Parametreli sorgu, formdaki kutudur: içine ne yazarsan yaz, emir olarak okunmaz.

**Basitçe:** SQL injection, kullanıcıdan gelen metni SQL cümlesinin içine yapıştırarak sorgu kurduğunda ortaya çıkar. Kullanıcı metnin içine SQL yazar, senin cümlen bozulur, onun cümlesi çalışır. Çözüm tek cümledir: kullanıcı verisini **asla** sorgu metnine yapıştırma, parametre olarak geçir.

**Teknik olarak:**

```csharp
// KÖTÜ — string birleştirme. Klasik açık.
var ad = Request.Query["ad"];
var sql = "SELECT * FROM Musteriler WHERE Ad = '" + ad + "'";
var liste = context.Musteriler.FromSqlRaw(sql).ToList();
```

Kullanıcı `ad` kutusuna `' OR 1=1 --` yazarsa cümle `WHERE Ad = '' OR 1=1 --'` hâline gelir ve bütün müşteriler döner. `'; DROP TABLE Siparisler; --` yazarsa tablo gider. Metin, kod olmuştur.

```csharp
// İYİ — parametreli. Kullanıcı ne yazarsa yazsın, değer olarak gider.
var liste = context.Musteriler
    .FromSqlInterpolated($"SELECT * FROM Musteriler WHERE Ad = {ad}")
    .ToList();
```

`FromSqlInterpolated` içindeki `{ad}`, metne yapıştırılmaz; `DbParameter` olarak gönderilir. Sunucuya giden şey şudur:

```sql
SELECT * FROM Musteriler WHERE Ad = @p0
```

Parametreli sorgunun iki kazancı vardır: **güvenlik** (veri asla kod olarak okunmaz) ve **plan tekrar kullanımı** (metin sabit kaldığı için plan cache çalışır).

### Adlandırmadaki tuzak: `Raw` ile `Interpolated`

```csharp
// FromSqlRaw + parametre: güvenli
context.Musteriler.FromSqlRaw(
    "SELECT * FROM Musteriler WHERE Sehir = {0}", sehir);

// FromSqlRaw + string interpolation: TEHLİKELİ
// Burada $ işareti C# tarafında birleştirir, EF Core'a hazır metin gider.
context.Musteriler.FromSqlRaw(
    $"SELECT * FROM Musteriler WHERE Sehir = '{sehir}'");   // açık!

// FromSqlInterpolated + $: güvenli — EF Core interpolasyonu parametreye çevirir
context.Musteriler.FromSqlInterpolated(
    $"SELECT * FROM Musteriler WHERE Sehir = {sehir}");
```

> Kural: `FromSqlRaw` görüyorsan başında `$` **olmamalı**. `$` görüyorsan metot `FromSqlInterpolated` **olmalı**. Bu tek cümle, EF Core'daki injection hatalarının çoğunu önler.

### Dinamik SQL tuzağı ve `sp_executesql`

Bazen sorgu metnini gerçekten dinamik kurmak gerekir — mesela sıralama kolonu kullanıcıdan geliyordur. Burada da kural bozulmaz: **değerler parametre, yapı ise beyaz liste.**

```sql
-- İYİ: değer parametre olarak, kolon adı beyaz listeyle
CREATE PROCEDURE sp_MusteriAra
    @Sehir NVARCHAR(50),
    @SiralamaKolonu SYSNAME = N'Ad'
AS
BEGIN
    SET NOCOUNT ON;

    -- Yapı beyaz listeden geçer: listede yoksa varsayılana düşer
    IF @SiralamaKolonu NOT IN (N'Ad', N'Soyad', N'KayitTarihi', N'Sehir')
        SET @SiralamaKolonu = N'Ad';

    DECLARE @sql NVARCHAR(MAX) =
        N'SELECT MusteriId, Ad, Soyad, Sehir
          FROM Musteriler
          WHERE Sehir = @p_Sehir
          ORDER BY ' + QUOTENAME(@SiralamaKolonu);

    EXEC sp_executesql
         @sql,
         N'@p_Sehir NVARCHAR(50)',   -- parametre bildirimi
         @p_Sehir = @Sehir;          -- parametre değeri
END
```

**Bu benzetme şurada bozulur:** Postane formu benzetmesi, kutuların içeriğinin asla emre dönüşmediğini söyler. Parametreli sorguda bu doğrudur. Ama uygulama içinde başka enjeksiyon yüzeyleri kalır: `LIKE` içinde kullanılan `%` ve `_` karakterleri, `IN (...)` listesinin elle kurulması, `ORDER BY` kolonunun dışarıdan gelmesi. Bunlar SQL injection değildir ama aynı kapıdan girer: **yapıyı dışarıdan alma.**

---

## 6. Fonksiyonlar: Scalar ve Table-Valued

> **Benzetme —** Markette terazi ile kasa. Terazi tek bir değer verir: ağırlık. Kasa ise bir fiş verir: satırlar hâlinde liste. İkisi de "hesap yapan alet"tir ama çıktıları farklı türdür. Bir de şu var: teraziyi her ürün için ayrı ayrı kullanırsan sıra uzar; kasa hepsini tek seferde işler.

**Basitçe:** SQL Server'da iki tür kullanıcı fonksiyonu vardır. Scalar fonksiyon tek bir değer döndürür (bir sayı, bir metin). Table-valued fonksiyon ise bir tablo döndürür, sorguda tablo gibi kullanılır.

**Teknik olarak:**

```sql
-- Scalar UDF: tek değer döner
CREATE FUNCTION dbo.fn_SiparisTutari (@SiparisId INT)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE @Tutar DECIMAL(18,2);
    SELECT @Tutar = SUM(BirimFiyat * Adet)
    FROM SiparisDetaylari
    WHERE SiparisId = @SiparisId;
    RETURN ISNULL(@Tutar, 0);
END
```

```sql
-- Inline table-valued function (iTVF): tablo döner, tek SELECT'ten oluşur
CREATE FUNCTION dbo.fn_MusteriSiparisleri (@MusteriId INT)
RETURNS TABLE
AS
RETURN
(
    SELECT s.SiparisId, s.SiparisTarihi,
           SUM(sd.BirimFiyat * sd.Adet) AS Tutar
    FROM Siparisler s
    JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
    WHERE s.MusteriId = @MusteriId
    GROUP BY s.SiparisId, s.SiparisTarihi
);
```

| Tür | Döndürdüğü | Gövde | Performans |
|---|---|---|---|
| Scalar UDF | Tek değer | `BEGIN...END` | Riskli — aşağıya bak |
| Inline TVF (iTVF) | Tablo | Tek `RETURN (SELECT ...)` | İyi — view gibi açılır |
| Multi-statement TVF (mTVF) | Tablo | Tablo değişkeni doldurulur | Kötü — tahmin yanlış olur |

### Scalar UDF'in performans zararı

Bu, SQL Server'da en sık rastlanan gizli yavaşlık sebebidir.

```sql
-- KÖTÜ: scalar UDF, dönen her satır için ayrı çalışır
SELECT s.SiparisId, s.SiparisTarihi, dbo.fn_SiparisTutari(s.SiparisId) AS Tutar
FROM Siparisler s
WHERE s.SiparisTarihi >= '2026-01-01';
```

100.000 sipariş dönüyorsa fonksiyon 100.000 kez çağrılır. Her çağrı küçük bir sorgudur. Dahası (SQL Server 2019 öncesinde) bu çağrılar plana **dâhil edilmez**: yürütme planına bakarsın, sorgu ucuz görünür, gerçekte dakikalarca çalışır. Paralel plan da üretilmez — scalar UDF içeren sorgu tek iş parçacığına düşer.

```sql
-- İYİ: aynı iş bir JOIN ile, tek seferde
SELECT s.SiparisId, s.SiparisTarihi, ISNULL(t.Tutar, 0) AS Tutar
FROM Siparisler s
LEFT JOIN (
    SELECT SiparisId, SUM(BirimFiyat * Adet) AS Tutar
    FROM SiparisDetaylari
    GROUP BY SiparisId
) t ON t.SiparisId = s.SiparisId
WHERE s.SiparisTarihi >= '2026-01-01';
```

> **Sürüm notu:** SQL Server 2019 ile gelen **Scalar UDF Inlining**, uygun scalar fonksiyonları otomatik olarak sorgu içine açar ve bu maliyeti büyük ölçüde kaldırır. Ancak her fonksiyon uygun değildir: içinde `WHILE`, `EXEC`, tablo değişkeni ya da yan etki varsa inline edilmez. Veritabanının uyumluluk seviyesi (compatibility level) 150 ve üstü olmalıdır. Yani "2019 kullanıyorum, sorun yok" demek yanlıştır; inline olup olmadığını plandan kontrol etmek gerekir.

> **`WHERE` içinde fonksiyon:** `WHERE dbo.fn_SiparisTutari(s.SiparisId) > 1000` yazarsan indeks kullanılamaz. Aynı şey yerleşik fonksiyonlar için de geçerlidir: `WHERE YEAR(SiparisTarihi) = 2026` indeksi bozar. Bunun yerine `WHERE SiparisTarihi >= '2026-01-01' AND SiparisTarihi < '2027-01-01'` yaz. Kolonu fonksiyona sokmak, o kolondaki indeksi devre dışı bırakır — buna **non-sargable** sorgu denir.

**Bu benzetme şurada bozulur:** Terazi ve kasa benzetmesi ikisini eşit gösterir. Gerçekte scalar UDF ile inline TVF arasındaki fark bir tercih meselesi değildir. Aynı işi yapan iki fonksiyondan biri saniyeler, diğeri dakikalar sürebilir. Modern yaklaşım: scalar UDF yazma; yazacaksan tek satırlık inline TVF olarak yaz ve `CROSS APPLY` ile kullan.

---

## 7. Trigger — Kendiliğinden Çalışan Kod

> **Benzetme —** Apartman girişindeki fotoselli lamba. Kimse düğmeye basmaz; kapıdan biri geçtiği anda lamba yanar. Faydalıdır — elin doluyken düğme aramazsın. Ama bir gün lamba yanmazsa ya da gece boyunca yanıp kalırsa, sebebini bulmak zordur; çünkü onu kimse "çalıştırmamıştır". Görünmez bir kural tarafından tetiklenmiştir.

**Basitçe:** Trigger, bir tabloya `INSERT`, `UPDATE` ya da `DELETE` yapıldığında **otomatik olarak** çalışan koddur. Kimse çağırmaz, kendiliğinden devreye girer.

**Teknik olarak:** **Trigger (tetikleyici)** — Bir tablo veya view üzerindeki veri değişikliğine bağlı olarak çalışan özel bir stored procedure.

```sql
CREATE TRIGGER trg_Urunler_FiyatLog
ON Urunler
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT UPDATE(Fiyat)     -- Fiyat kolonu değişmediyse hiç uğraşma
        RETURN;

    INSERT INTO FiyatGecmisi (UrunId, EskiFiyat, YeniFiyat, DegisimTarihi, Kullanici)
    SELECT i.UrunId, d.Fiyat, i.Fiyat, GETDATE(), SUSER_SNAME()
    FROM inserted i
    JOIN deleted  d ON d.UrunId = i.UrunId
    WHERE i.Fiyat <> d.Fiyat;
END
```

### `INSERTED` ve `DELETED` sanal tabloları

Trigger içinde iki özel tablo vardır. Bunlar diskte yoktur, sadece trigger çalışırken bellekte durur.

| İşlem | `inserted` içeriği | `deleted` içeriği |
|---|---|---|
| `INSERT` | Eklenen yeni satırlar | Boş |
| `UPDATE` | Satırların **yeni** hâli | Satırların **eski** hâli |
| `DELETE` | Boş | Silinen satırlar |

`UPDATE` işleminde ikisi birden dolar; eski ve yeni hâli anahtar üzerinden birleştirerek karşılaştırırsın. Yukarıdaki örnekte tam olarak bu yapılıyor.

### En büyük hata: satır bazlı düşünmek

Trigger, **değişen her satır için bir kez değil**, `INSERT`/`UPDATE` ifadesi başına **bir kez** çalışır. Tek ifade 5.000 satır güncellediyse trigger bir kez çalışır ve `inserted` tablosunda 5.000 satır bulunur.

```sql
-- KÖTÜ: tek satır varsayımı. Toplu güncellemede sadece BİR satır loglanır.
CREATE TRIGGER trg_Kotu ON Urunler AFTER UPDATE
AS
BEGIN
    DECLARE @UrunId INT, @YeniFiyat DECIMAL(18,2);
    SELECT @UrunId = UrunId, @YeniFiyat = Fiyat FROM inserted;  -- rastgele bir satır!
    INSERT INTO FiyatGecmisi (UrunId, YeniFiyat) VALUES (@UrunId, @YeniFiyat);
END
```

```sql
-- İYİ: küme mantığıyla. Kaç satır olursa olsun doğru çalışır.
CREATE TRIGGER trg_Iyi ON Urunler AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO FiyatGecmisi (UrunId, YeniFiyat, DegisimTarihi)
    SELECT UrunId, Fiyat, GETDATE() FROM inserted;
END
```

### Neden dikkatli kullanılmalı

| Risk | Açıklama |
|---|---|
| Görünmezlik | Kodda `UPDATE Urunler` yazar; trigger'ın ne yaptığı orada görünmez |
| Transaction'a dâhil | Trigger aynı transaction içindedir; yavaşsa `UPDATE` de yavaşlar |
| Hata yayılması | Trigger hata verirse asıl işlem de geri alınır |
| Zincirleme tetikleme | Trigger başka tabloyu günceller, o da kendi trigger'ını tetikler (varsayılan iç içe derinlik 32) |
| `@@IDENTITY` bozulması | Trigger başka tabloya `INSERT` yaparsa kimlik değeri şaşar |
| Hata ayıklama zorluğu | Sorunun kaynağını bulmak için tüm trigger'ları bilmek gerekir |

**Nerede makul:** Denetim/geçmiş (audit) kaydı, veri bütünlüğü kuralı, eski sistemle senkronizasyon. **Nerede kaçınılmalı:** İş mantığı. "Sipariş girilince müşteriye e-posta gönder" gibi bir iş trigger'a konmaz — hem transaction'ı uzatır hem de e-posta gönderimi başarısız olursa sipariş de iptal olur.

**Bu benzetme şurada bozulur:** Fotoselli lamba benzetmesi trigger'ın "yan iş" gibi göründüğünü anlatır. Gerçekte trigger yan iş değildir; asıl işlemin **tam ortasındadır**. Lamba yanmazsa kapıdan yine geçersin. Trigger hata verirse `UPDATE` de geri alınır. Trigger, ana işlemin kaderine bağlıdır — ve ana işlemin kaderini de belirler.

---

## 8. Transaction ve ACID

> **Benzetme —** Tapu devri. Alıcı parayı verir, satıcı tapuyu devreder. Bu iki adımın ikisi de olmalı. Para verilip tapu devredilmezse alıcı, tapu devredilip para alınmazsa satıcı mağdur olur. Bu yüzden işlem memurun önünde, tek oturumda yapılır: ya ikisi birden tamamlanır ya da hiçbiri olmamış sayılır ve herkes eski hâline döner.

**Basitçe:** Transaction, birden çok veritabanı işlemini tek bir bütün hâline getirir. Hepsi başarılı olursa kalıcı olur (`COMMIT`), biri bile başarısız olursa hepsi geri alınır (`ROLLBACK`). Yarım kalmış durum yoktur.

**Teknik olarak:** **Transaction (işlem)** — Atomik olarak yürütülen SQL ifadeleri kümesi. Dört özelliğiyle anılır: **ACID**.

| Harf | Adı | Benzetme |
|---|---|---|
| **A** | Atomicity (atomiklik) | **Kibrit çöpü:** Ya tamamen yanar ya hiç yanmaz. Yarım yanmış transaction olmaz. |
| **C** | Consistency (tutarlılık) | **Muhasebe defteri:** Girişten önce denk olan defter, çıkıştan sonra da denk olmalıdır. Kısıtlar, foreign key'ler, check'ler bozulmaz. |
| **I** | Isolation (yalıtım) | **Banka gişesi kabini:** Sen işlem yaparken arkandaki sıradaki adam senin yarım evrakını görmez. |
| **D** | Durability (kalıcılık) | **Noter tasdiki:** Onay verildikten sonra elektrik kesilse, sunucu çökse bile kayıt durur. SQL Server bunu önce log'a yazarak (write-ahead logging) sağlar. |

```sql
BEGIN TRANSACTION;

UPDATE Hesaplar SET Bakiye = Bakiye - 1000 WHERE HesapId = 1;
UPDATE Hesaplar SET Bakiye = Bakiye + 1000 WHERE HesapId = 2;

COMMIT TRANSACTION;
-- ya da hata durumunda: ROLLBACK TRANSACTION;
```

### `XACT_ABORT` — atlanan ayar

SQL Server'da bir çalışma zamanı hatası oluştuğunda transaction **otomatik olarak geri alınmaz**. Varsayılan davranışta çoğu hata sadece o ifadeyi iptal eder, sonraki satırlar çalışmaya devam eder.

```sql
-- GÜVENLİ: hata anında transaction tamamen iptal edilir
SET XACT_ABORT ON;
BEGIN TRANSACTION;
    UPDATE Hesaplar SET Bakiye = Bakiye - 1000 WHERE HesapId = 1;
    UPDATE Hesaplar SET Bakiye = Bakiye + 1000 WHERE HesapId = 2;
COMMIT TRANSACTION;
```

Pratik kural: transaction içeren her stored procedure'ün başına `SET XACT_ABORT ON;` yaz. `TRY...CATCH` ile birlikte kullanmak en sağlam kalıptır.

### `@@TRANCOUNT`, iç içe transaction ve savepoint

T-SQL'de **gerçek iç içe transaction yoktur**. `BEGIN TRANSACTION` iç içe yazılabilir ama sadece bir sayaç (`@@TRANCOUNT`) artar.

```sql
BEGIN TRANSACTION;          -- @@TRANCOUNT = 1
    BEGIN TRANSACTION;      -- @@TRANCOUNT = 2  (yeni transaction açılmadı!)
        INSERT INTO Siparisler (MusteriId, SiparisTarihi, Iptal) VALUES (1, GETDATE(), 0);
    COMMIT TRANSACTION;     -- @@TRANCOUNT = 1  (hiçbir şey kalıcı olmadı)
ROLLBACK TRANSACTION;       -- @@TRANCOUNT = 0  (HER ŞEY geri alındı)
```

Yani içteki bir `ROLLBACK`, dıştaki işlemi de öldürür. Bu yüzden bir SP kendi transaction'ını körü körüne açmamalı, önce `IF @@TRANCOUNT = 0` diye kontrol etmelidir.

**Savepoint (kayıt noktası)**, transaction'ın bir kısmını geri almanın tek yoludur:

```sql
BEGIN TRANSACTION;

INSERT INTO Siparisler (MusteriId, SiparisTarihi, Iptal) VALUES (17, GETDATE(), 0);
DECLARE @SiparisId INT = SCOPE_IDENTITY();

SAVE TRANSACTION KalemOncesi;          -- kayıt noktası koy

BEGIN TRY
    INSERT INTO SiparisDetaylari (SiparisId, UrunId, Adet, BirimFiyat)
    VALUES (@SiparisId, 9999, 1, 0);   -- olmayan ürün: FK hatası
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION KalemOncesi;  -- sadece kalem eklemeyi geri al
END CATCH

COMMIT TRANSACTION;                    -- sipariş başlığı kalıcı oldu
```

**Bu benzetme şurada bozulur:** Tapu devri benzetmesinde işlem ya olur ya olmaz; üçüncü hâl yoktur. Veritabanında üçüncü hâl vardır: **uncommittable transaction**. Hata olmuştur, transaction hâlâ açıktır ama artık `COMMIT` edilemez — sadece `ROLLBACK` kabul eder. Memur "ne devir ne iptal, kâğıt elimde asılı duruyor" demiş gibidir. Bu durumu temizlemezsen bağlantı kapanana kadar kilitler tutulmaya devam eder.

---

## 9. Isolation Level'lar ve Okuma Anomalileri

> **Benzetme —** Kütüphanede bir kitabı okuyorsun. Aynı anda bir başkası o kitabın sayfalarını değiştiriyor. Kütüphanenin dört ayrı politikası olabilir: (1) sayfa yarım yazılmışken bile okumana izin verir, (2) sadece yazımı bitmiş sayfaları gösterir, (3) senin okuduğun sayfaları sen bitirene kadar kilitler, (4) kitaba yeni sayfa eklenmesini bile yasaklar. Politika sıkılaştıkça doğruluk artar, ama diğer okuyucular sırada bekler.

**Basitçe:** Aynı anda birden fazla işlem çalışırken, birinin yarım işini diğerinin görüp görmeyeceğini isolation level belirler. Gevşek seviye hızlıdır ama tutarsız veri okuyabilirsin. Sıkı seviye doğrudur ama beklemeye ve kilide yol açar.

**Teknik olarak:** Önce üç anomaliyi tanı.

| Anomali | Ne olur | Senaryo |
|---|---|---|
| **Dirty read** (kirli okuma) | Henüz `COMMIT` edilmemiş, sonra geri alınacak veriyi okursun | A stoğu 100'den 5'e düşürdü, henüz commit etmedi; B bunu okuyup "stok az" dedi; A `ROLLBACK` yaptı. B'nin okuduğu değer hiç var olmadı. |
| **Non-repeatable read** (tekrarlanamayan okuma) | Aynı satırı iki kez okursun, değer değişmiştir | Raporun başında ürünün fiyatı 100 okundu; ortada başkası 120 yaptı ve commit etti; raporun sonunda aynı ürün 120 okundu. Rapor kendi içinde tutarsız. |
| **Phantom read** (hayalet okuma) | Aynı şartla iki kez sorarsın, ikincisinde **yeni satırlar** çıkar | `WHERE Sehir='Kayseri'` ile 800 satır sayıldı; araya yeni müşteri eklendi; aynı sorgu 801 döndü. Satır değişmedi, **küme** değişti. |

### Seviyeler ve hangi anomaliyi engellediği

| Isolation level | Dirty read | Non-repeatable read | Phantom read | Davranışı |
|---|---|---|---|---|
| READ UNCOMMITTED | Olur | Olur | Olur | Hiç kilit beklemez; yarım veriyi de okur |
| READ COMMITTED | Engeller | Olur | Olur | **SQL Server varsayılanı.** Sadece commit edilmiş veriyi okur |
| REPEATABLE READ | Engeller | Engeller | Olur | Okuduğu satırları transaction sonuna kadar kilitli tutar |
| SERIALIZABLE | Engeller | Engeller | Engeller | Aralık (range) kilidi koyar; yeni satır eklenmesini de engeller |
| SNAPSHOT | Engeller | Engeller | Engeller | Kilit kullanmaz; transaction başındaki anlık görüntüyü okur |

### SNAPSHOT — kilitsiz doğruluk

`SNAPSHOT`, diğerlerinden farklı çalışır. Kilit almak yerine satırların eski sürümlerini `tempdb`'de tutar ve sana transaction'ın **başladığı andaki** hâli gösterir.

Ayrıca `READ_COMMITTED_SNAPSHOT` (RCSI) diye bir veritabanı ayarı vardır. Açıldığında varsayılan `READ COMMITTED` davranışı kilit yerine sürüm okumaya döner. Uygulama kodunu değiştirmeden okuma bloklanmalarını büyük ölçüde bitirir; birçok üretim sisteminde ilk tercih edilen ayardır.

### `NOLOCK` neden tehlikeli

`WITH (NOLOCK)`, o sorgu için `READ UNCOMMITTED` demektir. "Sorgu yavaş, `NOLOCK` koyalım" en yaygın yanlış alışkanlıktır.

```sql
-- TEHLİKELİ: raporda olmayan verileri görebilirsin
SELECT SUM(Tutar) FROM Siparisler WITH (NOLOCK) WHERE SiparisTarihi >= '2026-01-01';
```

`NOLOCK` yalnızca dirty read'e yol açmaz. Daha az bilinen sorunları da vardır:

| Sorun | Açıklama |
|---|---|
| Kirli okuma | Geri alınacak veriyi okursun; rapordaki tutar hiç var olmamış olabilir |
| **Atlanan satır** | Sayfa bölünmesi (page split) sırasında var olan bir satırı hiç görmeyebilirsin |
| **Tekrarlanan satır** | Aynı satırı iki kez okuyabilirsin; `SUM` şişer |
| Hata (601) | Taranan veri hareket ederse sorgu "Could not continue scan" hatasıyla düşebilir |

Doğru çözüm `NOLOCK` değil, `RCSI` açmak ya da `SNAPSHOT` kullanmaktır: ikisi de kilit almaz ama **tutarlı** veri verir. `NOLOCK` yalnızca "yaklaşık doğru yeter" denilen yerlerde kabul edilebilir — canlı izleme ekranı, kaba sayaç. Para, stok, fatura söz konusuysa asla.

**Bu benzetme şurada bozulur:** Kütüphane benzetmesinde "sıkı politika = yavaş ama doğru" düz bir denge gibi görünür. `SNAPSHOT` bu dengeyi bozar: hem sıkı hem hızlıdır, çünkü kilit yerine **eski sürümü** okur. Bedeli hızdan değil, `tempdb` alanından ve yazma çakışmalarını ele alma zorunluluğundan ödenir. Yani seviyeler tek bir çizgi üzerinde sıralanmaz; `SNAPSHOT` çizginin dışındadır.

---

## 10. Kilitler ve Deadlock

> **Benzetme —** Dar bir sokakta iki araba karşı karşıya gelmiş. Biri geri gitse ikisi de geçecek, ama ikisi de "önce sen geri git" diyor. Kimse ilerleyemez. Trafik polisi gelir, birini zorla geri alır — o araba yolunu kaybeder ama trafik açılır. Veritabanında polisin adı **deadlock monitor**, geri alınan arabanın adı **deadlock victim**'dir.

**Basitçe:** Veritabanı, iki kişinin aynı satırı aynı anda bozmasını engellemek için satırları kilitler. Kilitler normalde sorun değildir; kısa sürer, sıra ilerler. Ama iki işlem birbirinin beklediği kilidi tutuyorsa ikisi de sonsuza kadar bekler. Buna deadlock denir.

**Teknik olarak:** Temel kilit türleri:

| Kilit | Adı | Kim kimi engeller |
|---|---|---|
| **S** | Shared (paylaşımlı) | Okuma için alınır. Başka S ile uyumlu, X ile değil |
| **X** | Exclusive (dışlayıcı) | Yazma için alınır. Hiçbir şeyle uyumlu değil |
| **U** | Update (güncelleme) | Güncellenecek satırı tararken alınır; X'e dönüşür |
| **IS/IX** | Intent (niyet) | Sayfa/tablo seviyesinde "altta kilidim var" işareti |

### İki oturumlu somut deadlock

```sql
-- OTURUM A
BEGIN TRANSACTION;
UPDATE Urunler    SET StokAdedi = StokAdedi - 1 WHERE UrunId = 5;   -- X kilidi: Urunler(5)
WAITFOR DELAY '00:00:05';
UPDATE Musteriler SET Bakiye = Bakiye - 100    WHERE MusteriId = 17; -- Musteriler(17) bekliyor
COMMIT TRANSACTION;
```

```sql
-- OTURUM B (aynı anda)
BEGIN TRANSACTION;
UPDATE Musteriler SET Bakiye = Bakiye - 100    WHERE MusteriId = 17; -- X kilidi: Musteriler(17)
WAITFOR DELAY '00:00:05';
UPDATE Urunler    SET StokAdedi = StokAdedi - 1 WHERE UrunId = 5;   -- Urunler(5) bekliyor
COMMIT TRANSACTION;
```

A `Urunler(5)` üzerinde X kilidi tutarken `Musteriler(17)`i ister; B tam tersini yapar. Döngü kapanır, ikisi de ilerleyemez. Deadlock monitor devreye girer ve oturumlardan birine `Msg 1205 ... chosen as the deadlock victim` hatasını verip geri alır.

SQL Server kurbanı, geri alınması **daha ucuz** olana (daha az log üretmiş olana) göre seçer. `SET DEADLOCK_PRIORITY LOW;` ile bir oturumu gönüllü kurban yapabilirsin.

### Nasıl önlenir

| Önlem | Açıklama |
|---|---|
| **Aynı sırayla kilitle** | En etkili önlem. Tüm kod yolları tablolara hep aynı sırada dokunsun (önce `Urunler`, sonra `Musteriler`) |
| **Transaction'ı kısa tut** | Kilit ne kadar kısa tutulursa çakışma o kadar az |
| Transaction içinde dış çağrı yapma | HTTP, dosya, kullanıcı beklemesi kilidi dakikalarca tutar |
| Doğru indeks | İndeks yoksa tablo taranır, gereksiz satırlar kilitlenir |
| İhtiyaç anında `UPDLOCK` | Okuyup sonra güncelleyeceksen okurken `WITH (UPDLOCK)` al; S→X yükseltmesinden doğan deadlock'u önler |
| `READ_COMMITTED_SNAPSHOT` | Okuma-yazma çakışmalarını büyük ölçüde bitirir (yazma-yazma deadlock'unu bitirmez) |
| Uygulamada yeniden deneme (retry) | 1205 hatasında kısa bir bekleyip tekrar dene |

### Deadlock graph'a kısa bakış

SQL Server her deadlock'u XML olarak kaydeder. En kolay erişim `system_health` Extended Events oturumudur; varsayılan olarak açıktır ve son deadlock'ları tutar.

Deadlock graph XML'inde üç şeye bakılır:

- **`process-list`** — Taraflar. Her birinin çalıştırdığı son ifade (`inputbuf`) ve `isolationlevel`'ı burada yazar.
- **`resource-list`** — Çakışılan kaynak: hangi tablo, hangi indeks (`objectname`, `indexname`), hangi kilit modu.
- **`victim`** — Kurban seçilen işlemin `id`'si.

Teşhis hemen hemen her zaman şu iki sonuçtan birine varır: ya tablolara farklı sıralarda dokunulmuştur, ya da bir indeks eksik olduğu için gereğinden çok satır kilitlenmiştir.

**Bu benzetme şurada bozulur:** Dar sokak benzetmesi iki arabayla sınırlıdır. Gerçek deadlock'lar üç, dört, beş oturumlu döngüler olabilir: A B'yi, B C'yi, C de A'yı bekler. Ayrıca kilitlenen şey hep "satır" değildir; sayfa, indeks anahtarı, hatta ayrılmış bir uygulama kilidi (`sp_getapplock`) da olabilir. Graph'a bakmadan "hangi iki satır çakıştı" diye tahmin yürütmek çoğu zaman yanlış yere götürür.

---

## 11. EF Core Tarafında SP ve Transaction

> **Benzetme —** Otomatik vitesli araba. Vites değiştirmeyi düşünmezsin, araba senin adına yapar. Ama bazı durumlarda manuel moda almak istersin: dik yokuşta, çekerken, kaygan yolda. EF Core da çoğu zaman transaction'ı senin adına yönetir; kontrolü ne zaman eline alacağını bilmen gerekir.

**Basitçe:** EF Core, `SaveChanges()` çağırdığında yaptığı tüm değişiklikleri **zaten** tek bir transaction içine alır. Elle transaction açman yalnızca birden fazla `SaveChanges`'ı ya da ham SQL'i tek bütün hâline getirmen gerektiğinde lazım olur.

**Teknik olarak:**

### Ham SQL ve stored procedure çağırma

```csharp
// 1) Sonuç kümesi dönen SP — entity tipine eşlenir
var siparisler = await context.Siparisler
    .FromSqlInterpolated($"EXEC sp_MusteriSiparisleri @MusteriId = {musteriId}")
    .ToListAsync();

// 2) Sonuç dönmeyen SP / komut — etkilenen satır sayısını döner
int etkilenen = await context.Database
    .ExecuteSqlInterpolatedAsync(
        $"EXEC sp_StokDus @UrunId = {urunId}, @Adet = {adet}");
```

`FromSql*` kullanırken iki kural var: sorgunun döndürdüğü kolonlar entity'nin **tüm** kolonlarını karşılamalıdır (eksik kolon çalışma zamanı hatası verir) ve bu metotlar yalnızca bir `DbSet` üzerinden çağrılabilir. SP çıktısını kendi DTO'na eşlemek istiyorsan tipi `modelBuilder.Entity<Dto>().HasNoKey()` ile keyless olarak tanımlarsın. Ayrıca `FromSqlRaw("SELECT ...")` sonucuna LINQ eklenebilir, ama `EXEC` çıktısına eklenemez.

### `SaveChanges`'ın kendi transaction'ı

```csharp
context.Siparisler.Add(yeniSiparis);
context.SiparisDetaylari.AddRange(kalemler);
context.Urunler.Update(guncelUrun);

await context.SaveChangesAsync();   // üçü de TEK transaction içinde
```

`SaveChanges` tüm bekleyen değişiklikleri tek transaction'da uygular. Biri başarısız olursa hepsi geri alınır. Ayrı bir `BeginTransaction` yazmana gerek yoktur.

### `BeginTransaction` — ne zaman gerekir

İki durumda: birden çok `SaveChanges` çağrısını birleştirmek, ya da EF Core işlemleriyle ham SQL'i aynı bütüne almak.

```csharp
await using var tx = await context.Database.BeginTransactionAsync();
try
{
    context.Siparisler.Add(siparis);
    await context.SaveChangesAsync();          // Id burada oluşur

    foreach (var k in kalemler) k.SiparisId = siparis.SiparisId;
    context.SiparisDetaylari.AddRange(kalemler);
    await context.SaveChangesAsync();

    await context.Database.ExecuteSqlInterpolatedAsync(
        $"EXEC sp_StokDus @UrunId = {urunId}, @Adet = {adet}");

    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

### `TransactionScope`

Birden fazla `DbContext`i ya da farklı veri kaynaklarını tek transaction'a almak için `System.Transactions.TransactionScope` kullanılır.

```csharp
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = System.Transactions.IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled);   // async için ŞART

using (var ctx1 = new MagazaDbContext(options)) { /* ... */ await ctx1.SaveChangesAsync(); }
using (var ctx2 = new LogDbContext(options))   { /* ... */ await ctx2.SaveChangesAsync(); }

scope.Complete();   // çağrılmazsa rollback
```

> **MvcCv bağlantısı:** MvcCv'deki `GenericRepository` her metodunda `SaveChanges()` çağırıyorsa, iki repository üzerinden yapılan iki işlem **iki ayrı transaction**'dır. Biri başarılı olup diğeri patlarsa veri yarım kalır. Bu sorunun standart çözümü **Unit of Work** desenidir: repository'ler değişiklikleri işler, `SaveChanges`'ı tek bir yerden — Unit of Work — çağırırsın. `DbContext` zaten bir Unit of Work'tür; üstüne yazılan katmanın bunu bozmaması gerekir.

**Bu benzetme şurada bozulur:** Otomatik vites benzetmesi, manuel moda geçmenin her zaman mümkün olduğunu ima eder. EF Core'da bazı kapılar kapalıdır. `ExecuteUpdate`/`ExecuteDelete` change tracker'ı **atlar**: bellekteki entity'ler eski değerleriyle kalır, aynı `DbContext` içinde sonra `SaveChanges` çağırırsan eski veriyi geri yazabilirsin. Otomatik vites, sen manuel moddayken sessizce kendi kararını uygulamış olur.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| View | İsimle saklanan `SELECT` ifadesi; veri tutmaz |
| Indexed view | Üstünde kümelenmiş indeks olan, sonucu diskte tutulan view |
| Stored procedure | Veritabanında saklanan, parametre alan T-SQL yordamı |
| Execution plan | Sorgunun nasıl çalıştırılacağının haritası |
| Parameter sniffing | Planın ilk çağrının parametre değerine göre üretilmesi |
| SQL injection | Kullanıcı verisinin SQL cümlesine kod olarak sızması |
| Scalar UDF | Tek değer döndüren kullanıcı fonksiyonu |
| Trigger | Veri değişikliğinde otomatik çalışan yordam |
| Transaction | Bölünemez şekilde yürütülen ifadeler kümesi |
| ACID | Atomicity, Consistency, Isolation, Durability |
| Savepoint | Transaction'ın bir kısmına dönülmesini sağlayan işaret |
| Dirty read | Commit edilmemiş veriyi okuma |
| Non-repeatable read | Aynı satırı iki kez okuyup farklı değer bulma |
| Phantom read | Aynı şartla iki kez sorup farklı satır kümesi bulma |
| SNAPSHOT isolation | Kilit yerine satır sürümü okuyan izolasyon seviyesi |
| Deadlock | İki veya daha çok işlemin birbirinin kilidini beklemesi |
| `TransactionScope` | Birden çok kaynağı tek transaction'a alan .NET yapısı |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `04-EF-Core-Temelleri-ve-Migration.md` (Perşembe)
