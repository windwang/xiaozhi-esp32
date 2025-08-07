# 服务器交互 API 文档 (详细版)

本文档详细描述了设备与云端服务器交互的所有 API，包括参数、来源和意义。

---

## 1. HTTP 客户端 API

设备通过 HTTP 客户端与云端进行带外(out-of-band)通信，主要用于获取配置、执行设备激活和下载固件。

### 1.1. 检查版本与获取配置

此 API 用于检查新固件版本，并获取 WebSocket/MQTT 的连接参数和设备激活质询(challenge)。

*   **描述**: 向服务器注册设备，检查更新并获取配置。
*   **方法**: `GET` 或 `POST` (当有 `board.GetJson()` 数据时为 `POST`)
*   **URL**: 
    *   **来源**: 首先尝试从 `wifi` 命名空间的 `ota_url` 设置项中读取。如果为空，则使用编译时配置的 `CONFIG_OTA_URL`。

#### 请求参数

**请求头 (Headers)**

| Header | 值示例 | 来源与说明 |
| :--- | :--- | :--- |
| `Activation-Version` | `2` | **来源**: 根据设备efuse中是否有序列号决定。`1`代表无序列号，`2`代表有序列号。 |
| `Device-Id` | `A1:B2:C3:D4:E5:F6` | **来源**: `SystemInfo::GetMacAddress()`，设备的MAC地址。 |
| `Client-Id` | `uuid-string` | **来源**: `board.GetUuid()`，每个设备的唯一标识符。 |
| `Serial-Number` | `SN123456789` | **来源**: (可选) `esp_efuse_read_field_blob` 从efuse的用户数据区读取。 |
| `User-Agent` | `my-board/1.0.0` | **来源**: `BOARD_NAME` 和 `esp_app_get_description()->version` 组合而成。 |
| `Accept-Language` | `zh-CN` | **来源**: `Lang::CODE`，当前设备的语言设置。 |
| `Content-Type` | `application/json` | **来源**: 固定值。 |

**请求体 (Body)**

*   **来源**: `board.GetJson()`，包含特定于开发板的硬件信息。仅在 `POST` 请求时发送。

#### 响应体 (JSON)

服务器返回一个 JSON 对象，其详细结构如下：

| 顶层字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `firmware` | Object | 包含固件更新信息的对象。 |
| `activation` | Object | 包含设备激活所需信息的对象。 |
| `mqtt` | Object | 包含 MQTT 连接参数的对象。设备会遍历此对象的所有键值对并存入 `mqtt` 设置中。 |
| `websocket` | Object | 包含 WebSocket 连接参数的对象。设备会遍历此对象的所有键值对并存入 `websocket` 设置中。 |
| `server_time`| Object | 包含服务器时间信息的对象。 |

**`firmware` 对象详情**

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `version` | String | 新固件的版本号，例如 `"1.0.1"`。 |
| `url` | String | 新固件的完整下载 URL。 |
| `force` | Number | (可选) 如果为 `1`，则强制设备更新。 |

**`activation` 对象详情**

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `message` | String | (可选) 用于提示用户的激活信息。 |
| `code` | String | (可选) 用于向用户展示的激活码。 |
| `challenge` | String | (可选) 用于设备端计算 HMAC 签名的质询字符串。 |
| `timeout_ms`| Number | (可选) 激活流程的超时时间（毫秒）。 |

**`server_time` 对象详情**

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `timestamp` | Number | 服务器当前的 Unix 时间戳（毫秒）。 |
| `timezone_offset`| Number | (可选) 时区与UTC的偏移量（分钟）。设备会使用此值来调整并设置本地时间。 |

---

### 1.2. 设备激活

*   **描述**: 当 `CheckVersion` 响应中包含 `challenge` 时，设备使用此 API 发送签名以完成激活。
*   **方法**: `POST`
*   **URL**: `CheckVersion` 的 URL + `/activate`

#### 请求参数

**请求头 (Headers)**

*   与 `1.1. 检查版本与获取配置` 中的请求头完全相同。

**请求体 (Body)**

*   **来源**: `Ota::GetActivationPayload()` 生成的 JSON 字符串。

| 字段 | 值示例 | 来源与说明 |
| :--- | :--- | :--- |
| `algorithm` | `hmac-sha256` | **来源**: 固定值，表示使用的签名算法。 |
| `serial_number`| `SN123456789` | **来源**: 从 efuse 读取的设备序列号。 |
| `challenge` | `challenge-string` | **来源**: `CheckVersion` API 响应中的 `activation.challenge` 字段。 |
| `hmac` | `hmac-hex-string` | **来源**: 使用 `esp_hmac_calculate`，以设备的 `HMAC_KEY0` 和服务器下发的 `challenge` 计算出的 HMAC-SHA256 值。 |

---

### 1.3. 固件下载

*   **描述**: 从指定的 URL 下载固件二进制文件 (`.bin`)。
*   **方法**: `GET`
*   **URL**: 
    *   **来源**: `CheckVersion` API 响应中的 `firmware.url` 字段。

--- 

## 2. WebSocket API (完整版)

### 2.1. 连接与鉴权

*   **端点**: WebSocket 的 URL、Token 等连接参数均来源于 `1.1 检查版本与获取配置` API 响应中的 `websocket` 对象。
*   **子协议**: `mcp`
*   **鉴权方式**: **Bearer Token**
    *   **来源**: `token` 的值来源于 `websocket.token` 字段。
    *   **实现**: 在建立 WebSocket 连接时，将 `token` 放入 `Authorization` 请求头中，值为 `Bearer <token>`。

### 2.2. 消息格式

WebSocket 中传输两种类型的消息：**JSON 文本消息**用于信令和控制，**二进制消息**用于传输音频数据。

### 2.3. JSON 消息协议

所有 JSON 消息都有一个顶层字段 `type`，用于区分消息类型。

#### 客户端 -> 服务器

| `type` | 描述 | `payload` 详情 |
| :--- | :--- | :--- |
| `hello` | **连接初始化**。在连接建立并鉴权后发送的第一个消息。 | 见 `4.1 Hello 消息` 章节。 |
| `stt` | **语音识别控制**。 | `{"state": "start"}`: 开始识别。<br>`{"state": "stop"}`: 停止识别。<br>`{"state": "wake_word_detected", "wake_word": "..."}`: 报告唤醒词。 |
| `tts` | **语音合成控制**。 | `{"state": "abort", "reason": "..."}`: 中断服务器的播报。`reason`可选。 |
| `mcp` | **喵伴协议消息**。 | `{"payload": {...}}`: 将 MCP 命令封装在 `payload` 中。详见 `5. MCP API` 章节。 |

#### 服务器 -> 客户端

| `type` | 描述 | `payload` 详情 |
| :--- | :--- | :--- |
| `hello` | **连接确认**。 | 包含 `session_id` 和服务器的 `audio_params`。 |
| `stt` | **语音识别结果**。 | `{"text": "..."}`: 返回识别出的文本。 |
| `tts` | **语音合成控制**。 | `{"state": "start"}`: 准备开始播报。<br>`{"state": "stop"}`: 播报结束。<br>`{"state": "sentence_start", "text": "..."}`: 返回一句话的文本。 |
| `llm` | **大模型状态**。 | `{"emotion": "happy"}`: 指示设备显示特定情感。 |
| `system` | **系统命令**。 | `{"command": "reboot"}`: 指示设备重启。 |
| `alert` | **警报信息**。 | `{"status": "...", "message": "...", "emotion": "..."}`: 指示设备显示警报。 |

| `mcp` | **喵伴协议消息**。 | `{"payload": {...}}`: 将 MCP 响应或事件封装在 `payload` 中。 |

### 2.4. 二进制音频协议

音频数据被封装在二进制帧中进行传输，协议支持多个版本。

**协议 V2 (大小端：网络字节序 - Big Endian)**

| 字段 | 长度 (Bytes) | 描述 |
| :--- | :--- | :--- |
| `version` | 2 | 协议版本号，固定为 `2`。 |
| `type` | 2 | 消息类型，`0` 代表音频。 |
| `reserved`| 4 | 保留字段。 |
| `timestamp` | 4 | 音频帧的时间戳。 |
| `payload_size`| 4 | `payload` 的长度。 |
| `payload` | `payload_size`| Opus 编码的音频数据。 |

**协议 V3 (大小端：网络字节序 - Big Endian)**

| 字段 | 长度 (Bytes) | 描述 |
| :--- | :--- | :--- |
| `type` | 1 | 消息类型，`0` 代表音频。 |
| `reserved`| 1 | 保留字段。 |
| `payload_size`| 2 | `payload` 的长度。 |
| `payload` | `payload_size`| Opus 编码的音频数据。 |

--- 

## 3. MQTT API

### 3.1. 连接与鉴权

*   **Broker 地址**: 从 `1.1 检查版本与获取配置` API 响应中获取。
*   **鉴权方式**: **Username/Password**
    *   **来源**: `client_id`, `username`, `password` 的值均来源于 `1.1` API 响应中的 `mqtt` 对象。

--- 

## 4. WebSocket & MQTT 通用 API

### 4.1. Hello 消息

*   **描述**: 在建立 WebSocket 或 MQTT 连接并**完成鉴权**后，客户端发送的第一个消息，用于协商参数和功能。
*   **消息类型**: `hello`

#### 消息体 (JSON)

| 字段 | 值示例 | 来源与说明 |
| :--- | :--- | :--- |
| `type` | `hello` | **来源**: 固定值。 |
| `version` | `3` | **来源**: 从 `websocket` 或 `mqtt` 设置中读取的 `version` 值。 |
| `transport` | `websocket` | **来源**: 根据当前使用的协议，固定为 `websocket` 或 `mqtt`。 |
| `features` | `{...}` | **来源**: 一个对象，描述客户端支持的功能。`aec` 来自 `CONFIG_USE_SERVER_AEC`，`mcp` 来自 `CONFIG_IOT_PROTOCOL_MCP`。 |
| `audio_params`| `{...}` | **来源**: 一个对象，描述客户端的音频参数。`format` 固定为 `opus`，`sample_rate` 为 `16000`，`channels` 为 `1`，`frame_duration` 为 `OPUS_FRAME_DURATION_MS` (60ms)。 |

--- 

## 5. MCP (喵伴协议) API - 基于 WebSocket

MCP (Model Context Protocol) 命令通过 WebSocket 连接中 `type` 为 `mcp` 的 JSON 消息进行传输，遵循 JSON-RPC 2.0 规范。

### 5.1. `self.get_device_status`

*   **描述**: 提供设备的实时信息，包括音频、屏幕、电池、网络等当前状态。
*   **参数**: 无
*   **返回**: 一个包含设备状态的 JSON 对象。

### 5.2. `self.audio_speaker.set_volume`

*   **描述**: 设置音频播放器的音量。
*   **参数**:
    *   `volume` (Integer, 0-100): 要设置的音量。
*   **返回**: `true` (成功) 或 `false` (失败)。

### 5.3. `self.screen.set_brightness`

*   **描述**: 设置屏幕亮度。 (仅在设备支持时可用)
*   **参数**:
    *   `brightness` (Integer, 0-100): 要设置的亮度。
*   **返回**: `true` (成功) 或 `false` (失败)。

### 5.4. `self.screen.set_theme`

*   **描述**: 设置屏幕的主题。 (仅在设备支持时可用)
*   **参数**:
    *   `theme` (String): "light" 或 "dark"。
*   **返回**: `true` (成功) 或 `false` (失败)。

### 5.5. `self.camera.take_photo`

*   **描述**: 拍摄一张照片并对其进行描述。 (仅在设备支持时可用)
*   **参数**:
    *   `question` (String): 关于照片的问题。
*   **返回**: 一个包含照片信息的 JSON 对象。