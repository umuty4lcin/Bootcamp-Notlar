# Hafta 2 · Salı — Kalıtım Yerine Kompozisyon

**Okuma süresi:** ~45 dk
**Neden bu konu:** Kalıtım, C# öğrenirken ilk öğretilen ve en çok kötüye kullanılan araçtır. Bootcamp'te göreceğin tasarım kalıplarının çoğu (Strategy, Decorator, Adapter) aslında "burada kalıtım yerine kompozisyon kullan" demenin adıdır. MvcCv'deki `GenericRepository` ve controller'ların repository'ye bağlanma biçimi de tam olarak bu tercihin üzerine kurulu.

---

## Önce Basitçe

Bir ev yaptırıyorsun. İki yol var. Birinci yolda hazır bir ev planı alırsın ve "bu planın üzerine bir oda ekleyeyim" dersin. Plan kimin ise, o kişi yarın planı değiştirirse senin evin de değişir. Duvarı kaldırırsa senin odan havada kalır. İkinci yolda ise evi parçalardan kurarsın: kapıyı bir yerden, pencereyi başka yerden alırsın. Beğenmediğin parçayı söküp yerine başkasını takarsın, evin geri kalanına dokunmadan.

Kodda birinci yol kalıtım, ikinci yol kompozisyondur. Kalıtım "ben onun bir türüyüm" der; kompozisyon "onu kullanıyorum" der. İkisi de kod paylaşmanı sağlar, ama bağın sıkılığı farklıdır. Kalıtımda taban sınıf değişince türeyen sınıf değişir — sen hiçbir şey yazmasan bile. Kompozisyonda ise aradaki bağ, senin çağırdığın metotların imzasıyla sınırlıdır.

Kalıtımın asıl bedeli görünmez olmasıdır. Kod ilk yazıldığında her şey temiz durur: ortak alanlar tek yerde, tekrar yok. Sorun altı ay sonra çıkar. Taban sınıfa yeni bir metot eklenir ve türeyenlerden biri o isimde zaten bir metot barındırıyordur. Ya da taban sınıfın bir metodu içeride kendi başka bir metodunu çağırıyordur ve sen o ikinciyi ezmişsindir — bir anda beklemediğin bir sıra ortaya çıkar. Kimse yanlış bir şey yapmamıştır, kod yine de yanlış çalışır.

İkinci mesele söz meselesidir. Kalıtım kurduğunda kullanıcıya bir söz verirsin: "nerede taban tip kabul ediliyorsa oraya benim tipimi de koyabilirsin." Bu sözü tutamıyorsan kalıtım yanlış araçtır. Kuş sınıfına `Uc()` koyup penguen türettiğinde sözü tutmamış olursun. Bu notun ortasındaki bölüm bu sözün adını koyuyor: Liskov Yerine Geçme Prensibi.

Ama kalıtım yasak değildir. Framework tiplerinden türetmek (`Controller`, `DbContext`, `Exception`), akışı sabit tutup sadece bir adımı değiştirmek gibi yerlerde doğru araçtır. Notun sonunda bu meşru kullanımları ayrı ayrı göreceksin. Ölçüt hep aynı: **sözü tutabiliyor musun, taban sınıfın değişmesine dayanabilir misin.** Şimdi detaya iniyoruz.

> **Ana benzetme:** Kalıtım, birinin **çırağı olmaktır**: ustanın yöntemi seninkidir, usta yöntemini değiştirirse seninki de değişir, başka bir ustaya aynı anda çırak olamazsın. Kompozisyon ise **taşeronla çalışmaktır**: işin bir parçasını ona verirsin, sözleşmede ne yazıyorsa onu istersin, beğenmezsen taşeronu değiştirirsin ve binan aynı kalır.

---

## Bu Notta Ne Var

1. Kalıtımın gerçek maliyeti: sıkı bağ
2. Kırılgan taban sınıf problemi
3. "is-a" mı "has-a" mı — karar testi
4. Kompozisyon ve delegasyon: nasıl kurulur
5. Derin hiyerarşilerin sorunu
6. LSP ihlali: söz verip tutmamak
7. Davranışı parametre olarak geçirmek
8. `sealed` — ne zaman, neden performans da kazandırır
9. Extension method ile davranış eklemek
10. Küçük arayüzler ve default interface method'lar
11. Kalıtımın hâlâ doğru olduğu durumlar

---

## 1. Kalıtımın Gerçek Maliyeti: Sıkı Bağ

> **Benzetme —** Bir apartmanda üst kattaki komşunun su tesisatı senin tavanından geçiyor. O adam banyosunu yeniletirken senin tavanını da açmak zorunda. Sen bir şey yapmadın, kararı o verdi, masraf sana da çıktı. Kalıtım tam olarak budur: taban sınıfın kararları türeyen sınıfa fatura edilir.

**Basitçe:** Kalıtım, iki sınıf arasında kurabileceğin **en sıkı** bağdır. Türeyen sınıf, taban sınıfın sadece `public` yüzeyine değil, `protected` iç detaylarına ve metotları birbirini çağırma sırasına da bağımlıdır.

**Teknik olarak:** Bağın sıkılığını ölçmenin pratik yolu, "taban sınıfın neyini değiştirirsem türeyeni bozarım?" sorusudur.

| Değişiklik | Kompozisyonda | Kalıtımda |
|---|---|---|
| `private` bir alanın adı değişti | Etkilenmez | Etkilenmez |
| `protected` bir metot kaldırıldı | Etkilenmez (görünmüyordu) | **Bozar** |
| Bir metodun iç çağrı sırası değişti | Etkilenmez | **Bozabilir** |
| Yeni bir `public` metot eklendi | Etkilenmez | **Ad çakışması yaratabilir** |
| `virtual` bir metot `sealed` yapıldı | Etkilenmez | **Bozar** |
| Sınıf `sealed` yapıldı | Etkilenmez | **Bozar** |

Kompozisyonda bağ yalnızca ilk satırdadır: çağırdığın metodun imzası. Kalıtımda bağ altı satırın tamamıdır.

```csharp
// Kalıtım: Bildirim, Loglayici'nin TÜM korumalı yüzeyine bağımlı
public class Loglayici
{
    protected string Bicimle(string m) => $"[{DateTime.UtcNow:O}] {m}";
    public void Yaz(string m) => Console.WriteLine(Bicimle(m));
}

public class BildirimGonderici : Loglayici
{
    public void Gonder(string mesaj)
    {
        Yaz(Bicimle(mesaj));   // Bicimle korumalı; kaldırılırsa bu sınıf derlenmez
    }
}
```

```csharp
// Kompozisyon: bağ tek satırlık bir sözleşme
public interface ILoglayici { void Yaz(string mesaj); }

public sealed class BildirimGonderici2
{
    private readonly ILoglayici _log;
    public BildirimGonderici2(ILoglayici log) => _log = log;

    public void Gonder(string mesaj) => _log.Yaz(mesaj);
}
```

İkinci sürümde `Loglayici`'nin içini tamamen yeniden yazabilirsin; `BildirimGonderici2` etkilenmez. Üstelik testte sahte bir `ILoglayici` verip çıktıyı doğrulayabilirsin.

### Kalıtım yüzeyi genişletir

Kalıtımın az konuşulan bir maliyeti daha var: türeyen sınıf, taban sınıfın **bütün public üyelerini** de dışarıya açar.

```csharp
public class OnbellekliSozluk : Dictionary<string, string>
{
    public void EkleVeSure(string k, string v, TimeSpan sure) { /* ... */ }
}

var c = new OnbellekliSozluk();
c.Add("a", "1");   // süre yok — önbellek kuralı delindi
c.Clear();         // bütün önbellek tek satırda gitti
```

Sen tek metot eklemek istedin, dışarıya kırk metot açtın. Bu kırk metodun hepsinin senin kuralınla uyumlu olduğunu garanti edemezsin.

> **Bu benzetme şurada bozulur:** Tesisat benzetmesi taban sınıfın "kötü niyetli" davrandığını ima ediyor. Gerçekte taban sınıfın yazarı genellikle **senin varlığından haberdar bile değildir**. Bir NuGet paketinin sürümünü yükselttiğinde bozulan şey, kimsenin hata yapmadığı bir değişiklikten doğar. Sorun kötü niyet değil, görünmeyen bağdır.

---

## 2. Kırılgan Taban Sınıf Problemi

> **Benzetme —** Lokantada merkez mutfak şubelere "çorbayı şöyle yap" diye tarif yollar. Bir şube, tarifin "tuz ekle" adımını kendi usulüne çevirmiştir. Merkez bir gün tarife "servis öncesi bir kez daha tuz kontrolü yap" satırı ekler ve o satır da şubenin tuz adımını çağırır. Şube hiçbir şey değiştirmedi ama çorbası artık iki kez tuzlanıyor.

**Basitçe:** Taban sınıftaki masum görünen bir değişiklik, türeyen sınıfları sessizce bozar. Buna **fragile base class (kırılgan taban sınıf) problemi** denir. Derleyici yakalamaz, testler yazılmamışsa kimse fark etmez.

**Teknik olarak:** Sorunun kaynağı, taban sınıfın kendi metotları arasındaki çağrıların türeyen sınıf için **yayınlanmamış bir sözleşme** oluşturmasıdır.

```csharp
// Taban sınıf — v1
public class Koleksiyon
{
    private readonly List<string> _ogeler = new();

    public virtual void Ekle(string oge) => _ogeler.Add(oge);

    public virtual void TopluEkle(IEnumerable<string> ogeler)
    {
        foreach (var o in ogeler) _ogeler.Add(o);   // Ekle'yi ÇAĞIRMIYOR
    }

    public int Adet => _ogeler.Count;
}

// Türeyen sınıf — sayaç tutmak istiyor
public class SayanKoleksiyon : Koleksiyon
{
    public int EklemeSayisi { get; private set; }

    public override void Ekle(string oge)
    {
        EklemeSayisi++;
        base.Ekle(oge);
    }

    public override void TopluEkle(IEnumerable<string> ogeler)
    {
        EklemeSayisi += ogeler.Count();
        base.TopluEkle(ogeler);
    }
}
```

Bu kod v1'de doğru çalışır. Şimdi taban sınıfın yazarı tekrarı temizlesin:

```csharp
// Taban sınıf — v2. Görünüşte zararsız bir sadeleştirme.
public virtual void TopluEkle(IEnumerable<string> ogeler)
{
    foreach (var o in ogeler) Ekle(o);   // artık Ekle'yi çağırıyor
}
```

`SayanKoleksiyon.TopluEkle` artık sayacı **iki kez** artırır: bir kez kendi gövdesinde, bir kez de `base.TopluEkle` içindeki her `Ekle` çağrısında. Türeyen sınıfa hiç dokunulmadı, derleme uyarı vermedi, kod sessizce yanlış çalışmaya başladı.

```csharp
// İYİ — kompozisyon: içerideki çağrı sırası dışarıyı ilgilendirmez
public sealed class SayanKoleksiyon2
{
    private readonly Koleksiyon _ic = new();
    public int EklemeSayisi { get; private set; }

    public void Ekle(string oge)
    {
        EklemeSayisi++;
        _ic.Ekle(oge);
    }

    public void TopluEkle(IEnumerable<string> ogeler)
    {
        foreach (var o in ogeler) Ekle(o);   // sayma kuralı BURADA, tek yerde
    }

    public int Adet => _ic.Adet;
}
```

Kompozisyon sürümünde `Koleksiyon`'un içinde ne olduğu önemsizdir. Sayma kuralı tamamen senin sınıfının içinde durur.

> Bu problem teorik değil. .NET tarihinde `Hashtable`'dan türeyen sınıflar, `System.Web`'deki bazı taban sınıflar ve pek çok kütüphane bu yüzden sürüm değiştirirken davranış kırdı. Bugün .NET ekibinin `public` sınıfları varsayılan olarak `sealed` işaretlemesinin sebebi budur.

### Korunmanın yolu: sözleşmeyi yazılı hâle getirmek

Kalıtıma izin vereceksen, türeyenlere **ne söz verdiğini** yazmak zorundasın: hangi adımın ne zaman çağrılacağını, hangi metodun içeriden başka bir sanal metot çağırıp çağırmadığını belgele ve akışı tutan metodu `virtual` yapma. 11. bölümdeki template method örneği bunun kalıplaşmış hâlidir. Bu, kırılganlığı ortadan kaldırmaz ama sınırlar.

> **Bu benzetme şurada bozulur:** Merkez mutfak benzetmesinde şube, merkezin tarifini okuyabiliyor. Kodda genellikle okuyamazsın: derlenmiş bir kütüphanenin içindeki çağrı sırası belgelenmemiştir. Belgelenmiş olsa bile bir sonraki sürümde değişmeyeceğinin garantisi yoktur. Kalıtım, **yazılı olmayan bir sözleşmeye** güvenmektir.

---

## 3. "is-a" mı "has-a" mı — Karar Testi

> **Benzetme —** Muhtarlıkta iki ayrı belge alırsın: ikametgâh ve vekâletname. İkametgâh "sen burada oturan birisin" der — bir **kimlik** beyanıdır. Vekâletname ise "senin adına şu kişi iş yapacak" der — bir **kullanım** ilişkisidir. İkisini karıştırırsan yanlış kapıya gidersin.

**Basitçe:** "B bir A mıdır?" sorusu doğruysa kalıtım düşünülebilir. "B'nin içinde bir A var mıdır / B bir A kullanır mı?" doğruysa kompozisyon kur. Şüphedeysen kompozisyon.

**Teknik olarak:** "is-a" testi tek başına yetersizdir, çünkü günlük dilde doğru olan cümle kodda yanlış olabilir. Üç ek soru sor:

1. **Yerine geçebilir mi?** Taban tip bekleyen her yere bu tipi koyabilir misin, sürpriz olmadan?
2. **Taban sınıfın tüm public üyeleri bu tip için anlamlı mı?** Biri anlamsızsa yüzey yanlış.
3. **İlişki ömür boyu sabit mi?** Çalışma zamanında değişiyorsa kalıtım kuramazsın — kompozisyon gerekir.

```csharp
// "Çalışan bir kişidir" — is-a doğru, yüzey uyumlu, ömür boyu sabit
public class Kisi { public string Ad { get; set; } = ""; }
public class Calisan : Kisi { public decimal Maas { get; set; } }
```

```csharp
// KÖTÜ — "Müşteri bir kullanıcıdır" ilişkisi çalışma zamanında değişebilir
public class Kullanici { }
public class Musteri : Kullanici { }
public class Personel : Kullanici { }
// Aynı kişi hem müşteri hem personel olamaz — kalıtım bunu yasaklar
```

```csharp
// İYİ — roller kompozisyonla tutulur, çalışma zamanında değişebilir
public sealed class Kullanici
{
    private readonly HashSet<string> _roller = new();
    public IReadOnlySet<string> Roller => _roller;
    public bool RolVar(string rol) => _roller.Contains(rol);
    public void RolEkle(string rol) => _roller.Add(rol);
}
```

Bir sınıfın çalışma zamanında tipini değiştiremezsin. Bir nesnenin sahip olduğu parçaları ise istediğin zaman değiştirebilirsin. Bu, kompozisyonun en somut üstünlüğüdür.

### Pratik ayırt etme tablosu

| Cümle | İlişki | Doğru araç |
|---|---|---|
| "Vadeli hesap bir hesaptır" | is-a | Kalıtım (LSP sağlanıyorsa) |
| "Siparişin bir adresi vardır" | has-a | Kompozisyon |
| "Controller bir repository kullanır" | uses-a | Kompozisyon + DI |
| "Admin bir kullanıcıdır" | is-a ama değişken | Kompozisyon (rol) |
| "Önbellekli repository bir repository'dir" | is-a **ve** has-a | Kompozisyon (Decorator) |
| "Kare bir dikdörtgendir" | matematikte is-a, kodda değil | Kompozisyon / ayrı tipler |

Son iki satır ilginç. "Önbellekli repository bir repository'dir" cümlesi doğrudur — aynı arayüzü uygular. Ama gerçeklemeyi devralmaz, **sarmalar**:

```csharp
public sealed class OnbellekliDeneyimRepository : IDeneyimRepository
{
    private readonly IDeneyimRepository _ic;      // sarmalanan
    private readonly IMemoryCache _cache;

    public OnbellekliDeneyimRepository(IDeneyimRepository ic, IMemoryCache cache)
    {
        _ic = ic; _cache = cache;
    }

    public List<TblDeneyimlerim> List()
        => _cache.GetOrCreate("deneyim-list", e =>
        {
            e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return _ic.List();
        })!;

    public void Insert(TblDeneyimlerim e) { _ic.Insert(e); _cache.Remove("deneyim-list"); }
}
```

Bu, **Decorator** kalıbıdır ve MvcCv'deki `DeneyimController`'a tek satır bile dokunmadan devreye alınabilir: DI kaydında `IDeneyimRepository` için bu sınıfı verirsin. Kalıtımla aynı şeyi yapmak `DeneyimRepository`'nin her metodunu `virtual` yapmayı ve içeriden çağrı sırasına güvenmeyi gerektirirdi.

> **Bu benzetme şurada bozulur:** İkametgâh/vekâletname ayrımı net görünüyor ama kodda çoğu ilişki ikisinin arasındadır. "Yönetici bir çalışandır" cümlesi is-a'dır; ama yöneticilik bir görev olduğu için kişi yarın yönetici olmaktan çıkabilir. Cümlenin doğruluğu değil, **ilişkinin ömrü** karar verdirir.

---

## 4. Kompozisyon ve Delegasyon: Nasıl Kurulur

> **Benzetme —** Bir müteahhit bina yapar ama elektriği elektrikçiye, sıhhi tesisatı tesisatçıya verir. Müşteri müteahhitle konuşur; müteahhit işi ilgili kişiye devreder. Elektrikçiyi değiştirmek binayı yıkmayı gerektirmez. Müşteri elektrikçinin kim olduğunu bilmez bile.

**Basitçe:** Kompozisyon, bir sınıfın işini başka nesnelere yaptırmasıdır. Dışarıya kendi yüzeyini gösterir, arkada işi devreder. Bu devretme işlemine **delegasyon** denir.

**Teknik olarak:** Kurulumun üç adımı var: parçayı alan (field) olarak tut, constructor'dan al, çağrıları ona devret. Parçayı **somut sınıf olarak değil arayüz olarak** almak, kompozisyonun esnekliğini açığa çıkarır.

```csharp
public interface IFiyatHesaplayici { decimal Hesapla(Siparis s); }
public interface IStokKontrol      { bool Yeterli(int urunId, int adet); }

public sealed class SiparisServisi
{
    private readonly IFiyatHesaplayici _fiyat;   // parça 1
    private readonly IStokKontrol _stok;         // parça 2

    public SiparisServisi(IFiyatHesaplayici fiyat, IStokKontrol stok)
    {
        _fiyat = fiyat;
        _stok  = stok;
    }

    public SiparisSonuc Olustur(Siparis siparis)
    {
        if (!_stok.Yeterli(siparis.UrunId, siparis.Adet))
            return SiparisSonuc.StokYok;

        var tutar = _fiyat.Hesapla(siparis);      // delegasyon
        return SiparisSonuc.Basarili(tutar);
    }
}
```

Bu sınıfın neye ihtiyacı olduğu constructor imzasında yazıyor. Testte iki sahte nesne verirsin, veritabanı gerekmez. Fiyatlandırma kuralı değişirse `IFiyatHesaplayici`'nin başka bir gerçeklemesini kaydedersin; `SiparisServisi` hiç değişmez.

### Delegasyonun bedeli: tekrar eden ileti metotları

Kompozisyonun tek gerçek dezavantajı, sarmaladığın yüzeyi elle yeniden yazman gerekmesidir.

```csharp
public sealed class SinirliListe
{
    private readonly List<int> _ic = new();
    private readonly int _enFazla;

    public SinirliListe(int enFazla) => _enFazla = enFazla;

    public int Adet => _ic.Count;                     // ileti
    public int this[int i] => _ic[i];                  // ileti

    public void Ekle(int x)                             // kural burada
    {
        if (_ic.Count >= _enFazla)
            throw new InvalidOperationException($"En fazla {_enFazla} öğe.");
        _ic.Add(x);
    }
}
```

Beş satır fazladan yazdın ama karşılığında `Insert`, `RemoveAt`, `Clear` gibi kuralını delen kırk metodu dışarı açmadın. Bu takas neredeyse her zaman kompozisyonun lehinedir.

### Hangi parça nereden gelir

| Yöntem | Ne zaman |
|---|---|
| Constructor parametresi | Ömür boyu sabit bağımlılık — varsayılan tercih |
| Property (`set`) | İsteğe bağlı, sonradan değişebilen parça (dikkatli kullan) |

```csharp
// Parça metot parametresi olarak — en gevşek bağ
public decimal ToplamHesapla(IEnumerable<Siparis> siparisler, IFiyatHesaplayici hesaplayici)
    => siparisler.Sum(hesaplayici.Hesapla);
```

> Constructor'a beşten fazla bağımlılık koyuyorsan sınıf muhtemelen çok iş yapıyordur. Uzun ctor imzası bir sorun değil, sorunun göstergesidir.

> **Bu benzetme şurada bozulur:** Müteahhit benzetmesi parçaların tamamen bağımsız olduğunu varsayıyor. Kodda parçalar birbirini etkileyebilir: fiyat hesaplayıcı stok durumuna bakmak isteyebilir. O noktada ya parçalardan biri diğerini alır (bağ zinciri uzar) ya da koordinasyonu üstteki sınıf yapar. İkincisi doğrudur; parçaların birbirini tanımaması kompozisyonun değerini korur.

---

## 5. Derin Hiyerarşilerin Sorunu

> **Benzetme —** Bir kurumda beş kademeli onay zinciri. Bir imza için evrak memurdan şefe, şeften müdüre, müdürden genel müdür yardımcısına gidiyor. Kimse tek başına yanlış yapmıyor ama bir evrakın nerede takıldığını anlamak için beş kişiyi tek tek aramak gerekiyor. Üç kademeden sonra kimse zincirin tamamını aklında tutamaz.

**Basitçe:** İki seviyelik kalıtım anlaşılır. Dört seviyede, bir metodun nereden geldiğini bulmak için dört dosya açman gerekir. Beş seviyede kimse hiyerarşinin tamamını bilmez.

**Teknik olarak:** Derinliğin somut maliyetleri:

| Sorun | Belirti |
|---|---|
| Okunabilirlik | Bir metodun gerçek gövdesini bulmak için zincir taranır |
| Kırılganlık | Her ara seviye kendi altındakileri bozabilir |
| Tek kalıtım kilidi | Zincirde yer kapladın; başka bir taban sınıf alamazsın |
| Test | Alttaki sınıfı test etmek için tüm zincirin kurulması gerekir |
| Değişim maliyeti | En üstteki küçük bir değişiklik tüm alt ağacı etkiler |

```csharp
// KÖTÜ — beş kademe. VipMusteriRaporu.Uret() nereden geliyor?
public abstract class Nesne { }
public abstract class Belge : Nesne { }
public abstract class Rapor : Belge { }
public abstract class MusteriRaporu : Rapor { }
public class VipMusteriRaporu : MusteriRaporu { }
```

Ara sınıflardan hiçbiri kendi başına anlamlı değildir; sadece kod paylaşmak için var. Bu, "kod paylaşımı için kalıtım" kokusunun en net hâlidir.

```csharp
// İYİ — tek seviye kalıtım + parçalar kompozisyonla
public interface IRapor { byte[] Uret(RaporIstegi istek); }

public sealed class MusteriRaporu2 : IRapor
{
    private readonly IVeriKaynagi _veri;
    private readonly IBicimlendirici _bicim;

    public MusteriRaporu2(IVeriKaynagi veri, IBicimlendirici bicim)
    {
        _veri = veri; _bicim = bicim;
    }

    public byte[] Uret(RaporIstegi istek) => _bicim.Bicimle(_veri.Getir(istek));
}
```

İkinci sürümde "VIP" davranışı yeni bir sınıf değil, farklı bir `IVeriKaynagi` ya da farklı bir `IBicimlendirici`dir. Kombinasyon sayısı arttıkça kalıtım ağacı patlar, kompozisyon ise sabit kalır.

### Kombinasyon patlaması

Üç özelliği (sıkıştırılmış, şifreli, imzalı) kalıtımla birleştirmeye kalkarsan sekiz sınıf yazman gerekir. Kompozisyonla üç sarmalayıcı yeter:

```csharp
IAkisYazici yazici = new DosyaYazici(yol);
yazici = new SikistiranYazici(yazici);
yazici = new SifreleyenYazici(yazici, anahtar);
yazici = new ImzalayanYazici(yazici, sertifika);
```

Sıra değiştirmek bile ücretsizdir. Bu, .NET'in `Stream` tiplerinin (`GZipStream`, `CryptoStream`, `BufferedStream`) çalışma biçimidir.

> Pratik ölçüt: **kalıtım derinliğin üçü geçiyorsa dur** (`object` sayılmaz).

> **Bu benzetme şurada bozulur:** Onay zinciri benzetmesi her kademenin bir iş yaptığını varsayıyor. Kalıtım zincirinde ara sınıfların çoğu hiçbir iş yapmaz — sadece "buraya bir ara sınıf lazım olabilir" diye vardır. İş yapmayan ara sınıf, maliyeti olan ama faydası olmayan saf yüktür.

---

## 6. LSP İhlali: Söz Verip Tutmamak

> **Benzetme —** Nöbetçi eczane listesi. Listede yazan her eczane, gece gidildiğinde açık olmak zorundadır. Bir eczane listeye adını yazdırıp gece kapalı kalırsa, sorun o eczanenin değil **listenin** sorunudur: artık listeye kimse güvenemez. Alt tip, üst tipin listesine adını yazdırmaktır.

**Basitçe:** **Liskov Substitution Principle (Yerine Geçme Prensibi)** şunu söyler: taban tip bekleyen her yere alt tipi koyabilmelisin ve program doğru çalışmaya devam etmeli. Alt tip, üst tipin verdiği sözü daraltamaz.

**Teknik olarak:** İhlal dört biçimde olur:

1. Alt tip, üst tipin kabul ettiği bir girdiyi reddeder (ön koşulu sıkılaştırır).
2. Alt tip, üst tipin garanti ettiği bir sonucu vermez (son koşulu gevşetir).
3. Alt tip, üst tipin değişmezini (invariant) bozar.
4. Alt tip, üst tipin atmadığı bir istisnayı atar.

### Klasik örnek: Kare / Dikdörtgen

```csharp
// KÖTÜ — matematikte kare bir dikdörtgendir, kodda değil
public class Dikdortgen
{
    public virtual int Genislik { get; set; }
    public virtual int Yukseklik { get; set; }
    public int Alan => Genislik * Yukseklik;
}

public class Kare : Dikdortgen
{
    public override int Genislik  { set { base.Genislik = base.Yukseklik = value; } }
    public override int Yukseklik { set { base.Genislik = base.Yukseklik = value; } }
}

// Dikdörtgen bekleyen kod
void Test(Dikdortgen d)
{
    d.Genislik = 5;
    d.Yukseklik = 4;
    Console.WriteLine(d.Alan);   // Dikdortgen -> 20, Kare -> 16
}
```

`Test` metodu yanlış yazılmadı. `Kare` sınıfı da kendi içinde tutarlı. Ama ikisi bir araya gelince beklenti bozuldu — çünkü `Dikdortgen`'in yayınlanmamış bir sözü vardı: "genişliği değiştirmek yüksekliği değiştirmez."

```csharp
// İYİ — ortak kavram paylaşılıyor, kalıtım kurulmuyor
public interface ISekil { int Alan { get; } }

public sealed class Dikdortgen2(int genislik, int yukseklik) : ISekil
{
    public int Genislik { get; } = genislik;
    public int Yukseklik { get; } = yukseklik;
    public int Alan => Genislik * Yukseklik;
}

public sealed class Kare2(int kenar) : ISekil
{
    public int Kenar { get; } = kenar;
    public int Alan => Kenar * Kenar;
}
```

Değişmezlik (`init`/`get`-only) tek başına sorunun yarısını çözdü: değiştirilemeyen bir nesnede "değiştirince ne olur" sorusu yoktur.

### Gerçekçi örnek: Hesap / VadesizHesap / VadeliHesap

```csharp
// KÖTÜ — VadeliHesap, taban sınıfın kabul ettiği çağrıyı reddediyor (ön koşul sıkılaştı)
public class Hesap
{
    public decimal Bakiye { get; protected set; }
    public virtual void ParaCek(decimal tutar)
    {
        if (tutar > Bakiye) throw new InvalidOperationException("Yetersiz bakiye.");
        Bakiye -= tutar;
    }
}

public class VadeliHesap : Hesap
{
    public DateOnly VadeSonu { get; init; }

    public override void ParaCek(decimal tutar)
    {
        if (DateOnly.FromDateTime(DateTime.Today) < VadeSonu)
            throw new InvalidOperationException("Vade dolmadan para çekilemez.");
        base.ParaCek(tutar);
    }
}
```

Şimdi şu metoda bak:

```csharp
void MaasOde(Hesap hesap, decimal tutar)
{
    if (hesap.Bakiye >= tutar)
        hesap.ParaCek(tutar);      // VadeliHesap gelirse patlar
}
```

`MaasOde` doğru yazılmış. Bakiyeyi kontrol etti, sonra çekti. Ama `VadeliHesap` taban sınıfın sözünü daralttı: "bakiye yeterse çekilir" sözü artık geçerli değil.

```csharp
// İYİ — çekilebilirlik sözleşmenin parçası; ortak yüzey dürüst
public abstract class Hesap2
{
    public decimal Bakiye { get; protected set; }

    public abstract bool CekilebilirMi(decimal tutar);

    public void ParaCek(decimal tutar)
    {
        if (!CekilebilirMi(tutar))
            throw new InvalidOperationException("Bu hesaptan şu anda para çekilemez.");
        Bakiye -= tutar;
    }
}

public sealed class VadesizHesap2 : Hesap2
{
    public override bool CekilebilirMi(decimal tutar) => tutar > 0 && tutar <= Bakiye;
}

public sealed class VadeliHesap2 : Hesap2
{
    public DateOnly VadeSonu { get; init; }
    public override bool CekilebilirMi(decimal tutar)
        => tutar > 0 && tutar <= Bakiye
           && DateOnly.FromDateTime(DateTime.Today) >= VadeSonu;
}
```

Çağıran taraf artık dürüst bir yüzeyle konuşur:

```csharp
void MaasOde2(Hesap2 hesap, decimal tutar)
{
    if (hesap.CekilebilirMi(tutar)) hesap.ParaCek(tutar);
    else Console.WriteLine("Bu hesaptan ödeme yapılamıyor.");
}
```

Kısıt taban sınıfa **soru olarak** taşındı. Yeni bir hesap türü kendi kısıtını cevaplar, çağıran kod değişmez.

### Gerçekçi örnek: Kus / Penguen

```csharp
// KÖTÜ — her kuş uçmaz
public class Kus { public virtual void Uc() => Console.WriteLine("Uçuyor"); }

public class Penguen : Kus
{
    public override void Uc() => throw new NotSupportedException("Penguen uçamaz.");
}

void Goc(IEnumerable<Kus> kuslar) { foreach (var k in kuslar) k.Uc(); }   // patlar
```

`NotSupportedException` fırlatan bir `override`, LSP ihlalinin en açık işaretidir. Yüzeyde olmaması gereken bir üye var demektir.

```csharp
// İYİ — yetenek ayrı arayüz; uçmayan kuş o arayüzü uygulamaz
public abstract class Kus2 { public abstract void Beslen(); }
public interface IUcabilir { void Uc(); }

public sealed class Kartal : Kus2, IUcabilir
{
    public override void Beslen() { }
    public void Uc() => Console.WriteLine("Uçuyor");
}

public sealed class Penguen2 : Kus2
{
    public override void Beslen() { }
    public void Yuz() => Console.WriteLine("Yüzüyor");
}

void Goc2(IEnumerable<IUcabilir> ucanlar) { foreach (var u in ucanlar) u.Uc(); }
```

Bu çözüm aynı zamanda Perşembe notunun konusu olan **Interface Segregation**'a açılan kapıdır: büyük bir `IKus` arayüzü yerine, yeteneklere göre bölünmüş küçük arayüzler.

> Kodunda `NotImplementedException`, `NotSupportedException` fırlatan bir `override` ya da `if (x is AltTip)` diye alt tipi ayıklayan bir çağıran varsa, LSP ihlali vardır. İkisi de aynı şeyi söyler: alt tip taban tipin yerine geçemiyor.

> **Bu benzetme şurada bozulur:** Nöbetçi eczane benzetmesi ihlalin hep alt tipin hatası olduğunu söylüyor. Çoğu zaman suçlu **taban tiptir**: `Kus` sınıfına `Uc()` koymak baştan yanlış bir genellemeydi. LSP ihlali gördüğünde önce alt tipi düzeltmeye çalışma; taban tipin verdiği sözün fazla geniş olup olmadığına bak.

---

## 7. Davranışı Parametre Olarak Geçirmek

> **Benzetme —** Terziye gidersin. "Ceket dik" dersin ve yanında bir kumaş uzatırsın. Terzi her kumaş için ayrı bir terzi olmak zorunda değildir; kumaş dışarıdan gelir, dikiş yöntemi aynı kalır. Kalıtımla çözseydin "yün ceket terzisi", "keten ceket terzisi" diye ayrı ayrı terziler yetiştirmen gerekirdi.

**Basitçe:** Sınıflar arasındaki tek fark küçük bir davranış parçasıysa, o parçayı yeni bir sınıf yerine bir **parametre** olarak geç. C#'ta bu, bir `Func<>`, `Action<>` ya da küçük bir arayüzle yapılır.

**Teknik olarak:** Kalıtımla çözülen "değişen tek adım" problemi, delege ya da arayüz parametresiyle çok daha ucuza çözülür.

```csharp
// KÖTÜ — her sıralama ölçütü için ayrı sınıf
public abstract class SiralayiciTemel
{
    public List<Urun> Sirala(List<Urun> urunler)
    {
        var kopya = new List<Urun>(urunler);
        kopya.Sort(Karsilastir);
        return kopya;
    }
    protected abstract int Karsilastir(Urun a, Urun b);
}

public class FiyataGoreSiralayici : SiralayiciTemel
{
    protected override int Karsilastir(Urun a, Urun b) => a.Fiyat.CompareTo(b.Fiyat);
}

public class AdaGoreSiralayici : SiralayiciTemel
{
    protected override int Karsilastir(Urun a, Urun b) => string.Compare(a.Ad, b.Ad);
}
```

```csharp
// İYİ — değişen parça parametre; sınıfa gerek yok
public static List<Urun> Sirala(List<Urun> urunler, Comparison<Urun> karsilastir)
{
    var kopya = new List<Urun>(urunler);
    kopya.Sort(karsilastir);
    return kopya;
}

var fiyataGore = Sirala(urunler, (a, b) => a.Fiyat.CompareTo(b.Fiyat));
var adaGore    = Sirala(urunler, (a, b) => string.Compare(a.Ad, b.Ad));
```

İki sınıf, iki dosya ve bir hiyerarşi yerine iki satır. Üstelik yeni bir ölçüt eklemek için hiçbir tipe dokunmuyorsun.

### Arayüz mü delege mi

| Durum | Tercih |
|---|---|
| Tek metotluk davranış, durum yok | `Func<>` / `Action<>` |
| Davranışın yanında durum ya da isim lazım | Küçük arayüz |
| Aynı davranış birden çok yerde yeniden kullanılacak | Küçük arayüz |
| DI konteynerinden gelecek | Küçük arayüz |
| Yalnızca o çağrıya özel | Lambda |

```csharp
// Küçük arayüz: davranışın adı ve durumu var
public interface IIndirimKurali
{
    string Ad { get; }
    decimal Uygula(decimal tutar);
}

public sealed class YuzdeIndirim : IIndirimKurali
{
    private readonly decimal _oran;
    public YuzdeIndirim(decimal oran) => _oran = oran;

    public string Ad => $"%{_oran * 100:0} indirim";
    public decimal Uygula(decimal tutar) => tutar * (1 - _oran);
}
```

Kullanan taraf kuralları sırayla uygular ve hangi kuralın geldiğini bilmez:

```csharp
public sealed class Kasa
{
    private readonly IReadOnlyList<IIndirimKurali> _kurallar;
    public Kasa(IReadOnlyList<IIndirimKurali> kurallar) => _kurallar = kurallar;

    public decimal Hesapla(decimal tutar)
        => _kurallar.Aggregate(tutar, (t, k) => k.Uygula(t));
}
```

Bu, **Strategy (strateji) kalıbının** ta kendisidir ve bootcamp'te tasarım kalıpları konusuna geldiğinde adını koyarak tekrar göreceksin. Buradaki asıl ders şu: strateji kalıbı yeni bir şey öğretmiyor, sadece "değişen davranışı kompozisyonla dışarı al" fikrinin isimlendirilmiş hâli.

> **Bu benzetme şurada bozulur:** Terzi benzetmesi kumaşı pasif bir malzeme gibi gösteriyor. Delege ya da strateji pasif değildir — içinde kod çalıştırır, istisna fırlatabilir, yavaş olabilir. Dışarıdan davranış alan bir sınıf, aldığı davranışın kötü olma ihtimaline karşı kendini savunmalıdır: `null` kontrolü, zaman aşımı, istisna yakalama. Kumaşa bunları yapmazsın.

---

## 8. `sealed` — Ne Zaman, Neden Performans da Kazandırır

> **Benzetme —** Bir dükkânın kepenginde "devren satılık değildir" yazması. Kısıtlayıcı görünür ama herkesin işini kolaylaştırır: kimse boşuna pazarlığa girmez, sahibi de dükkânın düzenini değiştirirken kimseye danışmak zorunda kalmaz.

**Basitçe:** `sealed`, "bu sınıftan türetilemez" demektir. Kalıtımın maliyetini hiç ödememenin en kestirme yoludur.

**Teknik olarak:** `sealed` bir sınıf yazdığında üç şey kazanırsın.

**1. Kırılgan taban sınıf riski sıfırlanır.** Kimse türetmiyorsa sınıfın içini istediğin gibi değiştirebilirsin; `protected` üyelerin ve çağrı sıraların sözleşme değildir.

**2. Sözleşme netleşir.** Türetilebilir bir sınıf yazmak ayrı bir iştir: her `virtual` üyenin nasıl ezileceğini düşünmek, sırayı belgelemek gerekir. Bunu yapmayacaksan kapıyı kapat.

**3. Performans.** JIT, `sealed` bir tipte iki iyileştirme yapabilir:

- **Devirtualization.** Sanal çağrı normalde metot tablosu üzerinden çözülür. Tip `sealed` ise başka bir gerçeklemenin olamayacağı kesindir; JIT çağrıyı doğrudan bağlar ve gövde küçükse satır içine (inline) alır.
- **Ucuz tip kontrolü.** `is` ve cast işlemleri normalde kalıtım zincirini tarar. `sealed` tipte tek bir referans karşılaştırması yeterlidir. Dizi yazmalarında (`array covariance` kontrolü) de aynı kazanç vardır.

```csharp
// Sıcak döngüde ölçülebilir fark
public sealed class Nokta
{
    public double X { get; init; }
    public double Y { get; init; }
    public double Uzunluk() => Math.Sqrt(X * X + Y * Y);
}

double Toplam(Nokta[] noktalar)
{
    double t = 0;
    for (int i = 0; i < noktalar.Length; i++)
        t += noktalar[i].Uzunluk();   // sealed -> doğrudan çağrı, çoğu zaman inline
    return t;
}
```

Tek çağrıda fark ölçülemez. Milyonlarca çağrıda ve kütüphane kodunda fark edilir. Bu yüzden .NET'in kendi kaynak kodunda `internal` sınıflar neredeyse istisnasız `sealed`'dir ve `CA1852` analiz kuralı bunu önerir.

> `sealed`'i sonradan **kaldırmak** kimseyi bozmaz; sonradan **eklemek** türetmiş herkesi bozar. Bu asimetri yüzünden şüphedeyken `sealed` yazmak daha az risklidir. Kütüphane yazıyorsan kural daha da katıdır: `public` ve türetilebilir bir sınıf, sonsuza kadar sürdürmen gereken bir sözdür.

> **Bu benzetme şurada bozulur:** "Devren satılık değildir" tabelası dükkânı tamamen kapalı gösteriyor. `sealed` bir sınıf kapalı değildir — arayüz uygulayabilir, kompozisyonla sarmalanabilir, extension method alabilir. Kapanan tek kapı kalıtımdır ve zaten en pahalı kapı odur.

---

## 9. Extension Method ile Davranış Eklemek

> **Benzetme —** Kütüphanedeki bir kitaba kendi not kâğıdını iliştirmek. Kitabın metnine dokunmazsın, ama senin rafında o kitap notlarınla birlikte durur. Başka biri aynı kitabı aldığında senin notlarını görmez. Notların kitabın içine giremez: kapağın arkasındaki mührü değiştiremezsin.

**Basitçe:** Extension method (genişletme metodu), var olan bir tipe, o tipin kodunu değiştirmeden metot eklemenin yoludur. Sınıfı açmadan, türetmeden yapılır.

**Teknik olarak:** `static` bir sınıf içinde, ilk parametresi `this` ile işaretlenmiş `static` bir metottur. Derleyici, `a.Metot(b)` yazımını `Sinif.Metot(a, b)` çağrısına çevirir. Yeni bir şey icat etmez — sadece okunuşu değiştirir.

```csharp
public static class StringUzantilari
{
    public static bool GecerliTcKimlikMi(this string? deger)
        => !string.IsNullOrWhiteSpace(deger)
           && deger.Length == 11
           && deger.All(char.IsAsciiDigit)
           && deger[0] != '0';

    public static string Kisalt(this string metin, int enFazla)
        => metin.Length <= enFazla ? metin : metin[..(enFazla - 1)] + "…";
}

"12345678901".GecerliTcKimlikMi();      // true
"uzun bir başlık".Kisalt(8);            // "uzun bi…"
```

LINQ'in tamamı bu mekanizmayla yazılmıştır: `Where`, `Select`, `OrderBy` — hepsi `IEnumerable<T>` üzerine yazılmış extension method'lardır. ASP.NET Core'daki `services.AddControllersWithViews()` ve `app.UseStaticFiles()` de öyle.

### Ne zaman uygun

| Durum | Uygun mu |
|---|---|
| Değiştiremediğin bir tipe yardımcı metot (`string`, `DateTime`) | Evet |
| Bir arayüze, gerçeklemeleri değiştirmeden ortak kolaylık eklemek | Evet |
| Akıcı (fluent) kurulum API'si yazmak (`IServiceCollection`) | Evet |
| Tipin **kendi sorumluluğu** olan bir davranış | Hayır — tipin içine yaz |
| Tipin `private` durumuna erişmesi gereken iş | Hayır — erişemez |
| Polimorfik davranış gerekiyor | **Hayır** — extension method sanal değildir |

### En sık yapılan hata: extension method sanal değildir

```csharp
// KÖTÜ — polimorfizm bekleniyor ama olmuyor
public class Sekil { }
public class Daire : Sekil { }

public static class SekilUzantilari
{
    public static string Anlat(this Sekil s) => "şekil";
    public static string Anlat(this Daire d) => "daire";
}

Sekil s = new Daire();
Console.WriteLine(s.Anlat());   // "şekil" — değişkenin statik tipine bakıldı
```

Bu, Pazartesi notundaki `new` ile metot gizleme tuzağının aynısıdır: çağrı, nesnenin gerçek tipine göre değil, **derleme zamanındaki tipe** göre çözülür.

```csharp
// İYİ — polimorfik davranış tipin içinde
public abstract class Sekil2 { public abstract string Anlat(); }
public sealed class Daire2 : Sekil2 { public override string Anlat() => "daire"; }
```

### Diğer sınırlar

- `private`/`protected` üyelere erişemez; sadece `public` yüzeyle çalışır.
- Aynı imzada gerçek bir metot varsa **o kazanır**. Tip sahibi yarın aynı isimde metot eklerse senin uzantın sessizce devre dışı kalır.
- `null` üzerinde çağrılabilir: `((string?)null).BosMu()` `NullReferenceException` atmaz. Bazen faydalıdır, ama okuyanı şaşırtır.
- `using` gerektirir. Uzantı, namespace'i import edilmediği sürece görünmez; bu yüzden uzantıları tahmin edilebilir bir namespace'e koy.

> Extension method, tipi değiştirmeden davranış eklemenin en ucuz yoludur. Ama davranış gerçekten o tipin işiyse uzantı yazma — o metot sınıfın içine aittir.

> **Bu benzetme şurada bozulur:** Not kâğıdı benzetmesi notun her zaman görüneceğini söylüyor. Extension method sadece `using` satırını yazan dosyada görünür. Aynı projedeki başka bir dosya senin uzantından habersiz olabilir ve aynı işi tekrar yazabilir. Not kâğıdı kitabın içinde durur; extension method okuyanın gözlüğündedir.

---

## 10. Küçük Arayüzler ve Default Interface Method'lar

> **Benzetme —** Bir elektrikçinin çanta düzeni. Her iş için tüm çantayı taşımaz; pense, tornavida ve test kalemini ayrı ayrı alır. Müşteri "priz tak" dediğinde gereken üç alet gelir, otuz alet değil. Büyük tek çanta, taşıyanı da müşteriyi de yorar.

**Basitçe:** Tek büyük arayüz yerine, yetenek başına küçük arayüzler yaz. Uygulayan taraf sadece gerçekten yapabildiğini üstlenir, kullanan taraf sadece ihtiyacı olana bağlanır.

**Teknik olarak:** Bu, SOLID'in **I** harfi — **Interface Segregation Principle (arayüz ayırma prensibi)**. Perşembe notunda ayrıntılı işlenecek; buradaki bağlantı şu: büyük arayüz, kalıtımın "gereksiz yüzey" sorununun arayüz hâlidir.

```csharp
// KÖTÜ — tek büyük arayüz; salt okunur repository de yazma metotlarını uygulamak zorunda
public interface IRepository<T>
{
    List<T> List();
    T? Find(int id);
    void Insert(T e);
    void Update(T e);
    void Delete(T e);
    void BulkImport(IEnumerable<T> e);
}

public class RaporRepository : IRepository<Rapor>
{
    public List<Rapor> List() => new();
    public Rapor? Find(int id) => null;
    public void Insert(Rapor e) => throw new NotSupportedException();   // LSP ihlali
    public void Update(Rapor e) => throw new NotSupportedException();
    public void Delete(Rapor e) => throw new NotSupportedException();
    public void BulkImport(IEnumerable<Rapor> e) => throw new NotSupportedException();
}
```

```csharp
// İYİ — yeteneğe göre bölünmüş arayüzler
public interface IOkuyucu<T>
{
    List<T> List();
    T? Find(int id);
}

public interface IYazici<T>
{
    void Insert(T e);
    void Update(T e);
    void Delete(T e);
}

public sealed class RaporRepository2 : IOkuyucu<Rapor>
{
    public List<Rapor> List() => new();
    public Rapor? Find(int id) => null;
}

public sealed class DeneyimRepository2 : IOkuyucu<TblDeneyimlerim>, IYazici<TblDeneyimlerim>
{
    // her iki yüzeyi de uygular
}
```

MvcCv'deki `HakkimdaController` gibi sadece listeleme yapan bir controller artık `IOkuyucu<T>` alır. Bağımlılığı imzasında dürüstçe yazar, yazma metotlarına hiç erişemez.

### Mixin benzeri yapılar ve default interface method'lar

C#'ta çoklu kalıtım yoktur; ama bir sınıf birden çok arayüz uygulayabilir. Default interface method'larla (C# 8) bu arayüzler gövdeli metot da taşıyabilir — sonuç, başka dillerdeki **mixin**'e benzer:

```csharp
public interface IIzlenebilir
{
    string Kimlik { get; }

    // Her uygulayan için hazır davranış — mixin benzeri
    void IzKaydet(ILogger logger, string olay)
        => logger.LogInformation("{Kimlik}: {Olay}", Kimlik, olay);
}

public sealed class Siparis2 : IIzlenebilir
{
    public int Id { get; init; }
    public string Kimlik => $"Siparis-{Id}";
}
```

Üç sınır Pazartesi notundakiyle aynıdır ve bu yapıyı gerçek mixin olmaktan alıkoyar:

1. Arayüz **durum tutamaz** — ortak bir sayaç ya da liste tanımlanamaz.
2. Default metot **sınıf referansından görünmez**; `((IIzlenebilir)s).IzKaydet(...)` gerekir.
3. Constructor yoktur; kurulum mantığı paylaşılamaz.

```csharp
var s = new Siparis2 { Id = 7 };
// s.IzKaydet(logger, "olustu");                 // derleme hatası
((IIzlenebilir)s).IzKaydet(logger, "olustu");    // çalışır
```

> Default interface method bir **sürüm uyumluluğu** aracıdır: yayınlanmış bir arayüze yeni üye eklerken uygulayanları bozmamak için vardır. Yeni tasarımda ortak davranışı extension method'a koymak daha okunaklıdır.

> **Bu benzetme şurada bozulur:** Elektrikçi çantası benzetmesi "ne kadar küçük o kadar iyi" izlenimi veriyor. Her metodu ayrı arayüze koyarsan kayıt tarafı okunmaz hâle gelir ve DI kurulumu şişer. Ölçüt boyut değil **birlikte değişme**: birlikte değişen metotlar aynı arayüzde kalsın, ayrı ayrı değişenler bölünsün.

---

## 11. Kalıtımın Hâlâ Doğru Olduğu Durumlar

> **Benzetme —** Apartmanın ortak yönetim planı. Kimse her daire için ayrı plan yazmaz; plan bir kere yapılır, daireler ona uyar. Kural sabit ve yazılıdır, dairenin değiştirebileceği yerler de baştan bellidir: kapı rengi serbest, taşıyıcı kolon değil.

**Basitçe:** Kalıtım kötü değil, fazla kullanılıyor. Doğru kullanıldığı üç net yer var: framework tipleri, template method ve gerçek soyut hiyerarşiler.

**Teknik olarak:**

**1. Framework tiplerinden türetmek.** Framework zaten türetilmek üzere tasarlanmıştır, sözleşmesi belgelenmiştir ve senin sınıfın yaprak (leaf) konumdadır.

```csharp
public sealed class DeneyimController : Controller      // ASP.NET Core zorunlu kılıyor
{
    private readonly IDeneyimRepository _repo;
    public DeneyimController(IDeneyimRepository repo) => _repo = repo;

    public IActionResult Index() => View(_repo.List());
}

public sealed class UygulamaDbContext : DbContext       // EF Core zorunlu kılıyor
{
    public UygulamaDbContext(DbContextOptions<UygulamaDbContext> o) : base(o) { }
    public DbSet<TblDeneyimlerim> Deneyimler => Set<TblDeneyimlerim>();
}

public sealed class StokYetersizException : Exception    // istisna hiyerarşisi kalıtımdır
{
    public StokYetersizException(string m) : base(m) { }
}
```

Buradaki ortak nokta: türediğin tip senin kontrolünde değil ama **türetilmek için tasarlanmış**. MvcCv'deki bütün controller'lar bu yüzden `Controller`'dan türer ve bu doğru bir kalıtımdır.

**2. Template method — akış sabit, adım değişken.**

```csharp
public abstract class ToplamaIsi
{
    // Akış kilitli: virtual değil
    public async Task CalistirAsync(CancellationToken ct)
    {
        var kayitlar = await KayitlariGetirAsync(ct);
        var temiz    = Temizle(kayitlar);
        await YazAsync(temiz, ct);
    }

    protected abstract Task<IReadOnlyList<Kayit>> KayitlariGetirAsync(CancellationToken ct);
    protected abstract Task YazAsync(IReadOnlyList<Kayit> kayitlar, CancellationToken ct);

    // İsteğe bağlı adım: varsayılanı var
    protected virtual IReadOnlyList<Kayit> Temizle(IReadOnlyList<Kayit> k)
        => k.Where(x => x is not null).ToList();
}

public sealed class GunlukToplamaIsi : ToplamaIsi
{
    protected override Task<IReadOnlyList<Kayit>> KayitlariGetirAsync(CancellationToken ct) => ...;
    protected override Task YazAsync(IReadOnlyList<Kayit> k, CancellationToken ct) => ...;
}
```

Bu kalıbın sağlıklı olmasının üç şartı var: akış metodu `virtual` **değil**, değişken adımlar `protected abstract`, ve taban sınıf hangi adımı ne zaman çağıracağını belgeliyor. Üçü de varsa kırılgan taban sınıf riski en aza iner.

**3. Gerçek soyut hiyerarşiler.** Ortak bir kavramın gerçekten var olduğu, alt tiplerin kapalı ve iyi tanımlı olduğu yerler: `Stream`, `Exception`, `Expression`, ödeme sağlayıcıları, dosya biçimleri. Burada tek seviye yeter.

> **Bu benzetme şurada bozulur:** Yönetim planı benzetmesi planın değişmeyeceğini varsayıyor. Framework'ler de sürüm değiştirir: ASP.NET Core'un `Controller` sınıfı yıllar içinde değişti, EF Core'un `DbContext`'i de. Fark şu ki bu değişiklikler **belgelenir ve duyurulur**; kendi taban sınıfındaki sessiz değişiklik duyurulmaz. Kalıtımı güvenli kılan şey kalıtımın kendisi değil, sözleşmenin yazılı olmasıdır.

---

## Tek Bakışta Özet

- Kalıtım, iki sınıf arasında kurulabilecek en sıkı bağdır; kompozisyonun bağı tek satırlık bir sözleşmedir.
- Kırılgan taban sınıf: taban sınıftaki masum bir değişiklik türeyenleri sessizce bozar; derleyici yakalamaz.
- "is-a" cümlesinin doğruluğu yetmez; yerine geçebilme, yüzey uyumu ve ilişkinin ömrü de sorulur.
- İlişki çalışma zamanında değişiyorsa kalıtım kuramazsın; roller ve durumlar kompozisyonla tutulur.
- LSP: alt tip, üst tipin sözünü daraltamaz. `NotSupportedException` fırlatan `override` açık ihlaldir.
- LSP ihlalinde önce taban tipin fazla geniş söz verip vermediğine bak — suçlu genelde odur.
- Kısıtı taban sınıfa **soru olarak** taşı (`CekilebilirMi`), çağıran kod dürüst bir yüzeyle konuşsun.
- Değişen tek davranış parçasını sınıf değil parametre yap: `Func<>`, `Action<>` ya da küçük arayüz.
- `sealed` şüphedeyken doğru varsayılandır; kaldırmak kolay, eklemek yıkıcıdır. Devirtualization ve ucuz tip kontrolü yan kazançtır.
- Extension method sanal değildir; polimorfik davranış için kullanma.
- Büyük arayüzü yeteneklere böl; ölçüt boyut değil, birlikte değişme.
- Kalıtım framework tiplerinde, template method'da ve gerçek soyut hiyerarşilerde hâlâ doğru araçtır.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Kompozisyon | Bir sınıfın işini, içinde tuttuğu başka nesnelere yaptırması |
| Delegasyon | Gelen çağrıyı içerideki parçaya devretme |
| Sıkı bağ (tight coupling) | Bir tipin başka bir tipin iç detaylarına bağımlı olması |
| Fragile base class | Taban sınıftaki değişikliğin türeyenleri sessizce bozması |
| is-a / has-a | Kimlik ilişkisi / sahiplik ilişkisi |
| LSP | Alt tipin, üst tipin yerine sorunsuz geçebilmesi prensibi |
| Decorator | Aynı arayüzü uygulayıp içerideki nesneyi sarmalayan kalıp |
| Strategy | Değişen davranışı dışarıdan parametre olarak alan kalıp |
| Template method | Akışı taban sınıfın tuttuğu, adımları türeyene bıraktığı kalıp |
| Extension method | Tipi değiştirmeden dışarıdan metot ekleme yolu |
| Interface Segregation | Büyük arayüzü yeteneklere göre bölme prensibi |
| Invariant (değişmez) | Nesnenin ömrü boyunca bozulmaması gereken kural |
| Leaf (yaprak) sınıf | Kendisinden türetilmeyen, hiyerarşinin ucundaki sınıf |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Kod tekrarını önlemek için kalıtım kurulur" | Tekrar için kompozisyon ya da yardımcı metot; kalıtım bir kimlik beyanıdır |
| "is-a cümlesi doğruysa kalıtım doğrudur" | Yerine geçebilme, yüzey uyumu ve ilişkinin ömrü de sağlanmalı |
| "Kare bir dikdörtgendir, kalıtım doğaldır" | Değiştirilebilir bir dikdörtgende kare LSP'yi bozar; ortak arayüz kullan |
| "`NotSupportedException` fırlatmak geçerli bir çözümdür" | Yüzeyin yanlış olduğunun işaretidir; arayüzü böl |
| "`sealed` esnekliği öldürür" | Arayüz, kompozisyon ve extension method açık kalır; kapanan tek kapı kalıtımdır |
| "Extension method ile polimorfik davranış yazılabilir" | Uzantılar statik tipe göre çözülür; sanal değildir |
| "Derin hiyerarşi iyi tasarım göstergesidir" | Üçten derin hiyerarşi bakım maliyeti üretir, değer üretmez |
| "Kalıtım artık kullanılmamalı" | Framework tipleri, template method ve gerçek soyut hiyerarşilerde doğru araçtır |
| "Default interface method çoklu kalıtım getirdi" | Durum tutamaz, ctor'ı yoktur, sınıf referansından görünmez |

---

## Sonraki

→ `03-SOLID-SRP-OCP-LSP.md` (Çarşamba)
