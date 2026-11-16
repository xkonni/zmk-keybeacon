# Feature Specification: Low-Overhead Event-Driven KeyBeacon

**Feature Branch:** `006-low-overhead-event-driven`  
**Status:** Proposed  
**Scope:** `zmk-keybeacon` firmware and cooperating host clients, including
`keybeacon-linux`

## 1. Purpose

KeyBeacon exposes useful keyboard state to a host without becoming part of the
keyboard's input path. It is an optional observability service, not a second
HID transport.

The design priorities, in order, are:

1. Do not add perceptible keyboard latency or unnecessary BLE/ATT traffic.
2. Deliver changes through GATT notifications and ZMK events, rather than
   polling.
3. Never prevent or delay Bluetooth, idle, suspend, or deep-sleep power
   management.
4. Recover cleanly when either side disconnects.

## 2. Non-goals

- KeyBeacon must not send key events, modify HID reports, or select the active
  host/profile.
- KeyBeacon must not keep a peripheral awake for the benefit of a host UI.
- A host must not scan continuously while the keyboard is connected.
- A host must not periodically read characteristics whose values are
  notification-driven.
- The protocol does not promise charging state when the firmware has no charger
  GPIO or equivalent source.

## 3. GATT service

The service UUID is:

`AA440AA0-F5ED-4C48-84A1-8062D20D3D55`

The service is enabled only on a BLE-capable central image when
`CONFIG_ZMK_KEYBEACON=y`. It must remain optional and must not alter the
peripheral or settings-reset images.

### 3.1 Status (`AA440AA1-...`)

- Properties: `READ`, `NOTIFY`
- Payload: `[layer_index, modifiers, UTF-8 layer_name...]`
- `layer_index`: active layer index.
- `modifiers`:
  - shift: `0x22`
  - ctrl: `0x11`
  - alt: `0x44`
  - GUI: `0x88`

The firmware shall subscribe to layer-state and keycode-state events. It shall
rebuild and notify only when the serialized payload changes. A keycode event
that does not change the published status must produce no ATT notification.

### 3.2 Connectivity (`AA440AA2-...`)

- Properties: `READ`, `NOTIFY`
- Payload: seven bytes:

`[capability, host, profile, max_slots, split_flags, output, charging]`

- `capability`:
  - bit 0: split keyboard
  - bit 1: host connection state available
  - bit 2: profile available
  - bit 3: split-link state available
  - bit 4: output endpoint available
  - bit 5: left charging available
  - bit 6: right charging available
- `host`: bit 0 is connected.
- `profile`: bits 0–6 are the active 1-based profile; bit 7 means profile open.
- `max_slots`: number of host/profile slots.
- `split_flags`: bit 0 left online, bit 1 right online.
- `output`: `0` unknown, `1` USB, `2` BLE.
- `charging`: bit 0 left charging, bit 1 right charging.

The firmware shall subscribe to active-profile, endpoint, and split-link events,
change-detect the serialized payload, and notify only after a change.

### 3.3 Battery (`AA440AA3-...`)

- Properties: `READ`, `NOTIFY`
- Payload: one byte for an integer keyboard, or `[left, right]` for a split.
- `0..100`: percentage.
- `255`: unavailable.
- `101..254`: reserved and must be treated as unavailable.

The firmware shall subscribe to battery events and notify only when the
serialized battery payload changes. It shall not use a periodic timer, delayed
work item, heartbeat, or battery-poll loop to maintain this characteristic.
Battery state may remain unavailable until the next battery event.

If the central already has a standard Battery Service value, a host may read
that value once when attaching and use it only to fill an unavailable central
battery byte. This is a compatibility fallback, not a polling mechanism.

### 3.4 Host-Link (`AA440AA4-...`)

- Properties: `READ` only.
- Payload: one byte; bits 0–6 are the 1-based profile slot for the current
  GATT connection; `0` means unknown.
- Bit 7 is reserved and must be ignored.
- The characteristic must not have a CCC descriptor or notify property.

This is connection-local context and cannot be represented correctly as a
broadcast notification. A host shall read it once after connecting and shall
not poll it. Changes to the active profile are delivered through Connectivity.

## 4. Firmware requirements

### Requirement F1: event-driven publication

Every published characteristic shall be backed by the corresponding ZMK event
subscription. Notification work must run in the normal application event path
and must be bounded; it must not block on host I/O.

### Requirement F2: change detection

Before calling `bt_gatt_notify`, firmware shall compare the new serialized
payload with the cached payload. Equal payloads shall not be notified.

### Requirement F3: no periodic activity

KeyBeacon shall create no periodic work, timer, heartbeat, or forced battery
refresh. In particular, a 1 Hz task is prohibited even if it suppresses
duplicate notifications: it still consumes CPU and can interfere with
power-management decisions.

### Requirement F4: sleep transparency

KeyBeacon shall not:

- acquire a power-management lock;
- mark a device or GPIO as a wakeup source;
- disable Bluetooth idle, connection parameter updates, or deep sleep;
- prevent the system from suspending because a host is subscribed;
- add a periodic wakeup solely for KeyBeacon;
- require the keyboard to advertise when it would otherwise stop advertising.

A sleeping keyboard may therefore stop producing updates. The host must accept
stale state while disconnected/asleep and refresh it after reconnecting.

### Requirement F5: split state

Split-link online/offline state shall be derived from actual split connection
events. Battery-event freshness or a time-to-live must not be used as a
substitute for link state.

### Requirement F6: optional compilation

Disabling `CONFIG_ZMK_KEYBEACON` must remove the service and all KeyBeacon
listeners from the build. Enabling it must not require changes to keymaps,
shields, power-management configuration, or peripheral images.

## 5. Host-client requirements

### Requirement H1: attach before scanning

The client shall first look for an existing OS connection exposing KeyBeacon.
This is the normal path because a keyboard connected as HID may stop
advertising. The client shall attach to that link without disconnecting it.

Only when no suitable existing connection is present may the client scan.
Scanning shall be bounded, backoff-controlled, and never run continuously while
an existing KeyBeacon connection is available. An explicit address option
should bypass scanning.

### Requirement H2: subscribe to notifications

After connecting, the client shall:

1. read the current values once to seed complete state;
2. read Host-Link once;
3. subscribe to Status, Connectivity, and Battery notifications;
4. update the UI from notifications.

It shall not poll Status, Connectivity, or Battery. It shall not poll
Host-Link; profile changes arrive through Connectivity.

### Requirement H3: bounded UI emission

A burst of notifications shall be coalesced or debounced so one UI update does
not synchronously perform unbounded IPC work. Host UI IPC must be asynchronous
or batched and must never block the BLE event loop.

The client may emit an infrequent UI refresh to recover from a restarted bar
process, but this refresh must not perform BLE reads. A suggested maximum
refresh rate is once per minute.

### Requirement H4: non-destructive teardown

When attached to an OS-owned HID connection, stopping the client shall remove
only the client's GATT session. It shall not call the operation that disconnects
the BlueZ device or otherwise drop the HID link.

### Requirement H5: reconnect behavior

On disconnect, the client shall publish an explicit disconnected state, then
retry with exponential backoff capped at a reasonable interval. It shall
resubscribe and re-prime state after reconnecting.

## 6. Host state and rendering

The Linux client shall expose, at minimum:

- layer index and name;
- active modifier flags;
- active profile and local Host-Link profile, when known;
- USB/BLE/unknown output;
- split left/right online state;
- battery values, with unavailable values rendered as unavailable;
- charging only when the capability bit says it is supported;
- connected/disconnected status.

Malformed payloads shall be ignored without terminating the client. A
notification shall not overwrite valid state unless its complete payload parses
successfully.

## 7. Acceptance scenarios

### A1: typing does not create avoidable traffic

Given a subscribed host and a keycode event that leaves the published status
unchanged, no KeyBeacon notification is sent. Given a modifier or layer
change, exactly one changed Status payload is eligible for notification.

### A2: battery remains idle

Given a keyboard whose battery percentage is unchanged, no battery ATT
notification and no firmware wakeup occurs, regardless of elapsed time.

### A3: sleeping keyboard

Given an idle keyboard entering deep sleep, KeyBeacon adds no wake lock or
timer. The keyboard is allowed to sleep and the host eventually displays
disconnected or stale state. On wake/reconnect, the host reads current values
and resumes notifications.

### A4: profile change

Given a connected host and an active profile change, Connectivity notification
updates the UI. The host does not read Host-Link repeatedly.

### A5: HID coexistence

Given an existing HID connection, the client attaches to it without scanning.
Stopping or reconnecting the client does not disconnect HID.

### A6: split disconnect

Given the peripheral split link disconnects, Connectivity reports the offline
flag from the link event. The flag does not expire merely because no battery
event arrived.

### A7: low-overhead UI

Given several notifications arrive close together, the client coalesces them
and does not open one blocking IPC socket per notification field.

## 8. Verification

- Instrument ATT operations and verify no steady-state reads occur after the
  initial prime, except an explicitly requested diagnostic operation.
- Verify no KeyBeacon timer/work item wakes the system periodically.
- Verify the firmware builds with KeyBeacon enabled and disabled, including
  split central, split peripheral, and settings-reset images.
- Test layer/modifier, profile, battery, split-link, reconnect, sleep, and
  HID coexistence behavior on real hardware.
- Confirm that stopping the host client leaves the keyboard HID connection and
  normal sleep behavior intact.
