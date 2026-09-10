# Aruba BLE Proxy

Home Assistant integration to use Aruba access points as Bluetooth Low Energy
scanner sources, with passive advertisement forwarding and an active BLE/GATT
connector.

This is not a device decoder and does not publish MQTT state. The intended direction is:

```text
Aruba AP -> WebSocket/protobuf -> Home Assistant Bluetooth stack
```

Current phase: 1.0. Passive BLE forwarding and active BLE/GATT are supported
within the limits documented below.

This project is not affiliated with, endorsed by, or sponsored by HPE Aruba
Networking. The included integration icon/logo assets are original project
artwork and do not use Aruba trademarks or logos.
Home Assistant 2026.3 and newer can load these local brand assets from the
integration's `brand/` directory.

See [SPEC.md](SPEC.md) for scope and architecture.

## Install with HACS

1. In HACS, open the menu and select **Custom repositories**.
2. Add `https://github.com/bot190/aruba-ble-proxy` with type **Integration**.
3. Find **Aruba BLE Proxy** in HACS and download it.
4. Restart Home Assistant.
5. Go to **Settings → Devices & services → Add Integration** and select
   **Aruba BLE Proxy**.
6. Complete setup and apply the generated Aruba CLI configuration. See the
   [setup and Aruba configuration guide](docs/INSTALL_MANUAL.md#add-integration)
   for settings and validation steps.

HACS installs the complete `custom_components/aruba_ble_proxy` directory,
including the bundled protobuf modules and brand assets. No separate checkout,
protobuf generation, or manual pip installation is needed in Home Assistant.
Existing manual installations can use HACS to manage the same integration;
keep the existing Home Assistant configuration entry.

This repository can be added as a custom repository; it is not a default HACS
catalog listing. The GitHub repository must be public. Maintainers should enable
issues and set a repository description and relevant topics. GitHub Actions
validates HACS metadata on pushes and pull requests. To publish a versioned
update, create a GitHub release whose tag matches the integration's
`manifest.json` version (currently `1.2.0`). Release ZIP files are not required.
Without releases, HACS can install from the default branch.

See the [HACS publishing requirements](https://hacs.xyz/docs/publish/integration/)
for default catalog submission, including Home Assistant Brands registration.

## Development

Install dependencies:

```bash
python3 -m pip install -e ".[dev]"
```

The generated Aruba protobuf Python files are committed under
`custom_components/aruba_ble_proxy/aruba_iot_ble/proto_generated`, so a fresh
clone is enough for normal development, tests, and manual Home Assistant
installation.

Regenerate them only when Aruba's upstream `.proto` files need to be refreshed:

```bash
scripts/generate-aruba-protobuf.sh
```

By default the script expects Aruba's
[`aos8-iot-server-example-websocket`](https://github.com/aruba/aos8-iot-server-example-websocket)
repository under `vendor/aos8-iot-server-example-websocket` and writes generated files into
`custom_components/aruba_ble_proxy/aruba_iot_ble/proto_generated`.
The `vendor/` directory is intentionally local-only and is not committed.
You can override paths:

```bash
ARUBA_PROTO_DIR=/path/to/proto_files/source \
ARUBA_PROTO_OUT=custom_components/aruba_ble_proxy/aruba_iot_ble/proto_generated \
scripts/generate-aruba-protobuf.sh
```

Run the standalone receiver for local protocol/debug testing:

```bash
aruba-ble-proxy-receiver --host 0.0.0.0 --port 7443 --log-level info
```

The standalone receiver accepts Aruba WebSocket connections, decodes BLE Data
protobuf messages, and logs normalized advertisements. It does not forward
advertisements into Home Assistant; that path is implemented by the custom
integration running inside Home Assistant.

For field testing, compact BLE summaries are easier to read:

```bash
aruba-ble-proxy-receiver --host 0.0.0.0 --port 7443 --log-level info --summary
```

If an Aruba access token is configured:

```bash
aruba-ble-proxy-receiver --access-token "secret"
```

The receiver accepts only the configured endpoint path (default
`/aruba-ble-proxy`), bounds WebSocket messages and concurrent connections, and
closes clients that repeatedly send invalid telemetry. The transport is still
plain `ws://`; expose port `7443` only to trusted Aruba AP networks or protect it
with equivalent firewall/VLAN controls.

The CLI also reads environment variables:

```bash
ARUBA_BLE_PROXY_HOST=0.0.0.0
ARUBA_BLE_PROXY_PORT=7443
ARUBA_BLE_PROXY_ACCESS_TOKEN=secret
ARUBA_BLE_PROXY_LOG_LEVEL=info
ARUBA_BLE_PROXY_SUMMARY=true
```

Command line flags override environment variables.

## Aruba CLI filter generation

Aruba Instant accepted at most 10 `serviceUUIDFilter` values per transport profile in local testing.
The generator emits a complete Aruba Instant CLI block:

- one BLE scanning IoT radio profile
- multiple BLE Data transport profiles
- one `serviceUUIDFilter` chunk per transport profile, with at most 10 UUIDs per chunk

Generate the CLI block:

```bash
aruba-ble-proxy-generate-aruba-cli \
  --endpoint-url ws://192.0.2.10:7443/test \
  --token example-access-token
```

Write it to a file instead of stdout:

```bash
aruba-ble-proxy-generate-aruba-cli \
  --endpoint-url ws://192.0.2.10:7443/test \
  --token example-access-token \
  --output aruba-ha-ble-config.txt
```

The default seed file is `custom_components/aruba_ble_proxy/data/ha_service_uuids_seed.txt`.
This is a practical compatibility list, not a universal BLE catch-all.

Generate cleanup commands for the same generated profiles:

```bash
aruba-ble-proxy-generate-aruba-cli --cleanup
```

## Home Assistant custom integration

The initial custom integration lives under:

```text
custom_components/aruba_ble_proxy
```

Implemented:

- config flow with endpoint, token, and Aruba profile settings
- generated Aruba CLI block during setup and options flow
- `aruba_ble_proxy.generate_cli` service with response data
- WebSocket receiver lifecycle inside Home Assistant
- Aruba BLE advertisements converted to `BluetoothServiceInfoBleak`
- Aruba APs registered as active-scan Home Assistant Bluetooth sources, matching Aruba's background scan and scan-response forwarding
- Aruba clusters may multiplex several AP scanner sources over one WebSocket; southbound actions remain routed by AP source
- forwarding into Home Assistant Bluetooth via `async_get_advertisement_callback`
- no recorder-backed diagnostic sensors; validation is done through Home Assistant Bluetooth sources and logs
- connectable Home Assistant Bluetooth scanner support for active BLE/GATT
- Aruba BLE action path for connect, disconnect, GATT read/write, and notifications
- active BLE connection slots per AP are configurable
- active GATT reads, characteristic discovery, and notifications are scoped by Aruba AP source
- narrow SwitchBot command service fallback when a device advertises SwitchBot service UUID `FD3D`
- passive BLE validated with BTHome and SwitchBot thermometer advertisements
- active BLE/GATT validated in long-running Home Assistant field use

Validated in a real Home Assistant setup:

- Aruba AP connects to the integration over WebSocket
- Aruba BLE Data advertisements are forwarded into Home Assistant Bluetooth
- BTHome events continue working with ESPHome BLE proxy and host Bluetooth disabled
- SwitchBot thermometer advertisements work through the passive path
- active BLE/GATT runs through Aruba AP sources without host Bluetooth or ESPHome BLE proxy

Known limits:

- BLE pairing, bonding, descriptor read/write, and unpairing are not implemented.
- The proxy does not decode or repair application protocols such as BTHome,
  Xiaomi, Shelly, or SwitchBot payloads.
- Aruba may not report a complete GATT characteristic discovery for every
  device. The only compatibility fallback in core is the narrow SwitchBot
  command-service fallback for devices advertising `FD3D`.
- Aruba BLE forwarding depends on Aruba IoT transport/profile filtering; this
  project is not a universal catch-all for every nearby BLE frame.

Active BLE notes are tracked in
[docs/ACTIVE_BLE_FEASIBILITY.md](docs/ACTIVE_BLE_FEASIBILITY.md). The Home
Assistant field-test checklist is in
[docs/HA_FIELD_TEST_RUNBOOK.md](docs/HA_FIELD_TEST_RUNBOOK.md).

Manual install instructions are in [docs/INSTALL_MANUAL.md](docs/INSTALL_MANUAL.md). The install
requires copying only `custom_components/aruba_ble_proxy`.

## Hardware Compatibility

Community-tested hardware and firmware combinations are tracked in
[docs/HARDWARE_COMPATIBILITY.md](docs/HARDWARE_COMPATIBILITY.md).

## License

This project is licensed under the **GNU General Public License v3.0**. See [LICENSE](LICENSE).
