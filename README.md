# Voltify: KYC Review Prototype

**Live demo:** https://voltify-kyc.netlify.app/

A lightweight prototype for automating triage of failed KYC (ID verification) checks,
built to prove the approach before asking engineering to build it for real.

---

## Scenarios

Three invented rejection scenarios this prototype handles end to end. Full triage note and
customer message for each: see Sample space data below.

- Scenario 1: Blurry ID
- Scenario 2: Expired ID
- Scenario 3: Name mismatch

---

## Biggest risk

**Risk:** the system is meant to reduce support workload, but a wrong auto-response can
add to it instead. If a customer gets an automated message that's incorrect or doesn't
make sense for their situation, they get frustrated, and support now has to handle both
the original case and an upset customer.

**Mitigation:** never auto-send on low-confidence data, escalate to a human instead.

---

## Assumptions

- Single static `index.html` file. No build step, no backend, deployed on Netlify.
- The 10 mocked queue rows are the deterministic path's real output (OCR.space, a
  traditional OCR API, not an LLM), verified against the actual code. The LLM readings
  documented below them were produced by reading the same card images directly in
  conversation (Claude), not through a live API call, so they're a comparison, not a
  second live path in the mocked data.
- Create Case does have a live LLM path: pick "LLM (live API)" and paste an OpenAI API key
  to run a real request (GPT-4o) against the same prompt documented below. That's a
  different model than the one used for the documented comparison, so a live run isn't
  guaranteed to match those readings exactly, the underlying questions being asked are the
  same either way. The key is only held in memory for that one request, never written to
  `localStorage`, but it's still typed into client-side JS and sent straight from the
  browser, fine for trying this out yourself, not a pattern to ship. A production version
  would hold the key behind a server-side proxy instead, the same gap that already exists
  for the OCR.space key used by the deterministic path.
- The only dates assumed present on the card are date of birth, issue date, and
  expiration. Other dates some IDs have are not accounted for.
- No real database. This is a portfolio/interview prototype, not a production system.
- New cases created through the app save to `localStorage` and reload with the page.

---

## Sample space data

Every mocked case in the queue is a verified result of running the actual deterministic
logic against its image, not hand-typed data. See them live:
**https://voltify-kyc.netlify.app/**

| Case | Scenario | What makes it a test case |
|---|---|---|
| Peter Parker | Name mismatch | No real date field on the card at all, only an `ID:` number |
| John Smith | Blurry ID | Deliberately blurry photo |
| Barbie | Name mismatch | Dashes instead of slashes, expiry printed as `NEVER` |
| Riley Morgan | (none, edge case) | Name and date both check out, but the case still arrived as an already-failed KYC check |
| Jordan Blake | Name mismatch | Name printed abbreviated, `J. BLAKE` |
| Devon Blackwood | (none, edge case) | Only one date on the card, `CLASS: PERMANENT` |
| Alex Anderson | (none, edge case) | Date printed DD/MM instead of MM/DD |
| Brenna Murphy | Expired ID | Name printed reversed, comma-separated |
| Janice Sample | Expired ID | Name printed reversed, comma-separated |
| Michael Motorist | Expired ID | Name printed reversed, comma-separated |

**This is a hand-picked demonstration set, not a statistical sample.** 10 cases is enough
to show each scenario type at least once, and to surface specific edge cases (reversed
names, unusual date formats, a case with no explainable failure reason). It is not enough
to measure an error rate or prove one approach is more accurate than the other, that
requires a larger, labeled, real dataset, see Further work.

---

Two approaches were tested for finding the failure reason behind each case above: a
deterministic rule-based pipeline, and an LLM reading the same images. Below: how each one
works, where each still has real gaps and whether those gaps could actually be mitigated,
and how their outputs compared case by case.

<details>
<summary><strong>Deterministic approach</strong> (click to expand)</summary>

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

### No LLM approach: how each check works

#### Image blur

- Draw the photo onto a canvas, shrunk to 300px on the long side.
- Convert to grayscale.
- Run a Laplacian filter, measure the variance.
- Score under 30: blurry. Over 100: clear. In between: unsure.

#### Name matching

- Split the entered name into words.
- Check each word on its own, anywhere in the OCR text.
- Not case-sensitive. Order doesn't matter.
- Every word has to be found for a match.

#### Expiry date

- Find every date in the OCR text, slash, dash, or dot separated.
- Whichever number is over 12 must be the day, that's how day/month order gets figured
  out automatically. If both numbers could be either, default to month first.
- Drop anything outside a plausible range (120 years back, 15 years forward).
- Sort what's left. The latest date is the expiry.
- If a label like "Exp" points at one clear date, use it as a cross-check only.
- Cross-check disagrees with the sorted guess: stop, send to a human.

## Challenges

What can go wrong with each part of the deterministic pipeline, whether it's actually been
mitigated in the current code, and the recommended fix where it hasn't.

- **Image**
  - **Sharpness ≠ legibility.** *Not mitigated.* A sharpness score can miss scratches,
    glare, bad cropping, or text that's technically sharp but unreadable.
    **Recommended mitigation:** add real image quality and legibility checks.
  - **OCR can misread text.** *Not mitigated.* A bad image can produce a plausible but
    incorrect name or date.
    **Recommended mitigation:** validate extracted fields before using them.
  - **Whole-image OCR.** *Not mitigated.* The current approach reads one text blob, so it
    doesn't know which text belongs to which field.
    **Recommended mitigation:** read known fields by their position on the ID.

- **Date**
  - **Ambiguous dates.** *Partially mitigated.* `03/04/2025` used to always default to
    month-first. Now, whichever number is over 12 auto-resolves as the day, so most real
    dates disambiguate correctly. When both numbers are 12 or under, there's genuinely
    nothing on the card to resolve it, and it still defaults to a guess.
    **Recommended mitigation:** use the document layout, or flag genuinely ambiguous dates
    for review instead of defaulting silently.
  - **Unusual date formats.** *Not mitigated.* Spelled-out dates (`15 JAN 2028`) or
    non-standard expiry values (like `NEVER`) are still missed, only slash/dash/dot
    numeric separators are supported.
    **Recommended mitigation:** support more formats and define how values like `NEVER`
    are handled.
  - **Limited validation.** *Not mitigated.* The current logic doesn't check whether the
    dates make sense for the person's age, or whether the validity period is plausible.
    **Recommended mitigation:** add these as secondary checks.

- **Name**
  - **Name matching isn't field-aware.** *Not mitigated.* The current approach looks for
    each name word anywhere in the OCR text. This can create false matches.
    **Recommended mitigation:** match names within the actual name fields.
  - **Name order is ignored.** *Partially mitigated, as a side effect.* Checking each word
    independently already lets reversed-name cards (Brenna Murphy, Janice Sample, Michael
    Motorist) match correctly. But this isn't a real fix, the system still can't tell a
    legitimate reversed name apart from a coincidental match or someone else's card.
    **Recommended mitigation:** identify first and last name fields instead of only
    checking whether the words exist.

</details>

---

<details>
<summary><strong>LLM solution</strong> (click to expand)</summary>

## LLM solution

For comparison, the card images used in the queue were read directly by a vision-capable
LLM, the same way the deterministic pipeline gets demonstrated on all 10 mocked rows.
Here's what it produced for each.

### Prompt

No live API call was made for these readings, they were produced by reading each image
directly within this conversation, not through a separate prompted API request. So this
isn't a transcript of an actual call. It's the prompt a real implementation would send,
written to ask for the same fields the deterministic pipeline checks (image quality, name,
expiry), in a fixed format so a malformed response is easy to detect (see Human in the
loop below). This prompt is intentionally simple, see Further work for why a hardened
version of it is the fairer comparison to run next:

```
Here's a photo of a customer's ID [image attached].

Read the card and tell me:
- Image Quality : "clear" , "unsure" , "blurry"
- First and Last name as printed, or NULL
- Is it expired? or NULL

Only report what's actually visible on the card, don't guess or fill in anything that
isn't printed. If you are unsure if the image quality is clear or blurry always return
"unsure"
```

**Blurry ID (John Smith)**

- Deterministic: refuses to read it. Sharpness score 12 auto-classifies as blurry before
  OCR ever runs.
- LLM: can still make out most of the card despite the blur, "JOHN SMITH," a birth date,
  an expiry, an ID number, using context (expected card layout) to fill in what individual
  pixels don't fully resolve.
- Risk: an LLM reading through blur isn't automatically a good thing. Getting a digit
  wrong while sounding confident is a worse failure than correctly saying "too blurry."

**Name mismatch (Peter Parker)**

- Deterministic: reads "LAST NAME: Man, FIRST NAME: Spider." No slash-formatted date
  anywhere on the card, so it stops at "insufficient dates found" and never even reaches a
  name decision.
- LLM: reads the same fields, easily recognizes this is a joke Spider-Man license, and
  would compare "Spider Man" against "Peter Parker" as a mismatch. `ID: 08 10 19 62` is
  clearly labeled as the ID number, not a date, so a careful read wouldn't try to parse it
  as one. There's no real date anywhere on this card either way.

**A hallucination that happened during this run, and why it matters**

On the first read of this card, the LLM read `ID: 08 10 19 62` as a plausible date instead
of what it actually is: a number clearly labeled `ID:` right on the card. That's the exact
failure mode this whole document warns about, pattern-matching something that looks
numeric into the shape you're expecting to find, instead of reading what's actually
labeled. It got caught and corrected here. In a real pipeline with no one checking, that
reading would have gone straight into a decision. This isn't a hypothetical risk being
described, it's the same mistake happening in the process of writing this comparison.

**Name + date mismatch (Jordan Blake)**

- Deterministic: "J. BLAKE" doesn't contain the token "Jordan," so it's flagged as a hard
  mismatch. Expiry 01/05/2024 is correctly read as expired.
- LLM: also reads the card correctly, but can reasonably judge "J. Blake" as very likely
  an abbreviation of "Jordan Blake," a judgment call the token check has no way to make.
  Same conclusion on the expired date.
- Why that's not necessarily good: KYC name checks are usually assumed to need an exact
  match, not a plausible one. The LLM accepting "J." as "Jordan" is a guess dressed up as
  a read. It might be right most of the time, but "probably the same person" isn't the bar
  identity verification is supposed to clear, and this is exactly the kind of judgment
  call that should go to a human instead of getting auto-approved (see Human in the loop,
  below).

**Insufficient dates, dash format (Barbie)**

- Deterministic: birth date prints as `03-10-1959` (dashes, not slashes) and expiry reads
  `NEVER`. Neither matches the parser's date regex, so it finds 0 dates and stops at
  "insufficient dates found."
- LLM: can parse `03-10-1959` as a date regardless of the dashes, no format rigidity. It
  can also read `EXPIRES: NEVER` as a real (if unusual) value rather than just failing to
  match a pattern.

**Insufficient dates, only one printed (Devon Blackwood)**

- Deterministic: only one date on the card at all (date of birth). Needs two to sort
  chronologically, so it stops at "insufficient dates found," same reason as Barbie's row
  but for a different cause.
- LLM: reads the same single date, but can also read `CLASS: PERMANENT` and reasonably
  infer that's *why* there's no expiry field, some ID types genuinely don't expire, rather
  than assuming data is simply missing. Again, a contextual read the deterministic system
  isn't built to make.

**Unexpected clear result (Riley Morgan)**

- Deterministic: name checks out (`Riley`, `Morgan` both found), expiry `11/14/2027` is
  unexpired. Both checks pass, but the case still needs a human look, since it arrived
  here as an already-failed KYC check with no automated reason pointing to why.
- LLM: reads the same fields the same way, `RILEY MORGAN`, `04/22/1994`, `11/14/2027`,
  and reaches the same conclusion: nothing here explains the original failure. Where an
  LLM pipeline could fail differently on a case like this isn't the reading, it's
  elsewhere: the API call itself timing out or erroring, a different failure mode than a
  text-extraction service returning bad data.

### Challenges

The same challenges documented for the deterministic approach above, checked one by one
against an LLM approach.

- **Image**
  - **Sharpness ≠ legibility.** *Not mitigated, different shape.* The prompt only asks for
    the same three-way call the deterministic check makes (clear, unsure, blurry), so
    that's the only image challenge being compared here, not image quality problems in
    general. No numeric score to set a threshold on, so "how blurry is too blurry" becomes
    a judgment call instead of a number. The instruction to default to "unsure" when in
    doubt is the only guardrail on that judgment. Actually improved in one way: an LLM can
    likely tell a scratch over a letter apart from ordinary blur, something a sharpness
    score treats the same, though the prompt doesn't ask about scratches or cropping
    specifically, so that's a possible improvement, not a tested one. The risk of reading
    through blur instead of correctly refusing it is covered under the John Smith case
    above, not repeated here.
  - **OCR can misread text** and **whole-image OCR** aren't image-quality questions, they're
    about extracting the name and date correctly. Covered under Name and Date below instead
    of here.

- **Date**
  - **Ambiguous dates.** *Not mitigated.* Not fixed by being a better reader. For
    `03/04/2025` (both numbers ≤12), there's no information on the card resolving which
    is which. Without a country/format hint, either printed on the card or given in the
    prompt, an LLM has the same missing information the deterministic parser does, and
    is guessing too.
  - **Unusual date formats.** *Mitigated.* An LLM reads spelled-out months, unusual
    separators, and unusual values like `EXPIRES: NEVER` without needing a new rule
    written for each one.
  - **Limited validation.** *Not mitigated.* Not automatic. An LLM won't spontaneously
    flag an implausible age or validity gap unless it's explicitly asked to. Same gap,
    just moved from "missing code" to "missing prompt instruction."

- **Name**
  - **Name matching isn't field-aware.** *Mostly mitigated.* An LLM naturally reads
    label-value pairs ("FULL NAME: ...") using the card's layout, so it isn't just
    checking presence anywhere in a flat text blob, it can check the right field. That
    also means it doesn't need a separate zone map the way flat OCR text does.
  - **Name order is ignored.** *Mostly mitigated, not gone.* An LLM can read which field is
    actually labeled "First Name" vs. "Last Name." But it still can't tell two different
    people apart if they genuinely share the exact same full name. No reading approach,
    deterministic or LLM, can solve that on its own.

</details>

---

## Output comparison

Every case in the queue, deterministic result vs. LLM reading, side by side.

| Case | Deterministic output | LLM output | Agree? |
|---|---|---|---|
| Peter Parker | Insufficient dates found → Needs Review, no message sent | Corrected read: no real date on the card, name reads "Spider Man" vs. entered "Peter Parker". First read hallucinated: misread the printed `ID: 08 10 19 62` (labeled as an ID number) as if it were a date | Yes on the corrected read, not on the first one |
| John Smith | Sharpness score 12, below threshold → blurry, auto-sent M1 | Can still make out most fields despite the blur | No. LLM reads through what the deterministic check correctly refuses |
| Barbie | 1 date only (dash format) → insufficient dates found → Needs Review | Can parse the dash-format birth date and the "NEVER" expiry as real values | No. LLM would extract more, but "NEVER" still isn't a computable expiry either way |
| Riley Morgan | Name and date both check out → unexpected clear result, Needs Review | Same reading, same conclusion | Yes |
| Jordan Blake | "Jordan" not found in "J. BLAKE" → hard mismatch; expiry 01/05/2024 expired | Same date reading; would judge "J." as a likely match for "Jordan," a guess the deterministic check doesn't make | Partial. Same date, different name verdict |
| Devon Blackwood | Only 1 date (DOB) → insufficient dates found → Needs Review | Same reading, could reasonably infer "CLASS: PERMANENT" explains the missing expiry | Yes on the outcome, LLM adds context |
| Alex Anderson | Name matches, DD/MM date now parsed correctly (not expired) → unexpected clear result | Same reading, same conclusion | Yes |
| Brenna Murphy | Name matches (reversed, comma-separated), expiry 08/20/2020 expired → auto-sent message | Same reading, same conclusion | Yes |
| Janice Sample | Name matches, expiry 08/05/2023 expired → auto-sent message | Same reading, same conclusion | Yes |
| Michael Motorist | Name matches (reversed, comma-separated), expiry 08/31/2021 expired → auto-sent message | Same reading, same conclusion | Yes |

---

## Actions after reason is found

Once a check finishes, this is what happens with the outcome, the case status it becomes
and the message that goes out. Same rules either way, whether the check that produced the
reason was deterministic or an LLM read, both approaches build the exact same case shape
(status, validation rows, timeline, follow-up action) from whatever they find.

### Case status

| Status | Meaning |
|---|---|
| **Needs Review** (red) | Couldn't confidently triage, or a customer disputed |
| **Resolved by Agent** (indigo) | A human manually closed it |
| **Pending Customer** (amber) | Automated message sent, waiting on the customer |
| **Closed** (green) | A resubmission passed every check automatically |

### Customer message

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

## Approaches comparison

| | Deterministic (this prototype) | LLM |
|---|---|---|
| Model | None. Laplacian-variance math + regex + string matching. | Claude (vision-capable multimodal model) |
| Why it decided something | Fully traceable. Every decision maps to one line of pseudocode. | Not traceable. No way to know why it accepted an abbreviation without asking it, and it could be wrong. |
| Hallucination risk in determining the failure reason | The actual task on both sides, never writing the message (that's always a fixed template once a reason is picked). A misread digit or field can produce the wrong reason. | Same task, higher risk: a wrong-but-confident field read produces the wrong reason, with nothing built in to catch it. |
| Cost per case | Free-tier OCR call, near-zero. | A paid API call per image, real ongoing cost at volume. |
| Speed | Fast, single OCR round trip. | Slower, and depends on the provider's response time. |
| Consistency | Same image always produces the same result. | Can vary between runs on the same image. |

---

## Human in the loop

Where each approach stops and sends a case to a human instead of deciding on its own:

| Trigger | Deterministic | LLM |
|---|---|---|
| Image quality too poor for a reliable read | Should not try to make a decision from unreliable text, e.g. John Smith. Sharpness score lands below the clear threshold or in the "unsure" range, nothing sent automatically. | Reports "unsure", or the image is too unclear to read reliably (John Smith). Reading through blur can produce a convincing but wrong value. |
| A value can't be parsed into a known format | Needs a defined rule before it can act, e.g. an unusual date format or `EXPIRES: NEVER`. Same reason fewer than two plausible dates on a card stops the check entirely. | `EXPIRES: NEVER` or `CLASS: PERMANENT`. The LLM can identify these values, but the system still needs a defined business rule before acting on them. |
| The name match is ambiguous | Should not make assumptions about whether two names are the same person, e.g. `J. BLAKE` vs. `Jordan Blake`. Only exact token presence counts, there's no interpretation step to get wrong. | `J. BLAKE` read as a likely match for `Jordan Blake`. The LLM can suggest it, but it shouldn't be auto-approved. |
| Different checks produce conflicting results | E.g. the extracted dates and a recognized label don't agree, stops rather than picking one. | No equivalent cross-check exists in a single read, nothing here re-reads the same field a second way to catch a disagreement. |
| Customer disputes the automated message | The only human path actually wired up in this prototype (Jordan Blake's mocked row shows it end to end: dispute, human review, agent resolves). | Same disclosure and dispute mechanism, this part of the workflow doesn't change based on which engine produced the message. |

The deterministic approach is intentionally conservative: when the rules can't reach a
clear answer, it escalates rather than guessing. The LLM can read more context than a
fixed set of rules, but that doesn't automatically make its interpretation safe to act on
either.

---

## Conclusion

The goal is simple: figure out why a KYC check failed, and take the right action. Tested
on the 10-case sample, the deterministic approach and the LLM agreed 7 times out of 10.

**Every disagreement, looked at closely, is either a fixable deterministic gap or a
judgment call neither approach should make automatically:**

- **Barbie** (unusual date format): the deterministic parser didn't support that
  separator. That's a rule you can add, it's not a reason to reach for an LLM.
- **John Smith** (blurry) and **Jordan Blake** (abbreviated name): these are genuine
  judgment calls, is this photo readable, is this abbreviation the same person, and an
  LLM answering them with unearned confidence is a worse outcome than the deterministic
  system's honest "I can't tell." These belong with a human either way.

There's no case in this sample where an LLM safely resolved something that couldn't be
handled just as well by widening the deterministic rules or by escalating to a human. For
the symbolic parts of the problem, dates and names, that's a strong conclusion: an
enumerable rule set beats a probabilistic reader on cost, traceability, and consistency,
and it doesn't hallucinate. For image quality, the problem is more continuous and
perceptual than symbolic, so a hardened deterministic check still closes most of the gap,
and an LLM's only defensible role is as a stricter secondary quality classifier, never as
a reader of a marginal image, and even then it still has to default to escalating anything
it isn't sure about.

**Recommended approach: deterministic-first, LLM only as a narrow, bounded quality
classifier, human escalation as the fallback for every category, not just runtime
autonomy for either engine.** Finish the deterministic gaps identified above before adding
any LLM step, and never let either approach auto-act on a low-confidence or ambiguous
read.

**The one legitimate exception:** an early-stage or low-volume product that needs to ship
now, where the engineering time to build the full deterministic pipeline isn't available
and API cost isn't the constraint, can reasonably start with a well-prompted LLM doing all
three checks as a v1. That's a real build-vs-buy tradeoff, not a compromise, as long as it
keeps the same non-negotiable rule (never auto-act on a low-confidence or ambiguous
answer) and is treated as a placeholder to replace with deterministic logic as volume or
regulatory scrutiny grows.

---

## Recommendations and further work

**Recommendation:** ship the deterministic approach as the primary engine, close its known
gaps (below) before adding any LLM step, keep image-quality checks deterministic first,
and route every low-confidence or judgment-call case to a human regardless of which engine
produced it.

**Further work, to include more data:**

- Test both approaches against a larger, labeled, real KYC dataset and measure the actual
  error rate. The 10 cases documented here are hand-picked demonstrations, not a measured
  sample, they can't tell you how often either approach is actually wrong.
- Re-run the LLM comparison with a hardened prompt, explicit warnings against the specific
  hallucination seen here (an ID number isn't a date), worked examples, and a
  self-verification step, then see whether the 3 disagreements actually shrink. The
  current comparison used a deliberately simple prompt, so it likely understates what a
  properly prompted LLM would do.

**Further work, to improve the deterministic approach:**

- Read known fields (name, DOB, expiry) by their position on the ID layout instead of
  scanning one flat text blob, this is the fix underlying most of the open Challenges
  above.
- Support more date formats (spelled-out months) and define explicit business rules for
  special values like `NEVER` or a `PERMANENT` class.
- Add secondary validation: does the birth date make sense for an ID holder, is the
  issue-to-expiry gap a plausible length.
- Add targeted image-quality checks beyond a single global sharpness score: per-region
  contrast (to catch localized scratches or damage), glare/saturation detection, and
  crop/aspect-ratio validation.

**Further work, to improve the LLM approach:**

- If used at all, restrict it to a narrow, bounded role, a stricter image-quality
  classifier only, never a reader of a marginal image, and never the sole decision-maker
  on a name or date judgment call.
- Force a strict, fixed response format so a malformed or incomplete answer is treated as
  a failure and escalated, never guessed at.
