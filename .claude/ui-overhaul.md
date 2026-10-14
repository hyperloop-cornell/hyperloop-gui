# Hub Manager UI Overhaul: Follow-up Work

The web client was redesigned (black + Cornell red, top tab bar, side sheets, three-step flash
plan) in web-client `feat/ui-overhaul`. That change is **frontend only**: no cloud-services,
rpi-hub-server or API contract changes. This file lists what the backend and contract need for
the new UI to be fully supported, with STM32 flashing first, and the frontend shortcuts to
delete once each backend change lands.

Written for humans and agents. Each item says why it matters, what to change in which repo,
and when it is done. Do them in the order of the table below unless there is a reason not to.

## Rules (read first)

- **The contract comes first.** Wire types live in cloud-services (`src/models.py`,
  `src/protocol/bench_v1.py`). After changing them, run `python -m src.protocol.export` in
  cloud-services to rewrite `contracts/openapi.json` (CI fails if it is stale), then
  `npm run gen:types` in web-client. Never hand-edit `web-client/src/types/api.gen.ts`.
- **Keep the mock in step.** `web-client/src/mock/` is typed from the contract. Update it in the
  same PR so `npm run dev:mock` shows the new behavior and `npm run type-check` passes.
- **Old hubs keep working.** Hubs are updated one at a time, so every new field is optional and
  the UI must behave as it does today when a field is missing.
- Branches are `feat/...` or `fix/...` with a PR into `main`. Merge submodule PRs first and the
  hyperloop-gui PR (which bumps the submodule pointers) last, because merging it deploys.

## What shipped (frontend only)

| Area | Where in web-client |
|---|---|
| Design tokens (colors, fonts) | `src/index.css`, the `@theme` block |
| Header, socket handlers, sheet host, toasts | `src/components/layout/AppShell.tsx` |
| Pages: Hubs, Devices, Telemetry, Flash | `src/pages/*.tsx` |
| Side sheets: hub, device, terminal, add streams, schema | `src/components/sheets/` |
| SVG charts, merge, export | `src/components/telemetry/ChartPanel.tsx` |
| Ports and connections per hub (shared) | `src/stores/deviceStore.ts` |
| Open sheet, toasts | `src/stores/uiStore.ts` |

### Frontend shortcuts to remove later

The UI works around missing backend support in these places. Each points to the item that
replaces it.

| Shortcut | File | Replaced by |
|---|---|---|
| A pending flash is shown as in progress (animated bar, elapsed time) because hubs never report `running` | `src/pages/Flash.tsx` (`FlashProgress`) | 1 |
| One "Compile and write" timeline row, since there are no stages | `src/pages/Flash.tsx` (`FlashProgress`) | 2 |
| Which tool flashes a board (`arduino-cli` or `openocd`) is a hard-coded map | `src/config/boards.ts` (`FLASHERS`, `flasherFor`) | 3 |
| The board list duplicates `rpi-hub-server/config/boards.yaml` | `src/config/boards.ts` (`BOARDS`) | 3 |
| Idle connections are closed by the browser, only while someone has the Devices page open | `src/stores/deviceStore.ts` (`enforceInactivity`), "Quiet · closes in" hint in `src/pages/Devices.tsx` | 8 |
| Device counts on Hubs cost two requests per hub every 30 s | `src/stores/deviceStore.ts` (`refresh`) | 7 |
| CPU, memory and disk show "—" for up to 30 s after a page load | `src/pages/Hubs.tsx`, `src/components/sheets/HubSheet.tsx` | 6 |

## Work list

| # | Change | Repos | Contract change | Size |
|---|---|---|---|---|
| 1 | Hub reports `running` when a task starts | rpi-hub-server | no | S |
| 2 | Flash stages and progress (`stage`, `progress`) | all three | yes | M |
| 3 | Hub reports each board's flasher; cloud serves the board list | all three | yes | M |
| 4 | Typed flash result and a "Show log" view | cloud-services, web-client | yes | S |
| 5 | Cancel a running flash | all three | yes | M |
| 6 | Latest health in REST (no empty meters on load) | cloud-services, web-client | yes | S |
| 7 | Port and connection counts in the hub list | cloud-services, web-client | yes | S |
| 8 | Close idle serial connections on the hub, not in the browser | rpi-hub-server, web-client | no | S |
| 9 | Terminal and chart history after a reload | cloud-services, web-client | yes | M |
| 10 | Custom chart schemas actually used for parsing | web-client (then cloud-services) | later | S |
| 11 | Self-host Uber Move | web-client | no | S |
| 12 | Frontend polish: code splitting, phone layout, tests | web-client | no | M |

Items 1 to 5 complete STM32 flashing support. Items 6 to 12 are independent.

## STM32 flashing: full support

What already works end to end: the STM32F407G-DISC1 is detected by USB id (`boards.yaml`,
`disco_f407vg`), `.ino` is compiled with arduino-cli and programmed with OpenOCD over the
on-board ST-LINK (`write, verify, reset`), `.bin/.elf/.hex` images skip the compile, several
ST-LINKs on one hub are told apart by serial number (`OpenOcdFlasher.adapter_serial`), and the
Flash page accepts those formats and says "OpenOCD over ST-LINK". What is missing is
visibility while it runs, control over a running flash, and one source of truth for boards.

### 1. Report `running` when a task starts

**Why:** `rpi-hub-server/src/bench/command_handler.py` reports a task when it is queued
(`handle_command`) and when it finishes (`_task_worker`), never in between. `BaseTask.run()`
sets `RUNNING` internally (`src/bench/tasks/base_task.py`) but nobody sends it. A 5 to 15 minute
STM32 compile therefore looks queued until it is done. The mock backend sends `running`, which
is why `npm run dev:mock` does not show the problem.

**Change (rpi-hub-server):** in `_task_worker`, emit a status right after the task starts:
create the run task, wait until `task.status == RUNNING` (or emit from inside `BaseTask.run()`
through a callback), then call `self._report_task_status(task)`. No contract change: `running`
is already a valid status.

**Then (web-client):** nothing is required; `hubStore.updateTaskStatus` records `started_at` and
the timeline marks the step active. Optionally restore distinct pending copy in
`FlashProgress` ("Queued on <hub>") now that pending really means queued.

**Done when:** flashing an `.ino` to the DISC1 shows "Compiling and flashing…" with the
"Compile and write" row active and its start time filled in while the compile runs.

### 2. Flash stages and progress

**Why:** one bar and one row cannot tell "compiling for 6 minutes" from "stuck". The contract
already has `TaskStatusBroadcast.progress` (0 to 100) and the UI already draws it (a determinate
bar replaces the animated one as soon as `progress` is set), but no hub sends it.

**Contract (cloud-services):** add an optional `stage` to the hub's task status message in
`src/protocol/bench_v1.py` and to `TaskStatusBroadcast`:
`"queued" | "compiling" | "writing" | "verifying" | "resetting"`. Pass it through unchanged like
`progress`.

**Hub (rpi-hub-server):**
- `src/uplink/hub_agent.py` `send_task_status_update`: forward `stage` (it already forwards
  `progress`).
- Flash tasks report `stage` at each step and `progress` where the tool exposes it. Tool runs
  need streaming output instead of waiting for exit (`run_tool` in
  `src/bench/flashing/tools.py`): add a per-line callback.
- OpenOCD's `program` command prints fixed markers on stderr: `** Programming Started **`,
  `** Programming Finished **`, `** Verify Started **`, `** Verified OK **`,
  `** Resetting Target **`. Map them to `writing`, `verifying`, `resetting`.
- arduino-cli: `compiling` while `compile` runs (no percentage; keep `progress` null), then
  `writing`. avrdude's upload prints `Writing | ####` progress bars, which can be turned into a
  percentage.
- Rate-limit progress messages (say one per second): they go over the cellular uplink too.

**Web-client:** add `stage` to `Task` (`src/types/index.ts`) and store it in
`hubStore.updateTaskStatus`. In `FlashProgress` (`src/pages/Flash.tsx`), replace the single
"Compile and write" row with Compile (`.ino` only), Write, Verify (OpenOCD only) and Reset, each
active or done from `stage`. Update `src/mock/server.ts` to step through the stages.

**Done when:** an STM32 `.ino` flash walks Compile, Write, Verify, Reset in the timeline, and
the bar fills during the write.

### 3. One source of truth for boards and flashers

**Why:** the web client hard-codes the board list (`BOARDS`) and which boards use OpenOCD
(`FLASHERS`) in `src/config/boards.ts`, copied from `rpi-hub-server/config/boards.yaml`. Adding
a board (for example another STM32 Discovery or Nucleo) means editing both, and they drift.

**Hub (rpi-hub-server):** `Board.summary()` in `src/bench/boards.py` also returns `flasher` and
`reset`, so every `PortInfo.board_profile` carries them. Send the full registry summary (id,
name, fqbn, artifacts, flasher) in the handshake.

**Contract (cloud-services):** add optional `flasher` (`"arduino-cli" | "openocd" | "none"`) and
`reset` to `BoardProfileInfo`. Add `GET /api/boards`, the union of the registries the connected
hubs reported, so the Flash page's "Board" dropdown lists what the hubs can actually flash.

**Web-client:** `flasherFor()` reads `board.flasher` and falls back to the current map only for
old hubs; the Board dropdown loads from `/api/boards` with `BOARDS` as the offline fallback.
Delete both once every hub is updated.

**Done when:** adding a board to `boards.yaml` on a hub makes it appear in the Flash page's
Board dropdown, with the right "Flashed with …" line, without a web-client change.

### 4. Typed flash result and "Show log"

**Why:** flash results already carry the tool output: OpenOCD returns
`{board_fqbn, flash_duration_ms, output, flasher}` with the last 2000 characters of its log.
`TaskStatusBroadcast.result` is an untyped dictionary, so the UI does not show it. When a flash
fails, the log is the first thing anyone needs.

**Contract (cloud-services):** define `FlashResult` (`output`, `flash_duration_ms`, `flasher`,
`board_fqbn`) and document it on `TaskStatusBroadcast.result` for `commandType == "flash"`.

**Web-client:** under the timeline in `FlashProgress`, add a "Show log" disclosure with the
output in the terminal style (mono, black background) and the duration.

**Done when:** a failed STM32 flash shows the OpenOCD error text on the Flash page.

### 5. Cancel a running flash

**Why:** `BaseTask.cancel()` exists on the hub, but there is no cloud endpoint or command for
it. A wrong board choice or a hung ST-LINK can only be waited out (the GUI gives up after
17 minutes, `DEFAULT_TIMEOUT_MS.flash` in `src/services/commandService.ts`).

**Contract and cloud (cloud-services):** `POST /api/hubs/{hub_id}/tasks/{task_id}/cancel`
(operators only), sending a `cancel` command to the hub.

**Hub (rpi-hub-server):** handle `cancel` in `command_handler.py`: call `task.cancel()` on a
running task (kill the arduino-cli or OpenOCD process), drop a queued one, and report
`cancelled`. After a cancelled OpenOCD run, reset the target so the board is not left halted.

**Web-client:** add a "Cancel" button in `FlashProgress` while the task is pending or running.
The `cancelled` state is already handled ("Flash cancelled", "Try again").

**Done when:** cancelling a flash mid-compile stops it on the hub within a few seconds and the
board keeps running its old firmware.

### STM32 notes for whoever does items 1 to 5

- The first STM32 compile on a Pi takes several minutes (the core is large). The DISC1 entry
  allows `compile_timeout: 900`; keep `DEFAULT_TIMEOUT_MS.flash` above the largest hub timeout.
- Images go to the cloud as base64 JSON. nginx allows `client_max_body_size 25m`
  (`deploy/nginx/hyperloop-gui.conf`), well above an STM32F4's 1 MB of flash even with debug
  `.elf` files.
- `.bin` files are written at `STM32_FLASH_BASE` (`src/bench/flashing/openocd.py`). A board with a
  different flash base needs that value per board in `boards.yaml`.
- Boards without an ST-LINK (bare chips through the UART or USB DFU bootloader) need a new
  flasher (`stm32flash` or `dfu-util`): a `flasher` value, a class next to `openocd.py`, and
  registry entries. Item 3 makes that a hub-only change.

## Other backend changes the new UI is ready for

### 6. Latest health in REST

**Why:** health arrives only as `health` WebSocket broadcasts, every 30 s. After a page load the
Hubs table and hub sheet show "—" until the next one.

**Change:** the cloud keeps the latest health per hub and includes it as an optional `health`
on `HubInfo` (or sends the latest health for every hub to a browser when it connects). In
web-client, seed `hubStore.health` from `fetchHubs`.

### 7. Port and connection counts in the hub list

**Why:** the Hubs page shows device counts, which today requires `GET /ports` and
`GET /connections` for every online hub every 30 s (`deviceStore.refresh`).

**Change:** add optional `portCount` and `connectionCount` to `HubInfo`, or
`GET /api/hubs?include=ports,connections`. web-client then fetches full port lists only on the
Devices and Flash pages.

### 8. Close idle connections on the hub

**Why:** "close a connection whose byte counters have not moved for 60 s" runs in the browser
(`deviceStore.refresh({ enforceInactivity: true })`), only while someone has the Devices page
open, and a view-only session gets 403 when it tries. The behavior was kept as it was before
the redesign, not endorsed.

**Change:** if the behavior is wanted, the hub's serial manager closes idle connections itself
(configurable per profile, off for `pod` mode) and reports a `device_event`. Then delete
`enforceInactivity`, `idleRemaining` and the "Quiet · closes in" hint in web-client.

### 9. Terminal and chart history after a reload

**Why:** the serial terminal keeps 500 lines per device and charts keep one hour, all in browser
memory. A reload or a second laptop starts empty, and the Custom time range can only cover what
this browser saw.

**Change:** `GET /api/hubs/{hub_id}/telemetry` already returns recent telemetry the cloud
keeps. Add `portId` and `since` filters. In web-client, when a device is subscribed, fetch its
recent telemetry and feed it through `telemetryStore.processTelemetry` before live data.

### 10. Custom chart schemas used for parsing

**Why:** schemas created under Telemetry → Schemas are saved to `localStorage`
(`customChartSchemas`), but `detectSensorType` in `src/services/sensorParser.ts` only reads
`src/config/sensor-mappings.json`, so a saved schema never produces a chart. This predates the
redesign.

**Change (web-client):** include `getCustomSchemas()` in detection, custom schemas first
(`getAllSchemas` in `src/lib/customSchemas.ts`). Later, store schemas in the cloud so the team
shares them instead of each browser having its own.

### 11. Self-host Uber Move

The font stack is `'Uber Move', 'Uber Move Text', 'UberMove', Archivo, system-ui, …`
(`--font-sans` in `src/index.css`). Uber Move is proprietary and on no public CDN, so today it is
used only on machines that have it installed; everyone else sees Archivo (loaded from Google
Fonts in `index.html`), which the design was drawn with.

To serve it to everyone, with a license that allows web embedding:

1. Put the `.woff2` files in `web-client/public/fonts/`. Do not commit them unless the license
   allows it in this repository; otherwise have CI copy them in before `npm run build`.
2. Add to `src/index.css`, above `@theme`, one block per weight used (400, 500, 600, 700, 800):

   ```css
   @font-face {
     font-family: 'Uber Move';
     src: url('/fonts/UberMove-Bold.woff2') format('woff2');
     font-weight: 700;
     font-display: swap;
   }
   ```

3. Remove the Archivo request from `index.html` once nothing falls back to it.

### 12. Frontend polish (no API change)

- **Code splitting.** The main chunk is about 1.6 MB (gzip 510 kB), mostly CodeMirror, jsPDF and
  html2canvas. Load the Flash editor and the PNG/PDF export with dynamic `import()`.
- **Phones.** The Hubs and Devices tables scroll sideways, as in the mockup. A stacked layout
  below 640 px would read better on a phone at the track.
- **Tests.** There are none beyond type-check and lint. Add Vitest for `src/lib/format.ts`, the
  chart geometry in `ChartPanel.tsx`, and the stores.
- **Mock.** Send a `health` message when a mock socket connects so `dev:mock` shows meters at
  once, and add `stage`/`progress` once item 2 lands.

## Checking a change

```bash
# web-client
npm run type-check && npm run lint && npm run build
npm run dev:mock        # in-browser fake hubs; log in with any NetID and "hyperloop-dev"

# end to end, against real code paths: cloud-services + rpi-hub-server with HUB_PROFILE=dev-sim
```

For STM32 changes, finish on real hardware: a DISC1 on a lab hub, flashed once with `.ino`
(compile path) and once with `.bin` (OpenOCD only), then "Open terminal" to see the new firmware
print. See `.claude/rpi-hub-setup.md` for the hub side (OpenOCD install, `lsusb` checks).
