# Voltify: KYC Review Prototype

**Live demo:** https://voltify-kyc.netlify.app/

A lightweight prototype for automating triage of failed KYC (ID verification) checks,
built to prove the approach before asking engineering to build it for real.

---

## How it works

```mermaid
flowchart TD
    subgraph System["System (deterministic, no LLM)"]
        B{Blur check}
        C[OCR extraction]
        D{Name / date check}
        E[Send templated message]
    end

    subgraph Support["Support agent"]
        F[Manual review]
        G[Resolve case]
    end

    subgraph Customer
        A[Submit ID photo + entered name]
        H[Receives automated message]
        I[Disputes the message]
        K[Resubmits a corrected ID]
    end

    A --> B
    B -->|blurry| E
    B -->|unsure| F
    B -->|clear| C
    C --> D
    D -->|not enough dates, or label disagrees| F
    D -->|name and/or date mismatch| E
    D -->|no issue found| F
    E --> H
    H --> I
    H --> K
    I -->|flags it via disclosure| F
    K -->|resubmission passes checks, closes automatically| G
    F --> G
```

Same flow, step by step:

```
INPUT: customer_id_submission (image, entered_name)

STEP 1: Image quality check (deterministic, on-device)
  # Laplacian-variance sharpness score, computed on the uploaded image via canvas
  # pixel data. No API call, no LLM.
  score = compute_sharpness(image)

  IF score < 30:                      # BLUR_THRESHOLD
      classification = "blurry"
  ELIF score > 100:                   # CLEAR_THRESHOLD
      classification = "clear"
  ELSE:
      classification = "unsure"

  IF classification == "blurry":
      SEND message = M1 + M5
      LOG triage_note = "blurry photo auto-detected, message sent"
      STOP

  IF classification == "unsure":
      flag_for_human_review(reason="low_confidence_blur_check")
      STOP

  # classification == "clear", proceed to extraction

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

STEP 6: Customer responds to the sent message
  IF customer disputes via the disclosure link (M5):
      flag_for_human_review(reason="customer_disputed")
  ELIF customer resubmits a corrected ID:
      re-run STEPS 1-4 on the new submission
      IF no name_issue AND no date_issue:
          CLOSE case automatically
          LOG triage_note = "resubmission passed, case closed automatically"
      # otherwise the new submission's own STEP 4 outcome applies again
```

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
- A dispute is the only thing that routes a sent message to a human. If the customer
  instead just fixes the problem and resubmits, and the resubmission passes, the case
  closes automatically. No human needed for the good-outcome path.

**If a real vision LLM were used instead** (the pseudocode's Step 1 still describes this
as the production design): every response would be checked against a fixed list of 3
allowed answers. A response that doesn't match one of them gets retried once. If it still
fails, don't guess, go straight to a human. That's the mechanism for catching a wrong or
hallucinated model answer.

---

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

## Example Outputs

Three invented rejection scenarios. Each one shows the internal triage note (STEP 4's
`LOG`) next to the actual customer message (M5 always appended, per the rule above).

**Scenario 1: Blurry ID**

Internal triage note:

`blurry photo auto-detected, message sent`

Customer:

"Your ID photo appears too blurry for us to verify. Please retake it in good lighting and
resubmit. This check was completed automatically. If you think something's wrong, flag it
here and we'll open a case for a specialist to review."

**Scenario 2: Expired ID**

Internal triage note:

`date expired, message sent`

Customer:

"The ID you submitted appears to be expired. Please upload a valid, unexpired ID. This
check was completed automatically. If you think something's wrong, flag it here and we'll
open a case for a specialist to review."

**Scenario 3: Name mismatch**

Internal triage note:

`name mismatch, message sent`

Customer:

"We noticed the name on your ID doesn't quite match what's on file. This check was
completed automatically. If you think something's wrong, flag it here and we'll open a
case for a specialist to review."

---

## Assumptions

- Single static `index.html` file. No build step, no backend, deployed on Netlify.
- No backend to hold a real vision LLM's API key securely, so this prototype uses
  OCR.space instead: a traditional OCR API, not an LLM, whose key can be called directly
  from the browser. That key ends up exposed in client-side JS, accepted for a disposable
  demo, not a production pattern.
- OCR.space's free-tier `ParsedText` is one flat, unstructured text blob. Extraction
  relies on regex and heuristics, not clean field access.
- Dates on the ID are assumed to be in `MM/DD/YYYY` format (US-style, slash-separated).
  The extraction regex only matches that shape, and always reads the first number as the
  month. A date like `03/04/2025` is read as March 4, not April 3.
- The only dates assumed present on the card are date of birth, issue date, and
  expiration. Other dates some IDs print, residency start dates, historical law or
  enactment references, endorsement dates, document version years, are not accounted for.
  Any of those would be treated as a plausible date and could throw off the chronological
  ordering the expiry check relies on.
- No sorting or filtering on the queue. Every case is shown in one flat list.
- No real database. This is a portfolio/interview prototype, not a production system.
- New cases created through the app save to `localStorage` and reload with the page.

---

## Biggest risk

**Risk:** the system is meant to reduce support workload, but a wrong auto-response can
add to it instead. If a customer gets an automated message that's incorrect or doesn't
make sense for their situation, they get frustrated, and support now has to handle both
the original case and an upset customer.

**Mitigation:** never auto-send on low-confidence data, escalate to a human instead. See
the human-in-the-loop mechanism above.

---

## Challenges

Grouped by what part of the ID they affect: image, date, or name.

### Image

| Challenge | Status | Details |
|---|---|---|
| Judging image quality | Addressed | - Blur needs real visual judgment. No simple rule can do that alone.<br>- Fix: an on-device sharpness score. Sorts the photo into blurry, clear, or unsure.<br>- Only solves blur. Doesn't catch scratches or bad sizing.<br>- Being rule-based, it can't fail the way an LLM would (an invalid response). That failure mode is shown by the "Riley Morgan" row instead. |
| Sharpness vs. legibility | Not addressed | - A blurry photo can still score "clear."<br>- A sharp photo can score low if it gets shrunk before checking.<br>- Doesn't catch a scratch over a character, text too small, or text cropped too large either. Sharpness only measures focus, not damage or framing.<br>- Still open. |
| OCR inventing data from noise | Not addressed | - A smudge on a blurry or glare-washed photo can get misread as a real character, instead of returning nothing.<br>- That invented text can still look like a valid date or name.<br>- Just checking "is something there" isn't enough. Needs real validation. Not implemented. |
| Reading the whole image instead of set fields | Not addressed | - OCR reads the entire photo, then the system sorts out the text after.<br>- Fix idea: map out where the name, DOB, ID number, and expiry sit on each country's ID layout, as fixed zones. Only read inside each zone. |

### Date

| Challenge | Status | Details |
|---|---|---|
| Picking the right date off a card with three dates | Addressed | - Example: Issue `03/15/2018`, Expiry `04/30/2028`, DOB `04/30/2000`.<br>- Searching near a label like "Exp," then guessing, can pick the DOB by mistake.<br>- Fix: dates always go DOB, then Issue, then Expiry, in that order. The system uses that order instead of guessing from a label. Label match is a cross-check only, never the decider.<br>- Solves the multiple-dates problem. Date format is a separate, still-open problem below. |
| Only one date format supported | Not addressed | - Only reads `MM/DD/YYYY`, slash-separated.<br>- Misses `DD/MM/YYYY`, dots, dashes, or spelled-out months like `15 JAN 2028`. |
| No sanity check on age or ID validity length | Not addressed | - Doesn't check if the birth date makes sense for an ID. A birth date implying a 5-year-old would still be accepted.<br>- Doesn't check if the issue-to-expiry gap matches a normal validity period (often around 7 years, varies by issuer).<br>- Fix idea: add both as extra checks on top of the date ordering. |

### Name

| Challenge | Status | Details |
|---|---|---|
| Name printed in swapped order | Addressed | - Example: card shows `SAMPLE` / `JANICE` instead of `Janice Sample`.<br>- A strict `"First Last"` match would wrongly flag this as a mismatch.<br>- Fix: check each word of the name separately. Order and line breaks don't matter anymore.<br>- Limit: finding a word doesn't confirm it came from the name field, or that it's the right person's name. |
| Name matching doesn't know which field it's in | Not addressed | - Matches a word anywhere on the card, not just the name field.<br>- Example: `"Jordan Blake"` would match `"123 Jordan Street"`.<br>- Fix idea: same field-mapping idea as the image fix above. Needs more investigation. |
| Two people with swapped names could match each other | Not addressed | - Example: `"James Robert"` and `"Robert James"` are different people.<br>- The system only checks if each word is present, not which field it's in.<br>- Entering one name could wrongly match the other person's ID.<br>- Fix idea: same field-mapping idea, matched by field label (First Name vs. Last Name), not by print position. This still works with the swapped-order fix above: a real person's name printed in reverse order still matches correctly once fields are matched by label instead of position. |

---

## Statistical significance

- The prototype is not making the KYC decision. It is identifying the likely reason an already-failed KYC check failed.
- Because of that, the goal is not to handle 100% of possible OCR or document errors. The goal is to automate the common, high-confidence cases and reduce repetitive CS work.
- Some edge cases will always exist, and some require more complex solutions like:
        - slicing the image to known geometric regions
        - defining date formats
        - extra image quality checks
- The likelyhood of the errors occoring 
- In further work, I would test with real data the error rate and how many cases can safely be automated this way vs. using an LLM 

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
| Closed | Customer resubmits a corrected ID, it passes the automated checks, case closes automatically, no human involved |

**Status model:**

- **Needs Review** (red): system couldn't confidently triage, or a customer disputed.
- **Resolved by Agent** (indigo): a human manually closed it, shows agent name.
- **Pending Customer** (amber): system sent an automated message, no agent involved, case
  is not resolved, waiting on the customer to act.
- **Closed** (green): a resubmission passed every check, no dispute, no agent involved.
