# BLE Proxies — replacing ESPresense with ESPHome Bluetooth Proxy

Modern HA-recommended path for room-level BLE tracking. Replaces the broken ESPresense MQTT-based stack with the native HA `bluetooth` integration fed by ESPHome `bluetooth_proxy:` devices.

## Why this is better than ESPresense

| | ESPresense (old) | ESPHome BLE Proxy (new) |
|---|---|---|
| Transport | MQTT broker | HA native API (encrypted) |
| Distance | RSSI-based, per-node | RSSI-based, HA aggregates |
| Discovery | MQTT discovery (fragile, write-once URLs) | mDNS auto-discovery |
| Maintenance | Custom firmware | Standard ESPHome dashboard |
| HA integration | iBeacon Tracker (also works) | iBeacon Tracker + Bluetooth + private_ble_device |
| Tracking targets | iBeacons, BLE-classic | iBeacons, AirTags (with FindMy add-on), thermometers, locks, weather sensors, etc. |

## Hardware inventory (8 nodes to convert)

| Room | Old IP (ESPresense) | New device name (ESPHome) | Status |
|---|---|---|---|
| Hayden's Office | 192.168.1.51 | `ble-proxy-office` | YAML drafted |
| Master Bedroom | 10.10.1.174 | `ble-proxy-master-bedroom` | TBD |
| Master Bathroom | 10.10.1.199 | `ble-proxy-master-bathroom` | TBD |
| Theater | 10.10.1.214 | `ble-proxy-theater` | TBD |
| Kitchen | 10.10.1.101 | `ble-proxy-kitchen` | TBD |
| Living Room | 10.10.1.21 | `ble-proxy-living-room` | TBD (currently offline in MQTT) |
| Shop | 10.10.1.251 | `ble-proxy-shop` | TBD |
| Master Bedroom (alt) | 192.168.3.16 (stale) | – | drop, duplicate |

All hardware is **ESP32-D0WD-V3** (verified via HA's earlier device registry). Standard `esp32dev` board target works for all of them.

## Rollout plan

Do node-by-node, not all at once — that way Murphy/the BLE network keeps working through the migration.

### Phase 1 — Office (proof of concept)

1. **Pull the office ESPresense node** off the wall (or wherever it lives).
2. **Plug it into a laptop via USB.**
3. **Generate firmware** via the ESPHome dashboard add-on in HA:
   - Settings → Add-ons → ESPHome → "Open Web UI"
   - "+ New Device" → "Continue"
   - Name: `ble-proxy-office` → install platform: `ESP32`
   - When the wizard finishes, click "Edit" on the device card
   - Replace the generated YAML with the contents of `firmwares/ble-proxy-office.yaml` from this repo
   - Add the missing secrets to your ESPHome `secrets.yaml` (see `firmwares/secrets.example.yaml`)
   - Click "Save", then "Install" → "Manual download" → wait for compile, download the `.bin`
4. **Web-flash the firmware**:
   - In Chrome/Edge, open https://web.esphome.io/
   - "Connect" → select the ESP32's serial port → "Install"
   - Upload the `.bin` from step 3
   - Wait ~30 seconds for flash + reboot
5. **Reattach the node** at its location, power it on.
6. **Verify in HA**:
   - Settings → Devices & Services → look for "Discovered" → ESPHome → `ble-proxy-office` → adopt
   - Settings → Devices & Services → Bluetooth → should show a new "scanner" for the office
   - Confirm `device_tracker.murphy` now lists *two* `source` MACs in its attributes (the EP1 + the new proxy) — this means HA's Bluetooth integration is fusing both feeds.

### Phase 2 — Rinse and repeat for the other 6

For each of the remaining 6 nodes (skip the duplicate Master Bedroom at 192.168.3.16), copy `ble-proxy-office.yaml` to a new file with the room name in the substitutions. Steps 1–6 above for each one. Each conversion takes ~10 minutes if the laptop is already set up.

Order suggestion (by room utility for tracking):

1. Master Bedroom
2. Master Bathroom
3. Living Room (and confirm it stops being "offline" — it was in MQTT land)
4. Kitchen
5. Theater
6. Shop

### Phase 3 — Cleanup

Once all 7 nodes are running ESPHome, the old ESPresense MQTT integration's clutter can be dropped:

1. Delete the 8 stale ESPresense devices from HA's device registry.
2. Optionally retire the iBeacon Tracker integration if you decide to use `private_ble_device` instead (single-device, more deterministic).
3. Update `dashboards_overview.md` if you have a presence dashboard built around ESPresense entities.

## Why `bluetooth_proxy: active: true`

Active scanning sends scan request packets to advertisers, prompting them to send scan response data (extra fields, device names). Passive-only ("active: false") just listens. For iBeacons like Murphy, active is faster + more reliable, and the BLE traffic increase is negligible.

If you ever see the proxy struggling under high BLE load (e.g. lots of phones nearby), flip to `active: false` per node.

## Why `framework: esp-idf` over `arduino`

ESP-IDF supports more concurrent BLE connections (~10) and is more memory-efficient. Arduino works for the basic proxy use case but caps at ~5 concurrent BLE connections — a problem if you later track many BLE devices simultaneously.
