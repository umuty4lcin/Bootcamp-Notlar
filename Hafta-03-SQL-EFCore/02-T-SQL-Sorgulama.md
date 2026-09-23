# Hafta 3 · Salı — T-SQL Sorgulama

**Okuma süresi:** ~55 dk
**Neden bu konu:** EF Core yazdığın LINQ'i SQL'e çevirir; ama çevirinin ne ürettiğini okuyamazsan neyin yavaş olduğunu da göremezsin. Ayrıca her projede LINQ'in yetmediği bir yer çıkar — rapor, toplu güncelleme, karmaşık sayfalama — ve oraya ham SQL yazarsın. MvcCv'de `GenericRepository` üzerinden geçtiğin sorguların altında tam olarak bu cümleler vardı. Bootcamp'te "bu sorgu neden 8 saniye sürüyor" sorusuna cevap verebilmen bekleniyor.

---

## Önce Basitçe

SQL'i öğrenmenin en kolay yolu onu bir dil değil, bir **sipariş formu** gibi görmektir. Formda birkaç kutu vardır: nereden alacağım, hangilerini alacağım, nasıl gruplayacağım, ne göstereceğim, nasıl sıralayacağım. Sen kutuları doldurursun, işi kimin nasıl yapacağına karışmazsın. Bu yüzden SQL'e **bildirimsel (declarative)** dil denir: "ne istediğini" söylersin, "nasıl yapılacağını" değil.

Ama formun kutuları yazdığın sırayla işlenmez. Sen `SELECT` ile başlayıp `FROM` ile devam edersin; veritabanı ise önce `FROM`'a bakar, sonra `WHERE`, sonra `GROUP BY` der. Bu ayrımı bilmezsen bir gün "`SELECT`'te verdiğim isim neden `WHERE`'de tanınmıyor" diye takılırsın. Cevap basittir: `WHERE` çalıştığında o isim henüz doğmamıştır.

İkinci büyük konu birleştirmedir. Veri normalize edilmiş, yani parça parça duruyor. "Hangi müşteri ne sipariş etti" sorusunun cevabı tek tabloda yok. `JOIN` iki defteri yan yana koyup eşleştirme işidir. Eşleşmeyenleri atarsan `INNER JOIN`, sol defteri ne olursa olsun korursan `LEFT JOIN` olur. Buradaki en yaygın hata, `LEFT JOIN` yazıp sonra onu farkında olmadan `INNER JOIN`'e dönüştürmektir.

Üçüncüsü gruplamadır. Bin satırdan on satırlık bir özet çıkarmak istersin: müşteri başına sipariş sayısı, şehir başına ciro. `GROUP BY` satırları öbeklere ayırır, toplama fonksiyonları her öbekten tek bir sayı üretir. `WHERE` gruplama öncesinde tek tek satırları eler, `HAVING` gruplama sonrasında öbekleri eler. İkisini karıştırmak hem yanlış sonuç hem gereksiz yavaşlık üretir.

Dördüncüsü, bir sorgunun içine başka bir sorgu koymaktır. Bazen ara bir sonuç gerekir: "ortalamanın üstünde harcayan müşteriler". Bunu alt sorguyla, CTE ile ya da window function ile çözersin. Üçünün de yeri vardır; hangisinin ne zaman daha okunaklı ve daha hızlı olduğunu bu notta göreceksin.

Beşincisi, satırlara bakarken grubu da görebilmektir. Window function'lar tam olarak bunu yapar: satırı yok etmeden, satırın yanına grubun bilgisini yazar. "Her müşterinin son siparişi", "her kategoriden en pahalı üç ürün", "her satırda çalışan toplam" gibi sorular bu araçla tek sorguda çözülür. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bir sorgu, **tapu dairesinde doldurduğun bir talep formudur**. Hangi defterden başlanacağını, hangi kayıtların eleneceğini, nasıl gruplanacağını ve çıktıda ne görmek istediğini kutulara yazarsın. Formu doldurma sıran ile memurun kutuları okuma sırası aynı değildir; memurun sırasını bilmeden formu doğru dolduramazsın. Dünkü notta defterleri kurduk, bugün form dolduruyoruz.

---

## Bu Notta Ne Var

1. `SELECT`'in mantıksal işlenme sırası
2. JOIN türleri ve `LEFT JOIN` tuzağı
3. `GROUP BY`, `HAVING` ve toplama fonksiyonları
4. Alt sorgular: skaler, satır, korelasyonlu
5. `EXISTS` / `IN` / `JOIN` ve `NOT IN` tuzağı
6. CTE ve özyinelemeli CTE
7. Window function'lar
8. Sayfalama: `OFFSET/FETCH` ve `ROW_NUMBER`
9. `CASE`, `COALESCE`, `ISNULL`, `NULLIF`
10. Set operatörleri: `UNION`, `INTERSECT`, `EXCEPT`
11. `MERGE`'e temkinli bakış

Dünkü notun şemasını kullanıyoruz: `Musteriler`, `Siparisler`, `SiparisDetaylari`, `Urunler`, `Kategoriler`.

---

## 1. `SELECT`'in Mantıksal İşlenme Sırası

> **Benzetme —** Çırak ustaya "şu tahtadan bana bir raf çıkar" der. Usta önce tahtayı tezgâha koyar, sonra çürük kısımları keser, sonra parçaları boyutlarına göre ayırır, sonra hangi yığınların işe yaramadığına bakar, sonra kalanları rendeler ve en son boyuna göre dizer. Çırak siparişi tek cümlede verdi; usta altı ayrı işlem yaptı ve sırası belliydi. Sen `SELECT` ile başlayan bir cümle yazıyorsun, motor `FROM` ile başlayan altı işlem yapıyor.

**Basitçe:** Yazdığın sıra ile çalıştırılan sıra farklıdır. Motor önce veriyi getirir, sonra eler, sonra gruplar, en son ne göstereceğine karar verir.

**Teknik olarak:** **Logical query processing order (mantıksal işlenme sırası)** — SQL'in yan tümcelerinin hangi sırayla değerlendirildiğini tanımlayan kural.

| Sıra | Yan tümce | Ne yapar |
|---|---|---|
| 1 | `FROM` / `JOIN` | Kaynak tabloları birleştirir, ara sonuç üretir |
| 2 | `WHERE` | **Tek tek satırları** eler |
| 3 | `GROUP BY` | Kalan satırları öbeklere ayırır |
| 4 | `HAVING` | **Öbekleri** eler |
| 5 | `SELECT` | Sütunları seçer, hesaplar, **takma ad (alias) burada doğar** |
| 6 | `DISTINCT` | Tekrarları atar |
| 7 | `ORDER BY` | Sıralar — alias'ı kullanabilir |
| 8 | `OFFSET` / `FETCH` | Sayfalar |

Pratik sonucu şudur:

```sql
-- HATA: 'SatirToplam' geçersiz sütun adı
SELECT Adet * BirimFiyat AS SatirToplam
FROM SiparisDetaylari
WHERE SatirToplam > 1000;         -- WHERE, SELECT'ten ÖNCE çalışır: alias henüz yok

-- Doğru 1: ifadeyi tekrarla
SELECT Adet * BirimFiyat AS SatirToplam
FROM SiparisDetaylari
WHERE Adet * BirimFiyat > 1000;

-- Doğru 2: alt sorgu / CTE ile isimlendir
SELECT * FROM (
    SELECT SiparisDetayId, Adet * BirimFiyat AS SatirToplam
    FROM SiparisDetaylari
) t
WHERE t.SatirToplam > 1000;

-- ORDER BY alias'ı görür, çünkü SELECT'ten SONRA çalışır
SELECT Adet * BirimFiyat AS SatirToplam
FROM SiparisDetaylari
ORDER BY SatirToplam DESC;
```

Aynı kural `GROUP BY` için de geçerlidir: `GROUP BY` alias göremez, `HAVING` göremez, `ORDER BY` görür.

```sql
-- HATA
SELECT YEAR(SiparisTarihi) AS Yil, COUNT(*) AS Adet
FROM Siparisler
GROUP BY Yil;                     -- Yil henüz yok

-- Doğru
SELECT YEAR(SiparisTarihi) AS Yil, COUNT(*) AS Adet
FROM Siparisler
GROUP BY YEAR(SiparisTarihi)
ORDER BY Yil;                     -- burada alias serbest
```

> **Bu benzetme şurada bozulur:** Usta işlemleri gerçekten o sırayla yapar. SQL Server ise **mantıksal sıraya uyan sonucu** üretmek zorundadır ama fiziksel olarak istediği sırayı seçebilir. Sorgu iyileştirici (query optimizer) `WHERE` koşulunu `JOIN`'in içine itebilir, gruplamayı paralelleştirebilir. Mantıksal sıra sonucun **anlamını** belirler, işin yapılış biçimini değil.

> **`SELECT *` yazma.** Gereksiz sütun çekmek ağ trafiği ve bellek demektir; ayrıca covering index'i işlevsiz bırakır (dünkü notun 11. bölümü). Tabloya sonradan sütun eklendiğinde de sorgun sessizce değişir.

---

## 2. JOIN Türleri ve `LEFT JOIN` Tuzağı

> **Benzetme —** Elinde iki liste var: düğüne davet ettiklerin ve hediye getirenler. "Hem davet ettiğim hem hediye getiren" dersen `INNER JOIN`. "Davet ettiğim herkes — hediye getirmişse yanına yazarım, getirmemişse boş kalır" dersen `LEFT JOIN`. "Hediye getiren herkes, davetli olmasa bile" dersen `RIGHT JOIN`. "İki listenin toplamı, eşleşenler yan yana" dersen `FULL JOIN`. "Her davetliye her masayı dene" dersen `CROSS JOIN`.

**Basitçe:** `JOIN`, iki tabloyu ortak bir sütun üzerinden eşleştirir. Türü, eşleşmeyen satırlara ne olacağını belirler.

Örnek veriyle çalışalım:

```sql
-- Musteriler
-- MusteriId | Ad     | Soyad  | Sehir
--     1     | Ahmet  | Yılmaz | Kayseri
--     2     | Ayşe   | Demir  | Ankara
--     3     | Mehmet | Kaya   | İzmir     <- hiç siparişi yok

-- Siparisler
-- SiparisId | MusteriId | Durum
--    101    |     1     |   3
--    102    |     1     |   0
--    103    |     2     |   9
```

### `INNER JOIN` — sadece eşleşenler

```sql
SELECT m.Ad, s.SiparisId
FROM Musteriler m
INNER JOIN Siparisler s ON s.MusteriId = m.MusteriId;
```

| Ad | SiparisId |
|---|---|
| Ahmet | 101 |
| Ahmet | 102 |
| Ayşe | 103 |

Mehmet yok: siparişi olmadığı için eşleşme bulunamadı.

### `LEFT JOIN` — sol tablo korunur

```sql
SELECT m.Ad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId;
```

| Ad | SiparisId |
|---|---|
| Ahmet | 101 |
| Ahmet | 102 |
| Ayşe | 103 |
| Mehmet | `NULL` |

Mehmet geldi, sağ taraftaki sütunları `NULL` oldu. "Hiç sipariş vermemiş müşteriler" sorgusu bu davranışa dayanır:

```sql
SELECT m.MusteriId, m.Ad
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId
WHERE s.SiparisId IS NULL;        -- eşleşmeyenleri yakala (anti-join)
```

### `RIGHT JOIN` — sağ tablo korunur

`LEFT JOIN`'in aynası. Pratikte nadiren yazılır; tabloların yerini değiştirip `LEFT JOIN` yazmak daha okunaklıdır çünkü `FROM`'daki tablo "ana tablo" olarak okunur.

```sql
-- Hiç satılmamış ürünleri bul (RIGHT JOIN ile)
SELECT u.UrunId, u.Ad
FROM SiparisDetaylari sd
RIGHT JOIN Urunler u ON u.UrunId = sd.UrunId
WHERE sd.SiparisDetayId IS NULL;

-- Aynı şey, daha okunaklı hâli
SELECT u.UrunId, u.Ad
FROM Urunler u
LEFT JOIN SiparisDetaylari sd ON sd.UrunId = u.UrunId
WHERE sd.SiparisDetayId IS NULL;
```

### `FULL JOIN` — iki taraf da korunur

Eşleşmeyen satırlar her iki taraftan da gelir, karşı tarafın sütunları `NULL` olur. Veri uzlaştırma (reconciliation) senaryolarında işe yarar: iki sistemin kayıtlarını karşılaştırıp "sende var bende yok" listesi çıkarmak.

```sql
SELECT a.Kod AS SistemA, b.Kod AS SistemB
FROM SistemA_Kayitlar a
FULL JOIN SistemB_Kayitlar b ON b.Kod = a.Kod
WHERE a.Kod IS NULL OR b.Kod IS NULL;    -- sadece uyuşmayanlar
```

### `CROSS JOIN` — kartezyen çarpım

Koşul yoktur; sol tablodaki her satır sağ tablodaki her satırla eşleşir. 1.000 x 1.000 = 1 milyon satır. Kasten kullanıldığı yer: takvim/matris üretmek.

```sql
-- Her müşteri için her ayın satırını üret (satış yapılmayan ay da görünsün)
SELECT m.MusteriId, a.Ay
FROM Musteriler m
CROSS JOIN (VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12)) AS a(Ay);
```

> **Kazara `CROSS JOIN`:** `ON` koşulunu yazmayı unutursan ya da eski `FROM A, B` sözdizimini `WHERE` koşulsuz kullanırsan kartezyen çarpım elde edersin. Sonuç satır sayısının beklenenden kat kat büyük olması bunun ilk işaretidir. Bu yüzden `JOIN ... ON` sözdizimini kullan, virgüllü yazımı bırak.

### `SELF JOIN` — tabloyu kendisiyle birleştirme

Aynı tabloya iki farklı takma adla bakarsın. `Kategoriler` ağacında üst kategori adını getirmek için:

```sql
SELECT k.Ad AS Kategori, ust.Ad AS UstKategori
FROM Kategoriler k
LEFT JOIN Kategoriler ust ON ust.KategoriId = k.UstKategoriId;
```

| Kategori | UstKategori |
|---|---|
| Elektronik | `NULL` |
| Telefon | Elektronik |
| Akıllı Telefon | Telefon |

`LEFT JOIN` şart: kök kategorinin üstü yoktur, `INNER JOIN` yazarsan Elektronik kaybolur.

### En yaygın hata: `LEFT JOIN`'i `INNER JOIN`'e çevirmek

`LEFT JOIN` sağ taraftan eşleşme bulamayınca sütunları `NULL` yapar. `WHERE`'de sağ tablonun bir sütununa `NULL` olmayan bir koşul yazarsan, o `NULL` satırlar elenir — yani `LEFT JOIN` yazmış olmanın hiçbir anlamı kalmaz.

```sql
-- YANLIŞ: LEFT JOIN yazıldı ama INNER JOIN gibi davranıyor
SELECT m.Ad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId
WHERE s.Durum = 3;                -- NULL Durum bu koşulu geçemez -> Mehmet elendi

-- DOĞRU 1: koşulu JOIN'in ON kısmına taşı
SELECT m.Ad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId AND s.Durum = 3;

-- DOĞRU 2: NULL'a açıkça izin ver
SELECT m.Ad, s.SiparisId
FROM Musteriler m
LEFT JOIN Siparisler s ON s.MusteriId = m.MusteriId
WHERE s.Durum = 3 OR s.SiparisId IS NULL;
```

**Kural:** `INNER JOIN`'de `ON` ile `WHERE` arasında sonuç farkı yoktur. `LEFT JOIN`'de **vardır**: `ON`'daki koşul eşleştirmeyi daraltır, `WHERE`'deki koşul birleştirme bittikten sonra satır eler.

> **Bu benzetme şurada bozulur:** Düğün listesinde bir kişi bir kez geçer. Veritabanında `JOIN` satırları **çoğaltır**: bir müşterinin üç siparişi varsa müşteri satırı üç kez tekrarlanır. `Siparisler` ve `SiparisDetaylari`'nı aynı sorguda birleştirip `SUM(s.KargoUcreti)` alırsan kargo ücreti detay satırı sayısı kadar toplanır — **fan-out** denen klasik hata. Çözüm: toplamları ayrı ayrı hesaplayıp (CTE veya `APPLY` ile) sonra birleştirmek.

---
## 3. `GROUP BY`, `HAVING` ve Toplama Fonksiyonları

> **Benzetme —** Market kasasında gün sonu sayımı. Önce bozuk ve iade ürünleri kenara ayırırsın — bu `WHERE`. Sonra kalanları reyonlara göre öbekler halinde dizersin — bu `GROUP BY`. Her öbeğin toplamını yazarsın — bu `SUM`. En son "cirosu 500 liranın altında kalan reyonları rapora yazma" dersin — bu `HAVING`. Ayırdığın bozuk ürünler hiçbir reyonun toplamına girmedi; rapordan çıkardığın reyonlar ise toplandıktan sonra elendi.

**Basitçe:** `GROUP BY` satırları öbeklere böler, toplama fonksiyonları her öbekten tek satır üretir. `WHERE` öbeklemeden önce, `HAVING` sonra eler.

```sql
SELECT m.Sehir,
       COUNT(*)            AS SiparisAdedi,
       SUM(sd.Adet * sd.BirimFiyat) AS Ciro
FROM Siparisler s
JOIN Musteriler m        ON m.MusteriId = s.MusteriId
JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
WHERE s.Durum <> 9                      -- iptaller hiç sayıma girmesin
GROUP BY m.Sehir
HAVING SUM(sd.Adet * sd.BirimFiyat) > 100000   -- küçük şehirleri rapordan çıkar
ORDER BY Ciro DESC;
```

| Yan tümce | Ne eler | Ne zaman | Toplama fonksiyonu kullanabilir mi |
|---|---|---|---|
| `WHERE` | Tek tek satırları | Gruplamadan **önce** | Hayır |
| `HAVING` | Grupları | Gruplamadan **sonra** | Evet |

**Performans kuralı:** Elenecek satırı `HAVING`'e bırakma. `WHERE`'de elediğin satır hiç gruplanmaz; `HAVING`'de elediğin satır önce gruplanır, sonra atılır. Aynı sonucu veren iki yazımdan `WHERE` olanı her zaman daha hızlıdır.

```sql
-- Yavaş: 2026 dışındaki her şey de gruplanıyor
GROUP BY YEAR(SiparisTarihi) HAVING YEAR(SiparisTarihi) = 2026

-- Hızlı: gruplamadan önce eleniyor (üstelik SARGable)
WHERE SiparisTarihi >= '2026-01-01' AND SiparisTarihi < '2027-01-01'
GROUP BY YEAR(SiparisTarihi)
```

**Kural:** `SELECT` listesindeki her sütun ya `GROUP BY`'da olmalı ya da bir toplama fonksiyonunun içinde olmalı. Aksi hâlde SQL Server hata verir (MySQL bazı modlarda sessizce rastgele bir değer döndürür — güvenme).

### Toplama fonksiyonları ve `NULL`

```sql
SELECT COUNT(*)            AS TumSatirlar,     -- 3
       COUNT(Sehir)        AS SehriOlanlar,    -- 2   (NULL'ları saymaz)
       COUNT(DISTINCT Sehir) AS FarkliSehir,   -- 2
       AVG(Puan)           AS OrtalamaPuan,    -- NULL'lar paydaya da girmez
       SUM(Puan)           AS ToplamPuan,
       MIN(KayitTarihi)    AS IlkKayit,
       MAX(KayitTarihi)    AS SonKayit
FROM Musteriler;
```

| Fonksiyon | `NULL` davranışı |
|---|---|
| `COUNT(*)` | Satır sayar, `NULL` umursamaz |
| `COUNT(kolon)` | `NULL` olmayan değerleri sayar |
| `COUNT(DISTINCT kolon)` | Benzersiz, `NULL` olmayan değerleri sayar |
| `SUM` / `AVG` / `MIN` / `MAX` | `NULL`'ları yok sayar; hiç satır yoksa `NULL` döner |

`AVG` farkı en çok kafa karıştıranıdır: 5 satırın 2'si `NULL` ise `AVG` toplamı **3'e** böler, 5'e değil. `NULL`'u sıfır saymak istiyorsan açıkça yazarsın:

```sql
SELECT AVG(Puan)             FROM Degerlendirmeler;   -- NULL'lar hesaba girmez
SELECT AVG(ISNULL(Puan, 0))  FROM Degerlendirmeler;   -- NULL'lar 0 sayılır
```

Ayrıca hiç satır dönmediğinde `COUNT` **0** döner ama `SUM` **`NULL`** döner. Uygulama tarafında `null` gelmesin istiyorsan sararsın:

```sql
SELECT ISNULL(SUM(Adet * BirimFiyat), 0) AS Ciro
FROM SiparisDetaylari WHERE SiparisId = 99999;   -- 0 döner, NULL değil
```

`STRING_AGG` ile grubun değerlerini tek metinde toplayabilirsin (SQL Server 2017+):

```sql
SELECT s.SiparisId,
       STRING_AGG(u.Ad, ', ') WITHIN GROUP (ORDER BY u.Ad) AS Urunler
FROM SiparisDetaylari sd
JOIN Urunler u ON u.UrunId = sd.UrunId
JOIN Siparisler s ON s.SiparisId = sd.SiparisId
GROUP BY s.SiparisId;
```

> **Bu benzetme şurada bozulur:** Markette bir ürün tek reyona girer. SQL'de ise bir satır **birden fazla gruplama düzeyinde** raporlanabilir: `GROUP BY GROUPING SETS`, `ROLLUP` ve `CUBE` aynı sorguda hem şehir bazında hem genel toplamı üretir. Kasa örneğinde bu, aynı ürünü hem reyon toplamına hem mağaza toplamına yazmak demektir.

```sql
SELECT m.Sehir, COUNT(*) AS Adet
FROM Siparisler s JOIN Musteriler m ON m.MusteriId = s.MusteriId
GROUP BY ROLLUP(m.Sehir);        -- her şehir + en altta genel toplam satırı (Sehir = NULL)
```

---

## 4. Alt Sorgular: Skaler, Satır, Korelasyonlu

> **Benzetme —** Bankada kredi başvurusu yapıyorsun. Memur formu doldururken "bir dakika" deyip yan masadan senin ortalama maaşını sorar, dönüp forma yazar. Bu bir alt sorgudur: ana işin ortasında yapılan, tek bir cevabı olan küçük bir soruşturma. Eğer memur her başvuran için ayrı ayrı yan masaya gidiyorsa iş uzar — korelasyonlu alt sorgunun maliyeti tam olarak budur.

**Basitçe:** Alt sorgu, bir sorgunun içine yerleştirilmiş başka bir sorgudur. Dış sorguya bir değer, bir sütun ya da bir tablo verir.

**Teknik olarak:** **Subquery (alt sorgu)** — Parantez içinde yazılan, başka bir sorgunun parçası olan `SELECT` ifadesi. Üç yerde bulunur: `SELECT` listesinde, `FROM`'da (türetilmiş tablo) ve `WHERE`/`HAVING`'de.

### Skaler alt sorgu — tek değer döndürür

```sql
-- Ortalamanın üstünde sipariş veren müşteriler
SELECT m.MusteriId, m.Ad
FROM Musteriler m
WHERE (SELECT COUNT(*) FROM Siparisler s WHERE s.MusteriId = m.MusteriId)
      > (SELECT COUNT(*) * 1.0 / COUNT(DISTINCT MusteriId) FROM Siparisler);
```

Skaler alt sorgu birden fazla satır döndürürse hata alırsın: *"Subquery returned more than 1 value."* Bu hata genelde alt sorgunun filtresinin eksik olduğunu gösterir.

### Satır/sütun listesi döndüren alt sorgu

```sql
-- Kayseri'deki müşterilerin siparişleri
SELECT * FROM Siparisler
WHERE MusteriId IN (SELECT MusteriId FROM Musteriler WHERE Sehir = 'Kayseri');
```

### Türetilmiş tablo (`FROM` içinde alt sorgu)

```sql
SELECT t.MusteriId, t.SiparisAdedi
FROM (
    SELECT MusteriId, COUNT(*) AS SiparisAdedi
    FROM Siparisler
    WHERE Durum <> 9
    GROUP BY MusteriId
) t
WHERE t.SiparisAdedi >= 5;
```

Türetilmiş tabloya **takma ad vermek zorunludur** (`t`), yoksa sözdizimi hatası alırsın.

### Korelasyonlu alt sorgu — dış sorguya bağımlı

Alt sorgu, dış sorgunun bir sütununa referans veriyorsa **correlated (korelasyonlu)** olur. Her dış satır için yeniden çalışır.

```sql
-- Her müşterinin son sipariş tarihi
SELECT m.MusteriId, m.Ad,
       (SELECT MAX(s.SiparisTarihi)
        FROM Siparisler s
        WHERE s.MusteriId = m.MusteriId) AS SonSiparis   -- m.MusteriId: korelasyon
FROM Musteriler m;
```

Okunaklıdır ama 100.000 müşteri varsa alt sorgu 100.000 kez çalışabilir. Modern SQL Server iyileştiricisi çoğu korelasyonlu alt sorguyu `JOIN`'e çevirebilir, ama her zaman değil. Büyük veri setinde `JOIN` + `GROUP BY` ya da `OUTER APPLY` daha güvenilirdir:

```sql
-- APPLY: her dış satır için alt sorguyu çalıştırır, "her gruptan ilk N" için ideal
SELECT m.MusteriId, m.Ad, son.SiparisId, son.SiparisTarihi
FROM Musteriler m
OUTER APPLY (
    SELECT TOP (1) s.SiparisId, s.SiparisTarihi
    FROM Siparisler s
    WHERE s.MusteriId = m.MusteriId
    ORDER BY s.SiparisTarihi DESC
) son;
```

`CROSS APPLY` eşleşme bulamayan dış satırı atar (inner join gibi), `OUTER APPLY` korur (left join gibi).

> **Bu benzetme şurada bozulur:** Memur yan masadan cevabı bir kez alıp not eder. SQL'de korelasyonlu alt sorgunun sonucu **her satır için yeniden** hesaplanır ve not tutulmaz. Fakat tersi de olur: iyileştirici, korelasyonsuz bir alt sorguyu bir kez hesaplayıp sonucu tekrar kullanır. Yani maliyet "kaç kez yazdığına" değil, **alt sorgunun dışarıya bağımlı olup olmadığına** bağlıdır.

---

## 5. `EXISTS` / `IN` / `JOIN` ve `NOT IN` Tuzağı

> **Benzetme —** Nöbetçi eczane listesinde bir eczanenin olup olmadığını üç türlü kontrol edebilirsin. Listeyi baştan sona okuyup adları toplarsın, sonra karşılaştırırsın (`IN`). Ya da listeyi okurken aradığını görür görmez durur, "var" dersin (`EXISTS`). Ya da iki listeyi yan yana koyup eşleştirirsin (`JOIN`). Üçü de doğru cevabı verir; ikincisi genelde en az işi yapar.

**Basitçe:** "Var mı yok mu" sorusunun üç yazımı vardır. `EXISTS` ilk eşleşmede durur, `IN` değer listesi üretir, `JOIN` satırları çoğaltabilir.

```sql
-- EXISTS: en az bir sipariş vermiş müşteriler
SELECT m.MusteriId, m.Ad
FROM Musteriler m
WHERE EXISTS (SELECT 1 FROM Siparisler s WHERE s.MusteriId = m.MusteriId);

-- IN: aynı sonuç
SELECT m.MusteriId, m.Ad
FROM Musteriler m
WHERE m.MusteriId IN (SELECT MusteriId FROM Siparisler);

-- JOIN: DISTINCT gerekir, yoksa müşteri sipariş sayısı kadar tekrar eder
SELECT DISTINCT m.MusteriId, m.Ad
FROM Musteriler m
JOIN Siparisler s ON s.MusteriId = m.MusteriId;
```

| Yazım | Davranış | Ne zaman |
|---|---|---|
| `EXISTS` | İlk eşleşmede durur (short-circuit) | Varlık kontrolü — varsayılan tercihin |
| `IN` | Değer listesiyle karşılaştırır | Kısa ve sabit liste: `IN (1,2,3)` |
| `JOIN` | Satırları çoğaltır | Sağ tablodan **sütun da** lazımsa |
| `NOT EXISTS` | Yokluk kontrolü — `NULL` güvenli | Anti-join için doğru seçim |
| `NOT IN` | Yokluk kontrolü — **`NULL` güvenli değil** | Alt sorguyla kullanma |

`EXISTS` içinde `SELECT 1` yazılır; `SELECT *` de yazılabilir, fark etmez — SQL Server sütunları hiç okumaz, sadece satırın varlığına bakar.

**Performans:** SQL Server iyileştiricisi `EXISTS` ve `IN`'i çoğu durumda aynı plana çevirir (semi join). Gerçek fark `JOIN` ile olandır: `JOIN` + `DISTINCT` yazarsan motor önce bütün eşleşmeleri üretip sonra tekrarları atar. `EXISTS` bu işi hiç yapmaz.

### `NOT IN` + `NULL` tuzağı

Bu, SQL'de sessizce yanlış sonuç veren en klasik hatadır. Dünkü notun üç değerli mantığı burada karşına çıkıyor.

```sql
-- Alt sorgunun döndürdüğü sütunda BİR TANE bile NULL varsa,
-- bu sorgu HİÇBİR satır döndürmez
SELECT * FROM Urunler
WHERE UrunId NOT IN (SELECT UrunId FROM SiparisDetaylari);
```

Sebebi: `x NOT IN (1, 2, NULL)` ifadesi `x <> 1 AND x <> 2 AND x <> NULL` demektir. Son parça `UNKNOWN`'dır; `TRUE AND UNKNOWN` = `UNKNOWN` olur ve `WHERE` yalnızca `TRUE` satırları geçirir. Hata mesajı yoktur, sadece boş sonuç vardır.

```sql
-- DOĞRU 1: NOT EXISTS (NULL güvenli, tercih edilen)
SELECT u.* FROM Urunler u
WHERE NOT EXISTS (SELECT 1 FROM SiparisDetaylari sd WHERE sd.UrunId = u.UrunId);

-- DOĞRU 2: LEFT JOIN + IS NULL (anti-join kalıbı)
SELECT u.* FROM Urunler u
LEFT JOIN SiparisDetaylari sd ON sd.UrunId = u.UrunId
WHERE sd.SiparisDetayId IS NULL;

-- DOĞRU 3: NOT IN kullanacaksan NULL'ları açıkça ele
SELECT * FROM Urunler
WHERE UrunId NOT IN (SELECT UrunId FROM SiparisDetaylari WHERE UrunId IS NOT NULL);
```

> **Kural:** Alt sorguyla birlikte `NOT IN` yazma. `NOT EXISTS` yaz. `NOT IN`'i yalnızca elle yazdığın sabit listelerde (`NOT IN (1, 2, 3)`) kullan.

> **Bu benzetme şurada bozulur:** Eczane listesinde "belirsiz" bir kayıt olmaz; ya vardır ya yoktur. Veritabanında `NULL` üçüncü bir durumdur ve `NOT IN`'in mantığını içeriden bozar. Üstelik `IN` tarafında aynı sorun **yoktur**: `x IN (1, 2, NULL)` aradığın değer 1 veya 2 ise `TRUE` döner. Tuzak sadece olumsuz kontrolde kuruluyor.

---
## 6. CTE ve Özyinelemeli CTE

> **Benzetme —** Uzun bir yemek tarifinde "önceden hazırladığımız sos" diye bir ara ürün vardır. Sos tarifin başında bir kez anlatılır, sonra "sosu ekle" denip geçilir. Tarifin gövdesi kısalır ve okunaklılaşır. CTE tam olarak budur: sorgunun başında bir ara sonucu isimlendirirsin, gövdede adıyla kullanırsın.

**Basitçe:** CTE, sorgunun başında `WITH` ile tanımladığın, adı olan geçici bir sonuç kümesidir. İç içe alt sorguların okunmaz hâle geldiği yerde işe yarar.

**Teknik olarak:** **CTE (Common Table Expression / ortak tablo ifadesi)** — `WITH ad AS (...)` biçiminde tanımlanan, yalnızca kendisinden hemen sonraki ifadede geçerli olan adlandırılmış sorgu.

```sql
WITH MusteriCiro AS (
    SELECT s.MusteriId,
           SUM(sd.Adet * sd.BirimFiyat) AS Ciro
    FROM Siparisler s
    JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
    WHERE s.Durum <> 9
    GROUP BY s.MusteriId
)
SELECT m.Ad, m.Soyad, c.Ciro
FROM MusteriCiro c
JOIN Musteriler m ON m.MusteriId = c.MusteriId
WHERE c.Ciro > 50000
ORDER BY c.Ciro DESC;
```

Birden fazla CTE'yi virgülle zincirleyebilirsin; sonraki öncekini kullanabilir:

```sql
WITH Gecerli AS (
    SELECT * FROM Siparisler WHERE Durum <> 9
),
AylikCiro AS (
    SELECT YEAR(SiparisTarihi) AS Yil, MONTH(SiparisTarihi) AS Ay,
           SUM(sd.Adet * sd.BirimFiyat) AS Ciro
    FROM Gecerli g
    JOIN SiparisDetaylari sd ON sd.SiparisId = g.SiparisId
    GROUP BY YEAR(SiparisTarihi), MONTH(SiparisTarihi)
)
SELECT * FROM AylikCiro ORDER BY Yil, Ay;
```

| | CTE | Türetilmiş tablo | Geçici tablo (`#tmp`) |
|---|---|---|---|
| Okunaklılık | Yüksek | Düşük (iç içe) | Orta |
| Birden çok yerde kullanım | Evet (ama her seferinde yeniden hesaplanır) | Hayır | Evet |
| Sonuç saklanır mı | **Hayır** | Hayır | Evet (tempdb) |
| Özyineleme | **Evet** | Hayır | Hayır |

> **Yanılgı:** CTE sonucu bir yere yazmaz; sadece okunaklılık sağlar. Aynı CTE'yi sorguda üç kez kullanırsan motor onu üç kez hesaplayabilir. Gerçekten bir kez hesaplanıp saklansın istiyorsan `#gecici` tablo veya tablo değişkeni kullanırsın.

### Özyinelemeli CTE — ağaç yapıları

Kendi kendine referans veren CTE'dir. İki parçası vardır: **anchor** (başlangıç satırları) ve **recursive member** (bir önceki adımın sonucunu kullanan parça). İkisi `UNION ALL` ile birleşir.

```sql
-- Kategori ağacını kökten yapraklara gez (dünkü notun Kategoriler tablosu)
WITH KategoriAgaci AS (
    -- Anchor: kök kategoriler
    SELECT KategoriId, Ad, UstKategoriId,
           0 AS Seviye,
           CAST(Ad AS NVARCHAR(1000)) AS Yol
    FROM Kategoriler
    WHERE UstKategoriId IS NULL

    UNION ALL

    -- Recursive: bir üst adımda bulunanların çocukları
    SELECT k.KategoriId, k.Ad, k.UstKategoriId,
           ka.Seviye + 1,
           CAST(ka.Yol + N' > ' + k.Ad AS NVARCHAR(1000))
    FROM Kategoriler k
    JOIN KategoriAgaci ka ON ka.KategoriId = k.UstKategoriId
)
SELECT Seviye, Yol
FROM KategoriAgaci
ORDER BY Yol
OPTION (MAXRECURSION 100);
```

| Seviye | Yol |
|---|---|
| 0 | Elektronik |
| 1 | Elektronik > Telefon |
| 2 | Elektronik > Telefon > Akıllı Telefon |

Aynı yapıyla ters yönde de gidilir: bir kategoriden başlayıp köke kadar yukarı çıkmak için anchor'a o kategoriyi koyar, recursive parçada `JOIN`'i ters çevirirsin.

> **`MAXRECURSION`:** Varsayılan sınır 100 adımdır; aşılırsa sorgu hata verir. `OPTION (MAXRECURSION 0)` sınırı kaldırır — ama veride bir döngü varsa (A'nın üstü B, B'nin üstü A) sorgu sonsuza kadar çalışır. Sınırı kaldırmadan önce `CHECK` kısıtı veya `Seviye < 20` koşuluyla kendini koru.

> **Bu benzetme şurada bozulur:** Yemek tarifindeki sos bir kez yapılır ve kenarda bekler. Özyinelemeli CTE'de ise her adım **bir önceki adımın çıktısını** girdi olarak alır; kenarda bekleyen bir şey yoktur, zincir ilerler. Tarif benzetmesi normal CTE için doğru, özyinelemeli CTE için yanlıştır — oradaki doğru benzetme, kutuyu açınca içinden başka bir kutu çıkan hediye paketidir.

---

## 7. Window Function'lar

> **Benzetme —** Sınıfta not listesi asılmış. Her öğrenci kendi satırını görüyor ama yanında bir de "sınıf ortalaması", "sınıftaki sıran" ve "senden bir önceki öğrencinin notu" yazıyor. Kimse listeden silinmedi; sadece her satıra, o satırın ait olduğu **grubun bilgisi** eklendi. `GROUP BY` sınıfı tek satıra indirir; window function öğrencileri korur, yanlarına grup bilgisini yazar.

**Basitçe:** Window function, satırları birleştirmeden her satırın yanına grup hesabı ekler. Satır sayısı değişmez.

**Teknik olarak:** **Window function (pencere fonksiyonu)** — `OVER (...)` yan tümcesiyle tanımlanan, her satır için bir "pencere" (ilgili satır kümesi) üzerinde hesap yapan fonksiyon. `PARTITION BY` pencereyi böler, `ORDER BY` pencere içindeki sırayı belirler.

```sql
SELECT s.SiparisId, s.MusteriId, s.SiparisTarihi,
       COUNT(*)  OVER (PARTITION BY s.MusteriId)               AS MusteriSiparisAdedi,
       ROW_NUMBER() OVER (PARTITION BY s.MusteriId
                          ORDER BY s.SiparisTarihi DESC)        AS SonluktanSira
FROM Siparisler s;
```

`GROUP BY` ile farkı tek cümlede: `GROUP BY` satırları **yok eder**, `OVER` satırları **korur**.

### Sıralama fonksiyonları

```sql
SELECT u.Ad, u.KategoriId, u.BirimFiyat,
       ROW_NUMBER() OVER (PARTITION BY u.KategoriId ORDER BY u.BirimFiyat DESC) AS Sira,
       RANK()       OVER (PARTITION BY u.KategoriId ORDER BY u.BirimFiyat DESC) AS Rank_,
       DENSE_RANK() OVER (PARTITION BY u.KategoriId ORDER BY u.BirimFiyat DESC) AS DenseRank_,
       NTILE(4)     OVER (PARTITION BY u.KategoriId ORDER BY u.BirimFiyat DESC) AS Ceyrek
FROM Urunler u;
```

Eşit değerlerde farkları şöyledir (fiyatlar 900, 800, 800, 700):

| Fiyat | `ROW_NUMBER` | `RANK` | `DENSE_RANK` |
|---|---|---|---|
| 900 | 1 | 1 | 1 |
| 800 | 2 | 2 | 2 |
| 800 | 3 | **2** | **2** |
| 700 | 4 | **4** | **3** |

- `ROW_NUMBER`: her satıra ayrı numara, eşitlik umursanmaz. Sayfalama ve tekil seçim için.
- `RANK`: eşitler aynı sırayı alır, sonra **atlama** olur (2, 2, 4).
- `DENSE_RANK`: eşitler aynı sırayı alır, atlama **olmaz** (2, 2, 3).
- `NTILE(n)`: satırları n eşit kovaya böler. Çeyreklik/yüzdelik dilim raporları için.

### `LAG` ve `LEAD` — önceki ve sonraki satır

```sql
-- Aylık ciro ve bir önceki aya göre değişim
WITH Aylik AS (
    SELECT YEAR(s.SiparisTarihi) AS Yil, MONTH(s.SiparisTarihi) AS Ay,
           SUM(sd.Adet * sd.BirimFiyat) AS Ciro
    FROM Siparisler s
    JOIN SiparisDetaylari sd ON sd.SiparisId = s.SiparisId
    GROUP BY YEAR(s.SiparisTarihi), MONTH(s.SiparisTarihi)
)
SELECT Yil, Ay, Ciro,
       LAG(Ciro)  OVER (ORDER BY Yil, Ay)            AS OncekiAy,
       LEAD(Ciro) OVER (ORDER BY Yil, Ay)            AS SonrakiAy,
       Ciro - LAG(Ciro, 1, 0) OVER (ORDER BY Yil, Ay) AS Degisim
FROM Aylik
ORDER BY Yil, Ay;
```

`LAG(Ciro, 1, 0)`: bir satır geriye bak, yoksa `0` kullan. Üçüncü parametreyi vermezsen ilk satırda `NULL` gelir.

### `SUM() OVER` — çalışan toplam

```sql
SELECT s.MusteriId, s.SiparisId, s.SiparisTarihi, t.Tutar,
       SUM(t.Tutar) OVER (PARTITION BY s.MusteriId
                          ORDER BY s.SiparisTarihi
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS KumulatifTutar,
       SUM(t.Tutar) OVER (PARTITION BY s.MusteriId)                          AS MusteriToplami,
       AVG(t.Tutar) OVER (PARTITION BY s.MusteriId)                          AS MusteriOrtalamasi
FROM Siparisler s
CROSS APPLY (SELECT SUM(Adet * BirimFiyat) AS Tutar
             FROM SiparisDetaylari WHERE SiparisId = s.SiparisId) t;
```

`OVER` içinde `ORDER BY` varsa varsayılan çerçeve `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`'dur — yani çalışan toplam üretir. `ORDER BY` yoksa pencerenin tamamı kullanılır, grup toplamı gelir. Eşit değerli satırlarda `RANGE` ile `ROWS` farklı sonuç verir; çalışan toplamda **`ROWS` yaz**, açıkça ve güvenlidir.

### "Her gruptan ilk N" problemi

Klasik sorudur: her kategorinin en pahalı 3 ürünü. Window function ile tek sorguda çözülür.

```sql
WITH Siralanmis AS (
    SELECT u.UrunId, u.Ad, u.KategoriId, u.BirimFiyat,
           ROW_NUMBER() OVER (PARTITION BY u.KategoriId
                              ORDER BY u.BirimFiyat DESC, u.UrunId) AS Sira
    FROM Urunler u
    WHERE u.Aktif = 1
)
SELECT KategoriId, Ad, BirimFiyat
FROM Siralanmis
WHERE Sira <= 3
ORDER BY KategoriId, Sira;
```

`ORDER BY` içine `u.UrunId` eklemek tesadüf değil: eşit fiyatlı ürünlerde sıralama belirsiz kalmasın diye **tie-breaker** koyarsın. Yoksa aynı sorgu iki çalıştırmada farklı 3 ürün döndürebilir.

> **`WHERE` içinde window function kullanılamaz.** Mantıksal sıra hatırla: `WHERE` (2), `SELECT` (5). Window function `SELECT` aşamasında hesaplanır, `WHERE` çalıştığında henüz yoktur. Bu yüzden CTE veya türetilmiş tabloya sarıp dıştan filtrelersin.

> **Bu benzetme şurada bozulur:** Not listesinde "sınıf ortalaması" herkes için aynıdır. Window function'da pencere satır satır **değişebilir**: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` yazarsan her satırın penceresi kendi konumuna göre kayar (hareketli ortalama). Yani "grup bilgisi" sabit bir kutu değil, satırla birlikte hareket eden bir çerçevedir.

---
## 8. Sayfalama: `OFFSET/FETCH` ve `ROW_NUMBER`

> **Benzetme —** Kütüphanede kitap listesi yirmişerli sayfalar hâlinde basılmış. Üçüncü sayfayı istiyorsan görevli ilk kırk kitabı sayıp atlar, sonraki yirmiyi verir. Liste alfabetik olduğu sürece bu çalışır. Ama liste karışık dizilmişse, aynı "üçüncü sayfa" isteği her seferinde başka kitaplar getirir. Sayfalamanın çalışması için **kesin bir sıra** şarttır.

**Basitçe:** Sayfalama, sonucu parçalara bölüp istenen parçayı döndürmektir. `ORDER BY` olmadan güvenilir değildir.

```sql
-- SQL Server 2012+ standart yazım
SELECT s.SiparisId, s.SiparisTarihi, s.Durum
FROM Siparisler s
WHERE s.MusteriId = @musteriId
ORDER BY s.SiparisTarihi DESC, s.SiparisId DESC   -- tie-breaker şart
OFFSET  (@sayfa - 1) * @sayfaBoyutu ROWS
FETCH NEXT @sayfaBoyutu ROWS ONLY;
```

`OFFSET/FETCH` yalnızca `ORDER BY` ile birlikte kullanılabilir — sözdizimi buna zorlar. `ROW_NUMBER` ile aynı işi yapan eski kalıp da hâlâ karşına çıkar:

```sql
WITH Sirali AS (
    SELECT SiparisId, SiparisTarihi,
           ROW_NUMBER() OVER (ORDER BY SiparisTarihi DESC, SiparisId DESC) AS rn
    FROM Siparisler WHERE MusteriId = @musteriId
)
SELECT SiparisId, SiparisTarihi FROM Sirali
WHERE rn BETWEEN (@sayfa - 1) * @sayfaBoyutu + 1 AND @sayfa * @sayfaBoyutu;
```

Toplam sayfa sayısını aynı sorguda almak için `COUNT(*) OVER ()` kullanılır — ikinci bir sorgu atmaktan kurtarır:

```sql
SELECT s.SiparisId, s.SiparisTarihi,
       COUNT(*) OVER () AS ToplamKayit        -- her satırda aynı değer
FROM Siparisler s
WHERE s.MusteriId = @musteriId
ORDER BY s.SiparisTarihi DESC, s.SiparisId DESC
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;
```

> **`OFFSET`'in maliyeti:** `OFFSET 100000` yazarsan SQL Server ilk 100.000 satırı gerçekten üretip atar. Derin sayfalarda (sonsuz kaydırmalı listeler, API'ler) bu ölümcüldür. Çözüm **keyset pagination**: "şu anahtardan sonrasını getir".

```sql
-- Keyset: son görülen satırın anahtarını parametre olarak alır, OFFSET yok
SELECT TOP (20) SiparisId, SiparisTarihi
FROM Siparisler
WHERE MusteriId = @musteriId
  AND (SiparisTarihi < @sonTarih
       OR (SiparisTarihi = @sonTarih AND SiparisId < @sonId))
ORDER BY SiparisTarihi DESC, SiparisId DESC;
```

Keyset sayfalama index'i doğrudan kullanır ve sayfa numarası büyüdükçe yavaşlamaz. Bedeli: rastgele bir sayfaya atlayamazsın, sadece ileri/geri gidebilirsin.

```csharp
// EF Core karşılığı: Skip/Take -> OFFSET/FETCH
var sayfa = await _db.Siparisler
    .Where(s => s.MusteriId == musteriId)
    .OrderByDescending(s => s.SiparisTarihi).ThenByDescending(s => s.SiparisId)
    .Skip((sayfaNo - 1) * boyut)
    .Take(boyut)
    .ToListAsync();
```

> **Bu benzetme şurada bozulur:** Kütüphanedeki liste sabittir; sen sayfa çevirirken kimse kitap eklemez. Veritabanında ise sen 2. sayfayı okurken başkası yeni kayıt ekleyebilir. Sonuç: aynı kaydı iki sayfada görürsün ya da bir kaydı hiç görmezsin. `OFFSET` bu soruna açıktır, keyset sayfalama değildir.

---

## 9. `CASE`, `COALESCE`, `ISNULL`, `NULLIF`

> **Benzetme —** Terzi ölçü alırken "beden 38'in altındaysa S, 42'ye kadar M, üstü L" der. Tek bir kurallar listesi, yukarıdan aşağı okunur, ilk uyan kural kazanır. `CASE` bu listedir. `COALESCE` ise "asıl telefonu yoksa iş telefonunu, o da yoksa ev telefonunu yaz" demektir: ilk dolu olanı alır.

**Basitçe:** `CASE` koşula göre değer üretir. Diğer üçü `NULL` ile başa çıkmanın kısa yollarıdır.

```sql
-- CASE: aranan (searched) biçim — en esnek
SELECT SiparisId,
       CASE
           WHEN Durum = 0 THEN N'Yeni'
           WHEN Durum = 1 THEN N'Hazırlanıyor'
           WHEN Durum = 2 THEN N'Kargoda'
           WHEN Durum = 3 THEN N'Teslim Edildi'
           WHEN Durum = 9 THEN N'İptal'
           ELSE N'Bilinmiyor'
       END AS DurumAdi
FROM Siparisler;

-- CASE: basit (simple) biçim — yalnızca eşitlik kontrolü yapar
SELECT CASE Durum WHEN 0 THEN N'Yeni' WHEN 9 THEN N'İptal' ELSE N'Diğer' END
FROM Siparisler;
```

`ELSE` yazmazsan eşleşmeyen satırlarda `NULL` döner. `CASE` toplama fonksiyonlarının içinde de kullanılır — koşullu sayım kalıbı:

```sql
SELECT m.Sehir,
       COUNT(*)                                                AS ToplamSiparis,
       SUM(CASE WHEN s.Durum = 3 THEN 1 ELSE 0 END)            AS TeslimEdilen,
       SUM(CASE WHEN s.Durum = 9 THEN 1 ELSE 0 END)            AS Iptal,
       COUNT(CASE WHEN s.Durum = 9 THEN 1 END)                 AS Iptal2   -- aynı sonuç
FROM Siparisler s JOIN Musteriler m ON m.MusteriId = s.MusteriId
GROUP BY m.Sehir;
```

| Fonksiyon | Ne yapar | Not |
|---|---|---|
| `ISNULL(a, b)` | `a` `NULL` ise `b` | SQL Server'a özel, **iki** argüman |
| `COALESCE(a, b, c, ...)` | İlk `NULL` olmayanı | ANSI standardı, N argüman |
| `NULLIF(a, b)` | `a = b` ise `NULL`, değilse `a` | Sıfıra bölmeyi engellemek için |
| `IIF(kosul, a, b)` | Kısa `CASE` | SQL Server 2012+ |

```sql
SELECT ISNULL(Sehir, N'Belirtilmemiş')            FROM Musteriler;
SELECT COALESCE(CepTelefon, IsTelefon, EvTelefon) FROM Musteriler;

-- NULLIF ile sıfıra bölme koruması
SELECT ToplamTutar / NULLIF(SiparisAdedi, 0) AS OrtalamaSepet FROM Ozet;
-- SiparisAdedi 0 ise payda NULL olur, sonuç NULL döner; hata almazsın

SELECT IIF(StokAdedi > 0, N'Stokta', N'Tükendi') FROM Urunler;
```

**`ISNULL` ile `COALESCE` arasındaki iki ince fark:**

1. `ISNULL` dönüş tipini **ilk argümana** göre belirler ve gerekirse kısaltır. `COALESCE` tüm argümanların en geniş tipini seçer.
2. `COALESCE` aslında bir `CASE` ifadesine açılır; alt sorgu içeren argümanlar **iki kez** değerlendirilebilir.

```sql
SELECT ISNULL(CAST(NULL AS NVARCHAR(2)), N'Kayseri');    -- 'Ka'  -- kesildi!
SELECT COALESCE(CAST(NULL AS NVARCHAR(2)), N'Kayseri');  -- 'Kayseri'
```

> **Bu benzetme şurada bozulur:** Terzi kuralları tek tek okur ve ilk uyanı seçer — `CASE` de öyle. Ama `CASE`'in `WHEN` sırası **yalnızca mantıksal** garantilidir; SQL Server toplama fonksiyonları ve sabit ifadelerde değerlendirme sırasını değiştirebilir. Bu yüzden `CASE WHEN Payda <> 0 THEN Pay / Payda ELSE 0 END` yazımı sıfıra bölmeyi her zaman engellemeyebilir. Garantili yol `NULLIF`'tir.

---

## 10. Set Operatörleri: `UNION`, `INTERSECT`, `EXCEPT`

> **Benzetme —** İki mahalle muhtarının seçmen listesi var. İkisini birleştirip aynı kişiyi bir kez yazmak `UNION`, tekrarları umursamadan alt alta eklemek `UNION ALL`. İki listede de olanlar `INTERSECT`, birincide olup ikincide olmayanlar `EXCEPT`.

**Basitçe:** Set operatörleri iki sorgu sonucunu alt alta birleştirir. `JOIN` yan yana koyar, bunlar alt alta.

Üç kuralı vardır: sütun sayıları eşit olmalı, sütun tipleri uyumlu olmalı, sütun adları ilk sorgudan alınır. `ORDER BY` yalnızca en sona bir kez yazılır.

```sql
-- UNION: tekrarları atar (arka planda sıralama/hash yapar)
SELECT Eposta FROM Musteriler
UNION
SELECT Eposta FROM BultenAboneleri;

-- UNION ALL: tekrarları atmaz, çok daha hızlı
SELECT SiparisId, 'Aktif' AS Kaynak FROM Siparisler
UNION ALL
SELECT SiparisId, 'Arsiv' FROM Siparisler_Arsiv
ORDER BY SiparisId;

-- INTERSECT: iki listede de olanlar
SELECT Eposta FROM Musteriler
INTERSECT
SELECT Eposta FROM BultenAboneleri;

-- EXCEPT: ilkinde olup ikincisinde olmayanlar
SELECT Eposta FROM Musteriler
EXCEPT
SELECT Eposta FROM BultenAboneleri;
```

| Operatör | Tekrarları atar mı | Karşılığı |
|---|---|---|
| `UNION` | Evet | `DISTINCT` + birleştirme |
| `UNION ALL` | Hayır | Düz birleştirme — **varsayılan tercihin** |
| `INTERSECT` | Evet | Kesişim (semi join benzeri) |
| `EXCEPT` | Evet | Fark (anti join benzeri) |

> **Performans:** Tekrar olmayacağını biliyorsan `UNION ALL` yaz. `UNION` her seferinde tekrar ayıklaması yapar ve bu işlem sıralama ya da hash maliyeti getirir. İki ayrı tablodan gelen kayıtlarda tekrar zaten mümkün değilse `UNION` yazmak bedava yavaşlıktır.

`INTERSECT` ve `EXCEPT`, `NULL`'ları **eşit sayar** — `IN`/`NOT IN`'in aksine. İki tabloyu karşılaştırıp farkı bulmak için `EXCEPT` en güvenli araçtır:

```sql
-- İki tablo birebir aynı mı? İki yönlü EXCEPT boş dönerse aynıdır.
SELECT * FROM TabloA EXCEPT SELECT * FROM TabloB
UNION ALL
SELECT * FROM TabloB EXCEPT SELECT * FROM TabloA;
```

> **Bu benzetme şurada bozulur:** Muhtarın listesinde "kimliği belirsiz" bir kayıt olmaz. `EXCEPT` ve `INTERSECT` ise `NULL = NULL` karşılaştırmasını **doğru** sayar; yani üç değerli mantık burada askıya alınır. Aynı karşılaştırmayı `WHERE a.Kod = b.Kod` ile yaparsan `NULL`'lar eşleşmez. Aynı veri, iki farklı sonuç.

---

## 11. `MERGE`'e Temkinli Bakış

> **Benzetme —** Depoya gelen yeni sayım listesini eldeki stok defterine işlemek gibi. Listede olup defterde olmayanı eklersin, ikisinde de olanı güncellersin, defterde olup listede olmayanı silersin. Tek seferde üç iş. Pratikte tecrübeli depo sorumlusu bunu üç ayrı geçişte yapar — çünkü tek geçişte yapılan hatayı fark etmek çok zordur.

**Basitçe:** `MERGE`, tek ifadede ekleme, güncelleme ve silme yapar. Güçlüdür ama bilinen hataları vardır; çoğu projede ayrı `UPDATE` ve `INSERT` tercih edilir.

```sql
MERGE INTO Urunler AS hedef
USING @GelenUrunler AS kaynak
    ON hedef.UrunId = kaynak.UrunId
WHEN MATCHED AND (hedef.BirimFiyat <> kaynak.BirimFiyat OR hedef.StokAdedi <> kaynak.StokAdedi)
    THEN UPDATE SET hedef.BirimFiyat = kaynak.BirimFiyat,
                    hedef.StokAdedi  = kaynak.StokAdedi
WHEN NOT MATCHED BY TARGET
    THEN INSERT (Ad, KategoriId, BirimFiyat, StokAdedi)
         VALUES (kaynak.Ad, kaynak.KategoriId, kaynak.BirimFiyat, kaynak.StokAdedi)
WHEN NOT MATCHED BY SOURCE
    THEN UPDATE SET hedef.Aktif = 0;      -- silmek yerine pasifleştir
```

Noktalı virgül **zorunludur**; `MERGE` ifadesini `;` ile bitirmezsen hata alırsın.

**Neden temkinli:** `MERGE` uzun yıllar boyunca eşzamanlılık (concurrency) ve tetikleyici (trigger) kaynaklı hatalarla anıldı. Bilinen riskler:

- Yoğun eşzamanlı çalışmada tekrar eden anahtar hataları verebilir; `WITH (HOLDLOCK)` ile kilitlemen gerekir.
- `WHEN NOT MATCHED BY SOURCE` yazmayı unutmak ya da `ON` koşulunu yanlış kurmak, beklenmedik satırları etkiler. `ON` koşulu filtre değildir; filtreyi `AND` ile `WHEN` tarafına yazarsın.
- Hata ayıklaması zordur: tek ifade üç davranışı birden saklar.

```sql
-- Güvenli kullanım için hedefi kilitle
MERGE INTO Urunler WITH (HOLDLOCK) AS hedef
USING ...
```

Çoğu senaryoda daha okunaklı ve güvenli alternatif, ayrı ifadeler yazmaktır:

```sql
BEGIN TRAN;

UPDATE h SET h.BirimFiyat = k.BirimFiyat, h.StokAdedi = k.StokAdedi
FROM Urunler h JOIN @GelenUrunler k ON k.UrunId = h.UrunId;

INSERT INTO Urunler (Ad, KategoriId, BirimFiyat, StokAdedi)
SELECT k.Ad, k.KategoriId, k.BirimFiyat, k.StokAdedi
FROM @GelenUrunler k
WHERE NOT EXISTS (SELECT 1 FROM Urunler h WHERE h.UrunId = k.UrunId);

COMMIT;
```

Transaction ve kilitleme konusu yarınki notun ana başlıklarından biri.

```csharp
// EF Core 7+ : toplu güncelleme için ExecuteUpdate / ExecuteDelete
await _db.Urunler
    .Where(u => u.KategoriId == 5)
    .ExecuteUpdateAsync(s => s.SetProperty(u => u.Aktif, false));
// Tek UPDATE ifadesi üretir; entity'leri belleğe çekmez, change tracker'ı da kullanmaz
```

> **Bu benzetme şurada bozulur:** Depo sorumlusu listeyi işlerken kimse depoya girip çıkmaz. `MERGE` ise varsayılan kilit seviyesinde çalışırken başka bir oturum aynı satırı ekleyebilir; `MERGE` "yok" diye karar verip `INSERT` denediğinde satır artık vardır ve benzersizlik hatası alırsın. Bu yüzden `WITH (HOLDLOCK)` yazılır.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Logical processing order | SQL yan tümcelerinin değerlendirilme sırası |
| Alias (takma ad) | `SELECT` aşamasında doğan sütun/tablo adı |
| Inner join | Yalnızca eşleşen satırları döndüren birleştirme |
| Outer join | Eşleşmeyen tarafı da koruyan birleştirme (`LEFT`/`RIGHT`/`FULL`) |
| Anti-join | "Karşılığı olmayanları bul" kalıbı (`NOT EXISTS`, `LEFT JOIN ... IS NULL`) |
| Fan-out | `JOIN`'in satırları çoğaltması sonucu toplamların şişmesi |
| Kartezyen çarpım | Koşulsuz birleştirmede her satırın her satırla eşleşmesi |
| Aggregate fonksiyon | Çok satırdan tek değer üreten fonksiyon |
| `HAVING` | Gruplama sonrasında grupları eleyen yan tümce |
| Skaler alt sorgu | Tek bir değer döndüren alt sorgu |
| Korelasyonlu alt sorgu | Dış sorgunun sütununa referans veren, her satır için çalışan alt sorgu |
| `APPLY` | Her dış satır için tablo değerli bir ifadeyi çalıştıran operatör |
| CTE | `WITH` ile tanımlanan, adı olan geçici sonuç kümesi |
| Özyinelemeli CTE | Kendine referans veren, anchor + recursive parçadan oluşan CTE |
| Window function | Satırı yok etmeden pencere üzerinde hesap yapan fonksiyon |
| `PARTITION BY` | Pencereyi gruplara bölen yan tümce |
| Frame (`ROWS`/`RANGE`) | Pencere içinde hangi satırların hesaba katılacağı |
| Keyset pagination | `OFFSET` yerine son görülen anahtardan devam eden sayfalama |
| Set operatörü | İki sonucu alt alta birleştiren operatör (`UNION`, `EXCEPT`) |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `03-Stored-Procedure-View-Transaction.md` (Çarşamba)
