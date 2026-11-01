# Presentation Review — Team Project 1: The Diagram Check

**A complete study guide for the TP1 presentation.**
Repo: `C:\Users\javro\cpsc415\TP1` · Team: Javier Romo & Ralston Raphael · Course: CPSC 415, AI Integration, Trinity College
Eval run: **October 26, 2026** · Spec approved Oct 18 · Plan approved Oct 19 · Stage per `CLAUDE.md`: *Test and Deploy — Build is complete*

---

## Table of contents

1. [The 30-second version](#1-the-30-second-version)
2. [What this project is](#2-what-this-project-is)
3. [How it works — outside to inside](#3-how-it-works--outside-to-inside)
4. [The seven core concepts](#4-the-seven-core-concepts)
5. [The rules the system enforces](#5-the-rules-the-system-enforces)
6. [What the eval proved](#6-what-the-eval-proved)
7. [Every file, and why it is there](#7-every-file-and-why-it-is-there)
8. [Money, time, and the run log](#8-money-time-and-the-run-log)
9. [The course-process frame](#9-the-course-process-frame)
10. [Defense prep](#10-defense-prep)
11. [Glossary](#11-glossary)

---

## 1. The 30-second version

**One-line pitch:** A student picks a science topic, the app draws a labeled diagram with an image model, and then a *second, independent* model looks at the resulting picture — not at the instructions — and marks it against a hand-written checklist the way a teacher would.

**The one thing to say if you only get thirty seconds:** The interesting engineering here is not the picture, it is the *trust boundary*. The model that draws the diagram is never allowed to grade it, the model that grades it is never allowed to see what it was asked to draw, and the final verdict is not a model's opinion at all — it is a pure function running in Python over the critic's raw observations. Everything else in the repo exists to defend that boundary.

**The result you are presenting:** The eval **failed the critic we shipped**. GPT-4o caught only 9 of 15 planted flaws and confidently graded a mislabeled circuit "good" four separate times. A cheaper model caught 15 of 15, was more accurate per part, and cost about 1/15 as much. That finding is the strongest thing in the project, and it is exactly the kind of finding the eval was built to produce.

---

## 2. What this project is

### The problem

A K–12 student (grades 5–10) learning basic science today looks at a static textbook image or a Google image-search result. Those pictures vary in quality, quietly omit parts, and never tell the student *what* is wrong with the picture. The student gets a diagram; the student does not get a mark-up.

The app gives them both: the diagram **and** the mark-up.

### What the app actually does

1. The student picks one of **five locked topics**: Plant cell, Animal cell, Water cycle, Simple circuit, Flower.
2. The app generates a flat-color 2D textbook-style diagram of that topic, with every part labeled.
3. The app sends **the generated image itself** to a vision model, along with that topic's checklist.
4. The vision model returns structured observations: which parts are present / missing / mislabeled / unclear, which forbidden parts wrongly appeared, whether each of three arrangement rules holds, which labels have garbled or misspelled text, and whether a person appears.
5. The student sees the diagram beside a per-part checklist (✓ / ✗ / ?), the arrangement notes, the label-text problems, a short narrative, and an overall verdict.
6. Every generated image carries a provenance stamp, and the UI carries a visible "AI-generated diagram" caption.

### The five topics and their checklists

| Topic | Required parts | Forbidden (must NOT appear) | Arrangement rules |
|---|---|---|---|
| **Plant cell** | 7: cell wall, cell membrane, cytoplasm, nucleus, chloroplast, vacuole, mitochondrion | — | A1 cell wall outermost, outside the membrane · A2 all organelles inside the membrane · A3 the vacuole is the largest organelle |
| **Animal cell** | 7: cell membrane, cytoplasm, nucleus, mitochondrion, ribosome, endoplasmic reticulum, Golgi apparatus | **cell wall, chloroplast** | A1 membrane is the outer boundary · A2 all organelles inside the membrane · A3 ER is next to the nucleus |
| **Water cycle** | 7: sun, ocean or lake, evaporation, condensation, cloud, precipitation, collection | — | A1 evaporation arrow rises from the water · A2 precipitation falls from the cloud · A3 the arrows form one closed loop |
| **Simple circuit** | 4: battery, wire, switch, light bulb | — | A1 all components lie on one loop · A2 battery shows + and − terminals · A3 loop closed except at an open switch |
| **Flower** | 9: petal, sepal, stamen, anther, filament, pistil, stigma, style, ovary | — | A1 anther on top of the filament · A2 stigma at the top of the style, ovary at its base · A3 sepals below the petals |

Note the **parts vs. labels** distinction: they are the same strings today, and the loader *enforces* that they match. But the concept is separate — "parts" is what the critic checks for; "labels" is what the generator is told to write. Keeping them separate in the data model is what makes it possible to say "draw these words, check for these concepts."

### What it deliberately does not do

From the intent, and worth knowing because "why didn't you build X?" is a likely question:

- No accounts, login, or saved history.
- No multi-turn editing — the critique is informational. Re-running makes a fresh attempt.
- No export, download, share links, or print view.
- No mobile layout (desktop browser is the target).
- No topics outside the five, and no free-form topics — a free-form topic would defeat the whole hardcoded-checklist idea.
- No non-English labels.
- No automatic regeneration when the critique is bad — that doubles the cost, and the student decides.
- No C2PA signed manifests (needs signing certificates we do not have).
- No streaming partial results.

### The models, and why

| Role | Model (as shipped) | The comparison we ran |
|---|---|---|
| Draws the diagram | `google/gemini-2.5-flash-image` ("Nano Banana"), ~$0.039/image | `google/gemini-3-pro-image` ("Nano Banana Pro"), ~$0.138/image |
| Critiques the diagram | `openai/gpt-4o`, ~$0.010/critique | `xiaomi/mimo-v2.6-flash`, ~$0.0005/critique |

The intent fixed Nano Banana and GPT-4o as the *starting* point, and required a real comparison against a cheap and a strong alternative for each. Both comparisons are done and the results are in `CHECKS.md`. Neither switch has been made in code yet — **the intent comes first**, so the recommendation is a new intent document.

---

## 3. How it works — outside to inside

### 3.1 The student's journey (the outside)

The page is styled as a page from a school notes app: a ruled-paper cover with a red margin line, a "Class / Drawn by / Checked by" header block like a textbook caption, and topic cards with small emoji icons.

1. The page loads and immediately asks the server for the topic list. If that request fails, an error callout appears and the Generate button is disabled — the student is never left with an empty page and a working button.
2. The student picks a topic and clicks **Generate diagram**.
3. The button **disables immediately**, a skeleton shimmer appears, and a status line ticks the elapsed seconds: *"Drawing the diagram and checking it · 12 s"*. A second click does nothing.
4. ~13–15 seconds later, the result appears: the diagram on the left with its "AI-generated diagram" caption; on the right a **Checklist** table with one row per required part (✓ Present / ✗ Missing / ✗ Mislabeled / ? Unclear), plus collapsible **Arrangement**, **Label text**, and **Should not appear** panels, plus a narrative in a "What the checker says" quote block.
5. Above it all, the **verdict** — Good / Needs work / Poor — circled like a teacher's red pen mark, next to "6 of 7 parts correct" and the line *"Verdict computed from the marks below, not by the AI."*
6. A **"Needs attention"** filter chip hides the parts that are fine, so a student can see only what went wrong.

### 3.2 The request lifecycle (the inside)

Everything below happens inside **one** HTTP request. The server keeps no state between requests.

| # | Step | Module | What happens |
|---|---|---|---|
| 1 | Browser | `app/static/app.js` | `POST /api/generate` with `{"topic_id": "plant-cell"}`. Button locked. |
| 2 | Validate | `app/main.py` | Unknown or missing topic → **400** naming the valid ids, and **no model call is made**. No API key → **503**. |
| 3 | Draw | `app/generator.py` → `app/prompt_builder.py` | Builds the prompt from the topic's labels plus the shared style block, calls OpenRouter with `modalities: ["image","text"]`, **60 s timeout, no automatic retry**. Decodes the base64 data URL into raw PNG bytes. |
| 4 | Stamp | `app/stamp.py` | Pillow re-encodes the PNG with `tEXt` chunks: `ai_generated=true`, `app=TP1 diagram loop`, `generator=<model>`, `topic=<id>`, `created=<ISO timestamp>`. **Pixels unchanged.** |
| 5 | Critique | `app/critic.py` → `app/openrouter.py` | Sends **the stamped PNG bytes** + the topic's checklist to the vision model. `temperature: 0`, `response_format: json_object`, **30 s timeout, one retry on 5xx/429**. The generation prompt is deliberately **not** included. |
| 6 | Validate | `app/validator.py` | Strips ```json fences, parses, Pydantic-validates, then cross-checks against the catalog. If it fails: **exactly one retry**, with the validation error appended to the critic's prompt. If that fails: the image still ships, with `critique: null` and a readable error. |
| 7 | Verdict | `app/verdict.py` | Computes `good` / `needs_work` / `poor` **in pure Python** from the validated observations. |
| 8 | Gate | `app/main.py` | If the critic reported `people_present: true`, the **image is withheld** (the critique still ships). |
| 9 | Respond | `app/main.py` → `app/static/app.js` | One JSON body: image data-URL, critique, critique error, verdict, arrangement rule text, usage/cost. The page renders it — every dynamic value through `textContent`, never `innerHTML`. |

### 3.3 Who hands what to whom

| Producer | Hands | To | Contract |
|---|---|---|---|
| `app/topics.json` | `Topic` objects (parts, forbidden parts, rules, labels) | generator, critic, validator, verdict, UI | One catalog drives everything. Changing a checklist is a data edit. |
| `app/prompt_builder.py` | the image prompt string | `app/generator.py` | Shared style block + the topic's exact label strings, each quoted. |
| `app/generator.py` | `GeneratedImage(png, model, usage)` | `app/main.py` | PNG bytes, model slug, token/cost usage. Raises `NoImageError` on a text-only reply. |
| `app/stamp.py` | PNG bytes with metadata | `app/main.py` | Same pixels, extra `tEXt` chunks. |
| `app/critic.py` | **raw reply text** (unparsed!) | `app/validator.py` | The critic does not parse its own output. Separation of concerns. |
| `app/validator.py` | a validated `Critique` | `app/verdict.py`, `app/main.py` | Never returns a half-validated critique. |
| `app/verdict.py` | `"good"` / `"needs_work"` / `"poor"` | `app/main.py` | No model involved, no I/O, no side effects. |
| `app/openrouter.py` | `ChatResult(data, usage, latency_s)` | generator, critic, **and `runs.jsonl`** | The single network boundary. Nothing else in the codebase makes HTTP calls. |
| `app/main.py` | the JSON response | `app/static/app.js` | One shape the UI can render without guessing. |

### 3.4 The topic catalog, in detail

`app/topics.json` holds two things: a shared `style_block` string with a `{labels}` placeholder, and the five topics. It is the single source of truth for the image prompt, the critic prompt, the validator's checks, the verdict's threshold, and the UI.

**The catalog is validated at startup, and not just for types.** Pydantic checks field types; the loader then runs five cross-field checks that types cannot catch:

| Rule | Catches |
|---|---|
| Topic ids unique | Two topics sharing an id |
| `required_parts` non-empty | `[]` silently poisoning every critique |
| Arrangement rule ids unique within a topic | A duplicate `A1` |
| `required ∩ forbidden = ∅` | "vacuole" listed as both required and forbidden |
| Required parts unique case-insensitively | "nucleus" and "Nucleus" |
| `labels` set == `required_parts` set | "Golgi body" in one list, "Golgi apparatus" in the other |

Any failure raises an error naming **both the failed rule and the offending topic id**, and the server **refuses to start**. This is plan risk R6: a bad catalog is static-looking data that would otherwise surface as a silently wrong verdict on the first real run. The tests assert the error message contains the rule and the topic, so a human reading a failure finds the bad entry in seconds.

### 3.5 The verdict logic, precisely

Computed in `app/verdict.py`, in this order:

1. **Bad count** = parts whose status is `missing` or `mislabeled`.
2. **`poor`** if any forbidden part was found, **or** if `2 × bad ≥ number of required parts` (half or more of the required parts are missing/mislabeled).
3. **`good`** only if *every* part is `present`, *every* arrangement rule is `ok`, and there are *no* text issues.
4. **`needs_work`** for everything else.

Two details worth knowing because they come up in testing:

- `unclear` does **not** count toward `poor` — it only blocks `good`.
- On the 4-part circuit, exactly 2 missing parts is `poor` (the "half or more" rule is inclusive). On the 7-part plant cell, 3 missing is `needs_work` but 4 missing is `poor`.

### 3.6 Provenance: what the stamp is and is not

`app/stamp.py` writes PNG `tEXt` chunks. This is **provenance, not security**: it honestly labels the file as AI-generated, names the model, the topic, and the timestamp, so a downstream consumer of the PNG can see where it came from. It is *not* a watermark, signature, or authenticity guarantee — anyone with Pillow can rewrite the chunks. The visible "AI-generated diagram" caption in the UI is the load-bearing user-facing signal; the PNG chunks are defense in depth (plan risk R8: some viewers strip PNG metadata). Gemini images also carry Google's invisible SynthID watermark, which we note but do not rely on, because we cannot verify it ourselves.

---

## 4. The seven core concepts

These are the ideas the presentation is really about. Each one is a design rule that shows up in several files at once.

### 4.1 The model never grades itself

The image model draws. The vision model critiques. The **verdict is computed by our code**. The critic is treated as a *sensor* that reports observations; our code is the thing that draws the conclusion. This is the same rule as the Week 5 receipt checker, and it is enforced at the boundary — the critic's JSON schema has no field for a verdict, and there is a test (`test_verdict_comes_from_code_not_the_model`) that feeds a critic reply claiming `"verdict": "good"` while reporting two missing parts, and asserts the API still returns `poor`.

**Why it matters:** an LLM asked to grade its own work is a well-documented failure mode. Removing the question removes the failure mode.

### 4.2 The critic reads the image, not the prompt

The critic request contains the PNG bytes and the topic's checklist. It does **not** contain the generation prompt, and the style block never leaves the generator. This is spec behavior 5, and there is a dedicated test asserting the generation prompt is absent from the critic request.

**Why it matters:** if the critic saw the prompt, it would be grading "did the model follow instructions?" — not "is this picture correct?" Those are different questions, and only the second one helps a student. The eval set exists to defend this: the five flawed images each contain something the prompt never asked for (a missing vacuole, a swapped label, a forbidden chloroplast), so a prompt-reading critic would score them clean.

**The proof that this matters in practice:** the circuit image that GPT-4o graded "good" came from the app's own prompt — a fully compliant, correctly-labeled drawing with the wire label pointing into the battery.

### 4.3 Model output is untrusted input

Everything a model returns passes through a validation chain before it can reach a human:

1. Strip ```json fences, find the first `{` and last `}`.
2. `json.loads` — fail loudly if it is not JSON.
3. Pydantic schema check (`Critique`) — types, enums, required fields.
4. Cross-checks against the catalog: no unknown part names, no duplicates, no omissions, every arrangement rule reported **exactly once** (so a skipped rule cannot pass as "ok"), narrative ≤ 80 words.
5. On failure: **exactly one retry**, with the specific validation error fed back into the critic's prompt so it can fix itself.
6. On second failure: the image still ships, with `critique: null` and `"Critique unavailable — try again."` **Never a half-validated critique.**

The same principle reaches the browser: the JS writes every model-derived string through `textContent`, never `innerHTML`, so a model cannot inject markup into the page.

### 4.4 The catalog is content, not code

The checklists live in a JSON file, not a Python dict. A teacher reviewing the checklists should not have to read Python. Changing "the vacuole is the largest organelle" to something else is a one-line diff, not a code change — and it changes the image prompt, the critic prompt, the validator's checks, the verdict threshold, and the UI all at once, with no code edit.

The trade-off we accepted: JSON has no comments and no types of its own, so we pay for it with a Pydantic model and five startup cross-checks.

### 4.5 One click = one paid call

The Generate button disables on click and re-enables only in a `finally` block. The image call has **no automatic retry** — a second attempt costs money, and the student can simply press the button again. The critique call **does** get one retry, because it is ~20× cheaper and because a schema-invalid reply is usually a fixable formatting slip. The browser test asserts that a second click while the request is in flight produces exactly one HTTP request.

### 4.6 Everything that costs money is measured

Every model call appends one line to `runs.jsonl`: timestamp, model, status, prompt tokens, completion tokens, cost (from OpenRouter's own `usage` field, with `usage: {include: true}` added to every request), latency, and error. The semester estimate in the spec is **summed from that file** and checked against OpenRouter's Activity page. The eval harness redirects that log to a throwaway file per image so eval traffic never pollutes the project's numbers.

### 4.7 Provenance, stated honestly

The stamp says "this came from an AI, here is which model, when, for what topic." It does not claim to be tamper-proof. The UI says "AI-generated diagram" in plain text under every picture. We never imply the image is a photograph, a textbook scan, or a human drawing.

---

## 5. The rules the system enforces

### 5.1 The 17 behaviors from the spec

| # | Behavior | Verified by |
|---|---|---|
| 1 | `GET /api/topics` returns exactly the five topics, each with only `id` and `display_name` | `test_api.py` |
| 2 | The server refuses to start if `topics.json` fails validation | `test_topics.py`, `test_api.py` |
| 3 | Unknown `topic_id` → **400**, and **no model call is made** | `test_api.py` |
| 4 | The image prompt contains the style block, every required label, and "No people" | `test_prompt_builder.py` |
| 5 | The critique request contains the image and the checklist, and **not** the generation prompt | `test_critic.py` |
| 6 | A reply wrapped in ```json fences parses the same as a bare JSON reply | `test_validator.py` |
| 7 | A non-JSON / schema-invalid / incomplete reply triggers exactly one retry with the error included; a second failure returns the image with `critique: null`. Never a half-validated critique | `test_validator.py`, `test_api.py` |
| 8 | `verdict()` returns good / needs_work / poor per the rules, covered by a table-driven test per boundary | `test_verdict.py` |
| 9 | Every PNG returned carries the provenance text chunks | `test_stamp.py`, `test_api.py` |
| 10 | Image model replies with text and no image → **502** with the model's text, and the critic is **not** called | `test_generator.py`, `test_api.py` |
| 11 | `people_present: true` → the image is **withheld**, the critique still ships | `test_api.py`, `test_browser.py` |
| 12 | With no API key: `/api/health` reports `key_present: false` and `/api/generate` returns **503** with no network call | `test_api.py` |
| 13 | Timeouts are 60 s (image) and 30 s (critique). Image timeout → **504**. Critique timeout → the image still ships with `critique_error: "critique timed out"` | `test_openrouter.py`, `test_api.py` |
| 14 | Each call appends one `runs.jsonl` line with model, tokens, cost, latency | `test_openrouter.py` |
| 15 | Browser check: pick a topic, click Generate, the button disables, and the image + caption + one checklist row per part + narrative + verdict all render | `test_browser.py` (Playwright) |
| 16 | **Eval:** the chosen critic raises no false ✗ on the five good diagrams and catches the planted flaw in ≥ 4 of the 5 flawed ones | `CHECKS.md` — **GPT-4o FAILED this** |
| 17 | **Eval:** a real end-to-end run completes in ≤ 45 s and costs ≤ $0.25 | `CHECKS.md` — passed (3 real runs) |

### 5.2 What the student sees when things go wrong

| Failure | HTTP | What the student sees |
|---|---|---|
| Unknown or missing topic | 400 | "Unknown topic 'foo'. Valid ids: plant-cell, …" — and **no money spent** |
| No API key configured | 503 | "OpenRouter key not configured" |
| Image model refused or returned no image | 502 | The model's own text as the reason; the critic is never called |
| Image call timed out | 504 | "The image model timed out. Try again." |
| Critique timed out | 200 | **The image still shows**, with "critique timed out" |
| Critique invalid twice | 200 | **The image still shows**, with "Critique unavailable — try again." |
| Critic found a person | 200 | The image is **replaced** by "This diagram was hidden because it showed a person." The critique still shows. |
| Garbled labels in the picture | 200 | Not an error — this is exactly what the label-text panel reports |
| OpenRouter 429 / 5xx on the image call | 502 | No automatic retry (cost discipline) |
| OpenRouter 429 / 5xx on the critique call | 200 | One retry after 2 s, then the image still shows |
| Double-click on Generate | — | The button is disabled; one click = one paid call |

### 5.3 The test suite

**141 tests across 12 files**, from 87 test functions (many are parametrized, so the collected case count is higher than the function count). **No test needs an API key and no test costs money.**

The layering:

| Layer | Files | What it proves |
|---|---|---|
| Pure logic, no I/O | `test_verdict`, `test_validator` (partly), `test_eval_score`, `test_topics` (partly), `test_stamp`, `test_prompt_builder` | The rules themselves, with no network and no key |
| Unit at the client boundary | `test_openrouter`, `test_critic`, `test_generator` | What we *send* to the model and how we parse what comes back — asserted on the recorded request body |
| API integration | `test_api` (16 tests) | The whole pipeline through `TestClient`: status codes, error paths, the stamp chunks, and the "verdict stays in code" guarantee |
| Real browser | `test_browser` (6 tests, Playwright + real uvicorn) | The only thing that catches frontend regressions: button disabling, double-click safety, the needs-attention filter, the hidden-image notice, error rendering |
| Eval-harness integration | `test_run_eval` (6 tests) | The eval CLI's two modes, with the model faked — that a network error or a malformed reply becomes an invalid row instead of aborting a paid run |

Two mechanisms make this safe and cheap:

- **`conftest.py`'s autouse `offline` fixture** replaces the HTTP transport with one that *raises on any outbound request*, and redirects the run log to a temp file. So even a test that forgets to fake the model cannot make a paid call or write to the real `runs.jsonl`.
- **The `fake_chat` fixture** replaces `openrouter.chat` with a recorder-and-player: it captures every request body for assertions and plays back canned replies in order. Any entry that is an exception is raised instead of returned. There is **no recorded API cassette** — every fake reply is hand-written, which means the tests cannot accidentally regress against a real API snapshot; the eval harness covers that against a live model.

The most important single test is `test_generate_returns_stamped_image_critique_and_code_verdict` — the full happy path through image → stamp → critique → validate → verdict in one assertion set.

---

## 6. What the eval proved

### 6.1 The eval set

Ten diagrams in `eval/images/` — **one good and one deliberately flawed per topic** — with hand-checked ground truth in `eval/expected.json`.

| Topic | The planted flaw |
|---|---|
| Plant cell | **Vacuole missing** (so arrangement rule A3 also fails) |
| Animal cell | **A chloroplast drawn and labeled** — a forbidden part in an animal cell |
| Water cycle | Evaporation spelled **"evaparashun"** |
| Simple circuit | **The "wire" label points into the battery** and the "battery" label floats — a real image from our first end-to-end run, which GPT-4o graded good at the time |
| Flower | **The anther and stigma labels are swapped** (the model also left out "pistil" — an unplanned extra miss) |

The good/flawed pairing is the whole design: it separates **hallucination** from **perception**. A critic that hallucinates invents complaints on the good images; a critic that only reads label *text* misses the wire-mislabel and the anther/stigma swap, which are about *where the leader line points*.

**How the set was made — and honestly.** The first attempt with Nano Banana produced garbled or misplaced labels in every "good" image, so none could serve as a correct reference. The final set was made with Nano Banana Pro, some of it after several tries. Two images were **reassigned mid-build**: one image the model was told to flaw came out clean, so it became the good image; and the good circuit from the app's own prompt failed A2 (no `+`/`−`), so the final one adds "mark the + and − terminals" to the prompt. Every prompt is recorded in `eval/images/prompts.json` (the original recipe) and `eval/images/prompts-replacements.json` (the patch log of what actually worked).

**The caveats you should state out loud:**
- **Claude Code checked each image label by label** and wrote `expected.json`. Ralston and Javier should spot-check the five flawed entries before the presentation.
- **Selection bias:** the flawed circuit was chosen *because* GPT-4o had already passed it. So the set leans toward GPT-4o's known weakness — although the flower swap was planted fresh and GPT-4o missed it too, so the weakness is not only an artifact of the selection.

### 6.2 The scoring rubric

There is no single numeric score. Five independent dimensions:

1. **Output validity** — did the reply parse, match the schema, name every required part exactly once with a legal status, name no invented parts, report every arrangement rule once, and keep the narrative ≤ 80 words? (Also tracked separately for first-try validity, before any retry.)
2. **Per-part accuracy** — for every required part, did the critic's present/not-present judgment match the hand-checked truth? A fraction from 0 to 1.
3. **False flags on good images** — complaints the critic raised about a diagram we know is correct. Counted per kind: a required part wrongly called missing/mislabeled/unclear, a forbidden part invented, an arrangement rule wrongly failed, a text issue raised where there is none. (On flawed images this is always 0 by definition — complaints there are correct.)
4. **Flaw detection on flawed images** — did the critic name the planted flaw *specifically*? A missing part counts only if that part is reported as not-present; a forbidden part only if that exact name appears; an arrangement rule only if that rule id is failed; a text flaw only if the issue names the planted label. (The "names the specific label" rule was added during review of PR #9 — the first scoring counted *any* text issue as a catch, which was too generous. A re-run confirmed both models still flag "evaparashun" specifically, 3 of 3, so the counts stand.)
5. **Cost and latency** — tokens and USD from OpenRouter's `usage` field.

**The pass/fail gate is behavior 16:** zero false flags across the good images **and** at least 80 % of the flawed runs caught, pooled across repeats. With 5 flawed images × 3 repeats, that means **at least 12 of 15**.

### 6.3 Critic comparison — the headline result

`python eval/run_eval.py critic --model <model> --repeats 3` — 10 images × 3 repeats = 30 calls per model, same images, same prompt.

| Critic | Valid first try | Mean part accuracy | False flags on good | Flaws caught | Tokens in / out | Cost (30 calls) | **Behavior 16** |
|---|---|---|---|---|---|---|---|
| `openai/gpt-4o` (as shipped) | 100 % | 0.92 | 0 | **9 / 15** | 45,719 / 10,691 | $0.20 | **FAIL** |
| `xiaomi/mimo-v2.6-flash` (cheap) | 100 % | 0.97 | 0 | **15 / 15** | 43,764 / 37,750 | $0.013 | **PASS** |

Scored per image instead of pooled, the result is the same: GPT-4o 3 of 5, mimo 5 of 5.

**What GPT-4o missed — identically, all three times:**
- **`simple-circuit-flawed`**: it marked every part present and gave the verdict **good**. The same false pass as on October 25.
- **`flower-flawed`**: it did not notice the anther/stigma swap (verdict `needs_work`, but only for the missing pistil label).

**What the two misses share:** both are a label pointing at the wrong part. **GPT-4o reads label *text* reliably, but does not trace where leader lines go.** That is a precise, defensible characterization of the weakness — and it is exactly what the cheap model caught:

- *"The wire label points to the battery, not a drawn wire."*
- *"[anther] Label points to the top of the pistil, not the pollen tip."* (plus the missing pistil label and the stigma swap)

**The trade-offs of the cheap model:**
- **Hidden reasoning:** mimo writes far more output tokens — one flower reply used 6,806 against a normal ~700. Still $0.002, but latency varies.
- **The 30 s timeout is the risk to watch.** No timeouts occurred in these runs.

**The decision rule from the spec:** *"If the cheap model matches GPT-4o on false flags and per-part accuracy within one part per image, we propose switching in a new intent."* mimo matches on false flags (both 0), **beats** GPT-4o on accuracy, and catches six more flaws at about 1/15 the cost. **The recommendation is a new intent to switch the critic to `xiaomi/mimo-v2.6-flash`.** The code is not changed here, because the intent comes first.

### 6.4 Generator comparison

`python eval/run_eval.py generate --model <model> --runs 3` — 5 topics × 3 runs = 15 images per model, all from the app's unchanged prompt. Scored by **looking at every image** (Claude Code did the scoring; the contact sheets in `eval/generator-comparison/` are Flash on the top row, Pro below, so anyone can check).

| Topic | Flash: labels ok | Flash: garbled | Flash: flat | Pro: labels ok | Pro: garbled | Pro: flat |
|---|---|---|---|---|---|---|
| plant-cell | 0/3 | 3/3 ("CELL MEMRANE", "chlorclospast", "vacuule") | 3/3 | 3/3 | 1/3 ("chloplast") | 3/3 |
| animal-cell | 0/3 | 3/3 ("mitohinotrion", "enderapemic retriclum") | 3/3 | 3/3 | 0/3 | 3/3 |
| water-cycle | 0/3 | 3/3 ("evaparation", "preciptation") | 3/3 | 3/3 | 0/3 | 1/3 (runs 2–3 shade water and mountains) |
| simple-circuit | 2/3 (run 2's labels float) | 0/3 | 3/3 | 3/3 | 0/3 | 3/3 |
| flower | 0/3 (labels on wrong parts, "style" twice) | 0/3 | 3/3 | 3/3 | 0/3 | 3/3 |
| **Total** | **2/15** | **9/15** | **15/15** | **15/15** | **1/15** | **13/15** |

| Model | Cost per image | Mean latency | 15 images |
|---|---|---|---|
| `google/gemini-2.5-flash-image` (Nano Banana, as shipped) | $0.039 | 6.4 s | $0.58 |
| `google/gemini-3-pro-image` (Nano Banana Pro) | $0.138 | 19.6 s | $2.08 |

**The decision rule:** *"We keep Nano Banana unless Pro removes garbled labels on topics where Nano Banana fails more than 1 in 3 runs. If that happens, we update the intent before switching."*

- **The rule triggers.** Nano Banana garbles or misplaces labels on four of five topics in *every* run. Pro removes most of it: 1/15 still garbled ("chloplast"), and 2/15 not flat.
- **The catch is cost.** A Pro image plus a mimo critique is about **$0.14 a run** — still under the $0.25 per-run ceiling, and about 20 s plus the critique, under the 45 s ceiling. But the spec's 250-run semester estimate becomes about **$35, over the intent's $25 budget**.
- **That trade-off is a team decision for the new intent** — correctness of the diagram versus the semester budget. This is the single most interesting open question in the project.

### 6.5 End-to-end runs (behavior 17)

| Date | Topic | Time | Cost | Within 45 s and $0.25? |
|---|---|---|---|---|
| Oct 25 | simple-circuit | 13.8 s | $0.044 | yes |
| Oct 25 | animal-cell (browser) | 15.0 s | $0.047 | yes |
| Oct 26 | flower (browser) | 12.9 s | $0.046 | yes |

### 6.6 The four findings for the presentation

1. **The critic was confidently wrong.** GPT-4o graded a mislabeled circuit "good" four times (once on Oct 25, three times in the eval). The eval existed to catch exactly this, and it did. Note the word *confidently*: it did not hedge or fail — it produced a valid, well-formed, 100 %-first-try-valid critique that was simply wrong about the picture. **Schema validity is not correctness.** That is the most transferable lesson in the project.
2. **The cheaper model was better at this task:** 15/15 against 9/15, at 1/15 the price.
3. **The prompt and the checklist disagree.** Rule A2 requires `+` and `−` battery terminals, but the image prompt never asks for them — so a correct-looking circuit can fail A2. This should be fixed in `topics.json` or the prompt builder through the new intent. (We found this *because* the eval set's good circuit had to be regenerated with "mark the + and − terminals" added to the prompt.)
4. **Nano Banana could not produce a single fully correct cell, water-cycle or flower diagram in 9 tries.**

---

## 7. Every file, and why it is there

### 7.1 Repo root — documents and configuration

| File | Role | What breaks without it |
|---|---|---|
| `CLAUDE.md` | **Project conventions.** Read by the agent at every session start. It states what the repo is, the four commands, the branch/PR rules, the working rules, and a running list of "common mistakes" (stacked PRs, "a animal cell", editing `expected.json` to flatter the critic). Graded: does it reflect how the team actually works? | The agent loses the house rules; the grade on the conventions artifact is lost. |
| `README.md` | The course's **artifact-chain template**: the intent → spec → plan → build → test → deploy chain, which file each stage produces, who writes vs. approves it, the graded rules, and how to submit. Explains the `tp1-submitted` tag flow. | A grader cannot see how the process artifacts fit together. |
| `intent/generation-loop.md` | **The intent** — the human-approved statement of the problem, audience, constraints, out-of-scope, success criteria, and open questions. Approved Oct 5, 2026. The source of truth the spec is validated against; the intent wins if they disagree. | The whole chain loses its origin. Everything else is derived from this. |
| `intent/TEMPLATE.md` | The shape every future intent must follow (goal / who / constraints / out of scope / success / open questions / approval). | New intents have no consistent shape, and the chain is not reproducible. |
| `spec.md` | **The design.** Every component states its language, its model, and the alternative that was weighed. Contains the topic catalog, the critique schema, the verdict rules, **17 numbered behaviors**, the failure-handling table, the cost estimate, and out-of-scope. Approved Oct 18, 2026. | The build has no contract; `REVIEW.md` pass 1 ("code must match spec.md") becomes unenforceable. |
| `plan.md` | **The build order** — 14 steps, the file each step creates, the *proof* for each step (exact command + test functions + the output that means done), and **ten named risks R1–R10 with mitigations**. Approved Oct 19, 2026. | No evidence of planning; the "plan approved before implementation" grade is lost. |
| `CHECKS.md` | **The results.** The eval set, the critic comparison, the generator comparison, the end-to-end runs, the four findings, and the spend. This is the file that carries the project's actual evidence. | The project has claims but no measurements. |
| `ANNOTATION.md` | **The four-question template** — what is this / why this choice / what breaks / what did I learn. Explicitly *"written by you, not the agent"*; it is what the Comprehension Defense asks about. Currently the template with no answers filled in. | **Action item: this is still empty and it is graded.** See §10 for prompts, but it must be your words. |
| `REVIEW.md` | **The review policy** a separate reviewing agent applies to every PR: four passes by severity (must-not-merge, must-fix, should-fix, nice-to-have) and a one-line-per-finding reporting format. | PRs get merged without an independent gate; a graded process artifact is missing. |
| `requirements.txt` | fastapi, uvicorn, pydantic, httpx, Pillow, python-dotenv, pytest, playwright. | Nothing installs; nothing runs. |
| `.env.example` | The shape of the secret file: `OPENROUTER_API_KEY=<your-key>`. The real `.env` is gitignored. | New teammates and graders cannot see what configuration is needed. |
| `.gitignore` | Layered by purpose: secrets (`.env`, `.claude/settings.local.json`), the run log, test artifacts, and build outputs across Python/Node/Java. | Secrets and per-call telemetry get committed. |
| `.claude/settings.local.json.example` | The template that points **Claude Code itself** at OpenRouter (`ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, four model slots + a subagent model). Filled in by each person; the real file is gitignored. | Nobody can run the agent against the project's own key without re-deriving the setup. |
| `runs.jsonl` | The live run log: one line per model call with tokens, cost, latency. Gitignored because it is per-call telemetry that changes constantly. | The cost estimate cannot be checked against real spend. |

### 7.2 `app/` — the application

| File | Role | What breaks without it |
|---|---|---|
| `app/__init__.py` | Package marker. | `from app.X import Y` fails everywhere. |
| `app/main.py` | **The FastAPI server.** The three endpoints (`/api/topics`, `/api/generate`, `/api/health`), the orchestration (generate → stamp → critique → validate → verdict), the error mapping (400/503/502/504), the person-image gate, and serving the static frontend. Loads and validates the catalog **at startup**, so a bad catalog means the server never starts. | No server, no UI, no product. |
| `app/schemas.py` | **The data model.** The `Topic` / `ArrangementRule` models, the `Critique` / `Part` / `ArrangementResult` / `TextIssue` models, `TopicCatalogError` (which names the failed rule and the topic id), the five cross-field catalog checks, and the cached catalog + style-block loaders. Also `Topic.with_article` — the "a plant cell" / "an animal cell" helper. | No typed contracts; the catalog's cross-checks never run; a bad catalog silently serves and produces silently wrong verdicts. |
| `app/topics.json` | **The catalog.** The five topics with their required parts, forbidden parts, arrangement rules, and label strings, plus the shared `style_block`. | The server cannot start; nothing can be drawn or checked. |
| `app/prompt_builder.py` | Assembles the image prompt: `"A labeled diagram of {topic.with_article}. "` + the style block with `{labels}` filled in with the topic's quoted label strings. | The image model gets no style guidance; outputs become inconsistent and off-spec. (Also the fix point for the A2 `+`/`−` mismatch.) |
| `app/generator.py` | **The image call.** One POST with `modalities: ["image","text"]`, 60 s timeout, **no retry**; decodes the base64 data URL into PNG bytes; raises `NoImageError` when the model replies with text and no image. Returns `GeneratedImage(png, model, usage)`. | No diagram can be produced. (Also the fix point for the "a animal cell" article bug that PR #3 caught.) |
| `app/critic.py` | **The critique call.** Builds the checklist prompt inline (required parts with their exact names and the four legal statuses, forbidden parts, the arrangement rules by id, the text-issue vocabulary, `people_present`, and the narrative limit), sends the PNG as a base64 `image_url` part, `temperature: 0`, `response_format: json_object`, 30 s timeout, one retry on 5xx/429. **The generation prompt is not included.** Returns raw text. | The vision step cannot run; the UI has no checklist; the verdict is undefined. |
| `app/validator.py` | **The trust boundary.** Strips ```json fences, parses, Pydantic-validates, then cross-checks: no unknown part names, no duplicates, no omissions, every arrangement rule exactly once, narrative ≤ 80 words. Owns the one-retry loop (`critique_with_retry`) and returns `(critique, error)` — never a half-validated critique. | Raw model output reaches the UI unchecked; an invented part silently appears as a checklist row; a skipped arrangement rule silently counts as "ok". |
| `app/verdict.py` | **The verdict, computed in code.** A pure function over a validated `Critique` and the topic: `poor` if a forbidden part was found or half-or-more of the required parts are missing/mislabeled; `good` only if all present, all rules ok, no text issues; `needs_work` otherwise. | The model would be grading itself. The central guarantee of the project disappears. |
| `app/stamp.py` | **Provenance.** Pillow writes `tEXt` chunks (`ai_generated=true`, `app`, plus generator/topic/created) into every PNG. Pixels unchanged. | Downstream consumers cannot tell an AI-generated diagram from anything else. (The critique still works — it takes PNG bytes.) |
| `app/openrouter.py` | **The only network boundary.** The key check, the bearer header, `usage: {include: true}`, explicit timeouts (no default — each caller states its spec value), the one-retry-after-2 s policy for 5xx/429, usage extraction, and the `runs.jsonl` append. `TRANSPORT` is the seam the tests swap. | Every LLM call stops. (Also: cost accounting, retry policy, and the run log all live here, not scattered.) |
| `app/checker.py` | ⚠️ **Orphaned legacy.** A pure-Python checker with a hand-coded `answer_key` comparison (`parse_critique`, `compare_part`, `build_report`, `compare_critics`) from an earlier, parallel design. **It is not imported by `main.py`, `validator.py`, `verdict.py`, the eval, or any test.** The UI label "What the checker says" refers to the validator's narrative, not to this file. | Nothing — deleting it changes zero behavior. Keeping it costs only confusion. **Do not demo it as part of the live loop.** |
| `app/static/index.html` | The page skeleton: the SVG icon sprite, the notebook-cover styling, the class-notes header, the topic form, the loading skeleton, the two-column result area, the checklist table, the three collapsible panels, the narrative quote block. | The page 404s; the student has nothing to look at. |
| `app/static/app.js` | The page's logic: load topics and render cards, POST the generate request, tick the elapsed-seconds status, render the checklist/lists/narrative/verdict, and handle the needs-attention filter. **Every model-derived string goes through `textContent`, never `innerHTML`.** | The page is inert — topics do not load, Generate does nothing. |
| `app/static/style.css` | Design tokens and components: the notebook-paper cover with the red margin line, the topic cards, the callouts, the checklist as a fixed-column table, the collapsible notes, the shimmer skeleton, a `prefers-reduced-motion` fallback, and the single hand-drawn element — the verdict, circled like a teacher's mark. | The page is unstyled; the verdict loses its character; the loading state is invisible. |

### 7.3 `eval/` — the measurement

| File | Role | What breaks without it |
|---|---|---|
| `eval/__init__.py` | Package marker, so `eval` is importable as a package. | `from eval.score import …` fails; the runner cannot be imported by the tests. |
| `eval/run_eval.py` | The eval CLI with two subcommands: `critic` (score a critic model over the eval set) and `generate` (produce N images per topic for hand-scoring). **Reuses `app.critic`, `app.validator`, and `app.generator` directly**, so it measures exactly what ships. Redirects the run log per image so eval traffic never pollutes the project's numbers. One failed call becomes an invalid row instead of aborting a paid run. | No model comparison; behaviors 16 and 17 are unverified claims. |
| `eval/score.py` | The scoring, as pure functions with no I/O: `score_critique` (one reply vs. the hand-written truth → part accuracy, false flags, flaw caught, verdict) and `summarize` (aggregate → the headline `behavior_16` boolean). | A bug here silently changes every number in `CHECKS.md` and can flip a FAIL to a PASS. |
| `eval/expected.json` | **Hand-written ground truth** for all ten images: per-part status, forbidden parts, per-rule booleans, text-issue flag, and a `flaw` descriptor (`kind` + `target`) for the flawed ones — plus a `note` on how each image was made. | The eval is meaningless: false flags and flaws-caught become arbitrary. **Never edit it to make the critic look better** — a disagreement is a finding for `CHECKS.md`. |
| `eval/images/*.png` | The ten eval diagrams (5 good + 5 flawed). Each carries the production stamp chunks, and most an extra `eval_role` chunk recording which numbered attempt it was. | The eval cannot run at all. |
| `eval/images/prompts.json` | The original recipe for the eval set: the flawed prompt per topic, and `"good": "app.prompt_builder.build_image_prompt(topic)"` — a symbolic pointer meaning *use the app's production prompt, unchanged*. | The eval set is not reproducible or defensible. |
| `eval/images/prompts-replacements.json` | The patch log for the four images that had to be regenerated after the first attempt failed the intended condition. | A future rebuild of the eval set repeats the same mistakes. |
| `eval/generator-comparison/*.jpg` | Five contact sheets (Flash top row, Pro below) so a human can audit the hand-scored generator table. | The generator comparison cannot be defended in the presentation. |

### 7.4 `tests/` — the proof

| File | Role | What a failure would mean |
|---|---|---|
| `tests/conftest.py` | The global harness: the autouse `offline` fixture (blocks all network, redirects the run log) and the `fake_chat` fixture (records requests, plays back replies). Plus `png_bytes`, `data_url`, and `reply` helpers. | Without it, a test could make a paid call or write to the real `runs.jsonl`. |
| `tests/test_topics.py` | The catalog loads with the five ids in order; the five invariants hold; five bad fixtures are rejected with a message naming both the rule and the topic. | Bad topic data reaches the live API and breaks every critique silently — the highest-leverage failure mode in the project. |
| `tests/test_stamp.py` | Every stamped PNG carries the four chunks; pixels are unchanged; a raw PNG has none of them. | Provenance disappears. |
| `tests/test_prompt_builder.py` | The prompt contains the style block, every label quoted exactly, "No people", and the right English article (`a plant cell` / `an animal cell`). | Every generated diagram silently degrades. |
| `tests/test_openrouter.py` | Bearer auth; missing key raises before any network call; timeouts pass through; `retry_5xx` retries 429/5xx once then succeeds, and gives up after one retry; the image call has `retry_5xx=False` so a 5xx never double-charges; client errors are not retried; every call appends one `runs.jsonl` line; failed attempts are logged too. | The HTTP layer becomes unreliable; the image call silently double-spends. |
| `tests/test_generator.py` | The image request sets the model, `modalities`, the built prompt, and the 60 s timeout with no retry; the data URL decodes back to the original PNG; a text-without-image reply raises `NoImageError` carrying the model's text. | The generation pipeline breaks or wastes money on retries. |
| `tests/test_critic.py` | The critic request carries the image as an `image_url` **and** the full checklist — every required part, every forbidden part, every arrangement rule — and **never** the generation prompt or the style block. Also the model, `temperature=0`, `response_format`, the 30 s timeout, the 5xx retry, and that raw text (fences included) is returned as-is. | A leak of the generation prompt into the critic request makes behavior 5 unenforceable and the whole eval set meaningless. |
| `tests/test_validator.py` | Fences stripped; JSON inside prose handled; part names canonicalized case-insensitively; every invalid shape rejected (not JSON, bad status, missing/duplicate/invented part, missing rule, over-long narrative); the retry passes the error back exactly once; a second failure returns `(None, CRITIQUE_UNAVAILABLE)`. | Invalid critiques reach the UI, and a half-checked checklist can appear. |
| `tests/test_verdict.py` | One table-driven test covering every boundary: all-clean → good; one unclear / one failed rule / one text issue → needs_work; 3 of 7 missing → needs_work but 4 of 7 → poor; exactly half of the 4-part circuit missing → poor; `unclear` does not count toward poor; any forbidden part → poor. | The verdict gate misfires — students are shown the wrong thing. |
| `tests/test_api.py` | The API contract end to end: five topics with id and display name only; a bad catalog stops the server; unknown/missing topic → 400 with **no model call**; no-key health + 503 + no call; the full pipeline returns a stamped PNG + critique + a code-computed verdict + usage; **the verdict ignores what the model claims**; an image refusal → 502 with the model text and no critic call; a provider error → 502; a person → the image is withheld; an image timeout → 504; a critique timeout → the image still returns; two bad critiques → image + error; the missing-topic message names the topic id, not "None". | Any single failure breaks the production user flow. |
| `tests/test_browser.py` | The real browser: five topic cards render; clicking Generate disables the button and shows the loading state with elapsed seconds; a second click is a no-op; the response renders the diagram, caption, one checklist row per required part with the right marks, the arrangement text, the text issue, the narrative, the verdict, and the score line; the needs-attention filter shows only problem rows and restores; the empty state appears when everything is clean; a failed topics load shows an error and disables Generate; a critique error still shows the image; a hidden image shows the notice instead; a 502 is shown and the button comes back. | The only layer that catches frontend regressions. |
| `tests/test_eval_score.py` | The scoring rubric itself: a perfect reply on a good image; a `None` critique scores zero; `unclear` and `missing` count alike when the truth is missing; each false-flag kind counted separately with case-insensitivity; an expected-forbidden part is not a false flag; an expected-false rule is not a flag; multiple text issues count once; false flags are 0 on flawed images; flaw-caught per kind and target (including the text-with-target rule PR #9 fixed); the verdict comes from app code; `summarize` totals and the behavior-16 logic; empty input is safe. | The rubric silently changes — an off-by-one in part accuracy, or a 0.8 threshold tweak, re-grades the same model as PASS. |
| `tests/test_run_eval.py` | The eval CLI's two modes with the model faked: first-try validity and the retry count (3 critic calls for 2 images when one fails first try); per-image run-log isolation; `--repeats` and `--out`; a transport failure and a malformed reply become "no (invalid)" rows instead of aborting the run; `run_generate` writes stamped PNGs with the right chunks and leaves the three hand-scored columns blank. | A bug in the runner invalidates every row in `CHECKS.md` — and a timeout that aborted a paid run would waste real money. |
| `tests/__init__.py` | Package marker. | `from tests.conftest import …` breaks in six files. |

### 7.5 `images/` — the Week 6 set (not part of the submitted main branch)

This folder exists only on the in-flight branch `feature/week06-image-set`, one commit ahead of `main` (`6199bd7 Integrate Week 6 image set and pure-Python checker`). It is **not** in the submitted TP1 work and should not be presented as part of it.

| File | Role |
|---|---|
| `images/manifest.json` | The spec for this image set: per topic, the prompt, a seed, an `answer_key` mapping part name → bool, the generator model, the image path, the media type, and the cost. Consumed by `app/checker.py`. |
| `images/*.png` (5) | One diagram per topic, generated with `black-forest-labs/flux.2-klein-4b`. |
| `app/checker.py` | The pure-Python checker for that Week 6 lab — the answer-key comparison design that predates the topic-catalog approach now used by `validator.py` + `verdict.py`. |

**Three image directories, three different purposes — worth knowing so you don't mix them up:**

| Directory | Purpose | Consumed by |
|---|---|---|
| `images/` | The Week 6 lab's input corpus (5 PNGs + manifest) | `app/checker.py` (Week 6 lab) |
| `eval/images/` | TP1's eval corpus for the **critic** (10 PNGs, `-good` / `-flawed`) | `eval/run_eval.py critic` + `eval/expected.json` |
| `eval/generator-comparison/` | Five hand-built contact sheets comparing Flash vs Pro | Nothing — humans only, so the hand-scored table is auditable |

They overlap by *topic* (all five science topics appear in both `images/` and `eval/images/`) but are **independent corpora** made with different generators, and they are not interchangeable.

---

## 8. Money, time, and the run log

### 8.1 The cost model

| Call | Tokens (approx.) | Cost per call |
|---|---|---|
| Image, `google/gemini-2.5-flash-image` | ~150 in, ~1,290 image tokens out | ~$0.039 |
| Critique, `openai/gpt-4o` | ~765 image + ~700 text in, ~600 out | ~$0.010 |
| **One run** | | **~$0.05** (≤ $0.06 with a critique retry) |

The intent's ceiling is **$0.25 per generation + critique**, so a run is about **5× under it**. The measured runs came in at $0.044–$0.047.

**Semester estimate:** ~250 development and demo runs ≈ $12.50. Generator comparison: 15 Pro images + 15 Nano Banana images ≈ $2.60. Critic comparison: 10 images × 2 models × 3 repeats ≈ $0.35. **Total ≈ $16**, under the intent's $25 and the assignment's ~$20 flag.

Switching the critic to `mimo-v2.6-flash` would save only about $2.50 — because **the image dominates the cost**, not the critique. That is why the generator decision matters more to the budget than the critic decision.

### 8.2 What `runs.jsonl` records

One line per model call: `ts` (ISO-8601 UTC), `model`, `status` (the HTTP status, or null on timeout), `prompt_tokens`, `completion_tokens`, `cost` (USD, from OpenRouter's `usage` field), `latency_s` (wall clock), `error` (the message, or null).

The repo's own live file currently holds two lines from October 26, eight seconds apart — a complete end-to-end run: the image call ($0.0388, 6.4 s, 1,302 completion tokens for the image) and the critique call ($0.0076, 7.2 s). That is the whole product in two lines.

### 8.3 The spend so far

The eval work on October 26 cost about **$5.90** of OpenRouter credit: ~$3.00 for the eval set (the first Flash attempt at ~$0.40 plus 19 Pro images at $0.138, since several came out wrong and were regenerated), $2.66 for the generator comparison, $0.21 for the critic comparison, and a few cents for spot checks. **Total on the key: $6.07**, including earlier development runs and other coursework.

### 8.4 The two decisions still open

1. **Switch the critic to `xiaomi/mimo-v2.6-flash`?** The evidence says yes (better accuracy, all flaws caught, 1/15 the cost). It needs a **new intent** first, because the course's chain is intent → spec → plan → build. We did not change the code in the same breath as the finding — that is the process working, not the process stalling.
2. **Switch the generator to Nano Banana Pro?** The evidence says the spec's rule triggers (Nano Banana fails more than 1 in 3 runs on four of five topics), but Pro pushes the semester estimate to about **$35, over the $25 budget**. This is a genuine trade-off — diagram correctness versus the budget — and it is a team decision for the new intent.

### 8.5 Two loose ends worth fixing before you present

- **`ANNOTATION.md` is still the empty template.** It is a graded artifact and it is what the Comprehension Defense asks about.
- **The submission tag has not been created.** The repo has `tp1-intent`, `tp1-plan`, and `tp1-test`, but not `tp1-submitted`. The README says to tag the submitted commit and put the repo URL plus the tag on Moodle.

---

## 9. The course-process frame

### 9.1 The artifact chain

This is the CPSC 415 methodology, and the whole repo is organized around it. Every stage leaves a short, version-controlled artifact. **The agent writes most of the code; you decide what gets built, steer it, verify it, and explain every choice.** These files are how you prove you understood what the agent built.

| Stage | Artifact | Written by | Approved by |
|---|---|---|---|
| Plan | `intent/generation-loop.md` | The agent, after interviewing you | **You** |
| Design | `spec.md` | The agent, from the approved intent | **You**, against the intent |
| Build | `plan.md`, then code on a branch | The agent | **You**, before any code |
| Test | tests, lint, CI | The agent | **You** confirm the loop actually ran |
| Deploy | a PR reviewed against `REVIEW.md` | A **separate reviewing agent** | **You** merge |
| Maintain | a new `intent/<name>.md` | Triggered by a bug, a ticket, or a model change | **You** triage |

The rule that runs through all of it: **if the spec and the intent disagree, the intent wins** until you change the intent. This is why the model-switch recommendation is a new intent rather than a code change.

### 9.2 The build discipline

- **One feature per branch, one PR per feature.** Branch names are `<prefix>/<kebab-name>` with the prefix `spec/`, `build/`, `polish/`, or `docs/`.
- **Never push to `main`.** Never commit `.env` or `.claude/settings.local.json`.
- **Tests are written and committed *before* the code they test** — a red commit, then a green one. You can see this in the git history: every PR has a `Review on #N (red)` commit and a `Review on #N (green)` commit before the merge. The review agent finds a flaw, the tests are written to catch it, and then the code is fixed.
- **A separate reviewing agent reviews every PR** against `REVIEW.md`, and a PR fails on any pass-1 or pass-2 finding.

### 9.3 What `REVIEW.md` forbids (and what actually got caught)

| Pass | Severity | Examples | Caught in this project |
|---|---|---|---|
| 1 | **Must not merge** | Secrets in the diff; code that does not match the spec, or a spec silently changed to match the code; deleted, skipped, or weakened tests; unguarded destructive operations | The review on #9 changed the spec and plan to match the new eval interface rather than the other way around |
| 2 | **Must fix before merge** | Model output used as fact without validation; missing error handling for the spec's failure cases; no test for the behavior the PR claims | The "text flaw must match its target" fix; network and malformed-reply errors becoming invalid rows instead of aborting a paid run |
| 3 | **Should fix** | Duplicated logic, dead code, unclear names; cost problems | `checker.py` is the leftover this pass would flag |
| 4 | **Nice to have** | Documentation and comment quality | — |

### 9.4 The mistakes the team logged, so you are not surprised by them

`CLAUDE.md` keeps a "Common mistakes" list — things the agent got wrong before:

- **Stacked PRs.** #3 and #4 were merged into stacked branches and had to be re-landed in #6 and #7.
- **Pull before reviewing.** A review on a stale checkout re-reported findings that were already fixed.
- **Hardcoded articles.** "a animal cell" shipped once — which is why `Topic.with_article` exists and why `test_prompt_builder.py` asserts the article.
- **Never edit `eval/expected.json` to make the critic look better.** A disagreement is a finding for `CHECKS.md`, not an edit to the ground truth.

### 9.5 The project arc in the git history

Seven pull requests merged into `main` — #1, #2, #5, #6, #7, #8, #9. (PRs **#3 and #4 are the stacked ones**: they were merged into stacked branches and had to be re-landed in #6 and #7 — the first entry on the "common mistakes" list.)

| PR | Branch | What it added |
|---|---|---|
| #1 | `spec/generation-loop` | The intent and the spec — the approved design everything else was built against |
| #2 / #3 | `build/core-steps-1-6` | The OpenRouter client, schemas, the topic catalog, the generator scaffold — PR #3 was stacked and re-landed in #6 |
| #4 | `build/model-calls-7-9` | The real model calls — the review caught the "a animal cell" article bug and an A2 minus-sign bug, both fixed red-then-green; also stacked, re-landed in #7 |
| #6 | `build/api-and-page-10-11` | The FastAPI endpoints and the page — the review caught the article regression on the critic side too |
| #5 / #7 | `polish/ui-and-cleanup` | The class-notes layout, the sticky diagram, the empty-filter copy |
| #8 | `docs/claude-md` | `CLAUDE.md` itself, gated by the same review policy |
| #9 | `build/eval-13` | The eval runner, `expected.json`, `CHECKS.md`, the behavior-16 scoring — the current tip of `main` |

One branch is in flight: `feature/week06-image-set` (one commit ahead of `main`, adding the Week 6 image set and `app/checker.py`). It has not been PR'd, and its name does not follow the required prefix convention — it would be renamed to something like `build/week6-checker` before opening the PR.

---

## 10. Defense prep

### 10.1 The trade-offs actually weighed (with the alternatives, since the spec requires naming them)

| Decision | Chosen | Alternative(s) | Why |
|---|---|---|---|
| Web framework | **FastAPI** | Flask; Java/Spring Boot | Request/response bodies *and* the critique schema are all Pydantic, so one validation layer covers both our API and the model's output. Async handlers keep a 30 s model call from blocking the server. Flask is simpler to start but needs a separate validation library and its dev server is single-request by default; Spring Boot is far heavier than two endpoints need. |
| Backend language | **Python 3.12** | Java with `java.net.http` | Both team members had used Python for OpenRouter calls in Weeks 2–5. Java works but splits the backend across two languages for one HTTP call. |
| Catalog format | **JSON file** | A Python dict in a module | Both are equally easy to load, but the dict mixes content with code, and a non-programmer reviewing the checklists would have to read Python. JSON keeps the teacher-facing content readable and diffable. |
| Frontend | **HTML, CSS, vanilla JS** | React + Vite | The intent asks for a simple frontend, and it needs no build step — so both of us can read every line in the presentation. React adds a toolchain and a second dependency tree for one screen and no client-side state worth managing. |
| Provenance | **PNG `tEXt` chunks + a visible caption** | C2PA signed manifests | C2PA is the stronger standard, but it needs signing certificates we do not have. C2PA is listed under out-of-scope. |
| Critique format | **Strict JSON**, validated in code, one retry | Prose the app parses | The schema is enforceable, and prose parsing would make a malformed critique indistinguishable from a valid one. |
| Critic model | **`openai/gpt-4o` as shipped** | `xiaomi/mimo-v2.6-flash` | The intent picked GPT-4o for reliable structured output. The eval showed mimo is *better at this task* and ~20× cheaper. **Recommendation: switch, via a new intent.** |
| Generator model | **`google/gemini-2.5-flash-image` as shipped** | `google/gemini-3-pro-image` | The intent picked Nano Banana for its in-image text rendering at ~$0.039. The eval says Pro is much better (15/15 vs 2/15 on correct label sets) but ~3.5× the price and 3× the latency. **The rule triggers; the budget does not allow it. A team decision.** |
| Verdict source | **Our code** | Let the model return a verdict | A model asked to grade its own work is a documented failure mode — and we have the receipts: GPT-4o graded a mislabeled circuit "good" four times. |
| Retry policy | **None on the image call, one on the critique** | Retry both | A second image attempt costs $0.039 and the student can just press the button again. A critique retry is ~$0.002 and fixes a formatting slip. |
| Test strategy | **Mocked HTTP + one real browser test** | Recorded API cassettes | A cassette would silently rot as the provider's response shape drifts. The live eval harness covers real-model behavior; the unit suite stays fast, free, and deterministic. |

### 10.2 Likely questions, and the answers

**"Why two models? Couldn't one model do both?"**
It could, but then the grader could not see the picture independently of the generator's instructions. Splitting them is what makes behavior 5 enforceable. Two models also cost about the same in total as one expensive model doing both jobs, and they can be swapped independently — which is exactly what the eval comparison needed.

**"What if the critic is wrong? The student sees a wrong checklist."**
That is a real failure and it is exactly what we measured. The mitigation is not "trust the critic" — it is (a) the verdict is computed from the observations by our code, so a bad critic produces a wrong *checklist* but never a fabricated *verdict rule*; (b) the eval set proves the failure rate before shipping; and (c) we found the fix. The system's response to an unreliable critic is to *measure* it and *replace* it, not to hide it.

**"Isn't the verdict just the critic's opinion with extra steps?"**
No. The critic reports observations — "nucleus: present", "A2: ok", "text issue: evaparashun". The verdict is a *rule over those observations*, in code, with the thresholds stated in the spec and covered by a table-driven test per boundary. If the critic's observations are right, the verdict is right by construction. If they are wrong, the checklist is wrong too — but the verdict is never a thing the model invented.

**"How do you know the critic is reading the image and not guessing from the checklist?"**
Three ways. The critic request provably contains the PNG and provably does not contain the generation prompt (there is a test for exactly that). The eval set's flawed images each contain something the prompt never asked for, so a checklist-reading critic would score them clean. And the scoreboard shows GPT-4o reading *label text* reliably while failing to trace leader lines — a pattern you cannot produce from the checklist alone.

**"Why is the whole thing one request? Why not stream?"**
One request keeps the pipeline atomic: the image, the critique, and the verdict are guaranteed to describe the same diagram, and there is no server-side state to get out of sync. Streaming would complicate that for a 13–15 second wait, which is inside the target.

**"What does the provenance stamp actually guarantee?"**
That the file declares its origin: AI-generated, by this model, for this topic, at this time. It is a label, not a signature — anyone with Pillow can rewrite the chunks. We say so in the spec and we do not claim otherwise. The visible "AI-generated diagram" caption under every picture is the user-facing guarantee; the chunks are defense in depth.

**"You said the eval set was hand-checked. By whom?"**
Claude Code checked each image label by label and wrote `expected.json`. We flag that plainly, and the honest limitation is in `CHECKS.md`: the set leans toward GPT-4o's known weakness, because the flawed circuit was chosen *after* GPT-4o had already passed it. The counter-evidence is that the flower swap was planted fresh and GPT-4o missed it too.

**"What would you do next?"**
Two things, in the course's order. First, a **new intent** to switch the critic to `mimo-v2.6-flash` — the evidence is in `CHECKS.md`, and the code change follows the intent, not the other way around. Second, resolve the **generator budget question**: either accept Nano Banana Pro and raise the semester ceiling to ~$35, or keep Nano Banana and add prompt-level mitigations for label garbling, or narrow the topic set to the two Nano Banana handles (circuit and flower) and use Pro for the rest. And fix the **A2 mismatch** — either the prompt asks for the `+`/`−` terminals or the rule stops requiring them.

### 10.3 "What breaks?" — the fragile points, in one list

| Fragile point | Blast radius | Mitigation today |
|---|---|---|
| Nano Banana drifts off the flat 2D textbook style | Aesthetic — the structural critique still works | The pinned shared style block; switch to Pro per the comparison |
| Nano Banana draws a person or face | **High** — a bad image reaching a student | The prompt says "No people, no faces, no hands"; the critic's `people_present` withholds the image. **The vision check is the real shield, not the prompt.** |
| GPT-4o (as shipped) misses leader-line errors | **High** — a wrong "good" on a wrong diagram | Measured; the replacement model is recommended |
| The critic returns invalid JSON or invents a part | Medium | Pydantic + the five cross-checks + one retry; the image still shows if the critique fails |
| The critic grades the prompt instead of the image | Medium–high | The prompt is never sent to it; the flawed eval images defend it |
| OpenRouter rate limit or transient 5xx | One failed run | Explicit timeouts; one retry on the critique only |
| A bad topic catalog | **Medium — poisons every critique until caught** | Startup validation with the failing rule and topic named |
| Pydantic too strict, rejecting a valid reply | Medium | Every omittable field has a default; realistic shapes are tested |
| PNG metadata stripped by a viewer | Low | The visible caption is the load-bearing signal |
| Cost overrun | Medium | Per-call logging; ~$16 vs. the $25 ceiling |
| Two team members on the same branch | Low | Non-overlapping step ownership, one branch per PR |

### 10.4 Prompts for your `ANNOTATION.md`

`ANNOTATION.md` is **your** artifact — the template says *"written by you, not the agent."* It asks four questions. Here is the raw material from this document for each; write it in your own words.

1. **What is this?** — A one-screen web app: pick one of five science topics, it draws a labeled diagram, and a second model marks that diagram against a hand-written checklist. The verdict is computed in Python. It does not store anything, does not let you edit the diagram, and does not work outside the five topics — deliberately.
2. **Why this choice?** — The alternatives and trade-offs are in §10.1 above: FastAPI over Flask (one validation layer for both the API and the model's output), JSON over a Python dict (a teacher can review the checklists), vanilla JS over React (no build step, so every line is readable in the presentation), PNG text chunks over C2PA (no signing certificates), and — the big one — two models over one, because the grader must not see the generator's instructions.
3. **What breaks?** — §10.3 is the table. The three sharpest: the critic missing leader-line errors (measured, and the reason we are switching), the generator producing garbled labels (measured, and the reason we are debating the budget), and a bad topic catalog poisoning every critique (mitigated by startup validation).
4. **What did I learn?** — The honest answer has a specific shape here: **the eval exists because the system looked fine.** GPT-4o returned 100 % valid, well-formed, confident critiques and was still wrong about the picture, four times. Schema validity is not correctness. The other lesson is the process one: the recommendation to change the model is *not* a code change, it is a new intent — and the chain (intent → spec → plan → build → test → deploy) is what keeps a finding from becoming an unreviewed patch.

---

## 11. Glossary

| Term | Meaning in this project |
|---|---|
| **OpenRouter** | The single gateway through which every model call goes. One key, one endpoint, one `usage` block with the cost. |
| **Nano Banana** | `google/gemini-2.5-flash-image` — the drawing model, ~$0.039/image. |
| **Nano Banana Pro** | `google/gemini-3-pro-image` — the stronger drawing model, ~$0.138/image. |
| **mimo** | `xiaomi/mimo-v2.6-flash` — the cheap vision model, ~$0.0005/critique. |
| **Topic catalog** | `app/topics.json` — the five topics with their parts, forbidden parts, arrangement rules, and labels. The single source of truth. |
| **Required part** | Something the diagram must contain and label. |
| **Forbidden part** | Something that must *not* appear (e.g. a chloroplast in an animal cell). Its presence forces a `poor` verdict. |
| **Arrangement rule** | A spatial/structural requirement (A1, A2, A3 per topic) — e.g. "the vacuole is the largest organelle." |
| **Label** | The exact string the generator is told to write. Today identical to the required parts, but a separate concept. |
| **The critic** | The vision model that inspects the PNG and reports observations. Never sees the generation prompt. |
| **The checklist** | The per-part table in the UI: part name, mark, note. |
| **Verdict** | `good` / `needs_work` / `poor`. Computed in Python from the observations. |
| **Behavior 16** | The critic's pass/fail gate: zero false flags on the good images, and ≥ 80 % of the flawed runs caught. |
| **Behavior 17** | The end-to-end gate: a real run completes in ≤ 45 s and costs ≤ $0.25. |
| **False flag** | A complaint the critic raised about a diagram we hand-checked as correct. |
| **Part accuracy** | The fraction of required parts where the critic's present/not-present call matches the hand-checked truth. |
| **Planted flaw** | The deliberate defect in a flawed eval image, recorded as a `kind` (`part` / `forbidden` / `arrangement` / `text`) plus a `target`. |
| **Provenance** | The PNG `tEXt` chunks and the visible caption saying the image is AI-generated. A label, not a signature. |
| **tEXt chunk** | A standard PNG metadata block holding a key/value pair (e.g. `ai_generated=true`). |
| **runs.jsonl** | The append-only log of every model call: model, status, tokens, cost, latency, error. |
| **Usage** | OpenRouter's `usage` block — prompt tokens, completion tokens, and the USD cost. Requested explicitly with `usage: {include: true}`. |
| **Mock transport** | `httpx.MockTransport` — the seam in `openrouter.py` that lets tests intercept every outbound request. |
| **Playwright** | The browser automation used by the one real-browser test. |
| **Intent / Spec / Plan** | The three upstream artifacts: *what should exist and why* → *how it will be built* → *in what order, with what proof*. |
| **Review passes** | `REVIEW.md`'s four severity tiers a separate agent applies to every PR: must-not-merge, must-fix, should-fix, nice-to-have. |
| **Red-then-green commit** | The TDD convention: a commit with the failing test, then a commit with the fix. Visible throughout the git history. |
| **Comprehension Defense** | The oral check where the four `ANNOTATION.md` questions are asked. |
| **Intent wins** | The chain's tie-breaker: if the spec and the intent disagree, the intent is authoritative until you change it. |

---

*Prepared from the `TP1` repo as it stands on `feature/week06-image-set` (one commit ahead of `main` at `fd4e07a`). All numbers quoted here come from `CHECKS.md`, `spec.md`, `runs.jsonl`, and `eval/expected.json` as written.*
