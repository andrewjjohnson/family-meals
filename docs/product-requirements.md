# Product requirements

This document is the MVP product contract for Family Meals. The source of truth for scope is Jira [FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8); this document expands it and should change only together with that issue.

Related: [architecture](architecture.md) · [data model](data-model.md) · [delivery plan](mvp-delivery-plan.md)

## Purpose

Family Meals helps a parent meet their own nutrition and fitness goals while cooking meals their children are more likely to enjoy. Three things are central:

- fast food logging;
- weekly meal prep;
- a family cookbook that improves as the household records what worked.

## Intended users

| User | Needs | Not in scope for them |
| --- | --- | --- |
| **Primary: a fitness-minded parent** who cooks for the household and batch-preps their own lunches | Log their own intake quickly and accurately. Reuse cooked batches across the week. Plan family dinners. Shop once. | |
| **Children in the household** (represented by profiles, not users) | Have their likes, dislikes, restrictions, and reactions remembered. Get plates adapted from the shared meal. | No accounts, calorie budgets, weight goals, body measurements, or diaries. |
| **Other adults in the household** (optional profiles) | Appear in plans and feedback. | No separate login. Multiple adult diaries are possible in the data model, but the MVP UI is built around one diary owner. |

## Product principles

1. **Logging comes first.** A first food can be logged without setup, an account, a profile, or naming a recipe.
2. **Data is honest.** Unknown nutrients are shown as unknown, estimates are labeled as estimates, and history is never rewritten.
3. **The household owns its data.** Everything is local and works without an account. Backups go only where the user chooses.
4. **The tone is calm and neutral.** No guilt, streak pressure, moral food scores, plate-clearing rewards, ads, or upsells.
5. **Everything is free.** No premium tier is planned. Future revenue (optional affiliate grocery shopping on public recipes) must not gate user features.

## Key journeys

### J1. Main family journey (FM-8 AC1)

1. **Enter or scan ingredients.** Scan a barcode (Open Food Facts), choose from the USDA-sourced starter catalog, or create a food manually.
2. **Save a recipe** with ingredients, instructions, intended yield, and optional components (for example chicken, rice, vegetables, sauce).
3. **Plan a week for an adult and children.** Add a Sunday cook, a family dinner, and leftover lunches, with serving variations for each child.
4. **Generate groceries** from the planned cooks, then check items off while shopping.
5. **Cook a batch** and record the actual measured yield per component.
6. **Serve different plates:** an adult bowl and a child's separated plate from the same components.
7. **Log actual adult intake** by confirming the planned portion or adjusting it. Child portions reduce the batch without a diary.
8. **Record family feedback** for each child (enjoyed, mixed, declined today, or not tried) against the variation actually served.

### J2. Fast one-off logging

Open Today, search a recent or saved food (or scan one), enter a quantity, and save. An unnamed combination of several foods is one entry. Repeating a recent combination makes a draft for confirmation. Recents never fill the recipe library unless the user saves one deliberately.

### J3. Meal-prep week (adult only)

Cook one batch on Sunday, weigh the yield, and log portions Monday through Thursday. Each portion's nutrition is calculated from the batch's frozen totals, and remaining quantity is visible.

### J4. Finding meals that worked

Filter the recipe library by family feedback (for example "Rowan enjoyed") and see the evidence: which variation, and when. Untried recipes remain visible with no predicted acceptance.

### J5. Backup and restore

Export a versioned file to a location the user chooses, then restore it on the same or another device. Restore previews the contents, validates them, and replaces local data atomically after confirmation.

## MVP scope

| Area | Included | Jira |
| --- | --- | --- |
| Platforms | iOS, Android, and web from one codebase. Local storage that survives restarts. No accounts or sync. | [FM-1](https://frostfiredigital-37308542.atlassian.net/browse/FM-1), [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9), [FM-10](https://frostfiredigital-37308542.atlassian.net/browse/FM-10) |
| Food data | Open Food Facts barcode lookup with manual barcode fallback. Manual foods. Small bundled USDA-sourced catalog. Calories, macros, fiber, sodium, and calcium, iron, potassium, and vitamins A, C, D, B12 when available. Unknown ≠ zero. | [FM-2](https://frostfiredigital-37308542.atlassian.net/browse/FM-2), [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11)–[FM-14](https://frostfiredigital-37308542.atlassian.net/browse/FM-14) |
| Recipes & batches | Recipes with ingredients, instructions, yields, and versions. Optional components. Cooked batches with measured yields and frozen nutrient snapshots. Mixed-dish portions by cooked weight. Component plates. | [FM-3](https://frostfiredigital-37308542.atlassian.net/browse/FM-3), [FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15)–[FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17), [FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) |
| Diary | Fast one-off logging, unnamed combinations, batch portions, edits and moves with history preserved, recents kept separate from saved meals. Daily totals with coverage. Optional adult goals and a weekly summary. | [FM-4](https://frostfiredigital-37308542.atlassian.net/browse/FM-4), [FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18), [FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20)–[FM-22](https://frostfiredigital-37308542.atlassian.net/browse/FM-22) |
| Family | Local adult and child profiles with preferences and restrictions. Serving variations. Quick feedback tied to what was served. | [FM-5](https://frostfiredigital-37308542.atlassian.net/browse/FM-5), [FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23)–[FM-25](https://frostfiredigital-37308542.atlassian.net/browse/FM-25) |
| Planning | Seven-day plan, prep events, leftover allocation with reservations, consolidated editable grocery list. | [FM-6](https://frostfiredigital-37308542.atlassian.net/browse/FM-6), [FM-26](https://frostfiredigital-37308542.atlassian.net/browse/FM-26)–[FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) |
| Release | Backup/restore, skippable onboarding, cross-platform validation, small household pilot. | [FM-7](https://frostfiredigital-37308542.atlassian.net/browse/FM-7), [FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29)–[FM-32](https://frostfiredigital-37308542.atlassian.net/browse/FM-32) |

### Explicit MVP boundaries

- No child calorie budgets, weight-loss goals, body measurements, or child diaries.
- No deficiency diagnosis, clinical advice, or food-safety duration claims for leftovers.
- No exact branded nutrients filled in from generic foods. No modeling of nutrient retention during cooking.
- No accounts, sync, ads, subscriptions, analytics uploads, or automatic cloud backup.

## Deferred scope

These are recorded so they are not built speculatively (FM-8 AC6):

| Deferred item | Constraint the MVP keeps for it |
| --- | --- |
| Public recipe publishing and distribution | Recipes stay private by default. The ODbL share-alike review is needed before publishing OFF-derived values. |
| **Optional affiliate grocery shopping** on public recipes (the preferred future monetization) | Ingredient identity and quantities never store retailer products, prices, or links. A future mapping table joins on food identity ([data model](data-model.md#grocerylist)). |
| Accounts and household sync | Stable client-generated UUIDs and a versioned backup format keep a later sync possible. |
| Live USDA API search | Only the bundled catalog. No API key in the client. |
| Website recipe import | Manual recipe entry only. |
| AI meal planning or acceptance prediction | Manual planning only. Feedback is never turned into predictions. |
| Wearable and exercise integrations | Goals are user-entered. No exercise calorie adjustment. |
| Full pantry management | Grocery list "already have" checks only. No inventory deduction. |

All user features remain free; no premium tier is planned.

## Release gate (FM-8 AC4)

The MVP is releasable to a pilot only when all of these hold. They are verified in [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) using the [verification checklist](mvp-delivery-plan.md#cross-platform-verification-checklist):

1. The main family journey (J1) works end to end on iOS, Android, and web.
2. Durable data survives app restart, browser reload, and a supported schema upgrade.
3. Failed lookups (not found, offline, timeout, permission denied, rate-limited) fall back to manual entry without losing input.
4. Unknown nutrients stay unknown through foods, recipes, batches, diary totals, and backups.
5. Family flows are fully usable without any child calorie target or child diary.
6. There are no open calculation or data-loss defects.

## Pilot questions

The pilot ([FM-32](https://frostfiredigital-37308542.atlassian.net/browse/FM-32)) is a small, voluntary, one-week household trial. It uses participant-reported measures only and uploads nothing automatically.

| # | Question | Measure | Signal worth continuing |
| --- | --- | --- | --- |
| P1 | Is logging fast enough to sustain? | Self-timed seconds to log (a) a recent food and (b) a batch portion. Days with at least one log out of 7. | Median ≤ 30 s for a repeat log. Logging on ≥ 5 of 7 days. |
| P2 | Does batch prep reduce effort? | Number of batch portions logged without re-entering ingredients. Participant rating of effort versus their current method (1–5). | ≥ 3 batch portions logged. Rating ≥ 4. |
| P3 | Does planning plus groceries save a shopping trip or list-writing time? | Self-reported minutes spent planning and making a list, compared with a typical week. Number of manual grocery edits. | Time reported as the same or lower. Edits mostly "already have" checks rather than corrections. |
| P4 | Does the app help remember what the kids enjoy? | Feedback records entered. Whether the participant used the feedback filter or history when planning. | Feedback recorded for ≥ 2 meals. Participant says it changed or confirmed a choice. |
| P5 | Are child adaptations easy? | Serving variations created or reused. Participant rating of ease (1–5). | ≥ 1 variation reused. Rating ≥ 4. |
| P6 | Is the nutrition data trusted? | Count of reported wrong totals or confusing "partial" states. | No calculation defects. "Partial" understood without explanation. |
| P7 | Would they keep using it? | Yes/no plus a reason. | Majority yes. Reasons inform the continue/simplify decision. |

The thresholds are starting hypotheses for a continue/simplify decision. They are not commitments.
