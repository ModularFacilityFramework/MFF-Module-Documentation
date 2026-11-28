# CryoEM module: the TemControl page

Module `cryoem` v0.1.41 and later. TemControl is the Windows microscope-control program. This is step 1: the
connection from the CryoEM page to **Nucleo** (the C++ service on nusbf that relays to instruments). TemControl
itself is not connected yet, and its own look and controls come later.

Path: CryoEM page ⇄ CryoEM backend ⇄ Nucleo ⇄ TemControl. The browser never talks to Nucleo.
Nucleo's API is described in Nucleo's own repository (`docs/instruments.md`).

## The page (CryoEM → TemControl)
- **Connection light.** Green: Nucleo answers and the instrument is online. Amber: Nucleo answers, the instrument
  is offline. Red: no connection to Nucleo, or Nucleo refuses. Grey: still checking, or not set up. A
  "SIMULATED" badge shows when the instrument is the simulator.
- **Send a command.** Pick a command from the list the instrument publishes (a text box when it has published
  none), give its arguments as a JSON object, press Send. The command's description and arguments are shown.
  Nothing is pre-chosen. Send is greyed, with the reason, without a connection or while the instrument is offline.
- **Replies and events.** A live table with time, entry number, type, command and details. "Show" opens the whole
  entry. Filter by type (state snapshots are hidden by default), search, and core's two-way column sorting. Newest
  first until a column is sorted. The page opens with recent history and keeps up to 500 entries plus the 20
  latest state snapshots. It says so if Nucleo started a new stream or if entries were lost.
- Per user, kept across leaving and reloading: the chosen command, the arguments typed for each command, the type
  filter, the search text, the sort order and the opened entries.
- The page does not belong to a project; the project tabs do not change it.

## Who may use it
- Core's page access: the manifest declares the page `temcontrol` ("TemControl"). With no grant it is for
  developers and administrators only. Grant it at Setup → Preferences → 🔐 Access. CryoEM's own "Page Access"
  tab does not cover it.
- The server checks it on every TemControl route (`requirePage('temcontrol')`). Calls made by another module on
  a user's behalf, and background calls, are refused. A developer's "view as" is read-only (core).
- Anyone with the page can send any command. There is no control lock: two users can command at the same time.

## Safety rules of the relay
- Each press of Send sends one command once. Nothing is retried automatically.
- Nothing is queued for an offline instrument (`queue_if_offline: false`).
- If there is no clear answer, the page says that it is not known whether the command reached the instrument, and
  asks the user to check the list before sending again.
- The MFF user name goes to Nucleo as `on_behalf_of` (made printable ASCII, at most 128 characters; a name that
  has to be changed also gets the user's id). The CryoEM backend logs each send.

## Setup (core's `App_backend/.env`, then restart the backend)
| Key | Meaning |
|---|---|
| `MODULE_CRYOEM_NUCLEO_URL` | Nucleo's address. No default: without it the page says "Not set up". Dev service: `http://127.0.0.1:8741` |
| `MODULE_CRYOEM_NUCLEO_INSTRUMENT` | The instrument's name in Nucleo. Default `temcontrol-sim` (the simulator), so nothing reaches a real microscope unless this is set. The real one is `temcontrol` |

- Authentication to Nucleo is MUNGE: the backend runs `munge -n` for every request. The Unix user the MFF backend
  runs as must be in that instrument's `controllers` list in Nucleo (or be a Nucleo administrator). On nusbf the
  MFF backends run as `namlb`.
- **Before the real microscope is connected:** `namlb` is a shared account, so any process running as `namlb` on
  nusbf can command an instrument that lists it, without MFF's access rules.
- Needs `munge` on the MFF server and Node 20.3 or later. No database change.
- The dev service on port 8741 runs from the `nucleo` session.

## Module API (under `/cryoem`, login and page `temcontrol` needed)
| Route | Answer |
|---|---|
| `GET /temcontrol/status` | Always 200 for an allowed user: `{ link, instrument, status, message? }`. `link`: `ok`, `not_configured`, `munge_failed`, `unreachable`, `no_answer`, `refused`, `rejected`, `server_error`, `bad_answer`. `status` is Nucleo's instrument status |
| `GET /temcontrol/stream?after=&wait=` | Nucleo's stream answer plus `instrument` and `after`. Without `after`: the latest entries, at once. `wait` is at most 20 s |
| `POST /temcontrol/commands` `{ command, args? }` | 202 `{ command, request_id }`. Errors `{ error, code }`: 400, 404, 409 `instrument_offline`, 422 `unknown_command`, 502 (`unreachable`, `refused` or Nucleo's code, `outcome_unknown`), 503 `not_configured` |

Nucleo's 401 and 403 are never passed on to the browser (to the app, 401 means the user's sign-in expired); they
become 502.

## For whoever connects TemControl (from `nucleo`)
After a Nucleo restart, a command that was delivered but not yet acknowledged is delivered again with the same id.
The agent must ignore an id it has already received, so that nothing is executed twice.
