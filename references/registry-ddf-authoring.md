# OMNA Registry & DDF Authoring — Publication Format, Validation & Submission

Authoritative reference for authoring, validating, and submitting LwM2M Objects and Reusable Resources against the OMNA LwM2M Registry. It covers the real publication format of the `OpenMobileAlliance/lwm2m-registry` repository, the DDF (Definition Description File) schema, ID allocation, URN composition, the validation fault model, registry synchronisation, and version-history handling.

Where a rule is enforced by a certification/registry platform rather than published verbatim in an OMA specification, it is annotated **platform gate code (not guaranteed official OMA)**. The two well-known official OMA validator faults (Type/Operation matrix and SenML Unit) are flagged explicitly. Spelling is Australian/British (optimise, behaviour, normalise).

## Table of Contents
1. [Registry Publication Format (DDF.xml, not JSON)](#registry-publication-format-ddfxml-not-json)
2. [The Three Concurrent Libraries](#the-three-concurrent-libraries)
3. [Object ID Allocation Ranges](#object-id-allocation-ranges)
4. [Object URN Composition](#object-urn-composition)
5. [DDF.xml Object & Resource Schema](#ddfxml-object--resource-schema)
6. [Resource Types & Operations](#resource-types--operations)
7. [Validation Fault-Code Table](#validation-fault-code-table)
8. [Fault 413 — Execute/Type Matrix](#fault-413--executetype-matrix)
9. [Fault 417 — SenML Unit Symbols](#fault-417--senml-unit-symbols)
10. [Fault 418 — RangeEnumeration Syntax](#fault-418--rangeenumeration-syntax)
11. [Registry Sync & Normalisation](#registry-sync--normalisation)
12. [Defensive Parsing](#defensive-parsing)
13. [XXE Hardening](#xxe-hardening)
14. [Object Version History Model](#object-version-history-model)
15. [Per-Tenant Custom Object Lifecycle](#per-tenant-custom-object-lifecycle)
16. [Common Submission Failures → Fix](#common-submission-failures--fix)
17. [Key Specification References](#key-specification-references)
18. [Related Files](#related-files)

---

## Registry Publication Format (DDF.xml, not JSON)

A common assumption is that the registry publishes a JSON index (`index.json` / `full-index.json`). This is **incorrect** for the live `OpenMobileAlliance/lwm2m-registry` `prod` branch — both of those paths return HTTP 404.

The master index is **`DDF.xml`**. It is a `<DDFList>` of `<Item>` entries, where each `<Item>` carries:

| Element | Meaning |
|---------|---------|
| `<ObjectID>` | The numeric Object ID |
| `<DDF>` | Path to that object's definition file |

The `<DDF>` path resolves at the repository root for current objects (e.g. `3303.xml`) or under `version_history/` for historic snapshots. Consumers MUST:

1. Fetch and parse `DDF.xml`.
2. For each `<Item>`, resolve its `<DDF>` path **relative to the index URL** (not the repo root assumed independently).
3. Fetch and parse each resolved DDF document.

A real synchronisation of `prod/DDF.xml` yielded approximately **365 objects / ~3900 resources**.

**Architectural rule:** *sync the registry; don't fork it inline.* Treat `DDF.xml` and the per-object DDF files as an upstream-tracked source. Do not embed a hand-edited copy of OMNA definitions into your own object library — pull them via a sync that records provenance and content hashes (see [Object Version History Model](#object-version-history-model)).

```
DDF.xml (DDFList)
 ├─ Item: ObjectID 3303, DDF "3303.xml"            → resolve relative to index URL
 ├─ Item: ObjectID 3304, DDF "3304.xml"
 └─ Item: ObjectID  3,   DDF "version_history/3-1_0.xml"
```

---

## The Three Concurrent Libraries

The platform maintains three concurrent libraries, queried as a merged view:

| Library | Source | Contents |
|---------|--------|----------|
| `standard` | OMNA GitHub `prod` branch | Mandatory + optional **production** objects |
| `full` | Extended listing of `extras/` + `pending/` | Pre-release / proposed / experimental objects |
| `custom` | Per-tenant vendor-private store | Tenant's own private objects |

A merged query overlays the libraries with **`custom` overriding `standard`/`full` on ID collision**, so a tenant's private definition of an ObjectID takes precedence over an upstream one.

**Natural key.** A registry entry is uniquely identified by the tuple:

```
(ObjectID, LWM2MVersion, library)
```

ObjectID is **not globally unique** across versions or libraries — the same ObjectID can legitimately exist at different `LWM2MVersion` values and in different libraries. Never key storage, caches, or diffs on ObjectID alone.

---

## Object ID Allocation Ranges

Classification is **order-sensitive**: bands overlap, so they must be tested in the sequence below. In particular the uCIFI Smart City carve-out sits inside the IPSO range and MUST be checked **before** the IPSO band.

| Order | Range | Classification | Notes |
|-------|-------|----------------|-------|
| 1 | `< 0` | invalid | Negative IDs are never valid |
| 2 | `0–99` | OMA core | Core objects (Security, Server, Device, etc.) |
| 3 | `3400–3449` | uCIFI Smart City | **Check this band FIRST** — carved out of the IPSO range |
| 4 | `3200–3499` | IPSO | IPSO Smart Objects (excluding the uCIFI carve-out) |
| 5 | `10262–10284` | SEW Digital Utility | Water metering — South East Water "SEW Digital Utility IoT LwM2M Technical Specification" |
| 6 | `32769–42768` | vendor/custom | Vendor-allocated objects |
| 7 | else | oma-extension | Any other registered OMA extension |

**Vendor-private gate = 32768.** A `custom` object with `ObjectID < 32768` is **rejected** unless an explicit OMNA-range override is held for that tenant. The OMNA experimental range is documented as **26241–32768**, but the platform gates strictly at **32768** (anything below is treated as OMNA-administered). This strict gate is **platform gate code (not guaranteed official OMA)** — OMA formally allocates IDs and the experimental band is documentary.

```
                0   100        3200  3400  3449  3499      10262 10284     26241        32768 32769      42768
OMA core      [===]
uCIFI                                  [==========]
IPSO                       [========================]    (minus uCIFI carve-out)
SEW Digital                                              [===========]
OMNA experimental                                                          [=================]
vendor/custom                                                                              gate→[=============]
```

---

## Object URN Composition

| Object class | URN form | Example |
|--------------|----------|---------|
| OMA-registered | `urn:oma:lwm2m:oma:<id>` | core/OMA objects |
| OMA extension (IPSO etc.) | `urn:oma:lwm2m:ext:<id>` | IPSO Temperature 3303 → `urn:oma:lwm2m:ext:3303` |
| Vendor | `urn:oma:lwm2m:x:<ObjectID>` | optionally with a trailing `:...` segment |

**Invariant:** the ObjectID encoded in the URN MUST equal the document's `<ObjectID>`. A mismatch is a URN fault (see fault **401**). The URN scheme prefix is `urn:oma:lwm2m:` as defined in `OMA-TS-LightweightM2M_Core-V1_2_2, §7.2` (Object/Resource identification).

---

## DDF.xml Object & Resource Schema

An object definition is a root `<Object>` element, or an `<Object>` nested under an `<LWM2M>` wrapper. Both layouts are accepted.

**Object-level elements:**

| Element | Required | Type / Values | Notes |
|---------|----------|---------------|-------|
| `<ObjectID>` | yes | integer | Must be parseable as int; non-integer → object skipped |
| `<Name>` | yes | string | Empty → fault 220 |
| `<ObjectURN>` | yes | URN | See [URN composition](#object-urn-composition) above; problems → fault 401 |
| `<LWM2MVersion>` | yes | e.g. `1.0`, `1.1`, `1.2` | Missing → fault 800 |
| `<ObjectVersion>` | recommended | e.g. `1.0` | Object schema version |
| `<Mandatory>` | yes | literal `Mandatory` / `Optional` | Exact string equality |
| `<MultipleInstances>` | yes | literal `Multiple` / `Single` | Exact string equality |
| `<Description1>` | recommended | string | **Note the trailing `1`** in the tag name; empty → fault 230 |
| `<Resources>` | yes | container | Holds `<Item>` resource entries |

**Resource-level elements** — each resource is an `<Item ID="...">` where `ID` is an **XML attribute** (not a child element), with children:

| Child | Values | Notes |
|-------|--------|-------|
| `<Name>` | string | Empty → fault 410 |
| `<Operations>` | `R` / `W` / `RW` / `E` | Normalised by upper-casing, keeping only R/W/E; invalid → fault 411 |
| `<Type>` | one of the 9 canonical types | See below; invalid → fault 412 |
| `<Mandatory>` | literal `Mandatory` / `Optional` | Exact string equality |
| `<MultipleInstances>` | literal `Multiple` / `Single` | Exact string equality |
| `<Units>` | SenML symbol, blank, or `TBD` | Invalid → fault 417 |
| `<RangeEnumeration>` | `lo..hi` form | Must use `..`; invalid → fault 418 |
| `<Description>` | string | Free text |

`<Mandatory>` and `<MultipleInstances>` are determined by **exact string equality** against their literal tokens at both object and resource level — any other value is treated as the non-mandatory / single-instance default.

---

## Resource Types & Operations

There are **9 canonical resource Types** (`OMA-TS-LightweightM2M_Core-V1_2_2, §7.4.3` data type table; `objlnk` v1.0+, `corelnk` v1.1+):

| Canonical token | DDF `<Type>` spelling | Notes |
|-----------------|-----------------------|-------|
| `string` | `String` | UTF-8 text |
| `integer` | `Integer` | Signed |
| `unsigned_integer` | `Unsigned Integer` | **Two-word DDF spelling** maps to `unsigned_integer` (v1.1+) |
| `float` | `Float` | |
| `boolean` | `Boolean` | |
| `opaque` | `Opaque` | Raw bytes |
| `time` | `Time` | Unix time |
| `objlnk` | `Objlnk` | Object link |
| `corelnk` | `Corelnk` | CoRE Link Format (v1.1+) |

Any `<Type>` token outside this allow-list → fault **412**.

**Operations.** Values are normalised by upper-casing and keeping only the characters R/W/E. Valid Operations are **exactly** `R`, `W`, `RW`, `E`. There is **no `RWE`** — a resource is either a value resource (readable and/or writable, *with* a Type) or an Executable command (`E`, *without* a Type), never both. Anything else → fault **411**.

---

## Validation Fault-Code Table

These are the platform's gate codes, modelled on OMNA validator behaviour. Codes **413** (Type/Operation) and **417** (SenML Unit) are the **well-known official OMA validator faults**; the remainder are **platform gate codes (not guaranteed official OMA)**.

| Code | Trigger | Origin |
|------|---------|--------|
| `200` | ObjectID in OMNA-administered range (i.e. `< 32768` without override) | platform gate |
| `220` | Empty Object Name | platform gate |
| `230` | Empty Object Description (`<Description1>`) | platform gate |
| `401` | URN problems: missing; not in `x:` vendor namespace for a custom object; non-numeric ID segment; URN ObjectID ≠ `<ObjectID>` | platform gate |
| `410` | Empty Resource Name | platform gate |
| `411` | Operations not one of `R` / `W` / `RW` / `E` | platform gate |
| `412` | Invalid Type (not in the 9-type allow-list) | platform gate |
| `413` | **Type/Operation matrix violation** | **official OMA validator fault** |
| `414` | Execute resource not Single-instance | platform gate |
| `417` | **Units not a valid SenML symbol** | **official OMA validator fault** |
| `418` | RangeEnumeration not using `..` | platform gate |
| `800` | Missing `<LWM2MVersion>` | platform gate |

**Pseudo-codes** (structural, not numeric):

| Pseudo-code | Trigger |
|-------------|---------|
| `RES_EMPTY` | Object declares zero resources |
| `RES_DUP` | Duplicate Resource ID within an object |
| `INVALID_XML` | Document is not well-formed XML |

---

## Fault 413 — Execute/Type Matrix

Fault 413 enforces the Type/Operation relationship from `OMA-TS-LightweightM2M_Core-V1_2_2, §7.4` (resource operation/data-type rules). The matrix is precise:

| Operation class | `<Type>` | `<Units>` | `<RangeEnumeration>` | `<MultipleInstances>` |
|-----------------|----------|-----------|----------------------|-----------------------|
| Execute (`E`) | MUST be absent → present is **413** | MUST be absent → present is **413** | MUST be absent → present is **413** | MUST be `Single` → `Multiple` is **414** |
| Value (`R` / `W` / `RW`) | MUST be present → missing is **413**; present-but-unknown is **412** | optional | optional | `Single` or `Multiple` |

An **Execute** resource is a command trigger: it carries no value, so declaring a Type, Units, or RangeEnumeration is meaningless and rejected, and it must be a single instance. A **value** resource (R/W/RW) must always declare a Type — a missing Type is 413, an unrecognised Type is 412.

---

## Fault 417 — SenML Unit Symbols

A `<Units>` value is valid **only** when it is one of:

- blank (no units),
- the literal `TBD` (placeholder telling IPSO to assign a unit later), or
- an **exact, case-sensitive** match in the SenML unit registry (`RFC 8428` primary, `RFC 8798` secondary, plus the live IANA SenML Units registry).

Matching is **case-sensitive** and against the symbol, not the name.

**Accepted symbol set (grouped):**

| Group | Symbols |
|-------|---------|
| SI base/derived | `m` `kg` `g` `s` `A` `K` `cd` `mol` `Hz` `rad` `sr` `N` `Pa` `J` `W` `C` `V` `F` `Ohm` `S` `Wb` `T` `H` `Cel` `lm` `lx` `Bq` `Gy` `Sv` `kat` |
| Area/volume/flow | `m2` `m3` `l` `m/s` `m/s2` `m3/s` `l/s` |
| Radiometric/acoustic/chemical | `W/m2` `cd/m2` `pH` `dB` `dBW` `Bspl` |
| Information | `bit` `bit/s` `B` |
| Geo/ratios/rates | `lat` `lon` `count` `/` `%` `%RH` `%EL` `EL` `1/s` `1/min` `beat/min` `beats` `S/m` |
| Power/energy/misc | `VA` `VAs` `var` `vars` `J/m` `kg/m3` `deg` `NTU` |
| Scaled secondary units | `ms` `min` `h` `MHz` `kW` `kVA` `kvar` `kWh` `km/h` `Mbit/s` `kbit/s` |

**Critical gotchas:**

- **Temperature is `Cel`** — *not* `C` (which is coulomb, electric charge) and *not* `°C`.
- **`TBD`** is the legal placeholder; use it rather than guessing a symbol when the unit is genuinely undecided.

> Do not invent unit symbols. If a needed unit is not in the registry, use `TBD` and flag it, or raise the unit upstream — `TODO(verify): confirm any unit not listed above against the live IANA SenML Units registry before submission`.

---

## Fault 418 — RangeEnumeration Syntax

`<RangeEnumeration>` must express a numeric range using the **`..`** separator.

| Value | Result |
|-------|--------|
| `0..100` | valid |
| *(blank)* | valid (no range) |
| `0-100` | **rejected** (numeric dash) → 418 |
| `0 to 100` | **rejected** (word "to") → 418 |

---

## Registry Sync & Normalisation

When persisting a synced OMNA resource whose source DDF leaves `<Operations>` or `<Type>` empty, the writer substitutes defaults:

| Missing field | Substituted default |
|---------------|---------------------|
| `<Operations>` empty | `R` |
| `<Type>` empty | `string` |

Consequently a synced resource may display `R` / `string` even when the source DDF omitted those elements. This is **tool-applied normalisation, not necessarily present in the source** — never treat a synced `R`/`string` as evidence the upstream DDF declared them. Record provenance so the substitution is auditable.

---

## Defensive Parsing

DDF ingestion must be tolerant of malformed individual entries without aborting the whole run:

| Condition | Handling |
|-----------|----------|
| Document has no `<Object>` | Document **skipped entirely** |
| `<ObjectID>` is non-integer | Document **skipped entirely** |
| Resource `<Item>` `ID` missing or non-numeric | That **resource skipped individually** (rest of object kept) |
| Object unreachable / unparseable | **Counted as skipped**, run continues |
| No index (`DDF.xml`) could be read at all | **Fail closed** — abort the sync |

The only fatal condition is total failure to read the index. Per-object and per-resource failures degrade gracefully and are reported in skip counts.

---

## XXE Hardening

Treat all DDF / registry XML as **untrusted input**. Every parse of untrusted DDF/registry XML MUST:

| Mitigation | Reason |
|------------|--------|
| Disable DOCTYPE declarations | Blocks DTD-based entity attacks |
| Disable XInclude | Blocks include-based file pulls |
| Disable external entity expansion | Blocks classic XXE / SSRF |
| Disable namespace awareness | Reduces parser attack surface; DDF does not require it |
| Validate fetch URL scheme is strictly `http`/`https` before fetching | Closes `file://` path-traversal / local-file disclosure |

The combination of disabled external entities and scheme validation closes both XXE document attacks and `file://` exfiltration via the `<DDF>` path or any resolved URL.

---

## Object Version History Model

History is **append-only**, keyed by a content hash:

- **Snapshot key:** `SHA-256` (lowercase hex) of the **canonical DDF XML**, UTF-8 encoded. The content hash serves as both change-detection and dedup key — identical content never produces a duplicate snapshot.
- **Snapshot tags:** `added`, `modified`, `removed`.
- **Retirement:** an object that disappears from the index receives a `removed` snapshot, then is retired.
- **Rollback/restore:** stays append-only — re-applies a chosen snapshot's DDF and records a **new `modified`** entry. History is never rewritten.

**Structured diff.** A DDF diff compares the **parsed model** (not raw XML), so whitespace and formatting never appear as changes. Resources are matched **by ID**, and the diff reports, in **deterministic order**:

| Change category | Meaning |
|-----------------|---------|
| Added | Resource present in new, absent in old |
| Removed | Resource present in old, absent in new |
| Retyped | Resource `<Type>` changed |
| RangeChanged | `<RangeEnumeration>` changed |
| OperationsChanged | `<Operations>` changed |

---

## Per-Tenant Custom Object Lifecycle

Custom (vendor-private) objects follow a status state machine:

```
draft ──▶ active ──▶ deprecated
            └──────▶ frozen
```

| Aspect | Rule |
|--------|------|
| Initial version | Created at SemVer `0.1.0` |
| Version bumps | Uploads bump `MAJOR` / `MINOR` / `PATCH` per SemVer |
| `frozen` | Blocks all further writes |

**Optional hardening pattern (implementation guidance, NOT a spec requirement).** A hardened platform protecting vendor IPR may:

- Envelope-encrypt custom DDF at rest with **AES-256-GCM** under a wrapped DEK (Data Encryption Key).
- Attach a **detached Ed25519 signature** over the canonical plaintext.
- Enforce **per-tenant PostgreSQL Row-Level Security (RLS)**.

These protect confidentiality, integrity, and tenant isolation of private definitions. They are deployment choices, not OMA registry requirements.

---

## Common Submission Failures → Fix

| Fault | One-line fix |
|-------|--------------|
| `200` | Use `ObjectID ≥ 32768` (or hold an OMNA-range override) |
| `220` | Add a non-empty `<Name>` |
| `230` | Add a non-empty `<Description1>` (note the trailing `1`) |
| `401` | Use `urn:oma:lwm2m:x:<id>` for vendor objects and make the URN ID equal `<ObjectID>` |
| `410` | Add a non-empty resource `<Name>` |
| `411` | Set `<Operations>` to exactly `R`, `W`, `RW`, or `E` |
| `412` | Use one of the 9 canonical Types (e.g. `Unsigned Integer` → `unsigned_integer`) |
| `413` | Remove Type/Units/Range from an Execute resource **or** add a Type to a value resource |
| `414` | Make the Execute resource `Single`-instance |
| `417` | Use an exact SenML symbol — temperature is `Cel` (not `C`/`°C`); or use `TBD` |
| `418` | Use `..` (e.g. `0..100`), not `0-100` or `0 to 100` |
| `800` | Add `<LWM2MVersion>` (e.g. `1.1`) |
| `RES_EMPTY` | Declare at least one resource |
| `RES_DUP` | Make every Resource `ID` unique within the object |
| `INVALID_XML` | Fix well-formedness; re-validate before submission |

---

## Key Specification References

| Reference | Section / Scope | Used for |
|-----------|-----------------|----------|
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.2 | Object/Resource identification, URN scheme |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.4 | Resource Operations / Type rules (fault 413 matrix) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.4.3 | Canonical data-type table (9 types) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.4.2 | Registration/registry behaviour |
| `RFC 8428` | SenML | Primary SenML unit symbol registry (fault 417) |
| `RFC 8798` | Additional SenML units | Secondary unit symbols (fault 417) |
| IANA SenML Units registry | live registry | Authoritative current unit list |
| `RFC 7252` | CoAP | Underlying transport for registry-facing operations |
| South East Water | "SEW Digital Utility IoT LwM2M Technical Specification" | ID band 10262–10284 (water metering) |
| `OpenMobileAlliance/lwm2m-registry` | `prod` branch, `DDF.xml` | Live registry publication format |

> Where a section number could not be independently confirmed against the published TS, treat it as indicative: `TODO(verify): confirm exact §-numbers against OMA-TS-LightweightM2M_Core-V1_2_2 before quoting in a submission.`

---

## Related Files

- [objects.md](objects.md) — Object model, core/IPSO/industry objects, ID ranges, reusable resources, OMNA registry process overview (this file is the authoring/validation deep-dive of that process).
- [protocol-details.md](protocol-details.md) — CoAP transport, registration/bootstrap flows, content-format negotiation underpinning registry-facing operations.
- [operation-coap-mapping.md](operation-coap-mapping.md) — LwM2M interface-to-CoAP operation mapping used when exercising registered objects.
