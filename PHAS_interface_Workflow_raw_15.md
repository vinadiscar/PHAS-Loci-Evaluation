07.05.2026
##### PHAS_interface_Workflow_raw_15.md
> > update from PHAS_interface_Workflow_raw_8.md 
# update mRNA tracks

- Previous approach used only tracks to represent genome alignment orientation, but not transcript sense/antisense orientation

### Check strandedness

```sh
# Convert gtf annotation file to bed12 file
 
cd /work/ma-discar/UCSC_utilities
./gtfToGenePred -genePredExt /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.gtf /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.genePred
./genePredToBed /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.genePred /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12

head /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12

# Check strandedness in one sample bam file 

conda activate rseqc_env 
infer_experiment.py \
    -r /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12 \
    -i /project/okamura-lab/Canran/H.longicornis/data/mRNA/OitaE1-NewAligned.sortedByCoord.out.bam

```
#### Result (Strandedness):
```sh
Reading reference gene model /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12 ... Done
Loading SAM/BAM file ...  Total 200000 usable reads were sampled


This is PairEnd Data
Fraction of reads failed to determine: 0.0589
Fraction of reads explained by "1++,1--,2+-,2-+": 0.0073
Fraction of reads explained by "1+-,1-+,2++,2--": 0.9338
```
- Result: The second fraction `1+-,1-+,2++,2--` is overwhelmingly dominant (93.38%). Therefore the library is RF stranded
- For an RF library, Read 1 aligns opposite to the transcript (`1+-,1-+`). Read 2 aligns same direction as the transcript (`2++,2--`).

---
# Update Previous Code: Convert Bam to bigwig file
```sh
cd /work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig
vi mRNA_bam_to_bw.sh

```
```sh
#!/bin/bash

set -euo pipefail

# Activate conda environment
source /work/ma-discar/anaconda3/etc/profile.d/conda.sh
conda activate bedtools_env

INPUT_DIR="/work/ma-discar/PHAS/Hlongicornis/data/mRNA"
OUTPUT_DIR="/work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig"
GENOME_SIZE="/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes"

mkdir -p "$OUTPUT_DIR"

for bam in "${INPUT_DIR}"/*.bam
do
    base=$(basename "$bam" .bam)

    echo "Processing $base"

    ############################################
    # Reverse-stranded paired-end RNA-seq
    #
    # infer_experiment.py:
    # 1+-,1-+,2++,2-- = transcript strand
    #
    # Transcript PLUS strand:
    #   FLAG 99  (read1 + / mate -)
    #   FLAG 147 (read2 - / mate +)
    #
    # Transcript MINUS strand:
    #   FLAG 83  (read1 - / mate +)
    #   FLAG 163 (read2 + / mate -)
    ############################################


    ############################################
    # PLUS transcript strand
    ############################################

    samtools view -b "$bam" -f 99 > "${OUTPUT_DIR}/${base}.plus.tmp.bam"
    samtools view -b "$bam" -f 147 >> "${OUTPUT_DIR}/${base}.plus.tmp.bam"

    bedtools genomecov \
        -ibam "${OUTPUT_DIR}/${base}.plus.tmp.bam" \
        -bg \
        -pc \
        > "${OUTPUT_DIR}/${base}.plus.bedgraph"


    ############################################
    # MINUS transcript strand
    ############################################

    samtools view -b "$bam" -f 83 > "${OUTPUT_DIR}/${base}.minus.tmp.bam"
    samtools view -b "$bam" -f 163 >> "${OUTPUT_DIR}/${base}.minus.tmp.bam"

    bedtools genomecov \
        -ibam "${OUTPUT_DIR}/${base}.minus.tmp.bam" \
        -bg \
        -pc \
        > "${OUTPUT_DIR}/${base}.minus.bedgraph"


    ############################################
    # Sort bedGraph (required for bedGraphToBigWig)
    ############################################

    sort -k1,1 -k2,2n \
        "${OUTPUT_DIR}/${base}.plus.bedgraph" \
        -o "${OUTPUT_DIR}/${base}.plus.bedgraph"

    sort -k1,1 -k2,2n \
        "${OUTPUT_DIR}/${base}.minus.bedgraph" \
        -o "${OUTPUT_DIR}/${base}.minus.bedgraph"


    ############################################
    # Convert to BigWig
    ############################################

    bedGraphToBigWig \
        "${OUTPUT_DIR}/${base}.plus.bedgraph" \
        "$GENOME_SIZE" \
        "${OUTPUT_DIR}/${base}.plus.bw"


    bedGraphToBigWig \
        "${OUTPUT_DIR}/${base}.minus.bedgraph" \
        "$GENOME_SIZE" \
        "${OUTPUT_DIR}/${base}.minus.bw"


    ############################################
    # Cleanup
    ############################################

    rm "${OUTPUT_DIR}/${base}.plus.tmp.bam"
    rm "${OUTPUT_DIR}/${base}.minus.tmp.bam"

    rm "${OUTPUT_DIR}/${base}.plus.bedgraph"
    rm "${OUTPUT_DIR}/${base}.minus.bedgraph"


    echo "Finished $base"

done

echo "All mRNA BigWig files completed!"
```

- ### Run `mRNA_bam_to_bw.sh`
```sh
chmod +x mRNA_bam_to_bw.sh
nohup ./mRNA_bam_to_bw.sh > mRNA_bam_to_bw.log 2>&1 &
```
- ### Check result of one sample

```sh
cd /work/ma-discar/UCSC_utilities
./bigWigInfo /work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig/DRR518008Aligned.sortedByCoord.out.plus.bw

version: 4
isCompressed: yes
isSwapped: 0
primaryDataSize: 17,470,631
primaryIndexSize: 217,604
zoomLevels: 10
chromCount: 2767
basesCovered: 1,233,199,760
mean: 17.566924
min: 1.000000
max: 120871.000000
std: 274.600251

./bigWigInfo /work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig/DRR518008Aligned.sortedByCoord.out.minus.bw

version: 4
isCompressed: yes
isSwapped: 0
primaryDataSize: 16,951,600
primaryIndexSize: 217,380
zoomLevels: 10
chromCount: 2839
basesCovered: 1,216,104,316
mean: 15.600860
min: 1.000000
max: 63961.000000
std: 200.123674
```
- ### Rename some samples with prefix DRR...
```sh
cd /work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig
vi rename_DRRbigwig.sh
```
- rename_DRRbigwig.sh
```sh
#!/bin/bash

declare -A samples=(
["DRR518007"]="Oita_Nymph_Fed"
["DRR518008"]="Oita_Nymph_Unfed"
["DRR518009"]="Okayama_Nymph_Fed"
["DRR518010"]="Okayama_Nymph_Unfed"
["DRR518011"]="Okayama_Larva_unfed"
["DRR518012"]="Okayama_Larva_fed"
["DRR518013"]="Okayama_adult_female_unfed_1"
["DRR518014"]="Okayama_adult_female_unfed_2"
["DRR518015"]="Okayama_adult_female_unfed_3"
["DRR518016"]="Okayama_adult_female_fed_1"
["DRR518017"]="Okayama_adult_female_fed_2"
["DRR518018"]="Okayama_adult_female_fed_3"
["DRR518019"]="Oita_Larva_unfed"
["DRR518020"]="Oita_Larva_fed"
["DRR518021"]="Oita_adult_male_unfed"
["DRR518022"]="Oita_adult_male_fed"
["DRR518023"]="Oita_adult_female_unfed_1"
["DRR518024"]="Oita_adult_female_unfed_2"
["DRR518025"]="Oita_adult_female_unfed_3"
["DRR518026"]="Oita_adult_female_fed_1"
["DRR518027"]="Oita_adult_female_fed_2"
["DRR518028"]="Oita_adult_female_fed_3"
)

for old in "${!samples[@]}"; do
    new="${samples[$old]}"

    for file in ${old}*.bw; do
        if [[ -e "$file" ]]; then
            newfile="${file/$old/$new_}"
            echo "Renaming:"
            echo "  $file -> $newfile"
            mv "$file" "$newfile"
        fi
    done
done
```

- #### Run:
```sh
chmod +x rename_DRRbigwig.sh
./rename_DRRbigwig.sh
```

- #### Renamed_log
```sh
Renaming:
  DRR518007Aligned.sortedByCoord.out.minus.bw -> Oita_Nymph_FedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518007Aligned.sortedByCoord.out.plus.bw -> Oita_Nymph_FedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518009Aligned.sortedByCoord.out.minus.bw -> Okayama_Nymph_FedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518009Aligned.sortedByCoord.out.plus.bw -> Okayama_Nymph_FedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518008Aligned.sortedByCoord.out.minus.bw -> Oita_Nymph_UnfedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518008Aligned.sortedByCoord.out.plus.bw -> Oita_Nymph_UnfedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518028Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_fed_3Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518028Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_fed_3Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518027Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_fed_2Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518027Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_fed_2Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518026Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_fed_1Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518026Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_fed_1Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518025Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_unfed_3Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518025Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_unfed_3Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518024Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_unfed_2Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518024Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_unfed_2Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518023Aligned.sortedByCoord.out.minus.bw -> Oita_adult_female_unfed_1Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518023Aligned.sortedByCoord.out.plus.bw -> Oita_adult_female_unfed_1Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518022Aligned.sortedByCoord.out.minus.bw -> Oita_adult_male_fedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518022Aligned.sortedByCoord.out.plus.bw -> Oita_adult_male_fedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518021Aligned.sortedByCoord.out.minus.bw -> Oita_adult_male_unfedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518021Aligned.sortedByCoord.out.plus.bw -> Oita_adult_male_unfedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518020Aligned.sortedByCoord.out.minus.bw -> Oita_Larva_fedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518020Aligned.sortedByCoord.out.plus.bw -> Oita_Larva_fedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518012Aligned.sortedByCoord.out.minus.bw -> Okayama_Larva_fedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518012Aligned.sortedByCoord.out.plus.bw -> Okayama_Larva_fedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518013Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_unfed_1Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518013Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_unfed_1Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518010Aligned.sortedByCoord.out.minus.bw -> Okayama_Nymph_UnfedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518010Aligned.sortedByCoord.out.plus.bw -> Okayama_Nymph_UnfedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518011Aligned.sortedByCoord.out.minus.bw -> Okayama_Larva_unfedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518011Aligned.sortedByCoord.out.plus.bw -> Okayama_Larva_unfedAligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518016Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_fed_1Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518016Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_fed_1Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518017Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_fed_2Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518017Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_fed_2Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518014Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_unfed_2Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518014Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_unfed_2Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518015Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_unfed_3Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518015Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_unfed_3Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518018Aligned.sortedByCoord.out.minus.bw -> Okayama_adult_female_fed_3Aligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518018Aligned.sortedByCoord.out.plus.bw -> Okayama_adult_female_fed_3Aligned.sortedByCoord.out.plus.bw
Renaming:
  DRR518019Aligned.sortedByCoord.out.minus.bw -> Oita_Larva_unfedAligned.sortedByCoord.out.minus.bw
Renaming:
  DRR518019Aligned.sortedByCoord.out.plus.bw -> Oita_Larva_unfedAligned.sortedByCoord.out.plus.bw
```
```sh
Source of bam files and equivalent names:
# \\fsz-p21.naist.jp\okamura-lab\Canran\H.longicornis\data\mRNA
# /home/OkamuraLabsharefolder/Vina/Vina_Presentations/ProgressMeeting/Progress Report_03.29.2024_FINAL.pptx
```

## Copy converted bigwig files 
- Copy to "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"
```sh
# Remove previous mRNA bigwig files and replace with the updated ones
cd "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/mRNA_bw"
rm -r bw
# Create new "bw" directory and copy new updated mRNA bw files 
mkdir bw

cp -r "/work/ma-discar/PHAS/Hlongicornis/data/mRNA/bigwig/" "/project/okamura-lab/Vina/PHAS/Hlongicornis/data/mRNA/"

cp /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/mRNA/bigwig/*.bw "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/mRNA_bw/bw"
```

- #### Restart PHASER to check updated mRNA tracks
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
bash start.sh
```
---
07.20.2026
# Update sRNA and mRNA tracks
- Use sRNA and mRNA bigwig files from lab's "/Volumes/Install macOS Mojave/Trackhubs/Hlo_developlib"
- ##### Note:
    - sRNA fed samples below are not in the Hlo_developlib directory. Generate bigwig samples using the same approach as what bw samples were genrated from the said directory. 
        
        - "N133" = "Okayama_Larva_fed"
        - "N134" = "Oita_Larva_fed
        - "N135" = "Okayama_Nymph_fed
        - "N136" = "Oita_Nymph_fed"
        - "N137" = "Oita_Adultmale_fed"
        - N138" = "Oita_Adultfemale_fed"
    
### Copy bw files
```sh
# copy mRNA bigwig files
cp "/Volumes/Install macOS Mojave/Trackhubs/Hlo_developlib/"*str1.bw \
   "/Volumes/Install macOS Mojave/Trackhubs/Hlo_developlib/"*str2.bw \
   "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/mRNA_bw"

# copy sRNA bigwig files
cp "/Volumes/Install macOS Mojave/Trackhubs/Hlo_developlib/"*plus.bw \
   "/Volumes/Install macOS Mojave/Trackhubs/Hlo_developlib/"*minus.bw \
   "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/coverage_rpm_sRNA_bw"
```

### Rename bigwig files
```sh
# Rename mRNA bw files
HaeL2018_OkayamaE1.normByRPM.str1.bw --> OkayamaE1Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaE1.normByRPM.str2.bw --> OkayamaE1Aligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaE5.normByRPM.str1.bw --> OkayamaE5Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaE5.normByRPM.str2.bw --> OkayamaE5Aligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaE10.normByRPM.str1.bw --> OkayamaE10Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaE10.normByRPM.str2.bw --> OkayamaE10Aligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaE15.normByRPM.str1.bw --> OkayamaE15Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaE15.normByRPM.str2.bw --> OkayamaE15Aligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaE20.normByRPM.str1.bw --> OkayamaE20Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaE20.normByRPM.str2.bw --> OkayamaE20Aligned.sortedByCoord.out.minus.bw

HaeL2018_Okayama-L-New.normByRPM.str1.bw --> Okayama-L-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_Okayama-L-New.normByRPM.str2.bw --> Okayama-L-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_Okayama-N-New.normByRPM.str1.bw --> Okayama-N-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_Okayama-N-New.normByRPM.str2.bw --> Okayama-N-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaA.normByRPM.str1.bw --> OkayamaAAligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaA.normByRPM.str2.bw --> OkayamaAAligned.sortedByCoord.out.minus.bw

HaeL2018_OkayamaAFed1.normByRPM.str1.bw --> OkayamaAFed1Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaAFed1.normByRPM.str2.bw --> OkayamaAFed1Aligned.sortedByCoord.out.minus.bw
HaeL2018_OkayamaAFed2.normByRPM.str1.bw --> OkayamaAFed2Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaAFed2.normByRPM.str2.bw --> OkayamaAFed2Aligned.sortedByCoord.out.minus.bw
HaeL2018_OkayamaAFed3.normByRPM.str1.bw --> OkayamaAFed3Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaAFed3.normByRPM.str2.bw --> OkayamaAFed3Aligned.sortedByCoord.out.minus.bw
HaeL2018_OkayamaAFed4.normByRPM.str1.bw --> OkayamaAFed4Aligned.sortedByCoord.out.plus.bw
HaeL2018_OkayamaAFed4.normByRPM.str2.bw --> OkayamaAFed4Aligned.sortedByCoord.out.minus.bw

HaeL2018_OitaE1-New.normByRPM.str1.bw --> OitaE1-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaE1-New.normByRPM.str2.bw --> OitaE1-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaE5.normByRPM.str1.bw --> OitaE5-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaE5.normByRPM.str2.bw --> OitaE5-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaE10.normByRPM.str1.bw --> OitaE10-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaE10.normByRPM.str2.bw --> OitaE10-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaE15.normByRPM.str1.bw --> OitaE15-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaE15.normByRPM.str1.bw --> OitaE15-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaE20.normByRPM.str1.bw --> OitaE20-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaE20.normByRPM.str1.bw --> OitaE20-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_Oita-L-New.normByRPM.str1.bw --> Oita-L-NewAligned.sortedByCoord.out.plus.bw
HaeL2018_Oita-L-New.normByRPM.str2.bw --> Oita-L-NewAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaN.normByRPM.str1.bw --> OitaNAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaN.normByRPM.str2.bw --> OitaNAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaM.normByRPM.str1.bw --> OitaMAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaM.normByRPM.str2.bw --> OitaMAligned.sortedByCoord.out.minus.bw

HaeL2018_OitaF.normByRPM.str1.bw --> OitaFAligned.sortedByCoord.out.plus.bw
HaeL2018_OitaF.normByRPM.str2.bw --> OitaFAligned.sortedByCoord.out.minus.bw

# Rename sRNA bw files

N002.HaeL2018.normRPMminus.bw --> N002.minus.coverage.rpm.bw
N002.HaeL2018.normRPMplus.bw --> N002.plus.coverage.rpm.bw

N003.HaeL2018.normRPMminus.bw --> N003.minus.coverage.rpm.bw
N003.HaeL2018.normRPMplus.bw --> N003.plus.coverage.rpm.bw

N004.HaeL2018.normRPMminus.bw --> N004.minus.coverage.rpm.bw
N004.HaeL2018.normRPMplus.bw --> N004.plus.coverage.rpm.bw

N005.HaeL2018.normRPMminus.bw --> N005.minus.coverage.rpm.bw
N005.HaeL2018.normRPMplus.bw --> N005.plus.coverage.rpm.bw

N006.HaeL2018.normRPMminus.bw --> N006.minus.coverage.rpm.bw
N006.HaeL2018.normRPMplus.bw --> N006.plus.coverage.rpm.bw

N007.HaeL2018.normRPMminus.bw --> N007.minus.coverage.rpm.bw
N007.HaeL2018.normRPMplus.bw --> N007.plus.coverage.rpm.bw

N008.HaeL2018.normRPMminus.bw --> N008.minus.coverage.rpm.bw
N008.HaeL2018.normRPMplus.bw --> N008.plus.coverage.rpm.bw

N009.HaeL2018.normRPMminus.bw --> N009.minus.coverage.rpm.bw
N009.HaeL2018.normRPMplus.bw --> N009.plus.coverage.rpm.bw

N052.HaeL2018.normRPMminus.bw --> N052.minus.coverage.rpm.bw
N052.HaeL2018.normRPMplus.bw --> N052.plus.coverage.rpm.bw

N010.HaeL2018.normRPMminus.bw --> N010.minus.coverage.rpm.bw
N010.HaeL2018.normRPMplus.bw --> N010.plus.coverage.rpm.bw

N011.HaeL2018.normRPMminus.bw --> N011.minus.coverage.rpm.bw
N011.HaeL2018.normRPMplus.bw --> N011.plus.coverage.rpm.bw

N012.HaeL2018.normRPMminus.bw --> N012.minus.coverage.rpm.bw
N012.HaeL2018.normRPMplus.bw --> N012.plus.coverage.rpm.bw

N013.HaeL2018.normRPMminus.bw --> N013.minus.coverage.rpm.bw
N013.HaeL2018.normRPMplus.bw --> N013.plus.coverage.rpm.bw

N014.HaeL2018.normRPMminus.bw --> N014.minus.coverage.rpm.bw
N014.HaeL2018.normRPMplus.bw --> N014.plus.coverage.rpm.bw

N015.HaeL2018.normRPMminus.bw --> N015.minus.coverage.rpm.bw
N015.HaeL2018.normRPMplus.bw --> N015.plus.coverage.rpm.bw

N016.HaeL2018.normRPMminus.bw --> N016.minus.coverage.rpm.bw
N016.HaeL2018.normRPMplus.bw --> N016.plus.coverage.rpm.bw

N017.HaeL2018.normRPMminus.bw --> N017.minus.coverage.rpm.bw
N017.HaeL2018.normRPMplus.bw --> N017.plus.coverage.rpm.bw

N018.HaeL2018.normRPMminus.bw --> N018.minus.coverage.rpm.bw
N018.HaeL2018.normRPMplus.bw --> N018.plus.coverage.rpm.bw

```

---
## Map N133- N138 sRNA seq data

##### fasta file location:
`/project/okamura-lab-hpc/20240311_BGI_ISE6_virus` 

```sh
# Copy N133-N138 .fasta files
cd /work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138

cp /project/okamura-lab-hpc/20240311_BGI_ISE6_virus/N13{3..8}clipl18t30ntcolid.fasta \
    /work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138
```
#### Installation
```sh
# Install ShortStack version 4.10
conda create -n shortstack4_env
conda install -c bioconda -c conda-forge shortstack=4.1.0
# version 4.1.0
```
#### Change:
'-v', '1', to '-v', '0',
```sh
which ShortStack
# /work/ma-discar/anaconda3/envs/shortstack4_env/bin/ShortStack
vi $(which ShortStack)
```

## Script
```sh
cd /work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138
vi N133_N138_sRNA_bw.sh
```
```sh
#!/bin/bash
#SBATCH -p cluster_short
#SBATCH -t 4:00:00
#SBATCH -n 12
#SBATCH --mem=80G
#SBATCH --job-name=HaeL_N133_N138_ShortStack
#SBATCH -o %j.out
#SBATCH -e %j.err
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp
#SBATCH --mail-type=ALL


################################################################################
# Activate environment
################################################################################

source ~/anaconda3/etc/profile.d/conda.sh
conda activate shortstack4_env

################################################################################
# Check version
################################################################################

echo "**** ShortStack version (This should be 4.1.0) ****"
ShortStack --version

################################################################################
# CAUTION
################################################################################

echo "**** CAUTION ****"
echo "ShortStack source code was modified:"
echo "Before:  -v 1"
echo "After:   -v 0"

################################################################################
# Directories
################################################################################

INDIR=/work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138

GENOME=/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.IGV.genome.fasta

OUTDIR=/work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138

mkdir -p ${OUTDIR}

################################################################################
# Run ShortStack
################################################################################

for LIB in N{133..138}
do

    echo "=================================================="
    echo "Processing ${LIB}"
    echo "=================================================="

    INPUT=${INDIR}/${LIB}clipl18t30ntcolid.fasta

    SAMPLE_OUT=${OUTDIR}/${LIB}

    echo "Input file:"
    echo "${INPUT}"

    echo "Output directory:"
    echo "${SAMPLE_OUT}"

    date

    ShortStack \
        --threads 1 \
        --outdir ${SAMPLE_OUT} \
        --dicermin 18 \
        --dicermax 30 \
        --readfile ${INPUT} \
        --genomefile ${GENOME} \
        --make_bigwigs

    date

done

################################################################################

echo "All analyses completed."

date
```
### Run Option 1 (with job scheduler)
 ```sh
sbatch N133_N138_sRNA_bw.sh
# Submitted batch job 65808
```

#### Note:
> ShortStack --make_bigwig automatically normalize by RPM

- ## Merge shortstack bigwig files

```sh
cd /work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138
vi merge_shortstack_bw.sh
```

```sh
#!/bin/bash
#SBATCH -p cluster_short
#SBATCH -t 4:00:00
#SBATCH -n 12
#SBATCH --mem=80G
#SBATCH --job-name=merge_shortstack_bw
#SBATCH -o %j.out
#SBATCH -e %j.err
#SBATCH --mail-user=discar.ma_divina_kristi.di5@naist.ac.jp
#SBATCH --mail-type=ALL


UCSC=/work/ma-discar/UCSC_utilities
export PATH=${UCSC}:${PATH}

CHROMSIZE=/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes

INDIR=/work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138


for SAMPLE in N133 N134 N135 N136 N137 N138
do

    echo "=================================================="
    echo "Processing ${SAMPLE}"
    echo "=================================================="


    SAMPLE_DIR=${INDIR}/${SAMPLE}

    OUTDIR=${SAMPLE_DIR}/merged_bw

    mkdir -p ${OUTDIR}


    PREFIX=${SAMPLE}clipl18t30ntcolid_condensed


    ########################################################################
    # Merge plus strand
    ########################################################################

    echo "Merging plus strand..."

    bigWigMerge \
    ${SAMPLE_DIR}/${PREFIX}_21_p.bw \
    ${SAMPLE_DIR}/${PREFIX}_22_p.bw \
    ${SAMPLE_DIR}/${PREFIX}_23-24_p.bw \
    ${SAMPLE_DIR}/${PREFIX}_other_p.bw \
    ${OUTDIR}/${SAMPLE}.plus.bedGraph


    ########################################################################
    # Merge minus strand
    ########################################################################

    echo "Merging minus strand..."

    bigWigMerge -threshold=-10000.0 \
    ${SAMPLE_DIR}/${PREFIX}_21_m.bw \
    ${SAMPLE_DIR}/${PREFIX}_22_m.bw \
    ${SAMPLE_DIR}/${PREFIX}_23-24_m.bw \
    ${SAMPLE_DIR}/${PREFIX}_other_m.bw \
    ${OUTDIR}/${SAMPLE}.minus.bedGraph


    ########################################################################
    # Convert to BigWig
    ########################################################################

    echo "Converting plus strand..."

    bedGraphToBigWig \
    ${OUTDIR}/${SAMPLE}.plus.bedGraph \
    ${CHROMSIZE} \
    ${OUTDIR}/${SAMPLE}.plus.bw


    echo "Converting minus strand..."

    bedGraphToBigWig \
    ${OUTDIR}/${SAMPLE}.minus.bedGraph \
    ${CHROMSIZE} \
    ${OUTDIR}/${SAMPLE}.minus.bw


    ########################################################################
    # Remove intermediate bedGraph files
    ########################################################################

    rm ${OUTDIR}/${SAMPLE}.plus.bedGraph
    rm ${OUTDIR}/${SAMPLE}.minus.bedGraph


    echo "${SAMPLE} completed!"
    echo ""

done


echo "=================================================="
echo "All samples completed!"
echo "=================================================="
```
- #### Run `merge_shortstack_bw.sh@
```sh
sbatch merge_shortstack_bw.sh
#Submitted batch job 67349
```

- ### Copy merged shortstack bigwig files
```sh
# Copy from /work/ma-discar to /project/okamura-lab/Vina

find /work/ma-discar/PHAS/Hlongicornis/data/sRNA/N133_N138 \
    -type f \
    -path "*/merged_bw/*.bw" \
    -exec cp -v {} /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/N133_N138/ \;

# Copy from /project/okamura-lab to /Volumes/Install macOS Mojave/Trackhubs and rename

cp /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/N133_N138/*.bw \
"/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/coverage_rpm_sRNA_bw/"

# Rename N133-N138

N133.minus.bw --> N133.minus.coverage.rpm.bw  
N133.plus.bw --> N133.plus.coverage.rpm.bw  

N134.minus.bw --> N134.minus.coverage.rpm.bw 
N134.plus.bw --> N134.plus.coverage.rpm.bw  

N135.minus.bw --> N135.minus.coverage.rpm.bw 
N135.plus.bw --> N135.plus.coverage.rpm.bw  

N136.minus.bw --> N136.minus.coverage.rpm.bw 
N136.plus.bw --> N136.plus.coverage.rpm.bw  

N137.minus.bw --> N137.minus.coverage.rpm.bw 
N137.plus.bw --> N137.plus.coverage.rpm.bw  

N138.minus.bw --> N138.minus.coverage.rpm.bw 
N138.plus.bw --> N138.plus.coverage.rpm.bw  

```

### Track order in IGV
- positive strand is shown above the negative strand.

```sh
 cd "/Volumes/Install macOS Mojave/Vina/PHASER/backend/app/services"
 vi track_service.py # modify to rearrange track order
```

#### Previous backed/frontend files to check 


    - track_service.py
    - tracks.py
    - IGVViewer.tsx

- #### track_service.py
```sh
"""Track Service (IGV Integration - FULL UPDATED VERSION)"""

import pandas as pd

from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    TRANSCRIPT_BED12,
    BW_DIR,
    MRNA_BW_DIR,
    FIVEPRIME_BW_DIR,
    COVERAGE_BW_DIR
)

from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


# =========================================================
# �� SAMPLE LABELS
# =========================================================
SAMPLE_LABELS = {
    "N002": "OkayamaE1",
    "N003": "OkayamaE5",
    "N004": "OkayamaE10",
    "N005": "OkayamaE15",
    "N006": "OkayamaE20",
    "N007": "Okayama_Larva_Unfed",
    "N133": "Okayama_Larva_fed",
    "N008": "Okayama_Nymph_Unfed",
    "N135": "Okayama_Nymph_fed",
    "N009": "Okayama_Adult_Unfed",
    "N052": "Okayama_Adult_Fed",
    "N010": "OitaE1",
    "N011": "OitaE5",
    "N012": "OitaE10",
    "N013": "OitaE15",
    "N014": "OitaE20",
    "N015": "Oita_Larva_Unfed",
    "N134": "Oita_Larva_fed",
    "N016": "Oita_Nymph_Unfed",
    "N136": "Oita_Nymph_fed",
    "N017": "Oita_Adultmale_Unfed",
    "N137": "Oita_Adultmale_fed",
    "N018": "Oita_Adultfemale_Unfed",
    "N138": "Oita_Adultfemale_fed",
}

# =========================================================
# 🔷 STRAIN COLORS
# =========================================================
STRAIN_COLORS = {
    "Okayama": "blue",
    "Oita": "#C2185B"
}


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # =====================================================
    # 🔷 LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    # 🔷 ALL TRACKS
    # =====================================================
    def get_all_tracks(self) -> List[TrackFile]:

        df = self._load_library_info()

        tracks = []

        for _, row in df.iterrows():

            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get(
                        "Sample",
                        row["library_ID"]
                    ),
                    category=row.get(
                        "Category",
                        "unknown"
                    ),
                    default_setting=row.get(
                        "default_setting",
                        "off"
                    ),
                    files={}
                )
            )

        return tracks

    # =====================================================
    # 🔷 TRACK BY ID
    # =====================================================
    def get_track_by_id(
        self,
        library_id: str
    ) -> Optional[TrackFile]:

        df = self._load_library_info()

        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get(
                "Sample",
                row["library_ID"]
            ),
            category=row.get(
                "Category",
                "unknown"
            ),
            default_setting=row.get(
                "default_setting",
                "off"
            ),
            files={}
        )

    # =====================================================
    # 🔷 GENOME FILES
    # =====================================================
    def get_genome_files(self) -> Dict[str, str]:

        return {

            "fasta": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "fai": (
                f"/api/files/genome/{GENOME_FAI.name}"
            ),

            "phas_annotation": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "gff3": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "gff3_index": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            )
        }

    # =====================================================
    # 🔷 FIND BIGWIG FILES
    # =====================================================
    def _find_bw_files(self, sample: str):

        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for: {sample}")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            if sample in name:

                print(f"[IGV DEBUG] Candidate: {name}")

                if "plus" in name:
                    plus_file = f

                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file

    # =====================================================
    # 🔷 SINGLE SAMPLE IGV CONFIG
    # =====================================================
    def get_igv_config(
        self,
        sample: str
    ) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:

            print(f"[IGV ERROR] Missing BW files: {sample}")

            return None

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = [

            # =================================================
            #  GENOME ANNOTATION (BED12)
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "bed",

                "url": (
                    f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
                ),


                "displayMode": "EXPANDED",

                "visibilityWindow": 500000,

                "height": 120,

                "color": "green"
            },

            # =================================================
            # 🔷 PHAS LOCI
            # =================================================
            {
                "name": "PHAS loci",

                "type": "annotation",

                "format": "bigBed",

                "url": (
                    f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
                ),

                "color": "orange",

                "height": 50
            },

            # =================================================
            # 🔷 PLUS STRAND
            # =================================================
            {
                "name": f"{sample} (+)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{plus_file.name}"
                ),

                "color": "red",

                "height": 60
            },

            # =================================================
            # 🔷 MINUS STRAND
            # =================================================
            {
                "name": f"{sample} (-)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{minus_file.name}"
                ),

                "color": "blue",

                "height": 60
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    #  ALL IGV TRACKS
    # =====================================================
    def get_all_igv_tracks(self):

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = []

        # =================================================
        #  GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "bed",

            "url": (
                f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
            ),


            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "black"
        })

        # =================================================
        # 🔷 PHAS LOCI
        # =================================================
        tracks.append({

            "name": "PHAS loci",

            "type": "annotation",

            "format": "bigBed",

            "url": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "color": "orange",

            "height": 50
        })

        # =================================================
        # 🔷 sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning sRNA BW directory...")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA",

                "color": STRAIN_COLORS.get(
                    meta["strain"],
                    "gray"
                ),

                "height": 50

            })

        # =================================================
        # 🔷 mRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning mRNA BW directory...")

        for f in MRNA_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} mRNA ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/mRNA_bw/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "mRNA",

                "color": "green",

                "height": 50
            })

        
        # =================================================
        # 🔷 5' END RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning 5' RPM BW directory...")

        for f in FIVEPRIME_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} 5' End ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/5prime_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_5prime",

                "color": "#8E24AA",

                "height": 50
            })

        # =================================================
        # 🔷 COVERAGE RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning coverage RPM BW directory...")

        for f in COVERAGE_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} Coverage ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/coverage_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_coverage",

                "color": "#F57C00",

                "height": 50
            })
        
        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 FILE RESOLUTION
    # =====================================================
    def resolve_file_path(
        self,
        relative_path: str
    ) -> Optional[Path]:

        if relative_path.startswith("5prime_rpm_sRNA_bw/"):

            resolved = (
                FIVEPRIME_BW_DIR
                / relative_path.replace(
                    "5prime_rpm_sRNA_bw/",
                    ""
                )
            )

        elif relative_path.startswith(
            "coverage_rpm_sRNA_bw/"
        ):

            resolved = (
                COVERAGE_BW_DIR
                / relative_path.replace(
                    "coverage_rpm_sRNA_bw/",
                    ""
                )
            )

        else:

            resolved = (
                TRACKHUB_BASE
                / relative_path
            )

        print(

            f"[FILE RESOLVE] "

            f"{relative_path} -> {resolved}"
        
        )


        return resolved

track_service = TrackService()

```


- ### IGVViewer.tsx
```sh
import { useEffect, useRef, useState } from "react";
import igv from "igv";
import styles from "./IGVViewer.module.css";

interface Props {
  sample: string;
  locus: {
    chrom: string;
    start: number;
    end: number;
  } | null;
}

const API_BASE = "http://163.221.246.151:8001";

export default function IGVViewer({
  sample,
  locus,
}: Props) {

  const igvContainerRef =
    useRef<HTMLDivElement | null>(null);

  const igvBrowserRef = useRef<any>(null);

  const isReadyRef = useRef(false);

  const [allTracks, setAllTracks] =
    useState<any[]>([]);

  const [loading, setLoading] =
    useState(false);

  const [filters, setFilters] =
    useState({
      samples: [sample],
      stage: "ALL",
      strain: "ALL",
      feeding: "ALL",
      rnaType: "ALL",
    });

  // =====================================================
  // 🔷 SAMPLE → MATCHING MRNA LABELS
  // =====================================================
  const mRNAMap: Record<
    string,
    string[]
  > = {

    // ===================================================
    // OKAYAMA
    // ===================================================
    N002: ["OkayamaE1"],
    N003: ["OkayamaE5"],
    N004: ["OkayamaE10"],
    N005: ["OkayamaE15"],
    N006: ["OkayamaE20"],

    N007: [
      "Okayama_Larva_Unfed",
      "Okayama-L-New",
    ],

    N133: [
      "Okayama_Larva_Fed",
    ],

    N008: [
      "Okayama_Nymph_Unfed",
      "Okayama-N-New",
    ],

    N135: [
      "Okayama_Nymph_Fed",
    ],

    N009: [
      "OkayamaA",
      "OkayamaAc",
      "Okayama_Adult_female_unfed",
    ],

    N052: [
      "OkayamaAFed",
      "Okayama_Adult_female_fed",
    ],

    // ===================================================
    // OITA
    // ===================================================
    N010: ["OitaE1"],
    N011: ["OitaE5"],
    N012: ["OitaE10"],
    N013: ["OitaE15"],
    N014: ["OitaE20"],

    N015: [
      "Oita_Larva_unfed",
      "Oita-L-New",
    ],

    N134: [
      "Oita_Larva_fed",
    ],

    N016: [
      "Oita_Nymph_Unfed",
      "OitaN",
    ],

    N136: [
      "Oita_Nymph_Fed",
    ],

    N017: [
      "OitaM",
      "Oita_adult_male_unfed",
    ],

    N137: [
      "Oita_adult_male_fed",
    ],

    N018: [
      "OitaF",
      "Oita_adult_female_unfed",
    ],

    N138: [
      "Oita_Adult_female_fed",
    ],
  };

  // =====================================================
  // 🔷 SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // 🔷 LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current)
      return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        // ===============================================
        // 🔷 CLEANUP OLD BROWSER
        // ===============================================
        if (
          igvBrowserRef.current
        ) {

          igvBrowserRef.current
            .destroy?.();

          igvBrowserRef.current =
            null;

          isReadyRef.current =
            false;
        }

        // ===============================================
        // 🔷 FETCH TRACK CONFIG
        // ===============================================
        const res = await fetch(
          `${API_BASE}/api/tracks/igv/all`
        );

        const config =
          await res.json();

        const tracksWithAbsoluteURL =
          config.tracks.map(
            (t: any) => ({
              ...t,

              url:
                API_BASE + t.url,

              indexURL:
                t.indexURL
                  ? API_BASE +
                    t.indexURL
                  : undefined,
            })
          );

        setAllTracks(
          tracksWithAbsoluteURL
        );

        // ===============================================
        // 🔷 CREATE BROWSER
        // ===============================================
        const browser =
          await igv.createBrowser(
            igvContainerRef.current!,
            {
              genome: {
                fastaURL:
                  API_BASE +
                  config.genome
                    .fastaURL,

                indexURL:
                  API_BASE +
                  config.genome
                    .indexURL,
              },

              locus: locus
                ? `${locus.chrom}:${locus.start}-${locus.end}`
                : undefined,

              tracks: [],
            }
          );

        igvBrowserRef.current =
          browser;

        setTimeout(() => {

          isReadyRef.current =
            true;

          setFilters(
            (prev) => ({
              ...prev,
            })
          );

        }, 500);

      } catch (err) {

        console.error(
          "IGV initialization error:",
          err
        );

      } finally {

        setLoading(false);
      }
    };

    loadIGV();

  }, [sample]);

  // =====================================================
  // 🔷 APPLY TRACKS
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !isReadyRef.current ||
      allTracks.length === 0
    ) {
      return;
    }

    const browser =
      igvBrowserRef.current;

    const applyTracks =
      async () => {

        try {

          await browser.removeAllTracks();

          // =============================================
          //  ANNOTATION TRACKS
          // =============================================
          const annotationTracks =
            allTracks.filter(
              (t) =>
                t.format === "bed" ||
                t.format === "bigBed"
            );

          // =============================================
          // 🔷 SIGNAL TRACKS
          // =============================================
          let signalTracks =
            allTracks.filter(
              (t) =>
                t.format !== "bed" &&
                t.format !== "bigBed"
            );

          signalTracks =
            signalTracks.filter(
              (t) => {

                // =======================================
                // 🔷 RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "ALL" ||
                  t.rnaType ===
                    filters.rnaType;

                if (!rnaMatch)
                  return false;

                // =======================================
                // 🔷 SHOW BEST SAMPLE MODE
                // =======================================
                if (
                  filters.samples
                    .length === 1
                ) {

                  // ===================================
                  // 🔷 SRNA
                  // ===================================
                  if (
                      t.rnaType === "sRNA" ||
                      t.rnaType === "sRNA_5prime" ||
                      t.rnaType === "sRNA_coverage"
                  ) {

                      return (
                          t.sample === sample
                      );
                  }

                  // ===================================
                  // 🔷 MRNA
                  // ===================================
                  if (
                    t.rnaType ===
                    "mRNA"
                  ) {

                    const allowedPatterns =
                      mRNAMap[
                        sample
                      ] || [];

                    return allowedPatterns.some(
                      (
                        pattern
                      ) => {

                        const cleanName =
                          t.name
                            .replace(
                              " (+)",
                              ""
                            )
                            .replace(
                              " (-)",
                              ""
                            );

                        // ===============================
                        // 🔷 EXACT MATCH LOGIC
                        // ===============================
                        return (

                          cleanName ===
                            pattern ||

                          cleanName.startsWith(
                            pattern +
                              " "
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "_"
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "-"
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "Aligned"
                          )
                        );
                      }
                    );
                  }

                  return false;
                }

                // =======================================
                // 🔷 SHOW ALL SAMPLES MODE
                // =======================================
                return (
                  (filters.stage ===
                    "ALL" ||
                    t.stage ===
                      filters.stage) &&

                  (filters.strain ===
                    "ALL" ||
                    t.strain ===
                      filters.strain) &&

                  (filters.feeding ===
                    "ALL" ||
                    t.feeding ===
                      filters.feeding)
                );
              }
            );

          // =============================================
          // 🔷 SORT TRACKS
          // =============================================
          signalTracks.sort(
            (a, b) =>
              a.name.localeCompare(
                b.name,
                undefined,
                {
                  numeric: true,
                  sensitivity:
                    "base",
                }
              )
          );

          const finalTracks = [
            ...annotationTracks,
            ...signalTracks,
          ];

          // =============================================
          // 🔷 LOAD TRACKS
          // =============================================
          for (const track of finalTracks) {

            await browser.loadTrack(
              track
            );
          }

        } catch (err) {

          console.error(
            "Track loading error:",
            err
          );
        }
      };

    applyTracks();

  }, [
    filters,
    allTracks,
    sample,
  ]);

  // =====================================================
  // 🔷 LOCUS UPDATE
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !locus ||
      !isReadyRef.current
    ) {
      return;
    }

    igvBrowserRef.current.search(
      `${locus.chrom}:${locus.start}-${locus.end}`
    );

  }, [locus]);

  // =====================================================
  // 🔷 TOGGLE SAMPLES
  // =====================================================
  const toggleAllSamples =
    () => {

      const allSamples =
        Array.from(
          new Set(
            allTracks
              .map(
                (t) => t.sample
              )
              .filter(Boolean)
          )
        );

      if (
        filters.samples
          .length === 1
      ) {

        setFilters((f) => ({
          ...f,
          samples: allSamples,
        }));

      } else {

        setFilters((f) => ({
          ...f,
          samples: [sample],
        }));
      }
    };

  return (
    <div className={styles.container}>

      <div className={styles.header}>
        Genome Browser (IGV)
        — {sample}

        {locus && (
          <div
            className={
              styles.defaultLabel
            }
          >
            Viewing Best Region
          </div>
        )}
      </div>

      <div className={styles.controls}>

        <button
          onClick={
            toggleAllSamples
          }
        >
          {filters.samples
            .length === 1
            ? "Show All Samples"
            : "Show Best Sample"}
        </button>

        <select
          value={filters.stage}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              stage:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Stages
          </option>

          <option value="Embryo">
            Embryo
          </option>

          <option value="Larva">
            Larva
          </option>

          <option value="Nymph">
            Nymph
          </option>

          <option value="Adult">
            Adult
          </option>
        </select>

        <select
          value={filters.strain}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              strain:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Strains
          </option>

          <option value="Okayama">
            Okayama
          </option>

          <option value="Oita">
            Oita
          </option>
        </select>

        <select
          value={filters.feeding}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              feeding:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Feeding
          </option>

          <option value="fed">
            fed
          </option>

          <option value="unfed">
            unfed
          </option>
        </select>

        <select
          value={filters.rnaType}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              rnaType:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Tracks
          </option>

          <option value="mRNA">
            mRNAseq
          </option>
        
          <option value="sRNA_5prime">
            5′ End RPM
          </option>

          <option value="sRNA_coverage">
            Coverage RPM
          </option>
        
        </select>

      </div>

      {loading && (
        <div
          className={
            styles.loading
          }
        >
          Loading IGV...
        </div>
      )}

      <div
        ref={igvContainerRef}
        className={styles.viewer}
      />

    </div>
  );
}
```

## Modified Code

- track_service.py 

```sh
"""Track Service (IGV Integration - FULL UPDATED VERSION)"""

import pandas as pd

from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    TRANSCRIPT_BED12,
    BW_DIR,
    MRNA_BW_DIR,
    FIVEPRIME_BW_DIR,
    COVERAGE_BW_DIR
)

from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


# =========================================================
# �� SAMPLE LABELS
# =========================================================
SAMPLE_LABELS = {
    "N002": "OkayamaE1",
    "N003": "OkayamaE5",
    "N004": "OkayamaE10",
    "N005": "OkayamaE15",
    "N006": "OkayamaE20",
    "N007": "Okayama_Larva_Unfed",
    "N133": "Okayama_Larva_fed",
    "N008": "Okayama_Nymph_Unfed",
    "N135": "Okayama_Nymph_fed",
    "N009": "Okayama_Adult_Unfed",
    "N052": "Okayama_Adult_Fed",
    "N010": "OitaE1",
    "N011": "OitaE5",
    "N012": "OitaE10",
    "N013": "OitaE15",
    "N014": "OitaE20",
    "N015": "Oita_Larva_Unfed",
    "N134": "Oita_Larva_fed",
    "N016": "Oita_Nymph_Unfed",
    "N136": "Oita_Nymph_fed",
    "N017": "Oita_Adultmale_Unfed",
    "N137": "Oita_Adultmale_fed",
    "N018": "Oita_Adultfemale_Unfed",
    "N138": "Oita_Adultfemale_fed",
}

# =========================================================
# 🔷 mRNA FILE → sRNA SAMPLE ID MAPPING
# =========================================================

MRNA_TO_SAMPLE = {

    # Okayama
    "OkayamaE1Aligned": "N002",
    "OkayamaE5Aligned": "N003",
    "OkayamaE10Aligned": "N004",
    "OkayamaE15Aligned": "N005",
    "OkayamaE20Aligned": "N006",

    "Okayama_Larva_unfedAligned": "N007",
    "Okayama_Larva_fedAligned": "N133",

    "Okayama_Nymph_UnfedAligned": "N008",
    "Okayama_Nymph_FedAligned": "N135",

    "Okayama_adult_female_unfed_1Aligned": "N009",
    "Okayama_adult_female_unfed_2Aligned": "N009",
    "Okayama_adult_female_unfed_3Aligned": "N009",

    "Okayama_adult_female_fed_1Aligned": "N052",
    "Okayama_adult_female_fed_2Aligned": "N052",
    "Okayama_adult_female_fed_3Aligned": "N052",


    # Oita
    "OitaE1Aligned": "N010",
    "OitaE5Aligned": "N011",
    "OitaE10Aligned": "N012",
    "OitaE15Aligned": "N013",
    "OitaE20Aligned": "N014",

    "Oita_Larva_unfedAligned": "N015",
    "Oita_Larva_fedAligned": "N134",

    "Oita_Nymph_UnfedAligned": "N016",
    "Oita_Nymph_FedAligned": "N136",

    "Oita_adult_male_unfedAligned": "N017",
    "Oita_adult_male_fedAligned": "N137",

    "Oita_adult_female_unfed_1Aligned": "N018",
    "Oita_adult_female_unfed_2Aligned": "N018",
    "Oita_adult_female_unfed_3Aligned": "N018",

    "Oita_adult_female_fed_1Aligned": "N138",
    "Oita_adult_female_fed_2Aligned": "N138",
    "Oita_adult_female_fed_3Aligned": "N138",
}

# =========================================================
# 🔷 SAMPLE DISPLAY ORDER
# =========================================================
SAMPLE_ORDER = [
    "N002", "N003", "N004", "N005", "N006",
    "N007", "N133",
    "N008", "N135",
    "N009", "N052",

    "N010", "N011", "N012", "N013", "N014",
    "N015", "N134",
    "N016", "N136",
    "N017", "N137",
    "N018", "N138"
]

# =========================================================
# 🔷 mRNA SAMPLE ORDER MAPPING
# =========================================================
MRNA_SAMPLE_ORDER = {


# =========================================================
# 🔷 STRAIN COLORS
# =========================================================
STRAIN_COLORS = {
    "Okayama": "blue",
    "Oita": "#C2185B"
}


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # =====================================================
    # 🔷 LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    # 🔷 ALL TRACKS
    # =====================================================
    def get_all_tracks(self) -> List[TrackFile]:

        df = self._load_library_info()

        tracks = []

        for _, row in df.iterrows():

            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get(
                        "Sample",
                        row["library_ID"]
                    ),
                    category=row.get(
                        "Category",
                        "unknown"
                    ),
                    default_setting=row.get(
                        "default_setting",
                        "off"
                    ),
                    files={}
                )
            )

        return tracks

    # =====================================================
    # 🔷 TRACK BY ID
    # =====================================================
    def get_track_by_id(
        self,
        library_id: str
    ) -> Optional[TrackFile]:

        df = self._load_library_info()

        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get(
                "Sample",
                row["library_ID"]
            ),
            category=row.get(
                "Category",
                "unknown"
            ),
            default_setting=row.get(
                "default_setting",
                "off"
            ),
            files={}
        )

    # =====================================================
    # 🔷 GENOME FILES
    # =====================================================
    def get_genome_files(self) -> Dict[str, str]:

        return {

            "fasta": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "fai": (
                f"/api/files/genome/{GENOME_FAI.name}"
            ),

            "phas_annotation": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "gff3": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "gff3_index": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            )
        }

    # =====================================================
    # 🔷 FIND BIGWIG FILES
    # =====================================================
    def _find_bw_files(self, sample: str):

        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for: {sample}")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            if sample in name:

                print(f"[IGV DEBUG] Candidate: {name}")

                if "plus" in name:
                    plus_file = f

                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file

    # =====================================================
    # 🔷 SINGLE SAMPLE IGV CONFIG
    # =====================================================
    def get_igv_config(
        self,
        sample: str
    ) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:

            print(f"[IGV ERROR] Missing BW files: {sample}")

            return None

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = [

            # =================================================
            #  GENOME ANNOTATION (BED12)
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "bed",

                "url": (
                    f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
                ),


                "displayMode": "EXPANDED",

                "visibilityWindow": 500000,

                "height": 120,

                "color": "green"
            },

            # =================================================
            # 🔷 PHAS LOCI
            # =================================================
            {
                "name": "PHAS loci",

                "type": "annotation",

                "format": "bigBed",

                "url": (
                    f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
                ),

                "color": "orange",

                "height": 50
            },

            # =================================================
            # 🔷 PLUS STRAND
            # =================================================
            {
                "name": f"{sample} (+)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{plus_file.name}"
                ),

                "color": "red",

                "height": 60
            },

            # =================================================
            # 🔷 MINUS STRAND
            # =================================================
            {
                "name": f"{sample} (-)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{minus_file.name}"
                ),

                "color": "blue",

                "height": 60
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    #  ALL IGV TRACKS
    # =====================================================
    def get_all_igv_tracks(self):

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = []

        # =================================================
        #  GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "bed",

            "url": (
                f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
            ),


            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "black"
        })

        # =================================================
        # 🔷 PHAS LOCI
        # =================================================
        tracks.append({

            "name": "PHAS loci",

            "type": "annotation",

            "format": "bigBed",

            "url": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "color": "orange",

            "height": 50
        })
            

        # =================================================
        # 🔷 mRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning mRNA BW directory...")

        files = sorted(
            MRNA_BW_DIR.glob("*.bw"),
            key=lambda x: (

                SAMPLE_ORDER.index(
                    MRNA_TO_SAMPLE.get(
                        x.name.replace(".sortedByCoord.out.plus.bw","")
                             .replace(".sortedByCoord.out.minus.bw",""),
                        "UNKNOWN"
                    )
                )
                if MRNA_TO_SAMPLE.get(
                    x.name.replace(".sortedByCoord.out.plus.bw","")
                         .replace(".sortedByCoord.out.minus.bw","")
                )
                else 999,

                # plus first, minus second
                0 if "plus" in x.name else 1
            ) 
        )

        for f in reversed(files):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} mRNA ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/mRNA_bw/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "mRNA",

                "color": "green",

                "height": 50
            })

        
        # =================================================
        # 🔷 5' END RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning 5' RPM BW directory...")

        files = sorted(
            FIVEPRIME_BW_DIR.glob("*.bw"),
            key=lambda x: (
                SAMPLE_ORDER.index(x.name.split(".")[0]),
                0 if ".plus." in x.name else 1
            )
        )

        for f in reversed(files):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} 5' End ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/5prime_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_5prime",

                "color": "#8E24AA",

                "height": 50
            })

        # =================================================
        # 🔷 COVERAGE RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning coverage RPM BW directory...")

        files = sorted(
            COVERAGE_BW_DIR.glob("*.bw"),
            key=lambda x: (
                SAMPLE_ORDER.index(x.name.split(".")[0]),
                0 if ".plus." in x.name else 1
            )
        )

        for f in reversed(files):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} Coverage ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/coverage_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_coverage",

                "color": "#F57C00",

                "height": 50
            })
        
        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 FILE RESOLUTION
    # =====================================================
    def resolve_file_path(
        self,
        relative_path: str
    ) -> Optional[Path]:

        if relative_path.startswith("5prime_rpm_sRNA_bw/"):

            resolved = (
                FIVEPRIME_BW_DIR
                / relative_path.replace(
                    "5prime_rpm_sRNA_bw/",
                    ""
                )
            )

        elif relative_path.startswith(
            "coverage_rpm_sRNA_bw/"
        ):

            resolved = (
                COVERAGE_BW_DIR
                / relative_path.replace(
                    "coverage_rpm_sRNA_bw/",
                    ""
                )
            )

        else:

            resolved = (
                TRACKHUB_BASE
                / relative_path
            )

        print(

            f"[FILE RESOLVE] "

            f"{relative_path} -> {resolved}"
        
        )


        return resolved

track_service = TrackService()
```

- IGVViewer.tsx
```sh
import { useEffect, useRef, useState } from "react";
import igv from "igv";
import styles from "./IGVViewer.module.css";

interface Props {
  sample: string;
  locus: {
    chrom: string;
    start: number;
    end: number;
  } | null;
}

const API_BASE = "http://163.221.246.151:8001";

export default function IGVViewer({
  sample,
  locus,
}: Props) {

  const igvContainerRef =
    useRef<HTMLDivElement | null>(null);

  const igvBrowserRef = useRef<any>(null);

  const isReadyRef = useRef(false);

  const [allTracks, setAllTracks] =
    useState<any[]>([]);

  const [loading, setLoading] =
    useState(false);

  const [filters, setFilters] =
    useState({
      samples: [sample],
      stage: "ALL",
      strain: "ALL",
      feeding: "ALL",
      rnaType: "ALL",
    });

  // =====================================================
  // 🔷 SAMPLE → MATCHING MRNA LABELS
  // =====================================================
  const mRNAMap: Record<
    string,
    string[]
  > = {

    // ===================================================
    // OKAYAMA
    // ===================================================
    N002: ["OkayamaE1"],
    N003: ["OkayamaE5"],
    N004: ["OkayamaE10"],
    N005: ["OkayamaE15"],
    N006: ["OkayamaE20"],

    N007: [
      "Okayama_Larva_Unfed",
      "Okayama-L-New",
    ],

    N133: [
      "Okayama_Larva_fed",
    ],

    N008: [
      "Okayama_Nymph_Unfed",
      "Okayama-N-New",
    ],

    N135: [
      "Okayama_Nymph_Fed",
    ],

    N009: [
      "OkayamaA",
      "OkayamaAc",
      "Okayama_Adult_female_unfed",
    ],

    N052: [
      "Okayama_Adult_Fed",
      "Okayama_adult_female_fed",
      "OkayamaAFed",
    ],

    // ===================================================
    // OITA
    // ===================================================
    N010: ["OitaE1"],
    N011: ["OitaE5"],
    N012: ["OitaE10"],
    N013: ["OitaE15"],
    N014: ["OitaE20"],

    N015: [
      "Oita_Larva_unfed",
      "Oita-L-New",
    ],

    N134: [
      "Oita_Larva_fed",
    ],

    N016: [
      "Oita_Nymph_Unfed",
      "OitaN",
    ],

    N136: [
      "Oita_Nymph_Fed",
    ],

    N017: [
      "OitaM",
      "Oita_adult_male_unfed",
    ],

    N137: [
      "Oita_adult_male_fed",
    ],

    N018: [
      "OitaF",
      "Oita_adult_female_unfed",
    ],

    N138: [
      "Oita_Adult_female_fed",
    ],
  };

  // =====================================================
  // 🔷 SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // 🔷 LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current)
      return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        // ===============================================
        // 🔷 CLEANUP OLD BROWSER
        // ===============================================
        if (
          igvBrowserRef.current
        ) {

          igvBrowserRef.current
            .destroy?.();

          igvBrowserRef.current =
            null;

          isReadyRef.current =
            false;
        }

        // ===============================================
        // 🔷 FETCH TRACK CONFIG
        // ===============================================
        const res = await fetch(
          `${API_BASE}/api/tracks/igv/all`
        );

        const config =
          await res.json();

        const tracksWithAbsoluteURL =
          config.tracks.map(
            (t: any) => ({
              ...t,

              url:
                API_BASE + t.url,

              indexURL:
                t.indexURL
                  ? API_BASE +
                    t.indexURL
                  : undefined,
            })
          );

        setAllTracks(
          tracksWithAbsoluteURL
        );

        // ===============================================
        // 🔷 CREATE BROWSER
        // ===============================================
        const browser =
          await igv.createBrowser(
            igvContainerRef.current!,
            {
              genome: {
                fastaURL:
                  API_BASE +
                  config.genome
                    .fastaURL,

                indexURL:
                  API_BASE +
                  config.genome
                    .indexURL,
              },

              locus: locus
                ? `${locus.chrom}:${locus.start}-${locus.end}`
                : undefined,

              tracks: [],
            }
          );

        igvBrowserRef.current =
          browser;

        setTimeout(() => {

          isReadyRef.current =
            true;

          setFilters(
            (prev) => ({
              ...prev,
            })
          );

        }, 500);

      } catch (err) {

        console.error(
          "IGV initialization error:",
          err
        );

      } finally {

        setLoading(false);
      }
    };

    loadIGV();

  }, [sample]);

  // =====================================================
  // 🔷 APPLY TRACKS
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !isReadyRef.current ||
      allTracks.length === 0
    ) {
      return;
    }

    const browser =
      igvBrowserRef.current;

    const applyTracks =
      async () => {

        try {

          await browser.removeAllTracks();

          // =============================================
          //  ANNOTATION TRACKS
          // =============================================
          const annotationTracks =
            allTracks.filter(
              (t) =>
                t.format === "bed" ||
                t.format === "bigBed"
            );

          // =============================================
          // 🔷 SIGNAL TRACKS
          // =============================================
          let signalTracks =
            allTracks.filter(
              (t) =>
                t.format !== "bed" &&
                t.format !== "bigBed"
            );

          signalTracks =
            signalTracks.filter(
              (t) => {

                // =======================================
                // 🔷 RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "ALL" ||
                  t.rnaType ===
                    filters.rnaType;

                if (!rnaMatch)
                  return false;

                // =======================================
                // 🔷 SHOW BEST SAMPLE MODE
                // =======================================
                if (
                  filters.samples
                    .length === 1
                ) {

                  // ===================================
                  // 🔷 SRNA
                  // ===================================
                  if (
                      t.rnaType === "sRNA" ||
                      t.rnaType === "sRNA_5prime" ||
                      t.rnaType === "sRNA_coverage"
                  ) {

                      return (
                          t.sample === sample
                      );
                  }

                  // ===================================
                  // 🔷 MRNA
                  // ===================================
                  if (
                    t.rnaType ===
                    "mRNA"
                  ) {

                    const allowedPatterns =
                      mRNAMap[
                        sample
                      ] || [];

                    return allowedPatterns.some(
                      (
                        pattern
                      ) => {

                        const cleanName =
                          t.name
                            .replace(
                              " (+)",
                              ""
                            )
                            .replace(
                              " (-)",
                              ""
                            );

                        // ===============================
                        // 🔷 EXACT MATCH LOGIC
                        // ===============================
                        return (

                          cleanName ===
                            pattern ||

                          cleanName.startsWith(
                            pattern +
                              " "
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "_"
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "-"
                          ) ||

                          cleanName.startsWith(
                            pattern +
                              "Aligned"
                          )
                        );
                      }
                    );
                  }

                  return false;
                }

                // =======================================
                // 🔷 SHOW ALL SAMPLES MODE
                // =======================================
                return (
                  (filters.stage ===
                    "ALL" ||
                    t.stage ===
                      filters.stage) &&

                  (filters.strain ===
                    "ALL" ||
                    t.strain ===
                      filters.strain) &&

                  (filters.feeding ===
                    "ALL" ||
                    t.feeding ===
                      filters.feeding)
                );
              }
            );

          // =============================================
          // 🔷 SORT TRACKS
          // =============================================

          const finalTracks = [
            ...annotationTracks,
            ...signalTracks,
          ];

          // =============================================
          // 🔷 LOAD TRACKS
          // =============================================
          for (const track of [...finalTracks].reverse()) {

            await browser.loadTrack(
              track
            );
          }

        } catch (err) {

          console.error(
            "Track loading error:",
            err
          );
        }
      };

    applyTracks();

  }, [
    filters,
    allTracks,
    sample,
  ]);

  // =====================================================
  // 🔷 LOCUS UPDATE
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !locus ||
      !isReadyRef.current
    ) {
      return;
    }

    igvBrowserRef.current.search(
      `${locus.chrom}:${locus.start}-${locus.end}`
    );

  }, [locus]);

  // =====================================================
  // 🔷 TOGGLE SAMPLES
  // =====================================================
  const toggleAllSamples =
    () => {

      const allSamples =
        Array.from(
          new Set(
            allTracks
              .map(
                (t) => t.sample
              )
              .filter(Boolean)
          )
        );

      if (
        filters.samples
          .length === 1
      ) {

        setFilters((f) => ({
          ...f,
          samples: allSamples,
        }));

      } else {

        setFilters((f) => ({
          ...f,
          samples: [sample],
        }));
      }
    };

  return (
    <div className={styles.container}>

      <div className={styles.header}>
        Genome Browser (IGV)
        — {sample}

        {locus && (
          <div
            className={
              styles.defaultLabel
            }
          >
            Viewing Best Region
          </div>
        )}
      </div>

      <div className={styles.controls}>

        <button
          onClick={
            toggleAllSamples
          }
        >
          {filters.samples
            .length === 1
            ? "Show All Samples"
            : "Show Best Sample"}
        </button>

        <select
          value={filters.stage}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              stage:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Stages
          </option>

          <option value="Embryo">
            Embryo
          </option>

          <option value="Larva">
            Larva
          </option>

          <option value="Nymph">
            Nymph
          </option>

          <option value="Adult">
            Adult
          </option>
        </select>

        <select
          value={filters.strain}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              strain:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Strains
          </option>

          <option value="Okayama">
            Okayama
          </option>

          <option value="Oita">
            Oita
          </option>
        </select>

        <select
          value={filters.feeding}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              feeding:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Feeding
          </option>

          <option value="fed">
            fed
          </option>

          <option value="unfed">
            unfed
          </option>
        </select>

        <select
          value={filters.rnaType}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              rnaType:
                e.target.value,
            }))
          }
        >
          <option value="ALL">
            All Tracks
          </option>

          <option value="mRNA">
            mRNAseq
          </option>
        
          <option value="sRNA_5prime">
            5′ End RPM
          </option>

          <option value="sRNA_coverage">
            Coverage RPM
          </option>
        
        </select>

      </div>

      {loading && (
        <div
          className={
            styles.loading
          }
        >
          Loading IGV...
        </div>
      )}

      <div
        ref={igvContainerRef}
        className={styles.viewer}
      />

    </div>
  );
}
```