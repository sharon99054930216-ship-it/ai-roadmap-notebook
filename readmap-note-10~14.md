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










