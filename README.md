# c2c-plugins

Build plugins for [resources2crate](https://github.com/benfoley/resources2crate) —
split out of that repo's `src/plugins/` so a deployment can pick which
plugins it bundles instead of shipping all of them.

This package has **no runtime dependency on resources2crate**. Every plugin
here is a factory, `createPlugin(deps)`, that resources2crate calls with the
specific functions from its own `crate.js`/`fs_helpers.js`/`github.js`/
`masp.js` that plugin needs. That's what lets this repo be developed,
tested, and version-controlled independently, with no circular package
dependency between the two repos.

## Consuming this package

resources2crate depends on it as `"c2c-plugins": "file:../c2c-plugins"` (a
sibling checkout) and imports `REGISTRY`/`INPUT_REGISTRY` from `index.js`,
filtering them by its `PLUGINS` env var before calling each selected
factory. See resources2crate's `ARCHITECTURE.md` and `src/plugins/index.js`
for the consuming side.

## The two conventions every plugin here follows

**1. Hook names are literal strings, not an imported constant.** A plugin's
`hooks` object is keyed by strings like `"crate:built"` or `"output:write"`
rather than an imported `HOOKS.CRATE_BUILT` — those strings are a stable
contract owned by resources2crate's `src/plugins/hooks.js`:

| Hook | String |
|---|---|
| `FOLDER_PICKED` | `"folder:picked"` |
| `PROFILE_SELECTED` | `"profile:selected"` |
| `CONFIG_PREPARE` | `"config:prepare"` |
| `FILES_ANALYZE` | `"files:analyze"` |
| `CRATE_BUILT` | `"crate:built"` |
| `CRATE_VALIDATE` | `"crate:validate"` |
| `OUTPUT_WRITE` | `"output:write"` |

If resources2crate ever renames one of these, every plugin here keyed to the
old string silently stops firing — there's no import to break loudly. Grep
this repo for the old string when that happens.

**2. Every plugin module exports `createPlugin(deps)`**, not a static
`plugin` object. `deps` is the exact set of resources2crate core functions
that plugin needs, assigned into module-level bindings the plugin's hook
handlers close over. Call it once, before the plugin's hooks can fire.

## Per-plugin dependencies

| Plugin | `deps` keys it needs |
|---|---|
| `xlsx-crate-input` | `readFileBytes`, `readJsonFromFolder`, `loadMasp`, `statFile` (handed to `xlsx_crate.js`'s own `configure(deps)` on each dynamic import) |
| `austlang` | `addLanguageEntities` |
| `ca-data-prep` | `writeFileAtPath` |
| `merge` | `readJsonFromFolder`, `graphEntityById` |
| `validate-crate` | `loadMasp` |
| `ro-crate-json-output` | `crateToJsonString`, `writeFile`, `fileExists` |
| `ro-crate-xlsx-output` | `crateToXlsxBytes`, `writeFile`, `fileExists` |
| `ro-crate-html-output` | `crateToPreviewHtml`, `crateToMultiPageHtml`, `writeFile`, `writeFileAtPath`, `readJsonFromFolder`, `readFileTextFromDirectory`, `verifyPermission`, `fileExists`, `bustCacheUrl`, `buildGitHubTreeUrl`, `fetchGitHubTextFile`, `listGitHubFolder` |
| `generic-input` (input mode) | `buildFileMetadata`, `buildCrate` |
| `docx-input` (input mode) | `writeFileAtPath` (handed to `docx_crate.js`'s own `configure(deps)` once its dynamic import resolves) |

`loadMasp` is a thunk — `() => import("../masp.js")` — rather than the
function itself, so `ro-crate-masp` (a heavy validator library) stays
dynamically imported from resources2crate's own tree instead of becoming a
static import anywhere in this package.

## Writing a new plugin here

```js
// src/my-thing/index.js
let someCoreFn;

export function createPlugin(deps) {
  ({ someCoreFn } = deps);
  return plugin;
}

const plugin = {
  name: "my-thing",
  optionSchema: { key: "enableMyThing", label: "…", default: false },
  hooks: {
    "crate:built": (ctx) => { /* ... */ },
  },
};
```

Then register it in this repo's `index.js` (`REGISTRY` for an additive
plugin, `INPUT_REGISTRY` for a mutually-exclusive input mode), and in
resources2crate's `src/plugins/index.js`, wire up the `deps` object it's
called with.
