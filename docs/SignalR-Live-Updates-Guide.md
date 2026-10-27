# SignalR live updates: the complete guide for the MCX IoT app

For: frontend and mobile developers, including beginners.
App version this was written against: **1.2.6** (Android WebView app with a single `index.html`).
Server: production gateway `https://microservices.alitis.se/alecomapigatewayiot`.

> **Honesty box.** I did not have a real production account, so I never opened an authenticated SignalR
> line. What is written here comes from three sources: (1) the app's own code, which I read line by line,
> (2) the files `Field-App-API-Guide.md`, `iot-v1.openapi.json` and the Postman collection, and (3) a few
> unauthenticated calls to production (they answered `401`, which proves the address and the need for a
> ticket). Each part below says which source it comes from. The things I could not check are collected in
> section 16.

---

## Contents

1. What SignalR is, in plain words
2. Words you will meet
3. The big picture (one diagram)
4. The addresses
5. Who may connect (login ticket and permission)
6. Step by step: opening the line
7. `Subscribe`: telling the server what you want
8. What a message looks like
9. Every event type and how to react
10. What this app really does with a message
11. The states of the line and the Profile screen
12. After the line was down: catching up
13. Code you can copy (browser, Android, terminal)
14. Test it without the app
15. When something is wrong
16. What I could not confirm
17. Cheat sheet

---

## 1. What SignalR is, in plain words

Normally an app **asks** the server a question and gets an answer. That is called a request. If the app
wants to know whether a gate command worked, it can ask again and again every second. That works, but it is
slow (you only learn at the next question) and wasteful (most answers say "nothing new").

**SignalR turns this around.** The app opens one long-lasting connection to the server and says once:
"tell me when something happens in these buildings." After that the **server pushes a small message** to the
phone at the moment something happens. The phone does not have to ask.

Think of it like a doorbell instead of walking to the door every minute to check whether anyone is there.

In this platform the "something" is, for example:

- a command you sent was accepted, delivered, confirmed or failed,
- a device reported something (an input changed).

The long-lasting connection is called **the line** in this guide. Every message that comes down the line is
called an **event**.

---

## 2. Words you will meet

| Word | Meaning |
| --- | --- |
| **SignalR** | A Microsoft technology for pushing messages from a server to apps. It hides the difficult network parts |
| **Hub** | The "room" on the server that you connect to. Here it is `/hubs/iot/v1` |
| **WebSocket** | The kind of network connection that stays open. SignalR uses it here |
| **Negotiate** | An optional first call where the client and server agree on how to talk. The app skips it |
| **Subscribe** | A method on the hub that you call to say which events you want |
| **Event** | One message pushed by the server. Its method name on the phone is `event` |
| **CloudEvents** | A standard shape for event messages (`id`, `type`, `time`, `data` and so on) |
| **Ticket / token** | The login proof (the access token) that you got when you signed in |
| **Reconnect** | Opening the line again after it broke |
| **Catch-up** | Asking the normal API for what you missed while the line was down |
| **Polling** | Asking the server again and again instead of being pushed to. The fallback |
| **Gateway** | The single front door of the platform: `https://microservices.alitis.se/alecomapigatewayiot` |
| **Command** | One press of a button, with its own id (`commandId`) |
| **Group run** | One press of a scene (several buttons). Its id is `groupRunId` |

---

## 3. The big picture

```
 PHONE (the app)                         GATEWAY / SERVER
 ───────────────                         ────────────────
 1. sign in ───────────────────────────► check password
    ◄─────────────── login ticket ─────  (access token)

 2. load buttons and buildings ────────► GET /iot/v1/me/app-menu, /sites ...
    ◄────────────────── answers ───────

 3. load the SignalR program (jsDelivr)

 4. open WebSocket  ───────────────────► wss://.../hubs/iot/v1  (ticket attached)
                                         checks ticket + permission IOT_EVENTS View
                                         401 / 403 if refused

 5. Subscribe({siteIds, eventTypes}) ──► remembers the filter for this line

                 ...  quiet  ...

 6. person holds a button ─────────────► POST /iot/v1/functions/{id}/commands
    ◄──── 202 accepted ────────────────
                                         the equipment is told, it answers
    ◄──── event: command.delivered ────  (pushed)
    ◄──── event: command.confirmed ────  (pushed)

 7. app wakes its follow-up ───────────► GET /iot/v1/commands/{commandId}
    ◄──── status: Confirmed ───────────
```

The important thing: the **press itself** is a normal request (step 6). SignalR only tells the phone
*sooner* that the result is ready. The final status is still read with a normal request (step 7).

---

## 4. The addresses

Source: API guide, section 6, and the app code.

| What | Address |
| --- | --- |
| Gateway | `https://microservices.alitis.se/alecomapigatewayiot` |
| The hub (the line) | `https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1` |
| Negotiate (optional check) | `https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1/negotiate?negotiateVersion=1` |
| Normal API | `https://microservices.alitis.se/alecomapigatewayiot/iot/v1/...` |

The hub is at the **root of the gateway, next to `/iot/v1`**, not inside it. A common mistake is to write
`.../iot/v1/hubs/iot/v1`.

A WebSocket address starts with `wss://` instead of `https://`. The SignalR program changes this by itself,
so you give it the `https://` address.

---

## 5. Who may connect

Source: API guide section 6 and the Postman collection, folder "11. Live updates".

### 5.1 The login ticket

The line needs the **same access token** as the normal API. With a SignalR client you do not add it to a
header yourself. You give the client a **function that returns the token** (`accessTokenFactory`). The client
puts it in the `access_token` part of the address.

Give it the **newest** token every time. The function is called again at every reconnect, and an old token
makes the reconnect fail with `401`.

### 5.2 The permission

| Who | Needs |
| --- | --- |
| A person | `IOT_EVENTS` View. From platform release **5.7.0** (`IoT.1.105`) every role that has `OPS_FUNCTIONS` View gets it automatically, so a normal field user can connect |
| A machine client | The scope `events.subscribe` |

### 5.3 When it is refused

| Answer | Meaning |
| --- | --- |
| `401` (`UNAUTHORIZED`) | No ticket, or an old ticket |
| `403` `STREAM_NOT_PERMITTED` | The person lacks the permission. Typical for an older site, a role created after the upgrade, or a role with `OPS_FUNCTIONS` **Manage** but not **View** |

No API call tells the app in advance whether the permission exists (`/me/access` shows access scopes, not
menu permissions). So the app **tries to connect** and, if it is refused, falls back to polling.

### 5.4 Shared (delegated) buttons

A button shared with you by **another organization** sends **no events** to you, because an event belongs to
its own organization. Follow those presses with `GET /iot/v1/commands/{commandId}`.

---

## 6. Step by step: opening the line

This is exactly what the app does (function `stream.start` in the app's API client).

### Step 1. Load the SignalR program

The app loads Microsoft's browser library from jsDelivr:

```
https://cdn.jsdelivr.net/npm/@microsoft/signalr@8.0.7/dist/browser/signalr.min.js
```

If the phone cannot reach jsDelivr, the line is never opened and the app stays in **polling** mode.

### Step 2. Build the connection

```javascript
const conn = new signalR.HubConnectionBuilder()
  .withUrl('https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1', {
    accessTokenFactory: () => getFreshAccessToken(),
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .configureLogging(signalR.LogLevel.Warning)
  .build();
```

| Setting | Why |
| --- | --- |
| `accessTokenFactory` | Gives the SignalR client a fresh token every time it needs one |
| `skipNegotiation: true` | Goes straight to the WebSocket. Only allowed together with `transport: WebSockets` |
| `transport: WebSockets` | Use only WebSockets (no long-polling fallback inside SignalR) |
| `withAutomaticReconnect([0, 2000, 5000, 10000, 30000])` | Retry after 0, 2, 5, 10 and 30 seconds, then give up |
| `LogLevel.Warning` | Only warnings in the log |

### Step 3. Register the listener **before** you start

```javascript
conn.on('event', ev => {
  // one message from the server
});
```

The server calls a method named exactly **`event`** on the phone. Register `conn.on('event', ...)` before
`conn.start()`, or you can miss the first messages.

### Step 4. Register the line-state callbacks

```javascript
conn.onreconnecting(() => setState('connecting'));
conn.onreconnected(async () => {
  setState('live');
  await conn.invoke('Subscribe', filter);   // the server forgot your filter: send it again
  resync();                                 // reload what you missed (section 12)
});
conn.onclose(() => setState('polling'));    // gave up: fall back to asking the server
```

### Step 5. Start, then subscribe

```javascript
await conn.start();
await conn.invoke('Subscribe', filter);
setState('live');
```

Nothing is sent to you **before** `Subscribe`.

### Step 6. Handle failure

If `start()` or `Subscribe` fails (no internet, `401`, `403`, jsDelivr blocked), set the state to `polling`.
The app keeps working, only slower (section 10).

---

## 7. `Subscribe`: telling the server what you want

You call the hub method `Subscribe` once, with one object. All three boxes are optional.

| Box | Meaning | What the app sends |
| --- | --- | --- |
| `siteIds` | Only events of these buildings | The `siteId` of every building the person has buttons for |
| `deviceIds` | Only events of these devices | Nothing |
| `eventTypes` | Only these kinds of event. A `*` at the end means "everything that starts like this" | `se.mcf.iot.device.*` and `se.mcf.iot.command.*` |

```json
{
  "siteIds": ["site_7Hq2k9", "site_4Lm3Xp"],
  "eventTypes": ["se.mcf.iot.device.*", "se.mcf.iot.command.*"]
}
```

Rules:

- An **empty filter** means everything the person is allowed to see.
- **Call `Subscribe` again after every reconnect.** The server forgets the filter when the line breaks.
- Call it again when your list of buildings changes. The app does this each time it reloads the buttons:
  if the line already exists, it calls `Subscribe` again with the new list.
- Where to get the `siteId`s: every button in the app menu has a `siteId` (see the main guide, sections 6 and 7).

I did not see the **answer** of `Subscribe` (the file that documents it was not given to me). The app ignores
the answer and treats "no error" as success.

---

## 8. What a message looks like

Each message is a **CloudEvents** object.

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

(The fields come from the OpenAPI schema `IotEventDto`; the example values are from the API guide.)

| Box | Meaning | What to do |
| --- | --- | --- |
| `id` | A unique number for this message | Remember the ones you saw. **Ignore a repeat.** Repeats are normal after a reconnect |
| `type` | What happened (section 9) | Choose the reaction from it |
| `subject` | What it is about. For a command, the command id | Match it to the thing on the screen |
| `time` | When it happened (UTC) | Remember the latest one, for catching up |
| `sequence` | The number of this event in its device's stream, **without gaps** | If a number is missing, you lost a message: catch up |
| `dataschema` | A path that describes `data`: `/iot/v1/schemas/events/{type}` | Read it when you need the exact boxes of `data` |
| `organisationid` | The organization the event belongs to | Ignore events of another organization than the chosen one |
| `data` | The details. For commands: `commandId` (and `groupRunId` for a scene member) and `deviceId` | Use it to find the press or the device |

The very same events are also sent as **webhooks** to partners and as **northbound MQTT** messages, and
`GET /iot/v1/events` lists them. So what you see in the stream equals what the list call shows.

---

## 9. Every event type and how to react

### 9.1 Command events (what happened to a button press)

Source: the description of `GET /iot/v1/events` in the OpenAPI file. The full type name always ends in
`.v1`, for example `se.mcf.iot.command.confirmed.v1`.

| Type | What happened | Final? | What the app does now | What you could show |
| --- | --- | --- | --- | --- |
| `se.mcf.iot.command.accepted` | The server took the press | No | Wakes the follow-up | "Accepted" |
| `se.mcf.iot.command.approval.requested` | A second person must approve first | No | Wakes the follow-up | "Waiting for approval". The other person sees an Approve button |
| `se.mcf.iot.command.delivered` | The command reached the equipment | No | Wakes the follow-up | "Sent", still waiting for feedback |
| `se.mcf.iot.command.confirmed` | The equipment confirmed | **Yes** | Wakes the follow-up | "Confirmed", turn the button green |
| `se.mcf.iot.command.failed` | It did not work | **Yes** | Wakes the follow-up | The red state |
| `se.mcf.iot.command.timedout` | No answer in time | **Yes** | Wakes the follow-up | "Check on site before you press again" |
| `se.mcf.iot.command.cancelled` | Cancelled | **Yes** | Wakes the follow-up | "Cancelled" |
| `se.mcf.iot.command.rejected` | Refused by a rule | **Yes** | Wakes the follow-up | The reason from the command |

### 9.2 Device events (what the equipment reports)

| Type | What happened |
| --- | --- |
| `se.mcf.iot.device.*` | Anything a device reports. One example from the API guide: `se.mcf.iot.device.input.changed.v1` (an input changed) |

The files I have do not list every device event type. To see which ones your sites really send, watch the
stream (section 14) or read `GET /iot/v1/events`.

A good reaction to a device event: ask `GET /iot/v1/devices/{deviceId}/state` and redraw that device.

### 9.3 Events that are *not* in the stream

Alarms (conditions) and notifications are **not** pushed by this stream in the way the app uses it. The app
reads them with `GET /iot/v1/notifications` and `GET /iot/v1/conditions`.

---

## 10. What this app really does with a message

This is the part that surprises people, so read it carefully. Source: the app code (`stream.start` and
`follow`).

When a message arrives, the app does exactly this:

1. If the `id` was seen before, **throw it away**.
2. Look in `data` for a `commandId` (a button) or a `groupRunId` (a scene).
3. If the app is **following that press right now** (the sheet is open and waiting), it **wakes the
   follow-up at once**. The follow-up then asks `GET /iot/v1/commands/{commandId}` (or
   `GET /iot/v1/group-runs/{groupRunId}`) and updates the screen.
4. It passes the message on inside the app under the name `event`. **No screen listens to that yet.**

So a message never changes the Events list, device states or the buttons by itself.

| Screen | Changed by a live message? | How it really gets its data |
| --- | --- | --- |
| Button sheet: the status panel | **Yes** (the next check happens at once) | The follow-up asks the server. Without SignalR it still works, only slower |
| Scene sheet: the list of buttons | **Yes**, the same way | The same follow-up on the group run |
| Events list (messages and alarms) | **No** | `GET /iot/v1/notifications` and `GET /iot/v1/conditions` every **60 seconds** while the screen is visible, when the app returns to the front, and after a reconnect |
| Home, buildings, buttons | **No** | When the person signs in, switches organization, returns to the app, taps "Reload buttons", and after a reconnect |
| Profile: "Live updates" | Shows the state of the line only | Section 11 |

### 10.1 The follow-up without SignalR

`follow()` asks for the status again and again. The waiting time grows:

| Ask number | Wait before the next ask |
| --- | --- |
| 1 to 3 | 0.8 seconds |
| 4 to 9 | 1.5 seconds |
| 10 and later | 3 seconds |

It stops when the command is **final** (`isFinal: true`), when the server answers `404`, `403` or `401`, or
after **2 minutes**. A live message cuts the waiting short: it wakes the follow-up immediately.

### 10.2 So what is SignalR good for here?

In this app it is a **speed-up for button results**. If you want the Events list, device states or buttons
to change at the moment something happens, you have to listen to `event` yourself and update the screen
(section 9 gives a reaction for each type).

---

## 11. The states of the line and the Profile screen

The app keeps one word for the state. It is shown in **Profile, row "Live updates"**.

| State | Word on screen | Meaning | Moves on when |
| --- | --- | --- | --- |
| `off` | Off | No line yet | The buttons are loaded and the line is started |
| `connecting` | Connecting... | Opening the line, or it broke and the app is retrying | Success: `live`. Closed for good: `polling` |
| `live` | Live | The line is open and the filter is set | The line breaks: `connecting` |
| `polling` | Every minute | No line. The app only checks the server | The next time the buttons load, the app tries to open the line again |

In demo mode there is no server, so the state goes straight to `polling`.

```
off ──► connecting ──► live ──► connecting ──► live
                 │                    │
                 └──────► polling ◄───┘   (when it gives up)
```

---

## 12. After the line was down: catching up

Messages sent while the phone had no line are **never replayed**. Do this after every reconnect:

1. Call `Subscribe` again with the same filter.
2. Reload the buttons (the app calls this "resync") so button states are right.
3. Reload the Events list: `GET /iot/v1/notifications` and `GET /iot/v1/conditions`.
4. For a device you show: `GET /iot/v1/devices/{deviceId}/state`.
5. Or read exactly what you missed, in order:
   `GET /iot/v1/events?deviceId={deviceId}&since={the last time you processed}`.
6. For a press that was still running: `GET /iot/v1/commands/{commandId}` once.

Tip: use the `sequence` number to detect a gap. If a device's next event is not the previous number plus
one, you lost something, so do step 5.

---

## 13. Code you can copy

### 13.1 Browser / WebView (what the app does)

```javascript
// 1. load the library once:
// <script src="https://cdn.jsdelivr.net/npm/@microsoft/signalr@8.0.7/dist/browser/signalr.min.js"></script>

const GATEWAY = 'https://microservices.alitis.se/alecomapigatewayiot';
const filter = {
  siteIds: mySiteIds,                                  // may be left out
  eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*']
};
const seen = new Set();

const conn = new signalR.HubConnectionBuilder()
  .withUrl(GATEWAY + '/hubs/iot/v1', {
    accessTokenFactory: () => getFreshAccessToken(),
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .build();

conn.on('event', ev => {
  if (!ev || (ev.id && seen.has(ev.id))) return;       // ignore repeats
  if (ev.id) seen.add(ev.id);
  const d = ev.data || {};
  const id = d.commandId || d.groupRunId;
  if (id) wakeFollowUp(id);                            // ask GET /commands/{id} now
  // add your own reactions here (section 9)
});

conn.onreconnecting(() => setStreamState('connecting'));
conn.onreconnected(async () => {
  setStreamState('live');
  await conn.invoke('Subscribe', filter);
  reloadButtonsAndEvents();                            // catch up (section 12)
});
conn.onclose(() => setStreamState('polling'));

async function startLine() {
  try {
    setStreamState('connecting');
    await conn.start();
    await conn.invoke('Subscribe', filter);
    setStreamState('live');
  } catch (e) {
    setStreamState('polling');                         // 401, 403, no internet ...
  }
}
```

### 13.2 Android (Kotlin, official client `com.microsoft.signalr`)

From the API guide. Add the package `com.microsoft.signalr:signalr` to Gradle.

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

Do the work of `handle` off the main thread, and update the screen on the main thread. On iOS: the API
guide notes there is **no official SignalR client for Swift**.

### 13.3 Terminal (Node.js), to watch real events

Install: `npm install @microsoft/signalr`. I wrote this from the app's code. **I have not run it against the
production server** because I have no ticket.

```javascript
const signalR = require('@microsoft/signalr');
const GATEWAY = 'https://microservices.alitis.se/alecomapigatewayiot';
const TOKEN = process.env.TOKEN;                       // the login ticket
const filter = { eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*'] };

const conn = new signalR.HubConnectionBuilder()
  .withUrl(GATEWAY + '/hubs/iot/v1', {
    accessTokenFactory: () => TOKEN,
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .build();

conn.on('event', ev => console.log(ev.time, ev.type, JSON.stringify(ev.data)));
conn.onreconnected(() => conn.invoke('Subscribe', filter));

(async () => {
  await conn.start();
  await conn.invoke('Subscribe', filter);
  console.log('Listening. Press a button in the app to see events.');
})();
```

Run it with `TOKEN=<your ticket> node watch.js`.

The platform's own guide also names a ready-made tool,
`node Postman/tools/iot_stream_client.mjs --token <ticket>`. I was not given that file.

---

## 14. Test it without the app

### 14.1 Check the ticket and the permission (works in Postman)

Postman cannot hold a SignalR line, but it can do the one HTTP step:

```
POST https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1/negotiate?negotiateVersion=1
Authorization: Bearer <your access token>
(empty body)
```

| Answer | Meaning |
| --- | --- |
| `200` | The ticket, the address and the permission are all fine |
| `401` | No ticket or an old ticket |
| `403` | Usually the permission is missing (`STREAM_NOT_PERMITTED`) |

What I really tested (no ticket): production answered `401` with this body, both for `negotiate` and for a
WebSocket attempt:

```json
{
  "success": false,
  "data": null,
  "message": "Authentication is required.",
  "errors": [{ "code": "UNAUTHORIZED", "message": "Authentication is required.", "domain": "Gateway" }]
}
```

Note: sending `negotiate` **without** a body and without `Content-Length: 0` gives `411` from the web server
(IIS). Send an empty body.

### 14.2 Watch real events

Use the Node program in section 13.3. Then press a button in the app and look for
`command.accepted`, `command.delivered`, `command.confirmed` in the terminal.

### 14.3 A good test order

1. Sign in (see the main guide, section 5) and keep the access token.
2. `negotiate` with the token. Expect `200`.
3. Run the Node program. Expect "Listening".
4. Press a button in the app or in Postman (`POST /iot/v1/functions/{id}/commands` with an `Idempotency-Key`).
5. See the events. Compare with `GET /iot/v1/commands/{commandId}`.
6. Turn off the Wi-Fi, press nothing, turn it on. See the reconnect. Then do the catch-up (section 12).

---

## 15. When something is wrong

| What you see | Likely cause | What to do |
| --- | --- | --- |
| Profile says "Every minute" and never "Live" | The phone cannot load the SignalR program from jsDelivr, or the line was refused | Check the phone's internet and that `cdn.jsdelivr.net` is reachable. Try `negotiate` (14.1). If it answers `403`, the person lacks the permission |
| `403 STREAM_NOT_PERMITTED` | The role lacks `IOT_EVENTS` View | Ask the administrator, or live with the one-minute checks. The app already falls back by itself |
| `401` when connecting | The ticket is old or missing | Make the ticket function return a **fresh** ticket (refresh it if it ends soon) |
| It says "Live" but the Events list does not change at once | By design (section 10): the Events list does not listen to messages | Wait up to a minute, or listen to `event` and refresh the list yourself |
| A press status changes late | The line is down, so the follow-up asks only every 0.8 to 3 seconds | Check the Profile word. The status still arrives, just slower |
| No messages at all although "Live" | `Subscribe` was not called, or `siteIds` does not include the building | Call `Subscribe` after the line opens. Test with an empty filter |
| The same message twice | Normal after a reconnect | Ignore a message whose `id` you already saw |
| Events missing after the phone slept or lost signal | Missed messages are never replayed | Do the catch-up (section 12) |
| No events for a shared (delegated) button | Events belong to the other organization | Follow that press with `GET /iot/v1/commands/{commandId}` |
| `411` on `negotiate` | The body is missing | Send an empty body with `Content-Length: 0` |
| Works in the terminal, not in a browser | A browser cannot add headers to a WebSocket, and the server may refuse the browser's origin | I did not test this. Use the phone app or the terminal program for real checks |

---

## 16. What I could not confirm

- The **answer of `Subscribe`** and any error codes it can throw.
- A **real event** from production. I only know the shape from the API guide and the OpenAPI schema.
- The list of **device event types** (only `se.mcf.iot.device.input.changed.v1` is named as an example).
- **Limits**: how many lines one person may open, how often, how big a filter may be.
- What happens while the **Android app is in the background** (whether the line stays open).
- Whether the platform sends events for **alarms and notifications** on this stream. The app does not rely on that.

The platform's own file **`Documentation/Api/Field-App-Event-Stream.md`** is said to describe the reconnect
recipe, every event type the app cares about, the limits and the refusal codes. I was not given it. If you
send it, this guide can be completed.

---

## 17. Cheat sheet

```
Address     https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1
Ticket      accessTokenFactory -> newest access token (called again at every reconnect)
Transport   WebSockets only, skipNegotiation: true
Listen      conn.on('event', ev => ...)          (register BEFORE start)
Start       await conn.start()
Subscribe   await conn.invoke('Subscribe', { siteIds?, deviceIds?, eventTypes? })
Types       se.mcf.iot.command.*   se.mcf.iot.device.*
Repeat      ignore a message whose id you saw
Reconnect   after 0, 2, 5, 10, 30 seconds; then Subscribe AGAIN and reload data
Missed      never replayed: GET /iot/v1/events?deviceId=..&since=..  or  /devices/{id}/state
Refused     401 = ticket, 403 STREAM_NOT_PERMITTED = permission  -> fall back to polling
Delegated   no events for another organization's buttons -> GET /iot/v1/commands/{id}
App fact    a message only speeds up the status of a press; screens do not listen to it
```
