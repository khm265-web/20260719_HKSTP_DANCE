SEEDANCE 2.0 官方規則 — 魔芋API版
================================
（基於「SEEDANCE_2_0_官方規則_通用版.md」改寫，適用於魔芋AI(moyu.info)中轉平台 +
 Seedance_gen_MOYU.py。Prompt engineering／model行為規則與原版一致（不受API供應商影響），
 僅JSON結構／API限制／asset處理三部份因應魔芋平台實際規則調整，改動處已標註⚠️【魔芋版差異】）

JOB 與 SHOT 關係
----------------
- 1 JOB = 1條 Seedance 生成影片
- ⚠️官方硬限制：單次生成 duration 必須為4-15秒（並非僅「上限15s」，4秒以下會被API拒絕）
  ⚠️【魔芋版差異】duration亦支援填 -1 = 智能指定，由模型自主選擇合適時長
  （實際時長須從查詢API回應的 `data.data.duration` 讀取；與計費相關，本pipeline暫不使用此選項）
- 1個JOB可含1~N個SHOT，SHOT時長總和須介於4-15s之間
- 排JOB時需計算：最後一個SHOT剩餘時長不足4s，應移至下一個JOB或合併，不可強行塞入
- 多於1個SHOT時，SHOT之間以 "CUT TO" 分隔
- SHOT若有對白，直接寫入該SHOT段落
- ⚠️官方提示：不應在prompt文字內為個別SHOT寫入精確秒數（例如"0-3秒"），
  timing支援不穩定，強制限時可能導致生成異常；應交由model自然分配pacing

IDENTITY-LOCK 規則
------------------
（此章節與供應商無關，內容同通用版一致，不重複列出——見官方原本推薦句式／
 「{KEY}: preserve exact...」句式，詳見通用版文件）

PREVENTATIVE CLAUSE 分層規則 / 官方核心原則 / 符號規範 / Character ID drift對策 /
Asset配置策略 / 其他任務類型句式 / 官方FAQ重點 / 排序原則
--------------------------------------------------------------------------------
（以上章節純屬model prompt engineering層面，同API供應商轉換無關，內容與通用版完全一致，
 請直接參照通用版文件，此處不重複）

⚠️ Pipeline共用（魔芋API版 py/json結構）
----------------------
{KEY} Tag系統規則
- Prompt文字內以 {KEY} tag（如 {CHAR_A}, {PROP_2}）標記角色／物件出現位置
- {KEY} 須對應 assets.json 內的key
- JSON只有一個 selected array，image/video/audio不分開三個array，
  由script讀取URL副檔名自動判斷media type：
  - .mp4 / .mov → 判定為video，依序resolve為 Video 1/2/3
    ⚠️【魔芋版差異】官方明文只支援 .mp4 / .mov，.webm官方未確認支援
    （原BytePlus原生版script容許.webm，魔芋API未見文件承認，建議勿用）
  - .wav / .mp3 → 判定為audio，依序resolve為 Audio 1/2/3
    ⚠️【魔芋版差異】官方明文只支援 .wav / .mp3，.m4a/.flac官方未確認支援
  - 其他（.jpg/.png/.webp/.bmp/.tiff/.gif） → 判定為image，依序resolve為 Image 1/2...
  三種type各自獨立由0開始計數，不會共用同一編號
- 每次須同時輸出：(a) prompt文字（含{KEY} tag） (b) 對應的selected array
  （順序須與tag出現次序相符——同一array內image/video/audio key可混合寫入，script會自動分類）
- 內部代號({KEY})僅出現於此中介tag系統，不會直接寫死為"image1"/"video1"/"audio1"等字眼
- 額外獨立reference：JSON可加`"ref_videos": ["URL", ...]`，用於放置不在assets.json內的獨立video URL
- ⚠️【魔芋版差異】role名稱（reference_image/reference_video/reference_audio/first_frame/
  last_frame）為魔芋官方文件明文定義嘅正式field名，並非script自訂慣例
  （原通用版註明「script自訂，未必為官方標準field名」——魔芋版已由官方文件確認為標準值，此點更正）

⚠️ {KEY}資產tag與官方{}對白符號不會衝突（同通用版，無差異）

JOB JSON 結構說明（魔芋API版）
每個JOB對應一個 .json（settings）+ 一個 .txt（prompt文字），JSON欄位如下：

{
    "name": "JOB編號_SC場次_SHOT範圍",      // 檔名／job識別，通常對應SCENE+SHOT區間
                                             // ⚠️【魔芋版差異】create_task()會將此值填入頂層payload的"prompt"欄位
                                             //   （魔芋平台強制要求頂層prompt非空，但實際內容仍由metadata.content的text item提供）
    "active": false,                        // true=此JOB會被執行；false=跳過
    "selected": ["CHAR_A", "PROP_A1", ...], // 此JOB使用的asset key（image/video/audio可混合），順序=tag被resolve為Image N/Video N/Audio N的順序
    "ref_videos": [],                       // 額外獨立video reference URL（不在assets.json內）
    "mode": "REFERENCE",                    // REFERENCE=以selected reference image生成；FIRSTFRAME=僅first_frame；LASTFRAME=僅last_frame；FIRSTLAST=first+last frame同時使用
    "first_frame": "",                      // mode=FIRSTFRAME/FIRSTLAST時必填，填asset key（非URL）——script會以ASSETS[key]查找URL，⚠️不可放入selected array
    "last_frame": "",                       // mode=LASTFRAME/FIRSTLAST時必填，填asset key（非URL），邏輯同上，⚠️不可放入selected array
    "ratio": "9:16",                        // 畫面比例，固定依STYLE BLOCK
                                             // ⚠️【魔芋版差異】官方新增"adaptive"選項（自動判斷最合適比例），本pipeline維持固定填值不使用adaptive
    "duration": 12,                         // 秒數，= 此JOB內所有SHOT時長總和，官方硬限制4-15秒（或填-1智能指定，見上）
    "resolution": "480p",                   // draft用低解析度，定稿再提高
                                             // ⚠️【已更正 2026-08】原文寫「魔芋API只支援480p/720p」為錯誤。
                                             //   魔芋OpenAPI spec的description只列出「480p / 720p」，但該欄位屬
                                             //   metadata透傳(passthrough)欄位，實際由上游Seedance 2.0模型判斷，
                                             //   上游支援：480p / 720p / 1080p / 4k（Fast/Mini版不支援1080p以上，
                                             //   Mini傳1080p會被降級為720p並按720p計費）。
                                             //   → 魔芋可生成4K，spec文件只是列舉不完整，並非平台限制。
                                             //   ⚠️ 計費隨解析度大幅上升（4k約為720p的5-6倍token），draft務必用480p
    "audio": ""                             // ⚠️已棄用 — 原本lookup audio_descs.json，已改用SHOT-level寫法，此欄位保留但script不再讀取
}
// ⚠️【魔芋版差異】以下欄位已從JSON結構移除（Seedance_gen_BYTEPLUS.py才有，魔芋API不支援）：
//   - "seed"：⚠️魔芋官方metadata參數表無此欄位，無法固定seed／無法reproduce結果，
//     JOB JSON若填此key亦無效，Seedance_gen_MOYU.py不會讀取
//   - "content_filter"：⚠️魔芋官方metadata參數表無此欄位，JOB JSON若填此key亦無效
// generate_audio並非job JSON欄位——create_task()寫死generate_audio=True於metadata內送往API，
//   job.json加此key亦無效，不會被讀取（此點與BytePlus原生版相同，只係位置由payload頂層改為metadata內）

- selected array數量／順序須與prompt .txt內{KEY}出現次序相符，錯位會導致tag對錯圖
- ⚠️ first_frame/last_frame獨立於selected array之外，填ASSET KEY（非URL，與selected array內的key格式相同），script會以ASSETS[key]自動查找URL，不經{KEY} tag／prompt文字resolve
- ⚠️ LASTFRAME mode（僅last_frame、無first_frame）script有實作，但魔芋官方文件僅列出「首帧」「首尾帧」「多模態參考」3種互斥場景，未見「last_frame單獨」此種——使用此mode前建議先實測確認API是否接受
- reference asset送API時的role值：`selected`內的image/video/audio分別使用`reference_image`/`reference_video`/`reference_audio`——⚠️【魔芋版差異】此為魔芋官方文件明文確認嘅標準field名（非script自訂）
- duration須等於JOB拆分邏輯計算出的累計秒數，不可隨意填寫
- active控制單一JOB是否被pipeline執行，方便批量開關而無須刪除JSON
- JOB JSON無`negative_prompt`欄位（不存在此API參數），constraint須寫在.txt prompt文字最後

通用PROMPT骨架 / AUDIO文字描述(SHOT-level) / CONSTRAINT句規則
--------------------------------------------------------------
（此三章節純屬model prompt engineering層面，同API供應商轉換無關，內容與通用版完全一致，
 請直接參照通用版文件，此處不重複）

JOB拆分邏輯範例
--------------
SHOT1  4s  -> JOB1
SHOT2  6s  -> JOB1（累計10s）
SHOT3  7s  -> 超出15s，移至JOB2
⚠️ 若最後剩餘的SHOT單獨取出少於4秒（例如僅剩2s），
   不可自行開設一個JOB（低於官方4秒下限會被API拒絕）——
   須與前一個JOB合併，或調整該SHOT時長至≥4s

REFERENCE 數量上限（魔芋官方文件確認數字）
------------------
- image：1-9張（多模態參考生視頻），每張<30MB，request body總大小不超過64MB
  ⚠️ 官方另列：圖生視頻-首帧僅1張、首尾帧固定2張、多模態參考1~9張——三種場景互斥不可混用
  尺寸限制：寬高比(寬/高)須介於0.4-2.5，寬高長度須介於300-6000px
- video：最多3條，單條時長2-15秒，全部reference video總長度合計不可超過15秒
  格式限定：僅.mp4 / .mov（魔芋官方文件已確認，.webm不支援）
  另有：reference video本身須為480p/720p（此為「輸入」reference video限制，與「輸出」resolution無關）、
  寬高比0.4-2.5、畫面像素409600-927408、單條≤50MB、FPS 24-60
- audio：最多3條，單條時長2-15秒，全部reference audio總長度合計不可超過15秒
  格式限定：僅.wav / .mp3（魔芋官方文件已確認，.m4a/.flac不支援）
  單條≤15MB
- 三種type獨立計數，不會共用同一上限

⚠️ MODEL LIMITATION / CONSTRAINT（魔芋API版對照）
-------------------------------------
- ⚠️【魔芋版差異】真人臉孔reference處理方式與BytePlus原生版不同：
  魔芋平台**要求先將真人／擬真人臉部素材上傳至平台「素材庫」**，
  以`asset://<ASSET_ID>`格式取代一般URL，方能通過模型真人審核；
  直接使用一般公網URL（例如R2/Cloudflare連結）上傳含真人臉孔嘅圖片可能觸發審核攔截。
  ⚠️ 此為魔芋平台額外處理步驟，非Seedance 2.0模型本身限制，
     本pipeline（assets.json存R2 URL）若涉及真人臉孔素材，需另外設計asset庫上傳流程，
     現行Seedance_gen_MOYU.py尚未實作此步驟
- JOB JSON無獨立`negative_prompt`欄位／API參數，不應在JSON內加入（此為真實限制，同通用版一致）
- 但prompt文字內可使用negative句式——官方明確示範使用
  "do not generate X" / "keep it X-free" / "no stutter or flicker" 此類完整句
- 寫法使用完整句子，而非"No X, No Y"逗號堆疊（官方示範全部為完整句）
- ⚠️ constraint不保證100%生效，如出現殘留，可配合：landscape生成後再crop、reference圖先去水印／文字
  ⚠️【魔芋版差異】「換seed重生」此對策在魔芋API不適用（無seed參數），
     殘留時只能重複提交同一payload，靠模型自然隨機性重新抽卡
- ⚠️ duration官方硬限制4-15秒（或填-1智能指定）
- ⚠️【已更正】輸出resolution可用 480p / 720p / 1080p / 4k（原文誤寫只有480p/720p）。
  魔芋文件的參數說明只列480p/720p，但metadata為透傳結構，1080p/4k由上游Seedance 2.0直接支援；
  若遇到平台端校驗攔截，可實測後改回720p。查詢API回應的 `data.data.resolution` 可核對實際輸出解析度。
- ⚠️ 單一JOB reference角色人數建議不超過4人，超過4人穩定性下降、容易出現人數錯亂／重複角色（同通用版）
- ⚠️【魔芋版新增】平台預設會於生成影片加浮水印，須於前端「視頻創作」介面手動關閉
  （純API呼叫方式是否受此影響、或有對應參數關閉，官方文件未提及，建議實測確認）
- ⚠️【魔芋版新增】平台不會保存用戶prompt歷史記錄（此為前端網站行為，API呼叫方式應不受影響，
  但提醒你嘅pipeline本身要做好prompt/job紀錄，不可依賴平台回查）
