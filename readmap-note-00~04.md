# part00 Setup & Tooling
## 名詞理解
1. Dev Loop（開發循環）指的是從「修改一行 Code」到「看到結果（報錯或成功）」所花費的時間。
2. ssh：   
用途：讓妳在宿舍或圖書館，就能連線操控實驗室那台裝了 RTX 4090 的電腦。  
功能：就像是遠端桌面，但它是文字版的，速度極快且穩定。
3. tmux：  
用途：AI 訓練一跑就是好幾個小時甚至幾天。如果妳的網路斷掉，程式會自動終止。  
功能：它能讓程式在伺服器後台「開一個獨立空間」跑。就算妳關掉筆電睡覺，隔天再連回去，訓練依然在繼續。
4. rsync：  
用途：用來把妳電腦裡的資料集（幾萬張魚類圖片）傳送到伺服器上。  
功能：它比一般的複製貼上更聰明。如果傳到一半斷掉，它下次會從斷掉的地方繼續傳（續傳），且只傳送有修改過的部分。
5. nvidia-smi：   
用途：這是最重要的指令。它會告訴妳：顯卡現在熱不熱？顯存（VRAM）還有多少空間？是誰在占用顯卡？  
功能：看看妳的模型有沒有成功吃到 GPU。如果妳看到顯存滿了（Out of Memory），那就是這行指令會告訴妳原因
6. htop：  
用途：顯示 CPU 使用率、記憶體（RAM）剩下多少。  
功能：就像 Windows 的「工作管理員」。當妳覺得伺服器卡卡的時候，打開它就能看到是不是哪個程式把資源吃光了。
7.  prompt(提問):學習如何精確描述問題。(出問題的程式碼片段，完整的錯誤訊息，我做了什麼操作？我想達成什麼目標？)
8.  diff(對比):用 git diff 或是編輯器的對比功能，看它到底改了哪裡。
9.  commit(存檔):只要一段 code 測試通過，立刻存檔。

## 使用Cursor製作README.md的檔案並在GitHub上開啟  

Cursor優點功能:    
1. 自動寫入檔案存到電腦:不需要重複的複製貼上行為  
2. 自動排版功能:圈選需要排版的文字後，按下右鍵，點選Refactor...，點選Add to chat (或是Ctrl+L)，輸入排版需求指令  
3. 詢問檔案內容或重點:按 Ctrl + L，輸入提問(點文字框中的Work for XXs可察看回覆內容)  
<img width="500" alt="image" src="https://github.com/user-attachments/assets/381bf65f-9d24-4beb-94dc-046ad55b830e" />

Cursor使用步驟:   
1.建立新專案資料夾:按下左上角的 File -> Open Folder，在桌面建立一個新資料夾叫 cursor-practice，然後點選「選擇資料夾」。  
2.輸入指令:在輸入框裡，直接打入話(ex:請幫我建立一個 Python 檔案叫 bmi.py，裡面要有一個計算 BMI 的函式，並能讓使用者輸入身高體重後印出結果。另外，再幫我建立一個 README.md 說明如何使用)，按下 Enter。  
3.存檔(能直接在檔案總管中找到):在檔案中可以找到已寫好的檔案  



## 練習下載 Git Bach 並連結 GitHub (git-test)  
Git使用功能優點:紀錄與追蹤 時光倒流 差異比對 雲端同步  

<img width="300" alt="image" src="https://github.com/user-attachments/assets/e05773ba-3852-4bd2-b1b0-43082b5bee4d" />  


Git使用順序是：  
1.改 Code:在電腦寫完一段程式，確定能跑。  

2.存檔:在 Git Bash 輸入  
git add .  
git commit -m "這次改了什麼（例如：調高 Learning Rate）"(這是在妳電腦裡記筆記)  

3.上傳:在 Git Bash 輸入 git push。 (這是把筆記同步到雲端 GitHub)  

4.查看修改內容:在 Git Bash 輸入 git log -p。  
綠色的字 (+)：代表新增的內容。  
紅色的字 (-)：代表刪掉的內容。 

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)這次多認識了很多AI的工具，也嘗試練習將它們下載來做使用，瞭解到使用 AI-native workflow 能大幅縮短 Dev Loop。  
(b)傳統的在chagpt或gemini上問問題後，都要將結果再貼到其他軟體上作執行，使用 AI-native edito 就可以更有效的處理問題。  


# part01 Machine Learning Basics
## 名詞理解
1.Loss function 損失函數:函數值越低表示模型誤差越小  
2.gradient descent 梯度下降:尋找最佳化的過程  
3.regularization 正規化:限制模型參數的大小，防止模型過度擬合  
4.generalization 泛化:模型在處理未見過的「新數據」時，依然能準確預測與應對的能力  
5.Bias-Variance trade-off 偏差變異權衡:  
Bias 偏差:模型對於數據規律的預設偏誤，模型太簡單，Underfitting (欠擬合:訓練集誤差高，測試集誤差也高)  
Variance 變異:模型在面對不同訓練數據時，預測結果的波動程度，模型太複雜，Overfitting (過擬合:訓練集誤差極低，測試集誤差高)  
6.Training Set:讓模型透過 Gradient Descent 去逼近 f(x)= y  
7.Validation Set:用來調整超參數（例如學習率、正規化係數λ、CNN 的層數）  
8.Test Set:一次性最終測試，如果因為Test Set結果不好，你回頭調整模型結構，再拿同一個Test Set去跑第二次，這個Test Set就變成了 Validation Set，已經失去了「未見過」的客觀性      
9.Tabular Data  
SVM 支援向量機:目標是找出一個能將不同類別資料完美分開的決策邊界  
Random Forest:透過建立多棵「決策樹」組合而成，並利用多數決（分類）或平均值（迴歸）來決定最終預測結果，能有效解決單一決策樹容易過度擬合的缺點  
XGBoost:梯度提升決策樹的開源機器學習演算法，極高的執行效率、準確性及防過擬合能力  
10.Regularization  
weight decay:透過在損失函數中增加懲罰項，強迫模型的權重保持較小的值，進而提升模型對未知數據的泛化能力(當Optimization（如梯度下降法）在更新權重時，看微分（梯度），懲罰項是 w^2一次次微分後近趨於0)  
Lasso:將不重要的特徵權重直接變成 0，達到「特徵選擇」的效果(懲罰項是 |w|，微分後為0)  
Dropout:隨機讓一部分的神經元停止運作（歸零）  
Early Stopping:在驗證集 Loss 剛開始回升的臨界點，立刻強制停止訓練  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？  
(a)這個part主要在說明機器學習，看到了一些新手可能會常犯的錯誤，像是把test sat拿去跑第二次的驗證;還有Underfitting跟 Overfitting 的解決方法等等。這幾周讀書會也有練習使用colab跑模型並用kaggle做模型最後的好壞判斷，使我在讀這個part時有更好的理解!       
(b)目前還沒發現現有方法的問題  

# part02 Deep Learning Basics
## 名詞理解
1.MLP:線性層負責空間變換，非線性負責扭曲摺疊，兩者透過「加深與加寬」，讓網路能模擬世界上的任何複雜規律。  
線性模型(單層感知機)，結構如下  
<img width="200" alt="image" src="https://github.com/user-attachments/assets/ac4e7307-4034-4aa4-a492-123961dde8e4" />  
MLP(多層感知機):想得到非線性的模型(加入sigmoid，ReLU)，因為線性的模型能力有限。結構如下(包含了隱藏層跟激活函數)   
<img width="250" alt="image" src="https://github.com/user-attachments/assets/fddeeb64-4689-4e75-b457-54e78591d186" />  

2.activation(激活函數)  
sigmoid:當輸入 x 太大或太小時，Sigmoid 的曲線會變得極度平坦（飽和區），此時導數趨近於 0。即便在最完美的 x=0 處，它的導數最大也只有 0.25。當網路疊到 10 層時，反向傳播要把各層導數乘起來：0.25^{10} → 0.00000095。梯度傳到前幾層早就消失了，這導致深層網路根本練不動。  
ReLU:只要輸入 x > 0，ReLU 的導數永遠是固定的1。不論網路疊到 100 層還是 1000 層，1x1x1...x1 = 1。梯度可以毫無損耗地一路傳回最前層，結決了梯度消失的問題。  
GeLU:在負數區域（大約在 0 ~ -1 之間）有一段平滑的「小凹槽」。即使輸入是微小的負數，它依然保有微小的梯度（斜率不為 0）這給了壞死神經元起死回生的機會。  

3.Backprop:讓幾十億個參數只需去一趟、回一趟就全數更新完畢。  
Forward(向前)：儲存激活值(Activation）  
Backprop(反向傳播):上游梯度 × 本地 Jacobian，當 Loss 算出來後，梯度從後往前傳  

4.Initialization(初始值):  
Logits 的分佈應該要對稱、均勻，且數值大小適中（通常在 0 附近輕微震盪）。這代表此時模型對所有類別都沒有偏見。  
假設你在做一個 n 分類的任務。在完全沒有訓練的情況下，一個合理的模型應該對所有類別一視同仁，猜對任何一類的機率都應該是均勻的 1/n_classes。  

5.Normalization(正規化)  
BatchNorm:(跨樣本，CNN）： 它是對同一個特徵、跨所有樣本（Batch）去算平均值和方差(缺點:被 Batch Size 綁架、無法應對變長句子(處理 NLP 變長句子時，後方的補零（Padding）會嚴重污染整體特徵分佈。)、推理與訓練的不一致性)。  
LayerNorm:（同樣本內，Transformer）： 它是對單一一個樣本、跨它自己所有的特徵去算平均值和方差。  
RMSNorm:現代大模型（如 LLaMA、Gemma、Mistral）為了追求極致速度而進化出的「減重版」元件。  
Pre-Norm vs Post-Norm:所有現代 LLM 用 Pre-Norm   
Post-Norm 缺點：深層網路的梯度爆炸/消失  
在 Post-Norm 中，每一層的輸出都會被強行做一次 LayerNorm，這會導致越往深層走，主幹道上的數值和梯度分佈會被層層扭曲。  
<img width="600" alt="image" src="https://github.com/user-attachments/assets/9778872f-55c0-4408-8766-c85e8d41021b" />  

6.Optimizer(最佳化)    
SGD(動量法):隨機梯度下降，哪裡陡就往哪裡走。如果遇到「兩側極陡、中間平緩」的狹長峽谷地形（震盪方向），SGD 就會在兩側牆壁不斷劇烈震盪，卡在原地走不出去。  
Adam(自適應學習率):結合了 Momentum（一階動量，管方向）與 RMSProp（二階動量，管步長），每個不同的參數都給不同的learning rate。  
AdamW(把 Weight Decay 算清楚):Adam 依照梯度算出更新量後，在最後一步直接減去權重  


## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？    
(a)看到不確定功能的名詞時，上網找影片學習，看讀書會的影片也蠻有幫助的。也發現在Roadmap文章中ai擅長使用簡潔的形容詞來描述內容，所以有些內容會看不太清楚它想表達的意思是什麼，我的做法是去了解此主題的大方向以及該主題內容中的各個名詞功能特色等等。  
(b)  
1.Activation的困境：Sigmoid → 存在飽和區，在深層網路的鏈鎖律下會引發梯度消失，導致底層權重更新停滯。  
2.Optimizer的困境：標準SGD → 缺乏動量與自適應步長，在病態曲率(狹長峽谷)地形中會產生嚴重的高頻震盪，極難收斂。  
3.Normalization的困境：BatchNorm/Post-Norm → BN 依賴微批次大小且易受 Padding 污染；Post-Norm 則因破壞了殘差的線性梯度，導致深層模型在初始化初期極易數值崩潰。  


# part03 Convolutional Neural Networks
## 名詞理解
1.CNN:把參數量砍掉好幾個數量級  
區域連接:一個神經元不再看整張圖，而是透過一個固定大小的滑動視窗只看局部特徵。  
權重共享:這個滑動視窗在整張圖片從左到右、從上到下移動時，使用的是同一組權重。  
2.CNN組成  
Convolution Layer(卷積層):特徵提取  
Pooling Layer(池化層):濾掉不重要的特徵，減少模型參數，防止過擬合  
Fully Connected Layer(全連接層):統計並歸類得到的特徵後做輸出  
3.Pooling   
MaxPool:取最大值，AveragePool:取平均  
<img width="500" alt="image" src="https://github.com/user-attachments/assets/f8c9350d-02dc-44b5-90b1-15811cfe6ffe" />  
<img width="700" alt="image" src="https://github.com/user-attachments/assets/15d5e122-12fc-4a14-841d-a95896246cb8" />  
GlobalAvgPool:可以是任意解析度輸入    
ex:如果最後特徵圖是 [1, 512, 7, 7]，GAP 會對這 512 張 7x7 的圖片各自計算平均值。最後輸出就變成 [1, 512, 1, 1]。你直接得到了 512 個數值，這 512 個數值就代表這 512 種特徵在整張圖上的平均強度。  
4.ResNet解決的問題  
深層網路退化問題:深層網路的 Training Error 比淺層網路還要高。  
殘差結構:直接在硬體電路上把 x 加進去，H(x) = F(x) + x →  F(x) = H(x) - x 這就是所謂的殘差。  
5.CNN vs ViT  
CNN:局部性和平移不變性 → 適合小資料  
ViT:把圖片打碎成一堆 Patch，然後強行去計算「任意兩個方塊之間的關係」→ 適合大資料  

## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？      
(a)看了關於CNN基礎與概念介紹的影片:https://www.youtube.com/watch?v=TEEuOuN3f1I&t=1517s  
在醫療影像中，CNN 依然是首選。在小資料下不易過擬合，在資源受限的邊緣裝置上，能實現低延遲且穩定的即時推論。  
(b)ViT的困境:醫療影像(罕見疾病)中通常只有幾百到幾千張。現有大模型直接拿來用，在小資料集上極易產生嚴重的過擬合。    

# part04 RNN, LSTM, & Seq2Seq
## 名詞理解  
1.序列不能用 MLP / CNN 原因:MLP需要固定輸入長度，CNN視窗有限、獨嘗內容時會忘記前面的內容  
2.RNN:  
使用了 h_t（Hidden State / 隱藏狀態），累積過去訊息。時間軸上共享權重，支援任意長度  
ex:給一句話 “The cat sat on the mat”，要做翻譯  
當模型讀到 The，大腦記下基本資訊 h_1。  
當讀到 cat，大腦結合剛才的記憶 h_1 與新詞 cat，更新記憶成 h_2（此時大腦知道主詞是單數貓）。  
一路讀到 mat 時，目前的 h_6 裡已經累積了「有一隻貓坐在某個東西上」的完整過去訊息。  
3.

