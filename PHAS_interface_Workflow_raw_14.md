##### PHAS_interface_Workflow_raw_14.md
- This version updates sRNA coverage tracks and adding 5' sRNA normalized tracks. 
#### Issues:
- In previous version (PHAS_interface_Workflow_raw_5):
    - There was misalignment on how the 5' end distribution plot was generated and how the sRNA track was generated
    - The previous IGV sRNA track was a coverage, not 5' end. 
    - In the previous approach, each read was treated as 1 count instead of using the original number of read counts. 

- #### Final Version is Version 2
    - RPM calculation in generating the 5' end plot should align the RPM calculation in the new 5' end and coverage sRNA tracks

# Update sRNA Tracks 
---
## Version 1
- Generate strand-specific coverage BigWigs directly from PHAS CSV files instead of bam files (same RPM values from the csv that was used in generating the 5' end plot). But the issue in using this approach is that, the flanking regions were not present which are important in the future analysis. 

```sh
cd /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/coverage_rpm_bw
vi csv_to_coverage_rpm_bigwig.py
```
- csv_to_coverage_rpm_bigwig.py

```sh
#!/usr/bin/env python3

"""
csv_to_coverage_rpm_bigwig.py

Generate strand-specific coverage BigWigs from sRNAminer PHAS CSV files.

Each read contributes its hit-rpm-norm-counts value to every genomic position
covered by the read.

Coverage is computed using a sweep-line (difference array) algorithm, which is
mathematically identical to base-by-base coverage but much faster.

Input:
    PHAS22-444_N137.csv

Output:
    PHAS22-444_N137.plus.coverage.rpm.bw
    PHAS22-444_N137.minus.coverage.rpm.bw
"""

import argparse
import os
import subprocess
import tempfile
from pathlib import Path
from collections import defaultdict

import pandas as pd


###############################################################################
# Command-line arguments
###############################################################################

def parse_arguments():

    parser = argparse.ArgumentParser(
        description="Generate coverage BigWigs from PHAS CSV files"
    )

    parser.add_argument(
        "--input-dir",
        required=True,
        help="Directory containing PHAS CSV files"
    )

    parser.add_argument(
        "--output-dir",
        required=True,
        help="Output directory"
    )

    parser.add_argument(
        "--chrom-sizes",
        required=True,
        help="Genome chromosome sizes"
    )

    return parser.parse_args()


###############################################################################
# Read CSV
###############################################################################

def read_csv(csv_file):

    df = pd.read_csv(csv_file)

    required_columns = [
        "chrom",
        "start",
        "end",
        "strand",
        "hit-rpm-norm-counts"
    ]

    for column in required_columns:

        if column not in df.columns:

            raise ValueError(
                f"{csv_file} is missing required column: {column}"
            )

    return df


###############################################################################
# Difference-array coverage
###############################################################################

def build_difference_arrays(df):

    plus = defaultdict(lambda: defaultdict(float))
    minus = defaultdict(lambda: defaultdict(float))

    for _, row in df.iterrows():

        chrom = row["chrom"]

        start = int(row["start"])

        end = int(row["end"])

        rpm = float(row["hit-rpm-norm-counts"])

        if rpm == 0:
            continue

        if row["strand"] == "+":

            plus[chrom][start] += rpm
            plus[chrom][end] -= rpm

        else:

            minus[chrom][start] += rpm
            minus[chrom][end] -= rpm

    return plus, minus

###############################################################################
# Convert difference arrays into coverage intervals
###############################################################################

def difference_to_intervals(diff):

    intervals = []

    for chrom in sorted(diff.keys()):

        events = diff[chrom]

        positions = sorted(events.keys())

        coverage = 0.0

        previous_position = None

        for position in positions:

            if previous_position is not None:

                if coverage != 0:

                    intervals.append(
                        (
                            chrom,
                            previous_position,
                            position,
                            coverage
                        )
                    )

            coverage += events[position]

            previous_position = position

    return intervals


###############################################################################
# Merge adjacent intervals with identical coverage
###############################################################################

def merge_intervals(intervals):

    if len(intervals) == 0:
        return []

    merged = []

    current = list(intervals[0])

    for interval in intervals[1:]:

        chrom, start, end, value = interval

        if (
            chrom == current[0]
            and start == current[2]
            and abs(value - current[3]) < 1e-12
        ):

            current[2] = end

        else:

            merged.append(tuple(current))
            current = list(interval)

    merged.append(tuple(current))

    return merged

###############################################################################
# Write bedGraph
###############################################################################

def write_bedgraph(intervals, outfile, negative=False):
    """
    Write coverage intervals to a bedGraph file.

    Parameters
    ----------
    intervals : list
        List of (chrom, start, end, coverage)

    outfile : str
        Output bedGraph path

    negative : bool
        If True, write negative coverage values
        (useful for minus strand display in IGV)
    """

    with open(outfile, "w") as out:

        for chrom, start, end, coverage in intervals:

            if coverage == 0:
                continue

            value = -coverage if negative else coverage

            out.write(
                f"{chrom}\t{start}\t{end}\t{value:.6f}\n"
            )


###############################################################################
# Convert bedGraph to BigWig
###############################################################################

def bedgraph_to_bigwig(
    bedgraph,
    chrom_sizes,
    output_bw
):
    """
    Convert a sorted bedGraph into BigWig.
    """

    subprocess.run(
        [
            "bedGraphToBigWig",
            bedgraph,
            chrom_sizes,
            output_bw,
        ],
        check=True,
    )


###############################################################################
# Sort bedGraph
###############################################################################

def sort_bedgraph(infile, outfile):
    """
    Sort a bedGraph file by chromosome and start coordinate.
    """

    with open(outfile, "w") as out:

        subprocess.run(
            [
                "sort",
                "-k1,1",
                "-k2,2n",
                infile,
            ],
            stdout=out,
            check=True,
        )


###############################################################################
# Updated process_csv()
###############################################################################

def process_csv(
    csv_file,
    output_dir,
    chrom_sizes,
):
    """
    Process one PHAS CSV.

    Generates:

        *.plus.coverage.rpm.bw
        *.minus.coverage.rpm.bw
    """

    name = Path(csv_file).stem

    print(f"[START] {name}")

    df = read_csv(csv_file)

    plus_diff, minus_diff = build_difference_arrays(df)

    plus_intervals = merge_intervals(
        difference_to_intervals(plus_diff)
    )

    minus_intervals = merge_intervals(
        difference_to_intervals(minus_diff)
    )

    with tempfile.TemporaryDirectory() as tmpdir:

        plus_bg = os.path.join(
            tmpdir,
            "plus.bedgraph",
        )

        minus_bg = os.path.join(
            tmpdir,
            "minus.bedgraph",
        )

        plus_sorted = os.path.join(
            tmpdir,
            "plus.sorted.bedgraph",
        )

        minus_sorted = os.path.join(
            tmpdir,
            "minus.sorted.bedgraph",
        )

        write_bedgraph(
            plus_intervals,
            plus_bg,
            negative=False,
        )

        write_bedgraph(
            minus_intervals,
            minus_bg,
            negative=True,
        )

        sort_bedgraph(
            plus_bg,
            plus_sorted,
        )

        sort_bedgraph(
            minus_bg,
            minus_sorted,
        )

        plus_bw = os.path.join(
            output_dir,
            f"{name}.plus.coverage.rpm.bw",
        )

        minus_bw = os.path.join(
            output_dir,
            f"{name}.minus.coverage.rpm.bw",
        )

        bedgraph_to_bigwig(
            plus_sorted,
            chrom_sizes,
            plus_bw,
        )

        bedgraph_to_bigwig(
            minus_sorted,
            chrom_sizes,
            minus_bw,
        )

    print(f"[DONE ] {name}")


###############################################################################
# Main
###############################################################################

def main():

    args = parse_arguments()

    input_dir = Path(args.input_dir)

    output_dir = Path(args.output_dir)

    output_dir.mkdir(
        parents=True,
        exist_ok=True,
    )

    csv_files = sorted(
        input_dir.glob("*.csv")
    )

    print()

    print(f"Found {len(csv_files)} CSV files")

    print()

    for i, csv_file in enumerate(csv_files, start=1):

        print(
            f"[{i}/{len(csv_files)}]"
        )

        process_csv(
            csv_file,
            output_dir,
            args.chrom_sizes,
        )

    print()

    print("Finished all files.")


if __name__ == "__main__":

    main()

```

- ### Run 
```sh
source ~/.bashrc
conda activate bamTobw_pipeline_env

nohup python csv_to_coverage_rpm_bigwig.py \
    --input-dir /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv \
    --chrom-sizes /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes \
    --output-dir /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/coverage_rpm_bw \
> csv_to_coverage_rpm_bigwig.log 2>&1 &
```

- #### Copy Files 

```sh
# Copy new generated bigwig files to `Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks`

cp -r /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/coverage_rpm_bw/ "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/bw"

``` 

# Version 2
- Use BAM files instead of csv files. 
- RPM calculation in generating the 5' end plot should align the RPM calculation in the new 5' end and coverage sRNA tracks

- RPM calculation:

                         read count
                         ----------
                             XM
                 RPM = ---------------- x 10^6
                     ∑ mapped_read_count​
Examples
    --------
    Read IDs 
    3306135-5  -> 5 
    4461435-18 -> 18

Where: 
    5 and 18 --> read counts
    XM --> number of genomic hits
    ∑ mapped_read_count​ --> total mapped reads to genome 

#### Check example of library with read counts
Example:
```sh
samtools view N137.trimmed.mc.fa.sorted.bam | tail

# Result
4461435-18	16	GWHAMMI00003890	53712	255	15M	*	0	0	TGGTAGAAAAGGTAA	IIIIIIIIIIIIIII	XA:i:0	MD:Z:15	NM:i:0	XM:i:9
12077880-2	16	GWHAMMI00003890	53712	255	16M	*	0	0	TGGTAGAAAAGGTAAT	IIIIIIIIIIIIIIII	XA:i:0	MD:Z:16	NM:i:0	XM:i:3
12006554-1	16	GWHAMMI00003890	54799	255	14M	*	0	0	ATAGTCCTCGAGAT	IIIIIIIIIIIIII	XA:i:0	MD:Z:14	NM:i:0	XM:i:4
10946924-1	0	GWHAMMI00003890	55588	255	22M	*	0	0	CATCGGCTGTGAAGCGTTGGCG	IIIIIIIIIIIIIIIIIIIIII	XA:i:1	MD:Z:15A6	NM:i:1	XM:i:20
3941865-1	16	GWHAMMI00003890	56470	255	17M	*	0	0	AGCTCCAAATGTGTCAA	IIIIIIIIIIIIIIIII	XA:i:0	MD:Z:17	NM:i:0	XM:i:1
3457014-1	16	GWHAMMI00003890	56561	255	14M	*	0	0	TGCCAAAGCCTAAA	IIIIIIIIIIIIII	XA:i:0	MD:Z:14	NM:i:0	XM:i:16
1839133-1	16	GWHAMMI00003890	57965	255	14M	*	0	0	AGAGCACTTCGGAG	IIIIIIIIIIIIII	XA:i:0	MD:Z:14	NM:i:0	XM:i:14
2701538-1	16	GWHAMMI00003890	60159	255	14M	*	0	0	GGATCAACCATGTA	IIIIIIIIIIIIII	XA:i:0	MD:Z:14	NM:i:0	XM:i:5
12224930-1	16	GWHAMMI00003890	60544	255	14M	*	0	0	AACTATGGGTCTCT	IIIIIIIIIIIIII	XA:i:0	MD:Z:14	NM:i:0	XM:i:11
3306135-5	0	GWHAMMI00003890	62544	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
```
- For example, `4461435-18` has read counts of 18, `3306135-5` has 5, or `12006554-1` has 1 .


#### Check example of genomic hits of a read
Example: `3306135-5`
```sh
(samtools_env) ma-discar@cc25dev0:/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam$ samtools view N137.trimmed.mc.fa.sorted.bam | grep "^3306135-5"
3306135-5	0	GWHAMMI00000001	126393348	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	16	GWHAMMI00000001	190044678	255	13M	*	0	0	GGAGAACCGTCTA	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000003	45233018	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000003	103956818	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000003	106219491	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	16	GWHAMMI00000004	207162286	255	13M	*	0	0	GGAGAACCGTCTA	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000005	98719396	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000007	150011009	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000007	150013628	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000008	93216285	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000008	143905304	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	16	GWHAMMI00000009	20178977	255	13M	*	0	0	GGAGAACCGTCTA	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000009	34434617	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000009	86902615	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00000009	121691629	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	16	GWHAMMI00000010	68714873	255	13M	*	0	0	GGAGAACCGTCTA	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00001036	649007	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
3306135-5	0	GWHAMMI00003890	62544	255	13M	*	0	0	TAGACGGTTCTCC	IIIIIIIIIIIII	XA:i:0	MD:Z:13	NM:i:0	XM:i:18
```
- In this exmple, `3306135-5` appeared 18 times. So it has hits in this particular library N137. Tag `XM` shows this number and this tag was used to represent the genomic hits in the formula. 
- Documentation: https://sources.debian.org/src/bowtie/1.3.1-1/MANUAL 


### Write Script
```sh

cd /work/ma-discar/PHAS/Hlongicornis/data/sRNA/bigwig_rpm
vi bam_to_bigwig_rpm.py
```

- ##### bam_to_bigwig_rpm.py
```sh
#!/usr/bin/env python3

import os
import glob
import argparse
import subprocess

from collections import defaultdict
from concurrent.futures import ProcessPoolExecutor

import pysam


# ============================================================
# Helpers
# ============================================================

def parse_read_id(read_id):
    """
    Extract collapsed abundance from read IDs.

    Examples
    --------
    3306135-5  -> 5
    4461435-18 -> 18
    """

    for sep in ("-", "_"):

        if sep in read_id:

            _, tail = read_id.rsplit(sep, 1)

            if tail.isdigit():

                return int(tail)

    return 1


def lib_id_from_filename(filename):
    """
    Extract library ID from BAM filename.
    """

    fn = os.path.basename(filename)

    suffixes = [

        ".fastq.trimmed.mc.fa.sorted.bam",
        ".trimmed.mc.fa.sorted.bam",
        ".fastq.trimmed.mc.fa.bam",
        ".trimmed.mc.fa.bam",
        ".bam"
    ]

    for suf in suffixes:

        if fn.endswith(suf):

            return fn[:-len(suf)]

    return os.path.splitext(fn)[0]


def get_genomic_hits(read):
    """
    In BAM files:

        unique read  -> XM:i:1
        9 mappings   -> XM:i:9

    Therefore:

        genomic_hits = XM
    """

    try:

        xm = read.get_tag("XM")

        return max(1, xm)

    except KeyError:

        return 1


# ============================================================
# Compute Mapped_Read_Count
# ============================================================

def compute_mapped_read_count(bam_file):
    """
    Compute:

        Mapped_Read_Count = sum(collapsed abundance)

    No division by XM.

    Example:

        4719706-6 contributes 6
    """

    total = 0

    bam = pysam.AlignmentFile(bam_file, "rb")

    seen = set()

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:

            continue

        read_id = read.query_name

        if read_id in seen:

            continue

        seen.add(read_id)

        total += parse_read_id(read_id)

    bam.close()

    return total


# ============================================================
# BedGraph helpers
# ============================================================

def collapse_positions(position_dict):

    intervals = []

    for chrom in sorted(position_dict):

        positions = sorted(position_dict[chrom])

        if not positions:

            continue

        start = positions[0]
        end = start + 1

        value = position_dict[chrom][start]

        for pos in positions[1:]:

            current = position_dict[chrom][pos]

            if pos == end and current == value:

                end += 1

            else:

                intervals.append(

                    (chrom, start, end, value)
                )

                start = pos
                end = pos + 1

                value = current

        intervals.append(

            (chrom, start, end, value)
        )

    return intervals


def write_bedgraph(signal, outfile):

    intervals = collapse_positions(signal)

    with open(outfile, "w") as out:

        for chrom, start, end, value in intervals:

            if value == 0:

                continue

            out.write(

                f"{chrom}\t"
                f"{start}\t"
                f"{end}\t"
                f"{value:.8f}\n"
            )


def bedgraph_to_bigwig(bedgraph, chrom_sizes, bigwig):

    subprocess.run(

        [

            "bedGraphToBigWig",
            bedgraph,
            chrom_sizes,
            bigwig

        ],

        check=True
    )


# ============================================================
# BAM processing
# ============================================================

def process_bam(job):

    bam_file, chrom_sizes, outdir = job

    sample = lib_id_from_filename(bam_file)

    print(f"\n[{sample}] Calculating Mapped_Read_Count ...")

    mapped_reads = compute_mapped_read_count(bam_file)

    print(

        f"[{sample}] "
        f"Mapped_Read_Count = {mapped_reads:,}"
    )

    rpm_factor = 1_000_000 / mapped_reads

    plus_5prime = defaultdict(lambda: defaultdict(float))
    minus_5prime = defaultdict(lambda: defaultdict(float))

    plus_cov = defaultdict(lambda: defaultdict(float))
    minus_cov = defaultdict(lambda: defaultdict(float))

    bam = pysam.AlignmentFile(

        bam_file,
        "rb"
    )

    processed = 0

    for read in bam.fetch(until_eof=True):

        if read.is_unmapped:

            continue

        abundance = parse_read_id(

            read.query_name
        )

        genomic_hits = get_genomic_hits(

            read
        )

        rpm = (

            abundance /
            genomic_hits
        ) * rpm_factor

        chrom = bam.get_reference_name(

            read.reference_id
        )

        start = read.reference_start
        end = read.reference_end

        # --------------------------------
        # Strand-specific signal
        # --------------------------------

        if read.is_reverse:

            five_prime = end - 1

            signal_5p = minus_5prime
            signal_cov = minus_cov

        else:

            five_prime = start

            signal_5p = plus_5prime
            signal_cov = plus_cov

        # --------------------------------
        # 5′ signal
        # --------------------------------

        signal_5p[chrom][five_prime] += rpm

        # --------------------------------
        # Coverage signal
        # --------------------------------

        for pos in range(start, end):

            signal_cov[chrom][pos] += rpm

        processed += 1

        if processed % 5_000_000 == 0:

            print(

                f"[{sample}] "
                f"{processed:,} alignments processed"
            )

    bam.close()

    outputs = [

        (

            plus_5prime,
            f"{sample}.plus.5prime.rpm"

        ),

        (

            minus_5prime,
            f"{sample}.minus.5prime.rpm"

        ),

        (

            plus_cov,
            f"{sample}.plus.coverage.rpm"

        ),

        (

            minus_cov,
            f"{sample}.minus.coverage.rpm"

        )
    ]

    for signal, prefix in outputs:

        bedgraph = os.path.join(

            outdir,
            prefix + ".bedgraph"
        )

        bigwig = os.path.join(

            outdir,
            prefix + ".bw"
        )

        print(

            f"[{sample}] Writing "
            f"{os.path.basename(bigwig)}"
        )

        write_bedgraph(

            signal,
            bedgraph
        )

        bedgraph_to_bigwig(

            bedgraph,
            chrom_sizes,
            bigwig
        )

        os.remove(

            bedgraph
        )

    print(

        f"[{sample}] Done."
    )


# ============================================================
# Main
# ============================================================

def main():

    parser = argparse.ArgumentParser(

        description=(
            "Generate strand-specific RPM-normalized "
            "5′-end and coverage BigWigs "
            "from collapsed Bowtie1 BAM files."
        )
    )

    parser.add_argument(

        "-i",
        "--input",
        required=True,

        help=(
            "Directory containing BAM files"
        )
    )

    parser.add_argument(

        "-c",
        "--chrom-sizes",
        required=True,

        help=(
            "Genome chrom.sizes file"
        )
    )

    parser.add_argument(

        "-o",
        "--output",
        required=True,

        help=(
            "Output directory"
        )
    )

    parser.add_argument(

        "--cores",

        type=int,

        default=1,

        help=(
            "Number of parallel workers"
        )
    )

    args = parser.parse_args()

    os.makedirs(

        args.output,

        exist_ok=True
    )

    bam_files = sorted(

        glob.glob(

            os.path.join(
                args.input,
                "*.bam"
            )
        )
    )

    if not bam_files:

        raise RuntimeError(

            f"No BAM files found:\n"
            f"{args.input}"
        )

    jobs = [

        (

            bam,
            args.chrom_sizes,
            args.output

        )

        for bam in bam_files
    ]

    with ProcessPoolExecutor(

        max_workers=args.cores

    ) as executor:

        executor.map(

            process_bam,
            jobs
        )


if __name__ == "__main__":

    main()
```

- ## Run `bam_to_bigwig_rpm.py`
```sh
# Install pysam
conda activate bedgraphtobigwig_env
conda install -c bioconda ucsc-bedgraphtobigwig
conda install -c bioconda pysam


 # Run 
nohup python /work/ma-discar/PHAS/Hlongicornis/data/sRNA/bigwig_rpm/bam_to_bigwig_rpm.py \
    -i /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam \
    -c /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes \
    -o /work/ma-discar/PHAS/Hlongicornis/data/sRNA/bigwig_rpm \
    --cores 2 \
    > bam_to_bigwig_rpm.log 2>&1 &
```
- ## Copy bigwig files output
```sh
cp -r /work/ma-discar/PHAS/Hlongicornis/data/sRNA/bigwig_rpm /project/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA 

cp -r /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/sRNA/bigwig_rpm "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw"

# Move files as follows:

mkdir -p "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/5prime_rpm_sRNA_bw"

mkdir -p "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/coverage_rpm_sRNA_bw"

mv *.5prime.rpm.bw \
"/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/5prime_rpm_sRNA_bw/"

mv *.coverage.rpm.bw \
"/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/sRNA_bw/bigwig_rpm/coverage_rpm_sRNA_bw/"
```

---
## View in IGV 
Update the following backend and frontend files:
- config.py
- track_service.py
- IGVViewer.tsx

## config.py
- replace "5prime_rpm_sRNA_bw" (original directory) with the new directories / "sRNA_bw" and / "bigwig_rpm"

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
# RPM BIGWIG ROOT
# =========================================================
RPM_BW_DIR = (
    TRACKHUB_BASE
    / "sRNA_bw"
    / "bigwig_rpm"
)

# =========================================================
# 5' END RPM sRNA BIGWIG DIRECTORY
# =========================================================
FIVEPRIME_BW_DIR = (
    RPM_BW_DIR
    / "5prime_rpm_sRNA_bw"
)

# =========================================================
# COVERAGE RPM sRNA BIGWIG DIRECTORY
# =========================================================
COVERAGE_BW_DIR = (
    RPM_BW_DIR
    / "coverage_rpm_sRNA_bw"
)

# =========================================================
#  API
# =========================================================
API_PREFIX = "/api"

```
## track_service.py
- Add:
```sh
from ..config import (
    ...
    FIVEPRIME_BW_DIR,
    COVERAGE_BW_DIR,
)
```
- Modify sRNA TRACKS

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
    FIVEPRIME_BW_DIR,
    COVERAGE_BW_DIR,
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
# STRAIN COLORS
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
    #  LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    #  ALL TRACKS
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
    #  TRACK BY ID
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
    #  GENOME FILES
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
    # FIND BIGWIG FILES
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
    #  SINGLE SAMPLE IGV CONFIG
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
            #  GENOME ANNOTATION
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
            #  PHAS LOCI
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
            #  PLUS STRAND
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
            # MINUS STRAND
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
    # ALL IGV TRACKS
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
        # GENOME ANNOTATION
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
        #  PHAS LOCI
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
        #  sRNA TRACKS
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
        #  mRNA TRACKS
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
        #  5' END RPM sRNA TRACKS
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
        #  COVERAGE RPM sRNA TRACKS
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
    #  FILE RESOLUTION
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

## IGVViewer.tsx
- replace `rnaType:BOTH` with `rnaType:ALL` 
- Update the RNA filter logic
- Add the new coverage option

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
  //  SAMPLE → MATCHING MRNA LABELS
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
  //  SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  //  LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current)
      return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        // ===============================================
        //  CLEANUP OLD BROWSER
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
        //  FETCH TRACK CONFIG
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
  //  APPLY TRACKS
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
                t.format ===
                  "gff3" ||
                t.format ===
                  "bigBed"
            );

          // =============================================
          //  SIGNAL TRACKS
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
                //  RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "ALL" ||
                  t.rnaType ===
                    filters.rnaType;

                if (!rnaMatch)
                  return false;

                // =======================================
                //  SHOW BEST SAMPLE MODE
                // =======================================
                if (
                  filters.samples
                    .length === 1
                ) {

                  // ===================================
                  //  SRNA
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
                  //  MRNA
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
                        //  EXACT MATCH LOGIC
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
                //  SHOW ALL SAMPLES MODE
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
          //  SORT TRACKS
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
          //  LOAD TRACKS
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
  //  LOCUS UPDATE
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
  //  TOGGLE SAMPLES
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
- ##### Restart PHASER and check alignment betwen 5' end distribution plot and 5' sRNA tracks
```sh
bash start.sh
```

