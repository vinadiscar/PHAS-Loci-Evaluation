
3.16.2026
# B. WORKFLOW Phase 1: FOUNDATION
* Mocksite (with Frontend skeleton) already performed: `PHASinterface_Mock_Phase1.md`
* For Phase 1 (Foundation) we only need to add one key piece:
`Replace the hard-coded PHAS table with data coming from a backend API.`
###### Foundation:
```sh
1️. Create the FastAPI backend
2. Serve the PHAS loci list from a .tsv file
3. Connect the React table to the backend
```
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/"
```
## 1. Create the FastAPI backend
Inside your project, install required Python pakages
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS"
mkdir backend
cd backend

pip install fastapi uvicorn pandas
```
## 2. Create the PHAS loci table
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend"
mkdir data

cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend/data"

cp "/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/sRNAminer/merged.22.PHAS.list.filter.xls" "/Volumes/Install macOS Mojave/Vina/PHAS/backend/data/phas_loci.tsv"

head phas_loci.tsv
```

## 3. Create the FastAPI server
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend/"
vi main.py
```

```sh
from fastapi import FastAPI
import pandas as pd

app = FastAPI()

# Load PHAS dataset
phas_df = pd.read_csv("data/phas_loci.tsv", sep=r"\s+")

@app.get("/")
def root():
    return {"message": "PHAS backend running"}

@app.get("/api/phas/list")
def get_phas_list():
    return phas_df.to_dict(orient="records")
```

## 4. Run the Server
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend/"
uvicorn main:app --reload --port 8001 # Use a different port 8001 instead of 8000 (EEVEE backend has 8000 already)
```
- IN another terminal, test if the API backend is running:
```sh
curl http://localhost:8001
#{"message":"PHAS backend running"}%
curl http://localhost:8001/api/phas/list
# {
# "PHASID":"PHAS22-1",
# "Chr_ID":"GWHAMMI00000001",
# "Start_Pos":14520462,
# "End_Pos":14521176,
# "Locus_Len":715,
# "Lib_Num":1,
# "Pvalue":7.84313121476865e-05,
# ...
# }
```
***API backend is working.
NOTE: Safari is not opening. 

## 4. Connect the Mock Interface site to Backend API
- Two independent Mock sites:
```sh
Frontend
http://163.221.246.151:5174

Backend
http://163.221.246.151:8001
```

Next, we connect them:

```sh
React frontend
      ↓
fetch("http://163.221.246.151:8001/api/phas/list")
      ↓
FastAPI backend
      ↓
PHAS loci table appears in UI
```
- ###### 1. Start the backend
Terminal 1
```sh
cd backend
uvicorn main:app --reload --port 8001
```
- ###### 2. Start the frontend
Terminal 2 
```sh
cd frontend
npm run dev -- --host --port 5174
```

- ###### 3. Open `http://163.221.246.151:5174/`

#### Modify PHASTable.tsx ver. 2

simple working version:

```sh
vi PHASTable.tsx
```

```sh
import { useEffect, useState } from "react"

export default function PHASTable(){

const [phasData, setPhasData] = useState([])

useEffect(() => {

fetch("http://163.221.246.151:8001/api/phas/list")
.then(res => res.json())
.then(data => {
console.log(data)
setPhasData(data)
})
.catch(err => console.error(err))

}, [])

return(

<div>

<h3>PHAS Loci Table</h3>

<table border="1">

<thead>
<tr>
<th>PHASID</th>
<th>Chr</th>
<th>Start</th>
<th>End</th>
</tr>
</thead>

<tbody>

{phasData.map((row:any, i:number) => (

<tr key={i}>
<td>{row.PHASID}</td>
<td>{row.Chr_ID}</td>
<td>{row.Start_Pos}</td>
<td>{row.End_Pos}</td>
</tr>

))}

</tbody>

</table>

</div>

)

}
```

#### Modify App.txs ver.2:
- include `import PHASTable from...`
```sh
import './App.css'
import Header from './components/Header'
import PHASTable from './components/PHASTable'

function App() {
  return (
    <div className="app">

      <Header />

      <main className="main">

        {/* Search */}
        <div className="panel search-panel">
          <h3>Search PHAS Loci</h3>
        </div>

        {/* PHAS Table */}
        <div className="panel table-panel">
          <PHASTable />
        </div>

        {/* Middle plots */}
        <div className="middle-row">

          <div className="panel">
            <h3>Phasing Score Plot</h3>
          </div>

          <div className="panel">
            <h3>Size Distribution</h3>
          </div>

        </div>

        {/* IGV Viewer */}
        <div className="panel igv-panel">
          <h3>Genome Viewer</h3>
          <p>IGV will appear here</p>
        </div>

        {/* Confidence */}
        <div className="panel confidence-panel">
          <h3>Confidence Score</h3>
        </div>

      </main>

    </div>
  )
}

export default App
```
##### Issue:
- the interface is now correctly connected (backend connected to frontend), but the table is static.
- PHAS Table is not yet interactive when the browser was opened. Modify `PHASTable.tsx`

#### Modify PHASTable.tsx ver. 3
- Make PHASTable interactive in the browser
```sh
cd frontend/src/components/
vi PHASTable.tsx
```
- PHASTable.tsx:
```sh
import { useEffect, useState } from "react"

export default function PHASTable(){

  const [phasData, setPhasData] = useState([])

  useEffect(() => {

    fetch("http://163.221.246.151:8001/api/phas/list")
      .then(res => res.json())
      .then(data => {
        console.log("PHAS DATA:", data)
        setPhasData(data)
      })
      .catch(err => console.error("Error fetching PHAS:", err))

  }, [])

  function handleRowClick(row:any){
    console.log("Selected locus:", row)

    const region = `${row.Chr_ID}:${row.Start_Pos}-${row.End_Pos}`

    alert("Load region in IGV:\n" + region)
  }

  return(

    <div>

      <h3>PHAS Loci Table</h3>

      <table border={1} style={{width:"100%", borderCollapse:"collapse"}}>

        <thead>
          <tr>
            <th>PHASID</th>
            <th>Chr</th>
            <th>Start</th>
            <th>End</th>
          </tr>
        </thead>

        <tbody>

          {phasData.map((row:any, i:number) => (

            <tr 
              key={i}
              onClick={() => handleRowClick(row)}
              style={{cursor:"pointer"}}
            >
              <td>{row.PHASID}</td>
              <td>{row.Chr_ID}</td>
              <td>{row.Start_Pos}</td>
              <td>{row.End_Pos}</td>
            </tr>

          ))}

        </tbody>

      </table>

    </div>

  )

}
```

- ##### Issue: 
![alt text](image-1.png)
    - From Browser Console: "Failed to load resource: COuld not connect to the server. Error Fetching PHAS. Type Error: Load Failed
    - I have a local backend because I ran `uvicorn main:app --reload --port 8001` Why is it failing? Because my frontend is calling `http://163.221.246.151:8001/api/phas/list` but this is a remote server (lab machine), not my local backend. 
- #### FIX: 
    -  Since my frontend is calling th network `http://163.221.246.151:8001/api/phas/list`, I have to change `uvicorn main:app --reload --port 8001` (this is designed for local host, not the network). 

- ###### 1. Start the backend
Terminal 1
###### Change:
```sh
cd backend
uvicorn main:app --reload --port 8001
```
###### to:
```sh
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8001
```
- ###### 2. Start the frontend
Terminal 2 
```sh
cd frontend
npm run dev -- --host --port 5174
```
- #### Connection is fixed!
- #### Another Issue: Cross-Origin Issue
![alt text](image.png)
Error says:
```sh
Origin http://163.221.246.151:5174 is not allowed by Access-Control-Allow-Origin
```
- ###### What this means: 
    - Eventhough front end (`http://163.221.246.151:5174`) and backend (`http://163.221.246.151:8001`) have the same IP, but they have different ports, which means different origin. Browser blocks it for security. 

- #### FIX:
    - Enable CORS in the backend by modifying `main.py`

- main.py:

```sh
cd backend
vi main.py
```

```sh
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import pandas as pd

app = FastAPI()

# ✅ CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # for development (allow all)
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load PHAS loci table
phas_df = pd.read_csv("data/phas_loci.tsv", sep="\t")

@app.get("/")
def root():
    return {"message": "PHAS backend running"}

@app.get("/api/phas/list")
def get_phas_list():
    return phas_df.to_dict(orient="records")
```

- #### Phase 1 Done!
    - `Backend`: Running, API Working, Table returns 602 rows
    - `Frontend`: Fetch successful, Data rendered, rows clickable, Console logs working
    - UI still looks messy...
    
![alt text](image-2.png)





-----
# B. Improve UI Layout
```sh
1. Layout (Fix the mess)
2. Scrollable Table
3. Panel Alignment
4. Clean Spacing
5. Optional (Highlight selected row)
```

### 1. Fix Overlap
- ###### Modify `App.tsx version 3`
```sh
import "./App.css"
import PHASTable from "./components/PHASTable"

function App() {
  return (
    <div className="main">
      <h1>PHASER</h1>
      <p>Interactive PHAS Evaluation Platform</p>

      {/* Search */}
      <div className="panel">
        <h2>Search PHAS Loci</h2>
        <input
          type="text"
          placeholder="Search..."
          style={{ width: "100%", padding: "10px" }}
        />
      </div>

      {/* Table */}
      <div className="panel table-panel">
        <h2>PHAS Loci Table</h2>
        <PHASTable />
      </div>

      {/* Middle Row */}
      <div className="middle-row">
        <div className="panel">
          <h3>Phasing Score Plot</h3>
        </div>

        <div className="panel">
          <h3>Size Distribution</h3>
        </div>
      </div>

      {/* Genome Viewer */}
      <div className="panel">
        <h2>Genome Viewer</h2>
        <p>IGV will appear here</p>
      </div>
    </div>
  )
}

export default App
```
### 2. Fix Messy Layout
- ###### Add `App.css`
```sh
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background-color: #eef2f5;
}

.main {
  display: flex;
  flex-direction: column;
  gap: 20px;
  padding: 20px;
}

/* Panels */
.panel {
  background: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
}

/* Table container */
.table-panel {
  max-height: 400px;
  overflow: hidden;
}

/* Row for plots */
.middle-row {
  display: flex;
  gap: 20px;
}

.middle-row .panel {
  flex: 1;
  height: 250px;
}
```

### 3. Scrollable, clean, clickable
- ###### Modify `PHASTable.tsx`
```sh 
import { useEffect, useState } from "react"

interface PHAS {
  PHASID: string
  Chr_ID: string
  Start_Pos: number
  End_Pos: number
}

function PHASTable() {
  const [data, setData] = useState<PHAS[]>([])
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null)

  useEffect(() => {
    fetch("http://127.0.0.1:8000/api/phas/list")
      .then((res) => res.json())
      .then((data) => {
        console.log("PHAS DATA:", data)
        setData(data)
      })
  }, [])

  const handleRowClick = (row: PHAS, index: number) => {
    setSelectedIndex(index)

    const region = `${row.Chr_ID}:${row.Start_Pos}-${row.End_Pos}`
    console.log("Selected locus:", row)
    console.log("Region:", region)
  }

  return (
    <div style={{ maxHeight: "300px", overflowY: "auto" }}>
      <table
        style={{
          width: "100%",
          borderCollapse: "collapse",
          fontSize: "14px",
        }}
      >
        <thead>
          <tr style={{ background: "#f0f0f0" }}>
            <th style={thStyle}>PHASID</th>
            <th style={thStyle}>Chr</th>
            <th style={thStyle}>Start</th>
            <th style={thStyle}>End</th>
          </tr>
        </thead>

        <tbody>
          {data.map((row, i) => (
            <tr
              key={i}
              onClick={() => handleRowClick(row, i)}
              style={{
                cursor: "pointer",
                background: selectedIndex === i ? "#d0ebff" : "white",
              }}
            >
              <td style={tdStyle}>{row.PHASID}</td>
              <td style={tdStyle}>{row.Chr_ID}</td>
              <td style={tdStyle}>{row.Start_Pos}</td>
              <td style={tdStyle}>{row.End_Pos}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

const thStyle = {
  padding: "10px",
  borderBottom: "2px solid #ccc",
  textAlign: "left" as const,
}

const tdStyle = {
  padding: "8px",
  borderBottom: "1px solid #eee",
}

export default PHASTable
```
- Result:
![alt text](image-3.png)

- UI interface not perfect (Overlapping). 
- Since working backend and frontend works and scrollable table, move forward, polish later!

--------------------------------
# B. WORKFLOW Phase 2: VISUALIZATION
- IGV Visualization
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHAS"
```

#### IGV Architecture:
```sh
PHASTable → select locus
        ↓
Frontend state updates
        ↓
IGV:
  - jump to locus
  - load selected tracks
        ↓
Backend:
  - serves BigWig files
  - returns track metadata
  ```
## 1. Prepare Data 
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
- ##### Write Scipt `bam_to_bw.sh`
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS/backend"
mkdir -p scripts
vi scripts/bam_to_bw.sh
```

```sh
#!/bin/bash

INPUT_DIR="/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam"
OUTPUT_DIR="/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/bw"
GENOME_SIZE="/Volumes/Install macOS Mojave/Vina/PHAS/backend/static/GWHAMMI00000000.genome.chrom.sizes"

mkdir -p $OUTPUT_DIR

for bam in ${INPUT_DIR}/*.bam
do
  base=$(basename $bam .fastq.trimmed.mc.fa.sorted.bam)

  echo "Processing $base"

  # PLUS strand
  samtools view -b -F 16 $bam \
    | bedtools genomecov -ibam - -bg \
    > ${OUTPUT_DIR}/${base}.plus.bedgraph

  # MINUS strand
  samtools view -b -f 16 $bam \
    | bedtools genomecov -ibam - -bg \
    > ${OUTPUT_DIR}/${base}.minus.bedgraph

  # Convert to BigWig
  bedGraphToBigWig ${OUTPUT_DIR}/${base}.plus.bedgraph $GENOME_SIZE ${OUTPUT_DIR}/${base}.plus.bw
  bedGraphToBigWig ${OUTPUT_DIR}/${base}.minus.bedgraph $GENOME_SIZE ${OUTPUT_DIR}/${base}.minus.bw

  # Clean up
  rm ${OUTPUT_DIR}/${base}.plus.bedgraph
  rm ${OUTPUT_DIR}/${base}.minus.bedgraph

done

echo "All done!"
```
- ##### Run `bam_to_bw.sh
```sh
chmod +x scripts/bam_to_bw.sh
./scripts/bam_to_bw.sh
```


## 2. Backend- Track API
- Create the file `backend/routers/track_router.py` 

```sh
cd backend
mkdir routers
vi track_router.py
```

```sh
from fastapi import APIRouter

router = APIRouter(prefix="/api/tracks")

@router.get("/")
def get_tracks():
    return [
        {
            "id": "sample1_plus",
            "name": "Sample1 (+)",
            "url": "/data/bw/sample1.plus.bw",
            "color": "#1f77b4"
        },
        {
            "id": "sample1_minus",
            "name": "Sample1 (-)",
            "url": "/data/bw/sample1.minus.bw",
            "color": "#d62728"
        }
    ]
```

- #### Connect in `main.py`

Open `main.py` and add this:

```sh
from routers import track_router   # 👈 ADD

app.include_router(track_router.router)  # 👈 ADD
```

- Updated `main.py`

```sh
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles  # 👈 ADD
import pandas as pd

from routers import track_router   # 👈 ADD THIS LINE

app = FastAPI()

# ✅ CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # for development
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# ✅ Serve BigWig files
app.mount("/data", StaticFiles(directory="static"), name="data")  # 👈 ADD

# ✅ Include Track Router
app.include_router(track_router.router)   # 👈 ADD THIS LINE

# Load PHAS loci table
phas_df = pd.read_csv("data/phas_loci.tsv", sep="\t")

@app.get("/")
def root():
    return {"message": "PHAS backend running"}

@app.get("/api/phas/list")
def get_phas_list():
    return phas_df.to_dict(orient="records")
```

## 3. Frontend- Track Selector
- Create `TrackSelector.tsx`
```sh
cd frontend/src/components/
vi TrackSelector.tsx
```

- ###### TrackSelector.tsx

```sh
import { useEffect, useState } from "react";

export default function TrackSelector({ onChange }) {
  const [tracks, setTracks] = useState([]);
  const [selected, setSelected] = useState([]);

  useEffect(() => {
    fetch("http://localhost:8001/api/tracks/")
      .then(res => res.json())
      .then(setTracks);
  }, []);

  const toggle = (track: any) => {
    let updated;

    if (selected.find((t:any) => t.id === track.id)) {
      updated = selected.filter((t:any) => t.id !== track.id);
    } else {
      updated = [...selected, track];
    }

    setSelected(updated);
    onChange(updated);
  };

  return (
    <div style={{ padding: "10px", borderLeft: "1px solid gray" }}>
      <h3>Tracks</h3>

      {tracks.map((track: any) => (
        <div key={track.id}>
          <input
            type="checkbox"
            onChange={() => toggle(track)}
          />
          {track.name}
        </div>
      ))}
    </div>
  );
}
```
- #### Connect to App

Modify `frontend/src/App.tsx`
- add:
```sh
- const [tracks, setTracks] = useState([]) #This is your shared state
- <TrackSelector onChange={setTracks} /> #This means: You are passing a function (setTracks), Into TrackSelector, Using the prop name onChange
- export default function TrackSelector({ onChange }) { 
```
- Modified `App.tsx`
```sh
import "./App.css"
import { useState } from "react"
import PHASTable from "./components/PHASTable"
import IGVViewer from "./components/IGVViewer"
import TrackSelector from "./components/TrackSelector"

function App() {
  const [locus, setLocus] = useState("")
  const [tracks, setTracks] = useState([])

  return (
    <div className="main">
      <h1>PHASER</h1>
      <p>Interactive PHAS Evaluation Platform</p>

      {/* Search */}
      <div className="panel">
        <h2>Search PHAS Loci</h2>
        <input
          type="text"
          placeholder="Search..."
          style={{ width: "100%", padding: "10px" }}
        />
      </div>

      {/* Table */}
      <div className="panel table-panel">
        <h2>PHAS Loci Table</h2>
        {/* 👇 IMPORTANT: pass setLocus */}
        <PHASTable onSelect={setLocus} />
      </div>

      {/* Middle Row */}
      <div className="middle-row">
        <div className="panel">
          <h3>Phasing Score Plot</h3>
        </div>

        <div className="panel">
          <h3>Size Distribution</h3>
        </div>
      </div>

      {/* Genome Viewer + Track Panel */}
      <div style={{ display: "flex" }}>
        
        {/* IGV Viewer */}
        <div className="panel" style={{ flex: 1 }}>
          <h2>Genome Viewer</h2>

          {/* 👇 REAL IGV */}
          <IGVViewer locus={locus} selectedTracks={tracks} />
        </div>

        {/* Track Selector */}
        <TrackSelector onChange={setTracks} />

      </div>
    </div>
  )
}

export default App
```

- Right now our table does not send locus yet. So we'll modify `PHASTable.tsx`.
- Add:
```sh
- function PHASTable({ onSelect }: { onSelect: (locus: string) => void })     # This allows the table to communicate with App.tsx
- onSelect(region) # This is the critical line
```
- #### Modify `PHASTable.tsx`
```sh
import { useEffect, useState } from "react"

interface PHAS {
  PHASID: string
  Chr_ID: string
  Start_Pos: number
  End_Pos: number
}

// ✅ [ADDED] accept onSelect as prop
function PHASTable({ onSelect }: { onSelect: (locus: string) => void }) {

  const [data, setData] = useState<PHAS[]>([])
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null)

  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/phas/list")
      .then((res) => res.json())
      .then((data) => {
        console.log("PHAS DATA:", data)
        setData(data)
      })
  }, [])

  const handleRowClick = (row: PHAS, index: number) => {
    setSelectedIndex(index)

    const region = `${row.Chr_ID}:${row.Start_Pos}-${row.End_Pos}`
    console.log("Selected locus:", row)
    console.log("Region:", region)

    // ✅ [ADDED] send locus to parent (App.tsx)
    onSelect(region)
  }

  return (
    <div style={{ maxHeight: "300px", overflowY: "auto" }}>
      <table
        style={{
          width: "100%",
          borderCollapse: "collapse",
          fontSize: "14px",
        }}
      >
        <thead>
          <tr style={{ background: "#f0f0f0" }}>
            <th style={thStyle}>PHASID</th>
            <th style={thStyle}>Chr</th>
            <th style={thStyle}>Start</th>
            <th style={thStyle}>End</th>
          </tr>
        </thead>

        <tbody>
          {data.map((row, i) => (
            <tr
              key={i}
              onClick={() => handleRowClick(row, i)}
              style={{
                cursor: "pointer",
                background: selectedIndex === i ? "#d0ebff" : "white",
              }}
            >
              <td style={tdStyle}>{row.PHASID}</td>
              <td style={tdStyle}>{row.Chr_ID}</td>
              <td style={tdStyle}>{row.Start_Pos}</td>
              <td style={tdStyle}>{row.End_Pos}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

const thStyle = {
  padding: "10px",
  borderBottom: "2px solid #ccc",
  textAlign: "left" as const,
}

const tdStyle = {
  padding: "8px",
  borderBottom: "1px solid #eee",
}

export default PHASTable

```
## 4. Create `IGVViewer.tsx`
- `frontend/src/components/IGVViewer.tsx`

- ### Install IGV.js
```sh
cd frontend
npm install igv
```
- ### Create `IGVViewer.tsx`
```sh
# Write script
cd frontend/src/components
vi IGVViewer.tsx
```
```sh
import { useEffect, useRef } from "react"
import igv from "igv"

interface Track {
  id: string
  name: string
  url: string
  color: string
}

export default function IGVViewer({
  locus,
  selectedTracks,
}: {
  locus: string
  selectedTracks: Track[]
}) {
  const igvContainer = useRef<HTMLDivElement | null>(null)
  const igvBrowser = useRef<any>(null)

  // ✅ Initialize IGV once
  useEffect(() => {
    if (!igvContainer.current) return

    igv.createBrowser(igvContainer.current, {
      genome: "hg38", // ⚠️ TEMP (we will fix later if custom genome)
      locus: "chr1:1-10000",
    }).then((browser) => {
      igvBrowser.current = browser
    })
  }, [])

  // ✅ Jump to locus
  useEffect(() => {
    if (igvBrowser.current && locus) {
      igvBrowser.current.search(locus)
    }
  }, [locus])

  // ✅ Load tracks
  useEffect(() => {
    if (!igvBrowser.current) return

    igvBrowser.current.removeAllTracks()

    selectedTracks.forEach((track) => {
      igvBrowser.current.loadTrack({
        name: track.name,
        format: "bigwig",
        url: `http://localhost:8001${track.url}`,
        color: track.color,
      })
    })
  }, [selectedTracks])

  return (
    <div
      ref={igvContainer}
      style={{ height: "500px", border: "1px solid #ccc" }}
    />
  )
}
```
## 5. Test the Integration
- Verify if everything works together.

Start both servers:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```
