# 《Wireless Sensing With Deep Spectrogram Network and Primitive Based Autoregressive Hybrid Channel Model》

## 我看這篇的目的是?
不知道該如何設計碩論的 AI model  

## 問題解決了嗎?怎麼解決的?
Ans:  


***

## Abstract

### 目前的系統大多利用機器學習模型如 SVM和 CNN來對雷達訊號分類，進而識別動作類型。
1. 如果採用更深層的神經網路（比 CNN 更複雜的架構）是否能進一步提升識別準確率。
2. 訓練機器學習模型需要大量數據，但實際上從實驗中蒐集這些數據成本高、時間久。
    * 雖然可以透過模擬的無線通道模型來產生訓練資料集，但現有的通道模型主要是為了通訊（如 5G 傳輸）設計，而非為感測應用打造。
    * 為了解決上述問題，本文提出一種「深層頻譜圖網路」（Deep Spectrogram Network, DSN），使用殘差映射（residual mapping）技術（如 ResNet 結構）來**提升動作辨識的準確率**
3. 本文設計一種「基於 primitive 的自迴歸混合通道模型」（PBAH），**這個模型可以有效模擬虛擬環境下的無線感測數據，用來快速生成大量訓練和測試資料**。
    * PBAH 通道模型所產生的資料與真實實驗非常相似
    * DSN 模型在辨識錯誤率上明顯優於傳統 CNN。
  
***

## I. Introduction

無線感測是一種潛力極高的技術，能夠用來解決智慧交通系統中的安全問題。例如，這項技術可以在地下停車場中偵測奔跑中的小孩。
* 無線感測可分為「模型驅動型」與「資料驅動型」兩類。
    * 「模型驅動型感測」的典型應用是定位。其原理是，根據 RF 信號的量測結果（例如到達時間差），再透過幾何關係推算裝置位置。
    * 「資料驅動型感測」通常更複雜。在人體動作辨識（HMR）中，無法單純從 RF 測量值直接判斷動作類型。因此需要使用機器學習技術，透過與歷史樣本比對，提取有用資訊


* 來自人體運動的**micro-Doppler features**會從接收到 RF 訊號中提取出來
    * 接著，這些特徵會被轉換為**時頻圖（spectrogram**的形式（即影像），並以機器學習方法進行影像分類。
### SVM/CNN/ResNet 如何做人體動作辨識
* 在文獻 [5](https://ieeexplore.ieee.org/abstract/document/4801689) 中，使用 STFT 產生時頻圖，並應用 SVM 進行分類，分類錯誤率低於 10%
* 在文獻 [6](https://ieeexplore.ieee.org/abstract/document/7314905) 中，將 SVM 換成 CNN 後，能夠達到更低的分類錯誤率
* 在文獻 [7](https://openaccess.thecvf.com/content_cvpr_2016/html/He_Deep_Residual_Learning_CVPR_2016_paper.html)，目前尚不清楚若進一步使用更深層的神經網路（如 ResNet）是否能進一步提升系統效能

### 目前研究的問題
訓練一個機器學習模型需要「大量的資料集」；從實驗中收集數據會「受到環境影響」，還需要「人工標註資料」每筆資料都需標記這是什麼動作→ 例如「走路」、「站立」、「快跑」  
* 為了降低資料收集成本，一種方法是透過「無線感測通道模型來模擬不同場景」並產生大量有標註的資料
* 一個理想的「感測通道模型」應該同時具備空間一致性、時間一致性、微多普勒一致性，並保留感測的不確定性
    * 空間一致性：相近位置的感測結果應該相似
    * 時間一致性：連續時間點的結果應平滑過渡
    * 微多普勒一致性：能產生如人體擺動般的細緻頻譜變化
    * 感測不確定性：保留隨機性，反映真實世界的雜訊與變異

### 本文貢獻
現有的通道模型都無法同時滿足上述所有特性，總結來說，本文針對兩大問題展開研究。為了解決這兩個問題，本文提出：
1. DSN 模型：進行高品質的人體動作辨識
     * 使用 SVD 去雜訊
     * 使用 STFT 轉為時頻圖
     * 使用 ResNet 架構分類
     * 其分類錯誤率低於 CNN
2. PBAH 模型：在虛擬環境中高效率生成 HMR 訓練與測試資料
     * 產生「逼真」的感測資料
     * 支援 HMR 所需的全部特性（空間/時間/微多普勒/不確定性）
     * 通道變化速率透過 KL 散度最小化來最佳化
     * 模擬資料與實驗資料高度匹配

***
## II. Problem Statement

### 系統定義
本文考慮一個設置在會議室內的無線感測系統，場景中包含一位目標人物，以及一些靜止的物體。
系統目的是透過雷達訊號的發射與處理，識別人體動作（例如站立、行走、奔跑）。

#### 雷達訊號建模
* 雷達每 T 毫秒發送一次**FMCW訊號**，FMCW 的掃描時間為 T₀，總共發送 C 次 FMCW，因此總感測時間為 T × C
* 令 sᵢ(t) 為第 i 次傳送的 FMCW 波形，則第 i 次對應的接收信號為：rᵢ(t, m) = hᵢ(t, m) * sᵢ(t) + nᵢ(t)
    * hᵢ(t, m) 是當「目標執行第 m 種動作」時的通道響應
    * nᵢ(t) 是加性白高斯雜訊。
* 雷達收集所有信號 {rᵢ(t, m)} 後，會傳送至處理器（如圖 1 右側）進行人體動作辨識。

### 對於上述系統，有兩個核心問題：
1. 如何設計一個機器學習模型來處理這些接收信號 {rᵢ(t, m)}
2. 如何生成一個虛擬資料集來訓練模型，並保證模型效果能與使用實驗資料時一致

***

## III. PROPOSED DEEP SPECTROGRAM NETWORK
* DSN（Deep Spectrogram Network）
    * 輸入是 {ri(t, m)}，也就是來自雷達的 C 條時間訊號
    * 輸出則是預測的人體動作類別 m̂（如站立、行走等）。

#### Step 1. Data Sampling
* 每條接收訊號 ri(t, m) 都會被離散化為一個**長度為 L 的向量 xi(m)**
    * 資料點數 L = 時間長度T₀ × 取樣頻率 (OFDM : symbol時長 *T* × 系統頻寬 *N/T* = *N* 個資料點)
    * 將 C 條 FMCW 訊號接成一個 L×C 的複數矩陣 X(m) (OFDM : 將 *P* 個 OFDM symbol 接成一個 *N × P* 的矩陣)



#### Step 2. Data Cleaning
因為接收訊號的 矩陣 X(m) 包含有用的訊號（人體反射）與干擾訊號（牆壁、桌椅反射，為了提取有用訊號，我們先進行
* 解調（dechirp）--> OFDM: Remove CP + *N* point DFT，還原頻域資料
* 取共軛（conjugate）處理得到 X̃(m) --> 相同
* 接著對 X̃(m) 進行 SVD 分解 --> 相同
* 將前 r−1 個主要成分（可能是干擾）移除後，剩下的訊號 Y(m) 被視為「去雜訊後」的資訊。(留下微弱但重要的動態訊號)

#### Step 3. Data Transformation (Short-time Fourier transform)
使用短時傅立葉轉換（STFT），透過**長度為 W 的滑動窗(Kaiser window**)來產生同時具有時間與頻率特徵的表示。  
* Kaiser window: 窗長 128 點，相當於 0.128 秒，滑動時間間隔為 1 毫秒
* Z(m) = STFT (y(m))
<img width="421" height="247" alt="image" src="https://github.com/user-attachments/assets/2d59b616-7ef3-4e57-ba40-b7a9e3a2660a" />

#### Step 4. Data Classification
使用 ResNet 架構作為分類骨幹，包含 5 個殘差模組與一個 softmax 輸出層。  
* 每個殘差模組包含 6 層：標準化、激活函數、卷積，共兩輪
* 每個模組會學習輸入與輸出之間的差值
* 最終輸出為預測的人體動作類別 m̂


***
## IV. PROPOSED BENCHMARK METRICS
### 訓練資料如何取得?
訓練機器學習模型（如 CNN/ResNet）需要大量資料，但在真實無線感測系統中，實驗蒐集資料的成本與耗時都非常高。  
* 透過模擬「無線通道模型」來合成資料集，是一個可行的解法。
* 但是當我們用模擬器來合成資料時，「什麼樣的模擬器才算夠好？」

### 通訊和雷達用途的訓練資料大不同
傳統 channel metrics（如衰減、延遲、AoA）不適合用來評估感測任務的模擬器效果。原因在於這些指標只反映通道「靜態特性」，但無法表現「動作變化」
* 為了找出針對「無線感測」更適合的評估指標，作者設計並實作了一個真實實驗：利用 USRP RIO 平台，在會議室中收集 radar 資料
* Fig.2 展示了系統架構
* Fig.3 展示 radar 回波經過 STFT 轉換後的影像（即 spectrogram）
    * micro-Doppler consistency：指模擬器生成的資料中，必須真實反映人體如四肢擺動等「非剛體」動作，在 spectrogram 上出現週期性曲線（如走路時手臂擺動）。
    * sensing uncertainty：雷達觀測常伴隨不確定性（例如衣服飄動、肢體變形等），模擬器也應能在一定程度上保留這種「不可預測性」。

<img width="443" height="350" alt="image" src="https://github.com/user-attachments/assets/5b5098c0-65e5-4a93-ab09-1388fed04cf6" />  

### 一個好的感測模擬器，應該具備哪些特性

#### Spatial Consistency
當兩個位置很靠近時，它們的無線通道應該相似  
* 「大尺度空間一致性」：兩個空間相近的位置上，功率衰減、延遲擴展和角度擴展都應該一致。這些是「統計性特徵」
* 「小尺度空間一致性」表示：在兩個接近的位置上，每條 multipath 的延遲與角度都應一致。


#### Time Consistency
當發射器、接收器或被觀測對象，在時間上只移動一點點時，無線通道應該是連續平滑變化  
* Deterministic time consistency： 當系統元件（TX、RX 或目標）緩慢移動時，通道的變化也應該是平滑且連續
* Stochastic time consistency: 則反映真實世界中，即使裝置不動，環境也不是完全靜止。


#### Micro-Doppler Consistency
人體中非剛體部分的動作（像手臂擺動、腿部彎曲等）在雷達信號上造成的細微頻移，所以一個好的感測模擬器，必須能夠模擬出「這些非剛體造成的細微頻率變化」   
* 在雷達的 time-frequency spectrogram 中，每一種動作（如走路、跑步）都會產生獨特的「頻率-時間曲線」
* 這些曲線就是「動作指紋（fingerprints）」，而且這些特徵非常依賴正確的 micro-Doppler 模擬

#### Sensing Uncertainty
spectrogram 裡還許多「亮點」、「模糊點」——這些是隨機性的 micro-Doppler shift。
* 這種「不穩定、不可預測的變化」稱為 sensing uncertainty，來源可能是：衣物飄動、突發動作、環境物體干擾等。    
* 它是 radar sensing 任務中必要的挑戰元素，用來測試演算法的健壯性（robustness）。

#### Comparison with Existing Channel Models

目前主流的無線通道模型可分為三大類：
1. 統計模型（Statistical): 只描述整體通道的統計性質，如衰減平均值、delay spread 分布等。
2. 決定性模型（Deterministic): 根據實際幾何位置、材質等精確模擬通道反應（如 ray tracing）。
3. 準決定性模型（Quasi-deterministic）:以幾何決定為主，但仍加上統計參數進行調整，常用於產業標準（如 3GPP/ITU）。

<img width="974" height="327" alt="image" src="https://github.com/user-attachments/assets/1d4a5460-41a5-46d0-9e91-516e70a71b3e" />

***


