# Network API

The `PlayOS::Network` module provides WiFi connectivity state, IP address
query, network scanning, and connection management. It is gated by the
optional capability `network.info` (enum: `Capability::NetworkInfo`).

Applications MUST check `Has(Capability::NetworkInfo)` before calling any
function in this module.

## API

```cpp
namespace PlayOS {
namespace Network {

enum class WiFiState {
    Unknown,       // No wireless hardware or state cannot be determined
    Absent,        // No wireless hardware detected
    Disconnected,  // Hardware present, not associated
    Connecting,    // Associated but IP assignment in progress
    Connected,     // Associated and has valid IPv4
};

enum class ConnectResult {
    Success,        // Connected and IP obtained
    WrongPassword,  // Authentication rejected
    Timeout,        // Did not connect within timeout (~15 s)
    Error,          // Other error
};

struct WiFiNetwork {
    std::string ssid;     // Network name
    int         signal;   // Signal strength 0–100
    bool        secured;  // Requires a password
    bool        active;   // Currently connected
};

WiFiState                 GetWiFiState();
const char*               PrimaryIP();
std::vector<WiFiNetwork>  ScanNetworks();
ConnectResult             Connect(const std::string& ssid, const std::string& psk);

} // namespace Network
} // namespace PlayOS
```

## Usage

```cpp
if (!PlayOS::Capabilities::Has(PlayOS::Capability::NetworkInfo)) {
    return; // no network hardware
}

// Check connectivity.
if (PlayOS::Network::GetWiFiState() == PlayOS::Network::WiFiState::Connected) {
    const char* ip = PlayOS::Network::PrimaryIP();
    // use ip
}

// Scan and connect.
auto networks = PlayOS::Network::ScanNetworks();
for (auto& net : networks) {
    if (net.ssid == "MyWiFi") {
        auto result = PlayOS::Network::Connect(net.ssid, "password");
        if (result == PlayOS::Network::ConnectResult::Success) {
            // connected
        }
    }
}
```

## Failure behavior

When `Capability::NetworkInfo` is absent:
- `GetWiFiState()` returns `WiFiState::Unknown`.
- `PrimaryIP()` returns an empty string.
- `ScanNetworks()` returns an empty vector.
- `Connect()` returns `ConnectResult::Error`.

## Blocking calls

`ScanNetworks()` and `Connect()` block for several seconds (scan time,
connection timeout). Applications SHOULD call these on a background thread
or during a loading screen. The shell's network settings screen is the
primary consumer.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `network.info` | `Capability::NetworkInfo` | Optional |

## Requirements

- `PrimaryIP()` MUST return the primary non-loopback IPv4 address, or
  an empty string if no address is assigned or the capability is absent.
- The returned pointer from `PrimaryIP()` is valid until the next call.
- `Connect()` MUST accept an empty PSK for open networks.
- `Connect()` MUST complete within approximately 15 seconds or return
  `Timeout`.

## Implementation note

The reference implementation (`playos/network.h`) delegates to platform
backends: Linux uses `libnm` (NetworkManager), Windows uses WLAN API. The
null backend returns `Unknown` / empty string / empty vector.

## Related

- [Network Porting](../12-device-model-and-porting/09-network-porting.md)
- [Cloud and Marketplace](../11-cloud-and-marketplace/01-overview.md)
