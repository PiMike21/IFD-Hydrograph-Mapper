# Hydrograph Phase Mapper for InfoDrainage

A standalone browser-based utility for mapping **InfoDrainage hydrograph files (`.idhyqx`)** into an **InfoDrainage model (`.iddx`)**. The tool works phase-by-phase and junction-by-junction, allowing each selected hydrograph to be added as a new hydrograph inflow or used to replace an existing catchment, base-flow, or hydrograph connection.

All file parsing and model editing happen locally in the browser. The page does not upload the selected files anywhere.

---

## Overview

Hydrograph Phase Mapper loads one InfoDrainage `.iddx` model together with one or more `.idhyqx` hydrograph files. It discovers junction (`jt`) rows within each model phase, identifies supported inflow connections routed to those junctions, and presents a mapping table for assigning replacement hydrographs.

Mappings are scoped to the active phase. For each junction, the user can either:

- add a new `InHydNod` hydrograph connection;
- replace the `FlowHyd` data in an existing `InHydNod`;
- convert an existing `AreaInflowNode` catchment connection to `InHydNod`; or
- convert an existing `BaseInflNod` base-flow connection to `InHydNod`.

The modified model is exported as a new `.iddx` file.

---

## Features

### Phase-Based Hydrograph Mapping

| Feature | Description |
| --- | --- |
| **Active phase selection** | Maps junctions within one InfoDrainage phase at a time |
| **Junction discovery** | Reads `jt` elements from each phase's `Junctions` collection |
| **Incoming-flow inspection** | Shows hydrograph, catchment, and base-flow connections routed to each junction |
| **Apply target selection** | Adds a new hydrograph connection or replaces a specific existing inflow |
| **Hydrograph selection** | Assigns any loaded `.idhyqx` file to a junction row |
| **Phase-specific clearing** | Removes all mapping selections for the active phase |
| **Cross-phase mapping state** | Keeps selections from other phases while changing the active phase |

### File Loading

- Drag-and-drop support for `.iddx` and `.idhyqx` files
- Separate file pickers for the model and hydrograph library
- One `.iddx` model loaded at a time
- Multiple `.idhyqx` hydrographs can be loaded
- Individual loaded files can be removed from the file-chip list
- Removing a hydrograph also removes mappings that reference that file
- Reset clears the model, hydrographs, mappings, search, selected phase, and sort state

### Search and Sorting

- Search within the active phase using phase labels, structure labels, and incoming-flow labels/summaries
- Sort structures alphabetically A–Z or Z–A
- Numeric-aware, case-insensitive structure sorting
- Large table views are capped at 2,000 rendered junction rows

### Auto-Match

The **Auto-match** action works on the currently active and filtered rows.

For each row, it looks for a loaded hydrograph whose:

- `FlowHyd` title appears in the row's searchable text; or
- `.idhyqx` filename without its extension appears in the row's searchable text.

When a hydrograph is matched, the tool chooses the replacement target using the following logic:

1. If an existing hydrograph connection contains the matched hydrograph title in its label or hydrograph summary, that connection is selected.
2. Otherwise, if the junction has exactly one existing hydrograph connection, that connection is selected.
3. Otherwise, the hydrograph is configured as a new connection.

### Mapping Summary

The interface reports:

- number of phases;
- total junction rows;
- visible rows in the current phase/search;
- number of loaded hydrographs;
- selections in the active view;
- mappings across all phases;
- per-phase junction and mapping totals; and
- current model and hydrograph file summaries.

---

## Supported InfoDrainage Elements

The mapper recognizes these inflow-node types when they are routed to a discovered junction:

| XML element | UI type | Export behavior |
| --- | --- | --- |
| `InHydNod` | Hydro | Replaces or inserts its `FlowHyd` element |
| `AreaInflowNode` | Catchment | Converts the node to `InHydNod` and inserts the selected `FlowHyd` |
| `BaseInflNod` | Base | Converts the node to `InHydNod` and inserts the selected `FlowHyd` |

Connections are associated with a junction by matching the inflow node's `ToDest` `ftGUID` to the junction GUID.

The model summary reports discovered junctions; SWC structures are omitted from the junction-row view.

---

## Export Behavior

### Adding a New Hydrograph Connection

When **Add new hydrograph connection** is selected, the exporter creates a new `InHydNod` that:

- receives the next available inflow-node `Index`;
- receives a new UUID/GUID;
- uses the hydrograph title as its label when available;
- copies X/Y coordinates from the destination junction;
- contains an empty `PollConcs` element;
- contains the tool's default `RainwaterTank` structure;
- routes `ToDest ftGUID` to the junction GUID;
- routes `ToDest ToLabel` to the junction label; and
- imports the selected hydrograph's `FlowHyd` element.

If the junction already contains `InletDetails` with at least one `IDetail`, the tool clones the first detail, gives it a new index and GUID, and points its first `FromSource` to the newly created hydrograph node.

### Replacing an Existing Hydrograph

For an existing `InHydNod`, the exporter:

- replaces the existing direct-child `FlowHyd` element when one exists; or
- appends the selected `FlowHyd` when the node has none.

The existing inflow-node element itself remains in place.

### Replacing a Catchment or Base Flow

For an existing `AreaInflowNode` or `BaseInflNod`, the exporter replaces that element with a new `InHydNod` shell while preserving available routing and identity information from the original node:

- `Index`
- `GUID`
- X/Y coordinates
- `ToDest ftGUID`
- `ToDest ToLabel`
- existing `RainwaterTank` XML when it can be serialized

The new node receives the selected hydrograph's `FlowHyd`. Its label uses the hydrograph title when available, otherwise the prior node label, otherwise `Hydrograph`.

Verify catchment/base-to-hydrograph conversions in InfoDrainage after export.

### Output Filename

Exports are downloaded using this pattern:

```text
<original-model-name>-hydrograph-mapped.iddx
```

Characters that are invalid in common filenames are replaced with underscores.

---

## File Validation and Limits

### `.iddx` Model

- Filename must end in `.iddx`
- Maximum file size: **100 MB**
- Content must be well-formed XML
- Root element must be `InfoDrainage`
- XML containing `<!DOCTYPE` is rejected

### `.idhyqx` Hydrographs

- Filename must end in `.idhyqx`
- Maximum file size per file: **50 MB**
- Content must be well-formed XML
- Root element must be `InfoDrainage`
- XML containing `<!DOCTYPE` is rejected
- File must contain a `FlowHyd` element

For each loaded hydrograph, the tool records its filename, `FlowHyd` title, and number of `TimeFlow` elements.

### Rendering Limit

The mapping table renders at most:

```text
2000 junction rows
```

If more matching rows exist, only the first 2,000 are shown. Refine the search to narrow the view.

---

## Expected Model Structure

The junction mapper looks for phases under:

```text
InfoDrainage
└── Phases
    └── Phase
        └── Nodes
            ├── Junctions
            │   └── jt
            └── InflowNodes
                ├── InHydNod
                ├── AreaInflowNode
                └── BaseInflNod
```

A phase contributes mapping rows only when it contains both a `Junctions` collection and an `InflowNodes` collection under `Nodes`.

---

## Interface

The page is organized into four main areas.

### Header Status

Displays:

- batch/model readiness;
- number of loaded hydrographs;
- total junction rows; and
- `.iddx` as the output format.

### File and Mapping Controls

Provides:

- drag-and-drop file loading;
- `.iddx` selection;
- multi-file `.idhyqx` selection;
- reset;
- active-phase selection;
- search;
- active-view selection count;
- visible-junction count;
- structure-routing preservation notice;
- Auto-match;
- phase-selection clearing; and
- export.

### Summary Panels

Shows:

- overall mapping totals;
- current `.iddx` information;
- uploaded hydrograph details; and
- mapping totals by phase.

### Mapping Table

Each row contains:

| Column | Purpose |
| --- | --- |
| **Incoming flows** | Lists existing hydro, catchment, and base-flow connections to the junction |
| **Structure (junction)** | Displays the junction label |
| **Phase** | Displays the row's phase |
| **Apply to** | Selects a new connection or an existing inflow to replace |
| **Replacement hydrograph** | Selects one of the loaded `.idhyqx` files |

---

## Getting Started

No package installation or build step is required. The standalone HTML file contains its CSS and JavaScript directly.

1. Open the HTML file in a browser.
2. Choose or drop one InfoDrainage `.iddx` file.
3. Add one or more `.idhyqx` hydrograph files.
4. Select the phase to edit.
5. Optionally search or change the structure sort order.
6. For each junction, choose an **Apply to** target.
7. Select a **Replacement hydrograph**.
8. Repeat for other phases as needed.
9. Choose **Export updated `.iddx`**.
10. Review the exported model in InfoDrainage, especially where catchment or base-flow nodes were converted to hydrograph nodes.

---

## Technical Implementation

| Layer | Implementation |
| --- | --- |
| Markup | HTML5 |
| Styling | Embedded CSS |
| Application logic | Vanilla JavaScript |
| XML parsing | Browser `DOMParser` |
| XML serialization | Browser `XMLSerializer` |
| Local file reading | Browser `File` API via `file.text()` |
| Export | `Blob`, object URL, and temporary download link |
| GUID generation | `crypto.randomUUID()` with a JavaScript UUID fallback |
| State | In-memory JavaScript object and `Map` |

No imported JavaScript framework, package dependency, backend service, or database integration is used.

---

## Responsive and Accessibility Behavior

The embedded CSS includes:

- responsive single-column layouts below 1120 px;
- narrower mobile spacing and single-column summary/control layouts below 700 px;
- full-width mobile buttons;
- sticky mapping-table headers;
- an `aria-live="polite"` status region;
- accessible remove-file button labels;
- visible focus treatment for inputs; and
- reduced-motion handling for the drag-over transition.

---

## Data and Privacy

Selected files are parsed locally in the browser and are not uploaded anywhere by this standalone page.

Loaded model data, hydrographs, mappings, search text, phase selection, and sort preference are kept in JavaScript memory for the current page session. The **Reset all** action clears that state.

This project is public. Content is intended for educational and reference use.
---

## Important Notes

- Mapping rows are created from `jt` junction elements.
- Only `InHydNod`, `AreaInflowNode`, and `BaseInflNod` are recognized as supported incoming-flow types.
- Mapping selections are stored by a composite phase/junction row identifier.
- Clearing selections affects the entire active phase, not only rows currently matching the search.
- Export is disabled until a model is loaded and at least one mapping exists.
- Auto-match is disabled until the active view contains rows and at least one hydrograph is loaded.
- Structure routing is preserved through `ToDest ftGUID` in the export logic.
- No project license is declared in the HTML file.

