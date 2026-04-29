# CapCAD JSON Format — Geometric Entities Reference

## Overview

The CapCAD JSON format represents geometric entities as structured JSON objects. Each entity follows a consistent pattern of a **container object** with a `Properties` block (holding a UUID) and one or more typed child arrays that carry the actual geometric data.

Entities described in this document:

- [`Point`](#point) — a single coordinate in 2D space
- [`Points`](#points) — an ordered collection of 2D coordinates
- [`CurveElement`](#curveelement) — a single polyline/arc segment sequence
- [`Curve`](#curve) — one or more CurveElement segments grouped as one geometric object
- [`Space`](#space) — a heterogeneous container that holds any mix of the above entities, including nested Spaces

---

## Common Conventions

### UUID Identifiers

Every entity and every sub-object within it carries a globally unique identifier in its `Properties` block:

```json
"Properties": {
  "Id": "f7439f43-2515-4cee-b4d3-44f45affe11d"
}
```

UUIDs follow the standard RFC 4122 format (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). There are two levels of identity:

| Level | Description |
|---|---|
| **Container Id** | Identifies the entity as a whole (e.g., the `Point`, `Curve`, or `Geometry` wrapper) |
| **Sub-object Id** | Identifies each individual coordinate node or segment within the entity |

### Format Version

Every top-level entity or Space carries a version field in its `Properties` block to allow readers to detect and handle format evolution:

```json
"Properties": {
  "Id": "f7439f43-2515-4cee-b4d3-44f45affe11d",
  "CapJson.Version": "1.1"
}
```

Parsers that encounter an unrecognised version should warn rather than fail silently.

### Coordinate System

The coordinate style is declared on each curve segment or curve-element vertex array using the `CoordinateStyle` field. Points do not carry a `CoordinateStyle` — their coordinates are always `[x, y]`.

| Value | Coordinates format | Description |
|---|---|---|
| `"xyb"` | `[x, y, bulge]` | 2D Cartesian with bulge. Used by `CurveElement` and `Curve` segments. |
| `"xyzb"` | `[x, y, z, bulge]` | 3D Cartesian with bulge. Reserved for future use. |

Coordinate values are expressed as 64-bit floating-point numbers.

### Units

The coordinate unit is declared at the `Space` level in its `Properties` block. Individual entities do not repeat it.

```json
"Properties": {
  "CapJson.Units": "m"
}
```

| Value | Description |
|---|---|
| `"m"` | Metres |
| `"ft"` | Feet |
| `"mm"` | Millimetres |

If omitted, units are considered unspecified. Parsers should not assume a default.

### Properties and xdata

Every entity and every sub-object accepts an open-ended `Properties` dictionary. Any key-value pair may be stored here and will survive the JSON round-trip. When entities are imported back into AutoCAD, arbitrary properties are persisted as **xdata** (extended entity data) on the corresponding AutoCAD object.

This mechanism allows domain-specific metadata — station values, design speeds, material codes, project references — to travel with the geometry without requiring format changes.

```json
"Properties": {
  "Id": "...",
  "CapJson.Version": "1.1",
  "Project.Name": "BR-040 Lot 3",
  "Alignment.Station": 1234.56,
  "Design.Speed": 80
}
```

Keys beginning with `CapJson.`, `Acad.`, or `Curve.` are reserved for format use. All other keys are available for application use.

### Bounding Box

Any entity or Space may carry a bounding box in its `Properties` block. This allows quick spatial filtering without parsing all coordinates.

```json
"Properties": {
  "CapJson.BBox": [-76405.04, 14563.86, -76351.49, 14585.24]
}
```

The value is a 4-element array: `[xMin, yMin, xMax, yMax]`. For 3D geometry: `[xMin, yMin, zMin, xMax, yMax, zMax]`.

### Bulge — Arc Encoding

Curve vertex coordinates carry a third value, the **bulge**, which encodes arc geometry between consecutive vertices. This convention is identical to AutoCAD's bulge representation.

```
[x, y, bulge]
```

The bulge is defined as:

```
bulge = tan(θ / 4)
```

where **θ** is the included (central) angle of the arc from the current vertex to the next vertex.

| Bulge value | Meaning |
|---|---|
| `0` | Straight line segment to the next vertex |
| `> 0` | Counter-clockwise arc to the next vertex |
| `< 0` | Clockwise arc to the next vertex |
| `1` | Semicircle (180° arc, CCW) |
| `-1` | Semicircle (180° arc, CW) |

The bulge value belongs to the **starting vertex** of each segment. The last vertex in a sequence has no segment after it; any bulge value on the last vertex is ignored.

**Example** — the two arc transitions in the sample curve:

```json
[-76403.638292135, 14583.0286973778, -0.414213562373121]  // CW arc to next vertex
[-76400.3595156526, 14583.3114446175, 0.414213562373934]   // CCW arc to next vertex
```

The value `±0.41421356...` corresponds to `tan(45° / 4)`, i.e., a 45° arc (one-eighth of a full circle).

---

## Point

A `Point` represents a single named location in 2D space.

### Structure

```json
{
  "Properties": {
    "Id": "<container-uuid>",
    "CapJson.Version": "1.1"
  },
  "Point": [
    {
      "Properties": {
        "Id": "<node-uuid>"
      },
      "Coordinates": [x, y]
    }
  ]
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `Properties.Id` | UUID string | Unique identifier for the Point entity. |
| `Properties.CapJson.Version` | string | Format version. |
| `Point` | array | Always contains exactly **one** point node object. |
| `Point[0].Properties.Id` | UUID string | Unique identifier for the coordinate node. |
| `Point[0].Coordinates` | `[number, number]` | The `[x, y]` position of the point. |

### Example

```json
{
  "Properties": {
    "Id": "f7439f43-2515-4cee-b4d3-44f45affe11d",
    "CapJson.Version": "1.1"
  },
  "Point": [
    {
      "Properties": {
        "Id": "53e0c8e6-08ed-435f-a01a-b93e0ed37f1c"
      },
      "Coordinates": [
        -76373.7428091232,
        14590.0438864508
      ]
    }
  ]
}
```

---

## Points

A `Points` entity is an **ordered collection** of 2D coordinate nodes under a single named container. It shares the same structure as `Point` but its `Point` array holds two or more nodes.

### Structure

```json
{
  "Properties": {
    "Id": "<container-uuid>",
    "CapJson.Version": "1.1"
  },
  "Point": [
    {
      "Properties": { "Id": "<node-uuid>" },
      "Coordinates": [x, y]
    },
    ...
  ]
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `Properties.Id` | UUID string | Unique identifier for the Points collection. |
| `Properties.CapJson.Version` | string | Format version. |
| `Point` | array | Ordered list of two or more point node objects. |
| `Point[n].Properties.Id` | UUID string | Unique identifier for each individual coordinate node. |
| `Point[n].Coordinates` | `[number, number]` | The `[x, y]` position of the nth point. |

### Example

```json
{
  "Properties": {
    "Id": "75a8b34c-e02b-47d1-89e3-e3827cf67d8b",
    "CapJson.Version": "1.1"
  },
  "Point": [
    {
      "Properties": { "Id": "698e02e6-fe22-465a-89d0-6d2a28488c30" },
      "Coordinates": [-76395.8521405227, 14576.1963283183]
    },
    {
      "Properties": { "Id": "0a88089e-5efb-493c-81b6-5749756d41e7" },
      "Coordinates": [-76397.7837515573, 14575.7772162978]
    }
  ]
}
```

---

## CurveElement

A `CurveElement` is a single polyline/arc segment sequence. It is always wrapped in a `Geometry` container that declares the object type as `"Curve"`. A `CurveElement` entity contains exactly **one** `CurveElement` segment in its array.

For entities with multiple segments, see [`Curve`](#curve).

### Structure

```json
{
  "Geometry": [
    {
      "Object": "Curve",
      "Properties": {
        "Id": "<geometry-uuid>",
        "CapJson.Version": "1.1"
      },
      "CurveElement": [
        {
          "Properties": { ...segment properties... },
          "CoordinateStyle": "xyb",
          "Coordinates": [
            [x0, y0, bulge0],
            [x1, y1, bulge1],
            ...
            [xN, yN, bulgeN]
          ]
        }
      ]
    }
  ]
}
```

### Fields

#### Geometry wrapper

| Field | Type | Description |
|---|---|---|
| `Geometry` | array | Outer container. Always contains exactly one geometry object for a CurveElement. |
| `Geometry[0].Object` | string | Geometry type. For curve-based entities this is always `"Curve"`. |
| `Geometry[0].Properties.Id` | UUID string | Unique identifier for the geometry object. |
| `Geometry[0].Properties.CapJson.Version` | string | Format version. |

#### Segment object (`CurveElement[0]`)

| Field | Type | Description |
|---|---|---|
| `Properties.Id` | UUID string | Unique identifier for the segment. |
| `Properties.Curve.Factor` | number | Scaling factor applied to the curve. Default: `1`. |
| `Properties.Curve.Initial` | number | Initial parameter value along the curve. Default: `0`. |
| `Properties.Acad.LinetypeScale` | number | AutoCAD linetype scale. Default: `1`. |
| `Properties.Acad.Linetype` | string | AutoCAD linetype name (e.g., `"ByLayer"`). |
| `Properties.Acad.Lineweight` | number | AutoCAD lineweight. `-1` means ByLayer. |
| `Properties.Acad.Layer` | string | AutoCAD layer name this segment belongs to. |
| `Properties.Acad.OriginalEntObjectId` | string | Original AutoCAD entity object ID (hex string). |
| `Properties.Acad.IndexColor` | number | AutoCAD color index. `256` means ByLayer. |
| `Properties.Acad.RgbColor` | string | RGB display color as `"R,G,B"`. Used for display when IndexColor is ByLayer. |
| `Properties.Acad.ConstantWidth` | number | Polyline constant width. `0` means no width. |
| `CoordinateStyle` | string | Coordinate format. See [Coordinate System](#coordinate-system). |
| `Coordinates` | array of `[x, y, bulge]` | Ordered list of vertices. See [Bulge — Arc Encoding](#bulge--arc-encoding). |

### Example

```json
{
  "Geometry": [
    {
      "Object": "Curve",
      "Properties": {
        "Id": "1442bdb2-78de-45fe-86de-e8232e1f749d",
        "CapJson.Version": "1.1"
      },
      "CurveElement": [
        {
          "Properties": {
            "Curve.Factor": 1,
            "Curve.Initial": 0,
            "Acad.Layer": "0",
            "Acad.RgbColor": "255,255,255",
            "Id": "8faec39d-1f7b-4da7-87bd-02bcb81e1eff"
          },
          "CoordinateStyle": "xyb",
          "Coordinates": [
            [-76405.0419679545, 14580.8009189192, 0],
            [-76403.638292135,  14583.0286973778, -0.414213562373121],
            [-76400.8978043648, 14583.6506088851, 0],
            [-76400.3595156526, 14583.3114446175, 0.414213562373934],
            [-76397.6190278824, 14583.9333561248, 0],
            [-76396.7971482582, 14585.2377639495, 0],
            [-76391.9354817203, 14583.128737293,  0]
          ]
        }
      ]
    }
  ]
}
```

---

## Curve

A `Curve` shares the same outer structure as a [`CurveElement`](#curveelement) but its `CurveElement` array contains **two or more** segment objects. Each segment is an independent polyline/arc sequence with its own properties (layer, color, width, etc.).

This allows a single named geometry object to carry segments belonging to different AutoCAD layers or having different display properties.

### Structure

```json
{
  "Geometry": [
    {
      "Object": "Curve",
      "Properties": {
        "Id": "<geometry-uuid>",
        "CapJson.Version": "1.1"
      },
      "CurveElement": [
        {
          "Properties": { ...segment-1 properties... },
          "CoordinateStyle": "xyb",
          "Coordinates": [ ... ]
        },
        {
          "Properties": { ...segment-2 properties... },
          "CoordinateStyle": "xyb",
          "Coordinates": [ ... ]
        }
      ]
    }
  ]
}
```

### Difference from CurveElement

| | CurveElement | Curve |
|---|---|---|
| `CurveElement` array length | Exactly 1 | 2 or more |
| Segments may have different layers | No (single segment) | Yes |
| Segments may have different colors | No | Yes |
| Segments may have different widths | No | Yes |

---

## Space

A `Space` is a heterogeneous container whose `Space` array can hold any combination of:

- `Point` (inline or structured)
- `Points` (structured)
- `CurveElement` (inline)
- `Curve` (structured, via `Geometry` wrapper)
- Nested `Space` objects

The order of items in the array is not significant. Inline and structured forms may be mixed freely in the same `Space`.

### Structure

```json
{
  "Properties": {
    "Id": "<space-uuid>",
    "CapJson.Version": "1.1",
    "CapJson.Units": "m",
    "CapJson.BBox": [xMin, yMin, xMax, yMax]
  },
  "Space": [
    ...items...
  ]
}
```

### Space-level Properties

| Field | Type | Description |
|---|---|---|
| `Properties.Id` | UUID string | Unique identifier for this Space. Optional. |
| `Properties.CapJson.Version` | string | Format version. |
| `Properties.CapJson.Units` | string | Coordinate unit. See [Units](#units). |
| `Properties.CapJson.BBox` | `[xMin, yMin, xMax, yMax]` | Bounding box of all contents. Optional. |

Any additional key-value pairs in `Properties` are treated as application metadata and are round-tripped as xdata on import. See [Properties and xdata](#properties-and-xdata).

---

### Item forms

#### Inline CurveElement

```json
{
  "Object": "CurveElement",
  "Properties": {
    "Curve.Factor": 1,
    "Curve.Initial": 0,
    "Acad.Layer": "<layer-name>",
    "Acad.RgbColor": "R,G,B",
    "Id": "<segment-uuid>"
  },
  "CoordinateStyle": "xyb",
  "Coordinates": [
    [x0, y0, bulge0],
    [x1, y1, bulge1],
    ...
  ]
}
```

---

#### Structured Curve (via Geometry wrapper)

```json
{
  "Geometry": [
    {
      "Object": "Curve",
      "Properties": {},
      "CurveElement": [
        {
          "Properties": { ...segment properties... },
          "CoordinateStyle": "xyb",
          "Coordinates": [ ... ]
        },
        {
          "Properties": { ...segment properties... },
          "CoordinateStyle": "xyb",
          "Coordinates": [ ... ]
        }
      ]
    }
  ]
}
```

---

#### Inline Point

```json
{
  "Point": [
    {
      "Properties": { "Id": "<node-uuid>" },
      "Coordinates": [x, y]
    }
  ]
}
```

---

#### Structured Points

```json
{
  "Properties": { "Id": "<container-uuid>" },
  "Point": [
    {
      "Properties": { "Id": "<node-uuid>" },
      "Coordinates": [x, y]
    },
    {
      "Properties": { "Id": "<node-uuid>" },
      "Coordinates": [x, y]
    }
  ]
}
```

---

#### Nested Space

A `Space` item is itself a full `Space` object — with its own `Properties` and `Space` array. Nesting depth is not limited.

```json
{
  "Properties": {
    "Id": "<nested-space-uuid>",
    "Project.Name": "Sub-section A"
  },
  "Space": [
    ...items...
  ]
}
```

---

### Inline vs structured summary

| Entity | Inline form | Structured form |
|---|---|---|
| `CurveElement` | `Object: "CurveElement"` + `Properties` (segment-level) + `CoordinateStyle` + `Coordinates` | `Geometry` wrapper with `CurveElement` array (one segment) |
| `Curve` | — | `Geometry` wrapper with `CurveElement` array (two or more segments) |
| `Point` | `Point` array, no container `Properties` | — |
| `Points` | — | Container `Properties.Id` + `Point` array with two or more nodes |
| `Space` | — | Full `Space` object with `Properties` + `Space` array |

---

## Appendix A — Bulge to Arc Geometry

Given two consecutive vertices **P1** = `(x1, y1)` and **P2** = `(x2, y2)` with bulge **b** at P1:

```
d     = distance(P1, P2)
θ     = 4 · arctan(b)              // included angle (radians)
r     = d / (2 · sin(θ / 2))       // arc radius
```

The sagitta (height of the arc above the chord):

```
s     = r · (1 − cos(θ / 2))
```

The arc center lies perpendicular to the chord at its midpoint, offset by:

```
offset = r − s   (on the left side of P1→P2 for b > 0, right side for b < 0)
```

| Bulge | θ (°) | Arc type |
|---|---|---|
| 0 | 0 | Straight line |
| 0.41421356 | 45° | Eighth-circle arc |
| 1 | 180° | Semicircle |
| > 1 | > 180° | Major arc |

---

## Appendix B — Reserved Property Key Prefixes

| Prefix | Owner | Description |
|---|---|---|
| `CapJson.` | CapCAD format | Format metadata: `Version`, `Units`, `BBox` |
| `Acad.` | AutoCAD integration | AutoCAD entity properties round-tripped from DWG |
| `Curve.` | CapCAD geometry | Curve computation parameters: `Factor`, `Initial` |

All other key prefixes are available for application use and are preserved as xdata on AutoCAD import.

---

*Document version: 1.1 — MultiCurve pending decision.*
