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



## 學習記錄：(a) 你看到了什麼，查到了什麼，瞭解到了什麼，你又關心什麼具體現象 (b) 為什麼現有方法的問題是什麼？
(a)單看文字說明會不太清楚，搭配李宏毅老師的影片可以了解得更清楚:https://www.youtube.com/watch?v=hYdO9CscNes  
(b)
