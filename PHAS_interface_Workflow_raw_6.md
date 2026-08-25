# Phase 4. Upgrade and sync IGV panel with phasing panel

##### Upgrade and connect search bar to Phas table, Phasing pattern panel, and IGV panel
##### PHAS_interface_Workflow_raw_6
> This is a raw file for upgrading Phasing + IGV panel
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

---
# Previous Codes:
## Frontend
### 1. App.tsx
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
### 2. SearchBox.tsx
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
### 3. PHASTable.tsx
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
###  4. PhasingPatternPanel.tsx
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

### 5. IGVViewer.tsx
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

  font-family: monospace;   /* looks better for coordinates */
}
```
#### PhasingPatternPanel.module.css
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
## Backend:
### 1. main.py
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

### 2. app/config.py
```sh
"""PHASER Configuration"""

from pathlib import Path

# 🔷 BASE DIRECTORY
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# 🔷 DATA
DATA_DIR = BASE_DIR / "data"
PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# 🔥 REQUIRED
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# 🔷 TRACK HUB (REAL DATA LOCATION)
TRACKHUB_BASE = Path("/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks")

# =========================================================
# 🔥 GENOME (USE FASTA FOR IGV)
# =========================================================
GENOME_FASTA = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.IGV.genome.fasta"
GENOME_FAI = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.IGV.genome.fasta.fai"

# (Optional, keep if useful elsewhere)
GENOME_2BIT = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.2bit"
GENOME_CHROMSIZES = TRACKHUB_BASE / "genome" / "GWHAMMI00000000.genome.chrom.sizes"

# =========================================================
# 🔷 ANNOTATION
# =========================================================
PHAS_ANNOTATION_BB = TRACKHUB_BASE / "annotation" / "merged.PHAS22.bb"

# =========================================================
# 🔷 BIGWIG DIRECTORY
# =========================================================
BW_DIR = TRACKHUB_BASE / "bw"

# =========================================================
# 🔷 API
# =========================================================
API_PREFIX = "/api"
```
### 3. app/routers/tracks.py
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


@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track


# 🔥🔥🔥 THIS IS THE MISSING ENDPOINT
@router.get("/igv/{sample}")
async def get_igv_config(sample: str):
    """
    Return IGV configuration for a given sample
    """
    config = track_service.get_igv_config(sample)

    if not config:
        raise HTTPException(status_code=404, detail=f"Sample {sample} not found")

    return config
```
### 4. app/routers/files.py
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
### 5. app/services/track_service.py
```sh
"""Track Service (IGV Integration - FINAL WORKING VERSION)"""

import pandas as pd
from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR
)
from ..models.phas import TrackFile


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # -----------------------------
    # 🔷 LOAD LIBRARY INFO
    # -----------------------------
    def _load_library_info(self) -> pd.DataFrame:
        if self._library_info is None:
            self._library_info = pd.read_csv(LIBRARY_INFO_FILE, sep="\t")
        return self._library_info

    # -----------------------------
    # 🔷 BASIC TRACK LIST
    # -----------------------------
    def get_all_tracks(self) -> List[TrackFile]:
        df = self._load_library_info()
        tracks = []

        for _, row in df.iterrows():
            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get("Sample", row["library_ID"]),
                    category=row.get("Category", "unknown"),
                    default_setting=row.get("default_setting", "off"),
                    files={}
                )
            )

        return tracks

    def get_track_by_id(self, library_id: str) -> Optional[TrackFile]:
        df = self._load_library_info()
        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get("Sample", row["library_ID"]),
            category=row.get("Category", "unknown"),
            default_setting=row.get("default_setting", "off"),
            files={}
        )

    # -----------------------------
    # 🔷 GENOME FILES
    # -----------------------------
    def get_genome_files(self) -> Dict[str, str]:
        return {
            "fasta": f"/api/files/genome/{GENOME_FASTA.name}",
            "fai": f"/api/files/genome/{GENOME_FAI.name}",
            "annotation": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
        }

    # -----------------------------
    # 🔥 HELPER: FIND FILES (ROBUST VERSION)
    # -----------------------------
    def _find_bw_files(self, sample: str):
        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for sample: {sample}")

        # 🔍 Scan all BW files
        for f in BW_DIR.glob("*.bw"):
            name = f.name

            if sample in name:
                print(f"[IGV DEBUG] Candidate match: {name}")

                if "plus" in name:
                    plus_file = f
                    print(f"[FOUND PLUS] {f}")

                elif "minus" in name:
                    minus_file = f
                    print(f"[FOUND MINUS] {f}")

        # 🚨 Final checks
        if not plus_file:
            print(f"[IGV WARNING] PLUS strand file not found for {sample}")
        if not minus_file:
            print(f"[IGV WARNING] MINUS strand file not found for {sample}")

        return plus_file, minus_file

    # -----------------------------
    # 🔥 IGV CONFIG
    # -----------------------------
    def get_igv_config(self, sample: str) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:
            print(f"[IGV ERROR] Missing files for sample: {sample}")
            return None

        print(f"[IGV SUCCESS] Using files:")
        print(f"  PLUS:  {plus_file}")
        print(f"  MINUS: {minus_file}")

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = [
            {
                "name": "PHAS loci",
                "type": "annotation",
                "format": "bigBed",
                "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
                "color": "green"
            },
            {
                "name": f"{sample} (+)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{plus_file.name}",
                "color": "red"
            },
            {
                "name": f"{sample} (-)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{minus_file.name}",
                "color": "blue"
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # -----------------------------
    # 🔷 FILE RESOLUTION
    # -----------------------------
    def resolve_file_path(self, relative_path: str) -> Optional[Path]:
        resolved = TRACKHUB_BASE / relative_path
        print(f"[FILE RESOLVE] {relative_path} → {resolved}")
        return resolved


# 🔷 Singleton
track_service = TrackService()
```

---
# Phase 4: Upgrade, Sync (Phasing + IGV) version 1
- Search dropdown to be fully wired up to IGV
- Make the "Best Sample" and the "Best Region" the default View
- Add default view Labels: Best sample and Best region

## App.tsx 
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

/* =========================
   🔥 Helper: parse best_region
========================= */
function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);
  return { chrom, start, end };
}

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

  // =============================
  // 🔥 CENTRAL SELECTION HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    // 🔥 ALWAYS derive BEST SAMPLE + STAGE
    if (phas.best_sample) {
      const match = phas.best_sample.match(/N\d+/);
      if (match) {
        derivedSample = match[0];
      }

      const bestStage = getStageFromSample(phas.best_sample);
      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(derivedSample || null);
    setActiveStage(derivedStage || "ALL");
    setView("all");
  };

  // =============================
  // 🔷 INITIAL LOAD
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        // 🔥 DEFAULT = PHAS22-1 → BEST SAMPLE + REGION
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
  // 🔷 SEARCH (FILTER ONLY)
  // =============================
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

    // 🔥 Auto-select exact match
    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );

    if (exact && exact.phas_id !== selectedPhas?.phas_id) {
      handleSelectPhasAndStage(exact);
    }
  }, [searchQuery, phasRecords]);

  // =============================
  // 🔷 LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);
      });
  }, [selectedPhas]);

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
          onSelect={(phasId) => {
            const phas = phasRecords.find((r) => r.phas_id === phasId);
            if (phas) {
              setSearchQuery(phasId); // optional but good UX
              handleSelectPhasAndStage(phas);
            }
          }}
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

            {/* 🔥 IGV PANEL (BEST REGION DEFAULT) */}
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
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```
### PhasingPatternPanel.tsx
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
- #### PhasingPatternPanel.module.css
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
### IGVViewer.tsx
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
      {/* 🔥 UPDATED HEADER */}
      <div className={styles.header}>
        <div>
          Genome Browser (IGV) — {sample}
          {locus && (
            <div className={styles.defaultLabel}>
              Viewing: Best Region
            </div>
          )}
        </div>
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
- #### IGVViewer.module.css
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

/* 🔥 NEW: stack title + label */
.header > div {
  display: flex;
  flex-direction: column;
}

/* 🔥 NEW: default label */
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
```
---
# Phase 4: Upgrade, Sync (Phasing + IGV) version 2

- include toggle sections
- include all samples

## Backend:
### app/services/track_services.py ver. 1 
```sh
"""Track Service (IGV Integration - FINAL WORKING VERSION)"""

import pandas as pd
from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR
)

# 🔥 NEW IMPORT
from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # -----------------------------
    # 🔷 LOAD LIBRARY INFO
    # -----------------------------
    def _load_library_info(self) -> pd.DataFrame:
        if self._library_info is None:
            self._library_info = pd.read_csv(LIBRARY_INFO_FILE, sep="\t")
        return self._library_info

    # -----------------------------
    # 🔷 BASIC TRACK LIST
    # -----------------------------
    def get_all_tracks(self) -> List[TrackFile]:
        df = self._load_library_info()
        tracks = []

        for _, row in df.iterrows():
            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get("Sample", row["library_ID"]),
                    category=row.get("Category", "unknown"),
                    default_setting=row.get("default_setting", "off"),
                    files={}
                )
            )

        return tracks

    def get_track_by_id(self, library_id: str) -> Optional[TrackFile]:
        df = self._load_library_info()
        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get("Sample", row["library_ID"]),
            category=row.get("Category", "unknown"),
            default_setting=row.get("default_setting", "off"),
            files={}
        )

    # -----------------------------
    # �� GENOME FILES
    # -----------------------------
    def get_genome_files(self) -> Dict[str, str]:
        return {
            "fasta": f"/api/files/genome/{GENOME_FASTA.name}",
            "fai": f"/api/files/genome/{GENOME_FAI.name}",
            "annotation": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
        }

    # -----------------------------
    # 🔥 HELPER: FIND FILES (ROBUST VERSION)
    # -----------------------------
    def _find_bw_files(self, sample: str):
        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for sample: {sample}")

        for f in BW_DIR.glob("*.bw"):
            name = f.name

            if sample in name:
                print(f"[IGV DEBUG] Candidate match: {name}")

                if "plus" in name:
                    plus_file = f
                    print(f"[FOUND PLUS] {f}")

                elif "minus" in name:
                    minus_file = f
                    print(f"[FOUND MINUS] {f}")

        if not plus_file:
            print(f"[IGV WARNING] PLUS strand file not found for {sample}")
        if not minus_file:
            print(f"[IGV WARNING] MINUS strand file not found for {sample}")

        return plus_file, minus_file

    # -----------------------------
    # 🔥 IGV CONFIG (SINGLE SAMPLE)
    # -----------------------------
    def get_igv_config(self, sample: str) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:
            print(f"[IGV ERROR] Missing files for sample: {sample}")
            return None

        print(f"[IGV SUCCESS] Using files:")
        print(f"  PLUS:  {plus_file}")
        print(f"  MINUS: {minus_file}")

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = [
            {
                "name": "PHAS loci",
                "type": "annotation",
                "format": "bigBed",
                "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
                "color": "green"
            },
            {
                "name": f"{sample} (+)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{plus_file.name}",
                "color": "red"
            },
            {
                "name": f"{sample} (-)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{minus_file.name}",
                "color": "blue"
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # -----------------------------
    # 🔥 NEW: GET ALL IGV TRACKS (Phase 3B)
    # -----------------------------
    def get_all_igv_tracks(self):

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = []

        # 🔷 ALWAYS include annotation track first
        tracks.append({
            "name": "PHAS loci",
            "type": "annotation",
            "format": "bigBed",
            "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
            "color": "green"
        })

        print("[IGV ALL] Scanning BW directory...")

        for f in BW_DIR.glob("*.bw"):
            name = f.name

            # 🔷 Extract sample (NXXX)
            sample = name.split(".")[0]

            # 🔷 Detect strand
            strand = "+" if "plus" in name else "-"

            # 🔷 Get biological metadata
            meta = parse_sample_metadata(sample)

            print(f"[IGV ALL] {sample} | {meta} | {strand}")

            tracks.append({
                "name": f"{sample} ({strand})",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{name}",

                # 🔥 NEW METADATA (for frontend filtering)
                "sample": sample,
                "strain": meta["strain"],
                "stage": meta["stage"],
                "feeding": meta["feeding"],
                "strand": strand,
                "label": meta["label"]
            })

        return {
            "genome": genome,
            "tracks": tracks
        }

    # -----------------------------
    # 🔷 FILE RESOLUTION
    # -----------------------------
    def resolve_file_path(self, relative_path: str) -> Optional[Path]:
        resolved = TRACKHUB_BASE / relative_path
        print(f"[FILE RESOLVE] {relative_path} → {resolved}")
        return resolved


# 🔷 Singleton
track_service = TrackService()
```
### app/services/track_services.py ver. 2

```sh
"""Track Service (IGV Integration - FINAL WORKING VERSION)"""

import pandas as pd
from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR
)

from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


# =========================================================
# 🔥 SAMPLE LABEL MAPPING (NEW)
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


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    def _load_library_info(self) -> pd.DataFrame:
        if self._library_info is None:
            self._library_info = pd.read_csv(LIBRARY_INFO_FILE, sep="\t")
        return self._library_info

    def get_all_tracks(self) -> List[TrackFile]:
        df = self._load_library_info()
        tracks = []

        for _, row in df.iterrows():
            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get("Sample", row["library_ID"]),
                    category=row.get("Category", "unknown"),
                    default_setting=row.get("default_setting", "off"),
                    files={}
                )
            )

        return tracks

    def get_track_by_id(self, library_id: str) -> Optional[TrackFile]:
        df = self._load_library_info()
        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get("Sample", row["library_ID"]),
            category=row.get("Category", "unknown"),
            default_setting=row.get("default_setting", "off"),
            files={}
        )

    def get_genome_files(self) -> Dict[str, str]:
        return {
            "fasta": f"/api/files/genome/{GENOME_FASTA.name}",
            "fai": f"/api/files/genome/{GENOME_FAI.name}",
            "annotation": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
        }

    def _find_bw_files(self, sample: str):
        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for sample: {sample}")

        for f in BW_DIR.glob("*.bw"):
            name = f.name

            if sample in name:
                print(f"[IGV DEBUG] Candidate match: {name}")

                if "plus" in name:
                    plus_file = f
                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file

    def get_igv_config(self, sample: str) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:
            print(f"[IGV ERROR] Missing files for sample: {sample}")
            return None

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = [
            {
                "name": "PHAS loci",
                "type": "annotation",
                "format": "bigBed",
                "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
                "color": "green"
            },
            {
                "name": f"{sample} (+)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{plus_file.name}",
                "color": "red"
            },
            {
                "name": f"{sample} (-)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{minus_file.name}",
                "color": "blue"
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    def get_all_igv_tracks(self):

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = []

        # 🔷 annotation first
        tracks.append({
            "name": "PHAS loci",
            "type": "annotation",
            "format": "bigBed",
            "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
            "color": "green"
        })

        print("[IGV ALL] Scanning BW directory...")

        for f in BW_DIR.glob("*.bw"):
            name = f.name

            sample = name.split(".")[0]
            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            # =====================================================
            # 🔥 HUMAN-READABLE LABEL (NEW)
            # =====================================================
            label = SAMPLE_LABELS.get(sample, sample)

            print(f"[IGV ALL] {sample} -> {label} | {meta} | {strand}")

            tracks.append({
                "name": f"{label} ({strand})",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{name}",

                # IMPORTANT: keep original sample for filtering
                "sample": sample,
                "strain": meta["strain"],
                "stage": meta["stage"],
                "feeding": meta["feeding"],
                "strand": strand,
                "label": label
            })

        return {
            "genome": genome,
            "tracks": tracks
        }

    def resolve_file_path(self, relative_path: str) -> Optional[Path]:
        resolved = TRACKHUB_BASE / relative_path
        print(f"[FILE RESOLVE] {relative_path} → {resolved}")
        return resolved


track_service = TrackService()
```
### track_service.py ver 3.

```sh
"""Track Service (IGV FINAL STABLE GROUPED VERSION)"""

import pandas as pd
from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR
)

from ..utils.sample_metadata import parse_sample_metadata
from ..models.phas import TrackFile


# =========================================================
# SAMPLE LABELS
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


class TrackService:

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    def _load_library_info(self) -> pd.DataFrame:
        if self._library_info is None:
            self._library_info = pd.read_csv(LIBRARY_INFO_FILE, sep="\t")
        return self._library_info

    # =========================================================
    # GENOME FILES
    # =========================================================
    def get_genome_files(self) -> Dict[str, str]:
        return {
            "fasta": f"/api/files/genome/{GENOME_FASTA.name}",
            "fai": f"/api/files/genome/{GENOME_FAI.name}",
            "annotation": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
        }

    def resolve_file_path(self, relative_path: str) -> Optional[Path]:
        return TRACKHUB_BASE / relative_path

    # =========================================================
    # STRAIN PRIORITY
    # =========================================================
    def get_strain_priority(self, meta_strain: str, sample: str):
        text = (meta_strain or "").lower() + sample.lower()

        if "okayama" in text:
            return 0, "Okayama"
        if "oita" in text:
            return 1, "Oita"

        return 99, "Unknown"

    # =========================================================
    # STAGE PARSING (E / L / N / A)
    # =========================================================
    def parse_stage(self, sample: str):
        s = sample.upper()

        if "E" in s:
            num = "".join([c for c in s if c.isdigit()])
            return "E", int(num) if num.isdigit() else 0

        if "LARVA" in s or "_L" in s:
            return "L", 0

        if "NYMPH" in s or "_N" in s:
            return "N", 0

        if "ADULT" in s or "_A" in s:
            return "A", 0

        return "Z", 0

    # =========================================================
    # FINAL IGV TRACK BUILDER (FIXED ORDER LOGIC)
    # =========================================================
    def get_all_igv_tracks(self):

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        temp_tracks = []

        # -----------------------------
        # COLLECT ALL TRACKS FIRST
        # -----------------------------
        for f in BW_DIR.glob("*.bw"):
            name = f.name
            sample = name.split(".")[0]
            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)
            label = SAMPLE_LABELS.get(sample, sample)

            strain_priority, strain_name = self.get_strain_priority(meta.get("strain"), sample)
            stage_code, stage_num = self.parse_stage(sample)

            temp_tracks.append({
                "name": f"{label} ({strand})",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{name}",

                "sample": sample,
                "strain": strain_name,
                "strand": strand,
                "label": label,

                "strain_priority": strain_priority,
                "stage_code": stage_code,
                "stage_num": stage_num
            })

        # -----------------------------
        # SORT RULES
        # -----------------------------
        stage_order = {
            "E": 0,
            "L": 1,
            "N": 2,
            "A": 3
        }

        def sort_key(t):
            return (
                t["strain_priority"],        # Okayama → Oita
                stage_order.get(t["stage_code"], 99),
                t["stage_num"],
                t["sample"]
            )

        temp_tracks.sort(key=sort_key)

        # -----------------------------
        # FINAL OUTPUT (IMPORTANT FIX)
        # -----------------------------
        tracks = []

        # PHAS annotation ALWAYS FIRST
        tracks.append({
            "name": "PHAS loci",
            "type": "annotation",
            "format": "bigBed",
            "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
            "color": "green"
        })

        # THEN ALL SORTED SIGNAL TRACKS
        tracks.extend(temp_tracks)

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =========================================================
    # SINGLE SAMPLE IGV CONFIG (UNCHANGED)
    # =========================================================
    def get_igv_config(self, sample: str) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:
            return None

        genome = {
            "fastaURL": f"/api/files/genome/{GENOME_FASTA.name}",
            "indexURL": f"/api/files/genome/{GENOME_FAI.name}"
        }

        tracks = [
            {
                "name": "PHAS loci",
                "type": "annotation",
                "format": "bigBed",
                "url": f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}",
                "color": "green"
            },
            {
                "name": f"{sample} (+)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{plus_file.name}",
                "color": "red"
            },
            {
                "name": f"{sample} (-)",
                "type": "wig",
                "format": "bigWig",
                "url": f"/api/files/bw/{minus_file.name}",
                "color": "blue"
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =========================================================
    # FILE FINDER
    # =========================================================
    def _find_bw_files(self, sample: str):
        plus_file = None
        minus_file = None

        for f in BW_DIR.glob("*.bw"):
            name = f.name
            if sample in name:
                if "plus" in name:
                    plus_file = f
                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file


track_service = TrackService()
```
### app/routers/tracks.py
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


# 🔥 IGV ROUTES FIRST (VERY IMPORTANT)
@router.get("/igv/all")
async def get_all_igv_tracks():
    return track_service.get_all_igv_tracks()


@router.get("/igv/{sample}")
async def get_igv_config(sample: str):
    config = track_service.get_igv_config(sample)

    if not config:
        raise HTTPException(status_code=404, detail=f"Sample {sample} not found")

    return config


# 🔥 GENERIC ROUTE LAST (ALWAYS LAST)
@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track
```
## Frontend
### IGVViewer.tsx
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

  const [allTracks, setAllTracks] = useState<any[]>([]);
  const [loading, setLoading] = useState(false);

  const [filters, setFilters] = useState({
    samples: [sample],
    stage: "ALL",
    strain: "ALL",
    feeding: "ALL",
  });

  // 🔥 sync best sample
  useEffect(() => {
    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));
  }, [sample]);

  // -----------------------------
  // 🔥 LOAD IGV
  // -----------------------------
  useEffect(() => {
    if (!igvContainerRef.current) return;

    if (igvBrowserRef.current) {
      igvBrowserRef.current.destroy?.();
      igvBrowserRef.current = null;
      isReadyRef.current = false;
    }

    const loadIGV = async () => {
      setLoading(true);

      try {
        const res = await fetch(`${API_BASE}/api/tracks/igv/all`);
        const config = await res.json();

        // -----------------------------
        // ✅ CLEAN VERSION: NO COLOR LOGIC
        // -----------------------------
        const tracksWithURL = config.tracks.map((t: any) => ({
          ...t,
          url: API_BASE + t.url,
        }));

        setAllTracks(tracksWithURL);

        const browser = await igv.createBrowser(
          igvContainerRef.current!,
          {
            genome: {
              fastaURL: API_BASE + config.genome.fastaURL,
              indexURL: API_BASE + config.genome.indexURL,
            },
            tracks: [],
          }
        );

        igvBrowserRef.current = browser;

        setTimeout(() => {
          isReadyRef.current = true;

          setFilters((prev) => ({ ...prev }));

          if (locus) {
            browser.search(`${locus.chrom}:${locus.start}-${locus.end}`);
          }
        }, 300);
      } catch (err) {
        console.error("IGV load error:", err);
      } finally {
        setLoading(false);
      }
    };

    loadIGV();
  }, [sample]);

  // -----------------------------
  // 🔥 APPLY FILTERS + ORDER
  // -----------------------------
  useEffect(() => {
    if (!igvBrowserRef.current || !isReadyRef.current || allTracks.length === 0)
      return;

    const browser = igvBrowserRef.current;

    const applyTracks = async () => {
      await browser.removeAllTracks();

      const annotation = allTracks.find((t) => t.type === "annotation");

      let signalTracks = allTracks.filter((t) => t.type !== "annotation");

      signalTracks = signalTracks.filter((t) => {
        return (
          filters.samples.includes(t.sample) &&
          (filters.stage === "ALL" || t.stage === filters.stage) &&
          (filters.strain === "ALL" || t.strain === filters.strain) &&
          (filters.feeding === "ALL" || t.feeding === filters.feeding)
        );
      });

      const uniqueSamples = Array.from(
        new Set(signalTracks.map((t) => t.sample))
      ).sort();

      const firstSample = uniqueSamples[0];

      signalTracks.sort((a, b) => {
        const priority = (s: string) => {
          if (s === sample) return 0;
          if (s === firstSample) return 1;
          return 2;
        };

        const pa = priority(a.sample);
        const pb = priority(b.sample);

        if (pa !== pb) return pa - pb;

        return a.name.localeCompare(b.name, undefined, {
          numeric: true,
          sensitivity: "base",
        });
      });

      const finalTracks = annotation
        ? [annotation, ...signalTracks]
        : signalTracks;

      for (const t of finalTracks) {
        await browser.loadTrack(t);
      }
    };

    applyTracks();
  }, [filters, allTracks]);

  // -----------------------------
  // 🔥 LOCUS update
  // -----------------------------
  useEffect(() => {
    if (!igvBrowserRef.current || !locus || !isReadyRef.current) return;

    igvBrowserRef.current.search(
      `${locus.chrom}:${locus.start}-${locus.end}`
    );
  }, [locus]);

  // -----------------------------
  // 🔥 toggle samples
  // -----------------------------
  const toggleAllSamples = () => {
    const allSamples = Array.from(
      new Set(allTracks.map((t) => t.sample).filter(Boolean))
    );

    if (filters.samples.length === 1) {
      setFilters((f) => ({ ...f, samples: allSamples }));
    } else {
      setFilters((f) => ({ ...f, samples: [sample] }));
    }
  };

  return (
    <div className={styles.container}>
      <div className={styles.header}>
        Genome Browser (IGV) — {sample}
        {locus && (
          <div className={styles.defaultLabel}>
            Viewing: Best Region
          </div>
        )}
      </div>

      <div className={styles.controls}>
        <button onClick={toggleAllSamples}>
          {filters.samples.length === 1
            ? "Show All Samples"
            : "Show Best Sample"}
        </button>

        <select
          value={filters.stage}
          onChange={(e) =>
            setFilters((f) => ({ ...f, stage: e.target.value }))
          }
        >
          <option value="ALL">All Stages</option>
          <option value="Embryo">Embryo</option>
          <option value="Larva">Larva</option>
          <option value="Nymph">Nymph</option>
          <option value="Adult">Adult</option>
        </select>

        <select
          value={filters.strain}
          onChange={(e) =>
            setFilters((f) => ({ ...f, strain: e.target.value }))
          }
        >
          <option value="ALL">All Strains</option>
          <option value="Okayama">Okayama</option>
          <option value="Oita">Oita</option>
        </select>

        <select
          value={filters.feeding}
          onChange={(e) =>
            setFilters((f) => ({ ...f, feeding: e.target.value }))
          }
        >
          <option value="ALL">All Feeding</option>
          <option value="fed">fed</option>
          <option value="unfed">unfed</option>
        </select>
      </div>

      {loading && <div className={styles.loading}>Loading IGV...</div>}

      <div ref={igvContainerRef} className={styles.viewer} />
    </div>
  );
}
```
### IGVViewer.module.css
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

/* 🔥 NEW: stack title + label */
.header > div {
  display: flex;
  flex-direction: column;
}

/* 🔥 NEW: default label */
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
### App.tsx
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";

import { fetchPHASLoci } from "./services/phasService";
import type { PHASLocus } from "./types/phas";
import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

/* =========================
   🔥 Helper: parse best_region
========================= */
function parseRegion(region: string) {
  const [chrom, coords] = region.split(":");
  const [start, end] = coords.split("-").map(Number);
  return { chrom, start, end };
}

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

  // =============================
  // 🔥 CENTRAL SELECTION HANDLER
  // =============================
  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
  ) => {
    setSelectedPhas(phas);

    let derivedSample = sample;
    let derivedStage = stage;

    // 🔥 ALWAYS derive BEST SAMPLE + STAGE
    if (phas.best_sample) {
      const match = phas.best_sample.match(/N\d+/);
      if (match) {
        derivedSample = match[0];
      }

      const bestStage = getStageFromSample(phas.best_sample);
      if (bestStage) {
        derivedStage = bestStage;
      }
    }

    setSelectedSample(derivedSample || null);
    setActiveStage(derivedStage || "ALL");
    setView("all");
  };

  // =============================
  // 🔷 INITIAL LOAD
  // =============================
  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        // 🔥 DEFAULT = PHAS22-1 → BEST SAMPLE + REGION
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
  // 🔷 SEARCH (FILTER ONLY)
  // =============================
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

    // 🔥 Auto-select exact match
    const exact = phasRecords.find(
      (r) => r.phas_id.toLowerCase() === q
    );

    if (exact && exact.phas_id !== selectedPhas?.phas_id) {
      handleSelectPhasAndStage(exact);
    }
  }, [searchQuery, phasRecords]);

  // =============================
  // 🔷 LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);
      });
  }, [selectedPhas]);

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
          onSelect={(phasId) => {
            const phas = phasRecords.find((r) => r.phas_id === phasId);
            if (phas) {
              setSearchQuery(phasId); // optional but good UX
              handleSelectPhasAndStage(phas);
            }
          }}
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

            {/* 🔥 IGV PANEL (BEST REGION DEFAULT) */}
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
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
```