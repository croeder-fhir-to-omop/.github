# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).
## DISCLAIMER
The IG includes many precautions for dealing with PII in the data. This code makes no guarantee to do so. Do not use it with PII or PHI.

## Repositories

| Repo | Description |
|---|---|
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | `transforms.py` (FHIR→OMOP via matchbox), `load_duckdb.py` (ETL into OMOP CDM 5.4), and sample FHIR fixtures |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment for interactive FHIR→OMOP exploration |
| [dqd_docker](https://github.com/croeder-fhir-to-omop/dqd_docker) | Runs the ETL then serves the OHDSI Data Quality Dashboard against the resulting OMOP CDM |
| [enchilada](https://github.com/croeder/enchilada) | a local terminology server

## Running

All images are published to Docker Hub. No repo clones are needed to run the pipeline.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose v2)
  - macOS: Apple Silicon or Intel
  - Windows 10 (version 1903 or later) or Windows 11 — Docker Desktop's installer enables WSL2 automatically
  - Linux: Docker Engine + Docker Compose v2
- **RAM**: 8GB minimum (16GB recommended). Docker Desktop's memory limit (Settings → Resources) should be at least 6GB.
- **Disk**: 10GB free space for images and data volumes.
  - macOS: Apple Silicon or Intel
  - Windows 10 (version 1903 or later) or Windows 11 — Docker Desktop's installer enables WSL2 automatically
  - Linux: Docker Engine + Docker Compose v2

> **Note:** Docker Hub shows a "Run in Docker Desktop" button on each image page (`croeder/jupyter`, `croeder/dqd`, `croeder/matchbox`). Avoid it — it starts only that one container in isolation, leaving the other services unreachable. The pipeline requires two services running together with shared networking, which only the compose file provides. Use the `curl` commands below instead.

### Option A — Automated ETL + Data Quality Dashboard

Runs matchbox, transforms all sample FHIR fixtures into OMOP CDM 5.4 (DuckDB), executes OHDSI Data Quality Dashboard checks, and serves two dashboards.

#### Starting

macOS / Linux / Git Bash:
```bash
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml | docker compose -f - up
```

PowerShell (Windows 10/11 — note `curl.exe`, not `curl`):
```powershell
curl.exe -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml | docker compose -f - up
```

Or clone just the one compose file you need:

```bash
git clone https://github.com/croeder-fhir-to-omop/dqd_docker
docker compose -f dqd_docker/docker-compose.yml up
```

| URL | Description |
|---|---|
| http://localhost:3838 | OHDSI Data Quality Dashboard |
| http://localhost:8088/etl_report.html | ETL report (per-fixture status, StructureMap used, root cause of any failures) |

On first run matchbox loads the OMOP IG (~1 min). Subsequent runs use the cached volume.

#### Stopping

```bash
docker compose -f dqd_docker/docker-compose.yml down      # keep data volumes
docker compose -f dqd_docker/docker-compose.yml down -v   # also remove volumes (fresh start)
```

### Option B — Interactive Jupyter Notebooks

Starts matchbox and a Jupyter notebook server with `transforms.py` pre-installed for hands-on FHIR→OMOP exploration.

#### Starting

macOS / Linux / Git Bash:
```bash
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/jupyter_docker/main/docker-compose.yml | docker compose -f - up
```

PowerShell (Windows 10/11 — note `curl.exe`, not `curl`):
```powershell
curl.exe -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/jupyter_docker/main/docker-compose.yml | docker compose -f - up
```

Or clone just the one compose file you need:

```bash
git clone https://github.com/croeder-fhir-to-omop/jupyter_docker
docker compose -f jupyter_docker/docker-compose.yml up
```

Open http://localhost:8888. Inside Jupyter you'll find two folders:

| Folder | Contents |
|---|---|
| `examples/` | Demo notebooks baked into the image — use as reference; changes don't persist |
| `work/` | Your working directory — files saved here persist to `./notebooks/` on the host |

#### Stopping

```bash
docker compose -f jupyter_docker/docker-compose.yml down      # keep data volumes
docker compose -f jupyter_docker/docker-compose.yml down -v   # also remove volumes (fresh start)
```

## Developing matchbox_scripts

To edit transforms, ETL logic, or fixtures and see your changes live, clone `matchbox_scripts` alongside the compose repo and use the dev overlay:

```bash
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts
git clone https://github.com/croeder-fhir-to-omop/jupyter_docker   # or dqd_docker

# Jupyter — scripts are bind-mounted; edits are visible immediately
docker compose -f jupyter_docker/docker-compose.yml \
               -f jupyter_docker/docker-compose.dev.yml up

# DQD — scripts are baked in at build time; --build picks up changes
docker compose -f dqd_docker/docker-compose.yml \
               -f dqd_docker/docker-compose.dev.yml up --build
```

All repos must be cloned **side by side into the same parent directory** when developing.

**Windows:** Use a WSL2 terminal — the `${HOME}` path in the Jupyter dev overlay's volume mount requires it.

### Adding FHIR fixtures

Fixtures are FHIR resource JSON files stored in `matchbox_scripts/`.

#### Existing resource types

Supported types: Condition, Patient, Procedure, AllergyIntolerance, Encounter, Immunization, MedicationStatement, Observation, and vital-sign/weight Observations. Add a JSON file whose name matches an existing glob pattern (e.g. `condition_diabetes.json`, `medication_warfarin.json`) — no code changes needed.

- **Option B / Jupyter**: Drop the file into `matchbox_scripts/` on your host — it appears immediately inside the container. No restart needed. Commit and push to persist for others.
- **Option A / automated ETL**: Add the file to `matchbox_scripts/`, then rebuild: `docker compose -f dqd_docker/docker-compose.yml -f dqd_docker/docker-compose.dev.yml up --build`.

#### New resource types

For resource types not listed above (e.g. Device, Death):

1. Add the FHIR JSON fixture to `matchbox_scripts/`
2. Add a `transform_<type>()` function to `matchbox_scripts/transforms.py`
3. Add a row to `FIXTURE_TRANSFORMS` in `matchbox_scripts/load_duckdb.py` (glob pattern, transform function, OMOP table, StructureMap name)
4. Commit and push, then rebuild for Option A

### Extending or replacing the conversion engine

#### Replacing matchbox

`transforms.py` is the only file that knows about matchbox. Every `transform_*()` function takes a FHIR resource dict and returns either `None` (to suppress the resource) or an OMOP-shaped dict that is itself in FHIR resource form: a `resourceType` field (e.g. `"ConditionOccurrence"`) plus OMOP column names as the remaining keys. That `resourceType` is how `load_duckdb.py` looks up the correct column list in `omop_to_csv.COLUMNS` and knows which fields to extract before inserting into DuckDB.

To swap in a different engine, rewrite `transforms.py` — replace the `_call()` helper to hit a different HTTP endpoint, rewrite individual functions to convert locally, or replace the file entirely. As long as the return value keeps this FHIR-envelope shape, nothing else in the pipeline needs to change.

#### If your engine produces CSV

If a replacement engine produces plain CSV row strings instead of FHIR-shaped dicts, three things break and need updating:

- **`load_duckdb.py`**: the `resourceType` dispatch disappears; key off the table name already present in each `FIXTURE_TRANSFORMS` entry instead
- **`omop_to_csv.COLUMNS`**: currently keyed by PascalCase resourceType (e.g. `"ConditionOccurrence"`); would need rekeying by snake_case table name (e.g. `"condition_occurrence"`)
- **Parsing**: the CSV string would need to be parsed into a column→value dict before the existing `insert()` function can use it

## How it works

1. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 11 StructureMaps
2. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts
3. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample fixtures and writes results into a DuckDB OMOP CDM 5.4 database
4. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
5. **dqd_docker** runs `load_duckdb.py` automatically on startup, then serves the OHDSI Data Quality Dashboard on port 3838

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.
