# pipekit

[![CI](https://github.com/SaElmas/pipekit/actions/workflows/ci.yml/badge.svg)](https://github.com/SaElmas/pipekit/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.12%2B-blue)
![License: MIT](https://img.shields.io/badge/license-MIT-green)

**Pipe routing and cut lists for engineers, in plain Python.**

pipekit takes a pipe route as a list of 3D points, fillets every corner with the pipe's bend radius and gives you a cut list: straight lengths, bend angles, elbow count and total pipe length. It grew out of a Rhino piping toolkit I wrote in 2016 for yacht engine-room outfitting. This version needs no CAD software and is fully typed and tested.

## Features

- **Pipe catalog:** load pipe specs (material, outside diameter, wall thickness, bend radius) from a CSV file, with validation and clear error messages.
- **Route filleting:** replace each sharp corner with a bend of the right radius, and catch routes where two bends would overlap on a short segment.
- **Cut list:** straight lengths between bends, bend angles, elbow count and total length, as a Markdown table or CSV.
- **Command line:** one command from a route file to a cut list.
- **Engineering-grade code:** `mypy --strict`, Ruff, pytest with coverage, and CI on every push.

## Installation

pipekit needs Python 3.12 or newer.

```bash
git clone https://github.com/SaElmas/pipekit.git
cd pipekit
uv sync            # or: python -m venv .venv && pip install -e .
```

## Quick start

Describe a route in millimetres as a JSON file, `route.json`:

```json
{
  "points": [
    [0, 0, 0],
    [2000, 0, 0],
    [2000, 1500, 0],
    [2000, 1500, 1000]
  ]
}
```

Run it with a pipe spec from the catalog:

```bash
pipekit bom route.json --spec SS316-DN50
```

```text
Spec: SS316-DN50  (OD 60.3 mm, wall 2.77 mm, ID 54.76 mm, bend radius 76.2 mm)

| #  | Item     | Length (mm) | Angle (deg) |
|----|----------|-------------|-------------|
| 1  | Straight | 1923.8      |             |
| 2  | Bend     | 119.7       | 90.0        |
| 3  | Straight | 1347.6      |             |
| 4  | Bend     | 119.7       | 90.0        |
| 5  | Straight | 923.8       |             |

Elbows: 2   Total pipe length: 4434.6 mm
```

Or use it from Python:

```python
from pipekit import load_catalog, fillet_route, cut_list

catalog = load_catalog("data/catalog.csv")
spec = catalog["SS316-DN50"]

segments = fillet_route(
    [(0, 0, 0), (2000, 0, 0), (2000, 1500, 0), (2000, 1500, 1000)],
    bend_radius=spec.bend_radius,
)
print(cut_list(segments).to_markdown())
```

## How the fillet works

At each corner the route turns by an angle θ between the incoming and outgoing directions. A bend of radius R replaces the corner:

- **Tangent length** (how much of each straight the bend uses): T = R · tan(θ / 2)
- **Arc length** (pipe length along the bend): L = R · θ

Each straight segment is shortened by the tangent lengths of the bends at both of its ends. If those add up to more than the segment's length, the bends would overlap, and pipekit raises an error that names the segment.

For the example above, R = 76.2 mm and θ = 90°, so T = 76.2 mm and L = 119.7 mm per bend.

## Catalog format

`data/catalog.csv`, one pipe spec per row, all dimensions in millimetres:

```csv
code,material,outside_diameter,wall,bend_radius
SS316-DN50,SS316,60.3,2.77,76.2
CPVC-DN40,CPVC,48.3,3.68,72.5
```

Supported materials: SS316, CPVC, PPR, CuNi, HDPE, GRP.

## Project structure

```text
pipekit/
├── src/pipekit/
│   ├── catalog.py     # Material, PipeSpec, load_catalog
│   ├── geometry.py    # Vec3, fillet_route
│   ├── bom.py         # cut list, Markdown and CSV output
│   └── cli.py         # the pipekit command
├── tests/             # pytest suite
├── data/catalog.csv   # sample pipe specs
└── pyproject.toml
```

## Development

```bash
uv sync
uv run pytest --cov=pipekit     # tests with coverage
uv run mypy --strict src        # type checking
uv run ruff check . && uv run ruff format .
```

Pre-commit hooks run Ruff on every commit, and GitHub Actions runs the full suite on every push and pull request.

## Roadmap

- [x] Catalog, route filleting, cut list, command line
- [ ] Fittings library (tees, reducers, flanges) in the cut list
- [ ] Weight and cost estimate per material
- [ ] Export the filleted route as DXF or STEP for CAD
- [ ] A Rhino and Grasshopper plugin built on pipekit

## About

Built by [Sait Elmas](https://saitelmas.com), a naval architect turned software engineer, with degrees in Computer Engineering and Mathematics from METU. [LinkedIn](https://linkedin.com/in/saitelmas)

## License

MIT. See [LICENSE](LICENSE).
