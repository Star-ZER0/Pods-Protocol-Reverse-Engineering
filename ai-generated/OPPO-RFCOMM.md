# zerOBuds RFCOMM 协议文档

## 数据包格式

### 整体结构

```
[Header] [TotalLen (LEB128)] [Reserved 2B] [Cmd 2B LE] [Seq 1B] [PayLen 2B LE] [Payload N字节]
```

| 字段     | 长度      | 说明                                                    |
|----------|-----------|---------------------------------------------------------|
| Header   | 1 字节    | 固定 `0xAA`                                             |
| TotalLen | 1+ 字节   | LEB128 编码，值 = 后续所有字段的总字节数 (= 7 + PayLen) |
| Reserved | 2 字节    | 固定 `0x0000`                                           |
| Cmd      | 2 字节    | 命令码，小端序 (Low Byte 在前)                          |
| Seq      | 1 字节    | 序列号 (`0x01`-`0xFE`: 主机, `0xFF`: 耳机广播)         |
| PayLen   | 2 字节    | Payload 长度，小端序                                    |
| Payload  | N 字节    | 负载数据                                                |

### TotalLen LEB128 编码

TotalLen 字段使用改进的 LEB128 编码，长度可变：

**编码规则：**
- 当 TotalLen < 128 时，直接用 1 字节表示
- 当 TotalLen >= 128 时，先将值减 1，再按标准 LEB128 编码（小端序，每字节低 7 位存放数据，最高位为继续标志）

**解码规则：**
- 若首字节最高位 (bit7) 为 0，该字节值即为 TotalLen
- 若首字节最高位为 1，继续读取直到遇到最高位为 0 的字节，将所有字节的低 7 位按小端序拼接成整数，再加 1 还原

**编码示例：**

| TotalLen | 编码结果     | 说明                              |
|----------|-------------|-----------------------------------|
| 7        | `07`        | < 128，单字节                     |
| 8        | `08`        | < 128，单字节                     |
| 127      | `7F`        | < 128，单字节                     |
| 128      | `FF 00`     | >= 128，减1=127，LEB128 编码      |
| 200      | `C7 01`     | >= 128，减1=199，LEB128 编码      |
| 300      | `FF 02`     | >= 128，减1=299，LEB128 编码      |

### 数据包示例

**短包 (TotalLen < 128)：**

```
AA 08 00 00 27 04 3A 01 00 00
```
- Header: `AA`
- TotalLen: `08` (1 字节, = 7 + 1)
- Reserved: `00 00`
- Cmd: `27 04` → 0x0427 (SetVolume)
- Seq: `3A`
- PayLen: `01 00` → 1
- Payload: `00` (volume = max)

**长包 (TotalLen >= 128)：**

```
AA FF 00 00 04 02 FF 2C 00 06 02 ...
```
- Header: `AA`
- TotalLen: `FF 00` → LEB128 解码: 0x7F | (0x00 << 7) = 127, 加1 = 128
- Reserved: `00 00`
- Cmd: `04 02` → 0x0204 (Notification)
- Seq: `FF`
- PayLen: `2C 00` → 44
- Payload: `06 02 ...`

### F4 文本 Payload

当 Payload 首字节为 `0xF4` 时，后续内容为 UTF-8 编码文本（通常为 JSON）。

---

## 命令码总览

### 方向判定规则

Cmd 为 16 位小端序整数 `[LowByte][HighByte]`。

**方向判定 (bit 15)：**

当最高位 (bit 15) 为 1 时（即 Cmd >= 0x8000），表示该包由耳机发出（响应或广播）；当最高位为 0 时，表示该包由主机发出（请求或设置）。

```
主机 → 耳机: Cmd < 0x8000  (bit15 = 0)
耳机 → 主机: Cmd >= 0x8000 (bit15 = 1)
```

> **例外：** `0x0204` (Notification) 是耳机主动推送的通知，其 Cmd < 0x8000，属于特殊命令码。

**类别判定 (LowByte)：**

Cmd 的 LowByte（第一字节）标识命令类别：

| LowByte | 类别     | 说明                           |
|---------|----------|--------------------------------|
| 0x01    | Query    | 查询类命令，请求耳机状态       |
| 0x02    | Broadcast| 广播类命令，订阅/推送通知      |
| 0x04    | Setting  | 设置类命令，修改耳机配置       |

响应命令码的 LowByte 与请求相同，HighByte 最高位置 1（即 +0x80）。

```
Query:    0x01XX → 0x81XX
Broadcast: 0x02XX → 0x82XX
Setting:  0x04XX → 0x84XX
```

### 命令码对照

#### Query 类 (0x01XX / 0x81XX)

| 命令码   | 方向 | 名称                | 说明              |
|----------|------|---------------------|-------------------|
| 0x0100   | →    | Handshake           | 初始握手          |
| 0x8100   | ←    | HandshakeResponse   | 握手响应          |
| 0x0106   | →    | QueryBattery        | 查询电量          |
| 0x8106   | ←    | BatteryResponse     | 电量查询响应      |
| 0x010C   | →    | QueryAnc            | 查询 ANC 模式     |
| 0x810C   | ←    | AncResponse         | ANC 查询响应      |
| 0x010D   | →    | QueryStatus         | 查询批量状态      |
| 0x810D   | ←    | QueryStatusResponse | 批量状态响应      |
| 0x0130   | →    | QueryVolume         | 查询音量          |
| 0x8130   | ←    | VolumeResponse      | 音量查询响应      |

#### Broadcast 类 (0x02XX / 0x82XX)

| 命令码   | 方向 | 名称                     | 说明              |
|----------|------|--------------------------|-------------------|
| 0x0200   | →    | QueryBroadcastCodes      | 请求广播码列表    |
| 0x8200   | ←    | BroadcastCodesResponse   | 广播码列表响应    |
| 0x0204   | ←*   | Notification             | 耳机主动通知      |
| 0x0205   | →    | SubscribeBroadcast       | 订阅广播码        |
| 0x8205   | ←    | SubscribeBroadcastResponse | 订阅响应        |

#### Setting 类 (0x04XX / 0x84XX)

| 命令码   | 方向 | 名称                     | 说明              |
|----------|------|--------------------------|-------------------|
| 0x0403   | →    | SetSetting               | 设置开关项        |
| 0x8403   | ←    | SetSettingResponse       | 设置响应          |
| 0x0404   | →    | SetAnc                   | 设置 ANC 模式     |
| 0x8404   | ←    | SetAncResponse           | ANC 设置响应      |
| 0x0422   | →    | SetSpatialAudio          | 设置空间音频      |
| 0x8422   | ←    | SetSpatialAudioResponse  | 空间音频设置响应  |
| 0x0427   | →    | SetVolume                | 设置音量          |
| 0x8427   | ←    | SetVolumeResponse        | 音量设置响应      |

#### 通知类 (0x05XX)

| 命令码   | 方向 | 名称                     | 说明              |
|----------|------|--------------------------|-------------------|
| 0x0510   | ←    | SpatialAudioNotification | 空间音频模式通知  |

> ←* 表示虽由耳机发出，但 Cmd < 0x8000，属于例外。

---

## 初始化过程

连接 RFCOMM 频道 15 后，需按以下顺序完成初始化握手，之后才能正常收发命令。

### 1. 初始握手 (0x0100 / 0x8100)

**发送：**

```
AA 07 00 00 00 01 [Seq] 00 00
```

- Cmd: `0x0100`
- Payload: 空 (PayLen = 0)

**期望响应 (0x8100)：**

```
AA 10 00 00 00 81 [Seq] 09 00 FF 77 5A EA E7 0E A0 07
```

- Payload: 9 字节，内容含义未知

**示例：**
```
→ AA 07 00 00 00 01 00 00 00
← AA 10 00 00 00 81 00 09 00 FF 77 5A EA E7 0E A0 07
```

---

### 2. 请求广播码列表 (0x0200 / 0x8200)

握手成功后，请求耳机支持的广播码列表。

**发送：**

```
AA 07 00 00 00 02 [Seq] 00 00
```

- Cmd: `0x0200`
- Payload: 空 (PayLen = 0)

**期望响应 (0x8200)：**

```
AA 12 00 00 00 82 [Seq] 0B 00 00 09 [Code1] [Code2] ... [CodeN]
```

- Payload: `[0x00] [Count] [Code1] [Code2] ... [CodeN]`
- Count: 广播码数量
- Code: 各广播码

**已知广播码对照：**

| 广播码 | 名称         | 说明               |
|--------|--------------|--------------------|
| 0x01   | Battery      | 电量主动广播       |
| 0x02   | WearStatus   | 佩戴状态主动广播   |
| 0x03   | AncMode      | 降噪模式主动广播   |
| 0xF1   | Action        | 耳机操作相关信息   |
| 0xF2   | JsonDebug    | JSON 相关调试信息  |

**示例：**
```
→ AA 07 00 00 00 02 00 00 00
← AA 12 00 00 00 82 00 0B 00 00 09 01 02 03 04 08 0B F1 F2 F3
```
解析: 9 个广播码 — 0x01, 0x02, 0x03, 0x04, 0x08, 0x0B, 0xF1, 0xF2, 0xF3

---

### 3. 订阅广播码 (0x0205 / 0x8205)

收到广播码列表后，发送订阅请求以激活所需的广播通知。

**发送：**

```
AA 11 00 00 05 02 [Seq] 0A 00 [Count] [Code1] [Code2] ... [CodeN]
```

- Cmd: `0x0205`
- Payload: `[Count] [Code1] [Code2] ... [CodeN]`
- 通常订阅所有广播码

**期望响应 (0x8205)：**

```
AA 1B 00 00 05 82 [Seq] 14 00 [Unknown 1B] [Count] [Code1] [Status1] [Code2] [Status2] ...
```

- Payload: `[0x01] [Count] [Code1] [Status1] [Code2] [Status2] ...`
- Status: `0x00`=已启用, `0x02`=该广播类别不存在

**示例：**
```
→ AA 11 00 00 05 02 00 0A 00 09 01 02 03 04 08 0B F1 F2 F3
← AA 1B 00 00 05 82 00 14 00 01 09 01 00 02 00 03 00 04 00 08 00 0B 00 F1 00 F2 00 F3 02
```
解析: 广播码 0x01-0xF2 状态为 0x00 (已启用), 0xF3 状态为 0x02 (不存在)

---

## 请求/响应详解

### 1. 查询电量 (0x0106 / 0x8106)

**发送：**

```
AA 07 00 00 06 01 [Seq] 00 00
```

- Cmd: `0x0106`
- Payload: 空 (PayLen = 0)

**期望响应 (0x8106)：**

```
AA 0B 00 00 06 81 [Seq] 04 00 [Idx1] [Val1] [Idx2] [Val2] [Idx3] [Val3]
```

- Payload 为多组 `(Index, RawValue)` 对：
  - Index: `01`=左耳, `02`=右耳, `03`=充电盒
  - RawValue: bit0-6 = 电量百分比, bit7 = 充电中标志

**示例：**
```
→ AA 07 00 00 06 01 3A 00 00
← AA 0B 00 00 06 81 3A 04 00 01 64 02 64 03 50
```
解析: L=100%, R=100%, Case=80%

---

### 2. 查询 ANC 模式 (0x010C / 0x810C)

**发送：**

```
AA 09 00 00 0C 01 [Seq] 02 00 01 01
```

- Cmd: `0x010C`
- Payload: `01 01`

**期望响应 (0x810C)：**

Payload 中查找 `01 01` 前缀，后跟 2 字节模式值：

| val1 | val2 | 模式                |
|------|------|---------------------|
| 0x08 | 0x00 | Off                 |
| 0x10 | 0x00 | NoiseCancellation   |
| 0x00 | 0x01 | Transparency        |
| 0x00 | 0x08 | Adaptive            |

**示例：**
```
→ AA 09 00 00 0C 01 3A 02 00 01 01
← AA 0C 00 00 0C 81 3A 05 00 00 01 01 00 08
```
解析: ANC Mode = Adaptive

---

### 3. 设置 ANC 模式 (0x0404 / 0x8404)

**发送：**

| 模式              | Payload         | 完整包示例                           |
|-------------------|-----------------|--------------------------------------|
| Off               | `01 01 01`         | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 01` |
| NoiseCancellation | `01 01 02`         | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 02` |
| NoiseCancellation (Smart) | `01 01 80` | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 80` |
| NoiseCancellation (Light) | `01 01 40` | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 40` |
| NoiseCancellation (Medium)| `01 01 20` | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 20` |
| NoiseCancellation (Deep)  | `01 01 10` | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 10` |
| Transparency      | `01 01 04`         | `AA 0A 00 00 04 04 [Seq] 03 00 01 01 04` |
| Adaptive          | `01 01 00 08`      | `AA 0B 00 00 04 04 [Seq] 04 00 01 01 00 08` |
| VocalEnhancement (On) | `01 01 00 02`  | `AA 0B 00 00 04 04 [Seq] 04 00 01 01 00 02` |
| VocalEnhancement (Off) | `01 01 00 01` | `AA 0B 00 00 04 04 [Seq] 04 00 01 01 00 01` |

降噪等级字节：

| 等级   | 值   |
|--------|------|
| Smart  | 0x80 |
| Light  | 0x40 |
| Medium | 0x20 |
| Deep   | 0x10 |

**期望响应 (0x8404)：**

与 0x810C 格式相同，包含当前 ANC 模式信息。

**示例：**
```
→ AA 0B 00 00 04 04 3A 04 00 01 01 00 08
← AA 08 00 00 04 84 3A 01 00 00
```

---

### 4. 查询批量状态 (0x010D / 0x810D)

**发送：**

```
AA 0C 00 00 0D 01 [Seq] [PayloadLen] [PayloadLen] [Count] [ParamId1] [ParamId2] ...
```

- Payload: `[Count] [ParamId1] [ParamId2] ...`
- Count: 要查询的参数数量
- ParamId: 要查询的参数 ID 列表

**期望响应 (0x810D)：**

```
AA 13 00 00 0D 81 [Seq] [PayloadLen] [PayloadLen] [Count] [ParamId1] [Val1] [ParamId2] [Val2] ...
```

- Payload: `[Count] [ParamId1] [Val1] [ParamId2] [Val2] ...`

**参数 ID：**

| ParamId | 名称             | 值                |
|---------|------------------|-------------------|
| 0x04    | AutoPlayPause    | 0x00=Off, 0x01=On |
| 0x06    | LowLatencyMode   | 0x00=Off, 0x01=On |
| 0x11    | DualDevice       | 0x00=Off, 0x01=On |
| 0x18    | HiRes            | 0x00=Off, 0x01=On |
| 0x1B    | SpatialSound     | 0x00=Off, 0x01=On |
| 0x28    | GameMode         | 0x00=Off, 0x01=On |
| 0x37    | WindowsSwiftPair | 0x00=Off, 0x01=On |
逆向结果：
| Dec | Hex | 官方条件 / 含义 |
| --- | --- | --- |
| 5 | `0x05` | 固定加入；日志里出现 `updateFeatureSwitchToStatus IGNORE 5` |
| 4 | `0x04` | `wearDetection`，佩戴检测 |
| 11 | `0x0B` | `hearingEnhancement` / `hearingEnhancementNew` |
| 12 | `0x0C` | `personalNoise` |
| 13 | `0x0D` | `clickTakePic` / `clickTakePicNew` |
| 15 | `0x0F` | `zenMode` |
| 17 | `0x11` | `multiDevicesConnect` |
| 9 | `0x09` | `vocalEnhance` |
| 19 | `0x13` | `headSetSoundRecord` |
| 24 | `0x18` | `highToneQuality` |
| 23 | `0x17` | `longPowerMode` |
| 21 | `0x15` | `smartCall` |
| 22 | `0x16` | `deviceLostRemind` |
| 20 | `0x14` | `voiceWake` |
| 25 | `0x19` | `voiceCommand` |
| 6 | `0x06` | `gameMode` |
| 27 | `0x1B` | `spatialTypes` |
| 29 | `0x1D` | `bassEngineSupport` |
| 28 | `0x1C` | `controlAutoVolumeSupport` |
| 30 | `0x1E` | `collectLogs` |
| 33 | `0x21` | `gameEqPkgList` |
| 31 | `0x1F` | 平台条件 `C0282d.m617e()`，官方代码未给出直观字段名 |
| 34 | `0x22` | `spineHealth` 相关 |
| 35 | `0x23` | `spineHealth` 相关 |
| 36 | `0x24` | `spineHealth` 相关 |
| 39 | `0x27` | `gameSoundList` 或支持 `0x0423` |
| 40 | `0x28` | `gameSoundList` 或支持 `0x0423` |
| 48 | `0x30` | `adaptiveVolume` |
| 49 | `0x31` | `adaptiveEar` |
| 50 | `0x32` | `speechPerception` |
| 52 | `0x34` | `meetingAssistant` |
| 53 | `0x35` | `longPressVolume` |
| 55 | `0x37` | `swiftPair` |
| 56 | `0x38` | `hearingOptimize` |
| 57 | `0x39` | `incomingCallControl` |

**示例：**
```
→ AA 0E 00 00 0D 01 3A 08 00 07 04 06 11 18 1B 28 37
← AA 17 00 00 0D 81 3A 10 00 00 07 04 01 06 00 11 01 18 00 1B 00 28 00 37 00
```
解析: AutoPlayPause=On, GameMode1=Off, DualDevice=On, HiRes=Off, SpatialSound=Off, GameMode2=Off, WindowsSwiftPair=Off

---

### 5. 设置开关项 (0x0403 / 0x8403)

**发送：**

```
AA 09 00 00 03 04 [Seq] 02 00 [ParamId] [Value]
```

- ParamId: 参数 ID (同批量状态)
- Value: `0x00`=Off, `0x01`=On

**期望响应 (0x8403)：**

```
AA 08 00 00 03 84 [Seq] 01 00 [Status]
```

- Status: `0x00`=成功, `0x02`=失败

**示例 (开启游戏模式成功)：**
```
→ AA 09 00 00 03 04 3A 02 00 06 01
← AA 08 00 00 03 84 3A 01 00 00
```

**示例 (开启游戏模式失败)：**
```
→ AA 09 00 00 03 04 3A 02 00 06 01
← AA 08 00 00 03 84 3A 01 00 02
```

---

### 6. 查询音量 (0x0130 / 0x8130)

**发送：**

```
AA 09 00 00 30 01 [Seq] 02 00 00 00
```

- Payload: `00 00`

**期望响应 (0x8130)：**

```
AA 09 00 00 30 81 [Seq] 02 00 [Status] [VolByte]
```

- Status: 状态码 (0x00=成功)
- VolByte: 音量值

| VolByte | 含义   |
|---------|--------|
| 0x00    | max    |
| 0x01    | 0      |
| 0x02    | 1      |
| ...     | ...    |
| 0x0A    | 9      |

**示例：**
```
→ AA 09 00 00 30 01 3A 02 00 00 00
← AA 09 00 00 30 81 3A 02 00 00 04
```
解析: Volume = 3

---

### 7. 设置音量 (0x0427 / 0x8427)

**发送：**

```
AA 08 00 00 27 04 [Seq] 01 00 [VolByte]
```

- Payload: `[VolByte]` (1 字节)
- VolByte: 同查询响应中的音量值

| VolByte | 含义   |
|---------|--------|
| 0x00    | max    |
| 0x01    | 0      |
| 0x02    | 1      |
| ...     | ...    |
| 0x0A    | 9      |

**期望响应 (0x8427)：**

```
AA 09 00 00 27 84 [Seq] 02 00 [Status] [VolByte]
```

**示例 (设置音量 3)：**
```
→ AA 08 00 00 27 04 3A 01 00 04
← AA 09 00 00 27 84 3A 02 00 00 04
```

**示例 (设置音量 max)：**
```
→ AA 08 00 00 27 04 3A 01 00 00
← AA 09 00 00 27 84 3A 02 00 00 00
```

---

### 8. 设置空间音频 (0x0422 / 0x8422)

**发送：**

```
AA 08 00 00 22 04 [Seq] 01 00 [Mode]
```

- Payload: `[0x01, 0x00, Mode]` (3 字节)
- Mode:

| Mode | 含义   |
|------|--------|
| 0x00 | 关闭   |
| 0x01 | 模式1  |
| 0x02 | 模式2  |

| Mode | 完整包示例                              |
|------|-----------------------------------------|
| 0x00 | `AA 08 00 00 22 04 [Seq] 01 00 00` |
| 0x01 | `AA 08 00 00 22 04 [Seq] 01 00 01` |
| 0x02 | `AA 08 00 00 22 04 [Seq] 01 00 02` |

**期望响应 (0x8422)：**

```
AA 08 00 00 22 84 [Seq] 01 00 [Mode]
```

- Status: 状态码 (0x00=成功)
- Mode: 当前空间音频模式

**示例 (设置空间音频模式 1)：**
```
→ AA 08 00 00 22 04 3A 01 00 01
← AA 08 00 00 22 84 3A 01 00 01
```

---

## 耳机主动通知 (0x0204)

耳机通过 Cmd `0x0204` 主动推送通知，Seq 通常为 `0xFF`。Payload 第一字节为 ReportType，用于区分通知类别。

### ReportType 总览

| ReportType | 名称             | 说明         |
|------------|------------------|--------------|
| 0x01       | Battery          | 电量通知     |
| 0x02       | BudsStatus       | 耳机状态（佩戴状态）     |
| 0x03       | AncModeChange    | ANC 模式切换通知 |
| 0x05       | GameMode         | 游戏模式通知 |
| 0x06       | ConnectedDevices | 连接设备通知 |
| 0xF1       | Action           | 触控手势通知 |
| 0xF4       | JsonDebug        | JSON 调试信息 |

---

### 0x01 电量通知

```
AA 0D 00 00 04 02 FF 08 00 01 [Count] [Idx1] [Val1] [Idx2] [Val2] ...
```

- Payload: `[0x01] [Count] [Idx1] [Val1] ...`
- 格式与 0x8106 电量响应相同

**示例：**
```
AA 0D 00 00 04 02 FF 08 00 01 03 01 64 02 64 03 50
```
解析: Battery L=100% R=100% Case=80%

---

### 0x02 佩戴状态通知

```
AA 0F 00 00 04 02 FF 08 00 02 [Count] [Id1] [Status1] [Id2] [Status2] ...
```

- Id: `01`=左耳, `02`=右耳, `03`=充电盒
- Status:

| Status | 含义         |
|--------|--------------|
| 0x00   | Disconnected |
| 0x04   | InCase       |
| 0x05   | Removed      |
| 0x07   | Wearing      |

**示例：**
```
AA 0F 00 00 04 02 FF 08 00 02 03 01 07 02 07 03 04
```
解析: WearStatus L:Wearing R:Wearing Case:InCase

---

### 0x03 ANC 模式切换通知

```
AA 0C 00 00 04 02 FF 05 00 03 01 01 [Val1] [Val2]
```
- Val1/Val2 解析：

| Val1 | Val2 | 模式              |
|------|------|-------------------|
| 0x08 | 0x00 | Off               |
| 0xNN | 0x00 | NoiseCancellation (Val1 为降噪等级) |
| 0x00 | 0x01 | Transparency      |
| 0x00 | 0x02 | Transparency(VocalEnhancement)  |
| 0x00 | 0x08 | Adaptive          |

**示例：**
```
AA 0C 00 00 04 02 FF 05 00 03 04 01 10 00
```
解析: ANC Smart→NoiseCancellation (Deep)

---

### 0x04 ANC 模式通知

Payload 包含 `01 01` 前缀 + 模式值（与 0x810C 响应格式相同）。

```
AA 0F 00 00 04 02 FF 08 00 04 03 01 10 02 07 03 04
```

---

### 0x05 游戏模式通知

```
AA 09 00 00 04 02 FF 02 00 05 [State]
```

- State: `0x00`=Off, `0x01`=On

---

### 0x06 连接设备通知

```
AA [TotalLen] 00 00 04 02 FF [PayLen] 00 06 [Count] [Device1] [Device2] ...
```

> 注意：连接设备通知的 TotalLen 通常 >= 128，使用 LEB128 多字节编码。

每个设备结构：

```
[MAC 6B LE] [ProfileFlags 1B] [ConnectionState 1B] [IsActive 1B] [NameLen 1B] [Name N字节 UTF-8]
```

- MAC: 6 字节小端序
- ConnectionState: `0x00`=Disconnected, `0x02`=Connected
- IsActive: `0x01`=当前活跃设备

**示例 (TotalLen < 128)：**
```
AA 33 00 00 04 02 FF 2C 00 06 02 B2 2B C5 5A 13 3C 0C 02 00 09 52 65 64 6D 69 20 4B 36 30 76 95 50 0E 41 00 10 02 01 0D 4C 45 47 49 4F 4E 2D 52 39 30 30 30 50
```
解析: 2 台设备 — Redmi K60 (Connected), LEGION-R9000P (Connected ACTIVE)

---

### 空间音频模式通知 (0x0510)

```
AA 08 00 00 10 05 [Seq] 01 00 [Mode]
```

- Mode:

| Mode | 含义         |
|------|--------------|
| 0x00 | 关闭         |
| 0x01 | 固定头部追踪 |
| 0x02 | 头部追踪     |

**示例 (关闭)：**
```
AA 08 00 00 10 05 FF 01 00 00
```

**示例 (固定头部追踪)：**
```
AA 08 00 00 10 05 FF 01 00 01
```

**示例 (头部追踪)：**
```
AA 08 00 00 10 05 FF 01 00 02
```

---

### 0xF1 触控手势通知

```
AA 0D 00 00 04 02 FF 06 00 F1 [Side] [00] [Gesture] [Action] [Context]
```

- Side: `0x01`=左耳, `0x02`=右耳
- Gesture:

| 值   | 名称          |
|------|---------------|
| 0x00 | ACK           |
| 0x01 | SingleTap     |
| 0x02 | DoubleTap     |
| 0x03 | TripleTap     |
| 0x04 | LongPress     |
| 0x06 | ExtraLongPress|
| 0x07 | SwipeUp       |
| 0x08 | SwipeDown     |

- Action:

| 值   | 名称            |
|------|-----------------|
| 0x00 | None            |
| 0x01 | PlayPause       |
| 0x03 | VoiceAssistant  |
| 0x05 | Previous        |
| 0x06 | Next            |
| 0x08 | ToggleANC       |
| 0x0B | VolumeUp        |
| 0x0C | VolumeDown      |
| 0x0E | AnswerCall      |
| 0x0F | HangUpCall      |
| 0x11 | GameMode        |

- Context:

| 值   | 名称   |
|------|--------|
| 0x01 | Call   |
| 0x02 | Media  |
| 0x03 | Idle   |

**示例：**
```
AA 0D 00 00 04 02 FF 06 00 F1 01 01 00 00 02
```
解析: Action L ACK None [Media]

---

## Seq 规则

| Seq 值    | 含义                          |
|-----------|-------------------------------|
| 0x01-0xFE | 主机发送的序列号              |
| 0xFF      | 耳机主动广播 (非响应)        |

- 非交互模式：使用随机 Seq (0x01-0xFE)
- 交互模式 (终端)：Seq 从 0x01 递增，到 0xFE 后回到 0x01
- 耳机响应的 Seq 与请求的 Seq 对应
- 耳机主动推送的通知 Seq 为 0xFF
