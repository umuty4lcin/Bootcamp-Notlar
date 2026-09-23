# Hafta 2 · Perşembe — SOLID 4-5: ISP, DIP ve Dependency Injection

**Okuma süresi:** ~48 dk
**Neden bu konu:** ASP.NET Core'un tamamı bu notun üzerine kurulu. `Program.cs`'teki her `AddScoped` satırı, her controller constructor'ı, her `IServiceProvider` çağrısı DIP ve DI'ın uygulanmış hâlidir. Framework bunu senin yerine yaptığı için çoğu kişi mekanizmayı hiç öğrenmeden kullanır — ve bir şey ters gittiğinde ne olduğunu anlayamaz. Bu not, framework devreye girmeden önce mekanizmayı elle kurar.

---

## Önce Basitçe

Bir sınıf tek başına iş göremez. Veritabanına yazması gerekir, e-posta atması gerekir, saate bakması gerekir. Bunlara o sınıfın **bağımlılıkları** denir. Asıl soru şudur: bu bağımlılıkları sınıf kendi mi bulsun, yoksa dışarıdan mı verilsin?

Kendi bulursa, sınıf içinde `new SqlBaglantisi()` yazar. Çalışır. Ama o sınıf artık SQL Server'a çivilenmiştir. Testte veritabanı gerekir, başka bir veritabanına geçmek imkânsızdır, o sınıfı kullanan herkes zincirin tamamını sürüklemek zorunda kalır. Dışarıdan verilirse, sınıf sadece "bana veri kaydedebilen bir şey lazım" der; kimin verdiğine karışmaz. Bu ikinci yaklaşımın adı **bağımlılık enjeksiyonu**dur.

Notun ilk yarısı bu fikrin iki ilkesini anlatıyor. ISP, "birine iş verirken, yapmayacağı işleri de listeye yazma" der. Bir sınıfa on metotlu bir arayüz dayatırsan, o sınıf kullanmadığı sekiz metodu da uygulamak zorunda kalır. DIP ise "üstteki kod alttakine bağlı olmasın, ikisi de ortadaki bir sözleşmeye bağlı olsun" der. İş kuralın SQL'i bilmemeli; SQL kodu iş kuralının sözleşmesine uymalı.

İkinci yarı tamamen pratiğe ayrıldı. Bağımlılığı bir sınıfa nasıl verirsin, bu iş kodun hangi noktasında yapılır, `new` nereye yazılır, konteynerin içinde ne var. Bir "DI konteyneri" yazacağız — otuz satır. Amaç, ASP.NET Core'un `builder.Services` üzerinden yaptığı şeyin sihir olmadığını görmen.

Bir de çok yaygın bir karışıklık var: DIP ile DI aynı şey sanılır. Değiller, ayrı bir bölümü hak edecek kadar farklılar. DIP bir **tasarım kararıdır** (neye bağlanıyorsun), DI ise bir **teslim yöntemidir** (o bağımlılık sana nasıl ulaşıyor). Birini yapıp diğerini yapmamak mümkündür.

Son bölüm DI'ın asıl kazancı üzerine: test edilebilirlik. Veritabanına giden bir kodu test edemezsin; yerine sahte bir nesne koyabildiğinde test edersin. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bir lokanta düşün. Kötü kurgu: aşçı her sabah kendi pazara gider, kendi sebzesini seçer, kendi kasabına uğrar. Aşçının yemekle ilgisi olmayan yarım günü gider ve pazar kapalıysa mutfak durur. İyi kurgu: aşçı mutfakta bekler, tedarikçi malzemeyi getirip tezgâha bırakır. Aşçı "domates lazım" der, domatesin nereden geldiğine karışmaz. DI budur. ISP ise şudur: aşçıya iş tarifi verirken "bulaşık da yıkarsın, kasa da tutarsın" diye yazmamaktır.

---

## Bu Notta Ne Var

1. ISP: şişman arayüz problemi
2. Rol arayüzleri ve arayüzü bölme
3. DIP: bağımlılık yönünü tersine çevirmek
4. Soyutlama kimin katmanında tanımlanır
5. DI: bağımlılığı dışarıdan vermenin üç yolu
6. Composition root: `new` nereye yazılır
7. Service locator neden anti-pattern
8. DIP ile DI aynı şey değildir
9. Elle DI konteyneri: sihir yok
10. ASP.NET Core'un yerleşik konteyneri ve yaşam süreleri
11. Test edilebilirlik: DI'ın asıl kazancı

---

## 1. ISP: Şişman Arayüz Problemi

> **Benzetme —** Bir iş ilanı düşün. "Aranan eleman: muhasebe tutacak, sosyal medya yönetecek, kargo takip edecek, forklift kullanacak." Bu ilana başvuran kişi dört işin dördünü de bilmek zorunda. Oysa şirketin gerçek ihtiyacı muhasebe. İlanı şişirdiğin için, uygun adayların çoğu eleniyor ve işe alınan kişi zamanının dörtte üçünde işe yaramayan bir yetkinlik taşıyor. Arayüz de bir iş ilanıdır: içine ne yazarsan, onu uygulayan sınıf ondan sorumlu olur.

**Basitçe:** Bir arayüze çok fazla metot koyarsan, onu uygulayan sınıflar kullanmadıkları metotları da yazmak zorunda kalır. O metotların gövdesi ya boş kalır ya da istisna fırlatır. İkisi de kötüdür.

**Teknik olarak:** **ISP (Interface Segregation Principle — Arayüz Ayrımı İlkesi)** — Hiçbir istemci (client), kullanmadığı metotlara bağımlı olmaya zorlanmamalıdır. Çok amaçlı tek bir arayüz yerine, küçük ve amaca özel arayüzler tercih edilir.

Dikkat: ilke, arayüzü **uygulayanı** değil **kullananı** merkeze alır. "Client" burada arayüzü çağıran koddur. Soru şudur: çağıran taraf bu arayüzün kaç metodunu gerçekten kullanıyor?

### Kötü kod: her şeyi yapan arayüz

```csharp
// KÖTÜ — tek arayüz, sekiz metot, hiçbir sınıf hepsini istemiyor
public interface IPersonel
{
    void MaasAl(decimal tutar);
    void IzinKullan(int gun);
    void FazlaMesaiYap(int saat);
    void SirketAracıKullan(string plaka);
    void PerformansDegerlendir(int calisanId);
    void ButceOnayla(decimal tutar);
    void IseAlim(string aday);
    void FaturaKes(decimal tutar);
}
```

Şimdi bunu üç farklı sınıf uygulasın:

```csharp
// Stajyer: sekiz metottan ikisini kullanıyor
public class Stajyer : IPersonel
{
    public void MaasAl(decimal tutar) { /* burs ödemesi */ }
    public void IzinKullan(int gun) { /* çalışır */ }

    public void FazlaMesaiYap(int saat) => throw new NotSupportedException("Stajyer mesai yapamaz");
    public void SirketAracıKullan(string plaka) => throw new NotSupportedException();
    public void PerformansDegerlendir(int calisanId) => throw new NotSupportedException();
    public void ButceOnayla(decimal tutar) => throw new NotSupportedException();
    public void IseAlim(string aday) => throw new NotSupportedException();
    public void FaturaKes(decimal tutar) => throw new NotSupportedException();
}

// Taşeron: maaş almıyor, izin kullanmıyor, fatura kesiyor
public class Taseron : IPersonel
{
    public void FaturaKes(decimal tutar) { /* çalışır */ }

    public void MaasAl(decimal tutar) => throw new NotSupportedException();
    public void IzinKullan(int gun) => throw new NotSupportedException();
    // ... altı metot daha, hepsi istisna
}
```

Sorunlar:

| Sorun | Sonuç |
|---|---|
| Boş/istisnalı gövdeler | Aynı zamanda LSP ihlali — alt tip sözleşmeyi karşılamıyor |
| Arayüze metot eklemek | **Tüm** uygulayan sınıfları derlenmez hâle getirir |
| Test sahteleri şişer | `Mock<IPersonel>` sekiz metot kurmayı gerektirir |
| Niyet kaybolur | Bir parametreye `IPersonel` yazınca ne istediğin anlaşılmaz |

> **İlişki notu:** ISP ile LSP iç içe geçer. Dünkü notta gördüğün `NotImplementedException` kokusunun kaynağı çoğu zaman şişman bir arayüzdür. LSP ihlalini görürsün, sebebini ISP'de bulursun.

### Şişmanlığı nasıl fark edersin

- Bir arayüzün metotlarının farklı alt kümeleri farklı sınıflarca kullanılıyorsa.
- Uygulayan sınıfların çoğunda boş gövde ya da `throw` varsa.
- Arayüz adı `IXServisi` gibi genel, metotları birbiriyle alakasızsa.
- Bir metot eklediğinde beş dosya kırmızı oluyorsa.

> **Bu benzetme şurada bozulur:** İş ilanı benzetmesinde ilanı şişirmenin bedelini işe alınan kişi öder. Kodda ise bedeli **arayüzü kullanan taraf** da öder. Bir controller'a `IPersonel` verdiğinde, controller o sekiz metodun hepsine teknik olarak erişebilir — yani yanlışlıkla çağırabilir. Küçük arayüz sadece uygulayanı değil, çağıranı da korur.

---

## 2. Rol Arayüzleri ve Arayüzü Bölme

> **Benzetme —** Bir apartmanda üç anahtar vardır: daire anahtarı, bina kapısı anahtarı, kazan dairesi anahtarı. Herkese tek bir "her yeri açan" anahtar vermek pratik görünür ama kimse bunu istemez. Kapıcıya kazan dairesi, sakine daire, postacıya sadece bina kapısı. Her rol kendi anahtarını taşır. Kodda da her çağıran taraf, sadece ihtiyacı olan sözleşmeyi taşımalıdır.

**Basitçe:** Şişman arayüzü, kullanım rollerine göre böl. Her parça tek bir yeteneği temsil etsin. Bir sınıf birden fazla rolü üstlenebilir — bu sorun değil, çünkü seçim onun.

**Teknik olarak:** **Role interface (rol arayüzü)** — Bir tipin tamamını değil, tek bir yeteneğini/rolünü tanımlayan küçük arayüz. Karşıtı **header interface**tir: somut sınıfın bütün public metotlarını birebir kopyalayan arayüz.

### İyi kod: rollere bölünmüş arayüzler

```csharp
// Her arayüz tek bir yeteneği temsil ediyor
public interface IMaasAlan    { void MaasAl(decimal tutar); }
public interface IIzinKullanan{ void IzinKullan(int gun); }
public interface IMesaiYapan  { void FazlaMesaiYap(int saat); }
public interface IYonetici    { void PerformansDegerlendir(int calisanId); void ButceOnayla(decimal tutar); }
public interface IFaturaKesen { void FaturaKes(decimal tutar); }

// Sınıflar sadece yapabildikleri rolleri üstleniyor
public class Stajyer : IMaasAlan, IIzinKullanan
{
    public void MaasAl(decimal tutar) { /* burs */ }
    public void IzinKullan(int gun) { /* ... */ }
}

public class Taseron : IFaturaKesen
{
    public void FaturaKes(decimal tutar) { /* ... */ }
}

public class DepartmanMuduru : IMaasAlan, IIzinKullanan, IMesaiYapan, IYonetici
{
    public void MaasAl(decimal tutar) { /* ... */ }
    public void IzinKullan(int gun) { /* ... */ }
    public void FazlaMesaiYap(int saat) { /* ... */ }
    public void PerformansDegerlendir(int calisanId) { /* ... */ }
    public void ButceOnayla(decimal tutar) { /* ... */ }
}
```

Tek bir `throw` kalmadı. Her sınıf sadece yapabildiğini vaat ediyor.

Çağıran taraf da netleşti:

```csharp
// Niyet artık imzada görünüyor
public void BordroIsle(IEnumerable<IMaasAlan> alanlar) { /* sadece maaş */ }
public void ButceTurunuOnayla(IYonetici onaylayan, decimal tutar) => onaylayan.ButceOnayla(tutar);
```

`BordroIsle` metoduna bakan biri, bu metodun izin ya da fatura işine karışmadığını imzadan anlar.

### CQS ayrımı: okuma ve yazma arayüzleri

En sık işine yarayacak bölme biçimi budur. Bir deponun okuma ve yazma yeteneklerini ayırmak:

```csharp
// Şişman depo arayüzü
public interface IUrunDeposu
{
    Urun? Getir(int id);
    IReadOnlyList<Urun> Listele();
    void Ekle(Urun urun);
    void Guncelle(Urun urun);
    void Sil(int id);
}

// Rollere bölünmüş hâli
public interface IUrunOkuyucu { Urun? Getir(int id); IReadOnlyList<Urun> Listele(); }
public interface IUrunYazici  { void Ekle(Urun urun); void Guncelle(Urun urun); void Sil(int id); }
```

Kazanç somut: bir rapor servisine `IUrunOkuyucu` verirsin. O servis veri değiştiremez — derleyici garanti eder. `IUrunDeposu` verseydin, yanlışlıkla `Sil` çağırmasını hiçbir şey engellemezdi.

> **MvcCv notu:** Projendeki `GenericRepository<T>` klasik bir şişman arayüzdür: `Ekle`, `Sil`, `Guncelle`, `GetAll`, `GetById`, `Where` hepsi bir arada. Küçük projelerde işe yarar. Ama salt okuma yapan bir controller'a bu arayüzü vermek, ona silme yetkisi vermek demektir. Bölmenin en ucuz yolu, generic repository'yi korumak ve üstüne dar arayüzler geçirmektir.

### Bölmenin sınırı

ISP'yi de abartabilirsin. Her metoda ayrı arayüz açmak, ISP değil gürültüdür.

```csharp
// AŞIRI — her metot ayrı arayüz, hiçbir çağıran taraf tek metot kullanmıyor
public interface ISiparisGetirici { Siparis Getir(int id); }
public interface ISiparisListeleyici { IReadOnlyList<Siparis> Listele(); }
public interface ISiparisSayici { int Say(); }
public interface ISiparisVarMi { bool VarMi(int id); }
```

Doğru ölçü: **arayüzü çağıran tarafların ihtiyacına göre böl.** Eğer her çağıran taraf `Getir` ve `Listele`'yi birlikte kullanıyorsa, o ikisi aynı arayüzde kalır.

| Bölme gerekçesi | Geçerli mi |
|---|---|
| Farklı sınıflar farklı alt kümeleri kullanıyor | Evet |
| Uygulayan sınıf bazı metotları yapamıyor | Evet |
| Okuma/yazma yetkisini ayırmak istiyorsun | Evet |
| "Arayüzler küçük olmalı" genel kuralı | Hayır, tek başına yetersiz |

> **Bu benzetme şurada bozulur:** Anahtar benzetmesinde anahtar sayısı arttıkça anahtarlık şişer ama kimse zorlanmaz. Kodda arayüz sayısı arttıkça **kayıt (registration) yükü** artar: her arayüzü konteynere ayrı ayrı bağlaman gerekir. Aynı sınıfı beş arayüzle kaydetmek `Program.cs`'i şişirir. Bu yüzden bölmeyi ihtiyaç doğurmalı, estetik değil.

---

## 3. DIP: Bağımlılık Yönünü Tersine Çevirmek

> **Benzetme —** Bir inşaatta usta ile çırak ilişkisini düşün. Kötü kurgu: usta "Ahmet'i çağırın, harcı o karsın" der. Ahmet izinliyse iş durur. İyi kurgu: usta "bana harç karan biri lazım" der. Kim geldiği önemli değil, harcı karabilen herkes olur. Usta artık Ahmet'e değil, **harç karma tarifine** bağlıdır. Ahmet de aynı tarife uyar. İkisi de ortadaki sözleşmeye bağlıdır, birbirine değil.

**Basitçe:** İş kuralını yazan kod, veritabanı kodunu tanımamalı. İkisi de ortadaki bir arayüzü tanımalı. Böylece veritabanı değiştiğinde iş kuralı değişmez.

**Teknik olarak:** **DIP (Dependency Inversion Principle — Bağımlılığın Tersine Çevrilmesi İlkesi)** iki cümleden oluşur:

1. Üst seviye modüller alt seviye modüllere bağımlı olmamalıdır. **İkisi de soyutlamalara bağımlı olmalıdır.**
2. Soyutlamalar ayrıntılara bağımlı olmamalıdır. **Ayrıntılar soyutlamalara bağımlı olmalıdır.**

**Üst seviye modül** — İş kuralını, politikayı içeren kod. "Sipariş nasıl işlenir", "fatura ne zaman kesilir".
**Alt seviye modül** — Mekanizmayı içeren kod. "SQL nasıl yazılır", "SMTP nasıl konuşulur", "dosya nasıl açılır".

Sezgiye ters gelen kısım şu: normalde üst seviye alta bağlıdır. DIP bu oku **tersine çevirir**.

### Kötü kod: bağımlılık aşağı doğru

```csharp
// KÖTÜ — üst seviye iş kuralı, alt seviye ayrıntıya çivilenmiş
public class FaturaServisi                    // üst seviye
{
    private readonly SqlFaturaDeposu _depo = new();       // alt seviyeye doğrudan bağlı
    private readonly SmtpEpostaGonderici _mail = new();   // alt seviyeye doğrudan bağlı

    public void Kes(Siparis siparis)
    {
        var fatura = new Fatura(siparis.Toplam);
        _depo.Kaydet(fatura);                             // SQL Server'a çivili
        _mail.Gonder(siparis.Email, "Faturanız hazır");   // SMTP'ye çivili
    }
}
```

Bağımlılık oku: `FaturaServisi → SqlFaturaDeposu → System.Data.SqlClient`

Sonuçları:

- Veritabanını PostgreSQL'e çevirmek için `FaturaServisi`'ni değiştirmen gerekir. Oysa fatura kesme kuralı değişmedi.
- Test yazmak için ayakta bir SQL Server ve SMTP sunucusu gerekir.
- `FaturaServisi`'ni başka bir projede kullanmak için veri erişim katmanını da taşıman gerekir.

### İyi kod: bağımlılık soyutlamaya

```csharp
// Soyutlama — iş kuralının ihtiyacını tarif eder
public interface IFaturaDeposu    { void Kaydet(Fatura fatura); }
public interface IEpostaGonderici { void Gonder(string adres, string konu); }

// Üst seviye — sadece soyutlamayı tanır
public class FaturaServisi(IFaturaDeposu depo, IEpostaGonderici mail)
{
    public void Kes(Siparis siparis)
    {
        var fatura = new Fatura(siparis.Toplam);
        depo.Kaydet(fatura);
        mail.Gonder(siparis.Email, "Faturanız hazır");
    }
}

// Alt seviye — soyutlamaya uymak zorunda
public class SqlFaturaDeposu : IFaturaDeposu
{
    public void Kaydet(Fatura fatura) { /* SQL Server */ }
}

public class SmtpEpostaGonderici : IEpostaGonderici
{
    public void Gonder(string adres, string konu) { /* SMTP */ }
}
```

Bağımlılık oku artık şöyle: `FaturaServisi → IFaturaDeposu ← SqlFaturaDeposu`

Alt seviye modül, yukarıyı gösteriyor. "Tersine çevirme" denen şey budur.

| Önce | Sonra |
|---|---|
| İş kuralı SQL'i biliyor | İş kuralı sadece "kaydet" sözleşmesini biliyor |
| Veritabanı değişince iş kuralı değişir | Veritabanı değişince yeni bir sınıf eklenir |
| Test için gerçek altyapı gerekir | Test için sahte nesne yeter |

> **Uyarı:** DIP "her şeye arayüz aç" demek değildir. Soyutlama, **dış dünyaya dokunan** ve **değişmesi muhtemel** şeyler için gerekir: veritabanı, ağ, dosya sistemi, saat, dış servis. Bir `decimal` hesabı yapan saf metot için arayüz açmak, dünkü notta gördüğün spekülatif soyutlamadır.

> **Bu benzetme şurada bozulur:** Usta-çırak benzetmesinde "harç karma tarifi" herkesin bildiği ortak bir standarttır, kimseye ait değildir. Kodda ise arayüzün bir dosyası, bir projesi, bir sahibi vardır. Bu arayüz **kimin projesinde** duracak sorusu, DIP'in en çok atlanan kısmıdır ve bir sonraki bölümün konusudur.

---

## 4. Soyutlama Kimin Katmanında Tanımlanır

> **Benzetme —** Şirket bir temizlik firmasıyla anlaşıyor. Sözleşmeyi kim yazar? Temizlik firması kendi standart sözleşmesini dayatırsa, şirket o firmanın çalışma biçimine uymak zorunda kalır ve firmayı değiştirmek istediğinde sözleşme de değişir. Şirket kendi şartnamesini yazarsa — "haftada üç gün, şu saatlerde, şu alanlar" — hangi firma gelirse gelsin şartnameye uyar. Şartname **talep edenin**dir, hizmeti verenin değil.

**Basitçe:** Arayüzü, onu **kullanan** katman tanımlar; uygulayan katman değil. `IFaturaDeposu`, veri erişim projesinde değil, iş kuralının olduğu projede durur.

**Teknik olarak:** Bu ayrıntı çoğu anlatımda atlanır ama DIP'in özü buradadır. Arayüzü uygulayanın yanına koyarsan, bağımlılık oku gerçekten tersine dönmez — sadece araya bir dosya koymuş olursun.

### Yanlış yerleşim

```
MvcCv.Web          -> MvcCv.Business  -> MvcCv.DataAccess
                                         IFaturaDeposu.cs      <-- arayüz burada
                                         SqlFaturaDeposu.cs
```

`MvcCv.Business` projesi, `IFaturaDeposu`'nu kullanmak için `MvcCv.DataAccess`'e **referans vermek zorunda**. Yani iş kuralı hâlâ veri erişim projesine bağlı. Arayüz eklemek bu bağı koparmadı, sadece görünmez yaptı.

### Doğru yerleşim

```
MvcCv.Web          -> MvcCv.Business        <- MvcCv.DataAccess
                      IFaturaDeposu.cs         SqlFaturaDeposu.cs
                      FaturaServisi.cs         (Business'a referans verir)
```

Ok yönüne dikkat et: `DataAccess`, `Business`'a referans veriyor. İş kuralı projesi hiçbir şeye bağlı değil.

Bu yerleşimin adı **Onion Architecture** ya da **Clean Architecture**tır. Merkezde iş kuralı durur, dış halkalar içeriye bağlanır. Hafta 5'te mimari notunda tekrar karşına çıkacak.

### Arayüzün adı da tüketiciye aittir

Sahiplik sadece dosya yeri meselesi değil. Arayüzün **şekli** de tüketicinin ihtiyacına göre çizilir.

```csharp
// KÖTÜ — arayüz, veritabanı gerçeklerini yansıtıyor (sağlayıcının dili)
public interface IFaturaDeposu
{
    SqlDataReader FaturaSorgusuCalistir(string sql);
    void BeginTransaction();
    void Commit();
}

// İYİ — arayüz, iş kuralının ihtiyacını yansıtıyor (tüketicinin dili)
public interface IFaturaDeposu
{
    Fatura? Getir(int faturaNo);
    void Kaydet(Fatura fatura);
    IReadOnlyList<Fatura> OdenmemisleriGetir(DateTime sonTarih);
}
```

İlkinde iş kuralı SQL'in varlığını bilir; `SqlDataReader` tipini görür. İkincisinde arayüzün arkasında SQL de olabilir, dosya da, bellek içi liste de. İş kuralı fark etmez.

> **Pratik test:** Arayüze bakıp "bunun arkasında veritabanı mı var, dosya mı, HTTP servisi mi" sorusuna cevap verebiliyorsan, arayüz sağlayıcının dilinde yazılmış demektir. İyi bir soyutlama, arkasındaki teknolojiyi sızdırmaz.

> **Bu benzetme şurada bozulur:** Şartname benzetmesinde şirket tek taraflı yazar ve firma uyar. Kodda ise bazen tüketici çok fazladır: aynı `IFaturaDeposu`'nu beş farklı servis kullanıyorsa, arayüz hepsinin ihtiyacının birleşimi hâline gelir ve şişmeye başlar. Bu noktada DIP ile ISP çakışır. Çözüm, servis başına dar arayüzler tanımlamaktır — aynı sınıf birden fazla dar arayüzü uygulayabilir.

---

## 5. DI: Bağımlılığı Dışarıdan Vermenin Üç Yolu

> **Benzetme —** Nöbetçi eczaneyi düşün. Eczacı, hangi ilacın stokta olduğunu depoya kendi gidip bakmaz; depo görevlisi sabah ilaçları rafa dizer. Eczacı raftakiyle çalışır. Malzemeyi kullanan kişi ile malzemeyi tedarik eden kişi farklıdır. Bağımlılık enjeksiyonunun tamamı bu ayrımdır.

**Basitçe:** Bir sınıfın ihtiyaç duyduğu nesneleri, o sınıf kendi üretmez; dışarıdan verilir. Vermenin üç yolu var: constructor'dan, property'den, metottan.

**Teknik olarak:** **Dependency Injection (bağımlılık enjeksiyonu)** — Bir bileşenin bağımlılıklarının, o bileşen tarafından oluşturulmak yerine dışarıdan sağlanması. **Inversion of Control (IoC — kontrolün tersine çevrilmesi)** kalıbının bir uygulamasıdır.

### 1) Constructor injection — varsayılan tercih

```csharp
public class SiparisServisi
{
    private readonly IStokKontrolu _stok;
    private readonly ISiparisDeposu _depo;

    public SiparisServisi(IStokKontrolu stok, ISiparisDeposu depo)
    {
        _stok = stok ?? throw new ArgumentNullException(nameof(stok));
        _depo = depo ?? throw new ArgumentNullException(nameof(depo));
    }
}

// C# 12 primary constructor ile aynı şey, daha kısa
public class SiparisServisi(IStokKontrolu stok, ISiparisDeposu depo)
{
    public void Olustur(SiparisKomutu k) { /* stok ve depo doğrudan kullanılır */ }
}
```

Neden varsayılan tercih:

| Özellik | Kazanç |
|---|---|
| `readonly` olabilir | Nesne kurulduktan sonra bağımlılık değişmez |
| Zorunludur | Bağımlılık verilmeden nesne oluşturulamaz |
| İmzada görünür | Sınıfın neye ihtiyacı olduğu tek bakışta anlaşılır |
| Yarım nesne olmaz | Constructor bitince nesne kullanıma hazırdır |

Son madde ayrıca bir tasarım sinyali verir: constructor'da altı parametre varsa, o sınıf muhtemelen SRP'yi ihlal ediyordur. Constructor parametre sayısı, bedava gelen bir tasarım ölçüsüdür.

### 2) Property injection — isteğe bağlı bağımlılıklar

```csharp
public class RaporUretici
{
    // Zorunlu: constructor'dan
    private readonly IRaporDeposu _depo;
    public RaporUretici(IRaporDeposu depo) => _depo = depo;

    // İsteğe bağlı: verilmezse varsayılan davranış devam eder
    public ILogger Logger { get; set; } = NullLogger.Instance;
}
```

Sadece **gerçekten isteğe bağlı** bağımlılıklar için kullanılır ve makul bir varsayılan (null object) şarttır. Zorunlu bir bağımlılığı property'den vermek, `NullReferenceException` üretme sözüdür.

### 3) Method injection — çağrı başına değişen bağımlılık

```csharp
public class FiyatHesaplayici
{
    // Kur bilgisi her çağrıda farklı olabilir; sınıfa ait değil, çağrıya ait
    public decimal Hesapla(Siparis siparis, IDovizKuru kur)
        => siparis.Kalemler.Sum(k => kur.Cevir(k.BirimFiyat, k.ParaBirimi) * k.Adet);
}
```

Bağımlılık sınıfın ömrü boyunca değil, tek bir çağrı boyunca gerekiyorsa metot parametresi doğru yerdir.

### Hangisi ne zaman

| Yol | Ne zaman | Dikkat |
|---|---|---|
| Constructor | Neredeyse her zaman | Parametre sayısı 4'ü geçiyorsa SRP'ye bak |
| Property | Gerçekten opsiyonel, makul varsayılanı var | Zorunlu bağımlılık için asla |
| Method | Bağımlılık çağrı başına değişiyor | İmzayı şişirir, ölçülü kullan |

> **Bu benzetme şurada bozulur:** Eczane benzetmesinde rafa dizilen ilaç fizikseldir ve tek bir tanedir. Kodda ise aynı bağımlılığın **kaç kopyası** olduğu ayrı bir karardır: her istek için yeni mi, uygulama boyunca tek mi? Bu soru DI'ın ayrı bir başlığıdır — yaşam süresi (lifetime) — ve 10. bölümde işlenir.

---

## 6. Composition Root: `new` Nereye Yazılır

> **Benzetme —** Bir tiyatro oyununu düşün. Oyuncular sahnede rollerini oynar; kimin hangi rolü oynayacağına perde açılmadan önce, kulisteki yönetmen karar verir. Sahnedeki oyuncu "ben kiminle oynayacağım" diye seçim yapmaz. Kodda da nesnelerin birbirine bağlanma kararı, uygulamanın en dışındaki tek bir noktada verilir.

**Basitçe:** DI'ın şöyle bir sorusu vardır: madem hiçbir sınıf kendi bağımlılığını `new`'lemiyor, o zaman bu nesneleri kim üretiyor? Cevap: uygulamanın giriş noktasındaki tek bir yer.

**Teknik olarak:** **Composition root (birleştirme kökü)** — Uygulamanın nesne grafiğinin kurulduğu, giriş noktasına mümkün olduğunca yakın tek nokta. Mark Seemann'ın terimidir.

| Uygulama tipi | Composition root |
|---|---|
| ASP.NET Core | `Program.cs` |
| Console | `Main` metodu |
| Windows Service / Worker | `Main` / `CreateHostBuilder` |
| Test projesi | Test sınıfının kurulum metodu |

### Framework'süz composition root

```csharp
// Program.cs — tüm new'ler burada, başka hiçbir yerde yok
public static class Program
{
    public static void Main(string[] args)
    {
        // 1) Altyapı — en dış halka
        var baglantiMetni = "Server=.;Database=MvcCv;Trusted_Connection=True;TrustServerCertificate=True";
        IFaturaDeposu depo = new SqlFaturaDeposu(baglantiMetni);
        IEpostaGonderici mail = new SmtpEpostaGonderici("smtp.sirket.com", 587);
        ILogger logger = new KonsolLogger();

        // 2) İş kuralı — altyapıyı alır
        var faturaServisi = new FaturaServisi(depo, mail);
        var siparisServisi = new SiparisServisi(depo, faturaServisi, logger);

        // 3) Giriş noktası — iş kuralını alır
        var uygulama = new SiparisUygulamasi(siparisServisi);
        uygulama.Calistir(args);
    }
}
```

Bütün bağlantı kararları tek ekranda. Hangi somut sınıfın hangi arayüzü karşıladığını görmek için tek dosya açman yeter.

### `new` yasak mı

Hayır. Yasak olan, **değiştirilmesi gerekebilecek bağımlılıkları** rastgele yerlerde `new`'lemektir. Şunları her yerde `new`'leyebilirsin:

```csharp
// Bunlar composition root gerektirmez
var liste = new List<Urun>();                       // veri yapısı
var siparis = new Siparis(musteriId, kalemler);     // varlık / değer nesnesi
var sonuc = new SiparisSonucu(true, siparis.Id);    // DTO
var sb = new StringBuilder();                       // yerel yardımcı
```

Ayrım şu: **davranış taşıyan ve yerine başkası konulabilecek** şeyler enjekte edilir; **veri taşıyan** şeyler yerinde `new`'lenir.

| `new` serbest | Enjekte edilir |
|---|---|
| Koleksiyonlar, DTO, varlık, değer nesnesi | Depo, e-posta, HTTP istemcisi, saat |
| İstisna nesneleri | Loglama, önbellek, dış servis istemcisi |
| `StringBuilder`, `Random` (deterministiklik gerekmiyorsa) | Rastgelelik testte sabitlenecekse |

### Ara katmanlar `new` yapmaz

En sık yapılan hata, composition root'u "ilan edip" sonra ara katmanlarda `new`'lemeye devam etmektir.

```csharp
// KÖTÜ — servis kendi bağımlılığını üretiyor, composition root devre dışı
public class SiparisServisi(IStokKontrolu stok)
{
    public void Olustur(SiparisKomutu k)
    {
        var mail = new SmtpEpostaGonderici("smtp.sirket.com", 587);  // burada olmamalı
        mail.Gonder(k.Email, "Siparişiniz alındı");
    }
}

// İYİ — bağımlılık dışarıdan gelir
public class SiparisServisi(IStokKontrolu stok, IEpostaGonderici mail)
{
    public void Olustur(SiparisKomutu k) => mail.Gonder(k.Email, "Siparişiniz alındı");
}
```

> **Bu benzetme şurada bozulur:** Tiyatroda yönetmen rol dağıtımını bir kez yapar ve oyun boyunca değişmez. Kodda ise bazı bağımlılıkların çalışma anında seçilmesi gerekir: kullanıcının seçtiği ödeme yöntemi, isteğin geldiği ülkeye göre vergi kuralı. Bunun çözümü composition root'u terk etmek değil, oraya bir **fabrika** (`IOdemeYontemiFabrikasi`) kaydetmektir. Seçim kararı çalışma anına kayar, ama nesne üretme sorumluluğu yine tek noktada kalır.

---

## 7. Service Locator Neden Anti-Pattern

> **Benzetme —** İki tür market var. Birincisinde raflar açık: sepetine ne koyduğun görünür, kasada herkes ne aldığını görür. İkincisinde tek bir tezgâh var, "bana bir şey lazım" dersin ve arkadan getirirler; ne istediğin fişe yazılmaz. İkinci dükkândan çıkan birinin çantasında ne olduğunu kimse bilemez. Service locator, ikinci dükkândır: sınıfın neye bağımlı olduğu dışarıdan görünmez.

**Basitçe:** Service locator, bağımlılığı constructor'dan almak yerine, bir "her şeyi veren" nesneden istemektir. Çalışır, ama sınıfın bağımlılıkları gizli kalır.

**Teknik olarak:** **Service locator** — Bileşenlerin bağımlılıklarını merkezi bir kayıt nesnesinden (`IServiceProvider`, statik bir `Container`) çalışma anında talep ettiği kalıp.

### Kötü kod: gizlenmiş bağımlılıklar

```csharp
// KÖTÜ — bağımlılıklar constructor'da görünmüyor
public class SiparisServisi
{
    private readonly IServiceProvider _saglayici;
    public SiparisServisi(IServiceProvider saglayici) => _saglayici = saglayici;

    public void Olustur(SiparisKomutu k)
    {
        var stok = _saglayici.GetRequiredService<IStokKontrolu>();
        var depo = _saglayici.GetRequiredService<ISiparisDeposu>();
        var mail = _saglayici.GetRequiredService<IEpostaGonderici>();
        // ...
    }
}
```

Neden kötü:

| Sorun | Açıklama |
|---|---|
| Gizli bağımlılık | Constructor `IServiceProvider` diyor; gerçek üç bağımlılık gövdede saklı |
| Geç hata | Eksik kayıt derlemede değil, o kod satırı çalıştığında patlar |
| Zor test | Sahte bir `IServiceProvider` kurup üç ayrı `GetRequiredService` senaryosu yazman gerekir |
| Sızan bağımlılık | Konteynere bağlanan her sınıf, DI altyapısını bilmek zorunda kalır |
| SRP baskısı görünmez | Constructor şişmediği için sınıfın on bağımlılığa çıktığını fark etmezsin |

Son madde önemlidir. Constructor injection'ın gizli faydası, kötü tasarımı **acı verici hâle getirmesidir.** Sekiz parametreli bir constructor yazarken rahatsız olursun ve sınıfı bölersin. Service locator o acıyı yok eder; sınıf sessizce büyür.

### İyi kod

```csharp
// İYİ — bağımlılıklar imzada
public class SiparisServisi(
    IStokKontrolu stok,
    ISiparisDeposu depo,
    IEpostaGonderici mail)
{
    public void Olustur(SiparisKomutu k) { /* ... */ }
}
```

### `IServiceProvider` hiç kullanılmaz mı

Kullanılır — ama yalnızca composition root'ta ve altyapı kodunda. Framework'ün kendisi zaten service locator gibi çalışır; fark, bunu **senin iş kodunun** yapmamasıdır.

```csharp
// Meşru kullanım: composition root'ta scope açmak (ör. bir arka plan görevinde)
using var scope = app.Services.CreateScope();
var migrasyon = scope.ServiceProvider.GetRequiredService<VeriTabaniBaslatici>();
await migrasyon.CalistirAsync();
```

> **Ayırt edici soru:** `IServiceProvider` çağrısı, uygulamanın **dış kabuğunda** mı yoksa iş kuralının **içinde** mi? Dışındaysa sorun yok. İçindeyse service locator'dır.

> **Bu benzetme şurada bozulur:** Market benzetmesi, service locator'ın hep gizlilik ürettiğini ima eder. Gerçekte bazı yerlerde gizlilik kaçınılmazdır: eklenti (plugin) mimarilerinde hangi tiplerin yükleneceği derleme anında bilinmez. Orada bir çözücüye (resolver) başvurmak tasarım hatası değil, zorunluluktur. Kural iş kuralı katmanı için geçerlidir, altyapı için değil.

---

## 8. DIP ile DI Aynı Şey Değildir

> **Benzetme —** Bir inşaat ruhsatı ile kamyonu düşün. Ruhsat, binanın **nasıl olacağına** dair karardır: kaç kat, hangi malzeme. Kamyon ise malzemeyi şantiyeye **taşıyan araçtır**. Ruhsatsız kamyon da olur (malzeme gelir ama plansız bina çıkar), kamyonsuz ruhsat da (plan vardır, malzeme elde taşınır). İkisi farklı şeydir. DIP ruhsat, DI kamyondur.

**Basitçe:** DIP bir tasarım ilkesidir: "neye bağlanıyorsun" sorusuna cevap verir. DI bir tekniktir: "o bağımlılık sana nasıl ulaşıyor" sorusuna cevap verir. Birini yapıp diğerini yapmamak mümkündür.

**Teknik olarak:** Üç kavramı ayırmak gerekir.

| Kavram | Ne | Sorusu |
|---|---|---|
| **DIP** | Tasarım ilkesi | Somuta mı soyuta mı bağlıyım? |
| **IoC** | Genel kalıp | Kontrolü kim elinde tutuyor? |
| **DI** | Uygulama tekniği | Bağımlılık nesneye nasıl teslim ediliyor? |

**IoC (Inversion of Control)** DI'dan geniştir: olay yönelimli programlama, template method deseni, framework'ün senin kodunu çağırması ("Hollywood prensibi") de IoC'dir. DI, IoC'nin bağımlılıklara uygulanmış hâlidir.

### DI var, DIP yok

En sık görülen kombinasyon budur. Enjeksiyon yapılır ama somut tip enjekte edilir.

```csharp
// DI var (constructor'dan geliyor) — DIP yok (somut sınıfa bağlı)
public class SiparisServisi(SqlSiparisDeposu depo, SmtpEpostaGonderici mail)
{
    public void Olustur(SiparisKomutu k) { /* ... */ }
}
```

Nesneler dışarıdan geliyor, yani DI uygulanmış. Ama `SiparisServisi` hâlâ SQL Server ve SMTP'yi tanıyor. Veritabanını değiştirmek bu sınıfı değiştirmeyi gerektirir. DIP ihlali sürüyor.

### DIP var, DI yok

Daha nadir ama mümkün. Soyutlamaya bağlısın, ama nesneyi yine kendin üretiyorsun.

```csharp
// DIP var (arayüze bağlı) — DI yok (kendi new'liyor)
public class SiparisServisi
{
    private readonly ISiparisDeposu _depo = new SqlSiparisDeposu();

    public void Olustur(SiparisKomutu k) => _depo.Kaydet(k);
}
```

Alan tipi arayüz, yani kodun geri kalanı soyutlamayla çalışıyor. Ama seçimi sınıf kendisi yaptığı için testte yerine başka bir şey koyamazsın. Bu kalıba bazen **service locator'ın kardeşi** denir: bağ gizlidir.

### İkisi birlikte

```csharp
// DIP + DI — arayüze bağlı ve dışarıdan alıyor
public class SiparisServisi(ISiparisDeposu depo)
{
    public void Olustur(SiparisKomutu k) => depo.Kaydet(k);
}
```

| Durum | DIP | DI | Test edilebilir mi |
|---|---|---|---|
| `new SqlSiparisDeposu()` gövdede | Hayır | Hayır | Hayır |
| `SqlSiparisDeposu` constructor'dan | Hayır | Evet | Kısmen (sahte üretmek zor) |
| `ISiparisDeposu` alanı, içeride `new` | Evet | Hayır | Hayır |
| `ISiparisDeposu` constructor'dan | Evet | Evet | Evet |

> **Bu benzetme şurada bozulur:** Ruhsat-kamyon benzetmesi ikisini tamamen bağımsız gösterir. Pratikte DIP'i uyguladığın anda DI neredeyse zorunlu hâle gelir: arayüze bağlandın, peki somut nesneyi kim seçecek? Cevap "dışarıdan gelecek" olmak zorundadır, yoksa sınıf içinde yine `new` yazarsın. Yani bağımsızdırlar ama tek yönlü olarak: DIP genelde DI'ı doğurur, DI tek başına DIP'i doğurmaz.

---

## 9. Elle DI Konteyneri: Sihir Yok

> **Benzetme —** Kütüphane kataloğunu düşün. Katalog, kitabın kendisi değildir; "şu konu şu rafta" diyen bir eşleme listesidir. Sen konuyu söylersin, katalog rafı söyler, görevli kitabı getirir. DI konteyneri de bundan ibarettir: "şu arayüz istendiğinde şu sınıfı üret" diyen bir sözlük ve onu okuyan birkaç satır kod.

**Basitçe:** Konteyner, `Tip → Üretici` eşlemesi tutan bir sözlüktür. `builder.Services.AddScoped<IUrunDeposu, SqlUrunDeposu>()` satırının yaptığı şey, bu sözlüğe bir satır eklemektir.

**Teknik olarak:** **IoC container (DI konteyneri)** — Kayıtlı tip eşlemelerine bakarak nesne grafiğini çalışma anında kuran bileşen. Kendi yazacağımız sürüm otuz satır; asıl olanlar bunun üzerine yaşam süresi yönetimi, açık generic desteği ve doğrulama ekler.

### Sözlük tabanlı basit konteyner

```csharp
public class BasitKonteyner
{
    private readonly Dictionary<Type, Func<BasitKonteyner, object>> _kayitlar = new();
    private readonly Dictionary<Type, object> _tekilNesneler = new();

    // Geçici kayıt: her istendiğinde yeni nesne
    public void Kaydet<TArayuz, TUygulama>() where TUygulama : TArayuz
        => _kayitlar[typeof(TArayuz)] = k => k.Olustur(typeof(TUygulama));

    // Hazır nesne kaydı (tekil)
    public void KaydetTekil<TArayuz>(TArayuz nesne)
        => _tekilNesneler[typeof(TArayuz)] = nesne!;

    // Fabrika kaydı: nesnenin nasıl üretileceğini sen söylersin
    public void Kaydet<TArayuz>(Func<BasitKonteyner, object> fabrika)
        => _kayitlar[typeof(TArayuz)] = fabrika;

    public T Coz<T>() => (T)Coz(typeof(T));

    public object Coz(Type tip)
    {
        if (_tekilNesneler.TryGetValue(tip, out var hazir)) return hazir;
        if (_kayitlar.TryGetValue(tip, out var fabrika)) return fabrika(this);
        if (!tip.IsAbstract && !tip.IsInterface) return Olustur(tip);

        throw new InvalidOperationException($"{tip.Name} için kayıt bulunamadı.");
    }

    // İşin özü: constructor parametrelerini özyinelemeli olarak çöz
    private object Olustur(Type tip)
    {
        var ctor = tip.GetConstructors()
                      .OrderByDescending(c => c.GetParameters().Length)
                      .First();

        var argumanlar = ctor.GetParameters()
                             .Select(p => Coz(p.ParameterType))
                             .ToArray();

        return ctor.Invoke(argumanlar);
    }
}
```

Kullanımı:

```csharp
var konteyner = new BasitKonteyner();

konteyner.KaydetTekil<IEpostaGonderici>(new SmtpEpostaGonderici("smtp.sirket.com", 587));
konteyner.Kaydet<ISiparisDeposu>(_ => new SqlSiparisDeposu(baglantiMetni));
konteyner.Kaydet<IStokKontrolu, StokKontrolu>();

// SiparisServisi'nin üç bağımlılığı özyinelemeli olarak çözülür
var servis = konteyner.Coz<SiparisServisi>();
servis.Olustur(komut);
```

`Coz<SiparisServisi>()` çağrısında olanlar sırasıyla:

1. `SiparisServisi` kayıtlı değil, ama somut bir sınıf — doğrudan üretilecek.
2. En çok parametreli constructor seçilir: `(IStokKontrolu, ISiparisDeposu, IEpostaGonderici)`.
3. Her parametre için `Coz` tekrar çağrılır. `IStokKontrolu` → `StokKontrolu`; onun da bağımlılıkları varsa aynı işlem bir kat daha iner.
4. Tüm argümanlar hazır olunca `ctor.Invoke` ile nesne üretilir.

Konteynerin tamamı budur: **bir sözlük, bir özyinelemeli çözücü, biraz reflection.**

### Gerçek konteynerlerin fazladan yaptıkları

| Yetenek | Ne işe yarar |
|---|---|
| Yaşam süresi yönetimi | Singleton / scoped / transient ayrımı ve `IDisposable` temizliği |
| Açık generic desteği | `IRepository<>` → `EfRepository<>` tek satırda |
| Döngüsel bağımlılık tespiti | `A → B → A` durumunda anlaşılır hata |
| Başlangıç doğrulaması | Eksik kayıtları ilk istekte değil, ayağa kalkarken bildirir |
| Çoklu kayıt | `IEnumerable<IOdemeYontemi>` ile tüm uygulamaları enjekte etme |
| Performans | Reflection yerine derlenmiş ifade ağaçları |

> **Not:** Yukarıdaki oyuncak konteyner döngüsel bağımlılıkta `StackOverflowException` ile çöker. Gerçek konteynerler bunu tespit edip anlamlı hata verir. Amacımız mekanizmayı görmekti; üretimde kendi konteynerini yazma.

> **Bu benzetme şurada bozulur:** Kütüphane kataloğunda bir konuya bir raf karşılık gelir; eşleme birebirdir. Konteynerde ise aynı arayüze birden fazla uygulama kaydedilebilir ve hangisinin geleceği kayıt sırasına, yaşam süresine, hatta scope'a bağlıdır. Katalog basit bir eşleme, konteyner ise kurallı bir çözücüdür.

---

## 10. ASP.NET Core'un Yerleşik Konteyneri

> **Benzetme —** Kendi kuyunu kazmakla şehir şebekesine bağlanmak arasındaki fark. Kuyu kazmayı bir kez yaparsan suyun nereden geldiğini anlarsın; ama günlük hayatta musluğu açarsın. Önceki bölümde kuyuyu kazdık, şimdi muslukta ne olduğunu görüyoruz.

**Basitçe:** ASP.NET Core kendi DI konteynerini getirir. Ayrı bir paket kurman gerekmez. `Program.cs`'te kayıtları yaparsın, geri kalanı framework halleder.

**Teknik olarak:** `Microsoft.Extensions.DependencyInjection` — .NET'in yerleşik konteyneri. Kayıtlar `IServiceCollection` üzerinde toplanır, `Build()` ile bir `IServiceProvider`'a dönüşür.

```csharp
// Program.cs — composition root
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

// Kendi servislerin
builder.Services.AddScoped<ISiparisDeposu, SqlSiparisDeposu>();
builder.Services.AddScoped<IStokKontrolu, StokKontrolu>();
builder.Services.AddSingleton<IEpostaGonderici, SmtpEpostaGonderici>();
builder.Services.AddTransient<IFaturaNumaratoru, FaturaNumaratoru>();

// Aynı arayüzün birden fazla uygulaması — IEnumerable<IOdemeYontemi> olarak enjekte edilir
builder.Services.AddScoped<IOdemeYontemi, KrediKartiOdeme>();
builder.Services.AddScoped<IOdemeYontemi, HavaleOdeme>();

var app = builder.Build();
```

Controller tarafında hiçbir şey yapman gerekmez; framework constructor'a bakar ve kayıtlı tipleri kendisi verir.

```csharp
public class SiparisController(ISiparisServisi servis) : Controller
{
    [HttpPost]
    public async Task<IActionResult> Olustur(SiparisViewModel model) { /* ... */ }
}
```

### Üç yaşam süresi

| Kayıt | Ömür | Tipik kullanım |
|---|---|---|
| `AddTransient` | Her istendiğinde yeni nesne | Hafif, durumsuz yardımcılar |
| `AddScoped` | HTTP isteği başına bir nesne | `DbContext`, repository, servis |
| `AddSingleton` | Uygulama ömrü boyunca tek nesne | Yapılandırma, önbellek, HTTP istemci fabrikası |

> **Sık yapılan hata — captive dependency:** Bir singleton'a scoped bir servis enjekte edersen, scoped nesne singleton'ın ömrüne hapsolur. `DbContext` bir singleton'a girerse tüm uygulama boyunca aynı kalır: change tracker şişer, eşzamanlı isteklerde hata verir. .NET 8'de geliştirme ortamında konteyner bunu ayağa kalkarken yakalar (`ValidateScopes`), ama yayın yapılandırmasında sessiz kalabilir.

`AddDbContext` varsayılan olarak **scoped** kaydeder — bu bilinçli bir tercihtir, değiştirmeden önce sebebini bil.

```csharp
builder.Services.AddDbContext<MvcCvContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Varsayilan")));
```

> **MvcCv notu:** Projendeki controller'lar repository'yi doğrudan `new`'liyorsa (`var repo = new GenericRepository<Cv>();`), orada hem DIP hem DI eksiktir. Tek bir hamleyle ikisi birden kazanılır: `GenericRepository` bir arayüzün arkasına alınır, `Program.cs`'te `AddScoped` ile kaydedilir, controller constructor'dan alır. `DbContext`'in de aynı isteğin içinde paylaşılması gerektiği için scoped kayıt burada özellikle önemlidir.

> **Köprü:** Pipeline'ın tamamı, `Program.cs`'in yapısı ve middleware sırası **Hafta 4'teki `02-Program-cs-ve-Pipeline.md`** notunun konusu. Bu bölüm sadece DI kısmına değindi; orada `builder`, `app`, middleware zinciri ve yapılandırma birlikte işleniyor.

> **Bu benzetme şurada bozulur:** Şebeke suyu benzetmesinde musluğu açan kişi suyun nereden geldiğini hiç bilmek zorunda değildir. Kodda ise bilmek zorundasın: yaşam süresini yanlış seçtiğinde ortaya çıkan hatalar (captive dependency, `DbContext` paylaşımı, eşzamanlılık) doğrudan bu seçimden doğar. Konteyner mekanizmayı gizler, ama sonuçlarını gizlemez.

---

## 11. Test Edilebilirlik: DI'ın Asıl Kazancı

> **Benzetme —** Bir oto tamircisinde arıza tespiti yapılırken, motoru aracın içinde çalıştırmak yerine test tezgâhına bağlarlar. Tezgâhta yakıt, elektrik ve soğutma sahte olarak sağlanır; motor kendi başına incelenir. DI, sınıfını test tezgâhına bağlayabilmenin yoludur. Bağımlılıklar dışarıdan geliyorsa, testte yerlerine sahtelerini koyarsın.

**Basitçe:** "Bu kodu neden test edemiyorum" sorusunun cevabı neredeyse her zaman aynıdır: kod kendi bağımlılığını kendi üretiyor. DI'ı uyguladığın anda test yazılabilir hâle gelir.

**Teknik olarak:** İki terim ayrılır.

**Stub** — Önceden belirlenmiş cevabı döndüren sahte nesne. "Bu çağrıda şunu döndür" dersin, sonucu kontrol edersin.
**Mock** — Çağrının **yapılıp yapılmadığını** doğrulamak için kullanılan sahte nesne. "Bu metot bir kez çağrıldı mı" dersin.

### Test edilemeyen kod

```csharp
// TEST EDİLEMEZ — üç bağımlılık da gövdede üretiliyor
public class AboneligYenileyici
{
    public bool Yenile(int aboneId)
    {
        var depo = new SqlAboneDeposu("Server=.;Database=Prod;...");
        var abone = depo.Getir(aboneId);

        if (abone.BitisTarihi > DateTime.Now) return false;   // gerçek saate bağlı

        abone.BitisTarihi = DateTime.Now.AddYears(1);
        depo.Guncelle(abone);

        new SmtpEpostaGonderici("smtp.sirket.com", 587)
            .Gonder(abone.Eposta, "Aboneliğiniz yenilendi");
        return true;
    }
}
```

Bu metodu test etmek için gerçek bir SQL Server, gerçek bir SMTP sunucusu ve doğru tarihi beklemek gerekir. Test değil, entegrasyon çilesi.

### Test edilebilir hâli

```csharp
public class AboneligYenileyici(
    IAboneDeposu depo,
    IEpostaGonderici mail,
    TimeProvider saat)
{
    public bool Yenile(int aboneId)
    {
        var abone = depo.Getir(aboneId);
        var simdi = saat.GetUtcNow().UtcDateTime;

        if (abone.BitisTarihi > simdi) return false;

        abone.BitisTarihi = simdi.AddYears(1);
        depo.Guncelle(abone);
        mail.Gonder(abone.Eposta, "Aboneliğiniz yenilendi");
        return true;
    }
}
```

### Elle yazılmış fake ile test

Kütüphane bile gerekmez; arayüzü uygulayan küçük bir sınıf yeterlidir.

```csharp
// Bellek içi sahte depo
public class SahteAboneDeposu : IAboneDeposu
{
    private readonly Dictionary<int, Abone> _veri = new();
    public Abone? Guncellenen { get; private set; }

    public void Ekle(Abone a) => _veri[a.Id] = a;
    public Abone Getir(int id) => _veri[id];
    public void Guncelle(Abone a) { _veri[a.Id] = a; Guncellenen = a; }
}

// Gönderilen postaları biriktiren sahte
public class SahteEpostaGonderici : IEpostaGonderici
{
    public List<(string Adres, string Konu)> Gonderilenler { get; } = new();
    public void Gonder(string adres, string konu) => Gonderilenler.Add((adres, konu));
}
```

Test:

```csharp
[Fact]
public void Suresi_Dolmus_Abonelik_Bir_Yil_Uzatilir_Ve_Mail_Atilir()
{
    // Arrange — bağımlılıkların hepsi sahte, hiçbiri dış dünyaya çıkmıyor
    var simdi = new DateTimeOffset(2026, 9, 23, 0, 0, 0, TimeSpan.Zero);
    var saat = new FakeTimeProvider(simdi);

    var depo = new SahteAboneDeposu();
    depo.Ekle(new Abone { Id = 7, Eposta = "umut@ornek.com",
                          BitisTarihi = new DateTime(2026, 8, 1) });

    var mail = new SahteEpostaGonderici();
    var yenileyici = new AboneligYenileyici(depo, mail, saat);

    // Act
    var sonuc = yenileyici.Yenile(7);

    // Assert
    Assert.True(sonuc);
    Assert.Equal(new DateTime(2027, 9, 23), depo.Guncellenen!.BitisTarihi);
    Assert.Single(mail.Gonderilenler);
    Assert.Equal("umut@ornek.com", mail.Gonderilenler[0].Adres);
}
```

Test milisaniyeler içinde çalışır, ağa çıkmaz, veritabanı istemez ve her makinede aynı sonucu verir. Bunu mümkün kılan tek şey, bağımlılıkların dışarıdan veriliyor olmasıdır.

> **Not:** Aynı sahteleri Moq gibi bir kütüphaneyle tek satırda üretebilirsin (`Mock.Of<IAboneDeposu>()`). Elle yazılmış sahteler, davranış biriktirmesi gerektiğinde (yukarıdaki `Gonderilenler` listesi gibi) çoğu zaman daha okunaklı kalır. Bootcamp'te muhtemelen ikisini de göreceksin.

> **Bu benzetme şurada bozulur:** Test tezgâhı benzetmesi, tezgâhta geçen motorun araçta da çalışacağını ima eder. Kodda bu garanti yoktur. Sahtelerle yazılan birim testi, **senin varsayımlarını** doğrular; gerçek veritabanının o sorguyu nasıl çalıştırdığını değil. Bu yüzden birim testi entegrasyon testinin yerine geçmez. DI birim testini mümkün kılar, tek başına doğruluğu garanti etmez.

---

## Tek Bakışta Özet

- ISP, arayüzü **kullananı** korur: kimse kullanmadığı metoda bağlı kalmamalı.
- Şişman arayüzün belirtisi boş gövde ve `NotSupportedException`'dır; aynı anda LSP ihlalidir.
- Rol arayüzü tek bir yeteneği tanımlar; bir sınıf birden fazla rolü üstlenebilir.
- En pratik bölme okuma/yazma ayrımıdır: `IUrunOkuyucu` verirsen o kod veri silemez.
- ISP de abartılır: her metoda ayrı arayüz açmak bölme değil gürültüdür.
- DIP: üst seviye alt seviyeye değil, **ikisi de soyutlamaya** bağlanır.
- Soyutlama **tüketicinin** katmanında tanımlanır; arayüzü sağlayıcının yanına koyarsan ok tersine dönmez.
- İyi bir arayüz arkasındaki teknolojiyi sızdırmaz — `SqlDataReader` döndüren arayüz kötü arayüzdür.
- DI'ın üç yolu: constructor (varsayılan), property (gerçekten opsiyonel olan), method (çağrı başına değişen).
- Constructor parametre sayısı bedava bir SRP ölçüsüdür; dörtten fazlaysa sınıfa tekrar bak.
- `new`, composition root'ta toplanır: `Program.cs` ya da `Main`. DTO ve varlık `new`'lemek serbesttir.
- Service locator bağımlılığı gizler, hatayı çalışma anına erteler ve kötü tasarımın acısını yok eder.
- DIP ile DI aynı şey değildir: DIP "neye bağlısın", DI "nasıl teslim edildi" sorusudur.
- Somut tipi constructor'dan almak DI'dır ama DIP değildir — en sık görülen yarım uygulama budur.
- Konteyner sihir değil: bir sözlük, bir özyinelemeli çözücü ve biraz reflection.
- ASP.NET Core'da üç yaşam süresi vardır; singleton'a scoped enjekte etmek captive dependency üretir.
- DI'ın asıl kazancı testtir: bağımlılık dışarıdan geliyorsa yerine sahtesini koyabilirsin.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| ISP | Kimse kullanmadığı metoda bağlı kalmamalı |
| Role interface | Tek bir yeteneği tanımlayan küçük arayüz |
| Header interface | Somut sınıfın tüm metotlarını kopyalayan arayüz |
| DIP | Üst ve alt seviyenin ortak bir soyutlamaya bağlanması |
| Üst seviye modül | İş kuralını, politikayı taşıyan kod |
| Alt seviye modül | Mekanizmayı taşıyan kod (SQL, SMTP, dosya) |
| IoC | Kontrolün çağıran taraftan framework'e/dışarıya geçmesi |
| DI | Bağımlılığın nesneye dışarıdan teslim edilmesi |
| Constructor injection | Bağımlılığın constructor parametresiyle verilmesi |
| Property injection | İsteğe bağlı bağımlılığın property ile verilmesi |
| Method injection | Bağımlılığın metot parametresiyle verilmesi |
| Composition root | Nesne grafiğinin kurulduğu tek nokta |
| Service locator | Bağımlılığın merkezi bir sağlayıcıdan çalışma anında istenmesi |
| IoC container | Kayıtlara bakarak nesne grafiğini kuran bileşen |
| Transient / Scoped / Singleton | Her istekte yeni / istek başına bir / uygulama boyunca tek |
| Captive dependency | Kısa ömürlü servisin uzun ömürlüye hapsolması |
| Stub | Önceden belirlenmiş cevabı döndüren sahte nesne |
| Mock | Çağrının yapıldığını doğrulamak için kullanılan sahte nesne |
| Null object | Hiçbir şey yapmayan güvenli varsayılan uygulama |
| Onion / Clean Architecture | Bağımlılıkların içeriye, iş kuralına doğru aktığı katman düzeni |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "DIP ile DI aynı şeydir" | DIP tasarım ilkesi, DI teslim tekniğidir |
| "Constructor'dan alıyorsam DIP'e uyuyorum" | Somut tip alıyorsan DI var, DIP yok |
| "DI demek konteyner demektir" | Konteyner sadece kolaylıktır; elle DI da DI'dır |
| "Arayüz veri erişim projesinde durur" | Arayüz tüketicinin katmanında tanımlanır |
| "Her sınıfın arayüzü olmalı" | Dış dünyaya dokunan ve değişmesi muhtemel olanların |
| "`IServiceProvider` enjekte etmek DI'dır" | Service locator'dır; bağımlılığı gizler |
| "`new` yazmak yasaktır" | DTO, varlık ve koleksiyon her yerde `new`'lenir |
| "ISP arayüzü uygulayanı korur" | Asıl koruduğu arayüzü **kullanan** taraftır |
| "Singleton en performanslı seçenektir" | Scoped servisi içine alırsa captive dependency üretir |
| "`AddDbContext` singleton yapılabilir" | Scoped olmalı; paylaşılan `DbContext` eşzamanlılıkta patlar |
| "Test için mutlaka Moq gerekir" | Arayüzü uygulayan küçük bir sınıf çoğu zaman yeter |
| "Birim testi geçtiyse kod doğrudur" | Sahteler varsayımını doğrular, gerçek altyapıyı değil |

---

## Sonraki

→ `05-Creational-Patterns.md` (Cuma)
