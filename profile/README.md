# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).

Conversion operates at the level of individual FHIR resources. FHIR Bundles are not handled; resources must be submitted individually. Most resource types produce a single OMOP record, though some (such as blood pressure panels) are applied through multiple StructureMaps and produce multiple records.

## DISCLAIMER
The IG includes many precautions for dealing with PII in the data. This code makes no guarantee to do so. Do not use it with PII or PHI.

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.

This project uses clinical terminology content from LOINC (Regenstrief Institute), SNOMED CT (SNOMED International), HL7 International, UCUM (Regenstrief Institute), CVX (CDC/NLM), ICD-10-CM (CDC/NCHS), RxNorm (NLM), and OMOP Standardized Vocabularies (OHDSI/Athena). Each carries its own license terms. See [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md) for details.

> **⚠ SNOMED CT users outside the United States:** SNOMED CT is not freely available in all countries. Users in SNOMED International member countries (including Australia, Canada, the Netherlands, and Poland) may access it through their national member organization at no cost. Users in non-member countries must obtain a commercial license from SNOMED International before using this project with SNOMED CT content. See [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md) for country-specific licensing links.

## Repositories

| Repo | Description |
|---|---|
| [fhir-omop-ig](https://github.com/croeder-fhir-to-omop/fhir-omop-ig) | HL7 FHIR-to-OMOP Implementation Guide — StructureMaps and ConceptMaps |
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | `transforms.py` (FHIR→OMOP via matchbox), `load_duckdb.py` (ETL into OMOP CDM 5.4), and sample FHIR fixtures |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment for interactive FHIR→OMOP exploration |
| [dqd_docker](https://github.com/croeder-fhir-to-omop/dqd_docker) | Runs the ETL then serves the OHDSI Data Quality Dashboard against the resulting OMOP CDM |
| [enchilada](https://github.com/croeder-fhir-to-omop/enchilada) | Local OMOP-backed FHIR terminology server |

## Docker Images

The matchbox image is published to Docker Hub under `croeder/matchbox`. Tags reflect which IG source was used at build time:

| Tag | IG Source |
|---|---|
| `croeder/matchbox:latest` | [`HL7/fhir-omop-ig`](https://github.com/HL7/fhir-omop-ig) `main` — the official upstream release |
| `croeder/matchbox:main` | [`croeder-fhir-to-omop/fhir-omop-ig`](https://github.com/croeder-fhir-to-omop/fhir-omop-ig) `main` — this organization's fork |
| `croeder/matchbox:<branch>` | `croeder-fhir-to-omop/fhir-omop-ig` branch `<branch>` |

The compose files use `croeder/matchbox:latest` by default. To run with the fork's main instead:

```bash
export MATCHBOX_IMAGE=croeder/matchbox:main
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml \
  | docker compose -f - up
```

To use a different tag when you have the compose file locally, set `MATCHBOX_IMAGE` in a `.env` file alongside it.

To see which IG source and commit a local or pulled image was built from:

```bash
docker inspect croeder/matchbox:latest | python3 -c "
import json, sys
labels = json.load(sys.stdin)[0]['Config']['Labels']
for k, v in labels.items():
    if k.startswith('fhir-omop-ig'):
        print(f'{k}: {v}')
"
```

```
fhir-omop-ig.source:     upstream
fhir-omop-ig.commit:     1ec215b3f...
fhir-omop-ig.build-date: 2026-06-23T19:05:00Z
```

## FHIR, OMOP, and IG Versions

| Component | Version |
|---|---|
| FHIR | R5 |
| OMOP CDM | 5.4 |
| HL7 FHIR-to-OMOP IG | 1.0.0 |

The tests here assume FHIR R5. matchbox calls the terminology server via R4 endpoints.

A profiles-based compose setup (`dqd_docker/docker-compose.profiles.yml`) exists for running multiple stacks in parallel and switching between FHIR R4 and R5 or between IG versions — see the `dqd_docker` repo for details.

## Vocabularies

Vocabulary use falls into two categories: code systems translated via ConceptMaps embedded in StructureMaps, and code systems looked up dynamically through a terminology server at runtime. Calls to the function translate() in the structure maps indicate which method to use by the inclusion of a concept map name. If the name is present, it is used. If it is an empty string, the configured server is used. ConceptMaps are used for HL7 terminologies as a distribution mechanism. If a configured server included these vocabularies, the concept maps would be redundant.

### ConceptMap translations (StructureMap-embedded)

These code systems are mapped to OMOP concept IDs by ConceptMaps referenced directly in StructureMaps:

| Source Vocabulary | Code System | StructureMap |
|---|---|---|
| FHIR Administrative Gender | `http://hl7.org/fhir/administrative-gender` | PersonMap |
| HL7 v3 ActCode (encounter class) | `http://terminology.hl7.org/CodeSystem/v3-ActCode` | EncounterVisit |
| Admit Source | `http://terminology.hl7.org/CodeSystem/admit-source` | EncounterVisit |
| Discharge Disposition | `http://terminology.hl7.org/CodeSystem/discharge-disposition` | EncounterVisit |
| Condition Clinical Status | `http://terminology.hl7.org/CodeSystem/condition-clinical` | ConditionMap |
| Allergy Intolerance Category | `http://hl7.org/fhir/allergy-intolerance-category` | Allergy |
| v3 RouteOfAdministration | `http://terminology.hl7.org/CodeSystem/v3-RouteOfAdministration` | ImmunizationMap |
| Immunization Origin | `http://terminology.hl7.org/CodeSystem/immunization-origin` | ImmunizationMap |

All ConceptMaps translate to OMOP concept IDs using the code system reference `https://fhir-terminology.ohdsi.org`.

### Terminology server lookups

Clinical code systems are resolved at runtime by calling a terminology server.

#### Athena vocabularies 

| Vocabulary | Athena name | Used for |
|---|---|---|
| SNOMED CT | SNOMED | Conditions, procedures, observations, allergy substances |
| LOINC | LOINC | Lab results, vitals, blood pressure panels |
| RxNorm | RxNorm | Medications, allergy substances |
| ICD-10-CM | ICD10CM | Diagnoses |
| CVX | CVX | Immunization vaccines |
| UCUM | UCUM | Units of measure |
| CDC Race & Ethnicity | Race | Patient race and ethnicity |

Clinical terminologies can be downloaded from [Athena](https://athena.ohdsi.org). The local server, enchilada, requires this download as an independent step.

#### HL7/FHIR-defined code systems

The ConceptMap translations above use HL7 and FHIR-defined code systems (administrative-gender, v3-ActCode, condition-clinical, etc.) that may not be available from terminology servers loaded only with Athena vocabularies.

- Local terminology server enchilada can support additional vocabularies via supplemental concept files (`concept_extra.tsv`, `concept_relationship_extra.tsv`, `vocabulary_extra.tsv`) placed in the working directory before startup.

- Hosted or cloud servers may also support additional vocabularies.

## Running

All images are published to Docker Hub. No git repository clones are needed to run the pipeline.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose v2)
  - macOS: Apple Silicon or Intel
  - Windows 10 (version 1903 or later) or Windows 11 — Docker Desktop's installer enables WSL2 automatically
  - Linux: Docker Engine + Docker Compose v2
- **RAM**: 16GB minimum. Docker Desktop's memory limit (Settings → Resources) should be at least 12GB.
- **Disk**: 10GB free space for images and data volumes.

> **Note:** Docker Hub shows a "Run in Docker Desktop" button on each image page (`croeder/enchilada`, `croeder/matchbox`, `croeder/dqd`, `croeder/jupyter`). Avoid it — it starts only that one container in isolation, leaving the other services unreachable. The pipeline requires two services running together with shared networking, which only the compose file provides. Use the `curl` commands below instead.

### Option A — Automated ETL + Data Quality Dashboard

Runs enchilada (local terminology server), matchbox (FHIR→OMOP transforms), ETL, OHDSI Data Quality Dashboard checks, and unit tests. Serves dashboards and reports.

#### Prerequisites: vocabulary files

enchilada needs two vocabulary files from [Athena](https://athena.ohdsi.org). See the [Vocabularies](#vocabularies) section above for the full list of required vocabulary bundles.

1. Go to https://athena.ohdsi.org and create a free account
2. Click **Download** and select the vocabulary bundles listed in the Vocabularies section
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

- **ETL report — test files**: results from `matchbox_scripts/test_files_r5/`, simple single-resource files (one per scenario) covering the happy path and basic edge cases for each StructureMap — no patient linking between files
- **ETL report — sample fixtures**: results from `matchbox_scripts/sample_fixtures_r5/`, a patient-centric set with four synthetic patients (p1–p4) whose encounters, conditions, observations, and medications cross-reference each other; includes explicit negative cases (files suffixed `_NEG`) and issue-tracked edge cases (files referencing `f2o-xxx` issue numbers)
- **Unit test report**: results from `matchbox_scripts/tests/test_r5_fml_transforms.py`, a pytest suite that calls matchbox's `$transform` endpoint directly and asserts specific OMOP field values — these are the primary correctness tests for the StructureMap implementations

On first run, enchilada loads the vocabulary CSVs (~1–2 min) and matchbox loads the OMOP IG (~1 min). Both are cached in Docker volumes and skipped on subsequent starts.

#### Terminology server

**enchilada** — a local OMOP-backed terminology server included in `docker-compose.yml`. Requires `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv` from Athena (see above). You can also supply supplemental concept files (`concept_extra.tsv`, `concept_relationship_extra.tsv`, `vocabulary_extra.tsv`) to add concept mappings for FHIR-specific code systems not present in Athena. These are optional; enchilada starts without them. This is the default.

enchilada runs over HTTPS with a self-signed certificate. The matchbox image includes a combined JKS truststore (`/certs/combined.jks`) that merges the JVM's default CA bundle with enchilada's self-signed cert, so matchbox can reach both enchilada (self-signed) and public HTTPS services (CA-signed) without disabling certificate validation. The enchilada cert and the combined truststore are baked into the published images — no manual certificate setup is required.

**Hosted/cloud** — any FHIR R4 terminology server can be configured via `MATCHBOX_FHIR_CONTEXT_TXSERVER`. The server must include OMOP concept vocabularies; not all public FHIR servers do — tx.fhir.org, for example, may not have them.

- **echidna** — a public [echidna.fhir.org](https://echidna.fhir.org) terminology service. Available free of charge with rate limits; no local vocabulary files required. API key authentication for higher limits is not currently supported in the matchbox txServer code path. The free tier enforces approximately 60 requests per minute; set `TRANSFORM_SLEEP=1` in the dqd container environment to throttle ETL calls accordingly.

- The default configuration uses locally loaded vocabularies obtained directly by the participant. Alternative terminology services may be used, but _users should review any applicable licensing requirements associated with the vocabularies being accessed._

To use a hosted server, set the terminology server URL when starting:

```bash
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml \
  | MATCHBOX_FHIR_CONTEXT_TXSERVER=https://echidna.fhir.org/r4 TRANSFORM_SLEEP=1 docker compose -f - up
```

Or if you have the compose file locally:
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

`matchbox_scripts` contains two build tools:

- **`build.py`** — the standard pipeline for the default r5/1.0.0 stack. Pass steps as arguments (`ig`, `mvn`, `docker`, `restart`, `etl`, `test`, `release`); no arguments runs the full pipeline.
- **`build_profiles.py`** — the multi-stack variant with `--fhir-version r4|r5` and `--ig-version 1.0.0|1.0.1` flags. Use this when working with stacks other than the default, or when running multiple stacks in parallel via `dqd_docker/docker-compose.profiles.yml`.

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

### How the ETL pipeline drives fixtures through StructureMaps

`load_duckdb.py` contains a dispatch table, `FIXTURE_TRANSFORMS`, where each row is:

```
(glob pattern,  transform function,   OMOP table,           StructureMap name)
('condition_*.json', transform_condition, 'condition_occurrence', 'ConditionMap')
```

For each row, every fixture file whose name matches the glob is passed to the transform function, which calls matchbox's `$transform` endpoint using the named StructureMap. The result is an OMOP-shaped dict that `load_duckdb.py` inserts into the named OMOP table. There is one transform function per StructureMap — not one per OMOP table, since multiple StructureMaps can target the same table (for example, `MeasurementMap`, `SimpleVitalSignsMap`, `BloodPressurePanelMap`, `BloodPressureSystolicMap`, and `BloodPressureDiastolicMap` all write to `measurement`). A single fixture file can also match multiple rows — blood pressure files are intentionally passed through three separate StructureMaps to produce panel, systolic, and diastolic records.

### Adding FHIR fixtures

Fixtures are FHIR resource JSON files stored in `matchbox_scripts/`. The two built-in sets serve different purposes: `test_files_r5/` contains simple single-resource files for exercising specific StructureMap scenarios; `sample_fixtures_r5/` contains a linked multi-patient set for broader regression coverage. You can add files to either set or bring your own data entirely.

#### Using your own FHIR data

To run the pipeline against your own FHIR resources without modifying the built-in fixture sets, drop your JSON files into `matchbox_scripts/sample_fixtures_r5/` — any file whose name matches an existing glob pattern in `FIXTURE_TRANSFORMS_R5` will be picked up automatically. Name files after the resource type they contain (e.g. `condition_*.json`, `observation_*.json`) to match the existing patterns. The ETL report on port 8088 will include your files alongside the built-in ones.

If you want a completely separate directory, you would need to edit `load_duckdb.py` to point at it — there is no command-line argument for this yet.

#### Existing StructureMaps

Add a JSON file whose name matches an existing glob pattern (e.g. `condition_diabetes.json`, `observation_bp_standing.json`) — no code changes needed. The glob patterns are defined in `FIXTURE_TRANSFORMS` in `load_duckdb.py`.

- **Option B / Jupyter**: Drop the file into `matchbox_scripts/` on your host — it appears immediately inside the container. No restart needed. Commit and push to persist for others.
- **Option A / automated ETL**: Add the file to `matchbox_scripts/`, then rebuild: `docker compose -f dqd_docker/docker-compose.yml -f dqd_docker/docker-compose.dev.yml up --build`.

#### New StructureMaps

To incorporate a new StructureMap (e.g. for Death or Device):

1. Add the FHIR JSON fixture to `matchbox_scripts/`
2. Add a `transform_<type>()` function to `matchbox_scripts/transforms.py` that calls `_call_r5(resource, 'YourMapName')`
3. Add a row to `FIXTURE_TRANSFORMS_R5` in `matchbox_scripts/load_duckdb.py` with the glob pattern, transform function, target OMOP table, and StructureMap name
4. Commit and push, then rebuild for Option A


### Working with the Implementation Guide

The `croeder/matchbox:latest` image has `hl7.fhir.uv.omop#1.0.0` baked in. If you are working on the IG itself — editing StructureMaps or ConceptMaps in `fhir-omop-ig/` — you need to build a new IG package and get it into matchbox. There are two paths depending on whether you want a quick local test or a published image.

#### Step 1 — Build the IG

The IG is built using the [bonfhir ig-toolbox](https://github.com/bonfhir/ig-toolbox) Docker container, which wraps the FHIR publisher JAR. The `build.py ig` step handles this automatically — it runs the bonfhir container against `fhir-omop-ig/`, calls the publisher with `tx.fhir.org` as the build-time terminology server (this is separate from the runtime terminology server used by the pipeline), then copies the output into place and strips transitive package dependencies.

Clone `fhir-omop-ig` and `matchbox_scripts` side by side, then:

```bash
git clone https://github.com/croeder-fhir-to-omop/fhir-omop-ig
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts
cd matchbox_scripts
python3 build.py ig
```

This produces `fhir-omop-ig/output/package.tgz` and copies it to `matchbox_docker/igs/hl7.fhir.uv.omop-1.0.0.tgz`, ready for the Docker build.

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

Clone `matchbox_docker` alongside your other repos, then use the build script:

```bash
git clone https://github.com/croeder-fhir-to-omop/matchbox_docker
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts

cd matchbox_scripts

# Default: builds from HL7/fhir-omop-ig upstream/main → croeder/matchbox:latest
python3 build.py ig mvn docker release

# Fork main: builds from croeder-fhir-to-omop/fhir-omop-ig main → croeder/matchbox:main
python3 build.py --ig-source main ig mvn docker release

# Branch: builds from a named fork branch → croeder/matchbox:<branch>
python3 build.py --ig-source fix-translate-rule-names ig docker release
```

The build script checks out the requested source in `fhir-omop-ig`, builds the IG and image, then restores the original branch. Anyone who then does `docker compose pull` or runs the curl command fresh gets the updated image.


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
2. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 11 StructureMaps
3. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts. Note: `MedicationStatement` is mapped to `drug_exposure`; `MedicationRequest` is intentionally out of scope as it represents a prescription order (an intention), not a completed act.
4. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample fixtures and writes results into a DuckDB OMOP CDM 5.4 database
5. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
6. **dqd_docker** runs `load_duckdb.py` automatically on startup, runs OHDSI Data Quality Dashboard checks and unit tests, and serves results on port 3838 (DQD) and port 8088 (ETL reports, unit test report)

