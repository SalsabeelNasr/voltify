# Voltify: KYC Review Prototype

**Live demo:** https://voltify-kyc.netlify.app/

A lightweight prototype for automating triage of failed KYC (ID verification) checks,
built to prove the approach before asking engineering to build it for real.

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

## Variables

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

## Assumptions

- Single static `index.html` file. No build step, no backend, deployed on Netlify.
- No API key infrastructure to call a real vision LLM from the browser.
- OCR.space, used for text extraction, is a traditional OCR service, not an LLM. It works
  from the browser today with no backend needed.
- Dates on the ID are assumed to be in `MM/DD/YYYY` format (US-style, slash-separated).
  The extraction regex only matches that shape, and always reads the first number as the
  month. A date like `03/04/2025` is read as March 4, not April 3.
- The only dates assumed present on the card are date of birth, issue date, and
  expiration. Other dates some IDs print, residency start dates, historical law or
  enactment references, endorsement dates, document version years, are not accounted for.
  Any of those would be treated as a plausible date and could throw off the chronological
  ordering the expiry check relies on.
- OCR.space's free-tier `ParsedText` is one flat, unstructured text blob. Extraction
  relies on regex and heuristics, not clean field access.
- The OCR.space API key is exposed in client-side JS. Accepted for a disposable demo
  prototype, not a production pattern.
- No sorting or filtering on the queue. Every case is shown in one flat list.
- No real database. This is a portfolio/interview prototype, not a production system.
- New cases created through the app save to `localStorage` and reload with the page.

---

## Challenges

- **Judging image quality**
  - Needs real visual judgment. No simple rule does that without an image API.
  - Fix: on-device Laplacian-variance sharpness score. Blurry / clear / unsure.
  - Deterministic, so no LLM retry-failure mode. That path is shown by the "Riley Morgan" row instead.

- **Name printed in swapped order**
  - Example: card shows `SAMPLE` / `JANICE` instead of `Janice Sample`.
  - An exact `"First Last"` match would wrongly flag this as a mismatch.
  - Fix: token-based match. Each word checked independently. Order and line breaks don't matter.

- **Right date among three on one card**
  - Example: Issue `03/15/2018`, Expiry `04/30/2028`, DOB `04/30/2000`.
  - Searching near a label like `"Exp"`, then guessing, can pick the DOB by mistake.
  - Fix: chronological order. DOB earliest, expiry latest, always. Issue date sits unused. Label match is a cross-check only, never the decider.

---

## Deterministic logic and the human-in-the-loop mechanism

No LLM is called anywhere in this implementation. Image quality, name matching, and date
matching are all rule-based:

- **Image quality:** an on-device Laplacian-variance sharpness score.
- **Name matching:** a token-based presence check, not a single exact-phrase comparison.
- **Expiry date check:** chronological ordering of every date on the card, cross-checked
  against a recognized label only when one exists and unambiguously names a single date.
- **Customer messages:** static, pre-written templates. No free text generation, so no
  hallucination risk in the messages themselves.

**Where the human checkpoint sits:**

- A borderline sharpness score ("unsure") goes to a human. Nothing gets sent
  automatically.
- Extraction refuses to guess: fewer than two plausible dates, or a genuine disagreement
  between the ordering guess and a recognized label, both escalate to a human.
- Every auto-sent message discloses that the check was automated (M5) and invites the
  customer to flag it for human review. Peter Parker's mocked row shows this end to end:
  customer disputes, a human reviews it, agent resolves the case.

**If a real vision LLM were used instead** (the pseudocode's Step 1 still describes this
as the production design): every response would be checked against a fixed list of 3
allowed answers. A response that doesn't match one of them gets retried once. If it still
fails, don't guess, go straight to a human. That's the mechanism for catching a wrong or
hallucinated model answer.

---

## Possible cases

The triage logic can produce these outcomes:

| Status | Scenario |
|---|---|
| Resolved by Agent | Name mismatch, automated message sent, customer disputes it, a human agent manually closes the case |
| Needs Review | Image quality inconclusive ("unsure"), routed to a human, no message sent |
| Pending Customer | Expired ID, automated message sent, case stays open until resubmission |
| Needs Review | Classification failed validation twice in a row, retry limit hit, escalated to a human |
| Pending Customer | Both name and expiry wrong at once, one concatenated automated message sent |
| Needs Review | Only one date found on the card, not enough to sort chronologically, escalated |

**Status model:**

- **Needs Review** (red): system couldn't confidently triage, or a customer disputed.
- **Resolved by Agent** (indigo): a human manually closed it, shows agent name.
- **Pending Customer** (amber): system sent an automated message, no agent involved, case
  is not resolved, waiting on the customer to act.

## Further work

- **Sharpness isn't the same as legibility.** A blurry photo can score "clear." A sharp, high-res photo can score artificially low if downscaling smooths away real detail before the check runs.
- **OCR can "succeed" and still be wrong.** No error, no short-text flag, but garbled text. Nothing catches this today.
- **Name matching has no field awareness.** Checks if a word appears anywhere in the document. `"Jordan Blake"` would match against an unrelated `"123 Jordan Street"`.
- **Date format is hardcoded.** Only `MM/DD/YYYY`, slash-separated. Misses `DD/MM/YYYY`, dots, dashes, and spelled-out months like `15 JAN 2028`.
- **Generic OCR sweep instead of a per-card template.** Runs OCR across the whole image and sorts out the text afterward, instead of using a bounding-box template tailored to that specific card design.
  - Fixed coordinates: since official ID layouts are standardized, map expected field zones (name, DOB, ID number, expiry) as relative coordinate percentages.
  - Targeted extraction: run OCR only inside the relevant box. Looking for the expiry date only reads the expiry zone, eliminating any risk of confusing it with an issue date or a stray serial number.
