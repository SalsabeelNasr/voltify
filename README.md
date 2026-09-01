# Voltify — KYC Review Prototype

**Live demo:** https://voltify-kyc.netlify.app/

A lightweight prototype for automating triage of failed KYC (ID verification) checks —
built to prove the approach before asking engineering to build it for real. Single static
`index.html` file, no build step, no backend, deployed via Netlify.

The page shows a "Review Queue" of KYC failure cases, each expandable to show the ID
document image, a Data Validation comparison (ID Document vs. User Profile / Policy Check),
a Verification Timeline (a factual log of what the system did), and an action footer. A
**Create Case** flow lets you upload a real ID image and run it through the same triage
logic live, entirely in the browser.

---

## Part 1: Design — Pseudocode, Prompt, Messages

> This is the original task design write-up. See **Implementation Notes** below for where
> the shipped `index.html` differs from it (mainly: expiry is checked against *today*, not
> against date of birth — the pseudocode below has a leftover DOB reference from an early
> draft; and the live Create Case extraction logic was substantially reworked after testing
> against a real ID template — see below).

### Pseudocode

```
INPUT: customer_id_submission (image, entered_name, entered_dob)

STEP 1: Image quality check (LLM, uses Prompt P1, with validation + retry)
  attempt = 1
  MAX_ATTEMPTS = 2

  WHILE attempt <= MAX_ATTEMPTS:
      llm_response = call_llm(image, prompt=P1)
      IF llm_response in ["blurry", "clear", "unsure"]:
          BREAK
      ELSE:
          attempt += 1

  IF attempt > MAX_ATTEMPTS:
      flag_for_human_review(reason="failed_validation_after_retry")
      STOP

  IF llm_response == "blurry":
      SEND message = M1 + M5
      LOG triage_note = "blurry photo auto-detected, message sent"
      STOP

  IF llm_response == "unsure":
      flag_for_human_review(reason="low_confidence_blur_check")
      STOP

  # llm_response == "clear", proceed to extraction

STEP 2: OCR extraction (deterministic)
  extracted_name, extracted_dob = run_ocr(image)

STEP 3: Compare fields (deterministic)
  name_issue = (trim(extracted_name) != trim(entered_name))
  date_issue = (extracted_dob < today)

STEP 4: Build outgoing message
  IF name_issue AND date_issue:
      SEND concatenated(M3, M4) + M5
      LOG triage_note = "name + date mismatch, message sent"
  ELIF name_issue:
      SEND M3 + M5
      LOG triage_note = "name mismatch, message sent"
  ELIF date_issue:
      SEND M4 + M5
      LOG triage_note = "date expired, message sent"
  ELSE:
      flag_for_human_review(reason="unexpected_clear_result")

STEP 5: flag_for_human_review(reason)
  DO NOT auto-send
  LOG triage_note = f"Needs human review: {reason}"
  Notify support queue
```

### Prompt — Image blur check

**P1:** *"Look at this ID image. Respond with exactly one word: 'blurry' if clearly blurry,
'clear' if clearly readable, 'unsure' if you are not highly confident either way. Do not
guess if unsure."*

### Message table

| ID | Trigger | Message Text |
|---|---|---|
| M1 | Image blurry | "Your ID photo appears too blurry for us to verify. Please retake it in good lighting and resubmit." |
| M2 | Image quality unsure / low confidence | *(no message sent, routed to human review)* |
| M3 | Name mismatch | "We noticed the name on your ID doesn't quite match what's on file." |
| M4 | Date expired | "The ID you submitted appears to be expired. Please upload a valid, unexpired ID." |
| M5 | Contact support footer (always appended) | "If you believe this is a mistake, contact our support team here: [link]." |
| M6 | Unexpected clear image but KYC still failed | *(no message sent, routed to human review)* |

**Concatenation rule:** If both M3 and M4 apply: *"We noticed the name on your ID doesn't
quite match what's on file, and the ID also appears to be expired. Please review both and
resubmit."* Always append M5: *"If you believe this is a mistake, contact our support team
here: [link]."*

---

## Part 2: LLM vs. Deterministic Logic, Human-in-the-Loop, Hallucination Mechanism

**Where the LLM is used:** Only one place — checking if the ID photo is blurry. This needs
visual judgment, which simple rules can't do without a dedicated image API.

**Where deterministic logic is used:**
- Name matching: exact text comparison after removing extra spaces.
- Expiry date check: simple date comparison.
- All customer messages: static, pre-written templates. No free text generation, so no
  hallucination risk in the messages themselves.

**Where the human checkpoint sits:** The LLM must respond with only one of three answers —
"blurry," "clear," or "unsure." It is told to say "unsure" instead of guessing. Anything
other than a confident "blurry" or "clear" goes to a human; nothing gets sent automatically.

**The specific mechanism to catch a wrong or hallucinated answer:** Every LLM response is
checked against a fixed list of 3 allowed answers. If the response doesn't match one of
them, we retry once. If it still fails, we don't guess — it goes straight to a human.

---

## Part 3: Biggest Risk (One Line)

**Risk:** If the LLM wrongly says a photo is "clear" when it's actually too blurry to read
properly, the workflow moves on to extract the name and date from that same bad image. The
extraction could pull garbled or wrong text, causing a false "name mismatch" or "date
invalid" message to the customer, when the real problem was just a bad photo the whole time.

**Fix:** Add a basic sanity check after extraction — if OCR returns empty, very short, or
clearly malformed text, treat that as a sign the image was actually unreadable, and route to
human review instead of trusting the extracted values.

---

## Implementation Notes — how the live prototype differs from Part 1

The design above is the original spec. Building the **Create Case** feature (upload a real
ID, run it through the same logic client-side) against actual ID templates surfaced a few
things worth documenting rather than silently papering over:

- **No LLM call for the live blur check.** Blur detection runs entirely on-device via a
  Laplacian-variance sharpness score computed on the uploaded image — no API call needed.
  Deterministic, so there's no retry-loop/hallucination surface for this step in the shipped
  version (the retry-limit *scenario* is still demonstrated by the "Riley Morgan" mocked row,
  representing what the live-LLM version of Step 1 would do).
- **Expiry is checked against today, not DOB.** Part 1's pseudocode above has a leftover
  `extracted_dob` / `date_issue = (extracted_dob < today)` from an early draft. The actual
  logic (mocked rows and live Create Case) always compares the **expiration** date to today —
  there is no DOB check anywhere in the shipped logic.
- **Date extraction uses chronological ordering, not label-matching, as the primary
  signal.** OCR.space returns one flat text blob with no structured fields or coordinates.
  Testing against a real Delaware DMV sample license surfaced a real bug: with three dates on
  the card (Issue 03/15/2018, Expiry 04/30/2028, DOB 04/30/2000), the original label-based
  heuristic extracted the DOB as the expiry, incorrectly flagging a valid ID as expired. Fix:
  since **DOB < Issue Date < Expiry Date always holds** for a real ID, the *latest* plausible
  date on the card (after filtering out clearly-implausible matches — more than ~120 years
  past or ~15 years future) is used as the expiry, regardless of label vocabulary or whether
  OCR preserved column layout. Label-matching (Exp, Expires, Expiry, Valid Thru, Date of
  Expiry, etc.) is now only a secondary cross-check: if it unambiguously names a *different*
  date than the ordering guess, that's a genuine conflict and the case is escalated to a
  human rather than silently trusting one signal over the other. Escalates when fewer than
  two plausible dates are found, or on a genuine ordering-vs-label disagreement — but *not*
  merely because more than one date exists on the card, since that's true of nearly every
  real ID.
- **Name matching is token-based, not a single contiguous-phrase match.** An exact
  `"First Last"` substring check breaks on reversed name order, names split across OCR lines,
  or a middle name/initial — all common on real IDs (the Delaware sample card itself prints
  the last name before the first). Each word of the entered name is now checked
  independently for presence in the OCR text. The Data Validation UI reflects this honestly:
  the Full Name row shows a per-token found/not-found breakdown (e.g. `"Devon" ✓ found ·
  "Blackwood" ✓ found`) instead of a fabricated single "extracted name" value.
- **A 6th mocked row ("Devon Blackwood")** demonstrates the "not enough dates to determine
  expiry by chronological order" escalation path, alongside the original 5 rows (Peter
  Parker, John Smith, Barbie, Riley Morgan, Jordan Blake) covering name mismatch,
  low-confidence blur, expired ID, LLM retry-limit exhaustion, and combined name+date
  failure respectively.
- **Create Case** (upload an ID + enter a profile name, run it through the same triage
  client-side): image read via `FileReader`, blur score computed on `<canvas>` pixel data,
  OCR via [OCR.space](https://ocr.space/ocrapi/freekey)'s free API (direct browser call), new
  cases persisted to `localStorage`. The OCR.space API key is visible in client-side JS —
  an accepted, disclosed shortcut for a disposable demo prototype, not a production pattern.
- **UI:** responsive layout below 640px width (header/Create Case button stack, status
  badges shrink, the document preview and Data Validation card stack into their own rows
  instead of a fixed sidebar). Removed the non-functional "Filter: Needs Review" button.

## Known limitations (disclosed, not hidden)

- OCR.space's free-tier `ParsedText` is one flat, unstructured text blob — extraction relies
  on regex/heuristics, not clean field access. This is noticeably fuzzier than the clean
  mocked data in the 5 original rows.
- No real backend or database — this is a portfolio/interview prototype.
- The API key is exposed client-side (see above).
