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
6.資料工程(Data):







