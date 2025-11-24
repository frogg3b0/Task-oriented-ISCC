# Interference-Aware Graph Based Resource Sharing for Device-to-Device Communication Underlaying Cellular Networks


## Abstract


## 作者將所有的 links 建立成一張 interference graph
* 「interference-aware」代表：每一條 edge（連結兩個使用者）不只是「有干擾」或「沒有干擾」，而是包含了實際可計算的干擾強度
* 基於這張干擾圖，作者提出一個演算法，用來做 RB 分配




## 演算法執行位置：基地台（BS）。


* 目標：取得接近最佳（near optimal）的分配結果
* 特點：複雜度很低，不像 brute force 要試遍所有組合（NP-hard）
* 結果：演算法的複雜度遠低於暴力搜尋，但網路總速率（network sum rate）非常接近最佳解


## Abstract 與你的 research 問題高度相關的地方


* 作者有一個核心元素：網路中的每條 link 都會對其他 link 造成干擾 → 把這些干擾關係變成一張 weighted interference graph
* 演算法中重要的概念：不讓所有裝置都在同一組（cluster），必須要考慮每條邊的干擾強度
* 與你的問題直接一致：你有 U 個使用者、P 個任務，使用者貢獻值越大越希望在較早的 time slot 提供資料，但含有干擾（interference/SINR-like）限制，因此不可能全部放在第 1 組


---


## Introduction


為了解決這個干擾問題，過去已有相當多研究（參考文獻 [4]–[7]）  


[4] C.-H. Yu, O. Tirkkonen, K. Doppler, and C. Riveiro, “Power optimization of device-to-device communication underlaying cellular communication,” in Proc. IEEE ICC 2009, Jun. 2009.  
[5] G. Fodor and N. Reider, “A distributed power control scheme for cellular network assisted D2D communications,” in Proc. IEEE GLOBECOM 2011, Dec. 2011.  
[6] P. J¨anis, V. Koivunen, C. Ribeiro, J. Korhonen, K. Doppler, and K. Hugl, “Interference-aware resource allocation for device-to-device radio underlaying cellular networks,” in Proc. IEEE Vehicular Technology Conference (VTC 2009-Spring), Apr. 2009.  
[7] C.-H. Yu, K. Doppler, C. B. Ribeiro, and O. Tirkkonen, “Resource shaing optimization for device-to-device communication underlaying cellular networks,” IEEE Transactions on Wireless Communications, vol. 10, no. 8, pp. 2752–2763, Aug. 2011.


---


### 當共用同一個 RB 時，干擾關係變得相當複雜
* 要找到「最佳」的 RB 分配（最佳 = maximize system sum-rate）需要考慮：
    * 每條 link 的 SNR
    * 每對 link 之間的干擾
    * 組合數量巨大（combinatorial explosion）
    * 因此，求最佳解的計算複雜度極高

* 因為 optimal resource sharing 計算量太大，因此：
    * BS 的 scheduling delay（排程延遲）會提高
    * BS 的壓力變重（需要做大量計算）
    * 這對真實系統非常不利，因為 BS 要很快決定誰跟誰共用 RB

* 真實系統不能只追求「最佳性能」，也要考慮複雜度
    * 因此需要一種能取得性能與複雜度的折衷的資源共享方法


* Graph theory（圖論）可以有效描述：
   * 互相影響的關係
   * 互相干擾的關係
   * 各種節點間的互動
   * 過去很多研究（[8], [9]）已經用 graph-based 方法做 resource management（例如 channel allocation）
   * 換句話說: 干擾 → 天然適合用圖來表示

[8] Y.-J. Choi, J. Kim, and S. Bahk, “QoS-aware selective feedback and optimal channel allocation in multiple shared channel environments,” IEEE Transactions on Wireless Communications, vol. 5, no. 11, pp. 3278– 3286, Nov. 2006.  
[9] R. Y. Chang, Z. Tao, J. Zhang, and C.-C. J. Kuo, “Multicell OFDMA downlink resource allocation using a graphic framework,” IEEE Transactions on Vehicular Technology, vol. 58, no. 7, pp. 3494–3507, Sep. 2009  

---

### 作者提出 3 種新的 per-link attribute，用來讓演算法更有能力「控制決策」：

* link attribute：判斷這個 link 是 cellular 還是 D2D（這會影響干擾計算方法）。
* resource attribute：包含兩個項目：L(Vᵢ)：依據接收 SNR 排序的 RB list、δ(Vᵢ)：目前最想要的 RB（list 的第一個）
* cluster attribute（τ(Vᵢ））：這條 link 目前已經被分配到哪個 RB cluster
* 這些新定義的屬性有助於控制基於幹擾感知圖的資源共享演算法的迭代過程
