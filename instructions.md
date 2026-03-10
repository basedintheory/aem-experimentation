# AEM Experimentation Plugin — Integration Instructions

## Overview

The AEM Experimentation plugin provides three content personalization features for AEM Edge Delivery Services sites:

1. **A/B Testing (Experiments)** — Randomly serve different content variants to visitors and measure conversion. Variants are defined via page metadata; no per-experiment code changes are needed.
2. **Audience Targeting** — Serve different content to different user segments (e.g., mobile vs. desktop, geo-based). Audience resolution functions are defined in project code; content mapping is done via metadata.
3. **Campaign Personalization** — Serve campaign-specific content when visitors arrive via campaign URLs (`?campaign=xmas` or `?utm_campaign=xmas`). Content mapping is done via metadata.

All three features are metadata-driven. Authors configure experiments, audiences, and campaigns in their page's metadata block in the document. The plugin reads these meta tags at runtime, resolves the appropriate variant, and swaps content before the page renders.

The plugin files are pre-installed at `plugins/experimentation/`. Do **not** modify any files inside that directory. You will only add or modify files in `scripts/` and `head.html`.

---

## Key Files to Add or Modify

1. **plugins/experimentation** — Already installed via git subtree. Do not modify.
2. **scripts/experiment-loader.js** — New file. Handles experimentation detection and eager loading.
3. **scripts/scripts.js** — Modify to add configuration and call the experiment loader.
4. **head.html** — Modify to add modulepreload hint and optional RUM sampling override.

---

## Plugin API

The plugin entry point is `plugins/experimentation/src/index.js`. It exports:

### `loadEager(document, config)`
Main initialization function. Called via the experiment loader during the eager phase of page loading (before LCP). This function:
- Adjusts RUM sampling rates for experiment/campaign/audience pages
- Runs campaign detection (checks `?campaign=` or `?utm_campaign=` query params)
- If no campaign matched, runs experiment detection (checks `experiment` meta tag)
- If no experiment matched, runs audience targeting (checks `audience:*` meta tags)
- Swaps page content with the resolved variant if applicable

### Config object

```js
{
  // Production host — controls debug overlay visibility.
  // The preview pill overlay is hidden when the current host matches this value.
  prodHost: 'www.example.com',

  // Alternative: a function that returns true on prod environments
  // isProd: () => !window.location.hostname.endsWith('hlx.page')
  //   && window.location.hostname !== 'localhost',

  // the storage type used to persist data between page views
  // (for instance to remember what variant in an experiment the user was served)
  // storage: window.sessionStorage,

  // Audience resolver map. Keys are audience names (kebab-case),
  // values are functions returning boolean.
  audiences: {},

  // Meta tag prefixes (defaults shown):
  audiencesMetaTagPrefix: 'audience',
  audiencesQueryParameter: 'audience',
  campaignsMetaTagPrefix: 'campaign',
  campaignsQueryParameter: 'campaign',
  experimentsMetaTagPrefix: 'experiment',
  experimentsQueryParameter: 'experiment',

  // Fragment experiment redecoration function (optional):
  // decorationFunction: (el) => { buildBlock(el); decorateBlock(el); },
}
```

---

## Integration Steps

### Step 1: Create `scripts/experiment-loader.js`

Create a new file `scripts/experiment-loader.js` with the following content:

```js
/**
 * Checks if experimentation is enabled.
 * @returns {boolean} True if experimentation is enabled, false otherwise.
 */
const isExperimentationEnabled = () => document.head.querySelector('[name^="experiment"],[name^="campaign-"],[name^="audience-"],[property^="campaign:"],[property^="audience:"]')
  || [...document.querySelectorAll('.section-metadata div')].some((d) => d.textContent.match(/Experiment|Campaign|Audience/i));

/**
 * Loads the experimentation module (eager).
 * @param {Document} document The document object.
 * @param {Object} config The experimentation configuration.
 * @returns {Promise<void>} A promise that resolves when the experimentation module is loaded.
 */
export async function runExperimentation(document, config) {
  if (!isExperimentationEnabled()) {
    window.addEventListener('message', async (event) => {
      if (event.data?.type === 'hlx:experimentation-get-config') {
        event.source.postMessage({
          type: 'hlx:experimentation-config',
          config: { experiments: [], audiences: [], campaigns: [] },
          source: 'no-experiments'
        }, '*');
      }
    });
    return null;
  }

  try {
    const { loadEager } = await import(
      '../plugins/experimentation/src/index.js'
    );
    return loadEager(document, config);
  } catch (error) {
    // eslint-disable-next-line no-console
    console.error('Failed to load experimentation module (eager):', error);
    return null;
  }
}
```

This file:
- Detects experimentation metadata via DOM queries (meta tags and section-metadata divs)
- Only imports the plugin when experimentation is actually enabled on the page
- Handles the sidekick/overlay communication when no experiments are present
- Wraps the import in a try/catch for resilience

### Step 2: Update `scripts/scripts.js`

Add the following import and configuration at the top of `scripts/scripts.js`:

```js
import {
  runExperimentation,
} from './experiment-loader.js';

const experimentationConfig = {
  prodHost: 'PUT_PROD_HOSTNAME_HERE',
  audiences: {
    mobile: () => window.innerWidth < 600,
    desktop: () => window.innerWidth >= 600,
    // define your custom audiences here as needed
  },
};
```

Replace `'PUT_PROD_HOSTNAME_HERE'` with the project's production hostname (e.g., `'www.mysite.com'`). If you cannot determine the production hostname, leave it as an empty string `''` — the preview overlay will show on all environments.

Then, add the following line early in your `loadEager()` function — before `loadHeader`, `loadFooter`, and `decorateBlocks` calls. The plugin must run before LCP because it may swap page content:

```js
async function loadEager(doc) {
  // ... existing early setup (decorateTemplatesAndThemes, etc.) ...

  await runExperimentation(doc, experimentationConfig);

  // ... existing loadHeader, decorateBlocks, waitForFirstImage, etc. ...
}
```

> **Note:** Unlike v1, there is no separate `loadLazy` call needed. The experiment loader and plugin handle both eager and lazy phases internally.

### Step 3: Add modulepreload to `head.html`

Add a modulepreload link hint for the experiment loader in `head.html`. This tells the browser to start fetching the module early, reducing latency when the eager phase imports it:

```html
<link rel="modulepreload" href="/scripts/experiment-loader.js" />
```

Add this line alongside the existing modulepreload links (typically after the `scripts/scripts.js` preload).

---

## Increasing RUM Sampling Rate for Low-Traffic Pages

When running experiments during short periods (a few days to 2 weeks) or on low-traffic pages (<100K page views a month), the default RUM sampling (1 out of 100) is unlikely to reach statistical significance. For those use cases, increase the sampling rate to 1 out of 10.

Edit your `head.html` and add this inline script **before** the `aem.js` or `lib-franklin.js` script load:

```html
<!-- insert this script tag before loading aem.js or lib-franklin.js -->
<script>
  window.RUM_SAMPLING_RATE = document.head.querySelector('[name^="experiment"],[name^="campaign-"],[name^="audience-"]')
    || [...document.querySelectorAll('.section-metadata div')].some((d) => d.textContent.match(/Experiment|Campaign|Audience/i))
    ? 10
    : 100;
</script>
<script type="module" src="/scripts/aem.js"></script>
<script type="module" src="/scripts/scripts.js"></script>
```

Then verify your `aem.js` file around line 20 has:
```js
const weight = new URLSearchParams(window.location.search).get('rum') === 'on' ? 1 : defaultSamplingRate;
```

If this line is not present, apply the changes from: https://github.com/adobe/helix-rum-js/pull/159/files

---

## Configuration Reference

### `prodHost` (string)
The production hostname (e.g., `'www.mysite.com'`). When the current page's host matches this value, the preview overlay pill is hidden.

### `isProd` (function, optional)
Alternative to `prodHost`. A function returning `true` when running on production. Use this for complex multi-domain setups:
```js
isProd: () => !window.location.hostname.endsWith('hlx.page')
  && window.location.hostname !== 'localhost',
```

### `audiences` (object)
A map of audience name strings to boolean-returning functions. Keys must be kebab-case and match the audience names used in metadata. **No audiences are built-in — all must be defined by the project.** Example:

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

### `storage` (Storage, default: `window.sessionStorage`)
The storage backend for persisting variant assignment across page navigations. Use `window.localStorage` if you want variant stickiness to survive browser restarts.

### Meta tag prefixes
- `experimentsMetaTagPrefix` (default: `'experiment'`) — Prefix for experiment meta tags
- `audiencesMetaTagPrefix` (default: `'audience'`) — Prefix for audience meta tags
- `campaignsMetaTagPrefix` (default: `'campaign'`) — Prefix for campaign meta tags

### Query parameter overrides
- `experimentsQueryParameter` (default: `'experiment'`) — Forces a specific experiment/variant via `?experiment=id/variant`
- `audiencesQueryParameter` (default: `'audience'`) — Forces a specific audience via `?audience=mobile`
- `campaignsQueryParameter` (default: `'campaign'`) — Forces a campaign via `?campaign=xmas` (also supports `?utm_campaign=`)

### `decorationFunction` (function, optional)
A function to redecorate DOM elements after fragment replacement. Fragment replacement is handled by an async observer which may execute before or after default decoration completes. Common cases:

1. Element inside a block needs redecoration: `(el) => { buildBlock(el); decorateBlock(el); }`
2. `.block` selector needs redecoration: switch block status to `"loading"` and call `loadBlock(el)`
3. `.section` selector needs redecoration: call `decorateBlocks(el)`
4. `main` selector needs redecoration: call `decorateMain(el)`

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
| `Experiment Requires Consent`| `true` to require user consent before running      |

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

## Extensibility & Integrations

The plugin exposes experiment data through two mechanisms:
1. **Events** — React immediately when experiments are applied
2. **Global Objects** — Access complete experiment details after page load

### Events

Listen for the `aem:experimentation` event to react when experiments, campaigns, or audiences are applied:

```js
document.addEventListener('aem:experimentation', (event) => {
  console.log(event.detail);
  // event.detail contains: { type, element, experiment/campaign/audience, variant }
});
```

### Global Objects

```js
const allExperiments = window.hlx.experiments; // array of page/section/fragment experiments
const allAudiences = window.hlx.audiences;     // array of resolved audiences
const allCampaigns = window.hlx.campaigns;     // array of resolved campaigns

// backward compatibility with V1
const experiment = window.hlx.experiment;
const audience = window.hlx.audience;
const campaign = window.hlx.campaign;
```

### Consent Management

The plugin provides consent management APIs. Import from the plugin:

```js
import {
  updateUserConsent,
  isUserConsentGiven,
} from '../plugins/experimentation/src/index.js';
```

Add `Experiment Requires Consent: true` metadata to experiments that need consent. Integrate your CMP (OneTrust, Cookiebot, or custom) to call `updateUserConsent(true/false)`. The recommended approach is to add consent initialization in `experiment-loader.js` before loading experimentation.

---

## Important Notes

- **Do not modify plugin files.** All files under `plugins/experimentation/` are managed upstream. Only modify project files in `scripts/`, `head.html`, and similar project-level locations.
- **All audience functions are developer-defined.** The plugin ships with zero built-in audiences. The `audiences` map must be defined in project code.
- **The experiment loader conditionally imports.** The `isExperimentationEnabled()` check ensures the plugin JS is only loaded on pages that actually use experimentation features. Do not remove this guard.
- **Bot detection is built in.** The plugin checks `navigator.userAgent` for bot/crawler patterns and skips all experimentation for bots.
- **Content swapping uses `fetch` + `DOMParser`.** When a variant page is served, the plugin fetches the variant URL, parses it safely with DOMParser (no script execution), and replaces `<main>` innerHTML. This means blocks in the swapped content will still be decorated normally by the project's `loadBlocks` flow.
- **`window.hlx.experiments`** is set globally as an array of all page/section/fragment experiments. Similarly `window.hlx.audiences` and `window.hlx.campaigns`. V1-compatible `window.hlx.experiment`, `window.hlx.audience`, and `window.hlx.campaign` are also set.
- **RUM tracking** happens automatically via `sampleRUM()` calls for checkpoints `experiment`, `campaign`, and `audiences`.
- **Consent management:** Use `updateUserConsent()` and `isUserConsentGiven()` APIs with your CMP. Experiments can require consent via metadata. The default `sessionStorage` does not require consent in most jurisdictions.
