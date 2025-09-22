# mainHumanMotionRadar

### 基本參數設定
```matlab
npulse = 2000; % number of pulses, total time 3000 * 0.001 = 3.0 seconds
bw = 10e6;     % FMCW 掃頻頻寬 10 MHz
fs = 1.25*bw;  % 取樣率 = 1.25 × bw = 12.5 MHz
fc = 60e9;     % 載波頻率 60 GHz
tm = 10e-6;    % chirp 時間 (掃頻時間) = 10 µs
c = 3e8;       % 光速
Tsamp = 2.5e-4;% 每個 frame 的時間長度 0.25 ms
```

---

### FMCW 波形定義

```matlab
wav = phased.FMCWWaveform('SampleRate',fs,'SweepTime',tm,...
    'SweepBandwidth',bw); % transmit signal
```
- `phased.FMCWWaveform`: 物件用來產生 線性調頻 (chirp) 訊號
- 設定了取樣率、掃頻時間、掃頻帶寬，這就是 FMCW 雷達的 發射基帶信號

---
### 發射端 (Tx)

```matlab
tx_pos = [3;1.5;1];  % TX 位置 (x=3, y=1.5, z=1)
tx_vel = [0;0;0];    % TX 靜止
radar_tx = phased.Platform('InitialPosition',tx_pos,'Velocity',tx_vel,...
    'OrientationAxesOutputPort',true); % 定義平台

tx = phased.Transmitter('PeakPower',1,'Gain',25); % 發射機 (1 W, 25 dB 增益)
```

- 輸出 平台的方向軸 (orientation axes)
- `'PeakPower',1`: 發射功率 = 1 W，訊號在時域的峰值功率
- `'Gain',25`: 把輸出訊號放大 25 dB 之後才發射出去

### 發射端2
```matlab
tx_pos_2 = [0;1;1]; % 另一個發射機位置
tx_vel_2 = [0;0;0]; % 靜止
radar_tx_2 = phased.Platform('InitialPosition',tx_pos_2,'Velocity',tx_vel_2,...
    'OrientationAxesOutputPort',true);
```

---

### 目標位置與參數

```matlab
ped_pos = [3.2;-1;0]; % 行人初始位置 (距離 radar 3 m 左右)
ped_height = 1.8;     % 行人身高
ped_speed = ped_height/2; % 行走速度 = 高度/2 = 0.9 m/s
ped_heading = 0;      % 朝向 0 度 (x 方向)
```

### 建立行人目標模型

```matlab
ped = phased.BackscatterPedestrian( ...  
    'InitialPosition',ped_pos, ...       % 行人的初始位置座標 
    'InitialHeading',ped_heading, ...    % 行人面向的方向（角度，單位：度）
    'PropagationSpeed',c, ...            % 傳播速度，通常設成光速 3×10^8 m/s
    'OperatingFrequency',fc, ...         % 雷達工作頻率（載波頻率），例如 60 GHz
    'Height',ped_height, ...             % 行人身高，用來決定行人的散射中心 
    'WalkingSpeed',ped_speed);           % 行人行走速度，用來模擬 Doppler shift
```
MATLAB 的 `phased.BackscatterPedestrian` 是一個「複合散射體」模型：  
- 它會自動建立一組「代表人體不同部位的散射點」
- 當雷達訊號照射時，會計算每個散射點的反射，然後加總得到行人的總回波
- 因為設定了 速度與方向，所以回波會帶有多普勒效應

---

### 通道建模
```matlab
chan_ped = phased.FreeSpace( ...
    'PropagationSpeed', c, ...
    'OperatingFrequency', fc, ...
    'TwoWayPropagation', true, ...
    'SampleRate', fs);
```

### 接收端
```matlab
rx = phased.ReceiverPreamp('Gain',25,'NoiseFigure',10); % 接收機 (25 dB 增益, NF=10 dB)，雜訊這邊到時候可以設為 0
```

### 預先分配接收信號矩陣
```matlab
xr = complex(zeros(round(fs*tm),npulse));
xr_ped_perfect = complex(zeros(round(fs*tm),npulse));

```

---

### 雷達模擬主迴圈

```matlab
for m = 1:npulse

    [pos_tx, vel_tx, ax_ego] = radar_tx(Tsamp); % tx position, velocity, and angle
    [pos_tx_2, vel_tx_2, ax_ego_2] = radar_tx_2(Tsamp);

    % 行人位置更新
    [pos_ped, vel_ped, ax_ped] = move(ped,Tsamp,ped_heading); % the person moves
    % 每隔 Tsamp 秒就更新一次行人的位置  
    
    [~,angrt_ped] = rangeangle(pos_tx,pos_ped(:,1),ax_ped(:,:,1)); % compute the reflection angle
    angrt_ped = repmat(angrt_ped, [1,16]); % reshape

    % 產生發射信號
    x = tx(wav()); % generate the tx signal
    
    % person
    xt_ped = chan_ped(repmat(x,1,size(pos_ped,2)),pos_tx,pos_ped,vel_tx,vel_ped); % channel to the person
    xt_ped = reflect(ped,xt_ped,angrt_ped); % reflection from the person
    % xr_ped(:,m) = rx(xt_ped + clutter(1:round(fs*tm))); % One can add clutter here. For example, clutter in a conference room can be modeled according to 802.11ay, which can be simulated by the WLAN toolbox of Matlab. 
    xr_ped_perfect(:,m) = rx(xt_ped); % perfect case without the environment
end
```

- `[pos_ped, vel_ped, ax_ped] = move(ped,Tsamp,ped_heading);`
    - `move`: 根據行人的速度、heading，每隔 `Tsamp`更新一次用戶的位置

- `[~,angrt_ped] = rangeangle(pos_tx,pos_ped(:,1),ax_ped(:,:,1));`
    - 「雷達看行人」的方向角
    <img width="740" height="556" alt="image" src="https://github.com/user-attachments/assets/767b22f8-0274-42d4-b446-eb046449c03a" />

- `angrt_ped = repmat(angrt_ped, [1,16]);`
