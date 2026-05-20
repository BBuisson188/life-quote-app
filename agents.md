# Quote Sheet Project Guide

## Project summary

This is a small insurance quote-sheet app that currently lives in a **single HTML file**. The file contains the UI, product/rate data wiring, quote logic, diagnostics, and premium display behavior.

The main goal is to keep the project **simple, stable, and easy to hand off**. Do not introduce extra files, frameworks, or major refactors unless the user explicitly asks for that.

## Core operating rules

1. **Read the entire HTML file before changing anything.** This project is small enough that understanding the full file is practical and important.
2. **Keep it single-file by default.** Do not split HTML/CSS/JS into separate files unless the user specifically requests it.
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

## Project structure assumptions

* Main file is likely `index.html`.
* The file likely contains:

  * UI form controls for insureds, riders, products, health/rate classes, and options.
  * Embedded CSS for layout and display.
  * Embedded JavaScript for quote calculations, validations, warnings, and premium summaries.
* Product/rate logic may be handled by in-file tables, objects, or arrays.

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
* The UI should keep this option compact and avoid a large redesign.
* The line should show a month/year note like `Level until Jun 2031` when the rate has a level period.
* The initial implementation should only quote the current subsequent premium. Do not project future-year premiums from the workbook unless the user explicitly asks for that later.
* Waiver of premium is not currently available for subsequent C4 rates. If waiver is selected and a C4 line is set to subsequent rates, block that line's quote and show a clear warning instead of silently quoting an incomplete premium.

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

* Do not split the app into multiple files unless asked.
* Do not rewrite the UI framework.
* Do not rename large sets of variables without a strong reason.
* Do not change wording or layout broadly unless the user requests it.
* Do not assume a warning should be promoted just because it exists in diagnostics.

## First-step expectation for a new agent

When first opening this project:

1. Read `agents.md`.
2. Read the full HTML file.
3. Summarize the app's structure and quote flow in plain English.
4. Identify where these likely live in code:

   * top premium display logic
   * rate class eligibility and auto-adjust logic
   * rider total assembly
   * warning prioritization
   * product list/options
5. Only then begin edits.
