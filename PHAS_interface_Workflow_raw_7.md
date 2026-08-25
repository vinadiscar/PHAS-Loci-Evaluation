05.6.2026
##### Upgrade and connect search bar to Phas table, Phasing pattern panel, and IGV panel
##### PHAS_interface_Workflow_raw_7
> This is a raw file for adding Confidence Panel to show scores and confidence level of PHAS loci

```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```
---
# Phase 5: Confidence Panel Integration ver. 1
- Interactive scoring (0–2 dropdowns)
- Live total and confidence classification
- Grouped UI (Phasing, Register, IGV, Statistical)
- Persistence using localStorage
- Clean integration with existing PHASTable

## File Structure 
##### Create:
```sh
src/components/ConfidencePanel/
├── ConfidencePanel.tsx
├── ConfidencePanel.module.css
```
##### Create:
```sh
src/utils/confidenceUtils.ts
```
##### Modify:
```sh
src/App.tsx
```
## 1. Create `confidenceUtils.ts`

- #### `src/utils/confidenceUtils.ts`
```sh
cd frontend/src/utils
vi confidenceUtils.ts
```
```sh
// src/utils/confidenceUtils.ts

export type ConfidenceLevel = "High" | "Moderate" | "Low";

export interface Scores {
  distBest: number;
  distCross: number;
  regBest: number;
  regCross: number;
  igvBest: number;
  igvCross: number;
  pValue: number;
}

export function computeTotal(scores: Scores): number {
  return (
    scores.distBest +
    scores.distCross +
    scores.regBest +
    scores.regCross +
    scores.igvBest +
    scores.igvCross +
    scores.pValue
  );
}

export function classifyScore(total: number): ConfidenceLevel {
  if (total >= 11) return "High";
  if (total >= 7) return "Moderate";
  return "Low";
}

export function getInterpretation(level: ConfidenceLevel): string {
  switch (level) {
    case "High":
      return "Strong phasing, clear periodicity, and cross-sample support";
    case "Moderate":
      return "Partial phasing or limited reproducibility";
    case "Low":
      return "Weak, inconsistent, or likely non-PHAS";
  }
}
```

## 2. Create `ConfidencePanel.tsx`
- ##### src/components/ConfidencePanel/ConfidencePanel.tsx
```sh
cd frontend/src/components/
mkdir ConfidencePanel
vi ConfidencePanel.tsx
```

- #### ConfidencePanel.tsx ver. 1 

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

export default function ConfidencePanel({ locus }: Props) {
  const [scores, setScores] = useState<Scores>(defaultScores);

  // ✍️ NEW: notes per locus
  const [notes, setNotes] = useState("");

  // Load scores
  useEffect(() => {
    const saved = localStorage.getItem("confidenceScores");
    if (saved) setScores(JSON.parse(saved));
  }, []);

  // Save scores
  useEffect(() => {
    localStorage.setItem("confidenceScores", JSON.stringify(scores));
  }, [scores]);

  // Load notes per locus
  useEffect(() => {
    if (!locus?.id) return;

    const savedNotes = localStorage.getItem(`notes_${locus.id}`);
    setNotes(savedNotes || "");
  }, [locus]);

  const update = (key: keyof Scores, value: number) => {
    setScores((prev) => ({ ...prev, [key]: value }));
  };

  const total = computeTotal(scores);
  const level = classifyScore(total);
  const interpretation = getInterpretation(level);

  const badgeClass =
    level === "High"
      ? styles.high
      : level === "Moderate"
      ? styles.moderate
      : styles.low;

  const Select = (value: number, key: keyof Scores) => (
    <select
      value={value}
      onChange={(e) => update(key, Number(e.target.value))}
    >
      <option value={0}>0</option>
      <option value={1}>1</option>
      <option value={2}>2</option>
    </select>
  );

  // 💾 SAVE FUNCTION (future PHAS table ready)
  const handleSave = () => {
    if (!locus?.id) return;

    const payload = {
      locusId: locus.id,
      scores,
      total,
      level,
      interpretation,
      notes,
      timestamp: new Date().toISOString(),
    };

    // per-locus notes persistence
    localStorage.setItem(`notes_${locus.id}`, notes);

    // full analysis snapshot (future PHAS table integration)
    localStorage.setItem(
      `phas_analysis_${locus.id}`,
      JSON.stringify(payload)
    );

    console.log("Saved PHAS analysis:", payload);
  };

  return (
    <div className={styles.panel}>
      <h3>Confidence Panel</h3>

      {!locus && <p>Select a locus to begin evaluation</p>}

      {locus && (
        <>
          <div className={styles.totalBox}>
            <span className={styles.total}>{total} / 14</span>
            <span className={`${styles.badge} ${badgeClass}`}>
              {level}
            </span>
          </div>

          {/* Phasing */}
          <div className={styles.section}>
            <h4>Phasing</h4>
            <div className={styles.row}>
              Best sample {Select(scores.distBest, "distBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.distCross, "distCross")}
            </div>
          </div>

          {/* Register */}
          <div className={styles.section}>
            <h4>Register</h4>
            <div className={styles.row}>
              Best sample {Select(scores.regBest, "regBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.regCross, "regCross")}
            </div>
          </div>

          {/* IGV */}
          <div className={styles.section}>
            <h4>IGV Pattern</h4>
            <div className={styles.row}>
              Best sample {Select(scores.igvBest, "igvBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.igvCross, "igvCross")}
            </div>
          </div>

          {/* Statistical */}
          <div className={styles.section}>
            <h4>Statistical Support</h4>
            <div className={styles.row}>
              P-value {Select(scores.pValue, "pValue")}
            </div>
          </div>

          {/* Interpretation */}
          <div className={styles.interpretation}>
            <strong>Interpretation:</strong>
            <p>{interpretation}</p>
          </div>

          {/* ✍️ NOTES */}
          <div className={styles.notesSection}>
            <strong>Notes</strong>
            <textarea
              className={styles.notesInput}
              placeholder="Write your observations about this locus..."
              value={notes}
              onChange={(e) => setNotes(e.target.value)}
            />
          </div>

          {/* 💾 SAVE */}
          <button className={styles.saveButton} onClick={handleSave}>
            Save Analysis
          </button>
        </>
      )}
    </div>
  );
```
## 3. Create `ConfidencePanel.module.css`
- ##### src/components/ConfidencePanel/ConfidencePanel.module.css

```sh
cd frontend/src/components/
mkdir ConfidencePanel
vi ConfidencePanel.module.css
```

- #### ConfidencePanel.module.css


```sh
.panel {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #f4f6f8;
  border-left: 1px solid #dcdcdc;
}

/* HEADER (your updated deep ocean version kept) */
.header {
  background: linear-gradient(90deg, #1f4d4d, #123941);
  color: white;
  padding: 12px 16px;
  font-weight: 600;
  border-bottom: 1px solid rgba(47, 62, 70, 0.25);
}

.header h3 {
  margin: 0;
  font-size: 16px;
}

/* CONTENT */
.content {
  padding: 14px;
  overflow-y: auto;
}

/* TOTAL */
.totalBox {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
}

.total {
  font-size: 26px;
  font-weight: bold;
}

.percent {
  font-size: 12px;
  color: #666;
}

/* BADGE */
.badge {
  padding: 6px 12px;
  border-radius: 999px;
  color: white;
  font-size: 12px;
  font-weight: 600;
}

.high {
  background: #2e7d32;
}

.moderate {
  background: #f9a825;
  color: #222;
}

.low {
  background: #c62828;
}

/* SECTION CARDS */
.section {
  background: white;
  border-radius: 8px;
  padding: 10px;
  margin-bottom: 10px;
  border: 1px solid #e0e0e0;
}

.section h4 {
  margin: 0 0 6px 0;
  font-size: 13px;
  color: #444;
}

/* ROW */
.row {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  margin: 4px 0;
  font-size: 13px;
}

/* SELECT */
select {
  padding: 2px 4px;
  font-size: 12px;
}

/* INTERPRETATION */
.interpretation {
  background: #ffffff;
  border: 1px solid #e0e0e0;
  border-left: 4px solid #4caf50;
  border-radius: 6px;
  padding: 10px;
  margin-top: 12px;
  font-size: 13px;
}

.interpretation strong {
  display: block;
  margin-bottom: 4px;
}

/* ========================= */
/* ✍️ NOTES SECTION (NEW) */
/* ========================= */

.notesSection {
  margin-top: 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.notesSection strong {
  font-size: 13px;
  color: #444;
}

.notesInput {
  min-height: 80px;
  resize: vertical;

  padding: 8px;
  font-size: 13px;

  border: 1px solid #dcdcdc;
  border-radius: 6px;

  background: #ffffff;
  color: #333;

  outline: none;
}

.notesInput:focus {
  border-color: #2f7f7f;
  box-shadow: 0 0 0 2px rgba(47, 127, 127, 0.15);
}

/* ========================= */
/* 💾 SAVE BUTTON (NEW) */
/* ========================= */

.saveButton {
  margin-top: 10px;
  padding: 8px 12px;

  background: #2f7f7f;
  color: white;

  border: none;
  border-radius: 6px;

  font-size: 13px;
  font-weight: 600;

  cursor: pointer;

  transition: 0.2s ease;
}

.saveButton:hover {
  background: #256666;
}

.saveButton:active {
  transform: scale(0.98);
}
```
## 4. Modify `src/App.tsx`

##### Previous Code: `App.tsx`
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
- ##### New Code: `App.tsx`
```sh
import { useState, useEffect } from "react";
import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";
import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";
import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel"; // ✅ NEW

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
  // 🔷 SEARCH
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
              setSearchQuery(phasId);
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
          <div
            className="mainContent"
            style={{ display: "flex", gap: "10px" }} // ✅ layout
          >
            {/* LEFT SIDE (existing panels) */}
            <div style={{ flex: 3 }}>
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
                setStage={setActiveStage}
                setView={setView}
              />

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

            {/* RIGHT SIDE (NEW PANEL) */}
            <div style={{ flex: 1, minWidth: "300px" }}>
              <ConfidencePanel locus={selectedPhas} />
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
# Phase 5: Confidence Panel Integration ver. 2
Upgrade:
- add free text "Notes" section
- add "Save button" 

## ConfidencePanel.tsx ver. 2
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

export default function ConfidencePanel({ locus }: Props) {
  const [scores, setScores] = useState<Scores>(defaultScores);

  // ✍️ NEW: notes per locus
  const [notes, setNotes] = useState("");

  // Load scores
  useEffect(() => {
    const saved = localStorage.getItem("confidenceScores");
    if (saved) setScores(JSON.parse(saved));
  }, []);

  // Save scores
  useEffect(() => {
    localStorage.setItem("confidenceScores", JSON.stringify(scores));
  }, [scores]);

  // Load notes per locus
  useEffect(() => {
    if (!locus?.id) return;

    const savedNotes = localStorage.getItem(`notes_${locus.id}`);
    setNotes(savedNotes || "");
  }, [locus]);

  const update = (key: keyof Scores, value: number) => {
    setScores((prev) => ({ ...prev, [key]: value }));
  };

  const total = computeTotal(scores);
  const level = classifyScore(total);
  const interpretation = getInterpretation(level);

  const badgeClass =
    level === "High"
      ? styles.high
      : level === "Moderate"
      ? styles.moderate
      : styles.low;

  const Select = (value: number, key: keyof Scores) => (
    <select
      value={value}
      onChange={(e) => update(key, Number(e.target.value))}
    >
      <option value={0}>0</option>
      <option value={1}>1</option>
      <option value={2}>2</option>
    </select>
  );

  // 💾 SAVE FUNCTION (future PHAS table ready)
  const handleSave = () => {
    if (!locus?.id) return;

    const payload = {
      locusId: locus.id,
      scores,
      total,
      level,
      interpretation,
      notes,
      timestamp: new Date().toISOString(),
    };

    localStorage.setItem(`notes_${locus.id}`, notes);

    localStorage.setItem(
      `phas_analysis_${locus.id}`,
      JSON.stringify(payload)
    );

    console.log("Saved PHAS analysis:", payload);
  };

  return (
    <div className={styles.panel}>

      {/* ✅ FIXED HEADER */}
      <div className={styles.header}>
        <h3>Confidence Evaluation</h3>
      </div>

      {!locus && <p>Select a locus to begin evaluation</p>}

      {locus && (
        <>
          <div className={styles.totalBox}>
            <span className={styles.total}>{total} / 14</span>
            <span className={`${styles.badge} ${badgeClass}`}>
              {level}
            </span>
          </div>

          {/* Phasing */}
          <div className={styles.section}>
            <h4>Phasing</h4>
            <div className={styles.row}>
              Best sample {Select(scores.distBest, "distBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.distCross, "distCross")}
            </div>
          </div>

          {/* Register */}
          <div className={styles.section}>
            <h4>Register</h4>
            <div className={styles.row}>
              Best sample {Select(scores.regBest, "regBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.regCross, "regCross")}
            </div>
          </div>

          {/* IGV */}
          <div className={styles.section}>
            <h4>IGV Pattern</h4>
            <div className={styles.row}>
              Best sample {Select(scores.igvBest, "igvBest")}
            </div>
            <div className={styles.row}>
              Cross-sample {Select(scores.igvCross, "igvCross")}
            </div>
          </div>

          {/* Statistical */}
          <div className={styles.section}>
            <h4>Statistical Support</h4>
            <div className={styles.row}>
              P-value {Select(scores.pValue, "pValue")}
            </div>
          </div>

          {/* Interpretation */}
          <div className={styles.interpretation}>
            <strong>Interpretation:</strong>
            <p>{interpretation}</p>
          </div>

          {/* ✍️ NOTES */}
          <div className={styles.notesSection}>
            <strong>Notes</strong>
            <textarea
              className={styles.notesInput}
              placeholder="Write your observations about this locus..."
              value={notes}
              onChange={(e) => setNotes(e.target.value)}
            />
          </div>

          {/* 💾 SAVE */}
          <button className={styles.saveButton} onClick={handleSave}>
            Save Analysis
          </button>
        </>
      )}
    </div>
  );
}
```
- ## ConfidencePanel.module.css ver. 2
```sh
.panel {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #f4f6f8;
  border-left: 1px solid #dcdcdc;
}

/* HEADER (your updated deep ocean version kept) */
.header {
  background: linear-gradient(90deg, #163f3f, #0f2f36);
  color: #ffffff;

  padding: 12px 16px;
  font-weight: 600;

  border-bottom: 1px solid rgba(42, 167, 161, 0.25);

  /* subtle depth so it feels like a real panel header */
  box-shadow: 0 1px 0 rgba(0, 0, 0, 0.08);

  /* slight accent separation from content */
  position: relative;
}

/* thin accent line (gives "tool header" feel like IGV panels) */
.header::after {
  content: "";
  position: absolute;
  left: 0;
  bottom: 0;

  width: 100%;
  height: 2px;

  background: linear-gradient(
    90deg,
    rgba(42, 167, 161, 0.0),
    rgba(74, 214, 195, 0.5),
    rgba(42, 167, 161, 0.0)
  );
}

.header h3 {
  margin: 0;
  font-size: 16px;
}

/* CONTENT */
.content {
  padding: 14px;
  overflow-y: auto;
}

/* TOTAL */
.totalBox {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
}

.total {
  font-size: 26px;
  font-weight: bold;
}

.percent {
  font-size: 12px;
  color: #666;
}

/* BADGE */
.badge {
  padding: 6px 12px;
  border-radius: 999px;
  color: white;
  font-size: 12px;
  font-weight: 600;
}

.high {
  background: #2e7d32;
}

.moderate {
  background: #f9a825;
  color: #222;
}

.low {
  background: #c62828;
}

/* SECTION CARDS */
.section {
  background: white;
  border-radius: 8px;
  padding: 10px;
  margin-bottom: 10px;
  border: 1px solid #e0e0e0;
}

.section h4 {
  margin: 0 0 6px 0;
  font-size: 13px;
  color: #444;
}

/* ROW */
.row {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  margin: 4px 0;
  font-size: 13px;
}

/* SELECT */
select {
  padding: 2px 4px;
  font-size: 12px;
}

/* INTERPRETATION */
.interpretation {
  background: #ffffff;
  border: 1px solid #e0e0e0;
  border-left: 4px solid #4caf50;
  border-radius: 6px;
  padding: 10px;
  margin-top: 12px;
  font-size: 13px;
}

.interpretation strong {
  display: block;
  margin-bottom: 4px;
}

/* ========================= */
/* ✍️ NOTES SECTION (NEW) */
/* ========================= */

.notesSection {
  margin-top: 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.notesSection strong {
  font-size: 13px;
  color: #444;
}

.notesInput {
  min-height: 80px;
  resize: vertical;

  padding: 8px;
  font-size: 13px;

  border: 1px solid #dcdcdc;
  border-radius: 6px;

  background: #ffffff;
  color: #333;

  outline: none;
}

.notesInput:focus {
  border-color: #2f7f7f;
  box-shadow: 0 0 0 2px rgba(47, 127, 127, 0.15);
}

/* ========================= */
/* 💾 SAVE BUTTON (NEW) */
/* ========================= */

.saveButton {
  margin-top: 10px;
  padding: 8px 12px;

  background: #2f7f7f;
  color: white;

  border: none;
  border-radius: 6px;

  font-size: 13px;
  font-weight: 600;

  cursor: pointer;

  transition: 0.2s ease;
}

.saveButton:hover {
  background: #256666;
}

.saveButton:active {
  transform: scale(0.98);
}
```
--- 

# Confidence Panel ver. 3

###### Upgrade System Behavior:
- connect Confidence Panel with the table
- When user clicks a row in the table: 
1. Click row in PHAS table
sets selectedLocus
2. Confidence Panel updates
shows:
score
confidence level
notes
previous analysis (if any)
3. User edits + saves in panel
updates global evaluations[locusId]
4. PHAS table updates automatically
New columns reflect:
Confidence Level
Notes

### Previous Code: `PHASTable.tsx`

```sh
import { useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";
import { getStageFromSample } from "../../utils/sampleMap";

import { AgGridReact } from "ag-grid-react";
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";

import { ModuleRegistry, AllCommunityModule } from "ag-grid-community";
ModuleRegistry.registerModules([AllCommunityModule]);

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
  const columnDefs = useMemo(() => {
    if (records.length === 0) return [];

    return Object.keys(records[0]).map((col) => {
      const key = col.toLowerCase();

      return {
        headerName: formatHeader(col),
        field: col,
        sortable: true,
        filter:
          key.includes("chr")
            ? "agSetColumnFilter"
            : key.includes("pvalue") ||
              key.includes("ratio") ||
              key.includes("phase") ||
              key.includes("length") ||
              key.includes("start") ||
              key.includes("end")
            ? "agNumberColumnFilter"
            : "agTextColumnFilter",

        valueFormatter: (params: any) => {
          return formatValue(col, params.value);
        },

        cellRenderer: (params: any) => {
          const value = params.value;
          const row = params.data;

          const isBestSample =
            col.toLowerCase().includes("best") &&
            col.toLowerCase().includes("sample");

          if (isBestSample && typeof value === "string") {
            const sampleMatch = value.match(/N\d+/);
            const sample = sampleMatch ? sampleMatch[0] : null;

            return (
              <span
                style={{ color: "#007bff", cursor: "pointer" }}
                onClick={(e) => {
                  e.stopPropagation();
                  const stage = getStageFromSample(value);
                  onSelectPhasAndStage(row, stage, sample);
                }}
              >
                {value}
              </span>
            );
          }

          return (
            <span className={getCellClass(col)}>
              {formatValue(col, value)}
            </span>
          );
        },
      };
    });
  }, [records]);

  const defaultColDef = {
    minWidth: 120,
    filter: true,
    floatingFilter: true, // 🔥 Excel-style input row
    resizable: true,
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
          <div
            className="ag-theme-alpine"
            style={{ height: 500, width: "100%" }}
          >
            <AgGridReact
              rowData={records}
              columnDefs={columnDefs}
              defaultColDef={defaultColDef}
              animateRows={true}
              rowSelection="single"

              onRowClicked={(event) => {
                const r = event.data;
                const sampleMatch = r.best_sample?.match(/N\d+/);
                const sample = sampleMatch ? sampleMatch[0] : null;

                onSelectPhasAndStage(r, null, sample);
              }}

              getRowStyle={(params) => {
                if (params.data.phas_id === selectedId) {
                  return {
                    backgroundColor: "#d1e7ff",
                    borderLeft: "4px solid #3498db",
                  };
                }
                return {};
              }}
            />
          </div>
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
  if (key.includes("phase")) return styles.phase;
  if (key.includes("pvalue")) return styles.pvalue;
  if (key.includes("abundance")) return styles.abundance;

  return "";
}
```

## Updated Version: `PHASTable.tsx`
```sh
import { useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";
import { getStageFromSample } from "../../utils/sampleMap";

import { AgGridReact } from "ag-grid-react";
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";

import { ModuleRegistry, AllCommunityModule } from "ag-grid-community";
ModuleRegistry.registerModules([AllCommunityModule]);

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
      ...Object.keys(records[0]).filter((k) => k !== "__evaluation").map((col) => ({
        headerName: col,
        field: col,
        sortable: true,
        filter: true,
      })),

      // ✅ NEW COLUMN: Confidence Level
      {
        headerName: "Confidence Level",
        valueGetter: (params: any) =>
          params.data.__evaluation?.level || "-",
      },

      // ✅ NEW COLUMN: Notes
      {
        headerName: "Notes",
        valueGetter: (params: any) =>
          params.data.__evaluation?.notes || "-",
      },
    ];
  }, [records]);

  return (
    <div className={styles.container}>
      {loading ? (
        <div className={styles.loading}>Loading...</div>
      ) : (
        <div className="ag-theme-alpine" style={{ height: 500, width: "100%" }}>
          <AgGridReact
            rowData={records}
            columnDefs={columnDefs}
            rowSelection="single"
            onRowClicked={(event) => {
              const r = event.data;
              const sample = r.best_sample?.match(/N\d+/)?.[0] || null;
              onSelectPhasAndStage(r, null, sample);
            }}
            getRowStyle={(params) =>
              params.data.phas_id === selectedId
                ? { backgroundColor: "#d1e7ff", borderLeft: "4px solid #3498db" }
                : undefined
            }
          />
        </div>
      )}
    </div>
  );
}
```

## App.tsx ver. 2

```sh
import { useState, useEffect } from "react";
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

  const [view, setView] = useState<"read5prime" | "register" | "all" | "hide">("all");

  const [selectedSample, setSelectedSample] = useState<string | null>(null);

  // ✅ GLOBAL EVALUATION STORE
  const [evaluations, setEvaluations] = useState<Record<string, any>>({});

  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
    sample?: string | null
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

  useEffect(() => {
    setLoading(true);

    fetchPHASLoci()
      .then((data) => {
        setPhasRecords(data.records);
        setFilteredRecords(data.records);

        const defaultPhas = data.records.find((r) => r.phas_id === "PHAS22-1");
        if (defaultPhas) handleSelectPhasAndStage(defaultPhas);
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
  }, [searchQuery, phasRecords]);

  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        const stages = data[selectedPhas.phas_id] || [];
        setAvailableStages(stages);
      });
  }, [selectedPhas]);

  // ✅ enrich rows with evaluation data
  const enrichedRecords = filteredRecords.map((r) => ({
    ...r,
    __evaluation: evaluations[r.phas_id],
  }));

  return (
    <div className="app">
      <Header currentState={{}} onExportSVG={() => {}} onExportPNG={() => {}} isIGVReady={!!selectedSample} />

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

        <PHASTable
          records={enrichedRecords}
          selectedId={selectedPhas?.phas_id || null}
          onSelectPhasAndStage={handleSelectPhasAndStage}
          loading={loading}
        />

        {selectedPhas && (
          <div className="mainContent" style={{ display: "flex", gap: "10px" }}>
            <div style={{ flex: 3 }}>
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
                setStage={setActiveStage}
                setView={setView}
              />

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

            <div style={{ flex: 1, minWidth: "300px" }}>
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={evaluations[selectedPhas?.phas_id]}
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
    </div>
  );
}
```


## ConfidencePanel.tsx ver.3 
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

export default function ConfidencePanel({ locus, evaluation, onSave }: Props) {
  const [scores, setScores] = useState<Scores>(defaultScores);
  const [notes, setNotes] = useState("");

  useEffect(() => {
    if (!locus) return;

    if (evaluation) {
      setScores(evaluation.scores);
      setNotes(evaluation.notes || "");
    } else {
      setScores(defaultScores);
      setNotes("");
    }
  }, [locus, evaluation]);

  const update = (key: keyof Scores, value: number) => {
    setScores((prev) => ({ ...prev, [key]: value }));
  };

  const total = computeTotal(scores);
  const level = classifyScore(total);
  const interpretation = getInterpretation(level);

  const handleSave = () => {
    if (!locus?.phas_id) return;

    const payload = {
      scores,
      total,
      level,
      interpretation,
      notes,
      timestamp: new Date().toISOString(),
    };

    localStorage.setItem(
      `phas_analysis_${locus.phas_id}`,
      JSON.stringify(payload)
    );

    onSave(locus.phas_id, payload);
  };

  const Select = (value: number, key: keyof Scores) => (
    <select value={value} onChange={(e) => update(key, Number(e.target.value))}>
      <option value={0}>0</option>
      <option value={1}>1</option>
      <option value={2}>2</option>
    </select>
  );

  const badgeClass =
    level === "High"
      ? styles.high
      : level === "Moderate"
      ? styles.moderate
      : styles.low;

  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>Confidence Evaluation</h3>
      </div>

      {!locus && <p>Select a locus</p>}

      {locus && (
        <>
          <div className={styles.totalBox}>
            <span className={styles.total}>{total} / 14</span>
            <span className={`${styles.badge} ${badgeClass}`}>{level}</span>
          </div>

          <div className={styles.section}>
            <h4>Phasing</h4>
            <div>Best {Select(scores.distBest, "distBest")}</div>
            <div>Cross {Select(scores.distCross, "distCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>Register</h4>
            <div>Best {Select(scores.regBest, "regBest")}</div>
            <div>Cross {Select(scores.regCross, "regCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>IGV</h4>
            <div>Best {Select(scores.igvBest, "igvBest")}</div>
            <div>Cross {Select(scores.igvCross, "igvCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>Statistical</h4>
            <div>P-value {Select(scores.pValue, "pValue")}</div>
          </div>

          <div className={styles.interpretation}>
            <strong>{interpretation}</strong>
          </div>

          <textarea
            value={notes}
            onChange={(e) => setNotes(e.target.value)}
            placeholder="Notes..."
          />

          <button onClick={handleSave}>Save Analysis</button>
        </>
      )}
    </div>
  );
}
```

## PHASTable.tsx ver 2

```sh
import { useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";
import { getStageFromSample } from "../../utils/sampleMap";

import { AgGridReact } from "ag-grid-react";
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";

import { ModuleRegistry, AllCommunityModule } from "ag-grid-community";
ModuleRegistry.registerModules([AllCommunityModule]);

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
      ...Object.keys(records[0]).filter((k) => k !== "__evaluation").map((col) => ({
        headerName: col,
        field: col,
        sortable: true,
        filter: true,
      })),

      // ✅ NEW COLUMN: Confidence Level
      {
        headerName: "Confidence Level",
        valueGetter: (params: any) =>
          params.data.__evaluation?.level || "-",
      },

      // ✅ NEW COLUMN: Notes
      {
        headerName: "Notes",
        valueGetter: (params: any) =>
          params.data.__evaluation?.notes || "-",
      },
    ];
  }, [records]);

  return (
    <div className={styles.container}>
      {loading ? (
        <div className={styles.loading}>Loading...</div>
      ) : (
        <div className="ag-theme-alpine" style={{ height: 500, width: "100%" }}>
          <AgGridReact
            rowData={records}
            columnDefs={columnDefs}
            rowSelection="single"
            onRowClicked={(event) => {
              const r = event.data;
              const sample = r.best_sample?.match(/N\d+/)?.[0] || null;
              onSelectPhasAndStage(r, null, sample);
            }}
            getRowStyle={(params) =>
              params.data.phas_id === selectedId
                ? { backgroundColor: "#d1e7ff", borderLeft: "4px solid #3498db" }
                : undefined
            }
          />
        </div>
      )}
    </div>
  );
}
```
- #### It worked! 
#### Issue:
- Although the table automatically updates the "Confidence Level" and "Notes" whenever the user update his/her evaluation in the Confidence Panel, but the analysis is not being saved whenever the interface is being refreshed or closed. 
- layout doesn't look good
#### Solve 1:
- add a rehydration step in App.tsx.

## App.tsx ver. 3
```sh
import { useState, useEffect } from "react";
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

  const [view, setView] = useState<"read5prime" | "register" | "all" | "hide">("all");

  const [selectedSample, setSelectedSample] = useState<string | null>(null);

  // ✅ GLOBAL STATE
  const [evaluations, setEvaluations] = useState<Record<string, any>>({});

  // ✅ HYDRATION FLAG (CRITICAL FIX)
  const [hydrated, setHydrated] = useState(false);

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
  // 🔥 FIXED HYDRATION (LOCALSTORAGE → STATE)
  // =============================
  useEffect(() => {
    const loaded: Record<string, any> = {};

    Object.keys(localStorage).forEach((key) => {
      if (!key.startsWith("phas_analysis_")) return;

      try {
        const raw = localStorage.getItem(key);
        if (!raw) return;

        const data = JSON.parse(raw);

        if (!data?.locusId) return;

        loaded[data.locusId] = data;
      } catch (e) {}
    });

    setEvaluations(loaded);
    setHydrated(true);
  }, []);

  // =============================
  // SEARCH
  // =============================
  useEffect(() => {
    const q = searchQuery.trim().toLowerCase();

    if (!q) {
      setFilteredRecords(phasRecords);
      return;
    }

    setFilteredRecords(
      phasRecords.filter((r) =>
        r.phas_id.toLowerCase().includes(q)
      )
    );
  }, [searchQuery, phasRecords]);

  // =============================
  // LOAD STAGES
  // =============================
  useEffect(() => {
    if (!selectedPhas) return;

    fetch("/phas_plots/organize_phas_images/stages.json")
      .then((res) => res.json())
      .then((data) => {
        setAvailableStages(data[selectedPhas.phas_id] || []);
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
      const match = phas.best_sample.match(/N\d+/);
      if (match) derivedSample = match[0];

      const bestStage = getStageFromSample(phas.best_sample);
      if (bestStage) derivedStage = bestStage;
    }

    setSelectedSample(derivedSample || null);
    setActiveStage(derivedStage || "ALL");
    setView("all");
  };

  // =============================
  // ENRICH TABLE DATA
  // =============================
  const enrichedRecords = filteredRecords.map((r) => ({
    ...r,
    __evaluation: evaluations[r.phas_id],
  }));

  return (
    <div className="app">
      <Header currentState={{}} onExportSVG={() => {}} onExportPNG={() => {}} isIGVReady={!!selectedSample} />

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

        {/* ✅ ONLY RENDER AFTER HYDRATION */}
        {hydrated && (
          <PHASTable
            records={enrichedRecords}
            selectedId={selectedPhas?.phas_id || null}
            onSelectPhasAndStage={handleSelectPhasAndStage}
            loading={loading}
          />
        )}

        {selectedPhas && (
          <div className="mainContent" style={{ display: "flex", gap: "10px" }}>
            <div style={{ flex: 3 }}>
              <PhasingPatternPanel
                phasId={selectedPhas.phas_id}
                stage={activeStage}
                view={view}
                allStages={availableStages}
                setStage={setActiveStage}
                setView={setView}
              />

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

            <div style={{ flex: 1, minWidth: "300px" }}>
              <ConfidencePanel
                locus={selectedPhas}
                evaluation={evaluations[selectedPhas?.phas_id]}
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
    </div>
  );
}
```

## ConfidencePanel.tsx ver. 3
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
  const [scores, setScores] = useState<Scores>(defaultScores);
  const [notes, setNotes] = useState("");

  // =============================
  // 🔥 SINGLE SOURCE OF TRUTH SYNC
  // =============================
  useEffect(() => {
    if (!locus?.phas_id) return;

    // 1. PRIORITY: App state (evaluation prop)
    if (evaluation) {
      setScores(evaluation.scores || defaultScores);
      setNotes(evaluation.notes || "");
      return;
    }

    // 2. FALLBACK: localStorage
    const saved = localStorage.getItem(
      `phas_analysis_${locus.phas_id}`
    );

    if (saved) {
      const parsed = JSON.parse(saved);
      setScores(parsed.scores || defaultScores);
      setNotes(parsed.notes || "");
    } else {
      setScores(defaultScores);
      setNotes("");
    }
  }, [locus, evaluation]);

  const update = (key: keyof Scores, value: number) => {
    setScores((prev) => ({ ...prev, [key]: value }));
  };

  const total = computeTotal(scores);
  const level = classifyScore(total);
  const interpretation = getInterpretation(level);

  const handleSave = () => {
    if (!locus?.phas_id) return;

    const payload = {
      locusId: locus.phas_id,
      scores,
      total,
      level,
      interpretation,
      notes,
      timestamp: new Date().toISOString(),
    };

    localStorage.setItem(
      `phas_analysis_${locus.phas_id}`,
      JSON.stringify(payload)
    );

    onSave(locus.phas_id, payload);
  };

  const Select = (value: number, key: keyof Scores) => (
    <select
      value={value}
      onChange={(e) => update(key, Number(e.target.value))}
    >
      <option value={0}>0</option>
      <option value={1}>1</option>
      <option value={2}>2</option>
    </select>
  );

  const badgeClass =
    level === "High"
      ? styles.high
      : level === "Moderate"
      ? styles.moderate
      : styles.low;

  return (
    <div className={styles.panel}>
      <div className={styles.header}>
        <h3>Confidence Evaluation</h3>
      </div>

      {!locus && <p>Select a locus</p>}

      {locus && (
        <>
          <div className={styles.totalBox}>
            <span className={styles.total}>{total} / 14</span>
            <span className={`${styles.badge} ${badgeClass}`}>
              {level}
            </span>
          </div>

          <div className={styles.section}>
            <h4>Phasing</h4>
            <div>Best {Select(scores.distBest, "distBest")}</div>
            <div>Cross {Select(scores.distCross, "distCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>Register</h4>
            <div>Best {Select(scores.regBest, "regBest")}</div>
            <div>Cross {Select(scores.regCross, "regCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>IGV</h4>
            <div>Best {Select(scores.igvBest, "igvBest")}</div>
            <div>Cross {Select(scores.igvCross, "igvCross")}</div>
          </div>

          <div className={styles.section}>
            <h4>Statistical</h4>
            <div>P-value {Select(scores.pValue, "pValue")}</div>
          </div>

          <div className={styles.interpretation}>
            <strong>{interpretation}</strong>
          </div>

          <textarea
            value={notes}
            onChange={(e) => setNotes(e.target.value)}
            placeholder="Notes..."
          />

          <button onClick={handleSave}>Save Analysis</button>
        </>
      )}
    </div>
  );
}
```
## PHASTable ver. 3
```sh
import { useMemo } from "react";
import type { PHASLocus } from "../../types/phas";
import styles from "./PHASTable.module.css";

import { AgGridReact } from "ag-grid-react";
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";

import { ModuleRegistry, AllCommunityModule } from "ag-grid-community";
ModuleRegistry.registerModules([AllCommunityModule]);

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
        .filter((k) => k !== "__evaluation")
        .map((col) => ({
          headerName: col,
          field: col,
          sortable: true,
          filter: true,
        })),

      {
        headerName: "Confidence Level",
        valueGetter: (p: any) => p.data.__evaluation?.level || "-",
      },
      {
        headerName: "Notes",
        valueGetter: (p: any) => p.data.__evaluation?.notes || "-",
      },
    ];
  }, [records]);

  return (
    <div className={styles.container}>
      {loading ? (
        <div className={styles.loading}>Loading...</div>
      ) : (
        <div className="ag-theme-alpine" style={{ height: 500 }}>
          <AgGridReact
            rowData={records}
            columnDefs={columnDefs}
            rowSelection="single"
            onRowClicked={(e) => {
              const r = e.data;
              const sample = r.best_sample?.match(/N\d+/)?.[0] || null;
              onSelectPhasAndStage(r, null, sample);
            }}
            getRowStyle={(p) =>
              p.data.phas_id === selectedId
                ? { backgroundColor: "#d1e7ff" }
                : undefined
            }
          />
        </div>
      )}
    </div>
  );
}
```

---
## Confidence Panel ver.4

- improve Confidence Panel Layout

### ConfidencePanel.tsx ver. 4
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

  /* =============================
     SYNC STATE
  ============================= */

  useEffect(() => {
    if (!locus?.phas_id) return;

    // Priority: evaluation prop
    if (evaluation) {
      setScores(
        evaluation.scores || defaultScores
      );

      setNotes(evaluation.notes || "");
      return;
    }

    // Fallback: localStorage
    const saved = localStorage.getItem(
      `phas_analysis_${locus.phas_id}`
    );

    if (saved) {
      const parsed = JSON.parse(saved);

      setScores(
        parsed.scores || defaultScores
      );

      setNotes(parsed.notes || "");
    } else {
      setScores(defaultScores);
      setNotes("");
    }
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

  const handleSave = () => {
    if (!locus?.phas_id) return;

    const payload = {
      locusId: locus.phas_id,
      scores,
      total,
      level,
      interpretation,
      notes,
      timestamp: new Date().toISOString(),
    };

    localStorage.setItem(
      `phas_analysis_${locus.phas_id}`,
      JSON.stringify(payload)
    );

    onSave(locus.phas_id, payload);
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
        <h3>Confidence Evaluation</h3>
      </div>

      {!locus && (
        <div className={styles.empty}>
          Select a PHAS locus
        </div>
      )}

      {locus && (
        <div className={styles.content}>

          {/* TOTAL SCORE */}
          <div className={styles.totalBox}>
            <div>
              <div className={styles.totalLabel}>
                Total Score
              </div>

              <div className={styles.total}>
                {total} / 14
              </div>

              <div className={styles.percent}>
                {percent}% confidence
              </div>
            </div>

            <span
              className={`${styles.badge} ${badgeClass}`}
            >
              {level}
            </span>
          </div>

          {/* PHASING */}
          <div className={styles.section}>
            <h4>Phasing Evidence</h4>

            <div className={styles.row}>
              <span>Best Sample</span>
              {Select(
                scores.distBest,
                "distBest"
              )}
            </div>

            <div className={styles.row}>
              <span>Cross-sample</span>
              {Select(
                scores.distCross,
                "distCross"
              )}
            </div>
          </div>

          {/* REGISTER */}
          <div className={styles.section}>
            <h4>Register Consistency</h4>

            <div className={styles.row}>
              <span>Best Sample</span>
              {Select(
                scores.regBest,
                "regBest"
              )}
            </div>

            <div className={styles.row}>
              <span>Cross-sample</span>
              {Select(
                scores.regCross,
                "regCross"
              )}
            </div>
          </div>

          {/* IGV */}
          <div className={styles.section}>
            <h4>IGV Evidence</h4>

            <div className={styles.row}>
              <span>Best Sample</span>
              {Select(
                scores.igvBest,
                "igvBest"
              )}
            </div>

            <div className={styles.row}>
              <span>Cross-sample</span>
              {Select(
                scores.igvCross,
                "igvCross"
              )}
            </div>
          </div>

          {/* STATISTICAL */}
          <div className={styles.section}>
            <h4>Statistical Support</h4>

            <div className={styles.row}>
              <span>P-value</span>
              {Select(
                scores.pValue,
                "pValue"
              )}
            </div>
          </div>

          {/* INTERPRETATION */}
          <div className={styles.interpretation}>
            <strong>
              Interpretation
            </strong>

            {interpretation}
          </div>

          {/* NOTES */}
          <div className={styles.notesSection}>

            <div className={styles.notesLabel}>
              Notes
            </div>

            <textarea
              className={styles.notesInput}
              value={notes}
              onChange={(e) =>
                setNotes(
                  e.target.value
                )
              }
              placeholder="Write your notes..."
            />

            <button
              className={styles.saveButton}
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
### ConfidencePanel.module.css ver. 4
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

  color: #ffffff;

  padding: 14px 18px;

  border-bottom: 1px solid rgba(255,255,255,0.05);

  position: relative;

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
    rgba(74, 214, 195, 0.55),
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
  font-size: 12px;
  font-weight: 600;
  color: #6b7b83;

  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.total {
  font-size: 30px;
  font-weight: 700;
  color: #173d42;
  line-height: 1;
}

.percent {
  margin-top: 3px;

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

  border: 1px solid rgba(0,0,0,0.05);
}

.high {
  background: rgba(56, 142, 60, 0.12);
  color: #2e7d32;
}

.moderate {
  background: rgba(249, 168, 37, 0.16);
  color: #8a5a00;
}

.low {
  background: rgba(198, 40, 40, 0.12);
  color: #b71c1c;
}

/* =========================
   SECTION CARDS
========================= */

.section {
  background: rgba(255,255,255,0.92);

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

/* subtle divider */
.section h4::after {
  content: "";

  flex: 1;
  height: 1px;

  background: linear-gradient(
    90deg,
    rgba(39, 88, 95, 0.18),
    rgba(39, 88, 95, 0)
  );
}

/* =========================
   ROWS
========================= */

.row {
  display: grid;

  grid-template-columns: 1fr auto;

  align-items: center;

  gap: 10px;

  padding: 6px 0;

  font-size: 13px;

  color: #41545b;

  border-bottom: 1px solid rgba(0,0,0,0.04);
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

  border: 1px solid #ccd6d9;

  background: #ffffff;

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
    0 0 0 3px rgba(47, 127, 127, 0.12);
}

/* =========================
   INTERPRETATION
========================= */

.interpretation {
  background: white;

  border-radius: 12px;

  padding: 12px 14px;

  border: 1px solid #dde5e7;

  border-left: 4px solid #43a047;

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
  gap: 6px;
}

.notesSection strong {
  font-size: 13px;
  font-weight: 650;

  color: #26464d;
}

.notesInput {
  min-height: 90px;

  resize: vertical;

  padding: 10px 12px;

  font-size: 13px;

  border-radius: 10px;

  border: 1px solid #d5dee1;

  background: #ffffff;

  color: #33444a;

  outline: none;

  transition: 0.15s ease;
}

.notesInput:focus {
  border-color: #2f7f7f;

  box-shadow:
    0 0 0 3px rgba(47,127,127,0.12);
}

/* =========================
   SAVE BUTTON
========================= */

.saveButton {
  margin-top: 4px;

  padding: 10px 14px;

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

  transition:
    transform 0.15s ease,
    box-shadow 0.15s ease,
    filter 0.15s ease;

  box-shadow:
    0 2px 6px rgba(47,127,127,0.18);
}

.saveButton:hover {
  filter: brightness(1.03);

  transform: translateY(-1px);

  box-shadow:
    0 4px 10px rgba(47,127,127,0.25);
}

.saveButton:active {
  transform: scale(0.98);
}
```
--- 
Jun 4, 2026

- Modify scoring guide partion
- Update: `src/utils/confidenceUtils.ts`

#### Previous:
```sh
Total        Score     	          Label	Interpretation
11 to 14	   High confidence	    Strong phasing, clear periodicity, and cross-sample support
7 to 10	     Moderate Confidence	Partial phasing or limited reproducibility
0-6	         Low Confidence	      Weak, inconsistent, or likely non-PHAS
```
#### Update:
```sh
Total        Score     	          Label	Interpretation
10 to 14	   High confidence	    Better phasing, clear periodicity, and cross-sample support
4 to 9	     Moderate Confidence	Partial phasing and periodicity
0-3	         Low Confidence	      Weak, inconsistent, or likely non-PHAS
```

## Updated `src/utils/confidenceUtils.ts`

```sh
cd frontend/src/utils
vi confidenceUtils.ts
```
```sh
// src/utils/confidenceUtils.ts

export type ConfidenceLevel = "High" | "Moderate" | "Low";

export interface Scores {
  distBest: number;
  distCross: number;
  regBest: number;
  regCross: number;
  igvBest: number;
  igvCross: number;
  pValue: number;
}

export function computeTotal(scores: Scores): number {
  return (
    scores.distBest +
    scores.distCross +
    scores.regBest +
    scores.regCross +
    scores.igvBest +
    scores.igvCross +
    scores.pValue
  );
}

export function classifyScore(total: number): ConfidenceLevel {
  if (total >= 10) return "High";
  if (total >= 4) return "Moderate";
  return "Low";
}

export function getInterpretation(level: ConfidenceLevel): string {
  switch (level) {
    case "High":
      return "Better phasing, clear periodicity, and cross-sample support";
    case "Moderate":
      return "Partial phasing or periodicity";
    case "Low":
      return "Weak, inconsistent, or likely non-PHAS";
  }
}
```