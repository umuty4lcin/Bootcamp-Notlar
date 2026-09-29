# Hafta 0 — Hızlı Tekrar: Ortam ve Zemin

**Okuma süresi:** ~3 dk

**Ne işe yarar:** Bu dosya yeni bir şey öğretmez. Haftanın notlarındaki
`Tek Bakışta Özet` ve `Sık Karıştırılanlar` bölümlerinin tek yerde toplanmış
hâlidir — zaten okuduğun şeyin hatırlatıcısı.

**Nasıl kullanılır:**

1. Baştan sona oku. Her maddede kendine sor: *"bunu birine anlatabilir miyim?"*
2. Cevap "hayır" olan maddenin üstündeki dosya adını not et.
3. Sadece o konunun tam nottaki bölümüne dön. Notu baştan okuma.

> Tekrar okumak, bildiğini sanmakla bilmeyi karıştırmanın en kolay yoludur.
> Bir maddeyi görünce "biliyorum" diye geçme — önce kapalı gözle anlatmayı dene,
> sonra satıra bak. Aradaki fark, gerçekten ne bildiğindir.

Terim aradığında bu dosya değil, yanındaki [`00-Terimler-Sozlugu.md`](00-Terimler-Sozlugu.md) açılır.

---

## Bu Dosyada Ne Var

1. **Ortam Kurulumu** — `01-Ortam-Kurulumu.md` · Cumartesi
2. **Git ve GitHub** — `02-Git-ve-GitHub.md` · Pazar

---

## 1. Ortam Kurulumu

*Kaynak: [`01-Ortam-Kurulumu.md`](01-Ortam-Kurulumu.md) · Cumartesi*

- Kurulum sırası bağımlılık sırasıdır: önce **.NET SDK**, sonra IDE, sonra veritabanı, en son yan araçlar.
- **SDK** kod üretir, **runtime** kodu çalıştırır. Geliştiricide SDK olur.
- Visual Studio'da en az **ASP.NET ve web geliştirme** workload'u seçilmeli.
- SQL Server için **Developer Edition** ve **Mixed Mode Authentication** seç; `sa` şifresini güvenli bir yere not et.
- Kurulum bitince her aracın sürüm komutunu çalıştır; sonra `dotnet new mvc` ile uçtan uca dene.
- Node ve Docker bugün kurulsun; Hafta 6'da uğraşmak istemezsin.
- Küçük denemeler tek repoda, ciddi projeler kendi repolarında.
- Hataların çoğu **PATH**, **kapalı servis** ve **meşgul port** üçlüsünden çıkar.

---

### Sık karıştırılanlar

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


---

## 2. Git ve GitHub

*Kaynak: [`02-Git-ve-GitHub.md`](02-Git-ve-GitHub.md) · Pazar*

- Git yerelde çalışır, GitHub barındırır. İkisi aynı şey değil.
- Her şey **üç bölge** ile açıklanır: çalışma dizini → staging → repo.
- **Commit** bir anlık görüntüdür ve parent'ıyla zincirlenir; geçmiş bu yüzden bir grafiktir.
- **Branch** bir kopya değil, sadece bir işaretçidir — bu yüzden ucuzdur.
- **Merge** dürüst geçmiş bırakır, **rebase** temiz geçmiş bırakır; paylaşılan dalda rebase yapılmaz.
- **Conflict** hata değil, karar talebidir.
- `fetch` güvenlidir, `pull` doğrudan uygular; ekip projesinde önce bak sonra birleştir.
- `.gitignore` sadece takip edilmeyen dosyalarda çalışır; sızan şifre geçmişte kalır.
- Paylaşılan dalda geri alma yöntemi `revert`'tür, `reset` değil.
- Bir şeyi kaybettiğini sandığında ilk bakılacak yer `git reflog`.
- `git clean` reflog'a yazmaz; öncesinde `-n` ile ne sileceğini gör.

---

### Sık karıştırılanlar

| Karıştırılan | Fark |
|---|---|
| **Git** ile **GitHub** | Git bilgisayarındaki program, GitHub repoları barındıran site |
| **`git add`** ile **`git commit`** | `add` bir sonraki kayda neyin gireceğini seçer, `commit` kaydı mühürler |
| **`git fetch`** ile **`git pull`** | `fetch` sadece indirir, `pull` indirip birleştirir |
| **`git restore`** ile **`git reset`** | `restore` dosya seviyesinde çalışır, `reset` branch işaretçisini oynatır |
| **`git revert`** ile **`git reset`** | `revert` iptal eden yeni commit ekler, `reset` geçmişi geri sarar |
| **`git reset --hard`** ile **`git clean`** | `--hard` takip edilen dosyaları eski hâline döndürür, `clean` takip edilmeyenleri siler |
| **Merge** ile **Rebase** | Merge geçmişi olduğu gibi bırakır, rebase yeniden yazar |
| **`main`** ile **`origin/main`** | İlki senin dalın, ikincisi uzaktakinin son bilinen hâli |
| **Fork** ile **Clone** | Fork GitHub'da hesabına kopya çıkarır, clone bilgisayarına indirir |
| **`.gitignore`** ile **`git rm --cached`** | İlki henüz takip edilmeyeni yok sayar, ikincisi takip edileni listeden çıkarır |

---


---

## Sonraki

Bulanık kalan madde varsa yukarıdaki kaynak satırından dosya adını al ve
sadece o bölümü oku. Haftanın tamamını yeniden okumana gerek yok.

*Bu dosya haftanın notlarından üretildi. Notlar güncellenince yeniden üretilir —
elle düzenleme, değişiklikler kaybolur.*
