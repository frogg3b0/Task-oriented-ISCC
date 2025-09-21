# 《Deep Residual Learning for Image Recognition》

## 我看這篇的目的是?
目前知道了如何使用 STFT 產生二維的時頻圖片，接下來需要使用影像辨識的技術，去分類這張圖片  
* 而目前針對該時頻圖辨識方式有三種
    1. SVM
    2. CNN
    3. **ResNet**

因此閱讀此篇的目的，正是希望學會如何使用 ResNet 去分類影像

## 問題解決了嗎?怎麼解決的?

***
## Abstract

### 問題背景: 為何使用 ResNet
隨著網路加深，訓練難度會增加，容易遇到梯度消失或收斂慢的問題。
* 因此提出「殘差學習」框架，幫助更深層的網路更容易訓練

### 如何執行 ResNet
不再直接學習輸入到輸出的完整映射，而是讓網路學習「輸入 → 輸出之間的差異（residual）」。

### 使用 ResNet 的優勢
* 使用殘差結構後，網路更容易收斂，並且加深層數能提升準確率，而非退化
* 在 ImageNet 上訓練到 152 層，比 VGG 深 8 倍，但運算量更低
* 集成模型在 ImageNet 測試集錯誤率僅 3.57%，拿下 ILSVRC 2015 分類任務冠軍
* CIFAR-10 上也測試了超深網路，甚至達到 1000 層
* 在 COCO 物件偵測任務上，單靠網路深度與殘差結構，就帶來 28% 的相對提升

***
## 1. Introduction

### 本文提出「深度殘差學習框架 (Deep Residual Learning)」來解決退化問題
* 傳統方法: 輸入 x → 輸出 H(x) 的完整映射
* ResNet 方法：學習 殘差函數 F(x) = H(x) − x，也就是「輸出與輸入的差異」

### 如何實現 𝐹(𝑥)+𝑥
在網路裡加「捷徑連接 (shortcut connection)」
* Shortcut = 跨過幾層，直接把輸入加回輸出.
* 在 ResNet 中，這個捷徑只做 identity mapping，這個捷徑不需要額外的權重、計算量也幾乎不變

<img width="363" height="187" alt="image" src="https://github.com/user-attachments/assets/c60c9246-7fe3-4844-88c0-5dd951019ca0" />  


* shortcut connections 並不是全新的概念，早在多層感知機 (MLPs) 的研究裡就出現過相關實踐與理論
    * 早期方法：在輸入層與輸出層之間直接加一條線性捷徑，幫助訓練收斂
    * 後續方法：將中間層直接連接到輔助分類器 (auxiliary classifiers)，以對抗梯度消失或爆炸。


***

## 3. Deep Residual Learning
## 3.1. Residual Learning
Residual Learning 的關鍵 insight：  
- 與其直接學 H(x)，不如學 H(x)-x。這讓網路只需在 identity 基礎上做小修正，極大降低了深網的優化難度。

***

## 3.2. Identity Mapping by Shortcuts

### 殘差學習不是應用在整個網路一次，而是應用在「每幾層」形成的 block 裡 
* **整個 ResNet 就是由一堆 residual blocks 疊加而成**
* 定義一個 residual block y = F(x, {Wi}) + x
    * 𝑥：這個 block 的輸入
    * 𝑦：這個 block 的輸出
    * 𝐹(𝑥,{𝑊𝑖})：這幾層網路學習到的 residual function

* 假設 residual block 裡有 2 個全連接層：𝐹(𝑥) = $W_2$ 𝜎( $W_1$ 𝑥)
* 在「加法」之後，還會再套一個 ReLU：𝑦=𝜎(𝐹(𝑥)+𝑥) ，這讓網路保持非線性，避免輸出退化成純線性。

### ResNet 保持高效的優勢
ResNet 的 identity shortcut（直接把 𝑥 加到輸出）不增加任何參數，也幾乎不增加計算量（只有逐元素加法）
* 在公式 𝑦=𝐹(𝑥)+𝑥 中，必須保證輸入 𝑥與輸出 𝐹(𝑥) 維度相同，才能逐元素相加。
* 當維度不同（例如卷積改變了通道數），就需要透過線性投影： $W_s$ 𝑥

### 小結 
ResNet 的 shortcut 幾乎不增加成本，維度不符時才用投影 ，通常 residual function 用 2–3 層卷積來保持表現力；整體結構高效且靈活

