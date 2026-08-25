04.01.2026
##### PHAS_interface_Workflow_raw_4
> This is a raw file for building Phasing Pattern panel
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

# PHASE 2: Phasing Pattern Panel

-	Display pre-computed figures from /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/plots/radar_pos5

##### General Steps
1.	Prepare Data/Images (Frontend cannot read PDF directly)
2.	Organize Image Data and Store in public folder (frontend/public/phas_plots/*.png)
3.	Modify phasing panel.tsx
4.	Modify App.tsx (autoselect search result)
5. Modify SearchBox.tsx
5.	File names must match Phas loci list names 
---
# 1. Prepare Data/Images
## 1A. Convert pdf files to png files
- Use GNU Parallel to speed the conversion process
```sh
cp -r /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213 /work/ma-discar/PHAS

cd work/ma-discar/PHAS
```
- #### Install Tools
```sh
conda create -n phas_pdf python=3.11
conda activate phas_pdf

# Install packages
conda install -c conda-forge pdf2image pillow
conda install -c conda-forge pytesseract
conda install -c conda-forge tesseract
conda install -c conda-forge poppler
```
- #### Write Script

```sh
cd /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/radar_pos5_png_convert
vi pdf_to_png_parallel.py
```
- pdf_to_png_parallel.py
```sh
import os
import sys
from pdf2image import convert_from_path
import pytesseract

# ---------------------------
# Directories 
# ---------------------------
SRC_DIR = "/project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/plots/radar_pos5"
DST_DIR = "/project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_convert"
os.makedirs(DST_DIR, exist_ok=True)

# ---------------------------
# OCR config
# ---------------------------
# Crop ratio for upper-right corner: (left, top, right, bottom) as fraction of page width/height
CROP_BOX_RATIO = (0.7, 0.0, 1.0, 0.2)  

# ---------------------------
# Accept PDF filename as argument
# ---------------------------
if len(sys.argv) != 2:
    print("Usage: python pdf_to_png_with_ocr_single.py <PDF_FILENAME>")
    sys.exit(1)

pdf_file = sys.argv[1]
pdf_path = os.path.join(SRC_DIR, pdf_file)
base_name = os.path.splitext(pdf_file)[0]

print(f"[INFO] Processing {pdf_file} ...")

# ---------------------------
# Convert PDF pages → images
# ---------------------------
try:
    pages = convert_from_path(pdf_path, dpi=150)  # lower DPI for speed
except Exception as e:
    print(f"[ERROR] Failed to convert {pdf_file}: {e}")
    sys.exit(1)

# ---------------------------
# Process each page
# ---------------------------
for i, page in enumerate(pages, start=1):
    width, height = page.size
    left = int(CROP_BOX_RATIO[0] * width)
    top = int(CROP_BOX_RATIO[1] * height)
    right = int(CROP_BOX_RATIO[2] * width)
    bottom = int(CROP_BOX_RATIO[3] * height)
    cropped = page.crop((left, top, right, bottom))

    # OCR the cropped region
    try:
        label_text = pytesseract.image_to_string(cropped, config='--psm 6').strip()
    except Exception as e:
        print(f"[WARNING] OCR failed on {pdf_file} page {i}: {e}")
        label_text = ""

    # Clean label text
    label_text = label_text.replace(" ", "").replace("/", "_")
    if not label_text:
        label_text = f"page{i:02d}"

    # Save PNG
    out_filename = f"{base_name}_{label_text}.png"
    out_path = os.path.join(DST_DIR, out_filename)
    try:
        page.save(out_path, "PNG")
    except Exception as e:
        print(f"[ERROR] Failed to save PNG {out_filename}: {e}")

print(f"[INFO] Done {pdf_file}")
```
###### Run 
```sh
cd /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/radar_pos5_png_convert

# Run 8 PDFs in parallel (adjust -j for cores available)
nohup bash -c '
cd /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/plots/radar_pos5 &&
ls PHAS22*.pdf | parallel -j 8 python /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/radar_pos5_png_convert/pdf_to_png_parallel.py {}
' > conversion.log 2>&1 &
```
## 1B. Rename Output file names to produce ideal, clean names
    - Example: `PHAS22-100_radar_pos5_PerLib_1...OitaE15 101 100%5.png` to `PHAS22-100_OitaE15.png`

```sh
cd /project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_rename_FINAL
vi rename_phas_png.sh
```

```sh
#!/bin/bash

INPUT_DIR="/project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_convert"
OUTPUT_DIR="/project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_rename_FINAL"

mkdir -p "$OUTPUT_DIR"

cd "$INPUT_DIR" || { echo "❌ Input directory not found"; exit 1; }

echo "🚀 Starting FINAL strict renaming..."

for f in *.png; do

  # --- Extract PHAS ID ---
  phas=$(echo "$f" | grep -oE '^PHAS22-[0-9]+')

  # --- Extract everything after "..." ---
  after=$(echo "$f" | sed -E 's/.*\.{3}//')

  # --- Remove extension ---
  after=$(echo "$after" | sed -E 's/\.png$//')

  # --- KEEP ONLY valid stage (letters + underscores + optional E##) ---
  stage=$(echo "$after" | grep -oE '^[A-Za-z_]+(E[0-9]+)?')

  # --- Clean stage ---
  stage_clean=$(echo "$stage" | sed -E 's/_+/_/g')
  stage_clean=$(echo "$stage_clean" | sed -E 's/^_+|_+$//g')

  # --- Skip if extraction failed ---
  if [[ -z "$phas" || -z "$stage_clean" ]]; then
    echo "⚠️ Skipped: $f"
    continue
  fi

  new_name="${phas}_${stage_clean}.png"
  out_path="$OUTPUT_DIR/$new_name"

  # Avoid overwrite
  if [[ -e "$out_path" ]]; then
    echo "⚠️ Exists: $new_name"
    continue
  fi

  cp "$f" "$out_path"

  echo "✅ $f → $new_name"

done

echo "🎉 DONE CLEANING!"
```
###### Run
```sh
chmod +x rename_phas_png.sh
./rename_phas_png.sh
```
## 1C. split one combined figure into two separate PNGs:
```sh
cd /project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_rename_FINAL
vi split_phas_plots.py
```
- split_phas_plots.py

```sh
import os
from PIL import Image

# --- INPUT (your cleaned files) ---
INPUT_DIR = "/project/okamura-lab/Vina/PHAS/plots/radar_pos5_png_rename_FINAL"

# --- OUTPUT DIRECTORIES ---
OUT_LEFT = "/project/okamura-lab/Vina/PHAS/plots/Read_5prime_end_distribution"
OUT_RIGHT = "/project/okamura-lab/Vina/PHAS/plots/Register_distribution"

# Create output folders
os.makedirs(OUT_LEFT, exist_ok=True)
os.makedirs(OUT_RIGHT, exist_ok=True)

# --- PROCESS FILES ---
files = sorted([f for f in os.listdir(INPUT_DIR) if f.endswith(".png")])

print(f"Found {len(files)} images...")

for i, fname in enumerate(files, 1):
    in_path = os.path.join(INPUT_DIR, fname)
    print(f"[{i}/{len(files)}] Processing: {fname}")

    try:
        img = Image.open(in_path)
        width, height = img.size

        # Split image into left/right halves
        mid = width // 2

        left_img = img.crop((0, 0, mid, height))
        right_img = img.crop((mid, 0, width, height))

        # Save with SAME filename in different folders
        left_img.save(os.path.join(OUT_LEFT, fname))
        right_img.save(os.path.join(OUT_RIGHT, fname))

    except Exception as e:
        print(f"❌ Error processing {fname}: {e}")

print("🎉 All images split successfully!")
```
###### Run
```sh
conda activate phas_pdf   # your env
python split_phas_plots.py
```
---
04.02.2026
## 2. Organize Image Data and save in `public/phas_patterns`

```sh
cd /project/okamura-lab/Vina/PHAS/plots/organize_phas_images
vi organize_phas_images.py
```
- organize_phas_images.py
```sh
import os
import shutil

# Base directory
BASE_DIR = "/project/okamura-lab/Vina/PHAS/plots"

# Input subfolders
READ5_DIR = os.path.join(BASE_DIR, "Read_5prime_end_distribution")
REGISTER_DIR = os.path.join(BASE_DIR, "Register_distribution")

# Output directory
OUTPUT_DIR = "/work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas_images"

os.makedirs(OUTPUT_DIR, exist_ok=True)


def process_folder(input_dir, image_type):
    # Safety check
    if not os.path.exists(input_dir):
        print(f"❌ ERROR: Directory not found -> {input_dir}")
        return

    for filename in os.listdir(input_dir):
        if not filename.endswith(".png"):
            continue

        name = filename.replace(".png", "")
        parts = name.split("_")

        # Need at least PHAS ID + something else
        if len(parts) < 2:
            print(f"⚠️ Skipping invalid file: {filename}")
            continue

        # First part = PHAS ID
        phas_id = parts[0]

        # Everything else = stage/condition
        stage = "_".join(parts[1:])

        # Create PHAS folder safely
        phas_folder = os.path.join(OUTPUT_DIR, phas_id)
        os.makedirs(phas_folder, exist_ok=True)

        # New filename
        new_filename = f"{stage}_{image_type}.png"

        src_path = os.path.join(input_dir, filename)
        dst_path = os.path.join(phas_folder, new_filename)

        # Avoid duplicate overwrite (optional but useful)
        if os.path.exists(dst_path):
            print(f"⚠️ File exists, skipping: {dst_path}")
            continue

        shutil.copy2(src_path, dst_path)

        print(f"[{image_type}] {filename} → {phas_id}/{new_filename}")


# Process both types
process_folder(READ5_DIR, "read5prime")
process_folder(REGISTER_DIR, "register")

print("✅ Done organizing ALL PHAS images!")
```
- ###### Run 
```sh
nohup python3 organize_phas_images.py > organize.log 2>&1 &
```

- #### Copy organized image data
```sh
cp -r /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas_images project/okamura-lab/Vina/PHAS/plots/

cp -r /Volumes/okamura-lab/Vina/PHAS/plots/organize_phas_images "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/phas_plots"
```

- #### Erase top labels for each figure for a cleaner images 



```sh
cd /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas/phas_images_clean

vi erase_titles.py
conda activate pillow_env
```
- erase_titles.py

```sh
from PIL import Image, ImageDraw
import os

input_dir = "/work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas"
output_dir = "/work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas/phas_images_clean"

os.makedirs(output_dir, exist_ok=True)

ERASE_HEIGHT = 80  # adjust based on your images

for root, dirs, files in os.walk(input_dir):
    for file in files:
        if file.endswith(".png"):
            input_path = os.path.join(root, file)

            rel_path = os.path.relpath(root, input_dir)
            out_folder = os.path.join(output_dir, rel_path)
            os.makedirs(out_folder, exist_ok=True)

            output_path = os.path.join(out_folder, file)

            img = Image.open(input_path).convert("RGB")
            draw = ImageDraw.Draw(img)

            width, height = img.size

            # Draw white rectangle over top area
            draw.rectangle([0, 0, width, ERASE_HEIGHT], fill="white")

            img.save(output_path)

print("✅ Title erased (painted white).")
```

- ##### Run 
```sh
nohup python erase_titles.py > erase_log.txt 2>&1 &
```

---
04.03.2026

# 3. Production-ready code for Phasing Pattern Panel


```sh
# User Experience
Search → type "PHAS"
       ↓
Table appears

Click PHAS22-96
       ↓
LEFT SIDEBAR appears
       ↓
User selects:
   - Stage (OitaE1, etc.)
   - Analysis (Read 5′ / Register)

       ↓
Main panel updates instantly

# Folder Structure
src/
├── components/
│   ├── Sidebar/
│   │   ├── Sidebar.tsx
│   │   └── Sidebar.module.css
│   │
│   ├── PhasingPatternPanel/
│   │   ├── PhasingPatternPanel.tsx
│   │   └── PhasingPatternPanel.module.css
```

##### Modify src/components panels

- App.tsx
- PHASTable.tsx
- SearchBox.tsx
- PhasingPatternPanel.tsx


## 3.1 Modify `App.tsx` 

- ##### Add PhasingPattern panel
- ##### Add side bar panel

```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

// ✅ GLOBAL STAGES (shared with Sidebar)
const STAGES = [
  "OitaE1","OitaE5","OitaE10","OitaE15","OitaE20",
  "Oita_Adultfemale_Unfed","Oita_Adultfemale_fed",
  "Oita_Adultmale_Unfed","Oita_Adultmale_fed",
  "Oita_Larva_Unfed","Oita_Larva_fed",
  "Oita_Nymph_Unfed","Oita_Nymph_fed",
  "OkayamaE1","OkayamaE5","OkayamaE10","OkayamaE15","OkayamaE20",
  "Okayama_Adult_Fed","Okayama_Adult_Unfed",
  "Okayama_Larva_Unfed","Okayama_Larva_fed",
  "Okayama_Nymph_Unfed","Okayama_Nymph_fed",
];

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  // 🔹 Sidebar state
  const [activeStage, setActiveStage] = useState(STAGES[0]);
  const [view, setView] = useState<"read5prime" | "register">("read5prime");

  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);
      })
      .finally(() => setLoading(false));
  }, []);

  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    // ✅ Auto-select exact match
    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  const showTable = searchQuery !== "" || selectedPhas !== null;

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        {showTable && (
          <PHASTable
            records={filteredRecords}
            selectedId={selectedPhas?.phas_id || null}
            onSelect={setSelectedPhas}
            loading={loading}
          />
        )}

        {/* 🔥 MAIN LAYOUT */}
        {selectedPhas && (
          <div className="layout">
            {/* LEFT SIDEBAR */}
            <Sidebar
              stages={STAGES}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            {/* RIGHT PANEL */}
            <div className="mainContent">
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;

``` 
---
## 3.2 Create Sidebar

```sh
cd src/components
mkdir Sidebar
vi Sidebar.tsx
vi Sidebar.module.css
```
- ### Sidebar.tsx
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register";
  setView: (v: "read5prime" | "register") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {
  return (
    <div className={styles.sidebar}>
      <h3>Stages</h3>
      <div className={styles.section}>
        {stages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${
              activeStage === s ? styles.active : ""
            }`}
            onClick={() => setActiveStage(s)}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>

      <h3>Analysis</h3>
      <div className={styles.section}>
        <div
          className={`${styles.item} ${
            view === "read5prime" ? styles.active : ""
          }`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>

        <div
          className={`${styles.item} ${
            view === "register" ? styles.active : ""
          }`}
          onClick={() => setView("register")}
        >
          Register
        </div>
      </div>
    </div>
  );
}
```
- #### Sidebar.modular.css
```sh
.sidebar {
  width: 260px;
  border-right: 1px solid #ddd;
  padding: 10px;
  height: 80vh;
  overflow-y: auto;
}

.section {
  margin-bottom: 20px;
}

.item {
  padding: 6px;
  cursor: pointer;
}

.item:hover {
  background: #f5f5f5;
}

.active {
  background: #007bff;
  color: white;
}

```
### 3.4 Create PhasingPatternPanel
- #### PhasingPatternPanel.tsx
```sh
import { useState } from "react";
import styles from "./PhasingPatternPanel.module.css";

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
}: Props) {
  const [error, setError] = useState(false);

  const src = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}/${stage}_${view}.png`;

  console.log("Rendering image:", src);

  return (
    <div className={styles.container}>
      <h2>{phasId}</h2>
      <h3>{stage.replaceAll("_", " ")}</h3>

      <img
        key={src}  // 🔥 important
        src={src}
        className={styles.image}
        onError={() => setError(true)}
      />

      {error && (
        <p style={{ color: "red" }}>
          ⚠️ Failed to load: {src}
        </p>
      )}
    </div>
  );
}

```
- #### App.css
```sh
/* Global Styles */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
    sans-serif;
  background: #f5f6fa;
  color: #2c3e50;
  line-height: 1.5;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.main {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding-bottom: 100px; /* Space for tab panel */
}

.error-message {
  margin: 0 24px 16px;
  padding: 12px 16px;
  background: #fdecea;
  color: #c0392b;
  border-radius: 8px;
  font-size: 14px;
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: #f1f2f6;
}

::-webkit-scrollbar-thumb {
  background: #bdc3c7;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #95a5a6;
}

/* IGV.js custom styling */
.igv-container {
  font-family: inherit !important;
}

.igv-navbar {
  background: #f8f9fa !important;
  border-bottom: 1px solid #ecf0f1 !important;
}

.layout {
  display: flex;
  margin-top: 20px;
}

.mainContent {
  flex: 1;
  padding: 20px;
}
```
### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/
- ##### It worked!
- ### Issues:
    - All samples were included for each PHAS analysis (In actual, not all samples have detected PHAS loci)
---
04.04.2026
# Phasing Panel ver. 2
### Solve:
### 1. Create `stages.json`
- ##### list developmental stages from .png files that exist inside each PHAS folder

```sh
cd /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas
vi list_stages.py
```
```sh
import os

# Base directory
BASE_DIR = "/work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas"

for phas_folder in sorted(os.listdir(BASE_DIR)):
    phas_path = os.path.join(BASE_DIR, phas_folder)

    if os.path.isdir(phas_path):
        stages = set()

        for f in os.listdir(phas_path):
            if f.lower().endswith(".png"):
                # Remove file extension
                name = f.replace(".png", "")

                # Remove suffix (_register or _read5prime)
                if name.endswith("_register"):
                    stage = name.replace("_register", "")
                elif name.endswith("_read5prime"):
                    stage = name.replace("_read5prime", "")
                else:
                    stage = name  # fallback (just in case)

                stages.add(stage)

        # Print results
        stage_list = sorted(stages)
        print(f"{phas_folder}: {len(stage_list)} stages")
        print("  " + ", ".join(stage_list))
```
```sh
nohup python3 list_stages.py > list_stages_output.log 2>&1 &
```
- ##### Extract unique stages per PHAS
```sh
cd /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas
vi generate_stages_json.py
``` 
- generate_stages_json.py
```sh
import os

# Base directory
BASE_DIR = "/work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas"

for phas_folder in sorted(os.listdir(BASE_DIR)):
    phas_path = os.path.join(BASE_DIR, phas_folder)

    if os.path.isdir(phas_path):
        stages = set()

        for f in os.listdir(phas_path):
            if f.lower().endswith(".png"):
                # Remove file extension
                name = f.replace(".png", "")

                # Remove suffix (_register or _read5prime)
                if name.endswith("_register"):
                    stage = name.replace("_register", "")
                elif name.endswith("_read5prime"):
                    stage = name.replace("_read5prime", "")
                else:
                    stage = name  # fallback (just in case)

                stages.add(stage)

        # Print results
        stage_list = sorted(stages)
        print(f"{phas_folder}: {len(stage_list)} stages")
        print("  " + ", ".join(stage_list))
```
- ##### Run:
```sh
nohup python3 generate_stages_json.py > list_stages_output.log 2>&1 &
```
### copy `stages.json` to:
`frontend/public/phas_plots/organize_phas_images/stages.json`
```sh
cp /work/ma-discar/PHAS/Hlo_PHAS_20251213/plots/organize_phas/stages.json /project/okamura-lab/Vina/PHAS/plots/organize_phas_images
cp /Volumes/okamura-lab/Vina/PHAS/plots/organize_phas_images/stages.json "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/phas_plots/organize_phas_images"
```
### 2. Modify App.tsx
- add new state
```sh
const [availableStages, setAvailableStages] = useState<string[]>([]);
```
- Add this `useEffect`
```sh
useEffect(() => {
  if (!selectedPhas) return;

  fetch("/phas_plots/organize_phas_images/stages.json")
    .then((res) => res.json())
    .then((data) => {
      const stages = data[selectedPhas.phas_id] || [];

      setAvailableStages(stages);

      // Reset to first valid stage
      if (stages.length > 0) {
        setActiveStage(stages[0]);
      }
    })
    .catch((err) => {
      console.error("Failed to load stages.json:", err);
      setAvailableStages([]);
    });
}, [selectedPhas]);
```

- #### App.tsx ver. 4
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  // 🔥 NEW: dynamic stages
  const [availableStages, setAvailableStages] = useState<string[]>([]);

  // Sidebar state
  const [activeStage, setActiveStage] = useState("");
  const [view, setView] = useState<"read5prime" | "register">("read5prime");

  // Load PHAS table
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search filtering
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // 🔥 NEW: Load stages.json dynamically
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];

        setAvailableStages(stages);

        // auto-select first stage
        if (stages.length > 0) {
          setActiveStage(stages[0]);
        } else {
          setActiveStage("");
        }
      })
      .catch((err) => {
        console.error("Failed to load stages.json:", err);
        setAvailableStages([]);
        setActiveStage("");
      });
  }, [selectedPhas]);

  const showTable = searchQuery !== "" || selectedPhas !== null;

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        {showTable && (
          <PHASTable
            records={filteredRecords}
            selectedId={selectedPhas?.phas_id || null}
            onSelect={setSelectedPhas}
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="layout">
            <Sidebar
              stages={availableStages}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            <div className="mainContent">
              {activeStage ? (
                <PhasingPatternPanel
                  phasId={selectedPhas.phas_id}
                  stage={activeStage}
                  view={view}
                />
              ) : (
                <p>No stages available for this PHAS.</p>
              )}
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;

```
### 3. Modify Sidebar
- Modify Sidebar props
Before:
```sh
<Sidebar
  stages={STAGES}
```
After:
```sh
<Sidebar
  stages={availableStages}
```
- Modify Sidebar.tsx
Replace
```sh
{stages.map((s) => (
```
with
```sh
{stages.length === 0 ? (
  <p>No stages available</p>
) : (
  stages.map((s) => (
    <div
      key={s}
      className={`${styles.item} ${
        activeStage === s ? styles.active : ""
      }`}
      onClick={() => setActiveStage(s)}
    >
      {s.replaceAll("_", " ")}
    </div>
  ))
)}
```
- #### Sidebar.tsx ver. 2 
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register";
  setView: (v: "read5prime" | "register") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {
  return (
    <div className={styles.sidebar}>
      <h3>Stages</h3>

      <div className={styles.section}>
        {stages.length === 0 ? (
          <p>No stages available</p>
        ) : (
          stages.map((s) => (
            <div
              key={s}
              className={`${styles.item} ${
                activeStage === s ? styles.active : ""
              }`}
              onClick={() => setActiveStage(s)}
            >
              {s.replaceAll("_", " ")}
            </div>
          ))
        )}
      </div>

      <h3>Analysis</h3>

      <div className={styles.section}>
        <div
          className={`${styles.item} ${
            view === "read5prime" ? styles.active : ""
          }`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>

        <div
          className={`${styles.item} ${
            view === "register" ? styles.active : ""
          }`}
          onClick={() => setView("register")}
        >
          Register
        </div>
      </div>
    </div>
  );
}
```
### 4. Modify PhasingPattern Panel
- Reset error when stage/view changes

- #### PhasingPatternPanel.tsx ver 
```sh
import { useState, useEffect } from "react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register";
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
}: Props) {
  const [error, setError] = useState(false);

  const src = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}/${stage}_${view}.png`;

  // 🔥 Reset error when inputs change
  useEffect(() => {
    setError(false);
  }, [phasId, stage, view]);

  console.log("Loading image:", src);

  return (
    <div className={styles.container}>
      <h2>{phasId}</h2>
      <h3>{stage.replaceAll("_", " ")}</h3>

      {!error ? (
        <img
          key={src}  // 🔥 force re-render
          src={src}
          className={styles.image}
          onError={() => setError(true)}
        />
      ) : (
        <p style={{ color: "red" }}>
          ⚠️ No image available for this stage/view
        </p>
      )}
    </div>
  );
}
```
### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/
#### It worked!
### Next: Upgrade!
---
# Phasing Panel ver. 3
✔ Default PHAS22-1 + OitaE10
✔ Auto-select earliest stage after search
✔ “All” + “Hide” options
✔ Analysis first, Samples second
✔ Side-by-side images (Register LEFT, Read 5′ RIGHT)
✔ Grid when “All samples”
✔ Clean UI-ready structure

## 1. App.tsx ver.

```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);

  // 🔥 DEFAULTS
  const [activeStage, setActiveStage] = useState("OitaE10");
  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  // Load PHAS
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        // 🔥 DEFAULT PHAS
        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // 🔥 Load stages dynamically
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];

        setAvailableStages(stages);

        if (stages.length > 0) {
          setActiveStage(stages[0]); // earliest stage
        } else {
          setActiveStage("Hide");
        }
      });
  }, [selectedPhas]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={setSelectedPhas}
          loading={loading}
        />

        {selectedPhas && (
          <div className="layout">
            <Sidebar
              stages={availableStages}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            <div className="mainContent">
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```

## 2. Sidebar.tsx

```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {
  return (
    <div className={styles.sidebar}>
      {/* 🔥 ANALYSIS FIRST */}
      <div className={styles.box}>
        <h3>Analysis</h3>

        {["all", "register", "read5prime", "hide"].map((v) => (
          <div
            key={v}
            className={`${styles.item} ${
              view === v ? styles.active : ""
            }`}
            onClick={() => setView(v as any)}
          >
            {v === "all"
              ? "All"
              : v === "read5prime"
              ? "Read 5′"
              : v === "hide"
              ? "Hide"
              : "Register"}
          </div>
        ))}
      </div>

      {/* 🔥 SAMPLES */}
      <div className={styles.box}>
        <h3>Samples</h3>

        {["All", "Hide"].map((s) => (
          <div
            key={s}
            className={`${styles.item} ${
              activeStage === s ? styles.active : ""
            }`}
            onClick={() => setActiveStage(s)}
          >
            {s}
          </div>
        ))}

        {stages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${
              activeStage === s ? styles.active : ""
            }`}
            onClick={() => setActiveStage(s)}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}

```

## 3. PhasingPatternPanel.tsx

```sh
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
}: Props) {
  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  if (view === "hide" || stage === "Hide") {
    return (
      <div className={styles.container}>
        <h2>{phasId}</h2>
        <p>No visualization selected.</p>
      </div>
    );
  }

  const renderImage = (s: string, type: string) => {
    const src = `${base}/${s}_${type}.png`;
    return <img key={src} src={src} className={styles.image} />;
  };

  return (
    <div className={styles.container}>
      <h2>{phasId}</h2>

      <div className={styles.grid}>
        {/* 🔥 SINGLE SAMPLE */}
        {stage !== "All" && (
          <>
            {(view === "all" || view === "register") &&
              renderImage(stage, "register")}

            {(view === "all" || view === "read5prime") &&
              renderImage(stage, "read5prime")}
          </>
        )}

        {/* 🔥 ALL SAMPLES GRID */}
        {stage === "All" &&
          allStages.map((s) => (
            <div key={s} className={styles.sampleBlock}>
              <h4>{s.replaceAll("_", " ")}</h4>

              <div className={styles.row}>
                {(view === "all" || view === "register") &&
                  renderImage(s, "register")}

                {(view === "all" || view === "read5prime") &&
                  renderImage(s, "read5prime")}
              </div>
            </div>
          ))}
      </div>
    </div>
  );
}
```

## 4. Sidebar.module.css

```sh
.sidebar {
  width: 260px;
  padding: 10px;
  height: 80vh;
  overflow-y: auto;
}

.box {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 10px;
  margin-bottom: 15px;
  background: #ffffff;
}

.item {
  padding: 8px;
  cursor: pointer;
  border-radius: 4px;
}

.item:hover {
  background: #f0f0f0;
}

.active {
  background: #007bff;
  color: white;
}
```

## 5. PhasingPatternPanel.module.css

```sh
.container {
  padding: 10px;
}

.grid {
  background: #ffffff;
  padding: 15px;
  border-radius: 8px;
}

.row {
  display: flex;
  gap: 20px;
}

.image {
  width: 100%;
  max-width: 400px;
  border: 1px solid #ddd;
}

.sampleBlock {
  margin-bottom: 25px;
}
```

### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/
#### It worked!
### Next: Upgrade!

---

# Phasing Panel ver. 4

Upgrade:
###### Sidebar
✔ Clean dashboard layout
✔ Show All / Hide All inline
✔ Sorted samples
###### Main Panel
✔ Professional dark header
✔ Dropdown for samples
✔ Show/Hide per figure
✔ Clean layout

## 1.  `App.tsx` ver. 

```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);

  const [activeStage, setActiveStage] = useState("OitaE10");
  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  // Load PHAS
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // Load stages
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);

        if (stages.length > 0) {
          setActiveStage(stages[0]); // earliest
        } else {
          setActiveStage("Hide");
        }
      });
  }, [selectedPhas]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={setSelectedPhas}
          loading={loading}
        />

        {selectedPhas && (
          <div className="layout">
            <Sidebar
              stages={availableStages}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            <div className="mainContent">
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
                setStage={setActiveStage}   {/* 🔥 sync dropdown */}
                setView={setView}           {/* 🔥 sync top bar */}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
## 2.  `Sidebar.tsx` ver. 
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {

  const sortedStages = [...stages].sort();

  return (
    <div className={styles.sidebar}>

      {/* ANALYSIS */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Analysis</h3>
          <div className={styles.actions}>
            <span onClick={() => setView("all")}>Show All</span>
            <span onClick={() => setView("hide")}>Hide All</span>
          </div>
        </div>

        <div className={`${styles.item} ${view === "register" ? styles.active : ""}`}
          onClick={() => setView("register")}>
          Register
        </div>

        <div className={`${styles.item} ${view === "read5prime" ? styles.active : ""}`}
          onClick={() => setView("read5prime")}>
          Read 5′
        </div>
      </div>

      {/* SAMPLES */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Samples</h3>
          <div className={styles.actions}>
            <span onClick={() => setActiveStage("All")}>Show All</span>
            <span onClick={() => setActiveStage("Hide")}>Hide All</span>
          </div>
        </div>

        {sortedStages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${activeStage === s ? styles.active : ""}`}
            onClick={() => setActiveStage(s)}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}
```
## 3. Sidebar.module.css
```sh
.sidebar {
  width: 260px;
  padding: 10px;
  height: 80vh;
  overflow-y: auto;
}

.box {
  border: 1px solid #ddd;
  border-radius: 10px;
  padding: 12px;
  margin-bottom: 15px;
  background: #ffffff;
}

.headerRow {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.actions span {
  font-size: 12px;
  margin-left: 8px;
  cursor: pointer;
  color: #007bff;
}

.item {
  padding: 8px;
  cursor: pointer;
  border-radius: 5px;
}

.item:hover {
  background: #f0f0f0;
}

.active {
  background: #007bff;
  color: white;
}
```
## 4. PhasingPatternPanel.tsx
```sh
import { useState } from "react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {

  const [hiddenImages, setHiddenImages] = useState<string[]>([]);

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  const toggleImage = (key: string) => {
    setHiddenImages((prev) =>
      prev.includes(key)
        ? prev.filter((k) => k !== key)
        : [...prev, key]
    );
  };

  const renderImage = (s: string, type: string) => {
    const key = `${s}_${type}`;
    const src = `${base}/${key}.png`;

    return (
      <div className={styles.figureBox} key={key}>
        <div className={styles.figureHeader}>
          <span>{type === "read5prime" ? "Read 5′" : "Register"}</span>
          <div>
            <button onClick={() => toggleImage(key)}>Hide</button>
          </div>
        </div>

        {!hiddenImages.includes(key) && (
          <img src={src} className={styles.image} />
        )}
      </div>
    );
  };

  return (
    <div className={styles.container}>

      {/* 🔥 TOP BAR */}
      <div className={styles.topBar}>
        <h2>{phasId}</h2>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          {allStages.map((s) => (
            <option key={s} value={s}>
              {s.replaceAll("_", " ")}
            </option>
          ))}
        </select>

        <div className={styles.topActions}>
          <span onClick={() => setView("all")}>Show All</span>
          <span onClick={() => setView("hide")}>Hide All</span>
        </div>
      </div>

      <div className={styles.grid}>
        {stage !== "All" && (
          <>
            {(view === "all" || view === "register") &&
              renderImage(stage, "register")}

            {(view === "all" || view === "read5prime") &&
              renderImage(stage, "read5prime")}
          </>
        )}

        {stage === "All" &&
          allStages.map((s) => (
            <div key={s}>
              <h4>{s}</h4>
              <div className={styles.row}>
                {(view === "all" || view === "register") &&
                  renderImage(s, "register")}

                {(view === "all" || view === "read5prime") &&
                  renderImage(s, "read5prime")}
              </div>
            </div>
          ))}
      </div>
    </div>
  );
}
```
## 5. PhasingPatternPanel.module.css

```sh
.container {
  padding: 10px;
}

.topBar {
  background: #2f3b45;
  color: white;
  padding: 12px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 15px;
}

.topBar h2 {
  margin-right: auto;
}

.topActions span {
  font-size: 12px;
  margin-left: 10px;
  cursor: pointer;
}

/* GRID */
.grid {
  background: white;
  padding: 15px;
  margin-top: 10px;
  border-radius: 8px;
}

.row {
  display: flex;
  gap: 20px;
}

/* FIGURE */
.figureBox {
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 8px;
}

.figureHeader {
  display: flex;
  justify-content: space-between;
}

.image {
  width: 100%;
  max-width: 400px;
}

```
### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/
#### It worked but there are some issues!
- clicking "Hide All' shows sample list or placeholder UI still renders
- Currently stacked vertically
### Next: Troubleshoot

---
# Phasing Panel ver. 5
###### Present issues: 
- clicking "Hide All' shows sample list or placeholder UI still renders
- Currently stacked vertically

###### Modify the following files:
- App.tsx
- PhasingPatternPanel.tsx
- Sidebar.tsx
- PhasingPatternPanel.module.css
- Sidebar.module.css

## 1. App.tsx ver
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);

  const [activeStage, setActiveStage] = useState("OitaE10");
  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  // Load PHAS
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // Load stages
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);

        if (stages.length > 0) {
          setActiveStage(stages[0]);
        } else {
          setActiveStage("Hide");
        }
      });
  }, [selectedPhas]);

  // 🔥 GLOBAL HIDE CONDITION
  const shouldShowPanel =
    selectedPhas &&
    view !== "hide" &&
    activeStage !== "Hide";

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={setSelectedPhas}
          loading={loading}
        />

        {selectedPhas && (
          <div className="layout">
            <Sidebar
              stages={availableStages}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            <div className="mainContent">
              {shouldShowPanel && (
                <PhasingPatternPanel
                  phasId={selectedPhas.phas_id}
                  stage={activeStage}
                  view={view}
                  allStages={availableStages}
                  setStage={setActiveStage}
                  setView={setView}
                />
              )}
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
## 2. Sidebar.tsx
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {

  const sortedStages = [...stages].sort();

  return (
    <div className={styles.sidebar}>

      {/* ANALYSIS */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Analysis</h3>
          <div className={styles.actions}>
            <span onClick={() => setView("all")}>Show All</span>
            <span onClick={() => setView("hide")}>Hide All</span>
          </div>
        </div>

        <div
          className={`${styles.item} ${view === "register" ? styles.active : ""}`}
          onClick={() => setView("register")}
        >
          Register
        </div>

        <div
          className={`${styles.item} ${view === "read5prime" ? styles.active : ""}`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>
      </div>

      {/* SAMPLES */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Samples</h3>
          <div className={styles.actions}>
            <span
              onClick={() => {
                if (sortedStages.length > 0) {
                  setActiveStage(sortedStages[0]);
                  setView("all");
                }
              }}
            >
              Show All
            </span>

            <span onClick={() => setView("hide")}>
              Hide All
            </span>
          </div>
        </div>

        {sortedStages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${activeStage === s ? styles.active : ""}`}
            onClick={() => {
              setActiveStage(s);
              setView("all");
            }}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 3. Sidebar.module.css
```sh
.sidebar {
  width: 260px;
  padding: 12px;
  height: 80vh;
  overflow-y: auto;
  background: #f8f9fa;
}

.box {
  border: 1px solid #ddd;
  border-radius: 10px;
  padding: 12px;
  margin-bottom: 15px;
  background: #ffffff;
}

/* HEADER */
.headerRow {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.headerRow h3 {
  font-size: 14px;
  margin: 0;
}

/* ACTIONS */
.actions span {
  font-size: 11px;
  margin-left: 8px;
  cursor: pointer;
  color: #007bff;
}

.actions span:hover {
  text-decoration: underline;
}

/* ITEMS */
.item {
  padding: 8px;
  cursor: pointer;
  border-radius: 6px;
  font-size: 13px;
  margin-top: 4px;
}

.item:hover {
  background: #f0f0f0;
}

.active {
  background: #007bff;
  color: white;
}
```
## 4. PhasingPatternPanel.tsx
```sh
import { useState } from "react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {

  const [hiddenImages, setHiddenImages] = useState<string[]>([]);

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  // 🔥 HARD STOP: show nothing
  if (view === "hide") return null;

  const toggleImage = (key: string) => {
    setHiddenImages((prev) =>
      prev.includes(key)
        ? prev.filter((k) => k !== key)
        : [...prev, key]
    );
  };

  const renderPair = (s: string) => {
    const readKey = `${s}_read5prime`;
    const regKey = `${s}_register`;

    return (
      <div className={styles.row} key={s}>

        {/* LEFT */}
        {(view === "all" || view === "read5prime") && (
          <div className={styles.figureBox}>
            <div className={styles.figureHeader}>
              <span>Read 5′</span>
              <button onClick={() => toggleImage(readKey)}>Hide</button>
            </div>

            {!hiddenImages.includes(readKey) && (
              <img src={`${base}/${readKey}.png`} className={styles.image} />
            )}
          </div>
        )}

        {/* RIGHT */}
        {(view === "all" || view === "register") && (
          <div className={styles.figureBox}>
            <div className={styles.figureHeader}>
              <span>Register</span>
              <button onClick={() => toggleImage(regKey)}>Hide</button>
            </div>

            {!hiddenImages.includes(regKey) && (
              <img src={`${base}/${regKey}.png`} className={styles.image} />
            )}
          </div>
        )}

      </div>
    );
  };

  return (
    <div className={styles.container}>

      <div className={styles.topBar}>
        <h2>{phasId}</h2>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          {allStages.map((s) => (
            <option key={s} value={s}>
              {s.replaceAll("_", " ")}
            </option>
          ))}
        </select>

        <div className={styles.topActions}>
          <span onClick={() => setView("all")}>Show All</span>
          <span onClick={() => setView("hide")}>Hide All</span>
        </div>
      </div>

      <div className={styles.grid}>
        {stage !== "All" && renderPair(stage)}

        {stage === "All" &&
          allStages.map((s) => (
            <div key={s}>
              <h4>{s}</h4>
              {renderPair(s)}
            </div>
          ))}
      </div>
    </div>
  );
}
```
## 5. PhasingPatternPanel.module.css
```sh
.container {
  padding: 10px;
}

/* TOP BAR */
.topBar {
  background: #2f3b45;
  color: white;
  padding: 12px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 15px;
}

.topBar h2 {
  margin-right: auto;
}

.topActions span {
  font-size: 12px;
  margin-left: 10px;
  cursor: pointer;
}

/* GRID */
.grid {
  background: white;
  padding: 15px;
  margin-top: 10px;
  border-radius: 8px;
}

/* 🔥 HORIZONTAL LAYOUT */
.row {
  display: flex;
  gap: 20px;
  align-items: flex-start;
  margin-bottom: 20px;
}

.figureBox {
  flex: 1;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 8px;
}

.figureHeader {
  display: flex;
  justify-content: space-between;
  margin-bottom: 6px;
}

.image {
  width: 100%;
  max-width: 100%;
}
```
### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/
- #### Issues found 
    - “Show All” not working
    - “Hide All” should NOT hide PHAS ID top bar
    - Per-figure button upgrade
    - Top bar too wide
---
# Phasing Panel ver. 6
## Upgrade: 
✅ Show All works everywhere
✅ Hide All hides only figures
✅ PHAS ID bar always visible
✅ Toggle buttons switch (Hide ↔ Show)
✅ Clean horizontal layout
✅ Smaller, cleaner UI
## 1. App.tsx ver. 
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import Sidebar from "./components/Sidebar/Sidebar";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("OitaE10");

  const [view, setView] = useState<
    "read5prime" | "register" | "all" | "hide"
  >("all");

  // Load PHAS
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // Load stages
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);

        if (stages.length > 0) {
          setActiveStage(stages[0]);
        }
      });
  }, [selectedPhas]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={setSelectedPhas}
          loading={loading}
        />

        {selectedPhas && (
          <div className="layout">
            <Sidebar
              stages={availableStages}
              activeStage={activeStage}
              setActiveStage={setActiveStage}
              view={view}
              setView={setView}
            />

            <div className="mainContent">
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
                setStage={setActiveStage}
                setView={setView}
              />
            </div>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
## 2. Sidebar.tsx ver. 

```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {

  const sortedStages = [...stages].sort();

  return (
    <div className={styles.sidebar}>

      {/* ANALYSIS */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Analysis</h3>
          <div className={styles.actions}>
            <span
              onClick={() => {
                setView("all");
                if (stages.length > 0) {
                  setActiveStage(stages[0]);
                }
              }}
            >
              Show All
            </span>

            <span onClick={() => setView("hide")}>
              Hide All
            </span>
          </div>
        </div>

        <div
          className={`${styles.item} ${view === "register" ? styles.active : ""}`}
          onClick={() => setView("register")}
        >
          Register
        </div>

        <div
          className={`${styles.item} ${view === "read5prime" ? styles.active : ""}`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>
      </div>

      {/* SAMPLES */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Samples</h3>
        </div>

        {sortedStages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${activeStage === s ? styles.active : ""}`}
            onClick={() => {
              setActiveStage(s);
              setView("all");
            }}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}

```
## 3. Sidebar.module.css ver. 

```sh
.sidebar {
  width: 260px;
  padding: 12px;
  height: 80vh;
  overflow-y: auto;
  background: #f8f9fa;
}

.box {
  border: 1px solid #ddd;
  border-radius: 10px;
  padding: 12px;
  margin-bottom: 15px;
  background: #ffffff;
}

.headerRow {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.headerRow h3 {
  font-size: 14px;
  margin: 0;
}

.actions span {
  font-size: 11px;
  margin-left: 8px;
  cursor: pointer;
  color: #007bff;
}

.actions span:hover {
  text-decoration: underline;
}

.item {
  padding: 8px;
  cursor: pointer;
  border-radius: 6px;
  font-size: 13px;
  margin-top: 4px;
}

.item:hover {
  background: #f0f0f0;
}

.active {
  background: #007bff;
  color: white;
}
```
## 4. PhasingPatternPanel.tsx ver. 

```sh
import { useState } from "react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {

  const [hiddenImages, setHiddenImages] = useState<string[]>([]);

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

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

          <button
            className={isHidden ? styles.showBtn : styles.hideBtn}
            onClick={() => toggleImage(key)}
          >
            {isHidden ? "Show" : "Hide"}
          </button>
        </div>

        {!isHidden && (
          <img src={`${base}/${key}.png`} className={styles.image} />
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
        <h2>{phasId}</h2>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          {allStages.map((s) => (
            <option key={s} value={s}>
              {s.replaceAll("_", " ")}
            </option>
          ))}
        </select>

        <div className={styles.topActions}>
          <span onClick={() => setView("all")}>Show All</span>
          <span onClick={() => setView("hide")}>Hide All</span>
        </div>
      </div>

      <div className={styles.grid}>
        {view !== "hide" && (
          <>
            {stage !== "All" && renderPair(stage)}

            {stage === "All" &&
              allStages.map((s) => (
                <div key={s}>
                  <h4>{s}</h4>
                  {renderPair(s)}
                </div>
              ))}
          </>
        )}
      </div>
    </div>
  );
}
```
## 5. PhasingPatternPanel.module.css ver. 

```sh
.container {
  padding: 8px;
}

/* smaller top bar */
.topBar {
  background: #2f3b45;
  color: white;
  padding: 8px 10px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.topBar h2 {
  margin-right: auto;
  font-size: 16px;
}

.topBar select {
  font-size: 12px;
}

.topActions span {
  font-size: 11px;
  margin-left: 8px;
  cursor: pointer;
}

.grid {
  background: white;
  padding: 12px;
  margin-top: 8px;
  border-radius: 8px;
}

.row {
  display: flex;
  gap: 15px;
  margin-bottom: 15px;
}

.figureBox {
  flex: 1;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 6px;
}

.figureHeader {
  display: flex;
  justify-content: space-between;
  margin-bottom: 5px;
  font-size: 12px;
}

.image {
  width: 100%;
}

/* buttons */
.hideBtn {
  background: #dc3545;
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
```
### Test: Phasing Panel
- `bash.start.sh` http://163.221.246.151:5174/

- ##### It worked, but some issues found and upgrade is needed.
✅ Fixes
- “Show All” works again (Sidebar + Top bar)
- “Samples → Show All / Hide All” restored
- Hide button collapses figure box (no empty space)
🚀 Upgrades
- Colored section headers (Analysis / Samples)
- Bigger “Show All / Hide All” buttons (not plain text)
- Export buttons (PNG/SVG per figure)
- Cleaner UI hierarchy
- Foundation for multi-select samples (checkbox-ready)

---
# Phasing Panel ver. 7
- "Show All" working for both ISdebar and main panel
## Sidebar.tsx ver. 
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {

  const sortedStages = [...stages].sort();

  return (
    <div className={styles.sidebar}>

      {/* ANALYSIS */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Analysis</h3>

          <div className={styles.actions}>
            <span
              onClick={() => {
                setView("all");
                if (stages.length > 0) {
                  setActiveStage(stages[0]);
                }
              }}
            >
              Show All
            </span>

            <span onClick={() => setView("hide")}>
              Hide All
            </span>
          </div>
        </div>

        <div
          className={`${styles.item} ${view === "register" ? styles.active : ""}`}
          onClick={() => setView("register")}
        >
          Register
        </div>

        <div
          className={`${styles.item} ${view === "read5prime" ? styles.active : ""}`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>
      </div>

      {/* SAMPLES */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Samples</h3>

          {/* ✅ FIX: added Show All / Hide All */}
          <div className={styles.actions}>
            <span
              onClick={() => {
                setView("all"); // show all stages
              }}
            >
              Show All
            </span>

            <span
              onClick={() => {
                setView("hide"); // hide everything
              }}
            >
              Hide All
            </span>
          </div>
        </div>

        {sortedStages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${activeStage === s ? styles.active : ""}`}
            onClick={() => {
              setActiveStage(s);
              setView("all"); // show both plots for selected stage
            }}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}
```
## PhasingPatternPanel.tsx
```sh
import { useState, useEffect } from "react";
import styles from "./PhasingPatternPanel.module.css";

interface Props {
  phasId: string;
  stage: string;
  view: "read5prime" | "register" | "all" | "hide";
  allStages: string[];
  setStage: (s: string) => void;
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function PhasingPatternPanel({
  phasId,
  stage,
  view,
  allStages,
  setStage,
  setView,
}: Props) {

  const [hiddenImages, setHiddenImages] = useState<string[]>([]);

  const base = `${window.location.origin}/phas_plots/organize_phas_images/${phasId}`;

  // ✅ Reset hidden images when switching context
  useEffect(() => {
    setHiddenImages([]);
  }, [stage, view, phasId]);

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

          <button
            className={isHidden ? styles.showBtn : styles.hideBtn}
            onClick={() => toggleImage(key)}
          >
            {isHidden ? "Show" : "Hide"}
          </button>
        </div>

        {!isHidden && (
          <img src={`${base}/${key}.png`} className={styles.image} />
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

      {/* TOP BAR */}
      <div className={styles.topBar}>
        <h2>{phasId}</h2>

        <select value={stage} onChange={(e) => setStage(e.target.value)}>
          {allStages.map((s) => (
            <option key={s} value={s}>
              {s.replaceAll("_", " ")}
            </option>
          ))}
        </select>

        <div className={styles.topActions}>
          {/* ✅ FIX: Proper Show All */}
          <span
            onClick={() => {
              setView("all");
              setHiddenImages([]);
            }}
          >
            Show All
          </span>

          {/* ✅ FIX: Proper Hide All */}
          <span
            onClick={() => {
              setView("hide");
              setHiddenImages(
                allStages.flatMap((s) => [
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

      {/* GRID */}
      <div className={styles.grid}>
        {view !== "hide" && (
          <>
            {/* ✅ FIX: Show ALL stages when view === "all" */}
            {view === "all" ? (
              allStages.map((s) => (
                <div key={s}>
                  <h4>{s.replaceAll("_", " ")}</h4>
                  {renderPair(s)}
                </div>
              ))
            ) : (
              renderPair(stage)
            )}
          </>
        )}
      </div>
    </div>
  );
}

```
---
# Phasing Panel ver. 8. 

## 1. Sidebar.tsx 
```sh
import styles from "./Sidebar.module.css";

interface Props {
  stages: string[];
  activeStage: string;
  setActiveStage: (s: string) => void;
  view: "read5prime" | "register" | "all" | "hide";
  setView: (v: "read5prime" | "register" | "all" | "hide") => void;
}

export default function Sidebar({
  stages,
  activeStage,
  setActiveStage,
  view,
  setView,
}: Props) {

  const sortedStages = [...stages].sort();

  return (
    <div className={styles.sidebar}>

      {/* ANALYSIS */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Analysis</h3>

          <div className={styles.actions}>
            <span
              onClick={() => {
                setView("all"); // show both plots
              }}
            >
              Show All
            </span>

            <span onClick={() => setView("hide")}>
              Hide All
            </span>
          </div>
        </div>

        <div
          className={`${styles.item} ${view === "register" ? styles.active : ""}`}
          onClick={() => setView("register")}
        >
          Register
        </div>

        <div
          className={`${styles.item} ${view === "read5prime" ? styles.active : ""}`}
          onClick={() => setView("read5prime")}
        >
          Read 5′
        </div>
      </div>

      {/* SAMPLES */}
      <div className={styles.box}>
        <div className={styles.headerRow}>
          <h3>Samples</h3>

          <div className={styles.actions}>
            {/* ✅ FIX: Show ALL stages */}
            <span
              onClick={() => {
                setActiveStage(s);
                setView("all");
              }}
            >
              Show All
            </span>

            {/* ✅ FIX: Hide everything */}
            <span
              onClick={() => {
                setView("hide");
              }}
            >
              Hide All
            </span>
          </div>
        </div>

        {sortedStages.map((s) => (
          <div
            key={s}
            className={`${styles.item} ${activeStage === s ? styles.active : ""}`}
            onClick={() => {
              setActiveStage(s);     // ✅ ONLY change stage
              // ❌ DO NOT touch view here
            }}
          >
            {s.replaceAll("_", " ")}
          </div>
        ))}
      </div>
    </div>
  );
}
```
- 

## PhasingPatternPanel.tsx
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

  useEffect(() => {
    if (view === "hide") setView("all");
    setHiddenImages([]);
  }, [stage, phasId]);

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
        {(view === "all" || view === "read5prime") && renderFigure(readKey, "Read 5′")}
        {(view === "all" || view === "register") && renderFigure(regKey, "Register")}
      </div>
    );
  };

  return (
    <div className={styles.container}>
      <div className={styles.topBar}>
        <h2>{phasId}</h2>

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

## App.tsx
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
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

  // Load PHAS
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) =>
      r.phas_id.toLowerCase().includes(q)
    );

    setFilteredRecords(filtered);

    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // Load stages
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);

        if (stages.length > 0) {
          setActiveStage("ALL");
        }
      });
  }, [selectedPhas]);

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={setSelectedPhas}
          loading={loading}
        />

        {selectedPhas && (
          <div className="mainContent">
            <PhasingPatternPanel
              phasId={selectedPhas.phas_id}
              stage={activeStage}
              view={view}
              allStages={availableStages}
              setStage={setActiveStage}
              setView={setView}
            />
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
## PhasingPatternPanel.module.css

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
  gap: 10px;
}
.topBar h2 {
  margin-right: auto;
  font-size: 16px;
}
.topBar select {
  font-size: 12px;
}
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
## PHASTable.tsx
```sh
import { useState, useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";

type SortOrder = "asc" | "desc";

interface PHASTableProps {
  records: PHASLocus[];
  selectedId: string | null;
  onSelect: (record: PHASLocus) => void;
  loading?: boolean;
}

export function PHASTable({
  records,
  selectedId,
  onSelect,
  loading,
}: PHASTableProps) {
  const [sortField, setSortField] = useState<string>("PHASID");
  const [sortOrder, setSortOrder] = useState<SortOrder>("asc");

  // 🔥 Dynamically get columns from TSV
  const columns = useMemo(() => {
    return records.length > 0 ? Object.keys(records[0]) : [];
  }, [records]);

  // 🔥 Sorting (generic)
  const sortedRecords = useMemo(() => {
    return [...records].sort((a, b) => {
      const valA = a[sortField];
      const valB = b[sortField];

      if (valA == null) return 1;
      if (valB == null) return -1;

      if (valA < valB) return sortOrder === "asc" ? -1 : 1;
      if (valA > valB) return sortOrder === "asc" ? 1 : -1;
      return 0;
    });
  }, [records, sortField, sortOrder]);

  const handleSort = (field: string) => {
    if (field === sortField) {
      setSortOrder(sortOrder === "asc" ? "desc" : "asc");
    } else {
      setSortField(field);
      setSortOrder("asc");
    }
  };

  const renderSortIndicator = (field: string) => {
    if (field !== sortField) return "";
    return sortOrder === "asc" ? " ▲" : " ▼";
  };

  return (
    <div className={styles.container}>
      <div className={styles.header}>
        <span className={styles.resultCount}>
          {records.length} PHAS loci
        </span>
      </div>

      <div className={styles.tableWrapper}>
        {loading ? (
          <div className={styles.loading}>Loading...</div>
        ) : (
          <table className={styles.table}>
            <thead>
              <tr>
                {columns.map((col) => (
                  <th
                    key={col}
                    className={styles.sortable}
                    onClick={() => handleSort(col)}
                  >
                    {formatHeader(col)}
                    {renderSortIndicator(col)}
                  </th>
                ))}
              </tr>
            </thead>

            <tbody>
              {sortedRecords.map((r, i) => (
                <tr
                  key={i}
                  onClick={() => onSelect(r)}
                  className={`${styles.row} ${
                    selectedId === String(r["PHASID"])
                      ? styles.selected
                      : ""
                  }`}
                >
                  {columns.map((col) => (
                    <td key={col} className={getCellClass(col)}>
                      {formatValue(col, r[col])}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
}

/* =========================
   🔧 Helpers (important)
   ========================= */

// Better column labels
function formatHeader(col: string) {
  return col
    .replace(/_/g, " ")
    .replace(/\b\w/g, (c) => c.toUpperCase());
}

// Smart formatting for your PHAS data
function formatValue(col: string, value: string | number) {
  if (value == null) return "-";

  // P-value formatting
  if (col.toLowerCase().includes("pvalue") && typeof value === "number") {
    return value.toExponential(2);
  }

  // Numbers
  if (typeof value === "number") {
    return Number.isInteger(value) ? value : value.toFixed(2);
  }

  return value;
}

// Apply your EXISTING CSS styles dynamically
function getCellClass(col: string) {
  const key = col.toLowerCase();

  if (key.includes("phasid")) return styles.phasId;
  if (key.includes("chr")) return styles.chromosome;
  if (key.includes("start") || key.includes("end") || key.includes("len"))
    return styles.numeric;
  if (key.includes("phase")) return styles.phase;
  if (key.includes("pvalue")) return styles.pvalue;
  if (key.includes("abudance") || key.includes("abundance"))
    return styles.abundance;
  if (key.includes("note")) return styles.note;

  return "";
}
```
---
# Phasing Panel ver. 9
- include functions for clicking "Best Sample" cell

## 1. src/utils/smapleMap.ts
- create `smapleMap.ts` file to create Samples mapping. 

```sh
# src/utils/smapleMap.ts
export const SAMPLE_MAP: Record<string, string> = {
  N002: "OkayamaE1",
  N003: "OkayamaE5",
  N004: "OkayamaE10",
  N005: "OkayamaE15",
  N006: "OkayamaE20",
  N007: "Okayama_Larva_Unfed",
  N133: "Okayama_Larva_fed",
  N008: "Okayama_Nymph_Unfed",
  N135: "Okayama_Nymph_fed",
  N009: "Okayama_Adult_Unfed",
  N052: "Okayama_Adult_Fed",

  N010: "OitaE1",
  N011: "OitaE5",
  N012: "OitaE10",
  N013: "OitaE15",
  N014: "OitaE20",
  N015: "Oita_Larva_Unfed",
  N134: "Oita_Larva_fed",
  N016: "Oita_Nymph_Unfed",
  N136: "Oita_Nymph_fed",
  N017: "Oita_Adultmale_Unfed",
  N137: "Oita_Adultmale_fed",
  N018: "Oita_Adultfemale_Unfed",
  N138: "Oita_Adultfemale_fed",
};

// Extract Nxxx
export function extractSampleId(text: string): string | null {
  const match = text.match(/N\d+/);
  return match ? match[0] : null;
}

// Convert to stage
export function getStageFromSample(text: string): string | null {
  const id = extractSampleId(text);
  if (!id) return null;
  return SAMPLE_MAP[id] || null;
}
```
- ## App.tsx ver.
```sh
// App.tsx
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);

  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");

  const [view, setView] = useState<"read5prime" | "register" | "all" | "hide">("all");

  // Load PHAS records
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find((r) => r.phas_id === "PHAS22-1");
        if (defaultPhas) setSelectedPhas(defaultPhas);
      })
      .finally(() => setLoading(false));
  }, []);

  // Search / Filter
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();
    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }
    const filtered = phasRecords.filter((r) => r.phas_id.toLowerCase().includes(q));
    setFilteredRecords(filtered);

    const exact = phasRecords.find((r) => r.phas_id.toLowerCase() === q);
    if (exact) setSelectedPhas(exact);
  }, [searchQuery, phasRecords]);

  // Load stages for selected PHAS
  useEffect(() => {
    if (!selectedPhas) return;
    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);
        if (stages.length > 0) setActiveStage("ALL");
      });
  }, [selectedPhas]);

  // Handler for row click or Best Sample click
  const handleSelectPhasAndStage = (phas: PHASLocus, stage: string | null = null) => {
    setSelectedPhas(phas);
    setActiveStage(stage || "ALL");
    setView("all");
    setSearchQuery(phas.phas_id); // update search bar to selected PHAS
  };

  return (
    <div className="app">
      <Header
        currentState={{}}
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID..."
          options={phasRecords.map((r) => r.phas_id)} // Pass PHAS IDs for dropdown
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelectPhasAndStage={handleSelectPhasAndStage}
          loading={loading}
        />

        {selectedPhas && (
          <div className="mainContent">
            <PhasingPatternPanel
              phasId={selectedPhas.phas_id}
              stage={activeStage}
              view={view}
              allStages={availableStages}
              setStage={setActiveStage}
              setView={setView}
            />
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```

- ## PhasingPatternPanel.tsx
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
        {(view === "all" || view === "read5prime") && renderFigure(readKey, "Read 5′")}
        {(view === "all" || view === "register") && renderFigure(regKey, "Register")}
      </div>
    );
  };

  return (
    <div className={styles.container}>
      <div className={styles.topBar}>
        <h2>{phasId}</h2>

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
- ## PHASTable ver.

```sh
import { useState, useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";
import { getStageFromSample } from "../../utils/sampleMap";

type SortOrder = "asc" | "desc";

interface PHASTableProps {
  records: PHASLocus[];
  selectedId: string | null;
  onSelectPhasAndStage: (phas: PHASLocus, stage: string | null) => void;
  loading?: boolean;
}

export function PHASTable({
  records,
  selectedId,
  onSelectPhasAndStage,
  loading,
}: PHASTableProps) {
  const [sortField, setSortField] = useState<string>("PHASID");
  const [sortOrder, setSortOrder] = useState<SortOrder>("asc");

  const columns = useMemo(
    () => (records.length > 0 ? Object.keys(records[0]) : []),
    [records]
  );

  const sortedRecords = useMemo(() => {
    return [...records].sort((a, b) => {
      const valA = a[sortField];
      const valB = b[sortField];

      if (valA == null) return 1;
      if (valB == null) return -1;

      if (valA < valB) return sortOrder === "asc" ? -1 : 1;
      if (valA > valB) return sortOrder === "asc" ? 1 : -1;
      return 0;
    });
  }, [records, sortField, sortOrder]);

  const handleSort = (field: string) => {
    if (field === sortField) setSortOrder(sortOrder === "asc" ? "desc" : "asc");
    else {
      setSortField(field);
      setSortOrder("asc");
    }
  };

  const renderSortIndicator = (field: string) =>
    field !== sortField ? "" : sortOrder === "asc" ? " ▲" : " ▼";

  return (
    <div className={styles.container}>
      <div className={styles.header}>
        <span className={styles.resultCount}>{records.length} PHAS loci</span>
      </div>

      <div className={styles.tableWrapper}>
        {loading ? (
          <div className={styles.loading}>Loading...</div>
        ) : (
          <table className={styles.table}>
            <thead>
              <tr>
                {columns.map((col) => (
                  <th
                    key={col}
                    className={styles.sortable}
                    onClick={() => handleSort(col)}
                  >
                    {formatHeader(col)}
                    {renderSortIndicator(col)}
                  </th>
                ))}
              </tr>
            </thead>

            <tbody>
              {sortedRecords.map((r, i) => (
                <tr
                  key={i}
                  className={`${styles.row} ${
                    selectedId === String(r["PHASID"]) ? styles.selected : ""
                  }`}
                  onClick={() => onSelectPhasAndStage(r, null)} // Row click selects PHASID
                >
                  {columns.map((col) => {
                    const value = r[col];
                    const isBestSample =
                      col.toLowerCase().includes("best") &&
                      col.toLowerCase().includes("sample");

                    if (isBestSample && typeof value === "string") {
                      return (
                        <td
                          key={col}
                          className={getCellClass(col)}
                          title="Double click to show best sample figures"
                        >
                          <span
                            style={{ color: "#007bff", cursor: "pointer" }}
                            onClick={(e) => {
                              e.stopPropagation();
                              const stage = getStageFromSample(value);
                              if (stage)
                                onSelectPhasAndStage(r, stage); // Single click works
                            }}
                          >
                            {value}
                          </span>
                        </td>
                      );
                    }

                    return (
                      <td key={col} className={getCellClass(col)}>
                        {formatValue(col, value)}
                      </td>
                    );
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
}

/* Helpers */
function formatHeader(col: string) {
  return col.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase());
}

function formatValue(col: string, value: any) {
  if (value == null) return "-";
  if (col.toLowerCase().includes("pvalue") && typeof value === "number")
    return value.toExponential(2);
  if (typeof value === "number")
    return Number.isInteger(value) ? value : value.toFixed(2);
  return value;
}

function getCellClass(col: string) {
  const key = col.toLowerCase();
  if (key.includes("phasid")) return styles.phasId;
  if (key.includes("chr")) return styles.chromosome;
  if (key.includes("start") || key.includes("end") || key.includes("len"))
    return styles.numeric;
  if (key.includes("phase")) return styles.phase;
  if (key.includes("pvalue")) return styles.pvalue;
  if (key.includes("abudance") || key.includes("abundance"))
    return styles.abundance;
  if (key.includes("note")) return styles.note;
  return "";
}
```
---
#### Update/Upgrade SearchBox

- ## Searchbox.tsx ver.

```sh
// SearchBox.tsx
import { useState, useEffect, useRef } from "react";
import styles from "./SearchBox.module.css";

interface SearchBoxProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  options?: string[]; // list of PHAS IDs
}

export function SearchBox({ value, onChange, placeholder, options = [] }: SearchBoxProps): JSX.Element {
  const [showDropdown, setShowDropdown] = useState(false);
  const [filteredOptions, setFilteredOptions] = useState<string[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);

  // Filter options whenever value changes
  useEffect(() => {
    const filtered = options.filter((opt) =>
      opt.toLowerCase().includes(value.toLowerCase())
    );
    setFilteredOptions(filtered);
  }, [value, options]);

  // Close dropdown when clicking outside
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (containerRef.current && !containerRef.current.contains(event.target as Node)) {
        setShowDropdown(false);
      }
    };
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  const handleSelect = (option: string) => {
    onChange(option);
    setShowDropdown(false);
  };

  return (
    <div
      className={styles.container}
      ref={containerRef}
      onFocus={() => setShowDropdown(true)}
    >
      <div className={styles.searchIcon}>
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
          <circle cx="11" cy="11" r="8" />
          <path d="M21 21l-4.35-4.35" />
        </svg>
      </div>

      <input
        type="text"
        className={styles.input}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder || "Search by PHAS ID..."}
        onFocus={() => setShowDropdown(true)}
      />

      {value && (
        <button className={styles.clearButton} onClick={() => onChange('')}>
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
            <path d="M18 6L6 18M6 6l12 12" />
          </svg>
        </button>
      )}

      {showDropdown && filteredOptions.length > 0 && (
        <div className={styles.dropdown}>
          {filteredOptions.map((opt) => (
            <div
              key={opt}
              className={styles.dropdownItem}
              onClick={() => handleSelect(opt)}
            >
              {opt}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

## SearchBox.module.css
```sh
/* PHASTable.module.css */

.container {
  margin: 0 24px 16px;
  border: 1px solid #e3e6ea;
  border-radius: 10px;
  overflow: hidden;
  background: white;
}

/* HEADER BAR */
.header {
  padding: 10px 16px;
  background: #f6f8fa;
  border-bottom: 1px solid #e3e6ea;
}

.resultCount {
  font-size: 13px;
  color: #6c7a89;
}

/* TABLE WRAPPER */
.tableWrapper {
  max-height: 300px;
  overflow-y: auto;
}

/* TABLE */
.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

/* STICKY HEADER */
.table thead {
  position: sticky;
  top: 0;
  background: #f6f8fa;
  z-index: 2;
}

/* HEADER CELLS */
.table th {
  padding: 10px 14px;
  text-align: left;
  font-weight: 600;
  color: #2c3e50;
  border-bottom: 2px solid #e3e6ea;
  font-size: 13px;
}

/* DATA CELLS */
.table td {
  padding: 10px 14px;
  border-bottom: 1px solid #f0f2f5;
}

/* ROW BEHAVIOR */
.row {
  cursor: pointer;
  transition: background 0.15s ease;
}

.row:hover {
  background: #f4f8fb;
}

/* SELECTED ROW */
.selected {
  background-color: #d1e7ff; /* blue highlight for selected PHAS row */
  border-left: 4px solid #3498db;
}

/* PHAS ID */
.phasId {
  font-family: monospace;
  font-weight: 600;
  color: #2980b9;
}

/* GENOMIC LOCATION */
.chromosome {
  color: #2c3e50;
  font-weight: 500;
}

.locus {
  font-family: monospace;
  font-size: 12px;
  color: #7f8c8d;
}

/* NUMERIC COLUMNS */
.numeric {
  font-family: monospace;
  text-align: right;
}

/* PHASE */
.phase {
  font-weight: 600;
  color: #8e44ad;
}

/* PVALUE */
.pvalue {
  font-family: monospace;
  color: #c0392b;
}

/* ABUNDANCE */
.abundance {
  font-family: monospace;
  color: #27ae60;
}

/* NOTE (confidence) */
.note {
  font-size: 12px;
  font-weight: 500;
  color: #34495e;
}

/* SORTABLE HEADERS */
.sortable {
  cursor: pointer;
  user-select: none;
}

.sortable:hover {
  background: #eef3f8;
}

/* LOADING STATE */
.loading {
  padding: 24px;
  text-align: center;
  color: #7f8c8d;
  font-size: 14px;
}

```

- ## App.module.css
```sh
.layout {
  display: flex;
  margin-top: 20px;
}

.mainContent {
  flex: 1;
  padding: 20px;
}
```
- #### App.css
```sh
/* Global Styles */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
    sans-serif;
  background: #f5f6fa;
  color: #2c3e50;
  line-height: 1.5;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.main {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding-bottom: 100px; /* Space for tab panel */
}

.error-message {
  margin: 0 24px 16px;
  padding: 12px 16px;
  background: #fdecea;
  color: #c0392b;
  border-radius: 8px;
  font-size: 14px;
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: #f1f2f6;
}

::-webkit-scrollbar-thumb {
  background: #bdc3c7;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #95a5a6;
}

/* IGV.js custom styling */
.igv-container {
  font-family: inherit !important;
}

.igv-navbar {
  background: #f8f9fa !important;
  border-bottom: 1px solid #ecf0f1 !important;
}

.layout {
  display: flex;
  margin-top: 20px;
}

.mainContent {
  flex: 1;
  padding: 20px;
}
```

---
# Upgrade/Modify Header Panel 
- ## Header.tsx
```sh
import { useState } from 'react';
import styles from './Header.module.css';

export function Header(): JSX.Element {
  const [copied, setCopied] = useState(false);

  // Share current URL to clipboard
  const handleShareClick = async () => {
    try {
      await navigator.clipboard.writeText(window.location.href);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch (err) {
      console.error('Clipboard write failed:', err);
      alert('Failed to copy URL. Please copy manually.');
    }
  };

  return (
    <header className={styles.header}>
      <div className={styles.logo}>
        <h1 className={styles.title}>PHASER</h1>
        <span className={styles.subtitle}>PHAS Evaluation Resource</span>
      </div>

      <div className={styles.actions}>
        <button className={styles.shareButton} onClick={handleShareClick}>
          {copied ? 'Copied!' : 'Share URL'}
        </button>
      </div>
    </header>
  );
}
```
## Header.module.css
```sh
/* ---------------- Header styling ---------------- */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 0 20px;          /* Small space on left and right */
  padding: 8px 20px;       /* Reduced vertical and horizontal padding */
  background-color: #2c2c2c; /* Dark gray header */
  border-bottom: 1px solid #1a1a1a;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  border-radius: 0;        /* Straight edges */
}

.logo {
  display: flex;
  flex-direction: column;
}

.title {
  font-size: 28px;       /* Slightly smaller than before */
  font-weight: 800;
  color: #f5f5f5;        /* Light gray text */
  line-height: 1.1;      /* Reduce extra spacing */
}

.subtitle {
  font-size: 14px;       /* Slightly smaller subtitle */
  font-weight: 600;
  color: #cccccc;        /* Medium light gray */
  margin-top: 2px;
  line-height: 1.1;
}

/* Header action buttons */
.actions {
  display: flex;
  gap: 12px;
}

.shareButton {
  padding: 6px 14px;     /* Slightly smaller button to match header */
  border-radius: 6px;
  border: none;
  cursor: pointer;
  background-color: #555555;
  color: #f5f5f5;
  font-weight: 600;
  font-size: 14px;
  transition: background-color 0.2s, transform 0.1s;
}

.shareButton:hover {
  background-color: #444444;
  transform: translateY(-1px);
}

.shareButton:active {
  transform: translateY(0);
}
```