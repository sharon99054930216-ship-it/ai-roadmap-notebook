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




