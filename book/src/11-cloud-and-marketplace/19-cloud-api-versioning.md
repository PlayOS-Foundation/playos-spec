# Cloud API Versioning

The Cloud API follows **semantic versioning** (MAJOR.MINOR.PATCH) with
strict compatibility guarantees.

## Version format

`MAJOR.MINOR.PATCH`

| Bump | Meaning | Compatible? |
|---|---|---|
| PATCH | Bug fixes, no API changes | ✅ Fully compatible |
| MINOR | New endpoints, new optional fields | ✅ Backward compatible |
| MAJOR | Breaking changes (removed fields, changed semantics) | ❌ Not compatible |

## Version negotiation

The Runtime sends its supported API version in the `Accept-Version`
header:

```
GET /catalog/v1/apps
Accept-Version: 1.0
```

The server responds with the version it used:

```
HTTP/1.1 200 OK
Content-Version: 1.2
```

If the server cannot satisfy the requested version (e.g., the Runtime
requests `2.0` but the server only supports `1.x`), it returns:

```
HTTP/1.1 406 Not Acceptable
Content-Version: 1.2
```

The Runtime then retries with a lower version.

## Backward compatibility rules

1. **Do not remove fields** — mark them as deprecated instead.
2. **Do not change field types** — add a new field with a new name.
3. **New fields must be optional** — existing clients ignore them.
4. **New endpoints are fine** — old clients don't call them.
5. **Deprecated fields** are supported for 2 MINOR versions after
   deprecation.

## Breaking changes (MAJOR)

A MAJOR version bump is required when:

- A field is removed.
- A field's type or semantics change.
- An endpoint's behavior changes in an incompatible way.
- Authentication requirements change.

Old MAJOR versions are supported for 12 months after a new MAJOR
version is released.

## Protocol buffer versioning

Cloud APIs use Protocol Buffers (`.proto` files). Protobuf's
[backward compatibility rules](https://protobuf.dev/programming-guides/proto3/)
apply:

- Do not change field numbers.
- Do not change field types.
- Reserved fields prevent accidental reuse.

## Service version independence

Each cloud service versions independently:

```
Accounts Service:     1.3.0
Saves Service:         0.9.1
Leaderboard Service:   2.1.0
Achievements Service:  1.0.5
```

The Runtime negotiates version per-service through the API Gateway.

## Deprecation timeline

```
T+0:  Field marked deprecated. Still functional.
T+6:  Deprecated field logs a warning.
T+12: Deprecated field returns default value (MINOR version removed).
T+24: Deprecated field removed (next MAJOR).
```

## Related

- [API Versioning (Platform API)](../06-platform-api/25-api-versioning.md)
- [Cloud Architecture](03-cloud-architecture.md)
- [Developer Portal](05-developer-portal.md)
