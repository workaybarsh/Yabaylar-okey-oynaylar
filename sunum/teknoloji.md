# Teknoloji Seçimi — Sunum (metin)

> Yabaylar Okey Oynaylar · 101 Okey + yapay zekâ rakipler
> Slayt destesi: [`teknoloji.html`](teknoloji.html) · Tarih: 16 Ekim 2026

---

## 1 — Kapak

**YABAYLAR OKEY OYNAYLAR**
# Teknoloji Seçimi

101 Okey + yapay zekâ rakipler · Windows + Android + Web.
Tek kod tabanı, arayüzsüz hakem motoru, TR + EN.

---

## 2 — Neyi inşa ediyoruz?

Karar, aşağıdaki gerçeklere göre verilmeli.

- **01 · 3 platform, 1 ürün** — Windows masaüstü, Android ve Web (PWA). Aynı oyun, aynı kurallar.
- **02 · Arayüzsüz hakem motoru** — Saf mantık; UI'dan bağımsız, %100 test edilebilir. Hamle *önermez*, doğrular.
- **03 · Sürükle-bırak masa** — Taş dizme, per açma, animasyonlar, dokunmatik. Görsel kalite önemli.
- **04 · BYOK + yerel çalışma** — Kullanıcı kendi API anahtarını girer; cihazda **şifreli** saklanır, sunucu yok.
- **05 · TR + EN** — Arayüz ve yapay zekâ tablo dili iki dilli. Motor dilden bağımsız (kod + parametre).
- **06 · LLM çağrıları** — OpenAI-uyumlu HTTP (DeepSeek/OpenAI/Ollama/Gemini/Claude) + streaming.

---

## 3 — Hangi kriterlere bakıyoruz?

- **Tek kod tabanı** — Aynı kod Windows + Android + Web'de. Üç ayrı uygulama yazmak yok.
- **Motorun paylaşımı** — Hakem motoru tüm platformlarda **aynı** çalışmalı; tek test seti.
- **Arayüz gücü** — Sürükle-bırak, animasyon, dokunmatik → "yerli" his.
- **Paketleme kolaylığı** — Mağaza/dağıtım: .exe, Play Store, tarayıcı.
- **Ekosistem & i18n** — TR/EN sözlük, LLM istemcisi, hazır bileşenler.
- **Geliştirici hızı** — Teknoloji kararı ilk prototipi ne kadar hızlandırır?

> Kriterler eşit ağırlıklı değil: önceliğimiz **tek kod tabanı + hızlı teslim**.

---

## 4 — Karşılaştırma

| Teknoloji | Tek kod | Motor | Arayüz | Paketleme | Ekosistem | Not |
|---|---|---|---|---|---|---|
| **Web-first PWA · TypeScript** | ●●● | ●●● | ●● | ●● | ●●● | ⭐ **ÖNERİLEN** |
| Flutter · Dart | ●●● | ●● | ●●● | ●●● | ●● | güçlü alternatif |
| React Native · Expo | ●●● | ●● | ●● | ●● | ●● | mobil odaklı |
| Tauri · Rust + Web UI | ●● | ●●● | ●● | ●● | ●● | masaüstü odaklı |
| Unity / Godot | ●●● | ● | ●●● | ●●● | ●● | ağır / gereksiz |

> ●●● güçlü · ●● orta · ● zayıf

---

## 5 — Öne çıkanlar (artı / eksi)

**Web-first PWA** *(öneri)*
- \+ tek dil (motor = TS), hızlı prototip
- \+ i18n ve LLM istemcisi hazır ekosistem
- \+ tarayıcıda anında; Tauri/Capacitor ile paket
- − WebView animasyon/performans sınırı
- − mobil "native his" için ekstra özen

**Flutter · Dart**
- \+ tek kod, gerçek native performans
- \+ güçlü animasyon, tutarlı görünüm
- − motor **Dart** ile yeniden yazılır
- − web çıktısı hâlâ en zayıf halka

**React Native · Expo**
- \+ mobilde olgun ve hızlı
- − web desteği (RNW) zayıf → 3. platform sorunlu
- − Windows masaüstü ekstra iş

**Ortak sonuç** — Motor zaten **dilden bağımsız** tasarlandı; asıl fark, arayüzü kaç kod tabanıyla
yazacağımız ve paketleme maliyeti.

---

## 6 — Öneri: Web-first PWA (TypeScript)

Tek dil, tek motor, üç platform. Motor tasarımı zaten TS-benzeri API'lerle yazıldı.

`Oyun Motoru (TS)` → `Masa Arayüzü` → `AI Katmanı` → `Sağlayıcı (BYOK)`

- Motor saf TS → tarayıcı, Android ve Windows'ta **aynı** kod, aynı testler (`Vitest`).
- Arayüz: **Vite** + **Svelte** (veya React) — hafif, hızlı, sürükle-bırak için ideal.
- LLM: `fetch` ile OpenAI-uyumlu istemci (DeepSeek/OpenAI/Ollama) + Gemini/Claude adaptörü.
- i18n: **i18next** ile TR/EN sözlükleri; motor yalnızca kod + parametre döner.

---

## 7 — Tek kod tabanı → üç platform

- **WEB · PWA** — Statik site + service worker. Çevrimdışı, kurulabilir, anında güncellenir.
- **ANDROID · Capacitor** — Web kodu native kabukta; Play Store paketi. (Alternatif: TWA)
- **WINDOWS · Tauri** — Rust kabuğu, ~küçük .exe. (Alternatif: Electron)

`Tek UI kodu` → `PWA` / `Capacitor` / `Tauri`

> Anahtar güvenliği: `Tauri Stronghold` / `Capacitor Secure Storage` → OS anahtarlığı
> (Windows Credential Manager, Android Keystore).

---

## 8 — TR + EN nasıl işler?

- **MOTOR · Dilden bağımsız** — Metin üretmez; `{ reason, params }` döner.
  `{ reason:"OPEN_BELOW_THRESHOLD", params:{have:74, need:101} }`
- **ARAYÜZ · Sözlük (i18n)** — Aynı kodu TR/EN metne çevirir.
  *TR:* "Açma için en az 101 puan gerekiyor; grupların 74."
  *EN:* "You need at least 101 to open; your melds total 74."
- **YAPAY ZEKÂ · Tablo dili** — Sistem promptu seçili dile göre kurulur; model zaten o dilde konuşur
  (doğal üslup, yapay kişilik yok).
- **TAŞLAR · Anahtar gösterim** — `kirmizi-9` dilden bağımsız anahtardır; renk adları ekranda
  yerelleştirilir.

---

## 9 — Riskler ve önlemler

| Risk | Önlem |
|---|---|
| WebView animasyon/performans | Canvas/GPU (WebGL) katmanı; kritik animasyonları ölç, gerekirse native köprü |
| Mobilde "native his" | Capacitor + native geçişler; dokunma/haptic ince ayarı |
| API anahtarı güvenliği (BYOK) | Cihaz anahtarlığı; anahtar sunucuya **gitmez** |
| Tek dile bağımlılık (TS) | Motor saf ve taşınabilir; ileride Dart/C# portu mümkün |
| Web'de LLM CORS | Yerel proxy veya sağlayıcı izinleri; Tauri/Capacitor'da sorun yok |

---

## 10 — Karar & sıradaki adım

**Karar: Web-first PWA**
- Motor + UI tek dilde (TypeScript)
- Windows (Tauri) · Android (Capacitor) · Web (PWA)
- TR/EN ve LLM entegrasyonu en kolay

**Sıradaki adım**
- Onay → motor iskeleti (`Vitest`) + testler
- Masa arayüzü prototipi (sürükle-bırak)
- AI katmanı (ret döngüsü) ve BYOK

> Alternatif: **Flutter** — native his öncelikliyse.
