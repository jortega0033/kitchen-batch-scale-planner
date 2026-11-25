# Kitchen Batch Scale Planner

## Product requirements

**Users and problem.** A volunteer kitchen coordinator plans several dishes for a prep shift. Recipes start at different serving counts and use different units for shared ingredients. A rushed conversion can leave the prep team with a shopping list that misses stock, shortfalls, or how many times to repeat a recipe.

**Core outcome.** A coordinator enters ingredient keys and recipe quantities, sets target portions and a maximum portion count per batch, records available stock and one optional pack size per needed ingredient, then gets a batch sheet and a reconciled shopping report. A print-ready page shows exact quantities, planned purchases, expected leftovers, and any requirement still uncovered. The coordinator can reopen and print the same revision after restarting.

**First durable slice.** Create one ingredient and one recipe, scale it to a new portion count, save it, and reopen the exact result. Use two synthetic recipes for the first integrated release to demonstrate mixed-unit consolidation, capped batches, and both covered and uncovered ingredients.

**User stories.**

- A recipe maintainer enters a decimal or fraction and sees the canonical quantity before using it in other plans.
- A shift coordinator divides a request into repeatable batches and can account for all portions and ingredients across the batches.
- A stock volunteer compares on-hand quantities against total requirements and sees whole pack purchases, leftovers, and residual shortages.
- A second coordinator reopens a saved revision, reviews the same arithmetic, and prints the handoff without losing edits from another tab.

**Scope.** Local browser app for one kitchen workspace. Ingredient keys are explicit, for example “grain mix” and “broth.” Volumes, masses, and counts are separate dimensions. Compatible units within one dimension are normalized. Each ingredient has one configured pack size per plan and volume or mass package amounts may be fractional. Counts may be fractional when used or held in stock. Count pack sizes must be whole. Portions and batch limits are whole numbers. The report uses fictional recipes and makes no dietary, allergy, nutrition, or safe handling recommendations. No automatic density conversion, recipe suggestions, pack price comparison, ingredient substitution, or inventory mutation. Live inventory data and shared accounts are outside this release.

**Usability.** Give every field a visible label with its accepted unit or syntax. Keep validation next to the field and summarize issues in a focusable region. Support the full create-to-print path by keyboard. Use meaningful text and table headers for prepared batch and shopping values, including text status for shortages. Show a reduced fraction whenever an exact result has no practical terminating decimal. Explain one conversion and the stock reconciliation behind each row. The print report includes plan revision, input assumptions, status, quantities, and a layout usable in grayscale.

## Technical design

### Runtime and boundaries

Run a Python 3 standard-library server bound to 127.0.0.1. Serve local HTML, CSS, and JavaScript to Chrome. Use Python fractions for the exact quantities, sqlite3 for durable storage, and unittest for deterministic tests. The browser sends quantity strings and renders the server's exact computed representations. No external fonts, libraries, CDN, or network calls. Preflight Python, its SQLite module, and Chrome before verification, then record their installed versions. Use a separate temporary data directory for each test suite.

Suggested modules are catalog and plan validation, exact units, batch allocation, stock reconciliation, SQLite revisions, and HTTP/browser entry points. Keep calculation functions independent of HTTP and storage. Treat output as plain text when rendering user labels. Require JSON for write routes, same-loopback origin for browser writes, and bounded request sizes.

### Data and exact arithmetic

| Record | Fields and rules |
| --- | --- |
| Ingredient | Immutable generated ID, display name, revision, and dimension `mass`, `volume`, or `count`. A dimension locks once used in a recipe. Separate ingredient IDs remain separate despite matching display names. |
| Recipe | Immutable ID, name, revision, base portion count in 1–100, and ordered ingredient lines. Each line references one ingredient ID and carries an exact quantity with compatible unit. One line per ingredient in each recipe. At least one line required. |
| Plan | ID, title, saved revision, a selected list of recipes with target portions in 1–500 and batch limit in 1–100, and inputs for available quantity and optional pack per ingredient. Each recipe may be selected once. Explicitly enter zero stock or an availability value for each required ingredient. A pack may be absent. |
| Revision | Immutable plan snapshot, algorithm version, timestamp, canonical exact inputs and computed report. Snapshot ingredient and recipe values at the save boundary. New recipe edits only affect new plans or explicit refresh of a working draft from the catalog. |

Accept nonnegative whole numbers, fixed decimals, or fractions expressed as `digits/digits`, with up to six digits per numeric component and up to three decimal places. Reject signs, exponents, mixed fractions, zero denominators, and empty values except that an empty pack field means no pack. Ingredient lines and pack amounts must be positive. Stock can be zero. Normalize mass to grams using `g` and `kg`, volume to millilitres using `mL` and `L`, and count to `each`. Use exact factors of 1,000 for kilogram and litre conversion. Where a fractional or decimal count is entered for a pack, accept only a mathematically whole quantity. Cap each normalized input at 1,000,000 and each aggregate required amount at 10,000,000 canonical units. A dimension mismatch fails validation with a field-level issue. Keep canonical rational data as reduced numerator and positive denominator strings, with exact zero as 0/1.

Limit the first release to 20 recipes per plan, 50 lines per recipe, 200 distinct selected ingredient IDs, and 100 generated batches per plan. The preview rejects an over-limit plan without truncating input or returning a partial report. Prune duplicate findings from repeated references while retaining the field paths so a user can correct each error. Stable output order uses the chosen recipe order, batch index, and ingredient ID.

### Batch and stock calculation

For each selected recipe with target portions `n`, base portions `b`, and batch cap `c`, produce consecutive full batches of `c` portions and one remainder if needed. Emit `floor(n/c)` full batches, then emit `n mod c` only if positive. Reject totals over the batch count cap before allocation. This creates the fewest batches for that recipe under the fixed cap because no batch contributes more than `c` servings. For a base ingredient amount `a`, each batch contributes `a × batch portions / b`. Sum across batches and recipes by exact ingredient ID. For every selected recipe the batch quantities for a line must sum to `a × n / b`.

For each required ingredient calculate the following exact quantities.

| Value | Formula |
| --- | --- |
| Need | Sum of scaled batch quantities |
| Before-shopping shortage | max(0, need − stock) |
| Packs to buy | ceiling(shortage ÷ pack size) if a pack is configured, otherwise 0 |
| Receipt quantity | pack size × packs to buy, or 0 |
| Covered | min(need, stock + receipt quantity) |
| Expected leftover | stock + receipt quantity − covered |
| Uncovered | need − covered |

The report shows all seven values and displays the configured pack size or a plain “No pack configured” label. A plan with an uncovered amount is a saved, printable plan marked “Supply gap remains.” It must not display “All requirements covered.” Receipts and leftovers are projections. The planner does not update stock inputs or change a recipe when a plan is computed or reopened. Report all exact values and keep any human-readable mixed-number formatting separate from the underlying data used for reconciliation.

### Interfaces and persistence

| Surface | Contract |
| --- | --- |
| Catalog editor | Create or edit ingredient and recipe inputs. Validate all references and dimensions before persisting. Retain the draft values after a validation problem. |
| Preview | Local request returns exact batch rows, shopping rows, plan status, algorithm version, and the revisions of the referenced catalog rows. Preview is read-only. Validation returns structured field path, error code, and plain-language message. Canceling or replacing a preview does not save input. The browser discards late responses for superseded drafts. |
| Save plan | Submit canonical input plus its base revision or “new” marker and the referenced catalog revisions from the preview. Within one transaction, check first for a recorded request with the same save ID, then test the current plan and catalog revisions, compute the report, write the snapshot, and increment the plan revision. A stale plan returns a conflict with the current revision. The browser retains unsaved values and offers reopening the current plan or creating a new draft from the retained values. A stale catalog input retains the draft and asks for a fresh preview. A repeated save after a lost response returns the existing revision when an identical input hash was recorded under the save ID. Reusing a save ID for different input fails. |
| Load and print | Return saved inputs and report from the same immutable revision. The print view is derived from that exact report. After restart, an unavailable historical algorithm version still allows reading a valid stored report. Do not recompute saved reports against edited recipes. A damaged or unsupported database returns a visible diagnostic and avoids overwriting stored data. |

SQLite constraints guard unique IDs and revisions. Each save writes its idempotency record with its snapshot in one transaction. Read and write transactions do not span a browser confirmation. Allow explicit reload when another tab is editing. If a save fails or the user closes the UI, no partly visible revision appears. Failure reports include a successful save's revision ID when the client retries after losing its response.

## Original verification fixtures

### Fixture A · Cedar Pockets and Brook Cups

Both recipes are synthetic planning examples.

| Recipe | Base portions | Target portions | Batch cap | Ingredient quantities for base portions |
| --- | ---: | ---: | ---: | --- |
| Cedar Pockets | 4 | 10 | 4 | 300 g Grain Mix, 150 mL Broth, 3 each Lime |
| Brook Cups | 3 | 5 | 3 | 1/4 kg Grain Mix, 0.24 L Broth, 3/2 each Lime, 2 each Crunch Garnish |

Expected servings and batch values in canonical units:

| Batch | Portions | Grain Mix, g | Broth, mL | Lime, each | Crunch Garnish, each |
| --- | ---: | ---: | ---: | ---: | ---: |
| Cedar 1 | 4 | 300 | 150 | 3 | 0 |
| Cedar 2 | 4 | 300 | 150 | 3 | 0 |
| Cedar 3 | 2 | 150 | 75 | 3/2 | 0 |
| Brook 1 | 3 | 250 | 240 | 3/2 | 2 |
| Brook 2 | 2 | 500/3 | 160 | 1 | 4/3 |
| **Total** | **15** | **3500/3** | **775** | **10** | **10/3** |

Expected shopping and residual report:

| Ingredient | Stock | Pack size | Need | Before-shopping shortage | Buy packs | Receipt quantity | Covered | Leftover | Uncovered |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Grain Mix, g | 500 | 250 | 3500/3 | 2000/3 | 3 | 750 | 3500/3 | 250/3 | 0 |
| Broth, mL | 500 | 200 | 775 | 275 | 2 | 400 | 775 | 125 | 0 |
| Lime, each | 4 | 3 | 10 | 6 | 2 | 6 | 10 | 0 | 0 |
| Crunch Garnish, each | 1 | None | 10/3 | 7/3 | 0 | 0 | 1 | 0 | 7/3 |

Fixture A's plan status is “Supply gap remains.” Add a configured 2-each Crunch Garnish pack while keeping stock 1 and the original need. The revised results are two packs, four received, 10/3 covered, 5/3 leftover, and zero uncovered. The resulting status is “All requirements covered.”

### Fixture B · Validation and revision boundaries

- Enter `1/3` kg and `0.333` kg as the same mass ingredient in separate recipes. Their canonical contributions are 1000/3 g and 333 g. Their exact difference is 1/3 g. A volume entry for that key is rejected with a dimension mismatch.
- Request 100 portions at cap 1 with a one-recipe plan to reach the 100-batch limit exactly. Add a second 1-portion recipe with cap 1. The attempt fails the plan batch limit without saving a truncated report.
- Create two previews from revision 1. Save the first with a corrected pack size to reach revision 2. Retry that save with the same save ID and input. It returns revision 2. Saving the second edit from revision 1 returns a stale revision conflict and leaves the draft in the second tab.
- Reopen revision 1 from history after editing the underlying recipe and restarting. Its displayed quantities, batch order, and print rows stay identical to the saved report.

These are calculated expectations and planned checks. The application and its browser checks remain to be built.

## Epic · Produce a checked prep and shopping plan offline

Finish a local planner that reconciles each required quantity back to its batch, reports supply gaps beside expected leftovers, and can reopen and print the exact saved revision. The planned effort is **13–20 engineering hours** for implementation and verification.

Start KB-01 with the smallest durable vertical route. Land KB-02 before building the batch allocator and shopping report, which may be worked on in parallel using the exact scaled-need interface. KB-05 integrates those lanes, then KB-06 closes the offline release gate.

### KB-01 · Create and reopen a first recipe plan

**Story KB-01.** As a coordinator, I can enter one recipe and reach a usable saved scaling preview before adding more dishes.

**Estimate.** 2–3 hours. **Dependencies.** None.

- **KB-01-A.** From a fresh local workspace, create an ingredient and a recipe with 300 g for four portions. Save a first-slice plan for 10 portions, cap 10, zero stock, and no pack. Verify the scale preview shows 750 g.
- **KB-01-B.** Reject zero base portions, an unknown ingredient ID, and a recipe without lines. Keep entered values visible with clear field issues.
- **KB-01-C.** Restart and load the saved plan. Verify the same inputs and exact result through the browser.

### KB-02 · Normalize quantities and aggregate compatible ingredients

**Story KB-02.** As a recipe maintainer, I can combine repeated ingredients without losing fractional quantities or mixing units.

**Estimate.** 2–4 hours. **Dependencies.** KB-01.

- **KB-02-A.** Test whole, decimal, and fractional candidate quantities, including zero stock, zero denominator, incompatible dimensions, and out-of-bound values.
- **KB-02-B.** Confirm the exact quantities in Fixture B and that matching ingredient IDs consolidate while two distinct IDs with the same label remain separate.
- **KB-02-C.** Validate Fixture A's total needs directly from base recipes and target portions against 3500/3 g, 775 mL, 10 each, and 10/3 each. Expose a stable exact quantity contract for KB-03 and KB-04.

### KB-03 · Allocate repeatable batches under a portion cap

**Story KB-03.** As a shift lead, I can see exactly what goes into each batch and how many portions it covers.

**Estimate.** 2–3 hours. **Dependencies.** KB-02.

- **KB-03-A.** Fixture A yields batches of 4, 4, 2 for Cedar Pockets and 3, 2 for Brook Cups. Verify every quantity in the batch table and sum each ingredient to the independent KB-02 total.
- **KB-03-B.** Cover one exact multiple, one remainder, and cap above target. Every batch is positive, within its cap, and the batch total equals the recipe target.
- **KB-03-C.** Exercise Fixture B's 100-batch edge and one-over request. Reject the one-over attempt before producing a report or writing a revision.

### KB-04 · Reconcile available stock, packs, and residual gaps

**Story KB-04.** As a stock volunteer, I can see how many whole packs to buy and identify an uncovered requirement.

**Estimate.** 3–4 hours. **Dependencies.** KB-02.

- **KB-04-A.** Match all Fixture A stock and pack arithmetic, including the exact-multiple Lime row and the no-pack Crunch Garnish row. Assert need = covered + uncovered and stock + receipt = covered + leftover per row.
- **KB-04-B.** Apply the Crunch Garnish pack variant and observe exactly two packs and zero uncovered. With 9 each stock and no pack for a 5-each need, verify zero order, 5 covered, and 4 leftover.
- **KB-04-C.** Reject an incompatible pack unit, a fractional count pack size and a zero pack size. Accept 0.2 L as a volume pack equivalent to 200 mL. Surface errors beside affected fields.

### KB-05 · Finish durable revisions and the browser handoff

**Story KB-05.** As a coordinator, I can finish the plan, keep an exact historical revision, and hand it to the prep and stock teams.

**Estimate.** 3–4 hours. **Dependencies.** KB-03 and KB-04.

- **KB-05-A.** Integrate catalog, recipes, stock, batch output and shopping output. From keyboard-only entry, a user can reach the output and understand its supply status and field errors.
- **KB-05-B.** Exercise the Fixture B save conflict, same-save retry, changed-input retry rejection, historical revision reopen, and edited-catalog case. Change a recipe between preview and save and verify a stale-catalog conflict preserves the draft. Confirm snapshot and save ID writes remain atomic.
- **KB-05-C.** Print a report with both tables, named unit columns, plan revision and shortage text. It contains the same exact rows and amounts as the saved report without layout clipping in the planned Chrome print check.

### KB-06 · Verify the complete offline workflow and release notes

**Story KB-06.** As a reviewer, I can reproduce a clean plan run and inspect the expected reconciliation.

**Estimate.** 1–2 hours. **Dependencies.** KB-05.

- **KB-06-A.** From a clean Linux workspace, document tool preflight and run all unit and integration checks against generated data. Compare batch and shopping outputs row by row with Fixture A.
- **KB-06-B.** In Chrome with external network access unavailable, enter, save, restart, reopen and print Fixture A, then repeat the variant. Record commands, actual results, and any browser limitations separately from the calculated expectations.
- **KB-06-C.** Confirm no on-hand amounts were mutated by preview, save, reload or print, and no real recipe, account or credential is required.
