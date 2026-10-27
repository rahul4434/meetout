# How to use SignalR in the MCX IoT app

Server: `https://microservices.alitis.se/alecomapigatewayiot`

SignalR keeps one connection open. The server **pushes** a message to your app when something happens
(for example a command was confirmed). Your app does not have to ask again and again.

> I could not test with a real login, so I never received a real event. The steps below come from the app's
> code and the platform's API guide. Section 8 lists what I could not confirm.

---

## 1. The address

```
https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1
```

It is at the gateway root, next to `/iot/v1`. It is **not** `.../iot/v1/hubs/iot/v1`.

## 2. What you need

- The **access token** from login. Give SignalR a function that returns the **newest** token. It is called
  again at every reconnect.
- The permission `IOT_EVENTS` View. Every role with `OPS_FUNCTIONS` View has it from platform 5.7.0.
- If it is missing you get `403 STREAM_NOT_PERMITTED`. With no or an old token you get `401`.

## 3. Step by step

### Step 1. Load the library

```html
<script src="https://cdn.jsdelivr.net/npm/@microsoft/signalr@8.0.7/dist/browser/signalr.min.js"></script>
```

(Android uses `com.microsoft.signalr`, see section 5.)

### Step 2. Build the connection

```javascript
const GATEWAY = 'https://microservices.alitis.se/alecomapigatewayiot';

const conn = new signalR.HubConnectionBuilder()
  .withUrl(GATEWAY + '/hubs/iot/v1', {
    accessTokenFactory: () => getFreshAccessToken(),
    skipNegotiation: true,
    transport: signalR.HttpTransportType.WebSockets
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .build();
```

### Step 3. Listen for events (before you start)

The server calls a method named **`event`**:

```javascript
const seen = new Set();

conn.on('event', ev => {
  if (seen.has(ev.id)) return;      // ignore a message you already got
  seen.add(ev.id);
  console.log(ev.type, ev.data);    // do something with it
});
```

### Step 4. Start, then call `Subscribe`

Nothing arrives until you call `Subscribe`.

```javascript
const filter = {
  siteIds: mySiteIds,   // optional: the siteId of your buildings (from the app menu)
  eventTypes: ['se.mcf.iot.device.*', 'se.mcf.iot.command.*']
};

await conn.start();
await conn.invoke('Subscribe', filter);
```

### Step 5. Handle reconnects

After the line breaks and comes back, the server has forgotten your filter. Subscribe again and reload
your data:

```javascript
conn.onreconnected(async () => {
  await conn.invoke('Subscribe', filter);
  reloadDataYouMissed();            // section 6
});
conn.onclose(() => startAskingTheServerEveryMinute());   // fallback
```

If `start()` or `Subscribe` fails (no internet, `401`, `403`), fall back to asking the server every minute.

## 4. `Subscribe` and the messages

### The filter

All three boxes are optional. An empty filter means everything the person may see.

| Box | Meaning |
| --- | --- |
| `siteIds` | Only events of these buildings |
| `deviceIds` | Only events of these devices |
| `eventTypes` | Only these types. A `*` at the end means "everything that starts like this" |

### One message

```json
{
  "id": "evt_8Kq2",
  "type": "se.mcf.iot.command.confirmed.v1",
  "subject": "cmd_5Rt7Yb",
  "time": "2026-10-09T08:00:03.412Z",
  "sequence": 41,
  "data": { "commandId": "cmd_5Rt7Yb", "deviceId": "dev_9Pz4Wc" }
}
```

| Box | Use |
| --- | --- |
| `id` | Ignore a message with an `id` you already got |
| `type` | What happened |
| `data.commandId` | Which button press it is about (for a scene: `groupRunId`) |
| `data.deviceId` | Which device |
| `time` | Remember the latest one, for catching up |
| `sequence` | A number per device, without gaps. A missing number means you lost a message |

### Event types

| Type (ends in `.v1`) | Meaning | Final? |
| --- | --- | --- |
| `se.mcf.iot.command.accepted` | The server took the press | No |
| `se.mcf.iot.command.approval.requested` | A second person must approve | No |
| `se.mcf.iot.command.delivered` | It reached the equipment | No |
| `se.mcf.iot.command.confirmed` | The equipment confirmed | Yes |
| `se.mcf.iot.command.failed` | It did not work | Yes |
| `se.mcf.iot.command.timedout` | No answer in time | Yes |
| `se.mcf.iot.command.cancelled` | Cancelled | Yes |
| `se.mcf.iot.command.rejected` | Refused by a rule | Yes |
| `se.mcf.iot.device.*` | A device reported something, for example `se.mcf.iot.device.input.changed.v1` | - |

## 5. Android (Kotlin)

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

There is no official SignalR client for Swift (iOS).

## 6. After the line was down

Messages sent while you had no line are **never replayed**. After a reconnect:

1. Call `Subscribe` again.
2. For a device you show: `GET /iot/v1/devices/{deviceId}/state`.
3. Or read what you missed: `GET /iot/v1/events?deviceId={deviceId}&since={last time you processed}`.
4. For a press still running: `GET /iot/v1/commands/{commandId}`.

Buttons shared with you by **another organization** send no events. Follow those with
`GET /iot/v1/commands/{commandId}`.

## 7. Test it

**Check the token and permission** (works in Postman):

```
POST https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1/negotiate?negotiateVersion=1
Authorization: Bearer <access token>
(empty body)
```

`200` = fine. `401` = token missing or old. `403` = permission missing. Send an empty body, or the web
server answers `411`.

**Watch real events from a terminal** (Postman cannot hold a SignalR line). Run `npm install @microsoft/signalr`,
then:

```javascript
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
  .build();

conn.on('event', ev => console.log(ev.time, ev.type, JSON.stringify(ev.data)));
conn.onreconnected(() => conn.invoke('Subscribe', filter));

(async () => {
  await conn.start();
  await conn.invoke('Subscribe', filter);
  console.log('Listening. Press a button to see events.');
})();
```

Run it with `TOKEN=<your access token> node watch.js`, then press a button. You should see
`command.accepted`, then `command.delivered`, then `command.confirmed`.

## 8. What I could not confirm

- The answer of `Subscribe` and its error codes.
- A real event from production (I had no login).
- The full list of device event types, and any limits on connections.
- I have not run the terminal program above.

The platform's own file `Documentation/Api/Field-App-Event-Stream.md` should cover these. I was not given it.

---

## Cheat sheet

```
Address     https://microservices.alitis.se/alecomapigatewayiot/hubs/iot/v1
Token       accessTokenFactory -> newest access token
Transport   WebSockets only, skipNegotiation: true
Listen      conn.on('event', ev => ...)   (before start)
Start       await conn.start()
Subscribe   await conn.invoke('Subscribe', { siteIds?, deviceIds?, eventTypes? })
Types       se.mcf.iot.command.*   se.mcf.iot.device.*
Repeat      ignore a message whose id you already got
Reconnect   Subscribe again + reload what you missed (never replayed)
Refused     401 = token, 403 STREAM_NOT_PERMITTED = permission -> ask the server every minute instead
```
