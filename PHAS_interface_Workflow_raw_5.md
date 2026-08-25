04.08.2026

##### PHAS_interface_Workflow_raw_5
> This is a raw file for IGV Genome browser integration panel
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

# PHASE 3A
## IGV Genome Browser Panel ver. 1

```sh
PHASTable (click row)
        ↓
best_sample (e.g. N137)
        ↓
GET /api/tracks/N137
        ↓
returns track URLs (/api/files/...)
        ↓
IGV loads tracks via Range streaming
        ↓
files.py streams BigWig/BigBed efficiently
```
---
## 1. Prepare Data
- ### Folder Structure

```sh
cd "/Volumes/Install macOS Mojave/Trackhubs"
mkdir PHAS_tracks

├── genome/
│   └── GWHAMMI00000000.IGV.genome.fasta,    
        GWHAMMI00000000.genome.chrom.sizes,     GWHAMMI00000000.IGV.genome.fasta.fai, GWHAMMI00000000.2bit
├── annotation/
│   └── merged.PHAS22.bb
├── bw/
│   ├── N137.trimmed.mc.fa.sorted.plus.bw
│   ├── N137.trimmed.mc.fa.sorted.minus.bw
│   ├──N012.fastq.trimmed.mc.fa.sorted.plus.bw
│   ├── N013.fastq.trimmed.mc.fa.sorted.minus.bw

```
### 1.1 Get Chrom. sizes
```sh
cd "/Volumes/Install macOS Mojave/Vina/PHASER"

awk '{for(i=1;i<=NF;i++){if($i ~ /^Len=/){split($i,a,"="); print $1"\t"a[2]}}}' \
/Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/genome/GWHAMMI00000000.genome.fasta.fai \
> /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/GWHAMMI00000000.genome.chrom.sizes"

#Check output content
head GWHAMMI00000000.genome.chrom.sizes

# confirm it's TAB-separated, not spaces.
cat -vet GWHAMMI00000000.genome.chrom.sizes | head
# GWHAMMI00000001^I348261740$ -> TAB-separated 
# ^I = TAB ✔️
```

### 1.2.  Convert BAM files to Bigwig Files
> - Use streaming to avoid storing BAM files in the directory
> - NOTE: Pipelines should be streaming, minimal storage, and reproducible. 


- ### Install missing tools
- samtools
- bedtools
- bedGraphToBigWig

```sh
conda create -n bamTobw_pipeline_env -c bioconda -c conda-forge \
  samtools bedtools ucsc-bedgraphtobigwig
```
- ### Write Scipt `bam_to_bw.sh`


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
OUTPUT_DIR="/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data/bw"
GENOME_SIZE="/Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/GWHAMMI00000000.genome.chrom.sizes"

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

- ### remove intermediate bedgraph files 

```sh
rm "/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data/bw"/*.bedgraph
```
### 1.3 Convert .gff3 annotation file to BigBEd file
- ##### Check if the annotation file is a single continuous region
    > ###### Single continuous region
    > > - One feature = one uninterrupted genomic interval
Example:
chr1   .   PHAS   1000   1200   .   +   .   ID=PHAS22-1
This means:
Start: 1000
End: 1200
-No gaps inside (This is one continuous block)

##### Check:
```sh
cut -f3 merged.PHAS22.gff3 | sort | uniq -c
# 602 PHAS22
```
- 602 entries are single continuous regions
- Therefore, proceed with simple Bed6 or Bigbed6 conversion

```sh
cd "/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data/annotation"

# 1. Convert GFF3 → BED6

awk -F'\t' 'BEGIN{OFS="\t"}
$3=="PHAS22" {
  split($9,a,";");
  split(a[1],b,"=");
  name=b[2];

  print $1, $4-1, $5, name, 0, $7;
}' merged.PHAS22.gff3 > merged.PHAS22.bed

head merged.PHAS22.bed

# 2. Sort BED file
sort -k1,1 -k2,2n merged.PHAS22.bed > merged.PHAS22.sorted.bed

# 3. Convert BED to Bigbed
bedToBigBed merged.PHAS22.sorted.bed \
../genome/GWHAMMI00000000.genome.chrom.sizes \
merged.PHAS22.bb
```

### 1.4 Copy files (genome (2bit, chromsizes), bigwig and annotation files to /Trackhubs/PHASER_tracks

`Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks`

```sh
# 1. Copy genome (2bit) files
cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.2bit "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/genome"
# 2. copy chrom sizes 
cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/GWHAMMI00000000.genome.chrom.sizes "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/genome"

# 3. Copy bigwig files
## Move from "/backend/Data"
cp -r "/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data/bw" "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"

# 4. copy annotation file
cp -r "/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data/annotation" "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"

# 5. Copy fasta and fasta indexed files
cp -r /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000 "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/genome"
```
---
## 2. Modify Backend (Serve Track Metadata)
- config.py
- track_service.py
- tracks.py
- files.py
- main.py
### Previous Codes (backup)
- ##### 1. 
```sh
(base) okamuralab@rnalab app % cat 
"""PHASER Configuration"""

from pathlib import Path

# 🔷 BASE DIRECTORY (CHANGE THIS TO YOUR ACTUAL PATH)
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# 🔷 DATA DIRECTORY
DATA_DIR = BASE_DIR / "data"

# 🔷 PHAS DATA FILE
PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# 🔷 LIBRARY INFO (dummy for now, required by backend)
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# 🔷 TRACK DIRECTORY (for .bw files)
TRACKHUB_BASE = BASE_DIR / "trackhub"

# 🔷 API SETTINGS
API_PREFIX = "/api"
```

- ##### 2. app/services/track_service.py

```sh
"""Track Service (Phase 1 - Metadata Only)"""

import pandas as pd
from typing import List, Optional

from ..config import LIBRARY_INFO_FILE
from ..models.phas import TrackFile


class TrackService:
    """Service for track metadata only"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    def _load_library_info(self) -> pd.DataFrame:
        """Load library information"""
        if self._library_info is None:
            self._library_info = pd.read_csv(LIBRARY_INFO_FILE, sep='\t')
        return self._library_info

    def get_all_tracks(self) -> List[TrackFile]:
        """Return all libraries (no files yet)"""
        df = self._load_library_info()
        tracks = []

        for _, row in df.iterrows():
            tracks.append(TrackFile(
                library_id=row['library_ID'],
                sample_name=row.get('Sample', row['library_ID']),
                category=row.get('Category', 'unknown'),
                default_setting=row.get('default_setting', 'off'),
                files={}   # ← EMPTY for now
            ))

        return tracks

    def get_track_by_id(self, library_id: str) -> Optional[TrackFile]:
        """Return one library"""
        df = self._load_library_info()
        filtered = df[df['library_ID'] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row['library_ID'],
            sample_name=row.get('Sample', row['library_ID']),
            category=row.get('Category', 'unknown'),
            default_setting=row.get('default_setting', 'off'),
            files={}   # ← EMPTY
        )

    def get_genome_files(self):
        """Not needed yet"""
        return {}

    def resolve_file_path(self, relative_path: str):
        """Not used yet"""
        return None


# Singleton instance
track_service = TrackService()
``` 
- ##### 3. models/phas.py
```sh
"""PHAS Data Models"""

from pydantic import BaseModel
from typing import List, Optional


# �� MAIN RECORD
class PHASRecord(BaseModel):
    """Single PHAS locus record"""
    phas_id: str
    chromosome: str
    start: int
    end: int
    length: int
    lib_num: int
    phas_hit: int
    avg_hit_pos: float
    phas_ratio: float
    pvalue: float
    phase: int
    abundance: float
    score: float
    note: str
    best_region: str
    best_sample: str


# 🔷 LIST RESPONSE
class PHASListResponse(BaseModel):
    """Response for PHAS list"""
    total: int
    records: List[PHASRecord]


# 🔷 SEARCH RESPONSE
class PHASSearchResponse(BaseModel):
    """Response for PHAS search"""
    query: str
    total: int
    records: List[PHASRecord]


# 🔷 DISPLAY REGION (FOR IGV)
class DisplayRegion(BaseModel):
    """Calculated display region for genome browser"""
    chrom: str
    start: int
    end: int


# 🔷 TRACK FILE (KEEP FROM EEVEE — STILL USEFUL)
class TrackFile(BaseModel):
    """Track file information"""
    library_id: str
    sample_name: str
    category: str = "small RNA"
    default_setting: str = "off"
    files: dict


class TrackListResponse(BaseModel):
    """Response for track list"""
    tracks: List[TrackFile]


# 🔷 OPTIONAL: KEEP BLAT (IF YOU STILL WANT ALIGNMENT)
class BlatQuery(BaseModel):
    """BLAT query request"""
    sequences: List[dict]


class BlatHit(BaseModel):
    """Single BLAT hit"""
    chromosome: str
    start: int
    end: int
    strand: str
    identity: float
    score: int
    query_start: int
    query_end: int


class BlatResult(BaseModel):
    """BLAT result for a single query"""
    query_name: str
    hits: List[BlatHit]


class BlatResponse(BaseModel):
    """Response for BLAT search"""
    results: List[BlatResult]
```

##### 4. router/files.py
```sh
"""Files API Router"""
from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import FileResponse, StreamingResponse
from pathlib import Path
import os
import mimetypes

from ..services.track_service import track_service
from ..config import TRACKHUB_BASE

router = APIRouter(prefix="/files", tags=["Files"])


def get_content_type(filename: str) -> str:
    """Get content type for a file"""
    ext = Path(filename).suffix.lower()
    content_types = {
        '.bw': 'application/octet-stream',
        '.bb': 'application/octet-stream',
        '.2bit': 'application/octet-stream',
        '.bed': 'text/plain',
        '.fa': 'text/plain',
        '.fasta': 'text/plain',
    }
    return content_types.get(ext, 'application/octet-stream')


@router.get("/{file_path:path}")
async def get_file(file_path: str, request: Request):
    """
    Serve track and genome files with Range request support

    This endpoint supports HTTP Range requests for efficient
    streaming of large files (required by igv.js for BigWig/BigBed files)
    """
    # Resolve the file path
    resolved_path = track_service.resolve_file_path(file_path)

    if not resolved_path or not resolved_path.exists():
        raise HTTPException(status_code=404, detail=f"File not found: {file_path}")

    # Security check - ensure file is within allowed directory
    try:
        resolved_path.resolve().relative_to(TRACKHUB_BASE.resolve())
    except ValueError:
        raise HTTPException(status_code=403, detail="Access denied")

    file_size = resolved_path.stat().st_size
    content_type = get_content_type(str(resolved_path))

    # Handle Range requests
    range_header = request.headers.get('range')

    if range_header:
        # Parse range header
        try:
            range_spec = range_header.replace('bytes=', '')
            start_str, end_str = range_spec.split('-')
            start = int(start_str) if start_str else 0
            end = int(end_str) if end_str else file_size - 1

            # Validate range
            if start >= file_size:
                raise HTTPException(status_code=416, detail="Range not satisfiable")

            end = min(end, file_size - 1)
            content_length = end - start + 1

            def iterfile():
                with open(resolved_path, 'rb') as f:
                    f.seek(start)
                    remaining = content_length
                    while remaining > 0:
                        chunk_size = min(8192, remaining)
                        data = f.read(chunk_size)
                        if not data:
                            break
                        remaining -= len(data)
                        yield data

            return StreamingResponse(
                iterfile(),
                status_code=206,
                media_type=content_type,
                headers={
                    'Content-Range': f'bytes {start}-{end}/{file_size}',
                    'Accept-Ranges': 'bytes',
                    'Content-Length': str(content_length),
                    'Access-Control-Allow-Origin': '*',
                    'Access-Control-Expose-Headers': 'Content-Range, Accept-Ranges, Content-Length'
                }
            )

        except (ValueError, IndexError):
            raise HTTPException(status_code=400, detail="Invalid range header")

    # Return full file if no range requested
    return FileResponse(
        resolved_path,
        media_type=content_type,
        headers={
            'Accept-Ranges': 'bytes',
            'Content-Length': str(file_size),
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Expose-Headers': 'Content-Range, Accept-Ranges, Content-Length'
        }
    )
```
5.) routers/tracks.py
```sh
"""Tracks API Router"""
from fastapi import APIRouter, HTTPException

from ..services.track_service import track_service
from ..models.phas import TrackListResponse, TrackFile

router = APIRouter(prefix="/tracks", tags=["Tracks"])


@router.get("", response_model=TrackListResponse)
async def get_all_tracks():
    """Get all available tracks with their sample names"""
    tracks = track_service.get_all_tracks()
    return TrackListResponse(tracks=tracks)


@router.get("/genome")
async def get_genome_files():
    """Get genome reference file paths"""
    return track_service.get_genome_files()


@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    """Get track information by library ID"""
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track
```
6) backend/main.py
```sh
"""
PHAS Backend Application
PHAS Evaluation Resource Platform
"""

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import API_PREFIX
from app.routers import phas, tracks, files  # removed eve, rpm

# Optional: keep BLAT only if needed
# from app.routers import blat

# Create FastAPI application
app = FastAPI(
    title="PHAS API",
    description="PHAS Evaluation Resource Platform - Backend API",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc"
)

# Configure CORS for remote access
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins for remote access
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["Content-Range", "Accept-Ranges", "Content-Length"]
)

# Include routers
app.include_router(phas.router, prefix=API_PREFIX)
app.include_router(tracks.router, prefix=API_PREFIX)
app.include_router(files.router, prefix=API_PREFIX)

# Optional BLAT (uncomment if needed)
# app.include_router(blat.router, prefix=API_PREFIX)


@app.get("/")
async def root():
    """Root endpoint"""
    return {
        "name": "PHAS API",
        "description": "PHAS Evaluation Resource Platform",
        "version": "1.0.0",
        "docs": "/docs"
    }


@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```
### FRONTEND: Previous codes (backup)

- #### 1. src/App.tsx
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

##### 2.) App.css
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

3. PHASTable.tsx
```sh
// PHASTable.tsx
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

export function PHASTable({ records, selectedId, onSelectPhasAndStage, loading }: PHASTableProps) {
  const [sortField, setSortField] = useState<string>("phas_id");
  const [sortOrder, setSortOrder] = useState<SortOrder>("asc");

  const columns = useMemo(() => (records.length > 0 ? Object.keys(records[0]) : []), [records]);

  const sortedRecords = useMemo(() => {
    return [...records].sort((a, b) => {
      const valA = (a as any)[sortField];
      const valB = (b as any)[sortField];

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
                  <th key={col} className={styles.sortable} onClick={() => handleSort(col)}>
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
                  className={`${styles.row} ${selectedId === r.phas_id ? styles.selected : ""}`}
                  onClick={() => onSelectPhasAndStage(r, null)}
                >
                  {columns.map((col) => {
                    const value = (r as any)[col];
                    const isBestSample = col.toLowerCase().includes("best") && col.toLowerCase().includes("sample");

                    if (isBestSample && typeof value === "string") {
                      return (
                        <td key={col} className={getCellClass(col)}>
                          <span
                            style={{ color: "#007bff", cursor: "pointer" }}
                            onClick={(e) => {
                              e.stopPropagation();
                              const stage = getStageFromSample(value);
                              if (stage) onSelectPhasAndStage(r, stage);
                            }}
                            title="Click to view this Best Sample"
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
  if (typeof value === "number") return Number.isInteger(value) ? value : value.toFixed(2);
  return value;
}

function getCellClass(col: string) {
  const key = col.toLowerCase();
  if (key.includes("phasid")) return styles.phasId;
  if (key.includes("chr")) return styles.chromosome;
  if (key.includes("start") || key.includes("end") || key.includes("len")) return styles.numeric;
  if (key.includes("phase")) return styles.phase;
  if (key.includes("pvalue")) return styles.pvalue;
  if (key.includes("abudance") || key.includes("abundance")) return styles.abundance;
  if (key.includes("note")) return styles.note;
  return "";
}
```
### Modified Codes
- ### 

```sh
"""PHASER Configuration"""

from pathlib import Path

# 🔷 BASE DIRECTORY
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# 🔷 DATA
DATA_DIR = BASE_DIR / "data"
PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# 🔷 TRACK HUB (YOUR ACTUAL DATA LOCATION)
TRACKHUB_BASE = Path("/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks")

# 🔥 GENOME FILES (IGV REQUIRES FASTA + FAI)
GENOME_FASTA = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.fa"
GENOME_FAI = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.fa.fai"

# 🔷 ANNOTATION (PHAS loci)
PHAS_ANNOTATION_BB = TRACKHUB_BASE / "annotation" / "merged.PHAS22.bb"

# 🔷 API
API_PREFIX = "/api"
```
- ### track_service.py
```sh
"""Track Service for PHASER (IGV-ready, FASTA-based genome)"""

from pathlib import Path
from typing import Dict, Optional

from ..config import (
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB
)


class TrackService:
    """Service for IGV track generation"""

    def _find_bw_files(self, sample: str) -> Dict[str, str]:
        """Find plus/minus BigWig files for a sample"""
        bw_dir = TRACKHUB_BASE / "bw"

        plus_file = None
        minus_file = None

        for f in bw_dir.glob(f"{sample}*.bw"):
            name = f.name.lower()

            if "plus" in name:
                plus_file = f.name
            elif "minus" in name:
                minus_file = f.name

        files = {}

        if plus_file:
            files["plus"] = f"/api/files/bw/{plus_file}"

        if minus_file:
            files["minus"] = f"/api/files/bw/{minus_file}"

        return files

    def get_tracks_for_sample(self, sample: str) -> Dict:
        """Return IGV configuration for a given sample"""

        bw_files = self._find_bw_files(sample)

        return {
            # 🔥 FIXED: IGV-compatible genome
            "genome": {
                "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
                "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
            },

            "tracks": [
                {
                    "name": f"{sample} Plus",
                    "type": "wig",
                    "format": "bigwig",
                    "url": bw_files.get("plus")
                },
                {
                    "name": f"{sample} Minus",
                    "type": "wig",
                    "format": "bigwig",
                    "url": bw_files.get("minus")
                },
                {
                    "name": "PHAS loci",
                    "type": "annotation",
                    "format": "bigbed",
                    "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
                }
            ]
        }

    def resolve_file_path(self, relative_path: str) -> Optional[Path]:
        """Resolve file path for IGV streaming"""
        return TRACKHUB_BASE / relative_path


# 🔷 Singleton instance
track_service = TrackService()
```
- ### tracks.py
```sh
"""Tracks API Router (PHASER - Metadata + IGV)"""

from fastapi import APIRouter, HTTPException

from ..services.track_service import track_service
from ..models.phas import TrackListResponse, TrackFile

router = APIRouter(prefix="/tracks", tags=["Tracks"])


# ===============================
# 🔷 EXISTING (KEEP FOR TABLE / METADATA)
# ===============================

@router.get("", response_model=TrackListResponse)
async def get_all_tracks():
    """Get all available tracks (metadata only)"""
    tracks = track_service.get_all_tracks()
    return TrackListResponse(tracks=tracks)


@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    """Get track metadata by library ID"""
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track


# ===============================
# 🔥 NEW (IGV ENDPOINT)
# ===============================

@router.get("/igv/{sample}")
async def get_igv_tracks(sample: str):
    """
    Return IGV configuration for a sample
    (used by IGVViewer)
    """
    return track_service.get_tracks_for_sample(sample)
```
- ### App.tsx
```sh
// App.tsx
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer"; // ✅ ADD THIS

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

  // ✅ NEW: selected sample for IGV
  const [selectedSample, setSelectedSample] = useState<string | null>(null);

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
        if (stages.length > 0) setActiveStage("ALL");
      });
  }, [selectedPhas]);

  // ✅ NEW: extract sample (VERY IMPORTANT)
  useEffect(() => {
    if (!selectedPhas) {
      setSelectedSample(null);
      return;
    }

    if (selectedPhas.best_sample) {
      const match = selectedPhas.best_sample.match(/N\d+/);
      if (match) {
        setSelectedSample(match[0]);
        return;
      }
    }

    setSelectedSample(null);
  }, [selectedPhas]);

  // Handler
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null
  ) => {
    setSelectedPhas(phas);
    setActiveStage(stage || "ALL");
    setView("all");
    setSearchQuery(phas.phas_id);
  };

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
          options={phasRecords.map((r) => r.phas_id)}
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

            {/* ✅ IGV PANEL */}
            {selectedSample && (
              <IGVViewer
                sample={selectedSample}
                locus={{
                  chrom: selectedPhas.chromosome,
                  start: selectedPhas.start,
                  end: selectedPhas.end,
                }}
              />
            )}
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
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

export default function IGVViewer({ sample, locus }: Props) {
  const igvContainerRef = useRef<HTMLDivElement | null>(null);
  const igvBrowserRef = useRef<any>(null);
  const isReadyRef = useRef(false);

  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // -----------------------------
  // 🔥 LOAD IGV (when sample changes)
  // -----------------------------
  useEffect(() => {
    if (!igvContainerRef.current || !sample) return;

    // 🔥 destroy previous IGV instance
    if (igvBrowserRef.current) {
      igvBrowserRef.current.destroy?.();
      igvBrowserRef.current = null;
      isReadyRef.current = false;
    }

    const loadIGV = async () => {
      setLoading(true);
      setError(null);

      try {
        console.log("Loading IGV for sample:", sample);

        const res = await fetch(`${API_BASE}/api/tracks/igv/${sample}`);
        const config = await res.json();

        console.log("IGV config:", config);

        const options = {
          genome: {
            fastaURL: API_BASE + config.genome.fastaURL,
            indexURL: API_BASE + config.genome.indexURL,
          },
          tracks: config.tracks.map((t: any) => ({
            ...t,
            url: API_BASE + t.url,
          })),
        };

        const browser = await igv.createBrowser(
          igvContainerRef.current!,
          options
        );

        igvBrowserRef.current = browser;

        // 🔥 Wait until IGV fully initializes
        setTimeout(() => {
          isReadyRef.current = true;
          console.log("IGV is ready");

          // 🔥 Initial jump (default PHAS22-1)
          if (locus) {
            browser.search(
              `${locus.chrom}:${locus.start}-${locus.end}`
            );
          }
        }, 500);

      } catch (err: any) {
        console.error("IGV load error:", err);
        setError(err.message || "Failed to load IGV");
      } finally {
        setLoading(false);
      }
    };

    loadIGV();
  }, [sample]);

  // -----------------------------
  // 🔥 UPDATE LOCUS (on click)
  // -----------------------------
  useEffect(() => {
    if (!igvBrowserRef.current || !locus) return;

    if (!isReadyRef.current) {
      console.log("IGV not ready yet, skipping locus update...");
      return;
    }

    console.log("Navigating to locus:", locus);

    try {
      igvBrowserRef.current.search(
        `${locus.chrom}:${locus.start}-${locus.end}`
      );
    } catch (e) {
      console.warn("IGV search failed:", e);
    }
  }, [locus]);

  return (
    <div className={styles.container}>
      <div className={styles.header}>
        Genome Browser (IGV) — {sample}
      </div>

      {loading && (
        <div className={styles.loading}>Loading IGV...</div>
      )}

      {error && (
        <div className={styles.error}>{error}</div>
      )}

      <div
        ref={igvContainerRef}
        className={styles.viewer}
        style={{
          display: loading || error ? "none" : "block",
        }}
      />
    </div>
  );
}
```
---
# IGV Genome Browser Panel ver. 2 
- #### App.tsx

`vi App.tsx`

```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";

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

  // 🔥 IGV STATE
  const [selectedSample, setSelectedSample] = useState<string | null>(null);

  // -----------------------------
  // 🔷 LOAD DATA
  // -----------------------------
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        // 🔥 DEFAULT PHAS22-1
        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          setSelectedPhas(defaultPhas);

          if (defaultPhas.best_sample) {
            const match = defaultPhas.best_sample.match(/N\d+/);
            if (match) {
              setSelectedSample(match[0]);
            }
          }
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // -----------------------------
  // 🔷 SEARCH (FILTER ONLY)
  // -----------------------------
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

    // 🔥 OPTIONAL: auto-select ONLY if exact match AND different
    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );

    if (exact && exact.phas_id !== selectedPhas?.phas_id) {
      handleSelectPhasAndStage(exact, null);
    }
  }, [searchQuery, phasRecords]);

  // -----------------------------
  // 🔷 LOAD STAGES
  // -----------------------------
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

  // -----------------------------
  // 🔥 CLICK HANDLER (FIXED)
  // -----------------------------
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null
  ) => {
    setSelectedPhas(phas);
    setActiveStage(stage || "ALL");
    setView("all");

    // ❌ IMPORTANT: DO NOT update searchQuery here
    // setSearchQuery(phas.phas_id);  ← removed

    // ✅ Update IGV sample only
    if (phas.best_sample) {
      const match = phas.best_sample.match(/N\d+/);
      if (match) {
        setSelectedSample(match[0]);
      }
    }
  };

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
          options={phasRecords.map((r) => r.phas_id)}
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

            {/* 🔥 IGV PANEL */}
            {selectedSample && (
               <IGVViewer
                key={`${selectedSample}-${selectedPhas.phas_id}`}
                sample={selectedSample}
                locus={{
                  chrom: selectedPhas.chromosome,
                  start: selectedPhas.start,
                  end: selectedPhas.end,
                }}
              />
            )}
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
(base) okamuralab@rnalab src % cat App.tsx 
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";

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

  // 🔥 IGV STATE
  const [selectedSample, setSelectedSample] = useState<string | null>(null);

  // -----------------------------
  // 🔷 LOAD DATA
  // -----------------------------
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        // 🔥 DEFAULT PHAS22-1
        const defaultPhas = data.records.find(
          (r) => r.phas_id === "PHAS22-1"
        );

        if (defaultPhas) {
          setSelectedPhas(defaultPhas);

          if (defaultPhas.best_sample) {
            const match = defaultPhas.best_sample.match(/N\d+/);
            if (match) {
              setSelectedSample(match[0]);
            }
          }
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // -----------------------------
  // 🔷 SEARCH (FILTER ONLY)
  // -----------------------------
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

    // 🔥 OPTIONAL: auto-select ONLY if exact match AND different
    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );

    if (exact && exact.phas_id !== selectedPhas?.phas_id) {
      handleSelectPhasAndStage(exact, null);
    }
  }, [searchQuery, phasRecords]);

  // -----------------------------
  // 🔷 LOAD STAGES
  // -----------------------------
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

  // -----------------------------
  // 🔥 CLICK HANDLER (FIXED)
  // -----------------------------
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null
  ) => {
    setSelectedPhas(phas);
    setActiveStage(stage || "ALL");
    setView("all");

    // ❌ IMPORTANT: DO NOT update searchQuery here
    // setSearchQuery(phas.phas_id);  ← removed

    // ✅ Update IGV sample only
    if (phas.best_sample) {
      const match = phas.best_sample.match(/N\d+/);
      if (match) {
        setSelectedSample(match[0]);
      }
    }
  };

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
          options={phasRecords.map((r) => r.phas_id)}
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

            {/* 🔥 IGV PANEL */}
            {selectedSample && (
               <IGVViewer
                key={`${selectedSample}-${selectedPhas.phas_id}`}
                sample={selectedSample}
                locus={{
                  chrom: selectedPhas.chromosome,
                  start: selectedPhas.start,
                  end: selectedPhas.end,
                }}
              />
            )}
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
#### PHASTable.tsx
```sh
// PHASTable.tsx
import { useState, useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";
import { getStageFromSample } from "../../utils/sampleMap";

type SortOrder = "asc" | "desc";

interface PHASTableProps {
  records: PHASLocus[];
  selectedId: string | null;
  onSelectPhasAndStage: (
    phas: PHASLocus,
    stage: string | null,
    sample?: string | null
  ) => void;
  loading?: boolean;
}

export function PHASTable({
  records,
  selectedId,
  onSelectPhasAndStage,
  loading,
}: PHASTableProps) {
  const [sortField, setSortField] = useState<string>("phas_id");
  const [sortOrder, setSortOrder] = useState<SortOrder>("asc");

  const columns = useMemo(
    () => (records.length > 0 ? Object.keys(records[0]) : []),
    [records]
  );

  const sortedRecords = useMemo(() => {
    return [...records].sort((a, b) => {
      const valA = (a as any)[sortField];
      const valB = (b as any)[sortField];

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

  const renderSortIndicator = (field: string) =>
    field !== sortField ? "" : sortOrder === "asc" ? " ▲" : " ▼";

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
              {sortedRecords.map((r, i) => {
                const sampleMatch = r.best_sample?.match(/N\d+/);
                const sample = sampleMatch ? sampleMatch[0] : null;

                return (
                  <tr
                    key={i}
                    className={`${styles.row} ${
                      selectedId === r.phas_id ? styles.selected : ""
                    }`}
                    onClick={() => {
                      onSelectPhasAndStage(r, null, sample);
                    }}
                  >
                    {columns.map((col) => {
                      const value = (r as any)[col];

                      const isBestSample =
                        col.toLowerCase().includes("best") &&
                        col.toLowerCase().includes("sample");

                      if (isBestSample && typeof value === "string") {
                        return (
                          <td key={col} className={getCellClass(col)}>
                            <span
                              style={{
                                color: "#007bff",
                                cursor: "pointer",
                              }}
                              onClick={(e) => {
                                e.stopPropagation();

                                const stage =
                                  getStageFromSample(value);

                                onSelectPhasAndStage(
                                  r,
                                  stage,
                                  sample
                                );
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
                );
              })}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
}

/* Helpers */
function formatHeader(col: string) {
  return col
    .replace(/_/g, " ")
    .replace(/\b\w/g, (c) => c.toUpperCase());
}

function formatValue(col: string, value: any) {
  if (value == null) return "-";

  if (
    col.toLowerCase().includes("pvalue") &&
    typeof value === "number"
  )
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
  if (key.includes("abundance")) return styles.abundance;
  if (key.includes("note")) return styles.note;

  return "";
}
```
#### IGVViewer.tsx
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

export default function IGVViewer({ sample, locus }: Props) {
  const igvContainerRef = useRef<HTMLDivElement | null>(null);
  const igvBrowserRef = useRef<any>(null);
  const isReadyRef = useRef(false);

  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // -----------------------------
  // 🔥 LOAD IGV (when sample changes)
  // -----------------------------
  useEffect(() => {
    if (!igvContainerRef.current || !sample) return;

    // 🔥 destroy previous IGV instance
    if (igvBrowserRef.current) {
      igvBrowserRef.current.destroy?.();
      igvBrowserRef.current = null;
      isReadyRef.current = false;
    }

    const loadIGV = async () => {
      setLoading(true);
      setError(null);

      try {
        console.log("Loading IGV for sample:", sample);

        const res = await fetch(`${API_BASE}/api/tracks/igv/${sample}`);
        const config = await res.json();

        console.log("IGV config:", config);

        const options = {
          genome: {
            fastaURL: API_BASE + config.genome.fastaURL,
            indexURL: API_BASE + config.genome.indexURL,
          },
          tracks: config.tracks.map((t: any) => ({
            ...t,
            url: API_BASE + t.url,
          })),
        };

        const browser = await igv.createBrowser(
          igvContainerRef.current!,
          options
        );

        igvBrowserRef.current = browser;

        // 🔥 Wait until IGV fully initializes
        setTimeout(() => {
          isReadyRef.current = true;
          console.log("IGV is ready");

          // 🔥 Initial jump (default PHAS22-1)
          if (locus) {
            browser.search(
              `${locus.chrom}:${locus.start}-${locus.end}`
            );
          }
        }, 500);

      } catch (err: any) {
        console.error("IGV load error:", err);
        setError(err.message || "Failed to load IGV");
      } finally {
        setLoading(false);
      }
    };

    loadIGV();
  }, [sample]);

  // -----------------------------
  // 🔥 UPDATE LOCUS (on click)
  // -----------------------------
  useEffect(() => {
    if (!igvBrowserRef.current || !locus) return;

    if (!isReadyRef.current) {
      console.log("IGV not ready yet, skipping locus update...");
      return;
    }

    console.log("Navigating to locus:", locus);

    try {
      igvBrowserRef.current.search(
        `${locus.chrom}:${locus.start}-${locus.end}`
      );
    } catch (e) {
      console.warn("IGV search failed:", e);
    }
  }, [locus]);

  return (
    <div className={styles.container}>
      <div className={styles.header}>
        Genome Browser (IGV) — {sample}
      </div>

      {loading && (
        <div className={styles.loading}>Loading IGV...</div>
      )}

      {error && (
        <div className={styles.error}>{error}</div>
      )}

      <div
        ref={igvContainerRef}
        className={styles.viewer}
        style={{
          display: loading || error ? "none" : "block",
        }}
      />
    </div>
  );
}
```
#### IGVViewer.module.css
```sh
/* Container wrapper for the IGV panel */
.container {
  margin-top: 24px;
  background: #ffffff;
  border-radius: 12px;
  border: 1px solid #e1e5ea;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);

  overflow: visible;   /* 🔥 important */
}

/* Header (optional title bar) */
.header {
  padding: 10px 16px;
  background: #f8f9fb;
  border-bottom: 1px solid #e1e5ea;
  font-size: 14px;
  font-weight: 600;
  color: #2c3e50;
}

/* IGV rendering area */
.viewer {
  height: 600px;   /* 🔥 increased */
  width: 100%;
}

/* Optional loading state */
.loading {
  height: 600px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #7f8c8d;
  font-size: 14px;
}

/* Optional error state */
.error {
  padding: 16px;
  color: #c0392b;
  background: #fdecea;
  font-size: 14px;
}

/* 🔥 Locus label (current region display) */
.locusLabel {
  padding: 8px 16px;
  font-size: 13px;
  color: #34495e;
  background: #f4f6f8;
  border-bottom: 1px solid #e1e5ea;

  font-family: monospace;   /* looks better for coordinates */
}
```
---
