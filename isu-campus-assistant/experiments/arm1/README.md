# Experimental Arm 1: the assistant as built (baseline)

Arm 1 measures the ISU Campus Assistant exactly as it is built, with no changes. It is the baseline that every
later arm is compared against: each later arm changes one thing and is run on the same questions and the
same saved data.

| What is held fixed | Value |
|---|---|
| Data | The corpus and Chroma store saved by the last crawl (don't re-crawl between arms) |
| Questions | The 50 questions in `questions.csv` |
| Retrieval | BM25 + MiniLM/Chroma, reciprocal rank fusion, course-code pin, section router |
| Context | Top 8 chunks to Gemini; up to 5 sources listed under each answer in the chat |
| Generator | The Gemini model chain chosen at startup (recorded in `summary.json`) |

## How to run it

1. Open `notebooks/ISU-CampusAssistant-RAG-Demo.ipynb` in Colab and start the assistant as usual (§10).
2. Scroll to **§13 · Experiment — Arm 1**, tick `RUN_ARM1`, and run the cells in that section.
3. It takes roughly 6–10 minutes, because it pauses between Gemini calls to stay under the free-tier limit.
   Untick `ASK_GEMINI` to score retrieval only, in seconds, without using any Gemini quota.
4. The cell shows a per-question table and a write-up draft. Everything is saved to
   `MyDrive/isu_catalog_cache/experiments/arm1/`, and `arm1_results.zip` is downloaded too.
5. **Grade the answers:** tick `GRADE_ARM1` and run the grading cell. It shows every answer with a
   dropdown; choose *correct*, *partial* or *wrong* and press **Save grades**. Grades are saved to Drive, so
   you can stop and carry on later. If you would rather grade in a spreadsheet, grade `results.csv` in Google
   Sheets or Excel, save it into the same Drive folder (CSV or Excel, any name), set `GRADE_WITH` to the
   spreadsheet option and run the cell.

## The question set

50 questions in six categories: 10 course codes, 10 course topics, 8 billing, 8 housing, 8 dining and 6 traps.
Each question has a rule for what counts as the right source:

- **Course codes:** a course with that code. The questions use different spellings, such as `IT 214`, `CTK303`,
  `com 110` and `IT-344`.
- **Topics and website questions:** a chunk from the expected section that contains one of the key phrases.
- **Traps:** nothing in the data answers these. A good reply says so instead of inventing an answer.

Before scoring, every rule is checked against the loaded data. A rule that matches nothing is reported and
left out rather than counted as a miss.

## Metrics

| Metric | Meaning |
|---|---|
| Hit@5 | The right source is among the top 5 search results |
| Hit@8 | The right source is among the 8 chunks given to Gemini |
| MRR | Average of 1 ÷ the rank of the first right source |
| Routing | Billing, housing and dining questions are sent to their section |
| Answered | Gemini wrote an answer instead of the busy or quota fallback |
| Cites source | The answer names the expected course code or links the expected website |
| Declined | A trap reply says it doesn't have the answer (automatic text check) |
| Manual grade | correct / partial / wrong, graded by hand in `results.csv` |

## Files

| File | Contents |
|---|---|
| `questions.csv` | The 50 questions and their scoring rules |
| `results.csv` | One row per question: rank, hits, routing, Gemini's answer, automatic checks, your grade |
| `summary.json` | Setup (data, models, settings) and every summary number |
| `arm1_writeup.md` | The write-up draft, filled with this run's numbers |

`questions.csv` is committed now. The other three files are created by your run in
`MyDrive/isu_catalog_cache/experiments/arm1/`; copy them here (or take them from `arm1_results.zip`).
