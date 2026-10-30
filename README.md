# Tigernet Runner — Public Wildfire Surveillance Dashboard & Automation

This public repository (`tigernet-runner`) hosts the Algeria Wildfire Live Intelligence Map frontend and the GitHub Actions automation workflow that coordinates satellite observations.

## Architecture & Security Boundary

- **Frontend (`web/`)**: Real-time interactive web dashboard rendering wildfire telemetry across Northern Algeria (lat 34.0°–37.5°N, lon -2.0°–8.5°E), with dynamic clustering, critical ignition pulse markers, and industrial site exclusion buffers.
- **Automation (`.github/workflows/monitor.yml`)**: Coordinates EUMETSAT satellite acquisition, deep learning inference, private state synchronization, and public live map publishing.
  > [!NOTE]
  > **Deployment Safety Precaution**: The 15-minute automated schedule (`cron: '*/15 * * * *'`) is temporarily disabled in `.github/workflows/monitor.yml` during initial setup. Surveillance cycles can be triggered manually via the **Actions** tab (`Run workflow`). The 15-minute schedule will be re-enabled after the first successful manual production test verifies secret authentication, checkpoint downloads, state synchronization, and Pages publishing.
- **Zero-Exposure Policy**:
  - Model weights (`.pth`) and inference source code are **never** committed to this repository.
  - Operational archives (`active_incidents.jsonl` and `industrial_hotspots.jsonl`) are **never** committed to this repository.
  - The workflow retrieves code and checkpoints ephemerally from `https://github.com/Kingramy/tigernet`, persists operational state back to a private `runtime-state` branch in `Kingramy/tigernet`, purges all temporary files, and commits **only** sanitized public GeoJSON and status summary data.

---

## Required GitHub Actions Secrets

To enable automated surveillance, configure the following secrets under **Settings -> Secrets and variables -> Actions** in the `tigernet-runner` repository:

| Secret Name | Exact Minimum Permissions | Description |
| :--- | :--- | :--- |
| `TIGERNET_PRIVATE_TOKEN` | Fine-grained PAT: **Repository: `Kingramy/tigernet` only**, `Contents: Read-only` | Clones private engine source code and downloads release assets from private release `model-v1`. |
| `TIGERNET_STATE_TOKEN` | Fine-grained PAT: **Repository: `Kingramy/tigernet` only**, `Contents: Read and write` | Retrieves and persists private operational state (`active_incidents.jsonl`, `industrial_hotspots.jsonl`, `temporal_state.npz`) on the private `runtime-state` branch. |
| `EUMETSAT_CONSUMER_KEY` | Plaintext API Key | EUMETSAT Data Store API Consumer Key for satellite data streaming. |
| `EUMETSAT_CONSUMER_SECRET` | Plaintext API Secret | EUMETSAT Data Store API Consumer Secret for satellite data streaming. |

---

## GitHub Pages Deployment Setup

Because the web application resides in the `web/` subdirectory, standard GitHub Pages branch publishing (which only supports the repository root `/` or `/docs`) cannot be used.

The repository uses **GitHub Actions-based deployment** via `actions/configure-pages@v5`, `actions/upload-pages-artifact@v4`, and `actions/deploy-pages@v4`.

### Required Configuration in GitHub Settings:

1. Navigate to your public `tigernet-runner` repository on GitHub.
2. Go to **Settings -> Pages**.
3. Under **Build and deployment -> Source**, select **GitHub Actions** (do NOT select *Deploy from a branch*).

### Deployment Characteristics:
- **Gated Deployment**: The `deploy` job executes only after the `surveillance` job succeeds (inference completes, state is persisted, workspace is sanitized, and public files are committed).
- **Subdirectory Isolation**: `upload-pages-artifact@v4` targets strictly `path: 'web'`, uploading only `index.html` and `data/` (`fires.geojson`, `latest_status.json`).
- **Permissions**: Minimum permissions required are `pages: write` and `id-token: write`, configured alongside `contents: write`.

---

## Initial Verification & Enabling 15-Minute Surveillance

Automated surveillance is designed to synchronize every 15 minutes with EUMETSAT MSG-15 RSS scanning intervals. For initial deployment safety, the cron schedule is temporarily disabled:

1. **Trigger First Manual Production Run**:
   - Navigate to the **Actions** tab in `tigernet-runner`.
   - Select **Tigernet Algeria Live Surveillance**.
   - Click **Run workflow** (branch: `main`, mode: `once`).
2. **Verify Execution Integrity**:
   - Validate that secrets authenticate without error.
   - Validate that the private engine and model checkpoints (`model-v1`) download into `_engine`.
   - Validate that operational state is cloned, updated, and pushed back to `Kingramy/tigernet:runtime-state`.
   - Validate that the runner workspace is sanitized (`rm -rf _engine _state *.pth *.jsonl *.npz`).
   - Validate that GitHub Pages deploys the updated `web/` dashboard.
3. **Re-Enable Automated Schedule**:
   - In `.github/workflows/monitor.yml`, uncomment the schedule block:
     ```yaml
     on:
       schedule:
         - cron: '*/15 * * * *'
       workflow_dispatch:
     ```
   - Commit and push this change to `main`. Continuous 15-minute observations will commence immediately.

---

## Workflow Concurrency & Execution Behavior

The surveillance workflow defines the concurrency group:
```yaml
concurrency:
  group: tigernet-surveillance
  cancel-in-progress: false
```

### GitHub Actions Concurrency Semantics:
- **Execution Limits**: Within a concurrency group, **at most one run executes** and **at most one pending run waits in the queue**.
- **Run Coalescing**: If multiple scheduled runs trigger while a run is in progress, GitHub Actions does **not** queue every scheduled run. Instead, a **newer pending run replaces (coalesces) the older pending run**.
- **Temporal Stale-State Safety Behavior**:
  If a cycle is delayed or skipped such that the elapsed time between satellite scenes exceeds 25 minutes ($> 25.0$ min):
  - `TemporalFireTracker` automatically detects `time_delta_mins > 25.0`.
  - It safely treats the current scene as a fresh baseline without applying an invalid temporal rate-of-rise multiplier.
  - This prevents stale $\Delta T$ grids from creating false positive alarms or inflated confidence values.
  - Spatial U-Net detection and industrial hotspot exclusion remain fully functional and protective.

---

## Public Data Files

- `web/data/fires.geojson`: Geospatial FeatureCollection consumed by the Leaflet dashboard to render active wildfire points. Contains only public-safe attributes (`timestamp`, `confidence`, `regime`, `brightness_temp_kelvin`, `fire_type`).
- `web/data/latest_status.json`: Live operational telemetry summary (active regime, last sensing UTC, feature counts).


---

## Copyright and Proprietary Software

Copyright (c) 2026 Ouail Chenafi. All rights reserved.

- **Proprietary Software**: Tigernet is proprietary software. All rights in the original source code, documentation, and original project materials are reserved by the copyright holder.
- **Copyright Holder**: Ouail Chenafi.
- **No General Open-Source License**: No general open-source license (such as MIT, Apache, or GPL) is granted for this repository, source code, or associated assets.
- **Commercial Reuse Restrictions**: Use, copying, modification, distribution, sublicensing, or commercial exploitation requires express prior written agreement from the copyright holder.
- **Third-Party Components**: Third-party components and libraries remain subject to their respective licenses.
- **GitHub Platform Terms**: This notice does not remove permissions users receive under GitHub's Terms of Service, including viewing and forking public repositories through GitHub.
