# FASTQ specification

**tl;dr**

- The read sequence of a FASTQ record holds only the insert and any read-through into the adapter. UMI, spacer and sample barcode bases are not part of it.
- A UMI sits in the read name. No index-read files are submitted.
- `labData.readNamePattern` is one of the regular expressions listed in the schema. It captures `readId` for pairing and `readNumber` for checking R1 and R2. An expression for UMI libraries also captures `umi`.
- `labData.adapters` lists the adapter trimming sequences of each data read, and states whether read-through was removed. No other trimming is allowed.
- Both fields are required whenever a FASTQ file is submitted, and forbidden otherwise.
- Long-read platforms submit BAM.

_The key words MUST, MUST NOT and MAY are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals._

## 1. Terms

| Term                          | Meaning                                                                                                                |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| record                        | one read in a FASTQ file: read name line, read sequence, separator line, quality line. Each line ends with LF or CR LF |
| read name                     | the first line of a record without the leading `@` and without the line ending. It includes the comment                |
| comment                       | the part of the read name behind the first whitespace                                                                  |
| `readId`, `readNumber`, `umi` | the parts of the read name that the named groups of `readNamePattern` capture                                          |
| insert                        | the bases of a read that come from the sample fragment                                                                 |
| read-through                  | the adapter bases behind the insert, which a read holds where the fragment is shorter than the read                    |
| data read                     | R1 or R2, as opposed to an index read                                                                                  |
| pair                          | the R1 file and the R2 file with the same `flowcellId` and `laneId`                                                    |

`files[]` stands for `labData.sequenceData.files[]`. A FASTQ file without `readOrder` counts as having `readOrder` `R1`.

## 2. Requirements on submitted files

Short reads from platforms with a fixed cycle count are submitted as FASTQ. Long-read platforms submit BAM. This spec covers FASTQ only. Single-cell libraries are out of scope, because their barcode read holds no insert.

- **One data read per file.** Each FASTQ file MUST hold the records of one data read. Interleaved files are not accepted.
- **Insert only.** The read sequence MUST hold only the insert, followed by any read-through. UMI, spacer and sample barcode bases MUST NOT appear in it.
- **Read name.** Every read name MUST match `readNamePattern`, and it MUST hold only the characters 0x20 to 0x7E. Where the library has a UMI, `readNamePattern` MUST be an expression with a `umi` group.
- **Adapter handling.** Read-through MAY be removed from the 3' end of a read sequence, replaced with `N`, or both. Where too few bases remain, the whole read sequence MAY be replaced with `N`. Where read-through was removed from any record, `adapters.trimmed` MUST be `true`.
- **No other changes.** The read sequence MUST NOT change beyond "Adapter handling". Quality trimming, polyG trimming, length filtering and read discarding MUST NOT be applied. Reads that the instrument's filter flagged as failed MAY still be left out.
- **Mates in order.** The R1 file and the R2 file of a pair MUST hold the same number of records, and their n-th records MUST have the same `readId`. Where `readNamePattern` has a `umi` group, their n-th records MUST also have the same `umi`.

The BfArM document "QS durch GRZ" (version 01.4) describes the raw data as "nach Adapter- aber ohne Qualitäts-Trimming, im Zweifelsfall ungetrimmte Rohdaten": after adapter trimming but without quality trimming, and untrimmed raw data when in doubt. This spec therefore allows adapter trimming but does not require it.

## 3. Read name

`readNamePattern` takes one of these expressions, and a validator matches it against the whole read name.

| # | Title                                                             | Expression                                                                          |
| - | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1 | Read number in the comment                                        | `^(?<readId>[^\s\/]+)\s(?<readNumber>[12]):.*$`                                     |
| 2 | Read number in the comment, UMI as the last colon-separated field | `^(?<readId>[^\s\/]+):(?<umi>r?[ACGTN]+(?:\+r?[ACGTN]+)*)\s(?<readNumber>[12]):.*$` |

| Read name                                                                     | # | `readId`                               | `readNumber` | `umi`             |
| ----------------------------------------------------------------------------- | - | -------------------------------------- | ------------ | ----------------- |
| `A00123:8:H5K3MDSXX:1:1101:1000:1000:ACGTACG+TCGGAGA 1:N:0:TAAGGCGA+AAGGCTAT` | 2 | `A00123:8:H5K3MDSXX:1:1101:1000:1000`  | `1`          | `ACGTACG+TCGGAGA` |
| `A00123:8:H5K3MDSXX:1:1101:1000:1000:ACGTACG+TCGGAGA 2:N:0:TAAGGCGA+AAGGCTAT` | 2 | `A00123:8:H5K3MDSXX:1:1101:1000:1000`  | `2`          | `ACGTACG+TCGGAGA` |
| `LH00123:5:22CFKLLT4:1:1101:2000:1000:rGTACGTACGT 1:N:0:TAAGGCGA`             | 2 | `LH00123:5:22CFKLLT4:1:1101:2000:1000` | `1`          | `rGTACGTACGT`     |
| `A00123:8:H5K3MDSXX:1:1101:3000:1000 1:N:0:TAAGGCGA+AAGGCTAT`                 | 1 | `A00123:8:H5K3MDSXX:1:1101:3000:1000`  | `1`          |                   |

The first two rows are the mates of one UMI library.

`readNumber` is the first colon-separated field of the comment. No other part of the comment is checked. `umi` is the last colon-separated field in front of the comment. Several UMIs are joined with `+`. A leading `r` marks a UMI part from an index read that was read as reverse complement, and the `r` is part of the UMI. The read name holds the UMI bases only, so UMI base qualities are not submitted.

`readId` holds no `/`, so a read name with a `/1` or `/2` suffix fails "Read name format" rather than "Mates in order".

Read names that match no listed expression MUST be rewritten to match one.

### Rules for the list

- Every expression MUST capture `readId` and `readNumber`. An expression for UMI libraries MUST also capture `umi`. A validator ignores any other named group, so a later expression MAY capture more.
- Group names MUST match `[A-Za-z][A-Za-z0-9]*`. A group name MUST NOT appear twice in one expression, because several engines reject that.
- Named groups are written `(?<name>...)`, as in ECMA-262, the regular expression dialect of JSON Schema. The text `(?<` MUST appear only where it opens a named group. Python's `re` module expects `(?P<name>...)`, so a Python validator replaces every `(?<` with `(?P<` before compiling.
- Expressions MUST use only literal characters, escaped characters, `.`, `\s`, anchors, character classes, non-capturing and named groups, and quantifiers. They MUST NOT use lookaround, backreferences or flags. A `/` MUST be escaped as `\/`, also inside a character class. Expressions then run unchanged in ECMA-262 with and without the `u` and `v` flags, in the Rust `regex` crate from version 1.9 and in Go from version 1.22.
- A new read name format is added as a new expression. An expression in the list MUST NOT change, because submissions refer to it by its exact text.

## 4. Adapters

Each item of `adapters.sequences.R1` and `adapters.sequences.R2` is one adapter trimming sequence, as the manufacturer of the adapters lists it. It is written 5' to 3' as it appears in the read. Where the manufacturer joins several sequences with `+`, each part is its own item. The sequences are recorded even where `trimmed` is `false`. Records may hold read-through whatever the value of `trimmed`, so a pipeline still trims with `adapters.sequences`.

A paired-end TruSeq library with a UMI and trimmed adapters:

```json
"adapters": {
  "sequences": {
    "R1": ["AGATCGGAAGAGCACACGTCTGAACTCCAGTCA"],
    "R2": ["AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT"]
  },
  "trimmed": true
},
"readNamePattern": "^(?<readId>[^\\s\\/]+):(?<umi>r?[ACGTN]+(?:\\+r?[ACGTN]+)*)\\s(?<readNumber>[12]):.*$"
```

## 5. Checks

The schema checks the form of the adapter sequences and that `readNamePattern` is one of the listed expressions. Where `sequencingLayout` is `paired-end`, it requires `sequences.R2` and a `readOrder` on every FASTQ file. Where it is `single-end`, it forbids `sequences.R2` and allows only `R1` as `readOrder`. The checks below go beyond the schema.

A submission with an error MUST be rejected. A warning is reported and does not reject the submission.

| Check                        | What it tests                                                                                                                                                                                                                  | Runs on                  | Result  |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------ | ------- |
| **Adapters match files**     | the keys of `adapters.sequences` are exactly the `readOrder` values of the FASTQ files of the lab data entry                                                                                                                   | the metadata file        | error   |
| **Read name format**         | every read name holds only the characters 0x20 to 0x7E and matches `readNamePattern`                                                                                                                                           | every record             | error   |
| **Mates in order**           | the requirement "Mates in order"                                                                                                                                                                                               | every record             | error   |
| **Read number**              | `readNumber` is `1` in every record of an R1 file and `2` in every record of an R2 file                                                                                                                                        | every record             | error   |
| **UMI not in read sequence** | where `readNamePattern` has a `umi` group: at most half of the records have a read sequence that starts or ends with a part of their own `umi`. A part is a `+`-separated piece of `umi`. A part with a leading `r` is skipped | the first 10,000 records | error   |
| **Undeclared UMI**           | where `readNamePattern` has no `umi` group: at most half of the records have a `readId` whose last colon-separated field holds anything other than digits                                                                      | the first 10,000 records | warning |
| **Record length**            | every record has `files[].readLength` bases. Where `adapters.trimmed` is `true`, a record may have fewer                                                                                                                       | every record             | error   |
| **Read length reached**      | where `adapters.trimmed` is `true`: at least one record of the file has `files[].readLength` bases                                                                                                                             | every record             | warning |

The check "UMI not in read sequence" catches a UMI that sits both in the read name and in the read sequence. Chance matches stay far below the threshold of one half. Even with two UMI parts of 3 bases each, checked at both ends, fewer than 1 in 15 records match by chance.

The check "Undeclared UMI" catches a UMI library submitted with an expression that has no `umi` group. Such an expression matches these read names too, with the UMI inside `readId`. In the read name format of section 3, the last colon-separated field of `readId` is the y coordinate of the cluster, which holds digits only. The check therefore also catches a UMI appended in another form, such as `:UMI_AACCGGTT` or `_AACCGGTT`.

The check "Read length reached" catches a `readLength` that is set higher than the record length, for example 151 where every record has 150 bases. It is a warning only, because in an amplicon panel every insert can be shorter than the read.

No check detects UMI bases that sit only in the read sequence. Such a record has no UMI in its read name, and its UMI bases look like insert bases.

## 6. Open points

- This spec does not say how a BAM carries a UMI.
