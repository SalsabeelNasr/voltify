# Voltify: KYC Review Prototype

**Live demo:** https://voltify-kyc.netlify.app/

A lightweight prototype for automating triage of failed KYC (ID verification) checks.
Built to prove the approach before asking engineering to build it for real.

- Single static `index.html` file
- No build step, no backend
- Deployed via Netlify

The page shows a Review Queue of KYC failure cases. Each row expands to show the ID
document image, a Data Validation comparison, a Verification Timeline (a factual log of
what the system did), and an action footer. A **Create Case** flow lets you upload a real
ID image and run it through the same triage logic live, in the browser.

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
  # OCR text. Not a single contiguous "First Last" match, which breaks on reversed name
  # order, names split across OCR lines, or a middle name/initial.
  name_issue = ANY token of entered_name NOT found in ocr_text

  # Expiry: every real ID follows DOB < Issue Date < Expiry Date. So the latest
  # plausible date on the card (after dropping obviously implausible matches, more
  # than ~120 years past or ~15 years future) is treated as the expiry. This does not
  # depend on recognizing a label like "Exp" or "Expires". Label vocabulary varies too
  # much across issuers, and OCR reading order does not reliably keep a label next to
  # its value on cards with side-by-side columns.
  plausible_dates = filter_plausible(find_all_dates(ocr_text))
  IF len(plausible_dates) < 2:
      flag_for_human_review(reason="insufficient_dates_found")
      STOP
  extracted_expiry = max(plausible_dates)

  # A recognized label is only a secondary cross-check, never the primary signal.
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

Step 1 in this prototype:

- This is a static browser demo with no backend or API key infrastructure.
- The live Create Case feature approximates the blur check on-device: a
  Laplacian-variance sharpness score computed on the uploaded image.
- It's deterministic, so the retry/invalid-response branch above doesn't trigger live.
- That failure mode is instead demonstrated by the "Riley Morgan" mocked row.

### Prompt: Image blur check

**P1:** *"Look at this ID image. Respond with exactly one word: 'blurry' if clearly
blurry, 'clear' if clearly readable, 'unsure' if you are not highly confident either
way. Do not guess if unsure."*

### Message table

| ID | Trigger | Message Text |
|---|---|---|
| M1 | Image blurry | "Your ID photo appears too blurry for us to verify. Please retake it in good lighting and resubmit." |
| M2 | Image quality unsure / low confidence | *(no message sent, routed to human review)* |
| M3 | Name mismatch | "We noticed the name on your ID doesn't quite match what's on file." |
| M4 | Date expired | "The ID you submitted appears to be expired. Please upload a valid, unexpired ID." |
| M5 | Automated-check disclosure (always appended) | "This check was completed automatically. If you think something's wrong, flag it here and we'll open a case for a specialist to review." |
| M6 | Unexpected clear image but KYC still failed | *(no message sent, routed to human review)* |

**Concatenation rule:** If both M3 and M4 apply, send: *"We noticed the name on your ID
doesn't quite match what's on file, and the ID also appears to be expired. Please review
both and resubmit."* Always append M5.

---

## LLM vs. deterministic logic, and the human checkpoint

**Where the LLM is used:** one place only, checking if the ID photo is blurry. This needs
visual judgment, which simple rules can't do without a dedicated image API.

**Where deterministic logic is used:**

- Name matching: token-based presence check, not a single exact-phrase comparison.
- Expiry date check: chronological ordering of every date on the card, cross-checked
  against a recognized label only when one exists and unambiguously names a single date.
- All customer messages: static, pre-written templates. No free text generation, so no
  hallucination risk in the messages themselves.

**Where the human checkpoint sits:**

- The LLM must respond with only one of three answers: "blurry," "clear," or "unsure."
- It's told to say "unsure" instead of guessing.
- Anything other than a confident "blurry" or "clear" goes to a human. Nothing gets sent
  automatically.
- On the deterministic side, extraction also refuses to guess: fewer than two plausible
  dates, or a genuine disagreement between the ordering guess and a recognized label,
  both escalate to a human.
- Every auto-sent message discloses that the check was automated (M5) and invites the
  customer to flag it for human review. Peter Parker's mocked row shows this end to end:
  customer disputes, a human reviews it, agent resolves the case.

**Catching a wrong or hallucinated LLM answer:** every LLM response is checked against a
fixed list of 3 allowed answers. If the response doesn't match one of them, retry once.
If it still fails, don't guess. Go straight to a human.

---

## Risks

Testing the prototype against an actual ID template (a Delaware DMV sample license), not
just the fictional demo cards, surfaced two real bugs. Both were deterministic parsing
bugs, not hallucination: OCR.space correctly read everything printed on the card in both
cases. The problem was what the extraction logic did with a correct but unstructured
read. A third risk is still open.

### Date bug (fixed)

- Test card printed three real dates: Issue `03/15/2018`, Expiry `04/30/2028`, DOB
  `04/30/2000`.
- The original logic searched for a date sitting near a label like `"Exp"`.
- When it didn't find one it trusted, it fell back to guessing, and picked the DOB.
- Result: a valid, unexpired ID got flagged as expired.

Label-matching alone is fragile. Label vocabulary varies a lot across issuers ("Exp,"
"Expires," "Valid Thru," "Date of Expiry"), and OCR reading order doesn't reliably
preserve column layout. Many IDs print labels like "Iss" and "Exp" side by side in two
columns on the same visual row, and OCR can interleave or separate a label from its value
when that happens.

**Fix:** use chronological ordering as the primary signal. Every real ID date follows
`DOB < Issue Date < Expiry Date`, a hard constraint, not a heuristic that sometimes
holds. The latest plausible date on the card (after filtering out obviously implausible
matches, like a barcode fragment that looks date-shaped) is the expiry, no label needed.
A recognized label is now only a secondary cross-check: if it unambiguously names a
different date than the ordering guess, that's a genuine conflict and the case escalates
to a human. The system never escalates just because a card has more than one date on it,
since that's true of nearly every real ID. It escalates only when there's truly not
enough information (fewer than two dates) or a real conflict.

### Name bug (fixed)

Testing against the same Delaware sample card (name printed as `SAMPLE` / `JANICE` on
two separate lines, last name before first) correctly flagged a mismatch against an
unrelated profile name. That part worked. But it exposed a broader risk: an exact
`"First Last"` phrase match would also false-flag a real, legitimate customer any time a
name is printed in reversed order, split across OCR lines, or with a middle name or
initial in between. None of those are edge cases on real IDs. This very template reverses
the order.

**Fix:** token-based matching instead of phrase matching. Each word of the entered name
is checked independently for presence anywhere in the OCR text. Reversed order, split
lines, and label text sitting between name parts no longer break the check. The Data
Validation UI matches what the system actually knows: instead of a fabricated single
"extracted name" value, the Full Name row shows a per-token breakdown, for example
`"Devon" ✓ found · "Blackwood" ✓ found`.

### Blurry image passing as clear (open)

If the LLM wrongly says a photo is "clear" when it's actually too blurry to read
properly, the workflow moves on to extract the name and date from that same bad image.
The extraction could pull garbled or wrong text, causing a false "name mismatch" or "date
invalid" message to the customer, when the real problem was just a bad photo.

**Fix (not yet implemented):** add a sanity check after extraction. If OCR returns empty,
very short, or clearly malformed text, treat that as a sign the image was actually
unreadable, and route to human review instead of trusting the extracted values. Today,
only the date side has a related safeguard: it escalates when there's too little
information (fewer than two plausible dates). A dedicated check for garbled or
too-short OCR text, applied before name/date comparison, is not yet implemented.

---

## What's in the queue

Six rows, five mocked plus one built from the notes above, each demonstrating a
different triage outcome:

| Customer | Status | Demonstrates |
|---|---|---|
| Peter Parker | Resolved by Agent | Name mismatch, auto message sent, customer disputed, human agent manually closed it |
| John Smith | Needs Review | Image too blurry to classify confidently ("unsure"), routed to human, no message sent |
| Barbie | Pending Customer | Expired ID, auto message sent, case stays open until customer resubmits |
| Riley Morgan | Needs Review | LLM classification response invalid twice in a row, retry limit hit, escalated to human |
| Jordan Blake | Pending Customer | Both name AND expiry wrong at once, concatenated auto message sent |
| Devon Blackwood | Needs Review | Only one date found on the card, not enough to sort chronologically, escalated |

**Status model:**

- **Needs Review** (red): system couldn't confidently triage, or a customer disputed.
- **Resolved by Agent** (indigo): a human manually closed it, shows agent name.
- **Pending Customer** (amber): system sent an automated message, no agent involved, case
  is not resolved, waiting on the customer to act.

## Try it: Create Case

Upload an ID image and enter a profile name to run it through the triage logic above,
live, in the browser.

- **Image read:** `FileReader`, no upload to a server.
- **Blur check:** Laplacian-variance sharpness score computed on `<canvas>` pixel data.
- **OCR:** [OCR.space](https://ocr.space/ocrapi/freekey)'s free-tier API, called directly
  from the browser. Its `ParsedText` response is one flat, unstructured text blob: no
  labeled fields, no coordinates. That's exactly why extraction relies on chronological
  ordering and token matching instead of clean field access. This is noticeably fuzzier
  than the clean, mocked data in the other rows: a real limitation of free-tier OCR.
- **Persistence:** new cases save to `localStorage` and reload with the page.
- The OCR.space API key is visible in client-side JS. That's an accepted shortcut for a
  disposable demo prototype, not a production pattern.

## Known limitations

- OCR.space's free-tier `ParsedText` is one flat, unstructured text blob. Extraction
  relies on regex and heuristics, not clean field access.
- No real backend or database. This is a portfolio/interview prototype.
- The OCR.space API key is exposed client-side.
- No sorting or filtering on the queue. Every case is shown in one flat list.
