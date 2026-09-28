# Hafta 0 · Cumartesi — Ortam Kurulumu

**Okuma süresi:** ~18 dk · **Uygulama süresi:** ~4,5 saat
**Neden bu konu:** Bootcamp boyunca yazacağın her satır kod, bugün kurduğun ortamın üzerinde çalışacak. Kurulum bir kez düzgün yapılırsa bir daha aklına gelmez; yarım bırakılırsa her hafta bir saatini yer.

---

## Önce Basitçe

Bir marangoz atölyesi düşün. Tezgâhı kurmadan, testereyi bilemeden, elektriği çekmeden tek bir sandalye bile yapamazsın. Ama bunları bir kez düzgün yaparsan yıllarca dönüp bakmazsın. Bugün yaptığın şey tam olarak bu: kendi atölyeni kuruyorsun. Kod yazmıyorsun, kod yazabileceğin yeri hazırlıyorsun.

Kuracağın şeyler dört gruba ayrılıyor. Birincisi **motor**: yazdığın kodu gerçekten çalıştıran şey. İkincisi **tezgâh**: kodu yazdığın, hatalarını gördüğün program. Üçüncüsü **depo**: verilerin durduğu yer. Dördüncüsü de **yan aletler**: tek başına iş yapmayan ama olmayınca işi tıkayan küçük araçlar.

Sıra önemli. Atölyede önce elektriği çekersin, sonra makineyi getirirsin. Burada da önce motoru (.NET SDK) kurarsın, çünkü diğer araçların çoğu "bu bilgisayarda .NET var mı" diye bakar. Sırayı bozarsan kurulum hata vermez ama araçlar birbirini görmez, sen de sebebini ararsın.

Kurulum bittikten sonra bir şey daha var: gerçekten çalışıyor mu diye bakmak. Yeni bir daireye taşındığında bütün muslukları açar, prizleri denersin. Yazılımdaki karşılığı **doğrulama turu**: her aracın kendi sürüm komutunu çalıştırıp cevap verdiğini görmek, sonra da uçtan uca küçük bir proje çalıştırmak. Bu notun ikinci yarısı buna ayrılmış.

Son olarak, bir düzen kurarsın. Atölyede vidalar bir çekmecede, kalaslar bir köşede durur. Projelerin de öyle: hangi dosya nerede, hangi proje hangi repoda, baştan belli olsun. Sonradan toparlamak her zaman daha pahalıdır.

Şimdi detaya iniyoruz.

> **Ana benzetme:** Ortam kurulumu, atölyeye tezgâh kurmaktır. Bir kez doğru kurulur, yıllarca üzerinde çalışılır; eksik kurulan her parça sonraki her işte ayağına dolanır.

---

## Bu Notta Ne Var

1. Kurulacaklar (sırayla) — .NET SDK, Visual Studio, SQL Server, yan araçlar
2. Doğrulama turu — kurduğun şeyin gerçekten çalıştığını kanıtlama
3. Klasör ve repo düzeni — yerelde ve GitHub'da neyin nerede duracağı
4. Yaygın kurulum hataları — belirti, sebep, çözüm
5. Bugünün çıktısı — günün sonunda elinde ne olmalı
6. Tek bakışta özet, terimler sözlüğü, sık karıştırılanlar

---

## 1. Kurulacaklar (sırayla)

> **Benzetme —** Bir binayı temelden çatıya doğru yaparsın. Önce temel atılır, sonra kolonlar dikilir, sıva en sona kalır. Sıvayı kolondan önce yapmaya kalkarsan iş bitmez, baştan başlarsın. Kurulum sırası da böyle: altta duran şey önce gelir.

**Basitçe:** Önce kodu çalıştıran motoru kurarsın, sonra kod yazdığın programı, sonra verinin durduğu depoyu, en son da küçük yardımcı araçları. Her adımda "kuruldu mu" diye kontrol edersin, sonrakine öyle geçersin.

**Teknik olarak:** Aşağıdaki bileşenler bağımlılık sırasına göre dizilmiştir. Visual Studio kurulumu kendi içinde bir .NET SDK getirir, ama komut satırından bağımsız çalışabilmek için SDK'yı ayrıca kurmak ve PATH'te görünür olduğunu doğrulamak işini kolaylaştırır.

### 1.1 .NET SDK

Sürüm olarak akademinin kullandığını takip et; belirtilmediyse en güncel **LTS** sürüm.

```bash
dotnet --info          # SDK ve runtime listesi
dotnet --list-sdks
```

Beklenen: en az bir SDK satırı ve bir runtime satırı. Boşsa PATH sorunu var.

Runtime'ları ayrıca listelemek istersen:

```bash
dotnet --list-runtimes
```

Burada üç ayrı runtime görmen normaldir:

| Runtime | Ne için |
|---|---|
| `Microsoft.NETCore.App` | Konsol ve sınıf kütüphaneleri — çekirdek |
| `Microsoft.AspNetCore.App` | Web uygulamaları ve API'ler |
| `Microsoft.WindowsDesktop.App` | WPF ve WinForms |

> **SDK ile Runtime farkı:** **Runtime (çalışma zamanı)** derlenmiş bir .NET uygulamasını *çalıştırır*. **SDK (yazılım geliştirme kiti)** ise derleyiciyi, `dotnet` komut satırı aracını ve şablonları içerir; yani kodu *üretir*. Geliştirici olarak sana SDK lazım, SDK zaten içinde runtime'ı getirir. Kullanıcıya sadece runtime yeter.

Birden fazla SDK sürümü kuruluysa, bir klasörde hangisinin kullanılacağını `global.json` ile sabitleyebilirsin:

```bash
dotnet new globaljson --sdk-version 8.0.400
```

Üretilen dosya şuna benzer:

```json
{
  "sdk": {
    "version": "8.0.400",
    "rollForward": "latestFeature"
  }
}
```

Bu dosya bulunduğu klasörde ve tüm alt klasörlerinde geçerlidir. Ekip projelerinde "bende çalışıyor, sende çalışmıyor" vakalarının önemli bir kısmını bu dosya önler.

### 1.2 Visual Studio 2022 Community

Kurulumda seçilecek **workload**'lar:

- ASP.NET ve web geliştirme ← **zorunlu**
- .NET masaüstü geliştirme
- .NET Multi-platform App UI (MAUI) ← Proje 19 için, şimdi kurmasan da olur
- Veri depolama ve işleme (SQL Server Data Tools)

> Rider veya VS Code + C# Dev Kit de çalışır, ama bootcamp anlatımları büyük ihtimalle VS 2022 üzerinden gidecek. En az bir kez VS'yi de kullanabiliyor ol.

**Workload (iş yükü)** — Birbiriyle ilgili bileşenlerin paketlenmiş hâli. Tek tek bileşen seçmek yerine "web geliştirme yapacağım" dersin, gereken derleyici, şablon ve hata ayıklayıcı birlikte gelir. Sonradan eklemek için Visual Studio Installer'ı açıp **Modify** demen yeterlidir; baştan kurmaya gerek yok.

Kurulum bittikten sonra Visual Studio'nun hangi SDK'ları gördüğünü şuradan doğrularsın: **Help → About Microsoft Visual Studio** ve yeni bir konsol projesi açarken çıkan **Framework** listesi.

### 1.3 SQL Server + SSMS

- **SQL Server 2022 Developer Edition** (ücretsiz, tam özellikli)
- **SQL Server Management Studio (SSMS)** — ayrı indirilir
- Kurulumda **Mixed Mode Authentication** seç, `sa` şifresini bir yere not et

Doğrulama: SSMS'te `localhost` veya `.\SQLEXPRESS` ile bağlan, şunu çalıştır:

```sql
SELECT @@VERSION;
```

**Neden Developer Edition:** Express sürümü ücretsizdir ama veritabanı boyutu ve bellek kullanımı sınırlıdır. Developer Edition, Enterprise ile aynı özelliklere sahiptir ve geliştirme/test için ücretsizdir; sadece üretimde kullanılamaz. Bootcamp boyunca sınıra takılmamak için Developer Edition daha rahat.

**Mixed Mode Authentication (karma kimlik doğrulama)** — SQL Server'a iki şekilde bağlanılabilmesi: Windows hesabınla ve SQL Server'ın kendi kullanıcı/şifre çiftiyle. Sadece Windows kimlik doğrulaması açık kalırsa, Docker içinden veya farklı bir kullanıcıyla bağlanman gerektiğinde tıkanırsın.

SSMS açmadan, komut satırından da kontrol edebilirsin:

```bash
sqlcmd -S localhost -E -Q "SELECT @@VERSION;"
```

`-E` Windows kimlik doğrulaması demektir. `sa` ile bağlanacaksan:

```bash
sqlcmd -S localhost -U sa -P "SifreniBuraya" -Q "SELECT name FROM sys.databases;"
```

İleride kodda kullanacağın **connection string (bağlantı cümlesi)** şuna benzeyecek:

```text
Server=localhost;Database=KutuphaneDb;Trusted_Connection=True;TrustServerCertificate=True;
```

> `TrustServerCertificate=True` yerel geliştirmede sertifika hatasını susturur. Üretimde kullanılmaz — orada gerçek bir sertifika olur.

### 1.4 Yan araçlar

| Araç | Ne için |
|---|---|
| Git for Windows | Sürüm kontrolü |
| GitHub hesabı + SSH key | Repo yönetimi |
| Postman veya Insomnia | API test (Hafta 5'ten itibaren şart) |
| VS Code | Markdown, JS/TS, hafif düzenleme |
| Node.js LTS | Hafta 6 (Angular) için — şimdi kur, sonra uğraşma |
| Docker Desktop | Hafta 6 için. WSL2 backend gerekir |

Bu araçların çoğunu tek tek indirmek yerine Windows'un paket yöneticisiyle kurabilirsin:

```bash
winget install Git.Git
winget install OpenJS.NodeJS.LTS
winget install Microsoft.VisualStudioCode
winget install Postman.Postman
winget install Docker.DockerDesktop
```

Kurulumdan sonra terminali kapatıp açmayı unutma; PATH değişiklikleri açık olan terminale yansımaz.

Git'i kurar kurmaz kimliğini tanımla — ilk commit'ini atmadan önce yapılması gereken tek ayar budur:

```bash
git config --global user.name "Adin Soyadin"
git config --global user.email "mail@ornek.com"
git config --global init.defaultBranch main
git config --list                       # ayarları doğrula
```

SSH anahtarı üretmek ve GitHub'a tanıtmak için:

```bash
ssh-keygen -t ed25519 -C "mail@ornek.com"
cat ~/.ssh/id_ed25519.pub               # çıkan metni GitHub > Settings > SSH keys altına yapıştır
ssh -T git@github.com                   # bağlantı testi
```

Son komut "Hi kullanici! You've successfully authenticated" derse anahtar çalışıyor demektir.

---

## 2. Doğrulama Turu

> **Benzetme —** Yeni bir daireye taşındığında eşyaları yerleştirmeden önce bütün muslukları açar, prizleri dener, kombiyi yakarsın. Amacın su akıtmak değil, "akıyor mu" diye bakmaktır. Beş dakikalık bu tur, iki hafta sonra ortaya çıkacak sürprizi bugün ortaya çıkarır.

**Basitçe:** Kurduğun her aracı tek tek çağırıp "orada mısın" diye sorarsın. Hepsi cevap verdikten sonra da küçük bir proje oluşturup çalıştırırsın. Bu ikincisi asıl testtir: parçaların tek tek çalışması yetmez, birlikte çalışması gerekir.

**Teknik olarak:** Aşağıdaki komutlar aracın hem kurulu olduğunu hem de PATH üzerinden erişilebildiğini birlikte doğrular. Bir araç kurulu olup PATH'te olmayabilir; o durumda kendi kurulum klasöründen çalışır ama komut satırından "bulunamadı" hatası alırsın.

Her satır hatasız çalışmalı:

```bash
dotnet --version
git --version
node --version
npm --version
docker --version
```

Beklenen çıktı biçimi kabaca şöyledir:

```text
8.0.400
git version 2.45.2.windows.1
v20.15.1
10.7.0
Docker version 27.0.3, build 7d4bcd8
```

Docker'ın sadece kurulu olması yetmez, motorunun da ayakta olması gerekir:

```bash
docker run --rm hello-world
```

Ardından bir uçtan uca test:

```bash
dotnet new mvc -o SanityCheck
cd SanityCheck
dotnet run
```

Tarayıcıda varsayılan MVC sayfası açılıyorsa ortam hazır. Sonra klasörü sil.

Aynı testi API tarafı için de yapmak istersen:

```bash
dotnet new webapi -o ApiCheck
cd ApiCheck
dotnet run
```

Konsolda yazan `https://localhost:xxxx` adresine `/swagger` ekleyip tarayıcıda açarsan API arayüzünü görürsün. Bu, ASP.NET Core runtime'ının ve geliştirme sertifikasının çalıştığını da kanıtlar.

Sertifika uyarısı alırsan yerel geliştirme sertifikasını bir kez yenilemen yeterli:

```bash
dotnet dev-certs https --trust
```

---

## 3. Klasör ve Repo Düzeni

> **Benzetme —** Bir dosya dolabı düşün. Faturalar bir gözde, sözleşmeler başka gözde, her yıl kendi klasöründe. Kimse bunu "düzen olsun" diye yapmaz; iki yıl sonra bir kâğıdı otuz saniyede bulmak için yapar. Klasör düzeni de bugün için değil, üçüncü ay için kurulur.

**Basitçe:** Notlar bir yerde, projeler başka yerde, kaynaklar ayrı bir yerde dursun. Küçük denemeler tek bir repoda toplanır; ciddi projelerin her biri kendi reposunda yaşar. Böylece işverene "şu projeye bak" derken tek bir bağlantı verirsin.

**Teknik olarak:** Aşağıdaki yapı, notların ve projelerin birbirine karışmamasını sağlar. Numaralı klasör adları (`01-`, `02-`) dosya gezgininde sıralamanın alfabetik değil mantıksal olmasını sağlar.

Yerelde:

```
BootCamp/
├── 01-Yol-Haritasi/
├── 02-Ders-Notlari/
├── 03-Projeler/
│   ├── H1-KutuphaneCLI/
│   ├── H2-Refactor/
│   └── ...
└── 04-Kaynaklar/
```

GitHub'da:

- `bootcamp-hazirlik` → kuluçka dönemi notları ve küçük denemeler (tek repo)
- Her ciddi proje **kendi reposunda** — portfolyoda 20 proje 20 repo demek

Proje klasörünün içi de bir düzene oturur. .NET tarafında yaygın kalıp şudur:

```
H1-KutuphaneCLI/
├── src/
│   └── Kutuphane.Cli/
│       └── Kutuphane.Cli.csproj
├── tests/
│   └── Kutuphane.Cli.Tests/
│       └── Kutuphane.Cli.Tests.csproj
├── .gitignore
├── README.md
└── Kutuphane.sln
```

Bu yapıyı komut satırından kurmak birkaç satır sürer:

```bash
dotnet new sln -n Kutuphane
dotnet new console -o src/Kutuphane.Cli
dotnet new xunit   -o tests/Kutuphane.Cli.Tests
dotnet sln add src/Kutuphane.Cli/Kutuphane.Cli.csproj
dotnet sln add tests/Kutuphane.Cli.Tests/Kutuphane.Cli.Tests.csproj
dotnet build
```

**Solution (çözüm, `.sln`)** — Birden fazla projeyi bir arada tutan kapsayıcı dosya. Kendisi kod içermez; hangi projelerin aynı işin parçası olduğunu söyler. Visual Studio bir `.sln` açar, `dotnet` komutu ise tek tek `.csproj` ile de çalışabilir.

---

## 4. Yaygın Kurulum Hataları

> **Benzetme —** Evde ışıklar gitti. Önce komşuya bakarsın: onlarda da yoksa sorun senin dairende değildir. Sonra sigortaya bakarsın. Arıza aramak, ihtimalleri ucuzdan pahalıya doğru elemektir. Kurulum hatalarında da önce terminali kapatıp açarsın, en son SDK'yı baştan kurarsın.

**Basitçe:** Aşağıdaki dört hata, kurulumda karşılaşacağın vakaların büyük çoğunluğunu kapsar. Hata mesajını okumadan çözüm aramak zaman kaybıdır; önce belirtiyi tabloda bul, sebebini anla, sonra uygula.

**Teknik olarak:**

| Belirti | Sebep | Çözüm |
|---|---|---|
| `dotnet` komutu bulunamıyor | PATH'e eklenmemiş | Terminali kapat aç; olmazsa SDK'yı repair et |
| SSMS `localhost`'a bağlanamıyor | SQL Server servisi kapalı | `services.msc` → SQL Server (MSSQLSERVER) → Başlat. Ayrıca SQL Server Configuration Manager'dan TCP/IP'yi etkinleştir |
| Docker Desktop açılmıyor | WSL2 yok / sanallaştırma kapalı | BIOS'ta virtualization aç, `wsl --install` çalıştır |
| `dotnet run` port hatası | Port meşgul | `Properties/launchSettings.json` içinden portu değiştir |

Birkaç ek vaka, ilerleyen haftalarda karşına çıkacak:

| Belirti | Sebep | Çözüm |
|---|---|---|
| `The SDK 'Microsoft.NET.Sdk' specified could not be found` | `global.json` kurulu olmayan bir sürümü işaret ediyor | O sürümü kur ya da `global.json` dosyasını sil |
| Tarayıcıda `NET::ERR_CERT_AUTHORITY_INVALID` | Yerel HTTPS sertifikası güvenilmiyor | `dotnet dev-certs https --trust` |
| `dotnet restore` paketleri indiremiyor | NuGet kaynağı bozuk ya da önbellek kirli | `dotnet nuget locals all --clear` sonra tekrar `restore` |
| `npm install` izin hatası veriyor | Yönetici gerektiren global klasöre yazma denemesi | Global kurulum yerine proje içi kurulum kullan |

Hangi portun meşgul olduğunu bulmak için:

```bash
netstat -ano | findstr :5001
```

Çıkan son sütun PID'dir; hangi programın tuttuğunu Görev Yöneticisi'nin Ayrıntılar sekmesinden PID'e bakarak görürsün.

> **Dikkat:** `dotnet run` port hatası verdiğinde ilk refleks portu değiştirmek olmasın. Çoğu zaman sebep, arka planda hâlâ çalışan bir önceki `dotnet run` işlemidir. Önce onu kapat.

---

## 5. Bugünün Çıktısı

> **Benzetme —** Pilotlar kalkıştan önce listeyi ezbere bilseler de tek tek okuyup işaretler. Amaç bilgiyi hatırlamak değil, atlamadığını kanıtlamaktır. Günün sonundaki bu liste de aynı işi görür.

**Basitçe:** Aşağıdaki dört maddenin hepsi işaretlenmeden bu günü kapatma. Eksik kalan bir madde, ilerleyen haftada iki katı zaman olarak geri döner.

**Teknik olarak:**

- [ ] Yukarıdaki tüm doğrulama komutları hatasız
- [ ] SSMS'ten SQL Server'a bağlanıldı
- [ ] `dotnet new mvc` çalışıp tarayıcıda açıldı
- [ ] `sa` şifresi ve bağlantı cümlesi güvenli bir yere not edildi

> Bağlantı cümlesini ve `sa` şifresini bir dosyaya yazacaksan, o dosyanın repoya girmediğinden emin ol. Bir kez commit edilen şifre geçmişte kalır; silmek yetmez, değiştirmek gerekir. Bu konunun ayrıntısı bir sonraki notta.

---

## Tek Bakışta Özet

- Kurulum sırası bağımlılık sırasıdır: önce **.NET SDK**, sonra IDE, sonra veritabanı, en son yan araçlar.
- **SDK** kod üretir, **runtime** kodu çalıştırır. Geliştiricide SDK olur.
- Visual Studio'da en az **ASP.NET ve web geliştirme** workload'u seçilmeli.
- SQL Server için **Developer Edition** ve **Mixed Mode Authentication** seç; `sa` şifresini güvenli bir yere not et.
- Kurulum bitince her aracın sürüm komutunu çalıştır; sonra `dotnet new mvc` ile uçtan uca dene.
- Node ve Docker bugün kurulsun; Hafta 6'da uğraşmak istemezsin.
- Küçük denemeler tek repoda, ciddi projeler kendi repolarında.
- Hataların çoğu **PATH**, **kapalı servis** ve **meşgul port** üçlüsünden çıkar.

---

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| SDK | Derleyici, `dotnet` aracı ve şablonları içeren geliştirme kiti |
| Runtime | Derlenmiş bir .NET uygulamasını çalıştıran katman |
| LTS | Long Term Support — uzun süre güncelleme alan kararlı sürüm |
| PATH | İşletim sisteminin çalıştırılabilir dosyaları aradığı klasör listesi |
| Workload | Visual Studio'da birlikte kurulan ilgili bileşen paketi |
| `global.json` | Bir klasörde kullanılacak SDK sürümünü sabitleyen dosya |
| Solution (`.sln`) | Birden fazla projeyi bir arada tutan kapsayıcı dosya |
| Project (`.csproj`) | Tek bir derleme biriminin tanım dosyası |
| SSMS | SQL Server Management Studio — veritabanı yönetim arayüzü |
| Mixed Mode Authentication | Hem Windows hem SQL kullanıcısıyla bağlanabilme |
| Connection string | Veritabanına nasıl bağlanılacağını tarif eden metin |
| WSL2 | Windows üzerinde Linux çekirdeği çalıştıran katman; Docker bunu kullanır |
| NuGet | .NET'in paket yöneticisi |
| `winget` | Windows'un komut satırı paket yöneticisi |
| Swagger | API uç noktalarını tarayıcıdan denemeyi sağlayan arayüz |

---

## Sık Karıştırılanlar

| Karıştırılan | Fark |
|---|---|
| **SDK** ile **Runtime** | SDK geliştirir ve içinde runtime'ı da getirir; runtime sadece çalıştırır |
| **Visual Studio** ile **Visual Studio Code** | İlki tam kapsamlı IDE (ağır, Windows odaklı), ikincisi eklentiyle genişleyen metin editörü |
| **SQL Server** ile **SSMS** | SQL Server veritabanı sunucusudur (arka planda çalışır), SSMS ona bağlanan yönetim arayüzüdür |
| **Express** ile **Developer Edition** | Express sınırlı ve üretimde kullanılabilir; Developer tam özelliklidir ama sadece geliştirme/test için |
| **`dotnet build`** ile **`dotnet run`** | `build` derler ve durur; `run` derleyip ardından çalıştırır |
| **`dotnet new mvc`** ile **`dotnet new webapi`** | İlki sayfa döndüren web uygulaması, ikincisi veri (JSON) döndüren API |
| **Docker Desktop** ile **Docker Engine** | Desktop masaüstü arayüzü ve WSL2 entegrasyonudur; Engine asıl konteyner motorudur |

---

## Sonraki

→ `Hafta-00-Kurulum/02-Git-ve-GitHub.md`
