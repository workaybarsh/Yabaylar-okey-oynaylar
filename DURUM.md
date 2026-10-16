# Durum ve Devam Notu

> Yeni bir sohbet/oturum başlarken hızlı bağlam için. Ayrıntılı tasarım: [`plan.md`](plan.md),
> [`README.md`](README.md).

## Proje
**101 Okey** oynanabilen ve aynı masada **yapay zekâ rakiplerle sohbet** edilebilen oyun.
Hedef platformlar: Windows + Android + Web (henüz düşünce/plan aşaması).
Repo: `github.com/workaybarsh/Yabaylar-okey-oynaylar`

## Tamamlananlar
- **Tam tasarım planı** (`plan.md`): 101 Okey kuralları, mimari katmanlar (motor = **hakem**),
  AI "serbest muhakeme + son denetim" modeli, bilgi görünürlüğü, hamle protokolü, ret döngüsü,
  sohbet/eş iletişimi, BYOK sağlayıcı modeli.

## Önemli kararlar (özet)
- **Oyun:** 101 Okey — yere açma **≥101 puan** ya da **5 çift**; tüm kural detayları harfiyen
- **Motor = hakem:** hamle önermez; **sadece sonuçta** doğrular, "şundan ötürü mümkün değil" der
- **AI serbest:** kötü seçim/hata/olmayanı deneme serbest; **tool/ipucu/aday liste YOK**
- **Yapay kişilik YOK:** her model (Gemini/DeepSeek/GPT/Claude) kendi doğal üslubuyla konuşur
- **Ret sınırı YOK** + **10 sn hız uyarısı** (yalnızca o tura özel)
- **Sağlayıcı:** BYOK — DeepSeek/OpenAI/Ollama/Gemini/Claude

## Sıradaki adım
1. **Oyun motoru** (arayüzsüz, test edilebilir): 106 taş, gösterge/okey, per doğrulama, 101 açma,
   sıra akışı — *her şeyin bel kemiği*
2. Masa arayüzü (sürükle-bırak)
3. AI oyuncu (serbest muhakeme + hakem ret döngüsü)
4. Sohbet & eş iletişimi
5. Sağlayıcı soyutlaması
6. Paketleme (Windows + Android + Web)

## Açık kararlar (plan.md)
- [ ] Platform/teknoloji (öneri: Web-first PWA)
- [ ] Kaç rakip, hangi modeller
- [ ] Çok oyunculu (gerçek kişiler) sonraya mı
