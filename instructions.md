# AEM Experimentation Plugin — Integration Instructions

## Overview

The AEM Experimentation plugin provides three content personalization features for AEM Edge Delivery Services sites:

1. **A/B Testing (Experiments)** — Randomly serve different content variants to visitors and measure conversion. Variants are defined via page metadata; no per-experiment code changes are needed.
2. **Audience Targeting** — Serve different content to different user segments (e.g., mobile vs. desktop, geo-based). Audience resolution functions are defined in project code; content mapping is done via metadata.
3. **Campaign Personalization** — Serve campaign-specific content when visitors arrive via campaign URLs (`?campaign=xmas` or `?utm_campaign=xmas`). Content mapping is done via metadata.

All three features are metadata-driven. Authors configure experiments, audiences, and campaigns in their page's metadata block in the document. The plugin reads these meta tags at runtime, resolves the appropriate variant, and swaps content before the page renders.

The plugin files are pre-installed at `plugins/experimentation/`. Do **not** modify any files inside that directory. You will only modify files in `scripts/` and `head.html`.

---

## Plugin API

The plugin entry point is `plugins/experimentation/src/index.js`. It exports:

### `loadEager(document, options, context)`
Main initialization function. Must be called during the eager phase of page loading (before LCP). This function:
- Adjusts RUM sampling rates for experiment/campaign/audience pages
- Runs campaign detection (checks `?campaign=` or `?utm_campaign=` query params)
- If no campaign matched, runs experiment detection (checks `experiment` meta tag)
- If no experiment matched, runs audience targeting (checks `audience:*` meta tags)
- Swaps page content with the resolved variant if applicable

### `loadLazy(document, options, context)`
Deferred work. Loads the preview overlay UI on non-production domains (localhost, `*.hlx.page`, `*.hlx.stage`). Should be called at the end of the lazy phase.

### Options object

```js
{
  // Production host — controls debug overlay visibility.
  // The preview pill overlay is hidden when the current host matches this value.
  prodHost: 'www.example.com',

  // Alternative: a function that returns true on prod environments
  // isProd: () => window.location.hostname === 'www.example.com',

  // Audience resolver map. Keys are audience names (kebab-case),
  // values are async functions returning boolean.
  audiences: {},

  // RUM sampling rate (default: 10, meaning 1-in-10).
  // Only applies to experiment/campaign/audience pages.
  rumSamplingRate: 10,

  // Storage backend for variant stickiness (default: window.sessionStorage)
  // Use window.localStorage for cross-session persistence.
  // storage: window.sessionStorage,

  // Meta tag and query parameter names (defaults shown):
  experimentsMetaTag: 'experiment',
  experimentsQueryParameter: 'experiment',
  audiencesMetaTagPrefix: 'audience',
  audiencesQueryParameter: 'audience',
  campaignsMetaTagPrefix: 'campaign',
  campaignsQueryParameter: 'campaign',

  // Experiment manifest location defaults:
  experimentsRoot: '/experiments',
  experimentsConfigFile: 'manifest.json',
}
```

### Context object

The plugin requires a `context` object with these methods from the project's `aem.js`/`lib-franklin.js`:

```js
{
  getAllMetadata,  // (scope: string) => Record<string, string>
  getMetadata,     // (name: string) => string
  loadCSS,         // (path: string) => Promise<void>
  loadScript,      // (path: string) => Promise<void>
  sampleRUM,       // (checkpoint: string, data?: object) => void
  toCamelCase,     // (str: string) => string
  toClassName,     // (str: string) => string
}
```

---

## Integration Steps

### Step 1: Add `getAllMetadata` helper to `scripts/scripts.js`

Check if the project already has a `getAllMetadata` function. If not, add it near the top of `scripts/scripts.js`:

```js
/**
 * Gets all the metadata elements that are in the given scope.
 * @param {String} scope The scope/prefix for the metadata
 * @returns an object with the metadata key-value pairs for the given scope
 */
export function getAllMetadata(scope) {
  return [...document.head.querySelectorAll(`meta[property^="${scope}:"],meta[name^="${scope}-"]`)]
    .reduce((res, meta) => {
      const id = toClassName(meta.name
        ? meta.name.substring(scope.length + 1)
        : meta.getAttribute('property').split(':')[1]);
      res[id] = meta.getAttribute('content');
      return res;
    }, {});
}
```

Also ensure these imports exist at the top of `scripts/scripts.js` (from `aem.js` or `lib-franklin.js`):

```js
import {
  getMetadata,
  loadCSS,
  loadScript,
  sampleRUM,
  toCamelCase,
  toClassName,
} from './aem.js';
```

> **Note:** The exact import source depends on the project. Some projects use `lib-franklin.js` instead of `aem.js`. Check which file exists in the `scripts/` directory.

### Step 2: Define the experimentation config and plugin context

Add the following near the top of `scripts/scripts.js`, after imports:

```js
const AUDIENCES = {
  mobile: () => window.innerWidth < 600,
  desktop: () => window.innerWidth >= 600,
  // Add project-specific audiences here
};

const experimentationConfig = {
  prodHost: 'PUT_PROD_HOSTNAME_HERE',
  audiences: AUDIENCES,
};

const pluginContext = {
  getAllMetadata,
  getMetadata,
  loadCSS,
  loadScript,
  sampleRUM,
  toCamelCase,
  toClassName,
};
```

Replace `'PUT_PROD_HOSTNAME_HERE'` with the project's production hostname (e.g., `'www.example.com'`). If you cannot determine the production hostname, leave it as an empty string `''` — the preview overlay will show on all environments.

### Step 3: Call `loadEager` in the eager phase

Find the `loadEager` function in `scripts/scripts.js`. Add the experimentation plugin call as early as possible inside it — before `loadHeader`, `loadFooter`, and `decorateBlocks` calls. The plugin must run before LCP because it may swap page content.

```js
async function loadEager(doc) {
  // ... existing early setup (decorateTemplatesAndThemes, etc.) ...

  // Experimentation plugin — must run before blocks are decorated
  if (getMetadata('experiment')
    || Object.keys(getAllMetadata('campaign')).length
    || Object.keys(getAllMetadata('audience')).length) {
    const { loadEager: runEager } = await import('../plugins/experimentation/src/index.js');
    await runEager(document, experimentationConfig, pluginContext);
  }

  // ... existing loadHeader, decorateBlocks, waitForFirstImage, etc. ...
}
```

The conditional check avoids importing the plugin on pages that don't use experimentation, audiences, or campaigns.

### Step 4: Call `loadLazy` in the lazy phase

Find the `loadLazy` function in `scripts/scripts.js`. Add the experimentation lazy call at the **end** of the function:

```js
async function loadLazy(doc) {
  // ... existing lazy loading code ...

  // Experimentation preview overlay (non-prod only)
  if (getMetadata('experiment')
    || Object.keys(getAllMetadata('campaign')).length
    || Object.keys(getAllMetadata('audience')).length) {
    const { loadLazy: runLazy } = await import('../plugins/experimentation/src/index.js');
    await runLazy(document, experimentationConfig, pluginContext);
  }
}
```

### Step 5: Add modulepreload to `head.html`

Add a modulepreload link hint for the plugin in `head.html`. This tells the browser to start fetching the module early, reducing latency when the eager phase imports it:

```html
<link rel="modulepreload" href="/plugins/experimentation/src/index.js" />
```

Add this line alongside the existing modulepreload links (typically after the `scripts/scripts.js` preload).

---

## Configuration Reference

### `prodHost` (string)
The production hostname (e.g., `'www.example.com'`). When the current page's host matches this value, the preview overlay pill is hidden. Also used to scope RUM performance queries.

### `isProd` (function, optional)
Alternative to `prodHost`. A function returning `true` when running on production. Use this for complex multi-domain setups:
```js
isProd: () => window.location.hostname.endsWith('.com'),
```

### `audiences` (object)
A map of audience name strings to async boolean-returning functions. Keys must be kebab-case and match the audience names used in metadata. **No audiences are built-in — all must be defined by the project.** Example:

```js
{
  mobile: () => window.innerWidth < 600,
  desktop: () => window.innerWidth >= 600,
  'new-visitor': () => !localStorage.getItem('returning'),
  us: async () => {
    const resp = await fetch('/geo-api');
    const geo = await resp.json();
    return geo.country === 'US';
  },
}
```

### `rumSamplingRate` (number, default: 10)
Controls RUM sampling for experiment/campaign/audience pages. Default is `10` (1-in-10 page views). Minimum is `10` — you cannot set a more aggressive rate.

### `storage` (Storage, default: `window.sessionStorage`)
The storage backend for persisting variant assignment across page navigations. Use `window.localStorage` if you want variant stickiness to survive browser restarts.

### Meta tag prefixes
- `experimentsMetaTag` (default: `'experiment'`) — The meta tag name for experiment ID
- `audiencesMetaTagPrefix` (default: `'audience'`) — Prefix for audience meta tags
- `campaignsMetaTagPrefix` (default: `'campaign'`) — Prefix for campaign meta tags

### Query parameter overrides
- `experimentsQueryParameter` (default: `'experiment'`) — Forces a specific experiment/variant via `?experiment=id/variant`
- `audiencesQueryParameter` (default: `'audience'`) — Forces a specific audience via `?audience=mobile`
- `campaignsQueryParameter` (default: `'campaign'`) — Forces a campaign via `?campaign=xmas` (also supports `?utm_campaign=`)

---

## How Experiments Work (for agent context)

### Metadata-driven approach

Experiments require **no code changes per experiment**. Authors configure everything in the page's metadata block in their document:

**Basic experiment (instant/inline):**
```
| Metadata            | Value                              |
|---------------------|------------------------------------|
| Experiment          | hero-test                          |
| Experiment Variants | /variant-page-1, /variant-page-2   |
```

The page containing this metadata is the "control." The plugin randomly assigns visitors to control or one of the challenger variants.

**Custom traffic split:**
```
| Metadata            | Value             |
|---------------------|-------------------|
| Experiment          | hero-test         |
| Experiment Variants | /variant-1        |
| Experiment Split    | 30                |
```
This gives 30% to variant-1, 70% to control. Splits are percentages for challengers; the remainder goes to control.

**Code-only experiment (no content swap):**
```
| Metadata            | Value     |
|---------------------|-----------|
| Experiment          | hero-test |
| Experiment Variants | 2         |
```
This creates 2 challenger variants without swapping content. Developers use CSS classes on `<body>` to differentiate.

### CSS classes applied to `<body>`

When an experiment runs, two CSS classes are added:
- `experiment-{experiment-id}` (e.g., `experiment-hero-test`)
- `variant-{variant-name}` (e.g., `variant-control` or `variant-challenger-1`)

When audiences resolve: `audience-{audience-name}` (e.g., `audience-mobile`)

When campaigns resolve: `campaign-{campaign-name}` (e.g., `campaign-xmas`)

### Additional experiment metadata

| Meta tag                     | Purpose                                           |
|------------------------------|---------------------------------------------------|
| `Experiment`                 | Experiment ID (required)                          |
| `Experiment Variants`        | Comma-separated variant URLs or a number          |
| `Experiment Split`           | Comma-separated percentages for challengers       |
| `Experiment Audience`        | Comma-separated audience names (OR logic)         |
| `Experiment Status`          | `Active` (default), `Inactive`, `On`, `Off`, etc. |
| `Experiment Start Date`      | JS Date string, e.g. `2024-01-01`                 |
| `Experiment End Date`        | JS Date string, e.g. `2024-03-31`                 |
| `Experiment Conversion Name` | Custom conversion event name (default: `click`)   |

### Audience metadata

```
| Metadata          | Value                   |
|-------------------|-------------------------|
| Audience: Mobile  | /my-page-for-mobile     |
| Audience: Desktop | /my-page-for-desktop    |
```

Each `Audience: {name}` meta tag maps an audience to a variant page URL.

### Campaign metadata

```
| Metadata            | Value                     |
|---------------------|---------------------------|
| Campaign: Xmas      | /my-page-for-xmas         |
| Campaign: Halloween | /my-page-for-halloween    |
| Campaign Audience   | mobile, iphone            |
```

Campaigns are triggered by `?campaign=xmas` or `?utm_campaign=xmas` query parameters. The optional `Campaign Audience` restricts the campaign to specific audiences.

### Full experiment manifests

For complex experiments with multiple pages/blocks, authors can use a manifest spreadsheet at `/experiments/{id}/manifest.json`. The manifest has two sheets:
- **settings** — experiment-level config (name, audience, status, blocks)
- **experiences** — variant definitions with percentage splits, page URLs, and block overrides

The plugin fetches and parses this manifest automatically when the `Experiment` meta tag contains an ID that doesn't include variant URLs.

### Variant assignment and stickiness

The plugin uses a Unified Experimentation Decisioning (UED) engine (`src/ued.js`) that:
1. Hashes the experiment ID with a device identifier using MurmurHash3
2. Maps the hash to a bucket (0–10000) and selects a treatment based on allocation percentages
3. Persists the assignment in `sessionStorage` (configurable via `storage` option) for 30 days so returning visitors see the same variant

### Execution order

The `loadEager` function runs features in this priority order:
1. **Campaigns** — checked first (requires query parameter)
2. **Experiments** — checked second (requires `experiment` meta tag)
3. **Audiences** — checked last (requires `audience:*` meta tags)

Only one feature can swap content per page load. If a campaign matches, experiments and audiences are skipped.

---

## Important Notes

- **Do not modify plugin files.** All files under `plugins/experimentation/` are managed upstream. Only modify project files in `scripts/`, `head.html`, and similar project-level locations.
- **All audience functions are developer-defined.** The plugin ships with zero built-in audiences. The `AUDIENCES` map must be defined in project code.
- **The plugin conditionally imports.** The `if (getMetadata(...))` guard ensures the plugin JS is only loaded on pages that actually use experimentation features. Do not remove this guard.
- **Bot detection is built in.** The plugin checks `navigator.userAgent` for bot/crawler patterns and skips all experimentation for bots.
- **Content swapping uses `fetch` + `DOMParser`.** When a variant page is served, the plugin fetches the variant URL, parses it safely with DOMParser (no script execution), and replaces `<main>` innerHTML. This means blocks in the swapped content will still be decorated normally by the project's `loadBlocks` flow.
- **`window.hlx.experiment`** is set globally when an experiment runs. It contains the full experiment config including `id`, `selectedVariant`, `status`, `variantNames`, `variants`, `resolvedAudiences`, and `run`. Similarly `window.hlx.campaign` and `window.hlx.audience` are set for their respective features.
- **RUM tracking** happens automatically via `sampleRUM()` calls for checkpoints `experiment`, `campaign`, and `audiences`.
- **Consent management:** If your project requires cookie consent, you can pass `storage: null` or a custom storage adapter. The default `sessionStorage` does not require consent in most jurisdictions since it doesn't persist beyond the session and doesn't identify users.
