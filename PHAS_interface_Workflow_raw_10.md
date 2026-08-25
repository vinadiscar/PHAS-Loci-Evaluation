# Add size distribution plot and an sRNA abundance line chart
> A. Add size distribution plot
> B. Add sRNA abundance line chart
> C. Import new charts to PHASER

```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

## A. Size Distribution Plot
### 1. Write Script
```sh
cd /project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sizeDistribution_script_log

vi size_distribution_plots.py
```
#### Previous Code:  

- size_distribution_plots_v2.py 

```sh 
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path
import re

# =====================================================
# INPUT / OUTPUT DIRECTORIES
# =====================================================

input_dir = Path(
    "/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv"
)

output_dir = Path(
    "/project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sizeDistribution_plots_v2"
)

output_dir.mkdir(parents=True, exist_ok=True)

# =====================================================
# SAMPLE NAME MAPPING
# =====================================================

replacement_mapping = {
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
    "N138": "Oita_Adultfemale_fed"
}

# =====================================================
# COLORS
# =====================================================

PLUS_COLOR = "#B77A80"    # dusty pink
MINUS_COLOR = "#23408E"   # dark blue

# =====================================================
# PROCESS FILES
# =====================================================

csv_files = sorted(input_dir.glob("*.csv"))

print(f"Found {len(csv_files)} files")

for csv_file in csv_files:

    print(f"Processing {csv_file.name}")

    try:

        df = pd.read_csv(csv_file)

        if df.empty:
            continue

        locus_id = df["locusID"].iloc[0]

        sample_match = re.search(
            r"(N\d+)",
            csv_file.stem
        )

        if sample_match:
            sample_id = sample_match.group(1)
            sample_name = replacement_mapping.get(
                sample_id,
                sample_id
            )
        else:
            sample_id = "Unknown"
            sample_name = "Unknown"

        # =============================================
        # Aggregate abundance by length and strand
        # =============================================

        agg = (
            df.groupby(
                ["length", "strand"]
            )["hit-rpm-norm-counts"]
            .sum()
            .reset_index()
        )

        lengths = list(range(18, 31))

        plus = []
        minus = []

        for L in lengths:

            p = agg[
                (agg["length"] == L)
                & (agg["strand"] == "+")
            ]

            m = agg[
                (agg["length"] == L)
                & (agg["strand"] == "-")
            ]

            plus.append(
                p["hit-rpm-norm-counts"].sum()
            )

            minus.append(
                -m["hit-rpm-norm-counts"].sum()
            )

        # =============================================
        # Symmetric Y-axis
        # =============================================

        max_abundance = max(
            max(plus),
            abs(min(minus))
        )

        ylim = max_abundance * 1.10

        # =============================================
        # Plot
        # =============================================

        plt.figure(
            figsize=(7, 4.5)
        )

        minus_bars = plt.bar(
            lengths,
            minus,
            color=MINUS_COLOR,
            width=0.8,
            label="- strand"
        )

        plus_bars = plt.bar(
            lengths,
            plus,
            color=PLUS_COLOR,
            width=0.8,
            label="+ strand"
        )

        plt.axhline(
            0,
            color="black",
            linewidth=1
        )

        plt.ylim(
            -ylim,
            ylim
        )

        plt.xlim(
            17.5,
            30.5
        )

        plt.xticks(
            lengths,
            fontsize=11
        )

        plt.yticks(
            fontsize=11
        )

        plt.xlabel(
            "sRNA Length (nt)",
            fontsize=14
        )

        plt.ylabel(
            "Total hit-normalized RPM",
            fontsize=14
        )

        plt.title(
            f"{locus_id}\n{sample_name}",
            fontsize=18,
            fontweight="bold",
            pad=12
        )

        # Cleaner publication style
        ax = plt.gca()

        ax.spines["top"].set_visible(False)
        ax.spines["right"].set_visible(False)

        # Legend order:
        # - strand first
        # + strand second
        plt.legend(
            handles=[
                minus_bars,
                plus_bars
            ],
            labels=[
                "- strand",
                "+ strand"
            ],
            frameon=False,
            fontsize=12,
            loc="upper right"
        )

        plt.tight_layout()

        # =============================================
        # Save PDF
        # =============================================

        output_pdf = (
            output_dir
            /
            f"{csv_file.stem}_{sample_name}_sizeDistribution.pdf"
        )

        plt.savefig(
            output_pdf,
            format="pdf",
            bbox_inches="tight"
        )

        plt.close()

        print(
            f"Saved: {output_pdf.name}"
        )

    except Exception as e:

        print(
            f"ERROR in {csv_file.name}: {e}"
        )

print("Finished.")
```
### Updated Code:
```sh
cd /project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sizeDistribution_script_log
vi size_distribution_plots.py
```
- ### size_distribution_plots.py
```sh
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path
import re

# =====================================================
# INPUT / OUTPUT DIRECTORIES
# =====================================================

input_dir = Path(
    "/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv"
)

output_dir = Path(
    "/project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sizeDistribution_plots"
)

output_dir.mkdir(parents=True, exist_ok=True)

# =====================================================
# SAMPLE NAME MAPPING
# =====================================================

replacement_mapping = {
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
    "N138": "Oita_Adultfemale_fed"
}

# =====================================================
# COLORS
# =====================================================

PLUS_COLOR = "#B77A80"    # dusty pink
MINUS_COLOR = "#23408E"   # dark blue

# =====================================================
# PROCESS FILES
# =====================================================

csv_files = sorted(input_dir.glob("*.csv"))

print(f"Found {len(csv_files)} files")

for csv_file in csv_files:

    print(f"Processing {csv_file.name}")

    try:

        df = pd.read_csv(csv_file)

        if df.empty:
            continue

        locus_id = df["locusID"].iloc[0]

        sample_match = re.search(
            r"(N\d+)",
            csv_file.stem
        )

        if sample_match:
            sample_id = sample_match.group(1)
            sample_name = replacement_mapping.get(
                sample_id,
                sample_id
            )
        else:
            sample_name = "Unknown"

        # =============================================
        # Aggregate abundance by length and strand
        # =============================================

        agg = (
            df.groupby(
                ["length", "strand"]
            )["hit-rpm-norm-counts"]
            .sum()
            .reset_index()
        )

        lengths = list(range(18, 31))

        plus = []
        minus = []

        for L in lengths:

            p = agg[
                (agg["length"] == L)
                & (agg["strand"] == "+")
            ]

            m = agg[
                (agg["length"] == L)
                & (agg["strand"] == "-")
            ]

            plus.append(
                p["hit-rpm-norm-counts"].sum()
            )

            minus.append(
                -m["hit-rpm-norm-counts"].sum()
            )

        # =============================================
        # Symmetric Y-axis with extra headroom
        # =============================================

        max_abundance = max(
            max(plus),
            abs(min(minus))
        )

        ylim = max_abundance * 1.25

        # =============================================
        # Plot
        # =============================================

        fig, ax = plt.subplots(
            figsize=(7, 5)
        )

        minus_bars = ax.bar(
            lengths,
            minus,
            color=MINUS_COLOR,
            width=0.8,
            label="- strand"
        )

        plus_bars = ax.bar(
            lengths,
            plus,
            color=PLUS_COLOR,
            width=0.8,
            label="+ strand"
        )

        ax.axhline(
            0,
            color="black",
            linewidth=1
        )

        ax.set_ylim(
            -ylim,
            ylim
        )

        ax.set_xlim(
            17.5,
            30.5
        )

        ax.set_xticks(
            lengths
        )

        ax.tick_params(
            axis="both",
            labelsize=11
        )

        ax.set_xlabel(
            "sRNA Length (nt)",
            fontsize=14
        )

        ax.set_ylabel(
            "Total hit-normalized RPM",
            fontsize=14
        )

        ax.set_title(
            f"{locus_id}\n{sample_name}",
            fontsize=14,
            fontweight="bold",
            pad=10
        )

        # =============================================
        # Full box around plot
        # =============================================

        for spine in ax.spines.values():
            spine.set_visible(True)
            spine.set_linewidth(1.2)

        # =============================================
        # Legend inside plot
        # =============================================

        ax.legend(
            handles=[
                minus_bars,
                plus_bars
            ],
            labels=[
                "- strand",
                "+ strand"
            ],
            loc="upper right",
            bbox_to_anchor=(0.98, 0.98),
            fontsize=11,
            frameon=True,
            facecolor="white",
            edgecolor="black",
            framealpha=1
        )

        plt.tight_layout()

        # =============================================
        # Save PNG
        # =============================================

        output_png = (
            output_dir
            /
            f"{csv_file.stem}_{sample_name}_sizeDistribution.png"
        )

        plt.savefig(
            output_png,
            dpi=300,
            bbox_inches="tight"
        )

        plt.close()

        print(
            f"Saved: {output_png.name}"
        )

    except Exception as e:

        print(
            f"ERROR in {csv_file.name}: {e}"
        )

print("Finished.")
```
- #### Run `size_distribution_plots.py`
```sh
nohup python size_distribution_plots.py > size_distribution.log 2>&1 &
```
---
# B. Abundance Line Chart
```sh
cd /project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sRNAabundance_LineChart/logs_script

vi generate_sRNA_abundance_plots.py
```
- ### generate_sRNA_abundance_plots.py
```sh
#!/usr/bin/env python3

import os
import re
import glob
import pandas as pd
import matplotlib.pyplot as plt

# =============================================================================
# Input/output directories
# =============================================================================

input_dir = "/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/csv"

output_dir = "/project/okamura-lab/Vina/PHAS/PHASloci_info_csv/sRNAabundance_LineChart"

os.makedirs(output_dir, exist_ok=True)

# =============================================================================
# Sample name mapping
# =============================================================================

replacement_mapping = {
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
    "N138": "Oita_Adultfemale_fed"
}

# =============================================================================
# Stage order
# =============================================================================

stage_order = [
    "OkayamaE1",
    "OkayamaE5",
    "OkayamaE10",
    "OkayamaE15",
    "OkayamaE20",
    "Okayama_Larva_Unfed",
    "Okayama_Larva_fed",
    "Okayama_Nymph_Unfed",
    "Okayama_Nymph_fed",
    "Okayama_Adult_Unfed",
    "Okayama_Adult_Fed",

    "OitaE1",
    "OitaE5",
    "OitaE10",
    "OitaE15",
    "OitaE20",
    "Oita_Larva_Unfed",
    "Oita_Larva_fed",
    "Oita_Nymph_Unfed",
    "Oita_Nymph_fed",
    "Oita_Adultmale_Unfed",
    "Oita_Adultmale_fed",
    "Oita_Adultfemale_Unfed",
    "Oita_Adultfemale_fed"
]

# =============================================================================
# Read all CSV files and calculate summed abundance
# =============================================================================

results = []

csv_files = glob.glob(os.path.join(input_dir, "*.csv"))

print(f"Found {len(csv_files)} CSV files.")

for csv_file in csv_files:

    filename = os.path.basename(csv_file)

    match = re.match(r"(PHAS\d+-\d+)_(N\d+)\.csv", filename)

    if match is None:
        print(f"Skipping unexpected filename: {filename}")
        continue

    locus = match.group(1)
    sample_code = match.group(2)

    if sample_code not in replacement_mapping:
        print(f"Skipping unknown sample: {sample_code}")
        continue

    sample_name = replacement_mapping[sample_code]

    try:
        df = pd.read_csv(csv_file)

        # Convert abundance column to numeric
        df["hit-rpm-norm-counts"] = pd.to_numeric(
            df["hit-rpm-norm-counts"],
            errors="coerce"
        )

        # Sum abundance across all sRNAs in the locus
        abundance = df["hit-rpm-norm-counts"].sum()

        results.append({
            "locus": locus,
            "sample": sample_name,
            "abundance": abundance
        })

    except Exception as e:
        print(f"Error processing {filename}: {e}")

# =============================================================================
# Create summary dataframe
# =============================================================================

summary_df = pd.DataFrame(results)

summary_csv = os.path.join(
    output_dir,
    "PHAS_abundance_summary.csv"
)

summary_df.to_csv(summary_csv, index=False)

print(f"Saved summary table: {summary_csv}")

# =============================================================================
# Generate one plot per PHAS locus
# =============================================================================

print("Generating plots...")

for locus, group in summary_df.groupby("locus"):

    plot_df = pd.DataFrame({
        "sample": stage_order
    })

    plot_df = plot_df.merge(
        group,
        on="sample",
        how="left"
    )

    # Fill missing samples with 0 abundance
    plot_df["abundance"] = plot_df["abundance"].fillna(0)

    # Create figure
    plt.figure(figsize=(12, 5))

    plt.plot(
        plot_df["sample"],
        plot_df["abundance"],
        marker="o",
        markersize=6,
        markerfacecolor="white",
        linewidth=2
    )

    plt.xlabel("Developmental stage", fontsize=12)
    plt.ylabel("Summed RPM-normalized abundance", fontsize=12)
    plt.title(locus, fontsize=14)

    plt.xticks(
        rotation=90,
        fontsize=10
    )

    plt.yticks(fontsize=10)

    # Add horizontal grid lines
    plt.grid(
        axis="y",
        linestyle="--",
        alpha=0.4
    )

    # Improve layout
    plt.tight_layout()

    # Save figure
    output_file = os.path.join(
        output_dir,
        f"{locus}_abundance.png"
    )

    plt.savefig(
        output_file,
        dpi=300,
        bbox_inches="tight"
    )

    plt.close()

print(f"Finished generating {summary_df['locus'].nunique()} abundance plots.")
print(f"Plots saved to: {output_dir}")

```
- ### Run
```sh
nohup python generate_sRNA_abundance_plots.py \
> abundance_plot.log 2>&1 &
```
---
# Import new charts to PHASER

## 1. Prepare/Organize Images
- ### A. Copy size distribution .png files from project disk to PHASER folder

```sh
cd frontend/public/phas_plots/organize_phas_images
vi copy_sizeDistribution.sh
```
- copy_sizeDistribution.sh
```sh
#!/bin/bash

SOURCE="/Volumes/okamura-lab/Vina/PHAS/PHASloci_info_csv/sizeDistribution_plots"
DEST="/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/phas_plots/organize_phas_images"

mkdir -p "$DEST"

for file in "$SOURCE"/*.png; do
    filename=$(basename "$file")

    # Extract PHAS ID (everything before the first underscore)
    phas_id="${filename%%_*}"

    # Create destination folder if needed
    mkdir -p "$DEST/$phas_id"

    # Copy the PNG
    cp "$file" "$DEST/$phas_id/"

    echo "Copied: $filename → $phas_id/"
done

echo "Finished!"
```

- ### Run
```sh
chmod +x copy_sizeDistribution.sh
nohup ./copy_sizeDistribution.sh > copy_sizeDistribution.log 2>&1 &
```

- ### B. Copy sRNA abundance .png files from project disk to PHASER folder

```sh
cd frontend/public/phas_plots/organize_phas_images
vi copy_sRNAAbundance.sh
```
- copy_sRNAAbundance.sh
```sh
#!/bin/bash

SOURCE="/Volumes/okamura-lab/Vina/PHAS/PHASloci_info_csv/sRNAabundance_LineChart"
DEST="/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/phas_plots/organize_phas_images"

mkdir -p "$DEST"

for file in "$SOURCE"/*.png; do
    filename=$(basename "$file")

    # Extract PHAS ID (everything before the first underscore)
    phas_id="${filename%%_*}"

    # Create destination folder if needed
    mkdir -p "$DEST/$phas_id"

    # Copy the PNG
    cp "$file" "$DEST/$phas_id/"

    echo "Copied: $filename → $phas_id/"
done

echo "Finished!"
```

- ### Run
```sh
chmod +x copy_sRNAAbundance.sh
nohup ./copy_sRNAAbundance.sh > copy_sRNAAbundance.log 2>&1 &
```
---

## 2. Import sRNA abunace to PHASER

### 2.1 Create new folders

```sh
src/components/
└── sRNAAbundancePanel/
    ├── sRNAAbundancePanel.tsx
    └── sRNAAbundancePanel.module.css
```
### sRNAAbundancePanel.tsx
```sh
import styles from "./sRNAAbundancePanel.module.css";

interface Props {
  locus: any;
}

export default function sRNAAbundancePanel({
  locus,
}: Props) {
  // No PHAS locus selected
  if (!locus) {
    return (
      <div className={styles.panel}>
        <div className={styles.header}>
          <h3>sRNA Abundance</h3>
        </div>

        <div className={styles.empty}>
          Select a PHAS locus
        </div>
      </div>
    );
  }

  // Construct abundance image path
  const imagePath =
    `/phas_plots/organize_phas_images/${locus.phas_id}/${locus.phas_id}_abundance.png`;

  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>sRNA Abundance</h3>

        <span className={styles.subtitle}>
          {locus.phas_id}
        </span>
      </div>

      <div className={styles.content}>
        <img
          src={imagePath}
          alt={`${locus.phas_id} abundance`}
          className={styles.image}
          onError={(e) => {
            e.currentTarget.style.display = "none";
          }}
        />

        <div className={styles.caption}>
          Developmental stage abundance profile
        </div>
      </div>
    </div>
  );
}
```
### sRNAAbundancePanel.module.css

```sh
.panel {
  background: white;
  border: 1px solid #e3e6ea;
  border-radius: 10px;
  overflow: hidden;
  margin-top: 16px;
}

.header {
  padding: 14px 18px;
  border-bottom: 1px solid #e3e6ea;
  background: #f8fafc;

  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header h3 {
  margin: 0;
  font-size: 18px;
  color: #2c3e50;
}

.subtitle {
  color: #6c7a89;
  font-size: 13px;
  font-weight: 500;
}

.content {
  padding: 20px;
  text-align: center;
}

.image {
  width: 100%;
  max-width: 1000px;
  height: auto;

  border: 1px solid #ecf0f1;
  border-radius: 8px;

  background: white;
}

.caption {
  margin-top: 12px;
  color: #7f8c8d;
  font-size: 13px;
}

.empty {
  padding: 50px 20px;
  text-align: center;
  color: #7f8c8d;
  font-size: 15px;
}
```
### Modify `App.tsx`

#### Previous Code: App.tsx
```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);
  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery
      .trim()
      .toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(
          data[selectedPhas.phas_id] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match =
        phas.best_sample.match(/N\d+/);

      if (match)
        derivedSample = match[0];

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage)
        derivedStage = bestStage;
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // =============================
  // MEMOIZED ENRICHED RECORDS
  // =============================
  const enrichedRecords = useMemo(() => {
    return filteredRecords.map((r) => ({
      ...r,
      __evaluation:
        evaluations[r.phas_id],
    }));
  }, [filteredRecords, evaluations]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={!!selectedSample}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) => r.phas_id
          )}
          onSelect={(phasId) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id === phasId
              );

            if (phas) {
              setSearchQuery(phasId);

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">
              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={setView}
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={
                  evaluations[
                    selectedPhas?.phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (prev) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```
### Update Code: `App.tsx`
- Add the import:
```sh
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
```
- Add:
```sh
<SRNAAbundancePanel
  locus={selectedPhas}
/>
```
## Full Updated `App.tsx`
```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);

  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] =
    useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery
      .trim()
      .toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(
          data[selectedPhas.phas_id] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match =
        phas.best_sample.match(/N\d+/);

      if (match) {
        derivedSample = match[0];
      }

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // =============================
  // MEMOIZED ENRICHED RECORDS
  // =============================
  const enrichedRecords = useMemo(() => {
    return filteredRecords.map((r) => ({
      ...r,
      __evaluation:
        evaluations[r.phas_id],
    }));
  }, [filteredRecords, evaluations]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={!!selectedSample}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) => r.phas_id
          )}
          onSelect={(phasId) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id === phasId
              );

            if (phas) {
              setSearchQuery(phasId);

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">

              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={setView}
              />

              <SRNAAbundancePanel
                locus={selectedPhas}
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={
                  evaluations[
                    selectedPhas.phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (prev) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```
---

## 3. Import Size Distribution to PHASER

## 3.1. Generate .json mapping file
- for naming and easy identification of files
- generate_sizeDistributionMap_json.sh
    > This will generate .json file: `sizeDistributionMap.json`
```sh
cd frontend/public/phas_plot/organize_phas_images
vi generate_sizeDistributionMap_json.sh
```

```sh
#!/bin/bash

BASE_DIR="."
OUTPUT="sizeDistributionMap.json"

echo "{" > "$OUTPUT"

first_phas=true

for phas_dir in "$BASE_DIR"/PHAS22-*; do

    [ -d "$phas_dir" ] || continue

    phas=$(basename "$phas_dir")

    if [ "$first_phas" = true ]; then
        first_phas=false
    else
        echo "," >> "$OUTPUT"
    fi

    echo "  \"$phas\": {" >> "$OUTPUT"

    first_stage=true

    for file in "$phas_dir"/*_sizeDistribution.png; do

        [ -f "$file" ] || continue

        filename=$(basename "$file")

        #
        # Remove:
        #
        # PHAS22-98_
        #
        tmp=${filename#${phas}_}

        #
        # Remove:
        #
        # N010_
        #
        tmp=${tmp#N*_}

        #
        # Remove:
        #
        # _sizeDistribution.png
        #
        stage=${tmp%_sizeDistribution.png}

        if [ "$first_stage" = true ]; then
            first_stage=false
        else
            echo "," >> "$OUTPUT"
        fi

        echo -n "    \"$stage\": \"$filename\"" >> "$OUTPUT"

    done

    echo "" >> "$OUTPUT"
    echo -n "  }" >> "$OUTPUT"

done

echo "" >> "$OUTPUT"
echo "}" >> "$OUTPUT"

echo "Generated:"
echo "$OUTPUT"
```
- ### Run:
```sh
chmod +x generate_sizeDistributionMap_json.sh
./generate_sizeDistributionMap_json.sh

#Check output
head -100 sizeDistributionMap.json
```

## 3.2 Create New Component Folders
```sh
frontend/src/components/SizeDistributionPanel/
├── SizeDistributionPanel.tsx
└── SizeDistributionPanel.module.css
```
- ## SizeDistributionPanel.tsx
```sh
import { useEffect, useState } from "react";
import styles from "./SizeDistributionPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
}

export default function SizeDistributionPanel({
  phasId,
  stage,
}: Props) {
  const [sizeMap, setSizeMap] =
    useState<Record<string, any>>({});

  useEffect(() => {
    fetch(
      "/phas_plots/organize_phas_images/sizeDistributionMap.json"
    )
      .then((res) => res.json())
      .then((data) => setSizeMap(data))
      .catch((err) =>
        console.error(
          "Failed to load sizeDistributionMap:",
          err
        )
      );
  }, []);

  const imageFile =
    stage !== "ALL"
      ? sizeMap[phasId]?.[stage]
      : null;

  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>
          sRNA Size Distribution
        </h3>
      </div>

      {stage === "ALL" ? (
        <div className={styles.empty}>
          Select a sample to view the
          size distribution plot.
        </div>
      ) : imageFile ? (
        <div className={styles.content}>
          <div className={styles.stage}>
            {stage.replaceAll("_", " ")}
          </div>

          <img
            src={`/phas_plots/organize_phas_images/${phasId}/${imageFile}`}
            alt={`${phasId} ${stage}`}
            className={styles.image}
          />
        </div>
      ) : (
        <div className={styles.empty}>
          No size distribution plot
          available for this sample.
        </div>
      )}
    </div>
  );
}
```
- ## SizeDistributionPanel.module.css

```sh
.panel {
  background: white;
  border: 1px solid #e3e6ea;
  border-radius: 10px;
  margin-top: 16px;
  overflow: hidden;
}

.header {
  padding: 12px 16px;
  border-bottom: 1px solid #e3e6ea;
  background: #f8fafc;
}

.header h3 {
  margin: 0;
  font-size: 16px;
  color: #2c3e50;
}

.content {
  padding: 16px;
}

.stage {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 12px;
  color: #34495e;
}

.image {
  width: 100%;
  border-radius: 8px;
  border: 1px solid #dfe6ee;
}

.empty {
  padding: 24px;
  text-align: center;
  color: #7f8c8d;
}
```

- ## Previous App.tsx
```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);

  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] =
    useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery
      .trim()
      .toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(
          data[selectedPhas.phas_id] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match =
        phas.best_sample.match(/N\d+/);

      if (match) {
        derivedSample = match[0];
      }

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // =============================
  // MEMOIZED ENRICHED RECORDS
  // =============================
  const enrichedRecords = useMemo(() => {
    return filteredRecords.map((r) => ({
      ...r,
      __evaluation:
        evaluations[r.phas_id],
    }));
  }, [filteredRecords, evaluations]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={!!selectedSample}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) => r.phas_id
          )}
          onSelect={(phasId) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id === phasId
              );

            if (phas) {
              setSearchQuery(phasId);

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">

              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={setView}
              />

              <SRNAAbundancePanel
                locus={selectedPhas}
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={
                  evaluations[
                    selectedPhas.phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (prev) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```

## Update App.tsx
- Add:
```sh
import SizeDistributionPanel from
  "./components/SizeDistributionPanel/SizeDistributionPanel";
```

- Place in the lay-out (since the abundance panel is already working, we place the size distribution panel immediately below it)

```sh
<PhasingPatternPanel
  phasId={selectedPhas.phas_id}
  stage={activeStage}
  view={view}
  allStages={availableStages}
  setStage={setActiveStage}
  setView={setView}
/>

<sRNAAbundancePanel
  phasId={selectedPhas.phas_id}
/>

<SizeDistributionPanel
  phasId={selectedPhas.phas_id}
  stage={activeStage}
/>

{selectedSample && (
  <IGVViewer
    ...
  />
)}
```
- Pass the appropriate props (phasId and activeStage).
## Updated App.tsx 
```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import SizeDistributionPanel from "./components/SizeDistributionPanel/SizeDistributionPanel"; // NEW
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);

  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] =
    useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery
      .trim()
      .toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(
          data[selectedPhas.phas_id] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match =
        phas.best_sample.match(/N\d+/);

      if (match) {
        derivedSample = match[0];
      }

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // =============================
  // MEMOIZED ENRICHED RECORDS
  // =============================
  const enrichedRecords = useMemo(() => {
    return filteredRecords.map((r) => ({
      ...r,
      __evaluation:
        evaluations[r.phas_id],
    }));
  }, [filteredRecords, evaluations]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={!!selectedSample}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) => r.phas_id
          )}
          onSelect={(phasId) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id === phasId
              );

            if (phas) {
              setSearchQuery(phasId);

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">

              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={setView}
              />

              <SRNAAbundancePanel
                locus={selectedPhas}
              />

              <SizeDistributionPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={
                  evaluations[
                    selectedPhas.phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (prev) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```
#### It worked, but there are some issues:
- the image is too big for the interface (not proportion with the previous panels)
- the size distribution figure is after the abundance figures. It should be before the abundance figure.
- When the dropdown menu "Show All" was selected, size distribution figures did not show, instead it says "Select a sample to view the size distribution plot". It should be: When "Show All" is clicked, all size distribution figures should also show (same as when the register and 5' distribution figures are all shown). 

#### Solve:
#### S1. Fix .css file to reduce the size
- change to:
```sh
.image {
  width: 100%;
  max-width: 600px;
  display: block;
  margin: 0 auto;
  border-radius: 8px;
  border: 1px solid #dfe6ee;
}
```
## Updated `SizeDistributionPanel.module.css`

```sh
.panel {
  background: white;
  border: 1px solid #e3e6ea;
  border-radius: 10px;
  margin-top: 16px;
  overflow: hidden;
}

.header {
  padding: 12px 16px;
  border-bottom: 1px solid #e3e6ea;
  background: #f8fafc;
}

.header h3 {
  margin: 0;
  font-size: 16px;
  color: #2c3e50;
}

.content {
  padding: 16px;
}

.stage {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 12px;
  color: #34495e;
}

.image {
  width: 100%;
  max-width: 600px;
  display: block;
  margin: 0 auto;
  border-radius: 8px;
  border: 1px solid #dfe6ee;
}

.empty {
  padding: 24px;
  text-align: center;
  color: #7f8c8d;
}
```
#### S2. Size Distribution should come before abundance

- Change this line in `App.tsx`:
```sh
<PhasingPatternPanel ... />

<SizeDistributionPanel
    phasId={selectedPhas.phas_id}
    stage={activeStage}
/>

<SRNAAbundancePanel
    locus={selectedPhas}
/>

<IGVViewer ... />
```
- ## Updated `App.tsx`
```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import SizeDistributionPanel from "./components/SizeDistributionPanel/SizeDistributionPanel"; // NEW
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);

  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] =
    useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery
      .trim()
      .toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(
          data[selectedPhas.phas_id] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match =
        phas.best_sample.match(/N\d+/);

      if (match) {
        derivedSample = match[0];
      }

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // =============================
  // MEMOIZED ENRICHED RECORDS
  // =============================
  const enrichedRecords = useMemo(() => {
    return filteredRecords.map((r) => ({
      ...r,
      __evaluation:
        evaluations[r.phas_id],
    }));
  }, [filteredRecords, evaluations]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={!!selectedSample}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) => r.phas_id
          )}
          onSelect={(phasId) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id === phasId
              );

            if (phas) {
              setSearchQuery(phasId);

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">

              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={setView}
              />

              <SizeDistributionPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={activeStage}
              />

              <SRNAAbundancePanel
                locus={selectedPhas}
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={
                  evaluations[
                    selectedPhas.phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (prev) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```

#### S3. Add toggle for all samples of size distribution figures
-  show the selected sample's size distribution when a stage is chosen.
- When All Samples is selected: (1) default to Best sample only, and 
(2) allow the user to switch to Show all size distributions
## Update App.tsx
replace:
```sh
<SizeDistributionPanel
  phasId={
    selectedPhas.phas_id
  }
  stage={activeStage}
/>
```
with this:
```sh
<SizeDistributionPanel
  phasId={selectedPhas.phas_id}
  stage={activeStage}
  allStages={availableStages}
/>
```

- ## Updated Code: `App.tsx`

```sh
import { useState, useEffect, useMemo } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import SizeDistributionPanel from "./components/SizeDistributionPanel/SizeDistributionPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);

  return { chrom, start, end };
}

export default function App(): JSX.Element {
  const [phasRecords, setPhasRecords] =
    useState<PHASLocus[]>([]);

  const [filteredRecords, setFilteredRecords] =
    useState<PHASLocus[]>([]);

  const [selectedPhas, setSelectedPhas] =
    useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] =
    useState("");

  const [loading, setLoading] =
    useState(false);

  const [availableStages, setAvailableStages] =
    useState<string[]>([]);

  const [activeStage, setActiveStage] =
    useState("ALL");

  const [view, setView] =
    useState<
      "read5prime" |
      "register" |
      "all" |
      "hide"
    >("all");

  const [selectedSample, setSelectedSample] =
    useState<string | null>(null);

  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  const [hydrated, setHydrated] =
    useState(false);

  // =============================
  // LOAD PHAS DATA
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);

        setFilteredRecords(
          data.records
        );

        const defaultPhas =
          data.records.find(
            (r) =>
              r.phas_id ===
              "PHAS22-1"
          );

        if (defaultPhas) {
          handleSelectPhasAndStage(
            defaultPhas
          );
        }
      })
      .finally(() =>
        setLoading(false)
      );
  }, []);

  // =============================
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch(
      "http://163.221.246.151:8001/api/evaluations"
    )
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});

        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});

        setHydrated(true);
      });
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q =
      searchQuery
        .trim()
        .toLowerCase();

    if (!q) {
      setFilteredRecords(
        phasRecords
      );

      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id
          .toLowerCase()
          .includes(q)
      )
    );
  }, [
    searchQuery,
    phasRecords,
  ]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch(
      "/phas_plots/organize_phas_images/stages.json"
    )
      .then((res) =>
        res.json()
      )
      .then((data) => {
        setAvailableStages(
          data[
            selectedPhas.phas_id
          ] || []
        );
      });
  }, [selectedPhas]);

  // =============================
  // SELECT HANDLER
  // =============================
  const handleSelectPhasAndStage =
    (
      phas: PHASLocus,
      stage: string | null =
        null,
      sample?: string | null
    ) => {
      setSelectedPhas(phas);

      let derivedSample =
        sample;

      let derivedStage =
        stage;

      if (phas.best_sample) {
        const match =
          phas.best_sample.match(
            /N\d+/
          );

        if (match) {
          derivedSample =
            match[0];
        }

        const bestStage =
          getStageFromSample(
            phas.best_sample
          );

        if (bestStage) {
          derivedStage =
            bestStage;
        }
      }

      setSelectedSample(
        derivedSample || null
      );

      setActiveStage(
        derivedStage || "ALL"
      );

      setView("all");
    };

  // =============================
  // ENRICH RECORDS
  // =============================
  const enrichedRecords =
    useMemo(() => {
      return filteredRecords.map(
        (r) => ({
          ...r,
          __evaluation:
            evaluations[
              r.phas_id
            ],
        })
      );
    }, [
      filteredRecords,
      evaluations,
    ]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={
          !!selectedSample
        }
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={
            setSearchQuery
          }
          placeholder="Search PHAS ID..."
          options={phasRecords.map(
            (r) =>
              r.phas_id
          )}
          onSelect={(
            phasId
          ) => {
            const phas =
              phasRecords.find(
                (r) =>
                  r.phas_id ===
                  phasId
              );

            if (phas) {
              setSearchQuery(
                phasId
              );

              handleSelectPhasAndStage(
                phas
              );
            }
          }}
        />

        {hydrated && (
          <PHASTable
            records={
              enrichedRecords
            }
            selectedId={
              selectedPhas?.phas_id ||
              null
            }
            onSelectPhasAndStage={
              handleSelectPhasAndStage
            }
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="responsiveLayout">
            <div className="leftPanel">
              <PhasingPatternPanel
                phasId={
                  selectedPhas.phas_id
                }
                stage={
                  activeStage
                }
                view={view}
                allStages={
                  availableStages
                }
                setStage={
                  setActiveStage
                }
                setView={
                  setView
                }
              />

              <SizeDistributionPanel
                phasId={
                  selectedPhas.phas_id
                }
                defaultStage={
                  selectedPhas.best_sample
                    ? getStageFromSample(
                        selectedPhas.best_sample
                      ) ||
                      "ALL"
                    : "ALL"
                }
                allStages={
                  availableStages
                }
              />

              <SRNAAbundancePanel
                locus={
                  selectedPhas
                }
              />

              {selectedSample && (
                <IGVViewer
                  key={`${selectedSample}-${selectedPhas.phas_id}`}
                  sample={
                    selectedSample
                  }
                  locus={
                    selectedPhas.best_region
                      ? parseRegion(
                          selectedPhas.best_region
                        )
                      : {
                          chrom:
                            selectedPhas.chromosome,
                          start:
                            selectedPhas.start,
                          end:
                            selectedPhas.end,
                        }
                  }
                />
              )}
            </div>

            <div className="rightPanel">
              <ConfidencePanel
                locus={
                  selectedPhas
                }
                evaluation={
                  evaluations[
                    selectedPhas
                      .phas_id
                  ]
                }
                onSave={(
                  locusId,
                  data
                ) => {
                  setEvaluations(
                    (
                      prev
                    ) => ({
                      ...prev,
                      [locusId]:
                        data,
                    })
                  );
                }}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
```
## Update `SizeDistributionPanel.tsx`

```sh
import { useEffect, useState } from "react";
import styles from "./SizeDistributionPanel.module.css";

interface Props {
  phasId: string;
  defaultStage: string;
  allStages: string[];
}

export default function SizeDistributionPanel({
  phasId,
  defaultStage,
  allStages,
}: Props) {
  const [sizeMap, setSizeMap] =
    useState<Record<string, any>>({});

  const [showAll, setShowAll] =
    useState(false);

  // ==========================
  // LOAD SIZE DISTRIBUTION MAP
  // ==========================
  useEffect(() => {
    fetch(
      "/phas_plots/organize_phas_images/sizeDistributionMap.json"
    )
      .then((res) => res.json())
      .then((data) => setSizeMap(data))
      .catch((err) =>
        console.error(
          "Failed to load sizeDistributionMap:",
          err
        )
      );
  }, []);

  // ==========================
  // RESET TO DEFAULT
  // WHEN PHAS LOCUS CHANGES
  // ==========================
  useEffect(() => {
    setShowAll(false);
  }, [phasId]);

  // ==========================
  // RENDER A SINGLE IMAGE
  // ==========================
  const renderImage = (
    currentStage: string
  ) => {
    const imageFile =
      sizeMap[phasId]?.[currentStage];

    if (!imageFile) return null;

    return (
      <div
        key={currentStage}
        className={styles.figureBox}
      >
        <div className={styles.stage}>
          {currentStage.replaceAll(
            "_",
            " "
          )}
        </div>

        <img
          src={`/phas_plots/organize_phas_images/${phasId}/${imageFile}`}
          alt={`${phasId} ${currentStage}`}
          className={styles.image}
        />
      </div>
    );
  };

  // ==========================
  // DEFAULT VIEW
  // ==========================
  const defaultImage =
    renderImage(defaultStage);

  // ==========================
  // COMPONENT
  // ==========================
  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>
          sRNA Size Distribution
        </h3>

        <div
          className={
            styles.headerActions
          }
        >
          <button
            className={
              styles.toggleBtn
            }
            onClick={() =>
              setShowAll(
                (prev) => !prev
              )
            }
          >
            {showAll
              ? "Show Best Sample"
              : "Show All"}
          </button>
        </div>
      </div>

      <div className={styles.content}>
        {showAll ? (
          allStages
            .map(renderImage)
            .filter(Boolean)
        ) : defaultImage ? (
          defaultImage
        ) : (
          <div
            className={
              styles.empty
            }
          >
            No size distribution
            plot available for
            the best sample.
          </div>
        )}
      </div>
    </div>
  );
}
```
- ## Update `SizeDistributionPanel.module.css`
```sh
.panel {
  background: white;
  border: 1px solid #e3e6ea;
  border-radius: 10px;
  margin-top: 16px;
  overflow: hidden;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;

  padding: 20px 24px;

  border-bottom: 1px solid #e3e6ea;
}

.header h3 {
  margin: 0;
  font-size: 16px;
  color: #2c3e50;
}

.content {
  padding: 16px;
}

.stage {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 12px;
  color: #34495e;
}

.image {
  width: 90%;
  max-width: 500px;
  display: block;
  margin: 0 auto;
  border-radius: 8px;
  border: 1px solid #dfe6ee;
}

.empty {
  padding: 24px;
  text-align: center;
  color: #7f8c8d;
}

.figureBox {
  margin-bottom: 24px;
}

.toggleBtn {
  border: 1px solid #dfe6ee;
  background: white;
  border-radius: 6px;
  padding: 6px 12px;
  cursor: pointer;
  font-size: 13px;
}

.toggleBtn:hover {
  background: #f5f7fa;
}

.headerActions {
  display: flex;
  gap: 10px;
}
```
---

# Modifications
## Update PhasingPatternPanel
- Change: `Best Sample → Show All → Hide All` to `Best Sample → Show All → Show Best Sample`

# Previous Code: `PhasingPatternPanel.tsx`
```sh
import { useState, useEffect } from "react";
import { Camera, ZoomIn } from "lucide-react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

/* =========================
   🔥 HELPERS
========================= */

const getStrain = (stage: string) => stage.split(/_|(?=[A-Z])/)[0];

const getLifeStage = (stage: string) => {
  const s = stage.toLowerCase();
  if (s.match(/e\d+/)) return "Embryo";
  if (s.includes("larva")) return "Larva";
  if (s.includes("nymph")) return "Nymph";
  if (s.includes("adult")) return "Adult";
  return "Other";
};

const getFeeding = (stage: string) => {
  const s = stage.toLowerCase();
  if (s.includes("unfed")) return "Unfed";
  if (s.includes("fed")) return "Fed";
  return "Unknown";
};

const getStageOrder = (stage: string) => {
  const s = stage.toLowerCase();
  const embryoMatch = s.match(/e(\d+)/);
  if (embryoMatch) return 100 + parseInt(embryoMatch[1]);
  if (s.includes("larva")) return 200;
  if (s.includes("nymph")) return 300;
  if (s.includes("adultfemale")) return 400;
  if (s.includes("adultmale")) return 410;
  if (s.includes("adult")) return 420;
  return 999;
};

/* =========================
   🔥 COMPONENT
========================= */

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {
  const [hiddenImages, setHiddenImages] = useState<string[]>([]);
  const [lifeStage, setLifeStage] = useState("All");
  const [strain, setStrain] = useState("All");
  const [feeding, setFeeding] = useState("All");

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  // 🔥 RESET HIDDEN IMAGES whenever PHAS ID or stage changes
  useEffect(() => {
    setHiddenImages([]);
    if (view === "hide") setView("all");
  }, [phasId, stage]);

  const filteredStages = allStages.filter((s) => {
    return (
      (lifeStage === "All" || getLifeStage(s) === lifeStage) &&
      (strain === "All" || getStrain(s) === strain) &&
      (feeding === "All" || getFeeding(s) === feeding)
    );
  });

  const groupedStages = Object.entries(
    filteredStages.reduce((acc: Record<string, string[]>, s) => {
      const loc = getStrain(s);
      if (!acc[loc]) acc[loc] = [];
      acc[loc].push(s);
      return acc;
    }, {})
  ).map(([location, stages]) => ({
    location,
    stages: stages.sort((a, b) => getStageOrder(a) - getStageOrder(b)),
  }));

  /* =========================
     🔥 ACTIONS
  ========================== */

  const downloadImage = (key: string) => {
    const link = document.createElement("a");
    link.href = `${base}/${key}.png`;
    link.download = `${key}.png`;
    link.click();
  };

  const zoomImage = (key: string) => {
    window.open(`${base}/${key}.png`, "_blank");
  };

  const toggleImage = (key: string) => {
    setHiddenImages((prev) =>
      prev.includes(key)
        ? prev.filter((k) => k !== key)
        : [...prev, key]
    );
  };

  const renderFigure = (key: string, label: string) => {
    const isHidden = hiddenImages.includes(key);

    return (
      <div className={styles.figureBox} key={key}>
        <div className={styles.figureHeader}>
          <span>{label}</span>

          <div className={styles.headerActions}>
            <div className={styles.iconWrapper} data-tooltip="Download as PNG">
              <Camera
                size={16}
                onClick={() => downloadImage(key)}
                className={styles.toolbarIcon}
              />
            </div>

            <div className={styles.iconWrapper} data-tooltip="Zoom image">
              <ZoomIn
                size={16}
                onClick={() => zoomImage(key)}
                className={styles.toolbarIcon}
              />
            </div>

            <button
              className={styles.hideBtn}
              onClick={() => toggleImage(key)}
            >
              {isHidden ? "Show" : "Hide"}
            </button>
          </div>
        </div>

        {!isHidden && (
          <img
            src={`${base}/${key}.png`}
            className={styles.image}
            alt={key}
          />
        )}
      </div>
    );
  };

  const renderPair = (s: string) => {
    const readKey = `${s}_read5prime`;
    const regKey = `${s}_register`;

    return (
      <div className={styles.row} key={s}>
        {(view === "all" || view === "read5prime") &&
          renderFigure(readKey, "Read 5′")}
        {(view === "all" || view === "register") &&
          renderFigure(regKey, "Register")}
      </div>
    );
  };

  return (
    <div className={styles.container}>
      <div className={styles.topBar}>
        {/* 🔥 UPDATED TITLE BLOCK */}
        <div>
          <h2>{phasId}</h2>
          {stage !== "ALL" && (
            <div className={styles.defaultLabel}>
              Default: Best Sample
            </div>
          )}
        </div>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          <option value="ALL">All Samples</option>
          {groupedStages.map(({ location, stages }) => (
            <optgroup key={location} label={location}>
              {stages.map((s) => (
                <option key={s} value={s}>
                  {s.replaceAll("_", " ")}
                </option>
              ))}
            </optgroup>
          ))}
        </select>

        <div className={styles.filters}>
          <div className={styles.filterBox}>
            <label>Life stage</label>
            <select value={lifeStage} onChange={(e) => setLifeStage(e.target.value)}>
              <option>All</option>
              <option>Embryo</option>
              <option>Larva</option>
              <option>Nymph</option>
              <option>Adult</option>
            </select>
          </div>

          <div className={styles.filterBox}>
            <label>Strain</label>
            <select value={strain} onChange={(e) => setStrain(e.target.value)}>
              <option>All</option>
              <option>Oita</option>
              <option>Okayama</option>
            </select>
          </div>

          <div className={styles.filterBox}>
            <label>Feeding status</label>
            <select value={feeding} onChange={(e) => setFeeding(e.target.value)}>
              <option>All</option>
              <option>Fed</option>
              <option>Unfed</option>
            </select>
          </div>
        </div>

        <div className={styles.analysisToggle}>
          <span
            className={view === "read5prime" ? styles.activeTab : ""}
            onClick={() => setView("read5prime")}
          >
            Read 5′
          </span>
          <span
            className={view === "register" ? styles.activeTab : ""}
            onClick={() => setView("register")}
          >
            Register
          </span>
        </div>

        <div className={styles.topActions}>
          <span
            onClick={() => {
              setStage("ALL");
              setView("all");
              setHiddenImages([]);
            }}
          >
            Show All
          </span>

          <span
            onClick={() => {
              setView("hide");
              setHiddenImages(
                filteredStages.flatMap((s) => [
                  `${s}_read5prime`,
                  `${s}_register`,
                ])
              );
            }}
          >
            Hide All
          </span>
        </div>
      </div>

      <div className={styles.grid}>
        {view !== "hide" &&
          (stage === "ALL"
            ? groupedStages.map(({ location, stages }) => (
                <div key={location}>
                  <h3 className={styles.groupHeader}>{location}</h3>
                  {stages.map((s) => (
                    <div key={s}>
                      <h4>{s.replaceAll("_", " ")}</h4>
                      {renderPair(s)}
                    </div>
                  ))}
                </div>
              ))
            : renderPair(stage))}
      </div>
    </div>
  );
}
```

# Previous Code: `PhasingPatternPanel.module.css`
```sh
.container {
  padding: 8px;
}

/* Top Bar */
.topBar {
  background: #2f3b45;
  color: white;
  padding: 8px 10px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  gap: 12px;
}

/* 🔥 NEW: Title wrapper (PHAS ID + label) */
.topBar > div:first-child {
  display: flex;
  flex-direction: column;
  margin-right: auto; /* keeps everything aligned */
}

/* Title */
.topBar h2 {
  font-size: 16px;
  margin: 0;
}

/* 🔥 NEW: Default label */
.defaultLabel {
  font-size: 11px;
  color: #cbd5db;
  margin-top: 2px;
  opacity: 0.9;
}

/* Dropdown */
.topBar select {
  font-size: 12px;
}

/* Actions */
.topActions span {
  font-size: 11px;
  margin-left: 8px;
  cursor: pointer;
}

/* Grid */
.grid {
  background: white;
  padding: 12px;
  margin-top: 8px;
  border-radius: 8px;
}

.row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 15px;
  margin-bottom: 15px;
}

.figureBox {
  width: 100%;
  max-width: 500px;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 6px;
  position: relative;
}

/* Figure Header */
.figureHeader {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
  font-size: 12px;
}

.headerActions {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* Icon Wrapper with tooltip */
.iconWrapper {
  position: relative;
  display: inline-block;
  cursor: pointer;
}

.iconWrapper::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 125%;
  left: 50%;
  transform: translateX(-50%);
  background: #333;
  color: white;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.2s ease;
  z-index: 10;
}

.iconWrapper:hover::after {
  opacity: 1;
}

.toolbarIcon {
  color: #555;
  padding: 2px;
  border-radius: 4px;
  transition: color 0.2s;
}

.toolbarIcon:hover {
  color: #000;
}

.image {
  width: 100%;
}

/* Buttons */
.hideBtn {
  background: #6c757d;
  color: white;
  border: none;
  padding: 3px 8px;
  font-size: 11px;
  border-radius: 4px;
  cursor: pointer;
}

.showBtn {
  background: #28a745;
  color: white;
  border: none;
  padding: 3px 8px;
  font-size: 11px;
  border-radius: 4px;
  cursor: pointer;
}

/* Analysis Toggle */
.analysisToggle {
  display: flex;
  gap: 8px;
  margin-left: 10px;
}

.analysisToggle span {
  font-size: 12px;
  padding: 4px 8px;
  border-radius: 6px;
  cursor: pointer;
  background: #444;
  color: white;
}

.analysisToggle span:hover {
  background: #666;
}

.activeTab {
  background: #007bff !important;
}

/* Group Header */
.groupHeader {
  margin-top: 20px;
  margin-bottom: 10px;
  font-size: 18px;
  font-weight: bold;
  color: #2f3b45;
  border-bottom: 2px solid #ddd;
  padding-bottom: 4px;
}

/* Filters */
.filters {
  display: flex;
  gap: 10px;
  align-items: center;
}

.filterBox {
  display: flex;
  flex-direction: column;
  font-size: 11px;
  color: white;
}

.filterBox select {
  font-size: 12px;
  padding: 2px 6px;
  border-radius: 4px;
}
```
# Updated Code: `PhasingPatternPanel.tsx`
```sh
import { useState, useEffect } from "react";
import { Camera, ZoomIn } from "lucide-react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

/* =========================
   🔥 HELPERS
========================= */

const getStrain = (stage: string) => stage.split(/_|(?=[A-Z])/)[0];

const getLifeStage = (stage: string) => {
  const s = stage.toLowerCase();
  if (s.match(/e\d+/)) return "Embryo";
  if (s.includes("larva")) return "Larva";
  if (s.includes("nymph")) return "Nymph";
  if (s.includes("adult")) return "Adult";
  return "Other";
};

const getFeeding = (stage: string) => {
  const s = stage.toLowerCase();
  if (s.includes("unfed")) return "Unfed";
  if (s.includes("fed")) return "Fed";
  return "Unknown";
};

const getStageOrder = (stage: string) => {
  const s = stage.toLowerCase();
  const embryoMatch = s.match(/e(\d+)/);
  if (embryoMatch) return 100 + parseInt(embryoMatch[1]);
  if (s.includes("larva")) return 200;
  if (s.includes("nymph")) return 300;
  if (s.includes("adultfemale")) return 400;
  if (s.includes("adultmale")) return 410;
  if (s.includes("adult")) return 420;
  return 999;
};

/* =========================
   🔥 COMPONENT
========================= */

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {
  const [hiddenImages, setHiddenImages] = useState<string[]>([]);
  const [lifeStage, setLifeStage] = useState("All");
  const [strain, setStrain] = useState("All");
  const [feeding, setFeeding] = useState("All");
  const [bestStage, setBestStage] = useState(stage);

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  // 🔥 RESET HIDDEN IMAGES whenever PHAS ID or stage changes
  useEffect(() => {
    setHiddenImages([]);
    
    if (view === "hide") {
      setView("all");
    }

    if (stage !== "ALL") {
      setBestStage(stage);
    }
  }, [phasId, stage]);  

  const filteredStages = allStages.filter((s) => {
    return (
      (lifeStage === "All" || getLifeStage(s) === lifeStage) &&
      (strain === "All" || getStrain(s) === strain) &&
      (feeding === "All" || getFeeding(s) === feeding)
    );
  });

  const groupedStages = Object.entries(
    filteredStages.reduce((acc: Record<string, string[]>, s) => {
      const loc = getStrain(s);
      if (!acc[loc]) acc[loc] = [];
      acc[loc].push(s);
      return acc;
    }, {})
  ).map(([location, stages]) => ({
    location,
    stages: stages.sort((a, b) => getStageOrder(a) - getStageOrder(b)),
  }));

  /* =========================
     🔥 ACTIONS
  ========================== */

  const downloadImage = (key: string) => {
    const link = document.createElement("a");
    link.href = `${base}/${key}.png`;
    link.download = `${key}.png`;
    link.click();
  };

  const zoomImage = (key: string) => {
    window.open(`${base}/${key}.png`, "_blank");
  };

  const toggleImage = (key: string) => {
    setHiddenImages((prev) =>
      prev.includes(key)
        ? prev.filter((k) => k !== key)
        : [...prev, key]
    );
  };

  const renderFigure = (key: string, label: string) => {
    const isHidden = hiddenImages.includes(key);

    return (
      <div className={styles.figureBox} key={key}>
        <div className={styles.figureHeader}>
          <span>{label}</span>

          <div className={styles.headerActions}>
            <div className={styles.iconWrapper} data-tooltip="Download as PNG">
              <Camera
                size={16}
                onClick={() => downloadImage(key)}
                className={styles.toolbarIcon}
              />
            </div>

            <div className={styles.iconWrapper} data-tooltip="Zoom image">
              <ZoomIn
                size={16}
                onClick={() => zoomImage(key)}
                className={styles.toolbarIcon}
              />
            </div>

            <button
              className={styles.hideBtn}
              onClick={() => toggleImage(key)}
            >
              {isHidden ? "Show" : "Hide"}
            </button>
          </div>
        </div>

        {!isHidden && (
          <img
            src={`${base}/${key}.png`}
            className={styles.image}
            alt={key}
          />
        )}
      </div>
    );
  };

  const renderPair = (s: string) => {
    const readKey = `${s}_read5prime`;
    const regKey = `${s}_register`;

    return (
      <div className={styles.row} key={s}>
        {(view === "all" || view === "read5prime") &&
          renderFigure(readKey, "Read 5′")}
        {(view === "all" || view === "register") &&
          renderFigure(regKey, "Register")}
      </div>
    );
  };

  return (
    <div className={styles.container}>
      <div className={styles.topBar}>
        {/* 🔥 UPDATED TITLE BLOCK */}
        <div>
          <h2>{phasId}</h2>
          {stage !== "ALL" && (
            <div className={styles.defaultLabel}>
              Default: Best Sample
            </div>
          )}
        </div>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          <option value="ALL">All Samples</option>
          {groupedStages.map(({ location, stages }) => (
            <optgroup key={location} label={location}>
              {stages.map((s) => (
                <option key={s} value={s}>
                  {s.replaceAll("_", " ")}
                </option>
              ))}
            </optgroup>
          ))}
        </select>

        <div className={styles.filters}>
          <div className={styles.filterBox}>
            <label>Life stage</label>
            <select value={lifeStage} onChange={(e) => setLifeStage(e.target.value)}>
              <option>All</option>
              <option>Embryo</option>
              <option>Larva</option>
              <option>Nymph</option>
              <option>Adult</option>
            </select>
          </div>

          <div className={styles.filterBox}>
            <label>Strain</label>
            <select value={strain} onChange={(e) => setStrain(e.target.value)}>
              <option>All</option>
              <option>Oita</option>
              <option>Okayama</option>
            </select>
          </div>

          <div className={styles.filterBox}>
            <label>Feeding status</label>
            <select value={feeding} onChange={(e) => setFeeding(e.target.value)}>
              <option>All</option>
              <option>Fed</option>
              <option>Unfed</option>
            </select>
          </div>
        </div>

        <div className={styles.analysisToggle}>
          <span
            className={view === "read5prime" ? styles.activeTab : ""}
            onClick={() => setView("read5prime")}
          >
            Read 5′
          </span>
          <span
            className={view === "register" ? styles.activeTab : ""}
            onClick={() => setView("register")}
          >
            Register
          </span>
        </div>

        <div className={styles.topActions}>
          <span
            onClick={() => {
              setStage("ALL");
              setView("all");
              setHiddenImages([]);
            }}
          >
            Show All
          </span>

          <span
            onClick={() => {
              setStage(bestStage);
              setView("all");
              setHiddenImages([]);
            }}
          >
            Show Best Sample
          </span>
        </div>
      </div>

      <div className={styles.grid}>
        {view !== "hide" &&
          (stage === "ALL"
            ? groupedStages.map(({ location, stages }) => (
                <div key={location}>
                  <h3 className={styles.groupHeader}>{location}</h3>
                  {stages.map((s) => (
                    <div key={s}>
                      <h4>{s.replaceAll("_", " ")}</h4>
                      {renderPair(s)}
                    </div>
                  ))}
                </div>
              ))
            : renderPair(stage))}
      </div>
    </div>
  );
}
```

# Updated Code: `PhasingPatternPanel.module.css`
```sh
.container {
  padding: 8px;
}

/* Top Bar */
.topBar {
  background: #2f3b45;
  color: white;
  padding: 10px 14px;
  border-radius: 6px;

  display: flex;
  align-items: center;
  gap: 14px;

  flex-wrap: nowrap;
  overflow-x: auto;
}

/* Title block */
.topBar > div:first-child {
  display: flex;
  flex-direction: column;
  min-width: 170px;
}

/* Title */
.topBar h2 {
  font-size: 20px;
  margin: 0;
  line-height: 1.2;
}

/* Default label */
.defaultLabel {
  font-size: 12px;
  color: #cbd5db;
  margin-top: 2px;
}

/* Dropdown */
.topBar select {
  font-size: 13px;
  padding: 3px 8px;
  min-width: 140px;
}

/* Actions */
.topActions {
  display: flex;
  gap: 12px;
  white-space: nowrap;
}

.topActions span {
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}

.topActions span:hover {
  text-decoration: underline;
}

/* Grid */
.grid {
  background: white;
  padding: 12px;
  margin-top: 8px;
  border-radius: 8px;
}

.row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 15px;
  margin-bottom: 15px;
}

.figureBox {
  width: 100%;
  max-width: 500px;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 6px;
  position: relative;
}

/* Figure Header */
.figureHeader {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
  font-size: 12px;
}

.headerActions {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* Icon Wrapper with tooltip */
.iconWrapper {
  position: relative;
  display: inline-block;
  cursor: pointer;
}

.iconWrapper::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 125%;
  left: 50%;
  transform: translateX(-50%);
  background: #333;
  color: white;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.2s ease;
  z-index: 10;
}

.iconWrapper:hover::after {
  opacity: 1;
}

.toolbarIcon {
  color: #555;
  padding: 2px;
  border-radius: 4px;
  transition: color 0.2s;
}

.toolbarIcon:hover {
  color: #000;
}

.image {
  width: 100%;
}

/* Buttons */
.hideBtn {
  background: #6c757d;
  color: white;
  border: none;
  padding: 3px 8px;
  font-size: 11px;
  border-radius: 4px;
  cursor: pointer;
}

.showBtn {
  background: #28a745;
  color: white;
  border: none;
  padding: 3px 8px;
  font-size: 11px;
  border-radius: 4px;
  cursor: pointer;
}

/* Analysis Toggle */
.analysisToggle {
  display: flex;
  gap: 8px;
  margin-left: 10px;
}

.analysisToggle span {
  font-size: 13px;
  font-weight: 500;
  padding: 6px 10px;
  border-radius: 6px;
  cursor: pointer;
  background: #444;
  color: white;
  white-space: nowrap;
}

.analysisToggle span:hover {
  background: #666;
}

.activeTab {
  background: #007bff !important;
}

/* Group Header */
.groupHeader {
  margin-top: 20px;
  margin-bottom: 10px;
  font-size: 18px;
  font-weight: bold;
  color: #2f3b45;
  border-bottom: 2px solid #ddd;
  padding-bottom: 4px;
}

/* Filters */
.filters {
  display: flex;
  gap: 12px;
  align-items: flex-end;
  white-space: nowrap;
}

.filterBox {
  display: flex;
  flex-direction: column;
  color: white;
}

.filterBox label {
  font-size: 12px;
  margin-bottom: 2px;
}

.filterBox select {
  font-size: 13px;
  padding: 3px 8px;
  border-radius: 4px;
}
```