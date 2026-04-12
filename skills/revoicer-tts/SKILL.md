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

## Step 2 — Clean and validate the word list

Check every word for non-alphabetic characters (numbers, punctuation, symbols, Chinese characters, etc.).

- If all words are clean plain English words, proceed silently.
- If any word contains irregular characters, show the user what was found and suggest a cleaned version. Ask whether to use the cleaned list or keep the original.

Example:
> 我发现单词列表里有一些符号，帮你整理了一下：
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

Show a summary of everything and ask for confirmation before generating:

---
**请确认以下参数：**

- 单词列表：hello、world、classic
- 音色：Emily（女声）
- 停顿时间：每个单词之间 2 秒（如未指定则显示"无停顿"）
- 保存位置：~/Downloads/revoicer/hello.mp3

确认后我就开始生成，或者告诉我需要修改哪里。

---

If the user requests changes, update and show the summary again. Only proceed once the user explicitly confirms.

## Step 5 — Get the session cookie

Read the cookie from the skill's own `config.json` (same directory as this SKILL.md):

```
<skill_dir>/config.json
```

Get the value of `ci_session`. If the file doesn't exist, ask the user to provide their cookie (see Cookie expired section below).

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
  "campaignId": "<campaign_id from config.json>"
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

## Step 7 — Download the MP3

Parse the response JSON, get `data.voice.download_link`, download the file to `output_dir/filename.mp3`. Create the output directory if it doesn't exist.

Use Python to execute steps 3 and 4 inline — no external script needed.

## Step 8 — Confirm to the user

On success, tell the user in a friendly tone:
- Where the file was saved
- Which words, which voice, pause setting if any

Example: "好了！已经生成了 3 个单词（hello、world、classic）的音频，用的是 Emily 女声，每个词之间停顿 2 秒。文件保存在 ~/Downloads/revoicer/hello.mp3。"

---

## Error handling

Never show raw errors. Always guide the user.

### Cookie expired

**Signs**: response is HTML (`<!DOCTYPE html>`), or `{"success":false}`.

Tell the user:

> 登录状态已过期，需要重新获取一下，步骤很简单：
> 1. 用 Chrome 打开 revoicer.app，登录账号
> 2. 按 F12，点顶部 **Application** 标签
> 3. 左侧 **Cookies** → `https://revoicer.app`
> 4. 找到 `ci_session` 那行，复制它的值
> 5. 把值告诉我，我来更新

Once the user provides the new value, update `ci_session` in the skill's `config.json`, then retry automatically.

### Network error

> 网络连接有问题，请检查网络后我们再试一次。

### Any other error

Investigate silently, fix if possible. If not, ask the user only for what's strictly needed.

---

## Porting to a new machine

This skill is self-contained. To use on a new machine:
1. Copy the entire skill folder
2. Update `config.json` with the new machine's `ci_session` cookie
3. The `campaignId` in `config.json` is tied to the revoicer account, not the machine — keep it as-is
