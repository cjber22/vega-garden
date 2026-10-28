# 🧩 Puzzle Flow — ERP Implementation as Interlocking Puzzle Pieces

![Puzzle Flow](thumbnail.png)

A hierarchical project plan — phases, modules, tasks and deliverables — visualized as a grid of **interlocking puzzle pieces**. Each piece is colored by its **completion percentage**, connected to its parent by dashed links, and shows a tooltip with the task's description, parent and progress.

Built once in **[Vega](https://vega.github.io/vega/)** and rendered in **Power BI** via **[Deneb](https://deneb-viz.github.io/)** from the exact same specification.

> [!IMPORTANT]
> **AI assistance disclosure:** This README was prepared with the assistance of a large language model (LLM), based on the author's Vega specification and Power BI project files. The sample dataset (`erp_sample_dataset.json`, and the data embedded in the Power BI semantic model) is **synthetic and LLM-generated** — it depicts a fictional ERP implementation project and contains no real data. The **initial draft of the SVG path for the puzzle piece** was produced with the assistance of an LLM and then tweaked manually by the author. The Vega specification and Deneb/Power BI configuration were authored by [Christian Bermudez](https://www.linkedin.com/in/cjber/). The puzzle-piece concept originated from an original design by [Christian Bermudez](https://www.linkedin.com/in/cjber/).

## 📖 Contents

- [What it shows](#-what-it-shows)
- [Repository layout](#-repository-layout)
- [Try it in the Vega Editor](#-try-it-in-the-vega-editor)
- [Open it in Power BI with Deneb](#-open-it-in-power-bi-with-deneb)
- [How it works](#-how-it-works)
- [Data contract](#-data-contract)
- [Customizing it](#-customizing-it)
- [Ideas & next steps](#-ideas--next-steps)
- [Credits](#-credits)

## 🎯 What it shows

The sample scenario is a small **ERP implementation** with 12 tasks across four levels:

| Level | Prefix | Examples |
|-------|--------|----------|
| Phase | `P` | Project Initiation, Requirements Gathering, Solution Design |
| Module | `M` | GL Configuration, AP Setup, AR Setup, Inventory Module |
| Task | `T` | Data Migration, Validation & UAT |
| Deliverable | `D` | End-User Training, Cutover & Go-Live, Post-Go-Live Support |

Reading the chart:

- **Position** — tasks are arranged in columns by tree depth (phase → module → task → deliverable) and wrapped into rows, so a long plan stays compact.
- **Color** — a `sqrt` color scale maps `progress` from 0 to 1: pale yellow at "not started", through light blue and green, to deep green at 100% complete. The square-root curve makes differences between early milestones easier to see.
- **Interlocking** — each piece's tabs/notches are sized from the column/row spacing signals, so pieces interlock visually across the grid.
- **Links** — dashed rules connect each task to its parent, preserving the hierarchy without a classic node-link tree look.
- **Tooltips** — hovering a piece shows the task name, description, parent and formatted progress.

## 📁 Repository layout

- [`Vega_Spec/`](Vega_Spec/) — the standalone spec (`puzzle_flow_spec.json`, sample data inline) and the sample dataset (`erp_sample_dataset.json`)
- [`PowerBI/`](PowerBI/) — the PBIP project with the Deneb visual (`PowerBI_example.pbip`)

## 🧪 Try it in the Vega Editor

No Power BI needed — the standalone spec has the sample data inlined:

1. Open the [Vega Editor](https://vega.github.io/editor/).
2. Copy the contents of [`Vega_Spec/puzzle_flow_spec.json`](Vega_Spec/puzzle_flow_spec.json).
3. Paste it into the editor (replace the sample spec) — the chart renders immediately.

To swap in your own data, edit the `values` array inside the spec's `dataset` data source, keeping the field names from the [data contract](#-data-contract) below.

## 🔌 Open it in Power BI with Deneb

The included PBIP project contains a one-page report with the visual already configured.

**Requirements:**

- [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (a recent version with PBIP support)
- The [Deneb custom visual](https://deneb.link/appsource?source=website&mktcmpid=homepage) — in Power BI Desktop, get it via **Visualizations → Get more visuals**. (The visual is bundled in the report definition; installing it in Desktop just makes editing seamless.)

**Steps:**

1. Clone or download this repository.
2. Open `PowerBI/PowerBI_example.pbip` in Power BI Desktop.
3. If prompted to trust/work with the PBIP files, accept.
4. The report page shows the Puzzle Flow visual — click it and open the Deneb editor (pencil icon on the visual) to explore or edit the spec.

**Field setup (if you rebuild it from scratch):**

| Deneb field well | Column |
|------------------|--------|
| Values | `input_data[id]`, `input_data[label]`, `input_data[parent]`, `input_data[description]`, `input_data[progress]` |

The semantic model contains a single table, `input_data`, loaded from a compressed inline M expression (the same synthetic dataset as `erp_sample_dataset.json`).

<details>
<summary><strong>📦 Importing just the sample data into your own model</strong></summary>

`Vega_Spec/erp_sample_dataset.json` is shaped exactly like the `dataset` data source Deneb exposes — `{"name": "dataset", "values": [...]}`. Two easy ways to load it elsewhere:

- **Vega Editor:** replace the spec's inline `values` with a `url` pointing to the file (or paste values from it).
- **Power Query:** ingest the JSON, expand `values`, and promote headers to get `id`, `label`, `progress`, `parent`, `description`.

</details>

## ⚙️ How it works

The spec is plain Vega (no Vega-Lite shortcuts) and leans on a few techniques worth stealing for your own charts:

1. **Stratify + tidy tree layout** — `dataset` is a flat task list with a `parent` pointer. Vega's `stratify` expects a single root, so a **dummy `_zero_` root node** is injected (the `_zero_` data source) and every task with no `parent` is re-pointed to it via the `parentNew` formula. The dummy node is then filtered back out (`datum.id !== "_zero_"`) so it never appears on screen. This trick is what allows **multiple root nodes** (tasks without a parent) in one chart.

   > [!TIP]
   > If you don't need multiple roots, you can remove the `_zero_` data source and the `parentNew` formula and stratify directly on `parent`. This keeps every mark backed 1:1 by a real row of the Deneb `dataset`, which makes per-node features in Deneb — such as **Power BI drill-through** — behave stably.

2. **Depth-based grid with row wrap** — `depth_counts` counts tasks per depth; `maxRows` is the largest count. Each task gets `col = depth - 1` and `row = (idx - 1) % maxRows`, so columns wrap into a dense grid instead of a sparse tree. Pixel positions come from `x = colOffset + col * colSpacing` and `y = rowOffset + row * rowSpacing`.

3. **Signal-driven puzzle path** — the puzzle outline is a single SVG `path` string assembled in the `pathVal` signal from spacing-derived signals (`notchStartV`, `notchStartH`, `notchHeightV`, `notchHeightH`). Every piece reuses this one path, so changing spacing automatically resizes the tabs and notches to stay interlocking.

4. **Progress color scale** — a `sqrt`-type scale over `[0, 0.01, 0.5, 0.99, 1]` with an RGB piecewise ramp: not-started (pale yellow) → in progress (blue → greens) → complete (deep green).

5. **Dashed hierarchy links** — `treelinks` + two lookups re-join the laid-out `x`/`y` back onto source/target pairs, drawn as subtle dashed `rule` marks behind the pieces.

6. **Label wrapping via regex** — long labels are wrapped at `wrapBoxChars` characters using a `replace(..., regexp(...), ...)` formula, then rendered with `lineBreak: "\n"`.

7. **Hand-rolled legend** — a group mark with 11 gradient swatches (one per 10% step) plus tick rules and labels, positioned off the `taskYExtent` signal so it always sits below the grid.

## 📋 Data contract

The chart expects one row per task with these fields:

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | Unique task identifier (e.g. `P1`, `M2`, `T1`) |
| `label` | string | Task name shown on the piece (auto-wrapped) |
| `progress` | number | `0` – `1` completion fraction, drives color + tooltip |
| `parent` | string \| null | `id` of the parent task; `null` for top-level phases |
| `description` | string | Free text for the tooltip |

Rules:

- **Validate your hierarchy upstream.** The spec performs **no validation** of the `parent` references. Every row's `parent` must either be `null` or point to an existing `id` — if it doesn't, the chart will silently fail to render (or render partially) **without any error in the log**. Make sure your data is clean *before* it reaches the visual (e.g. in Power Query or your source system).
- Depth is derived from the parent chain; **four levels** (phase → module → task → deliverable) fit the default 850×750 canvas. More levels may need smaller `colSpacing`/`rowSpacing` or a bigger canvas.
- Avoid duplicate `id`s — `stratify` will fail.

## 🎨 Customizing it

All the knobs live in the `signals` block at the top of the spec:

| Signal | Default | Effect |
|--------|---------|--------|
| `colSpacing` | `140` | Horizontal distance between columns (also sizes horizontal notches) |
| `rowSpacing` | `95` | Vertical distance between rows (also sizes vertical notches) |
| `colOffset` / `rowOffset` | `80` / `90` | Grid origin |
| `wrapBoxChars` | `14` | Max characters per label line |
| `legendWidth` / `legendHeight` | `300` / `20` | Legend swatch dimensions |

To change the color story, edit the `progressColor` scale's `range` (and `domain` breakpoints). To use a measure other than progress, rebind the fill `field` in the `puzzle_pieces` mark and the legend's `legend_values`.

## 💡 Ideas & next steps

This is the first chart in the [Vega Garden](../README.md) — **I'd love your ideas for what to build next.** A few directions I'm already curious about:

- Replacing the tidy-tree grid with a true node-link or radial layout
- Multi-metric pieces (e.g. budget burn vs. progress, owner avatars)
- Cross-filtering from Deneb selections back into the report page
- Variations for OKRs, product roadmaps, or CI/CD pipelines

👉 **Share your ideas** by [opening an issue](../issues) on this repository, or reach me on [LinkedIn](https://www.linkedin.com/in/cjber/). If you build a variant yourself, feel free to fork — and open a PR if you'd like it featured.

## 🙏 Credits

- **Concept, Vega specification & Power BI/Deneb implementation:** [Christian Bermudez](https://www.linkedin.com/in/cjber/)
- **Puzzle-piece SVG path:** initial draft produced with the assistance of an LLM, then tweaked manually by the author.
- **README & synthetic sample data:** prepared with the assistance of an LLM, at the direction of the author.
- Built on [Vega](https://vega.github.io/vega/), [Deneb](https://deneb-viz.github.io/) and [Power BI](https://powerbi.microsoft.com/).
- Repository format inspired by [Deneb-Showcase](https://github.com/PBI-David/Deneb-Showcase) by Davide Bacci.
