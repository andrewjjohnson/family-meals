# MVP delivery plan

This plan maps the existing Jira backlog to the four delivery phases in [FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8) AC5. It does **not** create new work items. Jira remains the backlog, and this file explains the order, the contracts each phase depends on, and how to check a release.

Related: [product requirements](product-requirements.md) · [architecture](architecture.md) · [data model](data-model.md)

Sources: all 32 FM issues (FM-1 to FM-32) were read through the Jira API on 2026-11-03. Phase labels and "is blocked by" links below come from that read. No dates or estimates are implied, consistent with the backlog.

## Phases at a glance

```text
Phase 0  FM-8 ─► FM-9 ─► FM-10
           └──────────► FM-11 (pure domain; can run alongside FM-9)
Phase 1  FM-11 + FM-9 ─► FM-12 ─► FM-13, FM-14
         FM-12 + FM-10 ─► FM-15 ─► FM-16 ─► FM-17 ─┐
         FM-12 + FM-10 ─► FM-18 ─────────────────┴► FM-19
                          FM-18 ─► FM-20 (also FM-15), FM-21
Phase 2  FM-21 ─► FM-22
         FM-15 ─► FM-23 ─► FM-24 (also FM-16) ─► FM-25 ─► FM-26 ─► FM-27 (also FM-19) ─► FM-28
Phase 3  FM-9 ─► FM-29
         FM-21 + FM-28 ─► FM-30
         FM-29 + FM-30 + FM-22 + FM-13 + FM-14 + FM-20 ─► FM-31 ─► FM-32
```

**Critical path:** FM-8 → FM-9 → FM-10 → FM-15 → FM-23 → FM-24 → FM-25 → FM-26 → FM-27 → FM-28 → FM-30 → FM-31 → FM-32. FM-15 also requires FM-12, which in turn needs FM-11. The recipe chain (FM-16 → FM-17 → FM-19) must also finish before FM-27.

## Phase 0: foundation and contracts

| Issue | Summary | Blocked by | Key contracts |
| --- | --- | --- | --- |
| [FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8) | Define MVP scope, domain contracts, and delivery sequence | — | Produces these documents. |
| [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) | Run a shared app with durable local storage on all target platforms | FM-8 | [Architecture decision](architecture.md#decision), [feasibility gates](architecture.md#feasibility-gates-for-fm-9), [writes and migrations](architecture.md#writes-and-atomicity), [identifiers](data-model.md#identifiers-and-shared-conventions) |
| [FM-10](https://frostfiredigital-37308542.atlassian.net/browse/FM-10) | Navigate logging, recipes, family, and weekly planning in a calm interface | FM-9 | [Product principles](product-requirements.md#product-principles) |

**Exit criteria**

- These documents are merged, and FM-8 is updated to link them.
- Every FM-9 feasibility gate has a recorded result. Any failed gate has a decision, such as the web fallback in [platform fallbacks](architecture.md#platform-fallbacks).
- The app shell runs on iOS, Android, and web. A saved record survives restart or reload, and the migration framework has been exercised once.
- The navigation shell reaches Today, Recipes, Week, Family, and Settings, with accessible controls.

## Phase 1: food capture, recipes, and diary

| Issue | Summary | Blocked by | Data model sections |
| --- | --- | --- | --- |
| [FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) | Normalize nutrients and calculate quantities with visible data coverage | FM-8 | [Quantities](data-model.md#quantities-and-units), [Nutrients](data-model.md#nutrients), worked examples 1, 2, 4 |
| [FM-12](https://frostfiredigital-37308542.atlassian.net/browse/FM-12) | Create, edit, and reuse manually entered foods | FM-9, FM-11 | [FoodRecord](data-model.md#foodrecord), [FoodSnapshot](data-model.md#foodsnapshot-value-object) |
| [FM-13](https://frostfiredigital-37308542.atlassian.net/browse/FM-13) | Scan or enter packaged-food barcodes with Open Food Facts lookup | FM-12, FM-10 | [Barcode scanning](architecture.md#barcode-scanning-expo-camera), [Open Food Facts](architecture.md#open-food-facts), [provider calls](architecture.md#provider-calls-privacy-and-keys) |
| [FM-14](https://frostfiredigital-37308542.atlassian.net/browse/FM-14) | Search a USDA-sourced starter catalog for common cooking ingredients | FM-12 | [USDA FoodData Central](architecture.md#usda-fooddata-central), [migrations: catalog version](architecture.md#migrations) |
| [FM-15](https://frostfiredigital-37308542.atlassian.net/browse/FM-15) | Create and maintain private recipes with instructions and versioned ingredients | FM-12, FM-10 | [Recipe](data-model.md#recipe), [RecipeVersion](data-model.md#recipeversion), example 3 |
| [FM-16](https://frostfiredigital-37308542.atlassian.net/browse/FM-16) | Build meals from separately portioned components | FM-15 | [Component](data-model.md#component), example 2 |
| [FM-17](https://frostfiredigital-37308542.atlassian.net/browse/FM-17) | Record actual cooked batches and measured yields | FM-16 | [CookedBatch](data-model.md#cookedbatch), example 1 |
| [FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) | Log individual foods and unnamed combinations for a chosen date | FM-12, FM-10 | [DiaryEntry](data-model.md#diaryentry) |
| [FM-19](https://frostfiredigital-37308542.atlassian.net/browse/FM-19) | Log cooked portions and track remaining batch quantities | FM-17, FM-18 | [CookedBatch derived quantities](data-model.md#cookedbatch), [How changes propagate](data-model.md#how-changes-propagate) |
| [FM-20](https://frostfiredigital-37308542.atlassian.net/browse/FM-20) | Repeat recent foods and combinations without cluttering the recipe library | FM-18, FM-15 | [Recents](data-model.md#recents-derived), saved meals on [Recipe](data-model.md#recipe) |
| [FM-21](https://frostfiredigital-37308542.atlassian.net/browse/FM-21) | Review daily macros and available micronutrients with clear coverage | FM-18 | [Aggregation and coverage](data-model.md#aggregation-and-coverage), example 4 |

**Exit criteria**

- A food can be entered three ways (scan or barcode entry, catalog, manual) and used in both a recipe and the diary.
- Worked examples 1–4 (including the 1b batch corrections) pass as automated fixtures through `src/domain`.
- A batch logged across several days keeps its snapshot after the recipe and source food are edited.
- Failed lookups (not found, offline, timeout, denied, rate-limited) continue to manual entry with the barcode retained.
- Open Food Facts and USDA attribution is visible. No secrets are in the bundle.

## Phase 2: family meals and weekly planning

| Issue | Summary | Blocked by | Data model sections |
| --- | --- | --- | --- |
| [FM-22](https://frostfiredigital-37308542.atlassian.net/browse/FM-22) | Set optional adult nutrition goals and review a simple weekly summary | FM-21 | `AdultGoal` under [HouseholdMember](data-model.md#householdmember) |
| [FM-23](https://frostfiredigital-37308542.atlassian.net/browse/FM-23) | Create local household profiles with food preferences and restrictions | FM-15 | [HouseholdMember](data-model.md#householdmember) |
| [FM-24](https://frostfiredigital-37308542.atlassian.net/browse/FM-24) | Save family serving variations that reuse a common recipe | FM-16, FM-23 | [ServingVariation](data-model.md#servingvariation), example 2 |
| [FM-25](https://frostfiredigital-37308542.atlassian.net/browse/FM-25) | Record quick family feedback and find recipes that worked | FM-24 | [RecipeFeedback](data-model.md#recipefeedback), example 6 |
| [FM-26](https://frostfiredigital-37308542.atlassian.net/browse/FM-26) | Plan a week of personal meal prep and family meals | FM-25 | [WeekPlan](data-model.md#weekplan-prepevent-and-plannedmeal) |
| [FM-27](https://frostfiredigital-37308542.atlassian.net/browse/FM-27) | Allocate prep batches and planned leftovers across the week | FM-26, FM-19 | [WeekPlan reservations](data-model.md#weekplan-prepevent-and-plannedmeal), example 5 |
| [FM-28](https://frostfiredigital-37308542.atlassian.net/browse/FM-28) | Generate an editable consolidated grocery list from planned cooking | FM-27 | [GroceryList](data-model.md#grocerylist), example 5 |

**Exit criteria**

- Journey J1 steps 3–8 work on at least one platform. Full cross-platform validation happens in Phase 3.
- Worked examples 5 and 6 pass as automated fixtures. Groceries count each prep event once. Eaten or skipped planned meals create exactly the expected diary entries and adjustments.
- There is no child calorie, goal, or diary UI anywhere.

## Phase 3: backup, validation, and household pilot

| Issue | Summary | Blocked by | Notes |
| --- | --- | --- | --- |
| [FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29) | Export and restore a complete local household backup | FM-9 | Backup format version is separate from the schema version ([migrations](architecture.md#migrations)). Round trip keeps IDs ([identifiers](data-model.md#identifiers-and-shared-conventions)). |
| [FM-30](https://frostfiredigital-37308542.atlassian.net/browse/FM-30) | Guide first use through logging and the family meal workflow | FM-21, FM-28 | Sample data must be removable and must never reach real totals. |
| [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) | Validate the full MVP journey and critical calculations across platforms | FM-29, FM-30, FM-22, FM-13, FM-14, FM-20 | Uses the [checklist](#cross-platform-verification-checklist) below. |
| [FM-32](https://frostfiredigital-37308542.atlassian.net/browse/FM-32) | Run a small household pilot and prioritize findings | FM-31 | Answers the [pilot questions](product-requirements.md#pilot-questions). |

**Exit criteria (the MVP release gate)**

The [release gate](product-requirements.md#release-gate-fm-8-ac4) holds on all three platforms, there are no open calculation or data-loss defects, and the pilot ends with a recorded continue/simplify decision.

## Dependency notes

These are observations, not changes. The Jira links remain authoritative.

- **Some description dependencies are transitive rather than linked.** FM-23 lists FM-9/FM-10, FM-26 lists FM-24, and FM-30 lists FM-25 in their descriptions without a direct link. Each is already implied by the link chain, which matches FM-8's note that dependency lists may mention transitive prerequisites.
- **FM-11 can start immediately after FM-8**, in parallel with FM-9. It is pure domain code with no storage or UI dependency.
- **FM-26 waits on FM-25** (feedback). Only FM-26 AC4 (showing feedback while choosing) needs it. If planning becomes the bottleneck, the link could be relaxed so that feedback display is added once FM-25 lands. That is a Jira decision for the product owner.
- **FM-29 is only blocked by FM-9.** Its round-trip criteria (AC4) need the Phase 2 records, so it is most useful once the Phase 2 schema is stable. Starting the export format earlier helps catch schema drift.

## Cross-platform verification checklist

This checklist is used by [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) and, in parts, at each phase exit. Run it on a physical iPhone, a physical Android phone, desktop Chrome, and Safari (macOS, plus iOS Safari added to the Home Screen). Simulators are fine during development, but camera checks need real devices.

### Persistence and upgrades

- [ ] Create a food, recipe, batch, diary entry, profile, plan, and grocery check. Then force-quit and relaunch (native), and reload and restart the browser (web). Everything is present.
- [ ] Install build N, add data, upgrade to build N+1 with a migration, and confirm that the data and computed totals are unchanged.
- [ ] Simulate a failing multi-record write: no partial records remain, and the form keeps its input.
- [ ] Double-tap Save on a diary entry, and double-tap "mark eaten" on a meal with an adult and a child: exactly one diary entry and one set of child adjustments are created. Undo eaten, then mark eaten again: the records are recreated once.
- [ ] Web: Settings shows whether persistent storage was granted and when the last backup was taken. Two tabs behave as documented in FM-9.

### Food capture and lookup

- [ ] Scan EAN-13 and UPC-A products on each platform. A product with leading zeros keeps its digits.
- [ ] Deny camera permission, then use manual barcode entry. On desktop without a camera, manual entry works.
- [ ] Airplane mode, then look up a new barcode: an offline state appears and the user can continue to manual entry with the barcode retained. A previously scanned barcode resolves offline.
- [ ] An unknown barcode reaches the not-found state and continues to manual entry.
- [ ] One lookup request is made per scan. The network log shows no repeated requests per frame.
- [ ] The network log shows only barcode lookups to Open Food Facts. No diary, profile, or feedback data leaves the device. The bundle contains no API keys.
- [ ] USDA catalog search works offline and distinguishes raw from cooked variants.
- [ ] The Data sources screen shows OFF (ODbL/DbCL) and USDA (CC0) attribution.

### Calculations

- [ ] Worked examples 1–6 in [data-model.md](data-model.md#worked-examples) reproduce exactly in the app UI on each platform.
- [ ] A food with missing micronutrients shows "partial" totals with the missing contributor named. A measured zero shows as 0.
- [ ] A backdated entry, and an entry moved to another day, land on the correct local date. Changing the device time zone does not move entries.

### Family and planning

- [ ] Create a child profile without a birth date, goal, or diary. No calorie target appears anywhere for the child.
- [ ] Restriction flags show "flagged" for a scanned product whose allergen tags match, and "unknown" when the source declares no allergens. They never show "safe" based on a name.
- [ ] Plan a Sunday cook plus three leftover lunches. The grocery list counts the cook once. Marking a lunch skipped creates no diary entry and releases the reservation. Logging unplanned food from the batch past its reservations shows the over-allocation warning.
- [ ] Record a batch, log a portion, then record a corrected yield: remaining and new portions follow the new yield, and earlier portions change only if confirmed.
- [ ] Change the week-start day: no planned meal or prep event disappears or moves date.
- [ ] Rebuild the grocery list after a plan change: valid checks are kept and changed quantities are explained.
- [ ] Feedback shows the variation revision actually served after the variation is edited.

### Backup

- [ ] Export, wipe (or use a fresh install), and restore: IDs, totals, history, portions, and allocations match.
- [ ] Import a corrupted or newer-version file: it is rejected with an explanation and existing data is unchanged.
- [ ] Back up on one platform and restore on another (for example iOS to web). Catalog foods referenced by recipes resolve after restore, and the week-start setting is restored.

### Accessibility and layout

- [ ] VoiceOver (iOS) and TalkBack (Android) announce the core controls on Today, Recipes, Week, and Family.
- [ ] Web: keyboard-only navigation through logging and planning works with a visible focus indicator.
- [ ] At the largest system text size, primary actions remain visible on a small phone.
- [ ] No guilt-based copy, ads, or subscription prompts appear.

## Next actionable step

Start **[FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9)**:

1. Scaffold the SDK 58 default template.
2. Configure expo-sqlite web (Metro `.wasm` plus COOP/COEP headers).
3. Implement the write queue and the `user_version` migration runner.
4. Record results for each [feasibility gate](architecture.md#feasibility-gates-for-fm-9).

[FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11) can run in parallel, starting with the worked examples as fixtures.
