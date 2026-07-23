# Network Startup

The Runtime enumerates network interfaces at startup and manages
connectivity throughout the device lifecycle.

## Startup sequence

1. Runtime queries the operating system for network interfaces.
2. For each interface found, creates a `NetworkInterface` entry.
3. Registers capabilities:
   - Any network interface → `network.info` present.
   - Wi-Fi interface → `network.wifi` present.
   - Ethernet interface → `network.ethernet` present.
4. Attempts to connect to known networks (saved Wi-Fi profiles).
5. Tests internet connectivity (captive portal detection).
6. Makes interface list available to applications.

## Interface enumeration

```cpp
int count = Network::GetInterfaceCount();
for (int i = 0; i < count; i++) {
    Network::InterfaceInfo info = Network::GetInterfaceInfo(i);
    // info.name, info.type, info.state, info.isMetered
}
```

## Connectivity states

| State | Description |
|---|---|
| `Disconnected` | No network connection |
| `Connecting` | Wi-Fi associating, DHCP in progress |
| `Connected` | Connected to local network |
| `Online` | Internet connectivity confirmed |
| `CaptivePortal` | Connected but captive portal detected |
| `Metered` | Connection has data limits |

Applications query the current state:

```cpp
Network::ConnectivityState state = Network::GetConnectivityState();
if (state == Network::ConnectivityState::Online) {
    // Safe to make cloud requests
}
```

## Wi-Fi management

The Runtime manages Wi-Fi connections:

- Scans for available networks.
- Connects to saved networks automatically.
- Prompts for password for new networks (via Shell UI).
- Roams between access points with the same SSID.

## Captive portal detection

On connection, the Runtime checks for captive portals by making a
request to a known endpoint (`http://detect.playos.org/generate_204`).
If a captive portal is detected, the Shell shows a web view for
authentication.

## Connectivity changes

Applications receive a `NetworkStateChanged` event when connectivity
changes. Applications should react gracefully:

- Pause network-dependent features when offline.
- Retry failed requests with exponential backoff.
- Show appropriate UI ("No internet connection").

## Network policy

- Applications declare `network.client` in their manifest to use the
  network.
- Background applications may have limited network access (TBD).
- Metered connections may trigger user confirmation for large
  downloads.

## Related

- [Network API](../06-platform-api/14-network-api.md)
- [Runtime Services](08-runtime-services.md)
- [Permissions](../10-package-format/09-permissions.md)
