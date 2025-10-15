```matlab
function [true_positives, false_alarms, miss_det] = run_radar_frame(iter, ...
    W, Tx_sym, Nu, Tx_anten_spac, Rx_anten_spac, Wave_Length, ...
    Ntx, Nrx, Nsub, Nsym, Sub_Spac, Tto, Pn_rad, c, ...
    Fc, BW, Dint, Aint, alpha, ...
    Tar_app, Tar_A, Tar_D, Tar_V, ...
    CFAR_tr, CFAR_gr, CFAR_td, CFAR_gd, CFAR_Ntc, CFAR_Pfa, ...
    true_positives, false_alarms, miss_det)
```

---
## Radar channel

```matlab
%% Radar channel=======================================================
    [Atar, Dtar, Vtar] = get_target_update(iter, Tar_app, Tar_A, Tar_D, Tar_V);      % 更新target參數

    TX_SV = get_steering_vector(Tx_anten_spac, Wave_Length, Ntx, Atar);     % 計算TX steering vector
    RX_SV = get_steering_vector(Rx_anten_spac, Wave_Length, Nrx, Atar);     % 計算RX steering vector

```

* `[Atar, Dtar, Vtar] = get_target_update(iter, Tar_app, Tar_A, Tar_D, Tar_V);`: 這行呼叫一個函式 get_target_update()，用來「更新目標在當前時間的真實狀態」
    * 輸入：
    * `iter`: 第幾個時間迭代（或 frame index）
    * `Tar_app`: 目標出現週期（例如每 10 frame 出現一次）
    * `Tar_A`: 初始角度
    * `Tar_D`: 初始距離
    * `Tar_V`: 初始速度（或平均速度）
 
    * 輸出:
    * `Atar`: 目標目前的角度 (Angle)
    * `Dtar`: 目標目前的距離 (Distance)
    * `Vtar`: 目標目前的速度 (Velocity)
    * 如果目標是靜止的，這個函式可能就直接回傳常數
    * 如果目標在移動，Dtar 可能會根據速度更新
 
* `TX_SV = get_steering_vector(Tx_anten_spac, Wave_Length, Ntx, Atar);`: 根據目標目前的角度，計算發射端 steering vector
* `RX_SV = get_steering_vector(Rx_anten_spac, Wave_Length, Nrx, Atar);`: 根據目標目前的角度，計算接收端 steering vector

### 修正版本(我的多目標 monostatic sensing)

```matlab
    for b = 1:16
        TX_SV(:,b) = get_steering_vector(Tx_anten_spac, Wave_Length, Ntx, A_tar(b));
        RX_SV(:,b) = get_steering_vector(Rx_anten_spac, Wave_Length, Nrx, A_tar(b));
    end
```
* 這樣 `TX_SV` 就是一個 Ntx × 16 矩陣
---

### 對每個子載波 `n`，計算它因為目標距離 `Dtar` 而產生的相位延遲
```matlab
RX_PSd = zeros(Nsub, 1);
for n = 0:Nsub-1
    RX_PSd(n+1) = exp(1j * 2*pi * Sub_Spac * n * 2*abs(Dtar)/c);
end
```

* 每個子載波的頻率為：𝑓𝑛= 𝑛⋅Δ𝑓 = `𝑛⋅Sub_Spac`
* 信號經過往返距離 2𝐷 所產生的相位延遲: RX_PSd(n) = ​exp(j2π Δfn 2D/c)​

### 修正版本(我的多目標 monostatic sensing)

```matlab
RX_PSd = zeros(Nsub, K_targets);
for b = 16
    for n = 0:Nsub-1
        RX_PSd(n+1, k) = exp(1j * 2*pi * Sub_Spac * n * 2*abs(D_tar(b))/c);
    end
end
```

* `RX_PSd(:,b)` 就是第 b 個身體部位在子載波上的延遲相位

---

### 對每個符號時間 `m`，計算它因為目標速度 `Vtar` 造成的 Doppler shift

```matlab
    RX_PSv = zeros(Nsym, 1);
    for m = 0:Nsym-1
        RX_PSv(m+1) = exp(1j * 2*pi * Tto * m * 2*Vtar / Wave_Length);      % 計算Velocity phase shift
    end
```

* 當目標相對於雷達移動速度為 𝑉𝑡𝑎𝑟 時，Doppler shift 為 fD​ = 2Vtar​​/λ
* 每個符號時間 𝑚𝑇𝑠 會累積相位，因此: RX_PSv(m) = exp( j2π Tto m 2Vtar/λ​)

### 修正版本(我的 monostatic sensing)
```matlab
RX_PSv = zeros(Nsym, K_targets);
for b = 1:16
    for m = 0:Nsym-1
        RX_PSv(m+1, b) = exp(1j * 2*pi * Tto * m * 2*V_tar(b) / Wave_Length);
    end
end

```
---

### 生成接收雜訊

```matlab
    Radar_Noise = sqrt(Pn_rad/2) * (randn(Nrx, Nsub, Nsym) + 1i * randn(Nrx, Nsub, Nsym));
```

* 令 `n = sqrt(Pn_rad/2) * (X + jY)`
* 因此 `Pn_rad` 為雷達雜訊功率

#### `randn` 語法舉例
<img width="189" height="460" alt="image" src="https://github.com/user-attachments/assets/ffd1906d-322e-4199-a199-29c644f22d01" />

---

## Radar Rx

### tx/rx 矩陣變數初始化
* 表示在「子載波 n、符號 m 、每個發射天線」的傳送/接收符號
  
```matlab
RX_Signal = zeros(Nrx, Nsub, Nsym);
TX_Signal = zeros(Ntx, Nsub, Nsym);
```

---



```matlab
    for n = 1:Nsub       % 逐子載波
        for m = 1:Nsym   % 逐符號處理

            for u = 1:Nu                                                % 把多使用者的發射波形疊加成陣列輸出
                TX_Signal(:, n, m) = TX_Signal(:, n, m) + W(:,:,n,u) * Tx_sym(:,n,u);
            end

            for a = 1:Nrx
                if (mod(iter,2*Tar_app) < Tar_app)                          % 有目標時的信號模型 (若目標皆存在，可刪除 if;else)
                    RX_Signal(a, n, m) = alpha * RX_SV(a) * TX_SV' * TX_Signal(:, n, m) * RX_PSd(n) * RX_PSv(m) + Radar_Noise(a, n, m);
                else                                                        % 無目標時只剩下雜訊
                    RX_Signal(a, n, m) = Radar_Noise(a, n, m);
                end

            end
        end
    end
```



* `RX_Signal(a, n, m) = alpha * RX_SV(a) * TX_SV' * TX_Signal(:, n, m) * RX_PSd(n) * RX_PSv(m) + Radar_Noise(a, n, m);`: 計算「每根接收天線、每個子載波、每個符號」上的接收訊號

### 修正版本(我的 monostatic sensing)
```matlab
    for n = 1:Nsub       % 逐子載波
        for m = 1:Nsym   % 逐符號處理

            for u = 1:Nu                                                % Nu 為通訊用戶數量，在我的情境中設成1即可
                TX_Signal(:, n, m) = TX_Signal(:, n, m) + W(:,:,n,u) * Tx_sym(:,n,u);
            end

            for a = 1:Nrx
                RX_Signal(a, n, m) = 0;

                for b = 1:16
                    RX_Signal(a, n, m) = RX_Signal(a, n, m) + alpha(b) * RX_SV(a,b) * (TX_SV(:,b)') * TX_Signal(:, n, m)  * RX_PSd(n,b) * RX_PSv(m,b);
                end

                RX_Signal(a, n, m) = RX_Signal(a, n, m) + Radar_Noise(a, n, m);
            end

        end
    end
```

* 這樣每個目標的回波都會根據自己的：𝜃𝑘（角度）𝐷𝑘（延遲）𝑉𝑘（多普勒）𝛼𝑘（反射係數）貢獻到接收信號中
---

## Radar Process

```matlab
    A_FFT = fftshift(fft(RX_Signal, 512, 1));                  
    A_FFT_sum = sum(sum(abs(A_FFT).^2, 2), 3);   % 一個長度 512 的向量，對「所有子載波 & 符號」的512個角度上求能量總和
```

* `fft(RX_Signal, 512, 1)`: 對 RX_Signal 的第 1 維（天線維度）進行 512 點 FFT (若你只做 8 點 FFT：角度網格粗糙，只能得到 8 個方向樣本；峰值落在哪個 bin 上難以精準分辨)
* `fftshift()`: 把負頻率部分移到前面，使得：[−𝑁/2,…,−1,0,1,…,𝑁/2−1]，對應到「左 → 右」的方向餘弦（即角度範圍 [-90°, +90°]）。
* 經過空間 DFT 後，來自不同角度的回波會被「分離」到不同的 FFT bin 中

<img width="475" height="114" alt="image" src="https://github.com/user-attachments/assets/a7e0b420-c880-497d-8061-a31a2205cab9" />  

<img width="246" height="102" alt="image" src="https://github.com/user-attachments/assets/b73309e9-f2d2-442c-8e04-b7026630a8b0" />  

---
 
```matlab
    A_grid = asin(linspace(-1, 1, 512)) * 180/pi;   % 把 sinθ 的範圍 [-1,1] 切分成512個離散的 bin，再對它取 arsin，以取得每個 bin 對應的角度
    [~, A_peak] = max(abs(A_FFT_sum));              % 找出能量最大的那個角度 bin 索引
    % A_est = A_grid(A_peak);                       % 當你知道哪個 bin（A_peak）的能量最大時，就可以用它對應的角度 A_grid(A_peak) 當作「估測角度」
    A_est = 22.5;
```

*  `A_grid = asin(linspace(-1, 1, 512)) * 180/pi;`: 當 FFT bin 的對應角度 𝜃𝑛 ≈ 真實目標角度 𝜃𝑘， 就會在那個 bin 上看到能量峰值
<img width="384" height="104" alt="image" src="https://github.com/user-attachments/assets/7ae01010-3615-47d6-84dd-eb1cc645e2e7" />

```matlab
    A_BF = get_steering_vector(Rx_anten_spac, Wave_Length, Nrx, A_est);     % 計算Beaforming weight
    RX_signal_BF = zeros(Nsub, Nsym);
    for n = 1:Nsub
        for m = 1:Nsym
            RX_signal_BF(n, m) = A_BF' * RX_Signal(:, n, m) / sqrt(Nrx);    % 計算Beamforming後的接收訊號
        end
    end
```

* `A_BF = get_steering_vector(Rx_anten_spac, Wave_Length, Nrx, A_est); `: 「根據估測的角度 `A_est`」，產生接收 steering vector
    * 輸出: `A_BF = [1; e^{j2π(d/λ)sinθ}; e^{j2π2(d/λ)sinθ}; …]`
 
* 

