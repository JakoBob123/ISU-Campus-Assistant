# Daily limit test: how many questions does the free server answer?

**Campus Assistant ISU (unofficial)** is the Android app, and it sends all of its questions to a free Hugging Face
Space. I wanted to find out how many questions one person can get answered (without being logged in to Hugging Face)
before the free GPU allowance runs out.

**Result (November 16, 2026): 3 answers, and after that every question got refused.**

---

## 1. How the program works

### The app and its server

```
 Android phone                         Hugging Face Space (ZeroGPU)
┌───────────────────────┐   HTTPS    ┌─────────────────────────────────────────────┐
│ Campus Assistant app  │ ─────────▶ │ android_space_app.py: the "ask" endpoint    │
│                       │    ask     │   1. search the saved ISU data              │
│                       │ ◀───────── │   2. borrow a GPU, Gemma 4 writes the answer│
└───────────────────────┘  streamed  │   3. send the text back as it is written    │
                                     └─────────────────────────────────────────────┘
```

1. **The phone sends a question** to the Space `JakoBob/Campus_Assistant_App` at
   `https://jakobob-campus-assistant-app.hf.space`. The app doesn't have a login, so there is no Hugging Face account
   attached to the request.
2. **The server searches** through ISU's saved catalog and web pages (6,797 chunks) and picks the 8 best matches. This
   part runs on the Space's CPU, so it doesn't need a GPU at all.
3. **The server borrows a GPU.** The Space runs on **ZeroGPU**, which is Hugging Face's free shared GPUs. Nothing is
   reserved ahead of time. A GPU gets requested for every answer (up to 60 seconds each, which is the
   `SPACE_GPU_SECONDS` setting), and then `google/gemma-4-12B-it` writes the answer using the 8 matches.
4. **The answer streams back** a few times a second. At the end it sends the finished text, a status and the source
   pages.

### What happens when no GPU is given

If Hugging Face won't give it a GPU, the server doesn't just show an error. Instead it:

1. tries one more time after 1.5 seconds (so 2 attempts total),
2. then replies with the status **`busy`** and the message *"The model is busy, or today's free GPU time has run out.
   Try again in a minute. Here are the catalog matches for now:"*, along with the matching ISU pages from step 2.

So even when a question gets refused, you still get the search results back in about 2 seconds. You just don't get a
written answer.

### How a call to the server works

Every question is really two web requests. It's the same whether it comes from the app or from this test:

1. `POST /gradio_api/call/ask` with `{"data": [question, "[]"]}`. The second item is the earlier conversation, and
   `"[]"` means there isn't one. What comes back is an event id.
2. `GET /gradio_api/call/ask/<event id>` returns a stream of events: `generating` (the text so far) and `complete`
   (the finished answer).

---

## 2. How the test works

The test program is `isu_catalog_cache/experiments/app_daily_limit/daily_limit_test.py`.

1. **Questions:** It loads the 50 Arm 1 questions from `ISU-CampusAssistant-RAG-Ollama.ipynb` and asks them in order
   (C01, C02, C03, ...).
2. **Asking:** Each question goes to the live Space using the two requests above. I didn't send a token or a login, and
   every question starts a new conversation, so the server sees exactly what it would see coming from the app.
3. **One at a time:** As soon as one question finishes, the next one gets sent. There's no pause in between.
4. **What gets recorded for each question:**
   - the status the server sent back (`answered`, `busy`, ...)
   - how many seconds until the first text showed up
   - total seconds, timed on my PC
   - the server's own time for the answer
   - the start of the reply
5. **When it stops:** after 3 questions in a row don't get answered, or after 150 questions.
6. **What it saves:** a table with one row per question (`run_<date>_<time>.csv`) and a summary
   (`summary_<date>_<time>.json`).

**If you want to run it again**, do this from the project folder:

```
.venv\Scripts\python.exe isu_catalog_cache\experiments\app_daily_limit\daily_limit_test.py . https://jakobob-campus-assistant-app.hf.space
```

### Test conditions

| | |
|---|---|
| Date and time | November 16, 2026, 12:59:08 to 12:59:47 (39 seconds) |
| Server | `https://jakobob-campus-assistant-app.hf.space`, running, hardware ZeroGPU (`zero-a10g`) |
| Model | `google/gemma-4-12B-it` |
| Login | none |
| Run from | my PC, on my home internet |
| Allowance before the test | not used yet today: the app's usage log had no questions earlier on November 16 |

---

## 3. Results

| # | Question | Status | First text | Total | What came back |
|---|---|---|---|---|---|
| 1 | C01: What are the prerequisites for IT 214? | **answered** | 13.9 s | 15.0 s | "The prerequisites for IT 214 are COM 110 and ENG 101." |
| 2 | C02: What is IT 140 about? | **answered** | 8.4 s | 11.5 s | A description of IT 140 |
| 3 | C03: How many credit hours is IT 150? | **answered** | 4.7 s | 5.4 s | "IT 150 is 3 credit hours." |
| 4 | C04: What's the difference between IT 354 and IT 454? | busy | 2.1 s | 2.1 s | The busy message and the catalog matches |
| 5 | C05: Describe TEC 283. | busy | 2.4 s | 2.4 s | The busy message and the catalog matches |
| 6 | C06: What does CTK303 cover? | busy | 2.4 s | 2.4 s | The busy message and the catalog matches |

| Summary | |
|---|---|
| Questions asked | 6 |
| Answered | 3 |
| Answered before the first refusal | **3** |
| Server time used by the 3 answers | 30.9 seconds |
| Median time for an answer | 11.5 s (first text after 8.4 s) |
| Time for a refused question | about 2.3 s |

The first answer took the longest at 15.0 seconds, and the third one was the fastest at 5.4 seconds.

### Why it stopped: the server's log

All the client ever sees is `busy`, so I had to get the actual reason from the Space's own run log (the Logs tab on the
Space's page, pulled using the account's token). For questions 4, 5 and 6 it shows this twice each (once for each
attempt):

```
[transformers] google/gemma-4-12B-it attempt 1 (busy): TimeoutError: GPU not available:
'You have exceeded your ZeroGPU runs limit. Authenticate with a Hugging Face token for more quota -
https://huggingface.co/settings/tokens'
```

So the limit is on the **number of GPU runs** for visitors who aren't logged in. It isn't a limit on seconds. The three
answers only used about 31 seconds of server time.

### The same pattern on earlier days

The app's usage log (it's a private dataset that stores the time, status and speed of each question, but never the
question itself) shows the same thing from normal use. These times are from the server's clock, in UTC:

| Date (UTC) | Answered | Busy |
|---|---|---|
| November 13, 21:14 to 21:57 | 2 | 6 |
| November 14, 02:24 to 17:23 | 1 | 6 |
| November 15, 00:29 | 1 | 0 |

None of those days had more than 3 answers.

---

## 4. What the result means

- **Someone using the app gets about 3 written answers a day.** After that, every question just gets the busy message
  and the catalog matches until the allowance resets.
- **The free Space is fine as a demo,** but it can't really handle an app that a lot of people are using.
- **An earlier estimate was wrong.** The project notes said visitors without an account get about 2 minutes of GPU time
  a day, which would be around a dozen questions. What I actually measured was 3 runs.

## 5. What this test does not show

- **Whether the limit is per connection or shared by everyone.** I only ran the test from one internet connection. If
  Hugging Face counts all the login-free requests to the Space together, then everyone using the app would be sharing
  the same 3 answers. To check this, ask one question from a phone on mobile data (a different connection) right after
  the limit gets hit on Wi-Fi. If it answers, the limit is per connection.
- **When the allowance resets.** I only ran the test once and didn't wait around for it to reset.
- **Whether the limit is always 3.** This is one run on one day, backed up by three days of usage logs. Hugging Face
  can also change its limits whenever it wants without telling anyone.
- **The limit for logged-in users.** The error message says a token gives you more quota, but I didn't measure that.
- **Answer quality.** The test only checked whether an answer came back and how fast it was.

## 6. Side effects of running the test

- It used up the free runs for the day on my home internet, so the app was getting refused on the same Wi-Fi
  afterwards too.
- It added 6 rows to the app's anonymous usage log (around 18:59 UTC on November 16, which is 12:59 PM here).
- It didn't change any code or any notebook.

## 7. Files

Everything is in `isu_catalog_cache/experiments/app_daily_limit/`:

| File | What it is |
|---|---|
| `daily_limit_test.py` | The test program |
| `run_2026-11-16_1259.csv` | One row per question: time, question id, status, seconds, start of the reply |
| `summary_2026-11-16_1259.json` | The totals in the summary table above |

