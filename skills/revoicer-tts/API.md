# Revoicer API 接口文档

Base URL: `https://revoicer.app`

## 鉴权

所有请求需携带 Cookie：

```
ci_session=<your_session_id>
```

Session 从浏览器登录后获取（Chrome DevTools → Application → Cookies）。

---

## 通用请求头

```
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Referer: https://revoicer.app/speak
Origin: https://revoicer.app
```

---

## 接口列表

### 1. 生成语音

`POST /speak/generate_voice`

**请求体**（form-urlencoded，`data` 字段为 JSON 字符串）：

```json
{
  "languageSelected": "en-GB",
  "voiceSelected": "en-US-JennyMultilingualNeural",
  "toneSelected": "Normal",
  "languageSelectedClone": "English",
  "speakingRateClone": "default",
  "text": "<p>hello</p>",
  "simpletext": "hello\n",
  "charCount": 5,
  "wordsCount": 1,
  "campaignId": "92727"
}
```

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `languageSelected` | 语言代码 | `en-GB` |
| `voiceSelected` | 声音 ID | `en-US-JennyMultilingualNeural` |
| `toneSelected` | 语调 | `Normal` |
| `languageSelectedClone` | 语言名称 | `English` |
| `speakingRateClone` | 语速 | `default` |
| `text` | HTML 格式文本 | `<p>hello</p>` |
| `simpletext` | 纯文本（换行结尾） | `hello\n` |
| `charCount` | 字符数 | `5` |
| `wordsCount` | 单词数 | `1` |
| `campaignId` | 账号 Campaign ID | `92727` |

**响应：**

```json
{
  "success": true,
  "data": {
    "voice": {
      "id": "10182681",
      "campaign_id": "92727",
      "voice_gender": "Female",
      "voice_id": "en-US-JennyMultilingualNeural",
      "voice_lang_name": "English - British",
      "voice_name": "Emily",
      "chars": "744",
      "text_length_display": "29",
      "download_link": "https://revoicer.s3.amazonaws.com/voices/10182681_xxx.mp3",
      "duration": "8",
      "tone": "Normal",
      "adding_music": "0",
      "music_link": ""
    },
    "text_length": 29,
    "message": "Your Voice Over have been generated!"
  }
}
```

关键字段：`data.voice.download_link` — MP3 文件直链（S3）

---

### 2. 预览语音

`POST /speak/preview_voice`

**请求体**（同 generate_voice，但不需要 `charCount` / `wordsCount` / `campaignId`）：

```json
{
  "languageSelected": "en-GB",
  "voiceSelected": "en-GB-RyanNeural",
  "toneSelected": "Normal",
  "speakingRateClone": "default",
  "languageSelectedClone": "English",
  "text": "<p>hello</p>",
  "simpletext": "hello\n"
}
```

**响应：**

```json
{
  "success": true,
  "data": {
    "preview_text": "<p>hello</p>",
    "text_length": 5,
    "message": "Your Voice Over preview has been created.",
    "url": "https://revoicer.s3.amazonaws.com/previews/preview_10645.mp3"
  }
}
```

关键字段：`data.url` — 预览 MP3 直链

---

### 3. 添加收藏声音

`POST /speak/addFav`

**请求体**（form-urlencoded）：

```
lang_code=en-GB&lang_id=en-GB-RyanNeural
```

**响应：**

```json
{
  "success": true,
  "data": { "good": "yes" }
}
```

---

### 4. 保存用户设置

`POST /speak/saveSettings`

**请求体**（form-urlencoded）：

```
settings=default_voice&value={"LanguageCode":"en-GB","Id":"en-GB-RyanNeural"}
```

**响应：**

```json
{
  "success": true,
  "data": []
}
```

---

## 已知声音 ID

| 声音 ID | 名称 | 性别 | 语言 |
|---------|------|------|------|
| `en-US-JennyMultilingualNeural` | Emily | Female | English (US) |
| `en-GB-RyanNeural` | James | Male | English (GB) |\n| `en-US-TonyNeural` | Axel | Male | English (US) |

---

### 5. 登录

`POST /user/authenticate`

**Referer**: `https://revoicer.app/user/login`

**请求体**（form-urlencoded）：

```
email=your@email.com&password=yourpassword&g-recaptcha-response=<token>
```

| 字段 | 说明 |
|------|------|
| `email` | 登录邮箱 |
| `password` | 登录密码 |
| `g-recaptcha-response` | Google reCAPTCHA v3 token（由页面 JS 自动生成） |

**响应**：登录成功后服务端设置 `ci_session` cookie。

> **注意**：`g-recaptcha-response` 是 reCAPTCHA v3 动态生成的 token，每次有效期约 2 分钟，无法在脚本中直接伪造，自动登录受此限制（见备注）。

---

## 备注

- 所有响应的 `Content-Type` 为 `text/html`，实际内容是 JSON
- `campaignId` 是账号固定值，从 HAR 中提取为 `92727`
- Session cookie 过期后需重新登录获取
- **自动登录的障碍**：登录接口需要 reCAPTCHA v3 token，该 token 由 Google JS 在浏览器环境中生成，与 IP、浏览器指纹绑定，脚本无法直接模拟。可行方案：
  - 使用 Playwright/Selenium 驱动真实浏览器完成登录，再提取 cookie
  - 使用第三方打码平台（如 2captcha）解决 reCAPTCHA
  - 手动登录后将 cookie 写入配置文件，脚本读取使用
