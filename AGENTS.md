# AGENTS.md

## Repository Overview
- Interactive Crosslink economics simulator delivered as a single static HTML document (`index.html`).
- Uses vanilla JavaScript, inline CSS, and [D3](https://d3js.org/) + `d3-sankey` loaded from CDN for the Sankey diagrams.
- No package.json, build tooling, or framework runtime; everything must continue to work when served as plain static assets.

## File Layout
- `index.html`: Entry point containing markup, styles, and the full application script, including chart rendering and UI logic.
- `.github/workflows/deploy.yml`: GitHub Pages deployment pipeline that uploads `index.html` on pushes to `main`.
- `AGENTS.md`: This guidance file. Update this first when repo assumptions change.

## Tooling & Runtime Expectations
- Development and preview: serve `index.html` over HTTP (e.g. `python -m http.server 5173`) so history/clipboard APIs work consistently. Opening via `file://` mostly works but may block clipboard access in some browsers.
- External dependencies are fetched at runtime from jsDelivr:
  - `https://cdn.jsdelivr.net/npm/d3@7`
  - `https://cdn.jsdelivr.net/npm/d3-sankey@0.12`
  Keep the app lightweight by avoiding additional libraries unless absolutely necessary.
- There is no automated lint/test pipeline. Manual testing in modern Chromium and Firefox is expected before shipping changes.

## Domain Model & Parameters
- Fixed economic inputs (matching code defaults):
  - `blockSubsidy = 1.5625` ZEC per block.
  - `blocksPerDay = 1152` (≈75s block time).
  - `shareDev = 0.20`, `shareMiners = 0.40`, `shareStakers = 0.40`.
- Adjustable controls (state keys → UI inputs):
  - `scaleMode`: `"per-block" | "per-day"` (select).
  - `finalizerSelectionProbability`: numeric input (0–1).
  - `commissionPct`: numeric input (0–100).
  - `exampleDelegatorPct`: numeric input (0–100).
- Defaults (`defaults` object in `index.html`): per-block scale, selection probability 0.01 (1%), 10% commission, 2% delegator share.
- Output metrics derive from these values and feed both the number cards and Sankey diagrams.

## Application Architecture
- Simple state container (`state`) backed by helper functions:
  - `propagateStateChange()` recalculates derived values, updates DOM text fields, re-renders Sankey charts, and syncs the URL query string.
  - `deriveValues()` computes all intermediate metrics used by the UI.
  - `renderSankey()` and `drawSankey()` construct diagrams using D3's Sankey utilities with consistent color palette defined near the top of the script.
- Event handling:
  - Inputs use `attachNumberInputHandlers` to clamp values, defer parsing while the user types, and trigger re-rendering.
  - `scale-mode` select toggles scale and refreshes outputs.
  - `reset-button` restores `defaults`. `copy-link-button` copies the current URL (with scenario parameters) to the clipboard, falling back to `document.execCommand` when needed.
  - `window.resize` triggers a redraw to keep SVG dimensions aligned with the container.
- Formatting helpers rely on `Intl.NumberFormat` instances; keep locale-agnostic formatting unless a new requirement demands otherwise.

## URL Parameters & Deep Linking
- Query params share state for bookmarking/sharing:
  - `p`: selection probability (`finalizerSelectionProbability`, 0–1).
  - `c`: commission percent (`commissionPct`, 0–100).
  - `d`: example delegator percent (`exampleDelegatorPct`, 0–100).
  - `scale`: `block` → per-block, `day` → per-day.
- `applyStateFromQuery()` hydrates state on load; `syncURL()` writes back when state changes. When modifying state keys, update both functions and keep parameter names stable to preserve compatibility.

## Styling & Accessibility Notes
- Dark-first palette defined in CSS `:root`; automatically adjusts label halos for light mode via media queries.
- Layout uses responsive CSS grid. Maintain existing breakpoints unless redesigning the layout.
- Sankey labels render with background rectangles for contrast; when adding nodes, ensure labels remain readable.
- Keep controls keyboard-accessible and consider ARIA attributes already present (e.g., `aria-describedby` for tooltips). Update any related text if new inputs are introduced.

## Deployment & Continuous Delivery
- GitHub workflow `.github/workflows/deploy.yml`:
  1. Runs on pushes to `main`.
  2. Checks out the repo.
  3. Uploads `index.html` as the Pages artifact via `actions/upload-pages-artifact@v3`.
  4. Deploys with `actions/deploy-pages@v4` to the `github-pages` environment.
- Renaming or relocating `index.html` requires updating the workflow `path` to keep deployments working.
- No other CI checks currently run; consider adding linting/tests before expanding the workflow.

## Contribution Guidelines
- Stay within the static-single-file architecture unless stakeholders approve a larger rework. If you must split files, update the workflow and this document.
- Prefer small, well-commented helper functions over introducing frameworks. Keep functions idempotent so they can be reused across multiple UI updates.
- Maintain consistent formatting (2-space indentation inside `<script>`/CSS blocks) and avoid non-ASCII characters.
- When adjusting economic assumptions, update both the constants in `index.html` and the documentation above.
- Document any new controls, outputs, or derived metrics in this file so future agents understand the data flow.

## Testing & Verification Checklist
- Manually test form inputs at boundary values (0, 1, 100) to confirm clamping logic.
- Validate both scale modes and ensure Sankey diagrams redraw correctly after window resizes.
- Verify URL sharing:
  1. Adjust parameters.
  2. Click "Copy link to scenario" (ensure clipboard success message appears).
  3. Open the copied link in a fresh tab to confirm state hydration.
- Confirm GitHub Pages deployment still references the correct asset path after structural changes (run the workflow in a PR if necessary).

## Known Limitations / Future Work
- No automated tests; behavior regression checks rely on manual QA.
- Visualization depends on external CDNs; offline use requires bundling assets locally.
- Economic model excludes transaction fees, slashing, privacy batching cadence, and other advanced tokenomics (see TODOs in project description). Add new sections here as features are implemented.
