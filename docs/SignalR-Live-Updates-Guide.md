# How to use SignalR in the MCX IoT app: every step, with examples

Server: `https://microservices.alitis.se/alecomapigatewayiot`

SignalR keeps **one connection open**. The server **pushes** a small message (an *event*) to your app the moment
something happens, for example "the gate command was confirmed". Your app does not have to ask again and again.

> **How much of this is tested?** I had no real login, so I never received a real event. The steps and the
> code come from the app's own code and from the platform's API guide. The JSON examples that show events are
> **illustrative**: I built them from the fields the documents name (`id`, `type`, `subject`, `time`,
> `sequence`, `data.commandId`, `data.deviceId`). A real event may carry more fields in `data`. Section 17
> lists everything I could not confirm.

## Contents

1. What you need before you start
2. The address and the permission
3. Step 1: load the library
4. Step 2: build the connection (every option explained)
5. Step 3: listen for events
6. Step 4: start and `Subscribe` (with filter examples)
7. What one message looks like, box by box
8. Every event type, with an example
9. Example A: pressing a gate, minute by minute
10. Example B: a button that needs a second person
11. Example C: a scene (several buttons)
12. Example D: a command that fails or times out
13. Tokens: keep them fresh
14. Reconnecting and catching up (with a worked example)
15. Turning events into screen changes (code)
16. Fallback when SignalR does not work
17. What I could not confirm
18. Testing, step by step
19. Android (Kotlin)
20. Cheat sheet

---

## 1. What you need before you start

| You need | Where to get it | Example |
| --- | --- | --- |
| The **access token** | The login answer (`POST .../alecomservicemanagementiot/api/v1/auth/login`) | `eyJhbGciOi...` |
| The **site ids** (optional filter) | Every button in the app menu has a `siteId` | `site_7Hq2k9` |
| The **permission** `IOT_EVENTS` View | The person's role | Every role with `OPS_FUNCTIONS` View has it from platform 5.7.0 |

Getting the site ids from the app menu (`GET /iot/v1/me/app-menu`):

```javascript
// menu is the "data" of the app-menu answer
function siteIdsFromMenu(menu) {
  const ids = new Set();
  for (const tile of menu.tiles) {
    for (const item of tile.items) {
      const f = item.function || item.group;            // a button or a scene
      if (f && f.siteId) ids.add(f.siteId);
    }
  }
  return [...ids];                                      // ["site_7Hq2k9", "site_4Lm3Xp"]
}
```

## 2. The address and the permission

```
https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1
```

- It is at the **root of the gateway**, next to `/iot/v1`. It is **not** `.../iot/v1/hubs/iot/v1`.
- Write it with `https://`. The SignalR program switches to `wss://` by itself.

If the person lacks the permission, the connection is refused:

| Answer | Meaning |
| --- | --- |
| `401` `UNAUTHORIZED` | No token, or an old token |
| `403` `STREAM_NOT_PERMITTED` | The role has no `IOT_EVENTS` View. Typical for an older site, a role created after the platform upgrade, or a role with `OPS_FUNCTIONS` Manage but not View |

No API call tells you in advance whether the permission exists. **Try to connect**, and if it is refused, use
the fallback of section 16.

## 3. Step 1: load the library

Browser or WebView (this is what the app does):

```html
<script src="https://cdn.jsdelivr.net/npm/@microsoft/signalr@8.0.7/dist/browser/signalr.min.js"></script>
```

Node.js: `npm install @microsoft/signalr`. Android: the package `com.microsoft.signalr:signalr` (section 19).
iOS: the API guide says there is **no official SignalR client for Swift**.

If the browser cannot load the script (no internet, jsDelivr blocked), `window.signalR` does not exist. Check
for that and use the fallback (section 16):

```javascript
if (!window.signalR) { startPolling(); return; }
```

## 4. Step 2: build the connection (every option explained)

```javascript
const GATEWAY = 'https://microservices.alitis.se/alecomapigatewayiot';

const conn = new signalR.HubConnectionBuilder()
  .withUrl(GATEWAY + '/hubs/iot/v1', {
    accessTokenFactory: () => getFreshAccessToken(),
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .configureLogging(signalR.LogLevel.Warning)
  .build();
```

| Option | What it does | Why |
| --- | --- | --- |
| `withUrl(address, options)` | Sets where to connect | Section 2 |
| `accessTokenFactory` | A function that returns the token. SignalR calls it **every time it opens the line**, including every reconnect, and puts the result in the `access_token` part of the address | You never put the token in a header yourself. Return the **newest** token (section 13) |
| `skipNegotiation: true` | Skips the first HTTP step (`negotiate`) and opens the WebSocket directly | Faster. SignalR only allows it when `transport` is WebSockets |
| `transport: WebSockets` | Use only WebSockets | No slower fallback inside SignalR. Our own fallback is section 16 |
| `withAutomaticReconnect([0, 2000, 5000, 10000, 30000])` | If the line breaks, retry after 0 ms, 2 s, 5 s, 10 s, 30 s. After the last one it gives up and calls `onclose` | Keeps the line alive over short signal losses |
| `configureLogging(Warning)` | Only warnings in the console | Use `LogLevel.Information` while you debug |

## 5. Step 3: listen for events

The server calls a method named exactly **`event`** on your app. Register the listener **before** you start
the connection, so you cannot miss the first message.

```javascript
const seen = new Set();                    // the ids you already handled

conn.on('event', ev => {
  if (!ev) return;
  if (ev.id && seen.has(ev.id)) return;    // a repeat: ignore it
  if (ev.id) seen.add(ev.id);
  handleEvent(ev);                         // your code (section 15)
});
```

Why the `seen` set: after a reconnect the same event can arrive twice. The `id` is unique per event.
(Keep it from growing forever: remove the oldest ids when it passes, say, 500.)

## 6. Step 4: start and `Subscribe` (with filter examples)

Nothing arrives **before** you call `Subscribe`.

```javascript
const filter = {
  siteIds: ['site_7Hq2k9', 'site_4Lm3Xp'],
  eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*']
};

await conn.start();                        // opens the WebSocket
await conn.invoke('Subscribe', filter);    // tells the server what you want
```

All three boxes of the filter are optional, and an **empty filter means everything the person may see**.

| Box | Meaning |
| --- | --- |
| `siteIds` | Only events of these buildings |
| `deviceIds` | Only events of these devices |
| `eventTypes` | Only these types. A `*` at the end means "everything that starts like this" |

Filter examples:

```javascript
// 1. Everything the person may see (good for a first test)
{}

// 2. All command results, in all buildings
{ eventTypes: ['se.mcf.iot.command.*'] }

// 3. Only the finished command results of one building
{ siteIds: ['site_7Hq2k9'],
  eventTypes: [
    'se.mcf.iot.command.confirmed.v1',
    'se.mcf.iot.command.failed.v1',
    'se.mcf.iot.command.timedout.v1'
  ] }

// 4. Everything about one device
{ deviceIds: ['dev_9Pz4Wc'] }

// 5. What the MCX app sends
{ siteIds: siteIdsFromMenu(menu),
  eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*'] }
```

Rules:

- **Call `Subscribe` again after every reconnect.** The server forgets the filter when the line breaks.
- Call it again when the list of buildings changes. A second call replaces the first filter. (The MCX app
  does this each time it reloads the buttons.)
- If you list `eventTypes` exactly (not with `*`), include the `.v1` at the end.

I did not see the **answer** of `Subscribe`. The MCX app ignores it and treats "no error" as success.

## 7. What one message looks like, box by box

```json
{
  "specversion": "1.0",
  "id": "evt_8Kq2",
  "type": "se.mcf.iot.command.confirmed.v1",
  "source": "...",
  "subject": "cmd_5Rt7Yb",
  "time": "2026-10-09T08:00:03.412Z",
  "organisationid": 7,
  "sequence": 41,
  "dataschema": "/iot/v1/schemas/events/se.mcf.iot.command.confirmed.v1",
  "datacontenttype": "application/json",
  "data": { "commandId": "cmd_5Rt7Yb", "deviceId": "dev_9Pz4Wc" }
}
```

| Box | Meaning | What to do with it |
| --- | --- | --- |
| `id` | A unique number for this message | Ignore a message with an `id` you already handled |
| `type` | What happened (section 8) | Decide the reaction from it |
| `subject` | What it is about. For a command, the command id | Match it to the thing on screen |
| `time` | When it happened (UTC) | Keep the latest one, for catching up (section 14) |
| `sequence` | The number of this event in its device's stream, **without gaps** | If you get 41 and then 43, you lost 42 |
| `dataschema` | A path that describes `data`: `/iot/v1/schemas/events/{type}` | Open it when you need the exact boxes of `data` |
| `organisationid` | The organization the event belongs to | Ignore events of another organization than the chosen one |
| `data.commandId` | Which button press it is about | Find the press on screen |
| `data.deviceId` | Which device | Refresh that device |

The same events are also sent as webhooks to partners and as MQTT messages, and `GET /iot/v1/events` lists
them. So the list call shows exactly what the stream sends.

## 8. Every event type, with an example

The eight command types come from the OpenAPI file. Each one ends in `.v1` and **only the `data` changes in
meaning**. (The examples are illustrative, see the note at the top.)

| Type | What happened | Final? | What to show |
| --- | --- | --- | --- |
| `se.mcf.iot.command.accepted.v1` | The server checked the press and will send it | No | "Sending..." |
| `se.mcf.iot.command.approval.requested.v1` | A second person must approve first | No | "Waiting for approval" |
| `se.mcf.iot.command.delivered.v1` | It reached the gateway or device | No | "Sent..." |
| `se.mcf.iot.command.confirmed.v1` | The device reports the new state | **Yes** | Success |
| `se.mcf.iot.command.failed.v1` | The device did not switch | **Yes** | Error |
| `se.mcf.iot.command.timedout.v1` | No confirmation in time. **It may have switched** | **Yes** | Warning: check on site, never re-send automatically |
| `se.mcf.iot.command.cancelled.v1` | Cancelled before it was sent | **Yes** | Info |
| `se.mcf.iot.command.rejected.v1` | Refused by a rule after acceptance | **Yes** | Error with the reason |

Device events: `se.mcf.iot.device.*`. The API guide names one example,
`se.mcf.iot.device.input.changed.v1` (an input on a device changed). The files I have do not list the others,
so subscribe with `se.mcf.iot.device.*` and log what arrives (section 18).

Example of a device event (illustrative):

```json
{
  "id": "evt_9Wd1",
  "type": "se.mcf.iot.device.input.changed.v1",
  "subject": "dev_9Pz4Wc",
  "time": "2026-10-09T08:05:41.100Z",
  "sequence": 118,
  "data": { "deviceId": "dev_9Pz4Wc" }
}
```

A good reaction: ask `GET /iot/v1/devices/{deviceId}/state` and redraw that device.

Remember the "final" rule: when you see a final type, the press is **finished**. Stop waiting for more.

## 9. Example A: pressing a gate, minute by minute

The person holds "Open the main gate". Times are examples.

**08:00:00.000. The app sends the press** (a normal request, not SignalR):

```http
POST /iot/v1/functions/fn_3kLm8Q/commands
Authorization: Bearer eyJhbGciOi...
Idempotency-Key: 2f1c9a1e-6b0e-4a0e-9d3e-2c1d5a7b8c90
Content-Type: application/json

{ "confirm": true, "location": { "latitude": 59.8586, "longitude": 17.6389 } }
```

**08:00:00.120. The server answers `202`:**

```json
{ "success": true,
  "data": { "commandId": "cmd_5Rt7Yb", "functionCode": "GATE-OPEN", "status": "Accepted",
            "isFinal": false, "acceptedAt": "2026-10-09T08:00:00.000Z",
            "expiresAt": "2026-10-09T08:00:10.000Z" } }
```

The screen shows "Sending...". Remember `cmd_5Rt7Yb`.

**Then the events arrive on the line** (each is one call of your `conn.on('event')`):

| Time | `type` | `data.commandId` | Your screen |
| --- | --- | --- | --- |
| 08:00:00.200 | `se.mcf.iot.command.accepted.v1` | `cmd_5Rt7Yb` | "Sending..." (no change) |
| 08:00:01.050 | `se.mcf.iot.command.delivered.v1` | `cmd_5Rt7Yb` | "Sent..." |
| 08:00:03.412 | `se.mcf.iot.command.confirmed.v1` | `cmd_5Rt7Yb` | "Confirmed", button turns green |

In your handler:

```javascript
function handleEvent(ev) {
  const id = ev.data && ev.data.commandId;
  if (!id || !pendingPresses[id]) return;          // not a press this phone is following
  const map = {
    'se.mcf.iot.command.accepted.v1':  ['Sending...',  false],
    'se.mcf.iot.command.delivered.v1': ['Sent...',     false],
    'se.mcf.iot.command.confirmed.v1': ['Confirmed',   true ],
    'se.mcf.iot.command.failed.v1':    ['Failed',      true ],
    'se.mcf.iot.command.timedout.v1':  ['No confirmation. Check on site.', true],
    'se.mcf.iot.command.cancelled.v1': ['Cancelled',   true ],
    'se.mcf.iot.command.rejected.v1':  ['Refused',     true ]
  };
  const hit = map[ev.type]; if (!hit) return;
  showStatus(id, hit[0]);
  if (hit[1]) { delete pendingPresses[id]; readFinalCommand(id); }   // final: read the full answer once
}
```

**After the final event, read the full command once** (it has the `detail` text and the history):

```http
GET /iot/v1/commands/cmd_5Rt7Yb
```

```json
{ "success": true,
  "data": { "commandId": "cmd_5Rt7Yb", "status": "Confirmed", "isFinal": true,
            "completedAt": "2026-10-09T08:00:03.412Z",
            "history": [
              { "status": "Accepted",  "at": "2026-10-09T08:00:00.000Z", "detail": null, "actor": "fake.officer" },
              { "status": "Delivered", "at": "2026-10-09T08:00:01.050Z", "detail": null, "actor": null },
              { "status": "Confirmed", "at": "2026-10-09T08:00:03.412Z", "detail": null, "actor": null } ] } }
```

(The first history entry is from the API guide. The others are illustrative.)

**Important:** the event tells you *that* it changed. The **final truth** is the `GET /commands/{commandId}`
answer. The MCX app works exactly like this: an event only makes it ask `GET` at once.

## 10. Example B: a button that needs a second person

The button has `confirmation.twoPersonApproval: true`.

1. The person presses. The server answers `202` with `status: "PendingApproval"`.
2. An event arrives: `se.mcf.iot.command.approval.requested.v1` with `data.commandId`.
3. Show "Waiting for approval" with a Cancel button.
4. A **different** person with the permission `OPS_APPROVE` Manage approves:

```http
POST /iot/v1/commands/cmd_5Rt7Yb/approve
{}
```

5. Then the normal events follow: `accepted`, `delivered`, `confirmed`.
6. If nobody approves before `expiresAt`, the press ends (the approve call would answer `409 APPROVAL_EXPIRED`).

The person who pressed can never approve their own press (`403 SELF_APPROVAL_NOT_ALLOWED`).

## 11. Example C: a scene (several buttons)

A scene runs several buttons **in order**, one after the other. You press the scene once:

```http
POST /iot/v1/groups/grp_5Tq9Lm/commands
Idempotency-Key: 7c2a...
{ "confirm": true }
```

The answer is `202` with a **group run** (`groupRunId`, a `status` that says it is still going, and `members[]`, one per button
with its own `commandId`, `sequence` and `status`).

What arrives on the line: the events of **each member command**, one after the other, because each member is
its own command with its own life (member 2 starts only after member 1 is final, or has been delivered and waits for its feedback).

| Time | `type` | `data.commandId` | Which member |
| --- | --- | --- | --- |
| 08:10:00.2 | `command.accepted.v1` | `cmd_A1` | 1 (Open the main gate) |
| 08:10:03.4 | `command.confirmed.v1` | `cmd_A1` | 1 done |
| 08:10:03.6 | `command.accepted.v1` | `cmd_A2` | 2 (Garage 12 - Open) |
| 08:10:06.0 | `command.confirmed.v1` | `cmd_A2` | 2 done |
| ... | ... | ... | ... |

How to connect an event to the scene on screen:

```javascript
// when the 202 arrives, remember every member command id:
const run = answer.data;                                   // the group run
run.members.forEach(m => memberOfRun[m.commandId] = { runId: run.groupRunId, seq: m.sequence });

// in handleEvent:
const m = memberOfRun[ev.data.commandId];
if (m) { updateRow(m.runId, m.seq, ev.type); refreshRun(m.runId); }

function refreshRun(runId) {                                // one request gives the whole picture
  return api('GET', '/group-runs/' + runId).then(r => drawScene(r));
}
```

`GET /iot/v1/group-runs/{groupRunId}` returns each member's `sequence`, `commandId`, `functionName`,
`status`, `isFinal` and `detail`, plus the run's own `status`. The run is finished when its `isFinal` is `true`. (The exact status words of a
run are not listed in the files I have.)

I have not seen the exact `data` of a scene-member event. If it does not hold the run id, look the run up with
`GET /iot/v1/commands/{commandId}` (the command has a `groupRunId`).

## 12. Example D: a command that fails or times out

**Failed.** `se.mcf.iot.command.failed.v1` arrives. Then read `GET /iot/v1/commands/{commandId}`. Show
`data.detail` to the person ("The device did not switch"). Pressing again is the **person's** decision.

**Timed out.** `se.mcf.iot.command.timedout.v1` arrives. This means "no confirmation in time", and **the
equipment may have switched anyway** (on the old `legacy` route it simply means "sent, not confirmed").
Show a warning: "Check on site before you press again." **Never press again automatically**: a gate that is
pulsed twice may open and close.

**Rejected.** `se.mcf.iot.command.rejected.v1`: a rule refused it after it was accepted (for example the
device went into maintenance). Show the `detail`.

## 13. Tokens: keep them fresh

SignalR calls your `accessTokenFactory` every time it opens the line, including each reconnect. Make it
return a token that is valid **now**:

```javascript
let token = { value: null, expiresAt: 0 };         // filled in at login: expiresAt = Date.now() + expiresIn * 1000

async function getFreshAccessToken() {
  if (Date.now() > token.expiresAt - 60000) {       // less than a minute left
    token = await refreshToken();                   // POST .../auth/refresh (see the main guide, section 5)
  }
  return token.value;
}
```

Rules from the main guide: refresh **before** it ends. A refresh gives you a **new** refresh token, so make
sure only one refresh runs at a time. If the refresh fails, send the person back to the login.

What I do not know: whether the server closes an **open** line when its token ends. The code above makes a
reconnect safe either way.

## 14. Reconnecting and catching up (with a worked example)

Events sent while the line was down are **never replayed**. So after every reconnect you must catch up.

**Timeline example.** The phone goes through a tunnel.

| Time | What happens |
| --- | --- |
| 09:00:00 | The line is live. Last event handled: `sequence 41`, `time 09:00:00` |
| 09:00:10 | The phone loses signal. `onreconnecting` fires. The screen says "Connecting..." |
| 09:00:40 | Signal is back. SignalR opens the line again (after 0, 2, 5, 10, 30 s retries). `onreconnected` fires |
| 09:00:40 | **You** call `Subscribe` again (the server forgot) and catch up |
| 09:00:41 | Events 42, 43, 44 happened while you were away. You never got them. The next event you receive is `sequence 45` |

What to do in `onreconnected`:

```javascript
conn.onreconnected(async () => {
  await conn.invoke('Subscribe', filter);          // 1. subscribe again
  await catchUp();                                 // 2. read what you missed
});

async function catchUp() {
  // a) the simple way: reload what the screens show
  await reloadButtons();                           // GET /iot/v1/me/app-menu ... (main guide, section 6)
  await reloadEventsList();                        // GET /iot/v1/notifications and /iot/v1/conditions
  for (const d of devicesOnScreen) await refreshDevice(d.deviceId);   // GET /iot/v1/devices/{id}/state

  // b) the exact way: read the events you missed, in order
  const rows = await api('GET', '/events', { query: { since: lastEventTime, pageSize: 200 } });
  rows.items.reverse().forEach(handleEvent);       // the list is newest first, so reverse it
}

// c) press still running? ask once
for (const id of Object.keys(pendingPresses)) readFinalCommand(id);
```

`GET /iot/v1/events` filters: `type` (exact, or with `*` at the end), `deviceId`, `functionId`, `since`
(ISO 8601, no zone means UTC), `page`, `pageSize` (default 50, at most 200).

**Detecting a gap with `sequence`:**

```javascript
const lastSeq = {};                                 // per device
function checkGap(ev) {
  const dev = ev.data && ev.data.deviceId; if (!dev || ev.sequence == null) return;
  const prev = lastSeq[dev];
  if (prev != null && ev.sequence !== prev + 1) catchUpDevice(dev);   // 41 then 45: something is missing
  lastSeq[dev] = ev.sequence;
}
```

A shared (delegated) button from **another organization** sends no events to you, because an event belongs to
its own organization. Follow that press with `GET /iot/v1/commands/{commandId}` (section 16).

## 15. Turning events into screen changes (code)

A small, complete example you can adapt. It keeps one "store" and redraws when an event arrives.

```javascript
const state = { line: 'off', presses: {}, devices: {} };       // line: off | connecting | live | polling

function setLine(s) { state.line = s; render(); }               // show "Live" / "Connecting..." / "Every minute"

function handleEvent(ev) {
  checkGap(ev);
  const t = ev.type, d = ev.data || {};

  if (t.startsWith('se.mcf.iot.command.')) {
    if (state.presses[d.commandId]) {                            // a press this phone follows
      state.presses[d.commandId].last = t;
      if (isFinal(t)) readFinalCommand(d.commandId);
    }
  } else if (t.startsWith('se.mcf.iot.device.')) {
    if (d.deviceId && state.devices[d.deviceId]) refreshDevice(d.deviceId);
  }
  render();
}

function isFinal(type) {
  return /command\.(confirmed|failed|timedout|cancelled|rejected)\.v1$/.test(type);
}
```

## 16. Fallback when SignalR does not work

If the library does not load, the line is refused (`401`, `403`), or it breaks for good (`onclose`), ask the
server instead. The MCX app does this for a running press:

```javascript
async function followPress(commandId) {
  const t0 = Date.now(); let n = 0;
  while (Date.now() - t0 < 120000) {                            // give up after 2 minutes
    const c = (await api('GET', '/commands/' + commandId)).data;
    showStatus(commandId, c.status);
    if (c.isFinal) return c;
    n++;
    await sleep(n < 4 ? 800 : n < 10 ? 1500 : 3000);           // 0.8 s, then 1.5 s, then 3 s
  }
}
```

Stop also when the server answers `404`, `403` or `401`. When SignalR **is** live, wake this loop up as soon
as an event for that command arrives, instead of waiting for the sleep to end.

For lists (alerts, device states) the MCX app simply asks again every **60 seconds** while the screen is
visible.

**Telling `401` from `403` when the line fails.** SignalR's own error text is vague, so ask the check-in
address once and read the status:

```javascript
async function whyRefused() {
  const r = await fetch(GATEWAY + '/hubs/iot/v1/negotiate?negotiateVersion=1', {
    method: 'POST', headers: { Authorization: 'Bearer ' + await getFreshAccessToken() }, body: ''
  });
  return r.status;   // 200 fine, 401 token problem, 403 permission (STREAM_NOT_PERMITTED)
}
```

## 17. What I could not confirm

- The **answer** of `Subscribe` and its error codes.
- A **real event**. The examples in this file are illustrative.
- The full list of **device event types** (only `se.mcf.iot.device.input.changed.v1` is named).
- The exact `data` of a **scene-member** event, and whether events carry a `groupRunId`.
- **Limits**: how many connections a person may open, how often, how big a filter may be.
- Whether the server closes an open line when the token ends, and what happens while the Android app is in the
  **background**.
- That the Node and browser programs in this file run against production. I never had a token.

The platform's own file `Documentation/Api/Field-App-Event-Stream.md` should cover the reconnect recipe, the
event types, the limits and the refusal codes. I was not given it. If you send it, this guide can be completed.

## 18. Testing, step by step

**What I really tested (no token):** production answers `401` for both calls below, so the address is right
and a token is required.

```
POST https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1/negotiate?negotiateVersion=1
-> 401 {"success":false,"message":"Authentication is required.","errors":[{"code":"UNAUTHORIZED","domain":"Gateway"}]}
```

Send an **empty body** (`Content-Length: 0`), or the web server answers `411`.

**With a token, in this order:**

1. **Login** and keep the access token (`POST .../alecomservicemanagementiot/api/v1/auth/login`).
2. **negotiate** with `Authorization: Bearer <token>` and an empty body. `200` means token, address and
   permission are fine. `403` usually means the permission is missing.
3. **Run this terminal program** (`npm install @microsoft/signalr`). Postman cannot hold a SignalR line.

```javascript
// watch.js        run:  TOKEN=<access token> node watch.js
const signalR = require('@microsoft/signalr');
const GATEWAY = 'https://microservices.alitis.se/alecomapigatewayiot';
const filter = { eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*'] };

const conn = new signalR.HubConnectionBuilder()
  .withUrl(GATEWAY + '/hubs/iot/v1', {
    accessTokenFactory: () => process.env.TOKEN,
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .configureLogging(signalR.LogLevel.Information)
  .build();

conn.on('event', ev => console.log(ev.time, ev.type, JSON.stringify(ev.data)));
conn.onreconnecting(e => console.log('reconnecting...', e && e.message));
conn.onreconnected(() => { console.log('reconnected'); conn.invoke('Subscribe', filter); });
conn.onclose(e => console.log('closed', e && e.message));

(async () => {
  await conn.start();
  await conn.invoke('Subscribe', filter);
  console.log('Listening. Press a button in the app to see events.');
})();
```

4. **Press a button** in the app (or send `POST /iot/v1/functions/{id}/commands` with an `Idempotency-Key`).
   You should see lines like this in the terminal (illustrative):

```
2026-10-09T08:00:00.200Z se.mcf.iot.command.accepted.v1 {"commandId":"cmd_5Rt7Yb", ...}
2026-10-09T08:00:01.050Z se.mcf.iot.command.delivered.v1 {"commandId":"cmd_5Rt7Yb", ...}
2026-10-09T08:00:03.412Z se.mcf.iot.command.confirmed.v1 {"commandId":"cmd_5Rt7Yb", ...}
```

5. **Compare** with `GET /iot/v1/commands/cmd_5Rt7Yb`. The status must be `Confirmed`.
6. **Test a reconnect.** Switch the Wi-Fi off for 20 seconds and on again. You should see
   "reconnecting...", then "reconnected". Press another button and check that events still arrive.
7. **Test the catch-up.** While offline, press a button from another device. After the reconnect, call
   `GET /iot/v1/events?since=<time before the press>` and check that the missed event is in the list.
8. **Find the device event types of your sites.** Leave the program running for a while with the device
   filter and note the `type` values you see.

**If something does not work**

| What you see | Likely cause | What to do |
| --- | --- | --- |
| `negotiate` answers `401` | Token missing or old | Log in again, use the new token |
| `negotiate` answers `403` | Permission missing (`STREAM_NOT_PERMITTED`) | Ask the administrator for `IOT_EVENTS` View, or use section 16 |
| `negotiate` answers `411` | No body | Send an empty body |
| `start()` works but nothing arrives | `Subscribe` was not called, or `siteIds` does not include the building | Call `Subscribe`. Test with `{}` |
| The same event twice | Normal after a reconnect | Ignore an `id` you already handled |
| Events missing after a signal loss | Never replayed | Section 14 |
| No events for a shared button | Events belong to the other organization | `GET /iot/v1/commands/{commandId}` |
| It works in the terminal but not in a browser page | A browser cannot add headers to a WebSocket, and the server may refuse the page's origin | Not tested by me. Use the app or the terminal program |

## 19. Android (Kotlin)

From the API guide. Add `com.microsoft.signalr:signalr` to Gradle.

```kotlin
val hub = HubConnectionBuilder.create("$gateway/hubs/iot/v1")
    .withAccessTokenProvider(Single.defer { Single.just(tokens.freshAccessToken()) })
    .build()

hub.on("event", { envelope: JsonObject -> handle(envelope) }, JsonObject::class.java)

hub.start().blockingAwait()
hub.invoke(
    JsonObject::class.java, "Subscribe",
    mapOf(
        "siteIds" to listOf(siteId),
        "eventTypes" to listOf("se.mcf.iot.device.*", "se.mcf.iot.command.*")
    )
).blockingGet()
```

Notes:

- `tokens.freshAccessToken()` must return a token that is valid now (section 13).
- `hub.start()` and `hub.invoke(...)` with `blockingAwait` / `blockingGet` **block the thread**. Run them on a
  background thread, never on the main thread.
- Run `handle(envelope)` off the main thread, and post the screen change to the main thread.
- Read the `id` with `envelope.get("id").asString` and ignore repeats, as in section 5.
- Call `hub.onClosed { error -> ... }` to find out when the line is gone, then open it again after a pause and
  repeat the `Subscribe`. Check in your library version whether it reconnects by itself, and if it does not,
  write the retry yourself (0, 2, 5, 10, 30 seconds works well).

## 20. Cheat sheet

```
Address     https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1
Token       accessTokenFactory -> a token valid NOW (called again at every reconnect)
Transport   WebSockets only, skipNegotiation: true
Listen      conn.on('event', ev => ...)         register BEFORE start
Start       await conn.start()
Subscribe   await conn.invoke('Subscribe', { siteIds?, deviceIds?, eventTypes? })     {} = everything
Types       se.mcf.iot.command.{accepted,approval.requested,delivered,confirmed,failed,timedout,cancelled,rejected}.v1
            se.mcf.iot.device.*
Final       confirmed, failed, timedout, cancelled, rejected   (then GET /iot/v1/commands/{id} once)
Repeat      ignore a message whose id you already handled
Gap         sequence per device has no gaps: 41 then 43 means 42 is lost
Reconnect   Subscribe AGAIN, then reload (events never replayed): GET /iot/v1/events?since=..
Refused     401 token, 403 STREAM_NOT_PERMITTED permission -> follow presses with GET /commands/{id}
Shared      another organization's buttons send no events -> GET /commands/{id}
Never       never press again automatically after timedout
```
