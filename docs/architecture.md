# Architecture

This document records the implementation decision for the Family Meals MVP ([FM-8](https://frostfiredigital-37308542.atlassian.net/browse/FM-8) AC3). [FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) turns it into a running app shell.

Related: [product requirements](product-requirements.md) · [data model](data-model.md) · [delivery plan](mvp-delivery-plan.md)

Evidence tags used below:

- **[V]**: verified against current official documentation or a live check on 2026-11-03 (source linked).
- **[A]**: assumption or inference. It must be confirmed in the named Jira story before anyone relies on it.

## Decision

**Adopt the Expo/React Native shared-code direction, conditionally.** It is a single Expo app (TypeScript, Expo Router) targeting iOS, Android, and web, with:

- `expo-sqlite` as the only persistence layer on all three platforms;
- `expo-camera` `CameraView` for barcode scanning on all three platforms, with manual barcode entry always available;
- pure TypeScript domain modules for units, nutrients, portions, ledgers, and groceries, shared by every platform and tested without a device;
- no backend, no accounts, no ORM, and no global state library for the MVP.

The decision is conditional on FM-9 proving, on real targets, that web SQLite data survives reloads and that native and web writes are atomic (see [feasibility gates](#feasibility-gates-for-fm-9)). If web persistence fails those gates, the fallback is described under [platform fallbacks](#platform-fallbacks). The fallback does not change the domain model or native stack.

### Why this is the smallest workable option

| Option | Assessment |
| --- | --- |
| **Expo (chosen)** | One codebase and one data layer. Official modules exist for SQLite and camera barcode scanning on iOS, Android, and web [V]. The default template includes Expo Router, TypeScript, and web dependencies [V]. |
| Separate native apps + web app | Three UI codebases and three persistence layers for a one-person MVP. Rejected. |
| Web/PWA only | iOS storage eviction and installation friction make it unreliable as the *only* client (see [web durability](#web-durability)). Rejected as primary; web remains a supported target. |
| Capacitor or bare React Native | Viable, but they add native-project maintenance that Expo's managed workflow avoids. There is no team preference recorded for them. Not chosen; revisit only if Expo blocks a core requirement. |

## Verified platform facts

### Expo SDK

- The current stable release is **SDK 58** (15 Oct 2026), React Native 0.88, React 19.3 [V: [changelog](https://expo.dev/changelog), [versions](https://docs.expo.dev/versions/latest/)]. On 2026-11-03, `npm view expo version` returned `58.0.23` [V: live check].
- `create-expo-app`'s default template includes Expo Router and TypeScript [V: [create-expo](https://docs.expo.dev/more/create-expo/)]. The template source lists `react-native-web` and a `web` script [V: [template package.json](https://github.com/expo/expo/blob/main/templates/expo-template-default/package.json)]. Whether the *published* template matches `main` exactly is [A]; FM-9 confirms.

### Persistence: expo-sqlite

Verified from the [expo-sqlite docs](https://docs.expo.dev/versions/latest/sdk/sqlite/) (npm `expo-sqlite@58.0.14`):

- Supported on Android, iOS, and Web, and included in Expo Go [V]. The web support has no alpha or experimental label [V].
- On web, it uses a prebuilt **WebAssembly** SQLite build **without full-text search** [V]. Web setup requires:
  - a Metro config change to bundle `.wasm` [V];
  - `Cross-Origin-Embedder-Policy` and `Cross-Origin-Opener-Policy` response headers, which enable `SharedArrayBuffer` [V].
- `withExclusiveTransactionAsync` is **not supported on web** [V]. `withTransactionAsync` works everywhere but is *not exclusive*: other queries issued while it is open run inside it [V].
- The docs show a `PRAGMA user_version` migration pattern run from `SQLiteProvider`'s `onInit` [V].
- Drizzle ORM is listed as a third-party integration [V].
- The docs do **not** describe which browser storage backs the web database, or behavior with multiple tabs, Safari, or private browsing [V: absent]. These are open questions for FM-9.

### Web durability

- Browser storage is best-effort by default and can be evicted under storage pressure, least-recently-used origin first. `navigator.storage.persist()` asks the browser to exempt the origin; Chromium and Safari decide by heuristics, and Firefox prompts the user [V: [MDN storage quotas](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)].
- Safari with tracking prevention deletes script-writable storage for a site that has had no user interaction in 7 days of browser use. Home Screen web apps are effectively exempt [V: [WebKit 2020](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/), [WebKit 2023](https://webkit.org/blog/14403/updates-to-storage-policy/)].
- Private browsing data is usually deleted when the session ends [V: MDN].
- Whether the 7-day rule covers the storage that expo-sqlite web uses is not documented. We assume it does, because eviction is per origin [A].

**Consequence:** web is a supported client, but it is not a safe sole copy of a household's data unless storage is persisted or backups are taken. The app shows the persistence state and backup age in Settings ([FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) AC5, [FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29)).

### Barcode scanning: expo-camera

- `CameraView` `barcodeScannerSettings` / `onBarcodeScanned` are supported on Android, iOS, and **Web**. Supported types include `ean13`, `ean8`, `upc_a`, and `upc_e` [V: [expo-camera docs](https://docs.expo.dev/versions/latest/sdk/camera/)].
- `launchScanner` (the system scanner UI) is Android/iOS only, and `scanFromURLAsync` reads only QR codes on iOS [V]. Neither is needed.
- `useCameraPermissions` works on all three platforms [V].
- `expo-camera@58.0.12` and later depend on `barcode-detector` [V: `npm view expo-camera dependencies` returned `{ 'barcode-detector': '^3.0.0' }`]. That package is a `BarcodeDetector` polyfill built on ZXing-C++ WebAssembly, so web decoding works without native `BarcodeDetector` support [A: inferred from the package; confirm on Safari and Firefox in FM-13].
- The native Barcode Detection API is experimental and limited. It ships in Chrome Android and in Chrome on macOS/ChromeOS, sits behind a flag in Safari, and is absent from Firefox [V: [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API)]. We do not depend on it directly.
- Camera access on web requires a secure context (HTTPS or localhost) [A: standard browser behavior; not stated on the Expo page].
- The legacy `expo-barcode-scanner` package is no longer maintained and is not used [A: last npm publish April 2024; its docs redirect to expo-camera].

### Open Food Facts

From the [OFF API docs](https://openfoodfacts.github.io/openfoodfacts-server/api/) and [terms of use](https://world.openfoodfacts.org/terms-of-use):

- **API v3 is "recommended for all new integrations"** [V]. Product read: `GET https://world.openfoodfacts.org/api/v3.6/product/{barcode}.json` [V: live check].
- Reads need no authentication, only a custom User-Agent of the form `AppName/Version (ContactEmail)` [V]. OFF also asks integrators to fill in an API usage form [V].
- Rate limits are **15 product reads per minute per IP** and 10 searches per minute [V]. Above a few hundred products, OFF asks for bulk data downloads instead [V].
- A staging server exists at `world.openfoodfacts.net` [V]. Use it during development.
- Licensing [V]:
  - The database is ODbL 1.0 and individual contents are DbCL 1.0. Images are CC BY-SA 3.0.
  - Attribution requires crediting Open Food Facts, naming the license, and linking to the source.
  - Derivative databases that are used publicly must be share-alike.
- Barcodes are normalized server-side: strip leading zeros, pad 9–12-digit codes to 13 digits, and pad codes of 7 digits or fewer to 8 [V: [barcode normalization](https://openfoodfacts.github.io/openfoodfacts-server/api/ref-barcode-normalization/)]. The app applies the same rule to compute `barcode.normalized` and also stores the scanned digits unchanged ([data model](data-model.md#foodrecord)).

**Live check on 2026-11-03** (one request for product `3017624010701`):

- The response returned `access-control-allow-origin: *` and allowed the `User-Agent` and `X-User-Agent` request headers. Browser calls are therefore possible without a proxy [V: live check]. CORS is not documented, so treat it as subject to change [A].
- In v3.6, nutrients are under `product.nutrition.aggregated_set.nutrients`. Each value carries `value`, `unit`, `source` (`packaging` or `estimate`), and `source_per` (`100g`). The legacy `product.nutriments` object came back **empty** for this product [V: live check]. The importer must read the v3.6 structure and must not depend on `nutriments`.
- Sodium was reported in **grams** (`0.043 g`), so it must be converted to mg. Fiber was absent and must stay unknown. Some values had `source: "estimate"` and modifier `~` [V: live check].

- The same response contained `allergens_tags` (`en:nuts`) and `traces_tags` (`en:milk`, `en:nuts`, `en:soybeans`) [V: live check]. These are imported into `FoodRecord.allergens` so household restrictions can be flagged. A product with neither field is `declared: false`, so its restriction status is `unknown`.

**Decision:** import only `packaging` (label) values by default. `estimate` values are excluded, so those nutrients stay unknown. This keeps "do not invent nutrient values" ([FM-11](https://frostfiredigital-37308542.atlassian.net/browse/FM-11), [FM-21](https://frostfiredigital-37308542.atlassian.net/browse/FM-21)) simple. FM-11 may revisit this with an explicit "estimated" status.

### USDA FoodData Central

From the [FDC API guide](https://fdc.nal.usda.gov/api-guide/) and [download datasets](https://fdc.nal.usda.gov/download-datasets/):

- FDC data is public domain under **CC0 1.0**. USDA asks users to cite FDC as the source [V].
- The API requires a data.gov key and has a default limit of 1,000 requests/hour/IP. USDA warns that exposed keys are deactivated [V].
- **Foundation Foods** has an October 2026 release in JSON and CSV. **SR Legacy** (April 2018) is the final release and will not be updated [V].

**Decision:** ship a small bundled catalog built offline from downloaded Foundation Foods data, using SR Legacy only for staples that Foundation Foods lacks. The client never calls the FDC API and contains no USDA key ([FM-14](https://frostfiredigital-37308542.atlassian.net/browse/FM-14)).

- **Catalog IDs** are deterministic (UUIDv5 of `usda:<fdcId>`), so references survive catalog upgrades and backups taken on other devices ([data model](data-model.md#identifiers-and-shared-conventions)).
- **Energy mapping** [A: not verified against the October 2026 files]: some Foundation Foods records may report energy only as Atwater-factor nutrients rather than the standard energy nutrient. The build script must check which energy nutrients each selected record has, and map them in this order: standard energy (kcal), then Atwater specific factors, then Atwater general factors. It records which one was used in the catalog's source notes. A food with none of these stays unknown. FM-14 confirms the nutrient IDs against the downloaded files.

## Application structure (for FM-9)

There is one Expo app; it is not a monorepo. The proposed layout:

```text
app/                 Expo Router routes: today, recipes, week, family, settings
src/domain/          Pure TS: units, nutrients, aggregation, portions, batch ledger, plan reservations, grocery merge
src/data/            SQLite open/config, migrations, repositories (one module per aggregate), backup export/import
src/providers/       Open Food Facts client, barcode normalization, USDA catalog loader
src/ui/              Shared components and tokens
catalog/             Generated USDA starter catalog (JSON) + build script + source notes
```

Rules:

- `src/domain` imports nothing from React, Expo, or SQLite. All calculations live here and run under unit tests on any machine.
- Repositories own SQL. Screens call repositories and domain functions; they never write SQL.
- Snapshots and nutrient profiles are stored as JSON text columns. Relational columns hold IDs, dates, and status fields that need indexing or joins.
- The ORM decision is reversible: start with plain SQL behind repositories. Adopt Drizzle (documented as compatible [V]) only if hand-written migrations or queries become a real maintenance cost.

### Writes and atomicity

- **All writes go through one in-process write queue** that runs each unit of work inside `withTransactionAsync`. Exclusive transactions are unavailable on web [V], and non-exclusive transactions absorb concurrent queries [V], so serializing writes in JavaScript gives the same behavior on every platform.
- A unit of work is one user action, such as "log batch portion" or "confirm planned meal eaten". Related records are written together or not at all ([FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) AC3).
- Idempotency keys (`clientRequestId`) prevent duplicate saves from repeated taps ([FM-18](https://frostfiredigital-37308542.atlassian.net/browse/FM-18) AC5).
- On a failed write, the transaction rolls back, the form keeps its input, and the user can retry.

### Migrations

- The schema version lives in `PRAGMA user_version`, following the documented expo-sqlite pattern [V]. Migrations are an ordered array of functions in `src/data/migrations`. Each one runs in its own transaction and bumps `user_version`.
- Migrations are forward-only. A database newer than the app (for example after a downgrade) opens read-only with a message and is never rewritten.
- Migration tests seed a database at version N and assert data after migrating to the latest version. FM-9 decides where these run: a Node SQLite binding in CI [A], device/E2E checks, or both.
- The **backup format has its own version** ([FM-29](https://frostfiredigital-37308542.atlassian.net/browse/FM-29)). Import upgrades older backup versions through the same domain-level transforms. Restore never runs raw SQL taken from a file.
- The USDA catalog has a `catalogVersion`. An app upgrade updates catalog rows in place by deterministic ID and bumps their `revision`. Removed items are archived, never deleted. User foods and `user_corrected` copies are untouched. Backups exclude catalog rows; restore checks catalog references against the installed catalog.

### Offline behavior

| Capability | Offline? |
| --- | --- |
| Diary, recipes, batches, family, plans, groceries, backup export | Yes. All local. |
| USDA starter catalog search | Yes. Bundled. |
| Previously looked-up barcodes | Yes. They are cached as `FoodRecord`s and an exact barcode match is served locally. |
| New barcode lookup | No. Shows an "offline" state and continues to manual entry with the barcode kept ([FM-13](https://frostfiredigital-37308542.atlassian.net/browse/FM-13) AC6). |
| Web app first load | Needs network. Offline reload of the web app shell (service worker) is **not** in MVP scope [A: decision]. Data already stored stays available once the page is loaded. |

### Provider calls, privacy, and keys

- **Only barcodes leave the device.** No diary entries, profiles, feedback, photos, or analytics are sent anywhere ([FM-9](https://frostfiredigital-37308542.atlassian.net/browse/FM-9) AC4, [FM-31](https://frostfiredigital-37308542.atlassian.net/browse/FM-31) AC5).
- **No secrets exist in the client.** OFF reads need none. The USDA key, if one is ever used, belongs to the catalog build script on a developer machine and is never committed. `EXPO_PUBLIC_` variables are embedded in plain text in the app bundle, so no secret may use them [V: [Expo env vars](https://docs.expo.dev/guides/environment-variables/)].
- **OFF client rules:**
  - Look up one request per distinct normalized barcode.
  - Pause scanning after the first detection, and never fire per camera frame or per keystroke.
  - Serve cached exact matches first.
  - Apply a timeout, and treat HTTP 429/503 as a "try again later" state that offers manual entry.
- **OFF identification on native:** send `User-Agent: FamilyMeals/<version> (<contact>)`. The contact address is a release decision (see open questions).
- **OFF identification on web:** browsers do not let scripts set `User-Agent`. The server's CORS policy allows `X-User-Agent` [V: live check], but OFF does not document this as satisfying its identification requirement [A]. The web client sends `X-User-Agent` with the same value, and FM-13 confirms with OFF before release.
- **Attribution:** a "Data sources" screen in Settings credits Open Food Facts (ODbL/DbCL, link) and USDA FoodData Central (CC0, citation). Food detail screens show the source and retrieval date for imported foods.
- **Share-alike:** OFF data stays in the user's private local database and private backups. Any future public recipe publishing must review ODbL share-alike obligations before exposing OFF-derived values (later scope).

## Platform fallbacks

| Situation | Behavior |
| --- | --- |
| Camera permission denied, no camera (desktop), or insecure context | Manual barcode field with the same lookup and validation. A short explanation, and a link to settings where the OS allows it. |
| Barcode not decoded on a browser | Manual entry. Scanning on web uses the expo-camera path (polyfilled decoding [A]). We add no second scanning library unless FM-13 shows a real browser gap. |
| Lookup not found, offline, timeout, rate-limited | Clear state. Continue to manual food creation with the barcode kept. |
| Web storage not persisted, or Safari not installed to the Home Screen | A Settings banner explains the risk and recommends a backup or Home Screen install. Logging is never blocked. |
| Web headers (COOP/COEP) cannot be set by the host | expo-sqlite web cannot initialize [A: inferred from the documented requirement]. Choose a static host that supports custom headers (FM-9 verifies on the preview host). |
| expo-sqlite web fails FM-9 gates | Keep SQLite on native. Implement the repository interface on web over IndexedDB, using the same domain modules and the same backup format. This is the only sanctioned abstraction point, and it is built only if the gate fails. |
| Multiple browser tabs | Unknown behavior [A]. FM-9 tests it. If it is unsafe, allow one writable tab (for example via the Web Locks API) and show the others a read-only message. |

## Feasibility gates for FM-9

FM-9 must record results for each gate. FM-8 did not run a device build.

1. Fresh `create-expo-app` on SDK 58 runs on an iOS simulator or device, an Android emulator or device, and a desktop browser.
2. expo-sqlite web initializes with the documented Metro and header configuration, both in the dev server and on the chosen preview host.
3. A record survives app restart (native), and browser reload and browser restart (web: Chrome and Safari at minimum).
4. A multi-record write that throws midway leaves no partial rows on native and web.
5. Migration from `user_version` 1 to 2 preserves seeded data on all three platforms.
6. The `navigator.storage.persist()` result is shown in Settings on web.
7. Two tabs open on the same origin: document the behavior and apply the single-writer guard if needed.
8. `CameraView` decodes EAN-13 and UPC-A on iOS, Android, Chrome, and Safari (scanning can move to FM-13 if it is not ready in FM-9).

## Open questions

| Question | Owner | Blocking? |
| --- | --- | --- |
| Contact string for the OFF User-Agent (an email or URL the project is willing to publish) | Product owner | Blocks shipping FM-13, not development (use staging). |
| Does OFF accept `X-User-Agent` or `app_name` as identification for browser clients? | FM-13 | Blocks web barcode lookup release only. |
| Which static host serves the web preview with COOP/COEP headers? | FM-9 / FM-32 | Blocks the web preview. |
| Storage backing and multi-tab safety of expo-sqlite on web | FM-9 | Blocks web durability claims. |
| Where migration tests run (Node binding vs. device) | FM-9 | No. |
