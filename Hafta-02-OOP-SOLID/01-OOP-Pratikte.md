# Hafta 2 · Pazartesi — OOP Dört Sütun, Pratikte

**Okuma süresi:** ~50 dk
**Neden bu konu:** OOP'un dört sütunu her mülakatta tanım olarak sorulur ama asıl iş, bunların hangi soruna çare olduğunu bilmektir. `virtual` mi `new` mi, `abstract class` mı `interface` mi, `Equals` override edilir mi — bu kararlar MvcCv gibi küçük bir projede bile her gün karşına çıkar. Bootcamp'in SOLID, tasarım kalıpları ve Dependency Injection konuları da doğrudan bu kararların üstüne kuruluyor.

---

## Önce Basitçe

Bir apartman düşün. Dairelerin kapısı var, kapının anahtarı sende. İçeride ne yaptığını komşun bilmez; ortak alanda ise kurallar herkes için aynıdır. Yöneticiye "daireme su geldi mi?" diye sorarsın, o sana cevap verir; su borusunun hangi kattan geçtiğini bilmek zorunda değilsin. Nesne yönelimli programlamanın tamamı bu iki fikrin etrafında döner: **içerisini gizlemek** ve **dışarıya sade bir kapı bırakmak**.

Kodda bu şu demek: bir sınıf yazarsın, içinde veriler ve o verilerle çalışan metotlar olur. Dışarıdan sadece izin verdiğin kadarı görünür. Veriyi doğrudan kurcalamak yerine, sınıfın sunduğu metotlardan geçmek zorunda kalırlar. Böylece kuralları tek yerde tutarsın. "Bakiye eksiye düşemez" kuralını bir kere yazarsın, otuz farklı yerde tekrar etmezsin.

İkinci fikir, birbirine benzeyen şeyleri tek bir çatı altında toplamaktır. Vadesiz hesap da bir hesaptır, vadeli hesap da. İkisinin de bakiyesi var, ikisine de para yatırılır. Farkları faiz hesabındadır. Ortak kısmı bir kere yazıp, farklı kısmı her hesap türünün kendisine bırakabilirsin. Kalıtım budur. Ama dikkat: kalıtım bir kolaylık aracı değil, bir **söz**dür. "Bu tip, ötekinin yerine geçebilir" diye söz verirsin. Sözünü tutamayacaksan kalıtım kurma.

Üçüncü fikir daha ilginç. Elinde bir "hesap" varsa ve ona "faizini işlet" dersen, o hesabın hangi tür olduğunu bilmene gerek kalmaz. Vadeliyse vadelinin kuralı, vadesizse vadesizin kuralı çalışır. Sen tek bir cümle kurdun, doğru davranış kendiliğinden seçildi. Buna çok biçimlilik denir ve kodundaki `if` yığınlarının çoğunu ortadan kaldıran şey budur.

Bu notta bu dört fikri tanım olarak değil, **hangi soruna çare oldukları** üzerinden göreceksin. Yanında da C#'ın bu fikirleri uygularken sunduğu araçlar var: erişim belirleyiciler, property'ler, `virtual`, `abstract`, `sealed`, `record`. Her birinin bir de tuzağı var; onları da ayrı ayrı işaretleyeceğiz. Şimdi detaya iniyoruz.

> **Ana benzetme:** Bir sınıf, **apartman dairesidir**. Kapısı, kapının kilidi ve zili vardır. Zile basan içeride ne olduğunu görmez, sadece kapıdan verilene bakar. Kalıtım, aynı projeden çıkmış iki bloktur: ortak temel aynıdır, iç düzen farklı olabilir. Çok biçimlilik ise kapıcının her daireye aynı şekilde "aidat" demesi ama her dairenin kendi tutarını ödemesidir.

---

## Bu Notta Ne Var

1. Dört sütun neye yarar
2. Kapsülleme ve erişim belirleyiciler
3. Property vs field, backing field, computed property
4. Kalıtım — ne devralınır, neyi bağlar
5. `virtual` / `override` / `new` ve method hiding tuzağı
6. Soyutlama: `abstract class` mı `interface` mi
7. Constructor zinciri, `base` ve sanal metot çağırma tuzağı
8. Static vs instance
9. `object`'in metotları ve override sözleşmesi
10. Nesne eşitliği: referans mı, değer mi — ve `record`
11. `sealed` — kapıyı kapatmak
12. Polimorfizmin gerçek faydası: `switch` yerine tip

---

## 1. Dört Sütun Neye Yarar

> **Benzetme —** Bir oto tamirhanesine arabanı bırakırsın. Ustaya "motorda ses var" dersin, o ne yaptığını sana anlatmaz; sadece "tamam" der ve akşam arabayı çalışır hâlde teslim eder. Tamirhanenin içinde kim hangi aleti kullanıyor, hangi vida nereye gidiyor seni ilgilendirmez. Sen sadece **kapıdaki sözleşmeyi** bilirsin: araba bırakılır, tamir edilmiş araba alınır.

**Basitçe:** Dört sütun, "kodu nasıl parçalara bölerim ki yarın değiştirdiğimde her şey çökmesin?" sorusunun dört ayrı cevabıdır. Ezberlenecek tanım değil, karşılaşılacak dört ayrı sorunun çözümüdür.

**Teknik olarak:** Her sütunun çözdüğü somut bir sorun vardır:

| Sütun | Çözdüğü sorun | Somut belirti |
|---|---|---|
| **Encapsulation (kapsülleme)** | Veri her yerden serbestçe değiştirilebiliyor, kural tek yerde tutulamıyor | 30 dosyada `bakiye -= tutar` yazılmış, biri eksi kontrolü unutmuş |
| **Abstraction (soyutlama)** | Çağıran taraf, çağırdığı şeyin iç detayına bağımlı | Controller, `SqlConnection` açıyor |
| **Inheritance (kalıtım)** | Aynı kod birden fazla tipte tekrar ediyor | Beş sınıfta aynı `Id`, `OlusturmaTarihi` alanları |
| **Polymorphism (çok biçimlilik)** | Tip başına `if`/`switch` çoğalıyor | Yeni tip eklediğinde 12 dosyada switch güncellenmesi gerekiyor |

```csharp
// Dört sütunun hepsinin tek örnekte göründüğü hâl

public abstract class Hesap                  // Soyutlama: "hesap" diye bir kavram var
{
    private decimal _bakiye;                 // Kapsülleme: alan dışarıya kapalı
    public decimal Bakiye => _bakiye;        // okuma serbest, yazma değil

    public void ParaYatir(decimal tutar)     // kural tek yerde
    {
        if (tutar <= 0)
            throw new ArgumentOutOfRangeException(nameof(tutar));
        _bakiye += tutar;
    }

    public abstract decimal AylikFaiz();     // Çok biçimlilik: her tür kendi bilir
}

public class VadesizHesap : Hesap            // Kalıtım: ortak kısım devralındı
{
    public override decimal AylikFaiz() => 0m;
}

public class VadeliHesap : Hesap
{
    private readonly decimal _oran;
    public VadeliHesap(decimal oran) => _oran = oran;
    public override decimal AylikFaiz() => Bakiye * _oran / 12m;
}
```

Çağıran tarafın gördüğü tek şey `Hesap` tipidir:

```csharp
// Hangi tür olduğunu bilmeden çalışır
decimal ToplamFaiz(IEnumerable<Hesap> hesaplar)
    => hesaplar.Sum(h => h.AylikFaiz());
```

Yarın `KatilimHesabi` eklediğinde bu metoda dokunmazsın. Dört sütunun tamamının varlık sebebi bu cümledir.

> **Bu benzetme şurada bozulur:** Tamirhane örneği "iç detayı hiç bilme" der. Gerçekte bir sınıfın iç detayını bilmen gerekebilir — performans sorunu ararken, bir hatayı ayıklarken. Soyutlama detayı **yok etmez**, sadece normal akışta görmeni gerektirmez. "Soyutladım, artık içeriye bakmam" diye bir kural yok; "içeriye bakmadan da kullanabilirim" diye bir kolaylık var.

---

## 2. Kapsülleme ve Erişim Belirleyiciler

> **Benzetme —** Bir bankanın kasası. Kasadaki para fiziksel olarak oradadır ama sen elini uzatıp alamazsın. Gişeye gider, "şu kadar çekeceğim" dersin; memur kimliğine bakar, bakiyeyi kontrol eder, sonra verir. Kasaya doğrudan erişim olsaydı her kural her seferinde yeniden düşünülmek zorunda kalırdı.

**Basitçe:** Kapsülleme, veriyi saklamak değil — **veriye giden yolu tek kapıya indirmektir**. O kapıda kuralı bir kez yazarsın, sonra kural kendiliğinden her yerde geçerli olur.

**Teknik olarak:** C#'ta altı erişim belirleyici (access modifier) vardır. Assembly (derlenmiş çıktı, `.dll`) kavramı üçünde belirleyicidir.

| Belirleyici | Kimler görür |
|---|---|
| `public` | Herkes — kendi assembly'si ve dışarısı |
| `private` | Sadece tanımlandığı tip (ve iç içe tipleri) |
| `protected` | Tanımlandığı tip ve ondan türeyenler — assembly fark etmez |
| `internal` | Aynı assembly içindeki her şey |
| `protected internal` | `protected` **VEYA** `internal` — aynı assembly'deki herkes, artı başka assembly'deki türeyenler |
| `private protected` | `protected` **VE** `internal` — sadece aynı assembly'deki türeyenler |

İki bileşik belirleyicinin adı yanıltıcıdır. `protected internal` daha **geniştir** (birleşim), `private protected` daha **dardır** (kesişim). Karıştırmamanın yolu: `private` kelimesi geçen daha dar.

Tip seviyesinde varsayılanlar farklıdır ve bu sık hata kaynağıdır:

| Bağlam | Belirleyici yazılmazsa |
|---|---|
| Namespace içindeki sınıf/arayüz | `internal` |
| Sınıf üyesi (alan, metot, property) | `private` |
| `interface` üyesi | `public` |
| `struct` üyesi | `private` |
| `enum` üyesi | `public` |

### Kapsüllemenin asıl noktası: kuralın tek yerde olması

```csharp
// KÖTÜ — alan public, kural her çağıranda tekrar yazılmak zorunda
public class Sepet
{
    public List<SepetSatiri> Satirlar = new();
    public decimal Toplam;
}

// çağıran taraf
sepet.Satirlar.Add(yeniSatir);
sepet.Toplam += yeniSatir.Fiyat;   // bunu unutan biri sepeti bozar
```

```csharp
// İYİ — dışarı sadece niyet ifadesi açılıyor
public class Sepet
{
    private readonly List<SepetSatiri> _satirlar = new();

    public IReadOnlyList<SepetSatiri> Satirlar => _satirlar;
    public decimal Toplam => _satirlar.Sum(s => s.Fiyat * s.Adet);

    public void SatirEkle(SepetSatiri satir)
    {
        if (satir.Adet <= 0)
            throw new ArgumentException("Adet pozitif olmalı.", nameof(satir));
        _satirlar.Add(satir);
    }
}
```

İkinci sürümde `Toplam`'ı güncellemeyi unutmak **mümkün değil**, çünkü saklanan bir toplam yok. Kapsüllemenin en güçlü hâli budur: yanlış durumun temsil edilemez olması.

### `readonly` ve `init`

```csharp
public class Musteri
{
    private readonly int _id;          // sadece ctor içinde atanır
    public int Id => _id;
    public required string Ad { get; init; }   // kurulurken atanır, sonra kilitlenir

    public Musteri(int id) => _id = id;
}
```

`readonly` alanlar ve `init` property'ler, "bu nesne kurulduktan sonra değişmeyecek" sözünü derleyiciye verdiğin yerlerdir. C# 11'den beri `required`, o sözün atlanmasını da engeller.

> **Bu benzetme şurada bozulur:** Kasa benzetmesi kapsüllemeyi bir **güvenlik** önlemi gibi gösteriyor. Değildir. `private` bir alan yansıma (reflection) ile okunabilir, değiştirilebilir; kötü niyetli kodu durdurmaz. Kapsülleme bir **bakım** önlemidir: kazara yapılan yanlışı engeller, kuralı tek yerde toplar. Güvenlik başka bir katmanın işidir.

---

## 3. Property vs Field, Backing Field, Computed Property

> **Benzetme —** Apartman kapısındaki zil ile kapının kendisi. Field, dairenin içindeki eşyadır. Property ise zildir: dışarıdan zile basarsın, içeride ne olduğuna sen karar verirsin. Bugün "buyurun" dersin, yarın "kimsiniz?" diye sorarsın, öbür gün hiç açmazsın. Kapıyı duvardan söküp yerine zil takmak ise komşunun evine girme şeklini değiştirir — herkesin yeniden alışması gerekir.

**Basitçe:** Field veriyi tutar, property ise o veriye erişimi yönetir. Dışarıya field açarsan sonradan araya kural koyamazsın; property açarsan koyabilirsin.

**Teknik olarak:** **Field (alan)** bellekte yer kaplayan değişkendir. **Property (özellik)** ise arka planda `get_X()` ve `set_X()` metotlarına derlenen bir üye çiftidir. Property'nin kendisi veri tutmaz.

```csharp
// Auto-property — derleyici görünmez bir backing field üretir
public string Ad { get; set; }

// Yukarıdakinin derleyici tarafından üretilen hâline yakın karşılığı
private string <Ad>k__BackingField;
public string get_Ad() => <Ad>k__BackingField;
public void set_Ad(string value) => <Ad>k__BackingField = value;
```

### Backing field ne zaman elle yazılır

Kural, doğrulama ya da yan etki gerektiğinde:

```csharp
public class Urun
{
    private decimal _fiyat;                 // elle yazılmış backing field

    public decimal Fiyat
    {
        get => _fiyat;
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Fiyat negatif olamaz.");
            _fiyat = value;
        }
    }
}
```

### Computed property — saklanmayan, hesaplanan değer

```csharp
public class Calisan
{
    public string Ad { get; set; } = "";
    public string Soyad { get; set; } = "";
    public DateOnly IseGirisTarihi { get; set; }

    // Hesaplanan: bellekte yer kaplamaz, her okumada üretilir
    public string TamAd => $"{Ad} {Soyad}";

    public int KidemYili
        => DateOnly.FromDateTime(DateTime.Today).Year - IseGirisTarihi.Year;
}
```

`=>` ile yazılan property'ye **expression-bodied property** denir ve sadece `get` içerir. `TamAd`'a atama yapılamaz — zaten yapılmamalı da, çünkü kaynağı `Ad` ve `Soyad`.

```csharp
// KÖTÜ — türetilmiş değer saklanıyor, senkron kalması elle sağlanıyor
public string TamAd { get; set; } = "";   // Ad değişirse bu bayatlar

// İYİ — tek kaynak var, türetilmiş değer her seferinde ondan üretiliyor
public string TamAd => $"{Ad} {Soyad}";
```

### Property'de yapılmaması gerekenler

> Property'ler ucuz olmalıdır. İçinde veritabanı sorgusu, dosya okuma ya da ağ çağrısı varsa o property olmamalı, metot olmalıdır. Sebebi psikolojik: `musteri.Siparisler` yazan kişi bunun bir alan okuması olduğunu varsayar ve döngü içinde çekinmeden kullanır. Debugger da property'leri otomatik değerlendirir; içinde yan etki varsa sadece nesneye bakmak bile onu tetikler.

```csharp
// KÖTÜ — property gibi görünen gizli veritabanı çağrısı
public List<Siparis> Siparisler => _context.Siparisler.Where(s => s.MusteriId == Id).ToList();

// İYİ — iş yapıldığı adında görünüyor
public Task<List<Siparis>> SiparisleriGetirAsync(CancellationToken ct = default)
    => _context.Siparisler.Where(s => s.MusteriId == Id).ToListAsync(ct);
```

> **Bu benzetme şurada bozulur:** Zil benzetmesi property'yi hep "araya bir şey koyabilirsin" diye anlatıyor. Ama auto-property (`{ get; set; }`) araya hiçbir şey koymaz — field ile davranışı birebir aynıdır. Farkı bugün değil, **yarın** ortaya çıkar: araya kural koyman gerektiğinde çağıran kodu değiştirmene gerek kalmaz. Property'nin bugünkü değeri sıfır, yarınki değeri yüksektir.

---

## 4. Kalıtım — Ne Devralınır, Neyi Bağlar

> **Benzetme —** Bir ustanın yanında yetişen çırak. Ustanın bildiği her şeyi öğrenir, aletlerini kullanır, müşteriye ustanın adıyla iş yapar. Ama ustanın kasasının anahtarı yoktur ve ustanın evine giremez. Bir de şu var: usta yarın çalışma yöntemini değiştirirse, çırağın işi de değişir. Çırak ustaya bağlıdır; ustanın her kararı çırağa yansır.

**Basitçe:** Kalıtım, bir sınıfın başka bir sınıfın üyelerini hazır alması ve "ben de onun bir türüyüm" demesidir. Kolaylık kısmı ikincil; asıl mesele o "onun bir türüyüm" sözüdür.

**Teknik olarak:** C#'ta **tek kalıtım (single inheritance)** vardır: bir sınıf en fazla bir taban sınıftan (base class) türer. Arayüz sayısı sınırsızdır.

Devralınanlar ve devralınmayanlar:

| Üye | Devralınır mı |
|---|---|
| `public` / `protected` metot, property, alan | Evet |
| `internal` üyeler | Aynı assembly içindeyse evet |
| `private` üyeler | Hayır (nesnede yer kaplar ama erişilemez) |
| Constructor | **Hayır** — ama `base(...)` ile çağrılır |
| Destructor / finalizer | Hayır (zincirleme çalışır) |
| Static üyeler | Tip üzerinden erişilebilir, ayrı kopya oluşmaz |
| `sealed` sınıfın üyeleri | Sınıftan türetilemez |

```csharp
public class BaseEntity
{
    public int Id { get; set; }
    public DateTime OlusturmaTarihi { get; set; } = DateTime.UtcNow;
    protected void Logla(string mesaj) => Console.WriteLine($"[{Id}] {mesaj}");
    private  void IcSayac() { }          // türeyende erişilemez
}

public class Deneyim : BaseEntity
{
    public string Kurum { get; set; } = "";

    public void Kaydet()
    {
        Logla("kaydedildi");   // protected — erişilebilir
        // IcSayac();          // private — derleme hatası
    }
}
```

MvcCv'deki `GenericRepository<T> where T : BaseEntity` kısıtı tam da bu yapıya dayanır: repository, hangi tip gelirse gelsin `Id` alanının var olduğunu bilir.

### Kalıtımın gizli bedeli: taban sınıfa bağımlılık

```csharp
public class Rapor
{
    public virtual void Yaz() => Console.WriteLine("Rapor");
}

public class AylikRapor : Rapor
{
    public override void Yaz()
    {
        base.Yaz();                    // taban sınıfın davranışına bağımlı
        Console.WriteLine("Aylık ek");
    }
}
```

Taban sınıfın yazarı `Yaz()` içine yarın bir günlükleme ekler; `AylikRapor` bunu bilmeden iki kez günlük yazmaya başlar. Türeyen sınıfı sen yazdın, davranışı başkası değiştirdi. Buna **fragile base class (kırılgan taban sınıf)** problemi denir ve Salı notunun ana konusudur.

### Kalıtım kurmadan önceki tek soru

"**B gerçekten bir A mıdır?**" — cevabı "hayır ama A'nın metotları işime yarıyor" ise kalıtım yanlış araçtır.

Bu sorunun cevabı "hayır" olduğu hâlde kalıtım kurulduğunda ne olduğunu Salı notunda ayrıntılı göreceksin.

> **Bu benzetme şurada bozulur:** Çırak benzetmesi kalıtımı tek yönlü gösteriyor: usta bilgi verir, çırak alır. Gerçekte bağ çift yönlüdür. Taban sınıfın bir metodu, türeyende ezilmiş (`override` edilmiş) başka bir metodu çağırabilir — yani usta farkında olmadan çırağın yöntemini kullanır. 7. bölümdeki constructor tuzağı doğrudan bundan doğar.

---

## 5. `virtual` / `override` / `new` ve Method Hiding Tuzağı

> **Benzetme —** Bir lokantanın merkez mutfağı bütün şubelere standart bir tarif gönderir. Tarifin bazı maddelerinin yanında "şube kendi usulünce yapabilir" notu vardır — bunlar `virtual`. Şube o maddeyi kendi usulüne çevirirse `override` etmiş olur ve merkezden gelen sipariş şubenin usulüyle pişer. Ama şube, merkezin "değiştirilemez" dediği bir maddeye kendi kâğıdını yapıştırırsa (`new`), merkezden gelen sipariş hâlâ eski tarifle pişer; sadece şubeye doğrudan gelen müşteri yeni tarifi yer. İki müşteri aynı yemeği ısmarlar, iki farklı tabak gelir.

**Basitçe:** `virtual` + `override`, çağrının **nesnenin gerçek tipine** göre seçilmesidir. `new` ise metodu gizlemektir: çağrı, elindeki **değişkenin tipine** göre seçilir. İkincisi neredeyse her zaman hatadır.

**Teknik olarak:** `virtual` işaretli üye, çalışma zamanında sanal metot tablosu (v-table) üzerinden çözülür. `new` işaretli üye ise derleme zamanında, referansın statik tipine göre çözülür.

```csharp
public class Odeme
{
    public virtual string Aciklama() => "Genel ödeme";
    public         string Etiket()   => "ODEME";
}

public class KrediKarti : Odeme
{
    public override string Aciklama() => "Kredi kartı";  // ezme
    public new      string Etiket()   => "KART";         // gizleme
}
```

```csharp
Odeme o = new KrediKarti();     // değişken tipi Odeme, nesne tipi KrediKarti

o.Aciklama();                   // "Kredi kartı"  -> nesnenin tipi kazandı
o.Etiket();                     // "ODEME"        -> değişkenin tipi kazandı

KrediKarti k = new KrediKarti();
k.Etiket();                     // "KART"         -> aynı nesne, farklı cevap
```

Son iki satır tuzağın tamamıdır: **aynı nesne, referansın tipine göre farklı cevap veriyor.** Polimorfizmin verdiği sözün tam tersi.

| Anahtar kelime | Ne yapar | Çağrı neye göre çözülür |
|---|---|---|
| (hiçbiri) | Normal metot | Değişkenin statik tipi |
| `virtual` | Ezilebilir metot tanımlar | Nesnenin çalışma zamanı tipi |
| `override` | Sanal metodu ezer | Nesnenin çalışma zamanı tipi |
| `new` | Taban üyeyi gizler | Değişkenin statik tipi |
| `abstract` | Gövdesiz sanal metot — ezmek zorunlu | Nesnenin çalışma zamanı tipi |
| `sealed override` | Ezmeyi burada bitirir | Nesnenin çalışma zamanı tipi |

### `new` ne zaman meşrudur

Neredeyse hiç. Tek makul senaryo: elinde değiştiremediğin bir taban sınıf var, taban sınıf sonradan senin metodunla aynı isimde bir üye ekledi ve sen çakışmayı susturmak istiyorsun. Bunun dışında `new` yazıyorsan, muhtemelen ya isim değiştirmen ya da kompozisyona geçmen gerekiyor.

```csharp
// KÖTÜ — davranışı değiştirmek için `new` kullanmak
public class Liste { public void Ekle(string s) { } }
public class LoglayanListe : Liste
{
    public new void Ekle(string s) { Console.WriteLine(s); base.Ekle(s); }
}
Liste l = new LoglayanListe();
l.Ekle("x");    // log YAZILMAZ — sessiz hata

// İYİ — taban sınıf sanal yapılıp `override` ediliyor
public class Liste2 { public virtual void Ekle(string s) { } }
public class LoglayanListe2 : Liste2
{
    public override void Ekle(string s) { Console.WriteLine(s); base.Ekle(s); }
}
Liste2 l2 = new LoglayanListe2();
l2.Ekle("x");   // log yazılır
```

> `new` yazmayı unutursan derleyici CS0108 uyarısı verir. Bu uyarıyı `new` ekleyerek susturmak refleks hâline gelmiş bir hatadır; önce isim çakışmasını incele.

> **Bu benzetme şurada bozulur:** Şube benzetmesi `new` kullanımını "şube kendi kâğıdını yapıştırdı" diye anlatıyor; sanki şube bilerek yapmış gibi. Gerçekte `new` çoğu zaman **fark edilmeden** olur: taban sınıfa yeni bir metot eklenir, türeyendeki aynı isimli metot bir anda gizleyici hâline gelir. Kimse bir şey yapmamıştır, kod yine de sessizce yanlış çalışmaya başlar.

---

## 6. Soyutlama: `abstract class` mı `interface` mi

> **Benzetme —** İki ayrı belge düşün. Birincisi bir **sürücü belgesi**dir: "bu kişi araç kullanabilir" der, nasıl kullandığına karışmaz. İkincisi bir **çıraklık sözleşmesi**dir: hem "şunları yapacaksın" der hem de ustanın atölyesini, aletlerini, çalışma saatlerini sana verir. Sürücü belgesi birden fazla olabilir — ehliyet, yat kaptanlığı, iş makinesi. Ama tek bir atölyenin çırağı olabilirsin.

**Basitçe:** Arayüz bir **yapabilirlik** bildirir, soyut sınıf bir **kimlik** ve hazır altyapı verir. Bir tip birçok yapabilirliğe sahip olabilir ama tek bir kimliği olur.

**Teknik olarak:** İkisi de doğrudan örneklenemez (`new` ile nesne üretilemez). Farklar şurada:

| Konu | `abstract class` | `interface` |
|---|---|---|
| Kaç tanesinden türeyebilir | 1 | Sınırsız |
| Durum (alan) tutabilir mi | Evet | **Hayır** (static alan hariç) |
| Constructor'ı var mı | Evet (türeyen çağırır) | Hayır |
| Erişim belirleyici | Serbest | Üyeler varsayılan `public` |
| Gövdeli metot | Evet | .NET Core 3.0 / C# 8'den beri evet |
| `static abstract` üye | Hayır | C# 11'den beri evet |
| Sürüm eklemesi | Yeni metot türeyenleri bozmaz | Gövdesiz yeni metot **bozar** |

### Default interface method'lar neyi değiştirdi, neyi değiştirmedi

C# 8 ile arayüzlere gövdeli metot yazılabilir oldu:

```csharp
public interface IBildirimGonderici
{
    Task GonderAsync(string alici, string mesaj);

    // Default implementation — uygulayanlar yazmak zorunda değil
    Task TopluGonderAsync(IEnumerable<string> alicilar, string mesaj)
        => Task.WhenAll(alicilar.Select(a => GonderAsync(a, mesaj)));
}
```

Bunun tek gerçek amacı **sürüm uyumluluğudur**: yayınlanmış bir arayüze yeni üye eklerken, o arayüzü uygulayan mevcut kodları bozmamak. "Artık soyut sınıfa gerek kalmadı" anlamına gelmez, çünkü değişmeyen üç sınır var:

1. **Durum tutamaz.** Arayüzde instance alan tanımlanamaz; ortak bir `_sayac` gerekiyorsa arayüz çözüm değildir.
2. **Çağrılması açık dönüşüm ister.** Default metot, sınıf referansı üzerinden görünmez.
3. **Constructor yoktur.** Kurulum mantığı paylaşılamaz.

```csharp
public class SmsGonderici : IBildirimGonderici
{
    public Task GonderAsync(string alici, string mesaj) => Task.CompletedTask;
}

var s = new SmsGonderici();
// s.TopluGonderAsync(...);                     // derleme hatası — sınıfta yok
((IBildirimGonderici)s).TopluGonderAsync(new[] { "a" }, "x");   // arayüz üzerinden çalışır
```

Bu davranış şaşırtıcıdır ve default interface method'ları "soyut sınıfın yerine geçer" sanmanın önündeki en pratik engeldir.

> Default interface method sürüm ekleme aracıdır, tasarım aracı değil. Yeni bir arayüz tasarlıyorsan gövdeli metot koymaya gerek yok — arayüzü sade tut, ortak davranışı bir extension method'a ya da bir soyut temel sınıfa koy.

### Karar kuralı

Ortak **durum** ya da kurulum mantığı paylaşılacaksa `abstract class`. Paylaşılmayacaksa ve tip başka hiyerarşilere de girecekse `interface`. Test için sahte (mock) nesne yazılacaksa `interface`. Birden çok yeteneği ayrı ayrı bildirmek gerekiyorsa tek büyük arayüz değil, birden çok küçük arayüz.

### İkisi birlikte: en yaygın kalıp

```csharp
// Sözleşme: dışarıya bu görünür, testte bu sahtelenebilir
public interface IRaporUretici
{
    string Ad { get; }
    byte[] Uret(RaporIstegi istek);
}

// Ortak altyapı: akış bir kez yazıldı, değişen adım türeyene bırakıldı
public abstract class RaporUreticiTemel : IRaporUretici
{
    private readonly ILogger _logger;
    protected RaporUreticiTemel(ILogger logger) => _logger = logger;

    public abstract string Ad { get; }

    public byte[] Uret(RaporIstegi istek)      // template method
    {
        _logger.LogInformation("{Ad} üretiliyor", Ad);
        return GövdeUret(istek);               // tek değişken parça
    }

    protected abstract byte[] GövdeUret(RaporIstegi istek);
}
```

Bağımlılıklar her zaman `IRaporUretici` üzerinden verilir; `RaporUreticiTemel` sadece kod paylaşımı içindir. "Arayüze bağlan, soyut sınıftan türe" kalıbı budur.

> **Bu benzetme şurada bozulur:** Ehliyet benzetmesi arayüzü "sadece bir kâğıt" gibi gösteriyor. Ama arayüz aynı zamanda bir **davranış sözüdür**: `IDisposable` uygulayan bir tip, `Dispose()` çağrıldığında kaynağı gerçekten bırakmalıdır. İmzayı sağlamak yetmez, sözü de tutmak gerekir. Derleyici imzayı kontrol eder, sözü kontrol edemez.

---

## 7. Constructor Zinciri, `base` ve Sanal Metot Çağırma Tuzağı

> **Benzetme —** Bir inşaat. Önce temel atılır, sonra kolonlar, sonra duvarlar, en son boya. Sıra bozulamaz. Şimdi şunu düşün: temel atan usta, "duvar rengini duvarcı belirlesin" diye duvarcıyı arar — ama duvarcı henüz şantiyeye gelmemiştir. Telefon çalar, kimse açmaz. Constructor içinde sanal metot çağırmak tam olarak budur.

**Basitçe:** Nesne kurulurken önce en tepedeki taban sınıfın constructor'ı çalışır, sonra aşağı doğru inilir. Alan atamaları ise ters sırada, her sınıfın kendi constructor'ından hemen önce yapılır. Bu sıra, kurulum sırasında türeyen sınıfın hazır olmadığı bir an yaratır.

**Teknik olarak:** Constructor devralınmaz. Türeyen sınıf, taban sınıfın bir constructor'ını `base(...)` ile çağırmak zorundadır; yazmazsa derleyici parametresiz `base()` çağrısını ekler ve taban sınıfta parametresiz constructor yoksa derleme hatası alırsın.

```csharp
public class Arac
{
    public string Plaka { get; }
    public Arac(string plaka) => Plaka = plaka;
}

public class Otomobil : Arac
{
    public int KapiSayisi { get; }

    // base(...) zorunlu — Arac'ın parametresiz ctor'ı yok
    public Otomobil(string plaka, int kapiSayisi) : base(plaka)
        => KapiSayisi = kapiSayisi;
}
```

Aynı sınıf içinde constructor'lar birbirini `this(...)` ile çağırabilir:

```csharp
public class Siparis
{
    public int Id { get; }
    public DateTime Tarih { get; }

    public Siparis(int id) : this(id, DateTime.UtcNow) { }   // varsayılanı devret
    public Siparis(int id, DateTime tarih) { Id = id; Tarih = tarih; }
}
```

### Çalışma sırası

```csharp
public class Ust
{
    private readonly string _u = Yaz("Ust alan");
    public Ust() => Yaz("Ust ctor");
    protected static string Yaz(string s) { Console.WriteLine(s); return s; }
}

public class Alt : Ust
{
    private readonly string _a = Yaz("Alt alan");
    public Alt() => Yaz("Alt ctor");
}

new Alt();   // sıra: Alt alan -> Ust alan -> Ust ctor -> Alt ctor
```

Türeyen sınıfın **alanları** taban constructor'dan önce, **constructor gövdesi** ise sonra çalışır.

### Tuzak: constructor içinde sanal metot

```csharp
// KÖTÜ — taban ctor, henüz kurulmamış türeyenin metodunu çağırıyor
public class RaporTemel
{
    protected RaporTemel() => Hazirla();          // sanal çağrı
    protected virtual void Hazirla() { }
}

public class AylikRapor : RaporTemel
{
    private readonly List<string> _satirlar = new();
    protected override void Hazirla() => _satirlar.Add("başlık");  // NullReferenceException
}
```

`Hazirla()` çağrıldığı anda `_satirlar` henüz `new()` ile atanmamıştır — sıra taban constructor'dadır. Kod derlenir, çalışırken patlar.

```csharp
// İYİ — kurulum, nesne tamamen kurulduğu andan sonraya alındı
public class RaporTemel2
{
    protected RaporTemel2() { }                    // ctor hiçbir sanal şey çağırmaz
    protected virtual void Hazirla() { }

    public static T Olustur<T>() where T : RaporTemel2, new()
    {
        var r = new T();
        r.Hazirla();
        return r;
    }
}
```

> Kural: **constructor içinde `virtual` veya `abstract` üye çağırma.** Aynı şey property'ler için de geçerlidir — sanal bir property'yi ctor'da okumak da aynı tuzaktır. Analiz kuralı CA2214 bunu yakalar.

> **Bu benzetme şurada bozulur:** İnşaat benzetmesi "sıra bozulamaz" diyor ve bunu bir güvence gibi sunuyor. Ama C#'ta sıra bilinçli olarak aşağıdan yukarı **değil**, yukarıdan aşağı işler; yani taban sınıf, kendisinden sonra kurulacak parçaya erişebilir. Bu bir hata değil, dilin tercihidir — C++ bu durumda taban sınıfın kendi metodunu çağırır, C# türeyenin metodunu çağırır. Dil değiştirince tuzağın şekli değişir, varlığı değişmez.

---

## 8. Static vs Instance

> **Benzetme —** Apartmanın asansörü ile dairenin buzdolabı. Asansör binaya aittir, herkes aynısını kullanır, ikinci bir tane yoktur. Buzdolabı daireye aittir; her dairede bir tane vardır ve içindekiler dairenin kendisine özeldir. Asansöre "hangi daireden biniyorsun" diye sormazsın; buzdolabı ise hangi dairede olduğunu bilir.

**Basitçe:** `static` üye tipe aittir, örneklere değil. Bir tane vardır ve `this` yoktur. Instance üye ise her nesnede ayrı ayrı bulunur.

**Teknik olarak:**

```csharp
public class Sayac
{
    public static int ToplamUretilen;     // tip başına bir tane
    public int KendiSayaci;               // nesne başına bir tane

    public Sayac() { ToplamUretilen++; KendiSayaci = 0; }

    public static Sayac Olustur() => new Sayac();       // static: this yok
    public void Artir() => KendiSayaci++;               // instance: this var
}
```

| Konu | `static` | Instance |
|---|---|---|
| Kaç kopya | Tip başına 1 | Nesne başına 1 |
| `this` erişimi | Yok | Var |
| Instance üyeye erişim | Nesne parametre olarak verilmeden hayır | Evet |
| `virtual` olabilir mi | Sınıfta hayır; arayüzde `static abstract` C# 11'den beri var | Evet |
| Test edilebilirlik | Zor (sahtelenemez) | Kolay |
| Ne zaman başlatılır | İlk erişimde (lazy, tip başlatıcı) | `new` anında |

### Static ne zaman doğru

Durum tutmayan, saf yardımcı fonksiyonlar için:

```csharp
public static class MetinYardimcisi
{
    public static string SlugYap(string metin)
        => string.Concat(metin.ToLowerInvariant()
                              .Select(c => char.IsLetterOrDigit(c) ? c : '-')).Trim('-');
}
```

### Static ne zaman yanlış

Durum tuttuğunda ve bağımlılık olduğunda:

```csharp
// KÖTÜ — gizli bağımlılık, testte değiştirilemez, çok kanallı erişimde bozulur
public static class Ayarlar
{
    public static string BaglantiCumlesi = "";
    public static Dictionary<string, string> Onbellek = new();   // thread-safe değil
}

public class UrunServisi
{
    public void Calis() => Console.WriteLine(Ayarlar.BaglantiCumlesi);  // ctor'da görünmez
}
```

```csharp
// İYİ — bağımlılık constructor'da görünür, testte sahtelenebilir
public interface IAyarlar { string BaglantiCumlesi { get; } }

public class UrunServisi2
{
    private readonly IAyarlar _ayarlar;
    public UrunServisi2(IAyarlar ayarlar) => _ayarlar = ayarlar;
    public void Calis() => Console.WriteLine(_ayarlar.BaglantiCumlesi);
}
```

İkinci sürümde sınıfın neye ihtiyacı olduğu **imzasında** yazar. Bootcamp'te Dependency Injection konusuna geldiğinde bu fark her örnekte karşına çıkacak.

> Static alan uygulama boyunca yaşar: içine koyduğun koleksiyon asla toplanmaz — .NET'te en sık görülen bellek sızıntısı sebebi budur. Ayrıca eşzamanlı erişimde korumasızdır.

> **Bu benzetme şurada bozulur:** Asansör benzetmesi "bir tane var" diyor ve bunu masum gösteriyor. Web uygulamasında bu tekillik asıl sorundur: `static` bir alan, aynı anda çalışan yüzlerce isteğin **ortak** alanıdır. Asansöre bir kişi biner; static alana yüz kişi aynı anda biner.

---

## 9. `object`'in Metotları ve Override Sözleşmesi

> **Benzetme —** Nüfus cüzdanı. Ülkedeki herkeste vardır, formatı aynıdır: ad, soyad, TC numarası. İki kişinin aynı kişi olup olmadığını cüzdana bakarak anlarsın. Kütüphane de kitapları numaraya göre raflara dizer; iki farklı kitap aynı numarayı taşırsa aynı rafa girer ve karışıklık çıkar. `GetHashCode` o raf numarasıdır, `Equals` ise "gerçekten aynı kişi mi" kontrolüdür.

**Basitçe:** C#'taki her tip `object`'ten türer ve üç metodu hazır alır: `ToString`, `Equals`, `GetHashCode`. Varsayılan hâlleri çoğu zaman işine yaramaz; ezmen gerekir. Ama ezerken uyulması gereken bir sözleşme vardır.

**Teknik olarak:**

| Metot | Varsayılan davranış |
|---|---|
| `ToString()` | Tipin tam adını döndürür (`Uygulama.Modeller.Urun`) |
| `Equals(object?)` | Referans eşitliği (`struct` için alan alan karşılaştırma) |
| `GetHashCode()` | Nesneye özel, tahmin edilemez bir tam sayı |
| `GetType()` | Çalışma zamanı tipi — ezilemez |

### `ToString` — günlük ve hata ayıklamanın temeli

```csharp
public class Siparis
{
    public int Id { get; init; }
    public decimal Tutar { get; init; }

    public override string ToString() => $"Siparis #{Id} — {Tutar:C}";
}
```

> Günlüğe yazarken `ToString(CultureInfo.InvariantCulture)` tercih edilir; `ToString()` mevcut kültürü kullanır ve günlüğü okuyan sistemin kültürü farklı olabilir.

### `Equals` ve `GetHashCode` sözleşmesi

Bu ikisi **birlikte** ezilir. Kurallar:

1. `a.Equals(b)` doğruysa `a.GetHashCode() == b.GetHashCode()` olmak **zorunda**.
2. Tersi zorunlu değil: hash kodları eşit olan nesneler farklı olabilir (çakışma normaldir).
3. `GetHashCode`, nesne bir sözlükte anahtarken **değişmemelidir**. Bu yüzden değişebilen alanlardan hash üretme.

```csharp
public sealed class Urun : IEquatable<Urun>
{
    public string Barkod { get; }
    public string Ad { get; set; } = "";

    public Urun(string barkod) => Barkod = barkod;

    public bool Equals(Urun? other)
        => other is not null && Barkod == other.Barkod;      // sadece değişmeyen alan

    public override bool Equals(object? obj) => Equals(obj as Urun);
    public override int GetHashCode() => Barkod.GetHashCode();
    public override string ToString() => $"{Barkod} {Ad}";
}
```

`Ad` değişebildiği için kimliğe dahil edilmedi. Etseydi, sözlükte duran bir ürünün adını değiştirdiğinde o ürünü bir daha bulamazdın:

```csharp
// KÖTÜ — hash, değişebilen alandan üretiliyor
public override int GetHashCode() => (Barkod, Ad).GetHashCode();

var sozluk = new Dictionary<Urun, int>();
var u = new Urun("8690000000000") { Ad = "Çay" };
sozluk[u] = 5;
u.Ad = "Kahve";                 // hash değişti
sozluk.TryGetValue(u, out _);   // false — nesne sözlükte ama bulunamıyor
```

Birden çok alandan hash üretirken elle çarpma yapma; `HashCode.Combine` var:

```csharp
public override int GetHashCode() => HashCode.Combine(Il, Ilce, Mahalle);
```

> `GetHashCode` sonucu **kalıcı değildir**: .NET string hash'lerini uygulama başına rastgeleleştirir. Hash kodunu veritabanına yazma, dosyaya kaydetme, ağdan gönderme — sadece o çalışma içinde geçerlidir.

> **Bu benzetme şurada bozulur:** Nüfus cüzdanı benzetmesi kimliğin sabit olduğunu varsayıyor. Kodda kimliği sen seçersin ve yanlış seçebilirsin. Bir `Musteri` nesnesinin kimliği veritabanı `Id`'si midir, e-postası mıdır, yoksa aynı bellek adresinde olması mıdır? Üçü de savunulabilir ve üçü farklı sonuç verir. Bu seçim kod yazarken verilen bir tasarım kararıdır, doğası gereği verilmiş bir gerçek değil.

---

## 10. Nesne Eşitliği: Referans mı, Değer mi — ve `record`

> **Benzetme —** İki tane aynı seri numaralı iki yüz liralık banknot düşünülemez; ama iki farklı yüz liralık banknot alışverişte **aynı değerdedir**. Kasiyer için ikisi eşittir, kriminal laboratuvar için değildir. Hangi eşitliği sorduğun, ne yapmak istediğine bağlıdır.

**Basitçe:** İki tür eşitlik var. Referans eşitliği "aynı nesne mi?" diye sorar, değer eşitliği "içindekiler aynı mı?" diye sorar. `class` varsayılan olarak birinciyi, `struct` ve `record` ikinciyi yapar.

**Teknik olarak:**

| Tip | Varsayılan `==` | Varsayılan `Equals` |
|---|---|---|
| `class` | Referans eşitliği | Referans eşitliği |
| `record` (class) | **Değer eşitliği** | Değer eşitliği |
| `struct` | Tanımsız — `==` yazılmalı | Alan alan karşılaştırma (yavaş, yansıma kullanabilir) |
| `record struct` | Değer eşitliği | Değer eşitliği (üretilmiş, hızlı) |
| `string` | Değer eşitliği (özel durum) | Değer eşitliği |

```csharp
public class AdresC { public string Il = ""; public string Ilce = ""; }
public record AdresR(string Il, string Ilce);

var c1 = new AdresC { Il = "Kayseri", Ilce = "Melikgazi" };
var c2 = new AdresC { Il = "Kayseri", Ilce = "Melikgazi" };
Console.WriteLine(c1 == c2);                          // False

var r1 = new AdresR("Kayseri", "Melikgazi");
var r2 = new AdresR("Kayseri", "Melikgazi");
Console.WriteLine(r1 == r2);                          // True
Console.WriteLine(ReferenceEquals(r1, r2));           // False — ayrı nesneler
```

### `record`'un hazır getirdikleri

C# 9 ile gelen `record`, şunları derleyiciye yazdırır: değer tabanlı `Equals`/`GetHashCode`/`==`/`!=`, okunabilir bir `ToString`, `with` ifadesi ve deconstruct.

```csharp
public record Musteri(int Id, string Ad, string Sehir);

var m1 = new Musteri(1, "Ayşe", "Kayseri");
var m2 = m1 with { Sehir = "Ankara" };        // kopya + tek alan değişik
Console.WriteLine(m1);                        // Musteri { Id = 1, Ad = Ayşe, Sehir = Kayseri }
var (id, ad, _) = m1;                         // deconstruct
```

`record`, `class`'ın bir türüdür — hâlâ referans tipidir, `null` olabilir, kalıtım kurabilir. Sadece eşitlik davranışı ve birkaç kolaylık farklıdır.

> `record`'un eşitliği **çalışma zamanı tipini de** karşılaştırır: `Musteri` ile ondan türeyen `KurumsalMusteri`, alanları aynı olsa bile eşit değildir. Elle yazılan `Equals` metotlarında sık atlanan bu noktayı `record` senin yerine doğru yapar. Buna karşılık değişebilen alanı olan bir `record`'u sözlük anahtarı yapma — 9. bölümdeki hash tuzağı orada da geçerlidir.

### Ne zaman hangisi

Nesnenin bir **kimliği** varsa (veritabanı `Id`'si, yaşam döngüsü) `class` yaz; eşitliği ya hiç ezme ya da yalnızca `Id` üzerinden ez. Nesne sadece veri taşıyorsa `record` yaz: DTO, API isteği/yanıtı, sorgu sonucu, değer nesnesi. EF Core entity'leri `class` olmalıdır — change tracking kimliğe dayanır.

> **Bu benzetme şurada bozulur:** Banknot benzetmesi "değer aynıysa eşit" diyor. `record`'da eşitlik **bütün** alanlara bakar; sen hangisinin kimliğe dahil olacağını seçemezsin. Bir alanı eşitlik dışında tutmak istiyorsan (örneğin `SonGuncelleme`), `record`'un ürettiği `Equals`'ı elle ezmen gerekir — o noktada `record` kullanmanın kazancı büyük ölçüde biter.

---

## 11. `sealed` — Kapıyı Kapatmak

> **Benzetme —** Tapu dairesinde bazı kayıtların üzerinde "şerh konulamaz" notu vardır. Kimse üzerine ek koyamaz, değiştiremez. Kısıtlayıcı görünür ama işlevi nettir: o kaydın bugünkü anlamı yarın da aynı kalacaktır. Kayıt sahibi de, ona güvenerek iş yapan da bunu bilir.

**Basitçe:** `sealed`, "bu sınıftan başka sınıf türetilemez" demektir. Metot üzerinde kullanıldığında ise "bu metot bir daha ezilemez" anlamına gelir.

**Teknik olarak:**

```csharp
public sealed class ParaBirimi { }              // türetilemez
// public class X : ParaBirimi { }               // derleme hatası

public class A { public virtual void Calis() { } }
public class B : A { public sealed override void Calis() { } }   // zincir burada biter
```

`record` tipler de `sealed` olabilir; `static` sınıflar zaten örtülü olarak `sealed`'dir.

### Neden varsayılan olarak `sealed` yazmak makul

1. **Sözleşme netliği.** Türetilebilir bir sınıf yazmak, her `virtual` üyenin nasıl ezileceğini düşünmeyi gerektirir. Bunu yapmayacaksan kapıyı kapat.
2. **Kırılgan taban sınıf riskini ortadan kaldırır.** Kimse türetmediyse, sınıfın içini değiştirmen kimseyi bozmaz.
3. **Performans.** JIT, `sealed` bir tipteki sanal çağrıları doğrudan bağlayabilir (devirtualization) ve gövdesi küçükse satır içine alabilir. Tip kontrolleri (`is`, cast) de ucuzlar — çalışma zamanının hiyerarşiyi taraması gerekmez.

Kazanç tek bir çağrıda ölçülemez; milyonlarca çağrılık sıcak döngülerde fark edilir. `sealed`'i performans için değil, **tasarım netliği** için yaz; performans yan kazançtır.

> **Bu benzetme şurada bozulur:** Şerh benzetmesi `sealed`'i geri dönüşü olmayan bir karar gibi gösteriyor. Aslında tersi doğrudur: `sealed`'i kaldırmak kimseyi bozmaz (kimse türetemiyordu zaten), eklemek ise türetmiş herkesi bozar. Bu yüzden şüphedeyken `sealed` yazmak **daha az** riskli olandır.

---

## 12. Polimorfizmin Gerçek Faydası: `switch` Yerine Tip

> **Benzetme —** Apartman kapıcısı her daireye "aidat" der ve geçer. Her daire kendi tutarını kendi bilir; kapıcının elinde "3. kat 2 numara öğrenci indirimli, 5. kat dükkân fazla ödüyor" diye bir liste yoktur. Yeni bir daire açıldığında kapıcının defterine dokunulmaz. Liste tutulsaydı, her yeni daire kapıcının defterini güncellemeyi gerektirirdi.

**Basitçe:** Polimorfizm, tipe göre dallanmayı ortadan kaldırır. Yeni bir tip eklediğinde çağıran kodu değiştirmezsin. Kazanç kısa kod değil, **değişmeyen kod**tur.

**Teknik olarak:**

```csharp
// KÖTÜ — tip başına switch. Yeni ödeme türü = bu metoda dokunmak
public decimal KomisyonHesapla(Odeme odeme)
{
    switch (odeme.Tur)
    {
        case OdemeTuru.Nakit:       return 0m;
        case OdemeTuru.KrediKarti:  return odeme.Tutar * 0.018m;
        case OdemeTuru.Havale:      return 2.50m;
        default: throw new NotSupportedException();
    }
}
```

```csharp
// İYİ — davranış tipin kendisinde
public abstract class Odeme
{
    public decimal Tutar { get; init; }
    public abstract decimal Komisyon();
    public abstract bool IadeEdilebilir { get; }
}

public sealed class NakitOdeme : Odeme
{
    public override decimal Komisyon() => 0m;
    public override bool IadeEdilebilir => false;
}

public sealed class KartOdeme : Odeme
{
    public override decimal Komisyon() => Tutar * 0.018m;
    public override bool IadeEdilebilir => true;
}

public sealed class HavaleOdeme : Odeme
{
    public override decimal Komisyon() => 2.50m;
    public override bool IadeEdilebilir => true;
}
```

Çağıran taraf artık tek satırdır ve yeni tür geldiğinde değişmez:

```csharp
decimal ToplamKomisyon(IEnumerable<Odeme> odemeler) => odemeler.Sum(o => o.Komisyon());
```

`abstract` üye sayesinde derleyici de yanında durur: yeni bir `Odeme` türü yazdığında `Komisyon()` ve `IadeEdilebilir` yazmazsan **derlenmez**. Switch'te unuttuğun `case` sessizce `default`'a düşerdi.

### Polimorfizmin diğer yüzü: arayüz üzerinden

Kalıtım şart değil. MvcCv'deki controller'ların repository'ye bağlanma biçimi aynı fikrin arayüz hâlidir:

```csharp
public class DeneyimController : Controller
{
    private readonly IDeneyimRepository _repo;      // somut sınıf değil, sözleşme
    public DeneyimController(IDeneyimRepository repo) => _repo = repo;

    public IActionResult Index() => View(_repo.List());
}
```

Controller, arkasında EF Core mu, önbellekli bir sarmalayıcı mı, testteki sahte bir nesne mi olduğunu bilmez. Çalışma zamanında hangi nesne verilirse onun metodu çalışır — polimorfizmin ta kendisi.

### `switch` ne zaman hâlâ doğru

Tip kümesi **kapalı** ve davranış tipin sorumluluğu **değilse**. Dışarıdan gelen bir HTTP durum kodunu ya da bir `enum` değerini ele alırken tipe dağıtmaya çalışmak gereksiz karmaşıklık üretir; C# 9'dan itibaren desen eşleme ile yazılan `switch` ifadesi bu işi okunaklı biçimde yapar.

Ölçüt şudur: davranış **verinin kendisine** aitse polimorfizm, **işleyen tarafa** aitse switch.

> **Bu benzetme şurada bozulur:** Kapıcı benzetmesi her dairenin kendi tutarını bilmesini doğal gösteriyor. Kodda her zaman böyle değildir. Ödeme tipine PDF makbuz basma kodunu koymak, ödeme sınıfını PDF kütüphanesine bağlar. Bu durumda davranış tipe değil, ayrı bir işleyiciye aittir — ve orada switch ya da ziyaretçi (visitor) kalıbı doğru cevaptır. Polimorfizm bir refleks değil, "bu davranış gerçekten bu tipin sorumluluğu mu?" sorusunun cevabıdır.

---

## Tek Bakışta Özet

- Dört sütun ezberlenecek tanım değil; her biri somut bir bakım sorununun çözümüdür.
- Kapsülleme veriyi saklamaz, veriye giden yolu tek kapıya indirir — kural tek yerde kalır.
- Public field yazma; auto-property bugün aynı, yarın farklıdır. Türetilmiş değeri saklama, computed property ile hesapla; içinde sorgu olan property zaten bir metottur.
- Constructor içinde sanal üye çağırma — türeyenin alanları henüz atanmamıştır.
- `override` nesnenin tipine, `new` değişkenin tipine bakar; `new` çoğu zaman sessiz hatadır.
- Arayüz yapabilirlik bildirir, soyut sınıf kimlik ve ortak altyapı verir; default interface method sürüm aracıdır, tasarım aracı değil.
- `Equals` ezersen `GetHashCode` da ez; hash'i **değişmeyen** alanlardan üret, `HashCode.Combine` kullan.
- `class` referans eşitliği, `record` değer eşitliği yapar; `record` ayrıca `with`, `ToString` ve deconstruct getirir.
- `sealed` şüphedeyken doğru varsayılandır: sözleşmeyi netleştirir, sonradan kaldırmak kimseyi bozmaz.
- Tip başına `switch` çoğalıyorsa davranışı tipe taşı; davranış işleyen tarafa aitse switch kalsın.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Encapsulation (kapsülleme) | Veriye erişimi tek kapıya indirip kuralı orada toplama |
| Inheritance (kalıtım) | Bir tipin başka bir tipin üyelerini devralması ve onun yerine geçebilmesi |
| Polymorphism (çok biçimlilik) | Aynı çağrının nesnenin gerçek tipine göre farklı davranması |
| Backing field | Bir property'nin değeri sakladığı gizli ya da açık alan |
| `init` | Sadece nesne kurulurken atanabilen property erişimcisi |
| Virtual method | Türeyen sınıfta ezilebilen, çalışma zamanında çözülen metot |
| Method hiding | `new` ile taban üyeyi gizleme; çağrı statik tipe göre çözülür |
| Default interface method | Arayüzde gövdesi olan üye (C# 8) — sürüm uyumluluğu aracı |
| Template method | Akışı taban sınıfın tuttuğu, değişen adımları türeyene bıraktığı kalıp |
| Assembly | Derlenmiş çıktı birimi (`.dll` / `.exe`) — `internal`'ın sınırı |
| Auto-property | Backing field'ı derleyicinin ürettiği `{ get; set; }` yazımı |
| Değer eşitliği | İki nesnenin içeriğinin aynı olması |
| `record` | Değer eşitliği, `with` ve okunabilir `ToString` üreten referans tipi |

---

## Sık Karıştırılanlar

| Yanlış bilinen | Doğrusu |
|---|---|
| "Kapsülleme güvenlik sağlar" | Bakım kolaylığı sağlar; `private` alan yansıma ile okunur |
| "`protected internal` dar, `private protected` geniş" | Tam tersi — `private` geçen daha dardır |
| "`new` ile metodu ezmiş olurum" | Ezmezsin, gizlersin; taban tip üzerinden çağrıda eski metot çalışır |
| "Constructor da devralınır" | Devralınmaz; `base(...)` ile çağrılır |
| "Taban constructor'dan sanal metot çağırmak güvenli" | Türeyenin alanları henüz atanmamıştır — `NullReferenceException` |
| "Default interface method geldi, soyut sınıfa gerek kalmadı" | Arayüz hâlâ durum tutamaz, ctor'ı yoktur, default metot sınıf referansından görünmez |
| "`Equals` ezmek yeterli" | `GetHashCode` da ezilmeli; yoksa sözlük ve küme davranışı bozulur |
| "`sealed` esnekliği öldürür, yazmamak daha güvenli" | `sealed` eklemek zor, kaldırmak kolaydır; şüphedeyken yazmak daha az risklidir |

---

## Sonraki

→ `02-Kompozisyon-ve-Kalitim.md` (Salı)
