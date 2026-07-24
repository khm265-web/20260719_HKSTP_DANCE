HKSTP 廣告片 — SEEDANCE 2.0 PROMPT 結構說明
================================

專案背景
--------
HKSTP 廣告片，90秒，Seedance 2.0（BytePlus API）生成。
內容：一班不同嘅人喺唔同地方跳現代舞。
Storyboard 每格分鏡圖已完成，係**實際正確畫面**（唔係草稿參考），
用嚟做每條短片嘅 FIRSTFRAME 去生成。
每格分鏡**獨立生成**，唔需要跨鏡頭角色一致性。

JOB 與 SHOT 關係
----------------
- 1 JOB = 1條 Seedance 生成影片
- ⚠️官方硬限制：單次生成 duration 必須 4-15秒（唔係得「上限15s」，4秒以下會被API拒絕）
- 因每格分鏡獨立生成，**預設 1個分鏡 = 1個JOB = 1個SHOT**
- 如單一分鏡需要拆做多於一個連續SHOT（例如同一地點內鏡頭有轉折），先用多SHOT + "CUT TO"，累計時長喺4-15s之間
- 排JOB時計算：最尾SHOT剩餘時長唔夠4s就搬去下一個JOB或合併，唔好硬塞
- 唔同分鏡（唔同JOB）之間唔需要動作/鏡頭連戲，各自獨立判斷
- ⚠️官方提示：唔好喺prompt文字入面幫個別SHOT寫死精確秒數（例如"0-3秒"），
  timing支援唔穩定，強制限時可能令生成異常；交俾model自然分配pacing

Prompt 結構總覽（跟研究結果：Subject/Action 行先，Style 放後定調）
----------------------------------------------------------
Seedance 2.0 對開頭 20-30 字權重最高，模型會用開頭鎖定主體同核心動作，
所以 **Subject + Action 要行先**，Style/Lighting 留到後段先寫。
Camera movement 必須同 subject movement 分開描述，混淆係最常見出錯原因。

Prompt 元素順序：
1. SHOT 結構總覽（shot 數量、總時長、畫面比例）
2. SUBJECT + ACTION（邊個跳緊咩動作，一個清晰動詞、現在式、單一動作）
3. IDENTITY LOCK（該JOB用到嘅角色外觀鎖定，SCENE開場寫一次）
4. ENVIRONMENT（場地描述）
5. CAMERA（鏡頭運動，同角色動作分開講）
6. STYLE / LIGHTING（畫風、色調、視覺基調——呢度先寫，唔放最前）
7. Quality Constraints / Preventative clause（正面敘述嘅guardrails）

IDENTITY-LOCK 規則
------------------
- 該JOB入面出現嘅每個 {CHAR_X}，identity-lock句只喺SCENE開場寫一次（SCENE描述之前）
- 因分鏡獨立生成，**identity-lock只需喺該JOB內一致，唔需要跨JOB/跨分鏡一致**
- Identity-lock句淨係可以含「外觀」語言（臉、髮型、衫著、形狀、顏色、比例）
- 唔可以含pose、動作、狀態、位置描述——呢啲屬於SHOT動作描述，寫喺SHOT段落入面
- 格式：
  `{KEY}: preserve exact [face/hairstyle/outfit 或 shape/color/proportions] from reference.`

CAMERA 規則
-----------
- Camera movement 同 subject movement 一定要分開句子描述
  - ✅ 正確：「舞者慢慢轉圈，鏡頭固定唔郁」
  - ❌ 錯誤：「鏡頭圍住跳舞嘅人轉」
- 避免「fast」呢類字眼：fast camera + fast cuts + busy scene 組合幾乎必然造成 jitter
- 如需快節奏，一次淨係揀一個元素加快（例如淨係加快camera，動作維持正常節奏）
- 連續一鏡到底類型：要明確講camera「唔做緊咩」，例如「no cuts, no zoom, natural head movement」

⚠️ 官方符號規範（特殊符號標記資訊類型）
--------------------------------------
官方建議用符號區分唔同資訊類型，幫模型準確理解：
- 音樂：（）  例：（fast-paced rock music playing in background）
- 音效：<>  例：< dog barking in the distance >
- 對白：{}  例：{Hello, world}
- 字幕：【】 例：【Chapter One: Departure】
⚠️ `{KEY}` 資產tag（`{CHAR_A}`等）同官方`{}`對白符號唔會衝突：
   `make_jobs()` 送prompt去API之前，已經將`{CHAR_A}`呢類`{KEY}`換成"Image N"文字，
   送到model嗰刻prompt入面根本冇晒`{CHAR_A}`呢啲字串，兩者運作階段完全分開，唔會互相污染。

PREVENTATIVE CLAUSE / QUALITY CONSTRAINTS 規則
----------------------------------------------
✅ 官方確認：Seedance支援完整句negative寫法（"do not generate X" / "X remains stable, no Y"），
   唔係"No X, No Y"逗號堆疊（逗號堆疊只係坊間慣例，官方示範全部係完整句）。
分層寫法：
- 全局性（成個SHOT都要守嘅限制）→ 寫喺STYLE/LIGHTING段落（正面敘述）
- 單一鏡頭運動性限制 → 寫入CAMERA描述句入面（正面敘述）
  例：「TRUE SLOW PUSH-IN: camera physically drifts forward... NOT a zoom, focal length stays fixed throughout.」
- 畫面雜訊/meta元素性（deformation、flicker、subtitle、logo、watermark、twin等，同主體/鏡頭控制無關）
  → 寫入片尾獨立CONSTRAINT句（完整句negative）
  官方原文示範用語：
  - 防字幕："keep it subtitle-free" / "avoid generating any text or subtitles"
  - 防logo："do not generate a logo"
  - 防浮水印："do not generate a watermark"
  - 防變形/閃爍："face remains stable without deformation; movements natural and smooth, no stutter or flicker"
  ⚠️ 呢類constraint都唔可以100%杜絕（官方明講subtitle/twin/watermark都只係降低機率），
     殘留時配合：landscape生成後再crop、reference圖先去水印/文字、換seed重生

Prompt Template
---------------
```
SHOT STRUCTURE: [1 shot, Xs, 16:9]

{CHAR_X} [動作：現在式、單一清晰動詞、跳緊咩舞步/幅度]

[IDENTITY LOCK BLOCK — 呢個JOB用到嘅所有{CHAR_X}，各一句]

SCENE
[場地描述 — 地點、固定視覺元素、氛圍]

CAMERA
[鏡頭運動描述，與角色動作分開講，注明NOT做緊咩]

STYLE
[畫風、色調、光影、整體視覺基調]

[Quality constraints / preventative clause，正面敘述]
[(音樂描述，如有)] [<音效描述，如有>]

[CONSTRAINT 句 — 片尾獨立完整句negative，防字幕/logo/watermark/變形，見上方規則]
```

如單一分鏡拆做多SHOT：
```
SHOT 1 — Xs
[鏡頭/構圖/角色動作]
[(音樂)] [<音效>]

CUT TO

SHOT 2 — Xs
[鏡頭/構圖/角色動作]
[(音樂)] [<音效>]

[...累計4-15s...]
```

⚠️ AUDIO 寫法（改跟官方SHOT-level做法）
-----------------
官方建議audio資訊跟individual SHOT寫，唔係成個JOB尾一句概括：
- 每個SHOT自己尾段用`()`標音樂、`<>`標音效，唔再用`Audio: {AUDIO}, NO BGM`呢種JOB尾總括寫法
- `()`/`<>`只可以描述**嗰個SHOT動作描述句已經提過**嘅動作/物件發出嘅聲
  （例：SHOT講緊角色行樓梯，先可以`<footsteps on stairs>`；SHOT冇提過嘅嘢唔可以喺audio先第一次出現）
  ——呢個係防視覺bleed嘅核心防線，audio純粹幫已寫好嘅畫面配聲，唔做敘事補完
- 環境音（風、人群murmur呢類冇具體物件嘅background texture）風險較低，可以照寫
- 如SHOT-level寫法都出現unrelated視覺內容，退後策略：淨保留純環境音texture，有具體物件聲響嘅一律拎走交返後製

{KEY} Tag 規則（Asset對應）
--------------------------
1. Prompt文字入面用 `{KEY}` tag（如 `{CHAR_A}`、`{LOC_2}`）標記邊個角色/地點出現喺邊
2. `{KEY}` 要對應 assets.json 入面既key
3. JSON只有一個 `selected` array，image/video/audio唔分開三個array，
   由script讀URL副檔名自動判斷media type：
   - `.mp4`/`.mov`/`.webm` → 判做video，順序resolve做 Video 1/2/3
   - `.wav`/`.mp3`/`.m4a`/`.flac` → 判做audio，順序resolve做 Audio 1/2/3
   - 其他（`.jpg`/`.png`等） → 判做image，順序resolve做 Image 1/2...
   三種type各自獨立由0開始計數，唔會爭用同一個編號
4. 每次要同時輸出：
   (a) prompt文字（含`{KEY}` tag）
   (b) 對應嘅 `selected` array（順序要match tag出現次序）
5. 內部代號（`{KEY}`）只會出現喺呢個中介tag系統，唔會直接寫死做"Image1"呢類字眼
   （`make_jobs()` 負責將 `{KEY}` 轉做"Image N"/"Video N"/"Audio N"文字，靠URL副檔名判斷）
6. 額外獨立reference：JSON可加`"ref_videos": ["URL", ...]`，放唔喺assets.json入面嘅獨立video URL
   （唔經`{KEY}` tag resolve，直接加落content array，唔會出現喺prompt文字入面）

REFERENCE 數量上限（官方準確數字）
------------------
- image：1-9張（R2V multimodal reference），每張<30MB
- video：最多3條，單條2-15秒，全部reference video總長度加埋唔可以超過15秒（`.mp4`/`.mov`）
- audio：最多3條，單條2-15秒，全部reference audio總長度加埋唔可以超過15秒（`.wav`/`.mp3`）
- 三種type獨立計數，唔會爭用同一個上限
- 官方建議每JOB用4-5個asset就夠，唔好用盡上限（太多asset模型難判優先次序，易style衝突/subject模糊）

⚠️ Character ID Drift（換臉/漂移）對策
------------------------------------------
症狀：生成角色中途「換臉」、變到似名人而被審查封鎖。
根因：face reference唔夠強（混合圖太雜、face佔比太細）。
對策：
- 除全身圖外，額外準備乾淨headshot（淨頭部、最好無表情、去背景/肩頸干擾）
- 越需要精準reference嘅asset，喺prompt/`selected`越前置（官方："place important assets first"）
- ⚠️唔好用multi-view/三視圖做角色reference——模型易當成多個唔同subject，反而加劇drift
- 角色數量建議唔好超過4人，超過穩定性明顯下降（易人數錯亂、重複角色）

JOB JSON 結構說明
-----------------
每個JOB對應一個 .json（settings）+ 一個 .txt（prompt文字）：

```json
{
    "name": "JOB編號_SC場次_SHOT範圍",
    "active": false,
    "selected": ["CHAR_A", "LOC_2", "SB_XX"],
    "ref_videos": [],
    "mode": "FIRSTFRAME",
    "ratio": "16:9",
    "duration": 9,
    "resolution": "480p",
    "seed": 2026,
    "content_filter": false
}
```

| 欄位 | 說明 |
|---|---|
| `name` | Job識別名，通常對應分鏡編號 |
| `active` | `true`先執行；`false`跳過 |
| `selected` | Asset key（image/video/audio可夾埋），順序=tag被resolve做Image/Video/Audio N嘅順序 |
| `ref_videos` | 額外獨立video reference URL（唔喺assets.json），唔經`{KEY}` tag resolve、唔出現喺prompt文字 |
| `mode` | **`FIRSTFRAME`**（HKSTP預設，分鏡圖做第一幀） |
| `ratio` | **必須手動填 `"16:9"`**（script fallback係9:16） |
| `duration` | 秒數，=呢個JOB所有SHOT時長總和，官方硬限制4-15秒（唔止上限，仲有下限） |
| `resolution` | draft用低解析度，定稿先加高 |
| `seed` | 固定seed；retry邏輯會用random覆蓋 |
| `content_filter` | `false`=放寬審查（+10%費用） |
| `negative_prompt` | **JSON冇呢個欄位**（唔存在呢個API參數），constraint要寫喺.txt prompt文字最尾（完整句negative） |

- `selected` array數量/順序要同prompt .txt入面`{KEY}`出現次序match，錯位會導致tag對錯圖
- `duration`一定要等於實際SHOT時長總和，唔可以隨便填，記得check係咪喺4-15秒之間
- `active`控制單一JOB是否被pipeline執行，方便批量開關而唔洗刪JSON

⚠️ MODEL LIMITATION（更新：對照官方文件）
-------------------------------------
- ⚠️官方明文：Seedance 2.0系列**唔支援直接上傳含真人臉孔嘅reference image/video**
  （本項目用原創動畫角色，正常唔會撞到，但如reference圖源頭混入真人相要留意）
- JOB JSON冇獨立`negative_prompt`欄位/API參數，唔好喺JSON度加（呢個係真限制）
- 但prompt文字入面**係可以用negative句式**——官方明確示範用
  "do not generate X" / "keep it X-free" / "no stutter or flicker" 呢類完整句
  （之前假設「prompt完全唔可以用negative」係錯嘅，已修正，見上方CONSTRAINT句規則）
- 寫法用完整句子，唔係"No X, No Y"逗號堆疊（官方示範全部完整句）
- ⚠️ constraint唔保證100%生效，殘留時配合landscape再crop、reference先去文字、換seed重生
- ⚠️ duration官方硬限制4-15秒（唔止「上限」，仲有下限）
