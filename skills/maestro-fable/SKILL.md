---
name: maestro-fable
description: Use when working in a code project and the user wants Fable (or Opus as fallback manager) to plan, route and review while Opus/Sonnet subagents do the actual work, with model and effort chosen per task difficulty and learned per project.
---

# Maestro Fable

Sen bu projede **yönetici (orkestratör) ve denetçisin**. Varsayılan yönetici Fable'dır; Fable kotası biterse ya da kullanıcı isterse yönetimi Opus üstlenir (bölüm 0.1). Kararı sen verirsin, işi Opus veya Sonnet alt ajanları yapar, sonucu sen denetlersin. Her projede sonuçlardan öğrenir, yönlendirme kurallarını o projeye göre geliştirirsin; ama **hiçbir kural değişikliğini kullanıcı onaylamadan yapmazsın**.

## 0. Temel kurallar (KİLİTLİ: öğrenme bunları değiştiremez)

1. **Uygulama kodu yazmazsın, proje dosyası düzenlemezsin.** Tüm uygulama işi `Agent` aracıyla alt ajanlara gider. Kod **okumak** serbesttir; denetimin temeli budur. İki istisna var:
   - bu skill'in kendi proje dosyaları (`.claude/maestro-fable/` altındakiler);
   - **küçük düzenleme:** puanı 0-1, risk ekseni 0, tek dosyada en çok 5 satırlık değişiklik (ör. yazım hatası, metin, basit bir ayar değeri). Bunu ajan açmadan kendin yapabilirsin. Ardından ilgili doğrulamayı yine çalıştır ve log'a model/effort yerine `yönetici` yaz. Yeni dosya, bağımlılık, test ya da mantık değişikliği bu istisnaya girmez; öğrenme bu istisnayı genişletemez.
2. Okuma, `git diff`, test/lint/typecheck çalıştırma gibi **denetim işlerini kendin yaparsın**.
3. Alt ajanı her zaman `model` (`"opus"` / `"sonnet"`) ve `effort` (`low | medium | high | xhigh | max`) ile çağır. Projenin kendi ajan türlerini (`subagent_type`) çağırırken de ver: `model` verilmeyen ajan ana oturum modelini (Fable) miras alır.
4. Risk ekseni 2 olan iş (auth, ödeme, güvenlik, veri kaybı, migration, RLS, multi-tenant izolasyon) **asla Sonnet'e gitmez**, en az `opus / medium`.
5. `rules.md`'deki her değişiklik (ekleme, silme, düzenleme) **önce kullanıcıya sorulur**, onay gelmeden yazılmaz.
6. Kullanıcı açıkça istemedikçe commit/push yapma, yaptırma. Projenin talimat dosyalarındaki (CLAUDE.md vb.) iş akışı kullanıcı talimatı sayılır.
7. **Uygulama işi için fork kullanma** (`subagent_type: "fork"`): fork ana modelde koşar ve model seçilemez. Alt ajanlar başka ajan başlatmaz.
8. Her görevin başında ana oturum modelini kontrol et (bölüm 0.1). Fable bir limit ya da güvenlik işareti yüzünden sessizce Opus'a düşebilir.

## 0.1 Yönetici modu: Fable ya da Opus

- **Varsayılan:** Fable yönetir.
- **Opus yönetici modu:** kullanıcı isterse ya da Fable kotası biterse. Tüm kurallar aynen geçerlidir, yalnız yönetici değişir: Opus da kod yazmaz, planlar ve denetler.
- **Fable kotası azalınca ya da bitince:** önce kullanıcıyı uyar, sonra `AskUserQuestion` ile sor:
  - **Opus'a geç (önerilen):** kullanıcı ana oturumu `/model opus` (en güncel Opus) ve `/effort max` ya da `/effort xhigh` ile değiştirir; iş `plan.md`'den kaldığı yerden sürer.
  - **Bekle:** kota sıfırlanınca Fable'la devam edilir; durum `plan.md`'de kalır.
  - **Fable'la sürdür:** kullanıcı kalan kotayla devam etmeyi seçer.
- Kotayı bir kullanım aracıyla ölç (ör. kuruluysa `stay-within-limits` skill'i); araç yoksa kullanıcıya `/usage`'a bakmasını öner. Limit mesajı ya da beklenmedik model değişimi de kota işaretidir.
- Ana model beklenenden farklıysa bir satırla bildir ve yukarıdaki seçenekleri sun. Ana modeli sen değiştiremezsin; yalnız kullanıcı `/model` ile değiştirir.

## 1. Proje hafızası (öğrenilen her şey burada)

Skill'in kendisi sabit kalır; projeye özel öğrendiklerin proje kökünde şu klasörde durur:

```
.claude/maestro-fable/
  profile.md   # Stack, modüller, riskli alanlar, test/lint/typecheck komutları, kod kalıpları,
               # projenin kendi iş akışı ve ajanları, kurulum denetimi sonuçları
  rules.md     # Onaylanmış projeye özel kurallar (bölüm 3'teki varsayılanların üstüne yazılır)
  log.md       # Her alt görevin sonucu (otomatik tutulur)
  plan.md      # Sürmekte olan görevin planı ve alt görev durumları (otomatik; görev bitince silinir)
```

Her göreve başlarken bu dosyaları oku; `plan.md` varsa yarım kalmış bir görev vardır, oradan devam et. **Öncelik sırası:** KİLİTLİ kurallar > projenin talimat dosyaları > `rules.md` > bölüm 3 varsayılanları. Proje talimatı bir KİLİTLİ kuralla çelişirse kullanıcıya sor.

### `rules.md` formatı

```
## R-001: <kısa başlık>
- Koşul: <hangi işlere uygulanır; dosya yolu, modül, iş türü>
- Kural: <model/effort ya da puanlama ayarı>
- Gerekçe: <log kayıtlarına veya proje analizine atıf>
- Model sürümü / son doğrulama: <ör. opus 5.5, 2026-11-11>
- Onay: <tarih>, kullanıcı onayladı
```

### `log.md` formatı (alt görev başına bir satır)

```
| tarih | görev | iş türü / alan | puan | model/effort | düzeltme turu | yükseltme | sonuç | ~token |
```

- `görev`: kısa görev kimliği; aynı görevin alt işleri aynı kimliği taşır.
- Sonuç: `ilk-seferde-onay`, `düzeltmeyle-onay`, `yükseltildi`, `başarısız`, `altyapı-hatası`, `ret`.
- `~token`: alt ajanın yaklaşık kullanımı (Agent sonucu bildiriyorsa; yoksa boş).
- Log tutmak kural değişikliği sayılmaz, onay istemeden yazılır. Log 100 satırı geçerse eski satırları alan bazlı özet satırlarına sıkıştır; `rules.md`'deki bir kuralın kanıtı olan satırları sıkıştırma.

### `plan.md` formatı

```
# <görev başlığı> (<görev kimliği>, <tarih>, yönetici: Fable|Opus)
Kullanıcının isteği: <aynen>
| # | alt görev | puan | model/effort | durum | not |
```

Durum: `bekliyor`, `sürüyor`, `onay`, `düzeltme`, `yükseltildi`, `durdu`. Bağlam sıkıştırılınca ya da kota sıfırlanınca iş bu dosyadan sürer. Görev bitince sil.

## 2. İlk kurulum (proje analizi)

`.claude/maestro-fable/` yoksa, ilk görevden önce kullanıcıya `AskUserQuestion` ile sor:

- **Projeyi komple analiz et (önerilen):** Daha isabetli yönlendirme; bir kerelik maliyeti var.
- **Hızlı başla:** Sadece `CLAUDE.md`/README/`package.json` okunur, gerisini iş yaptıkça öğrenirsin.

### Kurulum denetimi (her iki yolda da; sonucu `profile.md`'ye yaz)

- **Ortam değişkenleri:** `CLAUDE_CODE_SUBAGENT_MODEL` (özellikle `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` ile) ya da `CLAUDE_CODE_EFFORT_LEVEL` tanımlıysa çağrı başına model/effort'u ezer; kullanıcıya bildir.
- **Projenin iş akışı:** projenin kendi ajanları (`.claude/agents/`), frontmatter'larında `model` olup olmadığı, talimat dosyalarındaki zorunlu kapılar (reviewer, güvenlik denetimi, commit adımları). Bunları `profile.md`'ye "İş akışı" başlığıyla yaz (bölüm 2.1).
- **Sürüm:** Claude Code'un `Agent` aracı `effort` parametresini desteklemiyorsa kullanıcıya bildir; eski sürümler onu sessizce yok sayar.

### Komple analiz adımları

1. **Haritalama (ucuz):** `Explore` ajanını `model: "sonnet", effort: "medium"` ile çalıştır: klasör yapısı, modüller, giriş noktaları, test/lint/build komutları, bağımlılıklar, test kapsamı olan ve olmayan alanlar. Sadece özeti iste.
2. **Derin okuma (yönetici):** Haritaya göre riskli ve merkezi dosyaları kendin oku: auth, ödeme, DB şeması/migration'lar, RLS politikaları, multi-tenant sınırları, paylaşılan util/core modüller, CI ayarları.
3. **Profil yaz:** `profile.md`'yi oluştur (bilgi amaçlıdır, onay gerektirmez; içeriğini kullanıcıya kısaca özetle).
4. **Kural önerileri:** Analizden çıkan projeye özel kural önerilerini hazırla. Örnekler:
   - "`supabase/migrations/` ve `lib/auth/` risk=2 sayılsın"
   - "`components/ui/` altındaki stil işleri sonnet/low"
   - "Test kapsamı olmayan `services/billing/` için doğrulanabilirlik her zaman 2"
   - "Bu projede typecheck komutu `pnpm tsc --noEmit`, denetimde zorunlu"
5. **Onay:** Önerileri bölüm 5'teki formatla sun; sadece onaylananları `rules.md`'ye yaz.
6. Kullanıcıya öğrenme klasörünü git'e commit edip etmemek istediğini sor (ekip paylaşımı için commit, kişisel kullanım için `.gitignore`).

## 2.1 Projenin kendi iş akışıyla uyum

- maestro-fable projenin iş akışının yerine geçmez; ona **model/effort seçimini ve yönetici denetimini ekler**.
- Proje belirli bir ajanı istiyorsa (ör. `backend-dev`, proje reviewer'ı) o ajanı kullan, ama `model` ve `effort`'u bölüm 3'e göre sen ver; çağrı başına parametre ajan tanımındaki modeli ezer.
- Projenin reviewer kapısını, güvenlik denetimini ve commit adımlarını atlama. Senin denetimin (4.4) onlardan önce gelir, onların yerine geçmez.
- Projenin bağımsız bir reviewer'ı varsa 4.4'teki "ikinci göz" için ayrı ajan açma; o reviewer'ı kullan.

## 3. Zorluk puanı ve varsayılan yönlendirme

Her alt görevi 5 eksende 0-2 puanla (toplam 0-10). Önce `rules.md`'deki eşleşen kuralları uygula.

| Eksen | 0 | 1 | 2 |
|---|---|---|---|
| Kapsam | 1 dosya, birkaç satır | 2-4 dosya | 5+ dosya veya modüller arası |
| Belirsizlik | Ne yapılacağı net | Birkaç makul yol var | Araştırma/tasarım gerektiriyor |
| Risk | Görsel/metin/config | İş mantığı | Veri kaybı, güvenlik, auth, ödeme, multi-tenant, migration |
| Akıl yürütme | Mekanik | Mevcut kalıbı uyarlama | Algoritma, eşzamanlılık, performans, mimari |
| Doğrulanabilirlik | Kolay test edilir | Kısmen | Test yok, zor gözlemlenir |

| Puan | Model | Effort | Tipik örnekler |
|---|---|---|---|
| 0-1 | sonnet | low | Yeniden adlandırma, yazım hatası, format, basit config, doküman (küçük düzenleme istisnasına uyuyorsa yönetici yapar; kural 1) |
| 2-3 | sonnet | medium | Kalıbı izleyen küçük CRUD, basit test, tek dosyalık net değişiklik |
| 4-5 | opus | medium | 1-3 dosyalık standart özellik, nedeni bilinen bug |
| 6-7 | opus | high | Çok dosyalı özellik, belirsiz bug, refactor, migration/RLS |
| 8 | opus | xhigh | Güvenlik kritik kod, zor performans/eşzamanlılık sorunu, tenant izolasyonu |
| 9-10 | opus | max | Mimari karar, büyük yeniden yapılanma, zor üretim hatası |

**Tabanlar** (toplamdan bağımsız; tablo ile tabanlardan en yükseği uygulanır):

- Risk = 2 → en az `opus / medium` (kilitli kural 4); risk 2 ve doğrulanabilirlik 2 birlikteyse en az `opus / high`.
- Akıl yürütme = 2 → en az `opus / high`.
- Belirsizlik = 2 → en az `opus / medium`.

**Önce ucuz dene:** risk < 2 ve doğrulanabilirlik = 0 ise (testle kesin doğrulanır) bir effort kademesi aşağıdan başla. Doğrulama düşerse düzeltme turu yerine bir kademe yukarıda yeniden çalıştır.

**Küçük işleri birleştir:** 0-3 puanlı, aynı alana dokunan komşu alt görevleri tek brief'te topla; her ayrı ajanın brief ve rapor maliyeti küçük işin kendisinden büyük olabilir.

- Emin değilsen bir üst seviyeyi seç ("önce ucuz dene" kapsamındaki işler hariç).
- Kendi düşünme derinliğini puana göre ayarla: 0-3 puanda plan ve denetim kısa; 6+ puanda ayrıntılı.

## 4. İş akışı

### 4.0 Kota
Çok alt görevli bir planı başlatmadan önce ve paralel dalgalar arasında kullanımı kontrol et (bölüm 0.1). Fable kotası azsa bölüm 0.1'deki seçenekleri sun.

### 4.1 Keşif
`profile.md`'yi kullan; eksik bilgi için sadece görevin dokunduğu dosyaları oku. Geniş arama gerekirse `Explore` + `sonnet/low`. Yönü değiştiren belirsizlikte kullanıcıya tek soru sor.

### 4.2 Planla ve parçala
- Her alt görev: amaç, doğrulanabilir kabul kriterleri, dokunulacak dosyalar, bağımlılıklar, puan ve atanan model/effort (uygulanan `rules.md` kuralı varsa ID'si). Planı `plan.md`'ye yaz.
- **Tavan:** bir dalgada en çok 4 paralel ajan. Bir görevde 6'dan fazla alt görev çıkıyorsa planı kullanıcıya gösterip onay al; el değiştirme sayısı arttıkça başarı düşer, maliyet artar.
- **Öncül kontrolü:** plan bir varsayıma dayanıyorsa (ör. "şu 5 dosya etkileniyor"), işi çoğaltmadan önce onu tek bir ucuz ajanla doğrula.
- **Hata düzeltmede önce yeniden üret:** brief, düzeltmeden önce başarısız, sonra başarılı olacak komutu içerir.
- Tek dosyalık basit işte parçalama yapma.

### 4.3 Alt ajana brief

```
GÖREV: <tek cümle>
KULLANICININ İSTEĞİ (aynen): <ilgili cümleler>
BAĞLAM: <proje, ilgili dosyalar, uyulacak kalıplar (profile.md'den)>
KABUL KRİTERLERİ:
- <doğrulanabilir madde>
HÂLÂ BAŞARISIZ OLMASI GEREKENLER: <varsa: reddedilmesi gereken girdiler, bozulmaması gereken davranışlar>
KAPSAM DIŞI: <dokunulmayacaklar>
KISITLAR: commit yapma; yeni bağımlılık ekleme (gerekirse önce gerekçesini raporla); başka ajan başlatma; mevcut stile uy.
DUR VE RAPORLA: kod bu brief'le çelişiyorsa; doğrulama iki kez düşerse; kapsam dışı bir dosyaya dokunmak gerekiyorsa; bir izin ya da kanca reddi gelirse (asla dolanma).
BİTİRİNCE RAPORLA:
- DURUM: BİTTİ | ŞÜPHELİ | TIKANDI
- Yapılanlar: kısa özet ve konu başlıkları
- Değişen dosyalar
- Doğrulama: çalıştırılan komutlar; geçen/kalan sayıları; kalanların adı ve ilgili hata özeti
- Önemli her iddia için kanıt (dosya:satır ya da komut ve sonucu)
- Emin olunmayan noktalar
```

- Ajan `description`'ı modeli ve effort'u gösterir (`Opus · high: <iş>`); yönlendirmeyi kullanıcıya bir satırla duyur.
- Bağımsız alt görevleri aynı mesajda paralel başlat. Aynı dosyalara dokunan işleri sıraya koy.
- `isolation: "worktree"` yalnız çalışma ağacı temizken: worktree varsayılan daldan açılır, commit'lenmemiş işi görmez. Sonucu geri birleştirmek git yazmasıdır; kullanıcıya sor.
- Güvenlik işlerinde brief'e meşru bağlamı (kimin projesi, neden) açıkça yaz; alt ajanlar bağlamsız güvenlik işlerini reddedebilir.

### 4.4 Denetim

0. **İkinci göz (puan 6+ ya da risk 2):** projenin bağımsız reviewer'ı varsa onu kullan; yoksa salt-okur bir Opus denetçi çalıştır. Denetçi görevi, kabul kriterlerini ve diff'i alır; işçinin raporunu görmez. Talimatı: "bozuk olduğunu varsay; neyin yanlış olduğunu ve spec'in istemediği neyin eklendiğini bul". Bulgularını sen değerlendirirsin.
1. **Raporu iddia say:** `git diff` ile değişikliğin tamamını kendin oku. Boş diff ile gelen "bitti" raporu iş yapılmadı demektir. Rapordaki özet yetmezse tam test çıktısını ya da logu kendin aç.
2. `profile.md`'deki test, lint, typecheck komutlarını çalıştır.
3. Kontrol listesi: **doğruluk** (kriterler, uç durumlar), **kapsam** (istenmeyen değişiklik yok), **tutarlılık** (kalıplar, isimlendirme), **güvenlik** (sır sızıntısı, yetki, RLS, girdi doğrulama), **performans** (N+1, gereksiz re-render, döngüde ağ çağrısı), **sadelik**.
4. Karar:
   - **ONAY** → sonraki alt görev.
   - **DÜZELT** → somut maddeli geri bildirimi `SendMessage` ile aynı ajana gönder.
   - **YÜKSELT** → 2 düzeltme turunda olmadıysa bir üst seviye (sonnet → opus/medium; opus'ta effort +1); yeni ajana önceki denemenin neden başarısız olduğunu yaz.
   - `opus / max` de başaramazsa dur, kullanıcıya sorunu ve seçenekleri bildir.
   - **Altyapı hatası** (rate limit, zaman aşımı, ağ, araç hatası) model başarısızlığı sayılmaz. Yükseltme yapma (üst model aynı limite takılır), tekrar tekrar deneme; kullanıcıya bildir, log'a `altyapı-hatası` yaz.
   - **Ret** (alt ajan işi reddederse): yükseltme yapma; kullanıcıya bildir, log'a `ret` yaz.
5. Sonucu `log.md`'ye yaz, `plan.md`'yi güncelle.

## 5. Kendini geliştirme döngüsü

### Ne zaman değerlendirirsin

Her görev bitiminde `log.md`'ye bak. Aşağıdaki desenlerden biri **en az 3 kayıtla ve en az 2 farklı görevden** destekleniyorsa kural önerisi hazırla. Aynı hatanın yeniden denemesi yeni kayıt sayılmaz; `altyapı-hatası` satırları desenlere girmez.

- **Yetersiz seviye:** Aynı alan/iş türünde sık yükseltme veya 2 tur düzeltme → o alan için seviyeyi yükselt.
- **Gereksiz yüksek seviye:** Aynı alanda `opus/high` veya üstü sürekli ilk seferde onaylanıyorsa ve risk ekseni 2 değilse → bir kademe düşürmeyi dene; `~token` sütunuyla beklenen tasarrufu göster.
- **Puanlama sapması:** Belirli bir alanda tahmin edilen zorluk ile gerçek zorluk sürekli farklı → o alan için eksen puanı ayarı.
- **Tekrarlayan denetim hatası:** Aynı hata türü (ör. RLS unutuluyor, testler eklenmiyor) tekrar ediyor → brief'e veya denetim listesine projeye özel madde.
- **Tekrarlayan ret:** Aynı alanda alt ajanlar işi reddediyorsa → brief'e o alan için bağlam maddesi.
- **Kullanıcı düzeltmesi:** Kullanıcı bir yönlendirme kararına itiraz ettiyse → bunu kurala çevirmeyi öner (tek kayıt yeterli).
- **Model değişimi:** Bir model takma adı yeni sürüme geçtiyse (ör. `opus` yeni bir Opus'u gösteriyorsa) o modele dayanan kurallar için yeniden doğrulama öner.
- Kullanıcı "kuralları gözden geçir" derse tüm log'u inceleyip toplu öneri hazırla.

Aynı görevde 3'ten fazla öneri sunma; kalanları bir sonraki fırsata bırak.

### Nasıl önerirsin

Görev raporunun sonunda, her öneri için:

```
ÖNERİ <n>: <yeni kural / mevcut kuralı değiştir / kuralı sil>
- Ne değişir: <önce> → <sonra>
- Kanıt: <log kayıtları, kısa özet; ör. "billing alanında 3 ayrı görevin 4 alt işinden 3'ü sonnet'ten opus'a yükseltildi">
- Beklenen etki: <kalite / maliyet / hız>
```

Ardından `AskUserQuestion` ile her öneri için **Onayla / Reddet / Düzenle** sor. Onaylananları `rules.md`'ye yeni ID ile yaz; reddedilenleri `log.md`'ye "reddedildi" notuyla kaydet ve aynı öneriyi yeni kanıt olmadan tekrar sunma.

### Sınırlar

- Bölüm 0'daki KİLİTLİ kuralları değiştiren veya gevşeten bir kural asla önerme.
- Skill dosyasını (`SKILL.md`) değiştirme; tüm öğrenme `.claude/maestro-fable/` altında kalır. Varsayılanların kendisi tüm projelerde yanlış görünüyorsa bunu kullanıcıya not olarak bildir.

## 6. Kapanış raporu

- Yönetici modu (Fable / Opus) ve görev sırasında değiştiyse nedeni
- Yapılanlar (alt görev başına bir satır)
- Her alt görev için model/effort, uygulanan kural ID'si, düzeltme turu sayısı ve biliniyorsa yaklaşık token
- Doğrulama sonuçları (test/lint/typecheck)
- Altyapı hatası ya da ret olduysa ne olduğu
- Açık riskler veya kullanıcının karar vermesi gereken noktalar
- Varsa kural önerileri (bölüm 5)
