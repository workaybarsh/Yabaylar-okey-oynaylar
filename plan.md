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

> Bu başlık motorun temelidir. Aşağıdaki tüm kurallar **harfiyen** uygulanacaktır.

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
- Kalan taşlara göre ceza puanları hesaplanır.

**Kural detayları (harfiyen uygulanır)**
- Sahte okey (göstergenin eşi) davranışı uygulanır.
- Okeyi yerinden alma: elinde okey yerine geçen gerçek taş varsa (açmış olmak
  şartıyla) okeyi alabilir.
- Açmadan önce geçerli taş atma cezası: 101 ceza puanı uygulanır.
- Başlangıç taş sayısı (dağıtan/yürüyen farkı dahil) standart 101 Okey kuralına
  göre uygulanır.

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

**Üslup / kişilik (bilinçli karar):**
- AI'lara **yapay / yapmacık kişilik, ton veya karakter enjekte edilmez.**
- Her model (Gemini, DeepSeek, GPT, Claude, yerel modeller) **eğitildiği kendi
  doğal cevap verme biçimiyle** konuşur. Çeşitlilik zaten kendiliğinden gelir.
- Sistem promptu **nötr ve işlevseldir**: kurallar + görev. Karakter yazılmaz.

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

Programın cevabı ayrıştırabilmesi için tek bir biçim kullanılır. Bu bir **tool
değil**, yalnızca bir biçim/konuşma kuralıdır.

AI **serbestçe düşünür ve konuşur**; cevabın **sonunda** hamlesini özel bir blokta
verir:

```
(düşünce ve masa sohbeti — serbest metin, AI'nın kendi sesi)
<hamle>
{ ... JSON ... }
</hamle>
```

- Blok **dışındaki** metin = AI'nın kendi doğal sesi (düşünce + isteğe bağlı masa
  sohbeti). Model burada eğitildiği üslupla yazar.
- Blok **içindeki** JSON = makinenin okuyacağı hamle. **Tek zorunlu kısım budur.**

**JSON şeması**

```json
{
  "tip": "yerden_al",
  "tas": "kirmizi-9",
  "perler": [["kirmizi-5", "mavi-5", "siyah-5"]],
  "sohbet": "bu taş bana iyi geldi"
}
```

| Alan | Zorunlu | Açıklama |
|---|---|---|
| `tip` | ✅ | Hamle türü (aşağıda) |
| `tas` | duruma göre | Tek taş gerektiren hamlelerde |
| `perler` | duruma göre | Açma/işleme grupları |
| `sohbet` | ❌ | Masaya söylenecek söz (isteğe bağlı) |

**Hamle türleri**

| tip | Anlamı | Alan |
|---|---|---|
| `desteden_cek` | Kapalı desteden çek | — |
| `yerden_al` | Yerdeki son taşı al | `tas` |
| `at` | Elinden taş at | `tas` |
| `per_ac` | Yere per aç (≥101 puan ya da 5 çift) | `perler` |
| `per_isle` | Masadaki pere ekle | `hedef`, `tas` / `perler` |
| `okey_degistir` | Masadaki okeyi gerçek taşla değiştir | `hedef`, `tas` |
| `bitir` | Açtıktan sonra son taşı atıp eli bitir | `tas` |

**Taş gösterimi:** `renk-sayı` → `kirmizi-9`, `sari-3`, `mavi-13`.
Joker: `okey`. (Motor içeride her taşın ayrı kimliğini tutar; AI için okunabilir
gösterim yeterlidir.)

**JSON bozuksa:** ayrıştırma hatası da bir **ret** sayılır → AI'dan düzeltmesi
istenir.

---

## 7. Ret ve Yeniden Deneme

```
1. AI düşünür ve hamlesini uygulamaya çalışır
2. Motor doğrular:
   ├─ Mümkünse → uygula
   └─ Mümkün değilse → NEDENİNİ söyle ve geri gönder
3. AI tekrar düşünür (karar kendisine ait)
4. Ret sınırı YOK — geçerli hamle gerçekleşene kadar devam
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

**Hız uyarısı — 10 saniye (yalnızca o tura özel):**
- Tur başladığından itibaren **10 saniye** içinde geçerli bir hamle üretilemezse,
  sisteme **o tur için** şu talimat eklenir: *"Daha hızlı ol, şu an geçerli bir
  hamle bekleniyor."*
- Uyarı **sadece o tur için** geçerlidir; tur bitince kaldırılır, kalıcı değildir.
- Ret sınırı yine **yoktur** — uyarı yalnızca hızlandırma amaçlıdır, döngüyü
  sonlandırmaz.

---

## 8. Sohbet ve Eş İletişimi

- AI, oyuncularla **genel muhabbet** edebilir.
- Her model **kendi doğal üslubuyla** konuşur; yapay kişilik/ton eklenmez
  (bkz. bölüm 4).
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
   → Ayrıntılı motor tasarımı: [`MOTOR-TASARIM.md`](MOTOR-TASARIM.md)
2. **Masa arayüzü** — taşları göster, sürükle-bırak, sıra akışı.
3. **AI oyuncu** — sistem promptu + serbest muhakeme + hamle protokolü + ret döngüsü.
4. **Sohbet & eş iletişimi** katmanı.
5. **Sağlayıcı soyutlaması** — Gemini / DeepSeek / Ollama.
6. **Paketleme** — Windows + Android + Web.

---

## 11. Açık Kararlar

- [ ] Platform/teknoloji (Web-first PWA mı, Flutter mı?) — öneri: Web-first PWA.
  *Motor modül sınırları ve API dilden bağımsızdır → [`MOTOR-TASARIM.md`](MOTOR-TASARIM.md).*
- [x] 101 Okey kural detayları → **harfiyen uygulanacak** (bölüm 2).
- [x] Hamle protokolü → **JSON blok** (bölüm 6).
- [x] Yapay zekâ kişilikleri → **yapay kişilik yok**; modeller doğal üslubuyla.
- [x] Ret sınırı → **yok** + 10 saniye hız uyarısı (tura özel).
- [ ] Kaç rakip ve hangi modeller (Gemini/DeepSeek/GPT/Claude karışık mı?).
- [ ] Çok oyunculu (gerçek kişiler) sonraya mı?
