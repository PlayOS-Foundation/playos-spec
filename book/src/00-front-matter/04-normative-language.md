# Normative Language

This specification uses the key words defined in
[RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) and
[RFC 8174](https://datatracker.ietf.org/doc/html/rfc8174).

## MUST

The word "must" (or "required", "shall", "has to") means the
requirement is absolute. Implementations that violate a MUST
requirement are non-conformant.

> **Example:** "A Runtime Device MUST ship a device profile."

## MUST NOT

The word "must not" (or "shall not") means the prohibition is
absolute. Implementations that violate a MUST NOT prohibition are
non-conformant.

> **Example:** "The Platform API MUST NOT depend on Raylib."

## SHOULD

The word "should" (or "recommended") means the requirement is strongly
recommended but may be violated in specific circumstances with careful
consideration.

> **Example:** "Applications SHOULD query capabilities at runtime
> rather than at compile time."

## SHOULD NOT

The word "should not" (or "not recommended") means the behavior is
strongly discouraged but may be acceptable in specific circumstances.

> **Example:** "Implementations SHOULD NOT hardcode device-specific
> assumptions."

## MAY

The word "may" (or "optional") means the requirement is truly
optional. Implementations may include or omit the feature at their
discretion.

> **Example:** "An SDK Target MAY use the null backend for audio."

## Capability-specific language

When a requirement applies only when a capability is present, it is
qualified:

> **Example:** "When `input.touch` is present, the device MUST report
> touch events within 16 ms of physical contact."

## Schema conformance

References to "valid according to the schema" or "schema-valid" mean
the data passes validation against the referenced JSON Schema or TOML
schema. Schema violations are non-conformant.

## RFCs and ADRs

RFCs and ADRs may use the same normative language. When an RFC is
accepted and merged into the specification, its normative statements
become part of the specification.
