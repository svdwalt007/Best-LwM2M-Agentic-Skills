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
(ObjectID, ObjectVersion, library)      — equivalently (ObjectURN, library)
```

The Object URN already encodes both parts: `urn:oma:lwm2m:{oma|ext|x}:{ObjectID}[:{ObjectVersion}]` (Core TS 1.2.2 §7.2.1, Appendix D.2.1), with the `:{version}` segment omitted for 1.0 (§7.2.2). ObjectID is **not globally unique** across versions or libraries — `DDF.xml` lists Object 20 four times (`urn:oma:lwm2m:oma:20`, `:20:2.0`, `:20:2.1`, `:20:3.0`), each resolving to a different DDF file. Never key storage, caches, or diffs on ObjectID alone.

`LWM2MVersion` is **not** part of the key: it is the *minimum LwM2M Enabler version* that supports the Object (Core TS 1.2.2 Appendix J), not a version of the Object. Two Object Versions can share one `LWM2MVersion` (Security 1.1 and 1.2 both carry `1.1`), and one Object Version never carries two.

---

## Object ID Allocation Ranges

The normative classes are **non-overlapping** and defined in Core TS 1.2.2 Appendix D.2.1, Table D.2.1-1 "LwM2M Object Classes". Classification needs no ordering tricks — one range test per class:

| Range | Class | URN segment | Notes |
|-------|-------|-------------|-------|
| `< 0`, `65535` | invalid | — | Negative IDs are never valid; 65535 (MAX_ID) MUST NOT identify an Object (Core TS 1.2.2 Table 7.4-1) |
| `0–1023` | OMA | `oma` | Objects produced by OMA: core objects 0–7, companion-enabler objects 8–28, and further OMA objects such as 500–510 (incl. 504 Remote SIM Provisioning). The "0–99 core" cut-off used by some tools has no basis in the spec |
| `1024–2047` | reserved | — | Reserved for future use |
| `2048–10240` | SDO | `ext` | Third-party standards organisations and alliances — oneM2M (2048+), IPSO (3200–3350), GSMA (3351–3386, 3443), OpenAIS (3387–3406), uCIFI (3410–3449 less 3441–3443), Eclipse (3441–3442) |
| `10241–32768` | Vendors | `x` | Vendors or individuals; **published** on the registry (e.g. SEW Digital Utility 10262–10284, IoTerop 18830–18831) |
| `32769–42768` | Company Reserved | `x` | Bulk reservation of up to 50 IDs for private Objects **not published** on the registry |
| `42769–42800` | Test | `x` | Testing only; MUST NOT be used in production |
| `42801–65534` | reserved | — | Reserved for future use |

Owner sub-bands inside the SDO class (IPSO, GSMA, OpenAIS, uCIFI …) are **registry observations**, not spec-defined carve-outs: OMNA allocates `ext` IDs sequentially to whoever registers next. There is therefore no "uCIFI carve-out of the IPSO range" — IPSO-owned IDs stop at 3350 — and **Object 3443 inside the uCIFI run is GSMA's eSIM IoT object** (`urn:oma:lwm2m:ext:3443`, SGP.32 eIM↔IPA), so any owner inference from the ID alone is wrong. Read `<Owner>`/`<Source>` from `DDF.xml` instead of inferring from the number (registry-mirror evidence; verify against the live OMNA registry).

**Vendor-private gate.** A platform that stores tenant-private (`custom`) objects should require them to sit in the **Company Reserved** class (`32769–42768`) or the Test class, because everything at or below 32768 is either OMA/SDO-administered or a *published* vendor allocation that must be obtained from OMNA. A gate expressed as "reject `ObjectID < 32768`" is off by one against Table D.2.1-1 (32768 is the last *published* vendor ID; private IDs start at 32769). The spec defines **no** "OMNA experimental range 26241–32768"; that figure is not in Core TS 1.2.2 and should not be quoted. The strict gate itself is **platform gate code (not guaranteed official OMA)** — OMA formally allocates IDs and only the 32769–42800 bands need no registration.

```
        0        1023 1024    2047 2048            10240 10241            32768 32769        42768 42769 42800 42801  65534
OMA     [===========]
reserved               [=========]
SDO ext                            [==================]
Vendors x                                                [===================]
Company Reserved x                                                              [=============]
Test x                                                                                           [====]
reserved                                                                                               [=========]
```

---

## Object URN Composition

General form (Core TS 1.2.2 §7.2.1 and Appendix D.2.1): **`urn:oma:lwm2m:{oma|ext|x}:{ObjectID}[:{ObjectVersion}]`**

| Object class | URN form | Example |
|--------------|----------|---------|
| OMA (0–1023) | `urn:oma:lwm2m:oma:<id>[:<ver>]` | Server 1.2 → `urn:oma:lwm2m:oma:1:1.2`; Location 1.0 → `urn:oma:lwm2m:oma:6` |
| SDO / alliance (2048–10240) | `urn:oma:lwm2m:ext:<id>[:<ver>]` | IPSO Temperature 3303 → `urn:oma:lwm2m:ext:3303`; GSMA eSIM IoT → `urn:oma:lwm2m:ext:3443` |
| Vendor, Company Reserved, Test (10241–42800) | `urn:oma:lwm2m:x:<id>[:<ver>]` | `urn:oma:lwm2m:x:10262` |

**Version segment (§7.2.2).** Every Object Version gets its own URN; for version 1.0 the segment MAY be omitted, so `urn:oma:lwm2m:oma:44` and `urn:oma:lwm2m:oma:44:1.0` both denote Object 44 v1.0, while `urn:oma:lwm2m:oma:44:2.2` denotes v2.2. The DDF file on the OMNA portal is named after this URN (Appendix J).

**Invariants:** the ObjectID encoded in the URN MUST equal the document's `<ObjectID>`; the version segment, when present, MUST equal `<ObjectVersion>`; the class segment MUST match the ID range in Table D.2.1-1. A mismatch is a URN fault (see fault **401**). The URN scheme prefix is `urn:oma:lwm2m:` as defined in Core TS 1.2.2 §7.2.1 (Object Versioning — General Policy) and Appendix D.2.1.

---

## DDF.xml Object & Resource Schema

An object definition is a root `<Object>` element, or an `<Object>` nested under an `<LWM2M>` wrapper. Both layouts are accepted.

**Object-level elements:**

| Element | Required | Type / Values | Notes |
|---------|----------|---------------|-------|
| `<ObjectID>` | yes | integer | Must be parseable as int; non-integer → object skipped |
| `<Name>` | yes | string | Empty → fault 220 |
| `<ObjectURN>` | yes | URN | See [URN composition](#object-urn-composition) above; problems → fault 401 |
| `<LWM2MVersion>` | **optional** (schema) | e.g. `1.0`, `1.1`, `1.2` | Minimum LwM2M Enabler version supporting the Object; **MAY be omitted when it is 1.0** (Core TS 1.2.2 Appendix J). Platform fault 800 when missing is a local policy, not an OMA rule |
| `<ObjectVersion>` | **optional** (schema) | e.g. `1.0`, `2.1` | Object Version per §7.2; MAY be omitted when 1.0, or when the Object ships inside an Enabler release (Appendix J). Absent ⇒ treat as 1.0 |
| `<Mandatory>` | yes | literal `Mandatory` / `Optional` | Exact string equality |
| `<MultipleInstances>` | yes | literal `Multiple` / `Single` | Exact string equality |
| `<Description1>` | recommended | string | **Note the trailing `1`** in the tag name; empty → fault 230 |
| `<Resources>` | yes | container | Holds `<Item>` resource entries |

**Resource-level elements** — each resource is an `<Item ID="...">` where `ID` is an **XML attribute** (not a child element), with children:

| Child | Values | Notes |
|-------|--------|-------|
| `<Name>` | string | Empty → fault 410 |
| `<Operations>` | `R` / `W` / `RW` / `E` / **blank** | Normalised by upper-casing, keeping only R/W/E; invalid → fault 411. **Blank is valid**: it means no Device Management & Service Enablement operation is allowed — every resource of Objects 0 (Security), 21 (OSCORE) and 23 (COSE) is blank (Bootstrap-only), and Event Log 4016 `LogDataIncremental` is blank because it is Send-only |
| `<Type>` | one of the 10 canonical types (9 value types + `none`) | See below; invalid → fault 412 |
| `<Mandatory>` | literal `Mandatory` / `Optional` | Exact string equality |
| `<MultipleInstances>` | literal `Multiple` / `Single` | Exact string equality |
| `<Units>` | SenML symbol, blank, or `TBD` | Invalid → fault 417 |
| `<RangeEnumeration>` | `lo..hi` form | Must use `..`; invalid → fault 418 |
| `<Description>` | string | Free text |

`<Mandatory>` and `<MultipleInstances>` are determined by **exact string equality** against their literal tokens at both object and resource level — any other value is treated as the non-mandatory / single-instance default.

---

## Resource Types & Operations

There are **10 canonical resource Types**: the 9 value data types of Core TS 1.2.2 **Appendix C "Data Types (Normative)"** plus **`none`**, which Appendix D.1 prescribes for Execute resources ("MUST have no associated Data Type (none) encoded as an empty value in Object DDF file"). `objlnk` dates from v1.0; `unsigned_integer` and `corelnk` were added in v1.1.

| Canonical token | DDF `<Type>` spelling | Notes (Appendix C) |
|-----------------|-----------------------|--------------------|
| `string` | `String` | UTF-8 text; min/max length MAY be given in RangeEnumeration |
| `integer` | `Integer` | 8/16/32/64-bit signed; also used for enumerations |
| `unsigned_integer` | `Unsigned Integer` | **Two-word DDF spelling** maps to `unsigned_integer` (v1.1+); also used as a bitmask |
| `float` | `Float` | 32- or 64-bit |
| `boolean` | `Boolean` | 8-bit, 0 = False, 1 = True |
| `opaque` | `Opaque` | Binary octets; length constraints in RangeEnumeration |
| `time` | `Time` | Unix time, signed seconds since 1970-01-01 UTC |
| `objlnk` | `Objlnk` | Object link; `65535:65535` = null link |
| `corelnk` | `Corelnk` | CoRE Link Format (RFC 6690), v1.1+ |
| `none` | *(empty `<Type></Type>`)* | Execute resources only (Appendix D.1) |

Any `<Type>` token outside this allow-list → fault **412**.

**Operations.** Values are normalised by upper-casing and keeping only the characters R/W/E. Valid Operations are **exactly** `R`, `W`, `RW`, `E`, or **blank**. There is **no `RWE`** — a resource is either a value resource (readable and/or writable, *with* a Type) or an Executable command (`E`, *with* Type `none`), never both. A blank `<Operations>` is a legitimate declaration that the resource has no DM&SE operation (Bootstrap-only credentials in Objects 0/21/23; the Send-only Event Log resource 4016) and MUST NOT be rewritten to `R`. Anything else → fault **411**.

---

## Validation Fault-Code Table

These are the platform's gate codes, modelled on OMNA validator behaviour. Codes **413** (Type/Operation) and **417** (SenML Unit) are the **well-known official OMA validator faults**; the remainder are **platform gate codes (not guaranteed official OMA)**.

| Code | Trigger | Origin |
|------|---------|--------|
| `200` | ObjectID of a *custom* object outside the Company Reserved / Test classes (Core TS 1.2.2 Table D.2.1-1: private IDs are `32769–42768`, test `42769–42800`; platforms commonly gate at `< 32768`, which is off by one) without override | platform gate |
| `220` | Empty Object Name | platform gate |
| `230` | Empty Object Description (`<Description1>`) | platform gate |
| `401` | URN problems: missing; not in `x:` vendor namespace for a custom object; non-numeric ID segment; URN ObjectID ≠ `<ObjectID>` | platform gate |
| `410` | Empty Resource Name | platform gate |
| `411` | Operations not one of `R` / `W` / `RW` / `E` / blank | platform gate |
| `412` | Invalid Type (not in the 10-type allow-list: Appendix C types + `none`) | platform gate |
| `413` | **Type/Operation matrix violation** | **official OMA validator fault** |
| `414` | Execute resource not Single-instance | platform gate |
| `417` | **Units not a valid SenML symbol** | **official OMA validator fault** |
| `418` | RangeEnumeration not using `..` | platform gate |
| `800` | Missing `<LWM2MVersion>` — **platform policy only**: the element is optional in the LwM2M schema and MAY be omitted when the minimum Enabler version is 1.0 (Core TS 1.2.2 Appendix J). Registry DDFs such as Object 15 v1.0 omit it | platform gate |

**Pseudo-codes** (structural, not numeric):

| Pseudo-code | Trigger |
|-------------|---------|
| `RES_EMPTY` | Object declares zero resources |
| `RES_DUP` | Duplicate Resource ID within an object |
| `INVALID_XML` | Document is not well-formed XML |

---

## Fault 413 — Execute/Type Matrix

Fault 413 enforces the Type/Operation relationship from Core TS 1.2.2 **Appendix D.1 (Object Template)**: "A Resource, which supports the 'Execute' operation, MUST have no associated Data Type (none) encoded as an empty value in Object DDF file" and "MUST have 'Single' as value of the 'Instances' field". The matrix is precise:

| Operation class | `<Type>` | `<Units>` | `<RangeEnumeration>` | `<MultipleInstances>` |
|-----------------|----------|-----------|----------------------|-----------------------|
| Execute (`E`) | MUST be absent → present is **413** | MUST be absent → present is **413** | MUST be absent → present is **413** | MUST be `Single` → `Multiple` is **414** |
| Value (`R` / `W` / `RW` / blank) | MUST be present → missing is **413**; present-but-unknown is **412** | optional | optional | `Single` or `Multiple` |

An **Execute** resource is a command trigger: it carries no value, so declaring a Type, Units, or RangeEnumeration is meaningless and rejected, and it must be a single instance (the DevCapMgmt Object 15 v1.0 DDF declared `opDisable` as Multiple and was corrected to Single in v2.0). A **value** resource (R/W/RW, or blank-Operations Bootstrap-only/Send-only) must always declare a Type — a missing Type is 413, an unrecognised Type is 412.

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

> Do not invent unit symbols. If a needed unit is not in the registry, use `TBD` and flag it, or raise the unit upstream. The grouped list above is a convenience extract; the registry repository ships its own machine-readable copy (`LWM2M_senml_units.xml`) and the IANA SenML Units registry is the live authority — confirm any symbol not listed above against one of those before submission.

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

Some sync tools, when persisting an OMNA resource whose source DDF leaves `<Operations>` or `<Type>` empty, substitute defaults:

| Missing field | Substituted default | Why it is lossy |
|---------------|---------------------|-----------------|
| `<Operations>` empty | `R` | An empty `<Operations>` is a **normative statement** that the resource has no DM&SE operation — all of Objects 0, 21 and 23, and Event Log 4016 (Send-only). Rewriting it to `R` would advertise Read access to PSK secrets and OSCORE master secrets |
| `<Type>` empty | `string` | An empty `<Type>` is the `none` type of an Execute resource (Core TS 1.2.2 Appendix D.1), not a String |

Consequently a synced resource may display `R` / `string` even when the source DDF omitted those elements. This is **tool-applied normalisation, not present in the source** — never treat a synced `R`/`string` as evidence the upstream DDF declared them, and prefer a sync that preserves blank as blank (`operations = none`, `type = none`). Record provenance so any substitution is auditable.

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
| `200` | Use a Company Reserved ID (`32769–42768`) or Test ID (`42769–42800`) for a private object; anything lower must be allocated by OMNA (or hold an OMNA-range override) |
| `220` | Add a non-empty `<Name>` |
| `230` | Add a non-empty `<Description1>` (note the trailing `1`) |
| `401` | Use `urn:oma:lwm2m:x:<id>` for vendor objects and make the URN ID equal `<ObjectID>` |
| `410` | Add a non-empty resource `<Name>` |
| `411` | Set `<Operations>` to exactly `R`, `W`, `RW`, or `E` — or leave it blank for a Bootstrap-only / Send-only resource |
| `412` | Use one of the 10 canonical Types (Appendix C types, e.g. `Unsigned Integer` → `unsigned_integer`; empty = `none` for Execute) |
| `413` | Remove Type/Units/Range from an Execute resource **or** add a Type to a value resource |
| `414` | Make the Execute resource `Single`-instance |
| `417` | Use an exact SenML symbol — temperature is `Cel` (not `C`/`°C`); or use `TBD` |
| `418` | Use `..` (e.g. `0..100`), not `0-100` or `0 to 100` |
| `800` | Add `<LWM2MVersion>` (e.g. `1.1`) if the platform demands it; the OMA schema allows omission when the minimum Enabler version is 1.0 (Appendix J) |
| `RES_EMPTY` | Declare at least one resource |
| `RES_DUP` | Make every Resource `ID` unique within the object |
| `INVALID_XML` | Fix well-formedness; re-validate before submission |

---

## Key Specification References

| Reference | Section / Scope | Used for |
|-----------|-----------------|----------|
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.2.1 General Policy, §7.2.2 Object Version format | Type I / Type II evolution rules, URN `urn:oma:lwm2m:{oma,ext,x}:ObjectID[:version]`, 1.0 version omission |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.2.3 | When `ver` must be sent in Register/Update/Discover |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §7.4 (Table 7.4-1), §7.4.2 | 16-bit identifiers, MAX_ID 65535 reserved; Reusable Resource frozen characteristics |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | Appendix C | Canonical data-type table (9 value types) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | Appendix D.1 | Object/Resource template; Execute ⇒ Type `none`, Instances Single; RangeEnumeration `..` syntax (fault 413/414/418 rules) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | Appendix D.2.1 (Table D.2.1-1), D.2.2 | Object ID classes and ranges; Resource registry |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | Appendix J | `LWM2MVersion` / `ObjectVersion` optional, default 1.0; DDF file naming by URN |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | Appendix K | LwM2M XML schema (`LWM2M.xsd`, `LWM2M-v1_1.xsd`) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.2.1 Register Operation | Object list / `ver` attribute in the Register payload; Objects 0, 21, 23 excluded |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.4.2 Notify Operation, §6.4.6 Send Operation | Observe/Notify semantics cited by resource descriptions ("MAY be reported by Observe"); Send-only resources such as Event Log 4016 |
| `RFC 8428` | SenML | Primary SenML unit symbol registry (fault 417) |
| `RFC 8798` | Additional SenML units | Secondary unit symbols (fault 417) |
| IANA SenML Units registry | live registry | Authoritative current unit list |
| `RFC 7252` | CoAP | Underlying transport for registry-facing operations |
| South East Water | "SEW Digital Utility IoT LwM2M Technical Specification" | ID band 10262–10284 (water metering) |
| `OpenMobileAlliance/lwm2m-registry` | `prod` branch, `DDF.xml` | Live registry publication format |

> Section and appendix numbers above were checked against the text of OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A. Older Core releases number these differently (e.g. the data-type appendix and the OMNA guidelines moved between 1.0 and 1.1), so re-check before quoting against a different release.

---

## Related Files

- [objects.md](objects.md) — Object model, core/IPSO/industry objects, ID ranges, reusable resources, OMNA registry process overview (this file is the authoring/validation deep-dive of that process).
- [protocol-details.md](protocol-details.md) — CoAP transport, registration/bootstrap flows, content-format negotiation underpinning registry-facing operations.
- [operation-coap-mapping.md](operation-coap-mapping.md) — LwM2M interface-to-CoAP operation mapping used when exercising registered objects.
