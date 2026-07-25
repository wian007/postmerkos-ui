# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This repo adds a web UI to [meraki-builder](https://github.com/halmartin/meraki-builder) — custom firmware for Cisco
Meraki MS-series switches — as a single-page Preact app. The app runs entirely client-side and talks to a `configd`
CGI daemon on the switch itself (`/cgi-bin/config` and `/cgi-bin/status`) to read/write the switch's port
configuration and poll live status. There is no separate backend in this repo; auth is handled by the switch's PAM
login via `uhttpd`, not by the app.

## Commands

```
npm i               # install deps (node >=0.14 <=16; `nix develop` gives a matching devShell w/ nodejs-14)
npm run dev          # preact watch — live reload at http://localhost:8080, using local fixture data (no auth)
npm run build        # preact build --prerender, then copies ./assets/* into ./build/
npm run serve        # serve a production build with sirv on port 8080
npm run analyze      # preact build --analyze -p (bundle size analysis)
npm run lint         # eslint src (config: eslint-config-synacor)
npm run zip          # zip -r latest.zip build
```

There is no unit test suite/runner configured (no test script in package.json). `src/test/{8,24,48}/{config,status}`
are static JSON fixtures, not test code — see "Dev-mode data source" below.

## Dev-mode data source

`src/index.js` picks the API base at load time:

```js
let endpoint = "/cgi-bin";
if (!process.env.NODE_ENV || process.env.NODE_ENV === 'development') {
    endpoint = "/test/24";
}
```

In dev (`npm run dev`), config/status are fetched from `src/test/24/{config,status}` instead of a real switch. To test
against a different port-count layout, swap `24` for `8` or `48` (matching the fixture directories), which exist to
exercise the port-count-dependent grid layout logic in `ports.js`.

## Architecture

Single-file-per-component, all Preact class `Component`s (not functional components), rendered from one root:

```
index.js (App)
 ├─ uploadButton()      — inline render fn, posts state.config to /cgi-bin/config
 ├─ Legend (legend.js)  — info-icon modal explaining port-box colors/markers
 ├─ Ports (ports.js)    — visual grid of port boxes (read-only overview)
 │   └─ Port (port.js)  — single port box: color = link speed, dots = lacp/stp, corner = poe/vlan
 └─ Table (table.js)    — editable table, one row per port, inputs call updatePort(...)
     └─ Port (port.js)  — reused for the Legend's example icons
```

**State lives entirely in `App` (index.js)**, holding `config` (editable, in-memory), `configOnDisk` (last
loaded/uploaded config, used as the diff baseline), `status` (polled from the switch), and `diff`.

- `updatePort(portNumber, path, value)` — called by `Table` on every input change. Uses `lodash.set`/`cloneDeep` to
  immutably update `config.ports[portNumber]` at a dotted path (e.g. `"vlan.pvid"`, `"poe.mode"`), then recomputes
  `diff`.
- `updateDiff(config, configOnDisk)` — recursively diffs the two trees; result renders as `diff-background`/
  `diff-foreground` CSS classes in `Table`/`uploadButton` so unsaved changes are visually flagged before upload.
- Uploading POSTs the whole `config` JSON to `/cgi-bin/config`; success resets `configOnDisk` to the just-uploaded
  config and clears `diff`.
- `status` is polled every 3s via the custom `useInterval` hook (`hook.js`) inside `render()`.

**Data shapes** (see `src/test/24/config` and `.../status` for full examples):
- `config.ports["<n>"]`: `{ enabled, name, vlan: { pvid, allowed }, poe: { enabled, mode }, lacp, stp }`
- `status.ports["<n>"]`: `{ link: { established, speed }, poe: { power } }`, plus top-level `device`, `datetime`,
  `temperature: { cpu: [], poe: [] }`
- `Table`/`Ports` merge these per-port as `{ ...status.ports[n], ...config.ports[n] }` to render combined state.

**PoE detection**: `state.poe` is set in `App.componentDidMount` if `status.device` ends in `"P"` (Meraki's PoE model
suffix), and gates rendering of PoE-related columns/legend entries/badges throughout.

**Port layout heuristics** (`ports.js` constructor): grid columns/rows and assumed SFP-port count are *guessed* from
the total port count (`count > 10` → 2 rows and 4 SFP ports, else 1 row and 2 SFP ports; groups of 12). This is
brittle by design — it's tuned to known Meraki MS-series layouts (8/24/48 port variants), not derived from any
explicit "this is an SFP port" field in the data. If you touch this, check against all three test fixtures.

**Preact/React mixing**: this is a Preact-CLI app (aliases `react`/`react-dom` to `preact/compat`), but `index.js` and
`hook.js` import hooks (`useState`, `useEffect`, `useRef`) directly from `'react'` while using Preact `Component`
classes elsewhere — this is intentional/existing convention, not a bug to "fix" by switching imports.

## Release process

Pushing a git tag triggers `.github/workflows/main.yaml`, which:
1. Replaces the `{/* VERSION */} dev` placeholder comment in `src/index.js` with the tag name via `sed` (do not
   remove that comment — CI depends on its exact text to find the line).
2. Runs `npm run build`, zips `build/` as `<repo-name>.zip`.
3. Publishes it as a GitHub release asset.

The zip is what end users download and copy onto their switch (per README), so `npm run build` output must be
self-contained and served correctly by `uhttpd` from a plain directory (no server-side routing beyond static files).
