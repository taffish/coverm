coverm 0.8.0-r1

Purpose:
  Package CoverM for TAFFISH. CoverM calculates DNA read coverage and
  relative abundance for metagenomic contigs and genomes/MAGs from sorted
  BAM files or by mapping FASTA/FASTQ reads to reference sequences.

Usage:
  taf-coverm -- --help
  taf-coverm coverm genome --bam-files sample.bam --genome-fasta-files mag.fa
  taf-coverm coverm contig --single reads.fq --reference assembly.fa -m mean
  taf-coverm coverm make --single reads.fq --reference assembly.fa -o bams/
  taf-coverm coverm makedb --reference assembly.fa --mapper minimap2-sr -o db/
  taf-coverm coverm filter --bam-files in.bam --output-bam-files out.bam
  taf-coverm coverm cluster --genome-fasta-files *.fna --output-cluster-definition clusters.tsv

Packaged commands:
  coverm       Main CoverM command with genome, contig, make, filter,
               makedb, cluster, and shell-completion modes.
  samtools     BAM/SAM/CRAM reading, sorting, indexing, and validation.
  minimap2     Default short-read and long-read mapping backend.
  bwa          BWA-MEM mapping backend.
  bwa-mem2     BWA-MEM2 mapping backend.
  minibwa      Lightweight BWA-compatible mapper.
  strobealign  Alternative short-read mapping backend.
  skani        Common CoverM dereplication precluster/ANI backend.
  fastANI      Alternative ANI backend used by dereplication/cluster paths.

Upstream help and version:
  taf-coverm -- --version
  taf-coverm -- --help
  taf-coverm coverm genome --help
  taf-coverm coverm contig --help
  taf-coverm coverm make --help
  taf-coverm coverm makedb --help
  taf-coverm coverm filter --help
  taf-coverm coverm cluster --help

Inputs:
  Sorted BAM files        Use --bam-files for precomputed alignments.
  FASTA/FASTQ reads      Use --single, --coupled, -1/-2, or --interleaved.
  Reference FASTA/index  Use --reference for contig mode or mapping.
  Genome FASTA files     Use --genome-fasta-files, --genome-fasta-directory,
                         or --genome-definition for genome mode.

Key outputs:
  TSV coverage tables    Written to stdout or --output-file.
  BAM files              coverm make writes mapped, sorted BAM files.
  Mapping indexes        coverm makedb writes reusable mapper indexes.
  Filtered BAM files     coverm filter writes thresholded alignment files.
  Cluster TSV/list       coverm cluster writes representative/member outputs.

Platform and resources:
  Native container builds target linux/amd64 and linux/arm64.
  CoverM uses TMPDIR for temporary mapping/sorting files; set TMPDIR to a
  larger writable directory for production metagenomes if /tmp is small.
  No external database is bundled or required for normal coverage paths.

Boundaries:
  The image installs CoverM 0.8.0 from Bioconda plus common runtime tools.
  CoverM 0.8.0 adds makedb, ANIr/anir reporting, --min-mapq filtering, and
  additional mapper modes. This image bundles minibwa but not rammap or a
  separate strobealign-aemb executable because those are not available across
  the declared Bioconda linux/amd64 and linux/arm64 package set.
  It validates skani/fastANI-backed cluster dependencies, but production ANI
  clustering on large MAG collections remains a user-scale analysis task.
  The legacy dashing precluster option is not part of the publish smoke path;
  prefer packaged default paths unless you have a specific reason to override.

Detailed documentation:
  https://github.com/wwood/CoverM
  https://wwood.github.io/CoverM/

Wrapper options:
  taf-coverm --help       Show this TAFFISH help.
  taf-coverm --version    Show TAFFISH wrapper version.
  taf-coverm --compile    Compile the TAFFISH wrapper.
  taf-coverm -- --help    Pass option-leading arguments to CoverM.

Notes:
  Because this is a command-mode tool app, use "taf-coverm coverm genome ..."
  for CoverM subcommands. A bare "taf-coverm genome ..." is interpreted as a
  request to run a container executable named "genome".
