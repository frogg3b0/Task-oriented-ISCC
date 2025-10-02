# `backscatterPedestrian`語法背後的邏輯

### `backscatterPedestrian`建立一個物件，模擬行人反射的訊號。
- 此模型模擬 16 個身體部位的運動，以模擬自然運動。
- 該模型還模擬每個身體部分的雷達反射率。
- 從這個模型中，您可以獲得 『16 個身體部位』的位置和速度，以及身體移動時的總反向散射輻射。

<img width="560" height="337" alt="image" src="https://github.com/user-attachments/assets/b8a20987-c3d4-441f-85eb-f96dc07d971b" />

### 如何使用
- 建立行人後，您可以呼叫 `move` 來移動行人
- 若要取得反射訊號，使用 `reflect`

---

## Creation

```matlab
pedestrian = backscatterPedestrian( ...
              'Height',2,...                  % 行人身高 2m
              'WalkingSpeed',0.5, ...         % 移動速度 0.5m/s
              'PropagationSpeed',3e8, ...     % 電磁波傳播速度 3e8
              'OperatingFrequency',73e9, ...  % radar carrier frequency 
              'InitialPosition',[0;0;0], ...  % 行人初始位置 [0;0;0]
              'InitialHeading',90);...        % 行人初始朝向，以 x 軸方向為起點，故 90 為朝向 y 方向
```

---

## `backscatterPedestrian` 能用的方法

### 1. `move`
* 功能：更新行人的 位置與速度
* 模擬行人每個時間步的動作，產生不同 body segment 的位置
* 輸出：新的 position 和 velocity


### 2. `reflect`
* 功能：計算行人對雷達訊號的反射
* 輸入：傳入一段訊號 (例如 txSignal)
* 輸出：反射訊號，包含 Doppler shift 與 RCS (Radar Cross Section)


### 3. `plot`
* 功能：繪製一個「火柴人」(stick figure)
* 顯示 16 個 body segment 的當前位置

---

## Examples
* 計算沿 x 軸遠離原點移動的行人，行人最初距離雷達 100 公尺
* 雷達工作頻率為 24 GHz，位於原點，發射頻寬為 300 MHz 的 linear FM waveform
* 在行人開始移動時以及「移動兩秒後」捕捉反射訊號。

### 1. 定義雷達與波形
```matlab
c = physconst('Lightspeed');
bw = 300.0e6;
fs = bw;
fc = 24.0e9;

wav = phased.LinearFMWaveform('SampleRate',fs,'SweepBandwidth',bw);
x = wav();

channel = phased.FreeSpace('OperatingFrequency',fc,'SampleRate',fs, ...
    'TwoWayPropagation',true);
```
---

### 2. 建立行人模型

```MATLAB
pedest = backscatterPedestrian( ...
    'Height',1.8, ...
    'OperatingFrequency',fc, ...
    'InitialPosition',[100;0;0], ...
    'InitialHeading',0, ...
    'WalkingSpeed',0.5);
```

---

### 3-1 `move` 回傳 16 個部位的位置與速度，然後將行人移動提前兩秒鐘。

```matlab
[bppos,bpvel,bpax] = move(pedest,2,0);
```
* `move`: 更新行人的 16 個部位的位置 (bppos)、速度 (bpvel)、加速度 (bpax)
* 第二個參數 2 = 時間（秒），表示走了兩秒
* 第三個參數 0 = 朝向（heading），單位度，這裡表示沿 +x 方向走

---
### 3-2 傳送第一個脈衝到行人 (對應到 OFDM 則是第一個 Symbol)

```matlab
radarpos = [0;0;0];                                             % 雷達位置
xp = channel(repmat(x,1,16), radarpos, bppos, [0;0;0], bpvel);  % 把一個 Tx 信號複製成 16 個 column ，對應到行人各個身體部位
[~, ang] = rangeangle(radarpos, bppos, bpax);                   % 
y0 = reflect(pedest, xp, ang);
```
* `repmat(x,1,16)`: 把發射波形複製成 SampleRate × 16，分別送到 16 個身體部位。
* `channel(...)` : 模擬波形傳到這 16 個部位的路徑效應，輸出還是  SampleRate × 16
* `y0 = reflect(pedest,xp,ang)` :自動把這 16 個通道的反射波 加總成一條回波，輸出是 SampleRate × 1。

* `channel`: `phased.FreeSpace` 物件，功能是模擬 雷達到目標的傳播，它會根據：
    * 雷達位置 (radarpos = [0;0;0])
    * 目標位置 (bppos, 每個身體部位的 3D 座標)
    * 目標速度 (bpvel)
    * 幫每個部位加上 delay、path loss、Doppler shift

* `rangeangle`: 計算「從雷達出發到達每個身體部位的角度 」(azimuth, elevation)
    * `ang` 的維度是 2 × 16：每個部位都有一組 (az, el)

* `reflect()`:模擬每個部位反射回雷達的信號
    * 輸入：xp (傳到身體的信號 [Nsamp × 16])、ang (每個部位的入射角)
    * 輸出：y0 (每個部位反射回來的信號 [Nsamp × 16])

---

### 4-1 `move` 回傳 16 個部位的位置與速度，然後將行人移動提前兩秒鐘。
### 4-2 傳送第二個脈衝到行人 (對應到 OFDM 則是第二個 Symbol)

---


# 延伸思考: 如何將 `wav()` 換成 MIMO-OFDM

## [Apply OFDM in MIMO Simulation](https://www.mathworks.com/help/comm/ug/ofdm-with-mimo-simulation.html)

<img width="462" height="66" alt="image" src="https://github.com/user-attachments/assets/a1f664e8-a7b3-4634-be3c-f850623b763b" />

```matlab
ofdmMod = comm.OFDMModulator(
    FFTLength=128, ...
    PilotInputPort=false, ...  % 沒有導頻符號，只有 𝑠_𝑢𝑛,𝑝資料 → 所以 PilotInputPort=false                                       
    NumSymbols=14, ...
    InsertDCNull=false, ...    % 所有子載波都在用，沒有保留 DC → 所以 InsertDCNull=false
    NumTransmitAntennas=2);
```
