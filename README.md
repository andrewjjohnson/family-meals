# Family Meals

Family Meals helps a parent meet their own nutrition and fitness goals while cooking meals their children are more likely to enjoy. It combines:

- **fast food logging**: barcode lookup, a bundled ingredient catalog, manual foods, and one-tap repeats;
- **weekly meal prep**: cooked batches with measured yields, portions, and leftovers;
- **a family cookbook that improves over time**: serving variations for each child and quick feedback about what they enjoyed.

It is planned for iOS, Android, and web. It works locally without accounts or cloud sync, and every user feature is free.

## Status

**Planning (Phase 0).** This repository currently contains the MVP blueprint only; there is no application code yet. The app shell, with durable local storage on all three platforms, is the next task: [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9).

## Documentation

| Document | Contents |
| --- | --- |
| [docs/product-requirements.md](docs/product-requirements.md) | Users, key journeys, MVP and deferred scope, release gate, pilot questions |
| [docs/architecture.md](docs/architecture.md) | Expo/React Native decision, verified platform facts, persistence, barcode scanning, migrations, offline behavior, provider attribution and keys |
| [docs/data-model.md](docs/data-model.md) | Domain records, identifiers, invariants, how edits propagate, worked calculation examples |
| [docs/mvp-delivery-plan.md](docs/mvp-delivery-plan.md) | Jira issues by phase, dependencies, release criteria, cross-platform verification checklist |

## Jira

The project is **FM (Family Meals)**: <https://frostfiredigital-37308542.atlassian.net/jira/software/projects/FM>

- MVP scope index: [FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8)
- Epics: [FM-1](https://frostfiredigital-37308542.atlassian.net/browse/FM-1) foundation · [FM-2](https://frostfiredigital-37308542.atlassian.net/browse/FM-2) food data · [FM-3](https://frostfiredigital-37308542.atlassian.net/browse/FM-3) recipes & batches · [FM-4](https://frostfiredigital-37308542.atlassian.net/browse/FM-4) logging · [FM-5](https://frostfiredigital-37308542.atlassian.net/browse/FM-5) family · [FM-6](https://frostfiredigital-37308542.atlassian.net/browse/FM-6) planning · [FM-7](https://frostfiredigital-37308542.atlassian.net/browse/FM-7) release & pilot
- Stories carry `delivery-phase-0` … `delivery-phase-3` labels and blocking links.

Scope changes go to FM-8 first, and then to these documents.

## Data sources

- Food product data from [Open Food Facts](https://world.openfoodfacts.org), under the Open Database License (database) and Database Contents License (contents).
- Ingredient data from U.S. Department of Agriculture, Agricultural Research Service, [FoodData Central](https://fdc.nal.usda.gov) (CC0 1.0).
