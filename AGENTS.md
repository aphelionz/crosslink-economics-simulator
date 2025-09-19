# AGENTS.md

## Repository Overview
- Interactive Crosslink economics simulator delivered as a single static HTML document (`public/index.html`).
- Uses vanilla JavaScript and inline CSS with no external runtime dependencies.
- No package.json, build tooling, or framework runtime; everything must continue to work when served as plain static assets.
- UI currently includes the configuration form and textual summary only (Estimates and detailed metrics panels were removed).

## File Layout
- `public/index.html`: Entry point containing markup, styles, and the full application script for state, derived metrics, and UI logic.
- `.github/workflows/deploy.yml`: GitHub Pages deployment pipeline that uploads `index.html` on pushes to `main`.
- `AGENTS.md`: This guidance file. Update this first when repo assumptions change.

## Tooling & Runtime Expectations
- Development and preview: serve `index.html` over HTTP (e.g. `python -m http.server 5173`) so history/clipboard APIs work consistently. Opening via `file://` mostly works but may block clipboard access in some browsers.
- External dependencies are not required; keep the app lightweight by avoiding additional libraries unless absolutely necessary.
- There is no automated lint/test pipeline. Manual testing in modern Chromium and Firefox is expected before shipping changes.

## Domain Model & Parameters
- Fixed economic inputs (matching code defaults):
  - `roundReward = 1.5625` ZEC per round.
  - `roundsPerDay = 1152` (≈75s round cadence).
  - `shareStakers = 0.40` of each round's reward is assumed to flow to stakers (other protocol splits are informational only in this UI).
- Adjustable controls (state keys → UI inputs):
  - `finalizerWeight`: numeric input (0–1).
  - `pctShieldedStaked`: numeric input (0–100).
  - `commissionPct`: numeric input (0–100).
  - `delegatorZec`: numeric input (≥0) representing the delegator's stake in ZEC.
- Defaults (`DEFAULTS` object in `index.html`): 3,000,000 ZEC total shielded, selection probability 0.01 (1%), 10% commission, 60 ZEC delegation (≈2% of the default finalizer stake), 10% of shielded pool staked.
- Derived relationships powering the summary copy:
  - `totalStakedZec = totalShieldedZec × (pctShieldedStaked / 100)`
  - `finalizerStakeZec = totalStakedZec × finalizerWeight`
  - `userShareOfFinalizer = finalizerStakeZec > 0 ? clamp(delegatorZec / finalizerStakeZec, 0, 1) : 0`
  - `netFactor = 1 - commissionPct / 100`
  - `rewardPerPick = roundReward × shareStakers`
  - `expectedPicksPerDay = finalizerWeight × roundsPerDay`
  - `perPickZec = rewardPerPick × userShareOfFinalizer × netFactor`
  - `perDayZec = expectedPicksPerDay × rewardPerPick × userShareOfFinalizer × netFactor`
  - `perYearZec = perDayZec × 365`
  - `annualizedPct = delegatorZec > 0 ? (perDayZec × 365 / delegatorZec) × 100 : 0`

## Application Architecture
- Simple state container (`state`) backed by helper functions:
  - `derive(state)` centralizes domain math, including shielded staking assumptions and net reward projections.
  - `propagateStateChange()` handles recalculation, DOM updates, optional input syncing, and URL serialization.
- Event handling:
  - Inputs use `attachNumberInputHandlers` to clamp values, defer parsing while the user types, and trigger re-rendering.
  - `reset-button` restores `DEFAULTS`. `copy-link-button` copies the current URL (with scenario parameters) to the clipboard, falling back to `document.execCommand` when needed.
  - When the delegator amount meets or exceeds the finalizer stake, `derive` clamps the share to 100% and `render` unhides the inline coverage note under the delegation input.
- Formatting helpers rely on `Intl.NumberFormat` instances; keep locale-agnostic formatting unless a new requirement demands otherwise.

## URL Parameters & Deep Linking
- Query params share state for bookmarking/sharing:
  - `p`: selection probability (`finalizerWeight`, 0–1).
  - `ps`: percent of shielded pool staked (`pctShieldedStaked`, 0–100).
  - `c`: commission percent (`commissionPct`, 0–100).
  - `dz`: delegator amount in ZEC (`delegatorZec`, ≥0).
  - `d`: delegator share percent (legacy; still emitted for compatibility, but ignored when `dz` is present).
- `applyStateFromQuery()` hydrates state on load; it prefers `dz` and converts legacy `d` values into ZEC using the current `finalizerWeight` and `pctShieldedStaked`. `syncURL()` writes back when state changes and includes both `dz` (canonical amount) and `d` (derived share percent) so older links remain interpretable.

## Styling & Accessibility Notes
- Dark-first palette defined in CSS `:root`.
- Layout uses responsive CSS grid. Maintain existing breakpoints unless redesigning the layout.
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
- Confirm the summary copy updates when any input changes.
- Verify URL sharing:
  1. Adjust parameters.
  2. Click "Copy link to scenario" (ensure clipboard success message appears).
  3. Open the copied link in a fresh tab to confirm state hydration.
- Confirm GitHub Pages deployment still references the correct asset path after structural changes (run the workflow in a PR if necessary).

## Known Limitations / Future Work
- No automated tests; behavior regression checks rely on manual QA.
- Economic model excludes transaction fees, slashing, privacy batching cadence, and other advanced tokenomics (see TODOs in project description). Add new sections here as features are implemented.
