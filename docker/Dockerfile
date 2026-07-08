FROM mambaorg/micromamba:1.5.10-bookworm-slim

ARG COVERM_VERSION=0.8.0
ARG BWA_MEM2_VERSION=2.3
ARG MINIBWA_VERSION=0.3
ARG SKANI_VERSION=0.3.2

USER root

ENV DEBIAN_FRONTEND=noninteractive

RUN set -eux; \
    for attempt in 1 2 3 4 5; do \
      if apt-get -o Acquire::Retries=3 update \
       && apt-get -o Acquire::Retries=3 install -y --no-install-recommends \
            bash \
            ca-certificates \
            coreutils \
            findutils \
            grep \
            gzip \
            less \
            man-db \
            sed \
            tar; then \
        break; \
      fi; \
      if [ "${attempt}" = "5" ]; then exit 1; fi; \
      apt-get clean; \
      rm -rf /var/lib/apt/lists/*; \
      sleep 5; \
    done; \
    rm -rf /var/lib/apt/lists/*

USER mambauser

RUN micromamba install -y -n base \
      -c conda-forge \
      -c bioconda \
      "coverm=${COVERM_VERSION}" \
      "bwa-mem2=${BWA_MEM2_VERSION}" \
      "minibwa=${MINIBWA_VERSION}" \
      "skani=${SKANI_VERSION}" \
 && micromamba clean -a -y

ENV PATH=/opt/conda/bin:${PATH}

USER root

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN set -eux; \
    mkdir -p \
      /opt/coverm/share/doc/coverm \
      /opt/coverm/share/licenses/coverm \
      /opt/coverm/share/testdata; \
    micromamba list -n base > /opt/coverm/share/doc/coverm/conda-packages.txt; \
    { \
      printf "upstream_repo=https://github.com/wwood/CoverM\n"; \
      printf "upstream_tag=v%s\n" "${COVERM_VERSION}"; \
      printf "runtime_source=bioconda::coverm=%s\n" "${COVERM_VERSION}"; \
      printf "extra_runtime=bioconda::bwa-mem2=%s\n" "${BWA_MEM2_VERSION}"; \
      printf "extra_runtime=bioconda::minibwa=%s\n" "${MINIBWA_VERSION}"; \
      printf "extra_runtime=bioconda::skani=%s\n" "${SKANI_VERSION}"; \
      printf "dependency_note=CoverM Bioconda dependency resolution supplies samtools, minimap2, strobealign, bwa, fastANI, python-dashing, starcode, and shared runtime libraries.\n"; \
      printf "cluster_note=CoverM 0.8.0 uses skani-capable clustering paths; the image explicitly includes skani, fastANI, bwa-mem2, and minibwa-compatible dependencies.\n"; \
      printf "optional_mapper_note=CoverM 0.8.0 documents rammap and strobealign-aemb mapper modes; rammap is not bundled because the Bioconda package is not available for linux-aarch64, and strobealign-aemb is not distributed as a separate Bioconda executable.\n"; \
    } > /opt/coverm/share/doc/coverm/source.txt; \
    if [ -d /opt/conda/share/licenses/coverm ]; then \
      cp -a /opt/conda/share/licenses/coverm/. /opt/coverm/share/licenses/coverm/; \
    else \
      printf "CoverM is distributed under GPL-3.0-or-later. See https://github.com/wwood/CoverM\n" > /opt/coverm/share/licenses/coverm/LICENSE-note.txt; \
    fi

RUN cat > /opt/coverm/share/testdata/coverm-smoke-bam.sh <<'EOF'
#!/bin/sh
set -eu

tmp="${1:-/tmp/taf-coverm-bam-$$}"
rm -rf "$tmp"
mkdir -p "$tmp"
cd "$tmp"

refseq="GATTACAGTCGATCGTACCTGACCTAGGATCGATGCTAGCTTACGATCGGATCCGTAGCTAGTCCGATGACCTTACGGATCGTAGCTACGATTCGACCTAGGTCAGTACGATCGATGCTAGGCTAACGTCGATACGGTAGCTTACGATCGATCCGATGCTAGCTAGCTAGGATCGTACTGACCGATCGATGCTAGCTAACGATCGATCGTACGATCGATGCTAGCTAGCTA"
readseq="$(printf '%s' "$refseq" | cut -c 21-120)"
readlen="$(printf '%s' "$readseq" | wc -c | tr -d ' ')"
qual="$(printf '%*s' "$readlen" '' | tr ' ' 'I')"
reflen="$(printf '%s' "$refseq" | wc -c | tr -d ' ')"

printf '>contig1\n%s\n' "$refseq" > ref.fa
{
  printf '@HD\tVN:1.6\tSO:coordinate\n'
  printf '@SQ\tSN:contig1\tLN:%s\n' "$reflen"
  printf 'read1\t0\tcontig1\t21\t60\t%sM\t*\t0\t0\t%s\t%s\tNM:i:0\n' "$readlen" "$readseq" "$qual"
} > aln.sam

samtools view -bS aln.sam > aln.bam
samtools sort -o aln.sorted.bam aln.bam
samtools index aln.sorted.bam

coverm contig \
  --bam-files aln.sorted.bam \
  --methods mean anir covered_bases count length \
  --contig-end-exclusion 0 \
  --output-file contig.tsv
test -s contig.tsv
grep -F 'contig1' contig.tsv >/dev/null
grep -F 'Mean' contig.tsv >/dev/null
grep -F 'ANIr' contig.tsv >/dev/null
grep -F 'Covered Bases' contig.tsv >/dev/null

coverm genome \
  --bam-files aln.sorted.bam \
  --genome-fasta-files ref.fa \
  --methods mean anir covered_fraction length \
  --contig-end-exclusion 0 \
  --min-covered-fraction 0 \
  --output-file genome.tsv
test -s genome.tsv
grep -F 'Mean' genome.tsv >/dev/null
grep -F 'ANIr' genome.tsv >/dev/null
grep -F 'Covered Fraction' genome.tsv >/dev/null

coverm filter \
  --bam-files aln.sorted.bam \
  --output-bam-files filtered.bam \
  --min-mapq 0 \
  --min-read-aligned-length 10
test -s filtered.bam
samtools quickcheck filtered.bam

rm -rf "$tmp"
EOF

RUN cat > /opt/coverm/share/testdata/coverm-smoke-map.sh <<'EOF'
#!/bin/sh
set -eu

tmp="${1:-/tmp/taf-coverm-map-$$}"
rm -rf "$tmp"
mkdir -p "$tmp"
cd "$tmp"

refseq="GATTACAGTCGATCGTACCTGACCTAGGATCGATGCTAGCTTACGATCGGATCCGTAGCTAGTCCGATGACCTTACGGATCGTAGCTACGATTCGACCTAGGTCAGTACGATCGATGCTAGGCTAACGTCGATACGGTAGCTTACGATCGATCCGATGCTAGCTAGCTAGGATCGTACTGACCGATCGATGCTAGCTAACGATCGATCGTACGATCGATGCTAGCTAGCTA"
readseq="$(printf '%s' "$refseq" | cut -c 21-140)"
readlen="$(printf '%s' "$readseq" | wc -c | tr -d ' ')"
qual="$(printf '%*s' "$readlen" '' | tr ' ' 'I')"

printf '>contig1\n%s\n' "$refseq" > ref.fa
printf '@read1\n%s\n+\n%s\n' "$readseq" "$qual" > reads.fq

coverm makedb \
  --reference ref.fa \
  --mapper minimap2-sr \
  --output-directory coverm-db \
  --threads 1
test -s coverm-db/ref.fa.minimap2-sr.mmi

coverm make \
  --single reads.fq \
  --reference ref.fa \
  --mapper minimap2-sr \
  --output-directory made-bams \
  --threads 1
find made-bams -type f -name '*.bam' -print | grep -F '.bam' >/dev/null
first_bam="$(find made-bams -type f -name '*.bam' -print | sed -n '1p')"
samtools quickcheck "$first_bam"

coverm contig \
  --single reads.fq \
  --reference ref.fa \
  --mapper minimap2-sr \
  --methods count covered_bases length \
  --contig-end-exclusion 0 \
  --output-file mapped-contig.tsv \
  --threads 1
test -s mapped-contig.tsv
grep -F 'contig1' mapped-contig.tsv >/dev/null
grep -F 'Read Count' mapped-contig.tsv >/dev/null

rm -rf "$tmp"
EOF

RUN set -eux; \
    chmod 0755 /opt/coverm/share/testdata/coverm-smoke-bam.sh /opt/coverm/share/testdata/coverm-smoke-map.sh; \
    coverm --version 2>&1 | grep -Fx "coverm ${COVERM_VERSION}" >/dev/null; \
    coverm --help 2>&1 | grep -F "Calculate coverage" >/dev/null; \
    coverm genome --help 2>&1 | grep -F "coverm genome" >/dev/null; \
    coverm contig --help 2>&1 | grep -F "coverm contig" >/dev/null; \
    coverm make --help 2>&1 | grep -F "coverm make" >/dev/null; \
    coverm makedb --help 2>&1 | grep -F "coverm makedb" >/dev/null; \
    coverm filter --help 2>&1 | grep -F "coverm filter" >/dev/null; \
    coverm cluster --help 2>&1 | grep -iF "cluster" >/dev/null; \
    coverm contig --full-help-roff 2>&1 | grep -F "anir" >/dev/null; \
    coverm filter --full-help-roff 2>&1 | grep -F "min\\-mapq" >/dev/null; \
    coverm makedb --help 2>&1 | grep -F "minimap2-sr" >/dev/null; \
    samtools --version 2>&1 | grep -F "samtools" >/dev/null; \
    minimap2 --version >/dev/null; \
    { bwa 2>&1 || true; } | grep -iF "bwa" >/dev/null; \
    command -v bwa-mem2 >/dev/null; \
    command -v minibwa >/dev/null; \
    command -v strobealign >/dev/null; \
    command -v fastANI >/dev/null; \
    { skani --version 2>&1 || true; } | grep -iF "skani" >/dev/null; \
    command -v starcode >/dev/null; \
    command -v tee >/dev/null; \
    man --version 2>&1 | grep -F "man" >/dev/null; \
    /opt/coverm/share/testdata/coverm-smoke-bam.sh /tmp/coverm-build-bam; \
    /opt/coverm/share/testdata/coverm-smoke-map.sh /tmp/coverm-build-map

WORKDIR /root

ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=coverm
