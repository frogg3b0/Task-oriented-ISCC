# `backscatterPedestrian`語法背後的邏輯

### `backscatterPedestrian`建立一個物件，模擬行人反射的訊號。
- 此模型模擬 16 個身體部位的運動，以模擬自然運動。
- 該模型還模擬每個身體部分的雷達反射率。
- 從這個模型中，您可以獲得 『16 個身體部位』的位置和速度，以及身體移動時的總反向散射輻射。

<img width="560" height="337" alt="image" src="https://github.com/user-attachments/assets/b8a20987-c3d4-441f-85eb-f96dc07d971b" />

### 如何使用
- 建立行人後，您可以呼叫 `move` 來移動行人
- 若要取得反射訊號，使用 `reflect`
