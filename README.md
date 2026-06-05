# CF Convention

- **UUID**: 0c0a02d2-8a95-4303-8b62-16a50b439d74
- **Name**: "cf"
- **Namespace**: `cf:`
- **Schema URL**: "<https://github.com/christophenoel/zarr-cf/blob/master/schema.json>"
- **Spec URL**: "<https://github.com/christophenoel/zarr-cf/blob/master/README.md"
- **Extension Maturity Classification**: Proposal
- **Owner**: @christophenoel

## Description

CF semantic metadata — standard names, units, axis, calendar, and data-quality
attributes — for Zarr data variables and coordinates.

This convention attaches **meaning** to any data variable or coordinate. It is
the semantic layer that the [`coords`](https://github.com/zarr-conventions/coords)
convention explicitly defers to: where `coords` says *where* a coordinate's
values live, `cf` says *what it is* (its standard name, units, axis role,
calendar, valid range, flags, …). It is a Zarr-native, namespaced expression of
the descriptive attributes defined by the
[CF (Climate and Forecast) Metadata Conventions](https://cfconventions.org/),
sections 3 and 4.1–4.4.

`cf` is **one of a planned set of focused CF-related Zarr conventions** and
covers only the *semantic attributes* part. Other CF concerns are intended for
sibling conventions (e.g. cell bounds & methods, parametric vertical
coordinates); see [`CF-to-zarr.md`](../CF-to-zarr.md) for the full split.

All properties use the `cf:` namespace prefix and are placed at the root
`attributes` level following the
[Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

### Where the metadata lives

The CF metadata is always the **same object** — a
[CF attribute descriptor](#cf-attribute-descriptor) (`standard_name`, `units`,
`axis`, …). What changes is *where* that descriptor is attached. There are three
placements, integrated with the
[`coords`](https://github.com/zarr-conventions/coords) convention; a node MUST
carry at least one of them:

| # | Property | Lives on | Describes | CF source |
|---|----------|----------|-----------|-----------|
| 1 | `cf:attributes` | an **array** | that array's own variable | NetCDF — attrs on the variable |
| 2 | `cf:attributes` embedded in `coords:coordinates[name]` | the node declaring `coords` | one **coordinate** | NetCDF — attrs on the coordinate variable |
| 3 | `cf:variables` | a **group** | the group's child **data variables** | Zarr-native convenience (no CF equivalent) |

Placements 1 and 2 are both the `cf:attributes` descriptor — applied to an
array, or embedded in a coordinate's `coords:coordinates` entry; both are
specified under [`cf:attributes`](#cfattributes) below, including how `cf`
extends a coordinate and how readers resolve a `type: "array"` coordinate.
Placement 3 is specified under [`cf:variables`](#cfvariables). Worked snippets of
all three are in [Examples](#examples).

```jsonc
// group (…/dataset)
"cf:variables": {
  "sst": { "standard_name": "sea_water_temperature", "units": "K" },
  "sss": { "standard_name": "sea_water_salinity", "units": "1e-3" }
}
```

## Motivation

- Provides standardized cf metadata for Zarr datasets — a uniform way to carry
  CF descriptive vocabulary (`standard_name`, `units`, `axis`, `calendar`,
  flags) without re-implementing coordinate *location* or CRS.
- Composable with other Zarr conventions (e.g.,
  [`coords`](https://github.com/zarr-conventions/coords),
  [`spatial`](https://github.com/zarr-conventions/spatial),
  [`proj`](https://github.com/zarr-conventions/proj),
  [`multiscales`](https://github.com/zarr-conventions/multiscales)).
- Uses **integer-major + URL pin** versioning: the `schema_url` carries the
  major (`/refs/tags/v1/schema.json`); all v1.x changes are additive.

### Composes with

- **[`coords`](https://github.com/zarr-conventions/coords)** — `coords` locates
  a coordinate (which array / inline values / interval); `cf` gives it meaning.
  A coordinate's `cf:attributes` is embedded **inside** its `coords:coordinates`
  descriptor, so location and meaning are declared together.
- **[`proj`](https://github.com/zarr-conventions/proj)** /
  **[`spatial`](https://github.com/zarr-conventions/spatial)** — supply the CRS
  and the affine transform for spatial axes; `cf` adds their descriptive
  attributes. Orthogonal, applied on the same node.
- **[`multiscales`](https://github.com/zarr-conventions/multiscales)** — each
  level array can independently carry its own `cf:attributes`.

### Out of scope (delegated)

`cf` is deliberately limited to *semantics*. It does **not** locate coordinate
values (→ `coords`), define a CRS (→ `proj`), express an affine transform
(→ `spatial`), describe cell extent / bounds / methods (→ a future `cf-cells`
convention), formulate parametric vertical axes (→ a future `cf-vertical`
convention), or model feature collections / Discrete Sampling Geometries
(→ an extension of `coords`). See
[`CF-to-zarr.md`](https://github.com/zarr-conventions) for the full split.

## Convention Registration

The convention must be registered in `zarr_conventions`:

```json
{
  "zarr_conventions": [
    {
      "schema_url": "https://raw.githubusercontent.com/zarr-conventions/cf/refs/tags/v1/schema.json",
      "spec_url": "https://github.com/zarr-conventions/cf/blob/v1/README.md",
      "uuid": "0c0a02d2-8a95-4303-8b62-16a50b439d74",
      "name": "cf",
      "description": "CF semantic metadata — standard names, units, axis, calendar, and data-quality attributes — for Zarr data variables and coordinates."
    }
  ]
}
```

## Applicable To

This convention can be used with these parts of the Zarr hierarchy:

- [x] Group
- [x] Array

On an **array**, `cf:attributes` describes the array's **own variable** (a data
variable, or a `type: "array"` coordinate array). On a **group**, `cf:variables`
is an optional catalogue describing the group's child **data-variable** arrays.
Coordinate metadata, on either node type, is embedded inside the relevant
`coords:coordinates` descriptor.

## Properties

All properties use the `cf:` namespace prefix and are placed at the root
`attributes` level. A node MUST carry **at least one** of `cf:attributes`,
`coords:coordinates` (with embedded `cf:attributes`), or `cf:variables`.

| Field Name      | Type      | Required        | Description |
|-----------------|-----------|-----------------|-------------|
| `cf:attributes` | `object`  | Conditional\*   | The [CF attribute descriptor](#cf-attribute-descriptor) for **this array's own variable**, on a data-variable array or a `type: "array"` coordinate array. |
| `cf:variables`  | `object`  | Conditional\*   | **(Group only)** Optional map from a **data-variable** name to a [CF attribute descriptor](#cf-attribute-descriptor) — a convenience catalogue for a group's child arrays. |
| `cf:version`    | `integer` | Optional        | Major version pin (currently `1`). Optional because `schema_url` already pins the major. |

Coordinate descriptors additionally carry an embedded `cf:attributes` inside
each [`coords:coordinates`](https://github.com/zarr-conventions/coords) entry —
see [Where the metadata lives](#where-the-metadata-lives).

\* A node MUST provide at least one of `cf:attributes`, an embedded
`cf:attributes` in `coords:coordinates`, or `cf:variables`.

### Additional Properties

Additional properties are allowed.

### `cf:attributes`

A single [CF attribute descriptor](#cf-attribute-descriptor) describing **one
variable**. The same object is used in two placements.

**On an array (placement 1).** A Zarr array *is* a single variable, so a
top-level `cf:attributes` describes **that array's variable** — the same place
NetCDF keeps a variable's attributes. Use it on a **data-variable array**, or on
an explicit **coordinate array** (the sibling array a `coords` `type: "array"`
descriptor points at). A reader takes `attributes["cf:attributes"]` as the CF
metadata of the variable the array represents.

**Embedded in a coordinate (placement 2).** This is the primary way to describe
**coordinates**. `coords:coordinates` maps each coordinate name to a descriptor
that says only *where* its values are; because those descriptors are
`additionalProperties: true`, `cf` extends each one **in place** by adding a
single `cf:attributes` key:

```text
coords:coordinates[name] = { …coords location fields… , "cf:attributes": { …CF meaning… } }
                              └──────── owned by coords ───────┘   └──── owned by cf ────┘
```

One object then declares both halves of a coordinate — `coords` *where it is*,
the embedded `cf:attributes` *what it means*. It works for **every** coords
descriptor type, and for `inline`, `interval`, and `reference` coordinates
(which have no array of their own) it is the **only** place their CF metadata can
live. Validation splits cleanly: `coords` validates the location fields, `cf`
validates the embedded `cf:attributes`.

**Resolution for a `type: "array"` coordinate.** Such a coordinate MAY instead
carry its descriptor on the target coordinate array itself (placement 1); the
`coords:coordinates` entry then simply **omits** `cf:attributes`. A reader
resolves the coordinate's CF metadata as:

1. the `cf:attributes` **embedded** in its `coords:coordinates` descriptor, if
   present; otherwise
2. the top-level `cf:attributes` on the **array at `path`**.

Writers SHOULD use **one** location, not both; if both appear they SHOULD be
consistent and the embedded descriptor — declared with the coordinate's actual
use — is authoritative.

### `cf:variables`

**Group only; optional** (placement 3). A map from a **data-variable** name to a
[CF attribute descriptor](#cf-attribute-descriptor), declared once on a **group**
as a convenience catalogue for its child data-variable arrays (handy with
consolidated metadata).

- **Type**: object (map)
- **Keys**: a data-variable name.
- **Values**: a [CF attribute descriptor](#cf-attribute-descriptor).
- **Scope**: a Zarr-native convenience, **not** a NetCDF-CF construct. Not
  meaningful on an array (an array is one variable — use `cf:attributes`), and
  not for coordinates (use the embedded form under `cf:attributes`).
- **Overlap**: it overlaps placement 1 (both describe a data variable); the two
  SHOULD be consistent, and a variable's own array `cf:attributes` is
  authoritative if both are present.

#### CF attribute descriptor

Every field is **optional** — different variables use different subsets — and
descriptors are `additionalProperties: true`, so any CF attribute not listed
here passes through unchanged. The **Applies to** column notes whether a field
is meaningful on a coordinate, a data variable, or both; the schema does not
enforce it (a field that does not apply is simply omitted).

| Field | Type | Applies to | Description |
|-------|------|------------|-------------|
| `standard_name` | `string` | both | Name from the CF standard name table identifying the physical quantity. |
| `long_name` | `string` | both | Human-readable descriptive name. |
| `units` | `string` | both | UDUNITS units. For time, the `<unit> since <epoch>` form (e.g. `days since 1850-01-01`). |
| `comment` | `string` | both | Miscellaneous information about the variable. |
| `references` | `string` | both | References describing the variable or its production. |
| `source` | `string` | data variable | Method of production of the original data. |
| `axis` | `string` | coordinate | Axis role for a coordinate: `X`, `Y`, `Z`, or `T`. |
| `positive` | `string` | coordinate | Direction of increasing values for a vertical coordinate: `up` or `down`. |
| `calendar` | `string` | coordinate | Calendar for a time coordinate (`standard`, `proleptic_gregorian`, `noleap`, `360_day`, …). |
| `flag_values` | `array` | data variable | Values corresponding to `flag_meanings`. |
| `flag_masks` | `array` | data variable | Bit masks corresponding to `flag_meanings`. |
| `flag_meanings` | `string` | data variable | Space-separated flag names matching `flag_values` / `flag_masks`. |
| `valid_min` | `number` | both | Smallest valid value. |
| `valid_max` | `number` | both | Largest valid value. |
| `valid_range` | `number[2]` | both | `[min, max]` valid range. |
| `ancillary_variables` | `string` | data variable | Space-separated names of ancillary variables. |

See [Examples](#examples) for a worked snippet of each of the three placements.

### `cf:version`

Optional integer pinning the major version of the convention this metadata was
authored against. Currently `1`. Omitting it is fine — `schema_url` already
pins the major.

## Examples

Three end-to-end usage patterns follow, each as a minimal snippet; see the
[examples](examples/) directory for the complete, validated metadata documents.

### Approach 1 — self-describing coordinate array

A coordinate array carrying its own `cf:attributes` (placement 1 applied to a
coordinate array) — the array a `coords` `type: "array"` descriptor points at,
when its `coords:coordinates` entry omits the embedded form. Full file:
[examples/cf-coordinate-array.json](examples/cf-coordinate-array.json).

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time"],
  "attributes": {
    "zarr_conventions": [
      { "name": "cf", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/cf/refs/tags/v1/schema.json" }
    ],
    "cf:attributes": { "standard_name": "time", "units": "days since 1850-01-01", "calendar": "noleap", "axis": "T" }
  }
}
```

### Approach 2 — group catalogue of data variables

An optional `cf:variables` map on a group (placement 3). Full file:
[examples/cf-group-catalogue.json](examples/cf-group-catalogue.json).

```json
{
  "zarr_format": 3,
  "node_type": "group",
  "attributes": {
    "zarr_conventions": [
      { "name": "cf", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/cf/refs/tags/v1/schema.json" }
    ],
    "cf:variables": {
      "sst": { "standard_name": "sea_water_temperature", "units": "K" },
      "sss": { "standard_name": "sea_water_salinity", "units": "1e-3" }
    }
  }
}
```

### Approach 3 — data variable with coordinates

The comprehensive case (uses placements 1 and 2): a data-variable array
describes itself via a top-level `cf:attributes`, and each of its coordinates
carries an embedded `cf:attributes` inside its `coords:coordinates` descriptor.
Full file: [examples/cf.json](examples/cf.json).

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time", "x"],
  "attributes": {
    "zarr_conventions": [
      { "name": "cf",     "schema_url": "https://raw.githubusercontent.com/zarr-conventions/cf/refs/tags/v1/schema.json" },
      { "name": "coords", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json" }
    ],
    "coords:coordinates": {
      "time": {
        "type": "array", "path": "../time",
        "cf:attributes": { "standard_name": "time", "units": "days since 1850-01-01", "calendar": "noleap", "axis": "T" }
      },
      "x": {
        "type": "reference", "convention": "spatial",
        "cf:attributes": { "standard_name": "projection_x_coordinate", "units": "m", "axis": "X" }
      }
    },
    "cf:attributes": { "standard_name": "sea_water_temperature", "units": "K" }
  }
}
```

## Versioning and Compatibility

This convention follows the **integer-major with URL pin** contract (contract
#4 in the
[Zarr Conventions Guidance Implementation Contracts](https://zarr-conventions.github.io/zarr-conventions-guidance/implementation-contracts)):

- The `schema_url` and `spec_url` carry the integer major version
  (`/refs/tags/v1/...`, `/blob/v1/...`).
- All v1.x changes are additive: new optional fields and broadened ranges only.
- Breaking changes (renaming, retyping, removing, or semantically shifting
  existing fields) require a new major: tag `v2` and publish a fresh schema
  under `/refs/tags/v2/schema.json`.
- Readers SHOULD tolerate unknown additional fields per the conventions
  framework's safely-ignorable principle.

## Acknowledgements

The template is based on the [STAC extensions template](https://github.com/stac-extensions/template/blob/main/README.md).

The attribute vocabulary is drawn from the
[CF (Climate and Forecast) Metadata Conventions](https://cfconventions.org/).
This convention carries the CF *semantics* layer that the
[`coords`](https://github.com/zarr-conventions/coords) convention defers to.
