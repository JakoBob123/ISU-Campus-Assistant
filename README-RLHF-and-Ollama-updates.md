# ReggieBot: the RLHF notebook and the latest Ollama changes

This readme covers two pieces of work from November 5–6, 2026:

1. **A new RLHF notebook** (`ISU-CampusAssistant-RLHF.ipynb`). It learns which kind of answer I prefer,
   short or detailed, and uses that to choose between two answers.
2. **Changes to the Ollama program**: an exact vector search behind one small wrapper, and four small
   cleanups.

For the full history from Gemini to Ollama, see `CHANGES.md`. For setup and running the app, see `README.md`.

---

## Part 1 · The RLHF notebook

### What it does

```
ReggieBot's search  →  the same 8 excerpts  →  two Gemma 4 answers  →  I pick one  →  a small reward model learns my preference
                                                A: short and simple
                                                B: explains more
```

RLHF (reinforcement learning from human feedback) usually has two stages: first a **reward model** learns
what people prefer, then the main model is retrained using that reward. This notebook does the **first
stage**. It follows the IT244 reward-head notebook, adapted to ReggieBot.

### Only the sources we collected

Both answers come from ReggieBot's own saved data in `isu_catalog_cache/`: the ISU course catalog and the
billing, housing and dining pages. The notebook uses the chat app's own search and its own prompt rules
("use ONLY the catalog excerpts in CONTEXT", cite sources, never invent figures). There is no web search and
no Wikipedia.

A and B get **exactly the same 8 excerpts and the same prompt**. The only difference is one extra line that
sets the style:

| | Style | Instruction |
|---|---|---|
| **A** | Short and simple | One or two plain sentences, under 50 words, no lists |
| **B** | Explains more | About 120–200 words: the direct answer first, then details from the sources |

So when I choose, I'm choosing how the answer is written, not what information it was given.

### The notebook's sections

| Section | What it does |
|---|---|
| §1 Setup | Loads ReggieBot's saved data (no crawling) and checks that Gemma 4 is ready in Ollama |
| §2 Two answers | `retrieve()` runs ReggieBot's search; `write()` asks Gemma 4 for each style. Retries once if Ollama's model runner restarts |
| §3 Collect preferences | A small app: pick a question, press **Generate A / B**, then **Prefer A**, **Prefer B** or **Neither** |
| §4 Encoder | Loads the small frozen model the reward head reads (Qwen2.5-0.5B, on the CPU) |
| §5 Train | Trains the reward head on my choices and writes a report |
| §6 Use it | Best-of-2: for a new question, writes both styles, scores them, and shows the higher-scoring one |

### How the reward model works

- **The input:** question + evidence + one answer, read by **Qwen2.5-0.5B-Instruct**, which stays frozen.
  It turns that into one vector of 896 numbers (the last token's hidden state, scaled to length 1).
  The evidence is included so the head can learn whether an answer fits the sources, not just how it reads.
- **Why Qwen and not Gemma 4:** this Ollama version can't hand out Gemma 4's internal features. A separate,
  small reward model is the normal RLHF setup anyway.
- **What learns:** one linear layer with **896 weights** and no bias. It turns the vector into one score.
- **The loss:** Bradley–Terry, `-log(sigmoid(score_chosen − score_rejected))`, which pushes the chosen
  answer's score above the rejected one's.
- **Training settings:** AdamW, learning rate 0.01, weight decay 0.01, 100 epochs, seed 7. These are fixed,
  not tuned on the held-out questions.
- **Held-out questions:** each question is put in `train` or `heldout` from its own wording (about 1 in 4 is
  held out), before any answers are shown. The same question always lands in the same split, and held-out
  pairs never enter training.
- **A baseline to beat:** A is always the short answer, so "always pick my favorite style" is an easy rule.
  The report shows that baseline next to the head. A head that doesn't beat it has only learned my favorite
  length.

### Results so far (November 6)

| Measure | Result |
|---|---|
| Preferences saved | 12: 9 training, 3 held-out |
| My choices | A (short) 10 times, B (explains more) 2 times |
| Agreement with my choices, training pairs | 100% |
| Agreement on held-out pairs, before training | 33% |
| Agreement on held-out pairs, after training | **100%** |
| Baseline "always pick short", held-out pairs | **100%** |
| Final training loss | 0.135 |

**What this means:** the head learned my training choices and matched all 3 held-out choices. But the
simple rule "always pick short" also got all 3 right. So, so far, I can't tell whether it learned anything
beyond "this person likes short answers". Three held-out questions are also far too few to prove a quality
gain. To find out, I need more ratings, especially questions where the longer answer really is better (for
example, steps or requirements), so short isn't always the right choice.

### How to run it

1. Start Ollama. Open `ISU-CampusAssistant-RLHF.ipynb` in VS Code with the `.venv` kernel.
2. Run §1–§3. In the app, rate about 12 questions; at least 2 training and 1 held-out are needed.
3. Run §4–§6. The first run downloads the encoder (about 1 GB). Reading new answers through the encoder
   takes a little while on the CPU, but the results are saved, and the training itself takes seconds.

**Memory:** ReggieBot plus the encoder use about 3 GB of RAM (5.5 GB reserved), and Ollama needs its own on
top. If Windows says "the paging file is too small", close other heavy programs or make the page file
bigger. That's what fixed it on this PC.

### Files it saves

All in `isu_catalog_cache/experiments/rlhf/`:

| File | What's in it |
|---|---|
| `preferences.jsonl` | One line per rating: question, split, sources, evidence, both answers, which one I chose. No names or IDs |
| `features.pt` | The encoder's saved vectors, so retraining doesn't re-read every answer |
| `reward-head.pt` | The trained 896 weights, plus the numbers used to scale scores |
| `reward-report.json` | The results table above |

### Limits

- **Gemma 4 isn't retrained.** Best-of-2 changes which answer a user sees, not how Gemma 4 writes. Retraining
  it (for example LoRA with RLOO) would be the next stage.
- **A higher score doesn't mean a correct answer.** It means "more like what this person chose".
- **A is always on the left and always short,** so position or length could sway my choices.
- **One rater, 12 ratings.** That's enough for a classroom experiment, not to prove anything.

---

## Part 2 · Changes to the Ollama program

### Exact vector search, behind one small wrapper

**The problem:** ReggieBot finds information partly by meaning. Every chunk of ISU information (6,783 of
them) is stored as a list of numbers, and a question is matched to the chunks whose numbers are closest.
Chroma, the database that stores those numbers, also did the searching, and its search takes shortcuts to
be fast. The shortcuts could come out differently each time the program started, so the same question could
get different chunks. On one start that happened for **14 of the 50** test questions, which made test
results change from run to run.

**The change:**

- Chroma still **stores** the numbers, and still updates only the chunks that changed.
- ReggieBot now **searches** them itself by checking every chunk, with no shortcuts. That's one matrix
  multiplication over 6,783 × 384 numbers, which takes a few milliseconds.
- All the code that talks to Chroma is now in one class, `VectorStore` (`reggiebot/index.py`). Switching
  databases later means changing just that class.

**Checked:**

- I ran the 50 Arm 1 questions three times, each in a fresh start, and all three runs gave identical chunks.
- Scores are unchanged: right source in the top 8 for 44/44, top 5 for 43/44, MRR 0.93, routing 20/24.
- Compared with the October 19 results, every scored question kept its right source at the same rank.
  15 questions got different lower-ranked extra chunks, the same ones Chroma used to shuffle.

### Four small cleanups

| Cleanup | Before | After | Design idea |
|---|---|---|---|
| **Source links** | The chat decided whether to add links by checking how a message started (`ALREADY_LISTED`) | It uses the answer's status: links are added only when the model wrote the answer | DRY |
| **One Answerer** | Loading or refreshing data built a new Answerer when the model wasn't ready, which could swap out the model I picked (like the fake one) | One Answerer is kept and just re-checked (`ready_answerer()` in `core.py`) | High cohesion |
| **Usage log** | `usage_log.csv` saved time, session, status and speed | It also saves `retries` and `model`. An older log gets the new columns automatically, blank for old rows | Factor 5 (logs) |
| **The "THE" quirk** | "the" was read as the theatre subject code THE, and pieces of words like "deposit" were read as IT, which added wrong course counts to the prompt | Subject codes must be whole words, and ones that are everyday words (BE, BUS, HIS, IT, THE) count only in capitals | Correctness |

**Checked:**

- The wrong course counts are gone from 14 of 55 test questions, and real ones like "IT 214" still get theirs.
- Arm 1 retrieval was unchanged.
- One question through the chat page with the fake model got its source links and was logged with the new
  columns. That check wrote to a scratch file, not the real log.

### Tests

There are now **35 tests**, up from 15 before this work. Run them from the project folder:

```
python -m unittest discover tests -v
```

| File | What it checks |
|---|---|
| `tests/test_answerer.py` | Answers, statuses, retries, fallbacks and source links, using the fake model |
| `tests/test_index.py` | The exact search (order, section filter, ties, empty cases) and which subject codes a question names |
| `tests/test_app.py` | One Answerer kept across loads, and the usage log (written to a temporary folder) |

### Earlier Ollama changes (details in `CHANGES.md`)

| # | Change |
|---|---|
| 1 | Answers come from `gemma4:12b` in Ollama instead of Gemini: no API key, no quota |
| 2 | The project write-up (components, diagrams, edge cases, evaluation, cost, sharing) is in the notebook |
| 3 | Speed fix: `127.0.0.1` instead of `localhost` saved about 2 seconds per answer |
| 4 | The code moved into the `reggiebot/` package |
| 5 | A model interface and a fake model, for testing without Ollama |
| 6 | `answer()` returns one `Answer` with text, status, timing, model and retries |
| 7 | `app.py` and the `.bat` files start and stop the app without VS Code |

### After updating

- **Restart** the Ollama notebook's kernel, or restart `app.py`, so the new code loads.
- **Re-run Arm 1** in the notebook if you want a saved baseline that matches the exact search. Scores will
  be the same.
- **The Hugging Face Space doesn't have these changes yet.** It runs the copy of the code that was uploaded
  earlier. It gets them the next time it's rebuilt and uploaded from the Space notebook.
