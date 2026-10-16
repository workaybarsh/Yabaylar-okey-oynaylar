# Oyun Motoru — Genel Tasarım (101 Okey · arayüzsüz · **hakem**)

> **Durum:** Tasarım / planlama. Henüz kod yok.
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
- **Sahte okey (riziko):** üzerinde sembol olan **2 taş**. Gösterge hangi taşa denk gelirse
  gelsin, bu taşlar **okey taşının (gösterge+1) rengini ve sayısını alır**; yani **okey
  koordinatını temsil eder**. Joker değildir; yalnızca okeyin kendisi yerine geçer.
  → Plan.md'deki *"Sahte okey (göstergenin eşi) davranışı uygulanır"* maddesinin motor
  karşılığı budur; **tam davranış `RuleSet.sahteOkey` ile yapılandırılır** (§15, Açık Soru 1).

```
function deriveOkey(indicator: Tile): OkeyInfo          // 13→1 sarması burada
function isRealOkey(t: Tile, info: OkeyInfo): boolean   // t.color==info.color && t.number==info.number
function isFalseOkey(t: Tile): boolean                  // kind=="sahte_okey"
function isWild(t: Tile, info: OkeyInfo): boolean       // perde joker gibi kullanılabilir mi? (real okey; sahte okey'e göre config)
function effectiveOkeyCoord(t: Tile, info): {color?, number}   // per doğrulamada kullanılan "etkin" koordinat
```

**Uç durum:** Gösterge olarak **sahte okey** çıkarsa (renksiz taş) okey doğrudan hesaplanamaz;
bölge kuralı gereği *göstergenin altındaki taş* gösterge sayılır. Bu, `deriveOkey` içinde
`RuleSet.indicatorIfFalseOkey` ile ele alınır (varsayılan: bir alt taşı gösterge yap).

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
- **Artırmalı varyant (opsiyonel):** bir sonraki açan, önceki en yüksek açma değerinden
  **fazla** açmalı (`RuleSet.escalatingOpen`, varsayılan **kapalı**; §15, Açık Soru 3).
- **Sayı tutmadan elden bitme:** 21 taşı tek turda, kimse açmadan açıp bitiren oyuncu, açma
  eşiğini tutmasa da bitebilir (bkz. §8 "elden").

---

## 6. Sıra akışı ve hamle seti

**Dağıtım:** 106 taş karılır; her oyuncuya 21, **dağıtanın sağındaki** oyuncuya **22** taş.
Bir taş gösterge olarak açılır. **Oyun saat yönünün tersine** oynanır ve **22 taşlı oyuncu
(dağıtanın sağı) çekmeden atarak başlar.** Dağıtan/yürüyen farkı `RuleSet` ile
yapılandırılır (§15, Açık Soru 4).

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
   puansız biter; **elde kalan her joker için 101 ceza**. (Göstergeyi alıp bitirme §15,
   Açık Soru 6.)
3. **Dört oyuncu da çift açarsa** → tur iptal, yeniden dağıtılır, ceza yok.
4. *(opsiyonel)* **Eş/takım** modunda takım toplamı ile karar (§15, Açık Soru 8).

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
- (Eşli varyant) yerden çekilen taşla açma durumunda karşı takıma yazılan ceza — §15.

### 8.4 Ret ≠ ceza
Kural/fizik ihlali (`olmayan taş`, `geçersiz per`, `101 altı açma`, `sırasız hamle`,
`kullanılmayan yerden-alınan taş`) → **ret** (§1, plan.md §7).
**Kötü ama geçerli, hatta cezalı hamle reddedilmez** → uygulanır, ceza kaydedilir.

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

## 15. Kural yapılandırması ve açık sorular

Bölgesel/uygulama farkları tek yerde toplanır: `RuleSet`. **"Harfiyen"** hedefi = aşağıdaki
varsayılanların dondurulması; kararlar netleşince bu tablo kilitlenir.

```
interface RuleSet {
  openThreshold: number;        // varsayılan 101
  pairsToOpen: number;          // varsayılan 5
  runsAllowWrap: boolean;       // varsayılan false (101: 1 yalnızca en küçük)
  sahteOkey: "okeyYerine" | "joker" | "kilitli";   // Açık Soru 1
  jokerPairs: boolean;          // çiftte joker kullanımı
  escalatingOpen: boolean;      // artırmalı 101 (varsayılan false) — Açık Soru 3
  firstPlayer: "22li" | "dagitAn";                 // Açık Soru 4
  falseOkeyAsIndicator: "altTas" | "reddet";       // Açık Soru 2
  allowOkeyReclaim: boolean;    // okeyi yerinden alma (plan.md: true; pagat: false) — Açık Soru 5
  allowIndicatorDraw: "bitirebilirse" | "hayir" | "herZaman"; // Açık Soru 6
  deckExhausted: "puansiz" | "kalanlar";           // Açık Soru 7
  allPairsVoid: boolean;        // 4 oyuncu çift açarsa tur iptal (varsayılan true)
  teams: "yok" | "es";          // §8, takım puan toplama — Açık Soru 8
}
```

**Açık sorular (karar bekliyor):**

1. **Sahte okey semantiği.** *Joker gibi her taşın yerine mi geçer, yalnızca okey koordinatı
   mı?* Kaynaklar çelişiyor: pagat/LITE *"joker değildir, yalnızca okey yerine geçer"* der;
   Zynga *"okeyin sayı değerini ve rengini alır"* der. **Öneri:** varsayılan `"okeyYerine"`
   (okey koordinatını temsil eder), gerekirse `"joker"` ile genişlet.
2. **Gösterge sahte okey çıkarsa:** bir alt taş mı gösterge olur, red mi? *(Bölge turnuvası:
   alt taş.)*
3. **Artırmalı açma:** sonraki açan öncekinden fazla mı açmalı? (plan.md'de yok → varsayılan kapalı.)
4. **İlk oyuncu & taş sayısı:** 22 taşlı (dağıtanın sağı) başlar (TR Vikipedi/pagat) mi, dağıtan mı?
5. **Okeyi yerinden alma:** 101'de serbest mi? (plan.md ve ciftokey: **evet**; pagat: hayır.)
   → varsayılan **evet** (plan.md'ye sadık).
6. **Göstergeyi çekip bitirme:** deste bitince gösterge alınabilir mi?
7. **Deste bitişi puanlaması:** puansız + eldeki joker 101 mi, yoksa normal ceza mı?
8. **Eş/takım:** plan.md 2 takım diyor; **puanlar takım toplamı** üzerinden mi birleşir?
   (Yoksa bireysel skor + takım kazananı mı?) — `RuleSet.teams` bunu taşır.
9. **Çok oyunculu & tur sayısı** (plan.md §11) — motor `dealCount` ile parametrik olacak.

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

> Bazı kurallar bölgeden bölgeye ve uygulamadan uygulamaya değişir; çelişkiler §15'te
> **açık karar** olarak işaretlendi. Motor bu farkları `RuleSet` ile taşır.
