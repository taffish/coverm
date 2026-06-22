# coverm

`coverm` packages [CoverM](https://github.com/wwood/CoverM) for TAFFISH.

Package identity:

- name: `coverm`
- command: `taf-coverm`
- kind: `tool`
- version: `0.7.0-r1`
- license: Apache-2.0 for TAFFISH packaging
- upstream: <https://github.com/wwood/CoverM>

## What This App Packages

CoverM is a configurable DNA read coverage and relative-abundance calculator
for metagenomics. It can calculate coverage for genomes/MAGs with
`coverm genome`, calculate coverage for individual contigs with
`coverm contig`, generate BAM files with `coverm make`, threshold alignments
with `coverm filter`, and dereplicate/cluster genomes with `coverm cluster`.

This TAFFISH image installs CoverM `0.7.0` from Bioconda and keeps automatic
command mode enabled so the same container can expose CoverM and its important
runtime helpers.

## Scope

This app supports:

- `coverm genome`, `coverm contig`, `coverm make`, `coverm filter`,
  `coverm cluster`, and `coverm shell-completion`
- BAM-based coverage paths and raw FASTA/FASTQ read mapping paths
- common mapper/runtime helpers: `samtools`, `minimap2`, `bwa`, `bwa-mem2`,
  `strobealign`, `skani`, `fastANI`, and `starcode`
- native `linux/amd64` and `linux/arm64` container builds through Bioconda

This app does not:

- download or bundle any production metagenome, MAG set, reference database, or
  remote demo data
- replace production-scale validation of abundance estimates, dereplication
  parameters, or mapper choices
- promise that every legacy optional backend is smoke-tested; CoverM 0.7.0
  defaults to `skani` for preclustering and ANI, and that path is the packaged
  cluster focus

## Container Contents

- `coverm`: main CoverM command
- `samtools`: BAM/SAM/CRAM sorting, indexing, and validation
- `minimap2`: default short-read and long-read mapping backend
- `bwa` and `bwa-mem2`: alternative BWA mapping backends
- `strobealign`: alternative short-read mapping backend
- `skani` and `fastANI`: ANI/dereplication backends for cluster paths
- `starcode`, `man`, `tee`, shell/core utilities required by CoverM help and
  runtime workflows

The file `/opt/coverm/share/doc/coverm/conda-packages.txt` records the exact
Conda package set resolved during image build.

## Usage

Access CoverM help:

```sh
taf-coverm -- --help
taf-coverm coverm genome --help
taf-coverm coverm contig --help
```

Calculate genome/MAG coverage from a reference-sorted BAM:

```sh
taf-coverm coverm genome \
  --bam-files sample.sorted.bam \
  --genome-fasta-files mag1.fna mag2.fna \
  --methods mean relative_abundance covered_fraction \
  --output-file genome_coverage.tsv
```

Calculate contig coverage by mapping reads:

```sh
taf-coverm coverm contig \
  --single reads.fq.gz \
  --reference assembly.fa \
  --mapper minimap2-sr \
  --methods mean covered_bases count \
  --output-file contig_coverage.tsv
```

Generate BAM files for reuse:

```sh
taf-coverm coverm make \
  --coupled sample_R1.fq.gz sample_R2.fq.gz \
  --reference assembly.fa \
  --output-directory coverm_bams \
  --threads 8
```

Filter BAM alignments:

```sh
taf-coverm coverm filter \
  --bam-files sample.sorted.bam \
  --output-bam-files sample.filtered.bam \
  --min-read-percent-identity 95
```

Cluster genomes using the default skani-backed path:

```sh
taf-coverm coverm cluster \
  --genome-fasta-files genomes/*.fna \
  --precluster-method skani \
  --cluster-method skani \
  --output-cluster-definition clusters.tsv \
  --output-representative-list representatives.txt
```

## Command Mode

TAFFISH tool apps normally support automatic command mode. For non-option
leading commands, `taf-coverm coverm ...` runs the `coverm` executable inside
the same container.

Use `taf-coverm -- --help` or `taf-coverm -- --version` when passing
option-leading arguments to the default CoverM command.

CoverM itself has subcommands. Prefer:

```sh
taf-coverm coverm genome ...
taf-coverm coverm contig ...
taf-coverm coverm make ...
```

Do not write `taf-coverm genome ...`; `genome` is not a container executable.

## Inputs

| Input | Meaning | Notes |
| --- | --- | --- |
| `--bam-files` | reference-sorted BAM files | no mapping is performed; useful for reproducible cached alignments |
| `--single`, `--coupled`, `-1/-2`, `--interleaved` | raw FASTA/FASTQ reads | may be gzipped; CoverM maps reads using the selected mapper |
| `--reference` | contig/reference FASTA or supported mapper index | required for contig mode unless BAM files are supplied |
| `--genome-fasta-files`, `--genome-fasta-directory`, `--genome-definition` | genome/MAG definitions | used by genome mode and cluster mode |

## Output Notes

Coverage commands write TSV output to stdout or `--output-file`. The selected
`--methods` control which columns are produced, such as `mean`,
`relative_abundance`, `covered_fraction`, `covered_bases`, `count`, `rpkm`,
`tpm`, and related metrics.

`coverm make` writes BAM files under `--output-directory`. `coverm filter`
writes thresholded BAM files. `coverm cluster` writes cluster definitions,
representative lists, or representative FASTA directories depending on the
chosen output options.

## Resources, Databases, and Platform

No external database is needed for normal CoverM coverage, mapping, filtering,
or clustering. Production runs still need user-provided reads, references,
assemblies, BAMs, or genome FASTA files.

CoverM uses temporary files during mapping and sorting. For large metagenomes,
set `TMPDIR` to a writable location with sufficient space:

```sh
TMPDIR="$PWD/tmp" taf-coverm coverm genome ...
```

This app declares native `linux/amd64` and `linux/arm64` builds. The runtime
comes from Bioconda `coverm=0.7.0`; helper package build strings are recorded
inside the image for auditability.

## Boundaries

The image focuses on the upstream CoverM `0.7.0` command surface and common
runtime dependencies. The publish smoke uses tiny synthetic inputs to exercise
BAM-based `contig`, BAM-based `genome`, `filter`, and raw-read mapping through
`make`/`contig`. It does not benchmark coverage values, validate mapper
sensitivity on real metagenomes, or run production-scale dereplication.

CoverM documents a legacy `dashing` precluster option. This app intentionally
validates the CoverM 0.7.0 default `skani` path because it is available across
the declared native platforms. If a workflow explicitly requires dashing,
verify `taf-coverm dashing --help` in that image/platform first or use a
custom image.

## Troubleshooting

- If BAM input fails, confirm it is reference-sorted. CoverM expects
  `samtools sort`-style coordinate sorting unless a sharded/read-name path is
  explicitly requested.
- If mapping fills `/tmp`, set `TMPDIR` to a larger mounted directory.
- If a CoverM subcommand seems missing, use `taf-coverm coverm <subcommand> ...`
  rather than `taf-coverm <subcommand> ...`.

## Testing

The smoke test covers:

- CoverM `0.7.0` runtime version and key subcommand help
- presence of `samtools`, mapper helpers, ANI helpers, `man`, `tee`, and shell
  utilities
- BAM-based `coverm contig`, BAM-based `coverm genome`, `coverm filter`, and
  raw-read mapping through `coverm make` plus `coverm contig`

It does not replace full scientific validation on production datasets.

## License and Citation

TAFFISH app packaging: Apache-2.0.

CoverM itself is distributed under GPL-3.0-or-later. Cite upstream CoverM
according to upstream guidance:

Aroney, S. T. N., Newell, R. J., Nissen, J. N., Camargo, A. P., Tyson, G. W.,
and Woodcroft, B. J. 2025. CoverM: Read alignment statistics for
metagenomics. Bioinformatics 41:4. <https://doi.org/10.1093/bioinformatics/btaf147>
