# geoview

> **This repository hosts geoview releases**: the installable package (a `.whl` file) and the
> quick-start notebook, attached to each version on the
> [Releases page](https://github.com/ramaguirre/geoview-releases/releases).
> Found a problem or have a suggestion? Open an
> [issue](https://github.com/ramaguirre/geoview-releases/issues), with the version and what you did.

Interactive 3D viewer for geology data in Python: block models, drillholes, surfaces and
solids, points, and photo-textured meshes (e.g. core-tray photos). One Python API drives a
fast browser renderer (three.js / WebGL), in Jupyter or VS Code notebooks, or as a single
self-contained HTML file that anyone can open without Python.

- Millions of blocks stay smooth (GPU instancing; hidden blocks are skipped).
- Hover for values, click to select; clicks come back to Python as events.
- Leapfrog-style layers dock, colour maps, display filters, slicer and sections, clip box,
  vertical exaggeration, transparency that adds up, hole labels.
- Reads DataFrames, CSV / Parquet / Excel tables, Leapfrog OMF (v1), Wavefront OBJ, and
  PyVista meshes, point clouds and grids (and turns layers back into PyVista objects).

## Installation

You need **Python 3.11 or newer**. A virtual environment keeps geoview apart from your other
packages (recommended, not required).

**1. Create and activate an environment** (skip if you already have one):

```bash
python -m venv geoview-env
geoview-env\Scripts\activate          # Windows
source geoview-env/bin/activate       # macOS / Linux
```

**2. Install geoview** (the link is the wheel file on the release page; change the version
number for other releases):

```bash
pip install "geoview[textures] @ https://github.com/ramaguirre/geoview-releases/releases/download/v0.7.0/geoview-0.7.0-py3-none-any.whl"
```

`[textures]` adds Pillow, needed only for photo-textured meshes; `[pyvista]` adds PyVista,
for sending layers back to PyVista (`geoview[textures,pyvista]` for both). Leave out what you
don't need. Without internet access, download the `.whl` file from the release page and run
`pip install geoview-0.7.0-py3-none-any.whl` in its folder.

**3. Install a notebook front end**, if you don't have one:

```bash
pip install jupyterlab               # then run: jupyter lab
```

or use VS Code with its Jupyter extension (`pip install ipykernel`, then pick this
environment as the notebook's kernel).

**4. Check it works:** download `quickstart.ipynb` from the release page, open it, and run
all cells. It uses built-in synthetic data, so it needs no files of your own.

**Upgrading:** run the step 2 command with the new version's link and `--upgrade`.

## Quick start

```python
from geoview import Viewer, BlockModel, Points, demo

df = demo.block_model()                      # or your own DataFrame of block centroids
bm = BlockModel.from_dataframe(df, size=10)  # finds X/Y/Z; size = block size (or DX/DY/DZ columns)

v = Viewer(height=650)
v.add(bm.subset(bm.attributes["CU"] > 0.2), "Cu > 0.2")
v.add(Points.from_dataframe(demo.collars(), size=12), "collars", color_by="STATUS")
v.on_select(lambda sel: print(sel.record if sel else "nothing selected"))
v                                            # last line of a cell: the view appears
```

Share the view with anyone, no Python needed:

```python
v.to_html("scene.html", title="My model")
```

## Data

### Block models and points

```python
bm = BlockModel.from_dataframe(df)                       # X/Y/Z + DX/DY/DZ columns detected
bm = BlockModel.from_dataframe(df, xyz=("XC", "YC", "ZC"), size=(10, 10, 5))
sub = BlockModel.from_dataframe(df, size=("XINC", "YINC", "ZINC"))   # sub-blocks: per-block sizes
pts = Points.from_dataframe(samples, size=5)             # spheres of 5 m
bm.subset(bm.attributes["CU"] >= 0.3)                    # filter before sending: the cheapest speed-up
```

### Drillholes

```python
from geoview import Drillholes

dh = Drillholes.from_tables(collars, surveys, assays, radius=3)  # desurveyed by minimum curvature
traces = Drillholes.from_tables(collars)                          # collars only: straight traces
comp = Drillholes.from_points(df, at="mid")                       # one row per interval with its midpoint
```

Column names are detected case-insensitively (HOLEID/BHID/DHID, FROM/TO, AT/DEPTH/DISTANCE,
AZIMUTH/BRG, DIP, X/Y/Z...) or passed explicitly (`hole=`, `from_=`, `at=`, ...).
Conventions: azimuth clockwise from grid north; **dip negative downwards** (-90 = vertical),
or pass `dip_positive_down=True`. A hole without survey rows uses the collar's azimuth and
dip, else it is vertical.

### Surfaces and solids

```python
from geoview import Surface
from geoview.io import read_obj

topo = Surface.from_grid(x, y, z)            # a DEM: x (nx,), y (ny,), z (ny, nx); NaN = hole
pit = Surface(vertices, triangles)           # any triangulation; closed meshes are solids
shell = read_obj("pit_shell.obj")            # Wavefront OBJ
```

Triangles are flat-shaded so every one shows. Display modes: filled, wireframe, both, and
slicer edge.

### PyVista (both ways)

Anything you build in PyVista can go straight into the view, and any geoview layer can go
back to PyVista for more processing. Every data array comes along, and coordinates stay
float64.

```python
import pyvista as pv

v.add(mesh, "wireframe")                       # PolyData surface or solid
v.add(grid.threshold(0.5, scalars="cu"), "Cu > 0.5")   # grids and thresholded grids -> blocks
v.add(grid.slice_orthogonal(), "slices")       # a MultiBlock adds one layer per block
layer = geoview.from_pyvista(mesh)             # or convert explicitly

bm.to_pyvista()                                # BlockModel -> UnstructuredGrid of voxels
bm.to_pyvista(grid=True)                       # ... or a regular ImageData (NaN where no block)
surf.to_pyvista()                              # Surface / Points -> PolyData; Drillholes -> lines
```

| PyVista | geoview |
|---|---|
| PolyData with polygons | `Surface` (triangulated; cell data stays per face) |
| PolyData with lines | line segments (`Drillholes`, drawn as lines), `line` numbers each polyline |
| PolyData with only points | `Points` |
| ImageData, RectilinearGrid | `BlockModel`, one block per cell (point-only data is averaged to cells) |
| UnstructuredGrid of voxels / axis-aligned hexahedra (threshold, clip...) | `BlockModel`, a size per block |
| other grids | `Surface` of the outer boundary |
| MultiBlock | one layer per block |

Layers colour by the mesh's active scalars, as PyVista plots them. Strings and booleans
become categories; vectors split into `_x`/`_y`/`_z`. Empty cells (all NaN) and blanked
cells are dropped (`from_pyvista(grid, omit_empty=False)` keeps the NaN ones). Rotated grids
are placed correctly but drawn axis-aligned (a warning says so). About 0.3 s per million
cells either way. `to_pyvista` needs PyVista installed (`pip install "geoview[pyvista]"`).

### Leapfrog OMF

```python
from geoview.omf import omf_contents, read_omf

omf_contents("model.omf")                    # what's inside, without reading the arrays
for name, layer in read_omf("model.omf").items():
    v.add(layer, name)
```

| OMF element | geoview layer |
|---|---|
| point set | `Points` |
| line set / borehole | `Drillholes`, one interval per segment (`line` numbers each connected line) |
| surface, grid surface | `Surface` |
| volume (block model) | `BlockModel` (empty cells dropped; `omit_empty=False` keeps them) |

Data comes along as attributes; legend (mapped) data becomes categories with the legend's
colours. `elements=[...]` reads only some elements. Rotated block models are placed
correctly but drawn axis-aligned (a warning says so). OMF v2 files are not read yet.

### Photo-textured meshes

OBJ files with `.mtl` materials and images (core-tray photo panels, photogrammetry) keep
their photos. Needs the `[textures]` extra.

```python
import pathlib
from geoview.textures import read_textured_obj

trays = read_textured_obj(sorted(pathlib.Path("trays").glob("*.obj")))   # many files, one layer
trays.texture_scale = 0.5                    # half-size photos: a quarter of the GPU memory
v.add(trays, "core trays")                   # clicking a panel reports its hole and photo file
```

The limit is GPU memory: width × height × 4 bytes per photo, whatever the JPEG size. As a
guide, on a laptop with integrated graphics, about 9,400 photos of 444 × 279 px load in about
5 s at half size (1.9 GB of GPU memory), and around 5,000 at full size (4.4 GB). Very large
photo sets at full size are too big for a single HTML file: use half size, or a selection.

## In the view

- **Rotate**: left-drag. **Pan**: right-drag. **Zoom**: wheel. **Double-click** an object to
  rotate around it. **Fit** zooms to everything visible.
- **Hover** shows a record's values; **click** selects it (values in a panel, and an event in
  Python: `v.on_select(fn)`, `v.selected`).
- **Layers dock** below the view: eye (show/hide), colour-by, colours, opacity, remove; the
  selected layer's properties on the right (colour map, range, categories, display filter,
  blending, display mode). Drag the divider to resize; the **Layers** button hides it.
- **Toolbar**: Slice, Axes (coordinate labels), Z × (vertical exaggeration), projection
  (orthographic by default), Layers, Fit, Load.
- **Load** (or drop a file on the view) reads CSV / TSV / Parquet / Excel tables (collars,
  surveys, intervals, block models, points, recognised by their columns), OMF and OBJ files,
  and adds them as layers. Needs a live notebook; the HTML export has no Load button.

### Slicer and sections

**Slice** adds a slicer row to the layers list. Its properties: plane or box; orientation
(**view parallel**, the default: edge-on through the view centre; **view perpendicular**:
facing you; plan; N–S or E–W section; custom dip/azimuth); follow the camera; position;
thickness (0 = a cut); flip; **Look at plane** (Shift+click: from the opposite side); **Draw
line** (click two points on the view).

Mouse, while a plane slicer is on: **Ctrl + right-drag** slides it; **middle-drag** or
**Ctrl + left + right drag** sets its thickness.

Surfaces and solids can show just their **slicer edge**, and solids can **fill** the cut.
Block models can be shown as their exact cut on the plane (filled cells, outlined as a solid
or block by block), and drawn smaller (**Block size**) to see between them. Drillholes can
be drawn as **lines** instead of tubes.

### From Python

```python
v.set("collars", visible=False)
v.set("Cu > 0.2", color_by="LITH")
v.set("blocks", range=(0.2, 1.0), cmap="turbo")
v.set("assays", filter={"attribute": "CU", "range": (0.4, 5)})         # display filter
v.set("blocks", filter={"attribute": "LITH", "include": ["Porphyry"]})
v.set("blocks", opacity=0.12, blend="accumulate")    # opacity adds up with depth, like a cloud
v.set("assays", labels=True, display="lines")
v.section(45, thickness=40)                          # vertical 40 m slab trending 045, face-on
v.slice(line=[(x1, y1), (x2, y2)], thickness=40)     # vertical section through two map points
v.slice(dip=0, through=(x, y, 2100))                 # horizontal cut
v.clip_box(min=(x0, y0, z0), max=(x1, y1, z1))
v.clear_clip()
v.z_scale = 2
v.remove("collars")
v.fit()
```

## Coordinates

Real coordinates (e.g. UTM) stay in float64 in Python; the GPU only sees offsets from a
local origin, so large coordinates don't jitter. Tooltips, the info panel and events always
report real coordinates.

## Large models

Measured on a laptop with integrated graphics, on real porphyry-copper models:

| blocks shown | frame | click / hover | notebook load |
|---|---|---|---|
| ~1 million | 18 ms | 3 ms | ~2 s |
| 5 million sub-blocks | 12 ms | 8 ms | ~7 s |
| 10 million sub-blocks | 29 ms | 14 ms | — |

A layer holds at most 16.7 million records (geoview warns above 10 million). For larger
models, filter in Python first (`bm.subset(mask)`, by grade, category or region) or split
them into several layers.

## Licence

MIT: free to use, modify and share, including commercially, as long as the licence
notice is kept. See [LICENSE](LICENSE).
