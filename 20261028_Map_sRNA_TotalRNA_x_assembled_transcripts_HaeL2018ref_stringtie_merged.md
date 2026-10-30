10.28.2026
# sRNAseq and TotalRNASeq Mapped to assembled_transcripts_HaeL2018ref_stringtie_merged

## STAR Index
- build_transcriptome_stringtie_index.sh

```sh
#!/bin/bash

#SBATCH -p cluster_short
#SBATCH -t 4:00:00
#SBATCH -n 12
#SBATCH --mem=80G
#SBATCH --job-name=transcriptome_bowtie_index
#SBATCH -o %j.out
#SBATCH -e %j.err
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp

source /work/ma-discar/anaconda3/etc/profile.d/conda.sh
conda activate bowtie1_env

bowtie-build \
/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/assembled_transcripts_HaeL2018ref_stringtie_merged.fa \
HaeL2018_transcriptome
```
```sh
#Job ID: 135699
#Log: 135699
```

## A. sRNAseq x assembled_transcripts_HaeL2018ref_stringtie_merged.fa
/work/ma-discar/PHAS/PHAS_transcript_analysis/map_HlosRNA_transcriptome.sh

- map_HlosRNA_transcriptome.sh
```sh
(base) ma-discar@cc25dev0:~/PHAS/PHAS_transcript_analysis$ cat map_HlosRNA_transcriptome.sh
#!/bin/bash

#SBATCH -p cluster_long
#SBATCH -t 12:00:00
#SBATCH -n 12
#SBATCH --mem=40G
#SBATCH --job-name=map_HlosRNA_transcriptome
#SBATCH -o %j.out
#SBATCH -e %j.err
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp
#SBATCH --mail-type=ALL

set -euo pipefail

source /work/ma-discar/anaconda3/etc/profile.d/conda.sh
conda activate bowtie1_env

INPUT_DIR="/work/ma-discar/PHAS/PHAS_transcript_analysis/fasta"

INDEX="$HOME/PHAS/PHAS_transcript_analysis/HaeL2018_transcriptome"

OUTDIR="$HOME/PHAS/PHAS_transcript_analysis/bam"

mkdir -p "$OUTDIR"

for FASTA in "$INPUT_DIR"/*.fa
do

    BASENAME=$(basename "$FASTA" .fa)

    echo "========================================"
    echo "Mapping: $BASENAME"
    echo "========================================"

    bowtie \
    -v 1 \
    -m 20 \
    -a \
    --best \
    --strata \
    --no-unal \
    -p 12 \
    -S \
    -f \
    "$INDEX" \
    "$FASTA" \
    | samtools view -bS - \
    > "$OUTDIR/${BASENAME}.transcriptome.bam"

    samtools sort \
    "$OUTDIR/${BASENAME}.transcriptome.bam" \
    -o "$OUTDIR/${BASENAME}.transcriptome.sorted.bam"

    rm "$OUTDIR/${BASENAME}.transcriptome.bam"

    samtools index \
    "$OUTDIR/${BASENAME}.transcriptome.sorted.bam"

    samtools flagstat \
    "$OUTDIR/${BASENAME}.transcriptome.sorted.bam" \
    > "$OUTDIR/${BASENAME}.transcriptome.flagstat.txt"

    echo "Finished: $BASENAME"
    echo

done

echo "========================================"
echo "All transcriptome mapping finished."
echo "========================================"
```
```sh
# Job ID: 139199
# Log: 139199.out
```
## B. Total RNAseq x assembled_transcripts_HaeL2018ref_stringtie_merged
##### DRR5180XX
- STAR_map_totalRNA_transcriptome.slurm
```sh
#!/bin/bash
#SBATCH -t 4-04:00:00
#SBATCH -p cluster_long
#SBATCH -J STAR_map_totalRNA_transcriptome
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=24
#SBATCH -o STAR_map_totalRNA_transcriptome.%j.out
#SBATCH -e STAR_map_totalRNA_transcriptome.%j.err
#SBATCH --mem=192G
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp
#SBATCH --mail-type=ALL

set -euo pipefail

# ------------------------------------------------------------
# Paths
# ------------------------------------------------------------

FASTQ_DIR="/work/ma-discar/20231214_hloRNAseq_NIG/fastp"

STAR_INDEX="/work/ma-discar/PHAS/PHAS_transcript_analysis/bam_totalRNA/STAR_index_assembled_transcripts"

OUT_DIR="/work/ma-discar/PHAS/PHAS_transcript_analysis/bam_totalRNA"

CONDA="/work/ma-discar/anaconda3/bin/conda"
STAR_ENV="/work/ma-discar/anaconda3/envs/star-env"
SAMTOOLS_ENV="/work/ma-discar/anaconda3/envs/samtools_env"

THREADS="${SLURM_CPUS_PER_TASK:-24}"

# ------------------------------------------------------------
# Validate software, index and input files
# ------------------------------------------------------------

[[ -x "$CONDA" ]] || {
    echo "ERROR: Conda executable not found."
    exit 1
}

for file in Genome SA SAindex chrName.txt chrLength.txt genomeParameters.txt; do
    [[ -s "$STAR_INDEX/$file" ]] || {
        echo "ERROR: STAR index file missing: $STAR_INDEX/$file"
        exit 1
    }
done

# Check all paired-end FASTQ files before starting
for N in {07..28}; do
    LIB="DRR5180${N}"

    for READ in \
        "${FASTQ_DIR}/${LIB}_1.fastq.gz" \
        "${FASTQ_DIR}/${LIB}_2.fastq.gz"
    do
        [[ -s "$READ" ]] || {
            echo "ERROR: Missing FASTQ file: $READ"
            exit 1
        }
    done
done

mkdir -p "$OUT_DIR"

echo "STAR version:"
"$CONDA" run --no-capture-output -p "$STAR_ENV" STAR --version

echo "Starting total RNA-seq alignment to assembled transcripts."
echo "STAR index: $STAR_INDEX"
echo "Output directory: $OUT_DIR"
date

# ------------------------------------------------------------
# Map paired-end reads to the assembled transcriptome
# ------------------------------------------------------------

for N in {07..28}; do

    LIB="DRR5180${N}"

    READ1="${FASTQ_DIR}/${LIB}_1.fastq.gz"
    READ2="${FASTQ_DIR}/${LIB}_2.fastq.gz"

    PREFIX="${OUT_DIR}/${LIB}."
    STAR_BAM="${PREFIX}Aligned.sortedByCoord.out.bam"
    FINAL_BAM="${OUT_DIR}/${LIB}.transcriptome.sorted.bam"

    echo "============================================================"
    echo "Sample: $LIB"
    date

    # Skip completed samples
    if [[ -s "$FINAL_BAM" && -s "${FINAL_BAM}.bai" ]]; then
        echo "BAM and index already exist; skipping $LIB."
        continue
    fi

    # If BAM exists but its index is missing, try indexing it
    if [[ -s "$FINAL_BAM" && ! -s "${FINAL_BAM}.bai" ]]; then
        echo "Existing BAM has no index; indexing $LIB."

        "$CONDA" run --no-capture-output -p "$SAMTOOLS_ENV" \
            samtools index -@ "$THREADS" "$FINAL_BAM"

        continue
    fi

    # Do not silently overwrite an incomplete intermediate BAM
    if [[ -e "$STAR_BAM" ]]; then
        echo "ERROR: Intermediate STAR BAM already exists:"
        echo "$STAR_BAM"
        echo "Inspect this file before rerunning the sample."
        exit 1
    fi

    "$CONDA" run --no-capture-output -p "$STAR_ENV" \
        STAR \
        --runThreadN "$THREADS" \
        --genomeDir "$STAR_INDEX" \
        --readFilesIn "$READ1" "$READ2" \
        --readFilesCommand zcat \
        --outFilterMultimapNmax 20 \
        --outSAMmultNmax 20 \
        --outFilterMismatchNmax 999 \
        --outFilterMismatchNoverReadLmax 0.04 \
        --alignIntronMax 1 \
        --outSAMattributes NH HI AS nM NM \
        --outSAMtype BAM SortedByCoordinate \
        --outFileNamePrefix "$PREFIX"

    [[ -s "$STAR_BAM" ]] || {
        echo "ERROR: STAR BAM was not generated: $STAR_BAM"
        exit 1
    }

    mv "$STAR_BAM" "$FINAL_BAM"

    # Index the sorted BAM for IGV
    "$CONDA" run --no-capture-output -p "$SAMTOOLS_ENV" \
        samtools index -@ "$THREADS" "$FINAL_BAM"

    [[ -s "${FINAL_BAM}.bai" ]] || {
        echo "ERROR: BAM index was not generated: ${FINAL_BAM}.bai"
        exit 1
    }

    echo "Completed: $LIB"
    echo "BAM: $FINAL_BAM"
    echo "Index: ${FINAL_BAM}.bai"
    date

done

echo "All requested total RNA-seq samples processed."
date

```
```sh

#Job ID: 322373
#Log: STAR_map_totalRNA_transcriptome.322373.out 
```
##### N024_N048
- STAR_map_N024N048_totalRNA_PE_x_assembled_transcripts_HaeL2018ref_stringtie_merged.slurm
```sh
#!/bin/bash
#SBATCH -t 4-04:00:00
#SBATCH -p cluster_long
#SBATCH -J STAR_totalRNA_N024_N048
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=24
#SBATCH --mem=192G
#SBATCH -o STAR_totalRNA_N024_N048.%j.out
#SBATCH -e STAR_totalRNA_N024_N048.%j.err
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp
#SBATCH --mail-type=ALL

set -Eeuo pipefail
shopt -s nullglob

# ------------------------------------------------------------
# Paths
# ------------------------------------------------------------

FASTQ_DIR="/work/ma-discar/20211202_hloRNAseq_BGI/Clean"

STAR_INDEX="/work/ma-discar/PHAS/PHAS_transcript_analysis/bam_totalRNA/STAR_index_assembled_transcripts"

OUT_DIR="/work/ma-discar/PHAS/PHAS_transcript_analysis/bam_totalRNA/N024_N048"

STAR="/work/ma-discar/anaconda3/envs/star-env/bin/STAR"
SAMTOOLS="/work/ma-discar/anaconda3/envs/samtools_env/bin/samtools"
THREADS="${SLURM_CPUS_PER_TASK:-24}"

# ------------------------------------------------------------
# Validate software and index
# ------------------------------------------------------------

for EXE in "$STAR" "$SAMTOOLS"; do
    [[ -x "$EXE" ]] || {
        echo "ERROR: Executable not found: $EXE" >&2
        exit 1
    }
done

for FILE in Genome SA SAindex chrName.txt chrLength.txt genomeParameters.txt; do
    [[ -s "$STAR_INDEX/$FILE" ]] || {
        echo "ERROR: STAR index component missing: $STAR_INDEX/$FILE" >&2
        exit 1
    }
done

[[ -d "$FASTQ_DIR" ]] || {
    echo "ERROR: FASTQ directory not found: $FASTQ_DIR" >&2
    exit 1
}

# ------------------------------------------------------------
# Discover and validate paired-end samples
# ------------------------------------------------------------

READ1_FILES=("$FASTQ_DIR"/*_1.fq.gz)

if (( ${#READ1_FILES[@]} == 0 )); then
    echo "ERROR: No *_1.fq.gz files found in $FASTQ_DIR" >&2
    exit 1
fi

for READ1 in "${READ1_FILES[@]}"; do
    READ2="${READ1%_1.fq.gz}_2.fq.gz"

    [[ -s "$READ2" ]] || {
        echo "ERROR: Matching R2 file not found for: $READ1" >&2
        exit 1
    }
done

mkdir -p "$OUT_DIR"

echo "STAR version:"
"$STAR" --version

echo "Input directory: $FASTQ_DIR"
echo "STAR index: $STAR_INDEX"
echo "Output directory: $OUT_DIR"
echo "Paired-end samples found: ${#READ1_FILES[@]}"

# Print the sample names before processing
for READ1 in "${READ1_FILES[@]}"; do
    SAMPLE="$(basename "$READ1" _1.fq.gz)"
    echo "  $SAMPLE"
done

date

# ------------------------------------------------------------
# STAR alignment
# ------------------------------------------------------------

for READ1 in "${READ1_FILES[@]}"; do

    SAMPLE="$(basename "$READ1" _1.fq.gz)"
    READ2="${READ1%_1.fq.gz}_2.fq.gz"

    PREFIX="${OUT_DIR}/${SAMPLE}."
    STAR_BAM="${PREFIX}Aligned.sortedByCoord.out.bam"
    FINAL_BAM="${OUT_DIR}/${SAMPLE}.transcriptome.sorted.bam"

    echo "============================================================"
    echo "Mapping sample: $SAMPLE"
    date

    # Skip completed samples
    if [[ -s "$FINAL_BAM" && -s "${FINAL_BAM}.bai" ]]; then
        echo "BAM and index already exist; skipping $SAMPLE."
        continue
    fi

    # Recover by indexing an existing final BAM
    if [[ -s "$FINAL_BAM" ]]; then
        "$SAMTOOLS" quickcheck -v "$FINAL_BAM"
        "$SAMTOOLS" index -@ "$THREADS" "$FINAL_BAM"
        echo "Existing BAM indexed: $SAMPLE"
        continue
    fi

    # Avoid silently overwriting an intermediate BAM
    if [[ -e "$STAR_BAM" ]]; then
        echo "ERROR: Intermediate BAM already exists: $STAR_BAM" >&2
        echo "Inspect it before rerunning this sample." >&2
        exit 1
    fi

    "$STAR" \
        --runThreadN "$THREADS" \
        --genomeDir "$STAR_INDEX" \
        --readFilesIn "$READ1" "$READ2" \
        --readFilesCommand zcat \
        --outFilterMultimapNmax 20 \
        --outSAMmultNmax 20 \
        --outFilterMismatchNmax 999 \
        --outFilterMismatchNoverReadLmax 0.04 \
        --alignIntronMax 1 \
        --outSAMattributes NH HI AS nM NM \
	--limitBAMsortRAM 16000000000 \
        --outSAMtype BAM SortedByCoordinate \
        --outFileNamePrefix "$PREFIX"

    [[ -s "$STAR_BAM" ]] || {
        echo "ERROR: STAR BAM was not generated: $STAR_BAM" >&2
        exit 1
    }

    "$SAMTOOLS" quickcheck -v "$STAR_BAM"
    mv "$STAR_BAM" "$FINAL_BAM"
    "$SAMTOOLS" index -@ "$THREADS" "$FINAL_BAM"
    "$SAMTOOLS" quickcheck -v "$FINAL_BAM"

    [[ -s "${FINAL_BAM}.bai" ]] || {
        echo "ERROR: BAM index was not generated: ${FINAL_BAM}.bai" >&2
        exit 1
    }

    echo "Completed: $SAMPLE"
    echo "BAM: $FINAL_BAM"
    echo "BAI: ${FINAL_BAM}.bai"
    echo "STAR summary: ${PREFIX}Log.final.out"
    date

done

echo "All discovered paired-end samples processed."
date

```
```sh
#Job ID: 330319
#Log: STAR_totalRNA_N024_N048.330319.out
```

 