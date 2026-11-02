# My presentation notes (Javier)

TP1 presentation, Mon Nov 2, 1:30 PM, MC-225. My part: slides 10–14, 6:00 → 10:00 (4 minutes).
Personal file, not part of the repo.

**Timing:** slides 10–12 = 2:30 (what went wrong: 10 ≈ 1:00, 11 + 12 ≈ 1:30). Slides 13 + 14 = 1:30 (next steps and wrap-up).
**If I'm running long:** on slide 11, show the circuit, say GPT-4o called it good, and move on. On slide 12, just say the two scores.
**In the deck:** → next, ← back, **N** shows my speaker notes.

---

## Slide 9: hand-off from Ralston
He says something like "Javier will show what went wrong along the way." I start with slide 10.

---

## Slide 10: The artist couldn't spell (≈1:00)

| On screen | What I say | What I remember (the why) |
|---|---|---|
| Flash plant cell | "Everything Ralston showed works, but two things went wrong. Both come down to one problem: you can't trust an AI's output just because it looks good. Our first image model, Gemini Flash, misspelled at least one label in 9 of 15 runs." | Image models don't type letters, they draw pixels that look like letters. Cheaper models are worse at it. Our spec picked Flash *because* of its text rendering, and our own test proved that wrong. |
| **Click 1:** red circles | "Look: CELL MEMRANE, pointing at the nucleus. CHORCOPOPT." | The labels point at the wrong parts, not just misspelled. A student would learn something false. Only 2 of 15 Flash runs had every label in the right place. |
| **Click 2:** Pro plant cell | "The better model, Gemini Pro, fixed almost all of it." | Pro: 1 of 15 with a typo ("chloplast"). Measured on 5 topics × 3 runs = 15 images per model, same prompt, checked by hand. |
| **Click 3:** cost line | "But it costs about 3.5 times as much. That's about $35 for the semester, and our budget is $25. So we didn't switch quietly. It goes to a new intent." | $0.134 vs $0.039 per image. Spec estimated ~250 runs for the semester. The switch rule was written in the spec *before* testing: keep Flash unless Pro fixes the labels, and if it does, update the intent first. |

**If asked "was that just one bad picture?":** open `eval/generator-comparison/plant-cell.jpg`. All 3 Flash runs (top row) are misspelled; Pro (bottom row) only has "chloplast".

---

## Slide 11: The critic got fooled (≈0:40, the story)

| On screen | What I say | What I remember (the why) |
|---|---|---|
| The circuit picture | "The second problem is scarier. This is a real picture from our first run." | It's the actual image from our first end-to-end run on Oct 25, not one we made for the test. |
| **Click:** red rings on "wire" and "battery" | "The 'wire' label points into the battery, and 'battery' points at nothing." | Every word is spelled right. The mistake is *where* the labels point. |
| **Click:** verdict card, "good" | "We gave it to our critic, GPT-4o. It said 'good': every part present." | GPT-4o passed this picture 4 times in total: this first run, plus all 3 runs in the test (next slide). |
| **Click:** takeaway | "It looked right, so it passed. So we needed a test where we already know the answer." | This is the reason the test exists. |

---

## Slide 12: So we built a test (≈0:50, cut first if long)

| On screen | What I say | What I remember (the why) |
|---|---|---|
| The 5 broken pictures | "We made 5 broken pictures, one per topic, each with one mistake we planted on purpose, and I checked the answer key against them myself. Each picture went to the critic three times, and the results were the same every time." | The answer key is `eval/expected.json`; I spot-checked all 5 on Nov 2. 3 runs each to see if the critic is consistent, and it was. |
| **Click:** GPT-4o row, ✓✓✓✗✗, **3/5** | "GPT-4o caught 3 of the 5. It missed the circuit again, and the flower, where two labels were swapped." | Same as "9 of 15" in `CHECKS.md` (5 pictures × 3 runs). I say 3 of 5 so it doesn't sound like slide 10's "9 of 15 pictures", which was a different test (the artist, not the critic). |
| **Click:** MiMo row, ✓✓✓✓✓, **5/5** | "A much cheaper model, MiMo, caught all 5, with no false alarms on the good pictures, at about a fifteenth of the price." | $0.013 vs $0.20 for the 30 calls. Zero false alarms means it didn't just flag everything. |
| **Click:** takeaway | "Both misses were a label pointing at the wrong part: GPT-4o reads the words but doesn't follow the line. To be fair, we picked the circuit because GPT-4o had already missed it, but the flower was new and it missed that too." | An AI checking an AI still needs a test with known answers. |

**Short version if I'm out of time:** "GPT-4o called this broken circuit 'good.' On a test with 5 planted mistakes, it caught 3; a cheaper model caught all 5."

---

## Slide 13: Next (≈1:10)

| On screen | What I say | What I remember (the why) |
|---|---|---|
| **1** Switch the critic | "Three things, in order. One: switch the critic to the cheaper model that won. We haven't changed the code yet, on purpose: the intent changes first, then the code." | GPT-4o was picked in the intent for reliable JSON on vision tasks, and that part held up (100% valid). It just missed mistakes. The switch saves only ~$2.50 a semester because the *image* is most of the cost, so it's about accuracy, not money. |
| **Click: 2** Fix the circuit prompt | "Two: our checklist wants plus and minus on the battery, but our prompt never asked for them. So a correct-looking circuit could fail. It's a one-line fix." | We found it while building the eval: the "good" circuit from our own prompt failed that rule. Like grading a student on something that wasn't in the assignment. |
| **Click: 3** Decide on Pro | "Three: clean labels versus going over budget. That's a team decision. So far we've spent about six dollars, and one run costs about five cents." | The "$6.07" total includes other coursework on the key, so I say "about six dollars." |

---

## Slide 14: Questions (≈0:20)

**Say:** "In one sentence: the AI draws, a second AI looks, and our code decides the grade. Thank you. Questions?" → open the peer notes form.

---

## Questions that will probably come to me

| Question | My answer |
|---|---|
| What does `checker.py` do? | It compares a critic's answer to an answer key, part by part. Strict matching: `True` isn't `"true"`, "nucleus" isn't "Nucleus". Same input, same result every time, no API key needed. A reply that isn't JSON comes back as a structured failure, not a crash. |
| Why exact matching? | So the result never depends on another model's judgment. A harmless wording difference counts as a failure; that's the trade-off. |
| Are there tests for it? | Yes, 28: the 27 from Week 6 plus one Ralston asked for in review (the 200-character excerpt). They came over in PR #14. In Week 6 they were committed together with the code, not before it. |
| Did `checker.py` produce the 15/15 and 9/15? | No. The eval scores the app's own critique format with `eval/score.py`. `checker.py` reads the Week 6 format, and nothing in the app calls it yet. |
| How much did it cost? | About $6 total on the key (including other coursework), about 5 cents per run. |
| Why not ask the image model to check its own picture? | It tends to agree with itself. A second model plus a grade from code is harder to fool. *(Ralston's question, but I can answer it.)* |
| Why doesn't the critic see the prompt? | So it judges the picture, not what we asked for. It can't pass something just because we asked for it. |
| Who wrote what? | Ralston: generator, critic, server, page, eval. Me: the Week 6 image set and `checker.py`. We reviewed each other's PRs. |
| What would you change? | Start the eval earlier. Our default critic lost to a cheaper one. |
| Where was the agent confidently wrong? | It said `checker.py` had no tests. I knew I'd written them. It had only looked in TP1; they were in my Week 6 folder. That's how we found they never got copied over. |

## Ground rules
- Stay under my 4 minutes. Cut slide 12 to the short version first.
- Don't read from the screen for more than a few seconds.
- Only say things I could explain if asked.
