# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).
## DISCLAIMER
The IG includes many precautions for dealing with PII in the data. This code makes no guarantee to do so. Do not use it with PII or PHI.

## Repositories

| Repo | Description |
|---|---|
| [fhir-omop-ig](https://github.com/croeder-fhir-to-omop/fhir-omop-ig) | HL7 FHIR-to-OMOP Implementation Guide — StructureMaps and ConceptMaps |
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | `transforms.py` (FHIR→OMOP via matchbox), `load_duckdb.py` (ETL into OMOP CDM 5.4), and sample FHIR fixtures |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment for interactive FHIR→OMOP exploration |
| [dqd_docker](https://github.com/croeder-fhir-to-omop/dqd_docker) | Runs the ETL then serves the OHDSI Data Quality Dashboard against the resulting OMOP CDM |
| [enchilada](https://github.com/croeder/enchilada) | Local OMOP-backed FHIR terminology server |

## FHIR version

This setup tests the IG against **FHIR R5** data. The tests here assume R5. matchbox calls the terminology server via R4 endpoints.

A profiles-based compose setup (`dqd_docker/docker-compose.profiles.yml`) exists for running multiple stacks in parallel and switching between FHIR R4 and R5 or between IG versions — see the `dqd_docker` repo for details.

## Running

All images are published to Docker Hub. No repo clones are needed to run the pipeline.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose v2)
  - macOS: Apple Silicon or Intel
  - Windows 10 (version 1903 or later) or Windows 11 — Docker Desktop's installer enables WSL2 automatically
  - Linux: Docker Engine + Docker Compose v2
- **RAM**: 16GB minimum. Docker Desktop's memory limit (Settings → Resources) should be at least 12GB.
- **Disk**: 10GB free space for images and data volumes.

> **Note:** Docker Hub shows a "Run in Docker Desktop" button on each image page (`croeder/jupyter`, `croeder/dqd`, `croeder/matchbox`). Avoid it — it starts only that one container in isolation, leaving the other services unreachable. The pipeline requires two services running together with shared networking, which only the compose file provides. Use the `curl` commands below instead.

### Option A — Automated ETL + Data Quality Dashboard

Runs enchilada (local terminology server), matchbox (FHIR→OMOP transforms), ETL, OHDSI Data Quality Dashboard checks, and unit tests. Serves dashboards and reports.

#### Prerequisites: vocabulary files

enchilada needs two vocabulary files from [Athena](https://athena.ohdsi.org):

1. Go to https://athena.ohdsi.org and create a free account
2. Click **Download** and select a vocabulary bundle. The sample fixtures use these vocabularies:

| Vocabulary | Athena name | Used for |
|---|---|---|
| SNOMED CT | SNOMED | Conditions, procedures, observations |
| LOINC | LOINC | Lab results, vitals, blood pressure panels |
| RxNorm | RxNorm | Medications |
| ICD-10-CM | ICD10CM | Diagnoses |
| CVX | CVX | Immunizations / vaccines |
| UCUM | UCUM | Units of measure |
| CDC Race & Ethnicity | Race | Patient race and ethnicity |

3. Download and extract — you need `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv`

Place both files in your working directory before starting. They can be several GB; the download may take a few minutes.

> If you start without the vocabulary files, enchilada will warn but the stack will still come up. Terminology lookups will return no results until you restart with the files present.

#### Starting

macOS / Linux / Git Bash:
```bash
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml | docker compose -f - up
```

PowerShell (Windows 10/11 — note `curl.exe`, not `curl`):
```powershell
curl.exe -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml | docker compose -f - up
```

If your vocabulary files are not in the current directory:
```bash
CONCEPT_CSV=/path/to/CONCEPT.csv \
CONCEPT_RELATIONSHIP_CSV=/path/to/CONCEPT_RELATIONSHIP.csv \
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml | docker compose -f - up
```

Or download the compose file first if you want to keep or edit it:

```bash
curl -O https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml
docker compose up
```

| URL | Description |
|---|---|
| http://localhost:3838 | OHDSI Data Quality Dashboard |
| http://localhost:8088 | ETL reports and unit test report |

The port 8088 index links to three reports:

- **ETL report — test files**: results from `matchbox_scripts/test_files_r5/`, a set of targeted FHIR fixtures designed to exercise specific StructureMap behaviors and edge cases
- **ETL report — sample fixtures**: results from `matchbox_scripts/sample_fixtures_r5/`, a broader set of representative patient data
- **Unit test report**: results from `matchbox_scripts/tests/test_r5_fml_transforms.py`, a pytest suite that calls matchbox's `$transform` endpoint directly and asserts specific OMOP field values — these are the primary correctness tests for the StructureMap implementations

On first run, enchilada loads the vocabulary CSVs (~1–2 min) and matchbox loads the OMOP IG (~1 min). Both are cached in Docker volumes and skipped on subsequent starts.

#### Terminology server

Two terminology servers are supported:

**enchilada** — a local OMOP-backed terminology server included in `docker-compose.yml`. Requires `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv` from Athena (see above). You can also supply supplemental concept files (`concept_extra.tsv`, `concept_relationship_extra.tsv`, `vocabulary_extra.tsv`) to add concept mappings for FHIR-specific code systems not present in Athena. These are optional; enchilada starts without them. This is the default.

**echidna** — the public [echidna.fhir.org](https://echidna.fhir.org) terminology service. Available free of charge with rate limits; no local vocabulary files required. API key authentication for higher limits is not currently supported in the matchbox txServer code path.

To use echidna, set the terminology server URL when starting:

```bash
MATCHBOX_FHIR_CONTEXT_TXSERVER=https://echidna.fhir.org/r4 docker compose up
```

Or add a `.env` file alongside your compose file:
```
MATCHBOX_FHIR_CONTEXT_TXSERVER=https://echidna.fhir.org/r4
```

When using echidna, enchilada still starts (it is a healthcheck dependency) but matchbox routes all terminology lookups to echidna.

#### Stopping

```bash
docker compose down      # keep volumes (fast restart — vocabulary cache preserved)
docker compose down -v   # also remove volumes (full reload on next start)
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
docker compose down      # keep data volumes
docker compose down -v   # also remove volumes (fresh start)
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


### Working with the Implementation Guide

The `croeder/matchbox:latest` image has `hl7.fhir.uv.omop#1.0.0` baked in. If you are working on the IG itself — editing StructureMaps or ConceptMaps in `fhir-omop-ig/` — you need to build a new IG package and get it into matchbox. There are two paths depending on whether you want a quick local test or a published image.

#### Step 1 — Build the IG

The IG is built with the [bonfhir ig-toolbox](https://github.com/bonfhir/ig-toolbox) Docker container. Clone `fhir-omop-ig` alongside your other repos, then:

```bash
git clone https://github.com/croeder-fhir-to-omop/fhir-omop-ig
cd fhir-omop-ig
docker run --rm -v "$(pwd):/workspace" ghcr.io/bonfhir/ig-toolbox:latest build
```

This produces a package tarball at `fhir-omop-ig/output/package.tgz`. Copy it into `matchbox_docker/igs/` with the versioned name matchbox expects:

```bash
cp fhir-omop-ig/output/package.tgz matchbox_docker/igs/hl7.fhir.uv.omop-<version>.tgz
```

#### Path A — Local run (no image rebuild)

The published matchbox image loads configs in order:

```
/defaults/application.yaml  →  /config/application.yaml  (optional override)
```

Mount your new IG package and a config override into the running container — no rebuild needed.

**1. Write `config/application.yaml`** in your working directory:

```yaml
hapi:
  fhir:
    implementationguides:
      fhiromop:
        name: hl7.fhir.uv.omop
        version: <version>
        url: file:///igs/hl7.fhir.uv.omop-<version>.tgz

matchbox:
  fhir:
    context:
      igsPreloaded:
        - hl7.fhir.uv.omop#<version>
```

**2. Add volume mounts** to your `docker-compose.yml` under the `matchbox` service:

```yaml
volumes:
  - matchbox-db:/database
  - ./matchbox_docker/igs:/igs:ro
  - ./config:/config:ro
```

**3. Start** (use `down -v` first to clear the cached H2 database so matchbox reloads the IG from scratch):

```bash
docker compose down -v
docker compose up
```

#### Path B — Bake into the image and publish

Clone `matchbox_docker` alongside your other repos, place your built IG package in `matchbox_docker/igs/`, then use the build script:

```bash
git clone https://github.com/croeder-fhir-to-omop/matchbox_docker
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts

# Build and publish the matchbox image
cd matchbox_scripts
python3 build.py ig mvn docker release
```

This builds the IG, builds the matchbox JAR, bakes a new `croeder/matchbox:latest` image, and pushes it to Docker Hub. Anyone who then does `docker compose pull` or runs the curl command fresh gets the updated IG.


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

1. **enchilada** serves OMOP vocabulary lookups over HTTPS using local Athena CSV files, acting as a FHIR terminology server for matchbox
2. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 13 StructureMaps
3. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts
4. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample fixtures and writes results into a DuckDB OMOP CDM 5.4 database
5. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
6. **dqd_docker** runs `load_duckdb.py` automatically on startup, runs OHDSI Data Quality Dashboard checks and unit tests, and serves results on port 3838 (DQD) and port 8088 (ETL reports, unit test report)

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.
