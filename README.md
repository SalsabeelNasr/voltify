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

## How it works

```
INPUT: customer_id_submission (image, entered_name)

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
  ocr_text = run_ocr(image)   # one flat text blob, no structured fields or coordinates

STEP 3: Compare fields (deterministic)
  # Name: each word of the entered name is checked independently for presence in the
  # OCR text — not a single contiguous "First Last" match, which breaks on reversed
  # name order, names split across OCR lines, or a middle name/initial (all common on
  # real IDs).
  name_issue = ANY token of entered_name NOT found in ocr_text

  # Expiry: every real ID obeys DOB < Issue Date < Expiry Date — a hard constraint, not
  # a heuristic. So the LATEST plausible date on the card (after dropping obviously
  # implausible matches — more than ~120 years past or ~15 years future) is treated as
  # the expiry. This does not depend on recognizing a label like "Exp"/"Expires" — label
  # vocabulary varies too much across issuers, and OCR reading order doesn't reliably
  # keep a label next to its value on cards with side-by-side columns.
  plausible_dates = filter_plausible(find_all_dates(ocr_text))
  IF len(plausible_dates) < 2:
      flag_for_human_review(reason="insufficient_dates_found")
      STOP
  extracted_expiry = max(plausible_dates)

  # A recognized label is only a secondary cross-check, never the primary signal, and
  # only counted when it unambiguously names a single date.
  label_expiry = find_unambiguous_label_match(ocr_text,
      labels=["EXP","EXPIRES","EXPIRY","DATE OF EXPIRY","VALID THRU","VALID UNTIL","GOOD THRU"])
  IF label_expiry EXISTS AND label_expiry != extracted_expiry:
      flag_for_human_review(reason="date_ordering_and_label_disagree")
      STOP

  date_issue = (extracted_expiry < today)

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

Note on Step 1 in this prototype: since this is a static browser-only demo with no backend
or API key infrastructure, the live **Create Case** feature approximates the blur check
on-device (a Laplacian-variance sharpness score computed on the uploaded image) instead of
calling a real vision LLM. It's deterministic, so the retry/invalid-response branch above
doesn't trigger live — that failure mode is instead demonstrated by the "Riley Morgan"
mocked row, representing what the production LLM-backed version of Step 1 would do.

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

## Solving two real extraction bugs

The Step 2/3 logic above isn't the first version — it's what's left after testing the
prototype against an actual ID template (a Delaware DMV sample license), not just the
fictional demo cards. Both bugs were deterministic parsing bugs, not hallucination: OCR.space
correctly read everything printed on the card in both cases — the problem was what the
extraction logic did with a correct but unstructured read.

### The date bug

That test card printed three real dates — Issue `03/15/2018`, Expiry `04/30/2028`, DOB
`04/30/2000`. The original logic searched the OCR text for a date sitting near a label like
`"Exp"` and, failing to find one it trusted, fell back to guessing — and picked the DOB,
incorrectly flagging a valid, unexpired ID as expired.

Label-matching alone is fragile for two reasons: label vocabulary varies a lot across
issuers ("Exp," "Expires," "Valid Thru," "Date of Expiry," …), and OCR reading order doesn't
reliably preserve column layout — many IDs print labels like "Iss" and "Exp" side-by-side in
two columns on the same visual row, and OCR can interleave or separate a label from its value
when that happens.

**The fix:** use chronological ordering as the *primary* signal instead. Every real ID date
obeys `DOB < Issue Date < Expiry Date` — a hard constraint, not a heuristic that sometimes
holds. So the latest plausible date on the card (after filtering out obviously implausible
matches, like a barcode fragment that happens to look date-shaped) is the expiry, full stop —
no label needed. A recognized label is now only a secondary cross-check: if it unambiguously
names a *different* date than the ordering guess, that's a genuine conflict and the case
escalates to a human rather than trusting either signal blindly. This also means the system
never escalates just because a card has more than one date on it — that's true of nearly
every real ID, and escalating on that alone would make automation pointless. It escalates
only when there's truly not enough information (fewer than two dates) or a real conflict.

### The name bug

Testing against the same Delaware sample card (name printed as `SAMPLE` / `JANICE` on two
separate lines, last name before first) correctly flagged a mismatch against an unrelated
profile name — that part worked. But it exposed a broader risk in the underlying method: an
exact `"First Last"` contiguous-phrase match would also false-flag a **real, legitimate**
customer any time a name is printed in reversed order, split across OCR lines, or with a
middle name/initial in between — none of which are edge cases on real IDs (this very
template reverses the order).

**The fix:** token-based matching instead of phrase matching. Each word of the entered name
is checked independently for presence anywhere in the OCR text, so reversed order, split
lines, and label text sitting between name parts no longer break the check. The Data
Validation UI was updated to match what the system actually knows under this approach: since
there's no clean single "extracted name" anymore, only which words were found, the Full Name
row shows a per-token breakdown (e.g. `"Devon" ✓ found · "Blackwood" ✓ found`) instead of a
fabricated extracted-name value implying a cleaner read than the system actually has.

---

## LLM vs. deterministic logic, and the human-in-the-loop mechanism

**Where the LLM is used:** Only one place — checking if the ID photo is blurry. This needs
visual judgment, which simple rules can't do without a dedicated image API.

**Where deterministic logic is used:**
- Name matching: token-based presence check (see Step 3 above) — not a single exact-phrase
  comparison.
- Expiry date check: chronological ordering of every date found on the card, cross-checked
  against a recognized label only when one exists and unambiguously names a single date.
- All customer messages: static, pre-written templates. No free text generation, so no
  hallucination risk in the messages themselves.

**Where the human checkpoint sits:**
- The LLM must respond with only one of three answers — "blurry," "clear," or "unsure." It
  is told to say "unsure" instead of guessing. Anything other than a confident "blurry" or
  "clear" goes to a human; nothing gets sent automatically.
- On the deterministic side, extraction also refuses to guess: fewer than two plausible
  dates on the card, or a genuine disagreement between the chronological-ordering guess and
  a recognized label, both escalate to a human rather than picking one signal arbitrarily.

**The specific mechanism to catch a wrong or hallucinated LLM answer:** Every LLM response is
checked against a fixed list of 3 allowed answers. If the response doesn't match one of
them, we retry once. If it still fails, we don't guess — it goes straight to a human.

---

## Biggest risk (one line)

**Risk:** If the LLM wrongly says a photo is "clear" when it's actually too blurry to read
properly, the workflow moves on to extract the name and date from that same bad image. The
extraction could pull garbled or wrong text, causing a false "name mismatch" or "date
invalid" message to the customer, when the real problem was just a bad photo the whole time.

**Fix:** Add a sanity check after extraction — if OCR returns empty, very short, or clearly
malformed text, treat that as a sign the image was actually unreadable, and route to human
review instead of trusting the extracted values. The shipped extraction logic already covers
part of this on the date side (fewer than two plausible dates found escalates rather than
guessing); a dedicated check for garbled/too-short OCR text specifically on the name side is
not yet implemented.

---

## Try it: Create Case

Upload an ID image and enter a profile name to run it through the triage logic above, live,
entirely in the browser:

- **Image read:** `FileReader`, no upload to a server.
- **Blur check:** Laplacian-variance sharpness score computed on `<canvas>` pixel data
  (see note under Step 1 above).
- **OCR:** [OCR.space](https://ocr.space/ocrapi/freekey)'s free-tier API, called directly
  from the browser. Its `ParsedText` response is one flat, unstructured text blob — no
  labeled fields, no coordinates — which is exactly why extraction relies on the
  chronological-ordering and token-matching approach in Step 2/3 above rather than clean
  field access. This is noticeably fuzzier than the clean, mocked data in the other rows,
  worth noting as a real limitation of free-tier OCR, not something to hide.
- **Persistence:** new cases are saved to `localStorage` and reload with the page.
- The OCR.space API key is visible in client-side JS — an accepted, disclosed shortcut for
  a disposable demo prototype, not a production pattern.

## What's in the queue

Six rows, five mocked + one built from the notes below, each demonstrating a different
triage outcome:

| Customer | Status | Demonstrates |
|---|---|---|
| Peter Parker | Resolved by Agent | Name mismatch → auto message sent → customer disputed → human agent manually closed it |
| John Smith | Needs Review | Image too blurry to classify confidently ("unsure") → routed to human, no message sent |
| Barbie | Pending Customer | Expired ID → auto message sent → case stays open until customer resubmits |
| Riley Morgan | Needs Review | LLM classification response invalid twice in a row → retry limit hit → escalated to human, no guessing |
| Jordan Blake | Pending Customer | Both name AND expiry wrong at once → concatenated auto message sent |
| Devon Blackwood | Needs Review | Only one date found on the card → not enough to sort chronologically → escalated, no message sent |

**Status model:**
- **Needs Review** (red) — system couldn't confidently triage, or a customer disputed.
- **Resolved by Agent** (indigo) — a human manually closed it, shows agent name.
- **Pending Customer** (amber) — system sent an automated message, no agent involved, but
  the case is NOT resolved — it's waiting on the customer to act.

## Known limitations (disclosed, not hidden)

- OCR.space's free-tier `ParsedText` is one flat, unstructured text blob — extraction relies
  on regex/heuristics, not clean field access.
- No real backend or database — this is a portfolio/interview prototype.
- The OCR.space API key is exposed client-side (see above).
- No sorting or filtering on the queue — every case is shown in one flat list.
