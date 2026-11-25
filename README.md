# GRZ – Data Specification for the German Model Project Genome Sequencing

This repository contains the **data specification** for the **German Model Project Genome Sequencing (Modellvorhaben Genomsequenzierung)**, coordinated by the **Federal Institute for Drugs and Medical Devices (BfArM)**.

The schema defines the structure and validation rules for data submitted in the context of the project. It uses the [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/schema) standard to ensure syntactic and semantic validation of the **genomic data**.

Further information and documentation can be found on the [website](https://www.bfarm.de/DE/Das-BfArM/Aufgaben/Modellvorhaben-Genomsequenzierung/Informationen-und-downloads/_node.html) of the **Federal Institute** in the downloads section under **'Datensatzspezifikationen'**.


## Repository Structure

### `/GRZ`

This folder contains the core JSON Schema for the **data types** used in the model project.

#### Common
- `grz-schema.json`: General metadata schema for submissions to the GRZ

#### `GRZ/docs`

Specifications for fields whose rules go beyond what the schema can express.

- `readstructure-spec.md`: the normative rules and checks for `labData.readStructure`
- `readstructure-implementation-guide.md`: examples for submitters, and reference code for validation and extraction at the GRZ


### `/Prüfbericht`

This folder contains the schema and example data for **Prüfberichte** (data review reports).

- `Modellvorhaben_SubmissionSchema`: Schema definition for a review report
- `submission_example`: Example file conforming to the schema
