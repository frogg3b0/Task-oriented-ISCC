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

```matlab
preambleMod = comm.OFDMModulator(
    "CyclicPrefixLength", cyclicPrefixLength,...
    "FFTLength", Nsub,...
    "NumGuardBandCarriers", numGuardBandCarriers,...
    "NumSymbols", 1,...                                % 因為 preamble 只需要一個 OFDM 符號。
    "NumTransmitAntennas", Ntx,...
    "PilotCarrierIndices", preambleIdxs,
    "PilotInputPort", true);

preambleInfo = info(preambleMod);                      % 取得這個 modulator 的結構資訊，特別是 Data 輸入大小、Pilot 輸入大小 

```
**我們有了一個「專門發 preamble 的 OFDM 調變器」**  


### 建立 OFDM 解調器
```matlab
preambleDemod = comm.OFDMDemodulator(preambleMod);
preambleDemod.NumReceiveAntennas = Nrx;
```

* `preambleDemod = comm.OFDMDemodulator(preambleMod);`
    * 直接複製調變器的設定，建立一個解調器，確保參數一致。
    * 這樣它就知道如何把接收到的信號拆回子載波、拆回每根天線的 pilot。

preambleDemod.NumReceiveAntennas = Nrx;

設定接收天線數 = 8。

👉 結果：我們有了一個能「解析 preamble 符號」的解調器。  

### 整體流程總結
1. 用 MLS 生成一組已知序列
2. 複製給所有天線，但每根天線的子載波位置不同（靠 preambleIdxs 分配）
3. 用 comm.OFDMModulator 把 preamble 映射到子載波並加上 CP → 轉成時域信號
4. 收到信號後，用 comm.OFDMDemodulator 拆解，恢復每根天線的 pilot → 拿來做初始通道估計

### 把 preamble 實際發出去 → 經過通道

1. 生成發射的 preamble 訊號

```matlab
preambleSignal = preambleMod(zeros(preambleInfo.DataInputSize), preamble);

```
* 進行通道偵測時，幾乎所有子載波都用於前導碼，因此將剩餘子載波置零

2. 模擬發射

```matlab
txSignal = transmitter(preambleSignal);
```
* 這裡呼叫了先前建立的： `transmitter = phased.Transmitter('PeakPower', peakPower, 'Gain', 0);`
    * 它會把信號乘上對應的發射功率與增益
    * `txSignal` = 實際輸出的基帶信號
 
3. 經過散射通道

```matlab
channelSignal = channel(txSignal, [scattererPositions targetPositions],...
    [zeros(size(scattererPositions)) targetVelocities],...
    [scattererReflectionCoefficients targetReflectionCoefficients]);
```

channel 是 `phased.ScatteringMIMOChannel` 物件  

* 輸入：
    * txSignal → 你剛送出的多天線 OFDM preamble
    * [scattererPositions targetPositions] → 場景中所有散射體的位置（靜態 + 目標）
    * [zeros(size(scattererPositions)) targetVelocities] → 靜態散射體速度 = 0，目標有各自速度
    * [scattererReflectionCoefficients targetReflectionCoefficients] → 每個散射體的反射係數（複數，代表反射強度與相位）
 
* 它在做什麼：
    * 模擬每條多徑的延遲、Doppler shift、相位旋轉、以及不同 Tx–Rx 路徑的疊加，產生對應的「多天線接收信號」。
    * 結果：channelSignal = 經過環境散射後，到達接收端的多通道基帶信號
 
4. 加上接收雜訊

```matlab
rxSignal = receiver(channelSignal);
```

* `receiver(channelSignal)`: 它會在信號中加入符合熱噪聲模型的隨機雜訊：𝑁0=𝑘𝑇⋅𝐹
* `rxSignal` = 通道傳遞後、再加上接收端熱雜訊的實際接收信號

5. OFDM 解調
```matlab
[~, rxPreamblePilots] = preambleDemod(rxSignal);
```

* 使用前面建立的 OFDM 解調器 (preambleDemod)
* 功能：
    * 去除 CP
    * 做 FFT → 回到頻域
    * 把各天線、各子載波上的 pilot 值拆出來

* 輸出：
    * 第一個輸出（Data symbols）我們不要，所以用 ~ 忽略
    * 第二個輸出 → rxPreamblePilots，就是 接收到的 pilot 值
    * 👉 這是實際接收到的「量測值」，接下來要跟原本的已知 preamble 比對，推回通道
 
6. 估計通道矩陣
```matlab
channelMatrix = helperInterpolateChannelMatrix(Nsub, numGuardBandCarriers, squeeze(preamble), squeeze(rxPreamblePilots), preambleIdxs);
```
* 這個 helper function 幫你做「通道估計」
* 結果：`channelMatrix` 是一個三維 RDA 矩陣

7. 計算 precoding / combining 權重

```MATLAB
[Wp, Wc, ~, G] = diagbfweights(channelMatrix);
```
* diagbfweights 會根據通道矩陣求出：
    * Wp：發射端 precoding 權重
    * Wc：接收端 combining 權重
    * G：等效通道增益矩陣
    * 👉 這些權重之後會被用在 Data subframe 的資料傳輸與感測信號接收中。

### 使用 preamble 估計通道總結
* 模擬一個完整的 MIMO–OFDM 初始通道探測流程： 「已知 preamble → 通道 → 解調 → 比對 → 得到通道矩陣與 beamforming 權重。」

---

## Data Frame Transmission
* 在完成了 初始通道 (preamble) 之後，開始正式傳送資料（subframe A）以及為後續感測準備新的 pilots

1. 建立建立 OFDM 調變器與解調器（Subframe A）

```matlab
% Subframe A contains only data
subframeAMod = comm.OFDMModulator(
    "CyclicPrefixLength", cyclicPrefixLength,...
    "FFTLength", Nsub,...
    "NumGuardBandCarriers", numGuardBandCarriers,...
    "NumTransmitAntennas", Ntx,...
    "NumSymbols", subframeALength);                  % 計算 subframe A 的時間長度（以符號為單位）
subframeAInfo = info(subframeAMod);

subframeADemod = comm.OFDMDemodulator(subframeAMod);
subframeADemod.NumReceiveAntennas = Nrx;
```
* 一個「frame」裡面分成：
    * Subframe A：只傳 data symbols
    * Subframe B：同時傳 data + pilots（用來做感測與通道更新）

* 所以這裡先建的是 subframe A 的 OFDM 調變器/解調器

2. 安排 pilot 子載波索引/生成各天線的 Pilot 序列

```matlab
pilotIdxs = [
    (numGuardBandCarriers(1)+1):Mf:(Nsub/2),...
    (Nsub/2+2):Mf:(Nsub-numGuardBandCarriers(2))]';

pilots = zeros(numel(pilotIdxs), Ntx, Ntx);
for itx = 1:Ntx
    s = mlseq(Nsub-1, itx);
    pilots(:, itx, itx) = s(1:numel(pilotIdxs));
end
```


---
* Subframe B 用來同時傳資料與 pilot（既要維持通訊，也要更新通道給感測使用）。
* 一個 frame = Subframe A + Subframe B。
    * A：純 data
    * B：data + pilots
  
1. 建立建立 OFDM 調變器與解調器（Subframe B）

```MATLAB
subframeBMod = comm.OFDMModulator(
    "CyclicPrefixLength", cyclicPrefixLength,...
    "FFTLength", Nsub,...
    "NumGuardBandCarriers", numGuardBandCarriers,...
    "NumTransmitAntennas", Ntx,...
    "NumSymbols", Ntx,...                                 % 每根發射天線各佔用 1 個 OFDM symbol 來發 pilot。若 Ntx = 8 → Subframe B = 8 個 OFDM symbols
    "PilotCarrierIndices", pilotIdxs,...                  % 前一步設計的 pilot 子載波位置
    "PilotInputPort", true);
subframeBInfo = info(subframeBMod);

subframeBDemod = comm.OFDMDemodulator(subframeBMod);
subframeBDemod.NumReceiveAntennas = Nrx;

subframeBdataSubcarrierIdxs = setdiff(numGuardBandCarriers(1)+1:(Nsub-numGuardBandCarriers(2)), pilotIdxs); % 決定 Subframe B 的 Data 子載波索引
```
* `subframeBMod`: 能產生同時包含 data + pilot 的 OFDM 波形
* `subframeBDemod`: 能對應拆回頻域資料與 pilot 值


2. 計算速度解析度 (velocity resolution)

```MATLAB
Nframe = 24;                                                                                % 連續 Nframe 個 OFDM frame 做「coherent processing」
fprintf("Velocity resolution: %.2f (m/s).\n", dop2speed(1/(Nframe*Tofdm*Mt), waveLength));  % Velocity resolution = 4.21 m/s
```

3. 設定調變階數（64-QAM）

```MATLAB
bitsPerSymbol = 6;           % 每個QAM符號可攜帶6bit
modOrder = 2^bitsPerSymbol;  % 調變階數=64
```

---

4. 定義 Spatial multiplexing gain

* 在 MIMO-OFDM 系統裡，若環境有足夠多的散射路徑（rich scattering environment），每根發射天線和接收天線間的通道矩陣 𝐻 可能是「滿秩（full-rank）」的。
    * 這代表不同天線組合能形成互不干擾的「空間通道（spatial channels）」
    * 因此可以同時傳送多筆獨立資料，稱為 spatial multiplexing

```matlab
numDataStreams = 2;  % 發送兩條獨立的資料流
```

5. 計算每個 OFDM 調變器的輸入資料大小

```matlab
% Input data size for subframe A
subframeAInputSize = [subframeAInfo.DataInputSize(1) subframeAInfo.DataInputSize(2) numDataStreams];

% Input data size for subframe B
subframeBInputSize = [subframeBInfo.DataInputSize(1) subframeBInfo.DataInputSize(2) numDataStreams];
```

6. 為「雷達資料立方（Radar Data Cube）」預留空間

```matlab
radarDataCube = zeros(numActiveSubcarriers, Nrx, Nframe);
```
* 這是一個三維矩陣，維度如下：
    * `numActiveSubcarriers`: 頻率維度
    * `Nrx`: 接收天線維度
    * `Nframe`: 時間維度
 
* 在每個 frame 結束後，新的通道估計結果會存入這個 cube 的下一頁 (page)，最後就能用 FFT 或 MUSIC 等方法去估計目標的距離、速度與方向。

---

模擬每一個 frame 的：
* 資料生成與 QAM 調變
* Precoding（空間資料流映射）
* OFDM 調變與實際傳送
* 通道傳播（含目標移動與散射）
* 接收、解調、合併（Combining）
* BER 計算
* 通道重估（更新 precoder/combiner）
* 雷達資料立方更新


```matlab
for i = 1:Nframe
    %%
    % 隨機產生要傳的二進位資料（payload），使用 64-QAM 調變成複數符號
    subframeABin = randi([0,1], [subframeAInputSize(1) * bitsPerSymbol subframeAInputSize(2) numDataStreams]);
    subframeAQam = qammod(subframeABin, modOrder, 'InputType', 'bit', 'UnitAveragePower', true);
    
    %%
    % 使用 precoder 𝑊 將兩條 data streams 映射到八根發射天線。
    subframeAQamPre = zeros(size(subframeAQam, 1), subframeALength, Ntx);
    for nsc = 1:numActiveSubcarriers
        subframeAQamPre(nsc, :, :) = squeeze(subframeAQam(nsc, :, :))*squeeze(Wp(nsc, 1:numDataStreams,:));
    end
    
    %%
    % Subframe A：OFDM 調變
    subframeA = subframeAMod(subframeAQamPre);

    %%
    % Subframe B：資料 + Pilot 的組合與傳送準備，這一段幾乎重複 Subframe A 的步驟，但多了 pilot
    subframeBBin = randi([0,1], [subframeBInputSize(1) * bitsPerSymbol subframeBInputSize(2) numDataStreams]);
    subframeBQam = qammod(subframeBBin, modOrder, 'InputType', 'bit', 'UnitAveragePower', true);

    % Precodding
    subframeBQamPre = zeros(size(subframeBQam, 1), Ntx, Ntx);    
    for nsc = 1:numel(subframeBdataSubcarrierIdxs)
        idx = subframeBdataSubcarrierIdxs(nsc) - numGuardBandCarriers(1);
        subframeBQamPre(nsc, :, :) = squeeze(subframeBQam(nsc, :, :))*squeeze(Wp(idx, 1:numDataStreams,:));
    end

    % 加上 pilot（for sensing + channel estimation）
    subframeB = subframeBMod(subframeBQamPre, pilots);

    % Binary data transmitted in the ith frame
    txDataBin = cat(1, subframeABin(:), subframeBBin(:));
    
    % Reshape and combine subframes A and B to transmit the whole frame

    %%
    % 每個 frame = [Subframe A | Subframe B]。
    % 把整個 frame 合成時間序列訊號矩陣 [Sample × Symbols × Ntx] Precode data subcarriers for subframe B
    subframeA = reshape(subframeA, ofdmSymbolLengthWithCP, subframeALength, []);
    subframeB = reshape(subframeB, ofdmSymbolLengthWithCP, Ntx, []);
    ofdmSignal = [subframeA subframeB];

    % Preallocate space for the received signal
    rxSignal = zeros(size(ofdmSignal, 1), size(ofdmSignal, 2), Nrx);

    % Transmit one OFDM symbol at a time
    for s = 1:size(ofdmSignal, 2)
        % 更新目標位置與速度
        [targetPositions, targetVelocities] = targetMotion(Tofdm);

        % Transmit signal
        txSignal = transmitter(squeeze(ofdmSignal(:, s, :)));        
        
        % 經過多路散射通道
        channelSignal = channel(txSignal, [scattererPositions targetPositions],...
            [zeros(size(scattererPositions)) targetVelocities],...
            [scattererReflectionCoefficients targetReflectionCoefficients]); 
        
        % 加入熱雜訊
        rxSignal(:, s, :) = receiver(channelSignal);
    end

    % Separate the received signal into subframes A and B
    rxSubframeA = rxSignal(:, 1:subframeALength, :);
    rxSubframeA = reshape(rxSubframeA, [], Nrx);

    rxSubframeB = rxSignal(:, subframeALength+1:end, :);
    rxSubframeB = reshape(rxSubframeB, [], Nrx);

    % OFDM 解調
    rxSubframeAQam = subframeADemod(rxSubframeA);
    rxSubframeAQamComb = zeros(size(rxSubframeAQam, 1), size(rxSubframeAQam, 2), numDataStreams);

    for nsc = 1:numActiveSubcarriers
        rxSubframeAQamComb(nsc, :, :) = ((squeeze(rxSubframeAQam(nsc, :, :))*squeeze(Wc(nsc, :, 1:numDataStreams))))./sqrt(G(nsc,1:numDataStreams));
    end

    % Demodulate subframe B and apply the combining weights
    [rxSubframeBQam, rxPilots] = subframeBDemod(rxSubframeB);
    rxSubframeBQamComb = zeros(size(rxSubframeBQam, 1), size(rxSubframeBQam, 2), numDataStreams);

    for nsc = 1:numel(subframeBdataSubcarrierIdxs)
        idx = subframeBdataSubcarrierIdxs(nsc) - numGuardBandCarriers(1);
        rxSubframeBQamComb(nsc, :, :) = ((squeeze(rxSubframeBQam(nsc, :, :))*squeeze(Wc(idx, :, 1:numDataStreams))))./sqrt(G(idx, 1:numDataStreams));
    end

    % Demodulate the QAM data and compute the bit error rate for the ith
    % frame
    rxDataQam = cat(1, rxSubframeAQamComb(:), rxSubframeBQamComb(:));
    rxDataBin = qamdemod(rxDataQam, modOrder, 'OutputType', 'bit', 'UnitAveragePower', true);
    [~, ratio] = biterr(txDataBin, rxDataBin);
    fprintf("Frame %d bit error rate: %.4f\n", i, ratio);

    % 使用 subframe B 的 pilot 進行通道估計
    channelMatrix = helperInterpolateChannelMatrix(Nsub, numGuardBandCarriers, pilots, rxPilots, pilotIdxs);
    
    % 更新 precoder / combiner，準備下個 frame
    [Wp, Wc, ~, G] = diagbfweights(channelMatrix);

    % Store the radar data
    radarDataCube(:, :, i) = squeeze(sum(channelMatrix, 2));    
end
```  

---

## Radar Data Processing

### 把通訊過程中「估得的 channel matrices」當成雷達回波，進行 range–Doppler–angle 分析

```matlab
Y = fft(radarDataCube, [], 3);  % 在慢時間（frame）做 FFT 
Y(:, :, 1) = 0;                 % 把靜態散射體移除
y = ifft(Y, Nframe, 3);         % IFFT 回時域
```

### 利用 MIMO 幾何結構 + array steering；根據已知的 Tx / Rx 位置、方向；把回波功率映射到空間座標 (x,y)。 
```matlab
phm = helperPositionHeatmap('ReceiveArray', rxArray, 'ReceiveArrayOrientationAxis', rxOrientationAxis, 'ReceiveArrayPosition', rxPosition, ...
    'SampleRate', sampleRate, 'CarrierFrequency', carrierFrequency, 'Bandwidth', bandwidth, 'OFDMSymbolDuration', ofdmSymbolDuration, ...
    'TransmitArrayOrientationAxis', txOrientationAxis, 'TransmitArrayPosition', txPosition, 'TargetPositions', targetPositions, 'ROI', [0 120; -80 80]);

figure;
phm.plot(y)
title('Moving Scatterers')
```


