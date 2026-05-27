# FHIR to OMOP

Tools and infrastructure for converting clinical data from FHIR to the OMOP Common Data Model using the [HL7 FHIR-to-OMOP Implementation Guide](https://hl7.org/fhir/uv/omop/).

## Repositories

| Repo | Description |
|---|---|
| [matchbox](https://github.com/croeder-fhir-to-omop/matchbox) | FHIR validation and mapping server |
| [matchbox_docker](https://github.com/croeder-fhir-to-omop/matchbox_docker) | Docker configuration and IGs for running matchbox |
| [matchbox_scripts](https://github.com/croeder-fhir-to-omop/matchbox_scripts) | Shell scripts and sample FHIR resources for exercising the OMOP IG maps |
| [jupyter_docker](https://github.com/croeder-fhir-to-omop/jupyter_docker) | Jupyter notebook environment wired to matchbox for interactive FHIR→OMOP exploration |

## How it works

1. **matchbox** runs a FHIR server with the OMOP IG loaded, exposing a `$transform` operation for each StructureMap
2. **matchbox_scripts** provides sample FHIR resources (Condition, Patient, Procedure, Immunization, etc.) and scripts to call `$transform`
3. **jupyter_docker** launches a Jupyter environment connected to matchbox, with notebooks that demonstrate and test all 11 StructureMaps in the IG
