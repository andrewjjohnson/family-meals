# Data model

Domain contract for the Family Meals MVP ([FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8)). It defines what each record is responsible for, how records relate, and which invariants the shared code must enforce. Storage tables, column types, and TypeScript names are implementation details for [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) and later stories; the rules here are binding unless this document is updated.

Related: [product requirements](product-requirements.md) · [architecture](architecture.md) · [delivery plan](mvp-delivery-plan.md)

## Principles

1. **Four different things, four different records.** A *recipe definition* (`Recipe` + `RecipeVersion`), what was *actually cooked* (`CookedBatch`), what is *intended* (`WeekPlan`), and what was *actually eaten* (`DiaryEntry`) never share a record or overwrite each other.
2. **Snapshot at the boundary.** When a record depends on nutrition data that could later change, it copies the values it used. Later edits to foods or recipes never rewrite history.
3. **Unknown is not zero.** A nutrient that a source did not report is absent. Zero is a measured value. Totals report how much is known and which contributors are missing.
4. **Derive, don't double-store.** Remaining batch quantity, recent combinations, and grocery quantities are computed from source records, so a correction is applied exactly once by construction.
5. **Ingredient identity is independent of shopping.** Recipes refer to foods and quantities, never to retailer products. A future retailer mapping is a separate table keyed by food identity.
6. **Children are participants, not dieters.** Child profiles carry preferences, restrictions, servings, and feedback. They have no calorie budgets, goals, body measurements, or diaries.

## Identifiers and shared conventions

| Convention | Rule |
| --- | --- |
| Record IDs | Client-generated random UUID strings, created once and never reused or reassigned. Backup/restore keeps them unchanged ([FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29)). |
| Provider IDs | Barcodes, USDA `fdcId`s, and Open Food Facts codes are attributes, never primary keys. Barcodes are strings so leading zeros survive. |
| Child keys inside a record | Components, ingredient lines, diary items, plan slots, and grocery items have their own UUIDs so references survive reordering and edits. |
| Component identity across versions | `componentId` is stable across all versions of one recipe (the "sauce" in v1 and v3 is the same component). This lets variations and feedback refer to "the sauce" across versions. |
| Local dates | Calendar-day fields (`diaryDate`, `cookedOn`, plan dates) are `YYYY-MM-DD` strings in the user's local calendar, not UTC timestamps. Timestamps (`createdAt`, `updatedAt`) are ISO-8601 UTC for ordering and audit only. Changing time zone never moves a record to another day ([FM-21](https://frostfiredigital-37308542.atlassian.net/browse/FM-21) AC4). |
| Archive vs delete | User-facing entities (foods, recipes, members, variations) are *archived* (`archivedAt`), hidden from pickers but kept for history. Diary entries and feedback can be *removed* (soft-deleted with `deletedAt`, excluded from all totals and derivations; purged only by restore-replace). |
| Revisions | Mutable records referenced by snapshots carry an integer `revision` that increments on each content edit, so a snapshot can say which revision it copied. |
| Precision | Store user-entered numbers as entered plus canonical values at full precision. Round only for display ([FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC5). |

## Quantities and units

A `Quantity` is `{ amount, unit }` where `amount > 0` (zero is rejected; corrections use explicit adjustment records).

| Dimension | Canonical unit | Accepted input units | Conversion rule |
| --- | --- | --- | --- |
| Mass | `g` | g, kg, oz, lb | Fixed factors. |
| Volume | `ml` | ml, l, tsp, tbsp, cup, fl oz | Fixed US-customary factors; the chosen definitions are documented in the shared unit module. |
| Count | `serving` (or a named serving such as "tortilla") | serving definitions on the food or batch | Only via an explicit `ServingDefinition` that states its mass or volume. |

Rules:

- Mass ↔ volume conversion requires a food-specific serving definition or density supplied by the source or the user. The app never assumes 1 ml = 1 g.
- A quantity that cannot be converted to the food's nutrient basis is rejected with an actionable message (for example "Add the weight of one serving to use servings").
- Raw and cooked forms are different foods (or different yields). Raw weight is never treated as cooked weight.

## Nutrients

### NutrientProfile

A `NutrientProfile` is a map from nutrient key to a canonical amount, plus a `basis`:

```text
NutrientProfile {
  basis:   { amount: 100, unit: "g" | "ml" }  // or { amount: 1, unit: <serving name> } with a ServingDefinition
  values:  { [nutrientKey]: number }          // absent key = unknown; 0 = measured zero
  derived: nutrientKey[]                      // values computed by a documented rule (e.g. kJ -> kcal)
}
```

### Supported nutrient keys (MVP)

| Key | Canonical unit | Notes |
| --- | --- | --- |
| `energy_kcal` | kcal | kJ input converted at 1 kcal = 4.184 kJ and marked `derived`. When both are supplied, the kcal value is used. |
| `protein_g`, `carbohydrate_g`, `fat_g`, `fiber_g` | g | Carbohydrate is total carbohydrate as reported. |
| `sodium_mg` | mg | Open Food Facts reports sodium (in grams) alongside salt; convert g → mg. When only salt is present, sodium is derived at salt ÷ 2.5 and marked `derived`. |
| `calcium_mg`, `iron_mg`, `potassium_mg` | mg | |
| `vitamin_a_ug_rae` | µg RAE | IU values are **not** converted (the factor depends on the source compound); they stay unknown. |
| `vitamin_c_mg` | mg | |
| `vitamin_d_ug` | µg | IU converted at 1 µg = 40 IU and marked `derived`. |
| `vitamin_b12_ug` | µg | |

The list matches [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC1. Adding a nutrient is a shared-code change with a schema migration only if storage needs it; profiles are stored as JSON maps so most additions need none.

### Aggregation and coverage

Summing contributors produces, per nutrient:

| Status | Meaning | Display |
| --- | --- | --- |
| `complete` | Every contributor reported the nutrient. | `420 kcal` |
| `partial` | Some contributors are missing it. | `≥ 0.2 µg` with "1 of 3 items missing" and a list of missing contributors |
| `unavailable` | No contributor reported it. | `—` with "not reported by sources" |

Scaling a profile by a quantity keeps absent keys absent. A partial total is never displayed as a plain number without its marker.

## Entities

Each section lists responsibility, key fields (not exhaustive), relationships, and invariants.

### FoodRecord

**Responsibility:** one food the user can choose as an ingredient or log directly: a manual food, an Open Food Facts product, a USDA catalog item, or a user's correction of an imported item.

| Field | Notes |
| --- | --- |
| `id`, `revision`, `archivedAt` | |
| `origin` | `manual` · `openfoodfacts` · `usda_catalog` · `user_corrected` |
| `name`, `brand?`, `barcode?` | Barcode stored as provider-normalized digit string. |
| `preparationState` | `raw` · `cooked` · `drained` · `prepared` · `as_sold` · `unspecified` |
| `nutrients` | `NutrientProfile` |
| `servingDefinitions[]` | `{ id, label, quantity, gramsOrMl? }`, for example "1 tortilla = 45 g". |
| `source` | `{ provider, providerRecordId, datasetVersion?, retrievedAt, license, attributionText }`; null for manual foods. |
| `correctsFoodId?` | Set on `user_corrected` records; points at the imported record it overrides. |

Relationships: referenced (by snapshot) from `IngredientLine`, `DiaryItem`, and `CookedBatch` ingredients; referenced by ID from grocery item keys and preferences/restrictions.

Invariants:

- USDA catalog records are read-only and shipped with the app; edits create a `user_corrected` copy.
- Open Food Facts data is stored as retrieved. A user correction is a separate `user_corrected` record, so provider data stays distinguishable from household data ([FM-13](https://frostfiredigital-37308542.atlassian.net/browse/FM-13) AC3, AC5).
- At most one active non-correction record per normalized barcode. Scanning a known barcode offers the existing food rather than creating a duplicate ([FM-12](https://frostfiredigital-37308542.atlassian.net/browse/FM-12) AC5).
- Generic catalog values are never silently used to fill gaps in a branded product ([FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC4).
- Only label-sourced provider values are imported. Open Food Facts values marked `source: "estimate"` are left unknown ([architecture](architecture.md#open-food-facts)).

### FoodSnapshot (value object)

A copy of `{ foodId, foodRevision, name, brand, origin, preparationState, nutrients, servingDefinitions, source summary }` embedded wherever nutrition is calculated. It is the only nutrition input for historical records.

### Recipe

**Responsibility:** the long-lived identity of a cookbook entry: title, tags, notes, archive state, and which version is current. The same record type also holds *saved meals*, which are intentionally saved combinations without instructions ([FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20) AC4).

| Field | Notes |
| --- | --- |
| `id`, `kind` | `recipe` or `saved_meal`. A saved meal becomes a recipe once instructions are added. |
| `title`, `tags[]`, `notes`, `archivedAt` | Editing these does **not** create a version (they do not affect nutrition or cooking). |
| `currentVersionId` | Version used for new plans and batches. |
| `duplicatedFromRecipeId?` | Set when a recipe is copied for an experiment ([FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15) AC3). |

### RecipeVersion

**Responsibility:** an immutable definition of how to make the recipe: components, ingredient lines, instructions, intended yield, and times.

| Field | Notes |
| --- | --- |
| `id`, `recipeId`, `versionNumber`, `createdAt` | `versionNumber` starts at 1 and increments per recipe. |
| `components[]` | At least one `Component`. |
| `instructions`, `intendedServings`, `prepMinutes?`, `cookMinutes?` | |
| `basedOnVersionId?` | Previous version, for history display. |

Invariants:

- A version is **frozen once referenced** by a batch, plan, variation, diary item, or feedback. Changing ingredients, quantities, components, instructions, or intended yield of a frozen version creates a new version and moves `currentVersionId`.
- An unreferenced version may be edited in place. This avoids a new version for every keystroke while the user is still writing the recipe.
- Estimated recipe totals are computed from the version's ingredient `FoodSnapshot`s. If a referenced food has a newer revision, the UI says so and offers "update ingredient data", which creates a new version ([FM-12](https://frostfiredigital-37308542.atlassian.net/browse/FM-12) AC4).

### Component

**Responsibility:** a separately portionable part of a recipe version (chicken, rice, vegetables, sauce). A simple mixed dish has exactly one component, and the UI hides the concept.

| Field | Notes |
| --- | --- |
| `componentId` | Stable across versions of the same recipe. |
| `name`, `sortOrder` | |
| `ingredients[]` | `IngredientLine { id, foodSnapshot, quantity, canonicalAmount, note? }` |
| `expectedYield?` | `Quantity` in grams or servings, used for planning before a real yield is measured. |

Invariants:

- Nutrients of a portion are computed **per component** from that component's own yield. The whole meal's weight ratio is never applied to a plate whose component proportions differ ([FM-16](https://frostfiredigital-37308542.atlassian.net/browse/FM-16) AC4).

### CookedBatch

**Responsibility:** what was actually cooked on a date, with actual ingredients, actual measured yields, and frozen nutrient totals. It is the source for portion logging and leftover tracking.

| Field | Notes |
| --- | --- |
| `id`, `cookedOn`, `createdAt` | |
| `recipeVersionId?` | Null for a one-time ingredient list ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC1). |
| `prepEventId?` | The planned cook event this batch fulfills, if any ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC2). |
| `components[]` | `BatchComponent { componentId?, name, ingredients[] (actual lines with FoodSnapshots and substitution notes), yield, totals }` |
| `yield` | `{ mode: "mass", grams }` or `{ mode: "servings", count }`, positive. |
| `totals` | Aggregated `NutrientProfile` plus coverage, frozen at creation. |
| `storageNote?`, `finishedAt?` | User-entered notes; the app makes no food-safety claims. |
| `adjustments[]` | `BatchAdjustment { id, componentId?, kind, quantity, memberId?, date, note }` where `kind` is `household_served`, `discarded`, or `yield_correction`. |

Derived (not stored):

```text
remaining(component) = yield
                      − Σ active DiaryItem quantities drawn from it
                      − Σ household_served and discarded adjustments
                      ± Σ yield_correction adjustments
reserved(component)  = Σ quantities of planned (not eaten, not skipped) meals allocated to it
unallocated          = remaining − reserved
```

Invariants:

- Totals and ingredient snapshots never change after creation. Editing the recipe or a source food has no effect on an existing batch ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC3).
- Weight-based portioning requires a mass yield. Without a scale, the user defines equal servings instead ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC5).
- Mixed-dish portioning assumes the batch is evenly mixed, and the UI says so. Dishes that are not evenly mixed use components.
- Consumption beyond remaining shows a warning and asks for confirmation or a yield correction. It never blocks a truthful diary entry and never invents servings ([FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) AC5).
- Reservations never reduce `remaining` and never create diary entries.

### DiaryEntry

**Responsibility:** what one adult actually ate on a local date and meal. It is the only source for nutrition totals and goal comparisons.

| Field | Notes |
| --- | --- |
| `id`, `memberId` | Must reference an **adult** member. A default adult ("Me") is created silently on first launch so logging needs no setup. |
| `diaryDate`, `mealSlot` | Local date; slot is breakfast, lunch, dinner, or snack. |
| `items[]` | `DiaryItem { id, kind, foodSnapshot?, batchRef?, quantity, nutrients }` |
| `origin?` | `{ plannedMealId?, recipeVersionId?, variationId?, variationRevision?, savedMealId? }` for traceability. |
| `clientRequestId` | Idempotency key. A second save with the same key is ignored ([FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) AC5). |
| `createdAt`, `updatedAt`, `deletedAt?` | |

`DiaryItem.kind`:

- `food`: a `FoodSnapshot` × quantity.
- `batch_portion`: `{ batchId, componentId? }` × quantity. `nutrients` is copied from the batch totals scaled by `quantity / yield`.

Invariants:

- `nutrients` on each item is frozen when the item is saved, at full precision ([FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) AC3).
- **Corrections edit the entry in place.** Changing a quantity recomputes that item's nutrients from the same snapshot (food or batch), not from current data. Moving date or slot changes only `diaryDate`/`mealSlot`. Removing sets `deletedAt`. Batch remaining quantity is derived, so each correction affects it exactly once.
- Several items in one entry form an *unnamed combination*. Recents are derived from entries and are not a separate library.

### Recents (derived)

Recent foods and combinations are queries over active diary entries, grouped by a signature (sorted food or batch IDs plus quantities). The only stored state is `RecentDismissal { signature, dismissedAt }`. Dismissing a recent never touches diary history. Saving a recent creates a `Recipe` of kind `saved_meal` ([FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20)).

### HouseholdMember

**Responsibility:** a local person profile used for planning, servings, preferences, restrictions, and feedback.

| Field | Notes |
| --- | --- |
| `id`, `displayName`, `role` (`adult`, `child`), `archivedAt` | No birth date, account, or identifying details required ([FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23) AC1). |
| `preferences[]` | `{ id, subject (foodId, componentId, or free-text ingredient term), stance: like/dislike, note? }` plus optional texture, flavor, and presentation notes. |
| `restrictions[]` | `{ id, term or foodId, kind: allergy/dietary/avoid, note? }`, kept separate from preferences. |

`AdultGoal { id, memberId, effectiveFrom, energyKcal?, proteinG?, carbohydrateG?, fatG?, nutrientGoals? }` is a separate record ([FM-22](https://frostfiredigital-37308542.atlassian.net/browse/FM-22)).

Invariants:

- `AdultGoal` and `DiaryEntry` may only reference adults. The schema has no place for a child calorie budget, weight goal, or body measurement.
- Restriction matching reports `flagged`, `clear`, or `unknown`. Missing allergen or ingredient data produces `unknown`, never `clear` ([FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23) AC4).
- Archiving a member keeps their feedback and plan history.

### ServingVariation

**Responsibility:** a reusable, named way to serve a recipe to someone, such as "Rowan's separated plate, no sauce". It changes plates, not the shared preparation.

| Field | Notes |
| --- | --- |
| `id`, `revision`, `recipeId`, `basedOnVersionId`, `memberId?`, `name`, `archivedAt` | |
| `componentPortions[]` | `{ componentId, quantity?, omitted?, servedSeparately? }` |
| `substitutions[]` | `{ componentId, ingredientLineId, replacement: FoodSnapshot, quantity }`, which affects that plate's ingredients and nutrients. |
| `presentationNotes` | Cutting, separating, or tortilla on the side. Never affects nutrition. |

Invariants:

- A variation never edits the recipe version. Its own nutrient estimate and restriction flags are computed from its components and substitutions, not inherited from the base recipe ([FM-24](https://frostfiredigital-37308542.atlassian.net/browse/FM-24) AC3-4).
- Any edit increments `revision`. Plans, diary items, and feedback store `{ variationId, variationRevision }` plus a compact `servedSnapshot` (component quantities, substitutions, and notes actually used), so later edits do not change what was recorded as served.
- If the base recipe gets a new version, the variation keeps working through stable `componentId`s. A variation that refers to a component that no longer exists is shown as "needs review".

### RecipeFeedback

**Responsibility:** a quick, neutral note about how a specific served preparation went for one person.

| Field | Notes |
| --- | --- |
| `id`, `memberId`, `servedOn` | |
| `recipeId`, `recipeVersionId`, `variationId?`, `variationRevision?`, `servedSnapshot?` | Exactly what was served. |
| `batchId?`, `plannedMealId?` | Which cook and which planned meal, when known. |
| `response` | `enjoyed` · `mixed` · `declined_today` · `not_tried` |
| `note?`, `deletedAt?` | |

Invariants:

- Feedback never edits preferences. A `declined_today` response does not create a dislike ([FM-25](https://frostfiredigital-37308542.atlassian.net/browse/FM-25) AC2).
- Feedback records no amounts eaten and no scores. Library filters show the feedback records behind each result.

### WeekPlan

**Responsibility:** the intended cooking and eating for one week. It contains three kinds of child records:

- `PrepEvent { id, date, recipeVersionId, intendedYield (per component), linkedBatchId?, status: planned/cooked/cancelled }`: a planned cook. **This is the only plan record that creates grocery demand.**
- `PlannedMeal { id, date, slot, recipeVersionId, participants[], status: planned/eaten/skipped }`: who is expected to eat what. Each participant is `{ memberId, variationId?, variationRevision?, servedSnapshot, plannedQuantities per component, source }`.
- `source` is `{ type: "prep_event", prepEventId }` (served from a planned or linked cook) or `{ type: "batch", batchId }` (leftovers from an existing batch). It is never a standalone ingredient list; to plan a one-off meal, create a PrepEvent for it.

Fields: `id`, `weekStart` (local date; week start day is a user setting, default Monday).

Invariants:

- Plans never create, edit, or delete diary entries. Planned nutrition is labeled "planned" ([FM-26](https://frostfiredigital-37308542.atlassian.net/browse/FM-26) AC5).
- A planned participant quantity drawn from a PrepEvent or batch is a **reservation** against it. Reservations show as `reserved` and do not change `remaining`.
- Marking a planned meal **eaten** opens the portion confirmation flow, prefilled with the reserved quantities. On confirmation, adult participants get a `DiaryEntry` (with `origin.plannedMealId` and `clientRequestId = plannedMealId + memberId`); child participants get a `household_served` batch adjustment. The reservation is then released because the meal is no longer `planned`.
- Marking a planned meal **skipped** releases its reservations and creates nothing ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC4).
- Over-allocation (reserved > planned or actual yield) produces a warning, not an automatic change ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC3).
- A planned meal may draw leftovers from a PrepEvent or batch in an earlier week plan. Grocery demand always belongs to the week that contains the PrepEvent.

### GroceryList

**Responsibility:** an editable shopping list generated from a week's PrepEvents.

| Field | Notes |
| --- | --- |
| `id`, `weekPlanId`, `generatedAt` | One active list per week plan. |
| `items[]` | `GroceryItem { id, key, displayName, quantity, unit, contributions[], status: needed/have/purchased, userAdjustment?, isManual, changeNote? }` |

Generation rules:

- Expand each **non-cancelled PrepEvent** into ingredient lines (version lines, scaled to intended yield, with substitutions from planned variations). Planned meals with a `batch` source and leftover allocations contribute nothing. **Each cook is counted exactly once** ([FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) AC2).
- Merge key = `foodId` + `preparationState` + unit dimension. Quantities merge only when the key matches and units convert. Ambiguous mass/volume amounts and different branded foods stay as separate lines ([FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) AC3).
- Rebuilding matches items by `key`. A check (`have` or `purchased`) is kept if the required quantity is unchanged or lower. If the quantity increased, the item returns to `needed` and `changeNote` records the old and new amounts. Manual items and user adjustments are kept.
- No prices, retailer SKUs, or product links are stored. A future `RetailerProductLink { foodId, retailer, productRef }` table would join on `foodId`.

## How changes propagate

| Change | Effect |
| --- | --- |
| Edit a food | `revision` increments. Existing recipe versions, batches, and diary items keep their snapshots. New diary logs use the new data. Recipes using the food show "newer ingredient data available". |
| Archive a food | Hidden from search. All references remain valid. |
| Edit recipe metadata (title, tags, notes) | Updated in place; no new version. |
| Edit recipe content | New `RecipeVersion` if the current one is referenced, otherwise an in-place edit. Batches, diary items, feedback, and variations keep pointing at the version they used. Planned PrepEvents keep their pinned version and show "newer version available" with a switch action, which updates groceries on rebuild. |
| Archive a recipe | Hidden from the library and planner pickers. Batches, feedback, and diary history are untouched ([FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15) AC3). |
| Correct a diary entry | Edited in place from its own snapshot. Day totals and batch `remaining` update through derivation, exactly once. |
| Remove a diary entry | `deletedAt` set. It drops out of totals, recents, and batch consumption. |
| Log a batch portion | `DiaryItem(batch_portion)` with frozen nutrients. `remaining` decreases. |
| Child eats from a batch | `household_served` adjustment (optional `memberId`). No diary, no nutrition target. |
| Discard or re-weigh a batch | `discarded` or `yield_correction` adjustment. Earlier diary snapshots are not recomputed; the correction is visible in the batch history ([FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) AC4). |
| Reserve leftovers in a plan | Increases `reserved`, not `remaining`. No diary, no groceries. |
| Mark planned meal eaten or skipped | Eaten: confirmation creates diary entries/adjustments and releases the reservation. Skipped: releases the reservation only. |
| Edit a variation | `revision` increments. Earlier plans, diary items, and feedback keep their `servedSnapshot`. |
| Archive a household member | Hidden from new plans. Feedback and plan history stay readable. |
| Restore a backup | Replaces all local data atomically with the backup's records and IDs ([FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29) AC3). |

## Worked examples

The numbers below are illustrative, not sourced nutrient data. They were checked with a throwaway script that applies the aggregation rules above (absent key = unknown). [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) and [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) should turn them into automated fixtures.

### 1. A 1,200 g evenly mixed batch with a 300 g portion

Batch `B-chili` from recipe "Turkey chili" v1, single component, `yield = { mass, 1200 g }`. Frozen batch totals: 1,680 kcal, 132 g protein, 150 g carbohydrate, 60 g fat, 36 g fiber, 2,400 mg sodium, vitamin D `partial` 0.8 µg (canned beans did not report vitamin D).

The user logs 300 g for lunch. The fraction is 300 / 1,200 = 0.25:

| Nutrient | Portion | Status |
| --- | --- | --- |
| Energy | 420 kcal | complete |
| Protein / carbs / fat | 33 g / 37.5 g / 15 g | complete |
| Fiber / sodium | 9 g / 600 mg | complete |
| Vitamin D | ≥ 0.2 µg | partial: canned beans missing |

`DiaryItem { kind: batch_portion, batchId: B-chili, quantity: 300 g, nutrients: <above> }`. Batch remaining: 1,200 − 300 = 900 g. The UI shows "Assumes the batch is evenly mixed."

### 2. An adult bowl and a child's separated plate

Recipe "Chicken rice bowls" v1 has four components. Batch `B-bowls` records each component's actual cooked yield:

| Component | Ingredient totals (illustrative) | Measured cooked yield |
| --- | --- | --- |
| Chicken | 1,000 g raw breast: 1,200 kcal, 225 g P, 0 g C, 26 g F | 750 g |
| Rice | 400 g dry rice: 1,440 kcal, 28 g P, 316 g C, 2.4 g F | 1,100 g |
| Roasted vegetables | 600 g broccoli + 15 g oil: 336.6 kcal, 16.8 g P, 39.6 g C, 17.4 g F | 480 g |
| Sauce | 180 kcal, 3 g P, 36 g C, 0 g F | 120 g |

**Adult bowl** (diary): chicken 150 g, rice 200 g, vegetables 120 g, sauce 30 g. Each component is scaled by its own yield (150/750, 200/1,100, 120/480, 30/120): **631.0 kcal, 55.0 g protein, 76.4 g carbohydrate, 10.0 g fat**.

**Child plate**: the serving variation "Rowan: separated plate, no sauce" sets chicken 60 g, rice 80 g, vegetables 40 g, and sauce omitted. It is recorded as a `household_served` adjustment, not a diary entry. The variation's estimate (228.8 kcal, 21.4 g P, 26.3 g C, 3.7 g F) can be viewed on the variation, but it is never compared with any target.

Why components matter: applying the whole meal's ratio (3,156.6 kcal across 2,450 g) would give the adult 500 g bowl 644.2 kcal / 55.7 g protein and the child's 180 g plate 20.0 g protein instead of 21.4 g. The plates have different proportions, so the whole-meal ratio is wrong.

Remaining after dinner: chicken 540 g, rice 820 g, vegetables 320 g, sauce 90 g.

### 3. Editing a recipe after an earlier batch was logged

1. Monday: `B-chili` is cooked from "Turkey chili" **v1**, and 300 g is logged (420 kcal, example 1).
2. Tuesday: the user swaps to leaner turkey and halves the oil. v1 is referenced by `B-chili`, so saving creates **v2** and `currentVersionId → v2`.
3. Monday's diary item still shows 420 kcal (its own frozen nutrients). `B-chili` still points to v1 with its frozen totals.
4. Wednesday: the user logs another 300 g from `B-chili`. This is still 420 kcal because the food in the fridge was cooked from v1. Remaining: 600 g.
5. The next cook creates `B-chili-2` from v2 with new totals.
6. Separately, the user corrects the turkey `FoodRecord` (revision 1 → 2). v1, v2, both batches, and both diary items are unchanged. The recipe shows "newer ingredient data available"; accepting creates v3.

### 4. A food with missing micronutrients

An Open Food Facts tortilla reports, per 100 g: 310 kcal, 9 g P, 50 g C, 7.5 g F, 6 g fiber, 840 mg sodium, 120 mg calcium, 3 mg iron, and **vitamin C 0 mg** (stated on the label). It reports nothing for potassium or vitamins A, D, and B12. Those keys are absent, not zero.

Lunch: 90 g tortilla + 100 g cooked chicken from the USDA catalog (all MVP nutrients present):

| Nutrient | Day total | Status |
| --- | --- | --- |
| Energy, protein, carbs, fat | 444 kcal, 39.1 g, 45 g, 10.35 g | complete |
| Fiber, sodium, calcium, iron | 5.4 g, 830 mg, 123 mg, 3.7 mg | complete |
| Vitamin C | 0 mg | complete (both measured zero) |
| Potassium, vitamin A, D, B12 | ≥ 256 mg, ≥ 9 µg, ≥ 0.1 µg, ≥ 0.34 µg | partial: tortilla missing |

If the user then logs a café muffin entered manually with only "420 kcal", energy stays complete (864 kcal), but protein, carbohydrate, and fat become partial (≥ 39.1 g, ≥ 45 g, ≥ 10.35 g) with the muffin listed as missing. Macros can be unknown too.

### 5. Planned leftovers becoming actual consumption

Week plan starting Sunday 2026-11-08 (this household set its week to start on Sunday):

- `PrepEvent P1`: Sunday, "Chicken rice bowls" v1, intended chicken 750 g cooked.
- `PlannedMeal` Sunday dinner from P1: adult bowl (150 g chicken) + Rowan's separated plate (60 g).
- `PlannedMeal` Monday, Tuesday, and Wednesday lunch from P1: adult bowl, 150 g chicken each.

**Groceries:** only P1 expands, giving 1,000 g raw chicken, 400 g rice, 600 g broccoli, 15 g oil, and the sauce ingredients. The three lunches add nothing.

Sunday: the user cooks, records `B-bowls` (yield 750 g chicken), and links it to P1. The chicken component ledger:

| Step | Consumed (diary) | Household served | Remaining | Reserved | Unallocated |
| --- | --- | --- | --- | --- | --- |
| Sunday dinner marked eaten | 150 | 60 | 540 | 450 | 90 |
| Monday lunch eaten (150 g confirmed) | 300 | 60 | 390 | 300 | 90 |
| Tuesday lunch skipped | 300 | 60 | 390 | 150 | 240 |
| Wednesday lunch eaten, actually 180 g | 480 | 60 | 210 | 0 | 210 |

There are exactly three adult diary entries (Sunday, Monday, Wednesday), each created by an explicit confirmation with `clientRequestId = plannedMealId + memberId`, so a double tap cannot create a duplicate. Tuesday created nothing. Rebuilding the grocery list at any point still counts P1 once, and checks made before shopping are kept.

### 6. Feedback tied to the preparation actually served

At Sunday dinner, Rowan was served variation `V-sep` revision 1 (separated plate, no sauce) from `B-bowls`. Jules was served the base bowl with sauce.

```text
RecipeFeedback { memberId: Rowan, servedOn: 2026-11-08, recipeVersionId: bowls-v1,
                 variationId: V-sep, variationRevision: 1, servedSnapshot: {chicken 60, rice 80, veg 40, sauce omitted},
                 batchId: B-bowls, plannedMealId: PM-sun-dinner, response: enjoyed, note: "ate the carrots, left broccoli" }
RecipeFeedback { memberId: Jules, ..., variationId: null, response: declined_today }
```

- Jules's `declined_today` does not add a dislike for any ingredient.
- Later the parent edits `V-sep` to add a tortilla on the side (revision 2). Rowan's feedback still shows revision 1's snapshot.
- Filtering the library by "Rowan enjoyed" returns "Chicken rice bowls" with the evidence "Enjoyed: separated plate, no sauce (rev 1), Sun 8 Nov".
