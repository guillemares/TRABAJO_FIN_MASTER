# MEROPS Database Processing Report

Date: 18 November 2026

This report documents the changes made while preparing the MEROPS data for a peptide-protease or protease-inhibitor machine-learning dataset.

## Source location

- MEROPS archive: [DATABASES/MEROPS/meropsweb121.tar.gz](DATABASES/MEROPS/meropsweb121.tar.gz)
- Generated tables: [DATABASES/MEROPS/tables](DATABASES/MEROPS/tables)
- Processing notebook: [DATABASES/SCRIPTS/treat_databases.ipynb](DATABASES/SCRIPTS/treat_databases.ipynb)

## Database relationships

```text
peptidase_inhibitor_complex
        |
        +-- peptidase_code
        |
        +-- inhibitor_code
        |
        +-- Ki

domain
        |
        +-- code = peptidase_code or inhibitor_code
        |
        +-- sequence_id

sequence
        |
        +-- sequence_id
        |
        +-- sequence

alternative_id
        |
        +-- sequence_id
        |
        +-- alternative_id
        |
        +-- type = uniprot

seq_pdb
        |
        +-- uniprot_acc
        |
        +-- pdb_id
```

The direct relationship is therefore:

```text
MEROPS code
  -> domain.code
  -> domain.sequence_id
  -> sequence.sequence_id
  -> sequence.sequence
  -> alternative_id.sequence_id
  -> alternative_id.alternative_id
  -> seq_pdb.uniprot_acc
  -> seq_pdb.pdb_id
```

Not every MEROPS code has a corresponding sequence or PDB structure.

## Generated tables

### `peptidase_inhibitor_sequences.csv`

[peptidase_inhibitor_sequences.csv](DATABASES/MEROPS/tables/peptidase_inhibitor_sequences.csv) contains one preferred protease sequence and one preferred inhibitor sequence for each mapped complex.

Current size:

- 1,389 rows
- Approximately 1.2 MB

Main columns:

| Column | Description |
|---|---|
| `complex_id` | MEROPS complex identifier |
| `peptidase_code` | MEROPS peptidase code |
| `peptidase_sequence_id` | Selected sequence identifier |
| `peptidase_sequence` | Full peptidase amino-acid sequence |
| `peptidase_protein` | Protein name |
| `peptidase_start`, `peptidase_end` | Domain or sequence boundaries |
| `inhibitor_code` | MEROPS inhibitor code |
| `inhibitor_sequence_id` | Selected sequence identifier |
| `inhibitor_sequence` | Full inhibitor amino-acid sequence |
| `inhibitor_protein` | Inhibitor protein name |
| `inhibitor_start`, `inhibitor_end` | Domain or sequence boundaries |
| `Ki` | Original text Ki value |
| `ref` | Source reference |

The preferred mapping uses a holotype sequence when available. Otherwise, the sequence ID is sorted alphabetically and the first result is selected.

This selection is only a representative sequence. It does not prove that the exact experimentally used protein was represented.

## Source table summary

### `peptidase_inhibitor_complex.csv`

[peptidase_inhibitor_complex.csv](DATABASES/MEROPS/tables/peptidase_inhibitor_complex.csv) contains:

- 6,483 data rows
- 6,483 unique complex IDs
- 991 unique peptidase codes
- 1,675 unique inhibitor codes
- 1,639 rows with a non-empty Ki value
- 4,844 rows without Ki

The original `Ki` column is text rather than numeric. Examples include:

- `5 nM`
- `0.6 fM`
- `100,000 pM`
- `1.5 nM`
- `0.2-0.4 nM`
- `nM`

It is therefore necessary to distinguish:

1. Present versus missing Ki.
2. A numeric value in a recognized unit.
3. A range.
4. Text that does not contain a numeric concentration.

For plotting only, ranges may be represented by a geometric midpoint after conversion to molar concentration.

### `domain.csv`

The `domain` source table connects MEROPS codes to sequence IDs.

- 894 of 991 peptidase codes have a domain mapping.
- 477 of 1,675 inhibitor codes have a domain mapping.
- 1,368 MEROPS codes across both types have mappings.

There are 5,094 complex rows for which at least one side has no mapping.

This is the reason [peptidase_inhibitor_sequences.csv](DATABASES/MEROPS/tables/peptidase_inhibitor_sequences.csv) contains only 1,389 of the 6,483 source complexes.

## Partner reuse

The complex table is a bipartite graph:

- Each row connects a peptidase code to an inhibitor code.
- A peptidase can interact with several inhibitors.
- An inhibitor can interact with several peptidases.

Verified counts:

| Metric | Result |
|---|---:|
| Peptidases with one inhibitor | 345 |
| Peptidases with more than one inhibitor | 646 |
| Maximum inhibitor partners for one peptidase | 257 |
| Inhibitors with one peptidase | 968 |
| Inhibitors with more than one peptidase | 707 |
| Maximum peptidase partners for one inhibitor | 356 |

The notebook includes:

- Histograms of partner counts.
- Circular charts comparing one partner with multiple partners.
- Bar charts for the top 15 peptidases and inhibitors by partner count.

## Sequence coverage

### `sequence.csv`

[sequence.csv](DATABASES/MEROPS/tables/sequence.csv):

- 1,256,053 data rows plus header
- Approximately 2.9 GB
- 1.4 GB when stored as Parquet

The sequence file must not be loaded entirely into memory. The notebook streams it and retains only the sequence IDs required by the pair table.

The compact pair table contains 159,447 unique sequence IDs from the full-domain mapping check, although this is not the number stored in the final one-representative-per-code table.

## PDB coverage

The PDB mapping was evaluated for the 1,389 compact pair rows:

| Side | In PDB | Not in PDB |
|---|---:|---:|
| Peptidase | 537 | 852 |
| Inhibitor | 202 | 1,187 |

This means PDB coverage is based on a UniProt accession connected to a sequence ID and is not proof of an experimentally resolved peptidase-inhibitor complex. A PDB ID can represent a homolog or a different chain.

The source [seq_pdb.csv](DATABASES/MEROPS/tables/seq_pdb.csv) contains approximately 48,417 PDB mapping rows.

## Notebook additions

The following sections were appended to [treat_databases.ipynb](DATABASES/SCRIPTS/treat_databases.ipynb):

1. MEROPS sequence-pair construction.
2. Ki availability and unit parsing plots.
3. Ki histograms.
4. Sequence and PDB coverage plots.
5. Peptidase and inhibitor partner-count histograms.
6. Circular charts for entities with one versus multiple partners.
7. Top peptidase and inhibitor partner bar charts.

## Modeling implications

- Treat `Ki` as text initially; clean and convert it to numeric molar concentration before training.
- Preserve missing, range, and unrecognized Ki values separately.
- Do not assume PDB availability means the exact complex is resolved.
- Account for duplicate MEROPS codes, organisms, and sequence variants.
- Split train, validation, and test data by protein or complex group to reduce homologous leakage.
- The 1,389 paired rows are a strongly filtered subset of the 6,483 source complexes.
