3.30.2026
#### PHAS_interface_Workflow_raw_3
> This is a raw file for building a foundation for the interface. This is a new version of the design and strategy.
### Problem (Why UI is messy):
- my previous structure is flat and improvised (NOT Modular + layered architecture
- Everything is inside:
    - main.py
    - one React table
- No separation of:
    - logic
    - API
    - UI components
- Took time, UI is messy

# Version 2: 
## Phase 1: Recreate Foundation
1. Directory Structures 
2. Establish Connection
    - work on backend files
    - work on frontend files
3. Import Phas Loci Tables

Phase 2: Establish Phasing Pattern panel
Phase 3: Estblish IGV Panel
Phase 4: Confidence Evaluation Panel

# Phase 1: Recreate Foundation
# 1. Create Folders 
```sh
cd "/Volumes/Install macOS Mojave/Vina"
mkdir PHASER #new folder
#NOTE: previous PHAS folder was renamed to PHAS_2
```
## 1A. Backend folders 


```sh
backend/
 ├── main.py
 ├── requirements.txt
 └── app/
      ├── __init__.py
      ├── config.py
      ├── routers/
      ├── services/
      ├── models/
      ├── utils/
      └── __pycache__/
```
- ##### Create main backend structures
```sh
cd backend
touch main.py
touch requirements.txt
mkdir app
```
- ##### Inside App folder, create files and subfolders
    Add `app` directory. Inside`app`, add the needed subfolders. 
```sh
mkdir app

# Inside app
mkdir routers services models utils
touch __init__.py config.py # Create empty files (no content yet, just a place holder)

# Create essential files
cd backend
touch app/__init__.py
touch app/config.py
touch app/routers/__init__.py
touch app/services/__init__.py
touch app/models/__init__.py
touch app/utils/__init__.py

# Create first PHAS files
cd backend
touch app/routers/phas.py
touch app/services/phas_service.py
touch app/models/phas_model.py
touch app/utils/file_utils.py
```


---
## 4A Create Backend Files

- ### main.py
```sh
cd backend
vi main.py
```
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
    allow_origins=[
        "http://163.221.246.151:5174",
        "http://localhost:5174"
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
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
- ### `phas.py` (app/router/phas.py)
```sh
"""
PHAS API Router
"""

from fastapi import APIRouter, HTTPException, Query
from typing import Optional

# CHANGE: service + models
from ..services.phas_service import phas_service
from ..models.phas import PHASListResponse, PHASSearchResponse, PHASRecord, DisplayRegion

router = APIRouter(prefix="/phas", tags=["PHAS"])


@router.get("", response_model=PHASListResponse)
async def get_all_phas():
    """Get all PHAS loci"""
    records = phas_service.get_all()
    return PHASListResponse(total=len(records), records=records)


@router.get("/search", response_model=PHASSearchResponse)
async def search_phas(q: str = Query(..., min_length=1, description="Search query")):
    """Search PHAS loci by ID or annotation"""
    records = phas_service.search(q)
    return PHASSearchResponse(query=q, total=len(records), records=records)


@router.get("/{phas_id}", response_model=PHASRecord)
async def get_phas_by_id(phas_id: str):
    """Get PHAS locus by ID"""
    record = phas_service.get_by_id(phas_id)
    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")
    return record


@router.get("/{phas_id}/region", response_model=DisplayRegion)
async def get_display_region(phas_id: str):
    """Get display region for IGV"""
    region = phas_service.get_display_region(phas_id)
    if not region:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")
    return DisplayRegion(**region)
```
- ### `phas.py` (app/router/phas.py) ver 2.
- update to include all columns in the table
```sh
"""
PHAS API Router
"""

from fastapi import APIRouter, HTTPException, Query

from ..services.phas_service import phas_service
from ..models.phas import (
    PHASListResponse,
    PHASSearchResponse,
    PHASRecord,
    DisplayRegion,
)

router = APIRouter(prefix="/phas", tags=["PHAS"])


# �� GET ALL
@router.get("", response_model=PHASListResponse)
async def get_all_phas():
    """Get all PHAS loci"""
    records = phas_service.get_all()
    return PHASListResponse(total=len(records), records=records)


# 🔷 SEARCH
@router.get("/search", response_model=PHASSearchResponse)
async def search_phas(q: str = Query(..., min_length=1)):
    """Search PHAS loci by ID or annotation"""
    records = phas_service.search(q)
    return PHASSearchResponse(query=q, total=len(records), records=records)


# 🔷 GET BY ID
@router.get("/{phas_id}", response_model=PHASRecord)
async def get_phas_by_id(phas_id: str):
    """Get PHAS locus by ID"""
    record = phas_service.get_by_id(phas_id)
    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")
    return record


# 🔷 DISPLAY REGION (FIXED HERE)
@router.get("/{phas_id}/region", response_model=DisplayRegion)
async def get_display_region(phas_id: str):
    """Get display region for IGV"""
    record = phas_service.get_by_id(phas_id)

    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")

    padding = 200

    return DisplayRegion(
        chrom=record.chromosome,  # ✅ FIXED
        start=max(0, record.start - padding),
        end=record.end + padding,
    )
(base) okamuralab@rnalab routers % ls
__init__.py     __pycache__     blat.py         files.py        phas.py         tracks.py
(base) okamuralab@rnalab routers % cat  phas.py  
"""
PHAS API Router
"""

from fastapi import APIRouter, HTTPException, Query

from ..services.phas_service import phas_service
from ..models.phas import (
    PHASListResponse,
    PHASSearchResponse,
    PHASRecord,
    DisplayRegion,
)

router = APIRouter(prefix="/phas", tags=["PHAS"])


# �� GET ALL
@router.get("", response_model=PHASListResponse)
async def get_all_phas():
    """Get all PHAS loci"""
    records = phas_service.get_all()
    return PHASListResponse(total=len(records), records=records)


# 🔷 SEARCH
@router.get("/search", response_model=PHASSearchResponse)
async def search_phas(q: str = Query(..., min_length=1)):
    """Search PHAS loci by ID or annotation"""
    records = phas_service.search(q)
    return PHASSearchResponse(query=q, total=len(records), records=records)


# 🔷 GET BY ID
@router.get("/{phas_id}", response_model=PHASRecord)
async def get_phas_by_id(phas_id: str):
    """Get PHAS locus by ID"""
    record = phas_service.get_by_id(phas_id)
    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")
    return record


# 🔷 DISPLAY REGION (FIXED HERE)
@router.get("/{phas_id}/region", response_model=DisplayRegion)
async def get_display_region(phas_id: str):
    """Get display region for IGV"""
    record = phas_service.get_by_id(phas_id)

    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")

    padding = 200

    return DisplayRegion(
        chrom=record.chromosome,  # ✅ FIXED
        start=max(0, record.start - padding),
        end=record.end + padding,
    )
```
- ### `phas_service.py` ( /app/services/phas_service.py)
```sh
"""
PHAS Data Service
"""

import pandas as pd
from typing import List, Optional

from ..config import PHAS_DATA_FILE
from ..models.phas import PHASRecord


class PHASService:
    """Service for PHAS data operations"""

    def __init__(self):
        self._data: Optional[pd.DataFrame] = None

    # 🔷 LOAD DATA
    def _load_data(self) -> pd.DataFrame:
        """Load PHAS data from TSV file"""
        if self._data is None:
            self._data = pd.read_csv(PHAS_DATA_FILE, sep='\t')
        return self._data

    # 🔷 CREATE RECORD
    def _make_record(self, row) -> PHASRecord:
        """Create PHASRecord from a DataFrame row"""
        return PHASRecord(
            phas_id=row['PHASID'],
            chrom=row['Chr_ID'],
            start=int(row['Start_Pos']),
            end=int(row['End_Pos']),
            length=int(row['Locus_Len']),
            pvalue=float(row['Pvalue']),
            phase=int(row['Phase']),
            abundance=float(row['Abudance']),  # keep original typo
            score=float(row['Max_Region_Phase_Score']),
            note=row['Note'],
            best_region=row['Best_Region'],
            best_sample=row['Best_Sample'],
        )

    # 🔷 GET ALL
    def get_all(self) -> List[PHASRecord]:
        """Get all PHAS records"""
        df = self._load_data()
        return [self._make_record(row) for _, row in df.iterrows()]

    # 🔷 GET BY ID
    def get_by_id(self, phas_id: str) -> Optional[PHASRecord]:
        """Get PHAS record by ID"""
        df = self._load_data()
        filtered = df[df['PHASID'] == phas_id]

        if filtered.empty:
            return None

        return self._make_record(filtered.iloc[0])

    # 🔷 SEARCH
    def search(self, query: str) -> List[PHASRecord]:
        """Search PHAS by ID, chromosome, or note"""
        df = self._load_data()
        query_lower = query.lower().strip()

        mask = (
            df['PHASID'].str.lower().str.contains(query_lower, na=False) |
            df['Chr_ID'].str.lower().str.contains(query_lower, na=False) |
            df['Note'].str.lower().str.contains(query_lower, na=False)
        )

        filtered = df[mask]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 DISPLAY REGION (IGV)
    def get_display_region(self, phas_id: str) -> Optional[dict]:
        """Return genomic region for visualization"""
        record = self.get_by_id(phas_id)

        if not record:
            return None

        padding = 200

        return {
            "chrom": record.chrom,
            "start": max(0, record.start - padding),
            "end": record.end + padding,
        }

    # 🔷 OPTIONAL: FILTER BY PVALUE
    def filter_by_pvalue(self, threshold: float) -> List[PHASRecord]:
        """Filter PHAS loci by p-value"""
        df = self._load_data()
        filtered = df[df['Pvalue'] <= threshold]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 OPTIONAL: TOP BY SCORE
    def get_top_by_score(self, n: int = 10) -> List[PHASRecord]:
        """Get top PHAS loci by phasing score"""
        df = self._load_data()
        sorted_df = df.sort_values(by='Max_Region_Phase_Score', ascending=False)
        return [self._make_record(row) for _, row in sorted_df.head(n).iterrows()]


# 🔷 SINGLETON INSTANCE
phas_service = PHASService()
```
#### NOTE: 
- Define `app/config.py`. Make sure phas loci data is present
```sh
from pathlib import Path
PHAS_DATA_FILE = Path("data/phas_loci.tsv")
```

- ### config.py
```sh
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
### NOTE:

- Ensure `/Volumes/Install macOS Mojave/Vina/PHASER/data/phas_loci.tsv` is present
- created dummy file for now `data/library_info.tsv`
```sh
library_ID	Sample
TEST1	Sample1
TEST2	Sample2
```
- ### phas.py (app/models/phas.py)
```sh
"""PHAS Data Models"""

from pydantic import BaseModel
from typing import List, Optional


# 🔷 MAIN RECORD
class PHASRecord(BaseModel):
    """Single PHAS locus record"""
    phas_id: str
    chrom: str
    start: int
    end: int
    length: int
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


# 🔷 TRACK FILE 
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
- ### track_service.py
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

- ### tracks.py (app/routers/tracks.py)
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
---
03.31.2026
# Work on Frontend
```sh
cd frontend/src/components
```
## A. Header
- #### A1. Header.module.css
```sh
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 28px;
  background: linear-gradient(135deg, #1f2d3a 0%, #2c3e50 100%);
  color: white;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

/* LEFT SIDE */
.logo {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.title {
  margin: 0;
  font-size: 22px;
  font-weight: 700;
  letter-spacing: 1.5px;
}

.subtitle {
  font-size: 12px;
  opacity: 0.75;
}

/* RIGHT SIDE */
.actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

/* SHARE BUTTON */
.shareButton {
  padding: 8px 16px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s ease;
}

.shareButton:hover {
  background: #2980b9;
  transform: translateY(-1px);
}

/* EXPORT BUTTONS */
.exportButton {
  padding: 7px 12px;
  background: rgba(255, 255, 255, 0.12);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.exportButton:hover {
  background: rgba(255, 255, 255, 0.22);
  border-color: rgba(255, 255, 255, 0.45);
  transform: translateY(-1px);
}

.exportButton:active,
.shareButton:active {
  transform: scale(0.97);
}

.exportButton:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

- #### A2. Header.tsx
```sh
import { useState } from 'react';
import styles from './Header.module.css';

export function Header(): JSX.Element {
  const [copied, setCopied] = useState(false);

  const handleShareClick = async () => {
    // Temporary placeholder
    navigator.clipboard.writeText(window.location.href);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  const handleExportClick = (format: 'svg' | 'png') => {
    // Placeholder for future IGV export
    console.log(`Export ${format} clicked`);
  };

  return (
    <header className={styles.header}>
      <div className={styles.logo}>
        <h1 className={styles.title}>PHASER</h1>
        <span className={styles.subtitle}>
          PHAS Evaluation Resource
        </span>
      </div>

      <div className={styles.actions}>
        <button
          className={styles.exportButton}
          onClick={() => handleExportClick('svg')}
          title="Save as SVG"
        >
          SVG
        </button>

        <button
          className={styles.exportButton}
          onClick={() => handleExportClick('png')}
          title="Save as PNG"
        >
          PNG
        </button>

        <button
          className={styles.shareButton}
          onClick={handleShareClick}
        >
          {copied ? 'Copied!' : 'Share URL'}
        </button>
      </div>
    </header>
  );
}
```

## B. PHASTable

- #### B1. PHASTable.module.css
```sh
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

.row.selected {
  background: #e8f4fd;
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
- #### B2. PHASTable.tsx
```sh
import { useState } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";

type SortField = "phas_id" | "phase" | "abundance";
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
  const [sortField, setSortField] = useState<SortField>("phas_id");
  const [sortOrder, setSortOrder] = useState<SortOrder>("asc");

  const sortedRecords = [...records].sort((a, b) => {
    const valA = a[sortField];
    const valB = b[sortField];

    if (valA < valB) return sortOrder === "asc" ? -1 : 1;
    if (valA > valB) return sortOrder === "asc" ? 1 : -1;
    return 0;
  });

  const handleSort = (field: SortField) => {
    if (field === sortField) {
      setSortOrder(sortOrder === "asc" ? "desc" : "asc");
    } else {
      setSortField(field);
      setSortOrder("asc");
    }
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
          <table>
            <thead>
              <tr>
                <th onClick={() => handleSort("phas_id")}>PHAS ID</th>
                <th>Chr</th>
                <th>Start</th>
                <th>End</th>
                <th onClick={() => handleSort("phase")}>Phase</th>
                <th>P-value</th>
                <th onClick={() => handleSort("abundance")}>Abundance</th>
                <th>Note</th>
              </tr>
            </thead>

            <tbody>
              {sortedRecords.map((r) => (
                <tr
                  key={r.phas_id}
                  onClick={() => onSelect(r)}
                  className={selectedId === r.phas_id ? styles.selected : ""}
                >
                  <td>{r.phas_id}</td>
                  <td>{r.chromosome}</td>
                  <td>{r.start}</td>
                  <td>{r.end}</td>
                  <td>{r.phase}</td>
                  <td>{r.pvalue.toExponential(2)}</td>
                  <td>{r.abundance}</td>
                  <td>{r.note}</td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
}
```

### 4C PHASTable.tsx ver 2.

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
## C. Searchbox
- #### C2. SearchBox.tsx

```sh
// Search Box Component (PHAS version)

import styles from './SearchBox.module.css';

interface SearchBoxProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
}

export function SearchBox({ value, onChange, placeholder }: SearchBoxProps): JSX.Element {
  return (
    <div className={styles.container}>
      {/* Search Icon */}
      <div className={styles.searchIcon}>
        <svg
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          strokeWidth="2"
        >
          <circle cx="11" cy="11" r="8" />
          <path d="M21 21l-4.35-4.35" />
        </svg>
      </div>

      {/* Input */}
      <input
        type="text"
        className={styles.input}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={
          placeholder ||
          "Search PHAS ID or chromosome (e.g., PHAS22-1, Chr1)"
        }
      />

      {/* Clear Button */}
      {value && (
        <button
          className={styles.clearButton}
          onClick={() => onChange('')}
          title="Clear search"
        >
          <svg
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            strokeWidth="2"
          >
            <path d="M18 6L6 18M6 6l12 12" />
          </svg>
        </button>
      )}
    </div>
  );
}
```
- #### SearchBox.module.css
```sh
.container {
  position: relative;
  display: flex;
  align-items: center;
  margin: 16px 24px;
}

/* Search Icon */
.searchIcon {
  position: absolute;
  left: 14px;
  color: #7f8c8d;
  pointer-events: none;
  display: flex;
  align-items: center;
}

/* Input Field */
.input {
  width: 100%;
  padding: 12px 44px 12px 42px; /* space for icon + clear button */
  font-size: 14px;
  border: 2px solid #e3e6ea;
  border-radius: 8px;
  outline: none;
  background: #ffffff;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.input:focus {
  border-color: #3498db;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.12);
}

/* Placeholder */
.input::placeholder {
  color: #a0aab4;
  font-size: 13px;
}

/* Clear Button */
.clearButton {
  position: absolute;
  right: 12px;
  padding: 5px;
  background: none;
  border: none;
  color: #95a5a6;
  cursor: pointer;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.clearButton:hover {
  background: #ecf0f1;
  color: #5d6d7e;
}

/* Active click feedback */
.clearButton:active {
  transform: scale(0.9);
}
```
## D. App.tsx (integrate components)
```sh
// PHAS Main Application

import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";

import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  // Data state
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  // UI state
  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // Load PHAS data (replaces useEVEData)
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data);
        setFilteredRecords(data);
      })
      .catch(() => {
        setError("Failed to load PHAS data");
      })
      .finally(() => setLoading(false));
  }, []);

  // Search filtering (replaces useEVEData filter logic)
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) => {
      const id = r.phas_id?.toLowerCase() || "";
      const chr = r.chromosome?.toLowerCase() || "";

      return id.includes(q) || chr.includes(q);
    });

    setFilteredRecords(filtered);
  }, [searchQuery, phasRecords]);

  // Handle PHAS selection (simplified from handleEVESelect)
  const handleSelect = (phas: PHASLocus) => {
    setSelectedPhas(phas);
  };

  return (
    <div className="app">
      <Header
        currentState={{}} // placeholder (no URL system yet)
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID or chromosome..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={handleSelect}
          loading={loading}
        />

        {error && <div className="error-message">{error}</div>}

        {selectedPhas && (
          <div style={{ margin: "20px" }}>
            <h3>Selected PHAS</h3>
            <p><strong>ID:</strong> {selectedPhas.phas_id}</p>
            <p><strong>Chr:</strong> {selectedPhas.chromosome}</p>
            <p><strong>Range:</strong> {selectedPhas.start} - {selectedPhas.end}</p>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
## E. types/phas.ts
- #### phas.ts 
```sh
// PHAS Data Types

export interface PHASLocus {
  phas_id: string;
  chromosome: string;
  start: number;
  end: number;

  locus_length: number;
  lib_num: number;

  pvalue: number;
  phas_hit: number;
  phase: number;

  avg_hit_pos: number;
  phas_ratio: number;
  abundance: number;

  max_phase_score: number;
  note: string;

  best_region: string;
  best_sample: string;
}

// API response (list of PHAS loci)
export interface PHASListResponse {
  total: number;
  records: PHASLocus[];
}

// Search response (optional, future use)
export interface PHASSearchResponse {
  query: string;
  total: number;
  records: PHASLocus[];
}

// Genomic region (for future IGV / visualization)
export interface PHASRegion {
  chromosome: string;
  start: number;
  end: number;
  locus_string: string;
}
```
## E. types/phas.ts
- #### phas.ts 
```sh
// PHAS Data Types

export interface PHASLocus {
  phas_id: string;
  chromosome: string;
  start: number;
  end: number;

  locus_length: number;
  lib_num: number;

  pvalue: number;
  phas_hit: number;
  phase: number;

  avg_hit_pos: number;
  phas_ratio: number;
  abundance: number;

  max_phase_score: number;
  note: string;

  best_region: string;
  best_sample: string;
}

// API response (list of PHAS loci)
export interface PHASListResponse {
  total: number;
  records: PHASLocus[];
}

// Search response (optional, future use)
export interface PHASSearchResponse {
  query: string;
  total: number;
  records: PHASLocus[];
}

// Genomic region (for future IGV / visualization)
export interface PHASRegion {
  chromosome: string;
  start: number;
  end: number;
  locus_string: string;
}
```
## Test:
```sh
#Backend
cd backend 
uvicorn main:app --reload --host 0.0.0.0 --port 8001
#Frontend
cd frontend
npm run dev -- --host --port 5174
```


### Error:
![alt text](image-14.png)
ReferenceError: Can't find variable: fetchPHASLoci



App@http://163.221.246.151:5174/src/App.tsx:24:49
ErrorBoundary@http://163.221.246.151:5174/src/components/ErrorBoundary/ErrorBoundary.tsx:7:

![alt text](image-15.png)

#### Fix:
- Create the missing file:
`src/services/phasService.ts`
- ### phasService.ts

```sh
export async function fetchPHASLoci() {
  const res = await fetch("http://localhost:8001/api/phas");

  if (!res.ok) {
    throw new Error("Failed to fetch PHAS data");
  }

  return res.json();
}
```
Then in `App.tsx`
```sh
import { fetchPHASLoci } from "./services/phasService";
```
### App.tsx ver. 2
```sh
// PHAS Main Application

import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  // Data state
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  // UI state
  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // Load PHAS data (replaces useEVEData)
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data);
        setFilteredRecords(data);
      })
      .catch(() => {
        setError("Failed to load PHAS data");
      })
      .finally(() => setLoading(false));
  }, []);

  // Search filtering (replaces useEVEData filter logic)
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) => {
      const id = r.phas_id?.toLowerCase() || "";
      const chr = r.chromosome?.toLowerCase() || "";

      return id.includes(q) || chr.includes(q);
    });

    setFilteredRecords(filtered);
  }, [searchQuery, phasRecords]);

  // Handle PHAS selection (simplified from handleEVESelect)
  const handleSelect = (phas: PHASLocus) => {
    setSelectedPhas(phas);
  };

  return (
    <div className="app">
      <Header
        currentState={{}} // placeholder (no URL system yet)
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID or chromosome..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={handleSelect}
          loading={loading}
        />

        {error && <div className="error-message">{error}</div>}

        {selectedPhas && (
          <div style={{ margin: "20px" }}>
            <h3>Selected PHAS</h3>
            <p><strong>ID:</strong> {selectedPhas.phas_id}</p>
            <p><strong>Chr:</strong> {selectedPhas.chromosome}</p>
            <p><strong>Range:</strong> {selectedPhas.start} - {selectedPhas.end}</p>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```

#### Solved!
![alt text](image-16.png)

#### Error:
[Error] Failed to load resource: Could not connect to the server. (phas, line 0)
[Error] Unhandled Promise Rejection: TypeError: Load failed
[Log] Export png clicked (Header.tsx, line 29)
[Error] Unhandled Promise Rejection: TypeError: undefined is not an object (evaluating 'navigator.clipboard.writeText')

#### Fix
- Change the fetch URL.
- In `phasService.ts`:,
```sh
const res = await fetch("http://163.221.246.151:8001/api/phas");
```

#### Error:

TypeError: Spread syntax requires ...iterable[Symbol.iterator] to be a function


PHASTable@http://163.221.246.151:5174/src/components/PHASTable/PHASTable.tsx:22:13
div
App@http://163.221.246.151:5174/src/App.tsx:25:49
ErrorBoundary@http://163.221.246.151:5174/src/components/ErrorBoundary/ErrorBoundary.tsx:7:10

04.01.2026
#### Fix
- In `App.tsx`,

change 
```sh
fetchPHASLoci().then((data) => {
  setPhasRecords(data);
  setFilteredRecords(data);
});
```
to:
```sh
fetchPHASLoci().then((data) => {
  setPhasRecords(data.records);
  setFilteredRecords(data.records);
});
```

### App.tsx ver. 3
```sh
// PHAS Main Application

import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";

import "./App.css";

function App(): JSX.Element {
  // Data state
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);

  // UI state
  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // Load PHAS data (replaces useEVEData)
  useEffect(() => {
    setLoading(true);
    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);
      })
      .catch(() => {
        setError("Failed to load PHAS data");
      })
      .finally(() => setLoading(false));
  }, []);

  // Search filtering (replaces useEVEData filter logic)
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    const filtered = phasRecords.filter((r) => {
      const id = r.phas_id?.toLowerCase() || "";
      const chr = r.chromosome?.toLowerCase() || "";

      return id.includes(q) || chr.includes(q);
    });

    setFilteredRecords(filtered);
  }, [searchQuery, phasRecords]);

  // Handle PHAS selection (simplified from handleEVESelect)
  const handleSelect = (phas: PHASLocus) => {
    setSelectedPhas(phas);
  };

  return (
    <div className="app">
      <Header
        currentState={{}} // placeholder (no URL system yet)
        onExportSVG={() => {}}
        onExportPNG={() => {}}
        isIGVReady={false}
      />

      <main className="main">
        <SearchBox
          value={searchQuery}
          onChange={setSearchQuery}
          placeholder="Search PHAS ID or chromosome..."
        />

        <PHASTable
          records={filteredRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelect={handleSelect}
          loading={loading}
        />

        {error && <div className="error-message">{error}</div>}

        {selectedPhas && (
          <div style={{ margin: "20px" }}>
            <h3>Selected PHAS</h3>
            <p><strong>ID:</strong> {selectedPhas.phas_id}</p>
            <p><strong>Chr:</strong> {selectedPhas.chromosome}</p>
            <p><strong>Range:</strong> {selectedPhas.start} - {selectedPhas.end}</p>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```

#### Solved!
![alt text](image-17.png)

#### Missing Parts
- Chr column data missing
- Search Bar function is not connected to the table when searching

#### Fix:
- Fix data mapping issue or mismatch in backend and mapping files
    - In backend `phas_service.py` and `models/phas.py`, change 
"chrom" to "chromosome" and confirm the line `<td>{r.chromosome}</td>` exists in frontend `PHASTable.tsx`

## phas_service.py ver 2
```sh
"""
PHAS Data Service
"""

import pandas as pd
from typing import List, Optional

from ..config import PHAS_DATA_FILE
from ..models.phas import PHASRecord


class PHASService:
    """Service for PHAS data operations"""

    def __init__(self):
        self._data: Optional[pd.DataFrame] = None

    # 🔷 LOAD DATA
    def _load_data(self) -> pd.DataFrame:
        """Load PHAS data from TSV file"""
        if self._data is None:
            self._data = pd.read_csv(PHAS_DATA_FILE, sep='\t')
        return self._data

    # 🔷 CREATE RECORD
    def _make_record(self, row) -> PHASRecord:
        """Create PHASRecord from a DataFrame row"""
        return PHASRecord(
            phas_id=row['PHASID'],
            chromosome=row['Chr_ID'],
            start=int(row['Start_Pos']),
            end=int(row['End_Pos']),
            length=int(row['Locus_Len']),
            pvalue=float(row['Pvalue']),
            phase=int(row['Phase']),
            abundance=float(row['Abudance']),  # keep original typo
            score=float(row['Max_Region_Phase_Score']),
            note=row['Note'],
            best_region=row['Best_Region'],
            best_sample=row['Best_Sample'],
        )

    # 🔷 GET ALL
    def get_all(self) -> List[PHASRecord]:
        """Get all PHAS records"""
        df = self._load_data()
        return [self._make_record(row) for _, row in df.iterrows()]

    # 🔷 GET BY ID
    def get_by_id(self, phas_id: str) -> Optional[PHASRecord]:
        """Get PHAS record by ID"""
        df = self._load_data()
        filtered = df[df['PHASID'] == phas_id]

        if filtered.empty:
            return None

        return self._make_record(filtered.iloc[0])

    # 🔷 SEARCH
    def search(self, query: str) -> List[PHASRecord]:
        """Search PHAS by ID, chromosome, or note"""
        df = self._load_data()
        query_lower = query.lower().strip()

        mask = (
            df['PHASID'].str.lower().str.contains(query_lower, na=False) |
            df['Chr_ID'].str.lower().str.contains(query_lower, na=False) |
            df['Note'].str.lower().str.contains(query_lower, na=False)
        )

        filtered = df[mask]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 DISPLAY REGION (IGV)
    def get_display_region(self, phas_id: str) -> Optional[dict]:
        """Return genomic region for visualization"""
        record = self.get_by_id(phas_id)

        if not record:
            return None

        padding = 200

        return {
            "chrom": record.chrom,
            "start": max(0, record.start - padding),
            "end": record.end + padding,
        }

    # 🔷 OPTIONAL: FILTER BY PVALUE
    def filter_by_pvalue(self, threshold: float) -> List[PHASRecord]:
        """Filter PHAS loci by p-value"""
        df = self._load_data()
        filtered = df[df['Pvalue'] <= threshold]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 OPTIONAL: TOP BY SCORE
    def get_top_by_score(self, n: int = 10) -> List[PHASRecord]:
        """Get top PHAS loci by phasing score"""
        df = self._load_data()
        sorted_df = df.sort_values(by='Max_Region_Phase_Score', ascending=False)
        return [self._make_record(row) for _, row in sorted_df.head(n).iterrows()]


# 🔷 SINGLETON INSTANCE
phas_service = PHASService()
```
#### app/models/phas.py ver. 2
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
#### app/models/phas.py  ver. 3
- update to include all columns in the table 
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
(base) okamuralab@rnalab models % ls
__init__.py     __pycache__     phas.py
(base) okamuralab@rnalab models % cat phas.py
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
## `phas_service.py ver. 3
- update to include all columns in the table

```sh
"""
PHAS Data Service
"""

import pandas as pd
from typing import List, Optional

from ..config import PHAS_DATA_FILE
from ..models.phas import PHASRecord


class PHASService:
    """Service for PHAS data operations"""

    def __init__(self):
        self._data: Optional[pd.DataFrame] = None

    # 🔷 LOAD DATA
    def _load_data(self) -> pd.DataFrame:
        """Load PHAS data from TSV file"""
        if self._data is None:
            self._data = pd.read_csv(PHAS_DATA_FILE, sep="\t")
        return self._data

    # 🔷 CREATE RECORD (FIXED + COMPLETE)
    def _make_record(self, row) -> PHASRecord:
        """Create PHASRecord from a DataFrame row"""
        return PHASRecord(
            phas_id=row["PHASID"],
            chromosome=row["Chr_ID"],

            start=int(row["Start_Pos"]),
            end=int(row["End_Pos"]),
            length=int(row["Locus_Len"]),

            # 🔥 YOUR EXTRA COLUMNS (SAFE ACCESS)
            lib_num=int(row.get("Lib_Num", 0)),
            phas_hit=int(row.get("Phas_Hit", 0)),
            avg_hit_pos=float(row.get("Average_HitPos", 0)),
            phas_ratio=float(row.get("Phas_Ratio", 0)),

            pvalue=float(row["Pvalue"]),
            phase=int(row["Phase"]),
            abundance=float(row["Abudance"]),  # keep original typo

            score=float(row["Max_Region_Phase_Score"]),

            note=row.get("Note", ""),
            best_region=row.get("Best_Region", ""),
            best_sample=row.get("Best_Sample", ""),
        )

    # 🔷 GET ALL
    def get_all(self) -> List[PHASRecord]:
        """Get all PHAS records"""
        df = self._load_data()
        return [self._make_record(row) for _, row in df.iterrows()]

    # 🔷 GET BY ID
    def get_by_id(self, phas_id: str) -> Optional[PHASRecord]:
        """Get PHAS record by ID"""
        df = self._load_data()
        filtered = df[df["PHASID"] == phas_id]

        if filtered.empty:
            return None

        return self._make_record(filtered.iloc[0])

    # 🔷 SEARCH
    def search(self, query: str) -> List[PHASRecord]:
        """Search PHAS by ID, chromosome, or note"""
        df = self._load_data()
        query_lower = query.lower().strip()

        mask = (
            df["PHASID"].astype(str).str.lower().str.contains(query_lower, na=False)
            | df["Chr_ID"].astype(str).str.lower().str.contains(query_lower, na=False)
            | df["Note"].astype(str).str.lower().str.contains(query_lower, na=False)
        )

        filtered = df[mask]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 DISPLAY REGION (NOT USED ANYMORE BY ROUTER BUT KEEP SAFE)
    def get_display_region(self, phas_id: str) -> Optional[dict]:
        """Return genomic region for visualization"""
        record = self.get_by_id(phas_id)

        if not record:
            return None

        padding = 200

        return {
            "chrom": record.chromosome,  # ✅ FIXED
            "start": max(0, record.start - padding),
            "end": record.end + padding,
        }

    # 🔷 OPTIONAL: FILTER BY PVALUE
    def filter_by_pvalue(self, threshold: float) -> List[PHASRecord]:
        df = self._load_data()
        filtered = df[df["Pvalue"] <= threshold]
        return [self._make_record(row) for _, row in filtered.iterrows()]

    # 🔷 OPTIONAL: TOP BY SCORE
    def get_top_by_score(self, n: int = 10) -> List[PHASRecord]:
        df = self._load_data()
        sorted_df = df.sort_values(
            by="Max_Region_Phase_Score", ascending=False
        )
        return [
            self._make_record(row)
            for _, row in sorted_df.head(n).iterrows()
        ]


# 🔷 SINGLETON INSTANCE
phas_service = PHASService()
```

---
##### Solved!
![alt text](image-18.png)

##### Phase 1 Recreate Foundation established.
##### app architecture: 
```sh
PHAS TSV
   ↓
FastAPI (phas_service.py)
   ↓
/api/phas
   ↓
phasService.ts
   ↓
App.tsx
   ↓
PHASTable + SearchBox + Header
```
---
04.01.2026
# PHASE 2: Phasing Pattern Panel

-	Display pre-computed figures from /project/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/PHAS_candidate_sRNA_readInfo_from_bowtie/Hlo_PHAS_20251213/plots/radar_pos5

##### General Steps
1.	Convert pdf files to png files (Frontend cannot read PDF directly)
2.	Store in public folder (frontend/public/phas_plots/*.png
3.	Modify phasing panel.tsx
4.	Modify App.tsx (autoselect search result)
5.	File names must match Phas loci list names 

