# Haunt Patrol

Haunt Patrol is a [Kiro Crew](https://kiro.dev/docs/crew/) app that runs a nightly accessibility
patrol with [Haunt](https://github.com/xan-antx/haunt). Every night at midnight IST it builds and
serves Lantern Bills (Haunt's demo bill-payment site) on port 3000, sends three ghosts through the
pay-bill mission, stops the site, and writes a record and a plain-text summary that compares the
night with the one before. A Crew page shows the latest patrol: the summary, and for each ghost its
result, its finding counts and the findings themselves.

The three ghosts are Haunt's simulated users:

- **The Listener** moves through a page with only the keyboard and listens to what a screen reader
  would announce.
- **The Phantom** browses on slow 3G and an old phone.
- **Bebe** reads at 200% zoom, prefers Punjabi or Hindi, and watches out for tricky wording.

Each ghost runs with Haunt's scripted mind, so no AI model is called. Ghosts only visit the local
copy of Lantern Bills that the patrol starts.

## How it works with the Haunt repository

This repository holds only the Crew app. The ghosts, the browser engine and Lantern Bills live in
the main Haunt repository, [xan-antx/haunt](https://github.com/xan-antx/haunt), and Haunt Patrol
drives them from there, so you need a clone of both.

The app finds the Haunt clone through the `HAUNT_REPO_ROOT` environment variable, which must name
the folder that holds `engine\` and `demo-site\`. Set it for your Windows user: Crew runs an
installed copy of this app from `%USERPROFILE%\.kiro\crew\apps\haunt-patrol`, which has no Haunt
repository next to it. Without the variable the app looks in the folder above itself, which only
works for the copy of this app in the Haunt repository's `crew-app\` folder.

Each night the patrol:

1. checks the prerequisites in the Haunt clone (Node.js 24, the engine's and Lantern Bills'
   dependencies, Playwright's Chromium),
2. builds Lantern Bills into `<HAUNT_REPO_ROOT>\crew-app\.patrol\site-build` and serves it on port
   3000,
3. runs the Haunt engine for each ghost in turn, saving each run in `data\runs\` inside the app
   folder,
4. stops the site and writes the patrol record and `summary.md` to `data\patrols\`, with a log of
   every step in `data\logs\patrol.log`.

Apart from that build folder, which the Haunt repository ignores, the Haunt clone is not changed. In
particular, the patrol never writes into the clone's `runs\` folder, which the dashboard shows. The
runner refuses to start if it is pointed there.

## Install it in Kiro Crew on Windows

You need Windows 11, the [Kiro Crew desktop app](https://kiro.dev/docs/crew/installation/),
[Node.js 24 LTS](https://nodejs.org) or later and [Git for Windows](https://git-scm.com/download/win).
The commands are for Windows PowerShell 5.1 and also work in PowerShell 7. The examples clone into
`C:\Projects`; use any folder you like.

1. Clone the Haunt repository and install what the patrol needs:

   ```powershell
   git clone https://github.com/xan-antx/haunt.git C:\Projects\haunt
   Set-Location C:\Projects\haunt\engine
   npm install
   npx playwright install chromium
   Set-Location ..\demo-site
   npm install
   ```

2. Clone this repository and install its dependencies:

   ```powershell
   git clone https://github.com/xan-antx/haunt-crew.git C:\Projects\haunt-crew
   Set-Location C:\Projects\haunt-crew
   npm install
   ```

   The page bundle `ui\dist\index.mjs` is committed, so there is no build step.

3. Tell Haunt Patrol where the Haunt clone is:

   ```powershell
   [Environment]::SetEnvironmentVariable('HAUNT_REPO_ROOT', 'C:\Projects\haunt', 'User')
   ```

   Programs only see a new variable when they start, so fully quit Kiro Crew (right-click its icon in
   the notification area and choose **Quit**) and open it again from the Start menu.

4. In Kiro Crew, open **Apps**, choose **Install from local path** from the sources menu on the
   **Library** or **Discover** tab, and enter the folder that holds `app.json`:

   ```text
   C:\Projects\haunt-crew
   ```

5. Find the **Haunt Patrol** tile under **Apps → Library**, choose **Enable**, and accept the trust
   prompt for Haunt Patrol.

6. To run a patrol now instead of waiting for midnight IST, open **Schedule**, find
   **nightly-patrol** and choose **Run now**. Then open the **Haunt Patrol** page in the sidebar.

[INSTALL-WINDOWS.md](INSTALL-WINDOWS.md) walks through the same steps in more detail, with a check
after each one and a troubleshooting table.

## Skipped nights

If the Crew gateway is not running, or the machine is off or asleep at midnight IST, that night is
skipped. There is no catch-up run.

## Exit codes

The patrol script exits with one of these codes:

| Code | Meaning |
|---|---|
| 0 | At least one ghost saved a run. A ghost that gave up is a finding, not a failure. |
| 1 | A prerequisite is missing. |
| 2 | The site did not start. |
| 3 | The site started but no ghost saved a run. |
| 4 | The patrol crashed: an exception escaped, or the record could not be written. |

Patrol exit code 3 is not the engine's exit code 3 (a ghost's run saved with result `error`). The
engine's code is only stored in the record. The patrol's code is only the process exit status.

## How Crew runs it

- **Schedule.** The `nightly-patrol` cron is a script cron: Crew calls `run` in
  `cron/nightly_patrol.py`, which starts `node src/patrol.ts` and makes no model call. Crew refuses
  `command` crons on native Windows, so a plain command cron is not an option there.
- **Page and backend.** The page calls the backend through the gateway at
  `/apps/haunt-patrol/api/latest` and `/apps/haunt-patrol/api/summary`; the backend receives them as
  `/api/latest` and `/api/summary`. The gateway's health probe calls `/health` directly. If the
  backend is down, the page reads the static copies in `ui/dist/data/`.
- **Result and log.** Crew ignores a script cron's return value, so `run` raises Crew's
  `Report(summary)`, which records the summary as the run's result and sends it as a notification.
  A missing `HAUNT_REPO_ROOT`, engine, runner or `node`, a timeout, or a non-zero patrol exit code
  raises `RuntimeError`, and Crew marks the run as failed with that text. Every step (the repository
  root, the node path, the command, the exit code and the runner's output) is appended to
  `data/logs/patrol.log`.
- **Data.** Everything is written inside the app folder: records to `data/patrols/`, ghost runs to
  `data/runs/` (passed to `haunt run --out`), and the log to `data/logs/`. In an installed copy that
  is `%USERPROFILE%\.kiro\crew\apps\haunt-patrol\data\`. The runner and the backend both find the
  data folder from their own file location.

## Environment variables

- `HAUNT_REPO_ROOT`: the Haunt repository root (the folder with `engine/` and `demo-site/`).
  Required; see above.
- `HAUNT_DASHBOARD_URL`: where the Haunt dashboard runs. Default `http://localhost:3001`. The
  backend still sends it to the page, but the current page does not link to it.
- `KIROCREW_PROXY_SECRET`: set by the Crew gateway for the backend process. The backend uses it to
  verify the `X-KiroCrew-Proxy` header. Nobody sets it by hand.

## Runs in the dashboard

The page links to the [live Haunt dashboard](https://xan-antx.github.io/haunt/), which shows the
recorded demo runs. The dashboard lists the Haunt repository's `runs/` folder, and patrol runs are
kept in the app's `data/runs/` instead. A patrol run only appears there if you copy its folder into
the clone's `runs/` and run `npm run collect-runs` in `dashboard/`. The app never does either.

## Development

- `npm test` runs the tests and `npm run typecheck` the type check.
- `npm run patrol` runs one patrol by hand (with `HAUNT_REPO_ROOT` set). It writes to this folder's
  `data\`, which an installed copy does not read; use **Run now** in Crew to fill the page.
- `ui/dist/index.mjs` is a committed build artifact. Rebuild it with `npm run build` and commit it
  whenever `ui/src/` changes. After changing files, choose **Update** on the Haunt Patrol tile in
  Crew; the update keeps the records in `data\`.
- `npm run check:protected` fails when `git diff --name-only` in the Haunt clone lists a file under
  `engine/`, `dashboard/`, `demo-site/` or `demo-site-exorcised/`. It only sees unstaged changes to
  tracked files.

## Checked against the Crew docs

The manifest, cron, backend, proxy signing and UI build were checked on 2026-11-23 against
kiro.dev/docs/crew/apps (overview, manifest reference, build your first app, publishing) and
`docs/app-kit/` plus `docs/guides/windows-install.md` in the kirodotdev/KiroCrew repository.
`test/manifest.test.ts`, `test/crewProxy.test.ts` and `test/uiConfig.test.ts` pin the results.

How a script cron's result is read was checked on 2026-11-24 in the installed Kiro Crew's own source
(`kiro_crew/cron_script.py` and `kiro_crew/slack/gateway_runtime/cron_exec.py`): a returned value is
dropped and the run is stored with an empty result; `Report(text)` stores `text` and keeps the job;
`Done(text)` stores it and removes the job; any other exception fails the run with its text.
`test/cronScript.run.test.ts` runs the shim with Python against a stand-in runner.

## Unverified facts

- Whether a script cron sees environment variables the gateway inherited, such as a user-level
  `HAUNT_REPO_ROOT`. The install steps set it as a user variable and restart Crew.
- Whether `useAppApi()` reaches the app's own `/apps/haunt-patrol/api/` proxy path once it is listed
  in `permissions.api`. If it does not, the page falls back to the static files.
- The status code the gateway returns when its own signing step fails. Nothing relies on it.

## License

[MIT](LICENSE)
