# Campus Assistant ISU (unofficial): the Android app

ReggieBot as an Android app for Google Play. The app is the chat screen; the answering happens on a Hugging Face
Space, so it works without this PC. Everything is built and tested from one notebook:
**`ISU-CampusAssistant-Android.ipynb`**.

**Status (November 9, 2026):** the app and its server are built and tested on this PC, including in an Android
emulator. The server is **not uploaded** to Hugging Face yet, and the app is **not on Google Play** yet. The steps for
both are below.

For the rest of the project, see `README.md`. For the history of changes, see `CHANGES.md` (Part 4).

---

## How it works

```
 Android phone                        Hugging Face Space (ZeroGPU)                  private HF dataset
┌────────────────────────┐  HTTPS   ┌──────────────────────────────────────────┐   ┌──────────────────┐
│ Campus Assistant app   │ ───────▶ │ android_space_app.py                     │──▶│ reports/         │
│  chat, source links,   │   ask    │   endpoints: ask · report · status       │   │ usage_log.csv    │
│  Report button,        │ ◀─────── │ space_app.py: Gemma 4, the search, data  │   └──────────────────┘
│  About + privacy       │ streamed │ reggiebot/: the same package             │
└────────────────────────┘          └──────────────────────────────────────────┘
```

The server is the Space edition of ReggieBot under the new name, plus three endpoints the app calls:

| Endpoint | The app sends | It gets back |
|---|---|---|
| `ask` | the question and the last 4 turns of the conversation | the answer about 3 times a second while it is written, then the finished answer with its status and source pages |
| `report` | the answer's id, a reason (offensive, wrong, other) and an optional note | `ok` once the report is saved |
| `status` | nothing | the name, the model, how fresh the data is, and the privacy policy's address |

Reports and an anonymous usage count (time, status, speed; never the question) are copied every 5 minutes to a
private dataset only the developer can read.

## What the app does

- **Chat:** type a question or tap a starter question. The answer appears as it is written, with the pages it used
  listed underneath; tap one to open it.
- **Follow-up questions** work, because the last few turns go back to the server with each question.
- **Report this answer** under every answer. Google Play requires apps that generate content with AI to let users
  flag offensive content without leaving the app.
- **Unofficial notice** on the chat screen and the About screen. The About screen also links the privacy policy.
- **No account, no ads, nothing saved on the phone.** The conversation lasts until the app is closed.
- **Plain-language problems:** "The server is starting up", "Can't reach the server", and a **Try again** button.

## The files

| Path | What it is |
|---|---|
| `ISU-CampusAssistant-Android.ipynb` | Builds, tests, uploads and packages everything (sections listed below) |
| `android_space_app.py` | The app's server: `space_app.py` under the new name, plus the `ask`, `report` and `status` endpoints |
| `android_app/` | The Android app: Kotlin and Jetpack Compose, one screen for the chat and one for About |
| `android_app/app/src/test/` | The app's tests: answer formatting, the server's message format, and a full test against a running server |
| `android_app/outputs/` | The built app files: `-test.apk` (talks to the Space), `-emulator.apk` (talks to this PC), and later the `.aab` for Google Play |
| `android_app/store/` | Store listing text, draft Data safety and content rating answers, the icon, the feature graphic and 4 screenshots |
| `start_emulator.bat` · `stop_emulator.bat` | Double-click to try the app in the emulator on this PC, and to close it |

The only change to the shared code: `source_pages()` was split out of `with_sources()` in `reggiebot/ui.py`, so the
app lists the same source pages as the chat. The chat's output did not change.

### The notebook's sections

| Section | What it does |
|---|---|
| Settings | App name, store title, app id, version, contact email, the Space's name, folders |
| §1 | Tests the server on this PC with a tiny model, calling it over HTTP the way the app does |
| §2 | Builds the Space folder, including the privacy policy (`PRIVACY.md`) |
| §3 | Uploads the Space (off until you tick `UPLOAD_SPACE`) |
| §4 | Checks the live Space the way the app calls it |
| §5 | Installs Java 17 and the Android SDK on D: |
| §6 | Runs the app's tests and builds the test `.apk` |
| §6b | Runs the app in an emulator on this PC, answering with Gemma 4 through Ollama |
| §7 | Builds the signed release (`.aab`) for Google Play with your upload key |
| §8 | Writes the store listing materials |
| §9 | Publishing checklist for the Play Console |
| §10 | Reads the reports and the usage count |

## Try it on this PC (emulator)

1. Start **Ollama**. It writes the answers.
2. Double-click **`start_emulator.bat`**. It starts the app's server in a small minimized window, opens a virtual
   **Pixel 7 running Android 16**, and opens the app. It takes about 10 seconds, or a minute or two after a restart.
3. Use it like a phone: click to tap, drag or use the mouse wheel to scroll, click the question box and type on the
   keyboard. The **◁** and **○** buttons beside the phone are Back and Home.
4. Double-click **`stop_emulator.bat`** when you are done.

The emulator copy of the app talks to the server on this PC. It is a test copy only; never upload it.

## Put the server on Hugging Face

1. In the notebook's settings, set **`CONTACT_EMAIL`**. It appears publicly in the privacy policy and on Google Play.
2. Run the cells in order. In §1 you can set `RUN_LOCAL_TEST = False` to skip the slow test.
3. In **§2**, read the privacy policy it writes. It is a promise: for example, that reports are deleted within 12 months.
4. In **§3**, tick `UPLOAD_SPACE`. It creates the public Space `JakoBob/Campus_Assistant_App` and the private dataset
   `JakoBob/campus-assistant-app-reports`, asks for ZeroGPU, and uploads.
5. Wait 5–15 minutes for the first start; watch **Logs** on the Space's page. Then check it with **§4**.
6. Try it: drag `android_app/outputs/campus-assistant-1.0-test.apk` onto the emulator window, or install it on a phone.

**Limits on the free tier:** each person gets about 2 minutes of GPU time a day without a Hugging Face login, roughly
a dozen questions. Answers take about 10 seconds. After a couple of quiet days the Space sleeps, and the next question
wakes it, which takes a few minutes.

## Publish on Google Play

1. **§7:** create the upload key and build the `.aab`. Back up the key file and its password.
2. **§8:** read and edit the store materials.
3. **§9:** create the app in the Play Console, fill in the listing and the forms, and upload the `.aab`.

Things Google requires or checks:

| | |
|---|---|
| Developer account | One-time $25 fee and an identity check |
| Testing before going public | New personal accounts need **12 testers opted in for 14 days in a row**. **Internal testing** (up to 100 invited people) has no wait and is enough for a class |
| App title | 30 characters at most. "Campus Assistant ISU (unofficial)" has 33, so the store title is "Campus Asst. ISU (unofficial)" |
| Target Android version | Android 16 (API 36) since August 2026. The app targets it |
| AI-generated content | In-app reporting. The app has "Report this answer" |
| Privacy policy | A public web page. §2 writes `PRIVACY.md`, shown on the Space's page |
| Using ISU's name | Google rejects apps that look official when they are not. Check with the instructor or ISU before publishing publicly |

## How it was tested

- **36 Python tests** pass (`python -m unittest discover tests -v`).
- **10 app tests** pass (§6), including one where the app's own networking code got a status, a streamed answer and a
  saved report from the server.
- **The server** was tested on this PC over HTTP, the way the app calls it.
- **The release build** was tested with a throwaway key: it builds a correctly signed `.aab`.
- **In the emulator, with Gemma 4 answering:**
  - "IT 214 prerequisites" was answered correctly ("COM 110 and ENG 101") in about 2 seconds, with the catalog link.
  - Billing and meal plan questions cited ISU's web pages.
  - A follow-up question relied on the previous answer.
  - "What is my current account balance?" was correctly declined.
  - The report dialog, the About page and the source links (opening in Chrome) all worked.

## Setup on this PC

| What | Where |
|---|---|
| Java 17, the Android SDK, the emulator, Gradle's cache (~9 GB) | `D:\tools\AndroidTools` |
| The virtual phone, `CampusAssistant_Pixel7` | `D:\tools\AndroidTools\android-user\avd` |
| The app's build files and the Space folder | `D:\tools\AndroidTools\builds` |
| The upload key (after §7) | `D:\tools\AndroidTools\keys` |

- **No Android Studio:** it isn't needed; everything runs from the notebook or the `.bat` files.
- **Emulator driver:** the emulator uses Google's Android Emulator hypervisor driver. To remove it, run
  `D:\tools\AndroidTools\sdk\extras\google\Android_Emulator_Hypervisor_Driver\silent_install.bat -u` as administrator.
- **VS Code:** `.vscode/settings.json` stops VS Code's Java extensions from building `android_app/`.
- **Versions:** Android Gradle Plugin 8.13.2, Kotlin 2.2.21, Jetpack Compose BOM 2025.10.01, Gradle 8.14.3, Java
  17.0.21. They are proven releases rather than the newest.
