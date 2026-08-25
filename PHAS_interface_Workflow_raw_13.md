##### PHAS_interface_Workflow_raw_13.md
> This version shows scripts for generating and integrating 5' sRNA track to IGV PHASER.

## A. Convert bam files to 5' bigwig files (raw) 
```sh
conda activate bamTobw_pipeline_env
cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data
mkdir -p sRNA/5prime_bw

cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA
vi bam_to_5prime_bw.sh
```
- ##### bam_to_5prime_bw.sh

```sh
#!/bin/bash

set -euo pipefail

INPUT_DIR="/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam"
OUTPUT_DIR="/project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_bw"
GENOME_SIZE="/project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes"
  
mkdir -p "$OUTPUT_DIR"

for bam in "${INPUT_DIR}"/*.bam
do
  base=$(basename "$bam" .bam)

  echo "Processing $base"

  # PLUS strand 5'end
  samtools view -b -F 16 "$bam" \
    | bedtools genomecov -ibam - -bg -5 \
    > "${OUTPUT_DIR}/${base}.plus.bedgraph"

  # MINUS strand 5' end 
  samtools view -b -f 16 "$bam" \
    | bedtools genomecov -ibam - -bg -5 \
    > "${OUTPUT_DIR}/${base}.minus.bedgraph"

  # Convert to BigWig
  bedGraphToBigWig "${OUTPUT_DIR}/${base}.plus.5prime.bedgraph" "$GENOME_SIZE" "${OUTPUT_DIR}/${base}.plus.5prime.bw"
  bedGraphToBigWig "${OUTPUT_DIR}/${base}.minus.5prime.bedgraph" "$GENOME_SIZE" "${OUTPUT_DIR}/${base}.minus.5prime.bw"

done

echo "All done!"
```
- ##### Run `bam_to_5prime_bw.sh`
```sh
chmod +x bam_to_5prime_bw.sh
nohup ./bam_to_5prime_bw.sh > bam_to_5prime_bw.log 2>&1 &
```

- ### remove intermediate bedgraph files 

```sh
rm /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_bw/*.bedgraph
```

---
## B. Convert bam files to 5' bigwig files (RPM normalized) ver. 1 

### New Folder
```sh
cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA
mkdir 5prime_rpm_bw
```

### New Script
```sh
# Write script
vi bam_to_5prime_rpm_bw_.py
```
- ##### bam_to_5prime_rpm_bw_.py

```sh
#!/usr/bin/env python3

"""
bam_to_5prime_rpm_bw_.py

Generate strand-specific 5'-end BigWig files with genomic-hit RPM normalization.

Normalization:
    weight = (1 / genomic_hits) * (1e6 / total_mapped_reads)

where

    total_mapped_reads = number of unique reads that mapped
    genomic_hits       = number of genomic alignments for each read

"""

import os
import argparse
import subprocess
from collections import defaultdict

import pysam


# ----------------------------------------------------------
# PASS 1
# Count genomic hits and unique mapped reads
# ----------------------------------------------------------

def first_pass(bam):

    mapping_hits = defaultdict(int)
    mapped_reads = set()
    total_alignments = 0

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:
            continue

        read_id = read.query_name

        mapping_hits[read_id] += 1
        mapped_reads.add(read_id)
        total_alignments += 1

    bam.reset()

    return mapping_hits, len(mapped_reads), total_alignments


# ----------------------------------------------------------
# PASS 2
# Build 5' signal
# ----------------------------------------------------------

def second_pass(bam, mapping_hits, total_mapped_reads):

    plus_signal = defaultdict(float)
    minus_signal = defaultdict(float)

    rpm_scale = 1e6 / total_mapped_reads

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:
            continue

        read_id = read.query_name

        hits = mapping_hits[read_id]

        weight = rpm_scale / hits

        chrom = read.reference_name

        if read.is_reverse:

            fiveprime = read.reference_end - 1

            minus_signal[(chrom, fiveprime)] += weight

        else:

            fiveprime = read.reference_start

            plus_signal[(chrom, fiveprime)] += weight

    bam.reset()

    return plus_signal, minus_signal


# ----------------------------------------------------------
# Write BedGraph
# ----------------------------------------------------------

def write_bedgraph(signal, outfile):

    with open(outfile, "w") as out:

        for chrom, pos in sorted(signal.keys()):

            value = signal[(chrom, pos)]

            out.write(
                f"{chrom}\t{pos}\t{pos+1}\t{value:.6f}\n"
            )


# ----------------------------------------------------------
# Sort BedGraph
# ----------------------------------------------------------

def sort_bedgraph(infile, outfile):

    with open(outfile, "w") as out:

        subprocess.run(

            [
                "sort",
                "-k1,1",
                "-k2,2n",
                infile
            ],

            stdout=out,
            check=True

        )


# ----------------------------------------------------------
# Convert BigWig
# ----------------------------------------------------------

def make_bigwig(bg, chromsizes, bw):

    subprocess.run(

        [
            "bedGraphToBigWig",
            bg,
            chromsizes,
            bw
        ],

        check=True

    )


# ----------------------------------------------------------
# Process one BAM
# ----------------------------------------------------------

def process_bam(bamfile, chromsizes, outdir):

    sample = os.path.basename(bamfile).replace(".bam", "")

    print("=" * 60)
    print(sample)

    bam = pysam.AlignmentFile(bamfile, "rb")

    print("First pass...")
    mapping_hits, mapped_reads, total_alignments = first_pass(bam)

    print(f"Unique mapped read IDs : {mapped_reads:,}")
    print(f"Total genomic alignments: {total_alignments:,}")
    
    average_hits = total_alignments / mapped_reads
    print(f"Average genomic hits/read: {average_hits:.2f}")
    
    max_hits = max(mapping_hits.values())
    print(f"Maximum genomic hits/read: {max_hits}")

    print("Second pass...")
    plus, minus = second_pass(
        bam,
        mapping_hits,
        mapped_reads
    )

    bam.close()

    plus_bg = os.path.join(
        outdir,
        sample + ".plus.bedgraph"
    )

    minus_bg = os.path.join(
        outdir,
        sample + ".minus.bedgraph"
    )

    plus_sorted = os.path.join(
        outdir,
        sample + ".plus.sorted.bedgraph"
    )

    minus_sorted = os.path.join(
        outdir,
        sample + ".minus.sorted.bedgraph"
    )

    plus_bw = os.path.join(
        outdir,
        sample + ".plus.5prime.rpm.bw"
    )

    minus_bw = os.path.join(
        outdir,
        sample + ".minus.5prime.rpm.bw"
    )

    print("Writing BedGraph...")

    write_bedgraph(plus, plus_bg)
    write_bedgraph(minus, minus_bg)

    print("Sorting BedGraph...")

    sort_bedgraph(plus_bg, plus_sorted)
    sort_bedgraph(minus_bg, minus_sorted)

    print("Creating BigWig...")

    make_bigwig(
        plus_sorted,
        chromsizes,
        plus_bw
    )

    make_bigwig(
        minus_sorted,
        chromsizes,
        minus_bw
    )

    os.remove(plus_bg)
    os.remove(minus_bg)
    os.remove(plus_sorted)
    os.remove(minus_sorted)

    print("Done.")


# ----------------------------------------------------------
# Main
# ----------------------------------------------------------

def main():

    parser = argparse.ArgumentParser()

    parser.add_argument(
        "--bam-dir",
        required=True,
        help="Directory containing BAM files"
    )

    parser.add_argument(
        "--chrom-sizes",
        required=True,
        help="Genome chrom.sizes file"
    )

    parser.add_argument(
        "--output-dir",
        required=True,
        help="Directory for BigWig output"
    )

    args = parser.parse_args()

    os.makedirs(args.output_dir, exist_ok=True)

    bamfiles = sorted(

        os.path.join(args.bam_dir, x)

        for x in os.listdir(args.bam_dir)

        if x.endswith(".bam")

    )

    print(f"Found {len(bamfiles)} BAM files.\n")

    for bam in bamfiles:

        process_bam(
            bam,
            args.chrom_sizes,
            args.output_dir
        )

    print("\nFinished successfully.")


if __name__ == "__main__":
    main()
```
- ### Run 
```sh
conda activate bamTobw_pipeline_env

nohup python bam_to_5prime_rpm_bw_.py \
    --bam-dir /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam \
    --chrom-sizes /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes \
    --output-dir /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw \
> bam_to_5prime_rpm_bw_.log 2>&1 & 
```

#### Issues:
- In the version 2 script, the result shows RPM abundance in the 5' end mapping tracks with smaller number compared to the RPM abundance generated in the 5' end phasing distribution. 
#### Reasons
- In bam file, I treated each read with 1 sequence abundace or redundancy (collapsed reads) instead of showing the original number of sequences in each read
#### Solution:

- In the conversion of bam file to bigwig file, consider the original number of redundancy, not the collapsed reads. 
---
### Version 2
## B. Convert bam files to 5' bigwig files (RPM normalized) ver. 2

```sh
cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw
vi bam_to_5prime_rpm_bw_.py 
```
### New Script
```sh
#!/usr/bin/env python3
"""
bam_to_5prime_rpm_bw.py

Generate strand-specific 5'-end BigWig files from sRNAminer
collapsed BAM files.

Normalization
-------------

Each alignment contributes:

    weight = (redundancy / genomic_hits) × (1e6 / total_mapped_redundancy)

where

    redundancy
        = abundance encoded in the collapsed read ID
          (Number-Redundancy)

    genomic_hits
        = number of genomic alignments of the collapsed read

    total_mapped_redundancy
        = sum of redundancies of all mapped collapsed reads

This reproduces abundance-weighted 5' signals while accounting
for multi-mapping reads.
"""

import os
import argparse
import subprocess
from collections import defaultdict

import pysam


# ----------------------------------------------------------
# Parse redundancy from sRNAminer read IDs
# ----------------------------------------------------------

def parse_redundancy(read_id):
    """
    Extract redundancy from read IDs produced by sRNAminer.

    Example
    -------
    4461435-18 -> 18
    """

    try:
        _, redundancy = read_id.rsplit("-", 1)
        return int(redundancy)

    except Exception:
        raise ValueError(
            f"Cannot parse redundancy from read ID: {read_id}"
        )


# ----------------------------------------------------------
# PASS 1
# Count genomic hits and total mapped redundancy
# ----------------------------------------------------------

def first_pass(bam):

    mapping_hits = defaultdict(int)

    redundancy = {}

    total_redundancy = 0

    total_alignments = 0

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:
            continue

        read_id = read.query_name

        mapping_hits[read_id] += 1

        total_alignments += 1

        if read_id not in redundancy:

            r = parse_redundancy(read_id)

            redundancy[read_id] = r

            total_redundancy += r

    bam.reset()

    return (
        mapping_hits,
        redundancy,
        total_redundancy,
        total_alignments
    )


# ----------------------------------------------------------
# PASS 2
# Build strand-specific 5' signal
# ----------------------------------------------------------

def second_pass(
    bam,
    mapping_hits,
    redundancy,
    total_redundancy
):

    plus_signal = defaultdict(float)

    minus_signal = defaultdict(float)

    rpm_scale = 1e6 / total_redundancy

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:
            continue

        read_id = read.query_name

        hits = mapping_hits[read_id]

        weight = (
            redundancy[read_id]
            * rpm_scale
            / hits
        )

        chrom = read.reference_name

        if read.is_reverse:

            fiveprime = read.reference_end - 1

            minus_signal[(chrom, fiveprime)] += weight

        else:

            fiveprime = read.reference_start

            plus_signal[(chrom, fiveprime)] += weight

    bam.reset()

    return plus_signal, minus_signal


# ----------------------------------------------------------
# Write BedGraph
# ----------------------------------------------------------

def write_bedgraph(signal, outfile):

    with open(outfile, "w") as out:

        for chrom, pos in sorted(signal.keys()):

            value = signal[(chrom, pos)]

            out.write(
                f"{chrom}\t{pos}\t{pos+1}\t{value:.6f}\n"
            )


# ----------------------------------------------------------
# Sort BedGraph
# ----------------------------------------------------------

def sort_bedgraph(infile, outfile):

    with open(outfile, "w") as out:

        subprocess.run(

            [
                "sort",
                "-k1,1",
                "-k2,2n",
                infile
            ],

            stdout=out,
            check=True

        )


# ----------------------------------------------------------
# Convert to BigWig
# ----------------------------------------------------------

def make_bigwig(bg, chromsizes, bw):

    subprocess.run(

        [
            "bedGraphToBigWig",
            bg,
            chromsizes,
            bw
        ],

        check=True

    )


# ----------------------------------------------------------
# Process one BAM
# ----------------------------------------------------------

def process_bam(
    bamfile,
    chromsizes,
    outdir
):

    sample = os.path.basename(bamfile).replace(".bam", "")

    print("=" * 60)

    print(sample)

    bam = pysam.AlignmentFile(
        bamfile,
        "rb"
    )

    print("First pass...")

    (
        mapping_hits,
        redundancy,
        total_red,
        total_aln

    ) = first_pass(bam)

    print(f"Unique collapsed reads : {len(redundancy):,}")
    print(f"Total mapped redundancy: {total_red:,}")
    print(f"Total alignments       : {total_aln:,}")
    print(f"Average hits/read      : {total_aln / len(redundancy):.2f}")
    print(f"Maximum hits/read      : {max(mapping_hits.values())}")
    print(f"RPM scale factor       : {1e6 / total_red:.8f}")

    print("Second pass...")

    plus, minus = second_pass(

        bam,

        mapping_hits,

        redundancy,

        total_red

    )

    bam.close()

    for signal, strand in [

        (plus, "plus"),

        (minus, "minus")

    ]:

        bg = os.path.join(

            outdir,

            f"{sample}.{strand}.bedgraph"

        )

        sorted_bg = os.path.join(

            outdir,

            f"{sample}.{strand}.sorted.bedgraph"

        )

        bw = os.path.join(

            outdir,

            f"{sample}.{strand}.5prime.rpm.bw"

        )

        print(f"Writing {strand} BedGraph...")

        write_bedgraph(signal, bg)

        print(f"Sorting {strand} BedGraph...")

        sort_bedgraph(bg, sorted_bg)

        print(f"Creating {strand} BigWig...")

        make_bigwig(

            sorted_bg,

            chromsizes,

            bw

        )

        os.remove(bg)

        os.remove(sorted_bg)

    print("Done.")


# ----------------------------------------------------------
# Main
# ----------------------------------------------------------

def main():

    parser = argparse.ArgumentParser()

    parser.add_argument(

        "--bam-dir",

        required=True,

        help="Directory containing BAM files"

    )

    parser.add_argument(

        "--chrom-sizes",

        required=True,

        help="Genome chrom.sizes file"

    )

    parser.add_argument(

        "--output-dir",

        required=True,

        help="Directory for output BigWig files"

    )

    args = parser.parse_args()

    os.makedirs(

        args.output_dir,

        exist_ok=True

    )

    bamfiles = sorted(

        os.path.join(args.bam_dir, f)

        for f in os.listdir(args.bam_dir)

        if f.endswith(".bam")

    )

    print(f"Found {len(bamfiles)} BAM files.\n")

    for bam in bamfiles:

        process_bam(

            bam,

            args.chrom_sizes,

            args.output_dir

        )

    print("\nFinished successfully.")


if __name__ == "__main__":

    main()
```

- ### Run 
```sh
conda activate bamTobw_pipeline_env

nohup python bam_to_5prime_rpm_bw_.py \
    --bam-dir /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam \
    --chrom-sizes /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes \
    --output-dir /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw \
> bam_to_5prime_rpm_bw_.log 2>&1 & 
```

---
## C. Integrate normalized bw tracks to PHASER IGV 
- #### Copy Files 

```sh
# Copy new generated bigwig files to `Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks`

cp -r /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw/ "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/5prime_rpm_sRNA_bw"

```
---
# Version 3


```sh
cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw
vi csv_to_5prime_rpm_bigwig.py
```
### New Script
```sh
#!/usr/bin/env python3
"""
folder_csv_to_5prime_rpm_bigwig.py

Generate strand-specific 5' end RPM BigWig files
from sRNAminer PHAS CSV files.

Input filenames:

    PHAS22-442_N012.csv
    PHAS22-443_N012.csv
    PHAS22-444_N012.csv

The sample ID is extracted from the filename:

    PHAS22-442_N012.csv
                ^
                |
              N012


Required CSV columns:

    chrom
    fivep
    strand
    hit-rpm-norm-counts


Normalization:

    Uses existing sRNAminer RPM values.
    No additional normalization is performed.


Output:

    N012.plus.5prime.rpm.bw
    N012.minus.5prime.rpm.bw

"""


import argparse
import os
import re
import tempfile
import subprocess

import pandas as pd



def parse_arguments():

    parser = argparse.ArgumentParser(
        description="Convert sRNAminer PHAS CSV files to sample-level 5' RPM BigWigs"
    )

    parser.add_argument(
        "--csv-dir",
        required=True,
        help="Directory containing PHAS CSV files"
    )

    parser.add_argument(
        "--chrom-sizes",
        required=True,
        help="Genome chromosome sizes file"
    )

    parser.add_argument(
        "--output-dir",
        required=True,
        help="Output BigWig directory"
    )

    parser.add_argument(
        "--bedgraphToBigWig",
        default="bedGraphToBigWig",
        help="Path to bedGraphToBigWig executable"
    )


    return parser.parse_args()



def extract_sample_id(filename):

    """
    Extract sample ID.

    Example:

        PHAS22-442_N012.csv

        returns:

        N012
    """

    match = re.search(
        r"_(N\d+)\.csv$",
        filename
    )

    if not match:

        raise ValueError(
            f"Cannot extract sample ID from {filename}"
        )

    return match.group(1)



def check_columns(df, filename):

    required = [
        "chrom",
        "fivep",
        "strand",
        "hit-rpm-norm-counts"
    ]


    missing = [
        c for c in required
        if c not in df.columns
    ]


    if missing:

        raise ValueError(
            f"{filename} missing columns: {missing}"
        )



def create_bedgraph(
        df,
        strand,
        output_file
):


    data = df[
        df["strand"] == strand
    ].copy()


    if data.empty:

        return False



    # Combine reads with identical 5' ends

    data = (
        data
        .groupby(
            [
                "chrom",
                "fivep"
            ],
            as_index=False
        )
        [
            "hit-rpm-norm-counts"
        ]
        .sum()
    )


    # Convert to BED 0-based coordinates

    data["start"] = (
        data["fivep"] - 1
    )

    data["end"] = (
        data["fivep"]
    )


    data = data[
        [
            "chrom",
            "start",
            "end",
            "hit-rpm-norm-counts"
        ]
    ]


    data = data.sort_values(
        [
            "chrom",
            "start"
        ]
    )


    data.to_csv(
        output_file,
        sep="\t",
        header=False,
        index=False,
        float_format="%.8f"
    )


    return True



def convert_bigwig(
        bedgraph,
        chrom_sizes,
        output,
        converter
):

    cmd = [
        converter,
        bedgraph,
        chrom_sizes,
        output
    ]


    print(
        "Running:",
        " ".join(cmd)
    )


    subprocess.run(
        cmd,
        check=True
    )



def main():

    args = parse_arguments()


    os.makedirs(
        args.output_dir,
        exist_ok=True
    )


    csv_files = [
        f for f in os.listdir(args.csv_dir)
        if f.endswith(".csv")
    ]


    if not csv_files:

        raise RuntimeError(
            "No CSV files found"
        )



    # Group CSV files by sample

    sample_files = {}


    for f in csv_files:

        sample = extract_sample_id(f)

        sample_files.setdefault(
            sample,
            []
        ).append(f)



    print(
        "Detected samples:"
    )


    for s, files in sample_files.items():

        print(
            s,
            len(files),
            "files"
        )



    with tempfile.TemporaryDirectory() as tmp:


        for sample, files in sample_files.items():


            print(
                "\nProcessing sample:",
                sample
            )


            sample_df = []



            for filename in files:


                path = os.path.join(
                    args.csv_dir,
                    filename
                )


                print(
                    "Reading:",
                    filename
                )


                df = pd.read_csv(
                    path
                )


                check_columns(
                    df,
                    filename
                )


                df["fivep"] = pd.to_numeric(
                    df["fivep"]
                )


                df["hit-rpm-norm-counts"] = pd.to_numeric(
                    df["hit-rpm-norm-counts"]
                )


                sample_df.append(
                    df[
                        [
                            "chrom",
                            "fivep",
                            "strand",
                            "hit-rpm-norm-counts"
                        ]
                    ]
                )



            # Merge all PHAS loci for this sample

            merged = pd.concat(
                sample_df,
                ignore_index=True
            )



            for strand, label in [
                ("+", "plus"),
                ("-", "minus")
            ]:


                bedgraph = os.path.join(
                    tmp,
                    f"{sample}.{label}.bedgraph"
                )


                created = create_bedgraph(
                    merged,
                    strand,
                    bedgraph
                )


                if not created:

                    print(
                        "No",
                        strand,
                        "strand reads"
                    )

                    continue



                output_bw = os.path.join(
                    args.output_dir,
                    f"{sample}.{label}.5prime.rpm.bw"
                )


                convert_bigwig(
                    bedgraph,
                    args.chrom_sizes,
                    output_bw,
                    args.bedgraphToBigWig
                )


    print(
        "\nFinished all samples!"
    )



if __name__ == "__main__":

    main()
```

- ### Run 
```sh
source ~/.bashrc
conda activate bamTobw_pipeline_env

nohup python csv_to_5prime_rpm_bigwig.py \
    --csv-dir /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv \
    --chrom-sizes /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes \
    --output-dir /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw \
> csv_to_5prime_rpm_bigwig.log 2>&1 &
```

- #### Copy Files 

```sh
# Copy new generated bigwig files to `Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks`

cp -r /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/5prime_rpm_bw/ "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/5prime_rpm_sRNA_bw"

``` 



















- #### Update Codes to integrate tracks to PHASER IGV

 Update the following files:

- track_service.py
- tracks.py
- config.py
- IGVViewer.tsx
---
## Previous Codes:
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
    BW_DIR,
    MRNA_BW_DIR,
    GFF3_FILE,
    GFF3_INDEX
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
            # 🔷 GENOME ANNOTATION
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "gff3",

                "url": (
                    f"/api/files/annotation/{GFF3_FILE.name}"
                ),

                "indexURL": (
                    f"/api/files/annotation/{GFF3_INDEX.name}"
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
    # 🔷 ALL IGV TRACKS
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
        # 🔷 GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "gff3",

            "url": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "indexURL": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            ),

            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "green"
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

        resolved = TRACKHUB_BASE / relative_path

        print(
            f"[FILE RESOLVE] "
            f"{relative_path} -> {resolved}"
        )

        return resolved


track_service = TrackService()
```

- #### tracks.py
`backend/app/routers/tracks.py`
```sh
"""Tracks API Router"""
from fastapi import APIRouter, HTTPException

from ..services.track_service import track_service
from ..models.phas import TrackListResponse, TrackFile

router = APIRouter(prefix="/tracks", tags=["Tracks"])


@router.get("", response_model=TrackListResponse)
async def get_all_tracks():
    tracks = track_service.get_all_tracks()
    return TrackListResponse(tracks=tracks)


@router.get("/genome")
async def get_genome_files():
    return track_service.get_genome_files()


# IGV ROUTES FIRST (VERY IMPORTANT)
@router.get("/igv/all")
async def get_all_igv_tracks():
    return track_service.get_all_igv_tracks()


@router.get("/igv/{sample}")
async def get_igv_config(sample: str):
    config = track_service.get_igv_config(sample)

    if not config:
        raise HTTPException(status_code=404, detail=f"Sample {sample} not found")

    return config


# GENERIC ROUTE LAST (ALWAYS LAST)
@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track
```

- #### config.py
`backend/app/config.py`
```sh
"""PHASER Configuration"""

from pathlib import Path

# =========================================================
#  BASE DIRECTORY
# =========================================================
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# =========================================================
#  DATA
# =========================================================
DATA_DIR = BASE_DIR / "data"

PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# REQUIRED
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# =========================================================
# TRACK HUB
# =========================================================
TRACKHUB_BASE = Path(
    "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"
)

# =========================================================
#  GENOME
# =========================================================
GENOME_FASTA = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta"
)

GENOME_FAI = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta.fai"
)

GENOME_2BIT = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.2bit"
)

GENOME_CHROMSIZES = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.genome.chrom.sizes"
)

# =========================================================
#  ANNOTATION
# =========================================================
ANNOTATION_DIR = TRACKHUB_BASE / "annotation"

# PHAS loci track
PHAS_ANNOTATION_BB = (
    ANNOTATION_DIR / "merged.PHAS22.bb"
)

# Genome annotation
GFF3_FILE = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz"
)

GFF3_INDEX = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz.tbi"
)

# =========================================================
#  sRNA BIGWIG DIRECTORY
# =========================================================
BW_DIR = TRACKHUB_BASE / "bw"

# =========================================================
#  mRNA BIGWIG DIRECTORY
# =========================================================
MRNA_BW_DIR = (
    TRACKHUB_BASE
    / "mRNA_bw"
    / "bw"
)

# =========================================================
#  API
# =========================================================
API_PREFIX = "/api"
```
- #### IGV.tsx 
`frontend/src/components/IGVViewer/IGV.tsx`
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
      rnaType: "BOTH",
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
          // 🔷 ANNOTATION TRACKS
          // =============================================
          const annotationTracks =
            allTracks.filter(
              (t) =>
                t.format ===
                  "gff3" ||
                t.format ===
                  "bigBed"
            );

          // =============================================
          // 🔷 SIGNAL TRACKS
          // =============================================
          let signalTracks =
            allTracks.filter(
              (t) =>
                t.format !==
                  "gff3" &&
                t.format !==
                  "bigBed"
            );

          signalTracks =
            signalTracks.filter(
              (t) => {

                // =======================================
                // 🔷 RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "BOTH" ||
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
                    t.rnaType ===
                    "sRNA"
                  ) {

                    return (
                      t.sample ===
                      sample
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
          <option value="BOTH">
            Both
          </option>

          <option value="sRNA">
            sRNAseq only
          </option>

          <option value="mRNA">
            mRNAseq only
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


