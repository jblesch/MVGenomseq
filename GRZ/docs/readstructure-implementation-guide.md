# `readStructure` implementation guide

> [!WARNING]
> **Best effort.** This guide covers the sequencing setups known so far. It will be extended and corrected as new setups come up.

**tl;dr**

- **A library without a UMI:** copy the example "WGS without UMI" and set `length` to your cycle count. Then fill in `adapters` from `AdapterRead1`, `AdapterRead2` and `AdapterBehavior`.
- **A library with a UMI on Illumina:** set `TrimUMI,0` once in the sample sheet. Then copy the example that matches your `OverrideCycles`, and fill in `adapters` from `AdapterRead1`, `AdapterRead2` and `AdapterBehavior`.
- **Adapters:** mask them with `N`, or trim them from the 3' end. `adapters` records the adapter of each data read and whether it was trimmed. If you are not sure that your trimming meets the requirements, submit untrimmed records. Apply no other trimming.
- **barcode:** write `null` where the library has no index or the sequence is not known. `na` is not accepted.
- **GRZ side:** run the checks in three steps, then derive the fgbio read structures and the DRAGEN options from the declaration.

## Contents

**For submitters**

- [Find your case](#find-your-case)
- [From OverrideCycles to readStructure](#from-overridecycles-to-readstructure)
- [When the sequences of a kit differ in length](#when-the-sequences-of-a-kit-differ-in-length)
- [Conversion settings](#conversion-settings)
- [Examples](#examples)
- [Translate Illumina settings with code](#translate-illumina-settings-with-code)
- [Check your files before you submit](#check-your-files-before-you-submit)
- [What an error means](#what-an-error-means)

**For the GRZ**

- [Order of the checks](#order-of-the-checks)
- [Reference code](#reference-code)
- [Pipeline parameters](#pipeline-parameters)
- [Test vectors](#test-vectors)

# For submitters

## Find your case

| Your library                                                  | Example to copy              | What you change in the conversion                                                 |
| ------------------------------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------- |
| no UMI                                                        | WGS without UMI              | nothing                                                                           |
| Agilent SureSelect XT HS2: UMI at the start of R1 and R2      | Inline UMI                   | `TrimUMI,0`                                                                       |
| the same kit, converted without UMI cycles                    | Inline UMI with dark bases   | nothing                                                                           |
| TruSight Oncology 170 or 500: UMI from a published set        | Nonrandom UMI                | `TrimUMI,0`                                                                       |
| IDT Dual Index Adapters with UMIs: UMI behind the first index | UMI behind the index         | `TrimUMI,0` and `CreateFastqForIndexReads,1`                                      |
| Agilent SureSelect XT HS: UMI in place of the second index    | UMI in the second index read | `TrimUMI,0` and `CreateFastqForIndexReads,1`                                      |
| MGI, kit bases at the end of R2                               | MGI read barcode             | a demultiplexer that leaves the read barcode in R2, and no adapter trimming on R2 |
| Ion Torrent                                                   | none yet                     | not settled; see the open points of the specification                             |

Every case also declares `adapters`: the adapter sequences of each data read, and whether adapter bases were removed. See [Conversion settings](#conversion-settings).

The `trimmed` values of each example assume the `AdapterBehavior` that its text names. "WGS without UMI" assumes the default `trim`, and the other Illumina examples assume `mask`. Both behaviors are allowed. Set `trimmed` from your own `AdapterBehavior`: with BCL Convert's default `trim`, `trimmed` is `true`, and with `mask` it is `false`.

A kit with the same `OverrideCycles` as an example uses that example.

You set `readStructure` once per library kit and `OverrideCycles`. Every submission with that combination carries the same value. Only a `sampleBarcode` segment changes per sample.

## From OverrideCycles to readStructure

Declare what is in the file. Each part of `OverrideCycles` maps to one decision:

| `OverrideCycles`                                                  | In the file             | Declaration                                             |
| ----------------------------------------------------------------- | ----------------------- | ------------------------------------------------------- |
| `Y151` in a data read                                             | 151 template bases      | `insert`, `length` 151                                  |
| `U3` in a data read, with `TrimUMI,0`                             | 3 UMI bases             | `umi`, `length` 3                                       |
| `N2`                                                              | not written             | no segment                                              |
| `I8` in an index read, with no index-read files submitted         | in no submitted file    | no layout. `labData.barcode` holds the value            |
| `I8U9` in the first index read, with `CreateFastqForIndexReads,1` | 17 bases in the I1 file | layout `I1`: `sampleBarcode` 8, then `umi` 9            |
| `U10` in the second index read, with `CreateFastqForIndexReads,1` | 10 bases in the I2 file | layout `I2`: `umi` 10                                   |
| `I8` in an index read, with index-read files submitted            | 8 bases in that file    | `sampleBarcode` 8, `known`, with the index as its entry |

Three rules follow from the table.

- A spacer that is written to the file appears as `Y` in `OverrideCycles`. The sample sheet cannot tell it from template. Take its length and its base from the kit documentation, and declare a `spacer` segment.
- The lengths of one layout sum to the `readLength` of that file. `U7N1Y143` gives 150, not 151. Removed adapter bases do not change the sum, because `readLength` and `length` describe the untrimmed record.
- A layout lists segments in the order of the bases in the record, 5' to 3'.

Two more rules apply to every layout.

- A layout for R1 or R2 has exactly one `insert` segment. A layout for I1 or I2 has none.
- Declare one `umi` segment per UMI of the kit. A 7 base UMI is one `umi` segment of `length` 7, not two segments of 3 and 4, because each `umi` segment counts as one UMI.

Where a submitted file holds the barcode, a `sampleBarcode` segment declares it. That segment is always `known`, and its one entry is the index. `labData.barcode` may repeat the value, or be `null`. Where no submitted file holds the barcode, `labData.barcode` holds it:

- It holds each index in forward orientation. For the second index, take the sequence that the kit lists for the forward strand workflow. Two indexes are joined with `-`, first index then second index, for example `TAAGGCGA-AAGGCTAT`.
- If the library has no index, or you do not know its sequence, the value is `null`. Do not write `na`.

A library with a known barcode, and a library without one:

```json
"barcode": "TAAGGCGA-AAGGCTAT"
```

```json
"barcode": null
```

`null` is written without quotes, and the key is always present. These forms are rejected:

| Written             | Why it fails                                         |
| ------------------- | ---------------------------------------------------- |
| `"barcode": "na"`   | `na` is not a sequence                               |
| `"barcode": "null"` | with quotes it is a string, and it is not a sequence |
| `"barcode": ""`     | an empty string is not a sequence                    |
| no `barcode` key    | the key is required                                  |

## When the sequences of a kit differ in length

A segment has one `length`, the same in every untrimmed record. The GRZ finds a segment by counting bases. It never searches for a sequence.

So `length` is the number of bases the sequencer read for the segment. That is the number behind `U` in `OverrideCycles`. It can be larger than the length your kit documentation gives for one sequence.

**TruSight Oncology.** The UMIs are 6-mers and 7-mers, and the conversion reads 7 UMI cycles (`U7`). After a 6-mer, the seventh base is always the `T` of the adapter. A 7-mer never has `T` as its seventh base.

```
cycles, 7-mer:   ACGTACG T GATTACAGGC...   cycles 1 to 7: the UMI.           cycle 8: the adapter T
cycles, 6-mer:   TTGCAAT G ATTACAGGCT...   cycles 1 to 7: the UMI and the T. cycle 8: template
                 |umi 7|
```

Cycle 8 is the `N1` cycle, so it is not written to the file. Declare `umi` with `length` 7. Enter a 7-mer as it is. Enter a 6-mer with the `T` behind it, so `TTGCAA` becomes `TTGCAAT`. All entries then have 7 bases, and the declaration is exact. The sequences shown stand for the entries of the kit's list.

**Agilent SureSelect XT HS2.** A read starts with 3 UMI bases and then 1 or 2 dark bases: bases between the UMI and the template that carry no information. They are declared as a `spacer` segment. Template follows directly, so the shorter start cannot be completed with a constant base. Declare the longer form.

| Conversion                                                               | Layout                                                        |
| ------------------------------------------------------------------------ | ------------------------------------------------------------- |
| `U3N2Y146` with `TrimUMI,0`, the setting Illumina documents for this kit | `umi` 3 `random`, then `insert` 146                           |
| `Y151` without UMI cycles, the form Agilent's own trimmer expects        | `umi` 3 `random`, then `spacer` 2 `random`, then `insert` 146 |

A read with one dark base loses its first template base when the GRZ extracts the reads. The file you submit keeps every base, so nothing is lost in the archive.

If the sequences of your kit cannot be brought to one length, declare the segment as `random`. That is always a correct declaration.

## Conversion settings

BCL Convert, sample sheet v2. The lines that matter, shown for TruSight Oncology:

```
[BCLConvert_Settings]
OverrideCycles,U7N1Y143;I8;I8;U7N1Y143
TrimUMI,0
AdapterBehavior,mask
AdapterRead1,<adapter sequence for Read 1>
AdapterRead2,<adapter sequence for Read 2>
```

- `TrimUMI,0` leaves the UMI in the read sequence. The default is `1`. With the default the UMI is cut out of the read and exists only in the read name. Such a file breaks the requirement "UMI in the read sequence" and is not accepted. With `TrimUMI,0` BCL Convert still copies the UMI into the read name. That is harmless, because the read name is not read.
- `AdapterRead1` and `AdapterRead2` name the adapter of each data read. Copy them into `adapters` as the `sequences` of `R1` and `R2`. Where a setting joins several sequences with `+`, each sequence becomes one entry.
- `AdapterBehavior,mask` replaces adapter bases with `N` and keeps every record at full length. Declare `"trimmed": false`.
- `AdapterBehavior,trim` removes adapter bases from the 3' end, so records get shorter. Declare `"trimmed": true`. `trim` is the default as soon as the sample sheet names an adapter.
- Where the sample sheet names no adapter, BCL Convert neither masks nor trims. `adapters` is still required. Take the adapter sequences from your kit documentation, and declare `"trimmed": false`. Where a later step trims with another tool, such as cutadapt or fastp, list the adapters given to that tool, and declare `"trimmed": true`.
- Add `CreateFastqForIndexReads,1` when you submit index-read files. You must submit them when the UMI sits in the index cycles. BCL Convert writes the index FASTQ of a UMI only together with `TrimUMI,0`.

A record may end in `N` with either behavior. With `trim`, BCL Convert does not shorten a record below `MinimumTrimmedReadLength`. It masks the remaining adapter bases with `N` instead, so one record can be both shortened and masked. Where too few bases remain in front of the adapter, BCL Convert masks the whole record with `N`. `MaskShortReads` sets that limit. Such a record has `N` in place of its kit bases too. The data checks "Known sequence" and "Mate kit bases" leave out records that consist of `N` only. All of this is allowed. Masking alone keeps `"trimmed": false`. Where adapter bases were removed from any record of a read, declare `"trimmed": true` for that read.

Data that was already converted with `TrimUMI,1` has to be converted again from the per-cycle data. No tool can rebuild the file, because the read name holds the UMI bases without their quality scores.

Adapter trimming has two limits.

- A read whose fixed layout has a segment behind the insert must not be trimmed. In a fixed layout, the GRZ finds those segments by counting from the 5' end. Trimming would cut them off. Mask the adapter bases of such a read instead.
- Trimming removes the adapter bases only. Where the template ends before the read does, an untrimmed record continues with read-through: first the reverse complement of the kit bases in front of the mate's insert, then the adapter. The trimmer must start the removal where it finds the adapter, and it must not remove any base in front of that point. The reverse complement of the mate's kit bases then stays at the end of the record. So the adapter sequence you give the trimmer must not include that reverse complement. A partial match of the adapter at the very end of a record counts as found. cutadapt, for example, removes such a match by default from 3 bases on, and that is allowed.

A variable layout is different. Behind a variable insert, positions count from the 3' end, so every record must end at the same position relative to the molecule. Only adapter removal achieves that. A variable layout with a segment behind the insert therefore describes the record after adapter removal, and its `trimmed` must be `true`. Its `readLength` also refers to the record after adapter removal.

If you are not sure that a trimming step meets the two limits, submit untrimmed records. With BCL Convert, that is `AdapterBehavior,mask` with `"trimmed": false`. The BfArM document "QS durch GRZ" asks for untrimmed raw data when in doubt.

One R1 record of a library with a 3 base UMI directly in front of the template on both reads. The R2 UMI of this pair is `TTC`, and `<AdapterRead1>` stands for the R1 adapter:

```
untrimmed:  ACG GATTACAGGC GAA <AdapterRead1>...
            umi template   |   adapter
trimmed:    ACG GATTACAGGC GAA
                           the reverse complement of the R2 UMI, TTC
```

The data check "Mate kit bases" warns where trimmed records lack these bases. It removes trailing `N` from both records first. Then it looks only at pairs whose two records have the same length and each record is at least 3 bases shorter than the summed length of its own layout. A read-through pair always has two records of the same length. The margin of 3 bases leaves out most pairs that a trimmer shortened by a chance match of the adapter's first bases.

Quality trimming, polyG trimming, length filtering and read discarding are not accepted. A trimmer other than BCL Convert may apply some of them by default. fastp, for example, filters reads by quality and by length. On NextSeq and NovaSeq data, it also trims polyG tails. Turn these off with `-Q` (quality filtering off), `-L` (length filtering off) and `-G` (polyG trimming off).

## Examples

`readStructure` has one key per submitted read file. The key is the `readOrder` of the file, and the value lists the segments of one record, 5' to 3'.

`adapters` has one key per data read, `R1` and `R2`. Index reads have no entry.

In the examples, `files` stands for `sequenceData.files`, and other `labData` fields are left out.

The adapter sequences in the examples are placeholders: `"<AdapterRead1>"` for R1 and `"<AdapterRead2>"` for R2. Replace them with `AdapterRead1` and `AdapterRead2` of your sample sheet. Where it names no adapter, take the sequences from your kit documentation. An adapter sequence holds only `A`, `C`, `G` and `T`, so the schema rejects a placeholder that was not replaced.

### WGS without UMI

`OverrideCycles`: none, or `Y151;I8;I8;Y151`. The sample sheet names an adapter and sets no `AdapterBehavior`, so BCL Convert trimmed the adapter bases.

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 151 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "insert", "length": 151, "sequenceType": "random" }
    ],
    "R2": [{ "segmentType": "insert", "length": 151, "sequenceType": "random" }]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": true },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": true }
  }
}
```

`length` is your own cycle count. A 2x150 run declares 150 in all four places.

Trimmed records can be shorter than 151 bases. `readLength` and `length` still say 151, because they describe the untrimmed record. With `AdapterBehavior,mask`, declare `"trimmed": false`.

### Inline UMI

Agilent SureSelect XT HS2, converted with `U3N2Y146;I8;I8;U3N2Y146`, `TrimUMI,0` and `AdapterBehavior,mask`. Illumina documents the same `OverrideCycles` and `TrimUMI` for the IDT xGen Duplex Seq Adapter. The UMI is on both reads, and it is `random`.

```
record:    ACG GATTACAGGCTTAACGT...
           umi insert 146 ...
position:  0   3
```

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 149 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 149 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "umi", "length": 3, "sequenceType": "random" },
      { "segmentType": "insert", "length": 146, "sequenceType": "random" }
    ],
    "R2": [
      { "segmentType": "umi", "length": 3, "sequenceType": "random" },
      { "segmentType": "insert", "length": 146, "sequenceType": "random" }
    ]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

The two `N` cycles are not written, so a record has 149 bases. Check the record length of your own file before you copy this; see [Check your files before you submit](#check-your-files-before-you-submit). If your records have 151 bases, the next example is your case.

### Inline UMI with dark bases

The same kit, converted without UMI cycles, as `Y151;I8;I8;Y151`, with `AdapterBehavior,mask`. Agilent's own trimmer expects this form. The 3 UMI bases and the dark bases behind them are then in the file, and you declare them.

```
record:    ACG TT GATTACAGGCTTAACG...
position:  0   3  5
           umi 3, then spacer 2, then insert 146
```

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 151 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "umi", "length": 3, "sequenceType": "random" },
      { "segmentType": "spacer", "length": 2, "sequenceType": "random" },
      { "segmentType": "insert", "length": 146, "sequenceType": "random" }
    ],
    "R2": [
      { "segmentType": "umi", "length": 3, "sequenceType": "random" },
      { "segmentType": "spacer", "length": 2, "sequenceType": "random" },
      { "segmentType": "insert", "length": 146, "sequenceType": "random" }
    ]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

The kit has 1 or 2 dark bases, and the layout always counts 2; see [When the sequences of a kit differ in length](#when-the-sequences-of-a-kit-differ-in-length).

### Nonrandom UMI

TruSight Oncology 170 or 500, converted with `U7N1Y143;I8;I8;U7N1Y143`, `TrimUMI,0` and `AdapterBehavior,mask`. The UMI comes from a published set. Declare it as `known` and list the set in `sequences`.

The two sequences below are placeholders. The first stands for a 7-mer, the second for a 6-mer with the `T` that follows it; see [When the sequences of a kit differ in length](#when-the-sequences-of-a-kit-differ-in-length). Replace them with the full set of your kit.

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 150 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 150 }
  ],
  "readStructure": {
    "R1": [
      {
        "segmentType": "umi",
        "length": 7,
        "sequenceType": "known",
        "sequences": ["ACGTACG", "TTGCAAT"],
        "reverseComplement": false
      },
      { "segmentType": "insert", "length": 143, "sequenceType": "random" }
    ],
    "R2": [
      {
        "segmentType": "umi",
        "length": 7,
        "sequenceType": "known",
        "sequences": ["ACGTACG", "TTGCAAT"],
        "reverseComplement": false
      },
      { "segmentType": "insert", "length": 143, "sequenceType": "random" }
    ]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

Every entry has exactly the `length` of the segment.

### UMI behind the index

IDT Dual Index Adapters with UMIs, converted with `Y151;I8U9;I8;Y151`, `TrimUMI,0`, `CreateFastqForIndexReads,1` and `AdapterBehavior,mask`. The first index read holds the 8 index bases and then 9 UMI bases. BCL Convert writes the whole index read into the I1 file, so the file holds both.

Submit four files: R1, R2, I1 and I2. The I2 file holds only the second index. It is submitted because every index read of the run is submitted once one of them is.

```
I1 record: ACGTACGT ACGGTCAAG
position:  0        8
           sampleBarcode 8, then umi 9

I2 record: AAGGCTAT
           sampleBarcode 8
```

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "I1", "readLength": 17 },
    { "fileType": "fastq", "readOrder": "I2", "readLength": 8 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "insert", "length": 151, "sequenceType": "random" }
    ],
    "R2": [
      { "segmentType": "insert", "length": 151, "sequenceType": "random" }
    ],
    "I1": [
      {
        "segmentType": "sampleBarcode",
        "length": 8,
        "sequenceType": "known",
        "sequences": ["ACGTACGT"],
        "reverseComplement": false
      },
      { "segmentType": "umi", "length": 9, "sequenceType": "random" }
    ],
    "I2": [
      {
        "segmentType": "sampleBarcode",
        "length": 8,
        "sequenceType": "known",
        "sequences": ["AAGGCTAT"],
        "reverseComplement": false
      }
    ]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

The two `sampleBarcode` entries declare the barcode of this sample, `ACGTACGT-AAGGCTAT`. They are the only values in this example that change per sample. `labData.barcode` may repeat the value, or be `null`. The I2 entry is the second index in forward orientation. On an instrument that reads the second index from the other strand, the I2 file shows its reverse complement, and `reverseComplement` is `true`.

### UMI in the second index read

Agilent SureSelect XT HS, converted with `Y151;I8;U10;Y151`, `TrimUMI,0`, `CreateFastqForIndexReads,1` and `AdapterBehavior,mask`. The UMI takes the place of the second index, so the second index read holds 10 UMI bases and nothing else.

Submit four files: R1, R2, I1 and I2. The I1 file holds only the index.

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 151 },
    { "fileType": "fastq", "readOrder": "I1", "readLength": 8 },
    { "fileType": "fastq", "readOrder": "I2", "readLength": 10 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "insert", "length": 151, "sequenceType": "random" }
    ],
    "R2": [
      { "segmentType": "insert", "length": 151, "sequenceType": "random" }
    ],
    "I1": [
      {
        "segmentType": "sampleBarcode",
        "length": 8,
        "sequenceType": "known",
        "sequences": ["ACGTACGT"],
        "reverseComplement": false
      }
    ],
    "I2": [{ "segmentType": "umi", "length": 10, "sequenceType": "random" }]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

The library has one index. Its `sampleBarcode` entry declares the barcode, `ACGTACGT`.

### MGI read barcode

R2 holds 100 template bases, then the UMI, then the two barcodes. The barcodes appear in the record as the reverse complement of the stored values.

```
R2 record: ...GGCTTAACGT ACGGTCAA CCAACGTGAT GGCTATCACG
              insert 100 |umi 8 | |barcode | |barcode |
position:                100      108        118
```

```json
{
  "sequencingLayout": "paired-end",
  "files": [
    { "fileType": "fastq", "readOrder": "R1", "readLength": 150 },
    { "fileType": "fastq", "readOrder": "R2", "readLength": 128 }
  ],
  "readStructure": {
    "R1": [
      { "segmentType": "insert", "length": 150, "sequenceType": "random" }
    ],
    "R2": [
      { "segmentType": "insert", "length": 100, "sequenceType": "random" },
      { "segmentType": "umi", "length": 8, "sequenceType": "random" },
      {
        "segmentType": "sampleBarcode",
        "length": 10,
        "sequenceType": "known",
        "sequences": ["ATCACGTTGG"],
        "reverseComplement": true
      },
      {
        "segmentType": "sampleBarcode",
        "length": 10,
        "sequenceType": "known",
        "sequences": ["CGTGATAGCC"],
        "reverseComplement": true
      }
    ]
  },
  "adapters": {
    "R1": { "sequences": ["<AdapterRead1>"], "trimmed": false },
    "R2": { "sequences": ["<AdapterRead2>"], "trimmed": false }
  }
}
```

`sequences` holds the two barcodes in forward orientation. `reverseComplement: true` says that the record shows `CCAACGTGAT` where the declared value is `ATCACGTTGG`. The sample barcode is the two entries in read order, `ATCACGTTGG-CGTGATAGCC`. The order of the parts inside the read barcode depends on the kit.

R2 has segments behind its insert, so its adapter bases may only be masked, and its `trimmed` is `false`. R1 may be trimmed. Its `trimmed` is then `true`.

### Long reads as BAM

A BAM holds only the insert in its read sequence, and its `labData` entry has no `readStructure` and no `adapters`. Before submission, the kit bases move into tags: a UMI into `RX`, its base qualities into `QX`. One record, before and after:

```
FASTQ record:  ACGTTGATTACAGGCTTAACG     umi 3, spacer 2, insert 16
BAM record:    SEQ   GATTACAGGCTTAACG    the insert only
               RX:Z  ACG                 the UMI. Two UMIs of one record: ACG-TGC, 5' first
               QX:Z  FFF                 the UMI's base qualities
```

Torrent Suite writes the tags of tag sequencing into `ZT` and `YT`. Move them into `RX` as `<ZT>-<YT>`. Where a UMI is still in the read sequence, for example in a nanopore BAM, extract it before submission.

## Translate Illumina settings with code

The function `read_structure_from_illumina` turns BCL Convert settings into `readStructure`, `adapters` and the `readLength` of each file. It refuses settings whose output the GRZ does not accept. Python, standard library only.

| Setting                        | What the function does with it                                                                                                                                                                          |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OverrideCycles`               | `Y` becomes `insert` and `U` becomes `umi`. `N` gives no segment. `I` gives a `sampleBarcode` segment. Index reads give layouts where one of them holds a UMI, or where `submit_index_reads` is 1       |
| `TrimUMI`                      | must be 0 where `U` cycles exist. The default 1 is refused                                                                                                                                              |
| `CreateFastqForIndexReads`     | must be 1 where index-read files are submitted                                                                                                                                                          |
| `AdapterRead1`, `AdapterRead2` | the `sequences` of `R1` and `R2` in `adapters`. Sequences joined with `+` become separate entries                                                                                                       |
| `AdapterBehavior`              | `trim` gives `"trimmed": true`, and `mask` gives `false`. `trim` is the default as soon as the sample sheet names an adapter. `trim` is refused for a read whose layout has a segment behind the insert |
| `Index`, `Index2`              | the value of a `sampleBarcode` segment                                                                                                                                                                  |

```python
import re

CYCLES = re.compile(r"([YUIN])(\d+)")


def is_on(value):
    """Sample sheet flags are written as 1 or 0, or as true or false."""
    return str(value).strip().lower() in ("1", "true")


def parse_override_cycles(value):
    """Return one list of (letter, count) pairs per read, in run order."""
    reads = []
    for part in value.strip().split(";"):
        part = part.strip()
        pairs = CYCLES.findall(part)
        if not pairs or "".join(letter + count for letter, count in pairs) != part:
            raise ValueError(f"cannot read the OverrideCycles part {part!r}")
        reads.append([(letter, int(count)) for letter, count in pairs])
    return reads


def adapter_sequences(value, key):
    """Split AdapterRead1 or AdapterRead2 at "+" into the sequences of adapters."""
    sequences = [part.strip() for part in value.split("+")]
    if not all(re.fullmatch("[ACGT]+", part) for part in sequences):
        raise ValueError(f"cannot read the adapter of {key}: {value!r}")
    return list(dict.fromkeys(sequences))         # each sequence once, in the given order


def read_structure_from_illumina(override_cycles, trim_umi=1,
                                 create_fastq_for_index_reads=0,
                                 adapter_read1=None, adapter_read2=None,
                                 adapter_behavior=None, kit_adapters=None,
                                 index=None, index2=None,
                                 index2_reverse_complement=None,
                                 umi_sequences=None, submit_index_reads=0):
    """Translate BCL Convert settings into readStructure, readLength and adapters.

    override_cycles               OverrideCycles, for example "U3N2Y146;I8;I8;U3N2Y146"
    trim_umi                      TrimUMI. The BCL Convert default is 1
    create_fastq_for_index_reads  CreateFastqForIndexReads. The default is 0
    adapter_read1, adapter_read2  AdapterRead1 and AdapterRead2, or None where the
                                  sample sheet names no adapter for that read
    adapter_behavior              AdapterBehavior: None, "mask" or "trim". BCL Convert
                                  trims where an adapter is named and this is None
    kit_adapters                  the adapter sequences of the kit documentation, for
                                  example {"R1": "...", "R2": "..."}. Used for a read
                                  whose adapter the sample sheet does not name
    index, index2                 the index values of this sample, in forward orientation
    index2_reverse_complement     True where the I2 file shows the reverse complement of index2
    umi_sequences                 the set of a nonrandom UMI kit, otherwise None
    submit_index_reads            1 to submit the index-read files although none holds a UMI

    A read with Y cycles is a data read: R1, then R2. Every other read is an
    index read: I1, then I2. Where one index-read file is submitted, all are.
    """
    behavior = str(adapter_behavior or "trim").strip().lower()
    if behavior not in ("mask", "trim"):
        raise ValueError(f"cannot read AdapterBehavior,{adapter_behavior}")
    reads = parse_override_cycles(override_cycles)
    has_umi = any(letter == "U" for read in reads for letter, _ in read)
    if has_umi and is_on(trim_umi):
        raise ValueError("TrimUMI,1 moves the UMI into the read name. Use TrimUMI,0.")
    index_umi = any("U" in letters and "Y" not in letters
                    for letters in ({letter for letter, _ in read} for read in reads))
    submit_index = index_umi or is_on(submit_index_reads)
    if submit_index and not is_on(create_fastq_for_index_reads):
        raise ValueError("index-read files need CreateFastqForIndexReads,1.")

    read_structure, read_length = {}, {}
    data_reads = index_reads = 0
    for read in reads:
        letters = {letter for letter, _ in read}
        if "Y" in letters:                        # a data read: R1, then R2
            if "I" in letters:
                raise ValueError("index cycles inside a data read are not handled")
            data_reads += 1
            key = f"R{data_reads}"
        else:                                     # an index read: I1, then I2
            index_reads += 1
            key = f"I{index_reads}"
            if not submit_index:
                continue                          # no index-read file is submitted

        segments = []
        for letter, count in read:
            if letter == "N":                     # not written to the file
                continue
            if letter == "Y":
                segment = {"segmentType": "insert", "length": count,
                           "sequenceType": "random"}
            elif letter == "U" and umi_sequences:
                if any(len(entry) != count for entry in umi_sequences):
                    raise ValueError(f"every UMI entry needs {count} bases")
                segment = {"segmentType": "umi", "length": count,
                           "sequenceType": "known",
                           "sequences": sorted(set(umi_sequences)),
                           "reverseComplement": False}
            elif letter == "U":
                segment = {"segmentType": "umi", "length": count,
                           "sequenceType": "random"}
            else:                                 # I: the index of this index read
                value = index if key == "I1" else index2
                reverse = False if key == "I1" else index2_reverse_complement
                if value is None or reverse is None:
                    raise ValueError(f"{key} holds index bases. Give the index value "
                                     "and, for I2, its orientation.")
                if len(value) != count:
                    raise ValueError(f"the index of {key} needs {count} bases")
                segment = {"segmentType": "sampleBarcode", "length": count,
                           "sequenceType": "known", "sequences": [value],
                           "reverseComplement": reverse}
            segments.append(segment)
        read_structure[key] = segments
        read_length[key] = sum(segment["length"] for segment in segments)

    named = {"R1": adapter_read1, "R2": adapter_read2}
    adapters = {}
    for key in ("R1", "R2"):
        if key not in read_structure:
            continue
        if named[key]:                            # BCL Convert masked or trimmed
            value, trimmed = named[key], behavior == "trim"
        elif kit_adapters and kit_adapters.get(key):   # BCL Convert did neither
            value, trimmed = kit_adapters[key], False
        else:
            raise ValueError(f"adapters needs the adapter of {key}. The sample sheet "
                             "names none, so give the kit's adapter in kit_adapters.")
        if trimmed and read_structure[key][-1]["segmentType"] != "insert":
            raise ValueError(f"{key} has a segment behind the insert, so its adapter "
                             "must not be trimmed. Use AdapterBehavior,mask.")
        adapters[key] = {"sequences": adapter_sequences(value, key), "trimmed": trimmed}

    order = sorted(read_structure, key=["R1", "R2", "I1", "I2"].index)  # file order
    return {"readStructure": {key: read_structure[key] for key in order},
            "readLength": {key: read_length[key] for key in order},
            "adapters": adapters}


def split_insert(segments, *leading):
    """Declare kit bases that the conversion counted as template (Y cycles).

    leading: (segmentType, length) pairs, 5' to 3', at the start of the insert.
    """
    position = next(i for i, segment in enumerate(segments)
                    if segment["segmentType"] == "insert")
    rest = segments[position]["length"] - sum(length for _, length in leading)
    if rest < 1:
        raise ValueError("the kit bases are longer than the insert")
    segments[position]["length"] = rest
    segments[position:position] = [
        {"segmentType": kind, "length": length, "sequenceType": "random"}
        for kind, length in leading
    ]


def parse_sample_sheet(text):
    """Return [BCLConvert_Settings] as a dict and the [BCLConvert_Data] rows."""
    section, header, settings, rows = None, None, {}, []
    for line in text.splitlines():
        cells = [cell.strip() for cell in line.split(",")]
        if not cells[0]:
            continue
        if cells[0].startswith("["):
            section, header = cells[0].strip("[]").lower(), None
        elif section == "bclconvert_settings":
            settings[cells[0].lower()] = cells[1] if len(cells) > 1 else ""
        elif section == "bclconvert_data" and header is None:
            header = [cell.lower() for cell in cells]
        elif section == "bclconvert_data":
            rows.append(dict(zip(header, cells)))
    return settings, rows


def read_structure_for_sample(sample_sheet_text, sample_id, **more):
    """Translate the settings that a v2 sample sheet holds for one sample.

    AdapterRead1 and AdapterRead2 come from the row of the sample where it
    names them, otherwise from [BCLConvert_Settings]. Where the sample sheet
    names no adapter, pass kit_adapters.
    """
    settings, rows = parse_sample_sheet(sample_sheet_text)
    row = next(row for row in rows if row.get("sample_id") == sample_id)
    override_cycles = row.get("overridecycles") or settings.get("overridecycles")
    if not override_cycles:
        raise ValueError("no OverrideCycles. Write the cycles of the run in that "
                         "notation, for example Y151;I8;I8;Y151.")
    return read_structure_from_illumina(
        override_cycles,
        trim_umi=settings.get("trimumi", 1),
        create_fastq_for_index_reads=settings.get("createfastqforindexreads", 0),
        adapter_read1=row.get("adapterread1") or settings.get("adapterread1") or None,
        adapter_read2=row.get("adapterread2") or settings.get("adapterread2") or None,
        adapter_behavior=settings.get("adapterbehavior") or None,
        index=row.get("index") or None,
        index2=row.get("index2") or None,
        **more,
    )
```

Call it with the values of your sample sheet, or with the sample sheet itself. In the calls below, replace `<AdapterRead1>` and `<AdapterRead2>` with your adapters. The function refuses the placeholders unchanged.

```python
result = read_structure_from_illumina("U3N2Y146;I8;I8;U3N2Y146", trim_umi=0,
                                      adapter_read1="<AdapterRead1>",
                                      adapter_read2="<AdapterRead2>",
                                      adapter_behavior="mask")
result["readStructure"]    # R1 and R2: umi 3, then insert 146
result["readLength"]       # {"R1": 149, "R2": 149}
result["adapters"]         # R1 and R2: the adapter, "trimmed": False

result = read_structure_for_sample(open("SampleSheet.csv").read(), "sample_1")
```

Five things are not in the settings, so you supply them.

- **The cycles of a run without `OverrideCycles`.** Write them in the same notation, for example `Y151;I8;I8;Y151`.
- **Kit bases that the conversion counts as template.** `split_insert` declares them. This is the example "Inline UMI with dark bases":

```python
result = read_structure_from_illumina("Y151;I8;I8;Y151",
                                      adapter_read1="<AdapterRead1>",
                                      adapter_read2="<AdapterRead2>",
                                      adapter_behavior="mask")
for key in ("R1", "R2"):
    split_insert(result["readStructure"][key], ("umi", 3), ("spacer", 2))
```

- **The adapter of a sample sheet that names none.** Pass the sequences of your kit documentation as `kit_adapters`. BCL Convert neither masked nor trimmed then, so `trimmed` is `false`. Where a later step trims with another tool, such as cutadapt or fastp, pass the adapters given to that tool as `kit_adapters`. Then set `trimmed` to `true` in the result.
- **The set of a nonrandom UMI kit.** Pass the entries as `umi_sequences`.
- **The orientation of the second index.** `labData.barcode` holds each index in forward orientation. Where the I2 file holds index bases, `index2_reverse_complement` says whether the file shows the reverse complement of that value. That is the case on an instrument that reads the second index from the other strand. `RunInfo.xml` marks such a read with `IsReverseComplement` where the instrument writes that flag. The I1 file shows the first index as stored, so the function uses `false` there.

## Check your files before you submit

Two commands answer most questions. Both read the FASTQ only.

**Record lengths of the whole file.** A fixed layout without trimmed adapters needs exactly one line of output:

```sh
zcat sample_R1.fastq.gz \
  | awk 'NR % 4 == 2 { n[length($0)]++ } END { for (l in n) print l, n[l] }'
```

- One length, equal to the number of cycles of that read, not counting `N` cycles: the UMI is in the read, and nothing was trimmed.
- One length, shorter than that by the UMI length: the UMI was cut out. Convert again with `TrimUMI,0`.
- Many lengths, the longest equal to the number of cycles: bases were trimmed. Where only adapter bases were removed, declare `"trimmed": true` for that read. Where the layout of that read has a segment behind the insert, convert again with `AdapterBehavior,mask`. Quality trimming also gives many lengths, and it is not accepted.

**Bases at one position, over the first 10,000 records.** The command looks at the eighth base, which is position 7. Change the `8` to look at another base:

```sh
zcat sample_R1.fastq.gz \
  | awk 'NR % 4 == 2 { n[substr($0, 8, 1)]++ }
         NR == 40000 { exit }
         END { for (b in n) print b, n[b] }'
```

- One base in nearly all records: this base is a constant kit base. Declare it as a `spacer` segment.
- All four bases in similar numbers: this base is UMI or template.

## What an error means

| Check that failed                                                                                 | What it usually means                                                                                                                      | What to do                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Layouts match files                                                                               | a read file has no layout, or a layout has no file                                                                                         | add the missing layout, or remove the layout of a file you do not submit                                                                                                                                                                  |
| Trimmed read ends in the insert                                                                   | `trimmed` is `true` for a read whose fixed layout has a segment behind the insert                                                          | convert again without trimming that read, for example with `AdapterBehavior,mask`. Then declare `"trimmed": false`                                                                                                                        |
| Variable layout trimmed                                                                           | `trimmed` is `false` for a read whose variable layout has a segment behind the insert                                                      | remove the adapter bases, so that every record ends with the last segment of the layout. Then declare `"trimmed": true`                                                                                                                   |
| Neighbouring segments differ (warning)                                                            | two `random` segments of the same type follow each other                                                                                   | where the kit adds one UMI or one spacer there, merge them into one segment with the summed `length`                                                                                                                                      |
| Minimum at most maximum                                                                           | `minLength` is above `maxLength`                                                                                                           | swap the bounds                                                                                                                                                                                                                           |
| Length sum                                                                                        | the lengths in a layout do not sum to `readLength`                                                                                         | recount. An `N` cycle is not in the file and has no segment                                                                                                                                                                               |
| Length range                                                                                      | in a layout with a variable segment, `readLength` is below the fixed lengths plus `minLength`, or above the fixed lengths plus `maxLength` | check `readLength` and the bounds of the variable segment. Where a segment follows the variable insert, `readLength` refers to the record after adapter removal                                                                           |
| Entry length                                                                                      | an entry of `sequences` does not have `length` bases                                                                                       | enter every entry with exactly `length` bases. Enter a 6-mer of a 7-cycle UMI together with the base that follows it                                                                                                                      |
| Record length                                                                                     | records are shorter or longer than declared                                                                                                | run the first command of the section above. The cause is a trimmed UMI, trimmed adapters without `"trimmed": true`, quality trimming, or a wrong `length`                                                                                 |
| Known sequence (warning)                                                                          | a `known` segment is not where you declared it                                                                                             | check the UMI length, the spacer base and `reverseComplement`                                                                                                                                                                             |
| Mate kit bases (warning)                                                                          | trimmed records lack the reverse complement of the mate's kit bases at their end                                                           | give the trimmer an adapter sequence that does not include that reverse complement. Make sure that it removes no base in front of the adapter. Or mask instead of trimming. Also check the segments in front of the insert of each layout |
| UMI tag complete                                                                                  | some records of a BAM carry `RX`, others do not                                                                                            | extract the UMI of every record, or of none                                                                                                                                                                                               |
| One cycle count per entry                                                                         | files of one `readOrder` have different `readLength`                                                                                       | put runs with different cycle counts into separate `labData` entries                                                                                                                                                                      |
| Schema: a `sampleBarcode` segment that is `random`, or has two entries                            | the barcode was declared without its value, or with two values                                                                             | declare it `known`, with the index in forward orientation as its one entry                                                                                                                                                                |
| Schema: an `R1` or `R2` layout without an `insert` segment                                        | a data read was declared without template                                                                                                  | check which file the layout belongs to                                                                                                                                                                                                    |
| Schema: an `R1` or `R2` layout with two `insert` segments                                         | the template of one file was declared as two segments                                                                                      | declare it as one `insert` segment                                                                                                                                                                                                        |
| Schema: `minLength` on a segment that is not an `insert`                                          | a kit segment was declared with a varying length                                                                                           | declare the number of cycles the kit prescribes as `length`                                                                                                                                                                               |
| Schema: no `adapters` in an entry with a FASTQ file                                               | the adapter was not declared                                                                                                               | add `adapters`; see [Conversion settings](#conversion-settings)                                                                                                                                                                           |
| Schema: `adapters` lacks a data read of `readStructure`, or has a key for a read without a layout | the keys of `adapters` do not match the data reads                                                                                         | give `adapters` one key for each of `R1` and `R2` in `readStructure`, and no other key                                                                                                                                                    |

# For the GRZ

## Order of the checks

Run the three steps in this order, and stop at the first step that reports an error.

1. **Schema checks.** Reject duplicate keys while parsing the metadata. Many JSON parsers otherwise keep the last of two equal keys without a message. Then validate against `grz-schema.json`. The schema also tests that `adapters` has a key for each data read of `readStructure` and no other key, and that a layout holds at most one `insert` segment.
2. **Metadata checks.** They compare fields of the submission and need no file access.
3. **Data checks.** The record length check runs on every record, on the pass that already computes the checksum. The known sequence check runs on the record sample of each file. The mate kit bases check runs on the record samples of the R1 file and the R2 file of one pair.

Extraction runs only after all three steps passed without an error.

## Reference code

Python, standard library only: the rejection of duplicate keys, the metadata checks that look at one layout or at `adapters`, the three FASTQ data checks, and the sample barcode.

```python
import gzip
import json
from itertools import islice

SAMPLE = 10_000
COMPLEMENT = str.maketrans("ACGTN", "TGCAN")


def reject_duplicate_keys(pairs):
    """Use as object_pairs_hook, so a repeated key is an error."""
    keys = [key for key, _ in pairs]
    repeated = sorted({key for key in keys if keys.count(key) > 1})
    if repeated:
        raise ValueError(f"duplicate keys: {repeated}")
    return dict(pairs)


def reverse_complement(seq):
    return seq.translate(COMPLEMENT)[::-1]


def positions(segments):
    """Return (segment, start, stop) as slice bounds into a record.

    Before the variable segment, the bounds count from the 5' end. Behind it,
    they count from the 3' end as negative indices, so they fit every record.
    """
    out, start = [], 0
    for index, seg in enumerate(segments):
        if "length" in seg:
            stop = start + seg["length"]
        else:  # the variable segment ends where the 3' part begins
            stop = -sum(later["length"] for later in segments[index + 1:])
        out.append((seg, start, stop or None))
        start = stop
    return out


def check_layout(segments):
    """Return the errors and the warnings of the metadata checks on one layout.

    The schema already tests that a layout holds at most one insert segment.
    """
    errors, warnings = [], []
    for left, right in zip(segments, segments[1:]):
        if (left["segmentType"] == right["segmentType"]
                and left["sequenceType"] == right["sequenceType"] == "random"):
            warnings.append("neighbouring segments differ: two random "
                            f"{left['segmentType']} segments follow each other")
    for index, seg in enumerate(segments):
        for entry in seg.get("sequences", []):
            if len(entry) != seg["length"]:
                errors.append(f"entry length: segment {index} has the entry {entry} "
                              f"of {len(entry)} bases, declared {seg['length']}")
    for seg in segments:
        if "length" in seg:
            continue
        if seg["minLength"] > seg["maxLength"]:
            errors.append("minimum at most maximum: minLength is above maxLength")
    return errors, warnings


def check_adapters(read_structure, adapters):
    """Return the errors of the metadata checks "Trimmed read ends in the insert"
    and "Variable layout trimmed".

    The schema already tests that adapters has a key for each data read of
    readStructure, and no other key.
    """
    errors = []
    for key, adapter in adapters.items():
        segments = read_structure[key]
        fixed = all("length" in seg for seg in segments)
        behind = segments[-1]["segmentType"] != "insert"   # a segment behind the insert
        if fixed and behind and adapter["trimmed"]:
            errors.append(f"trimmed read ends in the insert: {key} is trimmed, "
                          "and its fixed layout has a segment behind the insert")
        if not fixed and behind and not adapter["trimmed"]:
            errors.append(f"variable layout trimmed: {key} has a variable layout with "
                          "a segment behind the insert, and trimmed is false")
    return errors


def front_length(segments):
    """The summed length of the segments in front of the insert."""
    position = next(i for i, seg in enumerate(segments)
                    if seg["segmentType"] == "insert")
    return sum(seg["length"] for seg in segments[:position])


def read_sequences(path, limit=None):
    """Yield the read sequence of each record of a gzip FASTQ file."""
    stop = None if limit is None else 4 * limit
    with gzip.open(path, "rt") as handle:
        for line in islice(handle, 1, stop, 4):
            yield line.rstrip("\n")


def hamming(a, b):
    """Count the mismatches. An N counts as a mismatch."""
    return sum(x != y or x == "N" for x, y in zip(a, b))


def matches(bases, expected):
    """Under 6 bases only an exact match counts, from 6 bases on distance 1 too."""
    tolerance = 0 if len(expected) < 6 else 1
    return len(bases) == len(expected) and hamming(bases, expected) <= tolerance


def segment_matches(bases, seg):
    """Do the bases of one record match an entry of the segment's set?"""
    return any(
        matches(bases, reverse_complement(entry) if seg["reverseComplement"] else entry)
        for entry in seg["sequences"]
    )


def check_record_length(segments, sequences, trimmed=False):
    """Return an error for the first record of a wrong length, else None.

    trimmed: the value of trimmed in the entry of this read in adapters. Where
    it is true and the insert is the last segment, the lower bound drops to the
    segments in front of the insert.
    """
    fixed = sum(seg.get("length", 0) for seg in segments)
    variable = next((seg for seg in segments if "length" not in seg), {})
    low = fixed + variable.get("minLength", 0)
    high = fixed + variable.get("maxLength", 0)
    if trimmed and segments[-1]["segmentType"] == "insert":
        low = front_length(segments)  # adapter bases removed from the 3' end of the insert
    declared = str(low) if low == high else f"{low} to {high}"
    for number, seq in enumerate(sequences, start=1):
        if not low <= len(seq) <= high:
            return (f"record length: record {number} has {len(seq)} bases, "
                    f"declared {declared}")
    return None


def check_known_sequences(segments, sample):
    """Return one warning per known segment that matches in under 95 percent.

    Records that consist of N only are left out.
    """
    sample = [seq for seq in sample if seq.strip("N")]
    warnings = []
    for index, (seg, start, stop) in enumerate(positions(segments)):
        if seg["sequenceType"] != "known" or not sample:
            continue
        hits = sum(segment_matches(seq[start:stop], seg) for seq in sample)
        rate = hits / len(sample)
        if rate < 0.95:
            warnings.append(
                f"known sequence: segment {index} ({seg['segmentType']}) at "
                f"position {start} matches in {rate:.1%} of {len(sample)} records"
            )
    return warnings


def ends_in_mate(record, mate, k):
    """Are the last k bases of record the reverse complement of the first k of mate?

    The k bases compare as one segment, by the rules of the known sequence check.
    """
    return k == 0 or matches(record[-k:], reverse_complement(mate[:k]))


def check_mate_kit_bases(read_structure, adapters, sample_r1, sample_r2):
    """Return a warning where under 95 percent of the qualifying pairs end in
    the reverse complement of the mate's kit bases, in both directions.

    A pair qualifies where its two records, without trailing N, have the same
    length and each is at least 3 bases shorter than the summed length of its
    own layout.
    """
    if not all(adapters.get(key, {}).get("trimmed") for key in ("R1", "R2")):
        return []
    layouts = read_structure["R1"], read_structure["R2"]
    if any("length" not in seg for layout in layouts for seg in layout):
        return []                         # the check needs two fixed layouts
    k1, k2 = (front_length(layout) for layout in layouts)
    full1, full2 = (sum(seg["length"] for seg in layout) for layout in layouts)
    pairs = []
    for r1, r2 in zip(sample_r1, sample_r2):
        r1, r2 = r1.rstrip("N"), r2.rstrip("N")      # trailing N are removed first
        if not (r1 and r2):
            continue                                 # a record of N only
        if len(r1) == len(r2) and len(r1) <= full1 - 3 and len(r2) <= full2 - 3:
            pairs.append((r1, r2))                   # a qualifying pair
    if not (k1 or k2) or not pairs:
        return []                         # no kit bases, or no qualifying pair
    hits = sum(ends_in_mate(r1, r2, k2) and ends_in_mate(r2, r1, k1)
               for r1, r2 in pairs)
    rate = hits / len(pairs)
    if rate < 0.95:
        return [f"mate kit bases: {rate:.1%} of {len(pairs)} qualifying pairs end in "
                "the reverse complement of the mate's kit bases"]
    return []


def sample_barcode(lab_data):
    """Return the sample barcode. A sampleBarcode segment wins over labData.barcode."""
    parts = [seg["sequences"][0]
             for key in ("R1", "R2", "I1", "I2")
             for seg in lab_data.get("readStructure", {}).get(key, [])
             if seg["segmentType"] == "sampleBarcode"]
    return "-".join(parts) if parts else lab_data.get("barcode")


FGBIO_LETTER = {"insert": "T", "umi": "M", "sampleBarcode": "S", "spacer": "S"}


def fgbio_read_structure(segments, trimmed=False):
    """One fgbio read structure for one layout, for example 3M2S146T.

    trimmed: the value of trimmed in adapters. The insert of a trimmed read
    varies in length.
    """
    variable = ["length" not in seg or (trimmed and seg["segmentType"] == "insert")
                for seg in segments]
    if any(variable[:-1]):
        raise ValueError("fgbio allows a variable segment only at the end")
    return "".join(f"{'+' if vary else seg['length']}{FGBIO_LETTER[seg['segmentType']]}"
                   for seg, vary in zip(segments, variable))


def dragen_umi_library_type(read_structure):
    """Return the value for --umi-library-type, or None without a UMI."""
    kinds = {}
    for read_order, segments in read_structure.items():
        for seg in segments:
            if seg["segmentType"] == "umi":
                kinds.setdefault(read_order, set()).add(seg["sequenceType"])
    if not kinds:
        return None
    found = set().union(*kinds.values())
    if len(found) != 1:
        raise ValueError("known and random UMI segments in one library")
    scheme = found.pop()
    if {"R1", "R2"} <= kinds.keys():
        return "random-duplex" if scheme == "random" else "nonrandom-duplex"
    if len(kinds) == 1 and scheme == "random":
        return "random-simplex"
    raise ValueError("no DRAGEN library type for this UMI placement")
```

Use for one file:

```python
metadata = json.loads(text, object_pairs_hook=reject_duplicate_keys)

lab_data = metadata["donors"][0]["labData"][0]    # one labData entry
read_structure, adapters = lab_data["readStructure"], lab_data["adapters"]
adapter_errors = check_adapters(read_structure, adapters)  # needs no file

fastq = lab_data["sequenceData"]["files"][0]       # one FASTQ file of that entry
path = fastq["filePath"]                           # relative to the files directory
read_order = fastq.get("readOrder", "R1")

segments = read_structure[read_order]
trimmed = adapters.get(read_order, {}).get("trimmed", False)  # index reads: False
layout_errors, layout_warnings = check_layout(segments)   # needs no file

error = check_record_length(segments, read_sequences(path), trimmed)  # every record
sample = list(read_sequences(path, SAMPLE))                   # first 10,000 records
warnings = check_known_sequences(segments, sample)
```

Use for the R1 file and the R2 file of one pair:

```python
sample_r1 = list(read_sequences(path_r1, SAMPLE))
sample_r2 = list(read_sequences(path_r2, SAMPLE))
warnings = check_mate_kit_bases(read_structure, adapters, sample_r1, sample_r2)
```

## Pipeline parameters

Every parameter below is derived from `readStructure` and `adapters`. Nothing is read from the file to decide it.

**fgbio read structures.** One string per submitted file, in the order of the files. `insert` maps to `T` and `umi` to `M`. `sampleBarcode` and `spacer` both map to `S`, which skips the bases. A variable segment maps to `+`. fgbio allows `+` only in the last segment, so a layout with segments behind its variable segment needs another extraction step.

Where `trimmed` is `true` for a read, its records vary in length, so its insert maps to `+` too. A trimmed read with a fixed layout always ends in the insert, so this `+` is the last segment.

| Example                      | Files          | Read structures           |
| ---------------------------- | -------------- | ------------------------- |
| WGS without UMI              | R1, R2         | `+T` `+T`                 |
| Inline UMI                   | R1, R2         | `3M146T` `3M146T`         |
| Inline UMI with dark bases   | R1, R2         | `3M2S146T` `3M2S146T`     |
| Nonrandom UMI                | R1, R2         | `7M143T` `7M143T`         |
| UMI behind the index         | R1, R2, I1, I2 | `151T` `151T` `8S9M` `8S` |
| UMI in the second index read | R1, R2, I1, I2 | `151T` `151T` `8S` `10M`  |
| MGI read barcode             | R1, R2         | `150T` `100T8M10S10S`     |

Extraction into an unaligned BAM, shown for "Inline UMI with dark bases". fgbio writes the `M` bases into `RX` and drops the `S` bases:

```sh
fgbio FastqToBam \
  --input sample_R1.fastq.gz sample_R2.fastq.gz \
  --read-structures 3M2S146T 3M2S146T \
  --sample "$DONOR" --library "$LABDATA" \
  --output sample.unaligned.bam
```

For "UMI behind the index" the I1 and I2 files are the third and fourth inputs, with the read structures `8S9M` and `8S`. For "UMI in the second index read" they have `8S` and `10M`.

The pipeline does not take the sample barcode from the read bases. The file is already demultiplexed, and the declaration holds the value. The function `sample_barcode` in the reference code returns it: the entries of the `sampleBarcode` segments where they exist, otherwise `labData.barcode`. So the pipeline never applies `reverseComplement` to a barcode, and only the known sequence check reads that field. One case is left: a nonrandom UMI with `reverseComplement: true`. There the whitelist is written as the reverse complement of `sequences`.

**DRAGEN.** `--umi-library-type` follows from the placement and the scheme of the `umi` segments. The function `dragen_umi_library_type` in the reference code implements this table:

| `umi` segments     | `sequenceType` | `--umi-library-type`                                                                                                        |
| ------------------ | -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| on R1 and R2       | `random`       | `random-duplex`                                                                                                             |
| on R1 and R2       | `known`        | `nonrandom-duplex`, with `--umi-nonrandom-whitelist` written from the `sequences` of the `umi` segments, one entry per line |
| on one `readOrder` | `random`       | `random-simplex`                                                                                                            |
| none               |                | UMI collapsing is off                                                                                                       |

DRAGEN names no library type for a single-ended nonrandom UMI. The function raises an error for that case. The GRZ treats every dual UMI as duplex-capable, as DRAGEN does.

The DRAGEN 4.5 documentation describes `--umi-source=sequence` with `--umi-sequence-position-r1` and `--umi-sequence-position-r2` for a UMI that is still in the read sequence. The value is the UMI length, or a string such as `U7N1` where a spacer follows the UMI. For "Inline UMI" that gives `3`, and for "Inline UMI with dark bases" it gives `U3N2`. This mapping has not been tested on real data.

**Adapter trimming.** A pipeline that trims adapters takes the `sequences` of each data read from `adapters`. cutadapt, for example, takes them as `-a` for R1 and as `-A` for R2, one option per sequence. Where `trimmed` is `false`, the records still hold the adapter bases, possibly masked with `N`. Where it is `true`, the adapter bases were removed before submission, and a record may still end in `N`. In both cases the reverse complement of the mate's kit bases stays behind the template where the template ends before the read does. Adapter trimming with these sequences leaves those bases in the record.

## Test vectors

**Translating Illumina settings.** These rows test `read_structure_from_illumina`, and the last row tests `read_structure_for_sample`. "The placeholder adapters" are `GATCGATCGATC` as `AdapterRead1` and `CTAGCTAGCTAG` as `AdapterRead2`. Where a row expects an example, the placeholder adapters take the place of `<AdapterRead1>` and `<AdapterRead2>` in it.

| Settings                                                                                                                                                                                          | Expected                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `Y151;I8;I8;Y151` with the placeholder adapters and the default `AdapterBehavior`                                                                                                                 | the example "WGS without UMI"                                                                        |
| `U3N2Y146;I8;I8;U3N2Y146` with `TrimUMI,0`, the placeholder adapters and `AdapterBehavior,mask`                                                                                                   | the example "Inline UMI"                                                                             |
| `Y151;I8;I8;Y151` with the placeholder adapters and `AdapterBehavior,mask`, then `split_insert` with `umi` 3 and `spacer` 2                                                                       | the example "Inline UMI with dark bases"                                                             |
| `U7N1Y143;I8;I8;U7N1Y143` with `TrimUMI,0`, the two placeholder UMI entries as `umi_sequences`, the placeholder adapters and `AdapterBehavior,mask`                                               | the example "Nonrandom UMI"                                                                          |
| `Y151;I8U9;I8;Y151` with `TrimUMI,0`, `CreateFastqForIndexReads,1`, the indexes `ACGTACGT` and `AAGGCTAT`, `index2_reverse_complement` false, the placeholder adapters and `AdapterBehavior,mask` | the example "UMI behind the index"                                                                   |
| `Y151;I8;U10;Y151` with `TrimUMI,0`, `CreateFastqForIndexReads,1`, the index `ACGTACGT`, the placeholder adapters and `AdapterBehavior,mask`                                                      | the example "UMI in the second index read"                                                           |
| `Y151;I8;I8;Y151` without `AdapterRead1` and `AdapterRead2`, and with the placeholder adapters as `kit_adapters`                                                                                  | `adapters` with the placeholder adapters and `"trimmed": false`                                      |
| `Y151;I8;I8;Y151` with `AdapterRead1,GATCGATCGATC+GGCCGGCCGGCC` and the placeholder `AdapterRead2`                                                                                                | `sequences` of R1 holds two entries                                                                  |
| `Y143U7;I8;I8;Y151` with `TrimUMI,0`, the placeholder adapters and `AdapterBehavior,mask`                                                                                                         | R1: `insert` 143, then `umi` 7, `"trimmed": false`                                                   |
| `U7N1Y143;I8;I8;U7N1Y143` with the default `TrimUMI,1`                                                                                                                                            | error                                                                                                |
| `Y151;I8U9;I8;Y151` with `TrimUMI,0` and the default `CreateFastqForIndexReads,0`                                                                                                                 | error                                                                                                |
| `Y151;I8U9;I8;Y151` with `TrimUMI,0`, `CreateFastqForIndexReads,1` and the first index only                                                                                                       | error, the I2 file holds the second index                                                            |
| `Y151;I8;I8;Y151` without `AdapterRead1` and `AdapterRead2`, and without `kit_adapters`                                                                                                           | error, `adapters` needs the adapter of each data read                                                |
| `Y143U7;I8;I8;Y151` with `TrimUMI,0`, the placeholder adapters and the default `AdapterBehavior`                                                                                                  | error, R1 has a segment behind the insert                                                            |
| a sample sheet with `Y151;I8;I8;Y151` and the placeholder adapters in `[BCLConvert_Settings]`, and `GGCCGGCCGGCC` in the `AdapterRead1` column of the sample's row                                | `sequences` of R1 holds `GGCCGGCCGGCC` only. R2 holds the placeholder `AdapterRead2` of the settings |

**Data checks on whole records.** The layout is that of the example "Nonrandom UMI" with a shorter insert: `umi` 7 `known` with the entries `ACGTACG` and `TTGCAAT`, then `insert` 13. The declared record length is 20.

| Record                  | Check                                | Expected                                                                                  |
| ----------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------- |
| `ACGTACGGATTACAGGCTTA`  | Record length                        | pass, 20 bases                                                                            |
| `ACGTACGGATTACAGGCTT`   | Record length                        | error, 19 bases                                                                           |
| `ACGTACGGATTACAGGCTT`   | Record length with `"trimmed": true` | pass, 19 bases                                                                            |
| `ACGTACG`               | Record length with `"trimmed": true` | pass, the lower bound: the `umi` segment in front of the insert                           |
| `ACGTAC`                | Record length with `"trimmed": true` | error, below 7                                                                            |
| `ACGTACGGATTACAGGCTTAC` | Record length with `"trimmed": true` | error, above 20                                                                           |
| `ACGTACGGATTACAGGCTTA`  | Known sequence                       | match, the 7-mer                                                                          |
| `TTGCAATGATTACAGGCTTA`  | Known sequence                       | match, the 6-mer with its `T`                                                             |
| `ACGTACAGATTACAGGCTTA`  | Known sequence                       | match, one mismatch on 7 bases                                                            |
| `ACGAACAGATTACAGGCTTA`  | Known sequence                       | no match, two mismatches                                                                  |
| `ACGTACNGATTACAGGCTTA`  | Known sequence                       | match, the `N` is the one allowed mismatch                                                |
| `NNNNNNNNNNNNNNNNNNNN`  | Known sequence                       | left out, the record consists of `N` only. It counts neither as a match nor as a mismatch |

**Record length with a variable segment.** The layout is `umi` 12 `random`, then `insert` `random` with `minLength` 1 and `maxLength` 600.

| Record    | Check                                | Expected                                                        |
| --------- | ------------------------------------ | --------------------------------------------------------------- |
| 13 bases  | Record length                        | pass, the lower bound                                           |
| 612 bases | Record length                        | pass, the upper bound                                           |
| 12 bases  | Record length                        | error, below 13                                                 |
| 613 bases | Record length                        | error, above 612                                                |
| 12 bases  | Record length with `"trimmed": true` | pass, the lower bound: the `umi` segment in front of the insert |

**Counting from the 3' end.** The layout is `umi` 4 `random`, then `insert` `random` with `minLength` 1 and `maxLength` 600, then `umi` 4 `random`, then `spacer` 3 `known` with the entry `ACG`. `positions` gives the slices `[0:4]`, `[4:-7]`, `[-7:-3]` and `[-3:]`. The layout has segments behind its variable insert, so `trimmed` is `true` for this read.

| Record                 | Check                                | Expected                                                                                  |
| ---------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------- |
| `GCTAGATTACACCGGACG`   | Known sequence                       | match, the spacer is the last 3 bases                                                     |
| `GCTAGATTACAGGCCGGACG` | Known sequence                       | match, a longer insert moves the spacer, and counting from the 3' end follows it          |
| `GCTAGATTACACCGGACGT`  | Known sequence                       | no match, one more base behind the spacer shifts the 3' part                              |
| `GCTACCGGACG`          | Record length with `"trimmed": true` | error, 11 bases, below 12. The insert is not the last segment, so the normal bounds apply |

**Matching one segment.** These rows test the known sequence check on the bases of one segment.

| Bases in the record | Entry in `sequences` | `reverseComplement` | Expected                              |
| ------------------- | -------------------- | ------------------- | ------------------------------------- |
| `ACGTACGA`          | `ACGTACGT`           | `false`             | match, distance 1 on 8 bases          |
| `ACGTACAA`          | `ACGTACGT`           | `false`             | no match, distance 2                  |
| `ACGTT`             | `ACGTA`              | `false`             | no match, 5 bases need an exact match |
| `CCAACGTGAT`        | `ATCACGTTGG`         | `true`              | match                                 |
| `ATCACGTTGG`        | `ATCACGTTGG`         | `true`              | no match                              |

**Mate kit bases.** These rows test the function `check_mate_kit_bases` on a sample of one pair. The layout of R1 and of R2 is `umi` 3 `random`, then `insert` 17, so the summed length is 20. `trimmed` is `true` for both reads. The pair comes from a molecule with the R1 UMI `ACG`, the template `GATTACAGGC` and the R2 UMI `TTC`. Two rows name another template.

| R1 record                                                              | R2 record                                       | Expected                                                                                                                                                            |
| ---------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACGGATTACAGGCGAA`                                                     | `TTCGCCTGTAATCCGT`                              | match, `GAA` and `CGT` are the reverse complements of `TTC` and `ACG`                                                                                               |
| `ACGGATTACAGGCGAANN`                                                   | `TTCGCCTGTAATCCGT`                              | match, the trailing `N` are removed before the comparison                                                                                                           |
| `ACGGATTACAGGC`                                                        | `TTCGCCTGTAATC`                                 | no match, trimming removed the reverse complement of the mate's UMI                                                                                                 |
| `ACGGATTACAGGCGAA`                                                     | `TTCGCCTGTAATCCGA`                              | no match, the R2 record ends in `CGA`, not `CGT`, and the check compares in both directions                                                                         |
| `ACGGATTACAGGCGNA`                                                     | `TTCGCCTGTAATCCGT`                              | no match, the `N` counts as a mismatch                                                                                                                              |
| `ACGGATTACAGGCTGAA`                                                    | `TTCAGCCTGTAATCCGT`                             | match, with the template `GATTACAGGCT`. Both records have 17 bases, exactly 3 fewer than the summed length                                                          |
| `ACGGATTACAGGCGAA`                                                     | `TTCGCCTGTAATC`                                 | does not run, the two records have different lengths, so the pair does not qualify                                                                                  |
| `ACGGATTACAGGCTTAAC`                                                   | `TTCACGTTAAGCCTGTAA`                            | does not run, with the template `GATTACAGGCTTAACGT`. Both records hold template only, and a trimmer shortened them to 18 bases, only 2 fewer than the summed length |
| `ACGGATTACAGGCGAAGATC`                                                 | `TTCGCCTGTAATCCGTCTAG`                          | does not run, both records have the summed length of 20 bases, so the pair does not qualify                                                                         |
| `NNNNNNNNNNNNNNNN`                                                     | `NNNNNNNNNNNNNNNN`                              | does not run, a pair with a record of `N` only does not qualify                                                                                                     |
| `ACGGATTACAGGC`                                                        | `TTCGCCTGTAATC`, with `"trimmed": false` for R2 | does not run, the check needs `trimmed` `true` for both reads                                                                                                       |
| `ACGGATTACAGGC`, with an R1 insert of `minLength` 1 and `maxLength` 17 | `TTCGCCTGTAATC`                                 | does not run, the check needs two fixed layouts                                                                                                                     |

A sample in which 19 of 20 qualifying pairs match passes. A sample in which 18 of 20 match gives a warning.

**Metadata checks on one layout.** These rows test the function `check_layout`.

| Layout                                                                                 | Check                        | Expected                                         |
| -------------------------------------------------------------------------------------- | ---------------------------- | ------------------------------------------------ |
| `umi` 4 `random`, `umi` 4 `random`, `insert` 143                                       | Neighbouring segments differ | warning                                          |
| `insert` 100, `umi` 8 `random`, `sampleBarcode` 10 `known`, `sampleBarcode` 10 `known` | Neighbouring segments differ | pass, two `known` segments may follow each other |
| `insert` with `minLength` 151 and `maxLength` 151                                      | Minimum at most maximum      | pass, a fixed length                             |
| `insert` with `minLength` 200 and `maxLength` 100                                      | Minimum at most maximum      | error                                            |
| `umi` 7 `known` with the entry `TTGCAA`, then `insert` 143                             | Entry length                 | error, the 6-mer lacks the `T` that follows it   |

**Metadata checks on adapters.** These rows test the function `check_adapters`. "The variable layout" is R1: `umi` 4 `random`, then `insert` with `minLength` 1 and `maxLength` 600, then `umi` 4 `random`.

| `readStructure`                                                             | `adapters`                                            | Check                           | Expected                                                     |
| --------------------------------------------------------------------------- | ----------------------------------------------------- | ------------------------------- | ------------------------------------------------------------ |
| the example "MGI read barcode"                                              | R1 with `"trimmed": false`, R2 with `"trimmed": true` | Trimmed read ends in the insert | error, R2 has segments behind the insert                     |
| the example "MGI read barcode"                                              | R1 with `"trimmed": true`, R2 with `"trimmed": false` | Trimmed read ends in the insert | pass                                                         |
| the variable layout                                                         | R1 with `"trimmed": true`                             | Variable layout trimmed         | pass                                                         |
| the variable layout                                                         | R1 with `"trimmed": false`                            | Variable layout trimmed         | error, the layout describes the record after adapter removal |
| R1: `umi` 12 `random`, then `insert` with `minLength` 1 and `maxLength` 600 | R1 with `"trimmed": false`                            | Variable layout trimmed         | pass, the insert is the last segment                         |

**Sample barcode.** These rows test the function `sample_barcode`.

| `labData.barcode`     | `sampleBarcode` entries            | Expected                |
| --------------------- | ---------------------------------- | ----------------------- |
| `"TAAGGCGA-AAGGCTAT"` | none                               | `TAAGGCGA-AAGGCTAT`     |
| `null`                | I1 `ACGTACGT`, I2 `AAGGCTAT`       | `ACGTACGT-AAGGCTAT`     |
| `"ACGTACGT-AAGGCTAT"` | I1 `ACGTACGT`, I2 `AAGGCTAT`       | `ACGTACGT-AAGGCTAT`     |
| `null`                | R2 `ATCACGTTGG`, then `CGTGATAGCC` | `ATCACGTTGG-CGTGATAGCC` |
| `null`                | none                               | `null`                  |

**Schema checks.** Each row is one `labData` entry, or one `readStructure` or `adapters` value of it.

| Value                                                                             | Expected                                          |
| --------------------------------------------------------------------------------- | ------------------------------------------------- |
| each of the seven examples, with the placeholders replaced                        | valid                                             |
| the key `R1` twice                                                                | rejected while parsing, before the schema runs    |
| the key `R1` twice in `adapters`                                                  | rejected while parsing, before the schema runs    |
| a `labData` entry with a FASTQ file and without `adapters`                        | invalid                                           |
| `adapters` in a `labData` entry without a FASTQ file                              | invalid                                           |
| `readStructure` with `R1` and `R2`, and `adapters` with `R1` only                 | invalid, R2 has no adapter                        |
| `readStructure` with `R1` only, and `adapters` with `R1` and `R2`                 | invalid, R2 has no layout                         |
| `readStructure` with `R1`, `R2`, `I1` and `I2`, and `adapters` with `R1` and `R2` | valid, an index read has no adapter               |
| `adapters` without a key                                                          | invalid                                           |
| an `adapters` key `I1`                                                            | invalid                                           |
| an adapter without `trimmed`                                                      | invalid                                           |
| an adapter with an empty `sequences`                                              | invalid                                           |
| an adapter sequence with an `N`                                                   | invalid                                           |
| a key other than `R1`, `R2`, `I1` and `I2`                                        | invalid                                           |
| an object without a key                                                           | invalid                                           |
| `readStructure` in a `labData` entry without a FASTQ file                         | invalid                                           |
| an `I1` layout with a `sampleBarcode` segment only                                | valid, an index-read file may hold only the index |
| an `insert` segment with `sequenceType` `known`                                   | invalid                                           |
| a `known` segment without `reverseComplement`                                     | invalid                                           |
| a `random` segment with `sequences`                                               | invalid                                           |
| a `sampleBarcode` segment with `sequenceType` `random`                            | invalid                                           |
| a `sampleBarcode` segment with two entries in `sequences`                         | invalid                                           |
| a segment with both `length` and `minLength`                                      | invalid                                           |
| an `R1` layout without an `insert` segment                                        | invalid                                           |
| an `R1` layout with two `insert` segments                                         | invalid                                           |
| an `I1` layout with an `insert` segment                                           | invalid                                           |
| a `umi` segment with `minLength` and `maxLength`                                  | invalid                                           |
| `sequences` with the same entry twice                                             | invalid                                           |
