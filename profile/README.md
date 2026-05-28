# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).

## Repositories

| Repo | Description |
|---|---|
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration and IGs for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | `transforms.py` (FHIR→OMOP via matchbox), `load_duckdb.py` (ETL into OMOP CDM 5.4), and sample FHIR fixtures |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment for interactive FHIR→OMOP exploration |
| [dqd_docker](https://github.com/croeder-fhir-to-omop/dqd_docker) | Runs the ETL then serves the OHDSI Data Quality Dashboard against the resulting OMOP CDM |

## Running

All compose files use `..` as their build context, so repos must be cloned **side by side into the same parent directory**.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose v2)
- Git

### Clone

```bash
git clone https://github.com/croeder-fhir-to-omop/matchbox_docker
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts
git clone https://github.com/croeder-fhir-to-omop/dqd_docker
git clone https://github.com/croeder-fhir-to-omop/jupyter_docker
```

### Option A — Automated ETL + Data Quality Dashboard

Runs matchbox, transforms all sample FHIR fixtures into OMOP CDM 5.4 (DuckDB), executes OHDSI Data Quality Dashboard checks, and serves two dashboards.

```bash
docker compose -f dqd_docker/docker-compose.yml up
```

| URL | Description |
|---|---|
| http://localhost:3838 | OHDSI Data Quality Dashboard |
| http://localhost:8088/etl_report.html | ETL report (per-fixture status, StructureMap used, root cause of any failures) |

On first run matchbox downloads the OMOP IG (~1 min). Subsequent runs use the cached volume.

### Option B — Interactive Jupyter Notebooks

Starts matchbox and a Jupyter notebook server with `transforms.py` pre-installed for hands-on FHIR→OMOP exploration.

```bash
docker compose -f jupyter_docker/docker-compose.yml up
```

Open http://localhost:8888 and navigate to `matchbox_scripts/notebooks/matchbox_demo.ipynb`.

### Adding your own FHIR fixtures

Fixtures are FHIR resource JSON files. There are two cases depending on whether the resource type is already supported.

**Supported resource types** (Condition, Patient, Procedure, AllergyIntolerance, Encounter, Immunization, MedicationStatement, Observation, and vital-sign/weight Observations) — just add a JSON file with a matching name pattern (e.g. `condition_myfix.json`, `medication_warfarin.json`) to `matchbox_scripts/`:

- *Option B / Jupyter*: Drop the file into `matchbox_scripts/` on your host — it appears immediately inside the container (the directory is a live bind-mount). You can also upload it via the Jupyter file browser, or skip the file entirely and pass a Python dict directly to `transform_*()` in a notebook cell. No restart needed. To persist the fixture for others, commit and push it to `matchbox_scripts`.
- *Option A / automated ETL*: Add the file to `matchbox_scripts/`, then rebuild and restart: `docker compose -f dqd_docker/docker-compose.yml up --build`. The fixtures are copied into the image at build time, so a rebuild is required.

**Replacing matchbox with a different conversion engine** — `transforms.py` is the only file that knows about matchbox. Every `transform_*()` function follows the same contract: it takes a FHIR resource dict and returns either an OMOP-shaped dict (with a `resourceType` key matching the entries in `omop_to_csv.py`) or `None` to suppress the resource. To swap in a different engine, rewrite `transforms.py` — replace the `_call()` helper to hit a different HTTP endpoint, rewrite individual functions to convert locally, or replace the file entirely. Nothing else in the pipeline (`load_duckdb.py`, the notebooks) needs to change.

**New resource types** (e.g. Device, Death, or any FHIR resource not listed above) — requires code changes:

1. Add the FHIR JSON fixture to `matchbox_scripts/`
2. Add a `transform_<type>()` function to `matchbox_scripts/transforms.py`
3. Add a row to `FIXTURE_TRANSFORMS` in `matchbox_scripts/load_duckdb.py` (glob pattern, transform function, OMOP table, StructureMap name)
4. Commit and push, then rebuild for Option A

### Stopping

```bash
docker compose -f dqd_docker/docker-compose.yml down       # keep data volumes
docker compose -f dqd_docker/docker-compose.yml down -v    # also remove volumes (fresh start)
```

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.

## How it works

1. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 11 StructureMaps
2. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts
3. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample fixtures and writes results into a DuckDB OMOP CDM 5.4 database
4. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
5. **dqd_docker** runs `load_duckdb.py` automatically on startup, then serves the OHDSI Data Quality Dashboard on port 3838
