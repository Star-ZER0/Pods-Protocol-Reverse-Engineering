# OPPO 耳机蓝牙协议文档

## 传输方式

- **SPP UUID**: `0000079A-D102-11E1-9B23-00025B00A5A5`

## 数据包格式 (小端序)

```
Header(1B) + TotalLen(LEB128) + Res(2B) + Cmd(2B LE) + Seq(1B) + PayLen(2B LE) + Payload(NB)
  0xAA       变长               00 00     命令码        序列号    载荷长度        载荷数据
```

### LEB128 编码规则

- TotalLen < 128: 单字节编码，值即为 TotalLen
- TotalLen >= 128: 先减1，再标准 LEB128 编码（每字节低7位为数据，最高位为续位标志）

### 字段说明

| 字段 | 长度 | 说明 |
|------|------|------|
| Header | 1B | 固定 `0xAA` |
| TotalLen | LEB128 | Res(2) + Cmd(2) + Seq(1) + PayLen(2) + Payload 的总长度 |
| Res | 2B | 保留字段，固定 `0x00 0x00` |
| Cmd | 2B LE | 命令码，低字节在前 |
| Seq | 1B | 序列号，范围 0x01-0xFE，循环递增 |
| PayLen | 2B LE | Payload 长度，低字节在前 |
| Payload | NB | 载荷数据 |

---

## 命令码总表

### 查询类 (0x01xx)

| 命令 | 代码 | 官方方法 | Payload | 说明 |
|------|------|----------|---------|------|
| 握手 | `0x0100` | `getRemoteCapability` | 空 | 连接后首次握手 |
| 查询 MTU | `0x0101` | `getRemoteMTU` | `00 02` (512 LE) | 查询远端 MTU |
| 查询 VID | `0x0102` | `getRemoteVID` | 空 | 查询厂商 ID |
| 查询 PID | `0x0103` | `getRemotePID` | 空 | 查询产品 ID |
| 查询版本 | `0x0105` | `getRemoteVersion` | 空 | 查询远端版本 |
| 查询电量 | `0x0106` | `getBatteryLevel` | 空 | 查询耳机/充电盒电量 |
| 查询触控按键 | `0x0108` | `getKeyFunction` | `<count> <deviceType...>` | 常请求 `02/03/01` |
| 查询当前降噪 | `0x010C` | `getCurrentNoiseReductionMode` | `01 01` | 查询降噪模式 |
| 查询降噪切换项 | `0x010C` | `getNoiseReductionSwitchMode` | `02 01` 或 `02 03`、`02 04` | 查询可切换降噪模式 |
| 查询智能降噪 | `0x010C` | `getIntelligentNoiseReductionMode` | `04 01` | 查询智能降噪 |
| 查询批量状态 | `0x010D` | `getFeatureSwitchStatus` | `<count> <featureIds...>` | 批量查询功能开关 |
| 查询 EQ 模式 | `0x010F` | `getCurrentEqualizerMode` | 空 | 查询当前 EQ |
| 查询编解码器 | `0x0114` | `getCurrentCodecType` | 空 | 查询当前编解码器类型 |
| 查询耳恢复数据 | `0x0118` | `getEarRestoreData` | 空 | |
| 查询三角信息 | `0x011C` | `getTriangleInfo` | 空 | |
| 查询耳扫描数据 | `0x011E` | `getEarScanData` | 空 | |
| 查询耳音调数据 | `0x0121` | `getEarToneData` | 空 | |
| 查询所有 EQ | `0x0122` | `getAllEqInfo` | 空 | 查询全部 EQ 信息 |
| 查询账户密钥 | `0x0125` | `getAccountKey` | 空 | |
| 查询空间音频类型 | `0x012A` | `getHeadsetSpatialType` | 空 | |
| 查询游戏声音 | `0x012B` | `getGameSoundInfo` | 空 | |
| 查询 AI 摘要类型 | `0x012E` | `getAISummaryType` | 空 | |
| 多 SPP 命令 | `0x012F` | `multiSPPCommandInfo` | 空 | |
| 查询提示音音量 | `0x0130` | | `00 00` | 查询提示音音量等级 |

查询类响应 (0x81xx):

| 命令 | 代码 | Payload | 说明 |
|------|------|---------|------|
| 握手响应 | `0x8100` | | 握手确认 |
| 电量响应 | `0x8106` | `[Index] [RawValue]...` | 返回电量信息 |
| ANC 响应 | `0x810C` | `[01 01 Val1 Val2]` | 返回降噪状态 |
| 批量状态响应 | `0x810D` | `[Unknown] [Count] [ParamId] [Value]...` | 返回各参数状态 |
| 音量响应 | `0x8130` | `[Unknown] [Level]` | 返回音量等级 |

### 广播类 (0x02xx)

| 命令 | 代码 | 方向 | 说明 |
|------|------|------|------|
| 请求广播码 | `0x0200` | → | 获取可用广播码列表 |
| 通知 | `0x0204` | ← | 设备主动推送状态变化 |
| 订阅广播 | `0x0205` | → | 订阅指定广播码 |

广播类响应 (0x82xx):

| 命令 | 代码 | 方向 | 说明 |
|------|------|------|------|
| 广播码响应 | `0x8200` | ← | 返回广播码列表 |
| 订阅响应 | `0x8205` | ← | 订阅确认 |

### 设置类 (0x04xx)

| 命令 | 代码 | 官方方法 | Payload | 说明 |
|------|------|----------|---------|------|
| 查找耳机 | `0x0400` | `BtOperate.findMode` | `[01]` 开启 / `[00]` 关闭 | |
| 设置触控按键 | `0x0401` | `BtOperate.setKeyFunction` | `<count> [deviceType, button, buttonAction, function]...` | |
| 功能开关 | `0x0403` | `BtOperate.setFeatureSwitch` | `[featureId, status]` | `status=01/00` |
| 设置当前降噪 | `0x0404` | `BtOperate.setCurrentNoiseMode` | `type=1`: `01 01 <modeBitmask...>`; `type=2`: `01 02 <level>` | |
| 设置降噪信息 | `0x0404` | `BtOperate.setNoiseReductionInfo` | `[action, type, valueLE(1..4 bytes)]` | |
| 贴合度检测开关 | `0x0405` | | `[status]` | |
| 设置 EQ 模式 | `0x0406` | `BtOperate.setEqMode` | `[eqModeType]` | |
| 关联设备 | `0x0408` | `BtOperate.setAssociatedDevice` | `[hostType][hostMac6][count][type][mac6][state]...` | 多设备相关 |
| 听感增强检测 | `0x040D` / `0x040E` | `BtOperate.hearingEnhancement` | 按 action/type 组合 | |
| 系统相机状态 | `0x040F` | `BtOperate.setCameraStatus` | `[status]` | |
| Zen mode 校验 | `0x0410` | `BtOperate.setZenModeVerify` | `ZenModeFileVertifyInformation.getData()` | |
| 耳机恢复数据 | `0x0411` | `BtOperate.setEarRestoreData` | `<count> <EarRestoreDataInfo...>` | |
| 个性化降噪 | `0x0412` | `BtOperate.setPersonalizedNoise` | `[value]` | |
| 自由对话恢复时间 | `0x0414` | `BtOperate.setFreeTalkRecoveryTime` | `[type]` | |
| 自定义 EQ | `0x0418` | `BtOperate.setCustomEq` | `[action,min,max,eqId,nameLen,name...,bandCount,(freqLE2,db)...]` | |
| 游戏 EQ 状态 | `0x0420` | | service 传 `game_type, game_status` | |
| 脊椎范围检测 | `0x0421` | `BtOperate.setSpineRange` | `[status, step]` | |
| 设置空间音频 | `0x0422` | `BtOperate.setSpatialAudioType` | `[type]` | |
| 游戏声音 | `0x0423` | `GameSoundInfo` | service 传 `game_type, game_status` | |
| LE Audio | `0x0424` | | service 传 `type, value` | |
| 设置提示音音量 | `0x0427` | | `[Level]` | |
| Debug 开关 | `0x0F00` | `BtOperate.setDebugSwitch` | `[i3, i4, i10]` | |

设置类响应 (0x84xx):

| 命令 | 代码 | Payload | 说明 |
|------|------|---------|------|
| 功能开关响应 | `0x8403` | `[featureId] [01] [00/02]` | 0x00=成功, 0x02=失败 |
| ANC 设置响应 | `0x8404` | | 设置确认 |
| 空间音频响应 | `0x8422` | `[00] [Type]` | 设置确认 |
| 音量设置响应 | `0x8427` | | 设置确认 |

### 通知类 (0x05xx)

| 命令 | 代码 | 方向 | 说明 |
|------|------|------|------|
| 空间音频通知 | `0x0510` | ← | 空间音频状态变化推送 |

---

## 功能开关参数 ID (FeatureSwitch)

| ParamId | 值 | 官方字段 | 说明 |
|---------|------|----------|------|
| | `0x04` | `wearDetection` | 佩戴检测 |
| | `0x05` | (固定加入) | 日志出现 `updateFeatureSwitchToStatus IGNORE 5` |
| | `0x06` | `gameMode` | 低延迟模式 |
| | `0x09` | `vocalEnhance` | 人声增强 |
| | `0x0B` | `hearingEnhancement` / `hearingEnhancementNew` | 听感增强 |
| | `0x0C` | `personalNoise` | 个性化降噪 |
| | `0x0D` | `clickTakePic` / `clickTakePicNew` | 点击拍照 |
| | `0x0F` | `zenMode` | 禅模式 |
| | `0x11` | `multiDevicesConnect` | 双设备连接 |
| | `0x13` | `headSetSoundRecord` | 耳机录音 |
| | `0x14` | `voiceWake` | 语音唤醒 |
| | `0x15` | `smartCall` | 智能通话 |
| | `0x16` | `deviceLostRemind` | 设备丢失提醒 |
| | `0x17` | `longPowerMode` | 长续航模式 |
| | `0x18` | `highToneQuality` | 高解析度音频 |
| | `0x19` | `voiceCommand` | 语音指令 |
| | `0x1B` | `spatialTypes` | 空间音效 |
| | `0x1C` | `controlAutoVolumeSupport` | 自动音量控制 |
| | `0x1D` | `bassEngineSupport` | 低音引擎 |
| | `0x1E` | `collectLogs` | 收集日志 |
| | `0x1F` | (平台条件) | |
| | `0x21` | `gameEqPkgList` | 游戏 EQ 包列表 |
| | `0x22` | `spineHealth` | 脊椎健康 |
| | `0x23` | `spineHealth` | 脊椎健康 |
| | `0x24` | `spineHealth` | 脊椎健康 |
| | `0x27` | `gameSoundList` | 游戏声音列表 (支持 0x0423) |
| | `0x28` | `gameSoundList` | 游戏模式 |
| | `0x30` | `adaptiveVolume` | 自适应音量 |
| | `0x31` | `adaptiveEar` | 自适应听感 |
| | `0x32` | `speechPerception` | 语音感知 |
| | `0x34` | `meetingAssistant` | 会议助手 |
| | `0x35` | `longPressVolume` | 长按音量 |
| | `0x37` | `swiftPair` | Windows 快速配对 |
| | `0x38` | `hearingOptimize` | 听力优化 |
| | `0x39` | `incomingCallControl` | 来电控制 |

---

## 通知类型 (0x0204 ReportType)

| ReportType | 值 | 说明 |
|------------|------|------|
| BATTERY | `0x01` | 电量变化通知 |
| WEAR_STATUS | `0x02` | 佩戴状态通知 |
| ANC_MODE_CHANGE | `0x03` | 降噪模式切换通知 |
| ANC_MODE | `0x04` | 降噪模式状态 |
| GAME_MODE | `0x05` | 游戏模式变化通知 |
| CONNECTED_DEVICES | `0x06` | 连接设备变化通知 |

---

## 协议详情

### 握手

```
→ AA 06 00 00 00 01 [Seq] 00 00
← AA 06 00 00 00 81 [Seq] 00 00
```

连接后发送握手包，收到响应后发送广播码查询和订阅。

### 电量

**查询:**
```
→ AA 06 00 00 06 01 [Seq] 00 00
```

**响应 (0x8106):**
```
Payload: [Index1] [RawValue1] [Index2] [RawValue2] [Index3] [RawValue3]
```

| Index | 说明 |
|-------|------|
| 0x01 | 左耳 |
| 0x02 | 右耳 |
| 0x03 | 充电盒 |

RawValue 编码:
- 低 7 位 (`& 0x7F`): 电量百分比 (0-100)
- 最高位 (`& 0x80`): 充电标志 (1=正在充电)

**通知 (0x0204, ReportType=0x01):**
```
Payload: [0x01] [Count] [Index1] [RawValue1] ...
```

### 降噪模式

**查询当前降噪 (0x010C, Payload=`01 01`):**
```
→ AA 08 00 00 0C 01 [Seq] 02 00 01 01
```

**响应 (0x810C):**
在 Payload 中查找 `01 01 [Val1] [Val2]` 模式：
根据如下cmd0x0204内容修改anc按钮实时更新状态的逻辑，去除原来的按钮状态刷新逻辑：ANC 模式切换通知  
 
 ``` 
 AA 0C 00 00 04 02 FF 05 00 03 01 01 [Val1] [Val2] 
 ``` 
 - Val1/Val2 解析： 
 
| Val1 | Val2 | 模式 |
|------|------|------|
| 0x08 | 0x00 | 关闭 |
| 0x02 | 0x00 | 降噪 (通用) |
| 0x80 | 0x00 | 降噪 (智能) |
| 0x40 | 0x00 | 降噪 (轻度) |
| 0x20 | 0x00 | 降噪 (中度) |
| 0x10 | 0x00 | 降噪 (深度) |
| 0x00 | 0x01 | 通透 (人声增强关) |
| 0x00 | 0x02 | 通透 (人声增强开) |
| 0x00 | 0x08 | 自适应 |

**查询降噪切换项 (0x010C, Payload=`02 01` 或 `02 03`、`02 04`):**
查询可切换的降噪模式列表。

**查询智能降噪 (0x010C, Payload=`04 01`):**
查询智能降噪模式状态。

**设置降噪模式 (0x0404):**
```
关闭:     Payload: 01 01 01
降噪:     Payload: 01 01 02
通透:     Payload: 01 01 04
自适应:   Payload: 01 01 00 08
```

**设置降噪等级 (0x0404):**
```
Payload: 01 01 [LevelValue]
智能: 0x80, 轻度: 0x40, 中度: 0x20, 深度: 0x10
```

**查询降噪等级 (0x010C):**
```
→ AA 09 00 00 0C 01 [Seq] 02 00 01 01
```

**设置人声增强 (0x0404):**
```
关闭: Payload: 01 01 00 01
开启: Payload: 01 01 00 02
```

### 批量状态

**查询 (0x010D):**
```
→ AA 0E 00 00 0D 01 [Seq] 07 04 06 11 18 1B 28 37
Payload: [Count=7] [ParamId1] [ParamId2] ... [ParamId7]
```

**响应 (0x810D):**
```
Payload: [Unknown 1B] [Count] [ParamId1] [Value1] [ParamId2] [Value2] ...
```

Value: `0x01` = 开启, `0x00` = 关闭

注意: 如果设备不支持某个 ParamId，响应中不会包含该参数（如 0x37 Windows 快速配对）。

### 设置开关项 (0x0403)

```
→ Payload: [ParamId] [Value]
Value: 0x01=开启, 0x00=关闭
```

**响应 (0x8403):**
```
← Payload: [ParamId] [01] [00/02]
0x00 = 成功, 0x02 = 失败 (需回滚并显示错误)
```

### 空间音频

**查询空间音频类型 (0x012A):**
```
→ Payload: 空
```

**设置 (0x0422):**
```
→ Payload: [Type]
0x00 = 关闭, 0x01 = 固定, 0x02 = 头部跟踪
```

**响应 (0x8422):**
```
← Payload: [00] [Type]
```

**通知 (0x0510):**
```
← Payload: [00] [Type]
0x00 = 关闭, 0x01 = 固定, 0x02 = 头部跟踪
```

### 提示音音量

**查询 (0x0130):**
```
→ Payload: 00 00
```

**响应 (0x8130):**
```
← Payload: [Unknown] [Level]
Level: 0-10, 其中 0 = 最大音量
```

**设置 (0x0427):**
```
→ Payload: [Level]
```

### EQ 模式

**查询当前 EQ (0x010F):**
```
→ Payload: 空
```

**查询所有 EQ (0x0122):**
```
→ Payload: 空
```

**设置 EQ 模式 (0x0406):**
```
→ Payload: [eqModeType]
```

**自定义 EQ (0x0418):**
```
→ Payload: [action, min, max, eqId, nameLen, name..., bandCount, (freqLE2, db)...]
```

### 佩戴状态通知 (0x0204, ReportType=0x02)

```
Payload: [0x02] [Count] [Id1] [Status1] [Id2] [Status2] ...
```

| Id | 说明 |
|----|------|
| 0x01 | 左耳 |
| 0x02 | 右耳 |
| 0x03 | 充电盒 |

| Status | 说明 |
|--------|------|
| 0x00 | 已断开 |
| 0x04 | 在充电盒中 |
| 0x05 | 已取出 |
| 0x07 | 正在佩戴 |

### 连接设备通知 (0x0204, ReportType=0x06)

```
Payload: [0x06] [Count] [Device1] [Device2] ...
```

每个设备:
```
[MAC 6B LE] [ProfileFlags 1B] [ConnectionState 1B] [IsActive 1B] [NameLen 1B] [Name NB]
```

| ConnectionState | 说明 |
|-----------------|------|
| 0x02 | 已连接 |

| IsActive | 说明 |
|----------|------|
| 0x01 | 活跃设备 |

MAC 为小端序，需要反转后以 `:` 分隔显示。

---

## 连接流程

1. 建立 SPP 连接 (UUID: `0000079A-D102-11E1-9B23-00025B00A5A5`)
2. 发送握手包 (`0x0100`)
3. 等待握手响应 (`0x8100`)
4. 发送广播码查询 (`0x0200`)
5. 订阅广播码 (`0x0205`)
6. 查询批量状态 (`0x010D`)
7. 查询电量 (`0x0106`)
8. 查询 ANC (`0x010C`)
9. 查询提示音音量 (`0x0130`)
