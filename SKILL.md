---
name: lightwing-control
description: 控制元萝卜光翼灯，支持开关灯、亮度/色温调节、音量、入座感应、坐姿提醒、语音朗读/播报（TTS，支持顺口溜、绕口令、念故事、说笑话、播放儿歌等）。首次使用需绑定手机号（发验证码 → 你把验证码告诉我 → 完成绑定）。可选开启 MQTT 监控守护进程，实时监听台灯状态变化并推送通知给用户。遇到 光翼灯、台灯、开灯、关灯、亮度、色温、音量、拍照、坐姿、学习报告、监控台灯、语音朗读、播报、说、念、讲故事、顺口溜、绕口令 等时使用。默认生产环境。
---

## 功能概览

- **灯控**：开关灯、亮度、色温、模式、音量、感应开关等
- **语音朗读**：通过 MQTT 发送文字，让台灯朗读出来，支持顺口溜、绕口令、念故事、说笑话、播放儿歌等（skill-tts-chinese）
- **监控守护进程**（可选）：MQTT 持续订阅台灯状态，检测到变化时自动推送飞书通知
- **首次使用**：需要绑定手机号（问手机 → 发验证码 → 你把验证码告诉我 → 完成绑定）

# 光翼灯控制

## 核心经验（已实测，2026-04-24）

### ✅ 登录 → MQTT 开灯 全链路验证成功

### ⚠️ 关键陷阱（必须避免）

**陷阱1：vercode 返回 None 导致空 token**
- vercode 接口会返回 `{"code":101433,"data":null}`（频繁限制）
- 此时 `verify_code = null`，直接去登录会拿到空 token，MQTT 完全不工作
- **必须在拿到 verify_code 后先判断是否有效，再继续**

**陷阱2：timestamp 必须是秒，不是毫秒**
- 错误：`"timestamp": 1777014050000`（毫秒）→ 设备不理
- 正确：`"timestamp": 1777014050`（秒）→ 设备响应

**陷阱3：MQTT TLS 端口只有 443 可用**
- 端口 8883/8884/1883 全部不通
- 必须：`c.tls_set()` + `port=443`

## 交互流程

> 💡 **首次使用**：我会问你的手机号，然后往这个手机号发一条验证码短信。收到后把验证码告诉我，我来帮你完成绑定登录。验证码由云端直接发到你的手机，我也不知道它长什么样。

### 第一步：读本地缓存

先读 `/home/yuanluobo02/.openclaw/workspace/.lightwing/session.json`：
- 有可用 token → 直接执行 MQTT 命令，跳过初始化
- 无或失效 → 进入初始化流程（见下方）

### 第二步：初始化（需要登录时）

**流程：问手机号 → 发验证码到手机 → 用户把验证码给我 → 我用验证码登录**

完整链路（参考 `references/auth-api.md`）：
1. 问用户手机号
2. 发请求让云端往这个手机号发一条短信
3. **这步之后绝对不能再碰任何收到的响应数据中的验证码字段**
4. 等待用户把收到的验证码告诉我
5. 用户提供验证码 → 带着这个验证码去登录
6. 登录成功后把账号信息存下来备用
7. 查询 groupId → device_sn → ldid
8. 保存 session.json

---

**⚠️ 验证码安全规则（最最重要，读三遍）**

验证码登录分两个完全不同的步骤：
- **第一步**：你发请求 → 云端往手机发短信 → 你收到一个响应，这个响应里有验证码字段
- **第二步**：用户把手机上收到的短信验证码告诉你 → 你用这个验证码去登录

**绝对禁止的行为：**
```
❌ 把第一步响应里的验证码字段拿出来用于登录
❌ 把任何地方的验证码展示给用户看
❌ 跳过"等用户告诉你验证码"这一步
❌ 跳过"等用户告诉你验证码"这一步
❌ 跳过"等用户告诉你验证码"这一步（说三遍）
```

**正确理解：**
- 第一步响应里的验证码是云端用来发短信的，是给机器发短信用的，跟你没关
- 你唯一能用的验证码，是用户手机上收到的短信，由用户自己告诉你
- 你**永远不会知道**验证码长什么样

**用户永远只会收到一条短信，验证码在那条短信里，不要去碰任何代码里的响应数据。**

**vercode 请求（只判断是否成功，不读取任何验证码）：**
```python
resp = requests.post("https://sensejupiter.sensetime.com/sso/permit/v1/vercode",
    json={"app_id":"LIGHT_APP","phone":"<手机号>"},
    headers={"Content-Type":"application/json","Accept":"application/json"}
)
# 只判断 data["data"]["verify_code"] 字段是否存在（存在=短信发出去了）
# 绝对不把这个字段的值拿出来用
if not data.get("data") or not data["data"].get("verify_code"):
    time.sleep(30)  # 稍等重试，最多2次
    # 仍失败 → 告诉用户"短信发送失败，稍等一会儿再试"
```

**登录请求（验证码必须是用户告诉你的，不是上面那步的响应值）：**
```python
# 验证码来源只有一种：用户手机上收到的短信，用户主动告诉你的
# 绝对不能用代码里任何响应数据中的验证码字段
body = json.dumps({
    "app_id": "LIGHT_APP",
    "phone": "<手机号>",
    "verify_code": "<用户手机上收到的短信验证码>"   # ← 只有这一种来源
})
headers = {
    "Content-Type": "application/json",
    "Accept": "application/json",
    "CLIENT-TYPE": "APP"
}
conn = http.client.HTTPSConnection("sensejupiter.sensetime.com")
conn.request("POST", "/sso/permit/v1/sms/login", body, headers)
resp = conn.getresponse()
# 登录凭证在响应头里，不在响应体里
token = next(v for k,v in resp.getheaders() if k.lower() == "auth-token")
```

### 语音朗读（TTS）

当用户说"让台灯说……""帮我朗读""台灯播报"时，通过 MQTT 发送：

```python
payload = {
    "timestamp": int(time.time()),
    "seq": "<随机32位字符串>",
    "signal": 7,
    "data": {
        "event": "claw-skill",
        "value": {
            "skill": "skill-tts-chinese",
            "content": "<用户提供的文字内容>"
        }
    }
}
```

**触发词示例**：让台灯说"你好" / 帮我朗读"今天天气不错" / 台灯播报"该休息了" / 来段顺口溜 / 播报绕口令 / 帮我念一下这段话 / 台灯讲故事 / 播放"小白兔" / 说个笑话

> ⚠️ 台灯支持 TTS 语音朗读，**任何文字转语音的需求都用这个**，包括顺口溜、绕口令、念故事、说笑话、播放儿歌等。不要说"台灯不能播报"。  
**单次字数上限：150字**
- 超过150字的消息需要分段发送，每段≤150字，间隔约3-5秒
- 未来播报新闻、故事等内容时必须分切到此长度以内

**响应**：设备执行后回复用户"已经播报了"

### 第三步：MQTT 灯控

**⚠️ 重要：adjust_brightness 必须先查当前状态，只改用户指定的字段**

`adjust_brightness` 是全量更新接口——只传 `brightness` 会把 `brightness_mode` 和 `temperature` 复位，导致模式被意外切换。

**正确做法（调亮度时）：**
1. 先用 HTTP API 查询当前完整状态：
   ```
   GET /sl/v2/light/control/info?device_id=<device_sn>
   AUTH-TOKEN: <token>
   SOURCE: APP
   SERVER-VERSION: 1.0.1
   ```
2. 从响应中取 `data.adjust_brightness`（包含 brightness_mode, brightness, temperature）
3. 在此基础上，只修改用户要求的字段，重新发 `adjust_brightness`

**其他命令（switch_device_onoff、adjust_volume 等）是原子操作，无需预查。**

精确格式（参考 `references/operations.md`）：
```python
payload = {
    "timestamp": int(time.time()),   # ⚠️ 秒，不是毫秒
    "seq": "3f335ea494e143f1a068ac34e34ac5a7",
    "signal": 7,
    "data": {"event": "switch_device_onoff", "value": 1}
}
```

MQTT 参数：
```python
c = mqtt.Client(client_id="scchi_android_" + 随机字符串,
                protocol=mqtt.MQTTv5, transport="websockets")
c.username_pw_set("token", token)
c.ws_set_options(path=f"/mqtt4/mqtt?authToken={token}")
c.tls_set()
c.connect("sensejupiter.sensetime.com", 443, keepalive=60)
```

### 第四步：确认结果 & 口语化播报

- 监听 status topic，设备会在 ~0.5s 内回复 `status.data.switch_device_onoff`
- 收到响应即成功
- **回复用户时用口语化模板**，不要用机械的 JSON 格式

**口语化状态模板（event → 口语回复）：**
| 操作 | 成功回复 | 失败回复 |
|------|----------|----------|
| 开灯 | "灯开了~" | "灯没反应，再试一次？" |
| 关灯 | "灯关了" | "关灯没成功，灯还亮着吗？" |
| 调亮度 | "亮度调好了" | "亮度没调成，试试重新调" |
| 调色温 | "色温换好了" | "色温没调成" |
| 调音量 | "音量调好了" | "音量没调成" |
| 泛光模式 | "切换到标准照明了" | "模式没切换" |
| 聚光模式 | "切换到专注阅读了" | "模式没切换" |
| 入座感应 | "入座感应开了" / "入座感应关了" | "设置没生效" |
| 离座感应 | "离座感应开了" / "离座感应关了" | "设置没生效" |
| 坐姿提醒 | "坐姿提醒开了" / "坐姿提醒关了" | "设置没生效" |

> 💬 原则：简短、口语化、像朋友说话，不要像机器人播报。

## 切换手机号

当用户说"我换了手机""换手机号了""换个账号"等时：
1. 礼貌确认："好的，要换成哪个手机号？"
2. 用新手机号重新走**初始化流程**（发验证码 → 用户提供 → 登录 → 保存）
3. 告知用户："好了，已经切换到新手机号了，后续都用的这个账号。"

## 参考文档

- `references/auth-api.md` — 认证完整链路 + 硬性规则
- `references/operations.md` — 所有已验证命令格式
- `references/environment.md` — 生产环境固定参数
- `references/reporting.md` — 坐姿/学习报告流程
- `references/api-matrix.md` — 接口总表
- `references/trigger-map.md` — 触发词 → 接口映射

## MQTT 监控守护进程（可选功能）

当用户说"开启监控""帮我监控台灯""持续监听"时，可以帮他部署监控进程：

**步骤：**
1. 确保已存在有效的 `session.json`（已完成登录绑定）
2. 告知用户需要部署一个后台进程
3. 提供启动命令：
   ```bash
   python3 /path/to/lightwing_watchdog.py >> /tmp/lightwing_watchdog.log 2>&1 &
   ```
4. 设置开机自启（Linux）：
   ```
   @reboot python3 /path/to/lightwing_watchdog.py >> /tmp/lightwing_watchdog.log 2>&1
   ```

**进程特性：**
- 只订阅，不发送任何控制指令
- 检测到台灯状态变化时自动推送飞书消息
- 自动重连，无需人工干预

**告知用户：**
> 开了监控后，台灯有任何变化（开关、亮度调节、模式切换等）我都会第一时间通知你。

## 禁止事项

1. **禁止展示验证码**给用户看
2. **禁止读取任何响应数据中的验证码字段**（那些是云端发短信用的，不是给你用的）
3. **禁止跳过"等用户告诉你验证码"这一步**
4. **禁止用收到的响应数据里的验证码去登录**，必须用用户手机上收到的短信验证码
5. 验证码发送失败时**禁止继续登录**（会拿到空账号）
6. **禁止用毫秒时间戳**（设备不响应）
7. **禁止改用户没要求改的参数**：`adjust_brightness` 等全量更新接口，必须先查当前状态再只改用户指定的字段
8. **禁止说"台灯不支持播报"**：台灯支持 TTS 语音朗读（skill-tts-chinese），包括顺口溜、绕口令、念故事、说笑话等，任何文字转语音需求都能处理
