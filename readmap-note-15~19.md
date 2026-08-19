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




