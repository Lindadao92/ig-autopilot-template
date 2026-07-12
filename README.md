# ig-autopilot

Schedule and auto-publish Instagram posts through Instagram's **official** publishing API, with captions drafted by Claude — running for free on GitHub Actions. No server, no third-party scheduler, no sketchy browser bots that risk your account.

> Uses the **Instagram API with Instagram Login** (`graph.instagram.com`, `IGA…` tokens) by default. Have a Facebook Graph API (`EAA…`) token instead? Set `IG_GRAPH_HOST=https://graph.facebook.com`. See [setup](#1-get-your-instagram-credentials-from-meta).

- 🗓️ **Schedule** posts by dropping entries in a JSON queue
- 📸 **Autopilot** — drop a photo in `media/`, push, and Claude *looks* at it, writes the caption, and schedules it on your cadence
- 💬 **Auto-replies** to comments on your posts, in brand voice — skips trolls and spam, never invents shipping answers
- 🔭 **Morning engagement briefing** — finds the best posts in your niche and pre-writes a comment for each; you post them by hand in 10 minutes (human-paced, so zero ban risk). *Requires a Facebook Graph API (`EAA…`) token — see the [scout notes](#morning-engagement-briefing-the-scout).*
- 🎨 **Content factory** — writes fresh one-liners in your voice and renders them as branded statement cards, so the queue never runs dry (and the lines that pop off double as product research)
- 🤖 **Draft captions** in your brand voice with the Anthropic API
- ✅ **Official API only** — the same Content Publishing API that Buffer and Later use, so there's no shadowban risk
- 💸 **Free** — GitHub Actions runs the cron; you only pay for Claude tokens (fractions of a cent per caption)
- 👀 **Review-first by default** — content lives as files, so a pull request *is* your approval step

---

## Example output

Real posts from the autopilot: you drop the photo, Claude *looks* at it, writes a
one-liner in your brand voice, and places the text in clear space so it never
covers your face.

<p>
  <img src="examples/example-1.jpg" width="200" alt="my ex texted. the ocean was less interesting so i replied">
  <img src="examples/example-2.jpg" width="200" alt="i invoice my exes for emotional labor still unpaid">
  <img src="examples/example-3.jpg" width="200" alt="went all in emotionally once. folded immediately.">
  <img src="examples/example-4.jpg" width="200" alt="i walk like i own the beach. i own a groupon.">
  <img src="examples/example-5.jpg" width="200" alt="my ex was a red flag. i promoted him to diversification.">
</p>

The content factory can also generate text-only statement cards to keep the queue
full when you're low on photos.

---

## Quick start — the whole thing in one folder

Once setup is done (below), your entire job is this:

1. Drop selfies into **`media/`** (drag them onto the folder on github.com, or `git push`).
2. That's it.

For each new photo, Claude writes a one-liner in your voice, burns it onto the
image in your brand font (placed in clear space, never on your face), reuses that
line as the caption, and schedules it. Already-posted photos are skipped by
filename, so you just keep adding — new selfies daily, no cleanup. See
[`media/README.md`](./media/README.md) for the folder rules.

Nothing waits for you unless you set `AUTOPILOT_REVIEW=true`.

## How it works

```
content/queue.json   →   src/publish.js   →   Instagram Graph API
   (you edit)          (runs on cron in         (container → publish)
                        GitHub Actions)
```

Every 30 minutes a GitHub Action checks the queue for posts whose `publish_at` time has passed and publishes them, then commits the updated status back to the repo. Captions can be drafted ahead of time with `src/generate.js`.

---

## ⚠️ Read this before you start

Three hard requirements from Meta — none are optional:

1. **Your account must be a Business account.** Instagram's API only works with professional accounts, and content publishing is safest on **Business** specifically (Creator support for publishing is inconsistent). Convert in the Instagram app: Settings → Account type → Switch to Business. You'll link it to a Facebook Page in the process.

2. **Media must be reachable at a public, direct file URL.** Instagram *fetches* your image from a URL. **On a public repo this is automatic** — the autopilot serves your photos straight from the repo's `raw.githubusercontent.com` URLs, with no hosting to set up. You only need external hosting (a **Supabase public bucket**, Cloudinary, or S3) if your repo is **private**, or you hand-write queue entries with relative paths. **Google Drive, Dropbox, and iCloud links do NOT work** (they return web pages, not files).

3. **Claude writes captions, not your photos.** You supply the images; the AI handles captions, hashtags, hooks, and scheduling — the words, not the pictures.

**The good news:** because you're only posting to *your own* account, you do **not** need Meta's 2–4 week App Review. That only applies to apps posting on behalf of *other* people. Solo, self-owned automation runs in "development mode" with you as the sole user.

---

## One-time setup (~30–45 min)

### 1. Get your Instagram credentials from Meta

1. Go to [developers.facebook.com](https://developers.facebook.com) → **My Apps** → **Create App** → type **Business**.
2. Add the **Instagram** product to the app.
3. Connect your Instagram Business account and its linked Facebook Page.
4. Generate a **long-lived access token** with the `instagram_business_basic` and `instagram_business_content_publish` scopes — and add `instagram_business_manage_comments` if you want the auto-reply agent. (Meta's Graph API Explorer or the app's token tool will do this. Short-lived tokens can be exchanged for long-lived ones that last ~60 days.)
5. Grab your **Instagram user ID** (numeric) — you can query it from the Graph API Explorer.

> **Which token do I have?** This tool defaults to the **Instagram API with Instagram Login** — tokens start with `IGA…` and hit `graph.instagram.com`. The scopes above produce exactly that. If you instead followed an older tutorial and got a **Facebook Graph API** token (starts with `EAA…`), it will fail with *"Cannot parse access token"* unless you set the repo Variable `IG_GRAPH_HOST=https://graph.facebook.com`. (The hashtag scout needs the `EAA…`/`graph.facebook.com` path — see its section.)

> **Token longevity tip:** long-lived tokens expire in ~60 days. Either run `npm run refresh-token` every ~50 days (and update the secret), or — the low-maintenance path — create a **Meta System User** and issue a **non-expiring** token, then you never touch it again.

### 2. (Optional) Media hosting — only for private repos

**If your repo is public, skip this** — the autopilot serves images straight from the repo, no hosting needed. You only need external hosting if your repo is **private**, or you hand-write queue entries with relative paths.

1. In your Supabase project, create a **public** Storage bucket, e.g. `media`.
2. Upload a photo. Its public URL looks like
   `https://<project>.supabase.co/storage/v1/object/public/media/photo.jpg`.
3. Use that URL as `media_url` in the queue. (Or set `MEDIA_BASE_URL` and use short relative paths.)

### 3. Configure the repo

**For local testing:**
```bash
cp .env.example .env      # then fill in the values
npm run dry-run           # shows what WOULD publish, posts nothing
```

**For GitHub Actions (the real deployment):**
Repo → **Settings** → **Secrets and variables** → **Actions**:

| Secret | Value |
|---|---|
| `IG_USER_ID` | your numeric IG user ID |
| `IG_ACCESS_TOKEN` | your long-lived token |
| `ANTHROPIC_API_KEY` | from console.anthropic.com (only for caption generation) |

Optional **Variables** (not secret): `IG_API_VERSION`, `MEDIA_BASE_URL`, `ANTHROPIC_MODEL`, and `IG_GRAPH_HOST` (only if you use a Facebook `EAA…` token — set it to `https://graph.facebook.com`).

That's it — the `Publish due Instagram posts` workflow now runs every 30 minutes.

---

## Usage

### Schedule a post
Add an entry to `content/queue.json`:
```json
{
  "id": "2026-07-15-launch",
  "status": "scheduled",
  "publish_at": "2026-07-15T17:00:00Z",
  "media_type": "IMAGE",
  "media_url": "https://<project>.supabase.co/storage/v1/object/public/media/photo.jpg",
  "caption": "The one you've been waiting for is finally here. ..."
}
```
- `publish_at` is **ISO 8601 in UTC** (the `Z`). 17:00 UTC ≈ 10am PT / 7pm Berlin.
- `status` must be `"scheduled"` to go out. Use `"draft"` while you're still editing.
- `media_type`: `IMAGE`, `REELS`, or `CAROUSEL`.

Commit it. Because it's a git commit (or PR), you get a clean review + audit trail. When the time passes, it publishes and the status flips to `published`.

### Or: just drop a photo in `media/` (autopilot)

The laziest workflow, and the best one:

```bash
cp ~/Desktop/photo.jpg media/
git add media/ && git commit -m "new photos" && git push
```

On push, a GitHub Action sends each **new** photo to Claude, which *looks at the image* (vision), writes a caption + hashtags + alt text in your `brand.json` voice, and schedules it into the next open slot on your cadence (default **daily at 17:00 UTC** — set `POST_DAYS` and `POST_TIME_UTC` repo Variables to change; `POST_TIME_UTC` accepts multiple comma-separated times for several posts a day). The publisher posts it when the time comes. Cost: roughly a cent per caption.

Rules of the road:
- **JPEG only** (`.jpg`/`.jpeg`) — Instagram's API doesn't accept other image formats. Keep files under ~4 MB.
- **The repo must be public** for zero-config hosting (Instagram fetches the image from the repo's raw URL). Private repo? Set `MEDIA_BASE_URL` to a public bucket that mirrors `media/` instead.
- Photos already queued are skipped automatically, so the folder can just accumulate.
- Want an approval step? Set the repo Variable `AUTOPILOT_REVIEW=true` (or run `npm run autopilot -- --review` locally) and posts land as `draft` for you to flip to `scheduled`.

### Draft captions with Claude
```bash
npm run generate -- "launch week, new product drop, cozy fall vibe" --variants 3
```
Prints 3 caption options. Add `--append` to drop the first into the queue as a draft. Edit `brand.json` to tune the voice — it's the single source of truth for tone, audience, and do's/don'ts.

You can also trigger this from the **Actions** tab (the `Generate captions` workflow) without touching a terminal.

### Auto-reply to comments (the engagement agent)

Twice an hour, the reply workflow scans your recent posts for new comments and answers them in your brand voice — playful thanks for compliments, jokes back at jokes. It skips trolls, spam, and your own comments, and if someone asks something it can't verifiably answer (price, shipping, restocks) it points them to the link in bio instead of inventing an answer. Reply speed is a genuine ranking signal, and this makes the account feel alive at any follower count.

- Needs the `instagram_business_manage_comments` scope on your token.
- **Start with the repo Variable `REPLY_DRY_RUN=true`** — it logs what it *would* say in the Actions output without posting anything. Flip it off once the voice feels right.
- Every handled comment is recorded in `content/replied.json`, so nothing is ever answered twice.
- Test locally with `npm run reply-dry-run`. Don't want the feature at all? Delete `.github/workflows/reply.yml`.

### Fully autonomous mode (no photos, no review)

Out of the box, nothing waits for a human: the review switches (`AUTOPILOT_REVIEW`, `REPLY_DRY_RUN`) are opt-in training wheels — leave them unset and the machine runs itself.

Every morning at 06:00 UTC the **content factory** checks how many future posts are queued. If it's below target (default 7, tune with `REFILL_TARGET_POSTS`), it asks Claude for fresh one-liners in your `brand.json` voice — checked against every line ever used in `content/lines.json` so it never repeats — renders each as a 1080×1350 statement card (rotating colorways, your handle as watermark), and hands them to the autopilot to caption and schedule. The queue literally cannot run dry.

Your real photos still slot in whenever you drop them in `media/` — the factory only tops up the gap. Watch which generated lines get the most saves and comments: those are your next product ideas. The content engine doubles as product research.

Note: card rendering is the repo's one dependency (`sharp`); the refill workflow installs it automatically. All other scripts remain dependency-free.

### Morning engagement briefing (the scout)

> ⚠️ **Requires a Facebook Graph API token.** Hashtag search (`ig_hashtag_search`) exists **only** on `graph.facebook.com` with an `EAA…` token — it is **not** part of the default Instagram Login API (`IGA…` tokens on `graph.instagram.com`). To use the scout, get an `EAA…` token and set the repo Variable `IG_GRAPH_HOST=https://graph.facebook.com`. With the default `IGA…` token the briefing comes back empty. Publishing and auto-replies work on either token. Don't want the scout? Delete `.github/workflows/scout.yml`.

Every morning at 9am Berlin time, the scout reads the **top posts** under your niche hashtags (Instagram's official, read-only hashtag API), has Claude pick the ~10 best targets, and pre-writes a comment for each in your voice. It commits a briefing to `content/engage/YYYY-MM-DD.md` with links and copy-paste-ready comments.

Then the human part: 10 minutes, from your phone. Open link → post the comment (or riff on it) → next. Skip anything that doesn't feel right. Because every action is you, at human pace, from the real app, there is **nothing for Instagram to detect** — this is just you being good at comments, faster.

Why the comments are written the way they are: never salesy, no brand plugs, no links. A genuinely funny comment gets profile taps; a promotional one gets reported. The joke *is* the marketing.

- Set `SCOUT_HASHTAGS` (repo Variable) to your niche, e.g. `datingmemes,relatable,singlelife`. Keep it to ~4–6: Instagram allows 30 *unique* hashtags per rolling week (querying the same ones daily is fine).
- Targets already suggested are tracked in `content/scouted.json` and never repeated.
- Don't want it? Delete `.github/workflows/scout.yml`.

**What this repo deliberately does NOT include:** bots that auto-like, auto-comment, or auto-follow other accounts. There's no official API for that; the unofficial routes impersonate the mobile app, violate Instagram's terms, and are the most common way small brand accounts get action-blocked or banned. The scout gives you the same reach mechanic with a human hand on every action.

### Media types cheat sheet
| Type | Requirements |
|---|---|
| **Image** | JPEG, aspect ratio 4:5 → 1.91:1, ≤ 8 MB, sRGB |
| **Reel** | MP4/MOV, 9:16, 5–90 seconds, H.264/HEVC |
| **Carousel** | 2–10 items; all cropped to the **first** item's aspect ratio |

Captions: ≤ 2,200 characters. Instagram does **not** render markdown/bold in captions.

---

## Good to know

- **Rate limit:** 100 API-published posts per rolling 24 hours. Carousels count as one. You will never hit this.
- **Cron isn't to-the-minute.** GitHub's scheduler is best-effort and can lag a few minutes (or, rarely, skip a slot under heavy load). Perfect for social; not for a countdown launch. Publish anything "due," don't rely on exact timing.
- **Inactive repos pause schedules.** GitHub disables scheduled workflows after 60 days with no repo activity. The auto-commit on each publish keeps it awake; a monthly manual run also does.
- **Stories & some sticker features** aren't fully supported by the publishing API. This tool focuses on feed images, carousels, and Reels.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| **`"Cannot parse access token"`** (OAuthException 190) | You have a Facebook `EAA…` token but the tool defaults to the Instagram Login API | Use an `IGA…` token, **or** set Variable `IG_GRAPH_HOST=https://graph.facebook.com` |
| **Posts never publish / image fetch fails** | Repo is private, so Instagram can't read the raw image URL | Keep the repo public, or set `MEDIA_BASE_URL` to a public bucket |
| **Morning briefing is empty** | Hashtag search isn't available on the Instagram Login API | Use a Facebook `EAA…` token + `IG_GRAPH_HOST=https://graph.facebook.com` |
| **Some photos never get captioned** | Non-JPEG (e.g. HEIC), or file over ~4.5 MB — silently skipped | Convert to JPEG, keep under ~4 MB |
| **A post went out a few minutes late** | GitHub cron is best-effort, not to-the-minute | Expected — fine for social; don't rely on exact timing |
| **Schedules stopped firing** | GitHub disables cron after 60 days of repo inactivity | Any commit re-arms it; each publish auto-commits, so active accounts stay awake |
| **Publishing stopped after ~60 days** | Long-lived token expired | `npm run refresh-token` (or use a non-expiring Meta System User token) |
| **Caption generation errors** | Bad `ANTHROPIC_API_KEY`, or `ANTHROPIC_MODEL` set to a model your account can't access | Check the key; set `ANTHROPIC_MODEL` to a model you have access to |

---

## Want a dashboard instead of JSON?

This repo is deliberately minimal (one dependency, files-as-database). If you'd rather have a Next.js + Supabase UI with a calendar, a draft-approval queue, and Vercel Cron doing the scheduling, that's a natural next version — the Graph API client in `src/instagram.js` drops straight in.

---

## License

MIT — see [LICENSE](./LICENSE). Fork it, ship it, make it yours.
