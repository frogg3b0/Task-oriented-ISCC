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

```MATLAB
txPosition = [0; 0; 0];                                                
txOrientationAxis = eye(3);     % 設定陣列方向矩陣為單位矩陣，代表面向 x 軸

rxPosition = [80; 60; 0];                                               
rxOrientationAxis = rotz(-90);  % 繞 z 軸逆時針旋轉 90°。    
```


