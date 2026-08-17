## part10 GPU Programming
## 名詞理解
1.硬體 NVIDIA H100  
HBM3（主記憶體）：容量最大（80GB），但頻寬相對極慢（~3.3 TB/s）。  
L2 Cache：所有 SM 共享的高速快取（~5–12 TB/s）。  
SMEM（共享記憶體）：程式員可控 的 Scratchpad（~30 TB/s）。  
Registers（暫存器）：最快、容量最小，由編譯器/硬體自動分配。  
2.CUDA 入門  
thread → warp (32) → block → grid  
Thread：最基礎的軟體執行單元。  
Warp：32 個 Thread 組成 1 個 Warp。  
GPU 硬體調度的基本單位。  
Warp 內部的 Thread 採用 SIMT (Single Instruction, Multiple Threads) 模式發射指令與執行。  
Block：由多個 Warp 組成，同一個 Block 內的 Thread 會被派發至同一個 SM (Streaming Multiprocessor) 上執行。  
Grid：單次 Kernel Launch 所發射的所有 Block 集合。  
3.Triton  
自動管理 SMEM:開發者不需要宣告 __shared__ 變數或手寫資料搬移邏輯，Triton 編譯器會自動規劃 L1/SRAM 快取與暫存器配置。  
自動記憶體合併存取:自動將同一個 Block 內連續的記憶體讀寫請求合併成單一高頻寬的匯流排傳輸，避免非對齊存取造成的效能懲罰。  
自動指令向量化與流水線:自動映射至 GPU 的 SIMD 向量指令集，並自動調度指令指令流水線來隱藏記憶體存取延遲。  
4.Optimization  
Tiling（分塊技術）  
將大矩陣分割成 BLOCK x BLOCK 的 Tile（小區塊）。  
批次將 Tile 搬移至晶片上的 Shared Memory 或 Registers（存取延遲僅個位數 Clock Cycles）。  
Kernel Fusion（算子融合）  
將連續多個運算的計算邏輯合併寫在同一個 Kernel 內。  
中間結果直接留在 Register / SMEM 中傳遞給下一個算子，完全不經過 HBM。  
Occupancy（佔有率）  
GPU 依靠超多執行緒來掩蓋延遲。   
當某個 Warp 因為等待 HBM 讀取而阻塞時，SM 的 Warp Scheduler 會瞬間切換至另一個就緒的 Warp 繼續執行計算。  
5.Profile    
Roofline Model:X 軸：算術強度 (Arithmetic Intensity, AI)，Y 軸：實際吞吐量 (Throughput / Performance)，斜線區：Bandwidth Bound 特徵：算術強度低，計算單元（ALU/Tensor Core）常處於等待資料傳輸的空轉狀態。水平線區：Peak FLOPS 特徵：算術強度高，記憶體頻寬已不是瓶頸，硬體計算核心已滿載。  
Nsight Compute:專為單一自訂 CUDA / Triton Kernel 打造的硬體級調優工具。  
Nsight Systems:NVIDIA 系統級分析工具，觀察 CPU、GPU、網路與 API 之間的宏觀互動。  
6.Memory-bound vs Compute-bound  
算術強度 (Arithmetic Intensity, AI):代表每從記憶體（HBM）搬移 1 Byte 的資料，GPU 可以進行多少次浮點運算。  
低強度(AI < 10) —— Memory-Bound  
瓶頸：HBM 記憶體頻寬餵不夠快，運算核心（Tensor Core / ALU）經常處於餓死、空轉的狀態。  
高強度(AI > 100) —— Compute-Bound  
瓶頸：記憶體頻寬充足，運算單元全速運轉，達到硬體極限。  
硬體天花板與 Ridge Point（以 H100 為例）  
當你的 Kernel 的算術強度低於硬體的 Ridge Point（例如只有 10 FLOP/Byte），盲目買更貴、算力更高的 GPU 也完全救不了效能，因為瓶頸根本不在運算能力，而在 HBM 頻寬。  
唯一解法：透過優化將資料留在晶片內的 SMEM / Register 進行重複利用（Data Reuse），降低對 HBM 的依賴。  
7.FlashAttention   
Tiling + Online Softmax（前向傳播不落地）  
Tiling：將 K, V 切成小 Block 載入高速的 SMEM 中。  
Online Softmax：利用數學技巧在分塊迭代的過程中，動態更新 running max 與 running sum（無需等整列算完才能算 Softmax 分母）。  
結果：永遠不 Materialize（生成）完整的 N x N 矩陣，中間結果全留在 Register / SMEM 內，最後只把最終的 O_i 寫回 HBM（僅 1 次 HBM 讀寫）。  
Recompute backward（反向傳播時「用算力換記憶體」）    
反直覺的優化：傳統需要把 N x N 的中間矩陣存起來供 Backward 使用。FlashAttention 在 Backward 時乾脆重新算一遍。  
物理依據：現代 GPU 的 FLOPs（算力）遠比 HBM 頻寬充裕。把資料從 HBM 讀寫一次的時間，足夠 GPU 重新把運算跑好幾次。因此「重算」反而比「從 HBM 讀寫」更快、更省記憶體。  
8.名詞表
Bank Conflict (記憶體衝突):Shared Memory (SMEM) 劃分為 32 個 Banks。當同一 Warp 的多個 Thread 同時存取同一個 Bank 不同的 Address 時發生。  
Coalesced Access (合併讀取):同一 Warp 內連續的 Thread 讀取連續的記憶體位址。  
Async Copy / TMA (非同步記憶體傳輸):以 H100 的 TMA (Tensor Memory Accelerator) 為代表，支援背景非同步將資料從 HBM 搬到 SMEM。  
Tensor Core (矩陣乘法加速器):專為半精度/低精度（FP16/BF16/FP8）矩陣乘法打造的硬體 ASIC，一個 Cycle 能吞掉一個小矩陣乘加。  
Roofline Model (效能天花板模型):以算術強度 (FLOP/byte) 為 X 軸、吞吐量為 Y 軸的模型。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
從傳統的 CUDA C++（Thread-Level） 到現代的 Triton（Block-Level），GPU 編程正在從「手動微觀控制每個執行緒與記憶體位址」轉向「以 Tile 陣列為單位的自動化編譯優化」。  
FlashAttention 透過演算法與硬體資料流的重組（Tiling + Online Softmax），硬生生將 Attention 的算術強度從 ~ 1 拉到 ~ 50+，解決了長文本的記憶體瓶頸。  
當 Kernel 算術強度低於硬體 Ridge Point 時，盲目追求算力或買更貴的 GPU 是無解的，唯一解法是透過 Tiling 與 Kernel Fusion 將資料留在晶片內的 SRAM（SMEM / Register）中重複利用。  
Profiling 工具（torch.profiler → Nsight Systems → Nsight Compute）構成了一套從應用層到微觀 Kernel 級的完整除錯與優化閉環。  
(b)  
傳統 CUDA 開發的極高門檻與脆弱性:開發者必須手動處理複雜的索引計算、__syncthreads() 同步屏障以及記憶體對齊。程式碼冗長（往往比 Triton 多出 10 倍以上），且極易因一個微小的邊界疏忽或 Warp Divergence 導致效能崩潰或數值錯誤。  
傳統深度學習算子（如 Naive Attention）的頻寬浪費:傳統實作將每個小運算（Matmul、Scale、Mask、Softmax、Dropout）拆成獨立的 Kernel。中間產生的巨型矩陣（如 N x N 的 Attention Matrix）必須不斷寫回慢速的 HBM 再讀出來。這造成了巨大的記憶體頻寬浪費O(N^2) 空間複雜度），並使硬體陷入嚴重的 Memory-Bound 泥沼，成為限制大模型支援長上下文（Long Context）的根本物理障礙。  


## part11 Training Infrastructure
## 名詞理解
1.Data Parallel·最簡單    
完整模型複本：每一張 GPU（或 Worker）都完整複製一份一模一樣的 Model Weights 與 Optimizer States。  
Batch Shard（資料切分）：將總 Batch Size 切成多個小分片（Micro-batch），分別送往不同的 GPU 進行獨立的前向傳播（Forward）。  
限制：Model Memory 瓶頸  
每一張 GPU 必須完整容納整個 Model（包含模型參數、梯度、Optimizer 狀態以及 Activation 記憶體）。  
隨著大語言模型進入 Billion-scale（百億、千億參數） 時代，單張 GPU 的 HBM（例如 80GB）根本塞不下完整的模型與狀態。  
2.ZeRO / FSDP · 切 optimizer  
Stage 1：切分 Optimizer States (切優化器狀態)  
將 Optimizer States 平均切成 N 份，分存到 N 張 GPU 上。每張卡只負責更新自己分配到的那部分參數。  
Stage 2：加切 Gradients (同時切分梯度)  
不僅切 Optimizer States，連 Gradients 也一併切片。當 Backward 算完某個分片的梯度時，直接 Reduce-Scatter 到對應負責的 GPU 上。  
Stage 3：加切 Parameters (同時切分模型參數) → FSDP  
連 Model Parameters 本身也徹底切碎（Sharding）分存。  
3.Tensor Parallel · 切矩陣  
切分方式 (Column-wise & Row-wise Sharding)  
Attention 層 (QKV Projection)：將 Q、K、V 的權重矩陣沿著 Attention Head 維度橫向或縱向切開，每張 GPU 只計算一部分的 Heads。  
MLP / FFN 層：將第一層權重（Col-wise）沿 Hidden Dimension 切開，第二層權重（Row-wise）相對應地切開，讓計算結果能在區塊內完美銜接。  
使用時機:當模型大到單一層（Single Layer）的權重矩陣根本塞不進單張 GPU 的 HBM 時，必須透過 Tensor Parallelism 將運算與權重拆開。  
4.Pipeline Parallel · 切層  
將模型的 N 個 Layers 垂直切成 K 個連續的區段（Stages），依序指派給 K 張（或 K 組）GPU  
痛點:在流水線剛啟動（Filling）與快結束（Draining）時，後段或前段的 GPU 因為拿不到資料而被迫處於閒置狀態（Idle）。這段空白期稱為 Pipeline Bubble，會直接降低硬體的整體運算利用率（MFU）。   
5.3D / 4D Parallel · 組合拳  
以超大規模叢集（如 16k GPUs）為例的典型配置：  
TP = 8 (Tensor Parallelism)：嚴格限制在單機內部（Intra-node），利用 8 張 GPU 透過高速 NVLink 扛住巨大通訊量。  
PP = 16 (Pipeline Parallelism)：將模型垂直切成 16 個 Stages，跨越不同的機器節點串聯。  
DP = 128 (Data Parallel / FSDP)：在巨型叢集規模下，將剩餘的節點組成 128 個資料平行群組，同步模型梯度。  
EP = N (Expert Parallelism, 專家平行)：若模型採用 MoE (Mixture of Experts) 架構（如 DeepSeek-V3），則會引入 EP，將不同的 FFN 專家（Experts）分佈到不同的 GPU 上。  
總 GPU 數量計算:Total GPUs = TP x PP x DP x EP = 8 x 16 x 128 x (EP Factor) = 16k+ GPUs  
6.Memory · activation checkpointing 與 offload  
Activation Checkpointing (活性值檢核點 / 空間換時間):Forward 時刻意不儲存大部分層的 Activations，只保留少數關鍵節點（Checkpoints）。等到 Backward 需要時，再從最近的 Checkpoint 重新執行一次 Forward 算出來。代價：運算量增加約 33%（因為部分的前向計算被執行了兩次）。這是一場極其划算的「用計算時間換取寶貴 HBM 空間」的交易。  
CPU / NVMe Offloading (記憶體卸載):當模型大到連 FSDP 切片後還是塞不進 GPU 的 HBM 時，可以將暫時用不到的資料（例如 Optimizer States 或甚至部分的 Model Parameters / Gradients）搬移（Offload）到 CPU 主機記憶體（RAM）甚至是極速的 NVMe SSD 中。  
低精度訓練時代 (BF16 與 FP8)  
BF16:擁有與 FP32 相同的動態範圍（8-bit exponent），徹底解決了 FP16 容易數值下溢（Underflow）或溢出（Overflow）的痛點。自 2022–2024 年起已成為大模型訓練的預設基礎資料類型。  
FP8:在 H100、B100 等新一代硬體上，FP8 支援硬體級的張量加速。  
7.容錯  
定期 Asynchronous Checkpoint (非同步檢查點存檔):採用非同步存檔（Async Checkpoint）。在 背景（Background）透過雙緩衝區將模型狀態緩慢傾印（Dump）到高速的 NVMe 儲存池或遠端 Blob 儲存（如 S3），主訓練迴圈完全不被卡住。  
Health Check 與自動備援替換 (Spare Node Auto-Swap):叢集監控系統持續進行心跳檢測（Health Check）。一旦偵測到某張 GPU 或節點出現異常（如 ECC 記憶體錯誤激增、溫度過高或 Kernel 逾時），調度系統會自動將故障節點隔離，並無縫替換為預備的 Spare Node。  
Deterministic Replay (確定性重放與可重現性):透過固定亂數種子、強制算子執行順序等技術實現 Deterministic Replay，確保訓練過程具備高度的 Reproducibility（可重現性）。  
8.主流 framework  
PyTorch FSDP2:2024 年後的 PyTorch 官方預設標配。  
Megatron-LM / Megatron-Core:Tensor Parallelism (TP) 與 Pipeline Parallelism (PP) 的教父級框架。  
DeepSpeed:ZeRO 技術的發源地與早期推動者。   
Ray Train:叢集調度與超參數搜尋引擎。  
9.fat tree / rail-aligned  
Intra-node (節點內部):8 張 GPU 透過 NVLink + NVSwitch 實現全互聯（Fully Connected），提供高達 900 GB/s 雙向頻寬。   
Inter-node (節點之間):每張卡透過 InfiniBand (如 NDR 400 Gbps × 8 = 3.2 Tbps per node) 或 RoCE 網路卡對外連線。  
Topology (拓撲設計):像樹狀結構一樣，愈往核心交換器（Spine Switch）頻寬愈大，確保任意兩點之間都能維持固定的高頻寬。  
10.名詞表   
NCCL(NVIDIA Collective Communications Library):GPU 叢集上的通訊基石（相當於分散式運算世界的 MPI），專門最佳化 GPU 之間的集體通訊效率。  
All-Reduce (同步梯度):每張卡持有部分資料（如梯度），透過 Ring 或 Tree 演算法將所有卡的資料相加後，讓每張卡都拿到完整加總結果，是 Data Parallelism 實現同步的關鍵。  
All-to-All (MoE 專屬通訊):每張卡需要將不同的 Token 派發給對應的 Expert 所在卡上。  
Bubble (Pipeline 氣泡):Pipeline Parallel 啟動與收尾時，部分 GPU 因為拿不到資料而被迫閒置的時間比例。Stage 越多，Bubble 通常也越大，需透過 1F1B 等排程來克服。  
Mixed Precision (混合精度訓練):保留 FP32 Master Weights 確保數值穩定，而前向/反向傳播採用 BF16 或 FP8。  
RDMA / IB (InfiniBand / RoCE):允許遠端 GPU 直接讀寫彼此的記憶體，全程繞過 CPU 與作業系統核心（Kernel Bypass）。提供高達數百 Gbps 的頻寬，是跨節點（Inter-node）萬卡訓練的命脈。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
從百億到千億甚至上兆參數的 Frontier Models（如 LLaMA 3 405B、DeepSeek-V3），單張 GPU 的 HBM 根本無法容納模型，催生了高度複雜的 3D / 4D 混合平行化架構（TP + PP + DP/FSDP + EP）。  
Meta 在 LLaMA 3 訓練中經歷了數百次自動中斷與恢復，體會到在 10k+ GPU 規模下，硬體故障不再是例外，而是必然會發生的常態。  
通訊頻寬與尾端延遲（Tail Latency）決定了萬卡叢集的生死。叢集的總吞吐量不取決於算力有多快，而是取決於 P99 Collective Time——只要有一張卡因為網路抖動或過熱慢了一毫秒，整張表的數萬張卡就必須集體空轉等待。  
記憶體優化本質上是一場「用時間換空間、用通訊換容量」的權衡。  
(b)  
傳統資料平行（DDP）的記憶體極限與浪費:在傳統 DDP 下，每一張 GPU 都必須完整複製一份 Model Weights、Gradients 與 Optimizer States。當模型規模突破數百億參數後，單卡 HBM 根本裝不下，導致純資料平行直接失效。  
跨機通訊頻寬的巨大鴻溝與木桶效應:儘管 InfiniBand 網路已經達到 400Gbps–800Gbps，但相較於 NVLink（900 GB/s），跨機網路依然慢了數個量級。如果平行化策略（如 TP）錯誤跨出節點邊界，龐大的通訊開銷會瞬間癱瘓整個運算叢集，導致算力利用率（MFU）慘跌。  




## part12 Diffusion & Flow Matching
## 名詞理解
1.DDPM  
Forward Process (前向加噪過程):將真實影像 x_0 依照變異數排程（Variance Schedule β_t），透過馬可夫鏈逐步加入高斯噪聲。  
Reverse Process (反向去噪過程):從純噪聲 x_T 開始，利用神經網路逐步預測並剝離噪聲，倒推回乾淨的影像 x_0。  
2.Latent Diffusion · Stable Diffusion 起源  
Encoder (編碼階段):使用預先訓練好的 VAE ，將高解析度的原始圖像（如 512 x 512 x 3）壓縮並對齊到低維度的 Latent 特徵空間（如 64 x 64 x 4）。  
Latent Diffusion (潛在擴散核心):擴散模型的加噪與去噪（U-Net 或 DiT）完全在這個低維度的 Latent 空間中執行。結合 Text Prompts（透過 CLIP / T5 等文字編碼器）進行 Cross-Attention 條件導引。  
Decoder (解碼還原階段):當擴散過程完成、在 Latent 空間去噪完畢後，透過 VAE Decoder 將 64 x 64 的 Latent 特徵還原放大為高畫質的 RGB 像素圖像（512 x 512）。  
3.Classifier-Free Guidance  
無須外掛分類器，直接讓神經網路本身同時具備「有條件（Conditional）」與「無條件（Unconditional）」的去噪能力。  
在訓練過程中有一定比例（通常為 10%）的機率，隨機將條件標籤 c 替換為空標籤 ∅（Dropout condition）。  
優點:提示詞對齊度、圖像品質。  
代價:喪失多樣性、飽和過曝、計算成本加倍。  
4.Flow Matching   
拋棄繁瑣隨機過程，改用 Optimal Transport (最優運輸) 或 Conditional Flow Matching (CFM) 概念。  
強制在真實資料 x_0 與噪聲 x_1 之間建立最短的直線軌跡，讓生成過程變成單純的向量場迴歸。  
推論過程:從純噪聲 x_1 開始，利用數值 ODE Solver（如 Euler method 甚至是 Runge-Kutta）沿著學到的速度場 dx/dt = v_θ(x, t) 一路「直直滑向」 x_0。  
等價關係:Flow Matching 數學上與 Rectified Flow (Liu 2022) 高度等價，兩者共同奠定了當代高速生成的理論基石。  
5.DiT  
Patchify：將 Latent 影像切成一塊塊的 Patches（如同 ViT），轉化為 Transformer 熟悉的 Token 序列。  
完全基於 Transformer 架構：將 Attention Mechanism 帶入擴散模型，全面解放空間建模的能力。  
adaLN-Zero 機制  
DiT 捨棄傳統的簡單相加或 FiLM 層，改用 Adaptive Layer Normalization (adaLN)。  
透過一個小型 MLP，把時間 t 與條件 c 轉化為 Scale (γ)、Shift (β) 參數，直接對 Transformer Block 的 LayerNorm 進行動態調控。  
Zero Initialization：將 MLP 的最後一層權重初始化為 0，確保訓練初期整個 Transformer 區塊等同於 Identity Mapping（恆等映射），讓深層網路訓練極其穩定。  
6.Video diffusion  
Spacetime Patch:傳統影片生成將影片視為獨立的畫面幀串聯，容易造成閃爍。Sora 將影片在時間軸與空間軸上同時切片，把影片轉化為一串三維的時空 Token 序列。  
DiT(Diffusion Transformer):結合前述的 Diffusion Transformer 架構，讓模型能夠同時處理超長距離的空間細節與時間維度的前後因果關係。  
核心技術挑戰:  
時間一致性:如何避免相鄰畫面之間出現不自然的抖動、形變或物件憑空閃爍。  
長片段生成:隨著影片秒數拉長，Transformer 的記憶體消耗呈爆炸性成長（需結合 Context Parallelism 與高效 Attention 架構）。  
相機運動與物理規律:如何精準對應複雜的相機運鏡指令（如環繞、推鏡），並符合真實世界的重力、碰撞與運動慣性。  
物體恆存性:確保物件在被遮擋、移出畫面又移回時，其外觀、身份與材質保持完全一致，不會無故「換臉」或消失。  
7.Sampler  
NFE:  
指在整個生成過程中，神經網路（Backbone）被呼叫（Forward）的總次數。  
代價：每多一次 NFE 就代表多一次模型前向運算，直接決定了生成速度的快慢。將採樣步數從 1000 步壓到 4 步，是近年生成式 AI 落地最核心的工程突破。  
採樣器演進與四大主流類型:  
DDPM-NFE：1000 步。  
特性：最原始、數學性質最穩定的隨機微分方程式（SDE）解法，但速度極慢，現今多用於理論基準而非實際推論。  
DDIM-NFE：50 步。  
特性：由 Song 於 2020 年提出，將加噪過程簡化為常微分方程式（ODE）。支援決定性（Deterministic）採樣，且允許跳步（Sub-sampling），大幅加速生成。  
DPM-Solver / DPM++ - NFE：15 ~ 25 步。  
特性：由 Lu 等人於 2022 年提出，利用高階數值 ODE 求解器。是 SDXL 的預設黃金標準，兼具極高畫質與合理速度。   
Euler / Heun - NFE：20 ~ 30 步。  
特性：標準的數值微分積分法，隨著 Flow Matching 的興起，其變體成為對應直線流軌跡的直覺解法。  
8.Conditioning  
Text:透過文字編碼器（如 CLIP、T5）將 Prompt 轉為特徵向量。  
Image & Spatial Control:由 Zhang 於 2023 年提出的 ControlNet 改變了遊戲規則。複製一份 U-Net/DiT 的 Encoder 作為鎖定的權重分支（Zero Convolution），外接任意空間結構條件（如 Canny 邊緣、Depth 深度圖、OpenPose 骨架、Segmentation 語義分割）。  
Reference & Identity:透過 Image Encoder（如 CLIP Image Encoder）將參考圖片轉為特徵，並透過額外的 Cross-Attention 注入到去噪網路中。  
Inpainting / Outpainting:將圖像透過 Mask 分成「保留區」與「重繪區」。輸入時將 Mask 頻道與噪聲拼接（Concatenate）進去噪網路。  
Action & Robotics:將機器人的視覺狀態（Images）與歷史感測器狀態（Robot State）作為 Condition，讓 Diffusion 直接生成連續的機器人動作序列（Action Chunks），展現出強大的連續控制與適應能力。  
9.名詞表  
Score:資料機率分佈的對數梯度（得分場）。早期 Score-based Generative Modeling 核心概念，指出擴散模型本質上是在學習把噪聲「推回」高密度資料區的向量場。  
v-prediction:捨棄傳統僅預測噪聲 (ε) 或純影像 (x_0) 的作法，改為預測綜合兩者的速度項 v = αε -  σx_0，大幅改善了高雜訊與低雜訊區間的數值穩定性，是 SD 2.1+ 與 Flow Matching 的標準作法。  
CFG Scale:文字到圖像（T2I）控制品質與對齊度的最大旋鈕。數值越高（如 SDXL 預設 s ≈ 7）提示詞黏著度越強，但過高會導致畫面過度飽和與失真。  
NFE:生成一張圖所需呼叫神經網路前向運算的總次數。透過採樣器優化與蒸餾技術，將傳統 DDPM 的 1000 步壓至現代主流的 15 ~ 28 步（甚至蒸餾到 1 ~ 4 步）。  
Distillation:透過 LCM、SDXL Turbo、Lightning 或 Schnell 等技術，將原本需要數十步的 Teacher 模型壓縮為 1–4 步的 Student 模型，實現即時生成（Real-time inference）。  
VAE:Latent Diffusion 的空間轉譯器。負責將高解析度像素空間壓縮至低維度 Latent 空間（例如 SD 1.5 進行 8x 下採樣轉為 4 通道），大幅降低運算成本。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
從傳統 DDPM 繁瑣的隨機漫步加噪過程，到 Latent Diffusion（SD 系列）將運算搬到低維度潛在空間，再到現代 Flow Matching（如 SD3、Flux）利用最優運輸將生成路徑拉直，看到了生成式 AI 在「數學乾淨度」與「採樣效率」上的劇烈演進。   
基礎架構（Backbone）全面由 U-Net 轉向 DiT (Diffusion Transformer)，並透過 adaLN-Zero 與 Spacetime Patch 成功解鎖了文字、圖像乃至影片（如 Sora）的大規模 Scaling。  
傳統擴散模型的最大核心痛點在於「曲折的生成軌跡」與「高昂的 NFE 運算成本」。Flow Matching 透過向量場回歸將軌跡變成直線，是數學與工程上極其優雅的典範轉移（Paradigm Shift）。  
現代生成模型的效能不再單純依賴單一演進，而是「模型架構（DiT） + 基礎數學（Flow Matching/EDM） + 高效採樣器（Euler/Distillation）」三者的強強結合。  
(b)  
傳統像素空間擴散與 U-Net 的運算天花板:若直接在高解析度像素空間進行擴散，運算量與 HBM 消耗呈爆炸性成長；而傳統 U-Net 骨幹在面對千億級或多模態大模型時，缺乏足夠優異的 Scaling Law（規模化效應），難以持續透過堆疊算力來突破畫質與語意理解的極限。  
採樣步數過多導致的推論延遲:早期 DDPM/DDIM 動輒需要 50 到 1000 步的迭代（高 NFE），導致生成一張圖或一段影片需要耗費大量時間，難以滿足即時互動（Real-time Inference）的產業落地需求，逼得學術與工業界必須透過蒸餾或 Flow Matching 來強行「拉直軌跡、壓低步數」。  



## part13 PEFT & Fine-tuning
## 名詞理解
1.LoRA  
大模型在進行下游任務微調（Fine-tuning）時，權重的變化量（ΔW = W' - W）雖然維度極高，但其實際表現出的「內稟秩（Intrinsic Rank）」非常低。  
在訓練過程中，基礎模型權重 W 被全面凍結（Frozen），只更新 A 與 B 的參數。這讓可訓練參數量從數十億驟降至幾百萬，記憶體消耗大幅降低。  
2.QLoRA核心技術   
NF4(NormalFloat 4-bit Quantization):研究發現預訓練模型的權重（Weights）數值分佈通常近似於常態分佈（Normal Distribution）。  
Double Quantization (雙重量化):量化時為了還原數值，會產生對應的縮放常數（Quantization Constants / Scales）。這些常數本身也佔用了不少記憶體。  
Paged Optimizers (分頁優化器):在訓練過程中，處理長序列或突發資料時，Optimizer States（如 Adam 的動態矩陣）容易瞬間暴增導致 OOM（Out of Memory）。  
3.DPO  
SFT → DPO 標準組合:    
SFT (Supervised Fine-Tuning)：先讓模型學會對話格式與基礎指令遵循。  
DPO (Direct Preference Optimization)：透過偏好數據（優選答案 vs. 劣選答案），直接在數學 Loss 上優化模型偏好，省去了傳統 RLHF 中複雜且不穩定的獎勵模型（Reward Model）與 PPO 訓練，成為 2024 年後開源模型的標配對齊方案。  
DPO 的四大主流演進變體:
ORPO (Odds Ratio Preference Optimization - 聯合優化):打破傳統「先 SFT 再 DPO」的兩階段限制，將 SFT 的語言建模 Loss 與 Preference Loss 合二為一，在單一訓練階段同時完成指令微調與偏好對齊，節省一半的訓練時間與資源。  
KTO (Kahneman-Tversky Optimization - 單標籤優化):傳統 DPO 必須依賴成對的資料（Pairwise: 哪個好、哪個不好）。KTO 借鑒行為經濟學理論，只需要單一的「好」或「壞」二元標籤即可訓練，大幅降低了高昂的人工偏好資料對齊成本。  
SimPO (Simple Preference Optimization - 參考模型免除):傳統 DPO 訓練時需要同時載入一個龐大的 Reference Model 來計算基線機率，消耗大量 VRAM。SimPO 巧妙地直接利用生成回覆的平均對數機率（Average Log-probability）作為隱含獎勵，完全不需要 Reference Model，訓練更輕量且表現優異。  
GRPO (Group Relative Policy Optimization - 群體相對策略優化):由 DeepSeek 團隊（如 DeepSeek-R1）大放異彩的強化學習對齊演算法。它捨棄了傳統 PPO 需要額外訓練一個巨大的 Critic 模型的作法，改為針對同一個 Prompt 同時生成一組（Group）多個輸出，透過群體內部的相對分數進行歸一化與優化，極大地降低了大模型 RL 階段的運算與記憶體瓶頸。  
4.Data 準備  
在大模型微調領域，「資料的品質與乾淨程度」往往比選擇哪一種微調演算法（LoRA vs. QLoRA）或超參數調整來得更具決定性。垃圾進，垃圾出（Garbage in, garbage out）。  
SFT (Supervised Fine-Tuning) 資料準備:現代開源框架（如 LLaMA-Factory、Axolotl）普遍採用類似 OpenAI ChatML 的結構  
Preference Data (偏好對齊資料) 準備:用於對齊（Alignment）階段的成對偏好資料，結構通常包含提示詞、勝出回答與落敗回答，確保 chosen 的回答品質、安全規範與格式遵循度明顯優於 rejected，才能讓模型學到正確的價值觀與偏好方向。  
5.實戰流程  
Step 1：挑選合適的基礎模型  
針對不同的硬體與任務需求，現代主流的輕量到中型開源模型是入門首選，例如 Llama 3.2 3B（超輕量邊緣部署）、Qwen 2.5 7B（全能中文/程式碼能力極強）或 Gemma 3 4B。  
Step 2：精心準備 SFT 資料 (Data Curation)  
核心原則：「少而精（Quality > Quantity）」遠勝過盲目堆疊龐大但雜亂的資料。  
規模：一般特定領域微調準備 1,000 到 10,000 筆高品質對話樣本即可見效。務必確保 JSON 格式正確並套用對應的 Chat Template。  
Step 3：高效訓練架構 (Unsloth + QLoRA 組合拳)  
工具配置：採用 Unsloth（大幅加速訓練並節省 VRAM 的開源加速庫）搭配 QLoRA。  
參數建議：4-bit Base Model 載入（極度省顯存） + Rank 設為 r = 16，且 Target Modules 建議掛載在所有 Linear 層（全線性層 LoRA），以確保模型能夠學習到足夠的領域知識。  
Step 4：客觀評估與驗證 (Evaluation)  
作法：透過標準基準測試（如 MT-Bench）或自行建立的驗證集（Custom Eval Set）來檢驗微調後的模型是否產生「災難性遺忘（Catastrophic Forgetting）」或推理能力退化。  
Step 5：偏好對齊與強化 (DPO Alignment)  
作法：準備 100 到 5,000 筆偏好對齊資料（Chosen vs. Rejected Pairs），透過 DPO 階段進一步校正模型的語氣、安全性與指令遵循傾向，使其更符合人類使用習慣。  
Step 6：合併與權重匯出 (Merge & Export)  
作法：訓練完成後，將 LoRA 權重與 4-bit/BF16 Base Model 進行融合（Merge），並匯出成標準的 safetensors 格式（用於伺服器端部署如 vLLM）或 GGUF 格式（用於消費級電腦的 Llama.cpp / Ollama 本地端離線運行）。  
6.PEFT  
零推論延遲的關鍵 (Zero Inference Latency)  
像 LoRA 與 DoRA 這類基於矩陣加法或權重合併架構的方法，在訓練完成後可以將更新量直接「融回（Merge）」原本的底座模型中，在推論時不會增加任何額外的計算負擔，這是它們能夠大規模落地工業界的最大原因。  
精確度與成本的權衡 (DoRA vs. LoRA)  
傳統 LoRA 假設權重更新的空間是均勻的，而 DoRA 透過數學上更符合神經元特性的「大小-方向分離」，在不增加太多訓練負擔的情況下，彌補了 LoRA 在某些複雜推理或微調任務上的表現上限。  
7.Multi-LoRA Serving  
傳統架構的痛點:若要同時為 1,000 個不同企業或客戶提供專屬微調模型（SaaS 模式），如果為每個客戶部署一個獨立的 Base Model，硬體成本（VRAM 與 GPU 數量）將呈天文數字成長，根本無法規模化。  
Multi-LoRA Serving 的破局解法:  
共用基底 + 輕量外掛：整個叢集只需要常駐 一個大型 Base Model（佔用大部分 VRAM），而每個客戶幾百 MB 的 LoRA 權重可以動態載入與卸載。  
成本效益：硬體成本從「1,000 個完整大模型」驟降為「1 個 Base + 1,000 個輕量 Adapter（每個約 100MB）」，讓 AI SaaS 的商業模式得以真正落地。  
核心工程挑戰與主流實現方案:  
動態切換與異構批次處理的挑戰:在同一個 Batch 裡面的不同 Requests 可能需要呼叫不同的 LoRA 專屬權重（例如 Request A 要客服風格 LoRA、Request B 要法律合約 LoRA）。系統必須在矩陣乘法時即時切換或組合對應的 Δ W = BA，而不能中斷整個 GPU 管道。  
S-LoRA:提出 Unified Paging（統一記憶體分頁） 與 Heterogeneous Batching（異構批次處理）。   
vLLM LoRA Support (產線標配):每個 incoming request 帶有對應的 adapter_id，結合 PagedAttention 機制在記憶體中高效共用 KV Cache 同時動態套用對應的 LoRA 矩陣，兼具高吞吐與低延遲。  
8.名詞表  
Rank r (低秩矩陣秩數):LoRA 最核心的超參數。決定了 Δ W 的表達能力與參數量。實務上通常設在 8 ~ 64 之間（若採用全線性層掛載，較小的 r 即可；若只掛載注意力層，則可開高一點）。  
NF4 (4-bit NormalFloat):QLoRA 專為大模型權重（近似常態分佈 $\mathcal{N}(0, 1)$）量身打造的最優 4-bit 資料型態，將傳統 INT4 量化的資訊損失降到最低。    
Chat Template (對話模板與特殊標記):不同開源模型各自定義的特殊控制字符（如 <|im_start|>、[INST]、<start_of_turn>）。如果資料與模型的模板不匹配，輕則回答錯亂，重則導致模型崩潰或無限鬼打牆。  
Reference Model (參考模型):在 DPO / PPO 等偏好對齊演算法中，用來限制新模型（Policy）不要偏離初始能力太遠的凍結基準模型。代價是會佔用大量 VRAM，因此催生出如 SimPO、ORPO 等免除 Reference Model 的變體。  
Continued Pre-training (持續預訓練):有別於 SFT（教導對話格式與指令遵循），持續預訓練是餵給模型大量的領域內原生文字（Raw Text），同樣以 Next-token prediction 為目標，用來將全新領域的專業知識「刻」進模型底層，通常會接在 SFT 與 DPO 之前。  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
 當模型規模突破數百億參數時，全量微調（Full Fine-Tuning）因需要存儲龐大的 Optimizer States 與梯度，導致 VRAM 需求呈爆炸性成長，單卡或小規模叢集根本無法負荷。  
 在工業界部署中，看到多租戶 AI SaaS 服務不再為每個客戶盲目複製獨立的大模型，而是透過「一個 Base Model + 上百個輕量 LoRA Adapter」的動態切換架構，徹底顛覆了硬體成本結構。  
 參數高效微調的本質是利用權重更新的低秩內稟特性（Low-Rank Intrinsic Dimension）；而資料的品質與 Chat Template 的正確性（如避免特殊字符錯置），往往比選擇哪種微調演算法更能決定模型成敗。  
(b)  
全量微調（Full Fine-Tuning）的記憶體災難:在傳統全量微調中，模型每一個參數都需要參與梯度更新並保存動態優化器狀態。面對數十億到數千億參數的大模型，這對硬體算力的要求極高，將多數中小企業與研究團隊徹底阻絕在外。  
多租戶 SaaS 部署的硬體成本困境:若採用傳統架構為成百上千個不同的企業客戶提供專屬微調模型，必須為每個客戶常駐一個完整的大模型個體，導致 VRAM 資源嚴重浪費，商業模式無法規模化。  
傳統對齊技術的繁瑣與不穩定:傳統 RLHF 流程必須先訓練一個複雜的獎勵模型（Reward Model），再透過強化學習（PPO）進行策略優化。這套流程涉及多個模型的同時加載與不穩定訓練，極易發生梯度崩潰，促使學術與工業界轉向更乾淨的直接偏好優化（如 DPO）。  

## part14 RAG & Prompt Engineering
## 名詞理解
1.RAG  
Chunking (切塊策略):將長篇文件切割成適當大小的片段（通常建議 Chunk Size: 512–1024 tokens，並設定 Overlap: 64 tokens 以防上下文語意在切口處斷開）。  
Embedding (向量化模型):將文字轉為高維度稠密向量（Dense Vectors）。主流開源與商用選擇包括 BGE、E5、OpenAI text-embedding-3、Voyage AI 與 Cohere Embed v3。    
Vector Database (向量資料庫):用於高效儲存與向量檢索（Approximate Nearest Neighbor, ANN）。主流工具包含 Qdrant、Weaviate、pgvector（PostgreSQL 擴充）、Pinecone 與 LanceDB。  
Query Processing & Hybrid Search (混合檢索):使用者輸入 Query 後先進行 Embedding 向量檢索（擅長語意理解），同時結合傳統關鍵字搜尋 BM25（擅長精確專有名詞、代號、人名）。  
Reranking (重排機制 - 效能分水嶺):初步檢索撈出較多候選（如 Top 50），再透過精準度極高的 Cross-Encoder（如 bge-reranker、Cohere Rerank）進行運算成本較高但極度精細的 Query-Document 交互比對。  
Context Stuffing (上下文組裝):將經過重排篩選出的 Top-N 高品質文檔片段（Context）與使用者的原始問題依照特定 Prompt Template 組合後，塞入 LLM 的 Context Window 中。    2.Embedding 模型  
2024+ 開源向量模型 SOTA (State-of-the-Art):  
BGE-M3 (FlagEmbedding)：支援多語言、多功能（Dense + Sparse + Multi-Vector），是當前企業級 RAG 的多功能標配。  
E5-Mistral / gte-Qwen：基於 LLM 架構（如 Mistral / Qwen）微調而成的 Embedding 模型，擁有極強的語意理解能力與超長 Context 支援。  
jina-embeddings-v3 / Stella：在特定任務表現優異，且支援先進的動態維度調整。  
現代 Embedding 的三大殺手級特性:    
多語言支援 (Multilingual)：原生支援跨語言檢索（如用中文 Query 撈出英文文件）。  
長上下文支援 (Long Context)：支援數千甚至 8k+ tokens 的輸入，適合直接將整篇論文或長報告轉成向量。  
馬特廖什卡表示學習 (Matryoshka Representation Learning, MRL)：允許在不重新訓練的情況下，動態截斷向量維度（例如將 1536 維壓到 512 維或 256 維）。能夠在幾乎不損失檢索準確率的前提下，大幅節省向量資料庫的儲存空間與計算開銷。  
3.Chunking-被低估    
傳統切塊  
Fixed Token Chunking (固定 Token 切割):設定固定大小（例如 512 tokens）與重疊（Overlap），不管三七二十一直接硬切。  
Sentence / Paragraph Boundary (基於語句與段落邊界):透過語言工具（如 spaCy、NLTK）識別自然語言的句點、換行與段落邊界進行切分。  
現代進階切塊  
Semantic Chunking (語意切塊):計算相鄰句子之間的 Embedding 語意相似度。當連續句子之間的相似度突然驟降（代表主題轉折、進入新段落），系統便自動在此處設為斷點。  
Hierarchical Chunking / Parent-Child Chunking (階層式切塊):將文件切成大範圍的 Parent Chunk（例如整段文章）與細粒度的 Child Chunk（例如句子或小段落）。  
異質資料的專門切法  
Code (程式碼):不能用 Token 亂切，必須依賴 AST (Abstract Syntax Tree) 或函式/類別（Function/Class）邊界進行語法級別的切塊，否則函數被卡斷會失去程式邏輯。  
Tables (表格):表格若被隨意切碎會變成無意義的字串。通常需轉換為 Markdown 格式或 HTML 結構，並將表頭（Header）與對應數值綁定在同一個 Chunk 中。  
PDFs (排版複雜文檔):傳統 PDF 解析容易把雙欄、頁首、頁尾、圖表文字混在一起。現代 RAG 強調 Layout-aware parsing（如使用 Marker、MinerU 工具），先辨識版面結構後再進行邏輯切塊。  
4.Prompt  
CoT (Chain of Thought - 思維鏈):強制模型將複雜問題拆解為多個步驟逐一推導，大幅提升數學、邏輯與程式碼編寫的正確率。  
Few-shot Prompting (少樣本提示):挑選示範時，樣本必須具備多樣性（Diverse），並刻意涵蓋各種邊界條件（Edge Cases），引導模型模仿正確的輸出邏輯與格式。  
Structured Output (結構化輸出強制):純文字的 LLM 回覆很難讓後端程式直接解析。解法：結合 JSON Schema 或底層語法約束解碼技術（Grammar-constrained decoding，如 Outlines、Instructor），在 Token 生成階段直接限制模型只能吐出符合特定格式的 JSON，徹底告別字串解析崩潰。  
Self-consistency (自我一致性):針對同一個複雜問題，讓 LLM 獨立採樣生成 $N$ 個不同的推理路徑與答案（Temperature > 0），最後透過多數決（Majority Vote）選出最穩健的最終答案。  
Reflexion / Self-Critique (反思與自我審查):多輪對話或 Agent 架構的經典延伸。讓模型在第一輪生成答案後，啟動第二輪的「自我檢查與批判」，針對潛在漏洞進行修正與優化，顯著降低低級錯誤率。  
5.進階 RAG  
HyDE (Hypothetical Document Embeddings - 假想文件嵌入):先讓 LLM 針對 Query 憑空寫出一段「虛構的理想答案」，再將這段假答案轉成 Embedding 去向量資料庫進行檢索。  
Multi-Hop Retrieval (多跳檢索):將主 Query 自動分解成多個子問題（Sub-queries），進行依序或疊代的多次檢索，逐步拼湊出完整答案。  
Agentic RAG (智慧體 RAG):模型可以自己決定「何時該發動檢索」、「要用什麼關鍵字搜」、「檢索出來的結果不夠好時是否要進行二次改寫或自我修正（Self-Correction）」。  
GraphRAG (Microsoft 知識圖譜檢索):透過 LLM 將文檔中的實體（Entities）與關係（Relations）萃取出來建構知識圖譜，並透過社區發現演算法（Community Detection）進行分層摘要。  
Late Chunking (延遲切塊):顛覆傳統「先切塊、再各自 Embedding」的順序。改為先將整篇長文件送入支援長上下文的 Embedding 模型中進行編碼，保留全篇的交叉上下文（Cross-chunk context）特徵後，才在最後階段進行切塊。  
6.Eval RAG  
Retrieval Evaluation (檢索端評估指標)  
Recall@k (召回率):在前 k 個檢索結果中，是否包含包含正確答案（Ground Truth）的文件。  
MRR (Mean Reciprocal Rank - 平均倒數排名):正確文件在檢索結果中排在第幾位。如果第一名就是正確答案，分數為 1；若排在第二名，分數為 0.5。排名越前面，MRR 越高。  
nDCG (Normalized Discounted Cumulative Gain - 正規化折讓累積增益):不僅考慮「有沒有找到」，還考慮「相關性排序的優劣」（排在最前面的文檔權重最高），是評估檢索品質非常嚴謹的綜合指標。  
Generation Evaluation (生成端評估指標)  
Faithfulness (忠實度 / 幻覺指標):生成的答案中，有多少成分是直接基於檢索到的 Context？如果模型自己瞎掰、加入外部沒有的事實，Faithfulness 分數就會暴跌。  
Answer Relevance (答案相關性):生成的回答是否有切中使用者一開始的提問（Query）？有沒有答非所問或離題。  
End-to-End Evaluation (端到端綜合評估)  
Human Eval (人工評估):由領域專家對最終生成的完整回答進行主觀打分（流暢度、準確度、實用性）。雖然是評估的黃金標準（Gold Standard），但成本極高、無法自動化。  
LLM-as-a-Judge (大模型當裁判):使用能力最強的旗艦模型（如 GPT-4 或 Claude 3.5 Sonnet）擔任裁判，依照預設的 Prompt rubric 對系統輸出的答案進行自動化打分與理由解析，兼具高效率與與人類高度一致的評分表現。  
7.名詞表  
Dense Retrieval (稠密向量檢索):將 Query 與 Doc 透過 Embedding 模型轉為向量並計算餘弦相似度。  
BM25:對精確匹配 (Exact Match)、罕見詞、人名、產品編號極其強大，是現代檢索不可或缺的基石。  
Cross-Encoder Rerank (交叉編碼重排):精準度遠高於雙編碼器（Bi-encoder），但計算成本高，通常只用在初篩後的 Top-50 候選精修。  
HyDE (Hypothetical Document Embedding):先用 LLM 針對 Query 生成一份「假答案」，再將假答案去進行向量檢索。  
Recall@k (召回率):衡量檢索端表現的主指標 — 正確的標準答案文檔出現在前 k 個檢索結果中的比例。  
Faithfulness (忠實度):生成端（Generation）的核心指標 — 確保模型產出的回答是否有明確的檢索來源（Source）支撐，用以防範幻覺。   
Reflexion (自我反思):讓 LLM 自我檢視生成結果、尋找錯誤並進行修正的機制，對複雜推理與高難度任務特別有效。  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)  
從最初簡單的「切塊 → 向量化 → 塞進 Prompt」的 Naive RAG，演進到現代融合了混合檢索（BM25 + Dense）、RRF 融合排序、Cross-Encoder 重排（Reranker）、HyDE 假想嵌入、甚至結合知識圖譜的 GraphRAG 與 Agentic RAG。    
提示詞工程從早期的「隨性描述」走向科學化的結構設計（如 CoT 思維鏈、JSON 格式強制約束、Self-consistency 等）。  
純靠向量檢索（Dense Retrieval）在面對精確名詞或編號時容易翻車，必須搭配傳統的 BM25 進行混合檢索（Hybrid Search），並透過高效的 Cross-Encoder Reranker 進行精準過濾，才是當前工業界最穩健的黃金標準。  
(b)  
傳統向量檢索的盲點與語意落差:純粹的 Dense Retrieval 依賴語意相似度，但在面對企業內部的專有名詞、產品編號、零件代號或特定人名時表現極差，常發生「有資料卻搜不到」的狀況。  
傳統 Chunking 的語意破碎化:固定長度的切塊常常把一句話或核心邏輯硬生生從中間切斷，導致 Embedding 抓不到完整上下文，直接汙染後續的向量資料庫品質。  
宏觀全域問題的失靈 (Local vs. Global Search):傳統向量 RAG 擅長尋找「局部具體事實」，但面對「總結過去一年所有研發專案的整體趨勢」這類宏觀、跨文件的全域性提問時，傳統 Chunk 檢索會完全失效，這也是逼迫技術轉向 GraphRAG 的核心原因。  





