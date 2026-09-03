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
