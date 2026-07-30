# CorticalConnectivity

Generates schematic diagrams of structural/functional connectivity for a chosen region of interest (currently **FST**, **MST**, or **MT**), built from a literature review of human and macaque tracer, DTI tractography, rs-fMRI, and inactivation studies. Node positions are plotted on top of a cortical surface snapshot, with edge/node styling (line weight, color, dashing) driven by the evidence behind each connection.

The whole pipeline lives in a single notebook: [network_diagram.ipynb](network_diagram.ipynb).

## Requirements

Python 3 with:

```bash
pip install pandas numpy plotly pillow matplotlib networkx kaleido
```

`kaleido` is required by plotly's `fig.write_image(...)` call used to export the final PDF.

> Note: the notebook also contains some unused, exploratory cells near the bottom (`pyvis`, `jaal`, `scikit-network`) left over from evaluating alternative plotting libraries. These are **not** part of the working pipeline and reference files (e.g. `facebook_combined.txt`) that aren't in this repo — you can ignore or delete them.

## Before you run it

1. **Set `datapath`.** Near the top of the second cell, `datapath` is hardcoded to a local absolute path:
   ```python
   datapath = "/Users/rje257/Documents/GitHub/CorticalConnectivity/"
   ```
   Change this to wherever you've cloned this repo.
2. **Run the notebook from the repo root.** The plotting cell loads `nodes.csv`/`edges.csv` with paths relative to the working directory (`os.path.join(species, "selectnodes.csv")`), so your Jupyter working directory must be the repo root, not a subfolder.

## Important flags

The first markdown cell in the notebook summarizes these, but here's what each one actually controls:

### Data selection

| Flag | Values | Effect |
|---|---|---|
| `species` | `"human"` or `"macaque"` | Which subfolder (`human/` or `macaque/`) to read data from and which surface image to plot on. |
| `mainROI` | `"FST"`, `"MST"`, or `"MT"` | The seed region — all edges/nodes are computed relative to this region's row in `evidence.csv`. |
| `includestudies` | `"all"` or a set, e.g. `{"tracer"}` | Restricts which `study_type` rows from `evidence.csv` are used. Defaults differ by species in the notebook (macaque includes `tracer`, `DTI tractography`, `inactivation`; human includes `DTI tractography`, `rs-fMRI`, since tracer/inactivation data isn't available in humans). |

### Edge/node styling

| Flag | Values | Effect |
|---|---|---|
| `edgeweight` | `"evidencecount"` or `"connectivitystrength"` | Controls line thickness / node size: either the number of supporting studies, or a rough average of reported connection strength (weak/moderate/strong), taking the max of forward and backward projection strength. |
| `edgecolor` | `"hierarchy"` or `"projections"` | Controls line/node edge color: hierarchical relationship to the main ROI (higher/lower/equal/unknown, based on Felleman & Van Essen-style hierarchy) or projection direction relative to the main ROI (sends/receives/bidirectional). |
| `edgedashed` | `"certainty"` | Determines solid vs. dashed styling: `"certain"` (positive evidence, or absence not explicitly tested) vs. `"conflicting"` (some studies find the connection, others explicitly test for and don't find it). |
| `comment` | `"evidencesource"` | Populates the hover text for each node with the abbreviated study codes that support connections to it (defined in `citations.txt`). |
| `remove_unconnected_nodes` | `0` or `1` | If `1`, drops nodes from `selectnodes.csv`/the plot that have no surviving connection to the main ROI after filtering. |

### Plot layout (final cell)

| Flag | Values | Effect |
|---|---|---|
| `nodesOnly` | `0` or `1` | If `1`, skips drawing connection lines and plots nodes only. |
| `roi` | `"FST"`, `"MST"`, `"MT"` | Should match `mainROI` above — used to look up the right `{roi}comments` column and label the output file. |
| `roicoarse` | `0` or `1` | Intended to toggle coarse vs. fine-grained ROI selection. **Not yet implemented** — currently has no effect. |
| `displaybrain` | `0` or `1` | If `1`, overlays the node/edge plot on the cortical surface image. If `0`, nodes are drawn on a blank canvas (a schematic/hierarchy-style layout). |
| `rotateon` | `0` or `1` | Only read when `displaybrain == 0`; rotates the layout to be vertical (hierarchy-style). |
| `medial` | `0` or `1` | Only read when `displaybrain == 0`; additionally plots the medial surface view. When plotting on the brain image, also selects which surface snapshot image is loaded (`veryinflated_montage_nolabels_subcort.tif` vs. `veryinflated_lat_white.tif`). |
| `subcortical` | `0` or `1` | Only read when `displaybrain == 0`; additionally plots subcortical regions. |

Known regions that currently can't be displayed on the surface image: for macaque — TF, TH, PIP, PO (V6/V6A), MDP (7m), and medial parietal areas (BA23, RSC, BA31); for human — V6 and RSC (too medial/ventral to appear on the chosen surface view).

## Output

Running the full pipeline for a given `species`/`mainROI`:

1. Rewrites `<species>/edges.csv` — the computed connection list for the current settings.
2. Rewrites `<species>/selectnodes.csv` — the node list filtered/annotated for the current settings.
3. Displays an interactive Plotly figure.
4. Saves a static image to the repo root as `<species>_<roi>_displaybrain<0|1>.pdf` (e.g. [human_FST_displaybrain0.pdf](human_FST_displaybrain0.pdf), [macaque_FST_displaybrain0.pdf](macaque_FST_displaybrain0.pdf)).

Because `edges.csv` and `selectnodes.csv` are regenerated (and overwritten) each run, treat them as build artifacts of `evidence.csv` and `nodes.csv` rather than source data to hand-edit.

## Repository structure

```
CorticalConnectivity/
├── network_diagram.ipynb   # main notebook — see above
├── human/                  # human data + assets
├── macaque/                # macaque data + assets
├── human_FST_displaybrain0.pdf   # example output figure
├── macaque_FST_displaybrain0.pdf # example output figure
└── testsurface.png         # scratch image used by an experimental/unused cell
```

Each species folder (`human/`, `macaque/`) has the same layout:

| File | Description |
|---|---|
| `nodes.csv` | Master list of all candidate ROIs for the species: id, plot coordinates (`x`,`y`, and normalized `center_x`,`center_y`), display `label`, `color`, marker `size`, and a `{roi}comments` column populated per run with hover-text citations. |
| `evidence.csv` | The literature review itself — one row per study/finding relating `Main` (the seed ROI, e.g. FST/MST/MT) to an `Affiliate` region. Columns capture wake state, study type, projection presence/strength in each direction with references, hierarchical level relative to `Main`, and free-text notes. This is the source data the notebook reads to build everything else. |
| `edges.csv` | **Generated** by the notebook — the filtered/summarized connection list (evidence count, strength, hierarchy, certainty, projection direction) for the current `mainROI`/flag settings. Overwritten on each run. |
| `selectnodes.csv` | **Generated** by the notebook — `nodes.csv` filtered down to nodes actually used/connected for the current run (if `remove_unconnected_nodes = 1`), plus hover-text comments. Overwritten on each run. |
| `abbreviations.csv` | Expands ROI/region abbreviations (e.g. `FST` → fundus of the superior temporal area) used throughout the other files. |
| `citations.txt` | Maps the short reference codes used in `evidence.csv` (e.g. `Bak18`, `Rol23`) to full citations. |
| `evidence.xlsm` | Working spreadsheet version of `evidence.csv`, used for editing/curating the review data before exporting to CSV. |
| `surface_snapshots/` | Cortical surface renderings (`.tif` images, plus an `.ai` source file) at different inflation levels (`white`, `midthickness`, `inflated`, `veryinflated`) and views (`lat`, `med`, `dor`, `ven`, and combined montages), used as the background image in the plots. These are pre-rendered assets — no need to regenerate them to use the notebook. |

Both species folders follow the identical schema, so the notebook's logic is species-agnostic aside from the `datapath`/`species` flag.
