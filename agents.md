# Quote Sheet Project Guide

## Project summary

This is a small insurance quote-sheet app built with plain HTML, CSS, and JavaScript. The end-of-term quote page contains the primary product/rate data wiring, quote logic, diagnostics, and premium display behavior. Separate plain-HTML pages are used for the new-business workflow and the year-by-year projection.

The main goal is to keep the project **simple, stable, and easy to hand off**. Do not introduce extra files, frameworks, or major refactors unless the user explicitly asks for that.

## Core operating rules

1. **Read the entire HTML file before changing anything.** This project is small enough that understanding the full file is practical and important.
2. **Keep the existing page boundaries.** Do not split embedded CSS/JavaScript into frameworks or additional support files unless the user specifically requests it. The approved pages are `index.html`, `new-quote.html`, and `projection.html`.
3. **Prefer minimal edits over rewrites.** Preserve working behavior and patch the actual logic that needs to change.
4. **Do not do speculative cleanup.** Avoid reorganizing large sections just because it seems cleaner.
5. **Protect existing quote behavior.** A small UI tweak can easily break premium logic, rider logic, or warning logic.
6. **When behavior is ambiguous, trust the latest user instruction over older code behavior.**
7. **Explain changes plainly.** The user is not asking for deep technical theory. Be practical and direct.

## User collaboration preferences

* The user often wants a plain-English explanation first, then the actual build/update.
* The user does **not** want long code dumps unless explicitly requested.
* Keep explanations practical and focused on what changed, why it changed, and what to test.
* Preserve the current look and flow unless a redesign is requested.
* Prefer compact UI for daily quoting. Debug-style fields such as base rate, waiver rate, and internal premium breakdowns should stay out of the main line cards unless the user asks to inspect calculation internals.
* Only show effective rate class when it differs from the selected rate class, such as preferred being calculated as non-tobacco because preferred is not available.
* The desktop tab order is intentionally custom for fast entry. It starts with client name, spouse-on-policy checkbox, primary DOB, spouse DOB, then each enabled coverage line in product / coverage / rate class / anniversary order. Spouse DOB stays in the tab order even when spouse coverage is unchecked.

## Project structure

* `index.html` is the end-of-term quote page and the source of truth for end-of-term inputs, rate tables, quote calculations, validations, saved drafts, and projection-data generation.
* `new-quote.html` is the separate new-business quote page.
* `projection.html` is the full-page year-by-year presentation opened from `index.html`. It reads a projection snapshot from `sessionStorage`; it is not a standalone quote calculator.
* Each HTML page keeps its own CSS and JavaScript embedded. Preserve this simple structure unless the user requests a broader redesign.

## Business logic notes gathered from prior work

These rules matter and should be preserved unless the user changes them.

### 1\) Coverage amount interpretation

Coverage selections like `50` mean **$50,000 of coverage**, but the rating math may use that value as **50 units of $1,000**.

Example the user explicitly confirmed:

* For a 45-year-old, non-tobacco, under-$150k band, a rate of `5.6` should calculate as:
* `5.6 \\\* 50 + 75 = 355`
* `355 \\\* 0.095 = 33.73` monthly

So in rating math, `50` may be correct as the multiplier because it represents 50 thousand-dollar units.

### 2\) Health / rate class options

The live app uses rate classes, not a separate health field.

Current class meanings:

* `PP` = Preferred Plus
* `P` = Preferred
* `N` = Non-Tobacco
* `T` = Tobacco

Different product lines allow different rate classes:

* Some products allow `PP`, `P`, `N`, and `T`.
* Some products allow `P`, `N`, and `T`, but not `PP`.
* Some older products may not allow preferred or preferred plus below the coverage threshold described below.

There is no current `Excellent` health path and no separate best-class quote path. If older notes mention `Excellent`, best class, or standard-vs-best branching, treat those notes as stale unless the user explicitly asks to rebuild that behavior.

### 3\) Preferred / preferred plus under $150k

Many product lines do not allow preferred or preferred plus when the insured's total eligible coverage is under **$150,000**.

Important: this threshold is based on the **total coverage for that insured**, not each individual product line by itself.

Example:

* Primary has multiple renewing product lines.
* If the primary's combined eligible coverage is at least 150, preferred / preferred plus may be allowed for products that support it.
* If the primary's combined eligible coverage is under 150, preferred / preferred plus may need to be treated as non-tobacco for the calculation.
* Spouse coverage should be evaluated separately using the spouse's own combined eligible coverage.

If the UI allows a preferred or preferred plus selection below $150,000, the quote logic may auto-adjust to non-tobacco. This is expected behavior for affected products.

### 4\) Auto-adjust warning priority

There is an existing warning/diagnostic for rate-class auto-adjustment when preferred is selected below $150,000 and the system switches to non-tobacco.

That warning should **not** appear as a top-priority sticky warning near the premium summary.

The user asked to remove that top-level warning for:

* primary
* spouse
* base
* riders

It may remain in lower-priority diagnostics if the current codebase still uses it there, but it should not be promoted to the top warning area.

### 5\) Top premium display

The live app currently shows one total quoted premium for the selected billing mode.

Do not add an `Excellent` health quote range, best-class range, or standard-vs-best stacked premium display unless the user explicitly asks for that feature. Older notes about a premium range were from a prior direction and should not be treated as current requirements.

### 6\) Waiver behavior affecting child rider

The user requested this explicit update:

* If a **waiver is added on the primary**, go ahead and **calculate that on the child rider also**.

When reviewing logic, verify whether this is already implemented everywhere premium totals are assembled.

### 7\) Coverage under 15 warning

Coverage under 15 can happen in historic / grandfathered situations.

The app should warn when a line has coverage under 15, because current rules generally do not allow reducing below 15. However, this warning should **not block the quote**, because older in-force policies may still legitimately have lower coverage.

This warning is important, but it is not the same as a missing required input or unavailable rate.

### 8\) Insurance age calculation

Insurance age for these products is intended to use **nearest birthday age at the anniversary date**.

The current implementation may look unusual because it came from workbook-style logic and nearest-birthday age can be messy around half-year boundaries. Do not replace this with a simple attained-age or last-birthday calculation unless the user explicitly asks and verifies the business rule.

If changing this logic, test dates around:

* exactly six months before a birthday
* just before and just after the half-birthday point
* leap-year birthdays
* anniversary dates close to the insured's birthday

### 9\) Sticky warning area should show quote issues

The user wanted the sticky premium box to show meaningful issues that prevent quoting, including missing required input states such as a missing primary DOB.

In other words:

* If something prevents a quote from being generated, surface it in the sticky premium/warning area.
* But do not surface low-priority informational diagnostics there.
* Coverage under 15 is a meaningful warning but should not block quote calculation.

### 10\) Combined coverage banding

Rate bands are based on combined enabled policy coverage across products and insureds. Both manual `IBR` and automated `C4-IBR` coverage are excluded and must not move any other product into a different band. Preferred eligibility remains a separate per-insured calculation.

For the manual `Other` product, the agent can use either one basic cost-per-thousand rate or four fixed manual bands: under $150,000, $150,000 through $249,999, $250,000 through $499,999, and $500,000 or more. The banded rate uses the same combined non-IBR policy coverage. Band thresholds are not editable and additional bands are not currently supported.

## Product-specific notes

### Removed / unavailable option

* `cust-10` is currently absent because specific rates for that product line were not available.
* It should not be reintroduced as a selectable product or calculation path until the user provides the needed rate information or explicitly asks for a placeholder.

### Known product issue from prior work

* `c4-30` previously produced zero quotes due to a logic issue.
* When touching product logic, verify that `c4-30` still calculates correctly and does not return zero when valid inputs exist.

### C4 subsequent rates

The C4 product family includes `C4-10`, `C4-15`, `C4-20`, and `C4-30`. These may be quoted using either:

* original anniversary rates, based on the original policy anniversary date
* subsequent anniversary rates, based on a new anniversary date and the C4 subsequent rate table

All C4 products use the same subsequent rate table.

For subsequent C4 quotes:

* The anniversary field represents the new anniversary date.
* Rate banding uses the combined eligible policy coverage across primary and spouse lines, consistent with original C4 banding. Preferred-class eligibility under $150,000 is still evaluated separately for each insured.
* The UI should keep this option compact and avoid a large redesign.
* The line should show a month/year note like `Level until Jun 2031` when the rate has a level period.
* The main quote line should only show the current subsequent premium. Future-year C4 values belong on the dedicated year-by-year projection page.
* Waiver of premium is not currently available for subsequent C4 rates. If waiver is selected and a C4 line is set to subsequent rates, block that line's quote and show a clear warning instead of silently quoting an incomplete premium.

### TermNow and Custom Advantage (C6) subsequent rates

TermNow and Custom Advantage use scheduled subsequent rates from the PLA-165 rate books. Guaranteed maximum rates are intentionally excluded from the app and must not be quoted or displayed.

* Supported initial terms are 10, 15, 20, 25, 30, and 35 years. The app labels Custom Advantage as `C6` and TermNow as `T.Now`.
* After the initial level term, subsequent rates are level in table-defined blocks before age 70 and change annually beginning at age 70.
* The normal scheduled-rate path expires at age 95.
* The 2018 Version 1 and May 2021 Version 2 scheduled subsequent tables match through age 70 and differ at ages 71 through 94. Custom Advantage initial rates match between books; TermNow initial rates differ and both versions are embedded.
* Version selection is stored per line. For an original-rate line, an original anniversary before July 1, 2015 defaults to Version 1; July 1, 2015 or later defaults to Version 2. This is only a best-guess default. Clicking the version button marks the selection as manual and prevents later automatic overrides. A subsequent-only entry has no original anniversary available, so it retains its existing/default selection until manually changed.
* On the main quote page, the compact heading button appears only when changing versions changes that line's current calculated quote. This includes affected original TermNow cases and subsequent TermNow/C6 cases at ages where the scheduled rates differ.
* On the projection page, the compact heading button appears only when at least one displayed anniversary row differs between versions. It starts with the version selected on the main quote page and recalculates only that line. Do not duplicate V1/V2 product names in the main product dropdown.
* Eligible TermNow and C6 lines can independently convert to DT100 when insurance age at the first subsequent anniversary is at least 70. The first conversion year retains coverage and premium; later coverage uses the separate PLA-165 DT100 face-amount factors and the conversion premium remains level through age 99. The year the insured turns 100 is `Expired`.
* The PLA-165 DT100 factors are shared by both product families and both rate-book versions, but they are not the same factors as the C4 DT100 table.
* Waiver is not calculated on the subsequent or DT100 path. Existing omitted-waiver/partial-total behavior applies when waiver would otherwise still be active.

### Year-by-year projection

The `Year-by-year breakdown` button on the end-of-term page opens `projection.html` in a normal full-page browser tab. Projection columns use `Primary` and `Spouse`, never client names. Rows represent each product's anniversary year and continue through the year the insured turns 100.

Projection behavior:

* The breakdown button stays hidden until at least one enabled, supported product line has a valid birth date, anniversary date, coverage amount, and calculable premium. Other incomplete or unsupported lines do not hide it once one eligible line is ready.
* On mobile, the page handles vertical scrolling while the table handles only horizontal scrolling. The anniversary-year column remains sticky for orientation, but header and total columns are not sticky on narrow screens so they cannot overlap or crowd the product columns.
* Each enabled base plan or rider gets its own coverage and selected-mode premium columns. A combined premium total appears at the right when multiple lines are present.
* A line does not show years before its entered anniversary date.
* Unsupported product families remain visible with `Rate unavailable`; unavailable values create a clearly marked partial total instead of silently contributing zero.
* Age 99 is the final in-force premium year. The year the insured turns 100 is shown as `Expired`.
* Waiver premium is removed at age 60. When a post-level-period waiver calculation is intentionally unavailable, the projection marks that line and total as partial.

For original C4 projections, coverage and premium remain level for the product's named 10-, 15-, 20-, or 30-year term and then use the shared C4 subsequent table. For a line entered with a subsequent anniversary, projection begins at that anniversary and does not recreate earlier years. Subsequent rates use their table-defined level duration through age 69 and change annually beginning at age 70.

An eligible C4 line shows `Convert to DT100` beside its plan name when insurance age at the first subsequent anniversary is at least 70. Conversion is independent for each line. The first conversion year keeps the same coverage and premium; later years keep that modal premium level and calculate decreasing coverage from the embedded C4 DT100 face-amount factors using basic annual premium before the $75 policy fee.

### Custom Exchange projection and ART conversion

Custom Exchange uses its original anniversary as policy year 1 and issue insurance age. The embedded death-benefit schedule currently supports issue ages 37 through 70.

* Policy years 1 through 10 keep the original face amount and level base premium.
* Policy year 11 begins the automatic decreasing death benefit using the issue-age column from the Custom Exchange table.
* The base premium and applicable policy fee remain level on the normal decreasing-term path.
* A supported Custom Exchange line shows `Convert to ART` beside its plan name. ART conversion begins in policy year 11.
* On ART, the original face amount remains level and premium changes annually using attained insurance age through age 99.
* ART uses the embedded scheduled annual rate table: `PP`/`P` use the Preferred group, `N` uses Select/NTU, and `T` uses TU. Existing per-insured preferred eligibility still applies.
* ART rate bands use combined eligible policy coverage. Existing table-rating factors and billing-mode factors apply. The $75 annual fee applies only when the converted line is the primary base policy.
* Waiver is not included after ART conversion; when it would otherwise still be active, the projection uses the existing omitted-waiver/partial-total indicator.

### C4 increasing benefit rider

Keep the existing manual `IBR` product available. `C4-IBR` is a separate automated option for old riders whose increases have already stopped.

For C4-IBR:

* Enter the current accumulated IBR coverage and the actual number of accepted increases, from 1 through 10.
* Do not ask for 5% or 10%. The current accumulated coverage is already known, so no coverage projection is needed.
* Do not apply a discount. If a discounted case is encountered later, use the manual IBR option until a verified rule is supplied.
* Use the insured's birth date and the C4-IBR original anniversary date to calculate issue insurance age using the existing nearest-birthday method.
* Increases 1 through 4 use the C4-20 rate at issue insurance age.
* Increases 5 through 10 use the C4-20 rate at insurance age on each corresponding anniversary. Increase number 5 occurs five years after the original anniversary date.
* Divide the current accumulated IBR coverage evenly across the accepted increases for premium calculation.
* C4-IBR inherits its band from that insured's regular C4 coverage. Do not add C4-IBR coverage to its own band or to the banding for other products.
* No policy fee or discount is applied to C4-IBR.
* Waiver is not approved for automated C4-IBR. If waiver is selected, block the C4-IBR line and direct the user to manual IBR when waiver must be included.

### Expandable quote details

Each base or rider line keeps its live modal premium compact by default. A small disclosure arrow in the result area expands calculation details for review, including cost per thousand, insurance age, effective class, banding coverage, annual coverage premium, waiver when applicable, policy fee, and annual totals before and after the fee.

### Saved quote updates

Saving uses the current client name as the only matching rule. If that name exactly matches a saved draft after trimming outside spaces, update the newest matching entry. If there is no exact match, create a new saved draft. Renaming a loaded quote therefore creates a new entry and leaves the originally loaded draft unchanged.

Saved-draft cards are intentionally compact. They show only the quote name, saved date/time, and the Load, Share, and Delete controls. The controls stay together on one non-wrapping row; quote details such as billing mode, spouse status, line counts, riders, and DOBs are not displayed in this list.

### Fill spouse line from primary

Each enabled spouse line shows a compact paste icon beside its heading only when the corresponding primary line is enabled and has a selected product, positive coverage, valid rate class, and valid anniversary date. Base plan maps to base plan and each rider maps to the rider with the same number.

The button fills spouse fields independently without field-level change tracking:

* Product copies only while the spouse product is still the default `Cust Exch`. When product copies, its original/subsequent basis and V1/V2 selection copy with it.
* Coverage copies only when the spouse value is blank, invalid, zero, or negative.
* Rate class copies only while the spouse class is still the default `N`, and only when the source class is allowed for the spouse product.
* Anniversary copies only when the spouse value is blank or invalid.
* Table rating is never copied.
* The button does not enable or create spouse riders; the matching spouse line must already be enabled.

### Saved-quote sharing

Each saved quote has a compact `Share` icon. The current sharing format supports the complete saved quote: all enabled primary and spouse base/rider lines, waiver selection, child rider and coverage, billing mode, DOBs, rating tables, manual product fields, rate basis, and V1/V2 selections. Old basic-only test links are not kept backward compatible.

* The share link contains a compact, gzip-compressed snapshot in the URL fragment. No database or server-side quote storage is used.
* The quote/client name, saved-quote ID, and saved timestamp are never included.
* Links expire after 24 hours according to the receiving browser's clock.
* Opening a link asks before loading it, removes the fragment from the address bar, leaves the quote name blank, and never saves the imported quote automatically.
* The recipient must enter a local quote name and use the normal Save quote button if they want to retain it.

### CRM text summary

The compact document icon in the sticky premium bar copies a plain-text summary of the live quote for pasting into a CRM. It never includes the client/quote name, DOBs, anniversaries, rate classes, table ratings, versions, or calculation details.

* A single line per insured is formatted like `Primary: Cust Exch - 150K` and `Spouse: Cust Exch - 80K`.
* When an insured has multiple enabled lines, the line type is included, such as `Primary Base Plan` or `Primary Rider 1`.
* An enabled child rider gets its own coverage line.
* The final line identifies the selected billing mode, such as `Total monthly premium: $85.21`.
* Export is blocked when any enabled line is incomplete, unavailable, unsupported, missing required waiver premium, or otherwise not fully quotable. A partial or zero total must not be copied.
* Successful clipboard copying briefly changes the icon to a checkmark. If direct clipboard access is unavailable, the app falls back to a manual copy prompt.
* When waiver premium is actually included on an end-of-term line, `(WP)` appears immediately after that product. The child rider is also marked `(WP)` when its waiver premium is included.
* The new-business page has the same compact CRM-copy icon. It produces one line per insured with product, term, coverage, class or class range, and that insured's separate application premium; and marks products and child riders with `(WP)` when waiver is selected. It does not include a household total, client names, ages, sex, tobacco status, or health wording.
* New-business birth dates use manual text entry rather than a browser calendar picker. They accept the same flexible formats as the end-of-term page, including `m/d/yy`, hyphen or period separators, and six- or eight-digit entries, then normalize valid dates to `MM/DD/YYYY` on blur.

## Change strategy

When asked to make a change:

1. Read the relevant UI inputs and calculation path.
2. Identify all places where the same logic is reused, especially:

   * primary vs spouse
   * base vs rider
   * allowed rate classes by product
   * per-insured coverage threshold for preferred / preferred plus
   * sticky summary vs detailed diagnostics
3. Make the smallest coherent fix.
4. Recheck totals, warnings, and display formatting.

## Testing checklist

After any meaningful quote-logic change, verify at least these cases:

1. **Primary only, valid rate case**

   * Quote appears
   * No false range display

2. **Primary with preferred / preferred plus where allowed**

   * Product dropdown allows only valid classes for that product
   * Preferred / preferred plus calculates when the insured's eligible coverage total supports it
   * Top premium shows one selected-mode total, not a best-class range

3. **Primary + spouse rider**

   * Primary and spouse coverage thresholds are evaluated separately
   * Each insured's line totals contribute to the one top premium total
   * No extra premium numbers appear
4. **Waiver + child rider**

   * If primary waiver triggers child rider waiver handling, confirm it is included correctly
   * Child rider total updates correctly with waiver on and waiver off

5. **Preferred / preferred plus under $150k**

   * Auto-adjust behavior works
   * Low-priority diagnostic may exist if desired
   * No top sticky warning for that auto-adjust
6. **Missing required inputs**

   * Blocking issues appear in the sticky summary area
7. **Removed product**

   * `cust-10` is absent from the UI and logic path
8. **Known product**

   * `c4-30` returns a valid non-zero quote when inputs are valid

9. **Insurance age**

   * Nearest-birthday age matches expected workbook / carrier examples
   * Boundary cases around half-birthdays are checked before changing the formula

## What not to do

* Do not create additional pages or split embedded CSS/JavaScript into support files unless asked.
* Do not rewrite the UI framework.
* Do not rename large sets of variables without a strong reason.
* Do not change wording or layout broadly unless the user requests it.
* Do not assume a warning should be promoted just because it exists in diagnostics.

## First-step expectation for a new agent

When first opening this project:

1. Read `agents.md`.
2. Read all of `index.html` and the full additional HTML page involved in the requested change.
3. Summarize the app's structure and quote flow in plain English.
4. Identify where these likely live in code:

   * top premium display logic
   * rate class eligibility and auto-adjust logic
   * rider total assembly
   * warning prioritization
   * product list/options
5. Only then begin edits.
