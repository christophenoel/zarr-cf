# CF Convention

- **UUID**: 0c0a02d2-8a95-4303-8b62-16a50b439d74
- **Name**: "cf"
- **Namespace**: `cf:`
- **Schema URL**: "<https://raw.githubusercontent.com/zarr-conventions/cf/refs/tags/v1/schema.json>"
- **Spec URL**: "<https://github.com/zarr-conventions/cf/blob/v1/README.md>"
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

All properties use the `cf:` namespace prefix and are placed at the root
`attributes` level following the
[Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

### Where the metadata lives

The simple, default approach is a **`cf:variables` catalogue on the parent
group**: one map, keyed by variable / coordinate name, describing every data
variable and coordinate of the group's child arrays in one place. This keeps CF
metadata centralized and is the recommended starting point.

A second mechanism lets you instead declare CF metadata **directly on an
individual array** — a data-variable array or a coordinate array — via
`cf:attributes`, mirroring how NetCDF stores a variable's attributes on the
variable itself. Use it when you prefer self-describing arrays or want the
metadata to travel with a specific array.

- **`cf:variables`** *(primary)* — a **map** from a variable / coordinate name
  to a CF descriptor, typically on the **parent group** (also valid on an
  array).
- **`cf:attributes`** *(per-array mechanism)* — the CF descriptor for **this
  array's own variable**, placed directly on the array's metadata.

A node MUST carry at least one of the two. They can coexist across a hierarchy
(a group catalogue plus a self-describing array), and an array MAY carry both
its own `cf:attributes` and a `cf:variables` map (e.g. for its non-array
coordinates).

### Providing metadata for each `coords` coordinate type

By default the parent group's `cf:variables` catalogue describes **every**
coordinate, regardless of its [`coords`](https://github.com/zarr-conventions/coords)
descriptor type. The per-array `cf:attributes` mechanism is additionally
available **only where the coordinate is backed by a real array**:

| `coords` descriptor | Backed by an array? | Default (group catalogue) | Per-array option |
|---|---|---|---|
| `type: "array"` (incl. `indexed_by` auxiliary) | Yes — sibling array at `path` | `cf:variables[name]` | **`cf:attributes`** on that coordinate array |
| `type: "inline"` | No — values in the descriptor | `cf:variables[name]` | — (no array to host it) |
| `type: "interval"` | No — values implied by start/end/step | `cf:variables[name]` | — |
| `type: "reference"` (e.g. `spatial`) | No — derived from a transform | `cf:variables[name]` | — |
| the **data variable** itself | Yes — it *is* a Zarr array | `cf:variables[name]` | **`cf:attributes`** on its own array |

### Relationship to NetCDF-CF

In NetCDF-CF **every coordinate is materialized as a variable (an array)**, and
attributes always sit on that variable. There is no inline, no start/step, and
no affine-transform coordinate in the classic CF model. That has a direct
consequence for alignment:

- **`type: "array"` coordinates are the only ones that map 1:1 to NetCDF-CF.**
  There, `cf:attributes` on the coordinate array *is* the NetCDF placement —
  full round-trip fidelity. Auxiliary coordinates (`indexed_by`) match CF's
  auxiliary coordinate variables the same way.
- **`inline`, `interval`, and `reference` are Zarr-native** representations with
  **no NetCDF-CF counterpart**. `inline` / `interval` trade a materialized
  coordinate variable for compactness; `reference` (e.g. to `spatial`) uses the
  GeoTIFF/GDAL affine model, whereas CF would instead store explicit
  `projection_x_coordinate` / `projection_y_coordinate` variables plus a scalar
  `grid_mapping` container. To export any of these to NetCDF-CF a writer must
  **expand them back into coordinate variable arrays**.
- Because those forms have no variable array to host attributes, their CF
  metadata necessarily lives in a `cf:` structure on the declaring node — the
  `cf:variables` map here. Keying that map by coordinate name mirrors how a CF
  data variable references its coordinates by name in its `coordinates`
  attribute, so it stays CF-ish in spirit even though it is not a 1:1 placement.

**Guidance:** for maximum NetCDF-CF alignment / round-trip, use `type: "array"`
coordinates and put attributes on the arrays via `cf:attributes`. Reach for the
compact `inline` / `interval` / `reference` forms (and the `cf:variables` map)
when Zarr-native compactness or the affine model matters more than CF
round-trip — a deliberate, documented divergence rather than an accidental one.

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
  A `cf:variables` key typically matches a key in the same node's
  `coords:coordinates` map. The two ride together on one node.
- **[`proj`](https://github.com/zarr-conventions/proj)** /
  **[`spatial`](https://github.com/zarr-conventions/spatial)** — supply the CRS
  and the affine transform for spatial axes; `cf` adds their descriptive
  attributes. Orthogonal, applied on the same node.
- **[`multiscales`](https://github.com/zarr-conventions/multiscales)** — each
  level array can independently carry its own `cf:variables`.

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

On a **group**, `cf:variables` is the primary placement — a catalogue
describing the data variables and coordinates of the group's child arrays in one
place. On an **array**, `cf:attributes` describes the array's **own variable**
(a data variable, or an explicit coordinate array); an array may also carry a
`cf:variables` map for its non-array coordinates.

## Properties

All properties use the `cf:` namespace prefix and are placed at the root
`attributes` level. A node MUST carry **at least one** of `cf:attributes` or
`cf:variables`.

| Field Name      | Type      | Required        | Description |
|-----------------|-----------|-----------------|-------------|
| `cf:variables`  | `object`  | Conditional\*   | **(Primary)** Map from a variable / coordinate name to a [CF attribute descriptor](#cf-attribute-descriptor), typically on the parent group. |
| `cf:attributes` | `object`  | Conditional\*   | **(Per-array mechanism)** The [CF attribute descriptor](#cf-attribute-descriptor) for this array's own variable, placed directly on its Zarr array metadata (NetCDF-aligned). |
| `cf:version`    | `integer` | Optional        | Major version pin (currently `1`). Optional because `schema_url` already pins the major. |

\* At least one of `cf:variables` or `cf:attributes` MUST be provided.

### Additional Properties

Additional properties are allowed.

### `cf:variables`

**The primary placement.** Map from a coordinate or variable **name** to a **CF
attribute descriptor**, typically declared once on the **parent group** as a
catalogue for its child arrays.

- **Type**: object (map)
- **Keys**: a coordinate key (matching a key in a `coords:coordinates` map) or a
  data-variable name.
- **Values**: a [CF attribute descriptor](#cf-attribute-descriptor).
- **Where**: usually the parent group; also valid on an array (e.g. to describe
  its non-array coordinates).

### `cf:attributes`

**The per-array mechanism.** The [CF attribute descriptor](#cf-attribute-descriptor)
for **this array's own variable**, placed directly on the Zarr array's
`attributes` — the same place NetCDF stores a variable's attributes. Use it on a
data-variable array, or on an explicit coordinate array (the target of a
`coords` `type: "array"` descriptor), when you prefer self-describing arrays.

#### CF attribute descriptor

Every field is **optional** — different variables use different subsets — and
descriptors are `additionalProperties: true`, so any CF attribute not listed
here passes through unchanged.

| Field | Type | Description |
|-------|------|-------------|
| `standard_name` | `string` | Name from the CF standard name table identifying the physical quantity. |
| `long_name` | `string` | Human-readable descriptive name. |
| `units` | `string` | UDUNITS units. For time, the `<unit> since <epoch>` form (e.g. `days since 1850-01-01`). |
| `comment` | `string` | Miscellaneous information about the variable. |
| `references` | `string` | References describing the variable or its production. |
| `source` | `string` | Method of production of the original data. |
| `axis` | `string` | Axis role for a coordinate: `X`, `Y`, `Z`, or `T`. |
| `positive` | `string` | Direction of increasing values for a vertical coordinate: `up` or `down`. |
| `calendar` | `string` | Calendar for a time coordinate (`standard`, `proleptic_gregorian`, `noleap`, `360_day`, …). |
| `flag_values` | `array` | Values corresponding to `flag_meanings`. |
| `flag_masks` | `array` | Bit masks corresponding to `flag_meanings`. |
| `flag_meanings` | `string` | Space-separated flag names matching `flag_values` / `flag_masks`. |
| `valid_min` | `number` | Smallest valid value. |
| `valid_max` | `number` | Largest valid value. |
| `valid_range` | `number[2]` | `[min, max]` valid range. |
| `ancillary_variables` | `string` | Space-separated names of ancillary variables. |

**Sketch A — primary: a catalogue on the parent group.** One map describes the
group's child variables and coordinates, whatever their `coords` type:

```jsonc
// group (…/dataset):
"cf:variables": {
  "time": { "standard_name": "time", "units": "days since 1850-01-01", "calendar": "noleap", "axis": "T" },
  "y":    { "standard_name": "projection_y_coordinate", "units": "m", "axis": "Y" },
  "x":    { "standard_name": "projection_x_coordinate", "units": "m", "axis": "X" },
  "sst":  { "standard_name": "sea_water_temperature", "units": "K" }
}
```

**Sketch B — per-array mechanism (NetCDF-aligned).** Where a variable is a real
array, it can instead describe **itself** on its own metadata:

```jsonc
// coordinate array (…/time):
"cf:attributes": {
  "standard_name": "time", "units": "days since 1850-01-01",
  "calendar": "noleap", "axis": "T"
}
```

### `cf:version`

Optional integer pinning the major version of the convention this metadata was
authored against. Currently `1`. Omitting it is fine — `schema_url` already
pins the major.

## Examples

See the [examples](examples/) directory for complete Zarr convention metadata
examples:

- [examples/cf.json](examples/cf.json) — **primary approach**: a group-level
  `cf:variables` catalogue describing a `time` coordinate (calendar), spatial
  `y` / `x` axes, a `sea_water_temperature` data variable (valid range +
  ancillary variable), and its quality-flag variable.
- [examples/cf-coordinate-array.json](examples/cf-coordinate-array.json) —
  **per-array mechanism**: a self-describing `time` coordinate array carrying
  its own `cf:attributes` (units + calendar + axis), the NetCDF-aligned
  placement that a `coords` `type: "array"` descriptor points at.

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
