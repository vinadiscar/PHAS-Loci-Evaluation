03.20.2027 
# PHAS Evaluation Resource Tool
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHAS"

#NOTE: PHAS was renamed to PHAS_2. Another directory (PHASER) will be used.
```

# Phase 2: Set-up IGV
- This Phase set-ups IGV for the interface.
## Overview:
```sh
BAM (raw data)
   ↓ (offline processing)
BigWig (.bw)
   ↓
FastAPI (static server)
   ↓
React + IGV.js
   ↓
Interactive visualization

Step 1. Prepare Data (Convert Bam to bigwig files, copy annotation file)
Step 2. Backend (FastAPI)- Modify `backend/main.py`
Step 3. Install IGV
Step 4. IGV Viewer (FINAL VERSION) `IGVViewer.tsx`
Step 5. Track Controls `TrackPanel.tsx`
Step 6. PHAS Table `PHASTable.tsx`
Step 7. Main App `App.tsx`
Step 8. Run Everything (Backend and Frontend)
```
## 1. Prepare Data (Convert Bam to bigwig Files)
- #### Get Chrom. sizes
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend"

awk '{for(i=1;i<=NF;i++){if($i ~ /^Len=/){split($i,a,"="); print $1"\t"a[2]}}}' \
/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/genome/GWHAMMI00000000.genome.fasta.fai \
> "/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/GWHAMMI00000000.genome.chrom.sizes"

#Check output content
head GWHAMMI00000000.genome.chrom.sizes

# confirm it's TAB-separated, not spaces.
cat -vet GWHAMMI00000000.genome.chrom.sizes | head
# GWHAMMI00000001^I348261740$ -> TAB-separated 
# ^I = TAB ✔️
```
- #### Convert BAM files to Bigwig Files
> - Use streaming to avoid storing BAM files in the directory
> - NOTE: Pipelines should be streaming, minimal storage, and reproducible. 


### Install missing tools
- samtools
- bedtools
- bedGraphToBigWig

```sh
conda create -n bamTobw_pipeline_env -c bioconda -c conda-forge \
  samtools bedtools ucsc-bedgraphtobigwig
```
### Write Scipt `bam_to_bw.sh`


```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend"
mkdir -p scripts

cd scripts
vi bam_to_bw.sh
```

```sh
#!/bin/bash

set -e  # stop on error

#!/bin/bash

set -euo pipefail

source ~/miniconda3/etc/profile.d/conda.sh
conda activate bamTobw_pipeline_env

INPUT_DIR="/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam"
OUTPUT_DIR="/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/bw"
GENOME_SIZE="/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/GWHAMMI00000000.genome.chrom.sizes"

mkdir -p "$OUTPUT_DIR"

for bam in "${INPUT_DIR}"/*.bam
do
  base=$(basename "$bam" .bam)

  echo "Processing $base"

  # PLUS strand
  samtools view -b -F 16 "$bam" \
    | bedtools genomecov -ibam - -bg \
    > "${OUTPUT_DIR}/${base}.plus.bedgraph"

  # MINUS strand
  samtools view -b -f 16 "$bam" \
    | bedtools genomecov -ibam - -bg \
    > "${OUTPUT_DIR}/${base}.minus.bedgraph"

  # Convert to BigWig
  bedGraphToBigWig "${OUTPUT_DIR}/${base}.plus.bedgraph" "$GENOME_SIZE" "${OUTPUT_DIR}/${base}.plus.bw"
  bedGraphToBigWig "${OUTPUT_DIR}/${base}.minus.bedgraph" "$GENOME_SIZE" "${OUTPUT_DIR}/${base}.minus.bw"

done

echo "All done!"
```
- ##### Run `bam_to_bw.sh
```sh
chmod +x bam_to_bw.sh
./bam_to_bw.sh
```
- ###### remove intermediate bedgraph files 

```sh
rm "/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/bw"/*.bedgraph
```
### Copy annotation file `merged.PHAS22.gff3`

```sh
cp "/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/sRNAminer/merged.PHAS22.gff3" "/Volumes/Install macOS Mojave/Vina/PHAS/backend/static"
```

## Step 2. Backend (FastAPI)- Modify `backend/main.py`


###### What changed:
- Static file serving (CORE FEATURE)
    - This is what allows IGV to load
- Dynamic track API

Updated Version `main.py`:
```sh
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
import pandas as pd
import os

app = FastAPI()

# ✅ CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # for development
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# ✅ Serve static files (BIGWIG, genome, etc.)
app.mount("/static", StaticFiles(directory="static"), name="static")

# ✅ Load PHAS loci table
phas_df = pd.read_csv("data/phas_loci.tsv", sep="\t")

@app.get("/")
def root():
    return {"message": "PHAS backend running"}

@app.get("/api/phas/list")
def get_phas_list():
    return phas_df.to_dict(orient="records")

# ✅ NEW: Return available BigWig tracks dynamically
@app.get("/api/tracks")
def get_tracks():
    bw_dir = "static/bw"
    tracks = []

    if not os.path.exists(bw_dir):
        return {"tracks": []}

    for file in os.listdir(bw_dir):
        if file.endswith(".bw"):
            track_type = "plus" if "plus" in file else "minus"

            tracks.append({
                "name": file,
                "type": "wig",
                "format": "bigwig",
                "url": f"http://127.0.0.1:8001/static/bw/{file}",
                "color": "red" if track_type == "plus" else "blue"
            })

    return {"tracks": tracks}
```
## Step 3. Install IGV
- install IGV.js
```sh
cd frontend
npm install igv

# Check if IGV is installed 
npm list igv

# frontend@0.0.0 /Volumes/Install macOS Mojave/Vina/PHAS/frontend
└── igv@3.8.0
```

## Step 4 Modify `IGVViewer.tsx` ver. 1 (for IGV)
- Things to improve:
Dynamic tracks from backend (/api/tracks)
Avoid reloading IGV unnecessarily
Better genome handling (no more hardcoded hg38)
Cleaner track updates (no flicker)
Scalable for many samples

```sh
cd frontend/src/components
vi IGVViewer.tsx
```
- ###### IGVViewer.tsx

```sh
import { useEffect, useRef, useState } from "react"
import igv from "igv"

interface Track {
  name: string
  url: string
  color: string
}

export default function IGVViewer({
  locus,
}: {
  locus: string
}) {
  const igvContainer = useRef<HTMLDivElement | null>(null)
  const igvBrowser = useRef<any>(null)

  const [tracks, setTracks] = useState<Track[]>([])

  // ✅ 1. Fetch tracks from backend
  useEffect(() => {
    fetch("http://127.0.0.1:8001/api/tracks")
      .then((res) => res.json())
      .then((data) => {
        setTracks(data.tracks || [])
      })
      .catch((err) => console.error("Error loading tracks:", err))
  }, [])

  // ✅ 2. Initialize IGV ONLY ONCE
  useEffect(() => {
    if (!igvContainer.current || igvBrowser.current) return

    igv.createBrowser(igvContainer.current, {
      genome: "hg38", // 🔥 TEMP → replace later with your genome
      locus: locus || "chr1:1-10000",
    }).then((browser) => {
      igvBrowser.current = browser
    })
  }, [])

  // ✅ 3. Jump to selected PHAS locus
  useEffect(() => {
    if (igvBrowser.current && locus) {
      igvBrowser.current.search(locus)
    }
  }, [locus])

  // ✅ 4. Load tracks dynamically (optimized)
  useEffect(() => {
    if (!igvBrowser.current || tracks.length === 0) return

    const browser = igvBrowser.current

    // Remove existing tracks
    browser.removeAllTracks()

    // Load new tracks
    tracks.forEach((track) => {
      browser.loadTrack({
        name: track.name,
        type: "wig",
        format: "bigwig",
        url: track.url, // already full URL from backend
        color: track.color,
        height: 50,
      })
    })
  }, [tracks])

  return (
    <div
      ref={igvContainer}
      style={{
        height: "500px",
        border: "1px solid #ccc",
        borderRadius: "8px",
      }}
    />
  )
}
```

# Test IGV ver. 1 
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
## Check Console Error

![alt text](image-4.png)

- From the screenshot, IGV is NOT renderring at all.
- No IGV error which means IGV was NEVER properly initialized OR the container is not rendering.

Most causes:
> 1. Missing IGV CSS
-IGV needs its CSS to display. Without it → blank viewer (exactly your case)
>2. IGV might be initializing BEFORE tracks load

## Solve ver. 1 
In `frontend/src/main.tsx`, add the following line `import 'igv/dist/igv.css'`
```sh
cd frontend/src
vi main.tsx
```
 ### 1. Modify `main.tsx`
 ```sh
 import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import 'igv/dist/igv.css'   // ✅ ADD THIS LINE (VERY IMPORTANT)
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
 ```
 # Test IGV ver. 1 
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
## Check Console Error 

![alt text](image-5.png)

- ###### Problem: `Failed to resolve import "igv/dist/igv.css"`
    > This means The igv package you installed does NOT include that CSS path

## Solve ver. 2 
#### Use CDN CSS

- REMOVE this in `main.tsx` (frontend/src/main.tsx)
```sh
import "igv/dist/igv.css"
```

- Modify `frontend/index.html` and 
Add inside `<head>`
```sh
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/igv@2.15.4/dist/igv.css"
/>
```
- ### modify `index.html`
```sh
cd frontend 
vi index.html
```
index.html:
```sh
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>frontend</title>

    <!-- ✅ ADD THIS LINE (IGV CSS via CDN) -->
    <link
      rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/igv@2.15.4/dist/igv.css"
    />

  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```
 # Test IGV ver. 2
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```

## Check Console Error
![alt text](image-6.png)

- ###### Problem: Failed to load resource: 404 https://cdn.jsdelivr.net/npm/igv@2.15.4/dist/igv.css
> -which means exact CSS file does NOT exist at that path (File path is wrong), so IGV still has no styling → invisible viewer

## Solve ver. 3

- ### FIX (Correct CDN URL) in `index.html` (frontend)
Add:
```sh
<head>
  ...
  <link
    rel="stylesheet"
    href="https://cdn.jsdelivr.net/npm/igv@2.14.1/dist/igv.min.css"
  />
</head>
```
```sh
cd frontend 
vi index.html
```
- index.html:
```sh
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- ✅ CORRECT IGV CSS -->
    <link
      rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/igv@2.15.5/dist/igv.min.css"
    />

    <title>PHAS Viewer</title>
  </head>

  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```
 # Test IGV ver. 3
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
## Check Console Error

![alt text](image-7.png)

- `igv@2.15.5/dist/igv.min.css → 404` even FAILED because:
    - IGV npm package ≠ CDN structure
    - CSS is not consistently published to CDN
    - So CDN approach is unreliable for IGV

## Solve ver 4 

- Use IGV from npm (NOT CDN)
I already installed IGV already (`npm install igv`), so we'll use the local package properly. 

#### Solve 1: Fix `main.tsx` (frontend/src)

```sh
import { StrictMode } from "react"
import { createRoot } from "react-dom/client"

// ✅ Import IGV CSS from node_modules (CORRECT PATH)
import "igv/dist/igv.css"

import "./index.css"
import App from "./App.tsx"

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```
#### Solve 2: REMOVE CDN completely in index.html (frontend)
Your index.html should go back to CLEAN version:
- no `<link >` for IGV anymore
```sh
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>PHAS Viewer</title>
  </head>

  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```
 # Test IGV ver. 4
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
- ### ERROR:
```sh
2:32:23 PM [vite] Internal server error: Failed to resolve import "igv/dist/igv.css" from "src/main.tsx". Does the file exist?
  Plugin: vite:import-analysis
  File: /Volumes/Install macOS Mojave/Vina/PHAS/frontend/src/main.tsx:5:7
  2  |  import { StrictMode } from "react";
  3  |  import { createRoot } from "react-dom/client";
  4  |  import "igv/dist/igv.css";
     |          ^
  5  |  import "./index.css";
  6  |  import App from "./App.tsx";
      at TransformPluginContext._formatLog (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:28999:43)
      at TransformPluginContext.error (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:28996:14)
      at normalizeUrl (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:27119:18)
      at process.processTicksAndRejections (node:internal/process/task_queues:105:5)
      at async file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:27177:32
      at async Promise.all (index 3)
      at async TransformPluginContext.transform (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:27145:4)
      at async EnvironmentPluginContainer.transform (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:28797:14)
      at async loadAndTransform (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:22670:26)
      at async viteTransformMiddleware (file:///Volumes/Install%20macOS%20Mojave/Vina/PHAS/frontend/node_modules/vite/dist/node/chunks/config.js:24542:20)
```
- Failed to resolve import "igv/dist/igv.css"
- Your installed igv package does NOT contain that CSS file.
- import `"igv/dist/igv.css"` is NOT valid for your installed version

#### Solve 4: 
##### 1. Remove this Line from `main.tsx` (frontend/src)
`import "igv/dist/igv.css"`
##### 2. Restart frontend
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
## Error: Still no IGV appeared

###### Possible cause: IGV is not being rendered in your React component tree
## Solve 5:
##### Check App.tsx if IGV viewer is imported then update it and add this missing line: `import IGVViewer from "./components/IGVViewer`

- Update `App.tsx`: (frontend/src/App.tsx)

NOTE: There were two App.tsx saved, in (frontend/src and froentend/src/components). Delete the one from frontend/src/component.

```sh
cd frontend/src
vi App.tsx
```
- App.tsx:
```sh
import "./App.css"
import PHASTable from "./components/PHASTable"
import IGVViewer from "./components/IGVViewer"
import { useState } from "react"

function App() {
  const [locus, setLocus] = useState("chr1:1-10000")

  return (
    <div className="main">
      <h1>PHASER</h1>
      <p>Interactive PHAS Evaluation Platform</p>

      {/* Search */}
      <div className="panel">
        <h2>Search PHAS Loci</h2>
        <input
          type="text"
          placeholder="chr1:1000-2000"
          value={locus}
          onChange={(e) => setLocus(e.target.value)}
          style={{ width: "100%", padding: "10px" }}
        />
      </div>

      {/* Table */}
      <div className="panel table-panel">
        <h2>PHAS Loci Table</h2>
        <PHASTable />
      </div>

      {/* Genome Viewer */}
      <div className="panel">
        <h2>Genome Viewer</h2>

        {/* ✅ THIS IS THE KEY */}
        <IGVViewer locus={locus} />
      </div>
    </div>
  )
}

export default App
```

 # Test IGV ver. 5
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
- ###### IGV APPEARED! 
---

# Check if tracks were Loaded

### Console Problem: 
>Failed to Load resource: could not connect to the server
> >Error loading tracks: - TypeError: Load failed

###### Explanation:
- Backend is running
- Frontend can connect
- ❌ BUT backend is returning empty tracks

## Solve: 

### Modify main.py (backend)
```sh
#Change:
"url": f"http://127.0.0.1:8001/static/bw/{file}",
#to:
"url": f"http://163.221.246.151:8001/static/bw/{file}",
```

### Modify IGVViewer.tsx (frontend/src/componenets)

```sh
#Change
fetch("http://127.0.0.1:8001/api/tracks")
#to
fetch("http://163.221.246.151:8001/api/tracks")
```
 # Test IGV ver. 6
Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```

## Connection Solved!

#### Concern: 

![alt text](image-8.png)
- Although connection was solved, the uploading of bigwig files seem to be slow. 
- UI is messy
- Putting styles directly inside .tsx or global CSS which leads to messy UI
   and hard-to-debug layout issues

- ##### Solution: 
    - build an clean, scalable, and stable IGV system
    - Create both IGVViewer.tsx and IGVViewer.module.css

---
# Build Stable IGV 
- build an clean, scalable, and stable IGV system
- Create both IGVViewer.tsx and IGVViewer.module.css files
    - IGVViewer.module.css (Styling) file controls layout (height, width), spacing, borders, and UI appearance (What does this component look like?)
    - IGVViewer.tsx (logic and structure)-> this is the main React component whihc handles IGV initialization (igv.createBrowser), reacting to selectedPHAS, loading tracks, rendering the container <div> (what does this component do?)

---
---
03.30.2026
###### New strategy for Mock: setting up foundation
# PHASE 1a: Establsih Connection
- #### Rename old PHAS folder to PHAS_2 and use new, clean folder "PHASER"
```sh
mv PHAS PHAS_2 
```
## 1. Copy EEVEE Folder to PHASER for mock
```sh 
cp -r "/Volumes/Install macOS Mojave/Ogawa/EEVEE" "/Volumes/Install macOS Mojave/Vina/PHASER"
```
## 2. Backend (FastAPI) — port 8001
- In `backend/app/main.py`, change 8000 to 8001

## 3. Frontend (Vite) — port 5174
- In `frontend/vite.config.ts`:
change port `5173` to `5174` and `8000` to `8001`

## 4. Run Backend and Frontend
- #### Run backend
```sh
cd PHASER/backend
uvicorn main:app --reload --host 0.0.0.0 --port 8001

```
- #### Run frontend
    - In PHASER/frontend:
    - remove `rm -rf node_modules` and `rm package-lock.json`
    - Instal npm: `npm install`

```sh
cd PHASER/frontend
npm run dev -- --host --port 5174
```
- #### Open in server
```sh
http://163.221.246.151:5174
```
- ### Connection worked!

# Phase 1.B: Create PHAS Table

```sh
frontend/src/components/PHASTable/PHASTable.tsx
```
#### 1. Create PHASTable.tsx

```sh
import styles from './PHASTable.module.css';

export type PHASRecord = {
  phas_id: string;
  locus: string;
};

interface PHASTableProps {
  records: PHASRecord[];
  selectedId: string | null;
  onSelect: (record: PHASRecord) => void;
}

export function PHASTable({ records, selectedId, onSelect }: PHASTableProps) {
  return (
    <div className={styles.container}>
      <div className={styles.header}>
        <span>{records.length} PHAS loci</span>
      </div>

      <div className={styles.tableWrapper}>
        <table className={styles.table}>
          <thead>
            <tr>
              <th>PHAS ID</th>
              <th>Locus</th>
            </tr>
          </thead>
          <tbody>
            {records.map((record) => (
              <tr
                key={record.phas_id}
                className={selectedId === record.phas_id ? styles.selected : ''}
                onClick={() => onSelect(record)}
              >
                <td>{record.phas_id}</td>
                <td>{record.locus}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```

#### 2. Add PHAS Data 

```sh
frontend/src/data/phasData.ts

cd frontend/src
mkdir data
vi phasData.ts
```
```sh
export const phasData = [
  {
    phas_id: "PHAS22-1",
    locus: "GWHAMMI00000001:14520462-14521176"
  },
  {
    phas_id: "PHAS22-2",
    locus: "GWHAMMI00000001:23591960-23592299"
  }
];
```

#### 3. Modify App.tsx
- simplified version for Foundation 
```sh
frontend/src/App.tsx
vi App.tsx
```

- App.tsx 
```sh
// PHAS Minimal Application (based on EEVEE)

import { useState, useCallback } from 'react';
import { PHASTable } from './components/PHASTable';
import { IGVViewer } from './components/IGVViewer/IGVViewer';
import { useIGV } from './hooks/useIGV';
import { phasData } from './data/phasData';
import './App.css';

function App(): JSX.Element {
  // IGV hook (reuse from EEVEE)
  const { browserRef, isReady } = useIGV();

  // Selected PHAS
  const [selectedId, setSelectedId] = useState<string | null>(null);

  // 🔥 Core function: when clicking PHAS
  const handlePHASSelect = useCallback(
    (record: { phas_id: string; locus: string }) => {
      setSelectedId(record.phas_id);

      // Move IGV to locus
      if (browserRef.current) {
        browserRef.current.search(record.locus);
      }
    },
    [browserRef]
  );

  return (
    <div className="app">
      <main className="main">

        {/* PHAS Table */}
        <PHASTable
          records={phasData}
          selectedId={selectedId}
          onSelect={handlePHASSelect}
        />

        {/* IGV Viewer */}
        <IGVViewer ref={browserRef} isReady={isReady} />

      </main>
    </div>
  );
}

export default App;
```

## Test the Server

- #### Run backend
```sh
cd PHASER/backend
uvicorn main:app --reload --host 0.0.0.0 --port 8001

```
- #### Run frontend
```sh
cd PHASER/frontend
npm run dev -- --host --port 5174
```
- #### Open in server
```sh
http://163.221.246.151:5174
```
![alt text](image-9.png)

### IGV worked!
- No header
- No seacrch Panel
- Genome is not yet changed

- ### Add Genome to `public/genome`

```sh
cp -r /Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/genome "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public"
```
In `frontend/src/hooks/useIGV.ts`, modify `useIGV.ts`for foundation (customize later).

###### useIGV.ts:

```sh
// Hook for IGV.js browser management (PHASER version)

import { useState, useEffect, useRef, useCallback } from 'react';
import igv from 'igv';

interface UseIGVResult {
  browserRef: React.RefObject<HTMLDivElement>;
  browser: igv.IGVBrowser | null;
  isReady: boolean;
  navigateToLocus: (locus: string) => Promise<void>;
}

// 🔥 CLEAN IGV CONFIG (H. longicornis)
const IGV_CONFIG: igv.IGVConfig = {
  genome: 'custom',

  reference: {
    id: 'HaeL2018',
    name: 'Haemaphysalis longicornis',
    fastaURL: '/genome/GWHAMMI00000000.genome.fasta',
    indexURL: '/genome/GWHAMMI00000000.genome.fasta.fai',
  },

  // Start clean (no EEVEE tracks)
  tracks: [],

  showNavigation: true,
  showRuler: true,
  showCenterGuide: true,
};

export function useIGV(): UseIGVResult {
  const browserRef = useRef<HTMLDivElement>(null);
  const browserInstanceRef = useRef<igv.IGVBrowser | null>(null);
  const [browser, setBrowser] = useState<igv.IGVBrowser | null>(null);
  const [isReady, setIsReady] = useState(false);

  // Initialize IGV
  useEffect(() => {
    if (!browserRef.current || browserInstanceRef.current) return;

    const initBrowser = async () => {
      try {
        browserRef.current!.innerHTML = '';

        const igvBrowser = await igv.createBrowser(
          browserRef.current!,
          IGV_CONFIG
        );

        browserInstanceRef.current = igvBrowser;
        setBrowser(igvBrowser);
        setIsReady(true);
      } catch (err) {
        console.error('Failed to initialize IGV browser:', err);
      }
    };

    initBrowser();

    return () => {
      if (browserInstanceRef.current) {
        igv.removeBrowser(browserInstanceRef.current);
        browserInstanceRef.current = null;
      }
    };
  }, []);

  // 🔥 Safe navigation
  const navigateToLocus = useCallback(
    async (locus: string) => {
      if (!browser || !locus) return;

      try {
        console.log("Navigating to:", locus);
        await browser.search(locus);
      } catch (err) {
        console.error(
          'IGV navigation failed. Likely chromosome mismatch:',
          locus
        );
      }
    },
    [browser]
  );

  return {
    browserRef,
    browser,
    isReady,
    navigateToLocus,
  };
}
```
- #### Restart server to test!

![alt text](image-10.png)

### IGV is hanging! 
- change genome .fasta to 2bit file (much faster loading, optimized for IGV)
- In `frontend/src/hooks/useIGV.ts`, modify `useIGV.ts` by including 2bit file instead of genome fasta file. 

```sh
faToTwoBit /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.IGV.genome.fasta /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.2bit

cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.2bit "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public/genome"
```


###### useIGV.ts (gennome 2bit file instead of fasta file):

`frontend/src/hooks/useIGV.ts`

```sh 
// Hook for IGV.js browser management (PHASER version, .2bit)

import { useState, useEffect, useRef, useCallback } from 'react';
import igv from 'igv';

interface UseIGVResult {
  browserRef: React.RefObject<HTMLDivElement>;
  browser: igv.IGVBrowser | null;
  isReady: boolean;
  navigateToLocus: (locus: string) => Promise<void>;
}

const IGV_CONFIG: igv.IGVConfig = {
  genome: 'custom',

  reference: {
    id: 'HaeL2018',
    name: 'Haemaphysalis longicornis HaeL2018',
    twoBitURL: '/genome/GWHAMMI00000000.2bit',
  },

  tracks: [],

  showNavigation: true,
  showRuler: true,
  showCenterGuide: true,
};

export function useIGV(): UseIGVResult {
  const browserRef = useRef<HTMLDivElement>(null);
  const browserInstanceRef = useRef<igv.IGVBrowser | null>(null);
  const [browser, setBrowser] = useState<igv.IGVBrowser | null>(null);
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    if (!browserRef.current || browserInstanceRef.current) return;

    const initBrowser = async () => {
      try {
        browserRef.current!.innerHTML = '';

        const igvBrowser = await igv.createBrowser(
          browserRef.current!,
          IGV_CONFIG
        );

        browserInstanceRef.current = igvBrowser;
        setBrowser(igvBrowser);
        setIsReady(true);
      } catch (err) {
        console.error('Failed to initialize IGV browser:', err);
      }
    };

    initBrowser();

    return () => {
      if (browserInstanceRef.current) {
        igv.removeBrowser(browserInstanceRef.current);
        browserInstanceRef.current = null;
      }
    };
  }, []);

  const navigateToLocus = useCallback(
    async (locus: string) => {
      if (!browser || !locus) return;

      try {
        await browser.search(locus);
      } catch (err) {
        console.error('IGV navigation failed. Check chromosome name:', locus);
      }
    },
    [browser]
  );

  return {
    browserRef,
    browser,
    isReady,
    navigateToLocus,
  };
}
```
- Restart the server to test IGV. 

### IGV worked!
### working foundation established:
- IGV loads H. longicornis genome
- Frontend is stable

- ### Error: 
[Error] TypeError: browserRef.current.search is not a function. (In 'browserRef.current.search(record.locus)', 'browserRef.current.search' is undefined)
	(anonymous function) (App.tsx:32)
	callCallback2 (chunk-UXDD7MME.js:3674)
	dispatchEvent
	invokeGuardedCallbackDev (chunk-UXDD7MME.js:3699)
	invokeGuardedCallback (chunk-UXDD7MME.js:3733)
	invokeGuardedCallbackAndCatchFirstError (chunk-UXDD7MME.js:3736)
	executeDispatch (chunk-UXDD7MME.js:7014)
	processDispatchQueueItemsInOrder (chunk-UXDD7MME.js:7034)
	processDispatchQueue (chunk-UXDD7MME.js:7043)
	dispatchEventsForPlugins (chunk-UXDD7MME.js:7051)
	batchedUpdates$1 (chunk-UXDD7MME.js:18913)
	batchedUpdates (chunk-UXDD7MME.js:3579)
	dispatchEventForPluginEventSystem (chunk-UXDD7MME.js:7173)
	dispatchEventWithEnableCapturePhaseSelectiveHydrationWithoutDiscreteEventReplay (chunk-UXDD7MME.js:5478)
	dispatchEvent (chunk-UXDD7MME.js:5472:93)
	dispatchDiscreteEvent (chunk-UXDD7MME.js:5449)
- PHAS table are just samples
- no search bar yet 
- no header yet
- no tracks yet 
##### Customize later.


### What was built
From my workflow (PHAS_interface_Workflow_raw_1):
- FastAPI backend 
- Loaded PHAS .tsv 
- Created API endpoint /api/phas/list 
- React frontend 
- Connected frontend → backend 
Displayed PHAS table (only samples)
### Problem (Why UI is messy):
- my current structure is flat and improvised (NOT Modular + layered architecture
- Everything is inside:
    - main.py
    - one React table
- No separation of:
    - logic
    - API
    - UI components

## NOTE: 
- ##### For update or revision, refer to `PHAS_interface_Workflow_raw_3.md`
    (Revision of new strategy)
