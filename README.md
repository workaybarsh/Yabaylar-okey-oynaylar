# Yabaylar Okey Oynaylar

101 Okey oynayabildiğin ve aynı masada **yapay zekâ rakiplerle sohbet edebildiğin**
bir oyun uygulaması.

> 🚧 Durum: **Tasarım / planlama aşaması.** Henüz kod yok.

---

## Bu proje nedir?

- **Oyun:** 101 Okey (106 taş, 4 oyuncu, 2 takım — karşılıklı eşler). Yere açmak
  için en az **101 puan** ya da **5 çift** gerekir.
- **Yapay zekâ rakipler:** Kuralları bilir, iyi oynamaya çalışır ama kendi
  muhakemesiyle **serbest** hareket eder. Hata yapabilir, kötü seçim yapabilir,
  olmayan bir şeyi deneyebilir.
- **Denetim modeli:** Motor (oyun kuralları) **sadece en sonda** hamleyi denetler.
  Mümkün olmayan bir hamle gelirse "şundan ötürü mümkün değil" diyerek geri
  gönderir, AI tekrar düşünür.
- **Yapay zekânın beynine dokunulmaz:** Tool, ipucu, aday liste, puanlama yok.
- **Sohbet:** AI oyuncularla muhabbet eder; eşiyle çaktırmadan, üstü kapalı iş
  birliği yapmaya çalışabilir. Oyuna dair bilgi sızdırmak yasak.

## Tasarım planı

Tam tasarım [`plan.md`](./plan.md) dosyasında:

- 101 Okey referans kuralları
- Mimari katmanlar (motor = hakem)
- AI serbest-muhakeme modeli
- Bilgi görünürlüğü
- Hamle protokolü ve ret döngüsü
- Sohbet & eş iletişimi
- Sağlayıcı ve hesap modeli (BYOK)
- Yol haritası ve açık kararlar

## Hesap / sağlayıcı modeli

Uygulama **lokal** çalışır ve **BYOK (Kendi API Anahtarını Getir)** modelini
kullanır:

| Sağlayıcı | Uyumluluk | Not |
|---|---|---|
| DeepSeek | OpenAI-uyumlu | Ucuz |
| OpenAI | Referans | Ücretli |
| Ollama | OpenAI-uyumlu | Yerel, ücretsiz, anahtarsız |
| Gemini | Adaptör | Ücretsiz katman var |
| Anthropic | Adaptör | Ücretli |

Kullanıcı kendi anahtarını girer; anahtar cihazda şifreli saklanır, sunucuya
gitmez.

## Yol haritası

1. Oyun motoru (101 Okey kuralları + hamle doğrulama)
2. Masa arayüzü
3. AI oyuncu (serbest muhakeme + hakem ret döngüsü)
4. Sohbet & eş iletişimi
5. Sağlayıcı soyutlaması
6. Paketleme (Windows + Android + Web)
