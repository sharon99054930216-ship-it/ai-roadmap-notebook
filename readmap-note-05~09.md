# part05 The Transformer
## 名詞理解
1.Transformer:解決不能平行，長距離依賴性弱的問題  
2.Self-Attention:  
Q = X1 · W_Q (將輸入的序列乘上矩陣Q)  
K = X2 · W_K (將輸入的序列乘上矩陣K)  
V = X3 · W_V (將輸入的序列乘上矩陣V)  
attention output：Attention(Q, K, V) = softmax(QKᵀ / √d) · V  
<img width="550" alt="image" src="https://github.com/user-attachments/assets/665b7652-44f0-40d4-a6a7-611f751b311c" />    
3.Multi-Head Attention（多頭注意力機制):能同時處理觀察多個內容間的關係，提高效率品質  
4.Transformer 結構:  
<img width="376" height="525" alt="image" src="https://github.com/user-attachments/assets/3eb016bd-0c81-424d-9b0a-5209859a1599" />    
5.GPT Decoder Block 架構:  
<img width="652"  alt="image" src="https://github.com/user-attachments/assets/45eaef67-70d2-4265-beca-bc07f51c754b" />  
第 1 行 (self.norm1(x))：把輸入的 Token 特徵進行歸一化，把數值分佈規範好。  
第 2~3 行 (self.mha(..., mask))：讓 Token 們互相看來看去、算內積（計算上下文權重），吸取彼此的資訊。  
第 4 行 (x = x + ...)：將吸取完上下文的特徵，以加法（Residual Connection）的方式灌回原始的高速公路上。  
第 5 行 (self.norm2(x))：將剛更新完上下文的資訊，再次做一次層歸一化（LayerNorm）。  
第 6 行 (self.ffn(...) 搭配 x = x + ...)：每個 Token 孤獨地進入那個放大 4 倍的 MLP 空間，去觸發和檢索模型死記硬背下來的物理常識或邏輯定理，最後再次用加法融入高速公路。  
6.Positional Encoding 的演進:  
從「加在 Embedding 上（絕對位置）」到「改在 Attention Score 上（相對距離）」，再到如今大模型（LLM）的「旋轉矩陣（RoPE）與長文本外推技術」，讓大模型能輕鬆讀懂超長文字  
7.Encoder / Decoder / Encoder-Decoder  
Encoder-BERT:預訓練任務，語意理解能力  
Decoder-ChatGPT、Claude、Gemini :大規模模型，文本生成能力，generative(生成式) + scaling(規模化) + in-context learning(上下文學習)能執行所有任務  
Encoder-decoder-T5 / BART:Encoder 處理輸入、Decoder 一邊用 cross-attention(交叉注意力) 看 Encoder 一邊輸出像接龍一樣逐字生成新文字。翻譯、摘要的最自然形式  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？
(a)單看文字說明會不太清楚，搭配李宏毅老師的影片可以了解得更清楚:https://www.youtube.com/watch?v=hYdO9CscNes  
(b)Transformer計算量隨長度呈二次方暴增，處理長文本極耗記憶體。  

# part06 Large Language Models 
## 名詞理解 
1.LLM(大型語言模型)  
狹義:現代 LLM 的「標準配方」。Decoder-only Transformer、自回歸（Autoregressive）與 Next-token Prediction、規模門檻    
廣義:以「能力湧現」為導向。In-context Learning(上下文學習)  
Decoder-only + 自回歸 + 海量資料預訓練」這套成功方程式（Recipe）至今依然是業界普世標準。  
2.Pretraining recipe(預訓練標準)  
Data Preparation（數據準備）→ Tokenizer Training（分詞器訓練）→ Tokenization & Packing（數據打包）→ Model Architecture（模型架構）→ Optimizer Setup（最佳化器設定）→ Loss Function（損失函數）→ Massive Training（大規模預訓練）→ Mid-training / Annealing（中期強化）→ Long-context Extension（長文本擴展）→ Post-training（後訓練與對齊）  
3.Data - LLM   
Data Mix 的黃金配比:Web (50-70%) 提供模型的「常識與通識」，是廣度的基底，Code (10-25%) & Math (2-8%) 不是為了讓模型當工程師，而是為了培養 Reasoning（推理）與 Chain-of-Thought（思維鏈）能力，Synthetic Data (5-30%)：用大模型生成高質量數據來餵養新模型（合成數據）已成為標配  
Dedup（去重):重複資料刪除，防止發生過擬合。    
4.Tokenizer(分詞器/標記器):將人類文字拆解成多個稱為 Token（詞元） 的微小單位，並將其轉換為模型能理解的數字
Tokenizer 影響什麼:
壓縮率 (Compression Ratio):好的 Tokenizer 能在高頻詞（如中文、常見 Code 語法）上有更高的壓縮率。同樣的上下文視窗內能塞入更多實質內容、推理速度變快、預訓練時的算力浪費變少。  
多語言公平性 (Multilingual Fairness):早期 (GPT-2)： 對中文支援極差，採取 Byte-by-byte 分詞，一個繁/簡漢字可能被拆成 3 個 Token。導致同樣一本書，中文訓練成本是英文的 3 倍。現代： 現代主流 Tokenizer 擴大了詞表，大幅優化（中日韓）的分詞，讓一個漢字儘量等於 1 個 Token。  
數學算術能力 (Digit-level Splitting):過去模型算術差，部分原因是因為 12345 有時被切成 12 和 345，有時被切成 123 和 45，導致模型難以學習位值概念。現代解法：模型在 Tokenizer 階段將數字逐位拆分（例如 123 必然拆成 1、2、3），這極大地改善了模型的底層數學算術與對齊能力。  
邊緣案例(Edge Cases):如結尾空格、複雜 Emoji、跨語言混雜等場景。Tokenizer 在這裡處理不當，會導致模型產生語意斷層，進而引發模型輸出陷入死循環或胡言亂語的怪異 Bug。  
5.MoE(混合專家模型):是一種機器學習架構，它將大型神經網絡拆分成多個專精不同子任務的「專家」子網絡。透過「路由（Router）」機制的調配，每次計算只會激活少數幾個專家。這項技術能在不大幅增加運算資源的情況下，使模型容量達到兆級參數規模。  
6.Long context(長上下文)怎麼做到的:   
位置編碼的數學外推:在數學上「縮小或平滑」旋轉位置編碼（RoPE）的頻率基數  
分階段的 Long-context Training 流水線:1. Base 階段： 先用 4k 或 8k 的短文本吃滿 90% 的 Token，2.修改 RoPE 超參數，餵入少量（幾十 B Token）經過特別清洗的長文檔，在短時間內將模型「拉長」到目標長度。  
Attention 機制變體  
Ring Attention： 將 Attention 的計算沿著序列維度切片，分發到多個 GPU 組成的「環形網絡」中並行計算，打破單張 GPU 的記憶體限制。  
Hybrid SSM-Attention： 將 Attention 與 Mamba / SSM（狀態空間模型，具備 O(N) 線性複雜度）混合交替。利用 SSM 處理超長背景，只在關鍵層保留 Attention，大幅降低計算開銷。
推論優化   
PagedAttention:像作業系統的虛擬記憶體分頁一樣，將不連續的 KV Cache 記憶體碎片化管理，完全消除記憶體碎片。  
KV Cache 量化： 將幾十 GB 的 FP16 KV Cache 壓縮到 INT8 或 INT4，直接讓單卡能容納的 Batch Size 與長度翻倍。  
7.基礎模型 → 對話模型
SFT（監督微調）:使用高質量的指令對（Pairs），以傳統的 Cross-Entropy Loss 直接微調 Base Model，教導它理解「問與答」的對話格式。LIMA (2023) 論文丟出震撼彈：僅用 1,000 條極致高質量、排版完美的樣本，就能訓出極其驚艷的 Chat Model。這證明了「數據質量」遠比「數據數量」重要。  
RLHF(基於人類回饋的強化學習)  
收集偏好數據： 給定一個 Prompt，讓 SFT 模型生成 A、B 兩個不同的回答，由人類標註員勾選哪一個更好。  
訓練獎勵模型 (Reward Model)： 用這些人類偏好數據去訓一個「裁判模型」，讓它學會像人類一樣對回答進行打分。  
近端策略最佳化(PPO)： 讓 LLM 作為 Agent 在環境中不斷試錯生成回答，最大化 Reward Model 給出的分數。同時引入 KL 散度約束，防止模型為了討好裁判而「走火入魔」，吐出面目全非的奇異文本。  
DPO(直接偏好優化):不用 RL 的強化學習  
證明了可以將 RLHF 的數學目標函數進行重寫，跳過獎勵模型與 PPO 階段，直接在偏好數據（ y_w 為贏家，y_l 為輸家）上進行監督學習  
RLAIF(規模化與憲法)  
Anthropic 的解法 ： 寫下一份「憲法/守則」（Constitution，例如：請選擇最客觀、最不具攻擊性的回答），然後叫另一個更強的大模型來扮演人類標註員，進行偏好打分。  
8.Inference(推理):預訓練決定了模型能不能進實驗室，而推論優化決定了模型能不能活在市場上

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？
(a)LLM的定義、預訓練標準流程、Tokenizer的影響、MoE的學習架構、對話模型需要的prompt
(b)LLM僅靠預測計算出下個字，缺乏邏輯、易幻覺且算力昂貴。

# part07 Vision-Language Models
## 名詞理解 
1.VLM:能看圖回答，能讀文件，能處理 GUI，是VLA (機器人) 的前置  
VLM = Vision Encoder + Projector + LLM  
2.CLIP(對比語言-影像預訓練):它能夠理解影像與自然語言之間的關聯，打破了傳統 AI 只能單獨處理圖片或文字的限制。  
CLIP 核心概念是將文字與影像映射到同一個向量空間中。  
雙編碼器架構：它包含一個「圖像編碼器（Image Encoder）」和一個「文本編碼器（Text Encoder）」。  
對比學習：在訓練過程中，模型會分析大量的（影像, 文字）配對，學習如何判斷哪一段文字最能描述哪一張圖片。  
語意對齊：訓練完成後，模型就能夠精確計算出任何圖片與任何句子之間的相似度。  
3.LLaVA(大型語言與視覺助理):結合了視覺編碼器與大型語言模型（LLM），賦予機器「閱讀圖片」並進行深度對話、邏輯推理或影像描述的能力。  
LLaVA 屬於多模態神經網路模型，主要包含三大核心模組：  
視覺編碼器（Vision Encoder）：通常使用預訓練好的 CLIP 模型，負責將輸入的影像轉換為密集的視覺特徵向量。  
投影層（Projection Layer）：作為視覺模型與語言模型之間的「橋樑」，將影像特徵轉換並對齊到與文字詞向量相同的維度空間。  
大型語言模型（LLM）：作為大腦核心，接收來自視覺端的特徵與人類的提示詞（Prompt），進行綜合邏輯推理並生成自然語言回應。  
4.Projector(投影):看似數學上的單純變換，實則是核心關鍵。它能將高維複雜數據降至低維，過濾雜訊、保留特徵，是實現高效能模型（如降維與自監督學習）不可或缺的基石。  
5.Resolution-高解析度:  
方案一:直接放大，當輸入圖片變大，ViT 的 Patch 數量會從 24x24 暴增到 72x72。這時必須對 ViT 原本的 2D 絕對位置編碼進行雙線性插值  
方案二:動態切片與縮圖拼接(Tile / AnyRes)，不改變 Vision Encoder（如 CLIP-ViT）的輸入規格，而是透過圖像切片與縮圖拼接來兼顧全局與局部細節  
方案三:原生動態解析度(Native Dynamic)，拋棄了繁瑣的「切片 + 縮圖」預處理，讓 Vision Encoder 直接原生支援任意解析度的輸入  
6.資料工程(Data):模型架構大家抄來抄去都差不多，真正拉開產品差距的，全是隱藏在幕後的資料工程   
一、兩階段資料的本質：從「名詞定義」到「邏輯推理」  
Stage 1:粗糙但海量的「連連看」資料 (Image-Caption Pairs)  
Stage 2:精緻且深入的「對話」資料 (Visual Instruction Tuning)  
7.Native Multimodal(原生多模態)  
拼接派（Modular / LLaVA 範式):  
非對稱性： 輸入可以是圖文，但輸出只能是文字  
級聯效應（Cascading）： 在語音對話中，它必須經歷「語音轉文字 (ASR) → 文字思考 → 文字轉語音 (TTS)」的三階段。這導致延遲極高  
原生派（Native Multimodal / GPT-4o 範式):  
對稱性（Any-to-Any）： 輸入是 Token（文字、像素、音訊），輸出也是 Token（文字、像素、音訊）。模型可以直接預測下一個視覺 Token 來「畫圖」，或是預測下一個音訊 Token 來「發聲」。  
流暢無縫（Fluid）： 語音對話不需要中介層，模型直接用音訊 Token 進行思維。它能聽懂你的嘆氣、能在回答時帶有幽默的語調，甚至能打斷你的說話。  
8.Grounding(視覺定位)，GUI (智能體)    
Robotics：精確指出畫面上要抓取或互動的物件位置，是 VLA模型的關鍵前置。   
UI Debug：在軟體測試中，能直接指出畫面上哪一個元素排版跑版或出錯。  
實際作法  
數字字串化：讓模型像說話一樣，直接預測並輸出代表座標的數字字串（例如生成 " [0.3, 0.4, 0.6, 0.7] "）。  
擴展詞表：將 Bounding Box 的座標範圍當作特殊的 Special Token（例如 <box_0> 等）直接放入模型的 Vocab 中，提高編碼與輸出效率。  
9.Video    
Sparse Sampling(稀疏採樣):不管影片多長，只均勻抽取其中的 8 / 16 / 32 幀（Frames）。每幀當作一張獨立圖片過 ViT，再拼起來丟給 LLM。   
Temporal Pooling(時空池化):在時間維度上運作。例如把相鄰 4 幀在同一個空間位置的 Token 拿出來做平均或拼接後用 Linear 層降維。  
Memory Module(記憶體模組):模型不一次性看完所有幀，而是設計一個固定大小的「記憶庫（Buffer）」。  
Dedicated Video Encoder(專用影片編碼器):直接使用 3D Transformer（同時在橫向、縱向、時間軸三個維度做 Attention）在海量影片資料上做預訓練，讓 Encoder 輸出的每一個 Token 原生就包含了一小段時間內的動態語意。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？
(a)VLM 多了視覺、CLIP的核心概念、LLaVA核心模組、高解析度解決方案  
(b)VLM 存在空間幾何盲區、長影片推理弱、高解析度 Token 爆炸等底層硬傷  

# part08 Vision-Language-Action
## 名詞理解 
1. 過去十年 robotics 的主流  
Classical Control + Motion Planner（傳統控制派）:依賴精確的 3D 幾何感測器（如光達、深度相機）重建環境，利用逆運動學和軌跡規劃算法來計算馬達轉角。  
Behavioral Cloning (BC / 行為複製 / 模仿學習):讓人類專家手動遙控機器人錄製 100 次「打開微波爐」的示範資料，然後用監督式學習強行讓神經網路背下「影像 → 馬達關節角度」的映射。  
Reinforcement Learning (RL / 強化學習):因為在現實世界撞壞機器人很貴，所以把機器人丟進模擬器裡，用 PPO、SAC 算法讓它瘋狂試錯幾億次，再透過 Sim-to-Real 技術遷移到實體機器人上。  
VLA idea：通常會將這些連續的數值對齊並歸一化（例如縮放到 [-1, 1] 之間），然後將這個區間均勻切成 256 或 1000 個格子。  
2.RT-2(Robotics Transformer 2):不需要為機器人設計特異的架構，只要把控制訊號(動作)『文字化』，規模化後的 VLM 就能原生展現出驚人的具身智能。  
賦予機器人強泛化力:當機器人面對從未見過的物體形狀、包裝、或是反光環境時，依然能看懂並成功執行任務。  
簡化機器人控制架構:訓練與推理時，輸入「圖像+語言指令」，輸出端就直接自回歸預測出 Action Token 序列。  
3.OpenVLA:解決了 RT-2 因為參數過大（55B）、代碼與權重閉源而無法在學術界和中小企業落地的痛點。  
DINOv2 + SigLIP:負責看懂「這是什麼物體」、「環境裡有什麼」; 自監督學習的視覺模型。  
Open X-Embodiment:匯集了全球超過 20 個機構、22 種不同機器人平台、高達 97 萬個操作片段的軌跡資料，並將其全部標準化。  
LoRA 微調:OpenVLA 在架構上就是標準的 Vision Encoder + Projector + LLM，只是輸出端換成了 Action Token。  
4.π_0  
Action Chunking:π_0一口氣預測未來 50 個步長的連續動作軌跡。  
Flow Matching運作流程:  
VLM 大腦感知：PaliGemma (3B) 看著影像與「打蛋」的指令，輸出高層級的語意特徵。  
動作專家去噪：Action Expert（一個輕量 Transformer）接收這個 VLM 隱向量，並拿著一串純隨機的噪聲。  
生成：透過 Flow Matching 算法，僅需極少步數的去噪，就直接將這串噪聲轉化為一條平滑、連續的 50 步物理動作軌跡。  
5.Data  
Open X-Embodiment:具身智能界的 ImageNet。 首次將跨機構、跨機械臂的資料標準化，是 OpenVLA、RT-2 的底座。但資料品質參差不齊。  
DROID:高質量、多場景。 研發人員把機械臂搬到真實的家庭、辦公室、戶外環境中採集，專治模型「離開實驗室就崩潰」的毛病。  
BridgeData V2:專精於特定高頻場景（切菜、收碗盤、抓取），是做專業服務型機器人的高純度滋養料。  
Sim2Real:利用 GPU 強化學習，在虛擬物理引擎裡同時運行 10,000 個 機器人手臂，幾小時內就能刷出幾億個步長的物理經驗。  
6.Dual-System:  
System 2:可以在後台利用思維鏈進行長時間的「慢思考」，慢慢評估最佳的物理抓取策略  
System 1:像人類的脊髓神經反應一樣，不受 System 2 思考時間的影響，持續以 500 Hz 的極高頻率維持身體平衡與手臂防抖  
7.硬體格局  
Tesla Optimus:從關節執行器、靈巧手、晶片到端到端 AI 大腦全部自己做。  
Figure:頂級 VLA 大腦與實體場景的強強聯手  
Physical Intelligence:模型的原生研發  
1X (NEO Gamma):家用場景切入、安全第一  
Unitree:價格屠夫、極速迭代  
Boston Dynamics:傳統控制學霸轉向電動化與 AI 結合  
8.名詞表   
Action Chunk(動作塊預測):模型不只預測「下一步 t+1」的動作，而是一口氣預測未來 N 步（例如未來 50 步）連續的物理軌跡。  
Sim2Real(模擬到真實遷移):在電腦物理模擬器（如 NVIDIA Isaac Lab）裡用強化學習訓練 Policy，再直接放進實體機器人身上執行。  
Cross-Embodiment (跨形態遷移):讓同一個 AI 大腦，能夠直接控制不同物理形態的機器人（例如從單臂換到雙臂、甚至人形機器人）。  
Teleoperation(遠端遙控數據採集):由人類工程師戴著 VR 頭盔或操作主從式機械臂，親自遙控機器人去執行任務並錄下數據。  
Whole-Body Control (WBC / 全身控制):同時協調人形機器人的下肢動態平衡（走跳、踏步）與上肢靈巧操作（抓取、搬運）。  
9.π_0 的架構核心  
傳統 L2 / MSE 損失：模型會輸出平均動作 (a_1 + a_2)/2，結果就是機器人直衝撞上障礙物。  
Flow Matching 解法：不直接預測動作概率分布，而是學習一個確定性向量場，透過積分將隨機噪聲平滑推動至合理的動作集合（可精確落點在 a_1 或 a_2 的其中一個峰值）。  
Flow Matching為什麼比 Diffusion 快?  
Diffusion 的採樣軌跡是非線性的曲折 SDE/ODE，通常需要 20～50 步去噪；而 Flow Matching 採用直線插值，只需 5～10 步歐拉積分即可精準收斂，完美達成 50 Hz+ 的高頻控制需求。  




