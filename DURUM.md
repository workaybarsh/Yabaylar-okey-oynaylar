# Durum ve Devam Notu

> Yeni bir sohbet/oturum başlarken hızlı bağlam için. Ayrıntılı tasarım: [`plan.md`](plan.md),
> [`MOTOR-TASARIM.md`](MOTOR-TASARIM.md), [`README.md`](README.md) · sunum: [`sunum/teknoloji.html`](sunum/teknoloji.html).

## Proje
**101 Okey** oynanabilen ve aynı masada **yapay zekâ rakiplerle sohbet** edilebilen oyun.
Hedef platformlar: Windows + Android + Web (henüz düşünce/plan aşaması).
Repo: `github.com/workaybarsh/Yabaylar-okey-oynaylar`

## Tamamlananlar
- **Tam tasarım planı** (`plan.md`): 101 Okey kuralları, mimari katmanlar (motor = **hakem**),
  AI "serbest muhakeme + son denetim" modeli, bilgi görünürlüğü, hamle protokolü, ret döngüsü,
  sohbet/eş iletişimi, BYOK sağlayıcı modeli.
- **Oyun motoru genel tasarımı** ([`MOTOR-TASARIM.md`](MOTOR-TASARIM.md)): taş modeli (106 taş,
  benzersiz kimlik), gösterge/okey/sahte okey türetimi (13→1 sarması), per doğrulama (set/seri/çift
  + joker yerleştirme), **101 puan / 5 çift açma**, sıra akışı **aşama makinesi** (çek + işle + at),
  bitiş + puanlama/ceza matrisi, görünürlük (`viewFor`), tohumlu RNG + serileştirme, hakem API'si,
  modül yapısı, test stratejisi ve **`RuleSet`** ile kural yapılandırması (açık sorular dahil).
- **Kilitli kural kararları** (16 Eki 2026): sahte okey = "okey yerine", gösterge alt taş (zincirleme),
  artırmalı açma yok, okeyi yerinden alma var, eş/takım puanları birleşik — bkz. `MOTOR-TASARIM.md` §15.
- **TR + EN dil kararı:** arayüz + AI tablo dili iki dilli; motor dilden bağımsız (kod + parametre).
- **Teknoloji sunumu** ([`sunum/teknoloji.html`](sunum/teknoloji.html) · metin [`sunum/teknoloji.md`](sunum/teknoloji.md)):
  5 seçeneğin karşılaştırması, **Web-first PWA** önerisi, paketleme (PWA / Tauri / Capacitor), i18n ve riskler.

## Önemli kararlar (özet)
- **Oyun:** 101 Okey — yere açma **≥101 puan** ya da **5 çift**; tüm kural detayları harfiyen
- **Motor = hakem:** hamle önermez; **sadece sonuçta** doğrular, "şundan ötürü mümkün değil" der
- **AI serbest:** kötü seçim/hata/olmayanı deneme serbest; **tool/ipucu/aday liste YOK**
- **Yapay kişilik YOK:** her model (Gemini/DeepSeek/GPT/Claude) kendi doğal üslubuyla konuşur
- **Ret sınırı YOK** + **10 sn hız uyarısı** (yalnızca o tura özel)
- **Sağlayıcı:** BYOK — DeepSeek/OpenAI/Ollama/Gemini/Claude
- **Diller:** **Türkçe + İngilizce** (arayüz + AI tablo dili); motor dilden bağımsız (kod + parametre)

## Sıradaki adım
> Motorun **genel tasarımı bitti** ([`MOTOR-TASARIM.md`](MOTOR-TASARIM.md)). Sıra, karar verilince
> bu tasarımı koda dökmekte.
1. **Dil/teknoloji kararı** (plan.md §11) — motor modül sınırları dilden bağımsız hazır
2. **Motor iskeleti + implementasyon** (arayüzsüz, test edilebilir): taş/deste, gösterge-okey,
   per doğrulama, 101/5 çift açma, sıra akışı, puanlama — *her şeyin bel kemiği*
3. Masa arayüzü (sürükle-bırak)
4. AI oyuncu (serbest muhakeme + hakem ret döngüsü)
5. Sohbet & eş iletişimi
6. Sağlayıcı soyutlaması
7. Paketleme (Windows + Android + Web)

## Açık kararlar (plan.md)
- [ ] Platform/teknoloji — **öneri: Web-first PWA** (sunum: [`sunum/teknoloji.html`](sunum/teknoloji.html))
- [ ] Kaç rakip, hangi modeller
- [ ] Çok oyunculu (gerçek kişiler) sonraya mı

## Motora özel kararlar (16 Eki 2026)
> Ayrıntı ve gerekçe: [`MOTOR-TASARIM.md`](MOTOR-TASARIM.md) §15 (`RuleSet` + Karar Günlüğü).
- [x] **Sahte okey:** "okey yerine" geçer (evrensel joker **değil**)
- [x] **Gösterge sahte okey çıkarsa:** bir alt taş gösterge olur; gerekirse **zincirleme** iner
- [x] **Artırmalı açma:** **yok** — her açan 101'den başlar
- [x] **Okeyi yerinden alma:** **var** (gerçek taşla değiştirip okeyi ele al)
- [x] **Eş/takım puanları:** **birleştirilir** (takım toplamı üzerinden)

## Kalan motor varsayılanları
- [ ] İlk oyuncu 22'li (dağıtanın sağı) · deste bitince gösterge çekme · deste bitişi puanı · tur sayısı
  → kaynak varsayılanı uygulanır; itiraz olursa yalnızca `RuleSet` değeri değişir

## Oturum kapanışı — 16 Eki 2026
Bu oturumda **kod yazılmadı**; tasarım tamamlandı. Olgunlaşan dosyalar:
`plan.md`, `MOTOR-TASARIM.md`, `DURUM.md`, `BASLA.md`, `README.md`, `sunum/`.

**Yarın buradan devam:**
1. **Platform kararı** → toplantı/sunum sonrası onay (öneri: Web-first PWA).
2. Onay gelirse: **motor iskeleti** (TypeScript + Vitest) — `MOTOR-TASARIM.md` §12 modül yapısı ve
   §11 API'si birebir kodlanır; ilk hedef `tiles/deck/okey/meld` + testler.
3. Hâlâ açık: kaç rakip & hangi modeller · çok oyunculu sonraya mı · tur sayısı (`dealCount`).
