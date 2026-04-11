# revoicer-tts-skill

A Claude Code skill for generating English word pronunciation audio via [revoicer.app](https://revoicer.app).

## Install

```bash
npx skills add cwj526/revoicer-tts-skill
```

## Setup

After installing, fill in your credentials in `~/.claude/skills/revoicer-tts/config.json`:

```json
{
  "ci_session": "YOUR_CI_SESSION_COOKIE_HERE",
  "campaign_id": "YOUR_CAMPAIGN_ID_HERE",
  "voice": "en-US-JennyMultilingualNeural",
  "language": "en-GB"
}
```

### How to get your `ci_session` cookie

1. Open [revoicer.app](https://revoicer.app) in Chrome and log in
2. Press `F12` → **Application** tab
3. Left panel: **Cookies** → `https://revoicer.app`
4. Find the `ci_session` row and copy its value

### How to get your `campaign_id`

1. Open [revoicer.app/speak](https://revoicer.app/speak) in Chrome
2. Press `F12` → **Network** tab
3. Generate any voice, find the `generate_voice` request
4. In the request payload, copy the `campaignId` value

## Usage

Just tell Claude what you need (in Chinese or English):

- `帮我生成这些单词的发音：hello, world, classic`
- `录一下这几个词，用男声，每个词停顿2秒：apple, banana, cherry`
- `generate audio for: good, better, best`

## Voices

| Key | Name | Gender |
|-----|------|--------|
| `emily` | Emily | Female (default) |
| `james` | James | Male |
