# AGENTS.md

## Repository Overview
- Crosslink economics experience delivered as static assets under `public/` with no build tooling.
- `index.html` hosts the interactive simulator; `issuance-explorer.html` provides complementary issuance visualizations.
- Shared styling now lives in `style.css`; both pages remain framework-free and work when served as plain static files.
- UI still focuses on the configuration form, textual summary, and light-weight visuals (estimates panel remains removed).

## File Layout
- `public/index.html`: Simulator entry point containing markup and the application script for state, derived metrics, and UI logic.
- `public/issuance-explorer.html`: Static issuance/Sankey overview that reuses shared styling.
- `public/style.css`: Shared stylesheet for both pages; keep definitions in sync if structural changes occur.
- `.github/workflows/deploy.yml`: GitHub Pages deployment pipeline that uploads the `public/` directory on pushes to `main`.
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
  - `pctShieldedStaked`: numeric input (0–100).
  - `commissionPct`: numeric input (0–100).
- `delegatorZec`: dropdown selecting quantized delegation amounts (0.1 → 100,000 ZEC).
- Defaults (`DEFAULTS` object in `index.html`): 3,000,000 ZEC total shielded, 10% commission, 10 ZEC delegation, 50% of the shielded pool staked.
- Derived relationships powering the summary copy:
  - `baselineStakedZec = totalShieldedZec × (pctShieldedStaked / 100)`
  - `totalStakedZec = baselineStakedZec + delegatorZec`
  - `delegatorShare = totalStakedZec > 0 ? clamp(delegatorZec / totalStakedZec, 0, 1) : 0`
  - `netFactor = 1 - commissionPct / 100`
  - `rewardPerRound = roundReward × shareStakers`
  - `rewardPerDay = rewardPerRound × roundsPerDay`
  - `perDayZec = rewardPerDay × delegatorShare × netFactor`
  - `perYearZec = perDayZec × 365`
  - `annualizedPct = delegatorZec > 0 ? (perYearZec / delegatorZec) × 100 : 0`

## Application Architecture
- Simple state container (`state`) backed by helper functions:
  - `derive(state)` centralizes domain math, including shielded staking assumptions and net reward projections.
  - `propagateStateChange()` handles recalculation, DOM updates, optional input syncing, and URL serialization.
- Event handling:
  - Range and number inputs bind directly to `updateState` so clamping happens centrally before re-render.
  - Delegation is selected from a dropdown of quantized ZEC amounts (0.1 → 100,000); `snapDelegationValue` keeps query params and state aligned with the allowed steps.
  - `reset-button` restores `DEFAULTS`. `copy-link-button` copies the current URL (with scenario parameters) to the clipboard, falling back to `document.execCommand` when needed.
  - When the delegator amount meets or exceeds the baseline staked pool, `derive` clamps the share to 100% and `render` unhides the inline coverage note under the delegation input.
- Formatting helpers rely on `Intl.NumberFormat` instances; keep locale-agnostic formatting unless a new requirement demands otherwise.
- Navigation bar links the simulator and issuance explorer. Set `aria-current="page"` on the active link when adding new pages.

## Issuance Explorer Page
- Purely presentational SVG Sankey diagrams—no extra scripts besides the shared stylesheet.
- Update the gradients or layout directly in the markup when reward assumptions change.
- Shares typography and panel styles with the simulator; keep `style.css` coherent when tweaking visuals.

## URL Parameters & Deep Linking
- Query params share state for bookmarking/sharing:
  - `ps`: percent of shielded pool staked (`pctShieldedStaked`, 0–100).
  - `c`: commission percent (`commissionPct`, 0–100).
  - `dz`: delegator amount in ZEC (`delegatorZec`, ≥0).
- Legacy `d` share parameter has been removed; only `dz` is supported going forward.
  - `applyStateFromQuery()` hydrates state on load using the `dz` amount when present, falling back to defaults otherwise.
  - `syncURL()` writes `dz` when state changes.

## Styling & Accessibility Notes
- Dark-first palette defined in CSS `:root`.
- Layout uses responsive CSS grid. Maintain existing breakpoints unless redesigning the layout.
- Keep controls keyboard-accessible and consider ARIA attributes already present (e.g., `aria-describedby` for tooltips). Update any related text if new inputs are introduced.

## Deployment & Continuous Delivery
- GitHub workflow `.github/workflows/deploy.yml`:
  1. Runs on pushes to `main`.
  2. Checks out the repo.
  3. Uploads the `public` directory as the Pages artifact via `actions/upload-pages-artifact@v3`.
  4. Deploys with `actions/deploy-pages@v4` to the `github-pages` environment.
- Renaming or relocating assets requires updating the workflow `path` to keep deployments working.
- No other CI checks currently run; consider adding linting/tests before expanding the workflow.

## Contribution Guidelines
- Keep the experience within lightweight static assets; if new pages or bundles are introduced, update the workflow and this document accordingly.
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
- Visit `issuance-explorer.html` via the navigation bar and confirm Sankey visuals render correctly.
- Confirm GitHub Pages deployment still references the correct asset path after structural changes (run the workflow in a PR if necessary).

## Known Limitations / Future Work
- No automated tests; behavior regression checks rely on manual QA.
- Economic model excludes transaction fees, slashing, privacy batching cadence, and other advanced tokenomics (see TODOs in project description). Add new sections here as features are implemented.
