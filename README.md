# capcad-json
CapCad JSON format — Open format for exchanging CAD geometry with AI

CapJSON is a compact, human-readable JSON format for representing 2D CAD geometry. It is designed to make geometric entities readable and writable by both CAD software and large language models, enabling a clean round-trip edit loop: read from CAD → send to AI → receive edited geometry → write back to CAD.

---

## Why

Existing CAD exchange formats (STEP, IGES, DXF) are designed for CAD-to-CAD transfer. They are verbose, deeply nested, or binary — none of which works well in an AI context. CapJSON is purpose-built for the CAD ↔ AI workflow: flat, minimal, and unambiguous.

---

## Entities

| Entity | Description |
|---|---|
| `Point` | A single named location in 2D space |
| `Points` | An ordered collection of 2D coordinates |
| `CurveElement` | A single polyline/arc segment sequence |
| `Curve` | Two or more CurveElement segments grouped as one object |
| `Space` | A heterogeneous container holding any mix of the above, including nested Spaces |

Arc geometry is encoded using the **bulge** convention — identical to AutoCAD's representation — where `bulge = tan(θ / 4)`.

---

## Quick example

A curve with two vertices and one arc transition:

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
            "Id": "8faec39d-1f7b-4da7-87bd-02bcb81e1eff"
          },
          "CoordinateStyle": "xyb",
          "Coordinates": [
            [-76405.04, 14580.80, 0],
            [-76403.64, 14583.03, -0.4142],
            [-76400.90, 14583.65, 0]
          ]
        }
      ]
    }
  ]
}
```

The third value in each coordinate triplet is the bulge. `0` means a straight line to the next vertex. `-0.4142` means a 45° clockwise arc.

---

## Specification

The full specification is in [`SPEC.md`](./SPEC.md).

It covers:

- UUID identity conventions
- Format versioning (`CapJson.Version`)
- Coordinate styles (`xyb`, `xyzb` reserved for 3D)
- Units declaration
- Bounding box
- Properties and xdata round-trip
- All entity schemas with fields, types, and examples
- Inline vs structured forms in Space
- Nested Space
- Bulge-to-arc geometry formulas
- Reserved property key prefixes

---

## Status

**v1.1 — stable.** MultiCurve entity pending decision.

Implemented in [CapCAD](https://capcad.io) — a CAD plugin for road traffic design running on AutoCAD and Civil 3D.

---

## License

[MIT](./LICENSE) — use freely, implement anywhere, no restrictions.
