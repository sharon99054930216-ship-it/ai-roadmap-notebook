## part15 Agents
## 名詞理解
1.ReAct loop - 骨架  
Thought (思考階段):LLM 結合當前的上下文歷史（History）與可用工具清單（Tools Spec），分析目前距離目標還差什麼，並規劃下一步的策略。  
Action (行動階段):若大腦判斷需要外部資訊（例如查天氣、搜資料、跑程式碼），就會精準輸出一組具體的 Tool Call（包含工具名稱與參數）。  
Observation (觀察環境回饋):系統執行工具後，將真實世界的反應結果（如 API 回傳的 JSON、網頁文字、程式報錯訊息）寫回對話歷史中。  
Final Answer (終止與輸出):當 LLM 判斷透過觀察到的資訊已經足以回答使用者的原始問題時，便會跳出循環，給出最終結案的 final_answer。  
2.Tool  
標準三要素  
Name (名稱)：簡潔明確的工具識別碼（例如 get_weather、execute_sql）。  
Description (描述)：全域最關鍵的靈魂。詳細說明該工具的用途、適用情境、限制及輸入範例。  
JSON Schema (參數定義)：嚴格規範輸入參數（args）的型態、必填欄位與格式。  
現代 LLM 的原生支援  
透過主流模型的 Function Calling API（如 OpenAI Tools、Anthropic Tool Use），模型能夠在思考過程中直接輸出符合 Schema 的結構化 JSON 呼叫，不再需要透過脆弱的 Regex 去字串解析。  
實務:
Tool Description 決定生死:描述必須極度精準，寫明「什麼時候用、能解決什麼問題、輸入的邊界條件是什麼」，把它當成給新手工程師看的 API 文件來寫。  
工具過多導致的混淆:當工具數量龐大時，必須引入 Router (路由機制) 或 Tool Grouping (工具分組)，先由高層 LLM 判斷任務屬於哪個領域，再動態載入該領域的子工具集。  
長執行時間的痛點:必須導入 Streaming Feedback / Async Execution，讓工具在執行中途定期回傳狀態（例如：「正在解析第 3/10 個網頁...」），維持 Agent 迴圈的連貫性。  
智慧容錯與自我修正:將工具的 Error Traceback 直接餵回對話歷史，引導模型讀取錯誤訊息、自主調整參數（例如把錯誤的日期格式改掉）後再次發動請求。  
3.MCP  
MCP 的核心願景：就像電腦硬體的 USB-C 標準一樣，定義了一套統一的介面協定，讓任何 AI 客戶端（Client）都能無縫接上任何外部服務器（Server）。  
核心架構:  
MCP Client (客戶端):即具備 AI 能力的應用程式或環境（例如 Claude Desktop、Cursor、各種 IDE 插件或自主 Agent 框架）。負責發起請求與調用。  
MCP Server (服務器):封裝了特定功能、資料庫或第三方工具的輕量化服務（例如官方或社群提供的 Filesystem、GitHub、Slack、Postgres、Playwright 瀏覽器自動化等）。  
通訊協定：JSON-RPC:客戶端與服務器之間透過輕量級的 JSON-RPC 標準進行溝通，傳輸通道支援 stdio（標準輸入輸出，適合本機端安全執行）或 HTTP / SSE（適合遠端雲端服務）。  
MCP 具有強大的商業與工程價值:
開發者只要寫一次某個 SaaS 工具（如 GitHub 專案管理）的 MCP Server，全世界所有的 Claude Desktop、Cursor 使用者或各種 Agent 框架就都能直接拿來用，不需要重複開發。  
4.Multi-agent   
Planner + Worker (規劃者與執行者):高層 Agent 負責將大任務拆解成子任務，多個底層 Worker Agent 平行或依序執行具體細節。  
Debate & Critic (辯論與批判審查):讓不同的 Agent 扮演對立角色（如正反方辯論，或生成者與審查者），透過多輪交鋒來找出程式碼漏洞或論述盲點。  
Supervisor / Swarm (主管與蜂群調度):引入一個總管 Agent（Supervisor），根據當前進展動態決定把任務交給下轄的哪一個專業子 Agent（如翻譯、搜尋、計算專家）。  
多代理協同真正贏的場景:  
Parallel Exploration (大規模平行探索):  
情境：深度研究報告（Research）、海量網頁爬蟲或大型代碼庫檢索（Code Search）。  
優勢：把任務拆成 10 個獨立子方向，分派給 10 個 Worker 同時往外擴散搜尋，最後再由總管收斂，大幅縮短牆鐘時間（Wall-clock time）。  
Strong Adversarial Roles / Red Teaming (強對立角色與紅隊測試):  
情境：資安漏洞挖掘、合約漏洞審查、攻擊性 AI 測試（Red Teaming）。  
優勢：必須刻意引入一個「不擇手段想找出漏洞」的攻擊型 Agent，與「嚴格防守」的防禦型 Agent 進行極端的對抗性互動，才能逼出單一模型想不到的極端邊界條件（Edge Cases）。  
5.Memory  
Agent 記憶的四大分層  
Scratchpad (草稿紙 / 短期記憶):存在於 Context Window 內，隨著任務結束或超過長度限制而清除，是 ReAct 迴圈正常運作的基礎。  
Episodic Memory (情境記憶 / 歷史經驗):讓 Agent 能夠從「過去失敗的踩坑經驗」中學習，下次遇到相似情境時知道該避開哪些錯誤。  
Semantic Memory (語義記憶 / 知識與偏好):通常透過向量資料庫（Vector DB）來儲存與檢索，提供跨對話的背景知識。  
Procedural Memory (程序記憶 / 標準工作流):類似人類的肌肉記憶或 SOP，讓 Agent 面對標準任務時可以調用預先寫好的執行步驟，減少即興發揮的錯誤率。  
主流記憶管理框架 (Memory Frameworks)  
Letta (前身為 MemGPT):借鑒作業系統的「虛擬記憶體（Virtual Memory）」概念，將 Agent 的記憶分為核心記憶（Core Memory，常駐在 Context 中）與外部存儲（Recall / Archival Memory），支援主動讀寫、搜尋與分頁管理。  
mem0:專注於為 LLM 應用打造輕量且智慧的「持續性記憶層（Personalized Memory Layer）」，能夠在多輪對話與不同任務中自動萃取並更新使用者的偏好。  
6.框架  
原生 API 裸寫 (Native SDKs) vs. 框架迷思:    
2025+ 主流：擁抱原生 SDK 裸寫  
作法：直接透過 OpenAI、Anthropic 或 Gemini 的原生 Function Calling API 寫迴圈。  
優勢：完全掌握控制權、沒有框架的黑盒子、除錯直覺，且隨著模型本身的智慧提高，許多原本需要複雜框架處理的邏輯，現在單靠原生 API 加上好 Prompt 就能完美解決。  
業界的黃金勸告：切勿過早導入框架 (Don't Over-engineer Early)  
黃金守則：「先親手寫 50 行的 Raw ReAct Loop」。當你真正踩過 Tool Call 解析、狀態傳遞與錯誤處理的坑、理解其底層運作後，再去評估是否真的需要複雜的框架。  
現代主流 Agent 開編排框架總覽:    
LangGraph:高度複雜、具備多條件分支、強大循環與人機協同（Human-in-the-loop）的精細化 Agent Flow。  
Pydantic AI:追求極佳開發體驗、強型別約束、減少執行期型別錯誤的開發團隊。  
OpenAI Agents SDK / Swarm:快速實驗、探索多代理（Multi-agent）協同的輕量解決方案。  
LlamaIndex Agents:任務高度依賴企業內部 RAG 檢索與文件處理的應用。  
7.Eval  
SWE-bench / SWE-bench Verified:給定真實的 GitHub Issue 與對應的程式碼庫（Repository），要求 Agent 自行分析、定位問題、修改程式碼並產出正確的 Patch（修補檔）。  
WebArena / VisualWebArena:在模擬的真實網站環境中（如電商平台、討論區、企業內部系統），執行複雜的網頁操作任務（例如：「幫我找到特定價格區間的筆電並完成下單」）。  
τ-bench / τ²-bench:模擬真實的客服場景與多輪對話，Agent 必須結合後台工具（如查詢訂單、處理退款、修改帳戶資料）來協助客戶解決問題。  
GAIA (General AI Assistants):設計真實世界中人類會交辦的綜合性多步任務，橫跨多模態檔案處理（PDF、試算表、影音）、網頁搜尋、工具組合與邏輯推理。  
OSWorld:在真實的作業系統環境（如 Linux / Ubuntu）中，透過完整的 GUI、終端機（Terminal）和檔案系統來跨應用程式完成任務（例如：「把下載資料夾裡的圖片轉檔，並寄給指定聯絡人」）。  
8.名詞表  
ReAct:將 Thought（思考）、Action（行動）、Observation（觀察）三元組交錯循環的經典 Agent 骨架，是所有自主代理的鼻祖。  
Function Calling:由 OpenAI 於 2023 年引入、現已成主流標配的 API 機制。讓 LLM 能直接輸出結構化的工具呼叫參數，取代過去脆弱的字串解析。  
MCP:Anthropic 推出的開放標準。將 Agent 與工具的串接標準化（如同 AI 生態的 USB-C），透過 JSON-RPC 實現隨插即用的模組化生態。  
ACI:專門為 LLM 設計的精簡 API 介面。透過降低複雜度來節省 Token 並大幅減少模型的犯錯率（如 SWE-agent 的實踐）。
Subagent (子智慧體):由主 Agent（Primary Agent）根據任務需求動態派發子任務給專職的子 Agent 進行分工（如 Claude Code 中的深度應用）。  
Compute Use (計算使用 / OS 操控):Anthropic 推進的前沿能力。讓大模型直接解析畫面截圖、自主操控滑鼠與鍵盤來完成作業系統層級的全自動任務。  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
AI 應用正從單純的「被動問答 Chatbot」急速躍升為能夠自主規劃、呼叫外部工具、操作電腦與執行複雜任務的「Autonomous Agents（自主系統）」。  
工業界出現了標準化破局的嘗試（如 Anthropic 推出的 MCP 協議，實現 Agent 與工具的隨插即用），以及針對高難度軟體工程與瀏覽器操作的權威基準（如 SWE-bench、OSWorld）。  
「工具的描述（Tool Description）與介面設計往往比演算法本身更決定成敗」。當工具數量超過一定規模時，若缺乏良好的路由（Routing）或 ACI 優化，模型的注意力會迅速崩潰。  
(b)
錯誤累積與級聯放大 (Error Cascading):在自動化迴圈中，一旦某一步的工具調用或參數解析出現輕微失誤，錯誤的 Observation 被寫回 History，後續的推理便會在錯誤基礎上繼續發散，缺乏強健的自動校正機制。  
工具過載與注意力渙散 (Tool Overload & Context Bloat):當系統掛載的工具數量過多時，LLM 容易產生選擇困難或混淆，經常選錯 API 或帶錯參數，暴露出當前動態工具路由與選擇機制的侷限性。  
複雜多代理架構的性價比陷阱 (Multi-Agent Overhead):盲目追求多代理分工（Planner-Worker、Debate 等模式）往往導致 Token 消耗量呈指數級暴增，且推論延遲高到無法實際落地，反而不如結構扎實的單一 Agent 搭配精準 Prompt 來得穩定高效。  




## part16 World Models
## 名詞理解
1.Dreamer  
RSSM:Dreamer 系列的核心是 RSSM。模型在內部學習一個壓縮的潛在狀態空間（Latent Space z_t），並建立動態轉移方程式：z_{t+1} = f(z_t, a_t)透過這個動態模型，AI 能夠在腦海中模擬當採取動作 a_t 後，潛在狀態會如何演變至 z_{t+1}。  
傳統 RL（如 PPO、DQN）必須在真實環境中進行大量的試錯互動，樣本效率（Sample Efficiency）極低。  
Dreamer 的解法:  
想像力訓練 (Latent Rollout)：AI 不直接與真實環境互動，而是直接在學好的 Latent Space 模擬器中進行想像推演（Rollout），並在腦海中優化其決策策略（Policy）。  
效益：大幅降低對真實環境數據的依賴，讓機器人或智慧體能夠在「腦內世界」快速迭代與學習。  
2.Genie的無監督學習解法  
零動作標籤 (Unlabelled Gameplay Videos)：僅僅透過觀看網路上大量的遊戲畫面影片（甚至不需要知道操控手把的按鍵），Genie 就能透過自監督學習（Self-supervised Learning）自動推導並學出一組隱含動作標記（Latent Action Tokens）。  
可控生成 (Controllable Generation)：學會隱含動作後，模型就能夠接受人類的動態指令，逐幀生成出符合物理互動的互動式環境（Playable Environments）。  
Genie 2 邁向 3D 與長時序模擬  
Genie 初代多半應用在規則相對單純的 2D 遊戲或切格畫面。  
Genie 2（2024 年末升級） 將這項技術大幅推進，能夠支援 3D 場景、高度真實的光影物理、以及分鐘級的長時序（Long-horizon）生成。  
它可以根據一段文字描述或初始圖片，生成出一個讓使用者可以自由探索、駕駛、奔跑的沈浸式 3D 互動世界。 
徹底解決數據稀缺痛點:  
這項技術最大的顛覆性在於：任何人類活動的影片（例如自動駕駛紀錄片、體育賽事、機器人操作影片）都可以直接拿來訓練世界模型。  
不需要額外的感測器標註，世界模型就能從海量的「視覺歷史」中學會預測未來，為機器人訓練（Robot Learning）與下一代遊戲引擎開闢了極具想像力的全新道路。  
3.NVIDIA Cosmos  
這是 NVIDIA 針對機器人（Robotics）與自動駕駛（Autonomous Driving）所推出的世界基礎模型平台，被黃仁勳形容為實體 AI 領域的「ChatGPT 時刻」。  
兩大核心:  
合成資料引擎 (Synthetic Data Engine)：為機器人與自駕車訓練產生大量逼真的物理世界視訊數據，解決現實資料不足的痛點。  
策略想像與模擬 (Policy Imagination Sim)：讓 AI 機器人在內建的物理世界模擬器中進行想像與試錯，加速決策模型的訓練。  
技術架構:  
雙重架構變體:同時提供 擴散模型（Diffusion） 與 自迴歸（Autoregressive） 兩種主流視訊生成架構，滿足不同任務對生成速度與物理一致性的需求。  
海量規模預訓練:採用高達 2000 萬小時（20 Million Hours） 的多元影片資料進行預訓練，讓模型內建深刻的 3D 空間概念、光影流動與基礎物理定律（如重力、碰撞、動量）。  
4.V-JEPA  
Meta AI 首席科學家 Yann LeCun 長期公開批評如 Sora、Genie 這類「逐像素生成（Pixel-level generation）」的世界模型。他認為，真實世界的細節（如背景樹葉的晃動、水波紋、光影雜訊）充滿了不確定性與過多的高頻細節。如果 AI 必須把每一個像素都完美生成出來，將消耗龐大的算力，且對理解物理本質沒有實質幫助。  
V-JEPA 的核心機制：特徵空間的預測  
Masked Patch 預測：將影片切割成多個 Patches，隨機遮蔽（Mask）其中的一部分，訓練模型去預測「被遮蔽區域在特徵空間（Feature Space）中的抽象表徵」，而不是還原出真實像素。  
無重構損失：模型不需要計算像素級別的誤差，因此專注於學習高層次的語意與「事件級（Event-level）」的物理與邏輯演變。  
V-JEPA 2 (2024) 的突破與優勢:
極致的算力效率 (High Compute Efficiency):雖然 V-JEPA 不像生成式模型那樣會畫出漂亮的影片，但在下游的動作識別（Action Recognition）與機器人規劃（Planning）任務上，V-JEPA 2 的表現已經能夠與那些吃掉海量算力的生成式對手平分秋色。  
戰略意義:它證明了世界模型不一定非得是個「影像生成器」不可。透過在潛在特徵空間中進行預測，模型能夠以極低的計算成本，掌握理解世界動態所需的關鍵物理與因果抽象。  
5.Sora vs Action-conditioned World Models    
OpenAI 的立場:透過在海量影音資料上進行規模化預訓練，大規模擴散模型（Diffusion Models）能夠在內部隱含地學會基礎的光影、空間三維結構與物理互動，因此具備作為「通用世界模擬器 (World Simulator)」的潛力。  
反方質疑:  
物理違和感與幻覺:Sora 在長時間生成時經常出現穿模、物體突然憑空消失或出現、違反質量守恆定律等非現實現象，顯示它只是在學「像素的統計規律」，而非真正理解底層物理。 
缺乏動作條件:傳統的 Sora 是基於文字提示詞（Text-to-Video）被動生成的，人類無法輸入具體的「控制指令（Action，例如向左轉、加速、施力）」，來實時操控世界如何演變。一個無法被動態控制的模型，很難直接拿來做機器人或自駕車的決策規劃。  
業界核心共識:  
Action-conditioning（動作條件化）：模型必須能精準接收外部動作輸入，並預測環境的連鎖反應。  
Long-horizon consistency（長時序一致性）：在長時間的推演中，必須嚴格維持物理定律與狀態的穩定，不能在幾秒內就發生邏輯崩潰。  
6.名詞表  
RSSM:Dreamer 系列的核心引擎。結合確定性（Deterministic）與隨機性（Stochastic）的潛在狀態並行建模，用來表達動態世界。  
JEPA:LeCun 倡導的非生成式流派。不預測像素，而是直接在特徵空間（Feature Space）中預測被遮蔽的區塊，專注於事件級抽象。  
Latent Rollout (潛在空間想像):AI 不在真實環境中試錯，而是在學好的 Latent Space 內進行多步推演與想像，大幅提升樣本效率。  
Object Permanence (物件恆存):檢驗世界模型是否具備真實物理常識的試金石（例如：物體被遮擋後是否還存在）。當前如 Sora 或 Veo 等模型在此項仍經常翻車。  
Sim-to-real (虛實轉移):在世界模型內訓練出來的決策策略（Policy），能否順利部署到真實世界的機器人或自駕車上，是 Cosmos 等平台的核心終極命題。  




## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
生成式 AI 正急速從「純語言與多模態理解」跨足到「理解物理定律與預測未來」的物理智慧時代。業界湧現出大量以視訊為基礎的世界模擬器（如 OpenAI Sora、Google Genie、NVIDIA Cosmos），同時也伴隨著關於「究竟該預測像素還是預測特徵」的強烈技術路線之爭（如 LeCun 的 V-JEPA 派）。  
能生成（Generate）不等於能規劃（Plan）」。純粹靠文字或隱含規律生成的影片模型，若缺乏動作條件化（Action-conditioning）與對底層物理因果的掌握，就無法作為機器人或自駕車真正的決策模擬器。  
(b)  
像素級生成的算力浪費與物理幻覺:以擴散模型為主體的像素級視訊生成器，花費了巨大的算力在渲染高頻雜訊與無關緊要的細節（如背景樹葉晃動），導致模型容易忽視底層物理邏輯，隨著時間拉長迅速出現穿模、物體扭曲或質量不守恒的現象。  
長時序一致性崩潰與虛實落差:現有世界模型在處理長達幾十秒或幾分鐘的連續動態時，場景與物體極易發生漂移。而在虛擬想像中訓練出來的策略（Policy），一旦直接放到真實世界的物理環境中，往往因為微小的環境誤差而導致決策徹底失效。  

## part17 Audio · Speech · Music
## 名詞理解
1.Neural codec  
就像文字處理需要透過 Tokenizer（如 BPE）將文字轉成 ID 一樣，Neural Codec 是現代 Audio LLMs（語音大語言模型）的基石。它能夠將高頻的連續音訊波形無失真地壓縮並轉化為離散的 Token 序列，讓 Transformer 模型可以直接進行預測與生成。  
業界代表性模型:
SoundStream (Google)：開創性的端到端神經音訊編解碼器，奠定了現代 RVQ 架構的基礎。  
EnCodec (Meta)：開源界極具代表性、廣泛應用於多模態語音與音樂生成系統的高效編解碼器。  
DAC (Descript Audio Codec)：以高保真音質著稱，能極佳地還原音樂細節與人聲情感。  
核心技術架構:  
Encoder (編碼器)採用卷積神經網路（Convolutional Neural Network）架構，將高採樣率（如 24kHz / 48kHz）的原始連續音訊進行下採樣，壓榨成密集的特徵表示。
RVQ (Residual Vector Quantization - 殘差向量量化)  核心的多層 Codebook（碼本）機制。透過層層遞進的殘差量化，將連續特徵對應到離散的代碼本中。最終將音訊壓縮至每秒約 75–150 個 Token (~75-150 tokens/sec) 的理想資訊密度。  
Decoder (解碼器)對應的逆向網路結構（Transposed Convolutions），負責接收 Token 序列並將其精準還原為高品質的真實音訊波形。  
2.Whisper  
Encoder-Decoder Transformer 架構  
輸入端：將原始音訊轉換為對數梅爾頻譜圖 (Log-Mel Spectrogram)。  
輸出端：透過神經網路編碼器處理後，由解碼器以自迴歸（Autoregressive）方式直接輸出文字 Token。  
海量弱監督預訓練:突破傳統語音辨識高度依賴人工精標資料的限制，使用高達 68,000 小時的網羅多語言、多口音、帶背景雜訊的混合資料進行預訓練，賦予模型極強的現實環境穩健性  
效能優化與開源衍生變體:  
Whisper-Turbo:透過將解碼器層數進行裁剪與微調（將解碼層從 32 層減至 4 層），在幾乎不失精準度（微幅 WER 差異）的前提下，實現高達約 4 倍至 8 倍的速度提升，成為即時語音辨識與批次處理的熱門選擇。  
Distil-Whisper:透過知識蒸餾（Knowledge Distillation）技術瘦身模型，大幅降低參數量與運算成本，同時保持優異的辨識率。  
Faster-Whisper:使用高效的 C++ 引擎（CTranslate2）進行底層重構與權重量化（Quantization），在 CPU 及 GPU 上展現出極致的推演速度與極低的記憶體佔用，是落地部署的工業界標配。  
3.TTS  
VALL-E:結合 Neural Audio Codec 將音訊轉為離散 Token，透過大型 Transformer 學習根據文字與短短幾秒的參考聲音去預測後續的聲音 Token，正式開啟 Zero-shot 語音複製的新紀元。  
XTTS-v2:開源界極具代表性的輕量化多語言超擬真 TTS 引擎。    
F5-TTS / E2 TTS:捨棄了傳統自迴歸（Autoregressive）模型容易產生的累積誤差與高延遲，全面改用 Flow Matching（流匹配） 擴散生成技術。  
StyleTTS2 / OpenVoice / GPT-SoVITS:  
StyleTTS2：以極高擬真度與風格控制（Style Transfer）著稱，接近人類播報員級別的表現。  
OpenVoice：強調能夠細膩控制語音的特定情感、重音與精準的零樣本音色複製。  
GPT-SoVITS：在中文與動漫/二創社群爆紅的整合型開源工具，具備極低的訓練門檻與極佳的長文本朗讀穩定度。  
4.Speech-to-speech 與 GPT-4o  
傳統管道  
流程：  
麥克風收音 → ASR (語音轉文字) → LLM (文字推理) → TTS (文字轉語音)  
痛點：  
極高延遲：多段模型串接導致牆鐘延遲通常高達 1 ~ 2 秒，對話節奏卡頓。  
資訊遺失：在轉成文字的過程中，說話者的笑聲、嘆氣、語調起伏、重音與情感等「副語言資訊（Paralinguistic cues）」全部被過濾掉。  
原生多模態語音模型 (Native Multimodal Audio Models):  
代表：GPT-4o Voice、Gemini Live / Advanced。  
運作機制：模型直接以原生的 Audio Tokens 進行雙向輸入與輸出，大腦在同一個統一的 Transformer 內同時處理文字與聲音。  
重大突破：延遲大幅縮減至 ~ 320ms 級別，實現真正的即時打斷與全雙工（Full-duplex）對話，且能精準感知與表達呼吸聲、笑聲、語速快慢與情緒起伏。  
開源界的對抗勢力:  
Moshi (Kyutai 2024):由法國 AI 研究機構 Kyutai 推出的開源即時語音對話模型，主打超低延遲與強大的雙向全雙工語音互動能力。  
Mini-Omni:輕量級開源的 Speech-to-Speech 實作方案，探索無須透過繁複串接即可直接輸出語音 Token 的架構。  
GLM-4-Voice:智譜 AI 推出的原生語音大模型，支援流式的語音端到端對話處理，推動開源中文語音全雙工的發展。  
5.Music  
MusicGen (Meta):基於文字提示詞（Text-to-Music），結合神經音訊編解碼器（Neural Codec Tokenizer）將文字直接轉換為音訊 Token 序列進行自迴歸生成，對旋律與樂器的控制力極佳。  
Stable Audio (Stability AI):採用潛在空間擴散模型 (Latent Diffusion) 架構，擅長生成高保真的時序音樂剪輯、環境音效與特定長度的音軌。  
Suno / Udio:商業閉源巨頭。能夠生成包含完整結構（前奏、主歌、副歌、間奏）與高擬真擬人化人聲（Vocals）的完整流行曲目，開啟了大眾化 AI 詞曲創作的時代。  
2024–2026 年核心爭議：版權與訓練資料合法性  
RIAA 訴訟與版權風暴:美國唱片業協會（RIAA）代表大型唱片公司（如 Universal, Sony, Warner）對 Suno 與 Udio 提起大規模版權侵權訴訟，核心爭議在於未經授權使用版權音樂作為 AI 模型訓練集（Training Data）。  
從對抗走向授權合作的產業轉折:經歷激烈的法律攻防後，部分平台（如 Udio 與 Universal/Warner、Suno 與 Warner）逐步轉向與大型唱片公司達成商業授權與合作協議（推動藝人參與、權利分潤及合法版權模型），標誌著 AI 音樂正朝向合規化與版權清晰化的方向演進。  
6.名詞表  
Mel Spectrogram (梅爾頻譜圖):將聲音以「頻率 × 時間」的 2D 視覺化圖形表示，模擬人類聽覺對頻率的非線性感知，是如 Whisper 等傳統音訊 Encoder 的標準輸入格式。  
RVQ (Residual Vector Quantization - 殘差向量量化):EnCodec、DAC 等神經編碼器的核心機制。透過多層 Codebook 層層遞進量化殘差，將連續音訊壓縮成離散特徵。  
Codec Token (編碼器標記):經由 RVQ 轉換出來的離散整數代碼。每秒約數十到數百個 Token，讓 Transformer 能夠像處理文字一樣直接進行預測與生成。  
VAD (Voice Activity Detection - 語音活動檢測):用於即時判斷「誰在說話、何時開始、何時結束（靜默）」的技術，是即時語音對話 Agent 實現自然打斷與輪流發言的必備組件。  
WER (Word Error Rate - 字詞錯誤率):衡量 ASR 辨識準確度的主要指標（計算插入、刪除、替換錯誤）；中文環境則常使用 CER（字元錯誤率）。  
Zero-shot Voice Clone (零樣本語音複製):自 VALL-E 開啟的新時代。只需提供 1–5 秒的參考音檔就能精準複刻聲線，但同時帶來極高的 Deepfake 與合規倫理風險。  
Inner Monologue (內心獨白 - Moshi 獨創設計):以法國 Kyutai 的 Moshi 為代表。模型在輸出語音 Token 之前，會先生成時間對齊的文字 Token 作為規劃前綴，大幅提升語言邏輯與事實正確性，同時具備同步 ASR 與 TTS 能力。  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
 聽覺人工智慧正在經歷劇烈的典範轉移。從過去笨重、多段串接的傳統管線（ASR → LLM → TTS），迅速演進到以 Neural Audio Codec 為基石、原生端到端（Native Multimodal Audio）處理全雙工語音的全新時代。  
 同時，音樂生成與超擬真語音複製（Zero-shot TTS）在開源與商業雙軌並進下爆發，諸如 Flow Matching 擴散生成與創新的「內心獨白（Inner Monologue）」架構，徹底改變了人機互動的自然度。  
 「語音不只是文字的聲音載體，更是包含無數副語言資訊（Paralinguistic Cues）的高維資料」。傳統串接式架構在將聲音轉成文字時，會粗暴抹除笑聲、停頓、語調與情感起伏；唯有透過原生音訊 Token 的統一建模，才能真正賦予機器人或語音助手類人的對話節奏與情感共鳴。  
(b)  
傳統串接式架構的延遲與資訊斷層:傳統 ASR → LLM → TTS 的串接方式，不僅累積了高達 1 到 2 秒的牆鐘延遲，更致命的是在第一步轉文字時，說話者的語調、情緒、重音與呼吸聲等副語言資訊被永久丟失，導致生成出的語音冰冷且缺乏互動感。  
自迴歸語音模型的累積誤差與運算瓶頸:早期基於 Transformer 的自迴歸語音生成模型在處理長序列音訊時，容易發生後半段崩潰、發音模糊或嚴重延遲（Time-to-First-Audio 過長）的問題，且對算力的消耗極大。  
版權合法性與訓練資料黑箱爭議:現代高保真音樂生成模型（如 Suno、Udio）雖能產出媲美專業母帶的商業級曲目，但由於其訓練過程大量攝取版權音樂，引發了強烈的法律訴訟與創作者權益爭議，成為技術商業化的一大隱患。  

## part18 Code LLMs & Agents
## 名詞理解
1.訓練  
Code-heavy pretraining (海量程式碼預訓練):使用大量的 GitHub 開源程式碼（限於 Permissive 授權條款，如 MIT、Apache 2.0 等）並經過嚴格清洗、過濾垃圾代碼與去重，建立強大的程式碼基底語料庫。  
FIM objective (Fill-in-the-Middle - 夾擠填充訓練):打破傳統單向的 Next-token prediction，讓模型學習「根據前文與後文，來補全中間缺漏的程式碼片段」。這是現代 AI 程式編輯器（如 Cursor、Copilot）進行游標處即時自動完成（Inline Completion）的核心技術基石。  
Repo-level packing (全庫檔案拼裝):將同一個 Repository（程式碼庫）中具備引用關係的多個檔案進行智慧化拼接與打包（Packing），讓模型在預訓練階段就能夠理解跨檔案、跨模組之間的函數調用與依賴關係。  
Long-context (超長上下文支援):支援 256k 到 1M} Tokens 的超長上下文視窗，讓工程師可以直接把整個專案的原始碼塞進 Prompt 裡，告別傳統靠 RAG 檢索經常漏掉關鍵上下文的痛點。  
Execution feedback / RLEF (依執行回饋進行強化學習):跳脫純文字的偏好對齊，直接讓模型產出程式碼並丟進沙盒環境中實際跑單元測試（Unit Tests）。  
2.Coding agent  
核心工具集的分類與分工  
讀取類 (Read):  
read_file：精準讀取特定檔案內容（支援指定行數區間，避免吃掉過多 Context）。  
list_files / glob：探索專案目錄結構，掌握專案佈局。  
grep / ripgrep：全庫關鍵字、正規表達式或符號搜尋，用於快速定位函數與變數定義。  
修改類 (Modify):  
str_replace / edit_file：區塊式的精準替換。  
write_file：整檔覆寫（通常用於全新檔案建立）。  
執行類 (Bash):  
bash：執行終端機指令（跑單元測試、編譯 build、執行 linter、檢查 git 狀態）。  
版本控制 (Git):  
git (diff / status / log / commit)：檢視變更痕跡、還原錯誤、提交變更。  
三大關鍵產品工程決策  
程式碼修改機制：
程式碼修改機制:要求模型給出 <<<<<<< SEARCH 與 ======= 及 >>>>>>> REPLACE 的夾擠區塊。  
Unified Diff / Patch:要求模型輸出標準的 Git diff 格式來套用修改。  
錯誤回饋循環:  
把終端機當成天然的 Compiler / Linter Feedback:當 Agent 透過 bash 執行測試失敗（Test Failure）或編譯報錯（Stderr）時，這些錯誤訊息不會被隱藏，而是完整且原封不動地餵回給 Agent 的 Context。自主閉環：Agent 根據錯誤堆疊追蹤（Stack trace）自動發動下一次的 grep → read_file → str_replace 循環，形成極具韌性的 Self-debugging 迴圈。  
上下文管理與過載防禦:  
超長輸出截斷:當 grep 或 bash 吐出超過數萬個字元的巨量輸出時（例如印出整個測試報告或大檔案），系統會在底層自動截斷或寫入暫存檔，僅回傳預覽路徑給模型，避免 Context 瞬間被垃圾資訊淹沒。  
交替邊界與動態壓縮:隨著 Agent 執行輪數（Turns）增加，Trajectory 會快速膨脹。系統會定期觸發總結機制，將早期的對話與工具調用歷史進行摘要壓縮（保留關鍵決策與目前狀態），確保模型在長迴圈（Long-horizon）任務中不會注意力渙散或遭遇 Token 溢出。  
3.Eval 全景  
HumanEval / MBPP:傳統的函數級別（Function-level）程式碼生成基準。  
LiveCodeBench:定期自動爬取各大程式競賽平台（如 LeetCode、AtCoder）的最新題目進行更新。  
SWE-bench Verified:從真實 GitHub 專案中挑選出包含真實 Issue 與對應 Patch 的測試集（精選 500 例嚴格驗證版）。  
SWE-bench Multimodal:延伸自 SWE-bench，專門針對包含網頁 UI 截圖的視覺與前端 Bug 修復任務。  
Aider polyglot:由知名 AI 程式工具 Aider 推出的多語言（支援 6 種主流程式語言）程式碼編輯與重構評估基準。  
Terminal-Bench:聚焦於完整的 Shell 終端機工作流程。  
RepoBench:針對大型專案的全庫級別（Repo-level）程式碼理解與補全基準。  
Spider 2.0:針對複雜資料庫管理與 SQL 查詢的真實世界 Agent 評估基準。  
4.pattern  
代表工具：Cursor Tab、GitHub Copilot  
核心特徵:  
超低延遲 (Ultra-low Latency)：必須在毫秒級內響應，隨時預測開發者接下來要打的程式碼。  
技術基底：高度依賴 FIM (Fill-in-the-Middle) 訓練目標，根據游標的前後文進行精準的片段補全。  
代表工具：Cursor Chat、Cline (前身 Claude Dev)  
核心特徵：  
全庫 RAG 檢索 (Retrieval-Augmented Generation)：將整個 Codebase 進行向量化或符號索引（AST/Symbol Indexing）。  
互動場景：開發者可以直接提問：「這個專案的認證邏輯在哪裡？」或「幫我解釋這個模組的資料流」，由 LLM 結合專案上下文進行精準回答。  
代表工具：Claude Code、Devin   
核心特徵：  
規劃與執行 (Plan & Exec)：跳脫單純的問答，Agent 能夠主動拆解任務、跨多個檔案進行編輯、呼叫終端機跑編譯與測試，並根據錯誤訊息進行自我修正（Self-correction）。  
互動場景：開發者給定一個具體的功能需求或 Bug，Agent 自主完成程式碼修改與驗證。  
代表工具：Devin、OpenHands (前身 OpenDevin)  
核心特徵：  
端到端工程交付 (End-to-End Delivery)：從接收自然語言任務描述開始，一路經歷環境設定、程式碼編寫、測試驗證、Git 提交，直到自主開啟並完善 Pull Request (PR)。  
檢驗重點：代表當前 AI 軟體工程自動化的最高自主級別。  
5.名詞表  
FIM (Fill-in-the-Middle - 夾擠填充):將文件拆成 Prefix / Middle / Suffix 並重新排列進行預訓練，讓模型學會「根據前後文補中間」，是所有 IDE 即時程式碼補全（Inline Completion，如 Cursor Tab）的技術根基。  
Repo-level context (跨檔依賴上下文):將同一個 Repository 中的多個檔案進行智慧化拼接與打包進行訓練，使模型能夠理解 import 與模組間的跨檔案依賴關係。  
RLEF (Reinforcement Learning from Execution Feedback - 執行回饋強化學習):讓模型產出程式碼後直接丟進沙盒跑測試、Linter 或 Build，以測試通過率（Pass/Fail）作為 Reward。相比傳統 RLHF 更具客觀性與工程指導價值。  
SWE-bench (真實 GitHub Issue 評估基準):包含 2,294 題（Verified 子集為精選 500 題）的真實軟體工程基準，要求 Agent 從真實 GitHub 專案中定位並修復 Issue，是目前各大 Frontier Coding 模型的頂級競技舞台。  
ACI (Agent-Computer Interface - 智慧體電腦介面):由 SWE-agent 提出，專門為 LLM 設計的精簡工具操作介面（如 read_file、edit、bash），旨在降低複雜度、節省 Token 並減少模型的犯錯率。  
Diff edit vs. SR edit (程式碼編輯形式):Agent 修改程式碼的兩種主流策略。Aider 採用標準的 Unified Diff，而 Claude Code 採用 Search-and-Replace（搜尋與替換），兩者各自有不同的錯誤模式與容錯特性。  
Self-debug (自我除錯與迴圈修復):Agent 透過 bash 執行測試並讀取 Error Trace，自動改寫程式碼直到測試通過的閉環能力。這是支撐 SWE-bench 取得高分（超過 70% 關鍵能力）的核心機制。  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
AI 程式編寫工具正經歷從「被動的程式碼補全（Inline Autocomplete）」與「側邊欄問答」到「自主軟體工程代理（Autonomous Coding Agents，如 Claude Code、Cursor、Devin）」的劇烈典範轉移。AI 不再只是給建議，而是直接升級為能夠自主閱讀整個 Repository、跨檔案修改、執行終端機指令、跑測試並提交 PR 的工程同事。   
「軟體工程的關鍵不在於寫出演算法，而在於全庫級的上下文理解與精準的 ACI（Agent-Computer Interface）工程設計」。一個優秀的 Coding Agent 之所以強大，往往不是因為模型本身有多聰明，而是因為它具備將編譯器、Linter 與測試錯誤即時回饋並進行自我除錯（Self-debug）的強健閉環。  
(b)  
上下文超載與長迴圈注意力渙散:隨著任務輪數增加或專案規模擴大，Trajectory 與檢索出的程式碼迅速膨脹，不僅消耗龐大的 Token 成本，也容易使模型在長序列中遺失最初的架構設計意圖，導致邏輯漂移。  
程式碼修改與 Patch 套用的脆弱性:無論是基於 str_replace（搜尋與替換）還是 Unified Diff，模型經常因為微小的縮排、行號偏移或程式碼重複匹配失敗，導致修改指令無法套用。這使得 Agent 常常需要浪費寶貴的計算輪數在修正「語法級別的 Patch 錯誤」而非解決業務邏輯 Bug。  
缺乏全域架構約束的「盲目重構」風險:在缺乏嚴格的測試防護網或 CI 閘道時，自主代理容易進行過度修改（Over-editing），在高度耦合的 legacy 程式碼中引發潛在的迴歸錯誤（Regression），暴露出當前模型對大型軟體全域相依性的底層理解仍有侷限。  


## part19 Evaluation
## 名詞理解
1.Capability benchmarks  
通識與專業知識  
MMLU / MMLU-Pro：覆蓋人文、社科、STEM 的經典通識基準（MMLU-Pro 透過增加選項與更難的問題來延緩飽和）。  
GPQA Diamond：專家級（具備博士學位水準）的物理、化學、生物學研究級科學問答，專門用來考驗模型的深層專業知識。  
數學與程式邏輯  
MATH / AIME：從高中競賽到美國數學邀請賽（AIME）級別的高階數學推理基準。  
HumanEval / LiveCodeBench / SWE-bench Verified：涵蓋從簡單函數級到真實 GitHub 專案修復的程式碼能力評估。  
綜合推理與遵循能力  
BBH / MUSR / DROP：針對多步複雜邏輯推理、字串處理與閱讀理解的基準。  
IFEval：嚴格檢驗模型是否能完美遵循複雜格式與指令限制的評估工具。  
HellaSwag / WinoGrande：傳統的常識與指代消解基準（目前已完全飽和）。  
突破飽和的下一代極限基準:  
Humanity's Last Exam (HLE):由 Center for AI Safety 與 Scale AI 聯合推出的多模態專家級基準（包含 2,500+ 道題目）。  
FrontierMath:由全球數十位頂尖數學家原創的數百道高難度現代數學題（涉及數論、代數幾何、範畴論等）。  
ARC-AGI 2:François Chollet 推出的全新升級版 ARC 抽象推理基準。  
2.Arena / preference  
優勢  
真實反映人類偏好：擺脫了傳統學術基準的死板與侷限，能夠全面捕捉真實世界中複雜、開放式、多樣化的使用者意圖。  
缺點  
冗長與排版迷思：統計發現，人類評審往往會無意識地偏好回答冗長（Verbose）、善用精美 Markdown 排版、語氣熱情討好（Sycophancy）的模型，即使其內容包含了事實錯誤或幻覺。  
「Style > Substance」：這導致許多模型為了刷高 Arena 排名，被刻意調校成「話多、愛用列點、態度逢迎」的風格，而非追求邏輯的精簡與絕對正確。  
進階變體與垂直領域競技場
Arena-Hard:利用自動化 LLM-as-a-Judge（以最強模型充當裁判）針對高難度、具備挑戰性的提示詞集進行過濾與評測，大幅降低對人工盲測的依賴並提升鑑別度。  
WildBench:專門收集真實世界中極其刁鑽、複雜、長時序的真實使用者提問（Wild Prompts），用來評估模型在真實高壓情境下的極限表現。  
CopilotArena (或 Coding Arena):專門針對程式碼生成與編輯任務所設立的盲測競技場，讓開發者在盲測中比較不同 Coding Assistant 的實際除錯與寫碼能力。  
3.LLM-as-judge  
引入強大的前沿模型（如 GPT-4、Claude 3.5 Sonnet、Gemini）來擔任自動化裁判（Judge），根據預設的評分標準（Rubrics）對受測模型的回答進行客觀評分或兩兩對決（Pairwise Comparison）。  
四大核心偏誤與科學修正對策:  
Position Bias (位置偏誤)  
問題：裁判模型在進行 A/B 對決時，往往會無意識地偏好排在前面（Model A）的回答。  
修正對策：隨機交換順序（Swap positions）。對每個測試樣本執行兩次評測（A vs. B 與 B vs. A），綜合雙方表現以消除順序帶來的影響。  
Length Bias (長度偏誤)  
問題：裁判模型極易被「長篇大論、洋洋灑灑」的回答所迷惑，誤以為字數多就是內容豐富。  
修正對策：長度控制評分（Length-controlled scoring，如 AlpacaEval 2.0）。在統計模型中引入長度懲罰或迴歸校正，將回答字數做為變數進行數學剝離，還原真實的質量分數。  
Self-Preference (自我偏誤)  
問題：當使用某個特定家族的模型（例如 GPT 系列）擔任裁判時，它往往會給予同系列模型產生的回答更高分數。  
修正對策：替換 Judge Model 或使用多模型交叉對照。避免單一模型獨裁，改用獨立、開源或多個強模型聯合仲裁（Ensemble Judging）。  
Lack of Chain-of-Thought Reasoning (缺乏透明推理)  
問題：直接要求模型輸出一個分數（如 1–10 分）容易流於黑箱與主觀隨機性。  
修正對策：先推理後評分（First reason, then score）。強制規定 Judge 在給出最終分數前，必須先逐步寫出詳細的優缺點分析與推理過程（CoT），大幅提升評分的穩定度與可解釋性。  
4.Contamination污染  
污染檢測技術  
N-gram 匹配 (N-gram Overlap):計算預訓練語料庫與評估基準題目之間的文字重疊度（N-gram 相似度），過濾高度相似的文本。  
損失函數尖峰分析 (Loss Spike Detection):觀察模型在特定題目上的預測 Loss（困惑度），若模型對某道題目的初始預測 Loss 異常低，通常代表該題目已存在於其訓練集中。  
改寫對比測試 (Paraphrase & Perturbation):將標準考題的語法、變數名稱或敘述方式進行同義改寫（Paraphrase），若模型在原題拿高分但在改寫題分數暴跌，即可證實模型依賴死記硬背而非真實理解。    
三大防禦對策  
動態更新與定期換題 (Live Benchmarks):推出每月或定期自動更新題目的基準（如 LiveBench、LiveCodeBench），確保題目在模型預訓練截止日之後才誕生，徹底阻斷偷看管道。  
私有盲測與保留集 (Private Holdout Sets):維護不公開的私有測試集（如企業內部或大賽專用的 Holdout Set），僅允許遠端評測，防止題目外洩至公開網路上。  
評估推理過程而非僅看最終答案 (Reasoning Trace Evaluation):不再依賴傳統的選擇題或簡單數值匹配，而是要求模型產出完整的思考路徑（Reasoning Trace / CoT）。即使題目被背下來，若缺乏動態推演過程也無法順利過關。  
5.eval  
五大核心評估維度 (Evaluation Dimensions):  
正確性 (Accuracy)：解決方案或回答事實是否正確。  
簡潔度 (Conciseness)：是否廢話過多、直達核心。  
語氣 (Tone)：是否符合品牌調性（專業、有同理心、不暴躁）。  
拒絕能力 (Refusal)：面對超出範圍或違規請求時，能否安全且精準地拒絕。  
工具調用 (Tool Use)：是否能正確觸發後端 API 或資料庫查詢。  
雙軌評測機制 (Human + LLM Evaluation):結合「人類專家抽樣盲測」與「自動化 LLM-as-a-Judge（針對 100 題進行批次自動評分）」，兼顧精準度與自動化效率。  
現代 LLM Ops 評估工具鏈 (Tooling Ecosystem):  
Braintrust：專為 AI 產品設計的資料集、評估與實驗追蹤協作平台。  
Langfuse：開源的 LLM 工程平台，支援追蹤 (Tracing)、評估 (Evaluation) 與提示詞管理。  
Weights & Biases (W&B) Weave：強大的實驗追蹤與模型評估視覺化工具。  
輕量派：自寫 Python 腳本 + Google Sheets：新創團隊早期常用的低成本靈活方案。  
6.名詞表  
Pass@k (程式碼通過率):讓模型對同一個題目生成 $k$ 個候選答案，只要其中有至少一個通過單元測試即算成功的比例。  
ELO / Bradley-Terry (競技場排名模型):從兩兩對決（Pairwise Comparison）的投票結果中推導出潛在實力評分（Latent Rating）。如 LMSYS Chatbot Arena 即採用 Bradley-Terry 的最大似然估計（MLE）來計算全球模型排名。  
Contamination (資料污染):評估基準在模型預訓練階段被「偷看過」的現象。可透過 N-gram 匹配或 Canary String 偵測，而推出「動態即時基準（Live Benchmarks）」是當前最強的防禦解法。  
Refusal rate (拒絕率與過度拒絕):衡量模型面對安全審查或越獄時的行為。值得注意的是，Over-refusal（過度拒絕合法的正常請求）往往是常被忽略但嚴重影響產品體驗的品質問題。  
Length bias (長度偏誤):裁判模型或人類評審會默默偏好「字數多、排版華麗」的回答。需透過長度控制評分（Length-controlled scoring）等數學手段進行校正。  
Eval-driven dev (評估驅動開發，類似 TDD):將 Eval Set 當成軟體工程中的單元測試來對待，規定每次調整 Prompt 或更換模型前必須先跑過測試集，分數達標才能進行部署。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
AI 評估基準正面臨空前的「飽和與污染危機」。傳統靜態基準（如 MMLU、HumanEval）在模型快速進化下紛紛突破 90%，高分背後往往反映的是背誦記憶而非真實推理；同時，社群透過 LMSYS Chatbot Arena 的盲測機制與 LLM-as-a-judge 自動化裁判，建立起以使用者偏好為核心的動態評估生態。  
「好的 Eval 是 AI 產品開發的單元測試（Unit Test）」。學術基準適合衡量前沿模型的極限，但在真實商業場景中，團隊必須建立針對特定業務（如客服、Agent 任務）的黃金測試集（Golden Dataset），並落實如 TDD 般的 Eval-driven dev 流程，每次改動前必先跑 Eval。  
(b)  
資料污染與分數通脹:開放式基準在網路爬蟲普及下頻繁流進預訓練集，導致模型在榜單上的高分失真，無法真實反映面對未知新題時的泛化與推理能力。  
裁判偏誤與風格盲點:無論是人類盲測（Arena）還是 LLM-as-judge，皆存在嚴重的長度偏誤、自我偏愛與過度包裝迷思，使模型容易被引導去追求「話術與排版」而非「邏輯與事實正確」。  
學術基準與真實業務的巨大鴻溝:傳統靜態考卷（如選擇題）無法有效衡量真實世界中複雜、動態、多輪互動的軟體代理（Agent）與客服系統表現，導致「榜單第一名不等於產品好用」。  




## part20 Data
## 名詞理解
1.Pretraining pipeline  
原始網頁擷取與解析 (Extraction):  
Common Crawl WARC:網路世界規模最大的開放式爬蟲資料庫（包含數十億個網頁的原始 WARC 檔案）。  
文字萃取工具 (trafilatura / resiliparse):專門用於剝離 HTML 標籤、廣告、導覽列與側邊欄雜訊，精準提取出核心文章與段落純文字的超高效解析器。  
語言識別與基本過濾 (Language ID & Heuristics):  
FastText Language ID:快速辨識並過濾出目標語言（例如繁體中文與英文），剔除外語或混合亂碼網頁。  
啟發式過濾 (Heuristic Filtering):設定硬性指標（如每行平均長度、特殊符號/表情符號比例、大寫字母佔比等），過濾掉機器生成的垃圾文章或 SEO 灌水網站。  
品質與安全分類 (Quality & Safety Classifiers):  
高品質分類器 (Quality Classifier):利用訓練好的 FastText 模型或小型 LLM，評分並篩選出達到百科全書、學術級別的高質量文本。  
毒性與 NSFW 過濾 (Toxicity / NSFW Filter):嚴格剔除含有暴力、色情（NSFW）、仇恨言論或惡意攻擊的有害語料，確保模型的合規性。  
大規模去重 (Deduplication):  
MinHash 去重 (MinHash LSH):採用 5-gram 與 Jaccard 相似度（閥值設定約 0.8），在分散式叢集中高效找出網路上成千上萬的重複站點、轉載文與鏡像站，大幅降低訓練冗餘並防止過擬合。  
多源資料黃金配比 (Data Mixing):  
綜合語料融合:將清洗完畢的網頁文本，與高價值的程式碼（Code）、專業書籍（Books）、學術論文（Academic）以及合成資料（Synthetic Data）按照特定比例進行融合混編（Data Blending），送入模型的預訓練階段。  
2.Dedup  
在所有預訓練資料清洗步驟中，去重（Deduplication）往往是用最少的運算成本，換取模型效能提升最顯著的關鍵環節。  
核心技術：MinHash + LSH  
MinHash:將每篇文章轉換為一組精簡的 Hash 特徵，能夠在保持極低空間佔用的同時，精準估算兩篇文檔之間的 Jaccard 相似度。  
LSH:將相似的 MinHash 雜湊值歸納到同一個「分桶（Buckets）」中。透過分桶機制的過濾，將大規模比對的計算複雜度降至近乎線性，高效找出網路上成千上萬的近似重複頁面（Approximate Near-Duplicates）。  
Lee et al. (2022) 經典研究:  
該研究指出，經過嚴格去重後的資料集在進行相同算力（Compute Budget）的預訓練時，模型的 Perplexity（困惑度）顯著下降。  
這證明了去除重複資料不僅能節省儲存與訓練時間，還能強迫模型去學習更多元、分佈更廣的真實語言規律，大幅降低過擬合（Overfitting）風險。  
3.Quality filter  
FineWeb-Edu (Hugging Face):專門訓練分類器以「教育價值（Educational Value）」為評分標準，大量拔高教科書、線上課程與學術討論文本的權重，過濾掉娛樂八卦與低俗內容。  
DCLM:採用精選的正例（如 OH-2.5 與 ELI5 - Explain Like I'm 5），偏好結構清晰、邏輯嚴密且具備良好科普解釋能力的語料。  
Phi 系列 (Microsoft):微軟的 Phi 路線主打「Textbook Quality（課本級品質）」。透過強大的 Teacher LLM 將複雜的網頁內容重新改寫與蒸餾成如教科書般精煉、結構化的合成/清洗資料。  
沒有絕對客觀的資料清洗:所謂的「品質過濾」本質上就是一種人為的價值觀裁決。當我們決定用 Wikipedia、教科書或 ELI5 作為正例訓練分類器時，我們就在資料源頭灌輸了特定的語言風格、學術權威感與文化視角。  
4.Synthetic data  
Phi-3 / Phi-4:利用強大的教師模型（Teacher LLM）根據大綱或主題，主動編撰出如同教科書般邏輯嚴密、結構清晰的高密度教學文本與程式碼範例。  
Tülu 3:透過引入「具備特定背景、情境與角色 Persona」的提示詞模板，再由 GPT-4 或 Claude 生成多樣化的對話與推理回答，大幅提升 SFT 階段指令的多樣性。  
OpenMath / MetaMath:針對高難度數學與邏輯題目進行逆向擴增、變數代換與多步思維鏈（CoT）重構，擴大模型在數理推理上的訓練樣本池。  
Magpie:巧妙利用對齊前（Pre-aligned）的 Base Model，直接誘發其在訓練時期的指令遵循行為，無須透過人工編寫 Prompt 就能自然產出海量的開端對話資料，大幅降低 SFT 資料收集成本。  
5.SFT / preference data  
監督微調資料集:  
Alpaca：早期的經典指令微調集，透過 Self-Instruct 機制由 GPT-3.5 自動生成，奠定開源指令微調基礎。  
ShareGPT：抓取真實人類與 ChatGPT 的多輪對話紀錄，具備極高的對話自然度與真實排版風格。  
UltraChat：涵蓋多元主題、大規模且多輪（Multi-turn）的高品質對話合成資料集。  
Tülu (AI2)：整合多種開源指令與對話數據的高標準開源 SFT 訓練集。  
OpenHermes：以高度多樣性與嚴格過濾著稱的頂級開源 SFT 訓練集，大幅提升開源模型的指令遵循與工具調用能力。  
偏好與對齊資料集:  
UltraFeedback：包含大規模、多維度（如正確性、真實性、誠實度）由強力模型自動打分與對比的頂級偏好資料集。  
HH-RLHF (Anthropic Helpful & Harmless)：專注於「有幫助 (Helpful)」與「無害 (Harmless)」平衡的安全對齊偏好資料。  
SHP (Stanford Human Preferences)：來自 Reddit 等真實網路社群人類投票結果的偏好對齊資料。  
Skywork-Reward：專為獎勵模型（Reward Model）訓練設計的高質量、細粒度偏好評分資料集。  
推理思維鏈資料集:  
OpenR1：開源社群為了重現 DeepSeek-R1 推理能力而推動的開放式思維鏈資料與訓練管線。  
AceReason / Bespoke-Stratos：包含逐步推理、自我驗證、錯誤修正等完整思考過程（Reasoning Traces）的高端合成邏輯資料集。  
6.爬蟲  
基本禮儀與識別:  
尊重 robots.txt：遵循網站管理員設定的爬蟲白名單與禁令。  
清晰的 User-Agent：主動宣告爬蟲身份與所屬機構，避免惡意偽裝。  
日益加劇的反 AI 爬蟲機制 (Anti-AI Scraping):隨著 Cloudflare 等主流 CDN 廠商全面升級防禦系統，針對 AI 訓練爬蟲的行為特徵分析、速率限制與瀏覽器指紋識別越來越普及，導致大規模自動化擷取的技術門檻與成本急速飆升。  
法律大環境的動盪與授權壁壘:  
標誌性版權訴訟:以 NYT v. OpenAI 與針對 Anthropic 的系列版權訴訟為首，正從法律層面挑戰「網頁公開資料是否屬於合理使用（Fair Use）」的核心定義。  
社群與內容平台的封閉化:如 Reddit 等大型論壇紛紛關閉免費 API 並轉向與科技巨頭簽署獨家商業授權協議，象徵著「整個互聯網都是免費開源訓練語料」的蠻荒擴張時代已經結束。  
2026 仍未塵埃落定的合規紅線:智慧財產權、著作權費與爬蟲合法性的全球法律框架仍處於高度動態變動的灰色地帶，成為當前各大 AI 企業在模型預訓練階段必須嚴陣以待的重大法務與合規風險。  
7.名詞表  
Common Crawl (大規模網絡爬蟲資料庫):每個月抓取數十億個網頁、規模達數 PB 的 Raw HTML 資料庫，是當今所有 Frontier Models 預訓練語料的根基。  
WARC:Common Crawl 的標準化封存格式，將每個網頁的原始標頭、中繼資料與 HTML 內容封裝為單一記錄。  
MinHash:對文件的 5-gram 特徵計算多個 Hash 的最小值，作為評估 Jaccard 相似度的極高效近似估計方法。  
LSH:將相似的 MinHash 歸納到同一個分桶中，將海量文件去重的計算複雜度從 O(N^2) 大幅降至近乎線性 O(N)。  
Data mixture:程式碼、網頁、書籍、學術論文等不同 Source 的黃金混編比例，直接決定了模型最終的綜合能力分佈（通常透過 Ablation 實驗調校）。
Synthetic:透過強模型寫程式給弱模型學（蒸餾），或利用 Base Model 自問自答（如 Magpie）來突破人類網路資料天花板的技術。  
Canary:刻意埋在訓練資料中的特殊標記字串，後續可用於檢測模型是否對特定敏感或版權資料進行了「過度死記硬背（Memorization）」。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
隨著互聯網公開高質量文本面臨枯竭（Data Wall），AI 產業的競爭焦點已從「盲目擴大爬蟲規模」徹底轉向「精細化策展（Curation）、自動化去重與高質量合成資料」。資料工程不再只是附屬步驟，而是決定模型天花板的核心護城河。  
「Data is the new oil, but curation is the refinery (資料是原油，但策展是煉油廠)」。模型的能力分佈與安全性高度取決於 Data Mixture（資料混編比例）與過濾器的設計；同時，過濾器的選擇本質上就是一場對模型世界觀與語氣風格的價值觀裁決。  
(b)  
互聯網枯竭與版權法務壁壘:隨著各大 CDN（如 Cloudflare）全面升級反爬蟲機制，加上 NYT 等版權訴訟與 Reddit 授權變動，過去那種「整個互聯網都是免費開源訓練語料」的蠻荒擴張時代已不復存在。  
合成資料的同質化與模式崩塌:依賴少數頂級閉源模型的輸出進行大規模蒸餾，會讓開源模型充斥著標準的「AI 腔調」，導致多樣性遞減並喪失真實人類語言的豐富細節。  
過濾器帶來的隱性偏見:自動化品質過濾器（如偏好維基百科或教科書風格）在拔高學術與邏輯密度的同時，也可能無意間抹殺了非正統表達、多元文化視角或日常俚語，使模型過於菁英化與單一化。  

