# GRZ – Data Specification for the German Model Project Genome Sequencing

This repository contains the **data specification** for the **German Model Project Genome Sequencing (Modellvorhaben Genomsequenzierung)**, coordinated by the **Federal Institute for Drugs and Medical Devices (BfArM)**.

The schema defines the structure and validation rules for data submitted in the context of the project. It uses the [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/schema) standard to ensure syntactic and semantic validation of the **genomic data**.

Further information and documentation can be found on the [website](https://www.bfarm.de/DE/Das-BfArM/Aufgaben/Modellvorhaben-Genomsequenzierung/Informationen-und-downloads/_node.html) of the **Federal Institute** in the downloads section under **'Datensatzspezifikationen'**.


## Repository Structure

### `/GRZ`

This folder contains the core JSON Schema for the **data types** used in the model project.

#### Common
- `grz-schema.json`: General metadata schema for submissions to the GRZ

#### `GRZ/vocabularies`

Controlled vocabularies too long to keep inline in the schema. Each file is a standalone JSON Schema declaring a `$id` and a string `enum`, and is pulled into `grz-schema.json` by `$ref`.

- `instrument-model.json`: sequencing instrument models

### `/Prüfbericht`

This folder contains the schema and example data for **Prüfberichte** (data review reports).

- `Modellvorhaben_SubmissionSchema`: Schema definition for a review report
- `submission_example`: Example file conforming to the schema

## Validating a submission

Your validator needs the vocabulary file as well as `grz-schema.json`. Cloning or downloading this repository gives you the files, but that alone is not enough: JSON Schema libraries do not search the filesystem for referenced files, so it must be registered explicitly.

- **Python** (`jsonschema`): build a `referencing.Registry` from the files in `GRZ/vocabularies/` and pass it to the validator
- **JavaScript** (`ajv`): `ajv.addSchema()` the vocabulary file before compiling the schema
- **CLI tools**: most accept a flag for additional schema files, e.g. `ajv-cli -r`

Alternatively, allow the validator to fetch the file from the published URL, which the `$ref` resolves to.

Validating `grz-schema.json` on its own, with no vocabulary file registered, fails to resolve the reference rather than validating your submission.
