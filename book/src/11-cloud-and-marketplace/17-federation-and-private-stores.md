# Federation and Private Stores

**Federation** allows multiple store sources to share catalog data.
**Private stores** restrict access to specific users or devices.

## Federation

When Store A and Store B agree to federate:

1. Store A publishes a `federation.json` manifest.
2. Store B adds Store A to its federation list.
3. Store B's catalog now includes Store A's applications.
4. Players browsing Store B see applications federated from Store A.

Federation is **mutual** — both stores must agree. It is **not**
transitive — Store C federating with Store B does not give access to
Store A's applications.

### Use cases

- OEM A's store federates with OEM B's store — players on either
  device can buy applications from either catalog.
- An enterprise store federates with the public marketplace — employees
  see both internal and public applications.
- A regional store federates with the global marketplace — local
  applications appear in the global catalog.

### Federation manifest

```json
{
  "storeId": "0193a8b2-...",
  "storeName": "OEM Store A",
  "federatedStores": ["0193c4d5-..."],
  "sharedCategories": ["games", "utilities"],
  "excludedApps": ["0193e6f7-..."]  // Excluded from federation
}
```

## Private stores

A **private store** restricts catalog access:

| Restriction | Mechanism |
|---|---|
| User authentication | OAuth 2.0 — only authenticated users see the catalog |
| Device authentication | Client certificate — only registered devices |
| IP allowlist | Only from specified IP ranges |
| Invite-only | Access code required to browse |

### Enterprise deployment

An enterprise can set up a private store for internal applications:

```yaml
# Enterprise store configuration
store:
  type: private
  auth: oidc
  idp: https://idp.company.com
  allowedDomains: ["company.com"]
  
federation:
  - storeId: "playos-marketplace"  # Also show public apps
    sharedCategories: ["utilities"]
```

Employees see both internal applications and (optionally) public
applications from the federated marketplace.

## Store discovery

Devices discover available stores through:

1. **Pre-configured** — included in the device profile or OEM image.
2. **Manual entry** — user enters a store URL.
3. **QR code** — scan a QR code to add a store (handheld-friendly).

## Related

- [Store Sources](08-store-sources.md)
- [OEM Stores](18-oem-stores.md)
- [Accounts and Identity](04-accounts-and-identity.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
