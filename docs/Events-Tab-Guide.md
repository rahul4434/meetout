# The Events tab: where its data comes from (API and SignalR)

For: frontend and mobile developers, including beginners.
App version this describes: **1.2.6**. Server: `https://microservices.alitis.se/alecomapigatewayiot`.

> **How much is tested?** I read the app's code line by line and the platform's API files. I had **no real
> login**, so I never saw a real answer for notifications or alarms. The JSON examples below are
> **illustrative**: built from the field names in the OpenAPI file. Section 12 lists what I could not confirm.

## Short answer

The Events tab is filled by **normal API calls**, not by SignalR:

| Rows | Source |
| --- | --- |
| Messages (command results, "Approval needed...") | `GET /iot/v1/notifications` |
| Alarms ("Gate B sensor offline...") | `GET /iot/v1/conditions` |
| Presses you made on this phone | Created **locally** when your press finishes |

The app asks again **every 60 seconds**, when the app comes back to the front, after a reconnect, and after
you tap Approve or Acknowledge. **No SignalR message is used to build this list.** SignalR only speeds up the
status of a button press. If you want the tab to update at the moment something happens, section 9 shows how.

## Contents

1. What the Events tab shows
2. The three sources, one by one
3. Call 1: notifications
4. Call 2: alarms (conditions)
5. Local rows (your own presses)
6. How the three are joined and drawn
7. The Approve button
8. The Acknowledge button
9. Making the tab live with SignalR
10. When the tab refreshes (full list)
11. Errors and what the app does
12. What I could not confirm
13. Cheat sheet
14. Worked example: reading a screenshot row by row

---

## 1. What the Events tab shows

```
Events                          <- title
2 open alerts                   <- count of open red rows
[ All ] [ Commands ] [ Alerts ] <- filter chips
TODAY
  Approval needed: Bollards - Lower              16:52   [Approve]
  Requested by fake.colleague · Maintenance · Central depot
  Gate B sensor offline                          16:20   [Acknowledge]
  Central depot · Warning · Active
YESTERDAY
  Depot gateway is back online                   14:58
```

Each row has: an icon (blue for a command, red for an alert), a title, a small line, a time, and sometimes a
button (**Approve** or **Acknowledge**). The tab icon in the bottom bar has a small orange dot when there is
something unread. The dot goes away when you open the tab. It comes back when a refresh finds an unread notification, and when
one of your own presses ends.

## 2. The three sources, one by one

| Source | Request or origin | Becomes | Refreshed |
| --- | --- | --- | --- |
| Notifications | `GET /iot/v1/notifications` (page 1, 50 items) | Blue or red rows, sometimes with **Approve** | Every 60 s and the other times in section 10 |
| Alarms | `GET /iot/v1/conditions` (50 items, only **open** ones) | Red rows with **Acknowledge** | Together with the notifications |
| Local presses | The press the person made on this phone, when it becomes final | A blue row "Name · Confirmed" | At once, when the press ends |

The two requests are sent **together** (in parallel). If one of them fails, the app treats it as "nothing"
and still draws the other one.

## 3. Call 1: notifications

```http
GET /iot/v1/notifications?page=1&pageSize=50
Authorization: Bearer <access token>
```

The app sends `page` and `pageSize` only. The API also accepts these filters:

| Parameter | Meaning |
| --- | --- |
| `unreadOnly` | `true` = only notifications not yet read. The unread count is `totalCount` of `?unreadOnly=true&pageSize=1` |
| `type` | One notification type (the types are listed by `GET /iot/v1/notifications/preferences`) |
| `page`, `pageSize` | Page number from 1. Default 50, at most 200 |

**Only a signed-in person may call it.** A partner client or service token gets `403 USER_TOKEN_REQUIRED`.

The answer (illustrative; the box names are from the API guide):

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "notificationId": "ntf_9Zr3",
        "type": "command.approval.requested",
        "severity": "Info",
        "title": "Approval needed: Bollards - Lower",
        "body": "Requested by fake.colleague · Maintenance",
        "siteId": "site_4Lm3Xp",
        "functionId": "fn_8Hd2Vk",
        "commandId": "cmd_7Wq1Zt",
        "occurredAt": "2026-10-09T16:52:10Z",
        "read": false
      },
      {
        "notificationId": "ntf_9Zr2",
        "type": "command.failed",
        "severity": "Warning",
        "title": "Garage 12 - Close was not confirmed",
        "body": "The device did not report a new state in time.",
        "siteId": "site_7Hq2k9",
        "functionId": "fn_6Jk1Lz",
        "commandId": "cmd_3Df9Rs",
        "occurredAt": "2026-10-09T16:16:40Z",
        "read": false
      }
    ],
    "page": 1, "pageSize": 50, "totalCount": 2, "totalPages": 1,
    "hasPreviousPage": false, "hasNextPage": false
  }
}
```

(The `type` words above are examples. The real notification types are listed by
`GET /iot/v1/notifications/preferences`.)

### How the app turns one notification into one row

```javascript
// n is one item of data.items
const ts       = Date.parse(n.occurredAt) || Date.now();                    // the time of the row
const isAlert  = /warn|crit|high|err/i.test(n.severity || '');              // red or blue
const kind     = isAlert ? 'alert' : /geo/i.test(n.type || '') ? 'geo' : 'cmd';
const open     = isAlert && !n.read;                                        // counts in "N open alerts"
const title    = n.title;
const small    = [n.body, buildingName(n.siteId)].filter(Boolean).join(' · ');
const hasApprove = /approv/i.test(n.type || '') && n.commandId && !n.read;  // shows the Approve button
```

| Box of the notification | Used for |
| --- | --- |
| `occurredAt` | The time shown (hh:mm, Stockholm time) and the group (Today, Yesterday, a date) |
| `severity` | Contains warn, crit, high or err: a **red alert row**. Otherwise a blue row |
| `read` | An unread red row counts as an "open alert". An unread notification also sets the orange dot |
| `type` | Contains "geo": a geofence row. Contains "approv" (with a `commandId`, and unread): shows **Approve** |
| `title` | The bold line |
| `body` + `siteId` | The small line: the text, then the building name (looked up from the buildings the app already loaded) |
| `commandId` | The press to approve (section 7) |
| `notificationId` | Used to mark it read after an approve |

## 4. Call 2: alarms (conditions)

An **alarm** here is a *supervision condition*: a rule watches something (for example "Gate B sensor is
offline") and raises a condition when it is true.

```http
GET /iot/v1/conditions?pageSize=50
Authorization: Bearer <access token>
```

Filters the API accepts (the app uses none, so it gets the alarms of all buildings):

| Parameter | Meaning |
| --- | --- |
| `state` | `open` (**the default**: pending or raised), `pending`, `raised`, `cleared`, `all` |
| `severity` | `Info`, `Warning` or `High` |
| `siteId`, `deviceId`, `ruleId` | Only this building, device or rule |
| `page`, `pageSize` | As above |

Because the default is `open`, the tab only shows alarms that are still pending or raised. A condition that
has been **cleared** is not in the list.

Needs `IOT_SUPERVISION` View. A person without it gets `403`, and the app simply shows no alarm rows.

The answer (illustrative; the boxes are from the schema `IotConditionDto`):

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "conditionId": "cnd_4Tn8",
        "ruleId": "rul_2Ab9",
        "ruleName": "Gate B sensor offline",
        "ruleKind": "DeviceOffline",
        "state": "Raised",
        "level": 1,
        "severity": "Warning",
        "subject": { "siteId": "site_4Lm3Xp", "deviceId": "dev_9Pz4Wc", "functionId": null, "channel": null },
        "since": "2026-10-09T16:20:00Z",
        "raisedAt": "2026-10-09T16:20:30Z",
        "acknowledgedAt": null,
        "acknowledgedBy": null,
        "clearedAt": null,
        "clearReason": null,
        "thresholdSeconds": 60
      }
    ],
    "page": 1, "pageSize": 50, "totalCount": 1, "totalPages": 1,
    "hasPreviousPage": false, "hasNextPage": false
  }
}
```

### How the app turns one condition into one row

```javascript
const ts   = Date.parse(c.since) || Date.now();
const open = !c.acknowledgedAt;                               // not yet acknowledged
row = {
  kind: 'alert',                                              // always a red row
  open,
  title: c.ruleName,
  small: [buildingName(c.subject.siteId), c.severity, open ? 'Active' : 'Acknowledged'].filter(Boolean).join(' · '),
  hasAcknowledge: open                                        // shows the Acknowledge button
};
```

So the small line looks like `Central depot · Warning · Active`.

## 5. Local rows (your own presses)

When a press made on this phone becomes **final**, the app adds one row to the list **at once**, without
asking the server:

| Part | Value |
| --- | --- |
| Title | `<button name> · <status word>`, for example `Open the main gate · Confirmed` |
| Small line | `<category> · by <the person's name>`, for example `Gates · by rahul.yadav` |
| Colour | Red if the press failed or was cancelled or the status was a warning. Blue otherwise |
| Time | The time the press finished |

These rows exist only on this phone and only until the app is restarted. The server's own record of the same
press comes back later as a notification (section 3), so you may see two rows for one press: the local one at
once, and the server's one after the next refresh.

## 6. How the three are joined and drawn

1. The two lists (local rows, and the rows made from notifications and alarms) are joined into one list.
2. The list is **sorted by time, newest first**.
3. The rows are grouped by day: **TODAY**, **YESTERDAY**, and for older rows a date like `7 OCT`.
4. The filter chips choose which rows to draw:

| Chip | Shows |
| --- | --- |
| **All** | Every row |
| **Commands** | Only rows of kind `cmd` (blue command rows) |
| **Alerts** | Every row that is **not** a command: red alerts and geofence rows |

5. The title line "N open alerts" counts the rows that are red **and** still open.
6. If the list is empty the tab says "Nothing here yet."
7. Times are shown in Stockholm time (`hh:mm`).

## 7. The Approve button

It appears only on a notification row where the type contains "approv", there is a `commandId`, and the
notification is unread. It is for the **second person** of a two-person button.

```http
POST /iot/v1/commands/{commandId}/approve
Authorization: Bearer <access token>
Content-Type: application/json

{}
```

What the app does:

1. Send the request above.
2. On success: show "Approved - the command will run", mark that notification read
   (`POST /iot/v1/notifications/{notificationId}/read`), and refresh the list at once.
3. On an error: show the error message as a short toast. Section 11 lists the codes.

Rules from the OpenAPI file: only a **person** with `OPS_APPROVE` Manage may approve. The person who pressed
can never approve their own command. The command must still be `PendingApproval` and inside its approval time.

## 8. The Acknowledge button

It appears on an alarm row that has no `acknowledgedAt`.

```http
POST /iot/v1/conditions/{conditionId}/acknowledge
Authorization: Bearer <access token>
Content-Type: application/json

{}
```

Needs `IOT_SUPERVISION` Manage. Acknowledging stops the alarm's escalation.

What the app does: on success it shows "Alarm acknowledged" and refreshes the list at once (the row then says
"Acknowledged", and the open count goes down). On `403` it shows "You can't acknowledge alarms". On another
error it shows the message.

## 9. Making the tab live with SignalR

Today a new alarm can take up to a minute to appear. To show it at once, listen to the SignalR `event` (see
`SignalR-Live-Updates-Guide.md`) and **call the same refresh**. The easiest and safest way is to refresh the
list when an event arrives, with a short wait so several events cause only one refresh:

```javascript
let refreshTimer = null;
function refreshEventsSoon() {                       // call this instead of loadInbox() directly
  clearTimeout(refreshTimer);
  refreshTimer = setTimeout(loadInbox, 500);         // many events in a row -> one refresh
}

conn.on('event', ev => {
  const t = ev.type || '';
  if (t.startsWith('se.mcf.iot.command.')) refreshEventsSoon();   // a command changed: its notification may exist now
  // alarms: see the note below
});

async function loadInbox() {
  const [nt, cd] = await Promise.all([
    api('GET', '/notifications', { query: { page: 1, pageSize: 50 } }).catch(() => null),
    api('GET', '/conditions',    { query: { pageSize: 50 } }).catch(() => null)
  ]);
  rows = buildRows(nt, cd);                          // sections 3 to 6
  redraw();
}
```

Notes:

- The event itself is **not** the row. The rows come from the two calls above. An event only tells you *when to
  ask*.
- I do not know whether alarm (condition) changes are sent on this SignalR line. The app subscribes only to
  `se.mcf.iot.device.*` and `se.mcf.iot.command.*`. So keep the 60-second refresh as a safety net for alarms.
- Do not append the event as a row yourself: the server's notification for it carries the proper title, body
  and read state.
- After a reconnect, refresh once (events missed while offline are never replayed).

## 10. When the tab refreshes (full list)

| Moment | What happens |
| --- | --- |
| The person signs in or switches organization | The buttons load, then the list loads |
| Every **60 seconds** | Refresh, but **only** while the app is in front (not hidden) and the person is signed in |
| The app returns to the front | Refresh (and the buttons reload too) |
| After a SignalR reconnect | The app reloads the buttons and the list ("resync") |
| After Approve | Refresh at once |
| After Acknowledge | Refresh at once |
| A local press ends | One local row is added at once (no request) |
| The person opens the Events tab | The orange dot is cleared (no request) |

There is no pull-to-refresh and no "load more": the app reads only the **first page of 50** from each call.

## 11. Errors and what the app does

| Call | Answer | What the app does | What you should do |
| --- | --- | --- | --- |
| notifications | `401` | Tries to refresh the token, else sends the person to the login | Normal token handling |
| notifications | `403 USER_TOKEN_REQUIRED` | Treated as "nothing" | Only a person's token may call it |
| notifications | `400 NOTIFICATION_TYPE_INVALID` | Treated as "nothing" | Only if you send `type`: use a type from the preferences call |
| conditions | `403` (no `IOT_SUPERVISION` View) | No alarm rows | Fine: the person simply has no alarms |
| conditions | `400 CONDITION_STATE_INVALID` or `SEVERITY_INVALID` | Treated as "nothing" | Only if you send `state` or `severity` wrongly |
| Either | `429 RATE_LIMITED` | Treated as "nothing" until the next refresh | Wait before asking again |
| approve | `403 SELF_APPROVAL_NOT_ALLOWED` | Toast with the message | The requester cannot approve their own press |
| approve | `403 APPROVER_NOT_PERMITTED` | Toast | The token is not a person's, or lacks `OPS_APPROVE` Manage |
| approve | `409 COMMAND_NOT_PENDING_APPROVAL` or `APPROVAL_EXPIRED` | Toast | Someone else approved it, or the time ran out. Refresh |
| approve | `409 FUNCTION_DISABLED`, `DEVICE_IN_MAINTENANCE`, `DEVICE_NOT_ACTIVE`, `FUNCTION_ROUTE_UNAVAILABLE` | Toast | The command stays pending |
| acknowledge | `403` | "You can't acknowledge alarms" | Needs `IOT_SUPERVISION` Manage |
| acknowledge | `409 CONDITION_NOT_RAISED` or `CONDITION_CLOSED` | Toast | It was already handled. Refresh |

"Treated as nothing" means the app catches the error and goes on without those rows. It does not show an
error banner on the Events tab.

## 12. What I could not confirm

- A **real answer** of the notifications or conditions call. The examples are illustrative.
- The **list of notification types** (`GET /iot/v1/notifications/preferences` returns it). The app only checks
  whether a type contains the words "approv" or "geo". If your server names its approval type differently, the
  Approve button will not appear.
- The **severity words** of notifications. The app treats `warn`, `crit`, `high` and `err` (any case) as red.
- Whether alarm changes are pushed on the SignalR line.
- Whether the same press can appear twice (a local row plus the server's notification). From the code it can.

## 13. Cheat sheet

```
Events tab = GET /iot/v1/notifications  +  GET /iot/v1/conditions  +  local rows of your own presses
Not SignalR. Refresh: every 60 s (app in front), on return to front, after reconnect, after Approve/Acknowledge.
Notification row : red if severity ~ warn|crit|high|err ; "geo" in type -> geofence row ; else blue
Approve button   : type ~ approv  AND  commandId  AND  unread   -> POST /iot/v1/commands/{id}/approve  then mark read
Alarm row        : title = ruleName ; small = building · severity · Active|Acknowledged ; since = time
Acknowledge      : no acknowledgedAt  -> POST /iot/v1/conditions/{id}/acknowledge
Conditions list  : default state=open (pending or raised); cleared ones are not shown
Chips            : All | Commands (kind cmd) | Alerts (everything that is not cmd)
Count            : "N open alerts" = red rows that are still open
Make it live     : on a SignalR event, call the same refresh (debounce ~500 ms); keep the 60 s timer
```

---

## 14. Worked example: reading a screenshot row by row

This is the demo data (the names say "FAKE"). A real account works the same way.

```
TODAY
 [blue ]  Approval needed: Bollards - Lower              14:06   [Approve]
          Requested by fake.colleague · Maintenance · Central depot (FAKE)
 [blue ]  Garage 12 - Open · Confirmed                   13:58
          Garage doors · by fake.officer
 [red  ]  Gate B sensor offline                          13:34   [Acknowledge]
          Central depot (FAKE) · Warning · Active
 [red  ]  Garage 12 - Close was not confirmed            13:30
          The device did not report a new state in time. · Fire station North (FAKE)
```

### Row 1: "Approval needed: Bollards - Lower" with **Approve**

It is a **notification** (`GET /iot/v1/notifications`):

```json
{ "notificationId": "ntf_2", "type": "command.approval.requested", "severity": "Info",
  "title": "Approval needed: Bollards - Lower", "body": "Requested by fake.colleague · Maintenance",
  "siteId": "site_2Lp8Xz", "commandId": "cmd_demoApprove",
  "occurredAt": "2026-10-09T14:06:00Z", "read": false }
```

| What you see | Why |
| --- | --- |
| Blue icon | `severity` is `Info`, not warn, crit, high or err |
| Title | `title` |
| Small line | `body`, then the building name found from `siteId` ("Central depot (FAKE)") |
| 14:06 | `occurredAt` |
| **Approve** button | `type` contains "approv" **and** `commandId` is set **and** `read` is `false` |

### Row 2: "Garage 12 - Open · Confirmed" (no button)

A **local row**. The phone made it when that press ended: `<button name> · <status word>`, then
`<category> · by <person's name>`. It comes from no request, so it has no `id` on the server side and no button.
The server's own record of the same press may show up later as a notification.

### Row 3: "Gate B sensor offline" with **Acknowledge**

An **alarm** (`GET /iot/v1/conditions`):

```json
{ "conditionId": "cnd_1", "ruleName": "Gate B sensor offline", "state": "Active",
  "severity": "Warning", "subject": { "siteId": "site_2Lp8Xz", "deviceId": "dev_16" },
  "since": "2026-10-09T13:34:00Z", "acknowledgedAt": null }
```

| What you see | Why |
| --- | --- |
| Red icon | Every alarm is a red row |
| Title | `ruleName` |
| Small line | The building (from `subject.siteId`), then `severity`, then "Active" because `acknowledgedAt` is empty |
| 13:34 | `since` |
| **Acknowledge** button | `acknowledgedAt` is empty |

### Row 4: "Garage 12 - Close was not confirmed" (no button)

A **notification**:

```json
{ "notificationId": "ntf_1", "type": "command.failed", "severity": "Warning",
  "title": "Garage 12 - Close was not confirmed", "body": "The device did not report a new state in time.",
  "siteId": "site_7Hq2k9", "commandId": null, "occurredAt": "2026-10-09T13:30:00Z", "read": false }
```

Red icon because `severity` is `Warning`. No button, because the type does not contain "approv". It counts in
"N open alerts" while it is unread.

### What happens when you tap

| Tap | Request | After it works |
| --- | --- | --- |
| **Approve** (row 1) | `POST /iot/v1/commands/cmd_demoApprove/approve` with `{}` | Toast "Approved - the command will run". The notification is marked read (`POST /iot/v1/notifications/ntf_2/read`). The list refreshes and the row loses its button |
| **Acknowledge** (row 3) | `POST /iot/v1/conditions/cnd_1/acknowledge` with `{}` | Toast "Alarm acknowledged". The list refreshes. The small line says "Acknowledged" and the open count goes down by one |

Who may tap: Approve needs `OPS_APPROVE` Manage and the person who made the press can never approve it.
Acknowledge needs `IOT_SUPERVISION` Manage (otherwise the app says "You can't acknowledge alarms").
