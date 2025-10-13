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
### 初始化與變數定義
```matlab
NSampPerPulse = round(fs/prf);               % 每個脈衝取樣點數    
Niter = 1e4;                                 % 模擬 pulse 數 (0.5 秒)
y     = complex(zeros(NSampPerPulse,Niter)); % 接收信號矩陣 (大小 50×10000)
rng(2018);                                   % 固定隨機種子 
```


### 這段程式的邏輯是建立一個迴圈：每個 pulse（脈衝重複週期）內：
1. 更新直升機位置與葉片角度
2. 產生發射波形
3. 傳播 → 散射 → 回傳 → 接收
4. 累積接收波形矩陣 𝑦(:,𝑚)


```matlab
for m = 1:Niter
    % 更新散射中心的即時位置與速度
    t = (m-1)/prf;                 % 第 m 個 pulse 的絕對時間點
    [scatterpos,scattervel,scatterang] = helicopmotion(t,tgtmotion,bladeang,bladelen,bladerate);

    % simulate echo
    x  = txant(tx(wav()),scatterang);                    % transmit
    xt = env(x,radarpos,scatterpos,radarvel,scattervel); % propagates to/from scatterers
    xt = helicop(xt);                                    % reflect
    xr = rx(rxant(xt,scatterang));                       % receive
    y(:,m) = sum(xr,2);                                  % 針對 16 個天線輸出求和，得到單一複數波形對應第 m 個 pulse
end
```

### Range–Doppler Response 可視化

```matlab
rdresp  = phased.RangeDopplerResponse(         % 計算並顯示Range–Doppler Map
    'PropagationSpeed',c,...
    'SampleRate',fs,...                        % 快時間取樣率
    'DopplerFFTLengthSource','Property',... 
    'DopplerFFTLength',128,...                 % 對慢時間 (pulse index) 做 128 點 FFT
    'DopplerOutput','Speed',...                % 橫軸單位以「速度」顯示
    'OperatingFrequency',fc);                  % 5 GHz，決定 λ 用於換算速度 

mfcoeff = getMatchedFilter(wav);               % 自動生成與發射波形匹配的濾波器
plotResponse(rdresp,y(:,1:128),mfcoeff);
ylim([0 3000])
```

<img width="959" height="577" alt="image" src="https://github.com/user-attachments/assets/e7e0f392-66b1-4e93-993f-38949a2064b7" />

* 在 Range–Doppler 圖上，我們看到三個明顯的回波峰值
* 但實際上它們全部都來自同一個直升機目標的不同散射點


### 計算主體的真實徑向速度

```matlab
tgtpos = scatterpos(:,1);
tgtvel = scattervel(:,1);
tgtvel_truth = radialspeed(tgtpos,tgtvel,radarpos,radarvel)
```

`tgtvel_truth = -43.6435`


```matlab
maxbladetipvel = [bladelen*bladerate;0;0];
vtp = radialspeed(tgtpos,-maxbladetipvel+tgtvel,radarpos,radarvel)
vtn = radialspeed(tgtpos,maxbladetipvel+tgtvel,radarpos,radarvel)
```

`vtp = 75.1853`    : 「葉尖朝向雷達」時的徑向速度
`vtn = -162.4723`  : 「葉尖遠離雷達」時的徑向速度

* 若我們不事先知道這些回波都屬於同一個物體，雷達偵測會誤以為有三個獨立目標。
* 為了解決這個問題，後續可使用：Micro-Doppler feature extraction

---

## Blade Return Micro-Doppler Analysis

* 在 Range–Doppler 圖 中，我們只能看到「某一段時間內的平均速度分佈」
* 但 micro-Doppler 現象是 隨時間週期性變化的頻移
* 因此需要一種可以顯示「頻率隨時間擺動」的工具 → Spectrogram


```matlab
mf  = phased.MatchedFilter('Coefficients',mfcoeff);
ymf = mf(y);                             % 對所有 pulse 回波執行 matched filter     
[~,ridx] = max(sum(abs(ymf),2));         % 找出目標距離在哪一格
pspectrum(ymf(ridx,:),prf,'spectrogram') % 針對那個距離格做時間–頻率分析
```

<img width="959" height="577" alt="image" src="https://github.com/user-attachments/assets/9954ca64-e50f-415f-9e96-f36f8174ef01" />

* 這張圖的 中心水平線代表直升機機身的固定 Doppler 頻率，而上下對稱的「波浪形」頻率曲線是由葉片旋轉所引起的 Doppler
* 當葉片沿著圓軌道旋轉時，它在雷達視線方向的徑向速度 𝑣𝑟(𝑡) 會隨時間以餘弦形式變化
* 在一個完整旋轉週期裡，你能看到有 4 條相位互相錯開的正弦曲線；這表示有 4 個葉尖
* 這個週期= 250 ms 就是葉片轉一圈的時間
* 在圖上，弧線的最高（或最低）頻率距離中心線約 ±4 kHz，這個偏移量反映葉尖的最大徑向速度

---

## 模擬汽車雷達辨識行人（FMCW-based micro-Doppler analysis）

```matlab
bw = 250e6;    % FMCW頻寬 = 250 MHz
fs = bw;       % 取樣率設為頻寬
fc = 24e9;     % 雷達載波頻率 = 24 GHz
tm = 1e-6;     % Chirp掃頻時間 = 1 microsecond
wav = phased.FMCWWaveform(   % 產生 FMCW chirp
    'SampleRate',fs,...
    'SweepTime',tm,...
    'SweepBandwidth',bw);
```

### 定義「自車、停車、行人」的物理狀態與回波路徑

```matlab
egocar_pos = [0;0;0];                   % 自車的初始位置
egocar_vel = [30*1600/3600;0;0];        % 自車速度

% MATLAB 用來模擬運動平台的物件
egocar = phased.Platform(
    'InitialPosition',egocar_pos,...
    'Velocity',egocar_vel,...
    'OrientationAxesOutputPort',true);
```
### Parked car 停車目標設定

```matlab
parkedcar_pos = [39;-4;0];
parkedcar_vel = [0;0;0];

parkedcar = phased.Platform(
    'InitialPosition',parkedcar_pos,...
    'Velocity',parkedcar_vel,...
    'OrientationAxesOutputPort',true);

parkedcar_tgt = phased.RadarTarget(       % 定義一個雷達反射物體（可設定 RCS）
    'PropagationSpeed',c,...
    'OperatingFrequency',fc,...
    'MeanRCS',10);                        % Radar Cross Section = 10 m²，表示金屬車體強反射回波

```
* 這輛車是一個靜止高反射體
* 它會產生強烈的「固定距離、零 Doppler」回波。
* 對雷達而言，這種固定反射體容易掩蓋掉附近弱小的移動物（例如行人）


### Pedestrian 行人設定

```matlab
ped_pos = [40;-3;0];
ped_vel = [0;1;0];
ped_heading = 90;
ped_height = 1.8;

ped = phased.BackscatterPedestrian(
    'InitialPosition',ped_pos,...
    'InitialHeading',ped_heading,...
    'PropagationSpeed',c,...
    'OperatingFrequency',fc,...
    'Height',1.6,'WalkingSpeed',1);

chan_ped = phased.FreeSpace(
    'PropagationSpeed',c,...
    'OperatingFrequency',fc,...
    'TwoWayPropagation',true,
    'SampleRate',fs);

chan_pcar = phased.FreeSpace(
    'PropagationSpeed',c,...
    'OperatingFrequency',fc,...
    'TwoWayPropagation',true,'SampleRate',fs);
```
