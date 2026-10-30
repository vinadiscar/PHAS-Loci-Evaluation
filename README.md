## 1. Set-Up PHASER
- Summary of Workflow is in: 
```sh
PHAS_interface_Workflow_SUMMARY_raw1-16_20261030.md
```
## 2. IGV-Genome- Level Analysis
- Reads mapped to Genome GWHAMMI00000000 :
##### Tracks Preparation
- sRNA 5' nucleotide and coverage tracks
Summary of analysis can be found in:
```sh
PHAS_interface_Workflow_raw_14.html
``` 
- mRNA tracks
Original mRNA files can be found in:
```sh
"/Volumes/NAIST-data021/Install macOS Mojave/Trackhubs/Hlo_developlib"
```
- sRNA tracks original source (genome-mapping bowtie)
```sh
https://github.com/CanranFeng/phasiRNA/blob/main/sRNAminer_PHAS.md#batch-modeoita--okayama
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bowtie
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam
```
- ##### Genome annotation uses: `HaeL2018_stringtie_merged_transcripts.gff3`
  
## 3. GLobal Characterization of PHAS loci 
#### 07.29.2026 Strand-specific sRNA 5' Nucleotide Analysis:
```sh
PHAS_interface_Workflow_raw_12.html
```
#### sRNA Size Distribution and sRNA Abundance
```sh
PHAS_interface_Workflow_raw_10.html
```

## 4. PHAS Loci Evaluation
- ver. 20260817 Phas loci evaluation summary is in:
```sh
PhasLoci_Classisfication_08172026_.xlsx
```
## MiSiPi.RNA Analysis
```sh
MiSiPi.RNA_Analysis.md
```
## 5. Transcriptome Level Analysis

- #### Transcriptome Reference:  `assembled_transcripts_HaeL2018ref_stringtie_merged.fa`
- #### Clean Total RNA-seq FASTQ files:
A. Libraries N024-N048
Source: /project/okamura-lab/raw_sequencing_data/20211202_hloRNAseq_BGI/F21FTSAPJT0258_INSjubcR/Clean
Copied: /work/ma-discar/20211202_hloRNAseq_BGI/Clean
B. Libraries DRR508XX
Source:  /project/okamura-lab-hpc/20240106_H_longicornis_Vina/fastp/fastp
Copied:  /work/ma-discar/20231214_hloRNAseq_NIG/fastp
- #### Mapping:
```sh
20261028_Map_sRNA_TotalRNA_x_assembled_transcripts_HaeL2018ref_stringtie_merged.md
```
#### Location of Original H. longicornis PHAS Files 
(Source: Canran Feng)
```sh
### sRNA genome-mapping bowtie 
- https://github.com/CanranFeng/phasiRNA/blob/main/sRNAminer_PHAS.md#batch-modeoita--okayama
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bowtie
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam

### HaeL phasiRNAs identification
- https://github.com/CanranFeng/phasiRNA/blob/main/sRNAminer_PHAS.md#batch-modeoita--okayama
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/sRNAminer

### PHAS candidate loci characterization
- https://github.com/CanranFeng/phasiRNA/blob/main/Tick_PHAS_sRNA-seq_characterization.md
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/plots
/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/misipi_RNA
```





