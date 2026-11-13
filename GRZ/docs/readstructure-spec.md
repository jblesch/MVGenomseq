# `readStructure` specification

*Normative text for the field `labData.readStructure` in `grz-schema.json` 2.0.0. Status: draft, 2026-11-12. Key words MUST, MUST NOT, SHOULD and MAY as in RFC 2119. Provenance markers as in section 1.*

**tl;dr**

- `readStructure` is an object. Each key is a `readOrder`, and each value lists the segments of one record in 5' to 3' order.
- Segments are found by counting bases from the start of the record, never by matching a sequence.
- A layout is unambiguous: it cuts each record into segments in exactly one way.
- It is required whenever a FASTQ file is submitted and optional with BAM.
- A submitted FASTQ holds every kit base in the read sequence. A UMI in the read name is not accepted. Adapter bases may be masked and must not be removed.
- A segment is `known` or `random`. A `known` segment is checked against the data, and a mismatch is an error.
- The GRZ extracts the kit bases, using the declaration. The submitter extracts nothing.

This file states the rules. Worked examples and reference code are in `readstructure-implementation-guide.md`.

Every requirement and every check in this file has a name that says what it does. The names are used with the same wording in the implementation guide.

## Contents

1. Terms
2. Requirements on submitted files
3. Schema
4. Field semantics
5. Positions
6. Unambiguous layouts
7. Checks
8. Derived values
9. Open points

## 1. Terms

| Term | Meaning |
|---|---|
| record | one read in a FASTQ file: name line, read sequence, separator, quality scores |
| kit base | a base that the library kit added: UMI, sample barcode or spacer |
| segment | a run of consecutive bases in a record that all have the same `segmentType` |
| layout | the ordered list of segments for one `readOrder` |
| fixed layout | a layout in which every segment has a `length` |
| position | the 0-based index of a segment's first base, counted from the 5' end of the record |
| record sample | the first 10,000 records of a file, or all records where the file has fewer |

`readOrder`, `readLength`, `sequencingLayout` and `fileType` are fields of `grz-schema.json` and keep their meaning there. `files[]` stands for `labData.sequenceData.files[]`. `labData.barcode` is a field of that schema as well. Section 4 states what this specification requires of it.

A marker in square brackets says where a rule comes from.

| Marker | Meaning |
|---|---|
| [GRZ] | a rule that this specification defines |
| [ILMN] | an Illumina convention: BCL Convert, DRAGEN or the sample sheet |
| [STD] | a vendor-neutral standard |

## 2. Requirements on submitted files

These requirements apply to FASTQ submissions.

| Requirement | Text | Provenance |
|---|---|---|
| **Kit bases stay in the read** | The read sequence MUST hold every base the demultiplexer wrote for that read, kit bases included. The submitter MUST NOT extract a segment. | [GRZ] |
| **No trimming** | Adapter bases MAY be masked with `N` and MUST NOT be removed. Quality trimming, polyG trimming, length filtering and read discarding MUST NOT be applied. | [GRZ] |
| **UMI in the read sequence** | A UMI MUST sit in the read sequence of a submitted file: in R1, in R2, or in an index-read file. A UMI that exists only in the read name does not satisfy this requirement. On BCL Convert, a UMI read in `U` cycles needs `TrimUMI,0`. A UMI read in `Y` cycles stays in the read sequence anyway. | [ILMN] [GRZ] |
| **Index-read files complete** | Index-read files (`readOrder` `I1` and `I2`) MUST be submitted when the UMI was sequenced in index cycles, and MAY be submitted otherwise. Where any index-read file is submitted, every index read of the run MUST be submitted. | [GRZ] |
| **One cycle count per entry** | With a fixed layout, all files of that `readOrder` within one `labData` entry MUST have the same `readLength`. Runs with different cycle counts go into separate `labData` entries. | [GRZ] |
| **Read name is not read** | No requirement and no check in this specification reads the read name or the read comment. | [GRZ] |

## 3. Schema

`readStructure` is a property of the `labData` item. It is an object keyed by `readOrder`. The value of each key is the layout of that file.

```json
"readStructure": {
  "type": "object",
  "additionalProperties": false,
  "minProperties": 1,
  "properties": {
    "R1": { "$ref": "#/$defs/readLayout" },
    "R2": { "$ref": "#/$defs/readLayout" },
    "I1": { "$ref": "#/$defs/indexReadLayout" },
    "I2": { "$ref": "#/$defs/indexReadLayout" }
  }
}
```

```json
"$defs": {

  "readLayout": {
    "type": "array",
    "minItems": 1,
    "items": { "$ref": "#/$defs/readSegment" },
    "contains": { "required": ["segmentType"],
                  "properties": { "segmentType": { "const": "insert" } } }
  },

  "indexReadLayout": {
    "type": "array",
    "minItems": 1,
    "items": {
      "allOf": [ { "$ref": "#/$defs/readSegment" } ],
      "properties": { "segmentType": { "enum": ["umi", "sampleBarcode", "spacer"] } }
    }
  },

  "readSegment": {
    "type": "object",
    "additionalProperties": false,
    "required": ["segmentType", "sequenceType"],
    "properties": {
      "segmentType": { "enum": ["insert", "umi", "sampleBarcode", "spacer"] },
      "length":    { "type": "integer", "minimum": 1 },
      "minLength": { "type": "integer", "minimum": 0 },
      "maxLength": { "type": "integer", "minimum": 1 },
      "sequenceType": { "enum": ["known", "random"] },
      "sequences": {
        "type": "array",
        "minItems": 1,
        "uniqueItems": true,
        "items": { "type": "string", "pattern": "^[ACGTN]+$" }
      },
      "reverseComplement": { "type": "boolean" }
    },
    "allOf": [
      { "oneOf": [
          { "required": ["length"],
            "properties": { "minLength": false, "maxLength": false } },
          { "required": ["minLength", "maxLength"],
            "properties": { "length": false } }
      ] },
      { "if":   { "required": ["sequenceType"],
                  "properties": { "sequenceType": { "const": "known" } } },
        "then": { "required": ["sequences", "reverseComplement"] } },
      { "if":   { "required": ["sequenceType"],
                  "properties": { "sequenceType": { "const": "random" } } },
        "then": { "properties": { "sequences": false, "reverseComplement": false } } },
      { "if":   { "required": ["segmentType"],
                  "properties": { "segmentType": { "const": "insert" } } },
        "then": { "properties": { "sequenceType": { "const": "random" } } },
        "else": { "properties": { "minLength": false, "maxLength": false } } },
      { "if":   { "required": ["segmentType"],
                  "properties": { "segmentType": { "const": "sampleBarcode" } } },
        "then": { "properties": { "sequenceType": { "const": "known" },
                                  "sequences": { "maxItems": 1 } } } }
    ]
  }
}
```

The definitions state four things about layouts and segments.

- An `R1` or `R2` layout holds an `insert` segment.
- An `I1` or `I2` layout holds no `insert` segment.
- Only an `insert` segment may use `minLength` and `maxLength`.
- A `sampleBarcode` segment is always `known` and holds one entry.

Three rules complete the schema.

- **One conditional on the `labData` item.** Where any `files[].fileType` is `fastq`, `readStructure` is required. Its JSON encoding is in `grz-schema.json`. [GRZ]
- **Duplicate keys.** Many JSON parsers, Python's `json` and JavaScript's `JSON.parse` among them, keep the last of two equal keys without a message. A validator MUST reject a metadata file in which a key of `readStructure` appears twice. [GRZ]
- **Dialect.** No keyword above needs a dialect newer than draft-07. `$defs` is the 2019-09 name of `definitions`, and the schema uses the name that its declared dialect prescribes. [STD]

Which layouts `readStructure` must hold is not part of the schema. The check "Layouts match files" in section 7 states it.

## 4. Field semantics

| Field | Required | Meaning |
|---|---|---|
| key `R1`, `R2`, `I1` or `I2` | at least one | the `readOrder` of the submitted file that the layout describes. It matches `files[].readOrder` |
| layout, the value of a key | yes | the segments of one record, listed 5' to 3' in the order they appear in the record |
| `segmentType` | yes | `insert` is template. `umi`, `sampleBarcode` and `spacer` are kit bases |
| `length` | one of the two length forms | the segment has this number of bases in every record |
| `minLength`, `maxLength` | one of the two length forms | the segment varies in length between these two bounds, both included. Allowed on an `insert` segment only |
| `sequenceType` | yes | `known`: the bases come from the set in `sequences`. `random`: they do not. An `insert` is always `random`, and a `sampleBarcode` always `known` |
| `sequences` | for `known`; forbidden for `random` | the set, inline. A set of one entry is a fixed sequence. Entries are unique |
| `reverseComplement` | for `known`; forbidden for `random` | `true`: the bases in the record are the reverse complement of the entries in `sequences`. `false`: they are the entries as written. No default |

Further rules on the fields:

- A layout MUST list every base of the record. A base that reaches the file and is not template is a kit base and MUST be declared as such. [GRZ]
- A cycle that the demultiplexer did not write (`N` in `OverrideCycles`) is not in the record and MUST NOT be declared. [ILMN] [GRZ]
- `labData.barcode` holds each index sequence in forward orientation, never its reverse complement. For the first index, that is the sequence the kit lists for the sample sheet. For the second index, kit documentation often lists two sequences, one per instrument workflow. `labData.barcode` holds the one listed for the forward strand workflow. A sample with two indexes has two different sequences. They are joined with `-`, first index then second index. [GRZ]
- Where the library has no index, or the sequence is not known, `labData.barcode` is `null`. The key is still required. The string `na` is not a value. [GRZ]
- Where a submitted file holds the barcode, a `sampleBarcode` segment declares it. Its one entry is that index in forward orientation, and `reverseComplement` states how the record shows it. This declaration takes precedence over `labData.barcode`, which MAY repeat the value or be `null`. No check compares the two. [GRZ]
- Locus primers of an amplicon panel are part of the `insert` segment. Adapter read-through is part of the last segment. [GRZ]
- The variable-length form is for platforms without a fixed cycle count, such as Ion Torrent. A platform that sequences by synthesis with a fixed cycle count MUST use `length` for every segment. [GRZ]

## 5. Positions

No position is declared. It follows from the order of the segments and their lengths.

- The first segment has position 0.
- Each further segment has the position of the segment before it plus that segment's `length`.
- A segment with `minLength` and `maxLength` ends at the end of the record.

Example: `umi` 3, `spacer` 2, `insert` 146. The UMI covers bases 0 to 2, the spacer covers bases 3 and 4, and the insert covers bases 5 to 150.

### Counting, never matching

A segment is found by counting bases. It is never found by matching a sequence. [GRZ]

- The GRZ cuts each segment at its position. It MUST NOT search a record for a sequence in order to decide where a segment starts or ends.
- `sequences` has one use. The known sequence check tests whether the bases at the position are the declared ones. `sequences` never moves a position.
- A kit segment therefore has the same number of bases in every record of a file. `length` is the number of bases the sequencer read for the segment. That is the number of cycles the conversion assigns to it, for example 7 for `U7` in `OverrideCycles`. It can be larger than the length the kit documentation gives for one sequence.

Some kits use sequences of different lengths for one segment. They MUST be declared with one `length`, as follows.

| Case | Real kit | Declaration | Result |
|---|---|---|---|
| A constant base follows the shorter sequence | TruSight Oncology. The UMIs are 6-mers and 7-mers, read in 7 UMI cycles. The seventh base is `T` after a 6-mer, and it is never `T` in a 7-mer | `umi` with `length` 7. A 6-mer is entered together with the `T` that follows it, so every entry has 7 bases | exact |
| Template follows the shorter sequence | Agilent SureSelect XT HS2. A read starts with 3 UMI bases, then 1 or 2 dark bases, then template | the longer form: `umi` 3, followed by `spacer` 2 where the dark bases are in the file | a read with one dark base loses its first template base when the GRZ extracts the reads. The submitted file keeps every base |

The second case is the price of counting. Illumina documents `U3N2Y146` for that kit, which counts in the same way. [ILMN] A segment whose sequences cannot be completed to one length MAY always be declared `random`.

## 6. Unambiguous layouts

A layout cuts each record into segments in exactly one way, and a random segment cannot be split into an equivalent pair. Five rules guarantee this. [GRZ]

| Rule | What it prevents | Stated by |
|---|---|---|
| Only an `insert` segment may vary in length, and a variable `insert` is the last segment of its layout | two variable segments in one layout. With `umi` 6 to 7 and `insert` 100 to 150, a record of 107 bases is 6 + 101 or 7 + 100 | the schema, and the check "Variable segment last" |
| An `R1` or `R2` layout holds exactly one `insert` segment. An `I1` or `I2` layout holds none | template in two places of one record | the schema, and the check "One insert per layout" |
| No two neighbouring `random` segments have the same `segmentType` | two declarations of one file. `umi` 8, and `umi` 4 followed by `umi` 4, describe the same bases and give the tags `ACGTACGT` and `ACGT-ACGT` | the check "Neighbouring segments differ" |
| `minLength` is strictly below `maxLength` | a segment of one length that is written as variable | the check "Minimum below maximum" |
| The entries of `sequences` are unique | a set that lists one sequence twice | the schema |

Two neighbouring `known` segments of the same `segmentType` are allowed, because two barcodes are two values.

A kit segment always has a `length`. A `known` segment is therefore always at a fixed position, and the known sequence check always runs on it.

## 7. Checks

The checks run in three steps. Each step needs more than the one before it.

- **Schema checks** need the metadata file only. They are the schema of section 3, including the rejection of duplicate keys.
- **Metadata checks** compare fields of one submission with each other. They read no file.
- **Data checks** read the FASTQ files.

A submission with an error MUST be rejected. A warning is reported and does not reject the submission. No extraction runs on a submission that has an error. [GRZ]

### Metadata checks

| Check | What it tests | Result |
|---|---|---|
| **Layouts match files** | every `readOrder` among the read files of the `labData` entry has a layout in `readStructure`, and every layout has a file with that `readOrder`. Where the read files carry no `readOrder`, the only key is `R1` | error |
| **Variable segment last** | a segment with `minLength` and `maxLength` is the last segment of its layout | error |
| **Minimum below maximum** | `minLength` is less than `maxLength` | error |
| **One insert per layout** | no layout holds more than one `insert` segment. The schema already requires one in an `R1` or `R2` layout and forbids it in an `I1` or `I2` layout | error |
| **Neighbouring segments differ** | no two neighbouring `random` segments have the same `segmentType` | error |
| **Length sum** | in a fixed layout, the lengths sum to `files[].readLength` for that `readOrder` | error |
| **Length range** | in a layout with a variable segment: the fixed lengths plus `minLength` is at most `files[].readLength`, and `files[].readLength` is at most the fixed lengths plus `maxLength` | error |
| **Entry length** | every entry in `sequences` has the segment's `length` | error |
| **One cycle count per entry** | the requirement of the same name in section 2: with a fixed layout, all files of that `readOrder` in the `labData` entry have the same `files[].readLength` | error |

### Data checks

| Check | What it tests | Runs on | Result |
|---|---|---|---|
| **Record length** | in a fixed layout, every record has exactly the summed length. In a layout with a variable segment, every record has at least the fixed lengths plus `minLength` and at most the fixed lengths plus `maxLength` | every record | error |
| **Known sequence** | for each `known` segment: at least 95 percent of the records match an entry in `sequences` | the record sample | error |
| **UMI composition** | for each `random` `umi` segment: at no position of the segment does one base exceed 90 percent of the records | the record sample | warning |

**How the known sequence check matches.** [GRZ]

1. Take the bases of the record at the segment's position, over the segment's `length`.
2. Where `reverseComplement` is `true`, compare them with the reverse complement of each entry. Otherwise compare them with each entry as written.
3. The comparison is by Hamming distance. Insertions and deletions are not allowed.
4. A segment shorter than 6 bases matches at distance 0 only. A segment of 6 bases or more matches at distance 0 or 1.
5. An `N` in the record counts as a mismatch.
6. The record matches when at least one entry matches.

## 8. Derived values

A pipeline derives these values from `readStructure`. Only the sample barcode falls back to another field, `labData.barcode`. [GRZ]

| Value | Derivation |
|---|---|
| UMI used | at least one segment with `segmentType` `umi` exists. This holds for FASTQ submissions; see section 9 for BAM |
| UMI placement | a `umi` segment in both the `R1` and the `R2` layout is a dual UMI. A `umi` segment in exactly one layout is a single-ended UMI. These are DRAGEN's terms [ILMN]. A `umi` segment in two layouts other than `R1` and `R2` has no derived placement; see section 9 |
| duplex | the GRZ treats every dual UMI as duplex-capable, as DRAGEN does. Whether the chemistry of a kit is duplex is not declared |
| UMI scheme | `sequenceType` of the `umi` segment: `random` or `known` |
| nonrandom UMI set | `sequences` of the `umi` segment |
| UMI tag | written at the GRZ by cutting each `umi` segment at its position |
| sample barcode | the entries of the `sampleBarcode` segments, in the order `R1`, `R2`, `I1`, `I2` and 5' to 3' within a layout, joined with `-`. Without such a segment, `labData.barcode`. The value is never taken from the read bases. `sampleBarcode` and `spacer` bases are cut and discarded |

The pipeline therefore never applies `reverseComplement` to a barcode. Only the known sequence check reads that field.

## 9. Open points

- A BAM submission with a UMI in `RX` tags and no `readStructure` records UMI usage nowhere.
- A long-read library with a UMI at both ends of a variable insert cannot be declared.
- The UMI placement of a library with a `umi` segment in two layouts other than `R1` and `R2`, for example in `I1` and `I2`.
- Variable-length platforms that submit FASTQ. The FASTQ requirements of this specification assume a fixed read length. How the requirement "No trimming" applies to Ion Torrent is undecided, and so is whether Torrent Suite leaves the molecular tag and the barcode in the read sequence.
- Whether an `N` inside an entry of `sequences` is a wildcard. Until that is decided it is compared as written, so it matches nothing but a masked base.
- Whether the record sample should be the first 10,000 records or records spread over the file. The first records of a file come from the edge of a tile.
- The key for single-end submissions. It is `R1` today because `files[].readOrder` offers nothing else.
- The entries and the number of the nonrandom UMI sets, and whether a vendor's set may be copied into a submission.
- The switch-over date for UMI libraries that were converted with `TrimUMI,1`. This needs a decision.
