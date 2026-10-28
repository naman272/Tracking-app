# Brain Dump — Spec

Personal capture-and-brief system for one user. Tap or hotkey, speak whatever's on your mind, and get a morning email with what to do today. Plan the day in about 10 minutes on a simple web page.

**Core principle:** the user never has to "log" anything. Capture must take one action. Everything else (sorting, deduping, prioritising, reminding) is automatic.

## Scope

**In:** a backend, an Android widget app, a Mac menu bar app, the morning email, and a planning web page.

**Out (for now):** iOS, Windows, multi-user support, sharing, calendar sync.

## Architecture

```
Android widget ─┐
                ├─► Backend API ─► Transcribe ─► LLM extract ─► Postgres
Mac menu bar  ──┘                                                │
                                   Cron 7:00 ─► LLM brief ─► Email (Resend)
                                   Planning page (served by backend)
```

### Stack
- **Backend:** Python + FastAPI, Postgres (Neon or Supabase), deployed on Railway.
- **Transcription:** OpenAI `gpt-4o-transcribe` (or Deepgram). Must handle Indian English and Hinglish.
- **LLM:** Claude API, used for task extraction and brief generation.
- **Email:** Resend.
- **Cron:** Railway cron.
- **Auth:** a single static bearer token stored in env and baked into the clients. The planning page uses the same token, passed via a magic link in the email.

## Data model

**captures**: `id, source (android|mac|web), audio_url, raw_text, transcript, created_at, processed_at, error`

**tasks**: `id, title, notes, area (beesz|fundraising|content|community|ops|personal|other), due_date, status (inbox|today|later|done|dropped), priority (1-3), source_capture_id, created_at, updated_at, last_surfaced_at, times_skipped`

## API
- `POST /capture` accepts multipart audio or JSON `{text}`, returns 202, and processes asynchronously.
- `GET /tasks?status=`
- `PATCH /tasks/{id}` updates status, title, due date, or area.
- `GET /plan/today` and `POST /plan/today` read and set today's top 3.
- `GET /plan` serves the planning page (HTML, mobile-friendly).

## Processing a capture
1. Store the raw audio or text, then transcribe it.
2. Send the transcript plus the list of open tasks to the LLM. It returns JSON with new tasks, updates to existing tasks (merge duplicates rather than creating new ones), and anything that isn't a task, which is ignored.
3. Parse natural dates ("Friday", "after demo day") relative to the user's timezone.
4. Never drop a capture. On failure, keep the transcript and retry.

## Morning brief (7:00 AM, timezone configurable, default Asia/Kolkata)
- **Suggested top 3 for today:** chosen by the LLM based on due dates, priority, and how long each task has been waiting.
- **Due or overdue:** listed explicitly.
- **New since yesterday:** what was captured.
- **Going stale:** tasks untouched for more than 7 days, each with one-tap "keep / drop" links.
- **Plan button:** opens the planning page.

Keep it short enough to read in 60 seconds. No motivational fluff.

## Planning page (10-minute flow)
1. Show the suggested top 3. The user can swap any of them.
2. Run through the rest of the inbox with three buttons: Today / Later / Drop.
3. Show a "Done" screen. Tasks moved to Later return in future briefs.

Include a mic button on the page for capturing while planning.

## Android app (Kotlin)
- A home-screen widget built with Jetpack Glance and a single mic button.
- Tap the widget to start a foreground service (microphone type) that records. A notification shows a Stop control. Tapping the widget again also stops recording, and so does 3 seconds of silence (configurable).
- Uploads go through WorkManager with retry, plus an offline queue.
- The widget shows its state (idle, recording, sending) and a count of today's captures.
- Opening the app shows a simple list of recent captures and a text input.
- minSdk 29.

## Mac app (Tauri v2)
- A menu bar app that launches at login.
- A global hotkey (default ⌥Space, configurable) opens a small floating panel that starts recording immediately.
- Press the hotkey again or Enter to send. Esc cancels. The panel also has a text field.
- Clicking the menu bar icon does the same thing as the hotkey.
- Includes an offline queue, mic permission onboarding, and auto-update optional later.

## Config (env)
`API_TOKEN, DATABASE_URL, OPENAI_API_KEY, ANTHROPIC_API_KEY, RESEND_API_KEY, BRIEF_EMAIL_TO, BRIEF_TIME=07:00, TIMEZONE=Asia/Kolkata`

## Milestones
1. **Backend + text capture + brief.** A `curl` text capture becomes tasks, and the morning email arrives with them. *Done when:* a real brief lands in the inbox.
2. **Voice + planning page.** Upload audio, then plan from the email link. *Done when:* the 10-minute flow works on a phone browser.
3. **Mac app.** Hotkey → speak → task appears in the next brief.
4. **Android widget.** Tap → speak → task appears in the next brief, and it works offline.

Build in order. Test each milestone end to end before starting the next.
