# RiNi — Personal AI Chat PWA

A single-page installable PWA: chat with an AI (via [Puter.js](https://js.puter.com) — no API keys, no backend), auto‑extracts durable "memories" from your conversations, and backs everything up to a private Telegram channel.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app — UI, chat logic, Puter.js AI calls, Telegram sync, sounds. Everything is inline (one file), per your single-file preference. |
| `manifest.json` | PWA metadata (name, icons, colors, install behavior). |
| `sw.js` | Service worker — caches the app shell so it opens instantly and works semi-offline once installed. |
| `icons/` | App icons (192, 512, maskable 512, apple-touch). |

## ⚠️ Read this before you push to GitHub

Your Telegram **bot token is hard‑coded in `index.html`** (`TG_BOT_TOKEN`), because that's the only way a static, backend‑less PWA can call the Telegram API directly from the browser, exactly like you asked for.

That means: **if this repo is public, anyone who views the page source has your bot token.** With it they could send messages as your bot, read/post in any chat the bot is in, or spam your channel. This is a real risk, not a formality — please actually decide before publishing:

- **Safest:** make the GitHub repo **private**, and/or host it somewhere only you access (e.g. a private GitHub Pages deployment, or just open `index.html` locally / from your phone's storage).
- **If it must be public:** create a **dedicated bot** used only for this app (not one reused elsewhere), keep the channel private, and treat the token as burned — if you ever suspect abuse, revoke it instantly via **@BotFather → `/revoke`** and generate a new one.
- **More robust (future upgrade):** put the Telegram calls behind a tiny serverless function (Cloudflare Worker / Vercel edge function) that holds the token server-side, and have the PWA call that instead. Happy to build that version if you want it later.

I've kept your token/channel ID in the code as-is since it's your own bot and this is how you asked it to work — just flagging this so it's a decision, not an accident.

## How the pieces work

### AI chat (Puter.js)
Loaded via `<script src="https://js.puter.com/v2/">`. Calls `puter.ai.chat(messages, {stream:true})` and streams the reply in with a typing‑cursor animation, then re-renders it as formatted markdown (bold/italic/lists/tables), code blocks (syntax highlighted), and LaTeX math (`$...$` / `$$...$$`) once the stream finishes.

- **Auth:** handled entirely by Puter — the first time it needs to sign you in, it shows its own popup automatically. If you're already signed in to Puter in that browser, nothing extra happens. The Settings tab also has a manual **Login / Sign Up** / **Sign Out** button for convenience.
- **Custom Instructions** (Settings tab) are sent as a system prompt on every message.
- **Memories** (see below) are also injected into the system prompt, so RiNi actually uses what it remembers.

### Memory Brain (Memories tab)
After every exchange, a small background call asks the AI to pull out at most 2 durable facts (preferences, goals, recurring context) as JSON. New, non‑duplicate facts are added to the timeline and the "AI remembered N memories" counter. This is best‑effort and never blocks the chat if it fails.

### Telegram cloud storage
Two things happen, for two different reasons:

1. **Full transcript log** — every user/AI exchange is posted as its own message to your channel. This is your "unlimited storage": open the channel in Telegram any time to read, search, or forward the complete history. The bot needs to be an **admin of the channel** for this to work.
2. **Compact synced index (pinned message)** — memories, custom instructions, and your session list are bundled into one JSON blob, sent as a message, and **pinned** (replacing the old pin). The **"Telegram cloud retrieval"** button on the Memories tab reads that pinned message (`getChat` → `pinned_message`) and merges it back in — this is what lets a second device pick up your memories.

   *Why not the full chat history too?* The Telegram Bot API has no "list channel messages" endpoint — a bot can only reliably re-read the **currently pinned message**, not arbitrary past ones. So full transcripts stay local per device (reliable, instant, offline-friendly) while the pinned index carries the cross-device bits. If you want true multi-device transcript sync later, the clean fix is a small backend (see the serverless note above) — happy to add it.

### Sounds & animation
All sound effects are synthesized on the fly with the Web Audio API (no audio files to fetch/host) — distinct tones for send, receive, tab switches, and toggles. The AI reply types itself in character-by-character (streamed live when Puter streams, or simulated if it doesn't), then gets replaced with fully formatted markdown once complete.

### Icons
I don't have live internet access from the sandbox that built this, so I couldn't pull real Flaticon assets — I generated a simple matching mark (blue/purple orb + spark, dark rounded square) instead, sized for `192`, `512`, a maskable `512`, and an Apple touch icon. Swap any file in `icons/` for a Flaticon icon of your choice (keep the same filenames, or update the paths in `manifest.json` and the `<link rel="icon">` tags in `index.html`).

## Deploying to GitHub Pages

1. Create a new repo (private, if you're keeping the bot token in the code — see warning above) and upload all files, keeping the `icons/` folder structure.
2. Repo **Settings → Pages** → Source: **Deploy from a branch** → branch `main`, folder `/ (root)` → Save.
3. Your app will be live at `https://<your-username>.github.io/<repo-name>/`.
4. Open it on your phone and use **"Add to Home Screen"** (iOS Safari) or the install prompt (Android Chrome) to install it as a real app icon.

## Telegram bot setup checklist

- [ ] Bot added to the channel as an **admin** (needed to post, pin, and read `pinned_message`).
- [ ] Channel ID confirmed as `-1003953146702` (shown in Settings → Cloud Status).
- [ ] Bot token confirmed working — open `https://api.telegram.org/bot<token>/getMe` in a browser; it should return your bot's info as JSON.

## Notes

- Everything (sessions, messages, memories, instructions, theme) is stored locally in `localStorage` first, so the app works instantly and offline; Telegram sync happens in the background a couple of seconds after you stop typing/chatting.
- This is a static site — no build step, no `npm install`. Just the four files plus `icons/`.
