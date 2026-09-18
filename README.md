# TITO — Barbados training (30 m)

**Threading Inputs to Outputs (TITO)** is AHWA Lab's framework for running the **EF5** hydrologic model with satellite QPE, ensemble nowcast/QPF products, scenario library flood inundation mapping (FIM) and impact based forecasting (IBF).

This tree is the **Barbados training package**: region key `Barbados`, **30 m only**. FIM is 30 m (11 parish units). StormLab domain is `barbados`.

Partners need **either Docker or Apptainer/Singularity, not both**.

## What this package does

| Piece | Role |
|--------|------|
| **EF5** | Distributed hydrologic model (CREST + KW) at **30 m**. |
| **IMERG** | Deterministic satellite QPE for the training hindcast. |
| **StormLab** | Ensemble precipitation (`barbados` domain). Run as **QPE** (no EF5 long-range). Training pack: 50 members. |
| **FIM** | Scenario-library inundation **after the forecast phase**, **30 m only**. 11 ADM1 parishes. |
| **IBF** | Receptor products chained after FIM (`ibf_regions`). |

**QPE-only EF5 chain (no long-range block):**

| Mode | Phase A | Phase B | Phase C |
|------|---------|---------|---------|
| **Hindcast IMERG + StormLab (offline only)** | IMERG QPE + dry | skipped | StormLab as QPE |
| **Ops STREAM-Sat + StormLab/AROME** | STREAM-Sat QPE + dry | SCaMPR gap | StormLab **or** AROME as QPE |
| **Ops IMERG + StormLab/AROME** | IMERG QPE + dry @ T−4 h | SCaMPR gap | StormLab **or** AROME as QPE |

**Hindcast is offline only.** The training event is Hurricane Tomas (**2010-10-30 00:00 UTC**) — too old for live downloads. Always pass `--offline`. There is no online hindcast command.

AROME has **no hindcast archive**. If `qpf_source` includes AROME, hindcast **stops**.

Lists in `region_forcing_map` are a Cartesian product (ops), each pair with SCaMPR gap-fill.

---

## Quick start

1. Install **Docker Desktop** or Docker Engine / **Apptainer**.
2. Load images once (see below).
3. Extract FIM stores once: `python fim_store/unzip_stores.py Barbados`
4. Run the **offline** hindcast (Hurricane Tomas):

```sh
# Linux / macOS / Git Bash / WSL
./tito-run.sh hindcast "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline
```

```bat
REM Windows CMD
tito-run.cmd hindcast "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline
```

```sh
# HPC Apptainer + offline precip
TITO_RUNTIME=apptainer ./tito-run.sh hindcast \
    "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline
```

`--regions Barbados` is the region key. `--offline` is **required** for this hindcast (old event; no network downloads).

---

## Loading Docker images

```text
dist/docker-archives/tito_latest.tar.gz
dist/docker-archives/ef5-container_latest.tar.gz
```

| Platform | Load once | Then run |
|----------|-----------|----------|
| **Linux / macOS** | `./load-docker-images.sh` or `./tito-run.sh load-images` | `./tito-run.sh hindcast "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline` |
| **Windows (CMD)** | `load-docker-images.cmd` or `tito-run.cmd load-images` | `tito-run.cmd hindcast "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline` |

```sh
docker images tito
docker images ef5-container
```

Expect `tito:latest` and `ef5-container:latest`.

---

## Offline mode

Classroom / no-network. **Required** for this hindcast — the 2010 event cannot be fetched live. Only with `--offline`. Allowed cycle: **2010-10-30 00:00 UTC**.

```sh
./tito-run.sh hindcast "2010-10-30 00:00" "2010-10-30 00:00" --regions Barbados --offline
```

IMERG QPE from warmup state `20101028_0800` → `20101030_0000`, then StormLab QPE (50 members).

Archive:

```text
offline_precips/
  imerg/*.tif
  imerg/_shared/201010300000/
  stormlab/barbados/ensQ1..50/stormlab.YYYYMMDDHHMM.tif
```

Refresh after a good run: `bash offline/materialize_offline_precips.sh`  
More: [offline/README.md](offline/README.md).

---

## Folder structure

```text
TITO_Barbados_Training/
  Caribbean_Comoros_config.py
  orchestrator.py  hindcast_manager.py
  tito-run.sh / tito-run.cmd
  tito_utils/
  EF5_conf/
    basic/          DEM/FAC/FDIR barbados_30m
    parameters/     CREST_Barbados_30m  KW_Barbados_30m
    templates/      ef5_Barbados_30m_control_template.txt
    states/  precip/  precipEF5/  qpf_store/
  outputs/<cycle>/barbados_30m/<product>/
  offline/  offline_precips/
  fim_config/  fim_store/Barbados/
```

### Cycle-first outputs

```text
outputs/20101030.000000/barbados_30m/
  imerg/
  stormlab/ensOut1_sl1/
  fim/imerg_stormlab/
  ibf/<Site>/
```

---

## Config highlights (`Caribbean_Comoros_config.py`)

```python
region_resolution_map = {"Barbados": "30m"}
regions_to_run = ["Barbados"]

region_forcing_map = {
    "Barbados": {"qpe_source": "IMERG", "qpf_source": "STORMLAB"},
}

HindCastDate = "2010-10-30 00:00"
HindCastEndDate = "2010-10-30 00:00"
qpe_gap_fill_mode = "IMERG_ONLY"  # hindcast: no SCaMPR gap

stormlab_ensemble_size = 50
ef5_max_workers = 1

fim_enabled = True          # 30 m after forecast
fim_regions = {"Barbados": {"enabled": True, "thresholds_m": [0.10, 0.30, 0.70, 1.00]}}
ibf_enabled = True
ibf_regions = {"Barbados": {"enabled": True, ...}}
```

Control template: `EF5_conf/templates/ef5_Barbados_30m_control_template.txt`

FIM stores: `python fim_store/unzip_stores.py Barbados` after clone. See [README_FIM.md](README_FIM.md).

---

## Reset after a crash

| OS | Command |
|----|---------|
| Linux / macOS | `./reset_tito.sh` |
| Preview | `./reset_tito.sh --dry-run` |
| Windows CMD | `reset_tito.cmd` |

Wipes `outputs/`, live precip, STREAM-Sat / StormLab runtime output. Does **not** wipe `offline_precips/`.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `docker load` 502 | Docker Desktop not ready |
| EF5 exit **137** | OOM: lower ensembles, `ef5_max_workers=1`, raise Docker RAM (~0.9 GB per concurrent member) |
| FIM `no_runs` | Check `outputs/<cycle>/barbados_30m/stormlab/` vs chain tag |
| FIM skip 90 m | This package is **30 m only** |
| Offline refused cycle | Only 2010-10-30 00:00 UTC |
| Hindcast without `--offline` | Not supported — Tomas is too old; always use `--offline` |
| Hindcast + AROME | Error: AROME has no archive — use STORMLAB or GFS |

---

## Contact

Naman Mehta - naman-mehta@uiowa.edu  
Vanessa Robledo - vanessa-robledodelgado@uiowa.edu  
AHWA Laboratory - [ahwa.lab.uiowa.edu](https://ahwa.lab.uiowa.edu/) - engr-ahwa-lab@uiowa.edu

## Cite

Robledo Delgado, V., & Vergara, H. (2025). Threading Inputs to Outputs (TITO) (v2.0.0). Zenodo. https://doi.org/10.5281/zenodo.17246491
