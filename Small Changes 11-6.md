# What changed: the Ollama notebook and the Space notebook

This file goes over how ReggieBot went from the original Gemini notebook to two new versions, and why I made each
change. I did this work between October 14 and 28, 2026.

| Notebook | What it is | Status |
|---|---|---|
| `ISU-CampusAssistant-RAG-Demo.ipynb` | The original version, which answers with Google Gemini | Unchanged |
| `ISU-CampusAssistant-RAG-Ollama.ipynb` | **The main program.** It answers with Gemma 4 running on my computer through Ollama | Reworked (Part 1) |
| `ISU-CampusAssistant-RAG-Space.ipynb` | **New.** The same program on a Hugging Face Space with ZeroGPU, so it has a permanent public address | New (Part 2) |
| `ISU-CampusAssistant-Compare.ipynb` | Gemini vs Ollama on the same 50 questions, with blind grading | Small updates so it keeps working |

---

## Part 1 · The Ollama notebook

### 1. Answers come from a local model instead of Gemini

**Why:** Gemini's free tier has daily and per-minute limits, so the app could run out of answers.

- The answers are now written by **`gemma4:12b` running in Ollama on my PC**. There's no API key, no quota and no
  cost per question.
- The prompt, the search and the source links are all the same as the Gemini version. The only thing that changed
  is the model.
- Settings: `OLLAMA_MODEL = "gemma4:12b"`, `OLLAMA_CONTEXT = 8192` tokens (Ollama's default can be too small, and it
  cuts the prompt off without telling you), `OLLAMA_KEEP_ALIVE = "30m"` and `OLLAMA_TIMEOUT = 300` s. I turned
  "thinking" off so the model just answers directly. Temperature is still 0.2.
- If Ollama isn't running or the model isn't downloaded, the chat shows the matching catalog entries with a short
  explanation instead of an error. Once Ollama is back, it reconnects by itself.
- `ollama_report()` replaces `gemini_report()`. It checks that Ollama is running, that the model is downloaded,
  that it actually answers, and how much of it fits in GPU memory.
- Experiment results get saved to `experiments/arm1_ollama/`, so they never overwrite the Gemini results.

### 2. The project write-up is inside the notebook

I added these sections to cover the project requirements:

| Requirement | Where in the notebook |
|---|---|
| Components: pre-determined vs probabilistic routes | **Design** section: a table of 17 components, plus `route_trace()`, which shows which components a question triggers |
| Workflow and sequence diagrams | **Design** section |
| Architecture diagram | **Design** section, with a table of what each class does |
| Must-have features and top 3 nice-to-have features | **Design** section |
| Edge cases where it won't answer | **Design** section: 15 scenarios, plus a cell that runs 7 of them |
| How output is evaluated | **§13**: retrieval scores, automatic answer checks, and then manual grading |
| Cost, reliability and speed | **§14**: times the answers and estimates the electricity cost |
| Sharing with 5–15 people | **§15**: public link, a waiting line and an anonymous usage count |

While doing this I found and fixed two problems with the evaluation:

- **The trap question check was undercounting.** Answers that said "do not contain" weren't being counted as
  declining. After the fix, the score went from 3/6 to 6/6, which matches what the answers actually say.
- **Time to first token** now gets measured on every answer, along with the full answer time.

### 3. A speed fix: about 2 seconds saved on every answer

**Why:** Ollama's own timings showed the model only needed under half a second, but answers were taking about 3
seconds to start.

The problem was the address `localhost`. On Windows it tries IPv6 first, which Ollama doesn't listen on, and then
waits about 2 seconds before falling back. I changed the setting to `OLLAMA_URL = "http://127.0.0.1:11434"`.

| | Before | After |
|---|---|---|
| Time to first token (median) | 3.1 s | **0.9 s** |
| Full answer (median) | 4.6 s | **2.2 s** |

The answers and scores didn't change.

### 4. The code moved into a Python package

**Why:** having one notebook with about 1,800 lines of code made it hard to change things safely, and two other
files had to dig code out of its cells.

All the code is now in `reggiebot/`, with one file for each stage:

| File | Notebook section | What it does |
|---|---|---|
| `config.py` | §1 | Every setting: model, `TOP_K`, sites to crawl, paths, sharing |
| `collect.py` | §2–§4 | Catalog API, website crawler, chunking, change detection |
| `index.py` | §5 | Hybrid search: BM25 + MiniLM/Chroma, rank fusion, course-code pin, section router |
| `answer.py` | §6 | The prompt, the model interface, streaming, retries, fallbacks |
| `core.py` | §7 | `CatalogApp`: the data, the index and the answerer together |
| `ui.py` | §8 | The ReggieBot chat page |

In the notebook, §1 loads the package, and each of §2–§8 still has its explanation and lists the functions it
covers, with the file and line where each one is. The crawler, the launch and all of the evaluation are still in
the notebook. **If you want to change a setting**, edit `reggiebot/config.py` and restart the kernel. **If you
want to change code**, edit the `.py` file and re-run the cell. `autoreload` picks it up.

After the move, Arm 1 gave the same scores, and 49 of the 50 questions returned the exact same chunks. The 50th one
was only different because of Chroma's random variation (see Known issues).

### 5. A model interface, and a fake model for testing

**Why:** so I (or someone else) can try the app without Ollama or a graphics card.

- The answerer talks to any model through three methods: `check()`, `warm_up()` and `generate(system, prompt)`.
- `OllamaModel` is the real model. `FakeModel` answers instantly with no AI and just lists what the search found,
  so the search, source links and chat page can be tried on any computer.
- The fake model can also pretend to fail (`fake:offline`, `fake:missing`, `fake:busy`, `fake:broken`) so you can
  see the fallback messages and retries.
- You can switch with `MODEL` in `config.py`, `python app.py --fake`, or `start_test_mode.bat`.

### 6. Answers come back as one result

**Why:** before this, the code that called the answerer had to guess what happened from the first few words of the
text. Timing was also saved on a shared object, so two answers at the same time could mix it up.

`answer()` now returns an `Answer` with `.text`, `.hits`, `.status`, `.timing`, `.model` and `.retries`. The status
is one of `answered`, `partial`, `no_matches`, `offline`, `missing`, `busy` or `rejected`. Arm 1, the speed report,
the edge cases, the chat page and the usage log all read it directly, so rewording a message can't mess up the
counts anymore.

### 7. Running it without VS Code

- `app.py` starts and stops the app from a terminal, so you don't need VS Code. While it's running it keeps the PC
  awake, and when it stops it frees up the graphics card.
- Double-click files: `start_reggiebot.bat`, `share_reggiebot.bat`, `start_test_mode.bat` and `stop_reggiebot.bat`.
- Options: `--share` (public link), `--hours 3` (stops by itself), `--refresh` (re-crawl first) and `--status`.
- I added a login for the shared link at one point but ended up taking it out, so anyone with the link can use it.

### Results (Arm 1, 50 questions, measured October 19)

| Measure | Result |
|---|---|
| Right source in the 8 chunks given to the model | 44/44 |
| Right source in the 5 source cards a user sees | 43/44 (MRR 0.93) |
| Billing, housing and dining questions routed to their section | 20/24 |
| Model wrote an answer | 50/50 |
| Answer cites the expected course or website | 44/44 |
| Trap questions declined | 6/6 |
| Time to first token / full answer (median) | 0.9 s / 2.2 s |
| Cost | $0 API, and about $0.03 of electricity per 1,000 answers (upper estimate) |
| Manual grading | Still need to do it (§13) |

---

## Part 2 · The Space notebook (new)

### Why a second version

A Hugging Face Space gives ReggieBot a **permanent public address** that still works when my PC is off. The free
shared GPU it uses (ZeroGPU) can't run Ollama, so the model has to be loaded inside the app with `transformers`
instead.

**Live Space:** https://huggingface.co/spaces/JakoBob/Campus_Assistant

### What I added

| File | What it does |
|---|---|
| `ISU-CampusAssistant-RAG-Space.ipynb` | Explains the Space version, tests it, builds the upload folder and uploads it |
| `space_app.py` | The Space's entry point: loads Gemma 4 when it starts and answers inside `@spaces.GPU` |
| `reggiebot/hf_model.py` | `TransformersModel`, which is the transformers version of the model interface from Part 1 |
| `space_build/` | Made by the notebook (75 MB). It's exactly what gets uploaded. Git ignores it |

The Space uses the same `reggiebot` package as the Ollama version. The only difference is how the model plugs in.

### The notebook's sections

| Section | What it does |
|---|---|
| Settings | Model (`google/gemma-4-12B-it`), the tiny local test model, GPU seconds per answer, Space name |
| §1 Try it on this computer | Runs the whole program with a tiny model (SmolLM2-135M) on the CPU to check that everything connects. The answers are bad, but that's expected |
| §2 Check Gemma 4's prompt | Downloads only Gemma 4's settings and tokenizer and shows the exact prompt it will read |
| §3 Build the Space folder | Copies the code, the saved data and the settings into `space_build/` |
| §4 Upload | Creates the Space, sets the model, asks for ZeroGPU and uploads (this is off until you tick `UPLOAD_SPACE`) |
| §5 Once it's running | What to check, speed, and how to update or stop it |

### Problems I ran into while deploying (and how I fixed them)

| Problem | Fix |
|---|---|
| After I moved the notebook into a subfolder: `No module named 'reggiebot'` | The notebook and `space_app.py` now search upward for the folder that has `reggiebot/` in it |
| Upload refused: `"short_description" length must be less than or equal to 60 characters` | Shortened it to "Student RAG assistant for ISU courses and campus info" |
| The upload kept failing even after that fix | VS Code was running an old copy of the notebook from memory. Closing its tab without saving and reopening it fixed it |
| Every answer failed on the Space: `RuntimeError: No CUDA GPUs are available` | The search was running MiniLM with PyTorch in the Space's main process, which stops ZeroGPU's GPU workers from finding the GPU. The search now uses the same MiniLM model through onnxruntime. I checked first, and it gave identical results on all 50 Arm 1 questions |
| Gemma 4's full processor needs torchvision | ReggieBot only sends text, so it falls back to the tokenizer, which has the same chat template |
| The fallback messages mentioned Ollama, which doesn't exist on the Space | The Space has its own wording ("The model is busy, or today's free GPU time has run out") |

### How they compare

| | Ollama version | Space version |
|---|---|---|
| Model | `gemma4:12b`, compressed (~8 GB) | `google/gemma-4-12B-it`, full bfloat16 (~24 GB) |
| Runs on | my RTX 4070 Ti | ZeroGPU: a GPU borrowed for each answer |
| Address | a new `gradio.live` link every time it starts | permanent |
| Answer time | about 2 s | about 9–13 s, mostly waiting for a GPU to be assigned |
| Limits | none | each visitor's daily GPU time: 2 min without an account, 5 with a free one, 40 with PRO |

I checked it on the live Space. "What are the prerequisites for IT 214?" gave "COM 110 and ENG 101" with the course
link, and "What is my current account balance?" got declined like it should, with links to the billing pages.

---

## Known issues and next steps

- **Chroma's search isn't perfectly repeatable.** On some starts it returns slightly different chunks. It hasn't
  changed a score so far, but it makes Arm 1 less repeatable. Switching to an exact vector search (which only takes
  milliseconds for 6,783 chunks) would fix it.
- **The word "the" matches ISU's theatre subject code `THE`,** so some prompts get a course count added that has
  nothing to do with the question. It hasn't affected any answers.
- **Manual grading** of the Arm 1 answers and of the blind Gemini vs Ollama comparison still needs to be done.
- **The Arm 1 evaluation code** is still in the notebook. It could be moved into `reggiebot/evaluate.py`.
- **The Space is public.** It uses ISU's name and colors, so I need to check with the instructor before sharing it
  with a lot of people.

## Running each version

- **Ollama version:** start Ollama, open `ISU-CampusAssistant-RAG-Ollama.ipynb`, pick the `.venv` kernel, then
  **Run All**. Or just double-click `start_reggiebot.bat`.
- **Test mode (no Ollama):** double-click `start_test_mode.bat`.
- **Tests:** `python -m unittest discover tests -v` (15 tests, takes a few seconds).
- **Space version:** open `ISU-CampusAssistant-RAG-Space.ipynb` and run §1–§3. If you want to update the live
  Space, tick `UPLOAD_SPACE` and run §4.
