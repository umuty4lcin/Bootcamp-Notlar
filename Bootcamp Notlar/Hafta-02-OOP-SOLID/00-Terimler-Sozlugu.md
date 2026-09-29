# Hafta 2 — Terimler Sözlüğü: OOP, SOLID ve Tasarım Kalıpları

**Ne işe yarar:** Bu dosya baştan sona okunmak için değil, **aranmak** için.
Haftanın 7 notundaki sözlükler burada birleştirildi: **88 terim**.
Aynı terim birden çok notta geçtiyse ilk tanımı alındı; "Nerede" sütunu
terimin ayrıntılı anlatıldığı dosyayı gösterir.

> Ctrl+F ile ara. Bir terimi bulamıyorsan başka haftanın sözlüğünde olabilir.

---

| Terim | Tanım | Nerede |
|---|---|---|
| Abstract Factory | Birbiriyle uyumlu ürün ailesini birlikte üreten kalıp | `05-Creational-Patterns.md` |
| Adapter | Uyumsuz bir arayüzü beklenen arayüze çeviren kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Aktör (actor) | Değişikliği talep eden taraf; departman ya da rol | `03-SOLID-SRP-OCP-LSP.md` |
| Alt seviye modül | Mekanizmayı taşıyan kod (SQL, SMTP, dosya) | `04-SOLID-ISP-DIP-ve-DI.md` |
| Assembly | Derlenmiş çıktı birimi (`.dll` / `.exe`) — `internal`'ın sınırı | `01-OOP-Pratikte.md` |
| Auto-property | Backing field'ı derleyicinin ürettiği `{ get; set; }` yazımı | `01-OOP-Pratikte.md` |
| Backing field | Bir property'nin değeri sakladığı gizli ya da açık alan | `01-OOP-Pratikte.md` |
| Captive dependency | Kısa ömürlü servisin uzun ömürlüye hapsolması | `04-SOLID-ISP-DIP-ve-DI.md` |
| Chain of Responsibility | İsteği işleyici zincirinden sırayla geçiren kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Cohesion (uyum) | Bir sınıf içindeki parçaların aynı işe hizmet etme derecesi | `03-SOLID-SRP-OCP-LSP.md` |
| Command | İşlemi nesne olarak temsil eden kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Composite | Parça ile bütünü aynı arayüzden kullandıran kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Composition root | Nesne grafiğinin kurulduğu tek nokta | `04-SOLID-ISP-DIP-ve-DI.md` |
| Constructor injection | Bağımlılığın constructor parametresiyle verilmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| Coupling (bağlılık) | İki kod parçasının birbirine yapışıklığı | `03-SOLID-SRP-OCP-LSP.md` |
| Decorator | Aynı arayüzü uygulayıp içerideki nesneyi sarmalayan kalıp | `02-Kompozisyon-ve-Kalitim.md` |
| Default interface method | Arayüzde gövdesi olan üye (C# 8) — sürüm uyumluluğu aracı | `01-OOP-Pratikte.md` |
| Delegasyon | Gelen çağrıyı içerideki parçaya devretme | `02-Kompozisyon-ve-Kalitim.md` |
| Değer eşitliği | İki nesnenin içeriğinin aynı olması | `01-OOP-Pratikte.md` |
| Değişmez (invariant) | Nesne ömrü boyunca doğru kalan kural | `03-SOLID-SRP-OCP-LSP.md` |
| DI | Bağımlılığın nesneye dışarıdan teslim edilmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| DIP | Üst ve alt seviyenin ortak bir soyutlamaya bağlanması | `04-SOLID-ISP-DIP-ve-DI.md` |
| Double-checked locking | Kilit almadan önce ve aldıktan sonra iki kez kontrol eden tekil üretim tekniği | `05-Creational-Patterns.md` |
| Encapsulation (kapsülleme) | Veriye erişimi tek kapıya indirip kuralı orada toplama | `01-OOP-Pratikte.md` |
| Extension method | Tipi değiştirmeden dışarıdan metot ekleme yolu | `02-Kompozisyon-ve-Kalitim.md` |
| Facade | Karmaşık alt sistemin önüne tek arayüz koyan kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Factory Method | Nesne üretimini bir metoda devreden, somut tipten ayıran kalıp | `05-Creational-Patterns.md` |
| Fat controller | İş kuralı ve altyapı taşıyan şişman controller | `03-SOLID-SRP-OCP-LSP.md` |
| Fragile base class | Taban sınıftaki değişikliğin türeyenleri sessizce bozması | `02-Kompozisyon-ve-Kalitim.md` |
| God class | Çok fazla sorumluluk biriktirmiş dev sınıf | `03-SOLID-SRP-OCP-LSP.md` |
| Header interface | Somut sınıfın tüm metotlarını kopyalayan arayüz | `04-SOLID-ISP-DIP-ve-DI.md` |
| Immutable | Oluşturulduktan sonra değişmeyen nesne | `03-SOLID-SRP-OCP-LSP.md` |
| Inheritance (kalıtım) | Bir tipin başka bir tipin üyelerini devralması ve onun yerine geçebilmesi | `01-OOP-Pratikte.md` |
| `init` | Sadece nesne kurulurken atanabilen property erişimcisi | `01-OOP-Pratikte.md` |
| Interface Segregation | Büyük arayüzü yeteneklere göre bölme prensibi | `02-Kompozisyon-ve-Kalitim.md` |
| Invariant (değişmez) | Nesnenin ömrü boyunca bozulmaması gereken kural | `02-Kompozisyon-ve-Kalitim.md` |
| IoC | Kontrolün çağıran taraftan framework'e/dışarıya geçmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| IoC container | Kayıtlara bakarak nesne grafiğini kuran bileşen | `04-SOLID-ISP-DIP-ve-DI.md` |
| is-a / has-a | Kimlik ilişkisi / sahiplik ilişkisi | `02-Kompozisyon-ve-Kalitim.md` |
| ISP | Kimse kullanmadığı metoda bağlı kalmamalı | `04-SOLID-ISP-DIP-ve-DI.md` |
| Iterator | Koleksiyonu iç yapısını açmadan gezdiren kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Kapsülleme | İç durumu dışarıdan saklayıp erişimi kontrollü hâle getirme | `07-Hafta-Ozeti.md` |
| Kompozisyon | Bir sınıfın işini, içinde tuttuğu başka nesnelere yaptırması | `02-Kompozisyon-ve-Kalitim.md` |
| `Lazy<T>` | Gecikmeli ve iş parçacığı güvenli başlatma sağlayan .NET tipi | `05-Creational-Patterns.md` |
| Leaf (yaprak) sınıf | Kendisinden türetilmeyen, hiyerarşinin ucundaki sınıf | `02-Kompozisyon-ve-Kalitim.md` |
| LSP | Alt tipin, üst tipin yerine sorunsuz geçebilmesi prensibi | `02-Kompozisyon-ve-Kalitim.md` |
| Method hiding | `new` ile taban üyeyi gizleme; çağrı statik tipe göre çözülür | `01-OOP-Pratikte.md` |
| Method injection | Bağımlılığın metot parametresiyle verilmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| Mock | Çağrının yapıldığını doğrulamak için kullanılan sahte nesne | `04-SOLID-ISP-DIP-ve-DI.md` |
| N+1 problemi | Bir sorgu + her satır için ek sorgu üreten kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| Null object | Hiçbir şey yapmayan güvenli varsayılan uygulama | `04-SOLID-ISP-DIP-ve-DI.md` |
| Observer | Durum değişikliğini abonelere bildiren kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| OCP | Açık/kapalı: genişlemeye açık, değişikliğe kapalı | `03-SOLID-SRP-OCP-LSP.md` |
| Onion / Clean Architecture | Bağımlılıkların içeriye, iş kuralına doğru aktığı katman düzeni | `04-SOLID-ISP-DIP-ve-DI.md` |
| Over-engineering | Problemden büyük çözüm kurma | `03-SOLID-SRP-OCP-LSP.md` |
| Polimorfizm | Aynı çağrının tipe göre farklı davranması | `03-SOLID-SRP-OCP-LSP.md` |
| Polymorphism (çok biçimlilik) | Aynı çağrının nesnenin gerçek tipine göre farklı davranması | `01-OOP-Pratikte.md` |
| Property injection | İsteğe bağlı bağımlılığın property ile verilmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| Proxy | Gerçek nesnenin yerine geçip erişimi denetleyen kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| `record` | Değer eşitliği, `with` ve okunabilir `ToString` üreten referans tipi | `01-OOP-Pratikte.md` |
| Rol arayüzü | Tek bir yeteneği tanımlayan dar arayüz | `07-Hafta-Ozeti.md` |
| Role interface | Tek bir yeteneği tanımlayan küçük arayüz | `04-SOLID-ISP-DIP-ve-DI.md` |
| Rule of three | Aynı kalıbı üçüncü kez yazınca soyutla | `03-SOLID-SRP-OCP-LSP.md` |
| Service locator | Bağımlılığın merkezi bir sağlayıcıdan çalışma anında istenmesi | `04-SOLID-ISP-DIP-ve-DI.md` |
| Service Locator | Konteyneri sınıfa enjekte edip tip çözme; bağımlılığı gizlediği için anti-pattern | `05-Creational-Patterns.md` |
| Singleton | Sınıfın tek örneğini garanti eden ve global erişim sunan kalıp | `05-Creational-Patterns.md` |
| SOLID | Nesne yönelimli tasarımın beş ilkesi | `03-SOLID-SRP-OCP-LSP.md` |
| Son koşul (postcondition) | Metot bittikten sonra garanti edilenler | `03-SOLID-SRP-OCP-LSP.md` |
| Speculative generality | Var olmayan ihtiyaç için kurulan soyutlama | `03-SOLID-SRP-OCP-LSP.md` |
| Spekülatif soyutlama | İleride lazım olur diye kurulan, kullanılmayan soyutlama | `07-Hafta-Ozeti.md` |
| SRP | Tek sorumluluk: değişmek için tek sebep | `03-SOLID-SRP-OCP-LSP.md` |
| Strategy | Değişen davranışı dışarıdan parametre olarak alan kalıp | `02-Kompozisyon-ve-Kalitim.md` |
| Strategy pattern | Değişken davranışı ayrı sınıflara taşıyan tasarım deseni | `03-SOLID-SRP-OCP-LSP.md` |
| Stub | Önceden belirlenmiş cevabı döndüren sahte nesne | `04-SOLID-ISP-DIP-ve-DI.md` |
| Sıkı bağ (tight coupling) | Bir tipin başka bir tipin iç detaylarına bağımlı olması | `02-Kompozisyon-ve-Kalitim.md` |
| Tasarım kalıbı | Tekrarlayan tasarım problemine yerleşmiş çözüm ve ortak dil | `07-Hafta-Ozeti.md` |
| Telescoping constructor | Parametre sayısı artarak çoğalan constructor aşırı yüklemeleri | `05-Creational-Patterns.md` |
| Template method | Akışı taban sınıfın tuttuğu, değişen adımları türeyene bıraktığı kalıp | `01-OOP-Pratikte.md` |
| Template Method | Akışı taban sınıfta sabitleyip adımları alt sınıfa bırakan kalıp | `06-Structural-ve-Behavioral-Patterns.md` |
| TimeProvider | .NET 8 ile gelen, saati soyutlayan standart tip | `03-SOLID-SRP-OCP-LSP.md` |
| Transient / Scoped / Singleton | Her istekte yeni / istek başına bir / uygulama boyunca tek | `04-SOLID-ISP-DIP-ve-DI.md` |
| Virtual method | Türeyen sınıfta ezilebilen, çalışma zamanında çözülen metot | `01-OOP-Pratikte.md` |
| YAGNI | "İhtiyacın olmayacak" — erken soyutlamaya karşı kural | `03-SOLID-SRP-OCP-LSP.md` |
| `yield return` | Derleyicinin durum makinesi ürettiği tembel üretim sözdizimi | `06-Structural-ve-Behavioral-Patterns.md` |
| Ön koşul (precondition) | Metot çağrılmadan önce sağlanması gerekenler | `03-SOLID-SRP-OCP-LSP.md` |
| Ön koşul / son koşul | Metodun çalışması için gereken şart / çalıştıktan sonra garanti ettiği durum | `07-Hafta-Ozeti.md` |
| Ürün ailesi | Birlikte kullanılması zorunlu, birbirini varsayan nesneler kümesi | `05-Creational-Patterns.md` |
| Üst seviye modül | İş kuralını, politikayı taşıyan kod | `04-SOLID-ISP-DIP-ve-DI.md` |

---

Haftanın özet ve tuzak listesi için: [`00-Hizli-Tekrar.md`](00-Hizli-Tekrar.md)

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
