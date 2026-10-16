# Oyun Motoru — Genel Tasarım (101 Okey · arayüzsüz · **hakem**)

> **Durum:** Tasarım / planlama. Henüz kod yok. **Kural kararları (Karar 1–5) kilitlendi → §15.**
> Bu doküman, [`plan.md`](plan.md) §2 (kurallar), §3 (katmanlar), §6 (hamle protokolü) ve
> §7 (ret döngüsü) bölümlerini **motor düzeyinde** somutlaştırır.
>
> **Dil/teknoloji kararı ertelendi.** Aşağıdaki modül sınırları ve API dilden bağımsızdır;
> tip örnekleri okunabilirlik için TS benzeri yazılmıştır, bağlayıcı değildir.

İlgili: [`DURUM.md`](DURUM.md) · [`BASLA.md`](BASLA.md) · [`README.md`](README.md)

---

## 1. Amaç, kapsam, ilkeler

Motor, oyunun **tek doğruluk kaynağıdır** ve **hakemdir**: hamle önermez, yardım etmez,
puan/pozisyon değerlendirmez. Bir hamle geldiğinde yalnızca *"kurala uygun mu?"* sorusuna
cevap verir; uygunsa uygular, değilse **nedenini** döner.

**Tasarım ilkeleri**

| İlke | Anlamı |
|---|---|
| **Arayüzsüz (headless)** | UI, ağ, dosya, saat, rastgelelik kaynağı yok. Saf girdi→çıktı. |
| **Hakem, öneri yok** | `legalMoves()` / ipucu / aday liste **public API'de yoktur**. Yalnızca `validate` (evet/hayır + gerekçe). |
| **Deterministik** | Tüm rastgelelik **enjekte edilebilir, tohumlu (seeded) RNG** ile. Aynı tohum → aynı oyun. |
| **Değişmez (immutable) durum** | Her geçiş yeni bir durum üretir: `(state, move) → state`. Eski durum bozulmaz (geri alma, AI replaysi, test için). |
| **Serileştirilebilir** | Durum tam olarak JSON'a yazılıp okunabilir (kaydet/yükle, AI promptu, fuzz). |
| **Hata ≠ çökme** | Kural ihlali bir **ret**tir: `{ ok:false, reason:<kod>, message:<insan dili> }`. İstisna (exception) fırlatmaz. |
| **Görünürlük ayrı** | Motor tüm durumu tutar; katmanlar **oyuncuya özel görünüm** (`view`) alır. Kapalı el sızmaz. |
| **Kural yapılandırılabilir** | Bölgede değişen noktalar `RuleSet` ile açık/kapalı yönetilir (§15). "Harfiyen" = bu varsayılanların kilitlenmesi. |

**Kapsam (bu aşama):** 106 taş, gösterge/okey/sahte-okey, per doğrulama, 101 puan açma,
5 çift açma, sıra akışı (çek + işle + at), bitiş, puan/ceza, görünürlük, serileştirme, testler.
**Kapsam dışı:** Arayüz, AI oyuncu, sohbet, sağlayıcı (bunlar motora *biner*, motora *girmez*).

---

## 2. Taş modeli ve kimlik

Deste **106 taş** = 4 renk × 13 sayı × 2 kopya (=104) **+ 2 "sahte okey" taşı**.

```
type Color  = "kirmizi" | "siyah" | "mavi" | "sari";
type Number = 1|2|3|4|5|6|7|8|9|10|11|12|13;
type TileKind = "normal" | "sahte_okey";   // "sahte okey": üzerinde sembol olan 2 taş

interface Tile {
  id: number;          // 0..105 — benzersiz fiziksel kimlik (motorda esas olan bu)
  kind: TileKind;
  color?: Color;       // yalnızca kind="normal"
  number?: Number;     // yalnızca kind="normal"
  copy?: 0 | 1;        // aynı renk+sayının 2 kopyasından hangisi
}
```

- **Kimlik motorda `id`'dir**, okunabilir `renk-sayı` yalnızca AI/UI için bir *gösterim*dir
  (`kirmizi-9`, `sari-3`, `mavi-13`; joker için `okey`). İki kopya aynı gösterime sahiptir;
  motor hangi fiziksel taşın kastedildiğini ayrıca çözer (§11).
- **Gösterge (işaret taşı)** dağıtımda açılan taştır ve deste dışında, ortada durur.
- **Deste**, kapalı çekme yığınıdır; sırası korunur (deterministik replay için).
- **Yer (discard)**: her oyuncunun attığı taşlar; **son atılan** alınabilir.
- **Masa (melds)**: açılmış perler; sahipleri/kim tarafından açıldığı izlenir.

**Değişmez (invariant):** 106 taş her zaman tam olarak bir yerde: deste ∪ gösterge ∪ eller ∪
yer ∪ masa. Test bunu her hamleden sonra doğrular (§13).

---

## 3. Gösterge / Okey / Sahte Okey türetimi

> Referans: pagat.com *Okey 101*, TR Vikipedi *Okey 101*, Altınstar 101 kuralları.
> Bu bölüm plan.md §2 "Kural detayları"nı somutlar.

```
interface OkeyInfo {
  indicator: Tile;                     // açılan gösterge taşı
  color: Color;                        // okey rengi (gösterge ile aynı)
  number: Number;                      // okey sayısı = gösterge sayısı + 1, 13'ten sonra 1'e döner
}
```

- **Okey (joker) = göstergenin bir üstü, aynı renk.** `13 → 1` sarması geçerli
  (gösterge `mavi-13` ise okey `mavi-1`). *Not:* bu, **seride 13→1 bağlantısından ayrıdır**
  (seride sarma yok — §4).
- **Gerçek okey taşları:** gösterge koordinatına **eşit olan 2 fiziksel taş** (`color`+`number`).
  Bunlar **herhangi bir taşın yerine geçen joker**dir (evrensel joker).
- **Sahte okey (riziko) — KARAR 1 (kilitli):** üzerinde sembol olan **2 taş**. Gösterge hangi
  taşa denk gelirse gelsin, bu taşlar **okey taşının (gösterge+1) rengini ve sayısını alır**;
  yani **okey koordinatını temsil eder**. **Evrensel joker DEĞİLDİR**; yalnızca okeyin kendisi
  yerine geçer (`RuleSet.sahteOkey = "okeyYerine"`).
  → Plan.md'deki *"Sahte okey (göstergenin eşi) davranışı uygulanır"* maddesinin motor
  karşılığı budur (bkz. §15 Karar 1).

```
function deriveOkey(indicator: Tile): OkeyInfo          // 13→1 sarması burada
function isRealOkey(t: Tile, info: OkeyInfo): boolean   // t.color==info.color && t.number==info.number
function isFalseOkey(t: Tile): boolean                  // kind=="sahte_okey"
function isWild(t: Tile, info: OkeyInfo): boolean       // yalnızca GERÇEK okey joker sayılır (KARAR 1)
function effectiveOkeyCoord(t: Tile, info): {color?, number}   // per doğrulamada kullanılan "etkin" koordinat
```

**Uç durum — KARAR 2 (kilitli):** Gösterge olarak **sahte okey** çıkarsa (renksiz taş) okey
doğrudan hesaplanamaz; **göstergenin bir altındaki taş** gösterge sayılır. O taş da sahte
okey ise **zincirleme olarak bir alt taşa** inilir (numaralı taşa ulaşana kadar).
`deriveOkey` bunu `RuleSet.indicatorIfFalseOkey = "altTasZincir"` ile uygular.

---

## 4. Per (grup) modeli ve doğrulama

Motor, bir grup taşı alır ve **"geçerli per mi?"** sorusuna + **puan değerine** cevap verir.
Per türleri:

### 4.1 Set (grup / aynı sayı)
- **Boy:** 3 veya 4.
- **Kural:** aynı sayı, **farklı renkler**. Aynı renk+sayı iki kez olamaz
  (örn. `mavi-10 + kirmizi-10 + mavi-10` → **geçersiz**).
- **Değer:** `sayı × boy` (joker, temsil ettiği sayıyla sayılır).

### 4.2 Seri (el / aynı renk ardışık)
- **Boy:** ≥3.
- **Kural:** tek renk, **ardışık** sayılar, 1..13 aralığında.
- **Sarma yok:** 101 Okey'de **1 yalnızca en küçüktür**; `12-13-1` **geçersiz**
  (`RuleSet.runsAllowWrap = false`).
- **Değer:** seriyi dolduran sayıların toplamı; ör. `7-8-9 = 24`, joker bir boşluğu
  dolduruyorsa o sayı da toplama girer (seri değeri jokerlerin *nerede* olduğundan bağımsızdır).

### 4.3 Çift (pair)
- **Boy:** tam 2.
- **Kural:** **birebir aynı** taş (aynı renk + aynı sayı) — ör. iki `siyah-9`.
  Farklı renk aynı sayı çift **değildir**. Joker yanına gelirse çift kurabilir
  (`RuleSet.jokerPairs`), iki joker birlikte de çift sayılabilir (config).
- **Değer:** `2 × sayı` (yalnızca el ceza puanlamasında; **açma eşiğine sayılmaz**).

### 4.4 Joker (joker yerine geçme)
- Gerçek okey **istenen her taşın** yerine geçer.
- Değer = temsil ettiği taşın değeri (açma toplamında ve elde kalan puanda).
- **Tek çözüm ≠ kabul:** bir grup, jokerlerin yerleştirilmesiyle **geçerli ve tutarlı** bir
  set/seri/çift oluşturuyorsa kabul edilir. Doğrulayıcı, "jokerleri en iyi şekilde yerleştir"
  problemidir (seri: boşluk/kenar doldurma; set: eksik renkleri tamamlama).

```
type GroupKind = "set" | "run" | "pair";
interface GroupValidation {
  ok: boolean;
  kind?: GroupKind;
  value?: number;            // per değeri (jokerler etkin değeriyle)
  reason?: string;           // geçersizse gerekçe (kural kodu)
  assignments?: Array<{tileId:number, as?:{color:Color, number:Number}}>; // jokerlerin neyi temsil ettiği
}
function validateGroup(tiles: Tile[], info: OkeyInfo, rules: RuleSet): GroupValidation
```

**Doğrulama algoritması (özet):**

```
Seri:
  renk = joker olmayan taşların tek ortak rengi (yoksa reddet)
  sayılar = joker olmayan taşların sayıları; hepsi aynı renk olmalı
  |grup| >= 3; sayılar 1..13; sarma yok
  W = joker sayısı
  → sayılar ∪ (W joker) tek bir ardışık blok (uzunluk ≥3) oluşturacak şekilde
    yerleştirilebiliyor mu? (klasik "fill-the-gaps" kontrolü)
  → value = bloğun sayılarının toplamı

Set:
  sayı = joker olmayan taşların tek ortak sayısı
  renkler = joker olmayan farklı renkler; tekrar eden renk varsa reddet
  W = joker sayısı; boy = |grup| ∈ {3,4}
  → farklı renk sayısı + W >= boy ve (renkler ∪ jokerler) farklı renkleri doldurabiliyor mu?
  → value = sayı × boy

Çift:
  |grup| == 2 ve (birebir aynı) veya (joker + herhangi) [config]
  → value = 2 × sayı
```

---

## 5. Açma (101 puan **veya** 5 çift)

> plan.md §2 "Açma (kritik kural)".

**Açma = oyuncunun ilk kez yere per koyması.** Koşul (tek turda, elden):

1. **Set/seri modu:** açılan perlerin **toplam değeri ≥ 101**, **VEYA**
2. **Çift modu:** en az **5 çift**.

Ek kurallar:

- **Mod karışmaz.** Set/seri ile açan bir daha **çift** açamaz; çift ile açan bir daha
  **yeni set/seri** açamaz. (Ancak çiftçi, başkalarının masadaki set/serilerine **işleyebilir**.)
- **Açmadan işleme olmaz.** Oyuncu açmadan masadaki pere ekleme yapamaz.
- **Yerden alınan taş zorunlu kullanılır.** Henüz açmamışken yerden alınan taş, o turun
  **açma perinde** kullanılmak zorundadır (kullanılmazsa geri bırakılır + desteden çekilir;
  ceza yok). Açmışsa da aldığı taşı kullanmak zorundadır.
- **Açma, bitirme hakkı doğurur** ama bitmek için yine de **son taş atılmalıdır**
  (açıp eli bitirmek, elde taş kalmamak demek değildir — bir *bitiş taşı* gerekir).
- **Artırmalı açma YOK (KARAR 3, kilitli):** bir sonraki açan da **101'den** başlar; önceki
  açanın değeri eşiği yükseltmez (`RuleSet.escalatingOpen = false`).
- **Sayı tutmadan elden bitme:** 21 taşı tek turda, kimse açmadan açıp bitiren oyuncu, açma
  eşiğini tutmasa da bitebilir (bkz. §8 "elden").

---

## 6. Sıra akışı ve hamle seti

**Dağıtım:** 106 taş karılır; her oyuncuya 21, **dağıtanın sağındaki** oyuncuya **22** taş.
Bir taş gösterge olarak açılır. **Oyun saat yönünün tersine** oynanır ve **22 taşlı oyuncu
(dağıtanın sağı) çekmeden atarak başlar.** Taş sayısı/sırası `RuleSet` ile yapılandırılır
(motor varsayılanı; §15).

**Normal tur = 1 çek/topla + (isteğe bağlı per işlemleri) + 1 at:**

| # | Adım | Zorunlu |
|---|---|---|
| 1 | Kapalı **desteden çek** *veya* yerdeki **son taşı al** | ✅ (ilk oyuncu hariç) |
| 2 | **Per aç / işle / okeyi yerinden al / son taşı geri al** | ➖ isteğe bağlı |
| 3 | Eline bir taş **at** (ya da `bitir` ile son taşı at) | ✅ (her turda, açsa bile) |

**Motor hamle türleri** (plan.md §6 protokolünün motordaki karşılığı):

| Motor hamlesi | Protokol `tip` | Not |
|---|---|---|
| `DRAW_FROM_DECK` | `desteden_cek` | Kapalı desteden çek |
| `TAKE_DISCARD` | `yerden_al` | `tas`; son atılanı al |
| `MELD` | `per_ac` | `perler`; açma veya sonraki yeni per(ler) |
| `LAY_OFF` | `per_isle` | `hedef`, `tas`/`perler`; masadaki pere ekle |
| `SWAP_OKEY` | `okey_degistir` | `hedef`, `tas`; masadaki okeyi gerçek taşla değiştir |
| `DISCARD` | `at` | `tas`; elinden at |
| `FINISH` | `bitir` | `tas`; son taşı atıp eli bitir (yalnızca açmışken) |
| `RETURN_TAKEN_DISCARD` | *(yardımcı)* | Yerden alınanı geri bırak + desteden çek (ceza yok) |
| `RETURN_LAST_LAID` | *(yardımcı)* | Yalnızca **son** konan taşı geri al (§8 ceza kuralı) |

> **Not:** `bitir` ile "elinde son 1 taş varken `at`" aynı olaydır; motor ikisini de kabul eder,
> `bitir` son taşı atma niyetini açıkça belirtir.

---

## 7. Aşama (phase) makinesi

```
                 ┌──────────────┐
   dağıtım  ───► │ DEAL_READY   │  (22 taşlı oyuncu atacak, çekmez)
                 └──────┬───────┘
                        │ DISCARD/FINISH(ilk) veya devam
                        ▼
        ┌──────────────────────────────────────┐
        │ AWAIT_DRAW(player)                    │◄──────────────┐
        │  • DRAW_FROM_DECK | TAKE_DISCARD      │               │
        └───────────────┬──────────────────────┘               │
                        │ çekildi/alındı                        │
                        ▼                                      │
        ┌──────────────────────────────────────┐               │
        │ AWAIT_ACT(player)                     │               │
        │  • MELD | LAY_OFF | SWAP_OKEY         │               │
        │  • RETURN_TAKEN_DISCARD | RETURN_LAST │               │
        │  • DISCARD | FINISH   → tur biter     ├───────────────┘
        └───────────────┬──────────────────────┘   (sonraki oyuncu)
                        │ el bitti / deste bitti / 4 çift
                        ▼
                 ┌──────────────┐
                 │ DEAL_END     │  → puanlama/ceza (§8) → sonuç
                 └──────────────┘
```

```
interface TurnState {
  player: number;                 // 0..3 (C=A'nın eşi vb.)
  hasDrawn: boolean;              // ilk oyuncu false ile başlar
  takenDiscard: number | null;    // alınan taşın id'si (kullanılmak zorunda)
  laidThisTurn: number[];         // bu tur konan/masaya işlenen taşlar (geri al/ceza için)
  openedThisTurn: boolean;
}

type Phase =
  | { kind: "DEAL_READY" }
  | { kind: "AWAIT_DRAW"; turn: TurnState }
  | { kind: "AWAIT_ACT";  turn: TurnState }
  | { kind: "DEAL_END";   result: DealResult };
```

**Olay (event) günlüğü:** her uygulanan hamle, UI/AI için `MoveEvent[]` olarak saklanır
(kim, ne, sonuç, varsa ceza kodu). Bu aynı zamanda sohbet/AI promptunun *nesnel* temelidir.

---

## 8. Bitiş ve puanlama / ceza

### 8.1 Tur bitiş koşulları
1. **Bir oyuncu bitirir** (açtıktan sonra son taşını atar) → puanlama.
2. **Deste biter** ve sıradaki oyuncu ne çekebilir ne de yerdekini kullanmak ister → tur
   puansız biter; **elde kalan her joker için 101 ceza**. (Bitirebiliyorsa göstergeyi çekip
   bitirebilir — `RuleSet.allowIndicatorDraw`.)
3. **Dört oyuncu da çift açarsa** → tur iptal, yeniden dağıtılır, ceza yok.
4. **Eş/takım modu (KARAR 5)** → puanlar **takım toplamı** üzerinden birleştirilir (bkz. §8.5).

### 8.2 Puanlama matrisi (tur sonu, tek el)
- Biten, set/seri açmış, **joker atarak bitmemişse**: **biten −101**; açanlar elindeki taşların
  toplamı; **çift açanlar ×2**; **açmayanlar 202**.
- Biten **joker atarak** bittiyse: her şey **×2** → biten **−202**, açanlar ×2, çiftçiler ×4,
  açmayanlar **404**.
- Biten **çift açmış** ise: biten **−202**; açanlar ×2; çiftçiler ×4; açmayanlar 404.
- Biten **çift + joker**: biten **−404**; açanlar ×4; çiftçiler ×8; açmayanlar 404.
- **Elden bitme** (kimse açmadan 21 taşı tek turda açıp bitirme): biten **−202**, diğerleri 404
  (jokerle bitirdiyse **−404 / 808**).
- **Elde kalan joker**: her biri **+101** (destenin bitmesiyle puansız biten turda da).

> Kesin matris, tablo halinde `scoring.ts` içinde **veriden** (data-driven) tutulur; testte
> kaynak örnekleriyle birebir karşılaştırılır.

### 8.3 101 ceza puanı gerektiren durumlar
- Masaya **atılan taş, masadaki bir pere eklenebiliyorsa** (bitirirken son taş hariç).
- Oyuncu açmaya kalkıp **101'i (ya da 5 çifti) tutturamayıp** taşlarını geri alıyorsa.
- Masaya **birden fazla taş** koyup **geri alıyorsa** (yalnızca **son** taş cezasız geri alınır).
- **Joker atılırsa**.
- (Eşli mod) yerden alınan taşla açma durumunda ceza **karşı takıma** yazılır (§8.5).

### 8.4 Ret ≠ ceza
Kural/fizik ihlali (`olmayan taş`, `geçersiz per`, `101 altı açma`, `sırasız hamle`,
`kullanılmayan yerden-alınan taş`) → **ret** (§1, plan.md §7).
**Kötü ama geçerli, hatta cezalı hamle reddedilmez** → uygulanır, ceza kaydedilir.

### 8.5 Eş / takım puanlaması (KARAR 5, kilitli)
- Oyuncular **2 takım**: A–C ve B–D (karşılıklı oturanlar eş).
- Her elin cezaları **eşler arasında birleştirilir**; turlar sonunda **toplamı düşük olan takım** kazanır.
- **Varsayılan (`teamScoreMode = "birlesik"`):** eşlerden biri eli bitirirse **diğer eşin cezası
  silinir**; aksi halde iki eşin el cezaları takım hanesine toplanır.
- `"bireyselSonraTopla"` modu (önce bireysel ceza, sonra takıma topla) `RuleSet` ile seçilebilir.

---

## 9. Görünürlük ve görünüm (view)

Motor tüm durumu tutar; `viewFor(state, player)` yalnızca izin verileni döner:

| Bilgi | Oyuncuya görünür |
|---|---|
| Kendi eli | ✅ |
| **Diğerlerinin eli** | ❌ (yalnızca taş **sayısı**) |
| Gösterge, okey bilgisi | ✅ |
| Masa (açılmış perler) | ✅ |
| Yerdeki atılan taşlar | ✅ |
| Destede kalan **sayı** | ✅ (içeriği ❌) |
| Sıra / kim oynuyor | ✅ |
| Tur ceza kayıtları | ✅ |

```
function viewFor(state: GameState, player: number): PlayerView
```

UI **ve** AI aynı `PlayerView`'ı kullanır → kapalı elin sızması yapısal olarak imkânsız.

---

## 10. Determinizm, RNG, serileştirme

- **RNG:** `interface Rng { next(): number; shuffle<T>(a: T[]): T[] }`. Varsayılan tohumlu
  PRNG (ör. xorshift/mulberry32). Testler sabit tohumla; üretimde UI tohumu üretir.
- **Replay:** aynı `seed` + aynı hamle dizisi → birebir aynı durum (golden test).
- **Serileştirme:** `toJSON(state)` → tüm alanlar (deste sırası dahil); `fromJSON` geri yükler.
- **Sürümleme:** snapshot'a `schemaVersion`; kural değişince eski kayıtlar reddedilir/taşınır.

---

## 11. Genel API (dilden bağımsız)

```
// Oluşturma
createGame(config: GameConfig, rng?: Rng): GameState
//   config: { players, teams?, ruleSet, dealer, seed? }

// Hakem çekirdeği
validateMove(state, player, move): MoveValidation
//   → { ok:true } | { ok:false, reason:string, message:string }
applyMove(state, player, move): ApplyResult
//   → { ok:true, state: GameState, events: MoveEvent[] }
//   | { ok:false, reason, message }        // ret; state DEĞİŞMEZ

viewFor(state, player): PlayerView
toJSON(state) / fromJSON(json): GameState

// Hamle ayrıştırma (plan.md §6 JSON bloğu)
parseMove(json): { ok:true, move } | { ok:false, reason }   // bozuk JSON = ret
```

> **Bilinçli olarak YOK:** `legalMoves()`, `bestMove()`, `hint()`, pozisyon puanı.
> Doğrulama **yalnızca** hamle geldikten sonra çalışır (hakem ilkesi).
> Testlerin ihtiyacı olan **hamle üretecı `test/` altında** yaşar ve **AI yoluna asla bağlanmaz**.

**Örnek hamle (protokol JSON):**
```json
{ "tip": "per_ac", "perler": [["kirmizi-5","mavi-5","siyah-5"],["kirmizi-7","kirmizi-8","kirmizi-9"]],
  "sohbet": "bu el iyi geldi" }
```
**Ret örneği (motorun sesi):**
```
{ ok:false, reason:"OPEN_BELOW_THRESHOLD",
  message:"Açma için en az 101 puan gerekiyor; grupların 74. (ya da 5 çift)" }
```

---

## 12. Modül yapısı (öneri — dilden bağımsız sınırlar)

Dil kararı verilince (plan.md §11) dosya adları uyarlanır; **sorumluluk sınırları aynı kalır**:

```
engine/
  tiles      → Tile, Color, Number, kimlik, gösterim ↔ id çözümü
  deck       → 106 taş kur, karıştır, dağıt, gösterge seç
  okey       → deriveOkey, isRealOkey/isFalseOkey/isWild, etkin koordinat
  meld       → validateGroup (set/seri/çift), joker yerleştirme, per değeri
  opening    → 101 / 5 çift değerlendirme, mod kilidi
  moves      → hamle tipleri + parse/serialize (protokol JSON)
  turn       → aşama makinesi, çek/al/işle/at/bitir, sıra devri
  scoring    → tur sonu matrisi + 101 cezaları (data-driven)
  state      → GameState, snapshot, viewFor (görünürlük)
  rng        → tohumlu PRNG
  errors     → reason kodları + insan dili mesajlar
  engine     → createGame/validate/apply (ince orkestrasyon)
  index      → public yüzey
test/
  deck.spec, okey.spec, meld.spec, opening.spec,
  turn.spec, scoring.spec, view.spec, replay.spec,
  invariants.spec (fuzz: 106 taş korunumu, determinizm)
```

---

## 13. Test stratejisi

| Katman | Ne test edilir |
|---|---|
| **Deste** | 106 taş, renk×sayı×2 + 2 sahte okey; dağıtım 21/21/21/22; gösterge ayrı |
| **Gösterge/Okey** | `13→1` sarması; kendi renginde +1; sahte okey eşlemesi; sahte okey göstergesi |
| **Per** | set (3/4, farklı renk, tekrar reddi), seri (≥3, tek renk, ardışık, **sarma yok**), çift (birebir), joker doldurma, per değeri |
| **Açma** | 101 eşiği (kaynak örnekleri: 102, tam 101, 100 red), 5 çift, mod kilidi, açmadan işleme reddi, yerden-alınanın zorunlu kullanımı |
| **Sıra** | ilk oyuncu çekmeden atar; çek **veya** al (ikisi birden değil); her turda at zorunlu; sırasız hamle reddi |
| **Bitiş/Ceza** | puanlama matrisi (tüm ×2 katlamaları), elde joker 101, atılan-işler taş cezası, son-taş istisnası, tek-taş geri alma |
| **Görünürlük** | `viewFor` başka eli **asla** sızdırmaz |
| **Replay/Fuzz** | sabit tohum → birebir tekrar; rastgele oyunlarda **106 taş korunumu** ve durum tutarlılığı |

Kural kaynaklarından alınan **örnek eller** (örn. `1-1-1, 10-11-12, 8-9-10, 7-8-9, 5-5-5 = 102`)
**golden test** olarak sabitlenir.

---

## 14. AI protokolü ile eşleme (özet — motor değişmez)

- AI **serbest metin** + sonda `<hamle>{JSON}</hamle>` üretir (plan.md §6). Motor yalnızca
  **bloğu** okur (`parseMove`).
- **Boş/bozuk JSON = ret** → AI'dan düzeltmesi istenir (plan.md §7).
- Motorun ret `message`'ı **AI'nın bir sonraki turunda gördüğü geri bildirimdir** (hakemin sesi).
  **Ret sınırı yok**; 10 sn'lik hız uyarısı **tura özel** olarak AI katmanında eklenir —
  **motor bunu bilmez**.
- Motor **hamle önermez**; AI'nın kötü/cezalı ama geçerli hamlesi uygulanır (plan.md §7).

---

## 15. Kararlaştırılan kurallar (`RuleSet`)

Bölgesel/uygulama farkları tek yerde toplanır: `RuleSet`. Aşağıdaki **1–5 karara bağlandı**
(proje sahibinin onayı, 16 Ekim 2026); motor bunları **kilitli** uygular. Kaynak varsayılanı
olan diğerleri (ilk oyuncu, deste bitişi vb.) motor varsayılanıdır ve gerekirse yalnızca
`RuleSet` değeri değiştirilerek uyarlanır — kod değişmez.

```
interface RuleSet {
  // ── Kilitli kararlar ──────────────────────────────────────────────
  openThreshold: 101,
  pairsToOpen: 5,
  runsAllowWrap: false,             // 101: 1 yalnızca en küçük (13→1 seride yok)
  sahteOkey: "okeyYerine",          // KARAR 1
  indicatorIfFalseOkey: "altTasZincir",  // KARAR 2
  escalatingOpen: false,            // KARAR 3
  allowOkeyReclaim: true,           // KARAR 4
  teams: "es",                      // KARAR 5
  teamScoreMode: "birlesik",        // KARAR 5 (bkz. §8.5)

  // ── Motor varsayılanları (kaynak temelli) ─────────────────────────
  jokerPairs: true,                 // çiftte joker yanına gelirse çift sayılır
  firstPlayer: "22li",              // dağıtanın sağı, 22 taş, çekmeden atar
  allowIndicatorDraw: "bitirebilirse", // deste bitince, bitirebiliyorsa göstergeyi al
  deckExhausted: "puansizJoker101", // puansız bitiş + eldeki her joker 101
  allPairsVoid: true,               // 4 oyuncu da çift açarsa tur iptal
  dealCount: null,                  // 3/5/7/9/11 — arayüz seçer
}
```

**Karar günlüğü:**

| # | Soru | **Karar** |
|---|---|---|
| 1 | Sahte okey semantiği | **Okey yerine geçer** — evrensel joker **değildir**; yalnızca okey koordinatını (gösterge+1) temsil eder |
| 2 | Gösterge sahte okey çıkarsa | **Bir alt taş** gösterge olur; o da sahte okey ise **zincirleme** bir alt taşa inilir |
| 3 | Artırmalı açma | **Yok** — her açan 101'den başlar |
| 4 | Okeyi yerinden alma | **Var** — masadaki okeyi gerçek taşı koyup ele al (açmış olma şartıyla) |
| 5 | Eş / takım puanları | **Birleştirilir** — puanlar takım toplamı üzerinden |

**Kalan, varsayılan uygulanacak noktalar:** ilk oyuncu & taş sayısı (22'li), deste bitince
göstergeyi çekip bitirme, deste bitişi puanlaması, tur sayısı (`dealCount`) — hepsi
kaynak varsayılanıyla yukarıdaki `RuleSet`'te tanımlı. İtiraz olursa yalnızca değer değişir.

---

## 16. Kaynaklar

- **pagat.com — Okey 101** (Erol Karataş / John McLeod):
  <https://www.pagat.com/rummy/okey101.html> — per/açma/puanlama/ceza referansı
- **TR Vikipedi — Okey 101**: <https://tr.wikipedia.org/wiki/Okey_101>
- **Altınstar — Okey 101 kuralları**: <https://www.altinstar.com/okey-nasil-oynanir/okey-101-kurallar>
- **ciftokey.com — 34/51/101 kuralları**:
  <https://www.ciftokey.com/kurallar.htm> — işleme, okeyi yerinden alma, çift/ceza
- **Zynga — Sahte Okey Nedir?**:
  <https://zyngasupport.zendesk.com/hc/tr/articles/115003539172-Sahte-Okey-Nedir>
- **LITE Games — Okey Rules** (false jokers):
  <https://info.lite.games/en/support/solutions/articles/60000688857-okey-rules>
- **VIP Okey — Okey/Sahte Okey nasıl belirlenir** (gösterge+1):
  <https://casualino-jsc.helpshift.com/hc/tr/29-vip-okey/faq/2780>

> Bazı detaylar bölgeden bölgeye ve uygulamadan uygulamaya değişir. Bu projede **Karar 1–5
> kilitlendi** (§15 Karar Günlüğü); kalan noktalar `RuleSet` motor varsayılanı olarak taşınır.
