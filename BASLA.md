# BASLA — yeni sohbet başlangıç notu

**Proje:** Yabaylar okey oynaylar — **101 Okey** oynanabilen ve yapay zekâ rakiplerle sohbet
edilebilen oyun (hedef: Windows + Android + Web).
**Çalışma dizini:** bu klasör.
**Tasarım:** [`plan.md`](plan.md) (genel) · [`MOTOR-TASARIM.md`](MOTOR-TASARIM.md) (oyun motoru).

---

## Yeni sohbete yapıştıracağın ilk mesaj

```
Proje: Yabaylar okey oynaylar — 101 Okey + yapay zekâ rakipler.

Önce şu dosyaları oku: DURUM.md, MOTOR-TASARIM.md ve plan.md (tam tasarım planı).

Özet:
- Oyun: 101 Okey (açma ≥101 puan ya da 5 çift), tüm kural detayları harfiyen uygulanacak.
- Mimari: Oyun motoru = HAKEM (hamle önermez, sadece sonuçta doğrular).
  AI serbest muhakeme eder; tool/ipucu/aday liste YOK. Yapay kişilik YOK (her model kendi üslubuyla).
  Ret sınırı YOK + 10 sn hız uyarısı (tura özel). Sağlayıcı: BYOK (DeepSeek/OpenAI/Ollama/Gemini/Claude).
- Hamle protokolü: model serbest metin + sonda <hamle>{JSON}</hamle> bloğu.

Durum: Oyun motorunun genel tasarımı bitti (MOTOR-TASARIM.md); kural kararları §15'te kilitlendi.
Sıradaki iş: dil/teknoloji kararı (plan.md §11) + motor iskeleti/implementasyonu
(arayüzsüz, test edilebilir): 106 taş, gösterge/okey, per doğrulama, 101 açma, sıra akışı.
```
