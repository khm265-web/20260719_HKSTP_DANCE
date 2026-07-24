SEEDANCE 2.0 官方規則 — 通用版
================================
（任何project都可以直接餵呢份，內容係官方平台規則/prompt句式，唔涉及specific project嘅美術風格或pipeline架構）

JOB 與 SHOT 關係
----------------
- 1 JOB = 1條 Seedance 生成影片
- ⚠️官方硬限制：單次生成 duration 必須 4-15秒（唔係得「上限15s」，4秒以下會被API拒絕）
- 1個JOB可含1~N個SHOT，SHOT時長總和要喺4-15s之間
- 排JOB時計算：最尾SHOT剩餘時長唔夠4s就搬去下一個JOB或合併，唔好硬塞
- 多於1個SHOT時，SHOT之間用 "CUT TO" 分隔
- SHOT有對白就直接寫入該SHOT段落
- ⚠️官方提示：唔好喺prompt文字入面幫個別SHOT寫死精確秒數（例如"0-3秒"），
  timing支援唔穩定，強制限時可能令生成異常；交俾model自然分配pacing

IDENTITY-LOCK 規則
------------------
- 每個出現喺SCENE入面嘅角色/物件，identity-lock句只喺SCENE開場寫一次（SCENE描述之前），
  唔喺個別SHOT內重複
- Identity-lock句淨係可以含「外觀」語言（臉、髮型、衫著、形狀、顏色、比例）
- 唔可以含pose、動作、狀態、位置描述（例如 "in ready-to-launch position"、
  "unaltered by pose or angle" 之後接位置字眼）
  呢啲屬於SHOT動作描述，寫喺SHOT段落入面
- 官方原本推薦句式：
  - 基本定義：Define [2-3個穩定靜態特徵，如衫著/髮型/外觀] in <Image/Video_N> as <Subject_N>
  - 簡單場景（未預先定義）：每次提到就用 <Subject_N>@<Image_N> 標記binding關係，
    例：Zhang San@Image 1
  - 預先定義場景：全片統一用返同一個名稱（例：定義咗做「警察」之後全程用「警察」，唔好中途轉稱呼）
  - 官方提醒：核心特徵揀2-3個就夠、要避免同一subject有矛盾特徵描述

PREVENTATIVE CLAUSE 分層規則
----------------------------
Preventative clause分三層寫入：
- 全局性（整個SCENE都要守嘅限制，例如唔要warm light、唔要色彩飽和度突變）
  → 寫喺 STYLE BLOCK（正面敘述）
- 單一SHOT性（淨係嗰個SHOT要守嘅限制，例如唔要zoom）
  → 寫入該SHOT嘅鏡頭描述句入面（正面敘述）
  例：「TRUE SLOW PUSH-IN: camera physically drifts forward... NOT a zoom, focal length stays fixed throughout.」
- 畫面雜訊/meta元素性（deformation、flicker、subtitle、watermark等，同主體/鏡頭控制無關）
  → 寫入片尾 CONSTRAINT 句（完整句negative，見下方詳細規則）

⚠️ 官方核心原則（來自官方prompt guide）
--------------------------------------------
1. Seedance = multimodal AI director，內部拆「spatial layer（畫面有乜）」+
   「temporal layer（時間點變化）」兩維度理解。好prompt係engineering指令，
   唔係copywriting：who + where + doing what + how camera moves + 時序。
2. 官方advanced formula（元素排序）：
   precise subject → action details → scene/environment → lighting & color tone
   → camera movement → visual style → image quality → constraints
   （即係主體/動作行先，style/quality/constraint殿後）
3. Shot sequencing用「Shot 1 / Shot 2 / Shot 3」timeline storyboard。
   ⚠️ 官方明講：唔好硬性限制每個shot嘅精確秒數（例如0-3秒），
      timing支援唔穩定，強制限時可能令生成異常。
4. 動作描述：具體到身體部位（手/腳/頭/肩）+ 幅度/速度/力度量化。
   官方建議優先slow/gentle/continuous小動作，避免sprint/big jump/violent roll呢類
   high-burst大動作（穩定性差）。
5. 單一SHOT只寫1種camera movement，唔好同時要push+pull+pan+move（會抖）。
6. 情緒要外化做具體動作（唔好寫"very sad"，要寫"lowering head, shoulders trembling,
   eyes reddening"），官方有情緒→動作對照表可參考。

⚠️ 官方符號規範（特殊符號標記資訊類型）
--------------------------------------
官方建議用符號區分唔同資訊類型，幫模型準確理解：
- 音樂：（）  例：（fast-paced rock music playing in background）
- 音效：<>  例：< dog barking in the distance >
- 對白：{}  例：{Hello, world}  （非中英文要標語言，如 says in Japanese {こんにちは}）
- 字幕：【】 例：【Chapter One: Departure】
註：（）<>音效符號跟SHOT-level寫法（見下方「AUDIO 文字描述」章節）；
   {}對白同【】字幕跟返SHOT走。

⚠️ Character ID drift（換臉/漂移）官方對策
------------------------------------------
症狀：生成角色中途「換臉」、變到似名人而被審查封鎖。
根因：face reference唔夠強（混合圖太雜、face佔比太細）。
對策：
- 除全身圖外，額外準備乾淨headshot（淨頭部、最好無表情、去背景/肩頸干擾）
- prompt講明：<Subject 1> 用 image1(headshot) 做facial reference、image2(full-body) 做styling reference
- 越需要精準reference嘅asset，喺prompt/selected越前置（官方："place important assets first"）
- ⚠️唔好用multi-view/三視圖做角色reference——模型易當成多個唔同subject，反而加劇drift
  （turnaround sheet要crop成單張frontal portrait）

⚠️ Asset配置策略（官方建議）
----------------------------
官方建議每JOB 4-5個asset就夠，唔好用盡上限（太多asset模型難判優先次序，易style衝突/subject模糊）。
四種功能角色：角色錨定(character image) + 場景定調(scene image) +
運鏡參考(camera movement video) + 節奏氛圍(audio)。
推薦組合：1-2張角色圖 + 1張場景圖 + 1條運鏡video + 1條audio。

⚠️ 其他任務類型官方句式（除Multimodal Reference外，其餘供參考）
----------------------------------------------------------------------
- Video Editing（局部/全局修改原片，冇提到嘅部分預設不變）：
  加元素：Clearly describe [Element_Features] + [Timing] + [Location]
  改元素：Strictly edit <Video_N>, and modify [Original_Characteristic] in it to [New_Characteristic]
  刪元素：講明要刪嘅嘢；想keep住嘅嘢要特別強調先保留得好
- Video Extension（沿時間軸延續原片，audio-video風格/主體/敘事要連貫）：
  延伸：Extend <Video_N> forward/backward to generate...
  多段接駁：<Video_1> + [Transition_Description] + followed by <Video_2> + ...
  ⚠️官方提醒：呢類task用"<Video_N>"直接指代，唔好講"reference <Video_N>"
  （會被誤判做reference task）
- Combined（reference一個asset + edit另一個asset）：
  Reference [Reference_Dimension] of <Image/Video_N>, strictly edit <Video_X>, [Specific_Edits]

⚠️ 官方FAQ重點（實務問題同對策）
-----------------------------------------------------
1. 對白語言要統一：dialogue入面唔好中英文夾雜（專有名詞除外）
2. 角色數量：reference真人數量建議唔好超過4人，超過穩定性明顯下降
   （易出現人數錯亂、重複角色）；多人場景建議分組生成image、再用image生成video
3. 片尾雜音：有narration嘅片段尾段容易有abrupt click/斷尾雜音，
   後製用Premiere/CapCut volume envelope做fade-out處理
4. Video extension接駁跳幀：前後兩段連接處容易有畫面跳動/回退，
   後製對策：前段尾剪走6格、後段頭剪走1格，逐個接駁點重複做
5. 反覆extension畫質衰退：多次continuation會累積畫質降級（尤其臉部易有色塊），
   對策：將原片轉做「純白3D模型」中繼版本再continuation
   （prompt參考："Convert the video into a white 3D model, no color, no texture, no shadows"）
6. 特效唔跟預期（例如倒數數字亂跳）：文字描述特效效果唔穩定，
   改用reference video定義個特效（"the way X appears should reference video 1"）
7. 中文發音錯（如有中文對白）：多音字/生僻字容易讀錯，
   可以用同音字替換（例："螭龙山"改寫做"吃龙山"）幫助發音準確——呢個純係workaround，唔保證100%
8. Reference audio聲線唔準：想準確還原某把聲，prompt要加聲音特質形容詞
   （例："Use the low, thick, warm, finely grainy middle-aged male voice of @Audio 1 to say"），
   台詞語氣風格越貼近reference audio，還原度越高

⚠️ 排序原則
-----------
Seedance 2.0 開頭20-30字權重最高，模型用嚟lock主體，
所以主體（IDENTITY LOCK）要行先，STYLE BLOCK呢類全局形容詞退後，
避免長篇style文字稀釋咗開頭對主體嘅鎖定。

⚠️ Pipeline共用（所有project同一套py/json結構）
----------------------
{KEY} Tag系統規則
- Prompt文字入面用 {KEY} tag（如 {CHAR_A}, {PROP_2}）標記邊個角色/物件出現喺邊
- {KEY} 要對應 assets.json 入面既key
- JSON只有一個 selected array，image/video/audio唔分開三個array，
  由script讀URL嘅副檔名自動判斷media type：
  - .mp4 / .mov / .webm → 判做video，順序resolve做 Video 1/2/3
  - .wav / .mp3 / .m4a / .flac → 判做audio，順序resolve做 Audio 1/2/3
  - 其他（.jpg/.png等） → 判做image，順序resolve做 Image 1/2...
  三種type各自獨立由0開始計數，唔會爭用同一個編號
- 每次要同時輸出：(a) prompt文字（含{KEY} tag） (b) 對應嘅selected array
  （順序要match tag出現次序——同一個array入面image/video/audio key可以夾埋寫，script會自動分類）
- 內部代號({KEY})只會出現喺呢個中介tag系統，唔會直接寫死做"image1"/"video1"/"audio1"呢類字眼
  （make_jobs()負責將{KEY}轉做"Image N"/"Video N"/"Audio N"文字，靠URL副檔名判斷，唔係靠assets.json嘅category）
- 額外獨立reference：JSON可加`"ref_videos": ["URL", ...]`，用嚟放唔喺assets.json入面嘅獨立video URL
  （呢啲唔會經{KEY} tag resolve，係直接喺build_content()加落content array，唔會出現喺prompt文字入面）
- ⚠️ script（Seedance_gen_BYTEPLUS.py）副檔名判斷包埋.webm/.m4a/.flac，
  呢啲喺官方文件搵唔到明文支援，如果用呢啲格式要留意會唔會分類錯或者API reject

⚠️ {KEY}資產tag（{CHAR_A}等）同官方{}對白符號唔會衝突：
   make_jobs()喺送prompt去API之前，已經將{CHAR_A}呢類{KEY}換成"Image N"文字，
   送到model嗰刻prompt入面根本冇晒{CHAR_A}呢啲字串——
   剩低嘅{}淨係你手寫嘅對白內容，兩者運作階段完全分開，唔會互相污染。

IDENTITY-LOCK句式（本pipeline實際用法）
用「{KEY}: preserve exact...」句式：
  {KEY}: preserve exact [face/hairstyle/outfit 或 shape/color/proportions] from reference.
（官方原本推薦句式見上方「IDENTITY-LOCK 規則」，供debug/對照參考）

JOB JSON 結構說明
每個JOB對應一個 .json（settings）+ 一個 .txt（prompt文字），JSON欄位如下：

{
    "name": "JOB編號_SC場次_SHOT範圍",      // 檔名/job識別，通常對應SCENE+SHOT區間
    "active": false,                        // true=呢個JOB會被執行；false=跳過
    "selected": ["CHAR_A", "PROP_A1", ...], // 呢個JOB用到嘅asset key（image/video/audio可以夾埋），順序=tag被resolve做Image N/Video N/Audio N嘅順序
    "ref_videos": [],                       // 額外獨立video reference URL（唔喺assets.json）
    "mode": "REFERENCE",                    // REFERENCE=用reference image生成；FIRSTFRAME/LASTFRAME/FIRSTLAST=用assets.json入面FIRSTFRAME/LASTFRAME key
    "ratio": "9:16",                        // 畫面比例，固定跟STYLE BLOCK
    "duration": 12,                         // 秒數，= 呢個JOB入面所有SHOT時長總和，官方硬限制4-15秒
    "resolution": "480p",                   // draft用低解析度，定稿先加高
    "seed": 2026,                           // 固定seed；retry邏輯會用random覆蓋
    "audio": "",                            // ⚠️已棄用 — 原本lookup audio_descs.json，已改用SHOT-level寫法，呢個欄位保留但唔再被script讀取
    "content_filter": false                 // 內容審查開關（false=關閉審查，billing +10%）
}

- selected array數量/順序要同prompt .txt入面{KEY}出現次序match，錯位會導致tag對錯圖
- duration一定要等於JOB拆分邏輯計出嚟嘅累計秒數，唔可以隨便填
- active控制單一JOB是否被pipeline執行，方便批量開關而唔洗刪JSON
- JOB JSON冇`negative_prompt`欄位（唔存在呢個API參數），constraint要寫喺.txt prompt文字最尾

通用PROMPT骨架
--------------
[IDENTITY LOCK BLOCK — 呢個SCENE用到嘅所有角色/物件，各一句]
[STYLE BLOCK]
[VIDEO REF BLOCK — 如selected有video reference，官方句式：
 Reference <Action/Camera_movement/Style> in <Video_N> to generate...（可省略）]

SCENE_XX
[場景描述 — 地點、固定視覺元素、氛圍，跨SHOT共用]

SHOT 1 — [Xs]
[角色動作(具體動詞、現在式)/鏡頭(單一主要camera movement)/構圖/對白(如有)/單一SHOT性preventative clause]
[(音樂描述，如有)]
[<音效描述，如有>]

CUT TO

SHOT 2 — [Xs]
...

[...累計最多15s...]

[CONSTRAINT 句 — 完整句negative寫法]

⚠️ AUDIO 文字描述（SHOT-level做法）
-----------------------
官方建議audio資訊跟individual SHOT寫（每個SHOT尾用符號標記：音樂用()、音效用<>），
唔係成個JOB尾一句概括。

寫法：
- 每個SHOT自己尾段加(音樂)/<音效>，唔好用成JOB一句總括嘅寫法
- (音樂)/<音效>只描述**嗰個SHOT入面已經有嘅嘢**發出嘅聲——
  即係嗰SHOT角色動作描述句已經提過嘅動作/物件，先可以喺audio度配返把聲
  （例：SHOT講緊角色行緊樓梯，先可以<footsteps on stairs>；
  唔可以SHOT冇提過嘅嘢，淨係喺audio度先第一次出現）
- 呢個係防返視覺bleed嘅核心防線：audio唔可以引入SHOT視覺描述冇講過嘅新物件/新動作，
  純粹幫已經寫好嘅畫面配聲，唔做「敘事補完」
- 環境音（風、人群murmur呢類冇具體物件嘅background texture）風險較低，可以照寫
- 對白繼續用{}，跟SHOT入面角色講嘢嗰句走，唔使獨立成行
- 如果SHOT-level寫法都出現unrelated視覺內容，退後策略：淨保留純環境音texture
  （風/人群/室內迴響呢類冇具體物件嘅），有具體物件聲響嘅一律拎走，交返後製處理

⚠️ CONSTRAINT 句規則（官方認可）
-----------------------
位置：成個prompt最尾（官方Example放喺全片描述最後一句）
✅ 官方確認：Seedance支援完整句negative寫法，唔使勉強改晒做正面。
   官方原文示範用語：
   - 防字幕："keep it subtitle-free" / "avoid generating any text or subtitles"
   - 防logo："do not generate a logo"
   - 防浮水印："do not generate a watermark"
   - 防變形/閃爍："face remains stable without deformation; movements natural and smooth, no stutter or flicker"
   - 防style drift："2D Japanese anime style"（用正面style word鎖返，官方建議）
   - 防twin/分身（多角色場景官方固定句）：
     "Throughout the video, characters with completely identical appearance, clothing, and accessories are prohibited.
      Do not generate duplicate avatars or a twin effect. Keep only a single corresponding character in the same frame."

寫法：用完整句子（"do not generate X" / "X remains stable, no Y"），唔係"No X, No Y"逗號堆疊。
   （逗號堆疊只係坊間慣例；官方示範全部係完整句，句式清晰模型理解更穩定）

⚠️ 但呢類constraint都唔可以100%杜絕（官方明講subtitle/twin/watermark都只係降低機率），
   如出現殘留，配合：landscape生成後再crop（字幕機率較低）、reference圖先去水印/文字、換seed重生。

分層原則：
- 角色外觀限制 → IDENTITY LOCK BLOCK（正面preserve exact）
- 單一SHOT鏡頭限制 → 該SHOT描述句
- 全局畫面/style限制 → STYLE BLOCK或片尾constraint句

JOB拆分邏輯範例
--------------
SHOT1  4s  -> JOB1
SHOT2  6s  -> JOB1（累計10s）
SHOT3  7s  -> 超出15s，移至JOB2
⚠️ 如果最尾剩返嘅SHOT單獨攞出嚟少過4秒（例如淨剩2s），
   唔可以自己開一個JOB（低於官方4秒下限會被API拒絕）——
   要同前一個JOB合併，或者調整返個SHOT時長至≥4s

REFERENCE 數量上限（官方準確數字）
------------------
- image：1-9張（R2V multimodal reference），每張<30MB
- video：最多3條，單條時長2-15秒，全部reference video總長度加埋唔可以超過15秒
  格式限定：.mp4 / .mov 先係官方文件列明支援（.webm未見官方文件確認）
- audio：最多3條，單條時長2-15秒，全部reference audio總長度加埋唔可以超過15秒
  格式限定：.wav / .mp3 先係官方文件列明支援（.m4a/.flac未見官方文件確認，用之前建議實測）
- 三種type獨立計數，唔會爭用同一個上限

⚠️ MODEL LIMITATION / CONSTRAINT（官方文件對照）
-------------------------------------
- ⚠️官方明文：Seedance 2.0系列**唔支援直接上傳含真人臉孔嘅reference image/video**
  （如reference圖源頭混入真人相/真人動作capture片，要留意呢個限制；
  官方有出「Create with ease」文件講變通方法，需要時再查）
- JOB JSON冇獨立`negative_prompt`欄位/API參數，唔好喺JSON度加（呢個係真限制）
- 但prompt文字入面係可以用negative句式嘅——官方明確示範用
  "do not generate X" / "keep it X-free" / "no stutter or flicker" 呢類完整句
  （「prompt完全唔可以用negative」呢個講法係錯，Seedance跟GPT Image 2唔同）
- 寫法用完整句子，唔係"No X, No Y"逗號堆疊（官方示範全部完整句）
- ⚠️ constraint唔保證100%生效（官方明講subtitle/twin/watermark都只係降低機率），
  殘留時配合landscape再crop、reference先去文字、換seed重生
- ⚠️ duration官方硬限制4-15秒（唔止「上限」，仲有下限）
- ⚠️ 單一JOB reference角色人數官方建議唔好超過4人，
  超過4人穩定性下降、易出現人數錯亂/重複角色
