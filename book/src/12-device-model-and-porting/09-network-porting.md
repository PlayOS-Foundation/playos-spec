# Network Porting

The network backend reports Wi-Fi state, IP address, and provides
Wi-Fi scan and connect functionality.

## Interface to implement

```cpp
// backends/network_backend.h
class INetworkBackend {
public:
    virtual Network::WiFiState GetWiFiState() = 0;
    virtual const char* PrimaryIP() = 0;
    virtual std::vector<Network::WiFiNetwork> ScanNetworks() = 0;
    virtual Network::ConnectResult Connect(
        const std::string& ssid, const std::string& psk) = 0;
};
```

`WiFiState` enum: `Unknown`, `Absent`, `Disconnected`, `Connecting`, `Connected`.

`ConnectResult` enum: `Success`, `WrongPassword`, `Timeout`, `Error`.

## Linux implementation (getifaddrs + nmcli)

The Linux backend uses two tools:

1. **`getifaddrs()`** — for Wi-Fi state (`IFF_UP` check on `wl*`
   interfaces) and primary IP address. Zero external dependencies.
2. **`nmcli`** (NetworkManager CLI) — for Wi-Fi scan and connect.
   Assumes NetworkManager is available (standard on all major Linux
   distributions).

### Scan flow

The backend calls `nmcli device wifi rescan` then parses the output
of `nmcli device wifi list`. Fields: SSID, SIGNAL, SECURITY, ACTIVE.

### Connect flow

Two-stage connect:
1. **Quick connect:** `nmcli device wifi connect <SSID> password <PSK>`
   — works for most routers.
2. **Profile fallback:** If quick connect fails (e.g., some routers
   require explicit `key-mgmt`), create a connection profile with
   `nmcli con add` and activate with `nmcli con up`.

The backend handles SSID and PSK escaping to prevent command injection.

## Windows implementation (WLAN API)

**Current gap: Windows network backend uses a stub approach.** The
current backend returns basic Wi-Fi state but scan and connect
functionality is incomplete. The design is:

- **State and IP:** `GetAdaptersAddresses` (iphlpapi).
- **Wi-Fi scan/connect:** WLAN API (`WlanScan`, `WlanConnect`).

## Captive portal support

Captive portal detection belongs to the network information layer,
not the connection backend. The runtime periodically probes a known
URL (e.g., `http://clients3.google.com/generate_204`) after
connecting to detect captive portals and presents a webview for
login. This is a runtime concern, not a backend concern.

## Porting steps

1. Create `src/backends/<platform>/<platform>_network_backend.cpp`.
2. Implement `INetworkBackend` — at minimum, `GetWiFiState` and `PrimaryIP`.
3. `ScanNetworks` and `Connect` are optional for wired-ethernet-only devices.
4. Call `Capabilities::RegisterCapability(Capability::NetworkInfo)`.
5. Register in CMake.

## Implementation gaps

| Platform | Backend status | Notes |
|---|---|---|
| Linux | ✅ Complete | getifaddrs + nmcli |
| Windows | ⚠️ Partial | State/IP works; scan/connect stub |
| Android | ❌ Missing | Need `WifiManager` via JNI |
| PS Vita | ❌ Missing | Need `sceNet` |

## Related

- [Backend Porting Model](03-backend-porting-model.md)
- [Network Module (Capability Model)](../05-capability-model/01-overview.md)
- [Device Profile Schema](02-device-profile-schema.md)
