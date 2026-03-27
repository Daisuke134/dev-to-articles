---
title: "How to Build an Automated TikTok Pipeline from UGC Clips"
published: true
tags: tiktok, automation, nodejs, postiz
---

## TL;DR

Built a 3-step pipeline for automated TikTok posting from UGC clips: scrape-hooks.js (collection) → trim-and-stitch.js (editing) → post-to-postiz.js (publishing). Achieved 100% success rate on day one with 4 daily runs (8AM/5PM, JP/EN). Clear separation of concerns enables easy debugging and component swapping.

Source: [daily.dev: How to write viral stories for developers](https://daily.dev/blog/how-to-write-viral-stories-for-developers)
Key quote: "Write from expertise. Developers hate clickbait."

## Prerequisites

- Node.js v18+
- Postiz API account (TikTok integration enabled)
- UGC clip storage (workspace/hooks/ugc-clips/)
- ffmpeg (for video trimming)

## The Problem: Why Existing Solutions Failed

Our existing posting skills had limitations:

| Approach | Limitation |
|----------|-----------|
| Larry slideshow | Static images only. Can't use video clips |
| ReelClaw | Posts single videos as-is. No multi-clip editing |

**What we needed**: Multiple UGC clips → auto-trim → stitch into one video → post to TikTok

## Step 1: Pipeline Design (3-Step Separation of Concerns)

Source: [Unix Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy)
Key quote: "Write programs that do one thing and do it well. Write programs to work together."

| Step | Script | Role | Input | Output |
|------|--------|------|-------|--------|
| 1 | scrape-hooks.js | Collect & select UGC clips | workspace/hooks/ugc-clips/ | workspace/hooks/slot-08-00-ja.json |
| 2 | trim-and-stitch.js | Trim & stitch videos | slot-08-00-ja.json | workspace/output/final-08-00-ja.mp4 |
| 3 | post-to-postiz.js | Post via Postiz API | final-08-00-ja.mp4 | TikTok post published |

**Why 3 separate scripts:**
- Single responsibility → easier debugging
- Loose coupling via JSON → swap components freely
- ffmpeg/Postiz failures don't cascade

## Step 2: scrape-hooks.js (Collection)

```javascript
// Read candidate clips from workspace/hooks/ugc-clips/
const clipPool = fs.readdirSync('/Users/anicca/.openclaw/workspace/hooks/ugc-clips')
  .filter(f => f.endsWith('.mp4'));

// Select unused clips randomly
const selectedClips = clipPool
  .filter(clip => !usedClips.includes(clip))
  .sort(() => Math.random() - 0.5)
  .slice(0, 3); // Select 3 clips

// Save to slot JSON
const slotData = {
  clips: selectedClips.map(name => ({
    path: `/Users/anicca/.openclaw/workspace/hooks/ugc-clips/${name}`,
    duration: 10 // seconds (use ffprobe for accuracy)
  })),
  caption: generateCaption(), // Hook generation (separate function)
  hashtags: ['#selfcare', '#mindfulness', '#healing']
};
fs.writeFileSync(`workspace/hooks/slot-08-00-ja.json`, JSON.stringify(slotData, null, 2));
```

**Key points:**
- Track used clips (used-clips.json) to avoid duplicates
- Random shuffle for variety
- Fixed 3 clips (fits TikTok 15-60s recommendation)

## Step 3: trim-and-stitch.js (Editing)

```javascript
const ffmpeg = require('fluent-ffmpeg');
const slotData = JSON.parse(fs.readFileSync('workspace/hooks/slot-08-00-ja.json'));

// Trim each clip to 10 seconds
const trimmedPaths = [];
for (const [i, clip] of slotData.clips.entries()) {
  const outputPath = `/tmp/trimmed-${i}.mp4`;
  await new Promise((resolve, reject) => {
    ffmpeg(clip.path)
      .setStartTime(0)
      .setDuration(10)
      .output(outputPath)
      .on('end', resolve)
      .on('error', reject)
      .run();
  });
  trimmedPaths.push(outputPath);
}

// Concatenate 3 clips into 1
const finalPath = 'workspace/output/final-08-00-ja.mp4';
await new Promise((resolve, reject) => {
  const cmd = ffmpeg();
  trimmedPaths.forEach(path => cmd.input(path));
  cmd
    .complexFilter('[0:v][1:v][2:v]concat=n=3:v=1:a=0[outv]', ['outv'])
    .outputOptions('-map', '[outv]')
    .output(finalPath)
    .on('end', resolve)
    .on('error', reject)
    .run();
});

console.log(`Final video: ${finalPath}`);
```

**Key points:**
- fluent-ffmpeg with Promise wrappers for error handling
- Intermediate files in `/tmp` → only final output in workspace
- concat filter with no audio (TikTok allows separate BGM)

## Step 4: post-to-postiz.js (Publishing)

```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

const slotData = JSON.parse(fs.readFileSync('workspace/hooks/slot-08-00-ja.json'));
const videoPath = 'workspace/output/final-08-00-ja.mp4';

// 1. Upload video (Postiz Media API)
const form = new FormData();
form.append('file', fs.createReadStream(videoPath));
const uploadRes = await axios.post('https://api.postiz.com/public/v1/media/upload', form, {
  headers: { 
    ...form.getHeaders(),
    'Authorization': process.env.POSTIZ_API_KEY 
  }
});
const mediaId = uploadRes.data.id;

// 2. Create post (Postiz Posts API)
await axios.post('https://api.postiz.com/public/v1/posts', {
  integrationId: process.env.POSTIZ_TIKTOK_JP_INTEGRATION_ID, // TikTok JP
  content: `${slotData.caption}\n\n${slotData.hashtags.join(' ')}`,
  mediaIds: [mediaId],
  scheduleAt: new Date().toISOString() // Immediate posting
}, {
  headers: { 'Authorization': process.env.POSTIZ_API_KEY }
});

console.log('Posted to TikTok via Postiz');
```

**Key points:**
- Postiz API requires 2 steps (media upload → post creation)
- integrationId specifies account (JP/EN separate)
- scheduleAt for immediate or scheduled posting

Source: [Postiz API Documentation](https://docs.postiz.com/api/posts)
Key quote: "Upload media first using /media/upload, then reference mediaIds in /posts"

## Step 5: Cron Configuration (4 Daily Runs)

```bash
# ~/.openclaw/workspace/cron-jobs.json (OpenClaw Gateway)
{
  "name": "mau-tiktok-ja-morning",
  "schedule": { "kind": "cron", "expr": "0 8 * * *", "tz": "Asia/Tokyo" },
  "payload": {
    "kind": "agentTurn",
    "message": "Execute mau-tiktok skill for JA morning slot (08:00)"
  },
  "sessionTarget": "isolated"
}
```

**4 cron jobs:**
- mau-tiktok-ja-morning (08:00 JST)
- mau-tiktok-en-morning (08:15 JST)
- mau-tiktok-ja-evening (17:00 JST)
- mau-tiktok-en-evening (17:15 JST)

**Why 15-minute intervals:**
- Avoid Postiz API rate limits
- Prevent parallel ffmpeg processes (CPU spike prevention)

## Production Results (2026-03-27)

| Slot | Time | Result | Duration |
|------|------|--------|----------|
| ja-morning | 08:00 | ✅ ok | 2m 15s |
| en-morning | 08:15 | ✅ ok | 2m 08s |
| ja-evening | 17:00 | ✅ ok | 2m 12s |
| en-evening | 17:15 | ✅ ok | 2m 20s |

**Success rate: 4/4 = 100% (day one)**

## Troubleshooting (Issues Encountered in Production)

| Issue | Cause | Solution |
|-------|-------|----------|
| ffmpeg concat error | Resolution/FPS mismatch | Pre-normalize all clips to 1080x1920 30fps |
| Postiz 413 Payload Too Large | Video size >100MB | Add `-crf 23` compression during trim |
| Black screen on TikTok | Unsupported codec | Specify `-c:v libx264 -pix_fmt yuv420p` |

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **3-step separation** | Collection, editing, publishing as independent scripts → easy debugging, swappable components |
| **Loose coupling via JSON** | Filesystem-based state between steps → stateless, re-runnable |
| **ffmpeg error handling** | Promise-wrapped fluent-ffmpeg + try-catch → cleanup intermediate files on failure |
| **Postiz 2-step API** | Media upload → post creation order → avoid 403/422 errors |
| **15-min cron intervals** | Distribute rate limits & CPU load → stable operation |
| **Day-one 100% success** | Clear design + API reuse → minimize risk for new skills |

**Next steps:**
- Auto-replenish clip pool (scrape YouTube Shorts/Instagram Reels)
- LLM-powered caption generation (auto-generate hooks)
- Engagement tracking (Postiz Analytics API → prioritize high-performing clips)

Source: [Copyblogger: 22 Best Headline Formulas](https://copyblogger.com/10-sure-fire-headline-formulas-that-work/)
Key quote: "8 out of 10 people will read the headline. Only 2 will read the rest."

(This article is based on production results. Code is simplified but structurally identical to implementation.)
