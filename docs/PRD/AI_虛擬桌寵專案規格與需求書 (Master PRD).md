# **拾暖伴伴 (shinuan) AI 虛擬桌寵專案規格與需求書 (Master PRD v2.1.28)**

**【版本資訊】** Master PRD v2.1.28（2026-11-20 凍結候選稿）  
**【本次修訂重點】**  
1\. 移除 8.E.2 TC-09 子情境 B-2 末尾殘留的舊版「長期僅於 08:00 餵食飲水但不陪伴情境驗證」段落（P2-01），避免與子情境 B-3 之精確交點 10 日低心情劇本並存；完整保留 B-2 原本 60 日低健康救援計算與 B-3 完整 10 日劇本、Fixture、精確交點及所有數值不變。  
**【核心規則保留聲明】** 本次修訂僅移除 B-2 殘留之舊版 10 日劇本並同步版本號，未更動既有 MoodHealthPenalty 公式、Recovery 數值、Health 40 / 10 保護規則、商品效果與其他各項自動化驗收測試預期結果。

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
* **MoodHealthPenalty：** 獨立的低心情健康扣減計算器（Mood 20\~39 時，below40Hours 精確跨越 72.0 小時後〔嚴格 \> 72.0〕，從該交點之後的時間區段開始套用 \-0.05/hr；Mood 0\~19 時，below20Hours 精確跨越 48.0 小時後〔嚴格 \> 48.0〕，從該交點之後的時間區段開始套用 \-0.1/hr 取代 \-0.05/hr 不疊加，若 below20Hours \<= 48.0 且 below40Hours \> 72.0 則仍承接維持 \-0.05/hr，其餘為 0/hr；全程受 Health \>= 40 安全地板保護）。  
* **LowMoodDuration：** 記錄寵物連續處於低心情區間的累積小時數物件 { below40Hours: number; below20Hours: number }（below40Hours 累計 Mood \< 40 的連續小時數，below20Hours 累計 Mood \< 20 的連續小時數），隨存檔持久化並在 Mood \>= 40 時立即歸零。  
* **RecoveryMultiplier：** NATURAL\_RECOVERY 階段依據 Mood 區間套用的分級恢復速度倍率（Mood \>= 70 為 \+2.5/hr、Mood 40\~69 為 \+1.25/hr、Mood \< 40 暫停）。  
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

* **對一般大眾 (User)：** 解決了「想養寵物但環境不允許」或「渴望日常陪伴」的痛點。這不是一款需要死板打卡的手遊，而是透過連動手機真實的時間與電量等環境資訊（MVP 階段先提供時間與電量情境；計步器與戶外散步列為 Phase E 後續擴充，MVP 階段不申請計步器權限），讓你感覺手機裡真的住著一隻有情緒的貓咪。  
* **對贊助廠商 (Brand)：** 解決了「寵物廣告轉換率低、惹人厭」的痛點。我們將真實的貓砂、主食罐置入遊戲商店中，玩家在選購飼料時自然閱讀真實成分與營養介紹，廠商付費取得「被精準受眾看見的機會」，實現零反感的**內容型原生廣告 (Native Advertising)** 與導購平台 (B2B2C)。

**【技術與發展藍圖】**  
採用「平台解耦」與「離線優先」架構，以純 TypeScript 打造跨平台遊戲核心（Game Core），先於 Windows 本機完成核心規則驗證與自動化測試，再分別透過原生適配器（Adapter）落地於 Android 全域懸浮窗與 iOS 桌面 Widget / App 內互動。初期依賴 2D 肢體動畫與腳本狀態機以零伺服器成本驗證市場；專案的終極目標，是在商業模式與核心玩法驗證成功後，於未來階段接入 LLM（大型語言模型）賦予貓咪深度對話能力（LLM 僅負責語言與情境表達，絕不直接修改核心數值或跳過 Game Core 規則）。

# **2\. 商業模式深度解析 (內容型原生廣告與 O2O 導購)**

有別於傳統手遊強迫觀看無關廣告或用數值懲罰逼迫課金，本專案的核心是「將商業產品化為遊戲商店的真實內容」，讓選擇權完全回歸玩家。

## **2.1 無反感原生廣告的五大核心原則**

1. **商品是選擇，不是強迫（免費照護保障）：** 玩家可自由使用基礎免費食物與飲水，足以持續維持貓咪健康與基本需求，絕不會因為沒買合作品牌商品而導致貓咪生病或受到不可逆懲罰。每日一次登入的標準免費照護量（3 份免費乾糧 \+ 4 份免費飲水 \+ 3 次互動），必須足以讓寵物在長期正常照護情境下維持基本健康，不得要求玩家透過付費商品、觀看廣告或提高登入頻率，才能避免寵物健康持續惡化。  
2. **廣告與一般商品並存：** 商店內同時提供一般遊戲道具與合作廠商商品，合作商品清楚標示「品牌合作 / 贊助內容」，維持畫面清爽。  
3. **商品介紹具備真實知識價值：** 點進飼料可查看真實品牌、原料成分、蛋白質/脂肪/水分營養分析、適用對象、資料來源及資料更新日期；MVP 階段之模擬商品須於 UI 明確標示「測試商品/模擬資料」。  
4. **品牌公平與平衡原則：** 品牌合作商品不得因贊助關係取得額外數值加成。所有商品的遊戲回復效果均依固定的遊戲平衡公式配置（例如主食罐補水高、乾糧飽食高、零食心情高），廠商購買的是「品牌曝光與虛擬櫃位」，贊助費不得直接影響商品數值。MVP 階段不採用自動數值公式，而是採「預先定義之品類定價級距上限區間（見 8.C.2）＋ PO 核准之固定商品效果白名單（food-catalog.v1.json）」；同一級距內允許依真實營養特性於合法區間內微調，但價格級距上限為硬性天花板，不得跨級突破，且一般商品與品牌合作商品一律適用同一張級距表，嚴禁以品牌合作身分、贊助金額或曝光合約作為效果參數。  
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
   2. 第二階段變異路徑（佔該圖層 20%）：針對設有稀有突變之 4 個圖層（ears、pupilColor、coatPattern、tail），變異路徑內 75%（即該圖層機率 15%）從該圖層合法通用特徵白名單隨機重組、25%（即該圖層機率 5%）觸發該圖層之稀有突變（折耳、異色瞳、星空斑紋、麒麟尾）；其餘無稀有突變之 3 個圖層（head、eyes、body），變異路徑 100% 從該圖層合法通用特徵白名單隨機重組。各圖層彼此獨立，單隻貓至少出現一種稀有突變之機率為 1 \- (1 \- 0.05)^4 \&approx; 18.55%，且允許同時出現多種稀有突變。  
   3. 若父母在某圖層特徵相同，父系與母系抽選結果均視為該特徵，機率不合併至突變池，突變仍依獨立機率抽選。  
4. **Seed 與版本永久鎖定規範：**  
   1. seed 僅用於生成初始遺傳結果，不作為既有寵物外觀的唯一保存依據。  
   2. 每隻寵物生成後，必須永久保存完整的最終 appearance、colorParams、geneticsVersion、prngAlgorithmVersion 與 rulesVersion。  
   3. **任何後續 Genetics、PRNG、Trait Pool 或 Compatibility Rule 更新，均不得重新生成或覆寫既有寵物的外觀。**  
5. **GrowthStage MVP 規範：** MVP 初期不實作實際年齡成長與外觀變化。ageInDays 僅作為遊戲累計天數資料預留，不影響外觀、動畫、健康或人格數值；growthStage 初期固定為 "adult"，保留未來版本擴充。

## **3.2 三層行為決策模型 (Behavior Decision Model)**

為避免「永久人格」與「暫時生理狀態」混淆，性格與行為拆分為三層獨立架構：

* **第一層：基礎人格 (BasePersonality)：** 由品種天性（權重 60%）與長期後天養成記憶（權重 40%）構成，決定 affection（親人）、laziness（慵懶）、tsundere（傲嬌）與 curiosity（好奇）（範圍 0\~100）。  
  * **後天養成記憶計算公式 (四維度獨立程式級數學定義)：** 統計視窗固定為「過去 30 日滾動視窗」（recent7Days 為最近 7 日，prior23Days 為第 8\~30 日）。若某維度在過去 30 日滾動視窗內之有效互動次數為 0（無有效歷史資料），則該維度之記憶分數直接設為該品種天性基準 memoryScore \= breedBase（使 finalPersonality \= round(breedBase \* 0.6 \+ breedBase \* 0.4) \= breedBase，精確回歸品種天性基準）；唯有當過去 30 日內存在有效互動紀錄時，才套用以下加權與線性插值公式：  
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

* **日常照護：** 每日免費提供基礎乾糧與過濾飲用水（0 積分、不設付費門檻，支援正常早晚多次登入與緊急救援補給食水）；以「每日一次登入之標準免費照護方案（3 份免費乾糧 \+ 4 份免費飲水 \+ 3 次摸頭與玩耍互動）」作為最低照護負擔基準，即足以長期維持貓咪基本健康，不得將免費照護設計成付費門檻，亦不得因未購買品牌商品而受到不可逆懲罰。  
* **原生廣告商店：** 玩家累積積分後可選購一般罐頭/零食或品牌合作商品，點擊商品詳情可查看真實成分與營養分析；購買後的商品進入背包，由玩家自主選擇使用時機。

# **5\. 技術架構與工程原則 (Monorepo \+ Platform Decoupling)**

## **5.1 嚴格平台解耦與開發階段規範**

1. **嚴格平台解耦 (Platform Decoupling)：** Game Core 必須為純 TypeScript 跨平台模組，所有時間、隨機數、存檔、外部設定與平台資訊均透過依賴注入（Dependency Injection）或外部輸入提供，必須能在 Windows Node.js 環境獨立執行與測試。  
2. **【Phase A 必須實作】：**  
   1. 純 TypeScript Game Core。  
   2. 本機 Config Loader（讀取 content/ 目錄及其子目錄中的版本化 JSON 設定檔，負責檔案讀取、完整性檢查與 Schema 驗證；Game Core 只接受已通過驗證的設定物件）。  
3. Clock / RandomGenerator 依賴注入。  
   1. 防損 JSON 存檔。  
4. Vitest 單元、整合及長時間模擬驗收測試。  
5. **【Phase A 預留介面】：**  
   1. GameRepository 介面。  
   2. PetViewModel 契約。  
   3. GameEvent 結構。  
6. **【Phase A 不實作 / 後續階段】：**  
   1. 遠端設定檔下載及更新機制（Phase D 導入）。  
   2. Android WindowManager 原生模組。  
   3. iOS WidgetKit。  
   4. SQLite。  
   5. Google Drive 備份。  
   6. Firebase 串門子與社交信箱。  
   7. LLM API。  
   8. **明確禁止在 Game Core 內使用：**`window` / `document`（Browser 專屬）、`fs` / `path` / `process`（Node.js 專屬）、`Date.now()`（導致時間不可控）、`Math.random()`（破壞固定 Seed 重現性）、React / React Native、Kotlin / Swift API 以及直接發起網路請求。  
7. **設定檔外部載入與驗證 (Config Loader)：**  
   1. **Phase A（Windows 本機驗證）：** 僅實作本機 JSON 載入及 Schema 驗證（config-loader 直接讀取專案內 content/ 目錄及其子目錄（genetics/、traits/、behavior-rules/、products/）之版本化靜態 JSON，執行 Schema 驗證後注入 Game Core），不實作遠端設定下載。Game Core 不負責讀取檔案或解析外部資料來源。  
   2. **Phase B \~ C：** 不要求實作 Remote Config Provider。  
   3. **Phase D（配合商店與品牌商品目錄導入）：** 實作 Remote Config Provider，必須支援遠端版本檢查、設定檔下載、本機快取、Schema 驗證及載入失敗時的 Fallback 機制；遠端服務平台不預先指定，由實際部署架構決定。  
      1. 遠端設定來源不得與 Game Core 耦合。  
      2. Game Core 只接受已通過 Schema 驗證的設定物件。  
      3. Remote Config Provider 負責處理遠端資料來源、版本檢查、下載及快取。  
      4. 實際遠端服務平台由未來部署架構決定，不在本階段指定 GitLab Raw、GitHub Raw 或其他特定服務。  
      5. 更換遠端 Provider 不得要求修改 Game Core 的遊戲規則或計算邏輯。  
8. **離線優先 (Offline-first) 與免防作弊設計：** 個人陪伴養成無 PvP 與排行榜，不建置伺服器結算；僅保留存檔版本遷移（Migration）、格式防損驗證與本機每日領取防重機制。  
9. **呈現與存檔分離：** 存檔僅保存基因、數值、記憶與時間戳，不儲存動畫影格；統一由 BehaviorEngine 輸出單一標準結構 BehaviorDecision，再由 GameEngine 轉換為跨平台唯讀契約 PetViewModel 交由各平台 Renderer 呈現。

## **5.2 技術選型與跨平台呈現契約 (PetViewModel)**

* 

| 架構層級 | 技術選型 | 開發階段與責任 |
| :---- | :---- | :---- |
| 遊戲核心 (100% 共用) | 純 TypeScript (packages/game-core) | **Phase A 必須實作。**負責品種傳承、性格、需求與健康平衡、TimeEngine、BehaviorEngine 與經濟規則，零平台依賴。 |
| 設定載入與契約層 | TypeScript (config-loader / contracts) | **Phase A** 實作本機 JSON 載入、Schema 驗證與跨模組資料契約。遠端 Provider 配合商店與品牌商品目錄於 **Phase D** 導入（Phase B \~ C 不要求實作）。 |
| 跨平台 UI 層 | React Native \+ TypeScript | **Phase B** 建立跨平台 UI 骨架，包含 App 主房間、領養、設定介面，以及商店與商品成分頁之靜態 UI Prototype（僅使用固定 Mock Data 驗證版面與動線，不含實際購買、積分與經濟結算）；正式商店功能（商品目錄、積分、購買、背包、商品探索任務與 Remote Config）統一於 **Phase D** 實作。 |
| Android 原生適配器 | Kotlin (WindowManager) | **Phase B** 驗證 Android 透明懸浮窗、生命週期管理、觸控拖曳、重力物理與系統電量事件回傳。 |
| iOS 原生適配器 | Swift / SwiftUI (WidgetKit) | **Phase B** 驗證桌面 Widget 狀態展示、App Intents 快捷操作佇列與 App Group 資料共享。 |
| 本機與雲端資料流 | JSON (Phase A) / SQLite (Mobile) | **Phase A** 使用防損 JSON 存檔；Mobile 階段導入 SQLite；後續才評估 Google Drive 個人備份與 Firebase 社交信箱。 |

* **跨平台統一呈現契約 (PetViewModel Schema)：**  
  * 平台端嚴禁自行計算需求或決定行為，一律唯讀消費 PetViewModel 完整欄位契約：  
    * `petId` (string)、`name` (string)、`appearance` (PetAppearance)  
    * `needs` ({ hunger, hydration, mood, health })  
    * `behavior` ({ animationId: AnimationId, tailState: TailState, iconIds: IconId\[\], soundId: SoundId \| null, durationMs: number })（註：interruptible 與 cooldownMs 僅供 Game Core 內部狀態機仲裁，不拋出給 ViewModel）  
    * `display` ({ isSleeping: boolean, isHealthCritical: boolean, recoveryState: RecoveryState, isCarePaused: boolean })（其中 isSleeping \= (behavior.animationId \=== "sleep")；isHealthCritical \= (pet.health \< 40)；recoveryState \= pet.recoveryState；isCarePaused \= pet.isCarePaused）  
    * `updatedAt` (number，Unix timestamp ms)

## **5.3 Monorepo 專案目錄結構**

本專案正式 Repository 名稱為 shinuan，本機專案根目錄及 GitHub Repository 均以 shinuan/ 為根目錄。以下為專案完整目標目錄結構，各目錄依照 Phase A、Phase B 及 Future Phase 逐步建立，不要求初期一次建立所有目錄或實作所有功能：

* **shinuan/** (根目錄)  
  * `AGENTS.md`、`CLAUDE.md`、`package.json`、`pnpm-workspace.yaml`、`pnpm-lock.yaml`、`tsconfig.base.json`  
  * **.claude/agents/：**architect.md、developer.md、qa.md、reviewer.md  
  * **.github/：**copilot-instructions.md  
  * **docs/：**PRD/、architecture/、decisions/、acceptance/、秘書工作.md  
  * **tasks/：**backlog/、in-progress/、review/、completed/、blocked/  
  * **packages/：**  
    * **game-core/：**純 TypeScript 遊戲核心（domain/、engine/、ports/）  
    * **contracts/：**跨模組資料契約  
    * **config-loader/：**設定檔載入與 Schema 驗證模組  
  * **apps/：**  
    * **local-cli/：**Phase A Windows 本機 CLI 測試介面  
    * **mobile/：**React Native 手機端專案（android/、ios/、src/）  
  * **content/：**版本化靜態設定檔（genetics/、traits/、behavior-rules/、products/）  
  * **assets/：**貓咪圖層素材、動畫、圖示與音效（依 Phase B 需求逐步建立）  
  * **tests/：**unit/、integration/、simulation/  
  * **test-results/：**測試執行結果、模擬輸出、QA 報告及驗收證據

### **目錄用途及開發階段說明：**

1. **根目錄工程設定：** package.json（Monorepo 根目錄套件及工作指令）、pnpm-workspace.yaml（pnpm Workspace 套件管理）、pnpm-lock.yaml（依賴版本鎖定）、tsconfig.base.json（共用 TypeScript 編譯設定）、AGENTS.md（跨 AI Agent 通用工程規範）、CLAUDE.md（Claude 專用專案工作規範）。  
2. **AI Agent 設定：** .claude/agents/（Architect、Developer、QA、Reviewer 角色設定及工作規範）、.github/copilot-instructions.md（GitHub Copilot 專案協作指引）。  
3. **文件管理：** docs/PRD/（正式產品需求文件及版本紀錄）、docs/architecture/（系統架構及技術設計文件）、docs/decisions/（重大技術決策及決策紀錄）、docs/acceptance/（功能驗收條件及測試規格）、docs/秘書工作.md（工作流程參考筆記，不取代正式 PRD、AGENTS.md 或 CLAUDE.md）。  
4. **任務管理：** tasks/backlog/（尚未開始）、tasks/in-progress/（正在執行）、tasks/review/（等待 QA 或 Reviewer 審查）、tasks/completed/（已完成並通過驗收）、tasks/blocked/（因需求、技術或外部依賴而暫停）。  
5. **核心套件 (packages/)：** game-core/（純 TypeScript 遊戲核心，包含 domain、engine 與 ports）、contracts/（跨模組資料契約，包含 Command、Event、ViewModel 與 Save Schema）、config-loader/（設定檔載入、Schema 驗證及 Fallback 模組）。  
6. **應用程式 (apps/)：** local-cli/（Phase A Windows 本機 CLI 測試介面）、mobile/（Phase B 後續建立的 React Native 手機端專案，包含 Android、iOS 與共用 src）。  
7. **設定與素材 (content/ 與 assets/)：** content/ 放置版本化靜態設定檔（Config Loader 載入範圍應涵蓋 genetics/、traits/、behavior-rules/、products/ 等子目錄，不得僅掃描 content/ 根目錄）；assets/ 依 Phase B 需求逐步建立貓咪圖層素材、動畫、圖示及音效。  
8. **測試管理（測試程式與測試結果分離）：** 統一於根目錄 tests/ 保存可執行的 Vitest 測試程式碼（tests/unit/ 單元測試、tests/integration/ 跨模組整合測試、tests/simulation/ 365 天長時間模擬及其他時間序列驗證）；test-results/ 專責保存測試執行結果、模擬輸出、QA 報告及驗收證據，不得將測試報告混入測試程式碼目錄。  
9. **Phase A 初期目錄建立範圍：** 優先建立根目錄工程設定、packages/game-core/、packages/contracts/、packages/config-loader/、apps/local-cli/、content/、tests/、test-results/、docs/ 及必要的 AI Agent 工作規範。apps/mobile/、完整 assets/ 及其他 Future Phase 目錄應依照實際開發階段建立，不得因目錄樹列出而提前擴大 Phase A 工作範圍。

## **5.4 AI 協作與工程管理規範**

### **5.4.1 管理原則**

本專案採用 AI 輔助開發及角色分工模式，以 PO 為最終需求決策與驗收負責人，由 AI Manager 負責任務拆解、派工、進度管理及成果整合。AI Agent 的工作必須遵循最新核准的 Master PRD、工程規範及任務驗收條件，不得以 AI 自行推測取代已核准的產品規格。

### **5.4.2 角色與職責**

1. **PO (Product Owner)：** 負責產品需求、功能優先順序及範圍決策；核准重大架構、遊戲規則及規格變更；決定是否接受新增功能及技術債；負責最終驗收。  
2. **AI Manager (秘書)：** 接收 PO 提出的需求；讀取 Master PRD、工程規範及既有任務紀錄；將需求拆解為可執行的開發任務；根據任務性質指派 Architect、Developer、QA 及 Reviewer；管理任務狀態、依賴關係、交付成果及阻塞問題；整合各角色成果並向 PO 回報；不得自行修改已核准的核心遊戲規則、數值及重大架構決策。AI Manager 為主要 Claude Agent，其工作規範由根目錄 CLAUDE.md 定義；.claude/agents/ 僅存放由 AI Manager 依任務需求呼叫的 Subagents（Architect、Developer、QA、Reviewer），AI Manager 不需要額外建立 manager.md Subagent。  
3. **Architect (架構師)：** 負責技術架構分析及設計；評估技術方案、模組依賴與實作風險；提供架構決策建議及必要文件；涉及重大架構變更時，必須先提交 PO 核准。  
4. **Developer (工程師)：** 依照已核准任務修改 VS Code 專案程式碼；遵循既有架構、型別、程式碼風格及測試規範；執行必要的開發階段驗證；提供修改檔案、實作內容及交付紀錄；不得自行擴大任務範圍或變更未授權的核心規則。  
5. **QA (品質驗證)：** 獨立執行單元、整合及長時間模擬測試；依照任務驗收條件驗證實際執行結果；提供可追溯的測試紀錄及測試結果；發現問題時建立缺陷紀錄並退回處理；QA 不得自行修改正式 Game Core 程式碼（必要測試程式及測試資料的調整應限於核准的測試範圍）；Developer 自行執行的開發驗證不得取代 QA 獨立驗收。Phase A 初期 QA 獨立性採流程與職責隔離，不代表作業系統權限或檔案系統隔離；QA 必須依據核准的驗收條件自行執行測試，不得修改正式 Game Core 程式碼，亦不得僅引用 Developer 的測試結果作為通過證據；未來如有必要，可透過 Git Worktree 或其他隔離機制強化 QA 作業獨立性。  
6. **Reviewer (審查者)：** 檢查程式碼品質、架構一致性及潛在風險；檢查修改是否符合 PRD、工程規範及任務驗收條件；確認測試覆蓋及錯誤處理是否足夠；提供審查結論及必要修改意見。

### **5.4.3 標準 AI 工作流程**

PO 提出需求 ➔ AI Manager 分析需求及拆解任務 ➔ Architect 技術分析（必要時） ➔ PO 核准重大設計或範圍變更（必要時） ➔ Developer 執行程式碼修改 ➔ QA 獨立執行測試 ➔ Reviewer 審查程式碼及測試結果 ➔ AI Manager 整合成果及更新任務狀態 ➔ PO 最終驗收。

7. 任務如不涉及架構變更，可由 AI Manager 直接指派 Developer，不必強制每個任務都經過 Architect。  
8. QA 與 Reviewer 必須保持驗證及審查職責，不得因 Developer 宣稱已完成而省略驗收。

### **5.4.4 任務管理規範**

所有正式開發任務必須具有以下資訊：任務識別碼（Task ID）、任務名稱及目標、相關 PRD 章節或規格條款、修改範圍及禁止修改範圍、執行角色及負責人、前置依賴及相關任務、明確驗收條件、測試要求、執行結果及交付紀錄。

| 狀態 | 說明 |
| :---- | :---- |
| Backlog | 尚未開始 |
| In Progress | 正在執行 |
| Review | 等待測試或審查 |
| Completed | 已通過驗收 |
| Blocked | 因問題或依賴而阻塞 |

任務狀態不得僅因程式碼已修改而標記為 Completed，必須完成規定的 QA 驗證及 Reviewer 審查。

### **5.4.5 變更治理及權限邊界**

9. **PRD 保護：** AI Agent 不得自行修改已核准的核心遊戲數值、公式、基因機率及健康保護規則。需要調整時，必須提出變更理由、影響範圍及驗證計畫，經 PO 核准後才能修改。  
10. **任務範圍保護：** Developer 不得自行擴大任務範圍；發現必要的額外修改時，應先回報 AI Manager。  
11. **Git 操作保護：** git commit、push、reset、clean 等可能影響版本歷史或工作目錄的操作，必須依 PO 授權執行，不得擅自丟棄或覆蓋未提交的使用者修改。  
12. **依賴套件保護：** 安裝、移除或更新依賴套件前，應先確認必要性、版本及影響範圍，依專案授權流程執行。  
13. **檔案保護：** 不得未經確認覆蓋既有設定檔、PRD 或其他正式文件；修改前應確認目前版本及工作目錄狀態。  
14. **測試獨立性：** Developer 的開發驗證不能取代 QA 驗收；QA 的測試結果必須可追溯（Phase A 初期採流程與職責隔離，未來可視需要導入 Git Worktree 等環境隔離機制）。  
15. **決策紀錄：** 重大架構及產品決策應保存於 docs/decisions/，避免不同 AI Agent 在不同工作階段產生互相矛盾的假設。

### **5.4.6 外部 AI 工具協作**

外部 AI 工具屬於輔助開發與審查工具，不屬於 AI Manager 可自動控制的 Subagent。外部 AI 的技術建議、程式碼及審查結果，應透過任務文件、技術報告或人工交接方式整合，不得直接視為已核准的產品規格、架構決策或測試通過證據。

# **6\. 長期留存與防流失機制 (User Retention)**

16. **破壞第四面牆的情境彩蛋：** 偵測深夜使用手機時，寵物做出催促睡覺的肢體反應（打哈欠、指著時鐘圖示），且不阻止玩家主動互動。  
17. **動態偏好與挑食機制：** 寵物偶爾對單一食物失去新鮮感，引導玩家探索商店內其他商品與成分。  
18. **照護暫停保護機制 (Care Pause)：** 針對忙碌或出差玩家提供無懲罰的暫停衰退保護，維持零壓力陪伴定位。  
19. **微型月卡與送養重置機制（商業驗證後導入）：** 月卡定位為純裝飾與便利（如專屬房間背景、特殊動畫、照護歷史紀錄），絕不將「避免寵物受苦」作為付費門檻；送養重置須經二次確認，重置目前寵物養成狀態但保留全域設定與已解鎖圖鑑。

# **7\. 開發與實作路線圖 (Windows 核心驗證 ➔ 雙平台 MVP)**

* **Phase A：Windows Local Core Prototype（第一優先，0 手機環境依賴）**  
  * 建立 Monorepo、packages/game-core、contracts、config-loader（Local 模式）與 Windows CLI 操作介面。  
  * 實作基因生成、三層行為決策、0.25 小時步進之時間區段模擬、健康救援狀態機、防損 JSON 存檔（.tmp 寫入驗證與 .bak 備份），並通過 Vitest 單元、整合與 365 天長時模擬測試。  
* **Phase B：Renderer Proof of Concept（渲染與跨平台骨架驗證）**  
  * 建立單隻「測試貓」與最少必要動畫（idle、walk、sit、sleep、eat、pet）（對應 8.B.3 之 P0-B；cat\_loaf 與 play 延至 Phase C），驗證 7 大圖層換裝相容性。  
  * 同步驗證 React Native 畫面、Android Kotlin 懸浮窗與 iOS SwiftUI Widget 讀取同一份 PetViewModel 的一致性後，再正式選定動畫引擎（Spine 2D / Rive / Sprite Sheet）。（商店與商品成分頁於本階段僅以固定 Mock Data 製作靜態 UI Prototype，不含購買與經濟結算，正式商店功能留待 Phase D）。  
* **Phase C：Pet MVP（雙平台單貓陪伴上線）**  
  * 開放領養（MVP 限制單一帳號飼養一隻貓，確保模擬與懸浮窗體驗穩定）、7 層外觀生成、基礎性格與互動、完整需求與健康循環、SQLite 存檔與版本遷移（Migration）。  
* **Phase D：商店與內容型原生廣告驗證**  
  * 上線積分系統、商品背包、商店與「成分與營養分析」詳情頁，放入 5\~10 款模擬與合作品牌商品（導入 Remote Config），驗證玩家探索意願與留存數據。  
* **Phase E：長期陪伴、社交與終極 LLM 願景**  
  * 依序擴充 Google Drive 個人雲端備份、Firebase 好友串門子、計步與戶外散步、回憶相簿；待留存與商業模式成熟後，於最後階段接入 LLM API（僅讀取性格與記憶進行語言表達，搭配 BYOK 或對話券控管成本）。

# **8\. 系統細部規格展開 (PM & RD 開發基準)**

## **8.A 貓咪基因與圖鑑設定表 (Trait Inheritance Spec)**

### **8.A.1 完整寵物與基因資料模型 (PetState & PetGenetics Schema)**

10. **1\. PetGenetics（生成後永久鎖定，隨版本升級不得重算）：**  
    * `seed` (string)：生成該貓咪時使用的隨機種子  
    * `prngAlgorithmVersion` (string)：偽隨機數演算法識別碼（如 "mulberry32-v1"）  
    * `geneticsVersion` (string)：遺傳邏輯版本號（如 "1.0.0"）  
    * `rulesVersion` (string)：品種與相容性設定檔版本號（如 "v1"）  
    * `fatherBreed` / `motherBreed` (BreedId)：父母品種識別碼  
    * `appearance` (PetAppearance)：最終確定的 7 大圖層 ID 與 colorParams（唯一渲染依據）  
11. **2\. PetState（主存檔核心實體）：**  
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
    * `lowMoodDuration` ({ below40Hours: number; below20Hours: number })：連續低心情持續時間（小時），隨存檔持久化以防重啟重置  
    * `isCarePaused` (boolean)：照護暫停保護開關（預設 false）  
    * `coins` (number)：持有積分（開局預設 200，不得為負數）  
    * `inventory` (Record\<ItemId, number\>)：背包物品數量表  
    * `lastSimulatedAt` (number)：最後模擬結算 Unix 時間戳 (ms)（時間單一權威來源）

### **8.A.2 初期收錄 6 大基礎品種初始特徵池 (breeds.v1.json) 與變異分布**

12. **權重池配置與稀有突變分布 (traits.v1.json)：** 各品種在七大圖層上的預設候選池規則如下：coatPattern（花色圖層）具 70% 主花色 / 30% 次花色雙候選池（米克斯 tabby\_mackerel 70% / calico 30%、曼赤肯 bicolor\_tuxedo 70% / calico 30%、布偶 colorpoint 70% / bicolor\_tuxedo 30%、英短 solid 70% / tabby\_mackerel 30%、異國 tabby\_mackerel 70% / colorpoint 30%、緬因 tabby\_mackerel 70% / tortoiseshell 30%）；其餘 6 大圖層（head、ears、eyes、pupilColor、tail、body）各品種預設為 100% 品種代表特徵（如表格 8.A.2 所列之七層完整定義），跨品種混合透過父母各 40% 與 20% 變異路徑實現多樣性。第二階段變異路徑（方案 A：各可突變圖層獨立 5%）：設有稀有突變之 4 個圖層（ears: "folded\_tip"、pupilColor: "heterochromia\_blue\_gold"、coatPattern: "starry\_rare"、tail: "kinked\_kirin"），在各自獨立的 20% 變異路徑中，75%（該圖層機率 15%）從該圖層合法通用特徵白名單等權重隨機抽取，25%（該圖層機率 5%）觸發該圖層之稀有突變；單隻貓至少出現一種稀有突變的理論機率為 1 \- (1 \- 0.05)^4 \&approx; 18.55%，且允許同一隻貓同時出現多種稀有突變。未設有稀有突變之 3 個圖層（head、eyes、body），其 20% 變異路徑 100% 從該圖層合法通用特徵白名單等權重隨機抽取。

| 品種 ID (BreedId) | 顯示名稱 | 初始預設特徵池 (Trait Pool \- 七層完整定義) | 基礎性格傾向 (0\~100) | 合法毛色 HEX 色碼區間 (Shader 基準) |
| :---- | :---- | :---- | :---- | :---- |
| `domestic` | 米克斯 | head: standard\_round, ears: pointed\_medium, eyes: almond, pupilColor: amber, coatPattern: tabby\_mackerel, tail: long\_straight, body: normal | 親人 60 / 慵懶 50 / 傲嬌 50 / 好奇 80 | 橘 \#E68A3E\~\#F5C284、虎斑 \#6E5D4F\~\#A89888、黑 \#2B2B2B、白 \#F8F8F8 |
| `munchkin` | 曼赤肯 | head: standard\_round, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: bicolor\_tuxedo, tail: long\_straight, body: short\_legs | 親人 75 / 慵懶 45 / 傲嬌 40 / 好奇 85 | 乳白 \#F5E6D3、三花混色 \#D98245 \+ \#333333、藍灰 \#7A8591 |
| `ragdoll` | 布偶貓 | head: standard\_round, ears: pointed\_medium, eyes: big\_round, pupilColor: sapphire, coatPattern: colorpoint, tail: fluffy\_plume, body: large\_fluffy | 親人 90 / 慵懶 75 / 傲嬌 20 / 好奇 50 | 奶油白底 \#FAF5EE 搭配海豹重點色 \#4A3728 或藍重點色 \#6B7A8F |
| `british_shorthair` | 英國短毛貓 | head: chubby\_round, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: solid, tail: short\_bobtail, body: chubby | 親人 55 / 慵懶 85 / 傲嬌 65 / 好奇 40 | 經典藍灰 \#6C7680\~\#8C96A0、銀漸層 \#D8DEE3、金漸層 \#D9A760 |
| `exotic` | 異國短/長毛 (加菲/喜馬拉雅) | head: flat\_face, ears: small\_round, eyes: big\_round, pupilColor: copper, coatPattern: tabby\_mackerel, tail: short\_bobtail, body: chubby | 親人 70 / 慵懶 80 / 傲嬌 55 / 好奇 45 | 橘紅 \#D97236、奶油重點色 \#F2E4D5 \+ 巧克力重點 \#5C4033 |
| `maine_coon` | 緬因貓 | head: square\_muzzle, ears: tufted\_large, eyes: almond, pupilColor: emerald, coatPattern: tabby\_mackerel, tail: fluffy\_plume, body: giant\_fluffy | 親人 80 / 慵懶 50 / 傲嬌 35 / 好奇 75 | 棕虎斑 \#594636\~\#8C735B、煙燻黑 \#3A3D40、紅虎斑 \#C86D3B |

### **8.A.3 七大外觀圖層枚舉與確定性相容修正規則 (appearance-compatibility.v1.json)**

13. **七大圖層枚舉：**  
    * `head`：`standard_round`、`chubby_round`、`flat_face`、`square_muzzle`  
    * `ears`：`pointed_medium`、`small_round`、`tufted_large`、`folded_tip`（稀有突變）  
    * `eyes`：`almond`、`big_round`、`droopy_cute`  
    * `pupilColor`：`amber`、`emerald`、`sapphire`、`copper`、`heterochromia_blue_gold`（稀有突變）  
    * `coatPattern`：`solid`、`tabby_mackerel`、`bicolor_tuxedo`、`colorpoint`、`calico`、`tortoiseshell`、`starry_rare`（稀有突變）  
    * `tail`：`long_straight`、`fluffy_plume`、`short_bobtail`、`kinked_kirin`（稀有突變）  
    * `body`：`normal`、`chubby`、`short_legs`、`large_fluffy`、`giant_fluffy`  
14. **確定性相容修正規則 (appearance-compatibility.v1.json)：**  
    \- **執行順序與單次循序套用規範：** 固定依序執行 **規則 1 ➔ 規則 2 ➔ 規則 3**（單次循序執行，不反覆迴圈重跑）。前一條規則修正後的外觀狀態直接作為下一條規則的判斷輸入（例如同時出現 head \=== "flat\_face"、body \=== "giant\_fluffy"、ears \=== "folded\_tip" 時，先由規則 1 將 body 修正為 "chubby"，後續檢查規則 3 時因 body 已非 "giant\_fluffy" 而不再觸發）；若多條規則依序檢查時均符合條件，則全部依序執行。規則 1 ➔ 2 ➔ 3 執行完畢後進行最終合法性檢查（確認七大圖層皆為合法枚舉且不再違反規則 1\~3），若仍不合法才觸發規則 4 回退 SAFE\_DEFAULT\_APPEARANCE。  
    * **規則 1（跨圖層：扁臉與大型骨架比例衝突）：** 若 head \=== "flat\_face" 且 body \=== "giant\_fluffy"，為避免扁臉搭配巨型緬因骨架比例失真，強制將 body 修正為 "chubby"。  
    * **規則 2（跨圖層：重點色與花色遮罩衝突）：** 若 coatPattern \=== "colorpoint" 且 pupilColor \=== "amber"，強制將 pupilColor 修正為 "sapphire"。  
    * **規則 3（跨圖層：稀有折耳與緬因大耳羽衝突）：** 若 ears \=== "folded\_tip" 且 body \=== "giant\_fluffy"，強制將 ears 修正為 "tufted\_large"。  
    * **規則 4（終極安全回退 SAFE\_DEFAULT\_APPEARANCE）：** 若經過單次規則替換後最終驗證仍不合法，直接退回完整七層安全預設組合 SAFE\_DEFAULT\_APPEARANCE：`{ head: "standard_round", ears: "pointed_medium", eyes: "almond", pupilColor: "amber", coatPattern: "solid", tail: "long_straight", body: "normal" }`。  
    * **註：** 未來若開放其他遺傳組合產生 flat\_face \+ giant\_fluffy，維持 body 修正為 "chubby" 以確保比例協調。

### **8.A.4 遺傳生成執行優先級 (Genetics Priority)**

為避免特殊骨架規則與一般 40/40/20 規則衝突，強制規定執行順序：

15. **步驟 1（特殊顯性骨架規則優先，僅作用於指定圖層）：**  
    * **曼赤肯 (munchkin) body 規則：**  
      * 若父母雙方皆為 `munchkin`（雙親同種）：子代 `body` 為 **75%** `short_legs` **/ 25%** `normal`（避免重複計算「另一方體型」）。  
      * 若僅一方為 `munchkin`（跨品種配對）：子代 `body` 為 **50%** `short_legs` **/ 25% 另一方品種之** `body`（若另一方為 maine\_coon 則為 giant\_fluffy，允許 short\_legs 與 giant\_fluffy 共存形成短腿巨型長毛體型） / **25%** `normal`。  
    * **異國短/長毛 (exotic) head 規則：**  
      * 若父母雙方皆為 `exotic`（雙親同種）：子代 `head` 為 **80%** `flat_face` **/ 20%** `standard_round`（避免 100% 鎖死單一頭型）。  
      * 若僅一方為 `exotic`（跨品種配對）：子代 `head` 為 **45%** `flat_face` **/ 55% 另一方品種之** `head`。  
16. **步驟 2（一般圖層獨立 40/40/20 抽選）：** 其餘未鎖定圖層依固定順序（head ➔ ears ➔ eyes ➔ pupilColor ➔ coatPattern ➔ tail ➔ body）逐一獨立抽選：40% 父系預設特徵、40% 母系預設特徵、20% 變異路徑（設有稀有突變之 ears、pupilColor、coatPattern、tail 四層於變異路徑內 75% 走該圖層全品種合法白名單重組、25% 觸發該圖層稀有突變，即每層獨立 5%；無稀有突變之 head、eyes、body 三層於變異路徑內 100% 走全品種合法白名單重組）。  
17. **步驟 3（確定性相容性修正）：** 依 8.A.3 固定順序（規則 1 ➔ 規則 2 ➔ 規則 3）執行單次循序衝突替換與最終合法性驗證（未通過則依規則 4 回退 SAFE\_DEFAULT\_APPEARANCE）。  
18. **步驟 4（動態毛色微調）：** 依據 `seed` 在合法 HEX 區間內進行 `±8%` 明度與飽和度微調。

## **8.B 動畫與行為決策仲裁規格 (BehaviorEngine & FSM Spec)**

### **8.B.1 行為決策引擎優先級、同級仲裁與中斷規則**

BehaviorEngine 輸出單一標準結構 BehaviorDecision（animationId、tailState、iconIds: string\[\]、soundId、durationMs、interruptible、cooldownMs）：

| 優先級 | 狀態類別 | 觸發條件（嚴格運算符） | 行為輸出（主動畫 / 尾巴 / 圖示陣列） | 同級仲裁、持續與中斷規範 |
| :---- | :---- | :---- | :---- | :---- |
| **100** | 緊急生理求救 (CRITICAL\_NEEDS) | `health < 30` 或 `hunger < 15` 或 `hydration < 15` | 匍匐警戒 (crouch\_alert) 或虛弱打瞌睡，夾尾/垂尾，彈出 🏥 / 🐟 / 💧 | **同級仲裁**：iconIds 可同時顯示所有達標圖示；主動畫依明確清單順序 **1\. health \< 30 ➔ 2\. hydration \< 15 ➔ 3\. hunger \< 15** 決定。可中斷任何低優先級行為（包含 Priority 90 與 Priority 40 eat），持續播放至 health \>= 30 && hunger \>= 15 && hydration \>= 15。 |
| **90** | 負面防禦反應 (NEGATIVE\_REACTION) | 10 秒內 \[now \- 10000, now\] `POKE > 3` 次（第 4 次觸發），或高空掉落 LANDING（定義為 Android 懸浮窗拖曳放開時，垂直落下距離超過螢幕高度 25% 觸發，短距離輕放不觸發） | 背對生氣 (turn\_back\_angry) 或炸毛，快速鞭打尾，彈出 💢 並哈氣 | 若同時發生，主動畫優先序為 **LANDING \> POKE**。優先級高於 Priority 80，可暫時喚醒睡眠中的貓咪執行 4,000 ms 動畫。結束後進入 15,000 ms 互動無效冷卻期（同時限制 PetCat 摸頭與 PlayWithPet 玩耍）。撫摸/玩耍冷卻（15,000 ms）與 Priority 60 提醒冷卻（60,000 ms）各自獨立計時。冷卻期間摸頭與玩耍仍可播放互動動畫，但不給心情與積分獎勵，餵食/補水仍完全有效；冷卻期間再次 Poke 不延長冷卻時間。 |
| **80** | 生理作息睡眠 (SLEEP\_MODE) | 本地時間 `23:00 ~ 07:00` 且無玩家主動互動 | 靠邊熟睡 (sleep) 或母雞蹲 (cat\_loaf)，休眠捲曲尾，彈出 💤 | **夜間可喚醒規則**：僅能被有效的玩家數值互動指令（FeedPet、PetCat、PlayWithPet）暫時喚醒並執行 Priority 40，互動結束後重新入睡；低電量、插線充電等系統事件不喚醒睡眠。 |
| **60** | 一般需求提醒 (MODERATE\_NEEDS) | `hunger < 40` 或 `hydration < 40` 或 `mood < 45` | 坐下 (sit) 盯著玩家，尾巴慢速左右掃動，彈出 🐟 / 💧 / 🎾 | **同級仲裁**：若同時符合多個條件，iconIds 合併顯示所有達標提醒圖示，但主動畫只選擇最高優先項目，優先順序為：1. hydration \&lt; 40 ➔ 2\. hunger \&lt; 40 ➔ 3\. mood \&lt; 45。每次提醒持續 `5,000 ms`，共用冷卻 `60,000 ms`（從提醒動畫首次開始播放的時間戳 startedAt 起算）。 |
| **40** | 玩家主動互動 (PLAYER\_INTERACTION) | 收到有效數值互動指令 FeedPet（使用食物）、PetCat（有效摸頭）或 PlayWithPet（有效玩耍） | FeedPet ➔ 進食 (eat) 舉問號尾微顫；PetCat ➔ 摸頭呼嚕 (pet) 彈出 ❤️；PlayWithPet ➔ 玩耍撲抓 (play) 彈出 🎾 | 1\. 數值與動畫解耦：三種互動皆先執行 GameCommand 數值與積分結算，動畫中斷絕不回滾已生效之數值。2. 持續與中斷：eat、pet、play 均持續 3,000 ms（不可被待機或 23:00 跨夜中斷，但可隨時被 Priority 100 緊急生理求救或 Priority 90 負面防禦中斷）。3. 夜間回歸：若於 23:00\~07:00 喚醒執行，動畫結束後自動返回 Priority 80 (SLEEP\_MODE)。4. 冷卻與併發：受 Priority 90 負面冷卻（15,000 ms）限制（冷卻期間摸頭/玩耍不給心情與積分）；同一 Tick 多筆指令依佇列順序執行（FeedPet \> PlayWithPet \> PetCat）。 |
| **10** | 性格待機巡邏 (IDLE\_AND\_PATROL) | 無上述高優先級事件時執行 | 漫步 (walk \+ 問號尾)、母雞蹲 (cat\_loaf)、貴妃躺 (side\_sleep，需 mood \>= 80\) 輪替 | 每次動作持續 `4,000 ~ 8,000 ms`，可隨時被高優先級事件中斷。 |

### **8.B.2 性格與小怪癖干預公式**

1. **傲嬌先拒後吃 (basePersonality.tsundere \>= 70)：** 使用免費基礎乾糧 (free\_basic\_kibble) 時，有 45% 機率先播放撇頭嫌棄 (sniff\_and\_turn\_away \+ 💦 圖示，持續 2,000 ms)，隨後自動接續 eat 進食；使用主食罐或零食則直接進食。  
2. **慵懶臥息比例 (basePersonality.laziness)：** 待機狀態下選擇「母雞蹲或貴妃躺」的機率為 30% \+ (laziness \* 0.5)%。  
3. **小怪癖加法修正 (cursor\_hunter)：** 具備 cursor\_hunter 時，偵測畫面點擊靠近之基礎機率（20%）套用 additiveProbabilityModifier: \+0.30，變為 50%。

### **8.B.3 動畫資產矩陣分級 (Animation Asset Matrix)**

4. **【唯一 AnimationId 規範】** BehaviorEngine、PetViewModel 與本動畫資產矩陣全專案強制使用同一組正式 AnimationId，不另設別名。  
5. **Phase B 必備核心動畫 (P0-B，共 6 支，對應第 7 章 Phase B 最少必要動畫)：**`idle`（待機）、`walk`（漫步）、`sit`（坐下）、`sleep`（熟睡）、`eat`（進食）、`pet`（被摸呼嚕）。  
6. **Phase C 必備核心動畫 (P0-C，共 2 支，Phase B 不要求實作)：**`cat_loaf`（母雞蹲/趴臥；Phase B 尚未實作前於待機與夜間模式回退播放 `sit` 或 `idle`）、`play`（玩耍；Phase B 尚未實作前於互動回饋回退播放 `pet` 或 `idle`）。  
7. **Phase C 進階體驗動畫 (P1，共 7 支)：**`drag`（Android 拎起）、`drop`（掉落）、`landing`（落地）、`turn_back_angry`（生氣）、`sniff_and_turn_away`（挑食撇頭）、`crouch_alert`（虛弱匍匐）、`side_sleep`（貴妃躺，mood \>= 80 待機動作；尚未實作前於待機輪替回退播放 `cat_loaf` 或 `idle`）。

## **8.C 經濟系統與數值平衡表 (Economy & TimeEngine Spec)**

### **8.C.1 時間步進計算順序與日夜區段公式 (TimeEngine Math)**

**【指令與時間模擬執行順序 (P1-4)】**  
所有具有遊戲數值效果的 `GameCommand`（如 FeedPet、PetCat 等）在執行前，必須先呼叫 `TimeEngine.advanceTo(commandTimestamp)` 將遊戲狀態推進至該指令的有效時間點，再執行指令並套用數值變化。每次會改變 Hunger、Hydration、Mood 或 Health 的 GameCommand 完成數值結算後，必須立即於該 GameCommand 的時間戳 (commandTimestamp) 重新評估相關狀態機條件（包含 NORMAL ➔ CRITICAL、CRITICAL ➔ RECOVERY\_BOOST 啟動、RECOVERY\_BOOST 中斷與恢復、NATURAL\_RECOVERY 恢復條件判斷，以及 Health \&gt;= 100 時的 NORMAL 重置與 recoveryBoostAccumulatedHours 歸零）；若符合轉換條件，轉換時間即為該 commandTimestamp，嚴禁延遲至下一個 SimulationStep。  
**【連續閾值交點切割與時間區段計算 (Threshold Crossing Split)】**  
為確保「一次補算 24 小時」與「任意時間點插入 GameCommand 分段補算」的最終結果一致，數值誤差小於 1e-9，TimeEngine 每次 advanceTo(now) 除按 0.25 小時（15 分鐘）基礎步長步進外，在推進時間區間 \[t\_start, t\_end\] 時，遇到以下任一狀態變化事件，必須建立精確交點切割 (Exact Boundary / Threshold Crossing Split)：  
1\. 日夜時段邊界（每日 07:00 與 23:00）。  
2\. 需求與健康狀態機門檻交點：Hunger \= 20、Hydration \= 20（扣血/額外懲罰門檻）、Hunger \= 50、Hydration \= 50（RECOVERY\_BOOST 門檻）、Hunger \= 60、Hydration \= 60、Mood \= 70（NATURAL\_RECOVERY 滿速門檻）、Mood \= 40（NATURAL\_RECOVERY 半速/低心情門檻）、Mood \= 20（嚴重低心情門檻）、Health \= 10（飢餓/缺水 Health 保護下限）、Health \= 40（CRITICAL / MoodHealthPenalty 安全地板門檻）與 Health \= 100（滿血完成門檻）。當 Health \> 10 且扣血速率為負時，必須計算 Health \= 10 之精確交點；抵達 Health \= 10 後，飢餓/缺水扣血速率立即歸零（0/hr），且後續區段不得使 Health 低於 10。**Signed Rate Threshold Crossing（帶正負號速率交點算式）：** 交點計算統一使用帶正負號的 ratePerHour（數值下降時 ratePerHour \< 0；數值上升時 ratePerHour \> 0；ratePerHour \= 0 時不存在由該區段速率產生的交點），交點時間戳公式為 t\_cross \= t\_start \+ ((targetThreshold \- currentValue) / ratePerHour) \* 3\_600\_000。只有當交點時間位於目前模擬區段內，且數值確實朝向該門檻移動時，才建立切割事件；當速率為 0、交點位於區段之外，或數值遠離門檻時，不得建立無效交點。所有交點仍須符合 TimeEngine 的精度要求，最終分段模擬結果誤差 \< 1e-9。**交點有效性與零時間事件過濾：** 數值門檻比較與時間戳去重之容差常數必須集中定義於核心共用常數模組（如 packages/game-core 之 TIME\_ENGINE\_CONSTANTS），嚴禁各模組自行設定不同容差：數值門檻比較統一採用 VALUE\_EPSILON \= 1e-9，時間戳事件去重統一採用 TIME\_EPSILON\_MS \= 1e-3（1 微秒，大於 IEEE 754 雙精度浮點於 Unix ms 時間戳量級之機器誤差）。建立 Threshold Crossing Event 時，除上述條件外，必須同時符合：(1) Math.abs(currentValue \- targetThreshold) \> VALUE\_EPSILON（數值已在 VALUE\_EPSILON 範圍內等於門檻時不建立交點）；(2) t\_cross 必須嚴格大於 t\_start \+ TIME\_EPSILON\_MS 且小於或等於 t\_end（t\_start \+ TIME\_EPSILON\_MS \< t\_cross \<= t\_end）；(3) 若計算規則與已處理事件時間差小於或等於 TIME\_EPSILON\_MS（Math.abs(t\_cross \- t\_existing) \<= TIME\_EPSILON\_MS），必須合併或忽略重複事件，不得重複建立零時間區段；(4) 交點處理完成後，必須立即更新目前數值與事件狀態，避免下一次迭代再次觸發相同交點。  
3\. 剩餘 Boost 額度耗盡時間點：t\_boost\_end \= t\_start \+ (2.0 \- recoveryBoostAccumulatedHours) \* 3\_600\_000。  
4\. 低心情門檻與持續時間到期交點：Mood \= 40、Mood \= 20 穿越交點，以及 below40Hours 累計滿 72.0 小時、below20Hours 累計滿 48.0 小時的精確時間交點。  
5\. 玩家數值 GameCommand 發生時間與 Care Pause 啟用/解除時間。  
當執行數值 GameCommand 時，直接推進至 commandTimestamp 並按實際經過小時數 (actualHours \= segmentMs / 3\_600\_000) 結算，更新 lastSimulatedAt。

1. **步驟 1（判定區段時段係數）：** 若該區段之起始時間落於 07:00 \~ 23:00，時段係數 periodFactor \= 1.0；若落於 23:00 \~ 07:00，periodFactor \= 0.3。  
2. **步驟 2（更新 hunger 與 hydration）：**  
   1. hunger 扣除 4.0 \* periodFactor \* actualHours（最低 0）。  
   2. hydration 扣除 4.0 \* periodFactor \* actualHours（最低 0；日間 \-4.0/hr、夜間 \-1.2/hr，每日 24 小時基礎總衰退 73.6，與 Hunger 一致，避免每日一次登入因 100 上限截斷而每日固定缺水扣血）。  
3. **步驟 3（更新 mood 與低需求額外懲罰 \- P0-1）：**  
   1. 基礎心情扣除 **1.25 \* periodFactor \* actualHours**（日間每小時 \-1.25，夜間每小時 \-0.375；每日 24 小時基礎心情總衰退為 23.0）。  
   2. 若步驟 2 更新後之 hunger \< 20 或 hydration \< 20，額外心情懲罰亦乘以時段係數，扣除 1.5 \* periodFactor \* actualHours（最低 0）。  
4. **步驟 4（決定飢餓/缺水 Health 扣血速率 starvationHealthRate — 保護下限 10）：** 本步驟僅判定該時間區段內之飢餓/缺水扣血速率，**不得在步驟 4.5 判定前先行修改 health**：若該區段滿足 hunger \< 20 或 hydration \< 20（且 health \> 10），則 starvationHealthRate \= \-2.0/hr（不受夜間係數減免，最低保護下限為 10）；當 health 精確抵達 10 時，starvationHealthRate 綠即歸零為 0/hr，後續區段不得使 health 低於 10；未滿足條件時亦為 0/hr。  
5. **步驟 4.5（決定獨立低心情健康懲罰速率 moodHealthRate 並與步驟 4 合併結算 — C. 混合模式，保護地板 40）：** 短期缺少陪伴僅降低 Mood，不影響 Health；唯有連續低心情達標才啟動極緩慢扣血，且與步驟 4 視為同一時間區段內之獨立速率合併結算，各自保留保護下限（飢餓/缺水最低 10；低心情最低 40）。TimeEngine 必須於 below40Hours \= 72.0 與 below20Hours \= 48.0 精確交點切割時間區段，不得先累加整個 0.25 小時步長再將整段時間套用扣血；僅從精確跨越門檻（嚴格 \> 72.0 或 \> 48.0）之後的時間區段起套用對應扣血速率：  
   1. **Mood \>= 40：** 健康無影響（moodHealthRate \= 0/hr），立即重置 below40Hours \= 0 與 below20Hours \= 0。  
   2. **Mood 20 \~ 39：** 累加 below40Hours \+= actualHours（同時重置 below20Hours \= 0）；當 below40Hours \> 72.0 時，超出 72.0 小時的部分 moodHealthRate \= \-0.05/hr（不受夜間係數減免），否則 moodHealthRate \= 0/hr。  
   3. **Mood 0 \~ 19：**  
   4. 同時累加 below40Hours \+= actualHours 與 below20Hours \+= actualHours。  
      1. 若 below20Hours \> 48.0：moodHealthRate \= \-0.1/hr（取代 \-0.05/hr，不得疊加，不受夜間係數減免）。  
      2. 若 below20Hours \<= 48.0 且 below40Hours \> 72.0：moodHealthRate \= \-0.05/hr（不受夜間係數減免）。  
      3. 其他情況（below20Hours \<= 48.0 且 below40Hours \<= 72.0）：moodHealthRate \= 0/hr。  
   5. **合併速率結算與 Health \= 40 / Health \= 10 精確交點切割：** 當區段起點 **health \> 40** 時，該區段之 Health 總扣血速率為 **totalHealthPenaltyRate \= starvationHealthRate \+ moodHealthRate**（例如同時滿足飢餓與 below20Hours \> 48.0 時為 \-2.0 \+ (-0.1) \= \-2.1/hr）。TimeEngine 必須依 Signed Rate Threshold Crossing 以 totalHealthPenaltyRate 精確切割 Health \= 40 交點，確保兩種扣血共同作用直至 health 精確抵達 40.0。當 **10 \< health \<= 40** 時，MoodHealthPenalty 立即停止（moodHealthRate 強制為 0/hr，絕不因低心情而將寵物推入 CRITICAL \< 40 或死亡），僅保留 starvationHealthRate（若滿足飢餓/缺水則維持 \-2.0/hr），並精確切割至保護下限 Health \= 10；抵達 Health \= 10 交點後，飢餓/缺水扣血速率立即歸零（totalHealthPenaltyRate \= 0/hr），後續區段不得使 health 低於 10（最終 health \= Math.max(10, health \+ totalHealthPenaltyRate \* actualHours)；若僅有低心情扣血且 starvationHealthRate \=== 0，則 health 嚴格受 40 地板保護不得低於 40）。  
6. **步驟 5（執行健康救援狀態機 Recovery State Machine \- P0-2, P0-4, P1-6）：**  
   1. **1\. NORMAL ➔ CRITICAL：** 當 health \< 40 時觸發一次轉換，僅在此轉換瞬間執行 recoveryBoostAccumulatedHours \= 0。  
   2. **2\. CRITICAL ➔ RECOVERY\_BOOST：** 處於 CRITICAL 狀態下，當滿足 hunger \>= 50 && hydration \>= 50 且 recoveryBoostAccumulatedHours \< 2.0 時轉入，每小時恢復 \+10.0 health（按 actualHours 計算），並累加 recoveryBoostAccumulatedHours \+= actualHours（上限 2.0）。  
   3. **3\. RECOVERY\_BOOST 中斷與額度耗盡轉換：** 若加速期間 hunger \< 50 或 hydration \< 50，退回 CRITICAL 暫停回血但完整保留 recoveryBoostAccumulatedHours 已累計時數；當 recoveryBoostAccumulatedHours \>= 2.0（2 小時額度用滿）時，立即轉為 NATURAL\_RECOVERY（即使此時 health \< 40 或尚未滿足 60/60/70，也不退回 CRITICAL 且不重置已累計時數）。  
   4. **4\. NATURAL\_RECOVERY 分級回血與暫停 (RecoveryMultiplier)：** 處於 NATURAL\_RECOVERY 下，當滿足 hunger \>= 60 && hydration \>= 60 時，依據當下 Mood 區間分級回血（按 actualHours 計算）：(1) Mood \>= 70（心情良好）：每小時恢復 \+2.5 health；(2) Mood 40 \~ 69（普通/略低落）：每小時恢復 \+1.25 health（陪伴不足仍可緩慢康復）；(3) Mood \< 40（明顯/嚴重低落）：暫停 NATURAL\_RECOVERY 回血，需先透過撫摸、玩耍或零食安撫情緒。未滿足 hunger \>= 60 && hydration \>= 60 則亦暫停回血（維持 NATURAL\_RECOVERY 狀態，唯有恢復至 NORMAL 後未來再度跌破 40，才視為全新救援週期重新獲得 2.0 小時 Boost 額度）。  
   5. **5\. 全域滿血重置規則 (Global Full Health Reset)：** 無論處於 CRITICAL、RECOVERY\_BOOST 或 NATURAL\_RECOVERY 任一階段，當 health \>= 100 時，health 綠即 Clamp 為 100，recoveryState 轉為 NORMAL 且 recoveryBoostAccumulatedHours \= 0。若同一時間點同時發生 health \< 40 與 hunger \< 20，依序先結算需求與 health 扣減，再觸發狀態機轉為 CRITICAL（僅在由 NORMAL 轉入 CRITICAL 的瞬間歸零累計時數）。  
   6. **6\. Care Pause 啟用前結算與凍結規則：** 啟用 Care Pause 前，必須先呼叫 TimeEngine.advanceTo(pauseStartTimestamp) 完成此前所有應有的離線模擬結算，再將 isCarePaused 設為 true 完全凍結 TimeEngine 衰退與 RecoveryState 累計時數；解除暫停時不補算暫停期間，直接將 lastSimulatedAt 更新為 resumeTimestamp 並執行一次狀態機檢查。

### **8.C.2 商品目錄與遊戲效果平衡表 (food-catalog.v1.json)**

所有商品依「品類基準 \+ 個性化微調」配置效果，品牌合作商品享有完全相同的平衡公式；Schema 明確區分兩個獨立欄位：productType ("regular" \| "sponsored" \| "free") 表示商品合作類型，dataType ("real" \| "mock") 表示資料真實性，並搭配 sourceName、sourceUrl、dataVerifiedAt 與 lastReviewedAt 確保資訊可追溯（UI 層對 mock 商品強制顯示「測試商品/模擬資料」標籤）：  
\- `sourceName: string`（必填：未取得官方資料時標示模擬資料來源，如 "MVP 模擬測試資料"；取得官方資料時標示官方來源）。  
\- `sourceUrl: string | null`（`mock` 商品可為 `null`；`real` 商品必須提供有效來源 URL）。  
\- `dataVerifiedAt: string | null`（官方資料核實日期，格式 `YYYY-MM-DD`；尚未取得官方資料核實之 `mock` 商品應為 `null`，不得將內部審查日期誤填為官方核實日期）。  
\- `lastReviewedAt: string | null`（記錄內部審查日期，格式 `YYYY-MM-DD`，如 "2026-11-19"，與官方核實日期 dataVerifiedAt 區分）。  
**【商品定價級距與固定效果白名單驗證準則】** MVP 階段不設自動營養換算公式，所有上架商品（含一般商品與品牌合作商品）均須由 PO 核准並登錄於固定商品效果白名單 food-catalog.v1.json。同一級距內允許依真實營養特性（如高水分慕斯罐水分略高、高肉含量商品飽食略高）微調數值，但**定價級距上下限為硬性天花板（定價級距優先於營養微調，嚴禁跨級突破）**，且嚴禁以品牌合作身分、贊助金額或曝光合約作為任何加成參數。QA 驗證時逐筆核對商品之 category、price 所屬級距及 gameEffect 是否完全落於下表合法區間：

| 商品類別 | 定價級距（積分） | 合法飽食區間 | 合法水分區間 | 合法心情區間 |
| :---- | :---- | :---- | :---- | :---- |
| 飲水 | 免費基礎（0） | \+0 | \+32 | \+2 |
| 乾糧 | 免費基礎（0） | \+25 | \-2 | \+2 |
| 乾糧 | 付費進階（80\~110） | \+26\~+28 | \-2\~-1 | \+5\~+6 |
| 主食罐 | 入門級（50\~70） | \+25 | \+20\~+21 | \+5\~+6 |
| 主食罐 | 標準級（71\~95） | \+25\~+27 | \+20\~+22 | \+7\~+8 |
| 主食罐 | 頂級（96\~120） | \+27\~+28 | \+20\~+22 | \+8\~+9 |
| 零食 | 標準級（40\~60） | \+8 | \-1 | \+14 |

| 商品 ID (itemId) | 商品名稱 | 類別 | 合作類型 (productType) | 資料真實性 (dataType) | 定價 (積分) | 真實營養與成分標示 (nutrition & ingredients) | 遊戲效果 (gameEffect) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| `free_basic_kibble` | 基礎日常乾糧 | 乾糧 | free (免費基礎) | mock | 0 (無限) | 粗蛋白 28% / 粗脂肪 12% / 水分 10%｜穀物、家禽副產品粉、維生素 | 飽食 \+25、水分 \-2、心情 \+2 |
| `free_fresh_water` | 過濾飲用水 | 飲水 | free (免費基礎) | mock | 0 (無限) | 純淨過濾水 100%（建議每日更換維持新鮮） | 飽食 \+0、水分 \+32、心情 \+2 |
| `shop_chicken_can` | 鮮燉嫩雞主食罐 | 主食罐 | regular (一般商品) | mock | 60 | 粗蛋白 12.5% / 粗脂肪 4.5% / 水分 78%｜去骨雞胸肉、雞高湯、牛磺酸 | 飽食 \+25、水分 \+20、心情 \+5 |
| `shop_salmon_mousse` | 野生鮭魚慕斯罐 | 主食罐 | regular (一般商品) | mock | 80 | 粗蛋白 11.8% / 粗脂肪 5.2% / 水分 80%｜野生鮭魚、魚油、洋車前子膳食纖維 | 飽食 \+25、水分 \+22、心情 \+8 |
| `brand_nu4pet_kitten` | 【陪心】屁孩罐－葡聚醣雞肉 | 主食罐 | sponsored (品牌合作) | mock | 90 | 粗蛋白 13.0% / 粗脂肪 5.0% / 水分 79%｜台灣鮮雞肉、β-葡聚醣、魚油、維生素 E | 飽食 \+26、水分 \+21、心情 \+8 |
| `brand_ziwi_venison` | 【Ziwi 巔峰】紐西蘭鹿肉主食罐 | 主食罐 | sponsored (品牌合作) | mock | 110 | 粗蛋白 12.0% / 粗脂肪 6.5% / 水分 78%｜紐西蘭放牧鹿肉、鹿肝、綠唇貝、海帶 | 飽食 \+28、水分 \+20、心情 \+9 |
| `brand_rc_hairball` | 【皇家】IH34 化毛配方乾糧 | 乾糧 | sponsored (品牌合作) | mock | 95 | 粗蛋白 34% / 粗脂肪 15% / 水分 8%｜脫水禽肉蛋白、洋車前子殼、甜菜漿 | 飽食 \+28、水分 \-1、心情 \+6 |
| `shop_freeze_dried` | 凍乾原肉丁零食 | 零食 | regular (一般商品) | mock | 50 | 粗蛋白 82% / 粗脂肪 8% / 水分 5%｜100% 冷凍乾燥雞胸肉丁 | 飽食 \+8、水分 \-1、心情 \+14 |

### **8.C.3 日常積分與「商品探索任務」平衡表 (EconomyEngine)**

開局預設 **200 積分**。dailyLimits 隨 GameState 一起保存，包含裝置本地日期 YYYY-MM-DD、lastClaimedTimestamp 與當日已發放獎勵事件 ID 集合（claimedRewardIds: string\[\]）；每日獎勵採「每個裝置本地日曆日最多領取一次 ＋ 20 小時最小實際冷卻間隔」雙重規則，防止短時間內透過調整時區重複領取，同時保留正常早晚作息彈性：玩家領取每日獎勵時，必須同時滿足：(1) currentLocalDate \> lastClaimedLocalDate（防止時鐘回撥或往西跨時區日期倒退重複領取）；且 (2) now \- lastClaimedTimestamp \>= 20 \* 3\_600\_000（防止短時間內透過調整時區重複領取，並保留正常早晚作息彈性）。若偵測時鐘回撥 (now \< lastClaimedTimestamp) 或因跨時區旅行/手動調整致本地日期倒退 (currentLocalDate \< lastClaimedLocalDate)，拒絕重複發放獎勵。不補發未登入日之簽到或互動積分，高健康度日結獎勵僅於實際跨日結算當下觸發一次（在取得官方正式核實前所有商品一律標示 dataType: "mock"、sourceName: "MVP 模擬測試資料"、sourceUrl: null、dataVerifiedAt: null，並以 lastReviewedAt: "2026-11-19" 記錄內部審查日期；取得官方正式核實後則更新為 dataType: "real"、填入實際官方 sourceName、有效 sourceUrl 與官方 dataVerifiedAt）：

7. **每日首次陪伴簽到：** `+40` 積分（每日上限 1 次）。  
8. **商品探索任務（不限品牌或一般商品，無廣告式領獎彈窗）：**  
   1. 每日首次開啟商店瀏覽：`+5` 積分。  
   2. 點開任一商品詳情頁並完整檢視營養成分分析（停留 `>= 3` 秒）：每款不重複商品 `+10` 積分（同商品當日重複開關不給分，每日探索獎勵上限 **30 積分**）。  
9. **有效摸頭與玩耍互動：** 每次有效互動 `+5` 積分與 `mood +3`（每日上限 `25` 積分）。  
10. **高健康度日結獎勵：** 跨日結算時若 `health >= 90` 且 `mood >= 80`，獎勵 `+25` 積分。

## **8.D UI/UX 介面與操作動線 (UI/UX Flow)**

1. **開局領養流程（MVP 限制單貓制）：**  
   1. 選擇父母品種（支援無限次重新隨機盲盒搭配，確定領養後鎖定外觀與 `seed`） ➔ 基因融合誕生過場 ➔ 預覽外觀並命名（1\~12 字元） ➔ 雙平台權限引導（Android 引導懸浮窗授權；iOS 引導桌面 Widget 教學）。  
2. **主房間與控制選項 (HomeScreen)：**  
   1. 頂部顯示四大狀態條與積分；底部為「背包餵食」、「商店」、「圖鑑」與「設定」。  
   2. **照顧保護與顯示設定：** 支援「暫時隱藏懸浮窗至通知列」、「貓咪顯示尺寸調整（小/中/大）」以及「照護暫停保護模式 (Care Pause)」。  
3. **商店與商品詳情頁 (ShopScreen & ProductDetailModal)：**（階段邊界：Phase B 僅建立固定 Mock Data 之靜態 UI Prototype，不含購買與經濟結算；正式商品目錄、積分購買、背包、探索任務與 Remote Config 統一於 Phase D 上線。）  
   1. 列表支援分類篩選與「品牌合作 / 測試商品」清晰標示；詳情頁完整呈現「真實品牌、原料、粗蛋白/脂肪/水分分析、資料來源與更新日期」，以及獨立區塊的「遊戲回復數值」與「購買放入背包」按鈕。

## **8.E 存檔架構、遷移規範與自動化驗收劇本**

### **8.E.1 雙階段存檔與時間戳單一權威規範 (Save Migration)**

1. **時間戳單一權威來源：** PetState.lastSimulatedAt 為遊戲模擬時間的唯一權威來源；資料庫外層之 created\_at 與 updated\_at 僅供維運追蹤，寫入時由 Adapter 注入之 Clock 同步填入，不得與 PetState.lastSimulatedAt 產生雙頭馬車。  
2. **Phase A (Windows 本機)：** 採用防損 JSON 寫檔流程（先寫入 save.tmp ➔ 驗證 JSON 結構 ➔ 備份舊檔為 save.bak ➔ 原子替換為 save.json）。  
3. **Phase C (手機端 SQLite)：**  
   1. `game_saves` 表包含：id、schema\_version（目前為 2；此 Schema 版本於 Master PRD v2.1.4 引入，v2.1.5 \~ v2.1.28 未變更）、game\_core\_version、pet\_state\_json（必須完整序列化保存整個 PetState，明確包含 genetics、appearance、basePersonality、quirks、memories、needs、health、recoveryState、recoveryBoostAccumulatedHours、lowMoodDuration、isCarePaused、coins、inventory、lastSimulatedAt）、daily\_limits\_json、created\_at、updated\_at。  
   2. `game_events` 表僅作為行為追蹤與除錯日誌（非主存檔來源），定期清理超過 30 天之舊紀錄。  
4. **存檔升級遷移規則 (Migration)：** 每次 App 升級偵測到 `schema_version` 落後時，先自動備份原始存檔再執行漸進式遷移腳本（`v1 -&gt; v2`），當載入 schema\_version \= 1 的舊存檔時，自動補齊 recoveryBoostAccumulatedHours: 0、lowMoodDuration: { below40Hours: 0, below20Hours: 0 } 與 isCarePaused: false 等預設值，**嚴禁因版本更新而重置或重新生成既有貓咪的外觀與個性**；若遷移失敗則安全回退至備份檔。（註：v2.1.5 \~ v2.1.28 未變更）

### **8.E.2 核心自動化驗收測試案例 (Vitest Acceptance Criteria)**

RD 完成 Phase A 時必須通過以下 9 項核心測試與照護負擔驗證：

1. **TC-01（基因確定性與 Golden Test）：** 固定 seed: "test-seed-001"、prngAlgorithmVersion: "mulberry32-v1"、geneticsVersion: "1.0.0"，連續生成 10 次結果 100% 一致；對於不同 seed（"test-seed-001" vs "test-seed-002"），驗證 PRNG 抽取序列確實不同並比對預先固定的 Golden Test 預期輸出，不要求不同 Seed 必須產出不同外觀。  
2. **TC-02（特殊骨架雙親與跨品種統計分布）：** 使用獨立固定種子序列 seed\_0001 \~ seed\_1000（每次獨立初始化 PRNG 確保狀態隔離）：munchkin \+ maine\_coon 驗證 short\_legs 佔 50% ± 5%、giant\_fluffy 佔 25% ± 5%、normal 佔 25% ± 5%（允許 short\_legs 與 giant\_fluffy 共存）；munchkin \+ munchkin 驗證 short\_legs 佔 75% ± 5%、normal 佔 25% ± 5%；exotic \+ domestic 驗證 flat\_face 佔 45% ± 5%、standard\_round 佔 55% ± 5%；exotic \+ exotic 驗證 flat\_face 佔 80% ± 5%、standard\_round 佔 20% ± 5%。  
3. **TC-03（時間步進、日夜邊界切割與分段一致性）：** 測試 22:50 \~ 23:05 跨夜邊界自動拆分為 10 分鐘日間（1.0）+ 5 分鐘夜間（0.3）之精確結算驗證；06:45 \~ 07:15 跨日、未滿 15 分鐘零頭累計、24 小時一次補算與 4 次 × 6 小時分段補算結果誤差 \< 1e-9，以及時鐘回撥後恢復正常時間的補算正確性。  
4. **TC-04（健康救援狀態機完整邊界）：** 驗證 health \= 40 不進 CRITICAL、health \= 39 進入 CRITICAL（僅轉換時重置累計時數）；餵食補水達 50/50 進入 RECOVERY\_BOOST（按 actualHours 計算）；中途飢餓跌至 49 暫停並保留已累計時數，重新餵食後繼續累計；當 recoveryBoostAccumulatedHours \>= 2.0 用滿後，即使 health \= 30（仍 \< 40）且未達 60/60/70 亦正確轉為 NATURAL\_RECOVERY 等待照護，絕不卡死或重置累計時數；單步跨越 40 與上限 100 Clamp 驗證。  
5. **TC-05（行為仲裁、滑動窗口與冷卻邊界）：** 驗證 Priority 100 \> 90 \> 80 \> 60 \> 40 \> 10；同觸發 Priority 100 多項條件時 iconIds 合併且主動畫依固定序執行；POKE 10 秒滑動窗口第 4 次觸發 Priority 90，eat 跨過 23:00 不被睡眠中斷。另須完整驗證 Priority 90 結束後 15 秒互動冷卻（冷卻期間再 Poke 不延長冷卻）：(1) 15 秒內 PetCat 可播放動畫但不增加 Mood / 積分；(2) 15 秒內 PlayWithPet 可播放動畫但不增加 Mood / 積分；(3) 冷卻結束後 PetCat 恢復正常獎勵；(4) 冷卻結束後 PlayWithPet 恢復正常獎勵；(5) 冷卻期間 FeedPet 不受互動冷卻限制，照常生效。  
6. **TC-06（傲嬌干預與記憶計算）：** tsundere \>= 70 餵免費乾糧依機率觸發撇頭再吃；驗證最近 7 天互動佔 70% 與歷史 30% 的人格計算公式。  
7. **TC-07（商品探索防重與跨日重置）：** 開局 200 積分，同商品同日重複開啟不給分、跨午夜（23:59 ➔ 00:01）正確重置上限、單日探索上限 30 積分、購買與背包使用扣減及 100 上限 Clamp。  
8. **TC-08（365 天長時穩定性模擬）：** 連續模擬 365 天隨機玩家行為（含 Care Pause 開關、長期未登入、只用免費糧水），驗證無 NaN、Infinity 或負數積分，且存檔序列化與還原一致。  
9. **TC-09（每日照護負擔量化驗收標準 — 固定照護時間與執行順序）：** 測試起點統一設為 Day 1 08:00，初始數值 hunger \= 80, hydration \= 80, mood \= 85, health \= 100，僅使用免費乾糧（+25 hunger, \-2 hydration, \+2 mood）、免費飲水（+32 hydration, \+2 mood）與每日摸頭/玩耍（mood \+3）。每次登入照護指令於該時間點一次連續執行（執行順序：FeedPet ➔ DrinkWater ➔ PetCat/PlayWithPet），每日以次日 08:00 作為完整 24 小時日結點：  
   1. **情境 A（每日早晚 08:00 與 20:00 登入 2 次，每次餵 2 糧 \+ 2 水 \+ 2 次互動）：** 每日總補充 Hunger \+100、Hydration \+120、Mood \+28，每日 08:00 日結時 Hunger \>= 80、Hydration \>= 85、Mood \>= 85，Health 維持 100（零扣血）。  
   2. **情境 B（每日僅 08:00 登入 1 次，標準免費照護方案 B：3 糧 \+ 4 水 \+ 3 次互動 — 驗證低壓力長期陪伴與救援能力）：** 每日 08:00 依序執行 3 份免費乾糧（+75 Hunger, \-6 Hydration, \+6 Mood）➔ 4 份免費飲水（+128 Hydration, \+8 Mood）➔ 3 次撫摸/玩耍互動（+9 Mood），每日總補充量為 Hunger \+75（日衰退 73.6，淨變化 \+1.4）、Hydration \+122（日衰退 73.6，淨變化 \+48.4）、Mood \+23（日衰退 23.0，淨變化 0）（實際狀態須嚴格套用 0\~100 上下限截斷 Clamp，不得僅以淨變化推算）。  
      \- **連續 60 日長期模擬驗收基準（須以 Game Core TimeEngine 實際跑完，輸出每日 08:00 照護前後完整數值、每日各需求跌破 40/20 時長、每日 Health 扣減/回血總量、RecoveryState 轉換時間與第 60 日結算數值）：**  
      **1\. 子情境 B-1（正常起始狀態：Hunger \= 80, Hydration \= 80, Mood \= 85, Health \= 100, RecoveryState \= "NORMAL"）：**  
      \- Day 1 08:00 照護後（套用 100 上限 Clamp）：Hunger \= 100.0、Hydration \= 100.0、Mood \= 100.0、Health \= 100.0。  
      \- Day 2 \~ Day 60 每日 08:00 穩態循環：**照護前** Hunger \= 26.4、Hydration \= 26.4、Mood \= 77.0、Health \= 100.0、RecoveryState \= "NORMAL"、below40Hours \= 0、below20Hours \= 0；**照護後** Hunger \= 100.0、Hydration \= 100.0、Mood \= 100.0、Health \= 100.0。  
      \- 每日風險監控：Hunger 與 Hydration 跌破 40 持續 9.0 小時（23:00 \~ 次日 08:00），跌破 20 持續 **0 小時**；Mood 跌破 40 持續 **0 小時**；每日 Health 扣減 **0**、回血 **0**。第 60 日結束（Day 61 08:00 照護前）維持 Health \= 100.0、RecoveryState \= "NORMAL"。  
      **2\. 子情境 B-2（低健康起始狀態：Hunger \= 30, Hydration \= 30, Mood \= 50, Health \= 10, RecoveryState \= "CRITICAL" — 驗證每日一次免費照護之長期救援能力）：**  
      \- **Day 1 08:00 照護後：** Hunger \= min(100, 30 \+ 75\) \= 100.0、Hydration \= min(100, 30 \- 6 \+ 128\) \= 100.0、Mood \= 50 \+ 23 \= 73.0、Health \= 10.0（RecoveryState \= "CRITICAL"）。  
      \- **Day 1 分段狀態轉換與回血（當日總回血 \+30.5）：**  
        \- 08:00 \~ 10:00（2.0h）：執行 RECOVERY\_BOOST (+10.0/hr) 回血 \+20.0，Health 升至 30.0；於 10:00 耗盡 2.0h Boost 額度並精確轉入 NATURAL\_RECOVERY。  
        \- 10:00 \~ 10:24（0.4h，此時 Mood 由 70.5 降至 70.0，滿足 Mood \>= 70）：以 \+2.5/hr 回血 \+1.0，Health 升至 31.0。  
        \- 10:24 \~ 18:00（7.6h，Mood 介於 40\~69 且 Hunger/Hydration \>= 60）：以 \+1.25/hr 回血 \+9.5，Health 升至 40.5。  
        \- 18:00 \~ 次日 08:00：Hunger/Hydration \< 60 暫停回血，且全程維持 \>= 26.4（\>= 20）故零扣血。Day 2 08:00 照護前 Health \= 10.0 \+ 30.5 \= 40.5、RecoveryState \= "NATURAL\_RECOVERY"。  
      \- **Day 2 \~ Day 4 穩態回血（已處於 NATURAL\_RECOVERY，無 Boost，每日淨回血 \+15.5 Health/日）：** 每日 08:00 照護後 Mood 均回升至 73.0，故 08:00 \~ 10:24（2.4h，Mood \>= 70）回血 2.4 × 2.5 \= \+6.0，10:24 \~ 18:00（7.6h，Mood 40\~69）回血 7.6 × 1.25 \= \+9.5，每日合計 \+15.5 且零扣血。各日 08:00 照護前精確為：Day 3 08:00 達 56.0（40.5 \+ 15.5）、Day 4 08:00 達 71.5（56.0 \+ 15.5）、Day 5 08:00 達 87.0（71.5 \+ 15.5）。  
      \- **Day 5 滿血重置驗證：** Day 5 08:00 起算（Health \= 87.0），經 08:00 \~ 10:24（2.4h × \+2.5/hr \= \+6.0）達 93.0；接續 10:24 \~ 16:00（5.6h × \+1.25/hr \= \+7.0），於 Day 5 16:00 精確達成 Health \= 93.0 \+ 7.0 \= 100.0 並自動重置為 RecoveryState \= "NORMAL"、recoveryBoostAccumulatedHours \= 0。  
      \- **Day 6 \~ Day 60：** 全程維持 Health \= 100.0、RecoveryState \= "NORMAL"（每日 08:00 照護前 Hunger \= 26.4、Hydration \= 26.4、Mood \= 50.0、below40Hours \= 0、below20Hours \= 0，每日 Health 扣減 0），證明每日一次免費照護具備完整長期救援與維持健康能力。  
   3. **3\. 子情境 B-3（長期僅於 08:00 餵食飲水但不陪伴情境驗證 — 隔離飢餓與缺水懲罰，依日間 1.25/hr、夜間 0.375/hr 實際衰退率與 TimeEngine 交點精確模擬）：** 初始 Fixture 明確指定為 startTime \= Day 1 08:00、hunger \= 80、hydration \= 80、mood \= 85、health \= 100、recoveryState \= "NORMAL"、lowMoodDuration \= { below40Hours: 0, below20Hours: 0 }。若玩家連續 10 日每日僅於 08:00 餵 3 糧 \+ 3 水但不互動（因 Hunger 與 Hydration 日衰退均為 73.6，3 糧 \+ 3 水使兩者每日照護後均達 100.0、次日照護前均為 26.4，全程嚴格維持 \>= 20，**完全隔離飢餓與缺水對 Health 及 Mood 的干擾**；每日 08:00 瞬間增加 Mood \+12，隨後 24 小時基礎衰退 Mood \-23，各日 08:00 照護後 Mood 依序為 Day 1: 97、Day 2: 86、Day 3: 75、Day 4: 64、Day 5: 53、Day 6: 42、Day 7: 31、Day 8: 20、Day 9: 12、Day 10: 12；不可僅用每日平均衰退量推算，須以 Game Core TimeEngine 實際模擬結果為最終驗收依據）：  
      1\. **階段 1（無扣血期，Day 1 08:00 \~ Day 9.067）：** Day 5 晚間雖曾短暫跌破 40，但 Day 6 08:00 照護後 Mood 拉回 42（重置 below40Hours \= 0），隨後依日間衰退率 1.25/hr 經 1.6 小時於 **Day 6.067**（Day 6 09:36）再次跌破 40（開始連續累計 below40Hours）；雖於 Day 6 末段短暫跌破 20，但 **Day 7 08:00** 的餵食飲水脈衝將 Mood 拉回 31，使 below20Hours 立即歸零重置（below40Hours 則持續累計）；直到 **Day 8 08:00** 脈衝後 Mood \= 20（重置 below20Hours \= 0）隨即再次跌破 20，才開始連續累計 below20Hours。在 below40Hours 未滿 72h 前全程零扣血。  
      2\. **階段 2（一般低落扣血期，Day 9.067 \~ Day 10.000）：** 當 **Day 9.067**（Day 9 09:36）below40Hours 精確跨越 72.0 小時後（嚴格 \> 72.0）且 below20Hours 尚未超過 48.0 小時（\<= 48.0）之區間（共 **22.4 小時**），從該交點之後的時間區段開始啟動 **\-0.05/hr** 緩慢扣血（此階段累計扣除 **1.12** 點 Health）。  
      3\. **階段 3（嚴重低落扣血期與 40 安全地板，Day 10.000 起）：** 當 **Day 10.000**（Day 10 08:00 結算交點）below20Hours 精確跨越 48.0 小時後（嚴格 \> 48.0），從該交點之後的時間區段開始切換為 **\-0.1/hr**（每日 \-2.4 點 Health，取代 \-0.05/hr 不疊加），且受 Health \>= 40 安全地板保護（絕不會因不陪伴而進入 CRITICAL 或死亡），玩家回來陪伴玩耍與使用商品即可輕鬆回滿。  
10. **情境 C（連續 3 日未登入後的初始救援，以及後續每日早晚照護恢復驗收 — 方案 A 持續照護劇本）：** 初始救援階段模擬玩家連續 3 日未登入後，於 Day 1 08:00 返回並執行一次初始救援照護（本情境之「連續 3 日未登入」僅描述初始救援前的離線狀態，不代表後續恢復階段仍維持每 3 日登入一次）。登入時貓咪處於 Hunger \= 0, Hydration \= 0, Mood \= 0, Health \= 10, recoveryState \= "CRITICAL"。玩家於 Day 1 08:00 連續執行初始救援照護（3 免費乾糧 \+ 3 免費飲水 \+ 2 凍乾原肉丁零食 \+ 10 次陪伴互動），照護後立即為 Hunger \= 0 \+ (3×25) \+ (2×8) \= 91 (\>= 60)、Hydration \= 0 \- (3×2) \+ (3×32) \- (2×1，含 2 份凍乾零食扣水) \= 88 (\>= 60)、Mood \= 0 \+ (3×2) \+ (3×2) \+ (2×14) \+ (10×3) \= 70 (\>= 70)。  
    \- **階段 1（初始救援與 2 小時急速脫險）：** 經過 2 小時（Day 1 08:00 \~ 10:00）RECOVERY\_BOOST (+10.0/hr)，Health 從 10 恢復至 30.0（此時耗盡 2.0h Boost 額度並精確於 10:00 交點轉為 NATURAL\_RECOVERY）。  
    \- **階段 2（自 Day 1 20:00 起之每日早晚持續恢復與自然回血暫停規則）：** 完成初始救援及 RECOVERY\_BOOST 後，自 Day 1 20:00 起進入持續恢復階段，每日固定於 08:00 與 20:00 各執行 2 份免費乾糧、2 份免費飲水及 2 次互動照護（單次補充 Hunger \+50、Hydration \+60、Mood \+14），直到 Health 達到 100。每次照護後，必須依 TimeEngine 實際計算 Hunger、Hydration 與 Mood，並依 RecoveryState 規則判斷是否可以回血；**不得假設每日早晚照護即可使 Hunger / Hydration 全程維持在 60 以上**（例如 08:00 照護達 100 後，日間每小時衰退 4.0，至 18:00 降至 60、18:00 \~ 20:00 低於 60）。當 Hunger \< 60 或 Hydration \< 60 時，若處於 NATURAL\_RECOVERY，必須暫停 Health 回復；當兩項需求均恢復至 60 以上時，立即依當下 Mood 區間恢復回血。  
    \- **完整 24 小時（含夜間）Recovery 計算要求：**  
    **1\.** 每次照護後，TimeEngine 必須依照日夜時段切割，逐段計算需求衰退、Mood 變化及 Health Recovery。  
    2\. Hunger \< 60 或 Hydration \< 60 時，暫停 NATURAL\_RECOVERY；當兩者均恢復至 60 以上時，立即依當下 Mood 區間恢復回血。  
    3\. **夜間 23:00 \~ 07:00 不得因為是夜間而自動暫停 Recovery。** 只要 Hunger、Hydration 符合門檻，且 RecoveryState 允許回血，就必須依當下 Mood 區間持續恢復。  
    4\. 每次 08:00 與 20:00 照護指令必須先完成此前時間區間的模擬，再依序執行 FeedPet ➔ DrinkWater ➔ PetCat/PlayWithPet。  
    5\. Health 達到 100 時，立即 Clamp 為 100，RecoveryState 重置為 NORMAL，recoveryBoostAccumulatedHours 歸零。  
    6\. 測試輸出必須完整記錄：(a) 每日 08:00、20:00 照護前後的 Hunger、Hydration、Mood、Health；(b) 每個日夜區段的起訖時間、各數值、RecoveryState、實際回血時數及回血量；(c) 每日 Hunger \< 60 或 Hydration \< 60 的起訖時間及累積暫停時數；(d) RecoveryState 每次轉換的精確時間；(e) Health 首次達到 100 的精確日期與時間（若未於預期時間內達到 Health 100，必須列出原因，不得以理論回血量或忽略夜間恢復的手算結果取代實際模擬）。  
    \- **逐段實際模擬參考時間表（最終驗收以 Game Core TimeEngine 實際執行輸出為準）：**  
      **\- Day 1 08:00 \~ 10:00（2.0h，日間）：** RECOVERY\_BOOST (+10.0/hr) 回血 \+20.0 ➔ Health \= 30.0（10:00 轉入 NATURAL\_RECOVERY；結束時 Hunger 83、Hydration 80、Mood 67.5）。  
      \- **Day 1 10:00 \~ 15:00（5.0h，日間）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+6.25 ➔ Health \= 36.25（15:00 時 Hydration 降至 60）。  
      \- **Day 1 15:00 \~ 15:45（0.75h，日間）：** Hydration \< 60（15:00 Hydration 降至 60、Hunger 為 63），**NATURAL\_RECOVERY 開始暫停**（15:45 時 Hunger 亦降至 60.0、Hydration 為 57.0、Mood 為 60.3125、Health \= 36.25）。  
      \- **Day 1 15:45 \~ 20:00（4.25h，日間）：** Hunger 與 Hydration 均 \< 60，**NATURAL\_RECOVERY 持續暫停**（Day 1 累積暫停 5.0h：Hydration \< 60 為 15:00 \~ 20:00 共 5.0h、Hunger \< 60 為 15:45 \~ 20:00 共 4.25h；結束時 Hunger 43、Hydration 40、Mood 55.0、Health \= 36.25）。  
      \- **Day 1 20:00 照護後：** Hunger \= 93、Hydration \= 100、Mood \= 69.0、Health \= 36.25。  
      \- **Day 1 20:00 \~ 23:00（3.0h，日間）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+3.75 ➔ Health \= 40.0（結束時 Hunger 81、Hydration 88、Mood 65.25）。  
      \- **Day 1 23:00 \~ Day 2 07:00（8.0h，夜間持續回血）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+10.0 ➔ Health \= 50.0（結束時 Hunger 71.4、Hydration 78.4、Mood 62.25）。  
      \- **Day 2 07:00 \~ 08:00（1.0h，日間）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+1.25 ➔ Health \= 51.25（照護前 Hunger 67.4、Hydration 74.4、Mood 61.0）。  
      \- **Day 2 08:00 照護後：** Hunger \= 100、Hydration \= 100、Mood \= 75.0、Health \= 51.25。  
      \- **Day 2 08:00 \~ 12:00（4.0h，日間）：** 需求均 \>= 60、Mood \>= 70 (+2.5/hr) 回血 \+10.0 ➔ Health \= 61.25（12:00 時 Mood 降至 70.0）。  
      \- **Day 2 12:00 \~ 18:00（6.0h，日間）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+7.5 ➔ Health \= 68.75（18:00 時 Hunger/Hydration 均降至 60.0）。  
      \- **Day 2 18:00 \~ 20:00（2.0h，日間）：** Hunger/Hydration \< 60，**NATURAL\_RECOVERY 暫停 2.0h**（結束時 Hunger 52、Hydration 52、Mood 60.0、Health \= 68.75）。  
      \- **Day 2 20:00 照護後：** Hunger \= 100、Hydration \= 100、Mood \= 74.0、Health \= 68.75。  
      \- **Day 2 20:00 \~ 23:00（3.0h，日間）：** 需求均 \>= 60、Mood \>= 70 (+2.5/hr) 回血 \+7.5 ➔ Health \= 76.25（結束時 Hunger 88、Hydration 88、Mood 70.25）。  
      \- **Day 2 23:00 \~ 23:40（2/3 h，夜間持續回血）：** 需求均 \>= 60、Mood \>= 70 (+2.5/hr) 回血 \+1.6667 ➔ Health \&approx; 77.9167（23:40 時 Mood 降至 70.0）。  
      \- **Day 2 23:40 \~ Day 3 07:00（22/3 h，夜間持續回血）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+9.1667 ➔ Health \&approx; 87.0833（結束時 Hunger 78.4、Hydration 78.4、Mood 67.25）。  
      \- **Day 3 07:00 \~ 08:00（1.0h，日間）：** 需求均 \>= 60、Mood 40\~69 (+1.25/hr) 回血 \+1.25 ➔ Health \&approx; 88.3333（照護前 Hunger 74.4、Hydration 74.4、Mood 66.0）。  
      \- **Day 3 08:00 照護後：** Hunger \= 100、Hydration \= 100、Mood \= 80.0、Health \&approx; 88.3333。  
      \- **Day 3 08:00 \~ 12:40（14/3 h，日間）：** 需求均 \>= 60、Mood \>= 70 (+2.5/hr) 回血 \+11.6667 ➔ 於 **Day 3 12:40** 精確達成 **Health \= 100.0**（此時 Hunger \&approx; 81.33、Hydration \&approx; 81.33、Mood \&approx; 74.17），立即 Clamp 為 100 並重置為 RecoveryState \= "NORMAL"、recoveryBoostAccumulatedHours \= 0。

### **8.E.3 Mood-Health 混合模式專屬自動化驗收測試清單 (TC-MH-01 \~ TC-MH-12)**

11. **TC-MH-01（Mood 40+ 無扣血）：** 驗證 Mood \= 50 持續 7 天，Health 維持 100，below40Hours 與 below20Hours 均保持 0。  
12. **TC-MH-02（Mood 20\~39 緩慢扣血與 72h 門檻）：** 驗證 Mood \= 30 時，前 72 小時 Health 不扣減；超過 72 小時後每小時扣減 0.05 Health。  
13. **TC-MH-03（Mood 0\~19 扣血與 48h 門檻）：** 驗證 Mood \= 10 時，前 48 小時 Health 不扣減；超過 48 小時後每小時扣減 0.1 Health（不與 \-0.05 疊加）。  
14. **TC-MH-04（Health 40 安全地板 — 隔離測試）：** 在保持 Hunger \>= 60 且 Hydration \>= 60（確保不觸發飢餓/缺水扣血）的前提下，設定初始 Health \= 40.5 且 Mood \= 10 持續滿 48h 後再推進 10 小時，驗證當 Health 降至 40.0 的交點時 MoodHealthPenalty 扣減量強制歸 0，最終 Health 精確停在 40.0，絕不降至 39.95 或低於 40。  
15. **TC-MH-05（低心情扣血與飢餓扣血獨立疊加、40 地板邊界與 10 下限斷言）：** 測試起點設為 Day 1 `08:00`，初始狀態明確指定為 `health = 50.0`、`hunger = 10.0`（`&lt; 20`，單獨觸發 `-2.0/hr` 飢餓扣血）、`hydration = 100.0`（全程維持 `&gt;= 20`，隔離缺水影響以僅測 `hunger &lt; 20`）、`mood = 10.0`（`&lt; 20`），且低心情計時器起始位於門檻邊界 `lowMoodDuration = { below40Hours: 72.0, below20Hours: 48.0 }`（依 8.C.1 規則，推進時間使 `below20Hours &gt; 48.0` 時立即自跨過 48 小時門檻後起算 `-0.1/hr`，非將等於 48 視為已超過門檻）。連續推進 **20.0 小時**（Day 1 `08:00` ➔ Day 2 `04:00`，結束時 `hydration = 34.0 &gt;= 20`）：(1) **階段 1（0 \~ 10/2.1 \&approx; 4.7619 小時）**：`health &gt; 40` 區間內總扣血速度為 `-2.1/hr`，於約 `4.7619 小時` 精確抵達 `health = 40.0`；(2) **階段 2（4.7619 \~ 19.7619 小時，共 15.0 小時）**：抵達 `health = 40.0` 交點後，`MoodHealthPenalty` 立即停止（變為 `0`），僅 `hunger &lt; 20` 繼續以 `-2.0/hr` 扣血，使 `health` 正常跌入 `CRITICAL`（`&lt; 40`）並於累計約 `19.7619 小時` 精確抵達保護下限 `health = 10.0`；(3) **階段 3（19.7619 \~ 20.0 小時）**：斷言 `20.0 小時` 結束時 `health` 嚴格維持於保護下限 `10.0`，不得低於 `10`。  
16. **TC-MH-06（Mood 恢復時立即重置計時器）：** 驗證 below40Hours 累計至 50 小時後，透過撫摸使 Mood 升至 45，below40Hours 立即歸零；再次降至 30 時需重新累計 72 小時才開始扣血。  
17. **TC-MH-07（NATURAL\_RECOVERY 分級速度 Mood \&gt;= 70）：** 驗證處於 NATURAL\_RECOVERY 且 Hunger \&gt;= 60, Hydration \&gt;= 60, Mood \= 75 時，恢復速度為 \+2.5/hr。  
18. **TC-MH-08（NATURAL\_RECOVERY 分級速度 Mood 40\~69）：** 驗證處於 NATURAL\_RECOVERY 且 Hunger \&gt;= 60, Hydration \&gt;= 60, Mood \= 50 時，恢復速度為 \+1.25/hr。  
19. **TC-MH-09（NATURAL\_RECOVERY 暫停 Mood \&lt; 40）：** 驗證處於 NATURAL\_RECOVERY 且 Hunger \&gt;= 60, Hydration \&gt;= 60, Mood \= 30 時，恢復暫停（+0/hr）。  
20. **TC-MH-10（離線模擬分段一致性）：** 驗證一次模擬 120 小時（跨越 72h 低心情門檻與 40 安全地板）與分 10 次每次 12 小時模擬，最終 Health、Mood、lowMoodDuration 結果誤差 \&lt; 1e-9。  
21. **TC-MH-11（存檔與讀檔持久化）：** 驗證 lowMoodDuration（below40Hours 與 below20Hours）跨越存檔與重啟生命週期時完整保留並正確續算（以下數值為測試 fixture，非新增遊戲規則或調整門檻）：  
    \- **測試前置狀態（Fixture）：** 測試起點設為 Day 1 08:00，health \= 100、mood \= 10（確保推進期間維持 Mood \&lt; 20，兩項計時器同步累加）、hunger \= 80、hydration \= 80（隔離飢餓/缺水干擾），且 lowMoodDuration \= { below40Hours: 55.5, below20Hours: 23.5 }。  
    \- **測試步驟與斷言：** (1) 建立上述初始狀態；(2) 寫入存檔；(3) 關閉並重新載入存檔；(4) 斷言讀檔後 below40Hours \=== 55.5 且 below20Hours \=== 23.5（誤差 \&lt; 1e-9）；(5) 推進模擬時間 1.0 小時；(6) 斷言推進後 below40Hours \=== 56.5 且 below20Hours \=== 24.5（由原累積值續算而非歸零，誤差 \&lt; 1e-9）。  
    \- **分階段 Adapter 執行時程： Phase A** 以 JSON Adapter 執行 TC-MH-11（不要求提前實作 SQLite）；**Phase C** 待 SQLite Adapter 完成後，以相同測試案例與驗收條件執行；兩種 Adapter 皆須保證 lowMoodDuration 之序列化與反序列化結果完全一致。  
22. **TC-MH-12（夜間係數不影響低心情扣血）：** 驗證夜間 (23:00\~07:00) 觸發低心情扣血時，扣血速率維持 \-0.05/hr 或 \-0.1/hr，不受 0.3 時段係數減免。

# **9\. 產品範圍與非目標 (Scope & Non-Goals)**

20. 單隻貓咪領養與 7 層確定性外觀生成、三層性格行為決策、0.25 小時步進離線時間差補算、健康救援狀態機、無懲罰 Care Pause 照護暫停模式、防損本機存檔與 Save Migration、Android 全域懸浮窗、iOS App 內養成與狀態 Widget、積分商店與商品營養分析探索。

## **9.2 MVP 明確排除 (Non-Goals，保留介面但延後至 Phase E)**

21. 多隻貓咪同時飼養、即時背景常駐耗電服務、伺服器端防作弊結算、好友即時連線與串門子、GPS/LBS 戶外散步與跳蚤系統、自動截圖相簿、LLM 自由對話與付費月卡。

# **10\. 隱私、權限與錯誤處理降級規範 (Privacy & Fallback)**

## **10.1 最小權限與資料隱私規範**

22. Android 僅於完成領養後引導授權 SYSTEM\_ALERT\_WINDOW（拒絕授權仍可於 App 內正常遊玩）；計步器與定位權限於 MVP 階段完全不申請。  
23. 問卷與統計僅於使用者明確同意下收集不可識別個人的匿名彙總數據，嚴禁販售個別使用者行為軌跡或定位紀錄。

## **10.2 異常降級與 Fallback 策略**

1. **遠端設定檔失敗：** 遠端設定載入功能於 Phase D 配合商店與品牌商品目錄導入 Remote Config Provider 後啟用（Phase A \~ C 不實作遠端下載）：  
   1. 網路中斷時，使用已驗證的本機快取設定。  
      2. 遠端 JSON 下載失敗或 Schema 驗證失敗時，不得將未驗證資料注入 Game Core。  
      3. 本機快取不存在或無法通過驗證時，使用內建且已驗證的預設設定檔。  
      4. 若無任何可用的有效設定檔，應停止載入依賴該設定的功能並回報明確錯誤，不得以不完整資料繼續執行。  
      5. Game Core 不得直接連線或依賴特定遠端服務；設定來源由 Remote Config Provider 管理。  
   2. **系統時間倒退異常：** 若偵測到裝置時間倒退（now \< lastSimulatedAt），TimeEngine 忽略負時間差（Δt \= 0）並記錄異常時間事件，**嚴禁覆寫或修改原本的 lastSimulatedAt**，待裝置時間恢復正常區間後再行補算，防止時間回撥造成異常暴增補算。  
   3. **權限撤銷：** 若使用者於系統設定撤銷 Android 懸浮窗權限，App 自動切換為「App 內模式」並保存進度，不產生閃退。

# **11\. 版本控制與開發工作拆分 (Versioning & Sprint Plan)**

## **11.1 獨立版本號管理**

24. 系統將 gameCoreVersion、prngAlgorithmVersion、geneticsVersion、contentVersion 與 saveSchemaVersion 獨立分開管理，確保更新商品目錄或調整數值平衡時，不會破壞既有貓咪外觀或玩家存檔。

## **11.2 Sprint 執行順序**

25. **Sprint 1（Phase A \- Windows Core 骨架與基因引擎）：** 建立 pnpm Monorepo、contracts、config-loader（Local 模式）、PetState 模型與 GeneticsEngine（通過 TC-01、TC-02）。  
26. **Sprint 2（Phase A \- 時間、健康救援與經濟引擎）：** 實作 TimeEngine（0.25 小時區段步進與時間倒退保護）、健康救援狀態機、EconomyEngine、背包系統與防損 JSON 存檔（通過 TC-03、TC-04、TC-07）。  
27. **Sprint 3（Phase A \- 行為決策、CLI 驗收與長時模擬）：** 實作 BehaviorEngine 三層決策與同級仲裁、CLI 互動操作介面、365 天長時模擬與每日照護負擔驗證（通過 TC-05、TC-06、TC-08、TC-09）。  
28. **Sprint 4（Phase B \- 雙平台渲染 PoC）：** 建立單隻測試貓，驗證 React Native、Android Kotlin 懸浮窗與 iOS SwiftUI Widget 消費同一份 PetViewModel 之一致性。

