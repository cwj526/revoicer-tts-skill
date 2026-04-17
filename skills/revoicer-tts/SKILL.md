---
name: revoicer-tts
description: Generate English word pronunciation audio via revoicer.app. Use this skill whenever the user wants to generate TTS audio, pronunciation recordings, or word audio files — even if they just say "帮我生成音频", "录一下这些单词的发音", "generate audio for these words", or similar. Triggers on any request involving word lists + audio/pronunciation/TTS generation.
---

# Revoicer TTS Skill

Generate English word pronunciation audio using revoicer.app. Designed for non-technical users — always communicate in plain, friendly Chinese, guide step by step, never expose raw errors.

## Step 1 — Collect parameters

Extract from the user's message:

| Parameter | Description | Default |
|-----------|-------------|---------|
| words | Word list (required) | — |
| voice | `emily` (女声) or `james` / `axel` (男声) | — |
| pause | Seconds of silence between words | none |
| output_dir | Where to save the MP3 | `~/Downloads/revoicer` |
| filename | Output filename (no .mp3) | first word in list |

For batch mode, also collect:
| Parameter | Description |
|-----------|-------------|
| group_size | Words per file (e.g. 50) |
| naming_pattern | e.g. `List_{n:02d}_US` |

## Step 2 — Clean and validate the word list

### Auto-clean silently (no user confirmation needed)

Apply these rules to every word before doing anything else:

- **Superscript digits** (`¹²³⁴⁵⁶⁷⁸⁹⁰`) → remove entirely
- **Non-breaking hyphen** (U+2011 `‑`) → replace with regular hyphen `-`
- **Ellipsis** (`…` or `...`) → replace with a single space, then strip
- **Extra whitespace** → collapse to single space and strip

These are cosmetic artifacts common in vocabulary lists and don't affect pronunciation.

### Ask the user only about remaining non-ASCII characters

After auto-cleaning, check for any remaining non-ASCII characters. Exclude accented Latin letters that are valid in English loanwords (é, è, ê, ë, ü, ö, ä, ñ, ç, etc.) — these are fine and should pass silently.

If anything else remains (e.g. Chinese characters, symbols, emoji), show the user:

> 我发现单词列表里有一些特殊字符，帮你整理了一下：
>
> 原始：`hello!`、`world.`、`(classic)`
> 整理后：`hello`、`world`、`classic`
>
> 用整理后的版本继续吗？

Wait for the user's response before moving on.

## Step 3 — Ask for voice if not specified

If the user did not mention a voice preference, always ask — do not assume a default:

> 请问用哪个音色？
> - **Emily**（女声）
> - **James**（男声）
> - **Axel**（男声）

Wait for the user's choice.

## Step 4 — Confirm all parameters

Show a summary and ask for confirmation before generating.

**Single file:**
> **请确认以下参数：**
> - 单词列表：hello、world、classic
> - 音色：Emily（女声）
> - 停顿时间：每个单词之间 2 秒（如未指定则显示"无停顿"）
> - 保存位置：~/Downloads/revoicer/hello.mp3

**Batch mode:**
> **请确认以下参数：**
> - 单词总数：1500 个
> - 分组：每组 50 个，共 30 个文件
> - 音色：Axel（美音男声）
> - 停顿时间：每个单词之间 2 秒
> - 命名格式：List_01_US.mp3 ～ List_30_US.mp3
> - 保存位置：~/Downloads/revoicer/

Only proceed once the user explicitly confirms.

## Step 5 — Get the session cookie

Read the cookie from the skill's own `config.json` (same directory as this SKILL.md):

```
<skill_dir>/config.json
```

Get the value of `ci_session` and `campaignId`. If the file doesn't exist, ask the user to provide their cookie (see Cookie expired section below).

## Step 6 — Build and send the request

**Endpoint**: `POST https://revoicer.app/speak/generate_voice`

**Headers**:
```
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Referer: https://revoicer.app/speak
Origin: https://revoicer.app
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Cookie: ci_session=<value from config.json>
```

**Body**: form-urlencoded, single field `data` containing this JSON:

```json
{
  "languageSelected": "<voice.lang>",
  "voiceSelected": "<voice.id>",
  "toneSelected": "Normal",
  "languageSelectedClone": "English",
  "speakingRateClone": "default",
  "text": "<html text — see below>",
  "simpletext": "<words joined by \n, ending with \n>",
  "charCount": <total chars in simpletext excluding newlines>,
  "wordsCount": <number of words>,
  "campaignId": "<campaignId from config.json>"
}
```

### Voice IDs

| Key | voice.id | voice.lang |
|-----|----------|------------|
| emily | `en-US-JennyMultilingualNeural` | `en-GB` |
| james | `en-GB-RyanNeural` | `en-GB` |
| axel  | `en-US-TonyNeural`             | `en-US` |

### Building the `text` field (HTML)

Each word becomes a `<p>` tag. If pause is set, add a pause span after every word **except the last**. The entire string must end with `**********Normal||||||||||`.

**Without pause:**
```
<p>hello</p><p>world</p>**********Normal||||||||||
```

**With pause (e.g. 2 seconds):**
```
<p>hello<span class="ql-label" contenteditable="false"><span class="badge badge-outline-danger">pause 2sec<button class="btn btn-danger btn-close-pause" contenteditable="false"><i class="mdi mdi-close"></i></button></span></span></p><p>world</p>**********Normal||||||||||
```

Pause label format: `pause 2sec`, `pause 1.5sec`, etc.

### Building `simpletext`

Words joined by newlines, with a trailing newline: `hello\nworld\n`

### Retry logic

Wrap every API call and download in a retry loop — transient SSL/network errors are common:

```python
import time, ssl

ctx = ssl.create_default_context()

def call_with_retry(fn, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            return fn()
        except Exception as e:
            if attempt == max_attempts - 1:
                raise
            wait = 2 ** (attempt + 1)  # 2s, 4s, 8s
            time.sleep(wait)
```

Use `context=ctx` in all `urlopen` calls to avoid SSL issues on macOS.

## Step 7 — Download the MP3

Parse the response JSON, get `data.voice.download_link`, download the file to `output_dir/filename.mp3`. Create the output directory if it doesn't exist.

Use Python to execute steps 6 and 7 inline — no external script needed.

## Step 8 — Batch mode

When generating multiple files, loop through each group and:

1. **Skip existing files** — check if the output file already exists before calling the API. If it does, print `[SKIP] filename` and move on. This allows safely resuming interrupted jobs.
2. **Show progress** — print `[OK] filename (N/total)` after each successful download.
3. **On failure** — retry up to 3 times (see Step 6). If all retries fail, print `[FAIL] filename` and continue to the next file rather than aborting the whole batch.

```python
for i, group in enumerate(groups):
    out_path = os.path.join(output_dir, f"{name}.mp3")
    if os.path.exists(out_path):
        print(f"[SKIP] {name}")
        continue
    # ... generate ...
    print(f"[OK] {name} ({i+1}/{total})")
```

After the batch completes, report how many were generated, skipped, and failed.

## Step 9 — Confirm to the user

On success, tell the user in a friendly tone:
- Where the files were saved
- How many generated / skipped / failed
- Which voice, pause setting

Example (batch): "好了！30 个音频全部生成完毕，用的是 Axel 美音，每个词之间停顿 2 秒。文件保存在 ~/Downloads/revoicer/。"

---

## Error handling

Never show raw errors. Always guide the user.

### Cookie expired

**Sign**: the response body is HTML (`<!DOCTYPE html>` or `<html`).

Tell the user:

> 登录状态已过期，需要重新获取一下，步骤很简单：
> 1. 用 Chrome 打开 revoicer.app，登录账号
> 2. 按 F12，点顶部 **Application** 标签
> 3. 左侧 **Cookies** → `https://revoicer.app`
> 4. 找到 `ci_session` 那行，复制它的值
> 5. 把值告诉我，我来更新

Once the user provides the new value, update `ci_session` in the skill's `config.json`, then retry automatically.

### API error `{"success": false}`

This is distinct from cookie expiry. Check the `message` field in the response:

```python
result = json.loads(body)
if not result.get("success"):
    msg = result.get("message", "未知错误")
    # suggest retry first, only escalate to cookie if retries all fail
```

Tell the user:

> 服务器返回了一个错误：「{msg}」。我先自动重试几次，如果还是不行再告诉你。

Only ask for a new cookie if retries all fail AND the error suggests an auth issue.

### Network error

Retry automatically (see Step 6). Only surface to the user if all retries fail:

> 网络连接有问题，已重试 3 次仍然失败。请检查网络后告诉我，我们再试一次。

### Any other error

Investigate silently, fix if possible. If not, ask the user only for what's strictly needed.

---

## config.json reference

```json
{
  "ci_session": "<cookie value>",
  "campaignId": "<campaign id>",
  "voice": "en-US-JennyMultilingualNeural",
  "language": "en-GB"
}
```

All keys use camelCase. `campaignId` is tied to the revoicer account, not the machine.

---

## Porting to a new machine

This skill is self-contained. To use on a new machine:
1. Copy the entire skill folder
2. Update `config.json` with the new machine's `ci_session` cookie
3. `campaignId` stays the same — it's account-level, not machine-level
