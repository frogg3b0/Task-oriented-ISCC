# Introduction to Micro-Doppler Effects

本範例介紹「微都卜勒效應 (micro-Doppler effect)」的基本概念，此效應出現在目標旋轉時的雷達回波中，透過分析 micro-Doppler 特徵，可以協助辨識目標類型。
* 一個移動的目標會因為 Doppler 效應 造成雷達回波的頻率偏移。
* 然而，大多數目標並非剛體 (rigid body)，除了整體運動外，其不同部分往往還有額外的振動或旋轉。
* 例如，直升機飛行時葉片旋轉；人行走時手臂自然擺動。
* 這些微尺度運動 (micro-scale motions) 會產生額外的都卜勒偏移，稱為 micro-Doppler 效應。
    * 這些微運動會調製回波訊號，使得 Doppler 頻率不再是單一值，而是形成「時間變化頻譜

---

### 本範例展示 micro-Doppler 效應在兩種應用中的用途：
1. 透過 micro-Doppler 特徵來估算「直升機葉片轉速」
2. 利用 micro-Doppler 特徵在汽車雷達回波中「辨識行人」

---

## Estimating Blade Speed of A Helicopter (估計直升機的葉片速度)
* 假設雷達位於原點
* 考慮一架帶有四個旋翼葉片的直升機。
    * 將直升機的位置指定為 （500， 0， 500）
    * 速度為 （60， 0， 0） m/s。
 
```matlab
radarpos = [0;0;0];
radarvel = [0;0;0];

tgtinitpos = [500;0;500];
tgtvel     = [60;0;0];
tgtmotion  = phased.Platform('InitialPosition',tgtinitpos,'Velocity',tgtvel);
```

---

整架直升機被簡化為 5 個散射中心 (scattering centers)：中心旋轉軸、四個葉尖    

```matlab
Nblades   = 4;                                    % 葉片數量
bladeang  = (0:Nblades-1)*2*pi/Nblades;           % 葉片初始角度
bladelen  = 6.5;                                  % 葉片長度 
bladerate = deg2rad(4*360);  % rps -> rad/sec     % 轉速
```

```matlab
c  = 3e8;
fc = 5e9;
helicop = phased.RadarTarget(                     % 五個散射中心的平均雷達散射截面
    'MeanRCS',[10 .1 .1 .1 .1],...
    'PropagationSpeed',c,...
    'OperatingFrequency',fc);
```

---

## Helicopter Echo Simulation

```MATLAB
fs     = 1e6;                        % 取樣率 
prf    = 2e4;                        % 脈衝重複頻率 (PRF) = 20 kHz → 每 50 µs 發射一次
lambda = c/fc;                       % 波長

wav = phased.RectangularWaveform(
    'SampleRate',fs,...
    'PulseWidth',2e-6,...
    'PRF',prf);

ura = phased.URA(           
    'Size',4,...                     % URA 陣列包含 4×4 = 16 個天線單元  
    'ElementSpacing',lambda/2);

tx  = phased.Transmitter;
rx  = phased.ReceiverPreamp;

env = phased.FreeSpace(
    'PropagationSpeed',c,...         % 電磁波傳播速度
    'OperatingFrequency',fc,...      % 電磁波運作頻率
    'TwoWayPropagation',true,        % 模擬往返傳播
    'SampleRate',fs);                % 取樣頻率

txant = phased.Radiator(             % 模擬天線向空間輻射信號 
    'Sensor',ura,...
    'PropagationSpeed',c,...
    'OperatingFrequency',fc);

rxant = phased.Collector(            % 模擬天線收集空間信號 
    'Sensor',ura,...
    'PropagationSpeed',c,...
    'OperatingFrequency',fc);
```

---

