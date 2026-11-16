# Tasks: Low-Overhead Event-Driven KeyBeacon

Tasks are grouped by the implementation boundary. Firmware tasks belong in
this repository; host-client tasks apply to `keybeacon-linux`.

## Firmware

- [x] 1.1 Remove the periodic battery work item
      (`batt_floor_work`, its callback, and its scheduling).
      Battery publication must be driven only by
      `zmk_battery_state_changed` and
      `zmk_peripheral_battery_state_changed`.
- [x] 1.2 Remove any battery timing state that is only required by the
      periodic work item, including `batt_last_emit_ms`.
- [ ] 1.3 Verify that AA1, AA2, and AA3 retain payload change detection and
      emit no notification when an event does not change the serialized value.
- [ ] 1.4 Rate-limit or deduplicate unresolved AA4 diagnostics so repeated
      reads cannot flood the log while retaining a useful diagnostic.
- [ ] 1.5 Confirm that the module adds no PM lock, wakeup source, forced
      advertising, connection-parameter override, or other sleep-preventing
      behavior.
- [ ] 1.6 Add a firmware/runtime verification path that demonstrates no
      periodic KeyBeacon wakeup occurs while the battery is unchanged.

## Linux host client

- [ ] 2.1 Read AA4 once after connection and retain the local profile slot;
      remove the steady-state AA4/link-slot polling loop.
- [ ] 2.2 Read the standard Battery Service once during initial connection
      setup, only as the central-battery fallback. Do not read BAS on a
      periodic timer.
- [ ] 2.3 Keep steady-state state updates notification-driven for AA1, AA2,
      and AA3. Profile changes must come from AA2 notifications.
- [ ] 2.4 Add notification coalescing/debouncing so a burst of BLE events
      produces a bounded number of UI updates without losing the final state.
- [ ] 2.5 Replace blocking per-field Ironbar IPC with asynchronous or batched
      IPC. A BLE callback must not wait on a socket response or open one
      socket per output field.
- [ ] 2.6 Keep the optional Ironbar recovery refresh independent of BLE reads;
      it may re-emit cached state but must not poll characteristics.
- [ ] 2.7 Add scan controls and/or longer scan backoff so reconnect handling
      does not repeatedly run long scans on an adapter carrying HID traffic.
      Existing OS-connected KeyBeacon devices must always take precedence.
- [ ] 2.8 Preserve non-destructive teardown for OS-owned HID connections and
      verify that stopping the client does not disconnect HID.

## Tests and acceptance

- [ ] 3.1 Add host tests proving that one connection performs only the initial
      reads and then uses notifications, with zero steady-state AA4/BAS reads.
- [ ] 3.2 Add host tests for notification bursts, including modifier changes,
      ensuring IPC/UI work is coalesced and the final state is emitted.
- [ ] 3.3 Add tests for reconnect, scan fallback, explicit address selection,
      and HID-safe teardown.
- [ ] 3.4 Test firmware on a split central and an integer keyboard with
      unchanged battery values; confirm no periodic battery notifications or
      wakeups occur.
- [ ] 3.5 Test sleep/deep-sleep entry and wake/reconnect with KeyBeacon
      enabled; confirm KeyBeacon does not keep the keyboard awake.
- [ ] 3.6 Measure ATT operations during idle and typing. Confirm there are no
      steady-state reads and that unchanged events produce no notifications.
- [ ] 3.7 Run the module compatibility and unit-test suite, and repair the
      README compatibility-table drift currently reported by
      `test_live_readme_check_mode_passes`.

## Completion criteria

- Firmware has no periodic KeyBeacon timer or delayed work.
- The host performs no steady-state GATT reads.
- Notification bursts do not block the BLE event loop or cause unbounded IPC.
- HID, reconnect, and deep-sleep behavior remain unaffected.
