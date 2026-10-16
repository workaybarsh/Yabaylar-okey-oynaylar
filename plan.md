# 101 Okey — Yapay Zekâ ile Oynanabilen Oyun Uygulaması

> Durum: Tasarım / planlama aşaması. Henüz kod yok.

---

## 1. Vizyon

Kullanıcının **101 Okey** oynayabildiği ve aynı masada yapay zekâ rakiplerle
**sohbet/muhabbet** edebildiği bir oyun. Yapay zekâ rakipler:

- Kuralları bilir ve **iyi oynamaya çalışır**,
- Ama **kendi muhakemesiyle serbest** hareket eder — hata yapabilir, kötü seçim
  yapabilir, olmayan bir şeyi denemeye çalışabilir,
- Yapay zekânın "beyninin" üstüne **hiçbir tool / yardımcı / puanlama / aday liste
  eklenmez**. Ona dokunulmaz.
- Denetim yalnızca **en sonda**, hamle uygulanırken yapılır.

---

## 2. Oyun: 101 Okey — Referans Kurallar

> ⚠️ Bu başlık, motorun temeli. Oynanacak tam kural seti kesinleşmeli (bkz. bölüm 11).

**Taşlar**
- 106 taş: 4 renk (kırmızı, siyah, mavi, sarı) × 13 sayı (1–13) × 2 kopya = 104,
  + 2 joker taşı.
- Gösterge (işaret taşı) açılır → **okey taşı = göstergenin bir üstü aynı renk**
  (13'ten sonra 1'e döner).
- Okey (joker) herhangi bir taşın yerine geçer.

**Düzen**
- 4 oyuncu, **2 takım**; karşılıklı oturanlar eş (A–C ve B–D).
- El başına 14 taş; bir taş gösterge olarak ortaya konur.
- Sırayla: **çek + at** döngüsü.

**Çekme (kararı oyuncuya/AI'ya ait)**
- Ortadaki (yer) son atılan taş **alınabilir**, VEYA
- desteden (kapalı) taş çekilir.
- AI bunu **kendisi belirler**.

**Per türleri**
- Aynı sayıdan farklı renkler (en az 3).
- Aynı renkten ardışık sayılar (en az 3, 1–13 arası).
- Çift (pair) — açma için 5 çift alternatifi.

**Açma (kritik kural)**
- Oyuncu, **yere açmadıkça** (per koymadıkça) oyunu bitiremez.
- **101 Okey'de açma eşiği: en az 101 puan** (açılan gruplardaki taş sayılarının
  toplamı), **VEYA 5 çift ile açma.**
- Okey (joker) yerine geçtiği taşın değeriyle sayılır.
- Açıldıktan sonra masadaki perlere ekleme yapılabilir (işleme).

**Bitme**
- Gerekli açmayı yaptıktan sonra eldeki taşları perlere dizip son taşı atan oyuncu
  **biter / eli kazanır**.
- Kalan taşlara göre ceza puanları hesaplanır (tam formül kesinleşmeli).

**Açık kalan kural detayları**
- Sahte okey (göstergenin eşi) davranışı,
- Okeyi yerinden alma (elinde gerçek taş varsa okeyi değiştirme),
- Açmadan önce geçerli taş atma cezası (101 ceza puanı) uygulanacak mı,
- 14 mü 15 mi başlangıç taşı (dağıtan/yürüyen farkı).

---

## 3. Mimari Katmanlar

```
┌───────────────────────────────────────────────┐
│  Arayüz (masa, taşlar, sohbet, düşünme anı)   │  platform bağımsız
├───────────────────────────────────────────────┤
│  Oyun Motoru (101 Okey kuralları, HAKEM)       │  saf kod, %100 güvenilir
│   - durum yönetimi, deste, gösterge, sıra      │
│   - hamle doğrulama (yalnızca sonuçta)         │
├───────────────────────────────────────────────┤
│  AI Oyuncu (serbest muhakeme)                  │  LLM
│   - kurallar + kendi durumu + ortak masa       │
│   - "iyi oynamaya çalış" talimatı              │
├───────────────────────────────────────────────┤
│  Sağlayıcı Arayüzü  →  sor(mesajlar)           │  tek fonksiyon
│   ├─ DeepSeek (OpenAI-uyumlu)                  │
│   ├─ OpenAI                                    │
│   ├─ Ollama (yerel, anahtarsız)                │
│   ├─ Gemini                                    │
│   └─ Anthropic / Claude                        │
└───────────────────────────────────────────────┘
```

**Motorun rolü: HAKEM.** Ön filtre yapmaz, hamle önermez, yardım etmez.
Sadece hamle geldiğinde "mümkün mü?" diye bakar.

---

## 4. AI Oyuncu Modeli — "Serbest Muhakeme + Son Denetim"

**Yapılmayacaklar (bilinçli karar):**
- ❌ Koç notu / ipucu
- ❌ Aday hamle listesi
- ❌ Motor puanlaması / pozisyon değerlendirmesi
- ❌ Function calling / tool

**AI'nın bildikleri:**
- 101 Okey kuralları,
- "Kazanmak için elinden gelenin en iyisini yap; kasıtlı kötü oynama" talimatı,
- Kendi durumu ve ortak masa bilgisi.

**AI'nın serbestlikleri:**
- Kötü seçim yapabilir, hata yapabilir, olmayan bir şeyi deneyebilir.
- Denetim yalnızca hamle anında.

---

## 5. Bilgi Görünürlüğü

| Bilgi | AI görür mü? |
|---|---|
| Kendi eli | ✅ Evet |
| Gelen taş | ✅ Evet |
| Ortak masa (atılan taşlar, açılan perler, gösterge, destede kalan sayı) | ✅ Evet |
| Diğer oyuncuların eli | ❌ Hayır |

---

## 6. Hamle Protokolü

- AI, kararını **düşünerek** açıklar ve hamleyi uygular.
- Programın anlayabilmesi için hafif bir konuşma protokolü kullanılır
  (bu bir tool değil, sadece format).

Örnek (kesin format belirlenecek):
```
DÜŞÜNCE: ...
HAMLE: at { kırmızı-9 }
SOHBET: "bu taş bana lazım değil"
```
veya JSON:
```json
{ "hamle": { "tip": "at", "tas": "kirmizi-9" },
  "sohbet": "bu taş bana lazım değil" }
```

Hamle tipleri: `desteden_cek`, `yerden_al`, `at`, `per_ac`, `per_isle`,
`okey_yerlestir`, `bitir`.

---

## 7. Ret ve Yeniden Deneme

```
1. AI düşünür ve hamlesini uygulamaya çalışır
2. Motor doğrular:
   ├─ Mümkünse → uygula
   └─ Mümkün değilse → NEDENİNİ söyle ve geri gönder
3. AI tekrar düşünür (karar kendisine ait)
4. Ret sınırı YOK — doğru/hamle gerçekleşene kadar devam
```

Örnek ret mesajı:
```
Bu hamle mümkün değil: "kırmızı-9" elinde yok.
Açma için en az 101 puan gerekiyor; senin grupların 74.
```
> ⚠️ Risk: sınırsız deneme, AI inatla yanlış denerse döngü/token tüketimi yaratır.
> Karar: sınır yok. Arayüzde "AI düşünüyor..." durumu gösterilecek.

**Ret edilen:** yalnızca kural/fizik ihlali (olmayan taş, geçersiz per, 101 altı
açma, sırasız hamle). Kötü-ama-geçerli hamle **reddedilmez**.

---

## 8. Sohbet ve Eş İletişimi

- AI, oyuncularla **genel muhabbet** edebilir.
- **Oyuna dair bilgi sızdırmak kural ihlali** → yapmaz.
- **Eş (partner) ile:** genel muhabbet + **aşırı bariz olmayan, çaktırmadan,
  üstü kapalı iş birliği** denenebilir (kuralı apaçık çiğnemeden, sezdirmeden).

---

## 9. Sağlayıcı ve Hesap Modeli

- Uygulama **lokal** çalışır.
- **BYOK (Kendi API Anahtarını Getir)** modeli: kullanıcı kendi anahtarını girer.
- Anahtar cihazda **şifreli** saklanır (Windows Credential Manager / Android
  Keystore); sunucuya gitmez, geliştiriciye maliyet yok.
- DeepSeek / OpenAI / Ollama **OpenAI-uyumlu** olduğu için tek istemci + base URL
  değişimi yeterli. Gemini ve Claude için küçük adaptör.
- ❌ Abonelik hesabıyla giriş (ChatGPT Plus vb.) desteklenmez: resmî API yok,
  kırılgan ve kural dışı.

Sağlayıcı seviyesi:

| Sağlayıcı | Uyumluluk | Anahtar |
|---|---|---|
| DeepSeek | OpenAI-uyumlu | Zorunlu (ucuz) |
| OpenAI | Referans | Zorunlu |
| Ollama | OpenAI-uyumlu | Gerekmez (yerel, bedava) |
| Gemini | Adaptör | Ücretsiz katman var |
| Anthropic | Adaptör | Ücretli |

---

## 10. Yol Haritası

1. **Oyun motoru** — 101 Okey kuralları, durum, hamle doğrulama. Arayüzsüz + test.
2. **Masa arayüzü** — taşları göster, sürükle-bırak, sıra akışı.
3. **AI oyuncu** — sistem promptu + serbest muhakeme + hakem ret döngüsü.
4. **Sohbet & eş iletişimi** katmanı.
5. **Sağlayıcı soyutlaması** — Gemini / DeepSeek / Ollama.
6. **Paketleme** — Windows + Android + Web.

---

## 11. Açık Kararlar

- [ ] Platform/teknoloji (Web-first PWA mı, Flutter mı?) — öneri: Web-first PWA.
- [ ] 101 Okey kural detayları (bölüm 2 sonundaki 4 madde).
- [ ] Hamle protokolü formatı: kısa komut mu, JSON mu?
- [ ] Yapay zekâ kişilikleri: kaç rakip, farklı karakterler?
- [ ] Çok oyunculu (gerçek kişiler) sonraya mı?
