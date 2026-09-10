# RiNi — Personal AI Chat PWA

A single-page installable PWA: one continuous chat with an AI (via [Puter.js](https://js.puter.com) — no API keys, no backend, free), a permanent memory that keeps learning about you, and a full backup that auto-restores from a private Telegram channel on reinstall.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app — UI, chat logic, sessions, Puter.js AI calls, Telegram sync, sounds. Everything is inline (one file), per your single-file preference. |
| `manifest.json` | PWA metadata (name, icons, colors, install behavior). |
| `sw.js` | Service worker — caches the app shell so it opens instantly and works semi-offline once installed. |
| `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `apple-touch-icon.png` | App icons, sitting flat alongside everything else — no subfolder. |

## ⚠️ Read this before you push to GitHub

Your Telegram **bot token is hard-coded in `index.html`** (`TG_BOT_TOKEN`), because that's the only way a static, backend-less PWA can call the Telegram API directly from the browser, exactly like you asked for.

That means: **if this repo is public, anyone who views the page source has your bot token.** With it they could send messages as your bot, read/post in any chat the bot is in, or spam your channel. Please actually decide before publishing:

- **Safest:** make the GitHub repo **private**, and/or host it somewhere only you access.
- **If it must be public:** use a **dedicated bot** only for this app, keep the channel private, and treat the token as burned — if you ever suspect abuse, revoke it instantly via **@BotFather → `/revoke`** and generate a new one.
- **More robust (future upgrade):** put the Telegram calls behind a tiny serverless function (Cloudflare Worker / Vercel edge function) that holds the token server-side instead. Happy to build that if you want it later.

## How the pieces work

### Sessions (with a shared brain across all of them)
Tap the hamburger icon to open the drawer: **New Chat** starts a fresh thread, and past sessions are listed below it (tap to switch, ✕ to delete). Each session's messages are separate — but **memories and custom instructions are global**, shared across every session, so RiNi doesn't "forget you" just because you started a new chat.

### Clear chat
The trash icon in the chat header wipes the messages in the **currently open session** (with a confirmation first). This only clears what's shown in the app — RiNi's extracted memories and the permanent Telegram transcript log are untouched, and the next background sync updates the cloud backup to match the cleared state too.

### Readable while it's still typing
Long replies used to render as a raw, unformatted blob that only "snapped" into proper markdown once the whole response finished — which could look broken mid-stream. Now the reply is re-rendered as clean, word-wrapped, formatted markdown every ~140ms while it streams in, so it stays readable the entire time, not just at the end.

### AI chat (Puter.js) — completely free
Loaded via `<script src="https://js.puter.com/v2/">`. Calls `puter.ai.chat(messages, {stream:true})` with **no model pinned**, so it always uses whichever model Puter provides for free by default — streamed in with a typing-cursor animation, then re-rendered as formatted markdown (bold/italic/lists/tables), syntax-highlighted code blocks, and **LaTeX math** (`$...$` inline, `$$...$$` display) once the stream finishes.

Earlier drafts of this app pinned an OpenAI model with a `web_search` tool for live browsing — that's been removed. Puter's "free and unlimited" story only holds for its own default models; the moment you request a named OpenAI/Anthropic/etc. model, Puter bills that usage to *the signed-in end user's own Puter account*, not you. Rather than surprise people with that, RiNi now sticks to Puter's free default model plus the zero-cost lookups described next.

- **Auth:** handled by Puter — it shows its own sign-in popup automatically the first time it's needed. The Settings tab also has a manual **Login / Sign Up** / **Sign Out** button.
- **Custom Instructions** (Settings tab) are sent as a system prompt on every message.
- **Memories** (below) are injected into the system prompt too, so RiNi actually uses what it remembers.

### Local, zero-cost features
No network call, no billing, ever:
- **Live date & time** — every single message includes the real current date/time (via `Intl`/`Date`, computed in the browser) in RiNi's system prompt, so it never guesses or assumes its training cutoff is "now." This is what makes "what's today's date" or "what time is it" always correct.
- Everything else (sessions/messages/memories/instructions/theme) is read straight from `localStorage` — instant, no loading spinners for basic app state.

### Free web lookups — Wikipedia + DuckDuckGo (text only, no images)
For messages that look informational (roughly: more than a couple of words, not just "hi"/"thanks"/small talk), RiNi automatically — no toggle, no button — calls two free, keyless, CORS-enabled public APIs in parallel:
- **Wikipedia's API** for a short topic summary.
- **DuckDuckGo's Instant Answer API** for a quick factual snippet.

Whatever comes back is fed into RiNi's context so it can answer with real information instead of guessing, and both sources are listed as tappable **source chips** under the reply. No images are pulled (removed on request) and there's no cost to you or the user either way — these are the same free endpoints Wikipedia/DuckDuckGo expose to any web page.

*Limitation, not a bug:* DuckDuckGo's public API only returns short *instant-answer* snippets, not full search-engine results — that's what DuckDuckGo itself exposes for free, nothing this app can work around.

### Memory Brain (Memories tab)
After every exchange, a small background call asks the AI to pull out at most 2 durable facts (preferences, goals, recurring context) as JSON. New, non-duplicate facts are added to the timeline and the "AI remembered N memories" counter. This is best-effort and never blocks the chat if it fails.

### A visible end to each reply
Every AI message ends with a small footer: a **Copy** button (copies the raw text/markdown to your clipboard, with a "Copied ✓" confirmation) and a timestamp — so a finished response reads as finished instead of trailing off.

### Telegram cloud storage — permanent memory + auto-restore
Three things happen, for three different reasons:

1. **Permanent transcript log** — every user/AI exchange is posted as its own message to your channel, forever. Open the channel in Telegram any time to read or search the complete history. The bot needs to be an **admin of the channel** for this.
2. **Full JSON backup** — periodically (a few seconds after you stop chatting), the entire conversation + memories + instructions are bundled into one JSON file and uploaded as a Telegram **document**. This is what makes full restore possible, since it isn't limited by Telegram's 4096-character message size.
3. **Compact pinned index** — a small JSON summary (memory count, instructions, and a pointer to the latest full backup's `file_id`) is sent as a message and **pinned**, replacing the old pin.

**Reinstalling the app, clearing site data, or opening it on a new device** triggers an automatic restore: on boot, if local storage is empty, RiNi reads the pinned index, downloads the full backup document by its `file_id`, and merges everything back in — no button tap required. The **"Telegram cloud retrieval"** button on the Memories tab does the same thing manually, any time.

### Sounds & animation
All sound effects are synthesized on the fly with the Web Audio API (no audio files to fetch/host) — distinct tones for send, receive, tab switches, and copy. The AI reply types itself in character-by-character (streamed live, or simulated if Puter doesn't stream that response), then gets replaced with fully formatted markdown once complete.

### Icons
I don't have live internet access from the sandbox that built this, so I couldn't pull real Flaticon assets — I generated a simple matching mark (blue/purple orb + spark, dark rounded square) instead. All four icon files sit flat next to `index.html` — swap any of them for a Flaticon icon of your choice (keep the same filenames, or update the paths in `manifest.json` and the `<link rel="icon">` tags).

## Deploying to GitHub Pages

1. Create a new repo (private, if you're keeping the bot token in the code — see warning above) and upload all eight files flat, at the repo root.
2. Repo **Settings → Pages** → Source: **Deploy from a branch** → branch `main`, folder `/ (root)` → Save.
3. Your app will be live at `https://<your-username>.github.io/<repo-name>/`.
4. Open it on your phone and use **"Add to Home Screen"** (iOS Safari) or the install prompt (Android Chrome) to install it as a real app icon.

## Telegram bot setup checklist

- [ ] Bot added to the channel as an **admin** (needed to post, pin, upload documents, and read `pinned_message`).
- [ ] Channel ID confirmed as `-1003953146702` (shown in Settings → Cloud Status).
- [ ] Bot token confirmed working — open `https://api.telegram.org/bot<token>/getMe` in a browser; it should return your bot's info as JSON.

## Notes

- Everything is stored locally in `localStorage` first, so the app works instantly and offline; Telegram sync happens in the background a few seconds after you stop chatting.
- This is a static site — no build step, no `npm install`. Just eight flat files.
