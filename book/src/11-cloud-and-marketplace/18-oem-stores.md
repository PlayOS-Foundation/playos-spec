# OEM Stores

An **OEM Store** is a marketplace operated by a hardware manufacturer
(OEM) for their devices. It is a fully functional store source that
can operate independently or federate with the PlayOS Marketplace.

## Why OEM stores

- **Pre-installed applications** — ship the device with curated
  applications.
- **Exclusive content** — OEM-specific applications or features.
- **Brand experience** — customized store UI matching the device brand.
- **Regional content** — applications relevant to the OEM's market.
- **Revenue** — OEM earns a share of sales through their store.

## How OEM stores work

An OEM store:

1. Runs the same Marketplace Catalog API as the PlayOS Marketplace.
2. Has its own developer portal for OEM-curated applications.
3. Sets its own revenue split (OEM decides the developer share).
4. Can federate with other stores (including PlayOS Marketplace).
5. Appears alongside other stores in the device's store list.

## Configuration

OEM stores are pre-configured in the device profile:

```toml
# In the OEM device image
[[stores]]
name = "ACME Game Store"
url = "https://store.acme-hardware.com"
enabled = true
isDefault = true  # Primary store for this device

[[stores]]
name = "PlayOS Marketplace"
url = "https://catalog.playos.org"
enabled = true
```

## OEM certification requirements

Devices that ship with an OEM store and the PlayOS compatibility mark
must:

1. Include the PlayOS Marketplace as an available store source.
2. Not block installation from other store sources.
3. Not restrict side-loading of .gpk packages.
4. Pass all Platform API conformance tests.
5. Display the full permission list before installation.

These requirements ensure that OEM devices remain part of the open
PlayOS ecosystem, not walled gardens.

## Customization

OEMs can customize:

- **Store branding** — logo, colors, featured content.
- **Default store** — which store appears first.
- **Pre-installed applications** — shipped on the device.
- **OEM-specific features** — vendor capabilities tied to the hardware.

OEMs **cannot**:

- Block other stores.
- Block side-loading.
- Hide permissions from the user.
- Modify the Platform API behavior.

## Related

- [Store Sources](08-store-sources.md)
- [Federation and Private Stores](17-federation-and-private-stores.md)
- [Certified Devices](../04-target-model/05-certified-devices.md)
- [Open Platform Principle](../02-platform-principles/07-open-platform.md)
