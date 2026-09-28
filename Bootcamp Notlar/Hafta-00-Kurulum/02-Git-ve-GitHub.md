# Hafta 0 · Pazar — Git ve GitHub

**Okuma süresi:** ~35 dk · **Tekrar için:** Git'te bir şey ters gittiğinde önce buraya bak.
**Neden bu konu:** Git, yazdığın her şeyin altındaki güvenlik ağıdır. Komutları ezberlemek değil, ne olup bittiğini görebilmek işe yarar; o görüş bir kez oturunca hiç kaybolmaz.

---

## Önce Basitçe

Bir tarih defteri düşün. Projende her anlamlı iş bittiğinde defteri açıp o anın fotoğrafını yapıştırıyorsun ve altına ne yaptığını yazıyorsun. Aradan üç ay geçip "bu satır ne zaman, niye böyle oldu" diye sorduğunda defteri geriye doğru karıştırıp buluyorsun. Git tam olarak bu defteri tutan programdır.

Defter senin bilgisayarında duruyor. Ama bilgisayarın bozulabilir, çantandan düşebilir. O yüzden defterin bir kopyasını da internette bir emanetçide tutuyorsun. İşte **GitHub** o emanetçi. Git ile GitHub'ı karıştırmak, fotoğraf makinesi ile bulut albümü karıştırmaya benzer: biri araç, diğeri o araçla ürettiğinin durduğu yer.

Bu defterin güzel tarafı şu: sayfalar arasında serbestçe gezinebilirsin. "Bir haftalık şu deneme işe yaramadı" dediğinde o sayfaları tamamen atabilir, ana hikâyeye geri dönebilirsin. Hatta ana hikâyeye hiç dokunmadan, yan bir defterde deneyip beğenirsen ana deftere ekleyebilirsin. Buna **branch (dal)** deniyor ve Git'in en çok kullanılan özelliği bu.

Bir şeyi deftere yazmak iki adımda oluyor. Önce "şunlar bu sayfaya girecek" diye bir kenara ayırıyorsun, sonra sayfayı kapatıp mühürlüyorsun. Bu ara adım kafa karıştırır ama asıl değeri şurada: beş dosya değiştirdiysen ve sadece ikisi aynı işe aitse, o ikisini ayrı bir sayfaya yazabiliyorsun. Defter böylece "ne zaman ne yapıldı"yı gerçekten anlatan bir şeye dönüşüyor.

Korkulacak bir şey yok. Git'in yaptığı neredeyse her şey geri alınabilir, çünkü Git bir şeyi silmekten çok eklemeyi sever. Bir kez deftere geçmiş bir şeyi kaybetmek şaşırtıcı derecede zordur. Tehlikeli olan komutlar sayılıdır ve hangileri olduğunu bu notun sonunda tek tek göreceksin.

Şimdi detaya iniyoruz.

> **Ana benzetme:** Git, projenin tarih defteridir. Her commit o anın fotoğrafıdır, her branch aynı deftere konmuş bir ayraçtır, GitHub ise defterin bir kopyasının durduğu emanetçidir.

---

## Bu Notta Ne Var

Git komut listesi her yerde var; eksik olan **komutların neyi değiştirdiğini gösteren zihinsel model**. Bu not Git'i "ezberlenecek komutlar" olarak değil, "üç bölgeli bir veri yapısı" olarak anlatır. Bir kere oturunca `git status` çıktısını okumak yeter, komut ezberlemeye gerek kalmaz.

1. Git nedir, ne değildir — Git ile GitHub ayrımı
2. Üç bölge modeli — çalışma dizini, staging, repo
3. Commit — Git'in atomu, içinde ne var
4. Branch, merge, rebase ve conflict
5. Uzak repo kavramları — remote, fetch, pull, push, PR
6. `.gitignore` ve neyi commit'lememeli
7. Geri alma komutları — hangisi neyi bozar
8. Tek bakışta özet, terimler sözlüğü, sık karıştırılanlar

---

## 1. Git Nedir, Ne Değildir

> **Benzetme —** Fotoğraf makinesi senin cebinde durur, internet olmadan da fotoğraf çeker. Bulut albüm ise o fotoğrafları yüklediğin, başkalarıyla paylaştığın yerdir. Makine bozulsa albüm durur, albüm kapansa makine çalışmaya devam eder. Git makinedir, GitHub albümdür.

**Basitçe:** Git bilgisayarında çalışan bir programdır ve internet istemez. GitHub ise Git'in ürettiği geçmişi internette barındıran bir site. İkisi aynı şirket, aynı ürün, hatta aynı kavram bile değil. Git'i öğrenmek GitHub'ı da öğrenmek demektir ama tersi doğru değildir.

**Teknik olarak:** **Git** — dağıtık bir sürüm kontrol sistemi (DVCS). Projenin geçmişini, "değişiklikler" olarak değil, **anlık görüntüler (snapshot)** dizisi olarak saklar.

Ayrımı netleştirelim:

| Kavram | Ne demek |
|---|---|
| **Git** | Bilgisayarında çalışan program. İnternet gerektirmez |
| **GitHub / GitLab / Bitbucket** | Git repolarını barındıran web servisleri. Git'in kendisi değil, "Git için bulut disk + sosyal katman" |
| **Repository (repo)** | Projenin dosyaları + tüm geçmişi. `.git` klasörünün içindeki her şey |
| **Dağıtık (distributed)** | Klonladığın her kopyada **tüm geçmiş** vardır. Sunucu çökse projenin tarihi kaybolmaz |

> **Sık yanlış bilinen:** Git dosyaların farkını (diff) saklamaz. Her commit'te değişen dosyaların **tam hâlini** saklar, değişmeyen dosyalar için bir öncekine işaretçi koyar. Diff'i sana gösterirken **hesaplar**. Bu yüzden geçmişte gezinmek hızlıdır.

Bir repo iki yoldan doğar. Ya sıfırdan başlatırsın:

```bash
mkdir KutuphaneCLI
cd KutuphaneCLI
git init                 # bu klasörde .git oluşturur
```

Ya da var olan bir repoyu indirirsin:

```bash
git clone https://github.com/kullanici/proje.git
git clone git@github.com:kullanici/proje.git    # SSH ile
```

`clone` sadece dosyaları değil, **tüm geçmişi** indirir. İndirdikten sonra internet kesilse bile `git log`, `git diff`, `git branch` gibi komutların hepsi çalışır. "Dağıtık" kelimesinin pratik anlamı budur.

`.git` klasörünün içine bir göz atmak modeli somutlaştırır:

```bash
ls -a                    # .git klasörünü gör
ls .git                  # HEAD, config, objects, refs ...
cat .git/HEAD            # ref: refs/heads/main
```

> `.git` klasörünü silersen proje dosyaların durur ama geçmişin tamamen gider. Repoyu "kapatmanın" yolu budur; yanlışlıkla yapılmaması gereken birkaç şeyden biri.

---

## 2. Üç Bölge Modeli — Git'in Tamamı Bu

> **Benzetme —** Kargoya paket göndermeyi düşün. Masanın üstü dağınıktır, her şey ortada durur — orası çalışma dizinidir. Göndereceklerini seçip koliye koyarsın — o koli staging'dir. Koliyi bantlayıp kayda geçirdiğinde artık bir gönderi numarası alır, içeriği sabittir — bu commit'tir. Sonra koliyi şubeye teslim edersin — push budur.
>
> Bu benzetme şurada bozulur: koliye koyduğun eşya masadan kalkar, ama `git add` dosyayı çalışma dizininden almaz. Kopyasını alır. Dosya hâlâ yerindedir ve stage'ledikten sonra tekrar değiştirirsen, aynı dosyanın iki farklı hâli aynı anda "staged" ve "modified" olarak görünür.

**Basitçe:** Git'te bir değişikliğin kaydedilmesi tek adımda olmaz. Önce "bunlar bu kayda girsin" diye ayırırsın (`git add`), sonra kaydı mühürlersin (`git commit`). Bu ikili adım başta gereksiz gelir; asıl amacı, bir kaydın tek bir mantıklı işi anlatmasını sağlamaktır.

**Teknik olarak:**

```
┌──────────────────┐   git add    ┌──────────────┐   git commit   ┌──────────────┐
│ Çalışma Dizini   │ ───────────► │  Staging     │ ─────────────► │  Yerel Repo  │
│ (Working Tree)   │              │  (Index)     │                │  (.git)      │
└──────────────────┘              └──────────────┘                └──────────────┘
   dosyaları düzenlediğin yer      "bir sonraki commit'e            kalıcı geçmiş
                                    şunlar girecek" listesi                │
                                                                           │ git push
                                                                           ▼
                                                                  ┌──────────────┐
                                                                  │  Uzak Repo   │
                                                                  │  (GitHub)    │
                                                                  └──────────────┘
```

**Çalışma dizini (working tree)** — Dosya gezgininde gördüğün klasör. Burada yaptığın hiçbir şey henüz Git için "var" değildir.

**Staging area / index** — Ara kat. Git'i diğer sürüm kontrol sistemlerinden ayıran esas fikir budur. Beş dosya değiştirdin ama sadece ikisi aynı işe ait — o ikisini stage'leyip ayrı commit atarsın. Commit'in "tek bir mantıklı iş" olmasını sağlayan mekanizma bu.

**Yerel repo** — `.git` klasörü. Commit ettiğin an değişiklik buraya yazılır ve pratikte kaybolmaz.

**Uzak repo (remote)** — GitHub'daki kopya. `push` ile gönderir, `pull` ile alırsın.

Bir dosyanın Git'e göre durumu:

| Durum | Anlamı |
|---|---|
| **Untracked** | Git bu dosyayı hiç görmedi. Yeni oluşturulmuş |
| **Modified** | Takip ediliyor, son commit'ten sonra değişmiş, henüz stage'lenmemiş |
| **Staged** | Bir sonraki commit'e girmek üzere işaretlenmiş |
| **Committed / Unmodified** | Son commit ile birebir aynı |

`git status` sana her zaman bu dört durumu söyler. Kafan karıştığında çalıştıracağın tek komut budur.

```bash
git status              # neredeyim
git add Program.cs      # tek dosyayı stage'le
git add .               # her şeyi stage'le
git commit -m "LINQ notları eklendi"
git log --oneline       # geçmişi oku
```

Hangi bölgede ne olduğunu görmek için `diff` komutunun iki hâli vardır ve karıştırılır:

```bash
git diff                # çalışma dizini ile staging arasındaki fark
git diff --staged       # staging ile son commit arasındaki fark
git diff HEAD           # çalışma dizini ile son commit arasındaki fark (ikisi birden)
```

Kısa çıktı okumayı hızlandırır:

```bash
git status -s
```

```text
 M Program.cs        # modified, henüz stage'lenmemiş
M  Startup.cs        # stage'lenmiş
MM Models/Kitap.cs   # stage'lenmiş, sonra tekrar değiştirilmiş
?? notlar.txt        # untracked
```

Soldaki sütun staging'i, sağdaki sütun çalışma dizinini gösterir. Üçüncü satır, benzetmenin kırıldığı yerin somut hâlidir.

Dosyanın tamamını değil, içindeki belirli parçaları stage'lemek de mümkündür:

```bash
git add -p Program.cs   # değişiklikleri parça parça sorar: y / n / s
```

Bu, "bir commit tek bir iş anlatsın" kuralını tek dosya içinde bile uygulamanı sağlar.

---

## 3. Commit — Git'in Atomu

> **Benzetme —** Eski aile albümlerinde fotoğrafın arkasına tarih, kimin çektiği ve bazen "bundan önceki: Bayram 1998" yazılırdı. Her fotoğraf hem kendi anını hem de zincirdeki yerini taşırdı. Commit de böyledir: içeriği, kim yazdığı, ne zaman yazıldığı ve **bir öncekinin kimliği** birlikte durur.

**Basitçe:** Commit, projenin o andaki tam fotoğrafıdır. Sadece "neyi değiştirdin"i değil, "değişimden sonra proje nasıl görünüyordu"yu saklar. Her fotoğraf bir öncekine bağlıdır; zincir bu yüzden kopmaz.

**Teknik olarak:** **Commit** — Projenin belirli bir andaki tam anlık görüntüsü. Her commit'in içinde şunlar var:

| Alan | İçerik |
|---|---|
| **Hash (SHA-1)** | `a3f5c9e...` — commit'in kimliği. İçeriğinden hesaplanır, bu yüzden geçmişi değiştirmek hash'leri değiştirir |
| **Parent** | Bir önceki commit'in hash'i. Geçmişi zincirleyen şey bu |
| **Author / Committer** | Kim yazdı, kim uyguladı (rebase'de farklı olabilir) |
| **Tarih** | Zaman damgası |
| **Mesaj** | Ne yapıldığının açıklaması |
| **Tree** | Dosya ve klasör yapısına işaretçi |

Parent alanı yüzünden commit geçmişi bir **yönlü asiklik grafiktir (DAG)** — düz bir liste değil. Merge commit'lerin iki parent'ı vardır; dallanma ve birleşme buradan gelir.

Bir commit'in içini açıp bakabilirsin:

```bash
git log -1                       # son commit'in tüm alanları
git show a3f5c9e                 # commit + içindeki diff
git cat-file -p a3f5c9e          # ham commit nesnesi: tree, parent, author
```

Son komutun çıktısı kabaca şudur:

```text
tree 8f2b1c4d...
parent 9c1e7a3b...
author Umut <mail@ornek.com> 1716120000 +0300
committer Umut <mail@ornek.com> 1716120000 +0300

Kullanıcı listesine sayfalama eklendi
```

Bu dört satır, yukarıdaki tablonun gerçekte diskte nasıl durduğudur.

Geçmişi grafik olarak okumak, DAG fikrini somutlaştırır:

```bash
git log --oneline --graph --all --decorate
```

### İyi commit mesajı

Kural: mesaj *"Bu commit uygulandığında ..."* cümlesini tamamlamalı.

| Kötü | İyi |
|---|---|
| `guncelleme` | `Kullanıcı listesine sayfalama eklendi` |
| `fix` | `Null kontrolü eklenerek sipariş kaydındaki hata giderildi` |
| `asdf` | `EF Core migration: Product tablosuna Stok kolonu` |

Türkçe ya da İngilizce fark etmez; **tutarlı** olsun. Ay 8'deki grup projesinde commit geçmişin başkası tarafından okunacak.

Uzun mesaj yazman gerekiyorsa `-m` yerine editörü aç: ilk satır özet, bir boş satır, sonra ayrıntı.

```bash
git commit                       # varsayılan editörü açar
```

```text
Sipariş kaydındaki null hatası giderildi

Musteri.Adres alanı null geldiğinde SiparisServis.Kaydet
patlıyordu. Adres kontrolü eklendi ve boş adres için
varsayılan değer atanıyor.
```

### Son commit'i düzeltmek

Mesajı yanlış yazdıysan ya da bir dosyayı eklemeyi unuttuysan:

```bash
git add unutulan-dosya.cs
git commit --amend -m "Düzeltilmiş mesaj"
```

> **Dikkat:** `--amend` eski commit'i düzeltmez, **yerine yenisini koyar**. Hash değişir. Push ettiğin bir commit'i amend etmek, bir sonraki bölümdeki rebase ile aynı sorunu yaratır: karşı taraftaki geçmişle uyuşmaz.

### Yarım kalan işi rafa kaldırmak

Bir işin ortasındayken acil başka bir şeye bakman gerekirse:

```bash
git stash                        # değişiklikleri rafa kaldır, dizini temizle
git stash list                   # rafta ne var
git stash pop                    # en son kaldırdığını geri getir ve raftan sil
git stash apply stash@{1}        # belirli birini geri getir, rafta da kalsın
```

**Stash** commit değildir; geçici bir cep. Uzun süre orada bir şey bırakma, unutulur.

---

## 4. Branch — Aslında Sadece Bir İşaretçi

> **Benzetme —** Bir kitabın arasına koyduğun ayracı düşün. Ayraç kitabı çoğaltmaz, sadece "ben buradayım" der. İkinci bir ayraç koyarsan ikinci bir kitabın olmaz; aynı kitapta iki işaret olur. Branch de böyledir: klasörün kopyası değil, bir commit'i gösteren küçük bir işaret.
>
> Bu benzetme şurada bozulur: kitap ayracı sen taşımadıkça yerinde durur. Branch ise sen commit attıkça **kendiliğinden** ileri kayar. Ayraç sanki okudukça kendi ilerliyormuş gibi düşün.

**Basitçe:** Branch açmak bir kopya oluşturmaz, bir isim oluşturur. Bu yüzden anlıktır ve neredeyse hiç yer kaplamaz. Yeni bir özelliğe başlarken branch açmak, ana hattı bozmadan denemek demektir; işe yaramazsa branch'i silersin, hiçbir şey olmaz.

**Teknik olarak:** **Branch (dal)** — Bir commit'i gösteren, taşınabilir bir isim. Klasörün kopyası değil, sadece 40 karakterlik bir dosya. Bu yüzden Git'te branch açmak neredeyse bedavadır.

**HEAD** — "Şu anda neredeyim" işaretçisi. Normalde bir branch'i, o da bir commit'i gösterir.

```
                    ┌── main
                    ▼
A ──── B ──── C ── D
               \
                E ──── F
                       ▲
                       └── feature/linq   ◄── HEAD (şu an buradasın)
```

Commit attığında branch işaretçisi otomatik ileri kayar. Branch değiştirdiğinde Git çalışma dizinini o commit'in içeriğine göre yeniden düzenler.

```bash
git branch                    # branch listesi
git switch -c feature/linq    # oluştur ve geç
git switch main               # geri dön
```

"Sadece bir dosya" iddiasını kendin doğrulayabilirsin:

```bash
cat .git/refs/heads/main      # tek satır: bir commit hash'i
cat .git/HEAD                 # ref: refs/heads/feature/linq
```

Sık kullanılan diğer branch komutları:

```bash
git branch -v                 # her branch'in son commit'i
git branch -d feature/linq    # birleştirilmiş branch'i sil
git branch -D feature/linq    # birleştirilmemiş olsa da sil (dikkat)
git branch -m yeni-isim       # bulunduğun branch'i yeniden adlandır
git switch -                  # bir önceki branch'e dön
```

**Detached HEAD** — HEAD'in bir branch yerine doğrudan bir commit'i göstermesi. `git switch a3f5c9e` gibi bir komutla oraya düşersin. Bu durumda attığın commit'lere hiçbir branch işaret etmez; branch değiştirince erişilemez hâle gelirler. Oradan çıkmanın yolu ya bir branch oluşturmaktır (`git switch -c denemeler`) ya da mevcut bir branch'e dönmektir.

### Merge ve Rebase — Aynı İşin İki Felsefesi

> **Benzetme —** İki kişi aynı raporun farklı bölümlerini yazdı. **Merge**, iki metni yan yana koyup üstüne "şu tarihte birleştirildi" notu düşmektir; kimin ne zaman ne yazdığı belli kalır. **Rebase** ise senin bölümünü alıp diğerinin sonuna temiz bir şekilde yeniden yazmaktır; okuyan tek bir kalemden çıkmış sanır, ama artık senin ilk müsveddelerin yoktur.

| | **Merge** | **Rebase** |
|---|---|---|
| Ne yapar | İki dalı birleştiren yeni bir **merge commit** oluşturur | Senin commit'lerini söküp hedef dalın ucuna **yeniden uygular** |
| Geçmiş | Gerçekte ne olduğunu gösterir, dallanmalar görünür | Düz, tek çizgi hâlinde okunur |
| Hash'ler | Değişmez | **Değişir** — commit'ler teknik olarak yeniden yazılır |
| Ne zaman | Paylaşılan dallarda (`main`) | Kendi lokal dalını temizlerken |

> **Altın kural:** Başkasının da çektiği bir dalı asla rebase etme. Hash'ler değişince karşı taraftaki geçmişle uyuşmaz ve ortalık karışır.

**Fast-forward merge** — Ayrıldığından beri `main` hiç ilerlememişse, Git merge commit oluşturmaz; sadece işaretçiyi ileri kaydırır. En temiz senaryo budur.

Komut karşılıkları:

```bash
git switch main
git merge feature/linq            # mümkünse fast-forward, değilse merge commit
git merge --no-ff feature/linq    # her hâlükârde merge commit oluştur
```

```bash
git switch feature/linq
git rebase main                   # kendi commit'lerini main'in ucuna taşı
git rebase --abort                # işler karışırsa başladığın yere dön
```

`--no-ff`, özellik dallarının geçmişte görünür kalmasını sağlar; ekipler çoğu zaman bunu tercih eder çünkü "bu beş commit tek bir özellikti" bilgisi korunur.

### Conflict (Çakışma)

> **Benzetme —** İki kişi aynı alışveriş listesinde aynı satırı değiştirmiş: biri "1 kg domates" yazmış, diğeri "2 kg salatalık". Listeyi birleştiren kişi hangisinin doğru olduğunu bilemez, ikisini de yan yana yazıp "siz karar verin" der. Git'in yaptığı tam olarak budur.

**Basitçe:** Çakışma, Git'in beceriksizliği değil dürüstlüğüdür. İki kişi aynı satırı farklı değiştirdiğinde Git tahmin yürütmez, kararı sana bırakır. Yapman gereken dosyayı açıp doğru hâli bırakmak, sonra normal akışa devam etmektir.

**Teknik olarak:** **Conflict** — Aynı dosyanın aynı satırlarının iki dalda farklı değiştirilmesi. Git hangisinin doğru olduğunu bilemez, kararı sana bırakır ve dosyaya şunu yazar:

```
<<<<<<< HEAD
senin daldaki hâli
=======
diğer daldaki hâli
>>>>>>> feature/linq
```

Yapılacak: işaretçi satırlarını sil, kalmasını istediğin hâli bırak, `git add` + `git commit`. Conflict bir hata değil, Git'in "burada insan kararı lazım" demesidir.

Çakışma sırasında işine yarayacak komutlar:

```bash
git status                        # hangi dosyalar çakıştı
git diff                          # çakışan bölgeleri göster
git add cakisan-dosya.cs          # çözüldü olarak işaretle
git commit                        # merge'ü tamamla (mesaj hazır gelir)
git merge --abort                 # vazgeç, birleştirme öncesine dön
```

Rebase sırasında çakışma çıkarsa akış biraz farklıdır: çözdükten sonra `commit` değil `git rebase --continue` dersin, çünkü rebase commit'leri tek tek yeniden uygular ve sıradakine geçmesi gerekir.

---

## 5. Uzak Repo Kavramları

> **Benzetme —** Kargo şubesini düşün. `origin` şubenin adıdır — istersen "Merkez" de diyebilirdin, isim sihirli değildir. `fetch` şubeye gidip "bana bir şey gelmiş mi" diye sormak ve paketi alıp kenara koymaktır; açmazsın. `pull` ise paketi alıp aynı anda açıp eşyaları dolaba yerleştirmektir. `push` senin gönderinin şubeye teslimidir.

**Basitçe:** Uzak repo, projenin internetteki kopyasıdır. Oraya gönderirsin, oradan alırsın. `fetch` güvenlidir çünkü sadece indirir; `pull` indirip doğrudan üzerine uygular. Ne geldiğini görmek istiyorsan önce `fetch`, sonra bak, sonra birleştir.

**Teknik olarak:**

| Terim | Anlamı |
|---|---|
| **remote** | Uzak repoya verilen takma ad. Varsayılan adı `origin` |
| **origin** | Klonladığın reponun standart takma adı. Sihirli bir kelime değil, sadece gelenek |
| **clone** | Uzak repoyu tüm geçmişiyle indirmek |
| **fetch** | Uzaktaki yenilikleri **indirir ama çalışma dizinine uygulamaz**. Güvenli |
| **pull** | `fetch` + `merge`. İndirir ve doğrudan birleştirir |
| **push** | Yerel commit'leri uzağa gönderir |
| **upstream** | Yerel branch'in hangi uzak branch'i takip ettiği (`-u` ile kurulur) |
| **fork** | Başkasının reposunun kendi hesabına kopyası. GitHub kavramı, Git kavramı değil |
| **Pull Request (PR)** | "Şu dalımı ana dala alır mısın" teklifi. Kod incelemesinin (code review) yapıldığı yer. Bu da GitHub kavramı |

`pull` yerine `fetch` + inceleme + `merge` yapmak, ne geldiğini görmeni sağlar. Ekip projelerinde tercih edilen yol budur.

Yeni bir yerel repoyu GitHub'a bağlamak:

```bash
git remote add origin git@github.com:kullanici/proje.git
git remote -v                     # bağlı uzak repoları listele
git push -u origin main           # ilk gönderim, upstream'i de kurar
```

`-u` bayrağı sadece ilk seferde gerekir. Sonrasında `git push` ve `git pull` hangi uzak branch'le çalışacağını bilir.

Güvenli akış:

```bash
git fetch origin                  # indir, hiçbir şeye dokunma
git log --oneline HEAD..origin/main   # bana ne gelmiş
git merge origin/main             # şimdi birleştir
```

`origin/main`, "uzaktaki main'in en son bildiğim hâli" anlamına gelen bir **remote-tracking branch**'tir. Sen onu doğrudan değiştiremezsin; sadece `fetch` günceller. `main` ile `origin/main` arasındaki fark, "benim elimdeki" ile "sunucudaki" arasındaki farktır.

Push reddedilirse sebebi neredeyse her zaman aynıdır: sen çekmeden önce başkası göndermiştir.

```text
! [rejected]        main -> main (fetch first)
```

Çözüm, zorlamak değil önce almaktır:

```bash
git pull --rebase origin main     # kendi commit'lerini gelenin üstüne taşı
git push
```

> **Dikkat:** `git push --force` karşı taraftaki commit'leri silebilir. Mecbur kalırsan `--force-with-lease` kullan: bu, senin son gördüğünden beri uzak dal değişmişse push'u reddeder ve başkasının işini ezmeni engeller.

**Pull Request akışı** (GitHub tarafı, Git tarafı değil):

```bash
git switch -c feature/sayfalama
# ... değişiklikleri yap ...
git add .
git commit -m "Kullanıcı listesine sayfalama eklendi"
git push -u origin feature/sayfalama
```

Ardından GitHub sana "Compare & pull request" düğmesini gösterir. PR açılır, ekip yorum yazar, onaylanınca `main`'e alınır. PR birleştirildikten sonra yerel tarafı toparlamak:

```bash
git switch main
git pull
git branch -d feature/sayfalama
git remote prune origin           # silinmiş uzak dalların izlerini temizle
```

---

## 6. .gitignore ve Neyi Commit'lememeli

> **Benzetme —** Taşınırken her şeyi koliye koymazsın. Çöpü, bozulmuş yiyeceği, komşudan ödünç aldığın merdiveni koliye koymak hem yer kaplar hem anlamsızdır. `.gitignore`, "bunlar koliye girmeyecek" listesidir.
>
> Bu benzetme şurada bozulur: koliye yanlışlıkla koyduğun bir şeyi çıkarıp atarsan mesele biter. Git'te ise bir kez commit edilen şey geçmişte kalır; sonradan silsen bile eski commit'lerin içinde durmaya devam eder.

**Basitçe:** Repoya sadece kaynak dosyalar girer. Derleme çıktısı, IDE ayarları, indirilen paketler ve şifre içeren dosyalar girmez. Bunları listeleyen dosyanın adı `.gitignore`'dur ve projenin kökünde durur.

**Teknik olarak:** **.gitignore** — Git'in görmezden geleceği dosya kalıplarını listeleyen dosya.

.NET projelerinde asla commit edilmeyecekler:

| Ne | Neden |
|---|---|
| `bin/`, `obj/` | Derleme çıktısı. Kaynaktan yeniden üretilir, repoyu şişirir |
| `.vs/`, `.idea/` | IDE'nin kişisel ayarları. Ekip arkadaşını ilgilendirmez |
| `appsettings.Development.json` | Bağlantı cümlesi, API anahtarı içerebilir |
| `*.user` | Kullanıcıya özel proje ayarları |
| `node_modules/` | Angular/React tarafında. `package.json`'dan yeniden kurulur |

```bash
dotnet new gitignore     # .NET için hazır .gitignore üretir
```

Dosyanın içi kabaca şöyle görünür:

```text
bin/
obj/
.vs/
*.user
appsettings.Development.json

# istisna: bu dosya ignore edilmesin
!appsettings.Example.json
```

Kalıp kuralları kısaca:

| Kalıp | Anlamı |
|---|---|
| `bin/` | Her seviyedeki `bin` klasörü |
| `/bin/` | Sadece kökteki `bin` klasörü |
| `*.log` | Uzantısı `.log` olan tüm dosyalar |
| `!onemli.log` | Yukarıdaki kurala rağmen bu dosya dahil edilsin |
| `logs/**/gecici` | `logs` altında kaç klasör derinde olursa olsun |

Bir dosyanın neden yok sayıldığını merak edersen Git sana hangi satırın sorumlu olduğunu söyler:

```bash
git check-ignore -v appsettings.Development.json
```

> **Önemli:** `.gitignore` sadece **henüz takip edilmeyen** dosyalar için çalışır. Bir dosyayı yanlışlıkla commit ettiysen sonradan ignore'a eklemek işe yaramaz; önce `git rm --cached` ile takipten çıkarman gerekir.
>
> Daha kritiği: bir şifre bir kez commit edildiyse **geçmişte kalır**. Sonraki commit'te silmek yetmez. O şifre yanmıştır, değiştirilmesi gerekir.

Takipten çıkarma komutu, dosyayı diskten silmez:

```bash
git rm --cached appsettings.Development.json
git commit -m "Geliştirme ayarları takipten çıkarıldı"
```

Yanlışlıkla `bin/` ve `obj/` commit edildiyse klasörün tamamı için:

```bash
git rm -r --cached bin obj
```

Yerel makineye özel, repoya girmeyecek ignore kuralların varsa onları `.gitignore` yerine `.git/info/exclude` dosyasına yazarsın; o dosya hiç commit edilmez.

---

## 7. Geri Alma Komutları — Hangisi Neyi Bozar

> **Benzetme —** Deftere yanlış bir şey yazdın. Üç seçeneğin var: kurşun kalemse silersin (`restore`), tükenmezse altına "yukarıdaki iptal, doğrusu şu" diye yazarsın (`revert`), ya da sayfayı yırtarsın (`reset`). Defter senin özel defterinse sayfayı yırtabilirsin. Ama defterin fotokopisi başkalarında da varsa, yırtmak işe yaramaz; herkesin elindeki sayfa numaraları kayar. O zaman doğru yol altına düzeltme yazmaktır.

**Basitçe:** Geri alma komutları arasındaki fark, "geçmişi değiştiriyor mu" sorusunda düğümlenir. Kendi bilgisayarındaki, henüz kimseye göndermediğin bir şeyi istediğin gibi düzeltebilirsin. Gönderdiğin bir şeyi ise silmeye çalışma; iptal eden yeni bir kayıt ekle.

**Teknik olarak:** Git'te en çok korkulan alan burası. Üç komutun farkı:

| Komut | Ne yapar | Geçmişi bozar mı |
|---|---|---|
| `git restore <dosya>` | Dosyayı son commit'teki hâline döndürür | Hayır |
| `git restore --staged <dosya>` | Stage'den çıkarır, değişiklik durur | Hayır |
| `git revert <commit>` | O commit'i **iptal eden yeni bir commit** oluşturur | Hayır — paylaşılan dallarda doğru yöntem |
| `git reset --soft <commit>` | Branch'i geri alır, değişiklikler stage'de kalır | Evet (yerel) |
| `git reset --hard <commit>` | Branch'i geri alır, **değişiklikleri siler** | Evet — dikkat |

Arada bir de `--mixed` vardır ve `reset`'in varsayılanıdır: branch'i geri alır, değişiklikler çalışma dizininde kalır ama stage'den çıkar.

| Mod | Branch işaretçisi | Staging | Çalışma dizini |
|---|---|---|---|
| `--soft` | Geri gider | Korunur (değişiklikler stage'de) | Dokunulmaz |
| `--mixed` (varsayılan) | Geri gider | Temizlenir | Dokunulmaz |
| `--hard` | Geri gider | Temizlenir | **Üzerine yazılır** |

Pratikte en sık kullanılan senaryolar:

```bash
git restore Program.cs                   # bu dosyadaki değişikliği çöpe at
git restore --staged Program.cs          # stage'den geri al, değişiklik dursun
git reset --soft HEAD~1                  # son commit'i aç, içeriği stage'de tut
git revert a3f5c9e                       # paylaşılan dalda o commit'i iptal et
```

`HEAD~1` "bir önceki commit", `HEAD~3` "üç önceki commit" demektir.

> `git reset --hard` commit edilmemiş çalışmayı gerçekten yok eder. Commit edilmiş bir şeyi ise `git reflog` ile geri getirebilirsin — Git son 90 günde HEAD'in gittiği her yeri kaydeder. "Her şeyi kaybettim" durumlarının çoğu `reflog` ile çözülür.

Kurtarma akışı şöyle işler:

```bash
git reflog                               # HEAD nereden nereye gitti
```

```text
a3f5c9e HEAD@{0}: reset: moving to HEAD~2
7d1b4e8 HEAD@{1}: commit: Sayfalama eklendi
9c1e7a3 HEAD@{2}: commit: Null kontrolü
```

Kaybettiğini sandığın commit `HEAD@{1}`'de duruyor. Geri almak için:

```bash
git reset --hard HEAD@{1}                # o noktaya dön
git switch -c kurtarma 7d1b4e8           # ya da yeni bir branch'te güvenceye al
```

Takip edilmeyen dosyaları temizleyen komut ayrıdır ve `reset` ile karıştırılır:

```bash
git clean -n                             # ne silineceğini göster (önce bunu çalıştır)
git clean -fd                            # untracked dosya ve klasörleri sil
```

`git clean` reflog'a yazmaz. Sildiği şey gerçekten gider. Bu yüzden `-n` olmadan çalıştırma alışkanlığı edinme.

---

## Tek Bakışta Özet

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

## Terimler Sözlüğü

| Terim | Tanım |
|---|---|
| Repository | Proje dosyaları + tam geçmiş |
| Working tree | Üzerinde çalıştığın dosyaların bulunduğu dizin |
| Index / Staging | Bir sonraki commit'e girecek değişikliklerin bekleme alanı |
| Commit | Projenin bir andaki anlık görüntüsü |
| Hash / SHA | Commit'in içeriğinden üretilen benzersiz kimlik |
| HEAD | Şu an bulunduğun konumu gösteren işaretçi |
| Branch | Bir commit'i gösteren taşınabilir isim |
| Detached HEAD | HEAD'in bir branch yerine doğrudan commit'i göstermesi |
| Merge | İki dalı birleştirme |
| Fast-forward | Merge commit gerektirmeyen, işaretçi kaydırmalı birleşme |
| Rebase | Commit'leri başka bir temele yeniden uygulama |
| Conflict | Aynı satırların iki dalda farklı değişmesi |
| Remote / origin | Uzak repo ve onun varsayılan takma adı |
| Remote-tracking branch | `origin/main` gibi, uzaktaki dalın son bilinen hâli |
| Upstream | Yerel bir branch'in takip ettiği uzak branch |
| Fetch / Pull / Push | İndir · indir+birleştir · gönder |
| Fork | Başkasının reposunun kendi hesabındaki kopyası (GitHub) |
| Pull Request | Değişikliklerin ana dala alınma teklifi (GitHub) |
| Stash | Yarım kalan işi geçici olarak rafa kaldırma |
| Reflog | HEAD'in geçmiş hareketlerinin kaydı — kurtarma aracı |
| Tag | Belirli bir commit'e verilen kalıcı isim (`v1.0` gibi) |
| DAG | Yönlü asiklik grafik — commit geçmişinin gerçek şekli |

---

## Sık Karıştırılanlar

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

## Sonraki

→ `Hafta-01-Modern-CSharp/01-NET-Calisma-Modeli.md`
