PHAS_interface_Workflow_SUMMARY_raw1-16_20261030.md
# Set-Up PHASER
- Create frontend and backend folders
- Add Panels 

## 1. Create Folders

```sh
cd "/Volumes/Install macOS Mojave/Vina"
mkdir PHASER
```
### 1A. Backend folders

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
Add app directory. Insideapp, add the needed subfolders.

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
- #### Create Backend Files 
- main.py
- phas.py (app/router/phas.py)
- phas_service.py ( /app/services/phas_service.py)
- config.py (app/config.py.)
- phas.py (app/models/phas.py)
- track_service.py (app/services/track_service.py)
- tracks.py (app/routers/tracks.py)

### 2A. Frontend Folders
#### A1. src/App.tsx 
#### A2. src/App.css
#### A2. src/components (.tsx and .module.css)***
- Header
- PHASTable
- SearchBox  
- PhasingPatternPanel
- IGVViewer 
- ConfidencePanel
- SizeDistributionPanel
- sRNAAbundancePanel

***Latest scripts can be located below (as of 20261023)

---

# ***Scripts

### 2A. Frontend Folders
##### A1. src/App.tsx 
```sh
import { useEffect, useMemo, useState } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import SRNAAbundancePanel from "./components/sRNAAbundancePanel/sRNAAbundancePanel";
import SizeDistributionPanel from "./components/SizeDistributionPanel/SizeDistributionPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";
import MiSiPi from "./components/MiSiPi/MiSiPi";
import TranscriptomeLevelPage from "./components/TranscriptomeLevelPage/TranscriptomeLevelPage";
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
  const [activePage, setActivePage] = useState<"genome" | "transcriptome">("genome");
  const [phasRecords, setPhasRecords] = useState<PHASLocus[]>([]);
  const [filteredRecords, setFilteredRecords] = useState<PHASLocus[]>([]);
  const [selectedPhas, setSelectedPhas] = useState<PHASLocus | null>(null);
  const [searchQuery, setSearchQuery] = useState("");
  const [loading, setLoading] = useState(false);
  const [availableStages, setAvailableStages] = useState<string[]>([]);
  const [activeStage, setActiveStage] = useState("ALL");
  const [view, setView] = useState<"read5prime" | "register" | "all" | "hide">("all");
  const [selectedSample, setSelectedSample] = useState<string | null>(null);
  const [evaluations, setEvaluations] = useState<Record<string, any>>({});
  const [hydrated, setHydrated] = useState(false);

  // Load PHAS data. The genome-level view remains the default page.
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find(
          (r: PHASLocus) => r.phas_id === "PHAS22-1",
        );

        if (defaultPhas) {
          handleSelectPhasAndStage(defaultPhas);
        }
      })
      .finally(() => setLoading(false));
  }, []);

  // Load shared evaluations.
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error("Failed to load evaluations");
        }
        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error("Evaluation load error:", err);
        setEvaluations({});
        setHydrated(true);
      });
  }, []);

  // Filter PHAS records by the search field.
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) => r.phas_id.toLowerCase().includes(q)),
    );
  }, [searchQuery, phasRecords]);

  // Load available stages for the selected locus.
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(data[selectedPhas.phas_id] || []);
      });
  }, [selectedPhas]);

  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null,
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    if (phas.best_sample) {
      const match = phas.best_sample.match(/N\d+/);
      if (match) derivedSample = match[0];

      const bestStage = getStageFromSample(phas.best_sample);
      if (bestStage) derivedStage = bestStage;
    }

    setSelectedSample(derivedSample || null);
    setActiveStage(derivedStage || "ALL");
    setView("all");
  };

  const enrichedRecords = useMemo(
    () =>
      filteredRecords.map((r) => ({
        ...r,
        __evaluation: evaluations[r.phas_id],
      })),
    [filteredRecords, evaluations],
  );

  return (
    <div className="app">
      <Header activePage={activePage} onPageChange={setActivePage} />

      {activePage === "genome" ? (
        <main className="main">
          <SearchBox
            value={searchQuery}
            onChange={setSearchQuery}
            placeholder="Search PHAS ID..."
            options={phasRecords.map((r) => r.phas_id)}
            onSelect={(phasId) => {
              const phas = phasRecords.find((r) => r.phas_id === phasId);
              if (phas) {
                setSearchQuery(phasId);
                handleSelectPhasAndStage(phas);
              }
            }}
          />

          {hydrated && (
            <PHASTable
              records={enrichedRecords}
              selectedId={selectedPhas?.phas_id || null}
              onSelectPhasAndStage={handleSelectPhasAndStage}
              loading={loading}
            />
          )}

          {selectedPhas && (
            <div className="responsiveLayout">
              <div className="leftPanel">
                <PhasingPatternPanel
                  phasId={selectedPhas.phas_id}
                  stage={activeStage}
                  view={view}
                  allStages={availableStages}
                  setStage={setActiveStage}
                  setView={setView}
                />

                <SizeDistributionPanel
                  phasId={selectedPhas.phas_id}
                  defaultStage={
                    selectedPhas.best_sample
                      ? getStageFromSample(selectedPhas.best_sample) || "ALL"
                      : "ALL"
                  }
                  allStages={availableStages}
                />

                <SRNAAbundancePanel locus={selectedPhas} />

                {selectedSample && (
                  <IGVViewer
                    key={`${selectedSample}-${selectedPhas.phas_id}`}
                    sample={selectedSample}
                    locus={
                      selectedPhas.best_region
                        ? parseRegion(selectedPhas.best_region)
                        : {
                            chrom: selectedPhas.chromosome,
                            start: selectedPhas.start,
                            end: selectedPhas.end,
                          }
                    }
                  />
                )}

                <MiSiPi
                  phasId={selectedPhas.phas_id}
                  stage={activeStage}
                  allStages={availableStages}
                />
              </div>

              <div className="rightPanel">
                <ConfidencePanel
                  locus={selectedPhas}
                  evaluation={evaluations[selectedPhas.phas_id]}
                  onSave={(locusId, data) => {
                    setEvaluations((prev) => ({
                      ...prev,
                      [locusId]: data,
                    }));
                  }}
                />
              </div>
            </div>
          )}
        </main>
      ) : (
        <main className="main">
          <TranscriptomeLevelPage selectedPhas={selectedPhas} />
        </main>
      )}
    </div>
  );
}
```
#### A2. src/App.css

```sh
/* ========================================
   Global Styles
======================================== */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html,
body,
#root {
  width: 100%;
  overflow-x: hidden;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, sans-serif;
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
  padding-bottom: 100px;
}

/* ========================================
   Error Message
======================================== */

.error-message {
  margin: 0 24px 16px;
  padding: 12px 16px;
  background: #fdecea;
  color: #c0392b;
  border-radius: 8px;
  font-size: 14px;
}

/* ========================================
   Scrollbar Styling
======================================== */

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

/* ========================================
   IGV.js Styling
======================================== */

.igv-container {
  font-family: inherit !important;
}

.igv-navbar {
  background: #f8f9fa !important;
  border-bottom: 1px solid #ecf0f1 !important;
}

/* ========================================
   Main Responsive Layout
======================================== */

.responsiveLayout {
  display: flex;
  gap: 20px;
  width: 100%;
  align-items: flex-start;
  padding: 20px;
  overflow-x: auto;
}

/* ========================================
   LEFT CONTENT AREA
======================================== */

.leftPanel {
  flex: 1;
  min-width: 900px;
}

/* ========================================
   CONFIDENCE PANEL
======================================== */

.rightPanel {
  width: 360px;
  min-width: 360px;
  flex-shrink: 0;
}
```
#### A3. src/components (.tsx and .module.css)
##### Header
- Header.tsx 
```sh
import { useState } from "react";
import styles from "./Header.module.css";

type AnalysisPage = "genome" | "transcriptome";

interface HeaderProps {
  activePage: AnalysisPage;
  onPageChange: (page: AnalysisPage) => void;
}

export function Header({ activePage, onPageChange }: HeaderProps): JSX.Element {
  const [copied, setCopied] = useState(false);

  const handleShareClick = async () => {
    const url = window.location.href;

    // Try clipboard API first.
    try {
      await navigator.clipboard.writeText(url);
      setCopied(true);
    } catch {
      // Fallback for non-HTTPS or older browsers.
      const textarea = document.createElement("textarea");
      textarea.value = url;
      document.body.appendChild(textarea);
      textarea.select();

      try {
        document.execCommand("copy");
        setCopied(true);
      } catch (err) {
        console.error("Fallback copy failed:", err);
        alert("Unable to copy URL. Please copy it manually: " + url);
      }

      document.body.removeChild(textarea);
    }

    window.setTimeout(() => setCopied(false), 2000);
  };

  return (
    <header className={styles.header}>
      <div className={styles.logo}>
        <h1 className={styles.title}>PHASER</h1>
        <span className={styles.subtitle}>PHAS Evaluation Resource</span>
      </div>

      <nav className={styles.analysisTabs} aria-label="Analysis level">
        <button
          type="button"
          className={`${styles.tabButton} ${
            activePage === "genome" ? styles.activeTab : ""
          }`}
          aria-current={activePage === "genome" ? "page" : undefined}
          onClick={() => onPageChange("genome")}
        >
          Genome-Level Analysis
        </button>
        <button
          type="button"
          className={`${styles.tabButton} ${
            activePage === "transcriptome" ? styles.activeTab : ""
          }`}
          aria-current={activePage === "transcriptome" ? "page" : undefined}
          onClick={() => onPageChange("transcriptome")}
        >
          Transcriptome-Level Analysis
        </button>
      </nav>

      <div className={styles.actions}>
        <button
          type="button"
          className={styles.shareButton}
          onClick={handleShareClick}
        >
          {copied ? "Copied!" : "Share URL"}
        </button>
      </div>
    </header>
  );
}
```

- Header.module.css
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


/* Add these rules to the END of your existing Header.module.css. */
.analysisTabs {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  flex: 1 1 auto;
  min-width: 0;
  flex-wrap: wrap;
  margin: 0 20px;
}

.tabButton {
  appearance: none;
  border: 1px solid #d4dee7;
  border-radius: 8px;
  background: #ffffff;
  color: #365064;
  cursor: pointer;
  font: inherit;
  font-size: 0.92rem;
  font-weight: 600;
  line-height: 1.25;
  padding: 10px 14px;
  transition: background-color 120ms ease, border-color 120ms ease, color 120ms ease;
  white-space: nowrap;
}

.tabButton:hover {
  border-color: #8db9c7;
  background: #f2f8fa;
}

.tabButton:focus-visible {
  outline: 3px solid rgba(31, 128, 133, 0.3);
  outline-offset: 2px;
}

.activeTab {
  border-color: #167e80;
  background: #167e80;
  color: #ffffff;
}

.activeTab:hover {
  border-color: #126d70;
  background: #126d70;
  color: #ffffff;
}

@media (max-width: 900px) {
  .analysisTabs {
    order: 3;
    flex-basis: 100%;
    margin: 12px 0 0;
  }

  .tabButton {
    white-space: normal;
  }
}
```
#### PHASTable
- PHASTable.tsx
```sh
import { useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";

import { AgGridReact } from "ag-grid-react";

import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";

import {
  ModuleRegistry,
  AllCommunityModule,
} from "ag-grid-community";

ModuleRegistry.registerModules([
  AllCommunityModule,
]);

interface PHASTableProps {
  records: any[];

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

  const columnDefs = useMemo(() => {
    if (!records.length) return [];

    return [
      ...Object.keys(records[0])
        .filter(
          (k) =>
            k !== "__evaluation"
        )
        .map((col) => ({
          headerName: col,
          field: col,
          sortable: true,
          filter: true,
        })),

      // CONFIDENCE LEVEL
      {
        headerName:
          "Confidence Level",

        valueGetter: (p: any) =>
          p.data.__evaluation
            ?.level || "-",

        sortable: true,

        // COMMUNITY FILTER
        filter: true,
      },

      // NOTES
      {
        headerName: "Notes",

        valueGetter: (p: any) =>
          p.data.__evaluation
            ?.notes || "-",
      },

      // REVIEWER COMMENTS
      {
        headerName:
          "Reviewer Comments",

        valueGetter: (p: any) =>
          p.data.__evaluation
            ?.comments || "-",
      },
    ];
  }, [records]);

  return (
    <div className={styles.container}>
      {loading ? (
        <div
          className={styles.loading}
        >
          Loading...
        </div>
      ) : (
        <div
          className="ag-theme-alpine"
          style={{ height: 500 }}
        >
          <AgGridReact

            // FIX FOR AG GRID v35+
            theme="legacy"

            rowData={records}

            columnDefs={columnDefs}

            rowSelection="single"

            // KEEP ROWS STABLE
            getRowId={(params) =>
              params.data.phas_id
            }

            onRowClicked={(e) => {
              const r = e.data;

              const sample =
                r.best_sample?.match(
                  /N\d+/
                )?.[0] || null;

              onSelectPhasAndStage(
                r,
                null,
                sample
              );
            }}

            getRowStyle={(p) =>
              p.data.phas_id ===
              selectedId
                ? {
                    backgroundColor:
                      "#d1e7ff",
                  }
                : undefined
            }
          />
        </div>
      )}
    </div>
  );
}
```
- PHASTable.module.css 
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
  overflow-x: auto;
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

.ag-theme-alpine .ag-body-horizontal-scroll {
  position: sticky;
  bottom: 0;
  z-index: 5;
  background: white;
}
```
#### SearchBox  
- SearchBox.tsx
```sh
import { useState } from "react";
import styles from "./SearchBox.module.css";

interface SearchBoxProps {
  value: string;
  onChange: (value: string) => void;
  options: string[]; // list of PHAS IDs
  placeholder?: string;
  onSelect: (phasId: string) => void; // new
}

export function SearchBox({ value, onChange, options, placeholder, onSelect }: SearchBoxProps) {
  const [showDropdown, setShowDropdown] = useState(false);

  const filteredOptions = options.filter((opt) =>
    opt.toLowerCase().includes(value.toLowerCase())
  );

  return (
    <div className={styles.container}>
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
        onChange={(e) => {
          onChange(e.target.value);
          setShowDropdown(true);
        }}
        placeholder={placeholder || "Search PHAS ID..."}
        onFocus={() => setShowDropdown(true)}
        onBlur={() => setTimeout(() => setShowDropdown(false), 100)} // slight delay to allow click
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
              onMouseDown={() => {
                onChange(opt);      // set search input
                onSelect(opt);      // trigger selection in App
                setShowDropdown(false);
              }}
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
- SearchBox.module.css

```sh
/* SearchBox.module.css */

/* Container now takes full width */
.container {
  position: relative;
  display: flex;
  align-items: center;
  margin: 16px 24px;
  width: 96.5%; /* changed from 100% to 80% */
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
  width: 100%;               /* full width */
  padding: 12px 44px 12px 42px; /* space for icon + clear button */
  font-size: 14px;
  border: 2px solid #e3e6ea;
  border-radius: 8px;
  outline: none;
  background: #ffffff;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
  box-sizing: border-box;    /* include padding in width */
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

/* Dropdown */
.dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  max-height: 200px;
  overflow-y: auto;
  background: white;
  border: 1px solid #ccc;
  z-index: 10;
  border-radius: 4px;
  width: 100%; /* full width */
  box-sizing: border-box;
}

.dropdownItem {
  padding: 6px 12px;
  cursor: pointer;
}

.dropdownItem:hover {
  background-color: #f0f0f0;
}
```

#### PhasingPatternPanel
- PhasingPatternPanel.tsx
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
   HELPERS
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
    COMPONENT
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

  //  RESET HIDDEN IMAGES whenever PHAS ID or stage changes
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
      ACTIONS
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
        {/*  UPDATED TITLE BLOCK */}
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
- PhasingPatternPanel.module.css
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
  min-width: 120px;
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
  min-width: 110px;
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

.filterBox select {
  width: 50px;
  min-width: 50px;
}
```
#### IGVViewer 
- IGVViewer.tsx
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
  // SAMPLE → MATCHING MRNA LABELS
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
      "Okayama_Larva_fed",
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
      "Okayama_Adult_Fed",
      "Okayama_adult_female_fed",
      "OkayamaAFed",
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
        // CREATE BROWSER
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
  // APPLY TRACKS
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
                t.format === "bed" ||
                t.format === "bigBed"
            );

          // =============================================
          // SIGNAL TRACKS
          // =============================================
          let signalTracks =
            allTracks.filter(
              (t) =>
                t.format !== "bed" &&
                t.format !== "bigBed"
            );

          signalTracks =
            signalTracks.filter(
              (t) => {

                // =======================================
                // RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "ALL" ||
                  t.rnaType ===
                    filters.rnaType;

                if (!rnaMatch)
                  return false;

                // =======================================
                // SHOW BEST SAMPLE MODE
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
                  // �� MRNA
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

          const finalTracks = [
            ...annotationTracks,
            ...signalTracks,
          ];

          // =============================================
          //  LOAD TRACKS
          // =============================================
          for (const track of [...finalTracks].reverse()) {

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
  // LOCUS UPDATE
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
- IGVViewer.module.css
```sh
/* Container wrapper for the IGV panel */
.container {
  margin-top: 24px;
  background: #ffffff;
  border-radius: 12px;
  border: 1px solid #e1e5ea;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);

  overflow: visible;
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

/*  NEW: stack title + label */
.header > div {
  display: flex;
  flex-direction: column;
}

/*  NEW: default label */
.defaultLabel {
  font-size: 12px;
  color: #7f8c8d;
  margin-top: 2px;
}

/* IGV rendering area */
.viewer {
  min-height: 300px;
  height: auto;
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

  font-family: monospace;
}

.controls {
  display: flex;
  gap: 10px;
  padding: 10px 16px;
  background: #f4f6f8;
  border-bottom: 1px solid #e1e5ea;
}

.controls button {
  padding: 4px 10px;
  border-radius: 6px;
  border: none;
  background: #007bff;
  color: white;
  cursor: pointer;
}

.controls select {
  padding: 4px 8px;
  border-radius: 6px;
  font-size: 12px;
}
```

#### ConfidencePanel
- ConfidencePanel.tsx
```sh
import { useEffect, useState } from "react";
import styles from "./ConfidencePanel.module.css";

import {
  computeTotal,
  classifyScore,
  getInterpretation,
  Scores,
} from "../../utils/confidenceUtils";

interface Props {
  locus: any;
  evaluation?: any;
  onSave: (locusId: string, data: any) => void;
}

const defaultScores: Scores = {
  distBest: 0,
  distCross: 0,
  regBest: 0,
  regCross: 0,
  igvBest: 0,
  igvCross: 0,
  pValue: 0,
};

export default function ConfidencePanel({
  locus,
  evaluation,
  onSave,
}: Props) {
  const [scores, setScores] =
    useState<Scores>(defaultScores);

  const [notes, setNotes] = useState("");
  const [comments, setComments] = useState("");

  /* =============================
     SYNC STATE
  ============================= */

  useEffect(() => {
    if (!locus?.phas_id) return;

    if (evaluation) {
      setScores(
        evaluation.scores ||
          defaultScores
      );

      setNotes(
        evaluation.notes || ""
      );

      setComments(
        evaluation.comments || ""
      );

      return;
    }

    setScores(defaultScores);
    setNotes("");
    setComments("");
  }, [locus, evaluation]);

  /* =============================
     UPDATE SCORE
  ============================= */

  const update = (
    key: keyof Scores,
    value: number
  ) => {
    setScores((prev) => ({
      ...prev,
      [key]: value,
    }));
  };

  /* =============================
     COMPUTED VALUES
  ============================= */

  const total = computeTotal(scores);

  const level = classifyScore(total);

  const interpretation =
    getInterpretation(level);

  const percent = Math.round(
    (total / 14) * 100
  );

  /* =============================
     SAVE
  ============================= */

  const handleSave = async () => {
    if (!locus?.phas_id) return;

    const payload = {
      locusId: locus.phas_id,
      scores,
      total,
      level,
      interpretation,
      notes,
      comments,
      timestamp:
        new Date().toISOString(),
    };

    try {
      const response =
        await fetch(
          `http://163.221.246.151:8001/api/evaluations/${locus.phas_id}`,
          {
            method: "POST",
            headers: {
              "Content-Type":
                "application/json",
            },
            body: JSON.stringify(
              payload
            ),
          }
        );

      if (!response.ok) {
        throw new Error(
          "Failed to save evaluation"
        );
      }

      onSave(
        locus.phas_id,
        payload
      );

      alert(
        "Analysis saved successfully"
      );
    } catch (error) {
      console.error(error);

      alert(
        "Failed to save analysis"
      );
    }
  };

  /* =============================
     SELECT COMPONENT
  ============================= */

  const Select = (
    value: number,
    key: keyof Scores
  ) => (
    <select
      value={value}
      onChange={(e) =>
        update(
          key,
          Number(e.target.value)
        )
      }
    >
      <option value={0}>0</option>
      <option value={1}>1</option>
      <option value={2}>2</option>
    </select>
  );

  /* =============================
     BADGE COLOR
  ============================= */

  const badgeClass =
    level === "High"
      ? styles.high
      : level === "Moderate"
      ? styles.moderate
      : styles.low;

  /* =============================
     UI
  ============================= */

  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>
          Confidence Evaluation
        </h3>
      </div>

      {!locus && (
        <div className={styles.empty}>
          Select a PHAS locus
        </div>
      )}

      {locus && (
        <div className={styles.content}>

          <div className={styles.guideSection}>
            <a
              href="/PHASER_Manual_Scoring_Guide.pdf"
              target="_blank"
              rel="noopener noreferrer"
              className={styles.scoringGuideLink}
            >
              📖 View Scoring Guide
            </a>
          </div>

          <div className={styles.totalBox}>
            <div>
              <div
                className={
                  styles.totalLabel
                }
              >
                Total Score
              </div>

              <div
                className={
                  styles.total
                }
              >
                {total} / 14
              </div>

              <div
                className={
                  styles.percent
                }
              >
                {percent}% confidence
              </div>
            </div>

            <span
              className={`${styles.badge} ${badgeClass}`}
            >
              {level}
            </span>
          </div>

          <div className={styles.section}>
            <h4>
              Phasing Evidence
            </h4>

            <div className={styles.row}>
              <span>
                Best Sample
              </span>

              {Select(
                scores.distBest,
                "distBest"
              )}
            </div>

            <div className={styles.row}>
              <span>
                Cross-sample
              </span>

              {Select(
                scores.distCross,
                "distCross"
              )}
            </div>
          </div>

          <div className={styles.section}>
            <h4>
              Register Consistency
            </h4>

            <div className={styles.row}>
              <span>
                Best Sample
              </span>

              {Select(
                scores.regBest,
                "regBest"
              )}
            </div>

            <div className={styles.row}>
              <span>
                Cross-sample
              </span>

              {Select(
                scores.regCross,
                "regCross"
              )}
            </div>
          </div>

          <div className={styles.section}>
            <h4>IGV Evidence</h4>

            <div className={styles.row}>
              <span>
                Best Sample
              </span>

              {Select(
                scores.igvBest,
                "igvBest"
              )}
            </div>

            <div className={styles.row}>
              <span>
                Cross-sample
              </span>

              {Select(
                scores.igvCross,
                "igvCross"
              )}
            </div>
          </div>

          <div className={styles.section}>
            <h4>
              Statistical Support
            </h4>

            <div className={styles.row}>
              <span>P-value</span>

              {Select(
                scores.pValue,
                "pValue"
              )}
            </div>
          </div>

          <div
            className={
              styles.interpretation
            }
          >
            <strong>
              Interpretation
            </strong>

            {interpretation}
          </div>

          <div
            className={
              styles.notesSection
            }
          >
            <div
              className={
                styles.notesLabel
              }
            >
              Notes
            </div>

            <textarea
              className={
                styles.notesInput
              }
              value={notes}
              onChange={(e) =>
                setNotes(
                  e.target.value
                )
              }
              placeholder="Write your notes..."
            />

            <div
              className={
                styles.notesLabel
              }
            >
              Reviewer Comments
            </div>

            <textarea
              className={
                styles.notesInput
              }
              value={comments}
              onChange={(e) =>
                setComments(
                  e.target.value
                )
              }
              placeholder="Write reviewer comments..."
            />

            <button
              className={
                styles.saveButton
              }
              onClick={handleSave}
            >
              Save Analysis
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
```
- ConfidencePanel.module.css
```sh
.panel {
  display: flex;
  flex-direction: column;

  height: 100%;

  background: #eef3f4;

  border-left: 1px solid #d7e0e3;
}

/* =========================
   HEADER
========================= */

.header {
  background: linear-gradient(
    135deg,
    #173d42 0%,
    #103037 100%
  );

  color: white;

  padding: 14px 18px;

  position: relative;

  border-bottom:
    1px solid rgba(255,255,255,0.05);

  box-shadow:
    0 1px 2px rgba(0,0,0,0.08),
    inset 0 -1px 0 rgba(255,255,255,0.03);
}

.header::after {
  content: "";

  position: absolute;

  left: 0;
  bottom: 0;

  width: 100%;
  height: 2px;

  background: linear-gradient(
    90deg,
    rgba(0,0,0,0),
    rgba(74,214,195,0.55),
    rgba(0,0,0,0)
  );
}

.header h3 {
  margin: 0;

  font-size: 15px;
  font-weight: 650;

  letter-spacing: 0.3px;
}

/* =========================
   EMPTY
========================= */

.empty {
  padding: 20px;

  color: #66757c;

  font-size: 13px;
}

/* =========================
   CONTENT
========================= */

.content {
  padding: 14px;

  overflow-y: auto;

  display: flex;
  flex-direction: column;

  gap: 12px;
}

/* =========================
   SCORING GUIDE
========================= */

.guideSection {
  display: flex;
  justify-content: flex-start;
  align-items: center;

  margin-bottom: 2px;
}

.scoringGuideLink {
  display: inline-flex;
  align-items: center;
  gap: 6px;

  padding: 6px 10px;

  background: rgba(47, 127, 127, 0.08);
  border: 1px solid rgba(47, 127, 127, 0.18);
  border-radius: 8px;

  color: #2f7f7f;

  font-size: 13px;
  font-weight: 600;

  text-decoration: none;

  transition:
    background 0.2s ease,
    border-color 0.2s ease,
    color 0.2s ease,
    transform 0.15s ease;
}

.scoringGuideLink:hover {
  background: rgba(47, 127, 127, 0.15);
  border-color: rgba(47, 127, 127, 0.35);

  color: #1f6060;

  transform: translateY(-1px);
}

.scoringGuideLink:active {
  transform: scale(0.98);
}

/* =========================
   TOTAL SCORE
========================= */

.totalBox {
  background: white;

  border-radius: 12px;

  padding: 14px 16px;

  border: 1px solid #dde5e7;

  box-shadow:
    0 1px 2px rgba(0,0,0,0.03);

  display: flex;
  justify-content: space-between;
  align-items: center;
}

.totalLabel {
  font-size: 11px;

  font-weight: 700;

  color: #6b7b83;

  text-transform: uppercase;

  letter-spacing: 0.6px;
}

.total {
  margin-top: 2px;

  font-size: 30px;

  font-weight: 700;

  color: #173d42;

  line-height: 1;
}

.percent {
  margin-top: 4px;

  font-size: 12px;

  color: #7b8b92;
}

/* =========================
   BADGES
========================= */

.badge {
  padding: 7px 13px;

  border-radius: 999px;

  font-size: 11px;
  font-weight: 700;

  letter-spacing: 0.4px;

  text-transform: uppercase;

  border:
    1px solid rgba(0,0,0,0.05);
}

.high {
  background:
    rgba(56,142,60,0.12);

  color: #2e7d32;
}

.moderate {
  background:
    rgba(249,168,37,0.16);

  color: #8a5a00;
}

.low {
  background:
    rgba(198,40,40,0.12);

  color: #b71c1c;
}

/* =========================
   SECTION
========================= */

.section {
  background:
    rgba(255,255,255,0.95);

  border-radius: 12px;

  padding: 12px 14px;

  border: 1px solid #dde5e7;

  box-shadow:
    0 1px 2px rgba(0,0,0,0.03);

  transition:
    transform 0.15s ease,
    box-shadow 0.15s ease;
}

.section:hover {
  transform: translateY(-1px);

  box-shadow:
    0 4px 10px rgba(0,0,0,0.05);
}

.section h4 {
  margin: 0 0 10px 0;

  font-size: 13px;

  font-weight: 650;

  color: #26464d;

  display: flex;
  align-items: center;

  gap: 6px;
}

.section h4::after {
  content: "";

  flex: 1;

  height: 1px;

  background: linear-gradient(
    90deg,
    rgba(39,88,95,0.18),
    rgba(39,88,95,0)
  );
}

/* =========================
   ROWS
========================= */

.row {
  display: grid;

  grid-template-columns:
    1fr auto;

  align-items: center;

  gap: 10px;

  padding: 7px 0;

  font-size: 13px;

  color: #41545b;

  border-bottom:
    1px solid rgba(0,0,0,0.04);
}

.row:last-child {
  border-bottom: none;
}

/* =========================
   SELECT
========================= */

select {
  padding: 5px 8px;

  font-size: 12px;

  border-radius: 6px;

  border:
    1px solid #ccd6d9;

  background: white;

  color: #26464d;

  outline: none;

  transition: 0.15s ease;
}

select:hover {
  border-color: #8bb9b7;
}

select:focus {
  border-color: #2f7f7f;

  box-shadow:
    0 0 0 3px rgba(47,127,127,0.12);
}

/* =========================
   INTERPRETATION
========================= */

.interpretation {
  background: white;

  border-radius: 12px;

  padding: 12px 14px;

  border: 1px solid #dde5e7;

  border-left:
    4px solid #43a047;

  box-shadow:
    0 1px 2px rgba(0,0,0,0.03);

  font-size: 13px;

  line-height: 1.5;

  color: #44575e;
}

.interpretation strong {
  display: block;

  margin-bottom: 6px;

  color: #173d42;
}

/* =========================
   NOTES
========================= */

.notesSection {
  display: flex;
  flex-direction: column;

  gap: 8px;
}

.notesLabel {
  font-size: 13px;

  font-weight: 650;

  color: #26464d;

  display: flex;
  align-items: center;

  gap: 6px;
}

.notesLabel::after {
  content: "";

  flex: 1;

  height: 1px;

  background: linear-gradient(
    90deg,
    rgba(38,70,77,0.15),
    rgba(38,70,77,0)
  );
}

.notesInput {
  min-height: 95px;

  resize: vertical;

  padding: 12px;

  font-size: 13px;

  line-height: 1.5;

  border-radius: 10px;

  border:
    1px solid #d5dee1;

  background:
    rgba(255,255,255,0.96);

  color: #33444a;

  outline: none;

  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease,
    background 0.15s ease;
}

.notesInput::placeholder {
  color: #94a4aa;

  font-style: italic;
}

.notesInput:focus {
  background: white;

  border-color: #2f7f7f;

  box-shadow:
    0 0 0 3px rgba(47,127,127,0.12);
}

/* =========================
   SAVE BUTTON
========================= */

.saveButton {
  align-self: flex-end;

  padding: 10px 16px;

  border: none;

  border-radius: 10px;

  background: linear-gradient(
    135deg,
    #2f7f7f,
    #276868
  );

  color: white;

  font-size: 13px;
  font-weight: 650;

  cursor: pointer;

  letter-spacing: 0.2px;

  transition:
    transform 0.15s ease,
    box-shadow 0.15s ease,
    filter 0.15s ease;

  box-shadow:
    0 2px 6px rgba(47,127,127,0.18);
}

.saveButton:hover {
  transform: translateY(-1px);

  filter: brightness(1.03);

  box-shadow:
    0 4px 10px rgba(47,127,127,0.24);
}

.saveButton:active {
  transform: scale(0.98);
}
```
#### SizeDistributionPanel

- SizeDistributionPanel.tsx
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
- SizeDistributionPanel.module.css 
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
#### sRNAAbundancePanel
- sRNAAbundancePanel.tsx
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
- sRNAAbundancePanel.module.css 
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

