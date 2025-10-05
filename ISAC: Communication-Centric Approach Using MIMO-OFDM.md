# Integrated Sensing and Communication II: Communication-Centric Approach Using MIMO-OFDM

[參考 MATLAB 官網說明](https://www.mathworks.com/help/phased/ug/integrated-sensing-and-communication-2-communication-centric-approach-using-mimo-ofdm.html)

---

## Introduction
#### ISAC 系統被視為一種關鍵策略，目的是解決以下兩個問題：
1. 頻譜資源日益擁擠（congested frequency spectrum）。
2. 雷達與通訊系統對大頻寬需求不斷上升（demand for large bandwidth）。

因此，將感測與通訊 整合進同一個框架，能有效提升頻譜利用率。

#### 在 ISAC 架構中：
* 雷達與通訊功能被整合（functional integration）。
* 共用硬體架構（share the same hardware infrastructure）。
* 共用相同的發射波形（common transmit waveform）。
也就是說，不需要獨立的雷達波形與通訊波形，而是由一個「整合波形」同時滿足兩者需求。

#### 這個範例展示了：如何在一個典型的 MIMO-OFDM 通訊系統 上，增加雷達感測能力。

* MIMO：多天線發射與接收，提供空間多樣性與多重路徑增益。
* OFDM：正交分頻多工，利用子載波分割頻寬，實現高數據率傳輸。
這裡的重點是「在不改變通訊主架構的情況下，附加雷達感測功能」。

---

## System Parameters

```matlab
rng('default');                                                         % Set the random number generator for reproducibility

carrierFrequency = 6e9;                                                 % Carrier frequency (Hz)
waveLength = freq2wavelen(carrierFrequency);                            % Wavelength
bandwidth = 100e6;                                                      % Bandwidth (Hz)
sampleRate = bandwidth; 
```

#### 建立一個 TX

```matlab
peakPower = 1.0;                                            % Peak power (W)
transmitter = phased.Transmitter('PeakPower', peakPower, 'Gain', 0);
```
* `Gain = 0 dB`：表示發射機本身不增加額外增益

### 建立一個 RX

```matlab
noiseFigure = 3.0;                                                      % Noise figure (dB)
referenceTemperature = 290;                                             % Reference temperature (K)
receiver = phased.Receiver('SampleRate', sampleRate, 'NoiseFigure', noiseFigure,...
    'ReferenceTemperature', referenceTemperature, 'AddInputNoise', true,...
    'InputNoiseTemperature', referenceTemperature, 'Gain', 0);
```
* `SampleRate = sampleRate` → 採樣率 = 100 MHz
* `NoiseFigure = 3 dB`
* `ReferenceTemperature = 290 K`
* `AddInputNoise = true` → 在接收端自動加入高斯白雜訊
* `InputNoiseTemperature = referenceTemperature` → 設定雜訊溫度
* `Gain = 0 dB` → 接收端不額外放大信號（理想化設定）

### 讓 TX/RX 都具有八個各向同性天線的均勻線性陣列 ，這些天線間隔半個波長

```matlab
Ntx = 8;                                                            
Nrx = 8;                                                      

element = phased.IsotropicAntennaElement('BackBaffled', true);  
txArray = phased.ULA(Ntx, waveLength/2, 'Element', element);
rxArray = phased.ULA(Nrx, waveLength/2, 'Element', element);
```

* `element = phased.IsotropicAntennaElement('BackBaffled', true)`
    * 定義天線元素為 各向同性 (isotropic)。
    * `'BackBaffled', true `→ 消除背面輻射（只向前輻射）。

* `txArray = phased.ULA(Ntx, waveLength/2, 'Element', element);`
    * 建立 Tx 陣列，8 個天線，間距 λ/2。

* `rxArray = phased.ULA(Nrx, waveLength/2, 'Element', element);`
    * 建立 Rx 陣列，8 個天線，間距 λ/2。

---

## ISAC Scenario (場景設定)
描述了發射機、接收機的位置、指向方向，以及環境中散射體的建模  

### 設定 TX/RX 的位置、朝向
```MATLAB
txPosition = [0; 0; 0];                                                
txOrientationAxis = eye(3);     % 設定陣列方向矩陣為單位矩陣，代表面向 x 軸

rxPosition = [80; 60; 0];                                               
rxOrientationAxis = rotz(-90);  % 繞 z 軸逆時針旋轉 90°。    
```
---

### TX/RX 之間的最大路徑長度、目標速度限制

```MATLAB
maxPathLength = 300;                  % 限制模擬中 最長的通道路徑 = 300 m
maxVelocity = 50;                     % 散射體最大速度 = 180 km/h
```

---

### 考慮三個移動散射體的場景

```MATLAB
targetPositions = [60 70 90;                                            % Target positions (m)
                   -25 15 30;
                   0 0 0]; 

targetVelocities = [-15 20 0;                                           % Target velocities (m/s)
                    12 -10 25;
                    0 0 0];
```

```MATLAB
targetMotion = phased.Platform('InitialPosition', targetPositions, 'Velocity', targetVelocities);            % MATLAB 內建物件，用來模擬目標隨時間移動的軌跡。
targetReflectionCoefficients = randn(1, size(targetPositions, 2)) + 1i*randn(1, size(targetPositions, 2));   % 使用 隨機複數 作為三個目標的反射係數。對應於 Radar Cross Section (RCS)，決定每個目標的回波強度。
```

---
### 靜態散射體 (Static Scatterers)

```MATLAB
regionOfInterest = [0 120; -80 80];                                     % Bounds of the region of interest
numScatterers = 200;                                                    % Number of scatterers distributed within the region of interest
[scattererPositions, scattererReflectionCoefficients] = helperGenerateStaticScatterers(numScatterers, regionOfInterest);
```
* `regionOfInterest`: 限定靜態散射體生成的區域：
    * x ∈ [0,120] m
    * y ∈ [-80,80] m
* `numScatterers = 200;`: 散射體個數，是「背景雜亂散射 (clutter)」
* `helperGenerateStaticScatterers`: 隨機生成散射體的位置與反射係數

---

## 使用 `phased.ScatteringMIMOChannel` 物件，建構發射端與接收端之間的傳播通道

```MATLAB
channel = phased.ScatteringMIMOChannel(
    'CarrierFrequency', carrierFrequency,...             % 載波頻率 = 6 GHz
    'TransmitArray', txArray,...                         % 發射端的天線陣列物件（8 元件 ULA）。
    'TransmitArrayPosition', txPosition,...              % 發射端位置 [0; 0; 0]
    'ReceiveArray', rxArray,...                          % 接收端的天線陣列物件（8 元件 ULA）。
    'ReceiveArrayPosition', rxPosition,...               % 接收端位置 [80; 60; 0]
    'TransmitArrayOrientationAxes',txOrientationAxis,... % 發射端方向矩陣：eye(3) → 朝向 +x 軸
    'ReceiveArrayOrientationAxes', rxOrientationAxis,... % 接收端方向矩陣：rotz(-90) → 朝向 -y 軸
    'SampleRate', sampleRate,...                         % 取樣頻率，對應於 OFDM 頻寬
    'SimulateDirectPath', true,...                       % 考慮 LOS
    'ScattererSpecificationSource', 'Input Port');       % 表示散射體的位置與反射係數由外部輸入，而不是在通道物件內部生成
```

## 使用 `helperVisualizeScatteringMIMOChannel` 來視覺化散射 MIMO 通道

```MATLAB
helperVisualizeScatteringMIMOChannel(channel, scattererPositions, targetPositions, targetVelocities)
title('Scattering MIMO Channel for Communication-Centric ISAC Scenario');
```
* `helperVisualizeScatteringMIMOChannel()`: 會根據 Tx/Rx 的位置 + 散射體位置 + 目標位置與速度，繪製出 ISAC 場景示意圖。

<img width="560" height="337" alt="image" src="https://github.com/user-attachments/assets/e7dad177-7804-4031-a7d7-2cb5c1f521e7" />

---
```matlab
Nsub = 2048;                                   % subcarrier 數量
subcarrierSpacing = bandwidth/Nsub;            % subcarrier 間隔 = 總頻寬/subcarrier數量
ofdmSymbolDuration = 1/subcarrierSpacing;      % OFDM 符號長度=1/Δf
```

### 為了確保 OFDM 子載波的正交性不會因為 Doppler shift 破壞
* 要求：子載波間隔 ≥ 10 倍的最大 Doppler 頻移
​
```
maxDopplerShift = speed2dop(maxVelocity, waveLength);
fprintf("Subcarrier spacing is %.2f times larger than the maximum Doppler shift.\n", subcarrierSpacing/maxDopplerShift); 
```
* 顯示 「子載波間隔約為最大 Doppler shift 的 `%.2f`倍」

---

### Cyclic Prefix 設定   
會根據 **maximum range of interest** ，計算 CP 的持續時間    

```MATLAB
cyclicPrefixDuration = range2time(maxPathLength);                       % CP 長度必須大於通道的最大延遲擴展，也就是「最大路徑長度對應的傳播時間」
cyclicPrefixLength = ceil(sampleRate*cyclicPrefixDuration);             % 所需 CP 樣本數
cyclicPrefixDuration = cyclicPrefixLength/sampleRate;                   % 調整 CP 的持續時間，使其具有整數樣本

Tofdm = ofdmSymbolDuration + cyclicPrefixDuration;                      % OFDM 符號總時長 (含 CP)
ofdmSymbolLengthWithCP = Nsub + cyclicPrefixLength;                     % 總樣本數 = 子載波數 + CP樣本數
```

### 在子載波上加入 Guard Bands 
* 為避免與鄰近頻段系統互相干擾，需要設定 guard band carriers
* 這些子載波不承載任何資料或導頻，只是空置，避免邊緣外溢

```MATLAB
numGuardBandCarriers = [9; 8];                                                % 前 9 個子載波 & 後 8 個子載波  = 保護區
numActiveSubcarriers = Nsub-numGuardBandCarriers(1)-numGuardBandCarriers(2);  % 可用子載波數 = 2048 − 17 = 2031
```

---

## 通訊導向 ISAC 系統的信號流程 (Signaling Scheme)

## 1. Initial channel sounding
* 發射端一開始會傳送 preamble (導頻前置序列)
* 接收端使用 preamble 來取得 初始通道估計 (channel estimate)
* 這對 OFDM 通訊 和 ISAC 感測 都至關重要，因為需要知道通道響應 (CIR) 才能正確解調 & 定位目標

## 2. Channel coding
* 接收端利用 初始通道估計 計算出：
    * Precoding、Combining weights
    * precoding 權重會透過 feedback 送回發射端
      
## 3. Data transmission
* 每個 frame 被分為 兩個 subframes：
    * Subframe A：僅傳送 數據符號 (data symbols) → 專注於通訊。
    * Subframe B：傳送 數據符號 + 導頻 (pilots) → 同時支援 通訊與感測
      
## 4. Channel estimate and precoding weights update
* 接收端接收到 Subframe B：
    * 使用其中的 pilots 重新估計通道
    * 更新 precoding & combining 權重。
    * 更新後的 precoding 權重回傳給發射端

## 5. Radar data cube formation
* Radar Data Cube 結構：
    * 維度 1 → 接收天線 (Rx)
    * 維度 2 → 子載波 / 頻率 bins (Range info)
    * 維度 3 → 時間 / frame index (Doppler info)
    * 這個 三維結構（通常稱為 Range-Doppler-Angle cube）是進行目標檢測與參數估計 (位置、速度) 的基礎
 
## 6. 重複 step3 ~ step5
上述 步驟 3 → 5 會不斷重複，直到傳輸 numFrames 個 frames。  

* 最終，系統會利用 Radar Data Cube 來進行：
    * 目標位置估計 (range, angle)
    * 相對速度估計 (Doppler)

<img width="1105" height="798" alt="image" src="https://github.com/user-attachments/assets/1297f841-4f20-4f27-aafb-575ab8868880" />


---

## 1. Initial Channel Sounding

### 一句話理解這段
使用第一個「symbol」，把裡面所有可用的 subcarrier 分配給各天線

* preamble (前置序列)：一個已知符號，發射端與接收端事先知道其內容。作用：
    * 在 系統剛開始傳輸時，先送一個 preamble OFDM 符號。
    * 接收端利用它估計初始通道矩陣 𝐻
    * 根據通道估計，計算 precoding 和 combining

* 在 OFDM 中，有很多子載波（這裡是 Nsub = 2048）。
* 我們想要在 preamble 符號裡放上已知 pilot，來做「初始通道估計」。
* 但是有 多根發射天線 (Ntx = 8)，如果每根天線都在同一個子載波上放 pilot，接收端就分不出誰是誰。
* 👉 所以要「分配不同的子載波給不同的天線」。

```matlab
idxs = [(numGuardBandCarriers(1)+1):Ntx:(Nsub/2-Ntx+1)...
    (Nsub/2+2):Ntx:(Nsub-numGuardBandCarriers(2)-Ntx+1)]';

numPreambleSubcarriers = numel(idxs);
```
* 這行要生成 第一根天線 (Tx1) 使用的子載波索引
    * 因為前 9 個子載波 & 後 8 個子載波當 guard band(不使用)，所以可用子載波範圍大概是 10 ~ 2040
    * `numGuardBandCarriers(1)+1 : Ntx : (Nsub/2 - Ntx + 1)` : 從「第 10 個子載波」開始，每隔 8 個選一個，一直到中間 (DC 前)
    * `(Nsub/2+2):Ntx:(Nsub-numGuardBandCarriers(2)-Ntx+1)`  : 從「DC 之後」開始，每隔 8 個選一個，一直到最後第 2040 個子載波

* 輸出 `idx`: 第一根天線要用的子載波索引（e.g., 10、18、26、34...)
* 輸出 `numPreambleSubcarriers = numel(idxs)`: 第一根天線總共用了多少個子載波

---

```matlab
preambleIdxs = zeros(numPreambleSubcarriers, 1, Ntx);
for i = 1:Ntx
    preambleIdxs(:, 1, i) = idxs + 1*(i-1);
end
```
* `preambleIdxs` : 初始化一個矩陣，用來將各天線占用的subcarrier index存入，大小為: numPreambleSubcarriers × 1 × Ntx)
* preambleIdxs(:,1,1) = idxs = [10; 18; 26; 34; ...]
* preambleIdxs(:,1,2) = idxs + 1 = [11; 19; 27; 35; ...]
使用 `for` 迴圈，讓每根天線都拿到「錯開的子載波組」

### 前面已經為每根天線分配好 subcarrier 去塞入 preamble
### 接下來要定義每個 subcarrier 上的 preamble

使用已知序列作為前導碼   


```matlab
preamble = mlseq(Nsub - 1);                       % MLS 特性：像隨機序列，但自相關性很好，非常適合做 通道估計
preamble = preamble(1 : numPreambleSubcarriers);  % 只取前 numPreambleSubcarriers 個元素
preamble = repmat(preamble, 1, 1, Ntx);           % repmat 把這個序列複製到每一根發射天線 (所有天線用相同的值，但因為前面已經分配了不同的子載波索引)
```

### 已經為每根天線分配好 subcarrier 去塞入 preamble，並且也知道每個 subcarrier 上的 preamble 的值
### 接著就是要用該 symbol 上的所有 subcarrier 的 preamble 

