# Daily limit test: how many questions does the free server answer?

**Campus Assistant ISU (unofficial)**, the Android app, sends its questions to a free Hugging Face Space. This test
measured how many questions one person, not logged in to Hugging Face, can get answered before the free GPU
allowance runs out.

**Result (November 16, 2026): 3 answers, then every question was refused.**

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

1. **The phone sends a question** to the Space `JakoBob/Campus_Assistant_App`, at
   `https://jakobob-campus-assistant-app.hf.space`. The app has no login, so the request carries no Hugging Face
   account.
2. **The server searches** ISU's saved catalog and web pages (6,797 chunks) and picks the 8 best matches. The search
   runs on the Space's CPU and needs no GPU.
3. **The server borrows a GPU.** The Space runs on **ZeroGPU**, Hugging Face's free shared GPUs: nothing is reserved,
   and a GPU is requested for each answer (up to 60 seconds each, the `SPACE_GPU_SECONDS` setting). The model
   `google/gemma-4-12B-it` then writes the answer from the 8 matches.
4. **The answer streams back** a few times a second, ending with the finished text, a status and the source pages.

### What happens when no GPU is given

If Hugging Face refuses the GPU, the server does not show an error. It:

1. tries once more, after 1.5 seconds (2 attempts in all),
2. then replies with status **`busy`** and the message *"The model is busy, or today's free GPU time has run out.
   Try again in a minute. Here are the catalog matches for now:"*, followed by the matching ISU pages from step 2.

So a refused question still returns the search results, in about 2 seconds, but no written answer.

### How a call to the server works

Every question is two web requests, the same from the app and from this test:

1. `POST /gradio_api/call/ask` with `{"data": [question, "[]"]}`. The second item is the earlier conversation; `"[]"`
   means none. The reply is an event id.
2. `GET /gradio_api/call/ask/<event id>` returns a stream of events: `generating` (the text so far) and `complete`
   (the finished answer).

---

## 2. How the test works

The test program is `isu_catalog_cache/experiments/app_daily_limit/daily_limit_test.py`.

1. **Questions:** it loads the 50 Arm 1 questions from `ISU-CampusAssistant-RAG-Ollama.ipynb` and asks them in order
   (C01, C02, C03, ...).
2. **Asking:** each question goes to the live Space with the two requests above. No token and no login are sent, and
   each question is a new conversation, so the server sees the same thing it sees from the app.
3. **One at a time:** the next question is sent as soon as the previous one finishes, with no pause.
4. **What is recorded for each question:**
   - the status the server returned (`answered`, `busy`, ...)
   - seconds until the first text arrived
   - total seconds, measured on this PC
   - the server's own time for the answer
   - the start of the reply
5. **When it stops:** after 3 questions in a row are not answered, or after 150 questions.
6. **What it saves:** a table with one row per question (`run_<date>_<time>.csv`) and a summary
   (`summary_<date>_<time>.json`).

**To run it again**, from the project folder:

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
| Run from | this PC, on the home internet connection |
| Allowance before the test | unused today: the app's usage log showed no questions earlier on November 16 |

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

The first answer was the slowest, 15.0 seconds, and the third the fastest, 5.4 seconds.

### Why it stopped: the server's log

The client only sees `busy`, so the reason was read from the Space's own run log (the Logs tab on the Space's page,
fetched with the account's token). For questions 4, 5 and 6 it shows, twice each (the two attempts):

```
[transformers] google/gemma-4-12B-it attempt 1 (busy): TimeoutError: GPU not available:
'You have exceeded your ZeroGPU runs limit. Authenticate with a Hugging Face token for more quota -
https://huggingface.co/settings/tokens'
```

So the limit is on the **number of GPU runs** for visitors who are not logged in. It is not a limit on seconds: the
three answers used only about 31 seconds of server time.

### The same pattern on earlier days

The app's usage log (a private dataset; it stores the time, status and speed of each question, never the question)
shows the same thing from normal use. Times are the server's clock, UTC:

| Date (UTC) | Answered | Busy |
|---|---|---|
| November 13, 21:14 to 21:57 | 2 | 6 |
| November 14, 02:24 to 17:23 | 1 | 6 |
| November 15, 00:29 | 1 | 0 |

No day shows more than 3 answers.

---

## 4. What the result means

- **A user of the app gets about 3 written answers a day.** After that, every question returns the busy message with
  the catalog matches until the allowance resets.
- **The free Space works as a demo,** but it can't serve an app that many people use.
- **An earlier estimate was wrong.** The project notes said visitors without an account get about 2 minutes of GPU
  time a day, roughly a dozen questions. The measured limit is 3 runs.

## 5. What this test does not show

- **Whether the limit is per connection or shared by everyone.** The test ran from one internet connection. If
  Hugging Face counts all login-free requests to the Space together, every user of the app would share the same 3
  answers. To check: ask one question from a phone on mobile data, a different connection, right after the limit is
  reached on Wi-Fi. An answer means the limit is per connection.
- **When the allowance resets.** The test ran once and did not wait for it.
- **Whether the limit is always 3.** This is one run on one day, supported by three days of usage logs. Hugging Face
  can change its limits without notice.
- **The limit for logged-in users.** The error message says a token gives more quota; this was not measured.
- **Answer quality.** Only whether an answer came back, and how fast.

## 6. Files

All in `isu_catalog_cache/experiments/app_daily_limit/`:

| File | What it is |
|---|---|
| `daily_limit_test.py` | The test program |
| `run_2026-11-16_1259.csv` | One row per question: time, question id, status, seconds, start of the reply |
| `summary_2026-11-16_1259.json` | The totals in the summary table above |

