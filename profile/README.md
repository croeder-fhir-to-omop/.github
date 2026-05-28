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

## License

All repositories in this organization are licensed under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0). Copyright 2026 Christophe Roeder.

[matchbox](https://github.com/croeder-fhir-to-omop/matchbox) is a fork of [ahdis/matchbox](https://github.com/ahdis/matchbox) and retains the original copyright of the ahdis contributors.

## How it works

1. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each of the 11 StructureMaps
2. **matchbox_scripts/transforms.py** calls `$transform` for each FHIR resource type and returns OMOP-shaped dicts
3. **matchbox_scripts/load_duckdb.py** runs all transforms against the sample fixtures and writes results into a DuckDB OMOP CDM 5.4 database
4. **jupyter_docker** imports `transforms.py` for interactive exploration — same code, human in the loop
5. **dqd_docker** runs `load_duckdb.py` automatically on startup, then serves the OHDSI Data Quality Dashboard on port 3838
