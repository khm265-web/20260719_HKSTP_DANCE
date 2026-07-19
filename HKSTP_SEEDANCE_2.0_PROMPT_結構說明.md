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
- 單次生成上限 15s
- 因每格分鏡獨立生成，**預設 1個分鏡 = 1個JOB = 1個SHOT**
- 如單一分鏡需要拆做多於一個連續SHOT（例如同一地點內鏡頭有轉折），先用多SHOT + "CUT TO"，累計時長 ≤15s
- 唔同分鏡（唔同JOB）之間唔需要動作/鏡頭連戲，各自獨立判斷

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
- 該JOB入面出現嘅每個角色，identity-lock句只喺SCENE開場寫一次（SCENE描述之前）
- 因分鏡獨立生成，**identity-lock只需喺該JOB內一致，唔需要跨JOB/跨分鏡一致**
- Identity-lock句淨係可以含「外觀」語言（臉、髮型、衫著、形狀、顏色、比例）
- 唔可以含pose、動作、狀態、位置描述——呢啲屬於SHOT動作描述，寫喺SHOT段落入面
- 格式（`FIRSTFRAME` mode下無reference image tag，直接用角色名稱）：
  `[角色名稱]: preserve exact [face/hairstyle/outfit 或 shape/color/proportions] as shown in first frame.`

CAMERA 規則
-----------
- Camera movement 同 subject movement 一定要分開句子描述
  - ✅ 正確：「舞者慢慢轉圈，鏡頭固定唔郁」
  - ❌ 錯誤：「鏡頭圍住跳舞嘅人轉」
- 避免「fast」呢類字眼：fast camera + fast cuts + busy scene 組合幾乎必然造成 jitter
- 如需快節奏，一次淨係揀一個元素加快（例如淨係加快camera，動作維持正常節奏）
- 連續一鏡到底類型：要明確講camera「唔做緊咩」，例如「no cuts, no zoom, natural head movement」

PREVENTATIVE CLAUSE / QUALITY CONSTRAINTS 規則
----------------------------------------------
Seedance 唔支援獨立Negative區塊，preventative clause要寫入正面敘述：
- 全局性（成個SHOT都要守嘅限制）→ 寫喺STYLE/LIGHTING段落
- 單一鏡頭運動性限制 → 寫入CAMERA描述句入面
  例：「TRUE SLOW PUSH-IN: camera physically drifts forward... NOT a zoom, focal length stays fixed throughout.」

Prompt Template
---------------
```
SHOT STRUCTURE: [1 shot, Xs, 16:9]

[角色名稱] [動作：現在式、單一清晰動詞、跳緊咩舞步/幅度]

[IDENTITY LOCK BLOCK — 直接用角色名稱敘述，唔用{KEY} tag（FIRSTFRAME mode下無reference image需要resolve）]

SCENE
[場地描述 — 地點、固定視覺元素、氛圍]

CAMERA
[鏡頭運動描述，與角色動作分開講，注明NOT做緊咩]

STYLE
[畫風、色調、光影、整體視覺基調]

[Quality constraints / preventative clause，正面敘述]

Audio: {AUDIO}, NO BGM
```

如單一分鏡拆做多SHOT：
```
SHOT 1 — Xs
[鏡頭/構圖/角色動作]

CUT TO

SHOT 2 — Xs
[鏡頭/構圖/角色動作]

[...累計 ≤15s...]
```

⚠️ AUDIO TAG 規則
-----------------
`{AUDIO}` 係literal template token，唔係要expand嘅變數。
Prompt文字output時必須keep住 `Audio: {AUDIO}, NO BGM` 呢個字面字串，
唔可以將佢替換做audio_descs.json入面嘅實際描述內容。
Resolve邏輯交由python script喺runtime處理，唔喺prompt生成階段做。

{KEY} Tag 規則（Asset對應）
--------------------------
1. Prompt文字入面用 `{KEY}` tag（如 `{CHAR_A}`、`{LOC_2}`）標記邊個角色/地點出現喺邊
2. `{KEY}` 要對應 assets.json 入面既key
3. JSON既 `selected` array 順序 = tag被resolve做 Image1/Image2... 既順序
4. 每次要同時輸出：
   (a) prompt文字（含`{KEY}` tag）
   (b) 對應嘅 `selected` array（順序要match tag出現次序）
5. 內部代號（`{KEY}`）只會出現喺呢個中介tag系統，唔會直接寫死做"Image1"呢類字眼
   （`make_jobs()` 負責將 `{KEY}` 轉做實際image順序）
6. **`FIRSTFRAME` mode下 `selected` 留空，prompt入面唔會有 `{KEY}` 需要resolve做Image N**（因為冇reference image，只有`first_frame_key`指定嘅第一幀），IDENTITY LOCK句直接寫角色名稱敘述，唔使用`{KEY}` tag

JOB JSON 結構說明
-----------------
每個JOB對應一個 .json（settings）+ 一個 .txt（prompt文字）：

```json
{
    "name": "JOB編號_SC場次_SHOT範圍",
    "active": false,
    "selected": [],
    "mode": "FIRSTFRAME",
    "first_frame_key": "SB_XX",
    "ratio": "16:9",
    "duration": 9,
    "resolution": "480p",
    "seed": 2026,
    "audio": "",
    "content_filter": false
}
```

| 欄位 | 說明 |
|---|---|
| `name` | Job識別名，通常對應分鏡編號 |
| `active` | `true`先執行；`false`跳過 |
| `selected` | **`FIRSTFRAME` mode時留空 `[]`**（唔再用嚟帶reference image，避免同first frame混淆） |
| `mode` | **`FIRSTFRAME`**（HKSTP預設，分鏡圖做第一幀） |
| `first_frame_key` | **`FIRSTFRAME` mode專用**，填實際 asset key（例：`"SB_XX"`），指定邊張分鏡圖做第一幀 |
| `ratio` | **必須手動填 `"16:9"`**（script fallback係9:16） |
| `duration` | 秒數，=呢個JOB所有SHOT時長總和，上限15s |
| `resolution` | draft用低解析度，定稿先加高 |
| `seed` | 固定seed；retry邏輯會用random覆蓋 |
| `audio` | 對應audio_descs.json嘅key，空字串=無獨立audio |
| `content_filter` | `false`=放寬審查（+10%費用） |
| `negative_prompt` | **禁用**，唔可以加呢個key |

- `FIRSTFRAME` mode下 `selected` 留空，`first_frame_key` 指定嗰張分鏡圖 asset key
- `selected` array數量/順序要同prompt .txt入面`{KEY}`出現次序match，錯位會導致tag對錯圖
- `duration`一定要等於實際SHOT時長總和，唔可以隨便填
- `active`控制單一JOB是否被pipeline執行，方便批量開關而唔洗刪JSON

⚠️ MODEL LIMITATION
-------------------
Seedance 2.0 / Seedance 2.0 Mini 不支援 negative prompt。
"Negative:" 區塊寫落prompt文字都唔會被模型當negative指令處理。
如需避免特定元素，改用：
- 正面描述鎖定想要嘅結果（例如唔想zoom，就明確寫死鏡頭運動方式）
- Preventative clause寫入正面敘述句入面，而唔係獨立Negative區塊
