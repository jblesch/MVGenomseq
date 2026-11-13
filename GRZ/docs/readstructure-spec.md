# `readStructure` specification

*Key words MUST, MUST NOT, SHOULD and MAY as in RFC 2119.*

**tl;dr**

- `readStructure` is an object. Each key is a `readOrder`, and each value lists the segments of one record in 5' to 3' order.
- Segments are found by counting bases from the start of the record, never by matching a sequence.
- `readStructure` is required whenever a FASTQ file is submitted, and absent otherwise.
- A submitted FASTQ holds every kit base in the read sequence. A UMI in the read name is ignored. Adapter bases may be masked and must not be removed.
- A submitted BAM holds only the insert in its read sequence. A UMI sits in the `RX` tag.
- A `known` segment is checked against the data, and a mismatch gives a warning.

## 1. Terms

| Term | Meaning |
|---|---|
| record | one read in a FASTQ file: name line, read sequence, separator, quality scores |
| kit base | a base that the library kit added at the same position in every record of a file: UMI, sample barcode or spacer |
| segment | a run of consecutive bases in a record that all have the same `segmentType` |
| layout | the ordered list of segments for one `readOrder` |
| fixed layout | a layout in which every segment has a `length` |
| position | the 0-based index of a segment's first base, counted from the 5' end of the record |
| record sample | the first 10,000 records of a file, or all records where the file has fewer |

`files[]` stands for `labData.sequenceData.files[]`.

## 2. Requirements on submitted files

Short reads are submitted as FASTQ. Long reads are submitted as BAM or FASTQ.

These requirements apply to FASTQ.

| Requirement | Text |
|---|---|
| **Kit bases stay in the read** | The read sequence MUST hold every base the demultiplexer wrote for that read, kit bases included. |
| **No trimming** | Adapter bases MAY be masked with `N` and MUST NOT be removed. Quality trimming, polyG trimming, length filtering and read discarding MUST NOT be applied. |
| **UMI in the read sequence** | A UMI MUST sit in the read sequence of a submitted file: in R1, in R2, or in an index-read file. A UMI that exists only in the read name does not satisfy this requirement. |
| **Index-read files complete** | Index-read files (`readOrder` `I1` and `I2`) MUST be submitted when the UMI was sequenced in index cycles, and MAY be submitted otherwise. Where any index-read file is submitted, every index read of the run MUST be submitted. |
| **One cycle count per entry** | With a fixed layout, all files of that `readOrder` within one `labData` entry MUST have the same `readLength`. Runs with different cycle counts go into separate `labData` entries. |
| **Read name is not read** | The read name and the read comment MAY hold anything. No check reads them. |

These requirements apply to BAM.

| Requirement | Text |
|---|---|
| **Insert only** | The read sequence MUST hold only the insert. Kit bases MUST NOT remain in it. |
| **UMI in RX** | Where the library has a UMI, every record MUST carry it in the `RX` tag. Several UMIs of one record are joined with `-`, 5' first. Their base qualities SHOULD sit in `QX`. A UMI in another tag, such as `ZT` or `YT`, MUST be moved to `RX`. |

## 3. Schema

`readStructure` is a property of the `labData` item.

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

Two rules complete the schema.

- **One conditional on the `labData` item.** Where any `files[].fileType` is `fastq`, `readStructure` is required. Otherwise it is absent. Its JSON encoding is in `grz-schema.json`.
- **Duplicate keys.** Many JSON parsers keep the last of two equal keys without a message. A validator MUST reject a metadata file in which a key of `readStructure` appears twice.

## 4. Field semantics

| Field | Meaning |
|---|---|
| key `R1`, `R2`, `I1` or `I2` | the `readOrder` of the FASTQ file that the layout describes |
| layout, the value of a key | the segments of one record, listed 5' to 3' in the order they appear in the record |
| `segmentType` | `umi`, `sampleBarcode` and `spacer` hold kit bases. `insert` holds all other bases |
| `length` | the segment has this number of bases in every record |
| `minLength`, `maxLength` | the segment varies in length between these two bounds, both included |
| `sequenceType` | `known`: the bases come from the set in `sequences`. `random`: they do not |
| `sequences` | the set, inline. A set of one entry is a fixed sequence |
| `reverseComplement` | `true`: the bases in the record are the reverse complement of the entries in `sequences`. `false`: they are the entries as written |

Further rules on the fields:

- A layout MUST list every base of the record.
- Only a platform without a fixed cycle count MAY use `minLength` and `maxLength`.
- `labData.barcode` holds each index in forward orientation, never its reverse complement. For the second index, that is the sequence the kit lists for the forward strand workflow. Two indexes are joined with `-`, first index then second index. Where the library has no index, or the sequence is not known, `labData.barcode` is `null`.
- Where a submitted file holds the barcode, a `sampleBarcode` segment declares it, with the index in forward orientation as its entry. This declaration takes precedence over `labData.barcode`, which MAY repeat the value or be `null`.

## 5. Positions

No position is declared. It follows from the order of the segments and their lengths.

- The first segment has position 0.
- Each further segment has the position of the segment before it plus that segment's `length`.
- Behind a segment with `minLength` and `maxLength`, positions count from the 3' end: the last segment ends at the last base of the record. The variable segment covers the bases in between.

A segment is found by counting bases, never by matching a sequence. The GRZ MUST NOT search a record for a sequence to decide where a segment starts or ends. Only the known sequence check reads `sequences`.

`length` is the number of bases the sequencer read for the segment, one per sequencing cycle. It can be larger than the length the kit documentation gives for one sequence. Where a kit uses sequences of different lengths for one segment, the segment MUST still have one `length`: a shorter sequence is entered together with the constant bases that follow it. A segment whose sequences cannot be completed to one length MAY be declared `random`.

## 6. Checks

The checks run in three steps. Each step needs more than the one before it.

- **Schema checks** need the metadata file only. They are the schema of section 3.
- **Metadata checks** compare fields of one submission with each other. They read no file.
- **Data checks** read the submitted files.

A submission with an error MUST be rejected. A warning is reported and does not reject the submission. No extraction runs on a submission that has an error.

### Metadata checks

| Check | What it tests | Result |
|---|---|---|
| **Layouts match files** | every `readOrder` among the FASTQ files of the `labData` entry has a layout in `readStructure`, and every layout has a FASTQ file with that `readOrder`. Where the FASTQ files carry no `readOrder`, as with long reads, the only key is `R1` | error |
| **Minimum at most maximum** | `minLength` is at most `maxLength` | error |
| **One insert per layout** | no layout holds more than one `insert` segment | error |
| **Neighbouring segments differ** | no two neighbouring `random` segments have the same `segmentType` | warning |
| **Length sum** | in a fixed layout, the lengths sum to `files[].readLength` for that `readOrder` | error |
| **Length range** | in a layout with a variable segment: the fixed lengths plus `minLength` is at most `files[].readLength`, and `files[].readLength` is at most the fixed lengths plus `maxLength` | error |
| **Entry length** | every entry in `sequences` has the segment's `length` | error |
| **One cycle count per entry** | the requirement of the same name in section 2 | error |

A layout cuts each record in exactly one way, because it holds at most one variable segment: only an `insert` may vary, and a layout holds one `insert`. Two variable segments would not cut uniquely: with `umi` 6 to 7 and `insert` 100 to 150, a record of 107 bases is 6 + 101 or 7 + 100. The warning flags a declaration that has a shorter form: two neighbouring `random` segments of one type could be one segment.

### Data checks

| Check | What it tests | Runs on | Result |
|---|---|---|---|
| **Record length** | in a fixed layout, every record has exactly the summed length. In a layout with a variable segment, every record has at least the fixed lengths plus `minLength` and at most the fixed lengths plus `maxLength` | every record | error |
| **Known sequence** | for each `known` segment: at least 95 percent of the records match an entry in `sequences` | the record sample | warning |
| **UMI tag complete** | in a BAM: either every record carries `RX`, or none does | every record | error |

**How the known sequence check matches.**

1. Take the bases of the record at the segment's position, over the segment's `length`.
2. Where `reverseComplement` is `true`, compare them with the reverse complement of each entry. Otherwise compare them with each entry as written.
3. The comparison is by Hamming distance. Insertions and deletions are not allowed.
4. A segment shorter than 6 bases matches at distance 0 only. A segment of 6 bases or more matches at distance 0 or 1.
5. An `N` in the record counts as a mismatch.
6. The record matches when at least one entry matches.

## 7. Derived values

A pipeline derives these values from `readStructure`. Only the sample barcode falls back to another field, `labData.barcode`.

| Value | Derivation |
|---|---|
| UMI used | FASTQ: at least one segment with `segmentType` `umi` exists. BAM: the records carry `RX` |
| UMI placement | a `umi` segment in both the `R1` and the `R2` layout is a dual UMI. A `umi` segment in exactly one layout is a single-ended UMI |
| duplex | the GRZ treats every dual UMI as duplex-capable. Whether the chemistry of a kit is duplex is not declared |
| UMI scheme | `sequenceType` of the `umi` segment: `random` or `known` |
| nonrandom UMI set | `sequences` of the `umi` segment |
| UMI tag | FASTQ: written at the GRZ by cutting each `umi` segment at its position. Each `umi` segment is one UMI, and several are joined with `-`, 5' first. BAM: the `RX` tag |
| sample barcode | the entries of the `sampleBarcode` segments, in the order `R1`, `R2`, `I1`, `I2` and 5' to 3' within a layout, joined with `-`. Without such a segment, `labData.barcode`. `sampleBarcode` and `spacer` bases are cut and discarded |

## 8. Open points

- A BAM submission records UMI usage only in its records (`RX`), not in the metadata.
- Whether reads of platforms without a fixed cycle count end at a constant position relative to the molecule. Counting from the 3' end needs that, for example for a UMI at both ends of a long read.
- The UMI placement of a library with a `umi` segment in two layouts other than `R1` and `R2`, for example in `I1` and `I2`.
- How the requirement "No trimming" applies to platforms without a fixed cycle count, and whether their FASTQ files keep the UMI and the barcode in the read sequence. Ion Torrent tag sequencing moves the tags into the BAM tags `ZT` and `YT`, so its FASTQ files lack them.
- Whether an `N` inside an entry of `sequences` is a wildcard. Until that is decided it is compared as written, so it matches nothing but a masked base.
- Whether the record sample should be the first 10,000 records or records spread over the file.
- Whether single-end submissions keep the key `R1`.
- The entries and the number of the nonrandom UMI sets, and whether a vendor's set may be copied into a submission.
- The switch-over date for UMI libraries whose existing files hold the UMI only in the read name.
