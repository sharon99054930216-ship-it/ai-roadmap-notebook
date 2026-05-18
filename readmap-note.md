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
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/381bf65f-9d24-4beb-94dc-046ad55b830e" />

Cursor使用步驟:   
1.建立新專案資料夾:按下左上角的 File -> Open Folder，在桌面建立一個新資料夾叫 cursor-practice，然後點選「選擇資料夾」。  
2.輸入指令:在輸入框裡，直接打入話(ex:請幫我建立一個 Python 檔案叫 bmi.py，裡面要有一個計算 BMI 的函式，並能讓使用者輸入身高體重後印出結果。另外，再幫我建立一個 README.md 說明如何使用)，按下 Enter。  
3.存檔(能直接在檔案總管中找到):在檔案中可以找到已寫好的檔案  



## 練習下載 Git Bach 並連結 GitHub (git-test)  
Git使用功能優點:紀錄與追蹤 時光倒流 差異比對 雲端同步  

<img width="615" height="403" alt="image" src="https://github.com/user-attachments/assets/e05773ba-3852-4bd2-b1b0-43082b5bee4d" />  


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
