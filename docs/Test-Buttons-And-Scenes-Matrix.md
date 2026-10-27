# Test buttons and scenes: one for every scenario

Use this list to create test buttons and scenes in the **portal** (the admin side). Then each row tells you
what to check in the app.

> **Safety first.** A button sends a real command to real equipment (a gate, a bridge, a barrier). Create
> these on a **test site with test devices**, never on a live site. Give the test buttons names that start
> with `TEST -` so nobody presses one by mistake.
>
> The field API (`iot-v1`) that the app uses can only **read** buttons and **press** them. It has no call to
> create a button or a scene. They are created in the portal.
> I could not open your portal or create anything, so this is a checklist to create them yourself.

## How to use

1. Create a test **site** with at least one **device**, and give the test account access to it.
2. Create the buttons of section 1, one per row. Set only the settings written in the "Portal settings" column.
   Leave everything else at its default.
3. Create the scenes of section 2 from those buttons.
4. Sign in to the app with the test account. Check the "What to check in the app" column for each row.

Words in the "Portal settings" column match the API names, so the portal's own labels may differ a little.
The names come from the API guide (`Field-App-API-Guide.md`).

---

## 1. Buttons

### 1.1 The basic kinds (one button each)

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B01 | TEST - Gate open | kind `gate.open`, safety `Normal`, confirmation off | Appears under **Gates**. Hold to send. Status goes Sending, Sent, **Confirmed**. Row turns green |
| B02 | TEST - Gate close | kind `gate.close` | Same, with the "close" icon |
| B03 | TEST - Garage open | kind `garage.open` | Under **Garage doors** |
| B04 | TEST - Garage close | kind `garage.close` | Under **Garage doors** |
| B05 | TEST - Bollard raise | kind `bollard.raise` | Under **Bollards** |
| B06 | TEST - Bollard lower | kind `bollard.lower` | Under **Bollards** |
| B07 | TEST - Signal priority | kind `signal.priority` | Under **Traffic lights** |
| B08 | TEST - Signal flash | kind `signal.flash` | Under **Traffic lights** |
| B09 | TEST - Bridge open | kind `bridge.open` | Under **Bridges** |
| B10 | TEST - Bridge close | kind `bridge.close` | Under **Bridges** |
| B11 | TEST - Bridge hold | kind `bridge.hold` | Under **Bridges** |
| B12 | TEST - Output on | kind `output.on` | Under **IoT** |
| B13 | TEST - Output off | kind `output.off` | Under **IoT** |
| B14 | TEST - Output pulse | kind `output.pulse` | Under **IoT**. The press may allow `pulseSeconds` |
| B15 | TEST - Input read | kind `input.read` | Under **Inputs**, if your menu shows that tile |

### 1.2 Safety and confirmation

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B20 | TEST - Safety Low | safety `Low` | Hold and it is sent. No reason asked |
| B21 | TEST - Safety Normal | safety `Normal`, confirmation off | Same as B20 |
| B22 | TEST - Confirm only | confirmation **required**, reason **not** required | The sheet asks to confirm. Sending without confirming is refused (`CONFIRMATION_REQUIRED`) |
| B23 | TEST - Reason required | reason **required** | The reason list (Operator request, Unit dispatch, Incident, Emergency, Maintenance, Test, Scheduled, Other) shows. Hold is refused until one is picked (`REASON_REQUIRED`) |
| B24 | TEST - Safety High | safety `High` | Yellow note "Safety class High: a reason is required." Reason list shows |
| B25 | TEST - Safety Critical | safety `Critical` | Same, with the word Critical |
| B26 | TEST - Safety LifeSafety | safety `LifeSafety` | Same, with the word LifeSafety |
| B27 | TEST - Reason Other | reason required | Pick **Other**: a text box "Describe the reason" appears. Without text the press is refused (`OTHER needs a reasonText`) |

### 1.3 Two-person approval

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B30 | TEST - Two person | `twoPersonApproval` on | Blue note "A second person must approve before it runs." After the press the status is **Waiting for approval** with a Cancel button |
| B31 | TEST - Two person + reason | `twoPersonApproval` on, safety `High` | Both notes show, with the reason list |

Check it with **two accounts**: the first presses, the second sees **Approve** in the Events tab (a notification
whose type contains "approv") and taps it. Then the command is sent. Try also: the same person tries to
approve their own press (refused, `SELF_APPROVAL_NOT_ALLOWED`), and nobody approves in time (`APPROVAL_EXPIRED`).

### 1.4 Place-only buttons (geofence)

Set the geofence **center** to a place you can stand at (or fake the phone's position), and the radius.

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B40 | TEST - Geofence 150 m | geofence radius 150 m, a center | Note "Works only on site (within 150 m). Your location is sent with the press." In the **Geofence** tab: inside the circle it is unlocked, outside it is grey with a lock |
| B41 | TEST - Geofence small | radius 25 m | Stand 40 m away: locked. Move closer: unlocked. Tests a poor GPS fix |
| B42 | TEST - Geofence no center | radius set, **no center**, but the button has a position | The circle uses the button's own position |
| B43 | TEST - Geofence + reason | geofence on, reason required | Reason list and geofence note both show |
| B44 | TEST - Geofence, location off | geofence on | Turn the phone's location **off**: the app asks to turn it on. Pressing without a position is refused (`NO_LOCATION`) |
| B45 | TEST - Geofence outside | geofence on | Press from far away: the server refuses (`422 OUTSIDE_GEOFENCE`) and the app says "You are outside the allowed area" |

### 1.5 Available or not

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B50 | TEST - Switched off | `enabled` false | The button is **grey** with the reason "Not available". It cannot be pressed |
| B51 | TEST - No operate right | the test account has view but **not operate** (`access.canOperate` false) | Grey and disabled |
| B52 | TEST - Outside shift | a schedule that is **closed** right now | Grey and disabled. When the shift starts, the app reloads and the button unlocks |
| B53 | TEST - No route | route `none` (device has no route now) | Red note "No route to the device right now", grey **Not available** button |
| B54 | TEST - Legacy route | route `legacy` | Yellow warning (the `routeWarning` text): sent through the old Station Manager, the result is **not confirmed** |
| B55 | TEST - Edge route | route `edge` | The normal confirmed result |
| B56 | TEST - Central route | route `central` | The normal confirmed result |

### 1.6 Results (needs a device or a simulator you can control)

| # | Name | How to make it happen | What to check in the app |
| --- | --- | --- | --- |
| B60 | TEST - Confirms | The device confirms | Status **Confirmed** |
| B61 | TEST - Fails | The device refuses or fails | Status **Failed**, with the detail text. The button does not stay green |
| B62 | TEST - Times out | The device never answers | Status **TimedOut**: "Check on site before you press again." The app must **never** press again by itself |
| B63 | TEST - Rejected | A rule refuses it after it was accepted, for example put the device in maintenance | Status **Rejected** with the detail |
| B64 | TEST - Cancel | A button with two-person approval: press, then tap **Cancel** before anyone approves | Status **Cancelled** |

### 1.7 Looks (appearance)

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B70 | TEST - Own colours | a background colour and a text colour | The button uses them |
| B71 | TEST - Own icon | upload an icon | The icon shows (it is fetched once and kept by its hash) |
| B72 | TEST - Pressed icon | upload a pressed icon | After the press the pressed icon shows |
| B73 | TEST - Highlight border | "before" border on with a colour | A coloured border before the press |
| B74 | TEST - After colours | after-button and after-text colours | The colours change after the press |
| B75 | TEST - Vibrate | after-vibrate on | The phone vibrates after the press |
| B76 | TEST - Flash | after-flash on | The phone's light blinks after the press |
| B77 | TEST - No icon | hide the icon (before and after) | The button has text only |

### 1.8 Where the button lives

| # | Name | Portal settings | What to check in the app |
| --- | --- | --- | --- |
| B80 | TEST - Two buildings | the same kind of button on **two sites** | The tile shows **Select site** with both buildings |
| B81 | TEST - One building | buttons on one site only | The tile skips the building list and shows the buttons |
| B82 | TEST - Shared by another organization | another organization **delegates** this button to the test account | It shows under **Shared with us**, with the note "Shared with you by another organization." No live events for it: the status comes from asking `GET /commands/{id}` |
| B83 | TEST - Long name | a name of about 60 characters | The name wraps or is cut neatly, and does not break the layout |
| B84 | TEST - Swedish letters | a name like `Öppna grinden åäö` | Letters show correctly |

---

## 2. Scenes

A scene runs several buttons **in order**, one after another. Build each scene from the test buttons above.

| # | Name | Members (in order) | What to check in the app |
| --- | --- | --- | --- |
| S01 | TEST - Scene simple | B01, B03, B04 | The sheet says "Runs 3 buttons in order." and lists the three buttons with numbers 1, 2, 3. Hold: the rows go Sent, Sending, then **Confirmed** one by one. At the end "Done - 3 of 3 confirmed" |
| S02 | TEST - Scene one member | B01 | Works with a single member |
| S03 | TEST - Scene long | 8 or more buttons | The list scrolls. Order is kept. Takes longer, the Running panel stays until it ends |
| S04 | TEST - Scene with reason | B01, B24 (High) | The app asks the reason after the first answer `422 REASON_REQUIRED` (the scene row itself has no safety class) |
| S05 | TEST - Scene with geofence | B01, B40 | The member with the geofence shows a "150 m" pill. Press outside: refused with `OUTSIDE_GEOFENCE`. The app must send the phone's position with the press |
| S06 | TEST - Scene with two-person member | B01, B30 | That member is **Rejected** with `APPROVAL_REQUIRED_IN_GROUP` and the others still run. The run ends as partly failed |
| S07 | TEST - Scene with a disabled member | B01, B50 | The disabled member shows "Not available" in the list. Check what the server does with it when the scene runs |
| S08 | TEST - Scene with a failing member | B01, B61, B03 | The run goes on after a failure. Final run status is partly failed. Row 2 shows **Failed**, rows 1 and 3 show **Confirmed** |
| S09 | TEST - Scene with a timeout | B01, B62 | Row 2 shows **TimedOut** with the warning, the run ends |
| S10 | TEST - Scene with a legacy member | B01, B54 | The legacy member shows the warning that it is not confirmed |
| S11 | TEST - Scene on two sites | B01 (site A), B09 (site B) | The scene's building name and each member's own building name are right |
| S12 | TEST - Empty scene | no members | Pressing it answers `409 GROUP_HAS_NO_MEMBERS`. The app shows the message |
| S13 | TEST - Scene own look | colours, icon, border, vibrate | The scene button uses its own look |
| S14 | TEST - Scene shared | delegated from another organization | Shows under Shared with us (members' events belong to the other organization) |

---

## 3. Things to check for **every** button and scene

- The **idempotency**: tap and hold twice quickly. Only **one** command is created (the app sends an
  `Idempotency-Key`). If the first answer was lost (switch the network off right after the hold), pressing again
  must not run it twice.
- The **Events tab** shows the result: a local row at once, and the server's notification at the next refresh.
- **Dark mode** and **light mode** look right.
- The **button list** updates after a change in the portal (pull the menu again with "Reload buttons" in
  Profile, or wait for the app to return to the front).
- **Without internet**: pressing shows "No connection" and "Safe to press again: the same press never runs twice."
- **Live updates** (see `SignalR-Live-Updates-Guide.md`): with the line "Live", the status of a press changes
  at once. With the line down, it still arrives, only slower.

## 4. What I could not confirm

- I have not seen your portal, so the **names of its settings** may differ from the API names used here.
- Whether the portal can make a button with a combination such as "reason required but confirmation off". The
  API guide says from `High` safety upwards a confirmation and a reason are always required.
- The exact **status words of a scene run** (the files only show `Running`-like and "partly failed" in my demo).
- The exact behavior of a scene with a **disabled** member (S07). Test it and note what you see.
