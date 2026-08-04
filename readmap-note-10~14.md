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









