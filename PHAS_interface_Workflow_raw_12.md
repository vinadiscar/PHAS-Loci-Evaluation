# Characterization of tick PHAS-producing loci

## Strand-specific 5' Nucleotide Analysis

 07.29.2026 

```sh
cd /work/ma-discar/PHAS/Hlongicornis/data/plots/5prime_Nucleotide_SizeDistribution

vi plot_5prime_nucleotide_strand_size_distribution.py
```

```sh 
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path
import re
import numpy as np


# =====================================================
# INPUT / OUTPUT DIRECTORIES
# =====================================================

input_dir = Path(
    "/work/ma-discar/PHAS/Hlo_PHAS_20251213/csv"
)

output_dir = Path(
    "/work/ma-discar/PHAS/Hlongicornis/data/plots/5prime_Nucleotide_SizeDistribution"
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
# NUCLEOTIDE COLORS
# =====================================================

NUCLEOTIDE_COLORS = {

    "U": "#4DAF4A",
    "A": "#E41A1C",
    "C": "#377EB8",
    "G": "#984EA3"

}


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



        # =================================================
        # Extract 5' nucleotide
        # =================================================

        df["nucleotide"] = (
            df["read_seq"]
            .str[0]
            .str.upper()
            .replace({"T": "U"})
        )


        # =================================================
        # Aggregate RPM by length, strand and nucleotide
        # =================================================

        agg = (

            df.groupby(
                [
                    "length",
                    "strand",
                    "nucleotide"
                ]
            )["hit-rpm-norm-counts"]

            .sum()

            .reset_index()

        )


        lengths = list(range(18,31))


        # =================================================
        # Prepare nucleotide data
        # =================================================

        plus_data = {

            nt: []

            for nt in ["U","A","C","G"]

        }


        minus_data = {

            nt: []

            for nt in ["U","A","C","G"]

        }



        for L in lengths:


            for nt in ["U","A","C","G"]:


                plus_value = agg[

                    (agg["length"] == L)

                    &
                    (agg["strand"] == "+")

                    &
                    (agg["nucleotide"] == nt)

                ]["hit-rpm-norm-counts"].sum()



                minus_value = agg[

                    (agg["length"] == L)

                    &
                    (agg["strand"] == "-")

                    &
                    (agg["nucleotide"] == nt)

                ]["hit-rpm-norm-counts"].sum()



                plus_data[nt].append(
                    plus_value
                )


                minus_data[nt].append(
                    minus_value
                )



        # =================================================
        # Symmetric y-axis
        # =================================================

        max_abundance = max(
            sum(plus_data[nt][i] for nt in ["U","A","C","G"])
            for i in range(len(lengths))
        )


        max_minus = max(
            sum(minus_data[nt][i] for nt in ["U","A","C","G"])
            for i in range(len(lengths))
        )


        ylim = max(
            max_abundance,
            max_minus
        ) * 1.25



        # =================================================
        # Plot
        # =================================================

        fig, ax = plt.subplots(
            figsize=(7,5)
        )



        # -----------------------------
        # Minus strand
        # -----------------------------

        bottom = np.zeros(len(lengths))


        for nt in ["U","A","C","G"]:


            values = np.array(
                minus_data[nt]
            )


            ax.bar(

                lengths,

                -values,

                bottom=-bottom,

                width=0.8,

                color=NUCLEOTIDE_COLORS[nt],

                label=nt

            )


            bottom += values



        # -----------------------------
        # Plus strand
        # -----------------------------

        bottom = np.zeros(len(lengths))


        for nt in ["U","A","C","G"]:


            values = np.array(
                plus_data[nt]
            )


            ax.bar(

                lengths,

                values,

                bottom=bottom,

                width=0.8,

                color=NUCLEOTIDE_COLORS[nt]

            )


            bottom += values




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
            "Summed 5′ nucleotide RPM",
            fontsize=14
        )


        ax.set_title(

            f"{locus_id}\n{sample_name}",

            fontsize=14,

            fontweight="bold",

            pad=10

        )


        # =================================================
        # Full box
        # =================================================

        for spine in ax.spines.values():

            spine.set_visible(True)

            spine.set_linewidth(1.2)



        # =================================================
        # Legend
        # =================================================

        ax.legend(

            title="5′ nucleotide",

            loc="upper right",

            bbox_to_anchor=(0.98,0.98),

            fontsize=11,

            frameon=True,

            facecolor="white",

            edgecolor="black",

            framealpha=1

        )


        plt.tight_layout()



        # =================================================
        # Save
        # =================================================

        output_png = (

            output_dir

            /

            f"{csv_file.stem}_{sample_name}_5prime_Nucleotide_SizeDistribution.png"

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
- #### Run 
```sh
nohup python plot_5prime_nucleotide_strand_size_distribution.py > plot_5prime_nucleotide_strand_size_distribution.log 2>&1 &
```

# Import 5' nucleotide distribution to PHASER

## 1. Copy images 
- from `/work/ma-discar/PHAS/Hlongicornis/data/plots/5prime_Nucleotide_SizeDistribution` to `frontend/public/phas_plots/organize_phas_images/{phasId}/{imageFile}`


### a) copy from /work/ma-discar to project disk
```sh
cp -r /work/ma-discar/PHAS/Hlongicornis/data/plots/5prime_Nucleotide_SizeDistribution /project/okamura-lab/Vina/PHAS/PHASloci_info_csv/5prime_nucleotide_size_distribution_plots
```
### b) copy from project disk to PHASER
```sh
cd "/Volumes/okamura-lab/Vina/PHAS/PHASloci_info_csv/5prime_nucleotide_size_distribution_plots/5prime_Nucleotide_SizeDistribution"

for file in *_5prime_Nucleotide_SizeDistribution.png; do
    phas=$(echo "$file" | cut -d "_" -f1)
    cp "$file" "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/phas_plots/organize_phas_images/$phas/"
done
```

#### Previous Code 

- ## SizeDistributionPanel.tsx
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
## Updated Scripts
- Add 5' nucleotide size distribution images 
- # SizeDistributionPanel.tsx

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
  // RENDER PAIRED IMAGES
  // ==========================
  const renderImage = (
    currentStage: string
  ) => {

    const imageFile =
      sizeMap[phasId]?.[currentStage];


    if (!imageFile) return null;


    // Generate paired 5' nucleotide image name
    const fivePrimeImageFile =
      imageFile.replace(
        "_sizeDistribution.png",
        "_5prime_Nucleotide_SizeDistribution.png"
      );


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


        <div className={styles.figurePair}>


          {/* ==========================
              Figure A:
              sRNA Size Distribution
          ========================== */}

          <div className={styles.figure}>

            <h4>
              sRNA Size Distribution
            </h4>

            <img
              src={`/phas_plots/organize_phas_images/${phasId}/${imageFile}`}
              alt={`${phasId} ${currentStage} size distribution`}
              className={styles.image}
            />

          </div>



          {/* ==========================
              Figure B:
              5' Nucleotide Size Distribution
          ========================== */}

          <div className={styles.figure}>

            <h4>
              5′ Nucleotide Size Distribution
            </h4>

            <img
              src={`/phas_plots/organize_phas_images/${phasId}/${fivePrimeImageFile}`}
              alt={`${phasId} ${currentStage} 5prime nucleotide size distribution`}
              className={styles.image}
            />

          </div>


        </div>

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

- ## SizeDistributionPanel.module.css
`frontend/src/components/SizeDistributionPanel/SizeDistributionPanel.module.css`

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


/* ==========================
   Paired figure layout
   ========================== */

.figurePair {
  display: flex;
  gap: 20px;
  align-items: flex-start;
}


.figure {
  flex: 1;
}


.figure h4 {
  margin-top: 0;
  margin-bottom: 12px;

  font-size: 14px;
  font-weight: 600;
  color: #34495e;
}


/* ==========================
   Image style
   ========================== */

.image {
  width: 100%;
  max-width: 500px;

  display: block;
  margin: 0 auto;

  border-radius: 8px;
  border: 1px solid #dfe6ee;
}


/* ==========================
   Empty message
   ========================== */

.empty {
  padding: 24px;
  text-align: center;
  color: #7f8c8d;
}


/* ==========================
   Figure spacing
   ========================== */

.figureBox {
  margin-bottom: 24px;
}


/* ==========================
   Toggle button
   ========================== */

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



