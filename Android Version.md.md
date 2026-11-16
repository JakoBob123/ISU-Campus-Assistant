# Campus Assistant ISU (unofficial) - Android app

This is the Android version of ReggieBot, my chatbot that answers questions about Illinois State University courses, billing, housing and dining. On Google Play it goes by "Campus Assistant ISU (unofficial)".

The phone app is really just the chat screen. The actual thinking happens on a Hugging Face Space, which means my PC doesn't have to be on for the app to work. This README covers how the whole thing fits together and how to build it yourself from the notebook (`ISU-CampusAssistant-Android.ipynb`).

**Quick disclaimer:** this is a student project. It is not made by, endorsed by, or affiliated with Illinois State University. The answers come from an AI model and they can be wrong, so double check anything important like deadlines or costs with the university.

## Contents

1. [What the app does](#what-the-app-does)
2. [How it works](#how-it-works)
3. [What's in the project](#whats-in-the-project)
4. [What you need before you start](#what-you-need-before-you-start)
5. [Settings you have to change](#settings-you-have-to-change)
6. [Setting it up, step by step](#setting-it-up-step-by-step)
7. [Publishing on Google Play](#publishing-on-google-play)
8. [Reading reports and usage](#reading-reports-and-usage)
9. [Things that might trip you up](#things-that-might-trip-you-up)

## What the app does

- **Chat.** You type a question or tap one of the starter questions. The answer shows up while it's being written instead of all at once, and underneath it you get the list of pages it used. Tap a page to open it.
- **Report this answer.** Every answer has this button under it. Google Play requires any app with AI generated content to let people flag offensive stuff without leaving the app, so that's why it's there. Reports go to a private dataset that only the developer can read.
- **Unofficial notice.** It's on the chat screen and again on the About screen. The About screen also links to the privacy policy.
- **No account, no ads, nothing saved on the phone.** The conversation only lasts until you close the app.

Stuff you can ask it:

- What a course covers and what its prerequisites are ("What are the prerequisites for IT 214?")
- Which courses teach a topic ("Which courses teach machine learning?")
- Paying your bill, payment plans and refunds
- Housing options and meal plans

It can't see anybody's student account. So it will never know your balance, your schedule or your housing assignment.

## How it works

### The big picture

```
 Android phone                        Hugging Face Space (ZeroGPU)                   private HF dataset
┌────────────────────────┐  HTTPS   ┌──────────────────────────────────────────┐   ┌──────────────────┐
│ Campus Assistant app   │ ───────▶ │ android_space_app.py                     │──▶│ reports/         │
│  chat, source links,   │   ask    │   endpoints: ask · report · status       │   │ usage_log.csv    │
│  Report button,        │ ◀─────── │ space_app.py: Gemma 4, the search, data  │   └──────────────────┘
│  About + privacy       │ streamed │ reggiebot/: the same package             │
└────────────────────────┘          └──────────────────────────────────────────┘
```

There are three pieces:

1. **The Android app** (`android_app/`, written in Kotlin with Jetpack Compose). This is the chat screen, the source links, the Report button and the About screen.
2. **The server** (`android_space_app.py`), which runs on a Hugging Face Space with ZeroGPU. It's built on top of `space_app.py` and the `reggiebot/` package from the earlier versions of the project. Those two are basically unchanged. The only edit was splitting a helper called `source_pages()` out of `with_sources()` so the app could reuse it.
3. **A private Hugging Face dataset** where the reports and an anonymous usage log end up.

The model that writes the answers is `google/gemma-4-12B-it`. It only answers from pages saved off ISU's public course catalog and websites. That saved data lives in `isu_catalog_cache/` as `corpus.jsonl`, `snapshot.json` and a Chroma vector database (about 67 MB total).

### What happens when you ask a question

1. You type a question in the app and hit send.
2. The app sends the question to the server over HTTPS, along with the last 4 turns of the conversation. The history is there so follow up questions like "how many credit hours is it?" make sense.
3. The server searches the saved ISU data for the pages that match, then hands those to Gemma 4 to write an answer.
4. While the answer is being written, the server sends it back to the phone about 3 times a second. That's why you see the text grow.
5. The last update has `done: true` in it, plus the status and the list of source pages (each one has a label and a URL). The app shows those under the answer.
6. Nothing is stored on the phone. The server doesn't store the question either, unless you report the answer.

### The three endpoints

The server has three things the app can call:

| Endpoint | What the app sends | What comes back |
|---|---|---|
| `status` | nothing | the name, the model, how fresh the data is, and the privacy policy address |
| `ask` | the question, plus the last 4 turns of the conversation | the answer about 3 times a second while it's written. The final update has `done: true`, the status and the source pages |
| `report` | the answer's id, a reason (`offensive`, `wrong` or `other`) and an optional note | `ok: true` once it's saved |

Every call is actually two requests, because the server is a Gradio app:

1. `POST /gradio_api/call/<name>` starts the call and gives back an event id.
2. `GET /gradio_api/call/<name>/<id>` streams the result. The stream has `generating` events while the answer is being written, a `complete` event at the end, or an `error` event if something broke.

One small thing: the `report` call takes five values. The first three are the id, the reason and the note. The test in the notebook sends the last two blank.

### Reports and the usage log

The server keeps two things, and both get copied to the private dataset every 5 minutes:

- **An anonymous usage log** (`usage_log.csv`), one row per question. It has the time, a random code for that request, whether it got answered, how long it took, the number of retries and the model's name. The question itself is not in there.
- **Reports** (`reports/`, saved as `.jsonl` files). A report is only saved if somebody taps "Report this answer". It stores the question, the answer, the reason they picked and whatever they typed in the note.

Both are only used for fixing problems and making the answers better. They are never sold or shared, and they get deleted within 12 months.

### Privacy

The notebook writes the privacy policy (`PRIVACY.md`) from what the app actually does. Short version:

- The app sends your question and the last few questions and answers of the current conversation. That's it.
- No account or sign in. It does not collect your name, email, contacts, location, photos, device identifiers or advertising ID.
- No ads, no analytics libraries, no tracking libraries.
- Hugging Face hosts the server and may keep its own standard technical logs (like IP addresses) under its own privacy policy. No data goes to any other company.
- The app is meant for university students and isn't directed at kids under 13.
- Reports aren't linked to a person. If someone wants a report deleted they email the contact address with the date and the question they asked.

If you change what the app does, you have to keep the privacy policy true. It's a promise you're making to the people who use it.

## What's in the project

| Piece | File or folder | Notes |
|---|---|---|
| The notebook that does everything | `ISU-CampusAssistant-Android.ipynb` | Run the sections in order |
| The app's server | `android_space_app.py` | The Space edition under the new name, plus the `ask`, `report` and `status` endpoints |
| What the server is built on | `space_app.py`, `reggiebot/` | Carried over from the earlier editions |
| The saved ISU data | `isu_catalog_cache/` | `corpus.jsonl`, `snapshot.json`, Chroma vectors |
| The Android app | `android_app/` | Kotlin and Jetpack Compose |
| Finished app files | `android_app/outputs/` | The `.apk` and `.aab` files get copied here so they're easy to find. Git ignores this folder |
| Store listing stuff | `android_app/store/` | Listing text, form answers, icon, feature graphic, screenshots |
| Emulator shortcuts | `start_emulator.bat`, `stop_emulator.bat` | Open and close the emulator setup without the notebook |

The other notebooks in the project and my original Space aren't touched by any of this.

## What you need before you start

- **A Windows PC.** The notebook downloads the Windows builds of Java and the Android SDK and uses `.bat` files, so it's written for Windows.
- **About 4 GB of free space** for the build tools, and around 3 GB more if you want the emulator.
- **Python with Jupyter**, plus the packages the project already uses. The notebook itself imports `requests`, `gradio`, `huggingface_hub` and `Pillow`.
- **The whole project folder.** The notebook has to sit inside the folder that contains `reggiebot/`. When it starts, it looks in the current folder and every folder above it for `reggiebot/`, and it stops with an error if it can't find one.
- **A Hugging Face account** and a write token from huggingface.co/settings/tokens. A free account can host two ZeroGPU Spaces.
- **A `.env` file** in the project folder with `HF_TOKEN=your_token_here`.
- **Ollama with Gemma 4**, but only if you want real answers in the emulator (step 6b).
- **An Android phone**, which is optional. You need to turn on USB debugging to install over USB.
- **A Google Play developer account**, but only if you're actually publishing. It's a one time $25 fee and an identity check.

You do NOT need Android Studio. Everything builds from the notebook.

## Settings you have to change

Everything is in the Settings cell at the top of the notebook. If you're building your own copy, change these first:

| Setting | What it is | What to do |
|---|---|---|
| `CONTACT_EMAIL` | Shown in the privacy policy and on Google Play | **Required.** It's empty by default and several steps refuse to run without it |
| `SPACE_REPO` | The Hugging Face Space for the server | Change `JakoBob` to your own Hugging Face username |
| `REPORTS_REPO` | The private dataset for reports and the usage log | Same, use your own username |
| `APP_ID` | The app's id on Google Play (`com.jakobob.campusassistant`) | Change it to your own. It can never change once the app is published |
| `TOOLS_DIR` | Where Java, the Android SDK, the emulator and Gradle's cache go | It's `D:/tools/AndroidTools` because my C: drive is almost full. Point it somewhere you have room |
| `APP_NAME` | The name inside the app | "Campus Assistant ISU (unofficial)" |
| `STORE_TITLE` | The name on Google Play | Play titles max out at 30 characters and the real name is 33, so the store one is "Campus Asst. ISU (unofficial)" (29). The notebook checks the length for you |
| `VERSION_CODE` | The build number | Raise it by 1 for every upload to Google Play |
| `VERSION_NAME` | The version people see | Starts at "1.0" |
| `SPACE_MODEL_ID` | The model on the Space | `google/gemma-4-12B-it` |
| `GPU_SECONDS` | GPU time allowed per answer on ZeroGPU | 60 |
| `LOCAL_TEST_MODEL` | Tiny model for testing on your own PC | `HuggingFaceTB/SmolLM2-135M-Instruct` |

A few other paths are built from those and you can leave them alone:

- `BUILD_DIR` (`TOOLS_DIR/builds/space`) is the folder that gets uploaded to the Space, about 75 MB.
- `GRADLE_BUILD_DIR` (`TOOLS_DIR/builds/gradle`) holds Gradle's working files, about 150 MB.
- `KEY_FILE` (`TOOLS_DIR/keys/campus-assistant-upload.jks`) is the upload key for Google Play.
- `PRIVACY_URL` points at `PRIVACY.md` inside the Space.

The settings cell also has a `server_url()` function. It asks Hugging Face for the Space's direct address. If the Space doesn't exist yet it guesses the address from the repo name (something like `https://yourname-campus-assistant-app.hf.space`).

## Setting it up, step by step

Run the Settings cell first, every time you open the notebook. Then go through the sections in order. Most cells have a switch at the top (like `UPLOAD_SPACE = False`) that you flip to `True` when you're ready, so nothing big happens by accident.

### Step 1: Test the server on your own computer

Leave `RUN_LOCAL_TEST = True` and run the cell.

This starts `android_space_app.py` in the background on port 7862, the same way it'll run on the Space, except with a tiny model (SmolLM2-135M, about 270 MB, runs on the CPU) instead of Gemma 4. Then it talks to the server over plain HTTP exactly like the Android app does:

1. calls `status`
2. asks "What are the prerequisites for IT 214?" and counts the streamed updates
3. asks a follow up ("How many credit hours is it?") with the history attached
4. sends a test report

After that it stops the server and prints which files the server wrote.

Good to know:

- Loading the data and the model takes about a minute. The cell waits up to 5 minutes before giving up, and if the server never starts it prints the end of the server's log.
- The server writes its reports and log into a temporary folder, never into the project. The test also blanks out the reports repo and the token so nothing gets uploaded.
- The tiny model gives bad answers. That's fine. This step checks that the wiring works, not that the answers are good.

### Step 2: Build the Space folder

Set `CONTACT_EMAIL` if you haven't, then run the cell. It puts together everything the Space needs in `BUILD_DIR`:

| In the folder | What it is |
|---|---|
| `android_space_app.py` | The file the Space runs |
| `space_app.py`, `reggiebot/` | Copied as they are right now (minus `__pycache__`) |
| `isu_catalog_cache/` | `corpus.jsonl`, `snapshot.json` and the Chroma vectors |
| `requirements.txt` | What the Space installs |
| `README.md` | The Space's settings and a short description |
| `PRIVACY.md` | The privacy policy. Google Play needs its address |

The requirements are pinned in the notebook: `spaces`, `torch==2.13.0`, `transformers==5.19.0`, `sentence-transformers==6.1.0`, `chromadb==1.5.9`, `rank-bm25`, `requests`, `beautifulsoup4`, `lxml`, `trafilatura`, `numpy`, `scikit-learn`, `python-dotenv` and `huggingface_hub`. The chromadb version matters because it's the one that built the saved vector database. Gradio isn't in the list because the Space gets it from `sdk_version` in its README, which is set to whatever Gradio version you have installed. The Space runs Python 3.12.

When it's done, the cell prints the big files and the total size. If `BUILD_DIR` already exists but doesn't look like a Space build folder, the cell refuses to delete it.

Run this step again any time you change the code or refresh the data, then upload again.

### Step 3: Upload the Space

This one actually puts things on the internet, so read it first.

1. Make sure `.env` has `HF_TOKEN=...` with a write token. If you added it just now, restart the kernel.
2. Safer option: make a fine grained token that can only write to `REPORTS_REPO` and put it in `.env` as `HF_REPORTS_TOKEN=...`. The Space keeps that one as its secret for saving reports. Without it, the Space gets your regular `HF_TOKEN`.
3. Set `UPLOAD_SPACE = True` and run the cell.

What the cell does:

- creates the private reports dataset
- creates the Space as **public** (it has to be, since the app calls it without signing in)
- sets the Space's variables: `SPACE_MODEL_ID`, `SPACE_GPU_SECONDS`, `REPORTS_REPO` and `PRIVACY_URL`
- saves the token as a Space secret
- asks for ZeroGPU hardware
- uploads `BUILD_DIR`

If the ZeroGPU request fails from the notebook, pick it by hand on the Space's page under Settings, then Hardware, then ZeroGPU.

Then wait. The first start takes several minutes because the Space installs the packages and downloads Gemma 4. Open **Logs** on the Space's page. You'll know it's ready when the log ends with `loaded: google/gemma-4-12B-it ...` and `[reports] copied to the private dataset ... every 5 minutes`.

The cell won't upload if `CONTACT_EMAIL` is empty, if there's no token, or if the build folder is missing or was built before you set the email.

### Step 4: Check the live server

Once the Space says **Running**, set `CHECK_LIVE = True` and run the cell. It calls `status` and then asks Gemma 4 one real question ("When will I get my refund?") and prints how long it took.

This uses a few seconds of your daily ZeroGPU time. If the Space was asleep, the first call wakes it up and that can take a few minutes. If it fails, just wait and run it again.

### Step 5: Install the Android build tools

Building an Android app needs Java 17 and the Android SDK. The notebook downloads both into `TOOLS_DIR`.

| Tool | From | Size |
|---|---|---|
| Java 17 (Eclipse Temurin 17.0.21) | adoptium.net | about 180 MB |
| Android SDK command line tools | dl.google.com | about 150 MB |
| Android 16 platform (`android-36`), build tools 35.0.0, platform-tools (`adb`) | installed by the SDK tools | about 300 MB |
| Gradle 8.14.3 and the app's libraries | downloaded by the first build in step 6 | 1 to 2 GB |

The Android SDK has a license you have to accept before it installs. Read it at [developer.android.com/studio/terms](https://developer.android.com/studio/terms). Then set both `INSTALL_TOOLS = True` and `ACCEPT_ANDROID_SDK_LICENSE = True` and run the cell. It shows download progress every 10%.

If you run it again later, it skips whatever is already installed.

Everything the tools use is kept inside `TOOLS_DIR` too, including Gradle's cache (`gradle-home`) and Android's user folder (`android-user`), so none of it lands on your C: drive.

### Step 6: Build and test the app

Run the cell. It does three things:

1. **Writes `android_app/app.properties`** with the server's address, the privacy policy's address, the app id and the version. The app reads this file when it's built. Don't edit it by hand, change the notebook settings instead.
2. **Runs the unit tests** (`MarkdownTest` for the answer formatting and `CampusServerTest` for the server's message format). With `TEST_WITH_LOCAL_SERVER = True`, it also starts the step 1 server so the app's real networking code gets tested against it: status, a streamed answer and a report, same as on a phone. It prints the test counts after.
3. **Builds the test app** and copies it to `android_app/outputs/campus-assistant-<version>-test.apk`.

The first build downloads Gradle and the libraries, which takes a few minutes. After that, builds take under a minute.

**Getting it on a phone:**

- Over USB: on the phone go to Developer options and turn on USB debugging, plug it in and allow the PC. With `INSTALL_ON_PHONE = True` the cell installs it for you. Then open "Campus Assistant" on the phone.
- Without USB: copy the `.apk` to the phone and open it. Android will ask you to allow installing apps from that source.

The test app talks to the real Space, so do step 3 before this or it won't have anything to talk to.

### Step 6b: Try it in an emulator (optional)

This lets you see the app working on your PC before uploading anything. It runs a virtual Pixel 7 with Android 16 in a window, with the Play Store and Chrome on it like a real phone.

In this mode the app talks to a server running on your PC (port 7863), and the server answers with Gemma 4 through Ollama, so the answers are real. **Start Ollama first.** If you don't have Ollama, set `ANSWER_WITH = "tiny"` to use the small test model instead (bad answers, but it runs).

Set `TRY_IN_EMULATOR = True` and run the first cell. It will:

1. install the emulator, the Android 16 system image and the hypervisor driver's installer if they're missing (about 2 GB)
2. create the virtual phone `CampusAssistant_Pixel7`, with the PC keyboard turned on and 4 GB of RAM so Android 16 runs smoothly
3. start the local server and check its status
4. build a copy of the app that points at `http://10.0.2.2:7863` (10.0.2.2 is how the emulator sees your PC)
5. open the emulator window, install the app and launch it

The first start of the virtual phone takes a minute or two. Then try it out: tap a starter question, or click the text box and type with your keyboard.

**One time thing:** the emulator needs to use your CPU's virtualization or it won't start. If the cell tells you it can't, double click `sdk\extras\google\Android_Emulator_Hypervisor_Driver\silent_install.bat` inside your tools folder and pick **Yes** when Windows asks. You don't need to restart. Then run the cell again. (To remove the driver later, run that same file with `-u`.)

The other two cells in this section:

- **Screenshot cell.** Saves a screenshot of the emulator to `android_app/store/screenshots/` and shows it in the notebook. Google Play only allows screenshots up to 2:1 and the Pixel 7 screen is 20:9, so it trims the status bar and gesture bar evenly to make it fit. You can use these for the store listing.
- **Stop cell.** Set `STOP_EMULATOR = True` and run it when you're done. It closes the emulator and stops the server.

Where things go:

| What | Where |
|---|---|
| The emulator and the Android 16 image (about 3 GB) | `TOOLS_DIR/sdk` |
| The virtual phone | `TOOLS_DIR/android-user/avd` |
| The emulator copy of the app | `android_app/outputs/campus-assistant-<version>-emulator.apk` |
| The emulator's log | `TOOLS_DIR/emulator.log` |

**Never upload the emulator `.apk` anywhere.** It talks to your PC, not the Space, so it won't work for anyone else.

After you've run this section once, you can skip the notebook: double click `start_emulator.bat` in the project folder to open everything (Ollama still has to be running) and `stop_emulator.bat` to close it.

### Step 7: Build the release for Google Play

Google Play doesn't take an `.apk`. It wants an Android App Bundle (`.aab`) signed with your upload key.

Set `BUILD_RELEASE = True` and run the cell. The first time, it creates the upload key at `KEY_FILE` (a 2048 bit RSA key with the alias `upload`) and asks you for a password, which needs at least 6 characters. Then it builds `android_app/outputs/campus-assistant-<version>-<build>.aab`.

Important:

- **Back up the key file and its password.** A USB stick and a password manager work. Google Play holds the key that actually signs the app for users (Play App Signing, which is the default). The upload key only proves an update came from you. If you lose it Google can reset it, but that takes days.
- The password never gets written into the project. If you don't want to type it every time, add `ANDROID_UPLOAD_KEY_PASSWORD=...` to `.env`.
- Every new upload needs a higher `VERSION_CODE`.
- The key is stored outside the project folder on purpose, so it can't end up in git.

### Step 8: Make the store listing materials

Run the cell. It writes these into `android_app/store/`:

| File | What it's for |
|---|---|
| `listing.md` | App name, short description, full description, contact email, category (Education) and the privacy policy link |
| `data-safety.md` | Draft answers for the Data safety form and the other Play Console forms |
| `content-rating.md` | Notes for the content rating questionnaire |
| `icon-512.png` | The 512 x 512 store icon (red with a white speech bubble and three red dots, same shape as the launcher icon) |
| `feature-graphic-1024x500.png` | The 1024 x 500 banner |

The cell checks Google's length limits for you: 30 characters for the title, 80 for the short description and 4000 for the full one.

Read and edit all of these before you paste them in. They describe the app as it is right now, so if you changed something they could be out of date. You also need at least two phone screenshots. Take them on your phone after step 6, or use the emulator ones from step 6b.

## Publishing on Google Play

This part happens in the Play Console, not the notebook.

1. **Developer account.** Sign up at [play.google.com/console](https://play.google.com/console). One time $25 fee plus an identity check. A personal account is fine.
2. **Create the app.** Use `STORE_TITLE` for the name, English as the default language, pick **App** and **Free**, and accept the declarations.
3. **Store listing.** Fill it in from `listing.md`, then add the icon, the feature graphic and at least two phone screenshots.
4. **App content.** Privacy policy address, app access, ads, content rating, target audience, data safety and the rest, using `data-safety.md` and `content-rating.md`. For target audience pick 18 and over (or 16 to 17 plus 18+). Don't pick any age under 13.
5. **Internal testing.** This is the fastest way to get it to people, up to 100 testers. Go to Testing, then Internal testing, then Create release. Upload the `.aab` from step 7 and add testers by email. They install it from a Play Store link, usually within an hour. For a class this is enough.
6. **Closed testing, if you want to go public.** New personal accounts need at least 12 testers opted in for 14 days in a row before Google allows a production release. After that you apply for production access and Google reviews the app.
7. **Updates.** Raise `VERSION_CODE` (and `VERSION_NAME`), run step 7 again and upload the new `.aab`.

**About using ISU's name:** Google Play rejects apps that look like they come from an organization they don't come from. The "(unofficial)" in the name and the notices in the app help, but check with your instructor or with ISU before publishing it publicly.

Things to keep safe:

| What | Where |
|---|---|
| The upload key and its password | `KEY_FILE` |
| The app id | `APP_ID`. It can never change once published |
| The Space and the reports dataset | `SPACE_REPO`, `REPORTS_REPO` |

## Reading reports and usage

Set `READ_REPORTS = True` in the last cell and run it. It downloads the private dataset to a temporary folder and prints:

- how many reports there are, broken down by reason
- the 20 most recent reports, with the time, reason, status, question, answer and note
- how many questions were asked in total, broken down by status
- questions per day for the last 14 days

Google expects reports of offensive content to actually get looked at and acted on, so once the app is out, check this regularly.

## Things that might trip you up

- **"No reggiebot/ folder here or above."** The notebook isn't inside the project folder. Move it back.
- **"CONTACT_EMAIL is empty."** Steps 2, 3 and 7 all need it. After you set it, run step 2 again before step 3, because the upload cell checks that your email is actually in the built `PRIVACY.md`.
- **"No HF_TOKEN found."** Add it to `.env` and restart the kernel.
- **The server "did not take the request."** If it's the Space, it was probably asleep and is still starting. Wait a few minutes and run the cell again.
- **The first answer in the app takes forever.** After a quiet period the Space goes to sleep, and the first answer can take a few minutes while it starts back up. After that answers take seconds.
- **Answers stop coming for the day.** ZeroGPU gives each person a limited number of answers per day.
- **The emulator won't start.** Do the one time driver install from step 6b.
- **Emulator answers fail.** Ollama probably isn't running. Start it and run the cell again.
- **No D: drive.** Change `TOOLS_DIR` in the settings. Also, on my PC, making a folder at the very top of D: needed administrator rights, which is why the tools go in `D:/tools/` and not straight on the drive.
- **Gradle fails.** The cell prints the last 80 lines of Gradle's output when a build fails, and the reason is usually in there.
- **"Install the Android build tools first."** Step 5 didn't finish. Run it again, it picks up where it left off.
- **Play Console rejects the upload.** You probably forgot to raise `VERSION_CODE`.
