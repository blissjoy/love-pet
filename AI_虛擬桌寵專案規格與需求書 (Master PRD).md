# **AI 虛擬桌寵專案規格與需求書 (Master PRD v2.1.2)**

**【版本資訊】** Master PRD v2.1.2（2026-11-19 定稿）｜修訂重點：依產品決策調整免費基礎乾糧與飲水為「一日一餐半/兩餐制」擬真數值並閉合 TC-09 B 與 TC-09 C；允許曼赤肯短腿與緬因巨型體型共存（50% short\_legs / 25% giant\_fluffy / 25% normal）；將所有狀態機門檻納入連續交點切割；補齊 30 日行為歷史資料結構、時區防重與全域滿血重置規則。

# **0\. 核心術語表 (Glossary)**

為確保產品、美術與雙平台工程實作命名完全一致，全專案統一採用以下標準術語：

* **Pet / PetState：** 玩家目前養成的單一寵物實體及其所有持續性資料（外觀、需求、性格、記憶、背包、積分）。  
* **GameState：** 整個遊戲的完整存檔狀態封裝（含 schemaVersion、pet 與 dailyLimits）。  
* **Genetics (PetGenetics)：** 寵物誕生時永久鎖定的遺傳資料（含 seed、PRNG 版本、規則版本與最終 Appearance）。  
* **Appearance (PetAppearance)：** 經相容性驗證後的最終七大外觀圖層組合與色彩參數，為畫面渲染的唯一依據。  
* **BasePersonality：** 寵物長期基礎人格數值（親人、慵懶、傲嬌、好奇，範圍 0\~100），由品種天性與帶時間衰減的行為統計構成。  
* **Quirk (QuirkModifiers)：** 小怪癖修正因子（每隻貓最多 2 個），以明確的加法或乘法係數微調特定行為機率，不改變 BasePersonality。  
* **CurrentState：** 寵物當下的生理與情緒即時數值（hunger、hydration、mood、health）與作息狀態。  
* **BehaviorDecision：** BehaviorEngine 輸出的完整行為決策物件（含 animationId、tailState、iconIds、soundId、durationMs、interruptible、cooldownMs）。  
* **PetViewModel：** Game Core 轉出的跨平台統一呈現資料契約，供 React Native UI、Android 懸浮窗與 iOS Widget 唯讀渲染。  
* **Renderer (PlatformRenderer)：** 各平台原生或跨平台呈現層（Kotlin Overlay / SwiftUI Widget / React Native），負責播放動畫與處理平台物理限制，不參與遊戲規則計算。  
* **Clock：** 外部注入的時間來源介面（now(): number），嚴禁在 Game Core 內直接呼叫 Date.now()。  
* **RandomGenerator：** 外部注入的確定性偽隨機數產生器介面（next(): number），確保固定 seed 可完全重現結果。  
* **SimulationStep：** TimeEngine 一般模擬的最大基礎分段步長 (Max Simulation Step)，預設為 0.25 小時（15 分鐘），非唯一結算單位；遇日夜時段邊界、門檻交點、GameCommand 與 Care Pause 啟用/解除時執行精確交點切割 (Exact Boundary / Threshold Crossing Split)。  
* **PeriodFactor：** 日夜時段衰退係數（日間 07:00\~23:00 為 1.0，夜間 23:00\~07:00 為 0.3）。  
* **RecoveryState：** 健康救援狀態機狀態枚舉（NORMAL、CRITICAL、RECOVERY\_BOOST、NATURAL\_RECOVERY）。  
* **RecoveryBoostAccumulatedHours：** 當前救援週期內實際滿足條件並累計的加速恢復時數（上限 2.0 小時）。  
* **CarePause：** 照護暫停保護模式，開啟期間停止需求衰退與健康扣減，解除時不進行離線補算。  
* **TraitPool：** 各基礎品種在七大外觀圖層上的預設候選特徵池與權重配置。  
* **CompatibilityRule：** 七大外觀圖層之間的確定性衝突替換與終極回退驗證規則。  
* **GeneticsVersion / PRNGAlgorithmVersion：** 遺傳邏輯與偽隨機數演算法版本號，用於確保舊貓外觀永久不變。  
* **DailyLimits：** 依裝置本地日期 YYYY-MM-DD 記錄的當日領取上限與已探索商品清單。  
* **GameCommand / GameEvent：** 外部輸入至 GameEngine 的操作指令與引擎輸出的不可變事實事件。  
* **Cooldown：** 行為或提醒觸發後的冷卻時間（ms），期間不重複觸發同類事件。  
* **Deterministic Test / Statistical Test：** 固定單一 seed 的確定性單元測試，以及使用獨立固定 seed 序列（1,000 組）驗證機率分布的統計測試。

# **1\. 專案概述：這是一款什麼樣的產品？ (Elevator Pitch)**

**【一句話產品介紹】**  
這是一款讓虛擬寵物「活」在你手機螢幕最上層的陪伴型 App。不需要刻意打開遊戲，當你在滑 LINE、看 YouTube 甚至看盤時，一隻專屬於你的虛擬貓咪會直接在螢幕邊緣走動、睡覺、陪伴著你。  
**【核心產品承諾（無壓力陪伴底線）】**  
本產品不設寵物死亡機制、不因玩家長期未開啟 App 而永久失去寵物、不設不可逆的健康傷害。健康值設有保護下限（最低降至 10），任何低健康狀態皆可透過免費基礎照護或救援模式恢復，並提供無懲罰的「照護暫停模式 (Care Pause)」。  
**【寫給首次閱讀本企劃的人：我們解決了什麼問題？】**

* **對一般大眾 (User)：** 解決了「想養寵物但環境不允許」或「渴望日常陪伴」的痛點。這不是一款需要死板打卡的手遊，而是透過連動手機真實的電量、時間與計步器，讓你感覺手機裡真的住著一隻有情緒的貓咪。  
* **對贊助廠商 (Brand)：** 解決了「寵物廣告轉換率低、惹人厭」的痛點。我們將真實的貓砂、主食罐置入遊戲商店中，玩家在選購飼料時自然閱讀真實成分與營養介紹，廠商付費取得「被精準受眾看見的機會」，實現零反感的**內容型原生廣告 (Native Advertising)** 與導購平台 (B2B2C)。

**【技術與發展藍圖】**  
採用「平台解耦」與「離線優先」架構，以純 TypeScript 打造跨平台遊戲核心（Game Core），先於 Windows 本機完成核心規則驗證與自動化測試，再分別透過原生適配器（Adapter）落地於 Android 全域懸浮窗與 iOS 桌面 Widget / App 內互動。初期依賴 2D 肢體動畫與腳本狀態機以零伺服器成本驗證市場；專案的終極目標，是在商業模式與核心玩法驗證成功後，於未來階段接入 LLM（大型語言模型）賦賦予貓咪深度對話能力（LLM 僅負責語言與情境表達，絕不直接修改核心數值或跳過 Game Core 規則）。

# **2\. 商業模式深度解析 (內容型原生廣告與 O2O 導購)**

有別於傳統手遊強迫觀看無關廣告或用數值懲罰逼迫課金，本專案的核心是「將商業產品化為遊戲商店的真實內容」，讓選擇權完全回歸玩家。

## **2.1 無反感原生廣告的五大核心原則**

1. **商品是選擇，不是強迫（免費照護保障）：** 玩家可自由使用基礎免費食物與飲水，足以持續維持貓咪健康與基本需求，絕不會因為沒買合作品牌商品而導致貓咪生病或受到不可逆懲罰。  
2. **廣告與一般商品並存：** 商店內同時提供一般遊戲道具與合作廠商商品，合作商品清楚標示「品牌合作 / 贊助內容」，維持畫面清爽。  
3. **商品介紹具備真實知識價值：** 點進飼料可查看真實品牌、原料成分、蛋白質/脂肪/水分營養分析、適用對象、資料來源及資料更新日期；MVP 階段之模擬商品須於 UI 明確標示「測試商品/模擬資料」。  
4. **品牌公平與平衡原則：** 品牌合作商品不得因贊助關係取得額外數值加成。所有商品的遊戲回復效果均依固定的遊戲平衡公式配置（例如主食罐補水高、乾糧飽食高、零食心情高），廠商購買的是「品牌曝光與虛擬櫃位」，贊助費不得直接影響商品數值。  
5. **自然探索不干擾：** 僅在玩家主動進入商店挑選商品時才呈現完整品牌介紹，不彈出干擾性蓋版廣告，亦不設計逼迫刷廣告的機制。

## **2.2 商業假設與變現路徑 (商業驗證指標)**

1. **核心市場假設 (Hypothesis) 與驗證指標：**  
   1. 假設 1「無法養貓者需要桌寵陪伴」➔ 驗證指標：Beta 訪談與 7 日/30 日留存率。  
   2. 假設 2「飼主願意閱讀成分分析且不排斥品牌商品」➔ 驗證指標：商店進入率、詳情頁停留時間與退出率。  
2. **使用者 ➔ 產品 ➔ 利潤轉換路徑（列為未驗證商業模型，Phase D 後逐步驗證）：**  
   1. **路徑 1：品牌櫃位曝光與知識探索 (CPA / 櫃位贊助費)：** 廠商提供經審核的商品圖文與營養成分上架虛擬櫃位，支付櫃位曝光費或有效探索費（同帳號同商品不重複計費）。  
   2. **路徑 2：實體導購分潤 (CPS)：** 遊戲內提供專屬折扣碼與電商連結，依實際成交金額抽取 10% \- 20% 分潤（後續商業驗證階段串接歸因追蹤）。  
   3. **路徑 3：去識別化統計洞察 (隱私優先)：** 僅在使用者明確同意下收集不可識別個人的匿名問卷統計，以彙總研究報告形式提供合作品牌；嚴禁出售個別使用者行為軌跡、定位紀錄或可識別資訊。

## **2.3 遊戲內可納入的實體產品類別清單**

1. **主食與副食品 (核心消耗品)：** 頂級乾糧、無穀主食罐（如陪心、皇家、希爾斯）、凍乾肉餅、機能性副食罐、貓草/貓薄荷。  
2. **保健醫療食品 (日常保養對應)：** 腸胃益生菌、排毛粉、泌尿道保健液、關節保養錠、維生素 B 群、體外驅蟲滴劑。  
3. **生活用品 (設備升級)：** 豆腐砂/礦砂（對應鏟屎任務）、陶瓷護頸食盆、恆溫貓窩、貓抓板、寵物推車/外出背包。  
4. **互動玩具 (耐久度消耗)：** 貓草魚、電動逗貓球、智慧雷射筆。

*（註：MVP Phase D 初期商品目錄僅開放「主食與副食品」及「零食」類別；保健醫療食品、生活用品與互動玩具列為 Phase E 後續擴充，暫不納入 MVP 實作。）*

## **3.1 品種特徵傳承與外觀生成機制 (Trait Inheritance System)**

為了讓每位玩家擁有獨一無二的專屬貓咪，並確保跨平台與版本更新後的外觀穩定性，採用「品種初始特徵池 \+ 獨立圖層抽選 \+ 相容性驗證」機制：

1. **父母品種選擇：** 玩家從 6 大基礎品種（米克斯、曼赤肯、布偶、英短、異國短/長毛、緬因）中挑選父與母。品種提供初始特徵池（Trait Pool）與基礎性格傾向，而非直接寫死最終外觀。  
2. **七大外觀圖層拆解：** 貓咪外觀拆解為「頭型 (head)、耳朵形狀 (ears)、眼睛形狀 (eyes)、瞳孔顏色 (pupilColor)、花色斑紋 (coatPattern)、尾巴長度 (tail)、體型 (body)」7 大圖層。  
3. **每圖層獨立抽選法則（40/40/20）：** 每個未鎖定圖層作為獨立抽選單位：  
   1. 第一階段來源抽選：40% 繼承父方品種預設特徵（若父方該圖層特徵池有多個候選值，則依池內權重再抽一次）、40% 繼承母方品種預設特徵、20% 進入變異路徑。  
   2. 第二階段變異路徑（佔總機率 20%）：其中 `75%`（即總機率 `15%`）從全品種特徵池隨機重組，`25%`（即總機率 `5%`）從稀有突變池抽取（異色瞳、麒麟尾、折耳、星空斑紋）。  
   3. 若父母在某圖層特徵相同，父系與母系抽選結果均視為該特徵，機率不合併至突變池，突變仍依獨立機率抽選。  
4. **Seed 與版本永久鎖定規範：**  
   1. seed 僅用於生成初始遺傳結果，不作為既有寵物外觀的唯一保存依據。  
   2. 每隻寵物生成後，必須永久保存完整的最終 appearance、colorParams、geneticsVersion、prngAlgorithmVersion 與 rulesVersion。  
   3. **任何後續 Genetics、PRNG、Trait Pool 或 Compatibility Rule 更新，均不得重新生成或覆寫既有寵物的外觀。**  
5. **GrowthStage MVP 規範：** MVP 初期不實作實際年齡成長與外觀變化。ageInDays 僅作為遊戲累計天數資料預留，不影響外觀、動畫、健康或人格數值；growthStage 初期固定為 "adult"，保留未來版本擴充。

## **3.2 三層行為決策模型 (Behavior Decision Model)**

為避免「永久人格」與「暫時生理狀態」混淆，性格與行為拆分為三層獨立架構：

* **第一層：基礎人格 (BasePersonality)：** 由品種天性（權重 60%）與長期後天養成記憶（權重 40%）構成，決定 affection（親人）、laziness（慵懶）、tsundere（傲嬌）與 curiosity（好奇）（範圍 0\~100）。  
  * **後天養成記憶計算公式 (四維度獨立程式級數學定義)：** 統計視窗固定為「過去 30 日滾動視窗」（recent7Days 為最近 7 日，prior23Days 為第 8\~30 日）。若某維度 30 日內無互動，記憶分數退回該品種天性基準 breedBase：  
    * **親人度 (affection)：** 有效行為為「有效摸頭（目標 21 次/7日，權重 60%）」與「餵食主食罐（目標 7 次/7日，權重 40%）」。r7 \= min(1, (pets7d / 21\) \* 0.6 \+ (wetFood7d / 7\) \* 0.4)，r23 \= min(1, (pets23d / 69\) \* 0.6 \+ (wetFood23d / 23\) \* 0.4)，affectionMemoryScore \= (r7 \* 0.7 \+ r23 \* 0.3) \* 100，finalAffection \= round(breedAffection \* 0.6 \+ affectionMemoryScore \* 0.4)。  
    * **好奇心 (curiosity)：** 有效行為為「有效玩耍（目標 14 次/7日）」。r7 \= min(1, plays7d / 14)，r23 \= min(1, plays23d / 46)，curiosityMemoryScore \= (r7 \* 0.7 \+ r23 \* 0.3) \* 100，finalCuriosity \= round(breedCuriosity \* 0.6 \+ curiosityMemoryScore \* 0.4)。  
    * **慵懶度 (laziness)：** 依據 7 日總互動次數（摸頭 \+ 玩耍 \+ 餵食）線性插值：interactions7d \<= 3 時 lazinessMemoryScore \= 100；interactions7d \>= 14 時 lazinessMemoryScore \= 20；介於 3 \~ 14 之間採線性插值 100 \- ((interactions7d \- 3\) / 11\) \* 80，finalLaziness \= round(breedLaziness \* 0.6 \+ lazinessMemoryScore \* 0.4)。  
    * **傲嬌度 (tsundere)：** 結合「高級罐頭/零食餵食（目標 7 次/7日，降低傲嬌）」與「冷落懲罰（7 日總互動 \< 3 次時升至 90）」：當 interactions7d \< 3 時 tsundereMemoryScore \= 90；否則 treatRatio \= min(1, premiumFood7d / 7)，tsundereMemoryScore \= 60 \- treatRatio \* 50（即滿標時降至 10），finalTsundere \= round(breedTsundere \* 0.6 \+ tsundereMemoryScore \* 0.4)。  
* **第二層：小怪癖修正 (QuirkModifiers)：** 每隻貓最多擁有 2 個小怪癖，採用明確的修飾型別微調行為觸發機率，不改變 BasePersonality 數值：  
  * additiveProbabilityModifier：加法百分點修正（如 \+0.30 代表機率直接加 30 個百分點）。  
  * multiplicativeWeightModifier：乘法權重修正（如 1.3 代表原始權重乘以 1.3 倍）。  
  * 多個 Quirk 同時作用於同一行為時，先套用加法修正再套用乘法修正，最終機率嚴格限制（Clamp）於 \[0.05, 0.95\] 區間。  
* **第三層：即時狀態調節 (CurrentStateModifier)：** 飢餓、水分、心情、健康與作息時段作為即時條件輸入 BehaviorEngine，影響當下動作選擇，絕不反向永久修改 BasePersonality。

# **4\. 核心功能與互動機制**

## **4.1 雙平台差異化呈現與生命週期規範**

* **Android 端 (全域懸浮與物理互動)：**  
  * 透過 Kotlin 原生 `WindowManager` (`SYSTEM_ALERT_WINDOW`) 建立透明懸浮視窗，支援螢幕漫步、邊界轉向、點擊摸頭、長按拖曳與虛擬重力掉落（拖曳與掉落僅觸發短暫抗議動畫，絕不扣減健康值）。  
  * **Android 懸浮窗生命週期規範：**  
    * 使用者隱藏桌寵至通知列或完全關閉 Overlay 時，僅停止畫面渲染，Game Core 仍透過時間戳正常運作，App 內主房間仍可正常操作。  
    * 若使用者於系統設定撤銷 Overlay 權限，App 自動切換為「App 內模式」並保存進度，不產生閃退；手機重新開機後 MVP 不強制背景自動啟動。  
    * 進入省電模式時自動降低動畫影格率（FPS）；懸浮視窗不搶奪系統輸入焦點，使用者正在輸入文字時絕不攔截鍵盤或觸控操作。  
* **iOS 端 (App 內完整互動 \+ 桌面 Widget)：**  
  * 明確定義 **iOS Widget 僅為寵物狀態展示與有限快捷操作入口 (App Intents)**，受 iOS 系統排程限制（約 15\~60 分鐘更新），不保證即時動畫或背景持續執行 `Game Core`；完整狀態計算與互動於開啟 App 時執行。

## **4.2 時間差模擬與照護暫停保護 (省電核心)**

* **時間區段拆分模擬 (Time-based Simulation)：** 捨棄耗電的背景常駐服務。系統記錄最後模擬時間戳 (lastSimulatedAt)，重新開啟時由 TimeEngine 將離線期間精準切分為 0.25 小時（15 分鐘）刻度，依「日間清醒（07:00\~23:00）」與「夜間休眠（23:00\~07:00）」區段循序計算需求與健康變化，確保分段模擬與單次長離線模擬結果完全一致。  
* **照護暫停保護模式 (Care Pause)：** 玩家忙碌或出差時可手動開啟 Care Pause（MVP 不限制暫停時長）。開啟期間停止 hunger、hydration、mood 衰退與 health 連鎖扣減，不進行離線補算；玩家仍可開啟 App 查看寵物與互動，解除暫停後重新以解除當下之時間戳作為模擬起點。  
* **環境情境動畫（不影響核心數值）：** 偵測到手機電量低於 20% 呈現打瞌睡、插線充電時呈現開心進食/充電特效，此類系統事件僅作為環境互動動畫，**不直接增減飽食度、水分或健康值**，避免玩家產生數值困惑。

## **4.3 肢體語言與圖示回饋 (MVP 無文字對話)**

* **肢體優先：** 初期完全捨棄文字對話氣泡，依賴 2D 肢體動畫（8 種尾巴情緒指標、母雞蹲、貴妃躺、背對生氣等）與動態圖示（🐟、💧、❤️、💢、🏥）搭配貓叫音效表達情緒。

## **4.4 照護與商店探索系統**

* **日常照護：** 每日免費提供無限量基礎乾糧與過濾飲用水，搭配摸頭與玩耍互動即可維持健康。  
* **原生廣告商店：** 玩家累積積分後可選購一般罐頭/零食或品牌合作商品，點擊商品詳情可查看真實成分與營養分析；購買後的商品進入背包，由玩家自主選擇使用時機。

# **5\. 技術架構與工程原則 (Monorepo \+ Platform Decoupling)**

## **5.1 四大核心架構原則與 Phase 實作邊界標籤**

1. **嚴格平台解耦 (Platform Decoupling)：**`Game Core` 必須為純 TypeScript 跨平台模組，所有時間、隨機數、存檔、外部設定與平台資訊均透過依賴注入（Dependency Injection）或外部輸入提供，必須能在 Windows Node.js 環境獨立執行與測試。  
2. **\[Phase A 必須實作\]：** 純 TypeScript Game Core、本地 Config Loader（讀取 content/\*.json，負責檔案讀取與 Schema 驗證，Game Core 只接受已驗證的設定物件）、Clock / RandomGenerator 依賴注入、防損 JSON 存檔與 Vitest 驗收測試。  
3. **\[Phase A 預留介面\]：** GameRepository 介面、PetViewModel 契約、GameEvent 結構。  
4. **\[Phase A 不實作 / Future Phase\]：** 遠端 GitLab Raw HTTP 拉取、Android WindowManager 原生模組、iOS WidgetKit、SQLite、Google Drive 備份、Firebase 串門子與 LLM API。  
   1. **明確禁止在 Game Core 內使用：**`window` / `document`（Browser 專屬）、`fs` / `path` / `process`（Node.js 專屬）、`Date.now()`（導致時間不可控）、`Math.random()`（破壞固定 Seed 重現性）、`React / React Native`、`Kotlin / Swift API` 以及直接發起網路請求。  
5. **設定檔外部載入與驗證 (Config Loader)：**  
   1. **Phase A（Windows 本機驗證）：** config-loader 直接讀取專案內 content/ 目錄之靜態 JSON，執行 Schema 驗證後注入 Game Core。  
   2. **Phase C / D（手機端擴充）：** config-loader 支援從遠端（GitLab Raw）檢查版本號、下載更新本機快取並驗證 Schema；網路中斷或 JSON 損壞時自動退回本機快取或內建預設版本，Game Core 本身不感知設定檔來源。  
6. **離線優先 (Offline-first) 與免防作弊設計：** 個人陪伴養成無 PvP 與排行榜，不建置伺服器結算；僅保留存檔版本遷移（Migration）、格式防損驗證與本機每日領取防重機制。  
7. **呈現與存檔分離：** 存檔僅保存基因、數值、記憶與時間戳，不儲存動畫影格；統一由 BehaviorEngine 輸出單一標準結構 BehaviorDecision，再由 GameEngine 轉換為跨平台唯讀契約 PetViewModel 交由各平台 Renderer 呈現。

## **5.2 技術選型與跨平台呈現契約 (PetViewModel) 實作分工**

| 架構層級 | 技術選型 | 責任與功能說明 |
| :---- | :---- | :---- |
| 遊戲核心 (100% 共用) | 純 TypeScript (packages/game-core) | 負責品種傳承、性格、需求與健康平衡、時間區段模擬 (TimeEngine)、行為仲裁與經濟規則，零平台依賴。 |
| 設定載入與契約層 | TypeScript (config-loader / contracts) | 負責靜態 JSON 讀取、本機快取管理、Fallback 保護與跨模組資料 Schema 驗證。 |
| 跨平台 UI 層 | React Native \+ TypeScript | 負責雙平台共用的 App 主房間、領養、商店、商品成分頁與設定介面。 |
| Android 原生適配器 | Kotlin (WindowManager) | 負責 Android 透明懸浮窗、生命週期管理、觸控拖曳、重力物理與系統電量事件回傳。 |
| iOS 原生適配器 | Swift / SwiftUI (WidgetKit) | 負責桌面 Widget 狀態展示、App Intents 快捷操作佇列與 App Group 資料共享。 |
| 本機與雲端資料流 | JSON (Phase A) / SQLite (Mobile) | 初期 (0 成本)：Windows 採防損 JSON 存檔，手機端採 SQLite；後期擴充 Google Drive 個人備份與 Firebase 社交信箱。 |

* **跨平台統一呈現契約 (PetViewModel Schema)：**  
  * 平台端嚴禁自行計算需求或決定行為，一律唯讀消費 PetViewModel 完整欄位契約：  
    * `petId` (string)、`name` (string)、`appearance` (PetAppearance)  
    * `needs` ({ hunger, hydration, mood, health })  
    * `behavior` ({ animationId: AnimationId, tailState: TailState, iconIds: IconId\[\], soundId: SoundId \| null, durationMs: number })（註：interruptible 與 cooldownMs 僅供 Game Core 內部狀態機仲裁，不拋出給 ViewModel）  
    * `display` ({ isSleeping: boolean, isHealthCritical: boolean, recoveryState: RecoveryState, isCarePaused: boolean })（其中 isSleeping \= (behavior.animationId \=== "sleep")；isHealthCritical \= (pet.health \< 40)；recoveryState \= pet.recoveryState；isCarePaused \= pet.isCarePaused）  
    * `updatedAt` (number，Unix timestamp ms)

## **5.3 Monorepo 專案目錄結構**

* **ai-virtual-pet/** (根目錄)  
  * **packages/**  
    * **game-core/：**純 TypeScript 遊戲核心（零平台依賴）  
      * `domain/`：寵物、基因、性格、記憶、需求、健康、背包、經濟  
      * `engine/`：GameEngine、TimeEngine、BehaviorEngine  
      * `ports/`：依賴注入介面（Clock、RandomGenerator、GameRepository、PlatformRenderer）  
    * **contracts/：**跨模組資料契約（Command / Event / ViewModel / Save Schema）  
    * **config-loader/：**靜態 JSON 載入、本機快取、Schema 驗證與 Fallback 模組  
  * **content/：**版本化靜態設定檔（genetics/、traits/、behavior-rules/、products/）  
  * **assets/：**貓咪圖層素材、動畫、圖示與音效（cats/、animations/、icons/、sounds/）  
  * **apps/**  
    * **local-cli/：**Phase 0 Windows 本機 CLI 測試介面  
    * **mobile/：**React Native 手機端專案  
      * `android/`：Kotlin 懸浮窗與物理引擎 Adapter  
      * `ios/`：SwiftUI Widget 與 App Intents Adapter  
      * `src/`：共用畫面（領養、商店、圖鑑、設定）  
  * **tests/：**Vitest 三層自動化測試（單元測試、整合測試、365 天長時間模擬）  
  * **docs/：**架構文件、遊戲規則與驗收標準（architecture/、game-rules/、acceptance-tests/）

# **6\. 長期留存與防流失機制 (User Retention)**

1. **破壞第四面牆的情境彩蛋：** 偵測深夜使用手機時，寵物做出催促睡覺的肢體反應（打哈欠、指著時鐘圖示），且不阻止玩家主動互動。  
2. **動態偏好與挑食機制：** 寵物偶爾對單一食物失去新鮮感，引導玩家探索商店內其他商品與成分。  
3. **照護暫停保護機制 (Care Pause)：** 針對忙碌或出差玩家提供無懲罰的暫停衰退保護，維持零壓力陪伴定位。  
4. **微型月卡與送養重置機制（商業驗證後導入）：** 月卡定位為純裝飾與便利（如專屬房間背景、特殊動畫、照護歷史紀錄），絕不將「避免寵物受苦」作為付費門檻；送養重置須經二次確認，重置目前寵物養成狀態但保留全域設定與已解鎖圖鑑。

# **7\. 開發與實作路線圖 (Windows 核心驗證 ➔ 雙平台 MVP)**

* **Phase A：Windows Local Core Prototype（第一優先，0 手機環境依賴）**  
  * 建立 Monorepo、packages/game-core、contracts、config-loader（Local 模式）與 Windows CLI 操作介面。  
  * 實作基因生成、三層行為決策、0.25 小時步進之時間區段模擬、健康救援狀態機、防損 JSON 存檔（.tmp 寫入驗證與 .bak 備份），並通過 Vitest 單元、整合與 365 天長時模擬測試。  
* **Phase B：Renderer Proof of Concept（渲染與跨平台骨架驗證）**  
  * 建立單隻「測試貓」與最少必要動畫（Idle、Walk、Sit、Sleep、Eat、Pet），驗證 7 大圖層換裝相容性。  
  * 同步驗證 React Native 畫面、Android Kotlin 懸浮窗與 iOS SwiftUI Widget 讀取同一份 PetViewModel 的一致性後，再正式選定動畫引擎（Spine 2D / Rive / Sprite Sheet）。  
* **Phase C：Pet MVP（雙平台單貓陪伴上線）**  
  * 開放領養（MVP 限制單一帳號飼養一隻貓，確保模擬與懸浮窗體驗穩定）、7 層外觀生成、基礎性格與互動、完整需求與健康循環、SQLite 存檔與版本遷移（Migration）。  
* **Phase D：商店與內容型原生廣告驗證**  
  * 上線積分系統、商品背包、商店與「成分與營養分析」詳情頁，放入 5\~10 款模擬與合作品牌商品（導入 Remote Config），驗證玩家探索意願與留存數據。  
* **Phase E：長期陪伴、社交與終極 LLM 願景**  
  * 依序擴充 Google Drive 個人雲端備份、Firebase 好友串門子、計步與戶外散步、回憶相簿；待留存與商業模式成熟後，於最後階段接入 LLM API（僅讀取性格與記憶進行語言表達，搭配 BYOK 或對話券控管成本）。

# **8\. 系統細部規格展開 (PM & RD 開發基準)**

## **8.A 貓咪基因與圖鑑設定表 (Trait Inheritance Spec)**

### **8.A.1 完整寵物與基因資料模型 (PetState & PetGenetics Schema)**

1. **PetGenetics（生成後永久鎖定，隨版本升級不得重算）：**  
   * `seed` (string)：生成該貓咪時使用的隨機種子  
   * `prngAlgorithmVersion` (string)：偽隨機數演算法識別碼（如 "mulberry32-v1"）  
   * `geneticsVersion` (string)：遺傳邏輯版本號（如 "1.0.0"）  
   * `rulesVersion` (string)：品種與相容性設定檔版本號（如 "v1"）  
   * `fatherBreed` / `motherBreed` (BreedId)：父母品種識別碼  
   * `appearance` (PetAppearance)：最終確定的 7 大圖層 ID 與 colorParams（唯一渲染依據）  
2. **PetState（主存檔核心實體）：**  
   * `id` (string)：唯一識別碼 (UUID)  
   * `name` (string)：貓咪名稱（1\~12 字元）  
   * `ageInDays` (number)：遊戲累計天數（預設 0）  
   * `growthStage` ("kitten" \| "young" \| "adult" \| "senior")：外觀成長階段（MVP 固定為 "adult"）  
   * `genetics` (PetGenetics)：不可變遺傳與外觀資料  
   * `basePersonality` (object)：基礎人格數值（0\~100）：affection（親人）、laziness（慵懶）、tsundere（傲嬌）、curiosity（好奇）  
   * `quirks` (QuirkId\[\])：小怪癖 ID 陣列（最多 2 個）  
   * `memories` (object)：行為統計記憶（totalPets、totalPlays、foodCategoryCounts、dailyBuckets: Record\<string, { pets: number; plays: number; wetFoods: number; premiumFoods: number; totalInteractions: number }\>，保留最近 30 個本地日期 YYYY-MM-DD 鍵值，超過 30 日自動滾動清理）。計算時機：每次執行有效互動指令（FeedPet、PetCat、PlayWithPet）後立即重算 basePersonality；每次開啟 App 跨日結算時亦重算一次。  
   * `needs` (object)：即時需求數值（嚴格限制 0\~100）：hunger（飽食）、hydration（水分）、mood（心情）  
   * `health` (number)：健康值（嚴格限制 10\~100，設 10 為保護下限）  
   * `recoveryState` ("NORMAL" \| "CRITICAL" \| "RECOVERY\_BOOST" \| "NATURAL\_RECOVERY")：健康狀態機階段  
   * `recoveryBoostAccumulatedHours` (number)：當前救援週期已累積之有效加速恢復時數（上限 2.0 小時）  
   * `isCarePaused` (boolean)：照護暫停保護開關（預設 false）  
   * `coins` (number)：持有積分（開局預設 200，不得為負數）  
   * `inventory` (Record\<ItemId, number\>)：背包物品數量表  
   * `lastSimulatedAt` (number)：最後模擬結算 Unix 時間戳 (ms)（時間單一權威來源）

### **8.A.2 初期收錄 6 大基礎品種初始特徵池 (breeds.v1.json) 與變異分布**

3. **權重池配置與稀有突變分布 (traits.v1.json)：** 每個品種在 7 大圖層上皆具備 70% 主特徵 / 30% 次要特徵的加權候選池：米克斯花色 (tabby\_mackerel: 70%, calico: 30%)、曼赤肯花色 (bicolor\_tuxedo: 70%, calico: 30%)、布偶花色 (colorpoint: 70%, bicolor\_tuxedo: 30%)、英短花色 (solid: 70%, tabby\_mackerel: 30%)、異國花色 (tabby\_mackerel: 70%, colorpoint: 30%)、緬因花色 (tabby\_mackerel: 70%, tortoiseshell: 30%)。第二階段變異路徑中，15% 全品種重組僅從合法通用特徵白名單（排除專屬極端骨架 flat\_face 與 giant\_fluffy，防止緬因或布偶抽到不協調骨架）中等權重隨機抽取；25% 稀有突變池（總機率 5%）採等權重分布（各 1.25%）：ears: "folded\_tip"、pupilColor: "heterochromia\_blue\_gold"、coatPattern: "starry\_rare"、tail: "kinked\_kirin"。

| 品種 ID (BreedId) | 顯示名稱 | 初始預設特徵池 (Trait Pool \- 七層完整定義) | 基礎性格傾向 (0\~100) | 合法毛色 HEX 色碼區間 (Shader 基準) |
| :---- | :---- | :---- | :---- | :---- |
| `domestic` | 米克斯 | head: standard\_round, ears: pointed\_medium, eyes: almond, pupilColor: amber, coatPattern: tabby\_mackerel, tail: long\_straight, body: normal | 親人 60 / 慵懶 50 / 傲嬌 50 / 好奇 80 | 橘 \#E68A3E\~\#F5C284、虎斑 \#6E5D4F\~\#A89888、黑 \#2B2B2B、白 \#F8F8F8 |
| `munchkin` | 曼赤肯 | head: standard\_round, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: bicolor\_tuxedo, tail: long\_straight, body: short\_legs | 親人 75 / 慵懶 45 / 傲嬌 40 / 好奇 85 | 乳白 \#F5E6D3、三花混色 \#D98245 \+ \#333333、藍灰 \#7A8591 |
| `ragdoll` | 布偶貓 | head: standard\_round, ears: pointed\_medium, eyes: big\_round, pupilColor: sapphire, coatPattern: colorpoint, tail: fluffy\_plume, body: large\_fluffy | 親人 90 / 慵懶 75 / 傲嬌 20 / 好奇 50 | 奶油白底 \#FAF5EE 搭配海豹重點色 \#4A3728 或藍重點色 \#6B7A8F |
| `british_shorthair` | 英國短毛貓 | head: chubby\_round, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: solid, tail: short\_bobtail, body: chubby | 親人 55 / 慵懶 85 / 傲嬌 65 / 好奇 40 | 經典藍灰 \#6C7680\~\#8C96A0、銀漸層 \#D8DEE3、金漸層 \#D9A760 |
| `exotic` | 異國短/長毛 (加菲/喜馬拉雅) | head: flat\_face, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: tabby\_mackerel, tail: short\_bobtail, body: chubby | 親人 70 / 慵懶 80 / 傲嬌 55 / 好奇 45 | 橘紅 \#D97236、奶油重點色 \#F2E4D5 \+ 巧克力重點 \#5C4033 |
| `maine_coon` | 緬因貓 | head: square\_muzzle, ears: tufted\_large, eyes: almond, pupilColor: emerald, coatPattern: tabby\_mackerel, tail: fluffy\_plume, body: giant\_fluffy | 親人 80 / 慵懶 50 / 傲嬌 35 / 好奇 75 | 棕虎斑 \#594636\~\#8C735B、煙燻黑 \#3A3D40、紅虎斑 \#C86D3B |

### **8.A.3 七大外觀圖層枚舉與確定性相容修正規則 (appearance-compatibility.v1.json)**

4. **七大圖層枚舉：**  
   * `head`：`standard_round`、`chubby_round`、`flat_face`、`square_muzzle`  
   * `ears`：`pointed_medium`、`small_round`、`tufted_large`、`folded_tip`（稀有突變）  
   * `eyes`：`almond`、`big_round`、`droopy_cute`  
   * `pupilColor`：`amber`、`emerald`、`sapphire`、`copper`、`heterochromia_blue_gold`（稀有突變）  
   * `coatPattern`：`solid`、`tabby_mackerel`、`bicolor_tuxedo`、`colorpoint`、`calico`、`tortoiseshell`、`starry_rare`（稀有突變）  
   * `tail`：`long_straight`、`fluffy_plume`、`short_bobtail`、`kinked_kirin`（稀有突變）  
   * `body`：`normal`、`chubby`、`short_legs`、`large_fluffy`、`giant_fluffy`  
5. **確定性相容修正規則 (appearance-compatibility.v1.json)：**  
   * **規則 1（跨圖層：扁臉與大型骨架比例衝突）：** 若 head \=== "flat\_face" 且 body \=== "giant\_fluffy"，為避免扁臉搭配巨型緬因骨架比例失真，強制將 body 修正為 "chubby"。  
   * **規則 2（跨圖層：重點色與花色遮罩衝突）：** 若 coatPattern \=== "colorpoint" 且 pupilColor \=== "amber"，強制將 pupilColor 修正為 "sapphire"。  
   * **規則 3（跨圖層：稀有折耳與緬因大耳羽衝突）：** 若 ears \=== "folded\_tip" 且 body \=== "giant\_fluffy"，強制將 ears 修正為 "tufted\_large"。  
   * **規則 5（終極安全回退 SAFE\_DEFAULT\_APPEARANCE）：** 若經過單次規則替換後最終驗證仍不合法，直接退回完整七層安全預設組合 SAFE\_DEFAULT\_APPEARANCE：`{ head: "standard_round", ears: "pointed_medium", eyes: "almond", pupilColor: "amber", coatPattern: "solid", tail: "long_straight", body: "normal" }`。

### **8.A.4 遺傳生成執行優先級 (Genetics Priority)**

為避免特殊骨架規則與一般 40/40/20 規則衝突，強制規定執行順序：

6. **步驟 1（特殊顯性骨架規則優先，僅作用於指定圖層）：**  
   * **曼赤肯 (munchkin) body 規則：**  
     * 若父母雙方皆為 `munchkin`（雙親同種）：子代 `body` 為 **75%** `short_legs` **/ 25%** `normal`（避免重複計算「另一方體型」）。  
     * 若僅一方為 `munchkin`（跨品種配對）：子代 `body` 為 **50%** `short_legs` **/ 25% 另一方品種之** `body`（若另一方為 maine\_coon 則為 giant\_fluffy，允許 short\_legs 與 giant\_fluffy 共存形成短腿巨型長毛體型） / **25%** `normal`。  
   * **異國短/長毛 (exotic) head 規則：**  
     * 若父母雙方皆為 `exotic`（雙親同種）：子代 `head` 為 **80%** `flat_face` **/ 20%** `standard_round`（避免 100% 鎖死單一頭型）。  
     * 若僅一方為 `exotic`（跨品種配對）：子代 `head` 為 **45%** `flat_face` **/ 55% 另一方品種之** `head`。  
7. **步驟 2（一般圖層獨立 40/40/20 抽選）：** 其餘未鎖定圖層依固定順序（head ➔ ears ➔ eyes ➔ pupilColor ➔ coatPattern ➔ tail ➔ body）逐一獨立抽選：40% 父系預設特徵、40% 母系預設特徵、20% 變異路徑（變異路徑內 75% 全品種特徵池重組、25% 稀有突變池）。  
8. **步驟 3（確定性相容性修正）：** 依 8.A.3 規則執行單次衝突替換與最終驗證。  
9. **步驟 4（動態毛色微調）：** 依據 `seed` 在合法 HEX 區間內進行 `±8%` 明度與飽和度微調。

## **8.B 動畫與行為決策仲裁規格 (BehaviorEngine & FSM Spec)**

### **8.B.1 行為決策引擎優先級、同級仲裁與中斷規則**

BehaviorEngine 輸出單一標準結構 BehaviorDecision（animationId、tailState、iconIds: string\[\]、soundId、durationMs、interruptible、cooldownMs）：

1. 

| 優先級 | 狀態類別 | 觸發條件（嚴格運算符） | 行為輸出（主動畫 / 尾巴 / 圖示陣列） | 同級仲裁、持續與中斷規範 |
| :---- | :---- | :---- | :---- | :---- |
| **100** | 緊急生理求救 (CRITICAL\_NEEDS) | `health < 30` 或 `hunger < 15` 或 `hydration < 15` | 匍匐警戒 (crouch\_alert) 或虛弱打瞌睡，夾尾/垂尾，彈出 🏥 / 🐟 / 💧 | **同級仲裁**：iconIds 可同時顯示所有達標圖示；主動畫依明確清單順序 **1\. health \< 30 ➔ 2\. hydration \< 15 ➔ 3\. hunger \< 15** 決定。可中斷任何低優先級行為（包含 Priority 90 與 Priority 40 eat），持續播放至 health \>= 30 && hunger \>= 15 && hydration \>= 15。 |
| **90** | 負面防禦反應 (NEGATIVE\_REACTION) | 10 秒內 \[now \- 10000, now\] `POKE > 3` 次（第 4 次觸發），或高空掉落 LANDING（定義為 Android 懸浮窗拖曳放開時，垂直落下距離超過螢幕高度 25% 觸發，短距離輕放不觸發） | 背對生氣 (turn\_back\_angry) 或炸毛，快速鞭打尾，彈出 💢 並哈氣 | 若同時發生，主動畫優先序為 **LANDING \> POKE**。優先級高於 Priority 80，可暫時喚醒睡眠中的貓咪執行 4,000 ms 動畫，結束後進入 15,000 ms 撫摸無效冷卻期。撫摸冷卻（15,000 ms）與 Priority 60 提醒冷卻（60,000 ms）各自獨立計時。冷卻期間摸頭不給心情/積分，但餵食/補水仍有效；冷卻期間再次 Poke **不延長** 冷卻時間。 |
| **80** | 生理作息睡眠 (SLEEP\_MODE) | 本地時間 `23:00 ~ 07:00` 且無玩家主動互動 | 靠邊熟睡 (sleep) 或母雞蹲 (cat\_loaf)，休眠捲曲尾，彈出 💤 | **夜間可喚醒規則**：僅能被有效的玩家數值互動指令（FeedPet、PetCat、PlayWithPet）暫時喚醒並執行 Priority 40，互動結束後重新入睡；低電量、插線充電等系統事件不喚醒睡眠。 |
| **60** | 一般需求提醒 (MODERATE\_NEEDS) | `hunger < 40` 或 `hydration < 40` 或 `mood < 45` | 坐下 (sit) 盯著玩家，尾巴慢速左右掃動，彈出 🐟 / 💧 / 🎾 | **同級仲裁**：iconIds 合併顯示所有達標提醒圖示；主動畫依 **hydration \< 40 \> hunger \< 40 \> mood \< 45** 決定。每次提醒持續 `5,000 ms`，共用冷卻 `60,000 ms`（從提醒動畫首次開始播放的時間戳 startedAt 起算）。 |
| **40** | 玩家主動互動 (PLAYER\_INTERACTION) | 收到 FeedPet（使用食物）或 PetCat（有效摸頭） | 進食 (eat) 舉問號尾微顫；摸頭呼嚕 (pet) 彈出 ❤️ | 所有 GameCommand 統一回傳 CommandResult { status: "applied" \| "rejected", reason?, state: GameState, decision: BehaviorDecision }。背包無該商品時 FeedPet 回傳 rejected（不扣道具）；飽食度達 100 時仍允許餵食零食/罐頭增進心情（正常扣 1 份道具，飽食 Clamp 於 100）；冷卻期間重複摸頭回傳 applied（觸發反應）但不給積分與心情。指令數值與動畫解耦：GameCommand 效果在執行當下立即結算，BehaviorDecision 中斷不撤銷數值。 |
| **10** | 性格待機巡邏 (IDLE\_AND\_PATROL) | 無上述高優先級事件時執行 | 漫步 (walk \+ 問號尾)、母雞蹲 (cat\_loaf)、貴妃躺 (side\_sleep，需 mood \>= 80\) 輪替 | 每次動作持續 `4,000 ~ 8,000 ms`，可隨時被高優先級事件中斷。 |

### **8.B.2 性格與小怪癖干預公式**

2. **傲嬌先拒後吃 (basePersonality.tsundere \>= 70)：** 使用免費基礎乾糧 (free\_basic\_kibble) 時，有 45% 機率先播放撇頭嫌棄 (sniff\_and\_turn\_away \+ 💦 圖示，持續 2,000 ms)，隨後自動接續 eat 進食；使用主食罐或零食則直接進食。  
3. **慵懶臥息比例 (basePersonality.laziness)：** 待機狀態下選擇「母雞蹲或貴妃躺」的機率為 30% \+ (laziness \* 0.5)%。  
4. **小怪癖加法修正 (cursor\_hunter)：** 具備 cursor\_hunter 時，偵測畫面點擊靠近之基礎機率（20%）套用 additiveProbabilityModifier: \+0.30，變為 50%。

### **8.B.3 動畫資產矩陣分級 (Animation Asset Matrix)**

5. **Phase B / C 必備核心動畫 (P0)：** `idle`（待機）、`walk`（漫步）、`sit`（坐下）、`lie_down`（趴臥/母雞蹲）、`sleep`（熟睡）、`eat`（進食）、`pet`（被摸呼嚕）。  
6. **Phase C 進階體驗動畫 (P1)：** `drag`（Android 拎起）、`drop`（掉落）、`landing`（落地）、`turn_back_angry`（生氣）、`sniff_and_turn_away`（挑食撇頭）、`crouch_alert`（虛弱匍匐）。

## **8.C 經濟系統與數值平衡表 (Economy & TimeEngine Spec)**

### **8.C.1 時間步進計算順序與日夜區段公式 (TimeEngine Math)**

**【指令與時間模擬執行順序 (P1-4)】**  
所有具有遊戲數值效果的 `GameCommand`（如 FeedPet、PetCat 等）在執行前，必須先呼叫 `TimeEngine.advanceTo(commandTimestamp)` 將遊戲狀態推進至該指令的有效時間點，再執行指令並套用數值變化。  
**【連續閾值交點切割與時間區段計算 (Threshold Crossing Split)】**  
為徹底保證「一次補算 24 小時」與「任意時間點插入 GameCommand 分段補算」的浮點結果在數學上 100% 完全一致（誤差 \< 1e-9），TimeEngine 每次 advanceTo(now) 除按 0.25 小時（15 分鐘）基礎步長步進外，在推進時間區間 \[t\_start, t\_end\] 時，遇到以下任一狀態變化事件，必須建立精確交點切割 (Exact Boundary / Threshold Crossing Split)：  
1\. 日夜時段邊界（每日 07:00 與 23:00）。  
2\. 需求與健康狀態機門檻交點：Hunger \= 20、Hydration \= 20（扣血/額外懲罰門檻）、Hunger \= 50、Hydration \= 50（RECOVERY\_BOOST 門檻）、Hunger \= 60、Hydration \= 60、Mood \= 70（NATURAL\_RECOVERY 門檻）、Health \= 40（CRITICAL 門檻）與 Health \= 100（滿血完成門檻）。交點時間戳算式 t\_cross \= t\_start \+ ((currentValue \- targetThreshold) / decayRatePerHour) \* 3\_600\_000。  
3\. 玩家數值 GameCommand 發生時間與 Care Pause 啟用/解除時間。  
當執行數值 GameCommand 時，直接推進至 commandTimestamp 並按實際經過小時數 (actualHours \= segmentMs / 3\_600\_000) 結算，更新 lastSimulatedAt。

1. **步驟 1（判定區段時段係數）：** 若該區段之起始時間落於 07:00 \~ 23:00，時段係數 periodFactor \= 1.0；若落於 23:00 \~ 07:00，periodFactor \= 0.3。  
2. **步驟 2（更新 hunger 與 hydration）：**  
   1. hunger 扣除 4.0 \* periodFactor \* actualHours（最低 0）。  
   2. hydration 扣除 5.0 \* periodFactor \* actualHours（最低 0）。  
3. **步驟 3（更新 mood 與低需求額外懲罰 \- P0-1）：**  
   1. 基礎心情扣除 **1.25 \* periodFactor \* actualHours**（日間每小時 \-1.25，夜間每小時 \-0.375；每日 24 小時基礎心情總衰退為 23.0）。  
   2. 若步驟 2 更新後之 hunger \< 20 或 hydration \< 20，額外心情懲罰亦乘以時段係數，扣除 1.5 \* periodFactor \* actualHours（最低 0）。  
4. **步驟 4（更新 health 連鎖扣減）：**  
   1. 若步驟 2 更新後之 hunger \< 20 或 hydration \< 20，扣除 health 2.0 \* actualHours（不受夜間係數減免，最低保護下限為 10）。  
5. **步驟 5（執行健康救援狀態機 Recovery State Machine \- P0-2, P0-4, P1-6）：**  
   1. **1\. NORMAL ➔ CRITICAL：** 當 health \< 40 時觸發一次轉換，僅在此轉換瞬間執行 recoveryBoostAccumulatedHours \= 0。  
   2. **2\. CRITICAL ➔ RECOVERY\_BOOST：** 處於 CRITICAL 狀態下，當滿足 hunger \>= 50 && hydration \>= 50 且 recoveryBoostAccumulatedHours \< 2.0 時轉入，每小時恢復 \+10.0 health（按 actualHours 計算），並累加 recoveryBoostAccumulatedHours \+= actualHours（上限 2.0）。  
   3. **3\. RECOVERY\_BOOST 中斷與額度耗盡轉換：** 若加速期間 hunger \< 50 或 hydration \< 50，退回 CRITICAL 暫停回血但完整保留 recoveryBoostAccumulatedHours 已累計時數；當 recoveryBoostAccumulatedHours \>= 2.0（2 小時額度用滿）時，立即轉為 NATURAL\_RECOVERY（即使此時 health \< 40 或尚未滿足 60/60/70，也不退回 CRITICAL 且不重置已累計時數）。  
   4. **4\. NATURAL\_RECOVERY 回血與暫停：** 處於 NATURAL\_RECOVERY 下，滿足 hunger \>= 60 && hydration \>= 60 && mood \>= 70 時每小時恢復 \+2.5 health（按 actualHours 計算）；未滿足則暫停回血（維持 NATURAL\_RECOVERY 狀態，唯有恢復至 NORMAL 後未來再度跌破 40，才視為全新救援週期重新獲得 2.0 小時 Boost 額度）。  
   5. **5\. 全域滿血重置規則 (Global Full Health Reset)：** 無論處於 CRITICAL、RECOVERY\_BOOST 或 NATURAL\_RECOVERY 任一階段，當 health \>= 100 時，health 立即 Clamp 為 100，recoveryState 轉為 NORMAL 且 recoveryBoostAccumulatedHours \= 0。若同一時間點同時發生 health \< 40 與 hunger \< 20，依序先結算需求與 health 扣減，再觸發狀態機轉為 CRITICAL（僅在由 NORMAL 轉入 CRITICAL 的瞬間歸零累計時數）。  
   6. **6\. Care Pause 啟用前結算與凍結規則：** 啟用 Care Pause 前，必須先呼叫 TimeEngine.advanceTo(pauseStartTimestamp) 完成此前所有應有的離線模擬結算，再將 isCarePaused 設為 true 完全凍結 TimeEngine 衰退與 RecoveryState 累計時數；解除暫停時不補算暫停期間，直接將 lastSimulatedAt 更新為 resumeTimestamp 並執行一次狀態機檢查。

### **8.C.2 商品目錄與遊戲效果平衡表 (food-catalog.v1.json)**

所有商品依「品類基準 \+ 個性化微調」配置效果，品牌合作商品享有完全相同的平衡公式；Schema 納入 dataType ("real" \| "mock")、sourceName、sourceUrl 與 dataVerifiedAt 確保資訊真實性（UI 層對 mock 商品強制顯示「測試商品/模擬資料」標籤）：

| 商品 ID (itemId) | 商品名稱 | 類別 | 合作標示 (dataType) | 定價 (積分) | 真實營養與成分標示 (nutrition & ingredients) | 遊戲效果 (gameEffect) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| `free_basic_kibble` | 基礎日常乾糧 | 乾糧 | 免費基礎 (mock) | 0 (無限) | 粗蛋白 28% / 粗脂肪 12% / 水分 10%｜穀物、家禽副產品粉、維生素 | 飽食 \+25、水分 \-2、心情 \+2 |
| `free_fresh_water` | 過濾飲用水 | 飲水 | 免費基礎 (mock) | 0 (無限) | 純淨過濾水 100%（建議每日更換維持新鮮） | 飽食 \+0、水分 \+32、心情 \+2 |
| `shop_chicken_can` | 鮮燉嫩雞主食罐 | 主食罐 | 一般商品 (mock) | 60 | 粗蛋白 12.5% / 粗脂肪 4.5% / 水分 78%｜去骨雞胸肉、雞高湯、牛磺酸 | 飽食 \+25、水分 \+20、心情 \+5 |
| `shop_salmon_mousse` | 野生鮭魚慕斯罐 | 主食罐 | 一般商品 (mock) | 80 | 粗蛋白 11.8% / 粗脂肪 5.2% / 水分 80%｜野生鮭魚、魚油、洋車前子膳食纖維 | 飽食 \+25、水分 \+22、心情 \+8 |
| `brand_nu4pet_kitten` | 【陪心】屁孩罐－葡聚醣雞肉 | 主食罐 | 品牌合作 (mock) | 90 | 粗蛋白 13.0% / 粗脂肪 5.0% / 水分 79%｜台灣鮮雞肉、β-葡聚醣、魚油、維生素 E | 飽食 \+26、水分 \+21、心情 \+8 |
| `brand_ziwi_venison` | 【Ziwi 巔峰】紐西蘭鹿肉主食罐 | 主食罐 | 品牌合作 (mock) | 110 | 粗蛋白 12.0% / 粗脂肪 6.5% / 水分 78%｜紐西蘭放牧鹿肉、鹿肝、綠唇貝、海帶 | 飽食 \+28、水分 \+20、心情 \+9 |
| `brand_rc_hairball` | 【皇家】IH34 化毛配方乾糧 | 乾糧 | 品牌合作 (mock) | 95 | 粗蛋白 34% / 粗脂肪 15% / 水分 8%｜脫水禽肉蛋白、洋車前子殼、甜菜漿 | 飽食 \+28、水分 \-1、心情 \+6 |
| `shop_freeze_dried` | 凍乾原肉丁零食 | 零食 | 一般商品 (mock) | 50 | 粗蛋白 82% / 粗脂肪 8% / 水分 5%｜100% 冷凍乾燥雞胸肉丁 | 飽食 \+8、水分 \-1、心情 \+14 |

### **8.C.3 日常積分與「商品探索任務」平衡表 (EconomyEngine)**

開局預設 **200 積分**。dailyLimits 隨 GameState 一起保存，包含裝置本地日期 YYYY-MM-DD、lastClaimedTimestamp 與當日已發放獎勵事件 ID 集合（claimedRewardIds: string\[\]）；若偵測時鐘回撥 (now \< lastClaimedTimestamp) 或因跨時區旅行/手動調整致本地日期倒退 (currentLocalDate \< lastClaimedLocalDate)，拒絕重複發放獎勵。不補發未登入日之簽到或互動積分，高健康度日結獎勵僅於實際跨日結算當下觸發一次（在取得官方正式核實前所有商品一律標示 dataType: "mock"、sourceName: "MVP 模擬測試資料"、dataVerifiedAt: "2026-11-19"）：

6. **每日首次陪伴簽到：** `+40` 積分（每日上限 1 次）。  
7. **商品探索任務（不限品牌或一般商品，無廣告式領獎彈窗）：**  
   1. 每日首次開啟商店瀏覽：`+5` 積分。  
   2. 點開任一商品詳情頁並完整檢視營養成分分析（停留 `>= 3` 秒）：每款不重複商品 `+10` 積分（同商品當日重複開關不給分，每日探索獎勵上限 **30 積分**）。  
8. **有效摸頭與玩耍互動：** 每次有效互動 `+5` 積分與 `mood +3`（每日上限 `25` 積分）。  
9. **高健康度日結獎勵：** 跨日結算時若 `health >= 90` 且 `mood >= 80`，獎勵 `+25` 積分。

## **8.D UI/UX 介面與操作動線 (UI/UX Flow)**

1. **開局領養流程（MVP 限制單貓制）：**  
   1. 選擇父母品種（支援無限次重新隨機盲盒搭配，確定領養後鎖定外觀與 `seed`） ➔ 基因融合誕生過場 ➔ 預覽外觀並命名（1\~12 字元） ➔ 雙平台權限引導（Android 引導懸浮窗授權；iOS 引導桌面 Widget 教學）。  
2. **主房間與控制選項 (HomeScreen)：**  
   1. 頂部顯示四大狀態條與積分；底部為「背包餵食」、「商店」、「圖鑑」與「設定」。  
   2. **照顧保護與顯示設定：** 支援「暫時隱藏懸浮窗至通知列」、「貓咪顯示尺寸調整（小/中/大）」以及「照護暫停保護模式 (Care Pause)」。  
3. **商店與商品詳情頁 (ShopScreen & ProductDetailModal)：**  
   1. 列表支援分類篩選與「品牌合作 / 測試商品」清晰標示；詳情頁完整呈現「真實品牌、原料、粗蛋白/脂肪/水分分析、資料來源與更新日期」，以及獨立區塊的「遊戲回復數值」與「購買放入背包」按鈕。

## **8.E 存檔架構、遷移規範與自動化驗收劇本**

### **8.E.1 雙階段存檔與時間戳單一權威規範 (Save Migration)**

1. **時間戳單一權威來源：** PetState.lastSimulatedAt 為遊戲模擬時間的唯一權威來源；資料庫外層之 created\_at 與 updated\_at 僅供維運追蹤，寫入時由 Adapter 注入之 Clock 同步填入，不得與 PetState.lastSimulatedAt 產生雙頭馬車。  
2. **Phase A (Windows 本機)：** 採用防損 JSON 寫檔流程（先寫入 save.tmp ➔ 驗證 JSON 結構 ➔ 備份舊檔為 save.bak ➔ 原子替換為 save.json）。  
3. **Phase C (手機端 SQLite)：**  
   1. `game_saves` 表包含：id、schema\_version、game\_core\_version、pet\_state\_json（含 genetics、appearance、basePersonality、quirks、memories、needs、health、recoveryState、isCarePaused、coins、inventory）、daily\_limits\_json、created\_at、updated\_at。  
   2. `game_events` 表僅作為行為追蹤與除錯日誌（非主存檔來源），定期清理超過 30 天之舊紀錄。  
4. **存檔升級遷移規則 (Migration)：** 每次 App 升級偵測到 `schema_version` 落後時，先自動備份原始存檔再執行漸進式遷移腳本（`v1 -> v2`），新增欄位必須填入預設值，**嚴禁因版本更新而重置或重新生成既有貓咪的外觀與個性**；若遷移失敗則安全回退至備份檔。

### **8.E.2 核心自動化驗收測試案例 (Vitest Acceptance Criteria)**

RD 完成 Phase A 時必須通過以下 9 項核心測試與照護負擔驗證：

1. **TC-01（基因確定性與 Golden Test）：** 固定 seed: "test-seed-001"、prngAlgorithmVersion: "mulberry32-v1"、geneticsVersion: "1.0.0"，連續生成 10 次結果 100% 一致；對於不同 seed（"test-seed-001" vs "test-seed-002"），驗證 PRNG 抽取序列確實不同並比對預先固定的 Golden Test 預期輸出，不要求不同 Seed 必須產出不同外觀。  
2. **TC-02（特殊骨架雙親與跨品種統計分布）：** 使用獨立固定種子序列 seed\_0001 \~ seed\_1000（每次獨立初始化 PRNG 確保狀態隔離）：munchkin \+ maine\_coon 驗證 short\_legs 佔 50% ± 5%、giant\_fluffy 佔 25% ± 5%、normal 佔 25% ± 5%（允許 short\_legs 與 giant\_fluffy 共存）；munchkin \+ munchkin 驗證 short\_legs 佔 75% ± 5%、normal 佔 25% ± 5%；exotic \+ domestic 驗證 flat\_face 佔 45% ± 5%、standard\_round 佔 55% ± 5%；exotic \+ exotic 驗證 flat\_face 佔 80% ± 5%、standard\_round 佔 20% ± 5%。  
3. **TC-03（時間步進、日夜邊界切割與分段一致性）：** 測試 22:50 \~ 23:05 跨夜邊界自動拆分為 10 分鐘日間（1.0）+ 5 分鐘夜間（0.3）之精確結算驗證；06:45 \~ 07:15 跨日、未滿 15 分鐘零頭累計、24 小時一次補算與 4 次 × 6 小時分段補算結果誤差 \< 1e-6，以及時鐘回撥後恢復正常時間的補算正確性。  
4. **TC-04（健康救援狀態機完整邊界）：** 驗證 health \= 40 不進 CRITICAL、health \= 39 進入 CRITICAL（僅轉換時重置累計時數）；餵食補水達 50/50 進入 RECOVERY\_BOOST（按 actualHours 計算）；中途飢餓跌至 49 暫停並保留已累計時數，重新餵食後繼續累計；當 recoveryBoostAccumulatedHours \>= 2.0 用滿後，即使 health \= 30（仍 \< 40）且未達 60/60/70 亦正確轉為 NATURAL\_RECOVERY 等待照護，絕不卡死或重置累計時數；單步跨越 40 與上限 100 Clamp 驗證。  
5. **TC-05（行為仲裁、滑動窗口與冷卻邊界）：** 驗證 Priority 100 \> 90 \> 80 \> 60 \> 40 \> 10；同觸發 Priority 100 多項條件時 iconIds 合併且主動畫依固定序執行；POKE 10 秒滑動窗口第 4 次觸發 Priority 90，15 秒冷卻期間再 Poke 不延長冷卻且餵食仍有效；eat 跨過 23:00 不被睡眠中斷。  
6. **TC-06（傲嬌干預與記憶計算）：** tsundere \>= 70 餵免費乾糧依機率觸發撇頭再吃；驗證最近 7 天互動佔 70% 與歷史 30% 的人格計算公式。  
7. **TC-07（商品探索防重與跨日重置）：** 開局 200 積分，同商品同日重複開啟不給分、跨午夜（23:59 ➔ 00:01）正確重置上限、單日探索上限 30 積分、購買與背包使用扣減及 100 上限 Clamp。  
8. **TC-08（365 天長時穩定性模擬）：** 連續模擬 365 天隨機玩家行為（含 Care Pause 開關、長期未登入、只用免費糧水），驗證無 NaN、Infinity 或負數積分，且存檔序列化與還原一致。  
9. **TC-09（每日照護負擔量化驗收標準）：** 初始數值 hunger \= 80, hydration \= 80, mood \= 85, health \= 100，僅使用免費乾糧（+15 hunger, \-1 hydration, \+1 mood）、免費飲水（+25 hydration, \+2 mood）與每日摸頭/玩耍（mood \+3）：  
   1. **情境 A（每日早晚登入 2 次，每次餵 2 糧 \+ 2 水 \+ 2 次互動）：** 每日總補充 Hunger \+100、Hydration \+120、Mood \+28，每日結束時 Hunger \>= 80、Hydration \>= 85、Mood \>= 85，Health 維持 100（零扣血）。  
   2. **情境 B（每日僅登入 1 次，餵 3 糧 \+ 3 水 \+ 3 次互動）：** 每日總補充 Hunger \+75、Hydration \+90、Mood \+21；離線末段允許 Mood 緩慢下降 (50\~65) 且離線末段觸發輕微低需求扣血 (Health 降至 80\~90)，登入配合商店/陪伴使 Mood \>= 70 啟動 NATURAL\_RECOVERY (+2.5/hr) 回滿；連續 7 日模擬 Health 始終 \>= 75 且絕不進入 CRITICAL (\< 40)。  
   3. **情境 C（每 3 日登入 1 次，救援與商店/陪伴恢復驗收，兩階段驗收）：** 登入時 Hunger=0, Hydration=0, Mood=0, Health=10, CRITICAL。玩家餵 3 糧 (+75, \-6, \+6) \+ 3 水 (+96, \+6) \+ 2 凍乾零食 (+16, \-2, \+28) \+ 10 次互動 (+30 Mood)，照護後立即為 Hunger=91 (\>=60), Hydration=88 (\>=60), Mood=70 (\>=70)。階段 1：2 小時內經 RECOVERY\_BOOST (+10.0/hr) 從 Health=10 恢復至 30.0。階段 2：轉入 NATURAL\_RECOVERY (+2.5/hr)，於 28 小時內穩定回血至 100。

# **9\. 產品範圍與非目標 (Scope & Non-Goals)**

## **9.1 MVP 明確納入 (Phase A \~ D)**

5. 單隻貓咪領養與 7 層確定性外觀生成、三層性格行為決策、0.25 小時步進離線時間差補算、健康救援狀態機、無懲罰 Care Pause 照護暫停模式、防損本機存檔與 Save Migration、Android 全域懸浮窗、iOS App 內養成與狀態 Widget、積分商店與商品營養分析探索。

## **9.2 MVP 明確排除 (Non-Goals，保留介面但延後至 Phase E)**

6. 多隻貓咪同時飼養、即時背景常駐耗電服務、伺服器端防作弊結算、好友即時連線與串門子、GPS/LBS 戶外散步與跳蚤系統、自動截圖相簿、LLM 自由對話與付費月卡。

# **10\. 隱私、權限與錯誤處理降級規範 (Privacy & Fallback)**

## **10.1 最小權限與資料隱私規範**

7. Android 僅於完成領養後引導授權 SYSTEM\_ALERT\_WINDOW（拒絕授權仍可於 App 內正常遊玩）；計步器與定位權限於 MVP 階段完全不申請。  
8. 問卷與統計僅於使用者明確同意下收集不可識別個人的匿名彙總數據，嚴禁販售個別使用者行為軌跡或定位紀錄。

## **10.2 異常降級與 Fallback 策略**

1. **遠端設定檔失敗：** 網路中斷或遠端 JSON 驗證失敗時，無縫使用本機快取或內建 `v1` 預設設定檔。  
   2. **系統時間倒退異常：** 若偵測到裝置時間倒退（now \< lastSimulatedAt），TimeEngine 忽略負時間差（Δt \= 0）並記錄異常時間事件，**嚴禁覆寫或修改原本的 lastSimulatedAt**，待裝置時間恢復正常區間後再行補算，防止時間回撥造成異常暴增補算。  
   3. **權限撤銷：** 若使用者於系統設定撤銷 Android 懸浮窗權限，App 自動切換為「App 內模式」並保存進度，不產生閃退。

# **11\. 版本控制與開發工作拆分 (Versioning & Sprint Plan)**

## **11.1 獨立版本號管理**

9. 系統將 gameCoreVersion、prngAlgorithmVersion、geneticsVersion、contentVersion 與 saveSchemaVersion 獨立分開管理，確保更新商品目錄或調整數值平衡時，不會破壞既有貓咪外觀或玩家存檔。

## **11.2 Sprint 執行順序**

10. **Sprint 1（Phase A \- Windows Core 骨架與基因引擎）：** 建立 pnpm Monorepo、contracts、config-loader（Local 模式）、PetState 模型與 GeneticsEngine（通過 TC-01、TC-02）。  
11. **Sprint 2（Phase A \- 時間、健康救援與經濟引擎）：** 實作 TimeEngine（0.25 小時區段步進與時間倒退保護）、健康救援狀態機、EconomyEngine、背包系統與防損 JSON 存檔（通過 TC-03、TC-04、TC-07）。  
12. **Sprint 3（Phase A \- 行為決策、CLI 驗收與長時模擬）：** 實作 BehaviorEngine 三層決策與同級仲裁、CLI 互動操作介面、365 天長時模擬與每日照護負擔驗證（通過 TC-05、TC-06、TC-08、TC-09）。  
13. **Sprint 4（Phase B \- 雙平台渲染 PoC）：** 建立單隻測試貓，驗證 React Native、Android Kotlin 懸浮窗與 iOS SwiftUI Widget 消費同一份 PetViewModel 之一致性。

