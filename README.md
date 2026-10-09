# MassCompose

MassCompose takes a mesh and the point masses bolted to it, like a battery, motor, or controller, and gives you one total mass, one center of mass, and one 3×3 inertia tensor about that center of mass.

`trimesh` already handles the mesh's own properties, so the part this package adds is the parallel-axis composition. The shifts have to come from the system center of mass instead of the origin, since a wrong tensor is still symmetric and positive semi-definite and looks fine until the simulation starts tumbling.

## Demo app

There's a small GUI demo in the releases page as `Demo.zip`. Unzip it and run `python run.py` (or `./run.sh` on Mac/Linux), which sets up a local `.venv` and installs everything the first time. After that you can skip the install checks by launching `app.py` with the venv's Python directly, `.venv\Scripts\python app.py` on Windows or `.venv/bin/python app.py` on Mac/Linux.

## Installation

```
pip install masscompose
```

It only needs `numpy` and `trimesh`. From a checkout, `pip install -e ".[dev]"` adds `pytest`.

## Usage

```python
import numpy as np
import trimesh

from masscompose import PointMass, compose, load_mesh, to_urdf_inertial

# Stand-in for your own file: a 300 x 200 x 8 mm plate, exported in mm
trimesh.creation.box(extents=[300.0, 200.0, 8.0]).export("chassis.stl")

# PETG at 40% infill. Units are always declared, never guessed
chassis = load_mesh("chassis.stl", mesh_units="mm", density=1240.0, infill=0.4)

payload = [
    PointMass([0.000, 0.000, 0.050], mass=0.420, name="battery"),
    PointMass([0.080, -0.040, 0.020], mass=0.075, name="esc"),
    PointMass([-0.120, 0.000, 0.090], mass=0.031, name="camera"),
]

mp = compose(chassis, payload)

print(f"mass = {mp.mass:.4f} kg")
print(f"com  = {np.round(mp.com, 5).tolist()} m")
print(to_urdf_inertial(mp))
```

```
mass = 0.7641 kg
com  = [0.00298, -0.00393, 0.0331] m
<inertial>
  <origin xyz="0.002983980735 -0.00392629044079 0.0330986284159" rpy="0 0 0"/>
  <mass value="0.76408"/>
  <inertia ixx="0.00139712657604" ixy="0.000231048057795" ixz="0.000290264872788" iyy="0.00320050197129" iyz="-3.92958852476e-05" izz="0.0036070176526"/>
</inertial>
```

Point mass positions are in meters and share the mesh file's frame after unit conversion. The tensor is about `mp.com`, which is why the `<origin>` carries it.

## If you only need the mesh

You don't need this package for that, since `trimesh` does it in three lines:

```python
mesh = trimesh.load("chassis.stl", force="mesh")
mesh.apply_scale(0.001)   # your file's units -> meters
mesh.density = 1240.0     # then read mesh.mass, mesh.center_mass, mesh.moment_inertia
```

Use MassCompose when there are discrete masses to add on top.

## Scope

| Does | Doesn't |
|---|---|
| One watertight mesh with a density or known mass | Multi-body assemblies or sub-bodies with their own orientation |
| N point masses at explicit XYZ | Point masses with their own extent |
| Required `mesh_units` (`"mm"`, `"cm"`, `"m"`) | Guess units from the file |
| Effective density through `infill` | Per-region or graded density |
| Raises on open meshes and bad inertia | Repair a broken mesh |
| URDF `<inertial>` export | MJCF, USD, SDF |

Everything is SI on output: kg, m, kg·m².

## API

```python
load_mesh(path, *, mesh_units, density=None, mass=None, infill=1.0, allow_open=False) -> MeshBody
```
Mass properties of one mesh, with `inertia` about the mesh's own COM. You need exactly one of `density` (kg/m³) or `mass` (kg), and `infill` can't be combined with `mass`. A non-watertight mesh raises `ValueError` unless `allow_open=True`, which turns it into a warning.

```python
compose(body: MeshBody, point_masses: Sequence[PointMass]) -> MassProperties
```
Total mass, system COM, and the inertia tensor about that COM. An empty list just returns the mesh's own properties, and the output is checked before it's returned.

```python
to_urdf_inertial(mp: MassProperties) -> str
```
The `<inertial>` block as a string, ready to paste inside a `<link>`. The `i**` attributes are plain matrix entries (`ixy` is `inertia[0, 1]`), so there's no sign flip.

```python
PointMass(position: np.ndarray, mass: float, name: str | None = None)
```
A discrete mass in the mesh's frame. `position` has to be shape `(3,)`, and a `(3, 1)` column gets rejected instead of flattened since a transposed vector gives a wrong but plausible answer.

```python
MeshBody(mass, com, inertia)         # returned by load_mesh
MassProperties(mass, com, inertia)   # returned by compose
```
