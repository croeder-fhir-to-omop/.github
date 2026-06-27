# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).

Conversion operates at the level of individual FHIR resources. FHIR Bundles are not handled; resources must be submitted individually. Most resource types produce a single OMOP record, though some (such as blood pressure panels) are applied through multiple StructureMaps and produce multiple records.

## Architecture

**enchilada** is a local FHIR terminology server backed by OMOP vocabulary files downloaded from Athena — it translates clinical codes (SNOMED, LOINC, RxNorm, etc.) to OMOP concept IDs. **matchbox** is a FHIR server with the OMOP Implementation Guide pre-loaded; it exposes a `$transform` operation for each StructureMap and calls enchilada at runtime to resolve codes. **matchbox_scripts** contains the Python transform functions and ETL script (`load_duckdb.py`) that drive FHIR test fixtures through matchbox and write results into a DuckDB OMOP CDM 5.4 database. **dqd_docker** runs the full ETL automatically on startup, executes OHDSI Data Quality Dashboard checks, and serves results on two HTTP ports. **jupyter_docker** is the interactive alternative — same transform code, but with a Jupyter notebook interface for hands-on exploration.

### How it works

1. **enchilada** serves OMOP vocabulary lookups over HTTPS using local Athena CSV files, acting as a FHIR terminology server for matchbox
2. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 11 StructureMaps
3. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts. Note: `MedicationStatement` is mapped to `drug_exposure`; `MedicationRequest` is intentionally out of scope as it represents a prescription order (an intention), not a completed act.
4. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample test fixtures and writes results into a DuckDB OMOP CDM 5.4 database
5. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
6. **dqd_docker** runs `load_duckdb.py` automatically on startup, runs OHDSI Data Quality Dashboard checks and unit tests, and serves results on port 3838 (DQD) and port 8088 (ETL reports, unit test report)

### Source → image → running stack

The pipeline has three layers. At the bottom are source files in git repos: [FML](https://hl7.org/fhir/mapping-language.html) StructureMaps and [FSH](https://hl7.org/fhir/uv/shorthand/) ConceptMaps in `fhir-omop-ig/`, Python ETL scripts in `matchbox_scripts/`, and FHIR test fixtures (JSON files) in `matchbox_scripts/sample_fixtures_r5/`. The middle layer is built Docker images: the IG publisher compiles FML/FSH into an IG package baked into the **matchbox** image (the FHIR mapping server); the Python scripts and test fixtures are baked into the **dqd** image (the ETL and reporting container). At the top is the running stack. Editing a source file has no effect until the affected image above it is rebuilt.

## Prerequisites

All images are published to Docker Hub. No git repository clones are needed to *run* the pipeline (clones are only needed for development — see [Developer workflows](#developer-workflows)).

### System requirements

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker + Docker Compose v2)
  - macOS: Apple Silicon or Intel
  - Windows 10 (version 1903 or later) or Windows 11 — Docker Desktop requires WSL2 as its backend. WSL2 is not pre-installed by default on Windows; Docker Desktop's installer will enable it automatically if it isn't already present.
  - Linux: Docker Engine + Docker Compose v2
- **RAM**: 16GB minimum. Docker Desktop's memory limit (Settings → Resources) should be at least 12GB.
- **Disk**: 10GB free space for images and data volumes.

> **Note:** Docker Hub shows a "Run in Docker Desktop" button on each image page (`croeder/enchilada`, `croeder/matchbox`, `croeder/dqd`, `croeder/jupyter`). Avoid it — it starts only that one container in isolation, leaving the other services unreachable. The pipeline requires services running together with shared networking, which only the docker compose file provides. Use the `curl` commands below instead.

### Vocabulary files

enchilada needs two vocabulary files from [Athena](https://athena.ohdsi.org). The download can be several GB and take 30+ minutes on a slow connection. **Do this before a Connectathon event, not on the day.**

1. Create a free account at [athena.ohdsi.org](https://athena.ohdsi.org)
2. Click **Download** and select the vocabulary bundles: SNOMED, ICD10CM, ICD9CM, RxNorm, LOINC, CVX, UCUM, Race
3. Download and extract — you need `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv`

Place both files in your working directory before starting. (See [Vocabularies](#vocabularies) below for what each is used for.)

> **Licensing note:** Creating an Athena account and downloading vocabulary files constitutes acceptance of Athena's terms of use, which incorporate the per-vocabulary license terms described in [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md). Review those terms before downloading, particularly for SNOMED CT if you are outside the United States.

> If you start without the vocabulary files, enchilada will warn but the stack will still come up. Terminology lookups will return no results until you restart with the files present.

## Quick start

Two ways to run the pipeline from the published images: an automated ETL + Data Quality Dashboard, or an interactive Jupyter notebook environment.

### Option A — Automated ETL + Data Quality Dashboard

Runs enchilada (local terminology server), matchbox (FHIR→OMOP transforms), the ETL, OHDSI Data Quality Dashboard checks, and unit tests, then serves dashboards and reports.

Place `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv` in a working directory, then:

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

Or download the docker compose file first if you want to keep or edit it:
```bash
curl -O https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml
docker compose up
```

On first run, enchilada loads the vocabulary CSVs (~1–2 min) and matchbox loads the OMOP IG (~1 min). Both are cached in Docker volumes and skipped on subsequent starts.

Once up, the stack serves:

| URL | What you see |
|---|---|
| http://localhost:3838 | OHDSI Data Quality Dashboard |
| http://localhost:8088 | ETL reports and unit test report |

The port 8088 index links to three reports:

- **ETL report — test files**: results from `matchbox_scripts/test_files_r5/`, simple single-resource files (one per scenario) covering the happy path and basic edge cases for each StructureMap — no patient linking between files
- **ETL report — sample test fixtures**: results from `matchbox_scripts/sample_fixtures_r5/`, a patient-centric set with four synthetic patients (p1–p4) whose encounters, conditions, observations, and medications cross-reference each other; includes explicit negative cases (files suffixed `_NEG`) and issue-tracked edge cases (files referencing `f2o-xxx` issue numbers)
- **Unit test report**: results from `matchbox_scripts/tests/test_r5_fml_transforms.py`, a pytest suite that calls matchbox's `$transform` endpoint directly and asserts specific OMOP field values — these are the primary correctness tests for the StructureMap implementations

### Option B — Interactive Jupyter Notebooks

Starts matchbox and a Jupyter notebook server with `transforms.py` pre-installed for hands-on FHIR→OMOP exploration.

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

### Stopping

```bash
docker compose down      # keep volumes (fast restart — vocabulary cache preserved)
docker compose down -v   # also remove volumes (full reload on next start)
```

> **Switching matchbox images:** run `docker compose down -v` before changing `MATCHBOX_IMAGE`, or the old IG will be reused from the cached volume. If volumes persist, find and remove them manually. Docker compose names volumes using the directory you ran it from as a prefix — if you ran from a directory called `mydata`, your volumes will be `mydata_enchilada-db`, `mydata_matchbox-db`, and `mydata_omop-db`:
> ```bash
> docker volume ls
> docker volume rm <dir>_enchilada-db <dir>_matchbox-db <dir>_omop-db
> ```
>
> When developing with the repos cloned side by side, `python3 build.py down` tears down the stack and removes all three volumes for you.

## Terminology server

matchbox resolves clinical codes against a FHIR terminology server. The default is the local **enchilada** server; a hosted server can be substituted.

**enchilada** — a local OMOP-backed terminology server included in `docker-compose.yml`. Requires `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv` from Athena (see [Vocabulary files](#vocabulary-files)). You can also supply supplemental concept files (`concept_extra.tsv`, `concept_relationship_extra.tsv`, `vocabulary_extra.tsv`) to add concept mappings for FHIR-specific code systems not present in Athena. These are optional; enchilada starts without them. This is the default.

enchilada runs over HTTPS with a self-signed certificate. The matchbox image includes a combined JKS truststore (`/certs/combined.jks`) that merges the JVM's default CA bundle with enchilada's self-signed cert, so matchbox can reach both enchilada (self-signed) and public HTTPS services (CA-signed) without disabling certificate validation. The enchilada cert and the combined truststore are baked into the published images — no manual certificate setup is required.

**Hosted/cloud** — any FHIR R4 terminology server can be configured via `MATCHBOX_FHIR_CONTEXT_TXSERVER`. The server must include OMOP concept vocabularies; not all public FHIR servers do — tx.fhir.org, for example, may not have them.

- **echidna** — a public [echidna.fhir.org](https://echidna.fhir.org) terminology service operated by Echidna Systems. Available free of charge with rate limits; no local vocabulary files required. API key authentication for higher limits is not currently supported in the matchbox txServer code path. The free tier enforces approximately 60 requests per minute; set `TRANSFORM_SLEEP=1` in the dqd container environment to throttle ETL calls accordingly. **Note:** echidna's [terms of use](https://echidna.fhir.org/terms/) restrict access to personal use and explicitly prohibit commercial use of the OMOP vocabulary data it provides. Users must not redistribute echidna data to third parties. echidna includes SNOMED CT content — using it does not eliminate the SNOMED CT license requirement for users outside SNOMED International member countries. See [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md) for details.

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

## Developer workflows

To modify StructureMaps, ConceptMaps, transforms, ETL logic, or test fixtures and see changes live, clone the repos **side by side into the same parent directory** and drive the stack with `build.py`.

```bash
git clone https://github.com/croeder-fhir-to-omop/fhir-omop-ig
git clone https://github.com/croeder-fhir-to-omop/matchbox_docker
git clone https://github.com/croeder-fhir-to-omop/matchbox_scripts
git clone https://github.com/croeder-fhir-to-omop/dqd_docker
```

Place `CONCEPT.csv` and `CONCEPT_RELATIONSHIP.csv` in a `data/` subdirectory of the parent — i.e., `fhir-to-omop-docker/data/CONCEPT.csv`. The `dqd_docker/CONCEPT.csv` and `dqd_docker/CONCEPT_RELATIONSHIP.csv` entries are symlinks that resolve there.

`matchbox_scripts` contains two build tools:

- **`build.py`** — the standard pipeline for the default r5/1.0.0 stack. Pass task commands or granular steps as arguments; no arguments runs the full pipeline.
- **`build_profiles.py`** — the multi-stack variant with `--fhir-version r4|r5` and `--ig-version 1.0.0|1.0.1` flags. Use this when working with stacks other than the default, or when running multiple stacks in parallel via `dqd_docker/docker-compose.profiles.yml`.

### Editing source and re-running

StructureMaps are written in [FHIR Mapping Language (FML)](https://hl7.org/fhir/mapping-language.html) (`.fml` files) and ConceptMaps in [FHIR Shorthand (FSH)](https://hl7.org/fhir/uv/shorthand/) (`.fsh` files), both in `fhir-omop-ig/`. FHIR test fixtures are JSON files in `matchbox_scripts/sample_fixtures_r5/` — file names must match one of the glob patterns the ETL pipeline recognizes (for example `condition_*.json`, `observation_*.json`, `patient.json`; the full list and the OMOP tables they write to is in the [matchbox_scripts README](https://github.com/croeder-fhir-to-omop/matchbox_scripts#sample-test-fixtures)).

After editing any of these, from `matchbox_scripts/`:

```bash
python3 build.py run local
```

`run local` detects which sub-projects have local changes, rebuilds only the affected images, wipes and restarts the stack, and re-runs the transforms. The IG build and matchbox startup take a few minutes — Docker Desktop's CPU graph shows progress. Results appear at http://localhost:8088.

| What changed | Layer rebuilt | Command (from `matchbox_scripts/`) |
|---|---|---|
| FML StructureMap or FSH ConceptMap | matchbox image | `python3 build.py run local` |
| FHIR test fixture | dqd image | `python3 build.py run local` |
| Python ETL script | dqd image | `python3 build.py run local` |
| matchbox Java source | matchbox image | `python3 build.py run local` |

To also run the test suite after the rebuild, use `test` in place of `run` (see [build.py task commands](#buildpy-task-commands)).

For interactive development against Jupyter, the dev overlay bind-mounts `matchbox_scripts` so edits are visible immediately without a rebuild:
```bash
docker compose -f jupyter_docker/docker-compose.yml \
               -f jupyter_docker/docker-compose.dev.yml up
```

**Windows:** Use PowerShell. The Jupyter dev overlay uses relative volume paths (no `${HOME}` dependency), so repos only need to be cloned side by side as described above.

For full details on transforms, ETL logic, fixtures, IG development, and extending the conversion engine, see the [matchbox_scripts README](https://github.com/croeder-fhir-to-omop/matchbox_scripts).

### Building and publishing images

Requires `fhir-omop-ig`, `matchbox_docker`, and `matchbox_scripts` cloned side by side. From `matchbox_scripts/`:

```bash
# Publish croeder/matchbox:latest (upstream HL7 IG)
python3 build.py ig docker release

# Publish croeder/matchbox:main (fork's main branch)
python3 build.py --ig-source main ig docker release

# Publish croeder/matchbox:<branch> (any fork branch)
python3 build.py --ig-source <branch> ig docker release
```

`ig` compiles the IG from the specified source, `docker` builds the matchbox image locally, `release` pushes it to Docker Hub. Always run `python3 build.py down` before switching to a newly published image so matchbox reloads the IG fresh from the new image rather than the cached volume.

To build from a source and run the stack **without** publishing, use the task commands instead: `python3 build.py run upstream` rebuilds from HL7 upstream `main`, and `python3 build.py run <branch>` rebuilds `fhir-omop-ig` from the named branch. Both restart the stack and run the ETL with the freshly built images.

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

## Reference

### build.py task commands

Task commands resolve to a step sequence automatically and are the preferred entry points:

| Command | Behavior |
|---|---|
| `python3 build.py run` | restart the stack and run ETL with current images |
| `python3 build.py run upstream` | rebuild from HL7 upstream `main`, restart, run ETL |
| `python3 build.py run local` | detect dirty sub-projects, rebuild what's needed, restart, run ETL |
| `python3 build.py run <branch>` | rebuild `fhir-omop-ig` from the named fork branch, restart, run ETL |
| `python3 build.py test` | same as `run` but also executes the test suite |
| `python3 build.py test upstream` | same as `run upstream` plus the test suite |
| `python3 build.py test local` | same as `run local` plus the test suite |
| `python3 build.py test <branch>` | same as `run <branch>` plus the test suite |

### build.py steps

Granular steps give surgical control and can be combined in sequence (e.g. `python3 build.py ig docker restart`). With no arguments, `build.py` runs the full default pipeline: `ig mvn docker restart etl test`.

| Step | Description |
|---|---|
| `ig` | rebuild the IG package only |
| `mvn` | rebuild the matchbox JAR only |
| `docker` | rebuild the matchbox Docker image only |
| `release` | build and push images to Docker Hub |
| `start` | start the stack without wiping volumes |
| `restart` | wipe matchbox-db and omop-db, then restart (forces IG reload; preserves the enchilada vocabulary cache) |
| `stop` | pause containers without removing them |
| `down` | tear down the stack and remove all volumes (matchbox-db, omop-db, enchilada-db) |
| `etl` | re-run the ETL and open the reports at localhost |
| `test` | run the unit and integration test suites |

### build.py options

| Flag | Description |
|---|---|
| `--ig-source <value>` | IG source for the build. Omitted = HL7/fhir-omop-ig upstream `main` (tagged `:latest`); `main` = fork `main` (tagged `:main`); any other value = that fork branch (tagged `:<branch>`) |
| `--tx-server <url>` | terminology server URL passed to the IG publisher (default `https://tx.fhir.org`; use `n/a` to skip terminology validation) |
| `--dry-run` | print the resolved steps, ig-source, and dev-overlay flag without executing anything |

### Docker Images

The matchbox image is published to Docker Hub under `croeder/matchbox`. Tags reflect which IG source was used at build time:

| Tag | IG Source |
|---|---|
| `croeder/matchbox:latest` | [`HL7/fhir-omop-ig`](https://github.com/HL7/fhir-omop-ig) `main` — the official upstream release |
| `croeder/matchbox:main` | [`croeder-fhir-to-omop/fhir-omop-ig`](https://github.com/croeder-fhir-to-omop/fhir-omop-ig) `main` — this organization's fork |
| `croeder/matchbox:<branch>` | `croeder-fhir-to-omop/fhir-omop-ig` branch `<branch>` |

To run with the upstream HL7 IG (`:latest`), use the [Quick start](#quick-start) commands as written. To run with the fork's main (`:main`), set `MATCHBOX_IMAGE` first:
```bash
export MATCHBOX_IMAGE=croeder/matchbox:main
curl -fsSL https://raw.githubusercontent.com/croeder-fhir-to-omop/dqd_docker/main/docker-compose.yml \
  | docker compose -f - up
```

To use a different tag when you have the docker compose file locally, set `MATCHBOX_IMAGE` in a `.env` file alongside it. Run `docker compose down -v` (or `python3 build.py down` if developing) before switching images so matchbox reloads the IG from scratch.

### FHIR, OMOP, and IG Versions

| Component | Version |
|---|---|
| FHIR | R5 |
| OMOP CDM | 5.4 |
| HL7 FHIR-to-OMOP IG | 1.0.0 |

The tests here assume FHIR R5. matchbox calls the terminology server via R4 endpoints.

A profiles-based compose setup (`dqd_docker/docker-compose.profiles.yml`) exists for running multiple stacks in parallel and switching between FHIR R4 and R5 or between IG versions — see the `dqd_docker` repo for details.

### Vocabularies

Vocabulary use falls into two categories: code systems translated via ConceptMaps embedded in StructureMaps, and code systems looked up dynamically through a terminology server at runtime. Calls to the function translate() in the structure maps indicate which method to use by the inclusion of a concept map name. If the name is present, it is used. If it is an empty string, the configured server is used. ConceptMaps are used for HL7 terminologies as a distribution mechanism. If a configured server included these vocabularies, the concept maps would be redundant.

#### ConceptMap translations (StructureMap-embedded)

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

#### Terminology server lookups

Clinical code systems are resolved at runtime by calling a terminology server.

**Athena vocabularies**

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

**HL7/FHIR-defined code systems**

The ConceptMap translations above use HL7 and FHIR-defined code systems (administrative-gender, v3-ActCode, condition-clinical, etc.) that may not be available from terminology servers loaded only with Athena vocabularies.

- Local terminology server enchilada can support additional vocabularies via supplemental concept files (`concept_extra.tsv`, `concept_relationship_extra.tsv`, `vocabulary_extra.tsv`) placed in the working directory before startup.
- Hosted or cloud servers may also support additional vocabularies.

### Repositories

| Repo | Description |
|---|---|
| [fhir-omop-ig](https://github.com/croeder-fhir-to-omop/fhir-omop-ig) | HL7 FHIR-to-OMOP Implementation Guide — StructureMaps and ConceptMaps |
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | `transforms.py` (FHIR→OMOP via matchbox), `load_duckdb.py` (ETL into OMOP CDM 5.4), and sample FHIR test fixtures |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment for interactive FHIR→OMOP exploration |
| [dqd_docker](https://github.com/croeder-fhir-to-omop/dqd_docker) | Runs the ETL then serves the OHDSI Data Quality Dashboard against the resulting OMOP CDM |
| [enchilada](https://github.com/croeder-fhir-to-omop/enchilada) | Local OMOP-backed FHIR terminology server |

## DISCLAIMER

The IG includes many precautions for dealing with PII in the data. This code makes no guarantee to do so. Do not use it with PII or PHI.

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.

This project uses clinical terminology content from LOINC (Regenstrief Institute), SNOMED CT (SNOMED International), HL7 International, UCUM (Regenstrief Institute), CVX (CDC/NLM), ICD-10-CM (CDC/NCHS), RxNorm (NLM), and OMOP Standardized Vocabularies (OHDSI/Athena). Each carries its own license terms. See [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md) for details.

> **⚠ SNOMED CT users outside the United States:** SNOMED CT is not freely available in all countries. Users in SNOMED International member countries (including Australia, Canada, the Netherlands, and Poland) may access it through their national member organization at no cost. Users in non-member countries must obtain a commercial license from SNOMED International before using this project with SNOMED CT content. See [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md) for country-specific licensing links.

> **⚠ LOINC users and distributors:** LOINC content may be used free of charge, but the [LOINC license](https://loinc.org/license/) requires that the LOINC copyright notice be preserved and that LOINC content not be modified. LOINC codes appear in the sample test fixtures distributed as part of this project's Docker images. Developers who extend those fixtures or redistribute derived images should review the LOINC license terms in [NOTICES.md](https://github.com/croeder-fhir-to-omop/.github/blob/main/profile/NOTICES.md).
