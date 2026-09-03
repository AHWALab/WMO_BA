# EF5 — Barbados Training (standalone Docker workspace)

Standalone EF5 setup for Barbados at **30 m only**. It runs the flood model
through a Docker container while **TITO Barbados training stays untouched**.
The EF5 image is the same `ef5-container` used by TITO; only the run layout
differs.

There is **one resolution (30 m)** and **one control file**:
`conf/control_30m.txt`.

This package does **not** ship `docker/ef5-container.tar`. Build the image
locally (see below).

## Folder layout

```text
EF5_BarbadosTraining/
├── data/                        → mounted as /data  (model inputs, read/write)
│   ├── basic/                   DEM, flow direction (DDM), flow accumulation (FAM)
│   │   └── DEM_barbados_30m.tif / FDIR_barbados_30m.tif / FAC_barbados_30m.tif
│   ├── parameters/
│   │   ├── CREST_Barbados_30m/
│   │   └── KW_Barbados_30m/
│   ├── pet/                     PET.01.tif … PET.12.tif climatology
│   ├── states/
│   │   └── 30m/                 warm-start states for 30 m (crest_SM, kwr_*)
│   └── precip/                  IMERG forcing: imerg.qpe.YYYYMMDDHHUU.30minAccum.tif
├── output/                      → mounted as /output (EF5 results)
│   └── 30m/                     results — 30 m domain run
├── conf/                        → mounted as /conf  (control files, read-only)
│   ├── control_30m.txt          the only control file — Barbados 30 m
│   └── basin_list/
│       └── Barbados_30m_basin_new.txt
├── docker/
│   ├── Dockerfile               builds ef5-container:latest from source (AHWALab/EF5)
│   ├── build_ef5.sh             build/reuse (Linux & macOS)
│   └── build_ef5.cmd            build/reuse (Windows CMD — no PowerShell)
├── docker-compose.yml           cross-platform launcher (works on all 3 OSes)
├── run_ef5.sh                   run EF5 (Linux / macOS / WSL)
├── run_ef5.cmd                  run EF5 (Windows CMD — no PowerShell)
├── README_english.md
└── README.md
```

## How the container accesses the folders

`run_ef5.sh` / `run_ef5.cmd` / `docker-compose.yml` bind-mount the three
folders into the container and run EF5 from the container root, so every path
in the control file is relative to `/`:

| Host folder | Container path | Used for                                         |
| ----------- | -------------- | ------------------------------------------------ |
| `./data`    | `/data`        | basic, parameters, pet, states, precip           |
| `./output`  | `/output`      | EF5 outputs (maxq/maxunitq/ts.\*.tif, logs, csv) |
| `./conf`    | `/conf`        | EF5 control files                                |

## Build the image

Build `ef5-container:latest` from `docker/Dockerfile` (clones AHWALab/EF5 and
compiles; needs internet; a few minutes). Optionally save your own archive
afterward.

**Linux / macOS** — `docker/build_ef5.sh`:

```bash
./docker/build_ef5.sh --rebuild       # compile from source (needs internet)
./docker/build_ef5.sh --status        # what will be used
./docker/build_ef5.sh --save          # optional: snapshot current image → docker/ef5-container.tar
```

**Windows (Command Prompt)** — pure CMD, no PowerShell:

```bat
docker\build_ef5.cmd -Rebuild
docker\build_ef5.cmd -Status
docker\build_ef5.cmd -Save
```

Reuse order after you have built once: already-loaded local image → optional
`docker/ef5-container.tar` if you created one → build from `docker/Dockerfile`.

## Run EF5

There is a single control file. If none is passed, the default is
`conf/control_30m.txt`.

**Linux / macOS / WSL:**

```bash
./run_ef5.sh                            # 30 m  → output/30m/
./run_ef5.sh conf/control_30m.txt       # same, explicit control file
./run_ef5.sh --bash                     # interactive shell in the container
```

**Windows (Command Prompt)** — prefer a **local** path (e.g. `C:\...`), not a mapped network drive:

```bat
run_ef5.cmd
run_ef5.cmd -Control control_30m.txt
run_ef5.cmd -Bash
```

| Platform            | Example                                                           |
| ------------------- | ----------------------------------------------------------------- |
| Linux / WSL / macOS | `./run_ef5.sh conf/control_30m.txt`                               |
| Windows (CMD)       | `run_ef5.cmd -Control control_30m.txt`                            |
| Any OS              | `docker compose run --rm ef5 /ef5/bin/ef5 /conf/control_30m.txt`  |

macOS (Docker Desktop) has no host networking, so `run_ef5.sh` automatically
delegates to `docker compose` there.

## Control file and outputs

| Control | Resolution | Purpose | Output folder | States |
| ------- | ---------- | ------- | ------------- | ------ |
| `conf/control_30m.txt` | 30 m | Barbados domain | `./output/30m/` | `data/states/30m/` |

Basin / gauge source list (reference) lives under
`conf/basin_list/Barbados_30m_basin_new.txt`.

Includes `maxq` / `maxunitq` / precip-accum grids (and soil moisture where
enabled).

Populate `data/precip/` with IMERG GeoTIFFs named
`imerg.qpe.YYYYMMDDHHUU.30minAccum.tif` before a run with precipitation;
missing files are treated as zero precipitation.

The example window in `conf/control_30m.txt` is `TIME_BEGIN=202608140800` to
`TIME_END=202608180000`. Edit those lines for a different simulation period.

## Windows notes

- Use **Command Prompt** with `run_ef5.cmd` / `docker\build_ef5.cmd` (no PowerShell scripts).
- **Docker bind mounts** from mapped network drives (`X:`) often fail or appear
  empty inside the container. Copy the repo to a **local** folder first:

```bat
xcopy /E /I X:\WMO_BA C:\EF5_BarbadosTraining
cd /d C:\EF5_BarbadosTraining
docker\build_ef5.cmd -Rebuild
run_ef5.cmd -Control control_30m.txt
```

- Direct Docker (no launchers):

```bat
docker compose build
docker compose run --rm ef5 /ef5/bin/ef5 /conf/control_30m.txt
```
