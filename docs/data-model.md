# Data model

The domain contract for the Family Meals MVP ([FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8)). It defines each record's responsibility, how records relate, and the invariants the shared code must enforce. Storage tables, column types, and TypeScript names are implementation details for [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) and later stories. The rules in this document are binding unless the document is updated.

Related: [product requirements](product-requirements.md) · [architecture](architecture.md) · [delivery plan](mvp-delivery-plan.md)

## Principles

1. **Four different things, four different records.** A *recipe definition* (`Recipe` + `RecipeVersion`), what was *actually cooked* (`CookedBatch`), what is *intended* (`PrepEvent` and `PlannedMeal`, viewed as a `WeekPlan`), and what was *actually eaten* (`DiaryEntry`) never share a record or overwrite each other.
2. **Snapshot at the boundary.** When a record depends on nutrition data that could later change, it copies the values it used. Later edits to foods or recipes never rewrite history. The only exception is an explicit, user-confirmed correction of a batch measurement ([CookedBatch](#cookedbatch)).
3. **Unknown is not zero.** A nutrient a source did not report is absent. Zero is a measured value. Every stored total also records which contributors were missing, so "partial" survives being copied.
4. **Derive, don't double-store.** Remaining batch quantity, reservations, recent combinations, "cooked" status, and grocery quantities are computed from source records. A correction is therefore applied exactly once by construction. Each relationship is stored on one side only.
5. **Ingredient identity is independent of shopping.** Recipes refer to foods and quantities, never to retailer products. A future retailer mapping is a separate table keyed by food identity.
6. **Children are participants, not dieters.** Child profiles carry preferences, restrictions, servings, and feedback. They have no calorie budgets, goals, body measurements, or diaries.

## Identifiers and shared conventions

| Convention | Rule |
| --- | --- |
| Record IDs | User-created records use client-generated random UUIDs, created once and never reused or reassigned. Backup/restore keeps them unchanged ([FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29)). |
| Catalog IDs | USDA catalog foods use **deterministic** UUIDs (UUIDv5 of `usda:<fdcId>` in a fixed project namespace). The same food therefore has the same ID on every device and in every catalog version. |
| Provider IDs | Barcodes, USDA `fdcId`s, and Open Food Facts codes are attributes, never primary keys. |
| Child keys inside a record | Components, ingredient lines, diary items, participants, and grocery items each get their own UUID, so references survive reordering and edits. |
| Component identity across versions | `componentId` stays the same across all versions of one recipe: the "sauce" in v1 and in v3 is the same component. This lets variations and feedback refer to "the sauce" across versions. |
| Local dates | Calendar-day fields (`diaryDate`, `cookedOn`, plan dates) are `YYYY-MM-DD` strings in the user's local calendar, not UTC timestamps. Timestamps (`createdAt`, `updatedAt`) are ISO-8601 UTC and are used for ordering and audit only. Changing time zone never moves a record to another day ([FM-21](https://frostfiredigital-37308542.atlassian.net/browse/FM-21) AC4). |
| Archive vs delete | User-facing entities (foods, recipes, batches, members, variations) are *archived* via `archivedAt`. Archived records are hidden from pickers but kept for history. Diary entries and feedback can be *removed*: they are soft-deleted with `deletedAt` and excluded from all totals and derivations. Backups exclude soft-deleted rows, so a restore-replace purges them. |
| Idempotency | Every record created by a single user action carries a `clientRequestId` generated when the form or confirmation sheet opens. Uniqueness is enforced only among non-deleted records. A retried save is ignored; a fresh attempt after an undo gets a new key. |
| Revisions | Mutable records referenced by snapshots carry an integer `revision` that increments on each content edit. A snapshot records which revision it copied. |
| Precision | Store user-entered numbers as entered plus canonical values, at full precision. Round only for display ([FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC5). |

## Quantities and units

A `Quantity` is `{ amount, unit }` with `amount > 0`. Zero and negative quantities are rejected. Corrections that set a value use absolute fields, not signed quantities (see [CookedBatch](#cookedbatch)).

| Dimension | Canonical unit | Accepted input units | Conversion rule |
| --- | --- | --- | --- |
| Mass | `g` | g, kg, oz, lb | Fixed factors. |
| Volume | `ml` | ml, l, tsp, tbsp, cup, fl oz | Fixed US-customary factors, documented in the shared unit module. |
| Count | a named serving ("serving", "tortilla", "bowl") | — | Only via a `ServingDefinition`. |

`ServingDefinition { id, label, measure: Quantity, grams?, ml? }`: at least one of `grams` and `ml` is required. Examples: `{ label: "tortilla", measure: 1 tortilla, grams: 45 }` and USDA's `{ label: "tbsp", measure: 1 tbsp, ml: 14.79, grams: 13.5 }`.

Rules:

- **Mass ↔ volume** conversion is allowed only through a serving definition that states both `grams` and `ml` (that is, a density for that food). The app never assumes 1 ml = 1 g. A recipe line of "2 tbsp olive oil" against a per-100 g food is valid only if the food has a definition that bridges volume and mass. Otherwise the user is asked to add one.
- **Count ↔ mass/volume** conversion is allowed only through the food's (or batch's) serving definition for that label.
- A quantity that cannot be converted to the needed basis is rejected with an actionable message, for example "Add the weight of one tbsp to use tablespoons".
- Raw and cooked forms are different foods or different yields. Raw weight is never treated as cooked weight.

## Nutrients

### NutrientProfile (per basis)

Used by foods, which describe nutrients per unit amount:

```text
NutrientProfile {
  basis:   { amount: 100, unit: "g" | "ml" } | { servingDefinitionId }
  values:  { [nutrientKey]: number }   // absent key = unknown; 0 = measured zero; null is invalid
  derived: nutrientKey[]               // computed by a documented rule (e.g. kJ -> kcal)
  definitions?: { carbohydrate?: "includes_fiber" | "excludes_fiber", vitamin_a?: "rae" | "re" }
}
```

### NutrientTotals (absolute amounts)

Used by everything that holds an amount of food: recipe and component estimates, batch component totals, diary items, plate and variation estimates, planned nutrition.

```text
NutrientTotals {
  values:     { [nutrientKey]: number }            // known amount; null is invalid
  incomplete: { [nutrientKey]: ContributorRef[] }  // non-empty => value is a lower bound ("partial")
}
ContributorRef { kind: "food" | "batch_component", id, label }  // label copied for display
```

`null` is banned in both types because JavaScript treats `null` as `0` in arithmetic. Unknown is represented only by an absent key.

### Supported nutrient keys (MVP)

| Key | Canonical unit | Notes |
| --- | --- | --- |
| `energy_kcal` | kcal | kJ input is converted at 1 kcal = 4.184 kJ and marked `derived`. When both kcal and kJ are supplied, the kcal value is used. USDA mapping: see [architecture](architecture.md#usda-fooddata-central). |
| `protein_g`, `fat_g`, `fiber_g` | g | |
| `carbohydrate_g` | g | Stored as the source reports it. Definitions differ: USDA "by difference" and US labels include fiber, while EU labels exclude it. The food records the definition in `definitions.carbohydrate` where known, and day totals that mix definitions show a note. No conversion is attempted in the MVP. |
| `sodium_mg` | mg | Open Food Facts reports sodium in grams alongside salt; convert g → mg. If only salt is present, sodium (mg) = salt (g) ÷ 2.5 × 1000, marked `derived`. For example, 1.0 g salt gives 400 mg. |
| `calcium_mg`, `iron_mg`, `potassium_mg` | mg | |
| `vitamin_a_ug` | µg | USDA reports µg RAE, while EU labels report µg retinol equivalents (RE). These are not interchangeable, so `definitions.vitamin_a` records which one applies and mixed totals show a note. IU values are **not** converted (the factor depends on the compound) and stay unknown. |
| `vitamin_c_mg` | mg | |
| `vitamin_d_ug` | µg | IU is converted at 1 µg = 40 IU and marked `derived`. |
| `vitamin_b12_ug` | µg | |

The list matches [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC1. Profiles and totals are stored as JSON maps, so adding a nutrient is a shared-code change. It needs no schema migration.

### Aggregation and coverage

Summing contributors into a `NutrientTotals` works per nutrient:

- A contributor **with** the key adds its value. If that contributor's own `incomplete[key]` is non-empty, those refs are carried into the result.
- A contributor **without** the key adds nothing, and the contributor itself is added to `incomplete[key]`.

Status for display:

| Status | Meaning | Display |
| --- | --- | --- |
| `none` | No contributors at all (e.g. a day with no active entries) | "Not logged". Excluded from weekly averages ([FM-22](https://frostfiredigital-37308542.atlassian.net/browse/FM-22) AC2). |
| `complete` | Key present and `incomplete[key]` empty | `420 kcal` |
| `partial` | Key present and `incomplete[key]` non-empty | `≥ 0.2 µg`, followed by "missing: canned beans (Turkey chili)" |
| `unavailable` | Contributors exist but none reported the key | `—`, followed by "not reported by sources" |

Scaling a `NutrientTotals` multiplies `values` and keeps `incomplete` unchanged. A partial total is never displayed as a plain number. A day containing only zero-energy foods (for example water) is `complete` at 0 kcal. That is a confirmed zero, distinct from `none`.

## Entities

Each section below gives the record's responsibility, key fields (not exhaustive), relationships, and invariants.

### FoodRecord

**Responsibility:** one food the user can pick as an ingredient or log directly. It is one of: a manual food, an Open Food Facts product, a USDA catalog item, or a user's correction of an imported item.

| Field | Notes |
| --- | --- |
| `id`, `revision`, `archivedAt` | |
| `origin` | `manual` · `openfoodfacts` · `usda_catalog` · `user_corrected` |
| `name`, `brand?` | |
| `barcode?` | `{ scanned, normalized }`. `scanned` keeps the digits exactly as read or typed, including leading zeros. `normalized` follows the provider's rules ([architecture](architecture.md#open-food-facts)) and is used for lookup and duplicate detection. |
| `preparationState` | `raw` · `cooked` · `drained` · `prepared` · `as_sold` · `unspecified` |
| `nutrients` | `NutrientProfile` |
| `servingDefinitions[]` | `ServingDefinition` (see [Quantities and units](#quantities-and-units)) |
| `allergens` | `{ declared: boolean, contains: string[], traces: string[] }`, using provider allergen tags where available. `declared: false` means the source did not state allergens. |
| `ingredientsText?` | As supplied by the provider or the user. |
| `source` | `{ provider, providerRecordId, datasetVersion?, retrievedAt, license, attributionText }`. Null for manual foods. |
| `correctsFoodId?` | Set on `user_corrected` records: the imported record this one overrides. |

Relationships: referenced by snapshot from `IngredientLine`, `DiaryItem`, `CookedBatch` ingredients, and variation extra foods. Referenced by ID from grocery item keys, preferences, and restrictions.

Invariants:

- USDA catalog records are read-only. An app upgrade updates them **in place** by their deterministic ID and bumps `revision`. A food removed from the catalog is archived, never deleted. Backups do not include catalog rows. Restore checks catalog references against the installed catalog and reports any that are missing.
- Open Food Facts data is stored as retrieved. A user correction is a separate `user_corrected` record, so provider data stays distinguishable from household data ([FM-13](https://frostfiredigital-37308542.atlassian.net/browse/FM-13) AC3, AC5).
- **Corrections supersede.** Each food may have at most one active `user_corrected` record pointing at it. Wherever a food is resolved, the active correction is used in place of the original: search, barcode scan, the recipe "newer data" check, and the grocery merge key. Editing a correction bumps the correction's `revision`.
- At most one active non-correction record exists per `barcode.normalized`. Scanning a known barcode offers the existing food (or its correction) and does not create a duplicate ([FM-12](https://frostfiredigital-37308542.atlassian.net/browse/FM-12) AC5).
- Generic catalog values are never used to silently fill gaps in a branded product ([FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) AC4).
- Only label-sourced provider values are imported. Open Food Facts values marked `source: "estimate"` stay unknown ([architecture](architecture.md#open-food-facts)).

### FoodSnapshot (value object)

A copy of `{ foodId, foodRevision, name, brand, origin, preparationState, nutrients, servingDefinitions, allergens, source summary }`, embedded wherever nutrition or restrictions are evaluated. It is the only input for historical records.

### Recipe

**Responsibility:** the long-lived identity of a cookbook entry: title, tags, notes, archive state, and which version is current. The same record type also holds *saved meals*: intentionally saved food combinations without instructions ([FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20) AC4).

| Field | Notes |
| --- | --- |
| `id`, `kind` | `recipe` or `saved_meal`. A saved meal becomes a recipe once instructions are added. |
| `title`, `tags[]`, `notes`, `archivedAt` | Editing these does **not** create a version, because they affect neither nutrition nor cooking. |
| `currentVersionId` | The version used for new plans and batches. |
| `duplicatedFromRecipeId?` | Set when a recipe is copied for an experiment ([FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15) AC3). |

### RecipeVersion

**Responsibility:** an immutable definition of how to make the recipe: components, ingredient lines, instructions, intended servings, and times.

| Field | Notes |
| --- | --- |
| `id`, `recipeId`, `versionNumber`, `createdAt` | `versionNumber` starts at 1 and increments per recipe. |
| `components[]` | At least one `Component`. |
| `instructions`, `intendedServings` (required, > 0), `prepMinutes?`, `cookMinutes?` | |
| `basedOnVersionId?` | The previous version, for history display. |

Invariants:

- A version is **frozen once referenced** by a batch, prep event, planned meal, variation, diary entry, or feedback. Changing a frozen version's ingredients, quantities, components, instructions, or intended servings creates a new version and moves `currentVersionId`.
- An unreferenced version may be edited in place. This avoids creating a version for every keystroke while the recipe is still being written.
- Estimated recipe totals are computed from the version's ingredient `FoodSnapshot`s. If a referenced food (after resolving corrections) has a newer revision, the UI says so and offers "update ingredient data", which creates a new version ([FM-12](https://frostfiredigital-37308542.atlassian.net/browse/FM-12) AC4).

### Component

**Responsibility:** a separately portionable part of a recipe version, such as chicken, rice, vegetables, or sauce. A simple mixed dish has exactly one component, and the UI hides the concept.

| Field | Notes |
| --- | --- |
| `componentId` | Stable across versions of the same recipe. |
| `name`, `sortOrder` | |
| `ingredients[]` | `IngredientLine { id, foodSnapshot, quantity, canonicalAmount, servingDefinitionId?, note? }` |
| `expectedYield?` | `{ grams?, servings? }` at `intendedServings`. Used for planning before a real yield is measured. If absent, planning uses servings of the whole recipe. |

Invariants:

- Portion nutrients are computed **per component**, from that component's own yield. The whole meal's weight ratio is never applied to a plate whose component proportions differ ([FM-16](https://frostfiredigital-37308542.atlassian.net/browse/FM-16) AC4).

### CookedBatch

**Responsibility:** what was actually cooked on a date. It holds the actual ingredients, the actual measured yields, and frozen nutrient totals. It is the source for portion logging and leftover tracking.

| Field | Notes |
| --- | --- |
| `id`, `cookedOn`, `createdAt`, `archivedAt` | |
| `recipeVersionId?` | Null for a one-time ingredient list ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC1). |
| `prepEventId?` | The planned cook this batch fulfils. This is the **only** stored link between plan and batch ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC2). |
| `components[]` | At least one `BatchComponent` (fields below). |
| `storageNote?` | User-entered. The app makes no food-safety claims. |
| `adjustments[]` | `BatchAdjustment` (fields below). |

`BatchComponent` fields:

- `batchComponentId`: required and unique within the batch. It copies the recipe's `componentId` when one exists; otherwise it is a new UUID.
- `name`
- `ingredients[]`: actual lines with `FoodSnapshot`s and substitution notes.
- `yield`: `{ grams?, servings? }`, positive, with at least one present. `servings` means equal servings.
- `totals`: `NutrientTotals`, frozen at creation.

`BatchAdjustment` fields: `id`, `clientRequestId`, `batchComponentId`, `kind`, `date`, `createdAt`, `note`, plus:

| `kind` | Extra fields | Meaning |
| --- | --- | --- |
| `household_served` | `quantity`, `memberId?` | Plated for someone without a diary (for example a child). |
| `discarded` | `quantity` | Thrown away. |
| `yield_remeasured` | `newYield` (absolute) | The original yield was wrong. New portions use `totals ÷ newYield`. |
| `remaining_reweighed` | `measuredRemaining` (absolute) | What is left was weighed. The difference from the computed remaining is recorded as unexplained loss. The per-unit nutrient density is unchanged. |

Batch-level yield and totals are not stored. Where needed (single-component dishes), they are the component's values.

Derived values (not stored) are computed per component by replaying its events in `createdAt` order:

```text
start: remaining = yield, currentYield = yield
consumption (active DiaryItem), household_served, discarded:  remaining -= quantity
yield_remeasured:     remaining += (newYield − currentYield); currentYield = newYield
remaining_reweighed:  remaining = measuredRemaining
reserved     = Σ quantities of planned meals (status planned, date ≥ today) drawing on this component
unallocated  = remaining − reserved
```

`remaining` and `unallocated` may be negative. They are displayed as "over by X" and never clamped to zero ([FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) AC5).

Invariants:

- A batch is **editable until first referenced** by a diary item, adjustment, or planned meal, in the same way as a recipe version. After that, its ingredients and totals are frozen. A batch entered with wrong ingredients is fixed by archiving it and recording a corrected batch. The user can then move each portion to the corrected batch by editing it like any other diary edit.
- Editing the master recipe or a source food never changes an existing batch ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC3).
- Every portion and adjustment names a `batchComponentId`. Portioning a multi-component batch without choosing a component is rejected.
- Weight-based portioning requires `yield.grams`. Without a scale, the user enters `yield.servings`, and portions are logged in servings ([FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) AC5). A component may have both.
- Mixed-dish portioning assumes the batch is evenly mixed, and the UI says so. Dishes that are not evenly mixed use components.
- Consumption beyond remaining shows a warning and asks the user to confirm or record a yield correction. It never blocks a truthful diary entry and never invents servings ([FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) AC5).
- Reservations never reduce `remaining` and never create diary entries.
- After a `yield_remeasured`, the app asks "Also update the N earlier portions?". If the user confirms, those diary items are recomputed with the new yield and the change is listed in the batch history. Otherwise they keep their stored `yieldUsed`. This user-confirmed correction of a measurement is the only case where a diary snapshot is rewritten.

### DiaryEntry

**Responsibility:** what one adult actually ate on a local date and meal. It is the only source for nutrition totals and goal comparisons.

| Field | Notes |
| --- | --- |
| `id`, `memberId` | Must reference an **adult** member. A default adult ("Me") is created silently on first launch, so logging needs no setup. |
| `diaryDate`, `mealSlot` | Local date. Slot is breakfast, lunch, dinner, or snack. |
| `items[]` | `DiaryItem` (see the kinds below). |
| `origin?` | `{ plannedMealId?, recipeVersionId?, variationId?, variationRevision?, savedMealId? }`, kept for traceability. |
| `clientRequestId` | See [idempotency](#identifiers-and-shared-conventions) ([FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) AC5). |
| `createdAt`, `updatedAt`, `deletedAt?` | |

`DiaryItem { id, kind, quantity, nutrients: NutrientTotals, ... }`, where `kind` is one of:

- `food`: `foodSnapshot` × `quantity`.
- `batch_portion`: `{ batchId, batchComponentId, yieldUsed }` × `quantity`. `nutrients` = component `totals` × `quantity / yieldUsed`. `yieldUsed` is the component's `currentYield` at the time of logging, in the same unit as `quantity`.

Invariants:

- `nutrients` on each item is frozen when the item is saved, at full precision ([FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) AC3). The diary items themselves are the record of what the adult ate.
- **Corrections edit the entry in place.**
  - Changing a quantity recomputes that item from its own `foodSnapshot`, or from the batch totals with its stored `yieldUsed`. Current food data is never used.
  - Moving the date or slot changes only `diaryDate`/`mealSlot`.
  - Removing an entry sets `deletedAt`.
  - Because batch remaining quantity is derived, each correction affects it exactly once.
- Several items in one entry form an *unnamed combination*. Recents are derived from entries, not stored as a separate library.

### Recents (derived)

Recent foods and combinations are queries over active diary entries, grouped by a signature:

- `food` items: resolved food ID + quantity.
- `batch_portion` items: recipe ID + component ID + quantity. A one-time batch uses its batch ID.

Repeating a recent creates a draft. A batch portion in the draft draws from the **newest** non-archived batch of that recipe that still has remaining quantity. If there is none, the user is asked to pick a batch or a food.

The only stored state is `RecentDismissal { signature, dismissedAt }`. Dismissing a recent never touches diary history.

Saving a recent creates a `Recipe` of kind `saved_meal`. In the MVP this is offered only for combinations made entirely of `food` items ([FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20)).

### HouseholdMember

**Responsibility:** a local person profile used for planning, servings, preferences, restrictions, and feedback.

| Field | Notes |
| --- | --- |
| `id`, `displayName`, `archivedAt` | No birth date, account, or identifying details are required ([FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23) AC1). |
| `role` | `adult` or `child`. **Fixed at creation.** A member who needs the other role gets a new profile, so the adults-only rules can never be broken retroactively. |
| `preferences[]` | `{ id, subject, stance: like/dislike, note? }`. `subject` is a foodId, a componentId, or a free-text ingredient term. Optional texture, flavor, and presentation notes. |
| `restrictions[]` | `{ id, term or allergen tag or foodId, kind: allergy/dietary/avoid, note? }`, kept separate from preferences. |

`AdultGoal { id, memberId, effectiveFrom, energyKcal?, proteinG?, carbohydrateG?, fatG?, nutrientGoals? }` is a separate record ([FM-22](https://frostfiredigital-37308542.atlassian.net/browse/FM-22)).

Invariants:

- `AdultGoal` and `DiaryEntry` may reference adults only. The schema has no field for a child calorie budget, weight goal, or body measurement.
- Restriction matching evaluates each food's `allergens` and `ingredientsText`. It reports:
  - `flagged` when a restriction matches the food's contains or traces lists, or its ingredient text;
  - `clear` only when the food has `allergens.declared: true` and nothing matches;
  - `unknown` in every other case, for example when nothing is declared or ingredient data is missing.

  Names are never used to decide `clear` ([FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23) AC4).
- Archiving a member keeps their feedback and plan history.

### ServingVariation

**Responsibility:** a reusable, named way to serve a recipe to someone, such as "Rowan's separated plate, no sauce". It changes plates, not the shared preparation.

| Field | Notes |
| --- | --- |
| `id`, `revision`, `recipeId`, `basedOnVersionId`, `memberId?`, `name`, `archivedAt` | |
| `componentPortions[]` | `{ componentId, quantity?, omitted?, servedSeparately? }` |
| `extraFoods[]` | `{ food: FoodSnapshot, quantity, replacesComponentId? }`: a food served on this plate that is not drawn from the batch. With `replacesComponentId`, the food is served *instead of* that component, for example plain yogurt in place of the sauce. Without it, the food is an addition, for example a tortilla on the side. |
| `presentationNotes` | Cut into strips, kept separate, sauce on the side. Never affects nutrition. |

Invariants:

- A variation never edits the recipe version. Its nutrient estimate and restriction flags are computed from its own component portions and extra foods, not inherited from the base recipe ([FM-24](https://frostfiredigital-37308542.atlassian.net/browse/FM-24) AC3–4).
- A replaced or omitted component is **not drawn from the batch** for that plate. Each extra food is a separate item:
  - For an adult it is logged as a `food` diary item.
  - For a child nothing is logged; the extra food is part of the served snapshot.
  - For a planned meal it adds grocery demand (see [GroceryList](#grocerylist)).

  Ingredient-level changes inside a shared component are not supported in the MVP, because a shared component is cooked once for everyone.
- Every edit increments `revision`. Planned meals and feedback store `{ variationId, variationRevision }` plus a compact `servedSnapshot`: the component quantities, extra foods, and notes actually used. Diary entries record the variation in `origin`, and their items hold what was eaten. Later edits therefore never change what was recorded as served.
- If the base recipe gets a new version, the variation keeps working through stable `componentId`s. A variation that refers to a component that no longer exists is shown as "needs review".

### RecipeFeedback

**Responsibility:** a quick, neutral note about how a specific served preparation went for one person.

| Field | Notes |
| --- | --- |
| `id`, `clientRequestId`, `memberId`, `servedOn` | |
| `recipeId`, `recipeVersionId`, `variationId?`, `variationRevision?`, `servedSnapshot?` | Exactly what was served. |
| `batchId?`, `plannedMealId?` | Which cook and which planned meal, when known. |
| `response` | `enjoyed` · `mixed` · `declined_today` · `not_tried` |
| `note?`, `deletedAt?` | |

Invariants:

- Feedback never edits preferences. A `declined_today` response does not create a dislike ([FM-25](https://frostfiredigital-37308542.atlassian.net/browse/FM-25) AC2).
- Feedback records no amounts eaten and no scores. Library filters show the feedback records behind each result.

### WeekPlan (PrepEvent and PlannedMeal)

**Responsibility:** the intended cooking and eating for a week. The stored records are dated **top-level** `PrepEvent`s and `PlannedMeal`s. A `WeekPlan` is a view of them: a date-range query from the user's week-start day (a setting, default Monday). Its stable identifier is that range's start date. With no container record, changing the week start, moving a meal to next week, or restoring a backup can never orphan or split anything.

`PrepEvent` (a planned cook) is **the only plan record that creates grocery demand from recipe ingredients**:

| Field | Notes |
| --- | --- |
| `id`, `date`, `recipeVersionId` | |
| `intendedServings` | Required, > 0. Scale factor = `intendedServings ÷ version.intendedServings`. Expected component yields = `expectedYield × scale` ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC1, [FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) AC1). |
| `status` | `planned` or `cancelled`. "Cooked" is derived: it means a `CookedBatch` has this `prepEventId`. |

`PlannedMeal` (who is expected to eat what):

| Field | Notes |
| --- | --- |
| `id`, `date`, `slot`, `status` (`planned`, `eaten`, `skipped`) | |
| `source` | One of:<br>• `{ type: "prep_event", prepEventId }`: served from that cook, or from its batch once cooked.<br>• `{ type: "batch", batchId }`: leftovers from an existing batch, including one-time batches.<br>• `{ type: "direct", recipeVersionId }`: a no-cook saved meal or recipe assembled at the meal. |
| `participants[]` | `{ id, memberId, variationId?, variationRevision?, servedSnapshot, portions[] }`. Each portion is `{ componentId, quantity }`, expressed in grams or servings. |

The recipe version of a meal is **derived from its source**: the prep event's version, the batch's version, or the direct version. Switching a prep event to a newer version therefore moves every meal served from it.

Invariants:

- Plans never create, edit, or delete diary entries by themselves. Planned nutrition is labeled "planned" ([FM-26](https://frostfiredigital-37308542.atlassian.net/browse/FM-26) AC5).
- A participant portion drawn from a prep event or batch is a **reservation** on the matching batch component, or on the expected yield before cooking. It counts toward `reserved` while the meal is `planned` and dated today or later.
  - A past-dated meal that is still `planned` shows as "unconfirmed" and stops holding its reservation. The user can mark it eaten or skipped.
- **Marking eaten** opens the confirmation flow, prefilled with the reserved portions. Confirming is one atomic write:
  1. The write checks that `status == planned`. If not, it does nothing and shows the existing result, which prevents double submission.
  2. Adult participants each get a `DiaryEntry` with `origin.plannedMealId`.
  3. Child participants each get a `household_served` adjustment.
  4. Extra foods become `food` items for adults, and are noted only for children.
  5. The status is set to `eaten`, which releases the reservation.

  Every created record carries the confirmation sheet's `clientRequestId`.
- **Undo eaten** is one atomic write: it soft-deletes the diary entries and deletes the adjustments created by that confirmation, then sets `status = planned`. Marking the meal eaten again uses a new `clientRequestId`.
- If a meal's source has no batch yet (a prep event not yet recorded as cooked), the eaten flow first asks the user to record what was cooked, prefilled from expected yields (servings are acceptable without a scale). It then continues.
  - A `direct` source logs the version's ingredient lines as `food` items, scaled to the participant's portion.
  - A source whose prep event was cancelled shows "needs a source" until the user picks a batch, a prep event, or a direct version.
- **Marking skipped** releases the reservations and creates nothing ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC4).
- **Over-allocation warning:**
  - Before cooking, the warning shows when reservations exceed the expected yield.
  - After cooking, it shows when any component's `unallocated` is below zero.
  - In both cases the plan itself is never changed automatically ([FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) AC3).
- If plan portions are in servings and the linked batch has only grams (or the reverse), the meal shows "needs review" until the user converts them or adds a serving count to the batch.
- A meal may draw leftovers from a prep event or batch dated in an earlier week. Grocery demand always belongs to the date range that contains the prep event (or the direct meal).

### GroceryList

**Responsibility:** an editable shopping list generated from the planned cooking in a date range.

| Field | Notes |
| --- | --- |
| `id`, `fromDate`, `toDate`, `generatedAt` | The range is copied from the week view when the list is created, so later changes to the week-start setting do not affect it. |
| `items[]` | `GroceryItem` (fields below). |

`GroceryItem` fields:

- `id`, `key`, `displayName`, `unit`
- `requiredQuantity`: from the plan.
- `userAdjustment`: a signed delta, default 0.
- `checkedQuantity`: the amount marked have/purchased, default 0.
- `contributions[]`
- `isManual`
- `previousRequiredQuantity?`: set on rebuild when the amount changed.
- `noLongerNeeded`: boolean.

Generation rules:

- **Demand sources:**
  - Each non-cancelled `PrepEvent` in the range contributes its version's ingredient lines × scale factor.
  - Each `direct` planned meal contributes its lines scaled to its participants' portions.
  - Each planned extra food contributes its quantity.
- Leftover reservations and meals with a `batch` source contribute nothing. **Each cook is counted exactly once** ([FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) AC2).
- **Merge key** = resolved `foodId`, plus `servingDefinitionId` for count units, plus the unit dimension. `preparationState` is not part of the key because `foodId` already fixes it.
  - Quantities merge only when the keys match and the units convert.
  - Ambiguous mass/volume amounts and different branded foods stay as separate lines ([FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) AC3).
- **Display** = still needed: `requiredQuantity + userAdjustment − checkedQuantity`. An item is shown as done when this is ≤ 0.
- **Rebuild** matches items by `key` and recomputes `requiredQuantity`:
  - If the amount changed, it stores `previousRequiredQuantity` and shows "was 1,000 g, now 1,500 g; 500 g still needed".
  - `checkedQuantity`, `userAdjustment`, and manual items are kept as they are.
  - If an item's key disappears: an unchecked item is dropped; a checked item stays, marked `noLongerNeeded`.
- No prices, retailer SKUs, or product links are stored. A future `RetailerProductLink { foodId, retailer, productRef }` table would join on `foodId`.

## How changes propagate

| Change | Effect |
| --- | --- |
| Edit a manual food | Its `revision` increments. Existing recipe versions, batches, and diary items keep their snapshots, and new logs use the new data. Recipes that use the food show "newer ingredient data available". |
| Correct an imported food | A `user_corrected` record is created (or the existing one gets a new revision) and supersedes the import everywhere the food is resolved. Snapshots are unchanged, and recipes show "newer ingredient data available". |
| Catalog upgrade | Catalog rows are updated in place by deterministic ID with a bumped `revision`, or archived if removed. User records are untouched. Recipes show "newer ingredient data available". |
| Archive a food | Hidden from search. All references remain valid. |
| Edit recipe metadata (title, tags, notes) | Updated in place. No new version. |
| Edit recipe content | A new `RecipeVersion` is created if the current one is referenced; otherwise the edit is made in place. Batches, diary items, feedback, and variations keep the version they used. A planned PrepEvent keeps its pinned version and shows "newer version available". Switching it moves every meal served from it, and groceries update on the next rebuild. |
| Archive a recipe | Hidden from the library and the planner's pickers. Batches, feedback, and diary history are untouched ([FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15) AC3). |
| Edit a batch before first use | Edited in place. Totals are recomputed. |
| Mistaken batch after use | Archive it and record a corrected batch. Move portions by editing the diary entries. |
| Correct a diary entry | Edited in place, from its own snapshot and stored `yieldUsed`. Day totals and batch `remaining` update through derivation, exactly once. |
| Remove a diary entry | `deletedAt` is set. The entry drops out of totals, recents, and batch consumption. |
| Log a batch portion | A `DiaryItem(batch_portion)` is created with frozen nutrients and `yieldUsed`. `remaining` decreases. |
| Child eats from a batch | A `household_served` adjustment is recorded (optional `memberId`). No diary entry and no nutrition target. |
| Discard part of a batch | A `discarded` adjustment is recorded. |
| Batch weighed differently than recorded | `yield_remeasured` changes future portions, and earlier portions only if the user confirms. `remaining_reweighed` sets remaining and keeps the per-gram values. Both appear in the batch history ([FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) AC4). |
| Reserve leftovers in a plan | `reserved` increases; `remaining` does not change. No diary entry and no groceries. |
| Mark a planned meal eaten, undo, or skipped | Eaten: one atomic confirmation creates the diary entries and adjustments and releases the reservation. Undo: removes exactly those records. Skipped: releases the reservation only. |
| Edit a variation | Its `revision` increments. Earlier planned meals and feedback keep their `servedSnapshot`. |
| Archive a household member | Hidden from new plans. Feedback and plan history stay readable. |
| Restore a backup | Replaces all user data and settings atomically, keeping the backup's records and IDs. Catalog references are checked against the installed catalog ([FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29) AC3). |

## Worked examples

The numbers below are illustrative, not sourced nutrient data. A throwaway script that applies the aggregation rules above (absent key = unknown) reproduced them. An independent review recomputed examples 1, 2, 4, and 5 and got the same figures. [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) and [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) should turn them into automated fixtures.

### 1. A 1,200 g evenly mixed batch with a 300 g portion

Batch `B-chili` is cooked from "Turkey chili" v1. It has a single component with `yield = { grams: 1200 }`. Its frozen component totals are:

- 1,680 kcal, 132 g protein, 150 g carbohydrate, 60 g fat, 36 g fiber, 2,400 mg sodium
- vitamin D 0.8 µg, with `incomplete.vitamin_d_ug = [canned beans]` because the beans did not report vitamin D.

The user logs 300 g for lunch. The fraction is 300 / 1,200 = 0.25:

| Nutrient | Portion | Status |
| --- | --- | --- |
| Energy | 420 kcal | complete |
| Protein / carbs / fat | 33 g / 37.5 g / 15 g | complete |
| Fiber / sodium | 9 g / 600 mg | complete |
| Vitamin D | ≥ 0.2 µg | partial: canned beans missing |

The diary item is `DiaryItem { kind: batch_portion, batchId: B-chili, batchComponentId, yieldUsed: 1200 g, quantity: 300 g, nutrients }`, where `nutrients` carries `incomplete.vitamin_d_ug`.

The batch's remaining quantity is 1,200 − 300 = 900 g. The UI shows "Assumes the batch is evenly mixed."

Later that day the user logs milk with 2.9 µg of vitamin D. The day shows **≥ 3.1 µg, partial**, with "missing: canned beans (Turkey chili)". Because the item stored its `incomplete` refs, the gap survives being copied into the diary.

**1b. Corrections.** Suppose the batch actually weighed 1,000 g, not 1,200 g. The 300 g portion above was logged first. Two correction paths are available:

- **`yield_remeasured { newYield: 1000 g }`**
  - Remaining becomes 1,000 − 300 = 700 g.
  - Later portions use 1.68 kcal/g, so eating the remaining 700 g logs 1,176 kcal.
  - If the user declines "update earlier portions", the logged total is 420 + 1,176 = 1,596 kcal. The batch history shows the first portion was calculated at the old yield.
  - If the user accepts, the first portion becomes 504 kcal and the total is exactly 1,680 kcal.
- **`remaining_reweighed { measuredRemaining: 600 g }`**, entered instead when the yield was right but some was lost
  - Remaining becomes 600 g, with 300 g recorded as unexplained loss.
  - Density stays 1.4 kcal/g, so a next 300 g portion is 420 kcal.

### 2. An adult bowl and a child's separated plate

Recipe "Chicken rice bowls" v1 has four components. Batch `B-bowls` records each component's actual cooked yield:

| Component | Ingredient totals (illustrative) | Measured cooked yield |
| --- | --- | --- |
| Chicken | 1,000 g raw breast: 1,200 kcal, 225 g P, 0 g C, 26 g F | 750 g |
| Rice | 400 g dry rice: 1,440 kcal, 28 g P, 316 g C, 2.4 g F | 1,100 g |
| Roasted vegetables | 600 g broccoli + 15 g oil: 336.6 kcal, 16.8 g P, 39.6 g C, 17.4 g F | 480 g |
| Sauce | 180 kcal, 3 g P, 36 g C, 0 g F | 120 g |

**Adult bowl** (logged to the diary): chicken 150 g, rice 200 g, vegetables 120 g, sauce 30 g. Each component is scaled by its own yield (150/750, 200/1,100, 120/480, 30/120). Result: **631.0 kcal, 55.0 g protein, 76.4 g carbohydrate, 10.0 g fat**.

**Child plate:** the serving variation "Rowan: separated plate, no sauce" sets chicken 60 g, rice 80 g, vegetables 40 g, and omits the sauce.

- The plate is recorded as `household_served` adjustments on the chicken, rice, and vegetable components. Nothing is drawn from the sauce, and no diary entry is created.
- The served plate's estimate, from this batch's measured yields, is 228.8 kcal, 21.4 g P, 26.3 g C, 3.7 g F. It can be viewed but is never compared with any target. Before cooking, the variation's estimate uses expected yields instead.

**Why components matter:** applying the whole meal's ratio (3,156.6 kcal across 2,450 g) would give the adult's 500 g bowl 644.2 kcal and 55.7 g protein, and the child's 180 g plate 20.0 g protein instead of 21.4 g. The plates have different proportions, so the whole-meal ratio gives the wrong answer.

Remaining after dinner: chicken 540 g, rice 820 g, vegetables 320 g, sauce 90 g.

### 3. Editing a recipe after an earlier batch was logged

1. Monday: `B-chili` is cooked from "Turkey chili" **v1**, and 300 g is logged (420 kcal; see example 1).
2. Tuesday: the user swaps to leaner turkey and halves the oil. v1 is referenced by `B-chili`, so saving creates **v2**, and `currentVersionId` moves to v2.
3. Monday's diary item still shows 420 kcal from its own frozen nutrients. `B-chili` still points to v1 and keeps its frozen totals.
4. Wednesday: the user logs another 300 g from `B-chili`. It is still 420 kcal, because the chili in the fridge was cooked from v1. Remaining is now 600 g.
5. The next cook creates `B-chili-2` from v2, with new totals.
6. Separately, the user corrects the turkey's fat value.
   - If the turkey is a manual food, its `revision` goes from 1 to 2.
   - If it is an imported food, a `user_corrected` record supersedes it.

   Either way, v1, v2, both batches, and both diary items are unchanged. The recipe shows "newer ingredient data available", and accepting creates v3.

### 4. A food with missing micronutrients

An Open Food Facts tortilla reports, per 100 g: 310 kcal, 9 g P, 50 g C, 7.5 g F, 6 g fiber, 840 mg sodium, 120 mg calcium, 3 mg iron, and **vitamin C 0 mg** (stated on the label). It reports nothing for potassium or vitamins A, D, and B12. Those keys are absent, not zero.

Lunch is 90 g of tortilla plus 100 g of cooked chicken from the USDA catalog (all MVP nutrients present):

| Nutrient | Day total | Status |
| --- | --- | --- |
| Energy, protein, carbs, fat | 444 kcal, 39.1 g, 45 g, 10.35 g | complete |
| Fiber, sodium, calcium, iron | 5.4 g, 830 mg, 123 mg, 3.7 mg | complete |
| Vitamin C | 0 mg | complete (both measured zero) |
| Potassium, vitamin A, D, B12 | ≥ 256 mg, ≥ 9 µg, ≥ 0.1 µg, ≥ 0.34 µg | partial: tortilla missing |

The user then logs a café muffin entered manually with only "420 kcal":

- Energy stays complete, at 864 kcal.
- **Every other nutrient** becomes partial, with the muffin listed as missing:
  - protein ≥ 39.1 g, carbohydrate ≥ 45 g, fat ≥ 10.35 g;
  - fiber ≥ 5.4 g, sodium ≥ 830 mg, calcium ≥ 123 mg, iron ≥ 3.7 mg;
  - vitamin C ≥ 0 mg.
- Potassium and vitamins A, D, and B12 now list both the tortilla and the muffin as missing.

Macros can be unknown too.

### 5. Planned leftovers becoming actual consumption

The household's week starts on Sunday, so the week view is 2026-11-08 to 2026-11-14:

- `PrepEvent P1`: Sunday, "Chicken rice bowls" v1, `intendedServings: 4`. With v1's 4 intended servings the scale is 1.0, and expected chicken is 750 g cooked.
- `PlannedMeal` Sunday dinner, source P1: an adult bowl (150 g chicken) and Rowan's separated plate (60 g).
- `PlannedMeal` Monday, Tuesday, and Wednesday lunch, source P1: an adult bowl with 150 g chicken each.

**Groceries:** only P1 expands, giving 1,000 g raw chicken, 400 g rice, 600 g broccoli, 15 g oil, and the sauce ingredients. The three lunches add nothing.

On Sunday the user cooks and records `B-bowls` with `prepEventId: P1`. P1 is now derived as cooked, and its yield of 750 g of chicken replaces the expected yield. The chicken component ledger:

| Step | Consumed (diary) | Household served | Remaining | Reserved | Unallocated |
| --- | --- | --- | --- | --- | --- |
| Sunday dinner marked eaten | 150 | 60 | 540 | 450 | 90 |
| Monday lunch eaten (150 g confirmed) | 300 | 60 | 390 | 300 | 90 |
| Tuesday lunch skipped | 300 | 60 | 390 | 150 | 240 |
| Wednesday lunch eaten, actually 180 g | 480 | 60 | 210 | 0 | 210 |

There are exactly three adult diary entries (Sunday, Monday, Wednesday) and one set of Rowan adjustments. Each confirmation was one atomic write guarded by `status == planned`, so a double tap on Sunday cannot add a second 60 g adjustment or a second diary entry. Tuesday created nothing.

Had the user also eaten an unplanned 150 g of chicken on Monday evening, remaining would be 240 against 300 reserved, leaving unallocated at −60. The over-allocation warning would show. Rebuilding the grocery list at any point still counts P1 once, and checks made before shopping are kept.

### 6. Feedback tied to the preparation actually served

At Sunday dinner on 2026-11-08 (example 5), Rowan was served variation `V-sep` revision 1 (separated plate, no sauce) from `B-bowls`.

```text
RecipeFeedback { memberId: Rowan, servedOn: 2026-11-08, recipeVersionId: bowls-v1,
                 variationId: V-sep, variationRevision: 1,
                 servedSnapshot: {chicken 60, rice 80, vegetables 40, sauce omitted},
                 batchId: B-bowls, plannedMealId: PM-sun-dinner, response: enjoyed,
                 note: "ate the chicken and rice, left most of the broccoli" }
```

During the week the parent edits `V-sep` to add a tortilla on the side as an extra food (revision 2). The next Sunday, 2026-11-15, Rowan is served revision 2 from a new batch `B-bowls-2`:

```text
RecipeFeedback { memberId: Rowan, servedOn: 2026-11-15, recipeVersionId: bowls-v1,
                 variationId: V-sep, variationRevision: 2, batchId: B-bowls-2,
                 response: declined_today }
```

- The first record still shows revision 1's snapshot. The later edit did not change it.
- `declined_today` does not add a dislike for any ingredient, and Rowan's preferences are unchanged.
- Filtering the library by "Rowan enjoyed" returns "Chicken rice bowls" with its evidence: "Enjoyed: separated plate, no sauce (rev 1), Sun 8 Nov · Declined today: rev 2, Sun 15 Nov".
