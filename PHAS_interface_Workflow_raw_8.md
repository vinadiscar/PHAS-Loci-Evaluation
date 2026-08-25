## Suggestions
> this addresses some additional comments for the interface

```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

1. In the interface table, add additional column "Reviewer's Comments" for Sensei's comments
2. add gene annotation track to the browser
3. add another switch to select  sRNAseq only, mRNAseq only or both
4. import Excel table Phas_manualAnnotation_ver2.xlsx in the interface

### 1. Add additional column for Sensei's comments
### PHASTable.tsx ver. 4
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
        valueGetter: (p: any) =>
          p.data.__evaluation?.level || "-",
      },

      {
        headerName: "Notes",
        valueGetter: (p: any) =>
          p.data.__evaluation?.notes || "-",
      },

      {
        headerName: "Reviewer Comments",
        valueGetter: (p: any) =>
          p.data.__evaluation?.comments || "-",
      },
    ];
  }, [records]);

  return (
    <div className={styles.container}>
      {loading ? (
        <div className={styles.loading}>
          Loading...
        </div>
      ) : (
        <div
          className="ag-theme-alpine"
          style={{ height: 500 }}
        >
          <AgGridReact
            rowData={records}
            columnDefs={columnDefs}
            rowSelection="single"
            onRowClicked={(e) => {
              const r = e.data;

              const sample =
                r.best_sample?.match(/N\d+/)?.[0] ||
                null;

              onSelectPhasAndStage(
                r,
                null,
                sample
              );
            }}
            getRowStyle={(p) =>
              p.data.phas_id === selectedId
                ? {
                    backgroundColor: "#d1e7ff",
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

### ConfidencePanel.tsx ver. 5

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

    // Priority: evaluation prop
    if (evaluation) {
      setScores(
        evaluation.scores || defaultScores
      );

      setNotes(evaluation.notes || "");
      setComments(evaluation.comments || "");

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
      setComments(parsed.comments || "");
    } else {
      setScores(defaultScores);
      setNotes("");
      setComments("");
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
      comments,
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

          {/* NOTES + COMMENTS */}
          <div className={styles.notesSection}>

            {/* NOTES */}
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

            {/* REVIEWER COMMENTS */}
            <div className={styles.notesLabel}>
              Reviewer Comments
            </div>

            <textarea
              className={styles.notesInput}
              value={comments}
              onChange={(e) =>
                setComments(
                  e.target.value
                )
              }
              placeholder="Write reviewer comments..."
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

It worked. 

##### Issue: 
- first click → jumps to top
- second click → stable
##### Cause: 
- AG Grid behavior caused by:
`columnDefs being recreated on every render` and/or `rowData reference changing`
- from App.tsx line:

    ```sh
    const enrichedRecords = filteredRecords.map((r) => ({
  ...r,
  __evaluation: evaluations[r.phas_id],
  }));
   ```
#### Solve: 
- memoize `enrichedRecords`.

- ## App.tsx ver. 4

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

  // GLOBAL STATE
  const [evaluations, setEvaluations] =
    useState<Record<string, any>>({});

  // HYDRATION FLAG
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
  // HYDRATE LOCAL STORAGE
  // =============================
  useEffect(() => {
    const loaded: Record<string, any> = {};

    Object.keys(localStorage).forEach((key) => {
      if (!key.startsWith("phas_analysis_"))
        return;

      try {
        const raw =
          localStorage.getItem(key);

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

        {/* ONLY RENDER AFTER HYDRATION */}
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
          <div
            className="mainContent"
            style={{
              display: "flex",
              gap: "10px",
            }}
          >
            <div style={{ flex: 3 }}>
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

            <div
              style={{
                flex: 1,
                minWidth: "300px",
              }}
            >
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
- Add filtering functions

## PHASTable.tsx ver. 5
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
- #### It worked!

#### Issue:
- interface uses a horizontal flex layout that does not adapt when the browser window becomes smaller. Result: right panel gets hidden, plots overflow, contents extend outside the screen.
#### Solution
Make the layout responsive by:
- allowing flex items to resize,
- stacking panels vertically on smaller screens,
- preventing horizontal overflow.

## App.tsx ver.
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
  // HYDRATE LOCAL STORAGE
  // =============================
  useEffect(() => {
    const loaded: Record<string, any> = {};

    Object.keys(localStorage).forEach((key) => {
      if (!key.startsWith("phas_analysis_"))
        return;

      try {
        const raw =
          localStorage.getItem(key);

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

## App.css
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
#### It worked!
#### Issue
- The left panel (plots + IGV) was shrinking too much when the window became small, causing IGV controls to overlap, plots to compress, and UI collisions with the Confidence Panel.
#### Solution

- keep the Confidence Panel fixed on the right,
give the left panel a safe minimum width,
and allow horizontal scrolling instead of breaking the UI.
- Edit `App.css`
## App.css ver. 

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
---
# 2. Insert gene annotation in the IGV

### Previous code:
- IGVViewer.tsx
- IGVViewer.module.css 
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
#### IGVViewer.module.css
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

## backend/main.py

```sh
(base) okamuralab@rnalab backend % cat main.py
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

## backend/app/config.py
```sh
"""PHASER Configuration"""

from pathlib import Path

# =========================================================
# 🔷 BASE DIRECTORY
# =========================================================
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# =========================================================
# 🔷 DATA
# =========================================================
DATA_DIR = BASE_DIR / "data"

PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# REQUIRED
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# =========================================================
# 🔷 TRACK HUB
# =========================================================
TRACKHUB_BASE = Path("/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks")

# =========================================================
# 🔷 GENOME
# =========================================================
GENOME_FASTA = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta"
)

GENOME_FAI = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta.fai"
)

GENOME_2BIT = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.2bit"
)

GENOME_CHROMSIZES = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.genome.chrom.sizes"
)

# =========================================================
# 🔷 ANNOTATION
# =========================================================
ANNOTATION_DIR = TRACKHUB_BASE / "annotation"

# PHAS loci track
PHAS_ANNOTATION_BB = (
    ANNOTATION_DIR / "merged.PHAS22.bb"
)

# Genome annotation
GFF3_FILE = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz"
)

GFF3_INDEX = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz.tbi"
)

# =========================================================
# �� BIGWIG DIRECTORY
# =========================================================
BW_DIR = TRACKHUB_BASE / "bw"

# =========================================================
# 🔷 API
# =========================================================
API_PREFIX = "/api"
```

## backend/app/services/track_service.py
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


# =========================================================
# 🎨 STRAIN COLOR MAP (NEW - ONLY ADDITION)
# =========================================================
STRAIN_COLORS = {
    "Okayama": "blue",
    "Oita": "#C2185B"
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
                "label": label,

                # =====================================================
                # 🎨 STRAIN COLOR (ONLY ADDITION)
                # =====================================================
                "color": STRAIN_COLORS.get(meta["strain"], "gray")
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
## backend/app/routers/tracks.py
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

## backend/app/routers/files.py
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

## backend/app/routers/files.py
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
## backend/app/routers/phas.py

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
```
### 2. Prepare annotation (gff) file 

```sh
# 1. Download gff GWH file through ftp FileZilla

# 2. Copy annnotation file
cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.gff "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/annotation"



# 4. Convert gff to gff3
conda activate gffread_env
gffread /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.gff \
-o /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.gff3

sort -k1,1 -k4,4n GWHAMMI00000000.gff3 > GWHAMMI00000000.sorted.gff3



# Copy to Volumes
cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.gff3 "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/annotation"

```
#### Note: gff3 file is heavy which causes hanging of IGV. Convert to .gff3.gz + .tbi

```sh
cd /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000

conda activate htslib_env
bgzip GWHAMMI00000000.sorted.gff3
tabix -p gff GWHAMMI00000000.gff3.gz

cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.sorted.gff3.gz "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/annotation"

cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.sorted.gff3.gz.tbi "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/annotation"
```
## Issue:
IGV not rendering Gene annoitation
## Possible cause: 
1. Converted GFF3 lacks the actual gene features instead it uses the `geneID=HaeL00001.gene` which is not valid GFF3 parent hierarchy for IGV.

## Solution 1: 
- use a GFF-aware tool that preserves parent-child structure.

```sh
# 1. Convert properly with gffread
gffread /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.gff \
-T \
-o /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.fixed.gff3

# 2. Proper sorting of gff3
gt gff3 \
-sortlines \
-tidy \
-o project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.fixed.sorted.gff3 \
/project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.fixed.gff3
```

## New Version of Codes:
- IGVViewer.tsx
- tracks_sefvice.py 
- tracks.py

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

  const [allTracks, setAllTracks] = useState<any[]>([]);

  const [loading, setLoading] = useState(false);

  const [filters, setFilters] = useState({
    samples: [sample],
    stage: "ALL",
    strain: "ALL",
    feeding: "ALL",
  });

  // =====================================================
  // 🔷 SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // 🔷 LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current) return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        // cleanup old browser
        if (igvBrowserRef.current) {

          igvBrowserRef.current.destroy?.();

          igvBrowserRef.current = null;

          isReadyRef.current = false;
        }

        const res = await fetch(
          `${API_BASE}/api/tracks/igv/all`
        );

        const config = await res.json();

        const tracksWithAbsoluteURL = config.tracks.map(
          (t: any) => ({
            ...t,

            url: API_BASE + t.url,

            indexURL: t.indexURL
              ? API_BASE + t.indexURL
              : undefined,
          })
        );

        setAllTracks(tracksWithAbsoluteURL);

        const browser = await igv.createBrowser(
          igvContainerRef.current!,
          {
            genome: {
              fastaURL:
                API_BASE +
                config.genome.fastaURL,

              indexURL:
                API_BASE +
                config.genome.indexURL,
            },

            locus: locus
              ? `${locus.chrom}:${locus.start}-${locus.end}`
              : undefined,

            tracks: [],
          }
        );

        igvBrowserRef.current = browser;

        setTimeout(() => {

          isReadyRef.current = true;

          setFilters((prev) => ({ ...prev }));

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
  // 🔷 APPLY TRACKS
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !isReadyRef.current ||
      allTracks.length === 0
    ) {
      return;
    }

    const browser = igvBrowserRef.current;

    const applyTracks = async () => {

      try {

        await browser.removeAllTracks();

        const annotationTrack = allTracks.find(
          (t) => t.format === "gff3"
        );

        let signalTracks = allTracks.filter(
          (t) => t.format !== "gff3"
        );

        signalTracks = signalTracks.filter((t) => {

          return (
            filters.samples.includes(t.sample) &&
            (filters.stage === "ALL" ||
              t.stage === filters.stage) &&
            (filters.strain === "ALL" ||
              t.strain === filters.strain) &&
            (filters.feeding === "ALL" ||
              t.feeding === filters.feeding)
          );
        });

        signalTracks.sort((a, b) =>
          a.name.localeCompare(
            b.name,
            undefined,
            {
              numeric: true,
              sensitivity: "base",
            }
          )
        );

        const finalTracks = annotationTrack
          ? [annotationTrack, ...signalTracks]
          : signalTracks;

        for (const track of finalTracks) {

          await browser.loadTrack(track);
        }

      } catch (err) {

        console.error(
          "Track loading error:",
          err
        );
      }
    };

    applyTracks();

  }, [filters, allTracks]);

  // =====================================================
  // 🔷 LOCUS UPDATE
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
  // 🔷 TOGGLE SAMPLES
  // =====================================================
  const toggleAllSamples = () => {

    const allSamples = Array.from(
      new Set(
        allTracks
          .map((t) => t.sample)
          .filter(Boolean)
      )
    );

    if (filters.samples.length === 1) {

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
        Genome Browser (IGV) — {sample}

        {locus && (
          <div className={styles.defaultLabel}>
            Viewing Best Region
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
            setFilters((f) => ({
              ...f,
              stage: e.target.value,
            }))
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
            setFilters((f) => ({
              ...f,
              strain: e.target.value,
            }))
          }
        >
          <option value="ALL">All Strains</option>
          <option value="Okayama">Okayama</option>
          <option value="Oita">Oita</option>
        </select>

        <select
          value={filters.feeding}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              feeding: e.target.value,
            }))
          }
        >
          <option value="ALL">All Feeding</option>
          <option value="fed">fed</option>
          <option value="unfed">unfed</option>
        </select>

      </div>

      {loading && (
        <div className={styles.loading}>
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
## tracks_service.py
- backend/app/services/tracks_service.py
```sh
"""Track Service (IGV Integration - FULL UPDATED VERSION)"""

import pandas as pd

from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR,
    GFF3_FILE,
    GFF3_INDEX
)

from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


# =========================================================
# 🔷 SAMPLE LABELS
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

# =========================================================
# 🔷 STRAIN COLORS
# =========================================================
STRAIN_COLORS = {
    "Okayama": "blue",
    "Oita": "#C2185B"
}


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # =====================================================
    # 🔷 LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    # 🔷 ALL TRACKS
    # =====================================================
    def get_all_tracks(self) -> List[TrackFile]:

        df = self._load_library_info()

        tracks = []

        for _, row in df.iterrows():

            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get(
                        "Sample",
                        row["library_ID"]
                    ),
                    category=row.get(
                        "Category",
                        "unknown"
                    ),
                    default_setting=row.get(
                        "default_setting",
                        "off"
                    ),
                    files={}
                )
            )

        return tracks

    # =====================================================
    # 🔷 TRACK BY ID
    # =====================================================
    def get_track_by_id(
        self,
        library_id: str
    ) -> Optional[TrackFile]:

        df = self._load_library_info()

        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get(
                "Sample",
                row["library_ID"]
            ),
            category=row.get(
                "Category",
                "unknown"
            ),
            default_setting=row.get(
                "default_setting",
                "off"
            ),
            files={}
        )

    # =====================================================
    # 🔷 GENOME FILES
    # =====================================================
    def get_genome_files(self) -> Dict[str, str]:

        return {

            "fasta": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "fai": (
                f"/api/files/genome/{GENOME_FAI.name}"
            ),

            "phas_annotation": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "gff3": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "gff3_index": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            )
        }

    # =====================================================
    # 🔷 FIND BIGWIG FILES
    # =====================================================
    def _find_bw_files(self, sample: str):

        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for: {sample}")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            if sample in name:

                print(f"[IGV DEBUG] Candidate: {name}")

                if "plus" in name:
                    plus_file = f

                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file

    # =====================================================
    # 🔷 SINGLE SAMPLE IGV CONFIG
    # =====================================================
    def get_igv_config(
        self,
        sample: str
    ) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:

            print(f"[IGV ERROR] Missing BW files: {sample}")

            return None

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = [

            # =================================================
            # 🔷 GENOME ANNOTATION
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "gff3",

                "url": (
                    f"/api/files/annotation/{GFF3_FILE.name}"
                ),

                "indexURL": (
                    f"/api/files/annotation/{GFF3_INDEX.name}"
                ),

                "displayMode": "EXPANDED",

                "visibilityWindow": 500000,

                "height": 120,

                "color": "green"
            },

            # =================================================
            # 🔷 PHAS LOCI
            # =================================================
            {
                "name": "PHAS loci",

                "type": "annotation",

                "format": "bigBed",

                "url": (
                    f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
                ),

                "color": "orange",

                "height": 50
            },

            # =================================================
            # 🔷 PLUS STRAND
            # =================================================
            {
                "name": f"{sample} (+)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{plus_file.name}"
                ),

                "color": "red",

                "height": 60
            },

            # =================================================
            # 🔷 MINUS STRAND
            # =================================================
            {
                "name": f"{sample} (-)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{minus_file.name}"
                ),

                "color": "blue",

                "height": 60
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 ALL IGV TRACKS
    # =====================================================
    def get_all_igv_tracks(self):

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = []

        # =================================================
        # 🔷 GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "gff3",

            "url": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "indexURL": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            ),

            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "green"
        })

        # =================================================
        # 🔷 PHAS LOCI
        # =================================================
        tracks.append({

            "name": "PHAS loci",

            "type": "annotation",

            "format": "bigBed",

            "url": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "color": "orange",

            "height": 50
        })

        print("[IGV ALL] Scanning BW directory...")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            print(
                f"[IGV ALL] "
                f"{sample} -> {label} | "
                f"{meta} | {strand}"
            )

            tracks.append({

                "name": f"{label} ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "color": STRAIN_COLORS.get(
                    meta["strain"],
                    "gray"
                ),

                "height": 50
            })

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 FILE RESOLUTION
    # =====================================================
    def resolve_file_path(
        self,
        relative_path: str
    ) -> Optional[Path]:

        resolved = TRACKHUB_BASE / relative_path

        print(
            f"[FILE RESOLVE] "
            f"{relative_path} -> {resolved}"
        )

        return resolved


track_service = TrackService()
```
## tracks.py 
- backend/app/routers/tracks.py
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

---
# IGV Panel ver.
# 3. Add another switch to select  sRNAseq only, mRNAseq only or both 

## 3.1 Prepare Files
- #### Get chromosome sizes:
```sh
cut -f1,2 \
/project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.IGV.genome.fasta.fai \
> /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes

#Check output content
head GWHAMMI00000000.genome.chrom.sizes

# confirm it's TAB-separated, not spaces.
cat -vet GWHAMMI00000000.genome.chrom.sizes | head
# GWHAMMI00000001^I348261740$ -> TAB-separated 
# ^I = TAB ✔️
```
- #### Convert .bam to .bw files 


```sh
# in ma-discar@cc21dev0 
cd /work/ma-discar/PHAS/Hlongicornis/data/mRNA
conda activate bamTobw_pipeline_env
vi bam_to_bw.sh
```
- bam_to_bw.sh

```sh
#!/bin/bash

set -euo pipefail

INPUT_DIR="/work/ma-discar/PHAS/Hlongicornis/data/mRNA"
OUTPUT_DIR="/work/ma-discar/PHAS/Hlongicornis/data/mRNA/bw"
GENOME_SIZE="/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.genome.chrom.sizes"

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
  bedGraphToBigWig \
    "${OUTPUT_DIR}/${base}.plus.bedgraph" \
    "$GENOME_SIZE" \
    "${OUTPUT_DIR}/${base}.plus.bw"

  bedGraphToBigWig \
    "${OUTPUT_DIR}/${base}.minus.bedgraph" \
    "$GENOME_SIZE" \
    "${OUTPUT_DIR}/${base}.minus.bw"

  # Optional: remove temporary bedGraph files
  rm "${OUTPUT_DIR}/${base}.plus.bedgraph"
  rm "${OUTPUT_DIR}/${base}.minus.bedgraph"

done

echo "All done!"
```
- ##### Run
```sh
chmod +x bam_to_bw.sh
nohup ./bam_to_bw.sh > bam_to_bw.log 2>&1 &

# Check if running
tail -f bam_to_bw.sh
```
- #### Rename some samples
```sh
Run          Sample Name
DRR518007    Oita_Nymph_Fed
DRR518008    Oita_Nymph_Unfed
DRR518009    Okayama_Nymph_Fed
DRR518010    Okayama_Nymph_Unfed
DRR518011    Okayama_Larva_unfed
DRR518012    Okayama_Larva_fed
DRR518013    Okayama_adult_female_unfed_1
DRR518014    Okayama_adult_female_unfed_2
DRR518015    Okayama_adult_female_unfed_3
DRR518016    Okayama_adult_female_fed_1
DRR518017    Okayama_adult_female_fed_2
DRR518018    Okayama_adult_female_fed_3
DRR518019    Oita_Larva_unfed
DRR518020    Oita_Larva_fed
DRR518021    Oita_adult_male_unfed
DRR518022    Oita_adult_male_fed
DRR518023    Oita_adult_female_unfed_1
DRR518024    Oita_adult_female_unfed_2
DRR518025    Oita_adult_female_unfed_3
DRR518026    Oita_adult_female_fed_1
DRR518027    Oita_adult_female_fed_2
DRR518028    Oita_adult_female_fed_3

Source: 
# \\fsz-p21.naist.jp\okamura-lab\Canran\H.longicornis\data\mRNA
# /home/OkamuraLabsharefolder/Vina/Vina_Presentations/ProgressMeeting/Progress Report_03.29.2024_FINAL.pptx


```
- #### Copy files from /work/ma-discar/ to project disk to Volumes/Install macOS Mojave
```sh
cp -r /work/ma-discar/PHAS/Hlongicornis/data/mRNA/bw /project/okamura-lab/Vina/PHAS/Hlongicornis/data/mRNA/bw

cp -r /Volumes/okamura-lab/Vina/PHAS/Hlongicornis/data/mRNA/bw "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/mRNA_bw"

# location: "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/mRNA_bw/bw"
```

## New version of codes:
- track_service.py
- config.py
- IGVViewer.tsx 

- ### track_service.py
`backend/app/services/track_service.py`
```sh
"""Track Service (IGV Integration - FULL UPDATED VERSION)"""

import pandas as pd

from typing import List, Optional, Dict
from pathlib import Path

from ..config import (
    LIBRARY_INFO_FILE,
    TRACKHUB_BASE,
    GENOME_FASTA,
    GENOME_FAI,
    PHAS_ANNOTATION_BB,
    BW_DIR,
    MRNA_BW_DIR,
    GFF3_FILE,
    GFF3_INDEX
)

from ..utils.sample_metadata import parse_sample_metadata

from ..models.phas import TrackFile


# =========================================================
# �� SAMPLE LABELS
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

# =========================================================
# 🔷 STRAIN COLORS
# =========================================================
STRAIN_COLORS = {
    "Okayama": "blue",
    "Oita": "#C2185B"
}


class TrackService:
    """Service for track operations"""

    def __init__(self):
        self._library_info: Optional[pd.DataFrame] = None

    # =====================================================
    # 🔷 LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    # 🔷 ALL TRACKS
    # =====================================================
    def get_all_tracks(self) -> List[TrackFile]:

        df = self._load_library_info()

        tracks = []

        for _, row in df.iterrows():

            tracks.append(
                TrackFile(
                    library_id=row["library_ID"],
                    sample_name=row.get(
                        "Sample",
                        row["library_ID"]
                    ),
                    category=row.get(
                        "Category",
                        "unknown"
                    ),
                    default_setting=row.get(
                        "default_setting",
                        "off"
                    ),
                    files={}
                )
            )

        return tracks

    # =====================================================
    # 🔷 TRACK BY ID
    # =====================================================
    def get_track_by_id(
        self,
        library_id: str
    ) -> Optional[TrackFile]:

        df = self._load_library_info()

        filtered = df[df["library_ID"] == library_id]

        if filtered.empty:
            return None

        row = filtered.iloc[0]

        return TrackFile(
            library_id=row["library_ID"],
            sample_name=row.get(
                "Sample",
                row["library_ID"]
            ),
            category=row.get(
                "Category",
                "unknown"
            ),
            default_setting=row.get(
                "default_setting",
                "off"
            ),
            files={}
        )

    # =====================================================
    # 🔷 GENOME FILES
    # =====================================================
    def get_genome_files(self) -> Dict[str, str]:

        return {

            "fasta": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "fai": (
                f"/api/files/genome/{GENOME_FAI.name}"
            ),

            "phas_annotation": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "gff3": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "gff3_index": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            )
        }

    # =====================================================
    # 🔷 FIND BIGWIG FILES
    # =====================================================
    def _find_bw_files(self, sample: str):

        plus_file = None
        minus_file = None

        print(f"[IGV DEBUG] Searching BW files for: {sample}")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            if sample in name:

                print(f"[IGV DEBUG] Candidate: {name}")

                if "plus" in name:
                    plus_file = f

                elif "minus" in name:
                    minus_file = f

        return plus_file, minus_file

    # =====================================================
    # 🔷 SINGLE SAMPLE IGV CONFIG
    # =====================================================
    def get_igv_config(
        self,
        sample: str
    ) -> Optional[Dict]:

        plus_file, minus_file = self._find_bw_files(sample)

        if not plus_file or not minus_file:

            print(f"[IGV ERROR] Missing BW files: {sample}")

            return None

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = [

            # =================================================
            # 🔷 GENOME ANNOTATION
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "gff3",

                "url": (
                    f"/api/files/annotation/{GFF3_FILE.name}"
                ),

                "indexURL": (
                    f"/api/files/annotation/{GFF3_INDEX.name}"
                ),

                "displayMode": "EXPANDED",

                "visibilityWindow": 500000,

                "height": 120,

                "color": "green"
            },

            # =================================================
            # 🔷 PHAS LOCI
            # =================================================
            {
                "name": "PHAS loci",

                "type": "annotation",

                "format": "bigBed",

                "url": (
                    f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
                ),

                "color": "orange",

                "height": 50
            },

            # =================================================
            # 🔷 PLUS STRAND
            # =================================================
            {
                "name": f"{sample} (+)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{plus_file.name}"
                ),

                "color": "red",

                "height": 60
            },

            # =================================================
            # 🔷 MINUS STRAND
            # =================================================
            {
                "name": f"{sample} (-)",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/bw/{minus_file.name}"
                ),

                "color": "blue",

                "height": 60
            }
        ]

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 ALL IGV TRACKS
    # =====================================================
    def get_all_igv_tracks(self):

        genome = {

            "fastaURL": (
                f"/api/files/genome/{GENOME_FASTA.name}"
            ),

            "indexURL": (
                f"/api/files/genome/{GENOME_FAI.name}"
            )
        }

        tracks = []

        # =================================================
        # 🔷 GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "gff3",

            "url": (
                f"/api/files/annotation/{GFF3_FILE.name}"
            ),

            "indexURL": (
                f"/api/files/annotation/{GFF3_INDEX.name}"
            ),

            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "green"
        })

        # =================================================
        # 🔷 PHAS LOCI
        # =================================================
        tracks.append({

            "name": "PHAS loci",

            "type": "annotation",

            "format": "bigBed",

            "url": (
                f"/api/files/annotation/{PHAS_ANNOTATION_BB.name}"
            ),

            "color": "orange",

            "height": 50
        })

        # =================================================
        # 🔷 sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning sRNA BW directory...")

        for f in BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA",

                "color": STRAIN_COLORS.get(
                    meta["strain"],
                    "gray"
                ),

                "height": 50
            })

        # =================================================
        # 🔷 mRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning mRNA BW directory...")

        for f in MRNA_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if "plus" in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} mRNA ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": f"/api/files/mRNA_bw/bw/{name}",

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "mRNA",

                "color": "green",

                "height": 50
            })

        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    # 🔷 FILE RESOLUTION
    # =====================================================
    def resolve_file_path(
        self,
        relative_path: str
    ) -> Optional[Path]:

        resolved = TRACKHUB_BASE / relative_path

        print(
            f"[FILE RESOLVE] "
            f"{relative_path} -> {resolved}"
        )

        return resolved


track_service = TrackService()
```

- ### config.py
`bakcend/app/config.py`
```sh
"""PHASER Configuration"""

from pathlib import Path

# =========================================================
# 🔷 BASE DIRECTORY
# =========================================================
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# =========================================================
# 🔷 DATA
# =========================================================
DATA_DIR = BASE_DIR / "data"

PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# REQUIRED
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# =========================================================
# 🔷 TRACK HUB
# =========================================================
TRACKHUB_BASE = Path(
    "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"
)

# =========================================================
# 🔷 GENOME
# =========================================================
GENOME_FASTA = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta"
)

GENOME_FAI = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.IGV.genome.fasta.fai"
)

GENOME_2BIT = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.2bit"
)

GENOME_CHROMSIZES = (
    TRACKHUB_BASE
    / "genome"
    / "GWHAMMI00000000.genome.chrom.sizes"
)

# =========================================================
# 🔷 ANNOTATION
# =========================================================
ANNOTATION_DIR = TRACKHUB_BASE / "annotation"

# PHAS loci track
PHAS_ANNOTATION_BB = (
    ANNOTATION_DIR / "merged.PHAS22.bb"
)

# Genome annotation
GFF3_FILE = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz"
)

GFF3_INDEX = (
    ANNOTATION_DIR
    / "GWHAMMI00000000.sorted.gff3.gz.tbi"
)

# =========================================================
# 🔷 sRNA BIGWIG DIRECTORY
# =========================================================
BW_DIR = TRACKHUB_BASE / "bw"

# =========================================================
# 🔷 mRNA BIGWIG DIRECTORY
# =========================================================
MRNA_BW_DIR = (
    TRACKHUB_BASE
    / "mRNA_bw"
    / "bw"
)

# =========================================================
# 🔷 API
# =========================================================
API_PREFIX = "/api"
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

  const [allTracks, setAllTracks] = useState<any[]>([]);

  const [loading, setLoading] = useState(false);

  const [filters, setFilters] = useState({
    samples: [sample],
    stage: "ALL",
    strain: "ALL",
    feeding: "ALL",
    rnaType: "BOTH",
  });

  // =====================================================
  // 🔷 SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // 🔷 LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current) return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        if (igvBrowserRef.current) {

          igvBrowserRef.current.destroy?.();

          igvBrowserRef.current = null;

          isReadyRef.current = false;
        }

        const res = await fetch(
          `${API_BASE}/api/tracks/igv/all`
        );

        const config = await res.json();

        const tracksWithAbsoluteURL = config.tracks.map(
          (t: any) => ({
            ...t,

            url: API_BASE + t.url,

            indexURL: t.indexURL
              ? API_BASE + t.indexURL
              : undefined,
          })
        );

        setAllTracks(tracksWithAbsoluteURL);

        const browser = await igv.createBrowser(
          igvContainerRef.current!,
          {
            genome: {
              fastaURL:
                API_BASE +
                config.genome.fastaURL,

              indexURL:
                API_BASE +
                config.genome.indexURL,
            },

            locus: locus
              ? `${locus.chrom}:${locus.start}-${locus.end}`
              : undefined,

            tracks: [],
          }
        );

        igvBrowserRef.current = browser;

        setTimeout(() => {

          isReadyRef.current = true;

          setFilters((prev) => ({ ...prev }));

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
  // 🔷 APPLY TRACKS
  // =====================================================
  useEffect(() => {

    if (
      !igvBrowserRef.current ||
      !isReadyRef.current ||
      allTracks.length === 0
    ) {
      return;
    }

    const browser = igvBrowserRef.current;

    const applyTracks = async () => {

      try {

        await browser.removeAllTracks();

        const annotationTracks = allTracks.filter(
          (t) =>
            t.format === "gff3" ||
            t.format === "bigBed"
        );

        let signalTracks = allTracks.filter(
          (t) =>
            t.format !== "gff3" &&
            t.format !== "bigBed"
        );

        signalTracks = signalTracks.filter((t) => {

          const rnaMatch =
            filters.rnaType === "BOTH" ||
            t.rnaType === filters.rnaType;

          return (
            filters.samples.includes(t.sample) &&
            (filters.stage === "ALL" ||
              t.stage === filters.stage) &&
            (filters.strain === "ALL" ||
              t.strain === filters.strain) &&
            (filters.feeding === "ALL" ||
              t.feeding === filters.feeding) &&
            rnaMatch
          );
        });

        signalTracks.sort((a, b) =>
          a.name.localeCompare(
            b.name,
            undefined,
            {
              numeric: true,
              sensitivity: "base",
            }
          )
        );

        const finalTracks = [
          ...annotationTracks,
          ...signalTracks,
        ];

        for (const track of finalTracks) {

          await browser.loadTrack(track);
        }

      } catch (err) {

        console.error(
          "Track loading error:",
          err
        );
      }
    };

    applyTracks();

  }, [filters, allTracks]);

  // =====================================================
  // 🔷 LOCUS UPDATE
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
  // 🔷 TOGGLE SAMPLES
  // =====================================================
  const toggleAllSamples = () => {

    const allSamples = Array.from(
      new Set(
        allTracks
          .map((t) => t.sample)
          .filter(Boolean)
      )
    );

    if (filters.samples.length === 1) {

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
        Genome Browser (IGV) — {sample}

        {locus && (
          <div className={styles.defaultLabel}>
            Viewing Best Region
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
            setFilters((f) => ({
              ...f,
              stage: e.target.value,
            }))
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
            setFilters((f) => ({
              ...f,
              strain: e.target.value,
            }))
          }
        >
          <option value="ALL">All Strains</option>
          <option value="Okayama">Okayama</option>
          <option value="Oita">Oita</option>
        </select>

        <select
          value={filters.feeding}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              feeding: e.target.value,
            }))
          }
        >
          <option value="ALL">All Feeding</option>
          <option value="fed">fed</option>
          <option value="unfed">unfed</option>
        </select>

        <select
          value={filters.rnaType}
          onChange={(e) =>
            setFilters((f) => ({
              ...f,
              rnaType: e.target.value,
            }))
          }
        >
          <option value="BOTH">Both</option>
          <option value="sRNA">sRNAseq only</option>
          <option value="mRNA">mRNAseq only</option>
        </select>

      </div>

      {loading && (
        <div className={styles.loading}>
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
#### It worked. Toggles for mRNA, sRNA and both can be seen in the interface.  
#### Improve: 
- Show best sample stage for mRNAseq same as sRNAseq
- Modify `IGVVIewer.tsx
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
      rnaType: "BOTH",
    });

  // =====================================================
  // 🔷 SAMPLE → MATCHING MRNA LABELS
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
      "Okayama_Larva_Fed",
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
      "OkayamaAFed",
      "Okayama_Adult_female_fed",
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
  // 🔷 SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // 🔷 LOAD IGV
  // =====================================================
  useEffect(() => {

    if (!igvContainerRef.current)
      return;

    const loadIGV = async () => {

      setLoading(true);

      try {

        // ===============================================
        // 🔷 CLEANUP OLD BROWSER
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
        // 🔷 FETCH TRACK CONFIG
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
        // 🔷 CREATE BROWSER
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
  // 🔷 APPLY TRACKS
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
          // 🔷 ANNOTATION TRACKS
          // =============================================
          const annotationTracks =
            allTracks.filter(
              (t) =>
                t.format ===
                  "gff3" ||
                t.format ===
                  "bigBed"
            );

          // =============================================
          // 🔷 SIGNAL TRACKS
          // =============================================
          let signalTracks =
            allTracks.filter(
              (t) =>
                t.format !==
                  "gff3" &&
                t.format !==
                  "bigBed"
            );

          signalTracks =
            signalTracks.filter(
              (t) => {

                // =======================================
                // 🔷 RNA FILTER
                // =======================================
                const rnaMatch =
                  filters.rnaType ===
                    "BOTH" ||
                  t.rnaType ===
                    filters.rnaType;

                if (!rnaMatch)
                  return false;

                // =======================================
                // 🔷 SHOW BEST SAMPLE MODE
                // =======================================
                if (
                  filters.samples
                    .length === 1
                ) {

                  // ===================================
                  // 🔷 SRNA
                  // ===================================
                  if (
                    t.rnaType ===
                    "sRNA"
                  ) {

                    return (
                      t.sample ===
                      sample
                    );
                  }

                  // ===================================
                  // 🔷 MRNA
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
                        // 🔷 EXACT MATCH LOGIC
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
                // 🔷 SHOW ALL SAMPLES MODE
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
          // 🔷 SORT TRACKS
          // =============================================
          signalTracks.sort(
            (a, b) =>
              a.name.localeCompare(
                b.name,
                undefined,
                {
                  numeric: true,
                  sensitivity:
                    "base",
                }
              )
          );

          const finalTracks = [
            ...annotationTracks,
            ...signalTracks,
          ];

          // =============================================
          // 🔷 LOAD TRACKS
          // =============================================
          for (const track of finalTracks) {

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
  // 🔷 LOCUS UPDATE
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
  // 🔷 TOGGLE SAMPLES
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
          <option value="BOTH">
            Both
          </option>

          <option value="sRNA">
            sRNAseq only
          </option>

          <option value="mRNA">
            mRNAseq only
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
#### Achieved! 
#### Issue: 
- mRNA tracks do not appear when other toggles (Stages, Strains, Feeding) are clicked. 
---
# 4. Import Excel table Phas_manualAnnotation_ver2.xlsx in the interface
#### Flow:
```sh
1. Open PHAS locus
2. Evaluate confidence
3. Add notes/labels
4. Save

System Automatically:
- updates metadata
- updates table
- updates Excel file
```
#### Modify:
- ConfidencePanel.tsx
- backend/main.py
#### ADD
- backend/app/routers/metadata.py
- backend/app/services/metadata_service.py
- frontend/src/services/metadataService.ts

#### Store Excel here:
- backend/Data/phas_metadata.xlsx
```sh
cp /Volumes/okamura-lab/Vina/PHAS/Phas_manualAnnotation_template_ver2.xlsx "/Volumes/Install macOS Mojave/Vina/PHASER/backend/Data"
```

#### Previous code:
###### `ConfidencePanel.tsx`
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

    // Priority: evaluation prop
    if (evaluation) {
      setScores(
        evaluation.scores || defaultScores
      );

      setNotes(evaluation.notes || "");
      setComments(evaluation.comments || "");

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
      setComments(parsed.comments || "");
    } else {
      setScores(defaultScores);
      setNotes("");
      setComments("");
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
      comments,
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

          {/* NOTES + COMMENTS */}
          <div className={styles.notesSection}>

            {/* NOTES */}
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

            {/* REVIEWER COMMENTS */}
            <div className={styles.notesLabel}>
              Reviewer Comments
            </div>

            <textarea
              className={styles.notesInput}
              value={comments}
              onChange={(e) =>
                setComments(
                  e.target.value
                )
              }
              placeholder="Write reviewer comments..."
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
###### `backend/main.py`
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

#### main.py
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
# Part 1: Excel Synchronization
```sh
ConfidencePanel
      ↓
Save Analysis
      ↓
Excel file updates
```
## Step 1: ADD NEW ROUTER FILE
```sh
backend/app/routers/metadata.py

vi metadata.py
```
- ## metadata.py

```sh
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
import pandas as pd
from pathlib import Path

router = APIRouter(prefix="/metadata", tags=["metadata"])

# Excel file path
EXCEL_PATH = Path("Data/phas_metadata.xlsx")


class MetadataPayload(BaseModel):
    locusId: str
    scores: dict
    total: int
    level: str
    interpretation: str
    notes: str
    comments: str
    timestamp: str


@router.post("/save")
async def save_metadata(payload: MetadataPayload):

    # Create empty dataframe if file does not exist
    if EXCEL_PATH.exists():
        df = pd.read_excel(EXCEL_PATH)
    else:
        df = pd.DataFrame(columns=[
            "PHASID",
            "P-value(statistical support from sRNAminer)",
            "5' Distribution (Best Sample)",
            "5' Distribution (Cross-Sample)",
            "Register Bias (Best sample)",
            "Register Bias (Cross-sample)",
            "IGV (Best Sample)",
            "IGV (Cross-sample)",
            "Manual Score",
            "Label",
            "Notes",
            "Reviewer Comments",
            "Timestamp"
        ])

    # Build updated row
    row = {
        "PHASID": payload.locusId,
        "P-value(statistical support from sRNAminer)": payload.scores.get("pValue", 0),
        "5' Distribution (Best Sample)": payload.scores.get("distBest", 0),
        "5' Distribution (Cross-Sample)": payload.scores.get("distCross", 0),
        "Register Bias (Best sample)": payload.scores.get("regBest", 0),
        "Register Bias (Cross-sample)": payload.scores.get("regCross", 0),
        "IGV (Best Sample)": payload.scores.get("igvBest", 0),
        "IGV (Cross-sample)": payload.scores.get("igvCross", 0),
        "Manual Score": payload.total,
        "Label": payload.level,
        "Notes": payload.notes,
        "Reviewer Comments": payload.comments,
        "Timestamp": payload.timestamp
    }

    # Update existing row if PHASID already exists
    if payload.locusId in df["PHASID"].values:
        df.loc[df["PHASID"] == payload.locusId, row.keys()] = row.values()
    else:
        df = pd.concat([df, pd.DataFrame([row])], ignore_index=True)

    # Save Excel
    df.to_excel(EXCEL_PATH, index=False)

    return {
        "status": "success",
        "message": "Metadata saved to Excel"
    }
```
## Step 2: EDIT main.py

Add this import:
```sh 
from app.routers import phas, tracks, files, metadata
```
Add this line:
```sh
app.include_router(metadata.router, prefix=API_PREFIX)
```
#### main.py
```sh
"""
PHAS Backend Application
PHAS Evaluation Resource Platform
"""

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import API_PREFIX
from app.routers import phas, tracks, files, metadata

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
app.include_router(metadata.router, prefix=API_PREFIX)

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

## Step 3: ADD FRONTEND SERVICE 
`frontend/src/services/metadataService.ts`
```sh 
cd frontend/src/services
vi metadataService.ts
```
- ### metadataService.ts
```sh
export async function saveMetadata(payload: any) {

  const response = await fetch(
    "http://localhost:8001/api/metadata/save",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    }
  );

  if (!response.ok) {
    throw new Error("Failed to save metadata");
  }

  return response.json();
}
```

## Step 4: EDIT ConfidencePanel.tsx
ADD import:
```sh
import { saveMetadata } from "../../services/metadataService";
```
Then INSIDE handleSave(), add:
```sh
saveMetadata(payload)
  .then(() => {
    console.log("Excel metadata saved");
  })
  .catch((err) => {
    console.error("Excel save failed", err);
  });
```
- ### ConfidencePanel.tsx
```sh
import { useEffect, useState } from "react";
import styles from "./ConfidencePanel.module.css";

import {
  computeTotal,
  classifyScore,
  getInterpretation,
  Scores,
} from "../../utils/confidenceUtils";

import { saveMetadata } from "../../services/metadataService";

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

    // Priority: evaluation prop
    if (evaluation) {
      setScores(
        evaluation.scores || defaultScores
      );

      setNotes(evaluation.notes || "");
      setComments(evaluation.comments || "");

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
      setComments(parsed.comments || "");
    } else {
      setScores(defaultScores);
      setNotes("");
      setComments("");
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

  const handleSave = async () => {
    if (!locus?.phas_id) return;

    const payload = {
      locusId: locus.phas_id,

      // Optional if available in locus
      pvalue:
        locus.pvalue ||
        locus.p_value ||
        "",

      scores,

      total,

      level,

      interpretation,

      notes,

      comments,

      timestamp:
        new Date().toISOString(),
    };

    /* =============================
       LOCAL STORAGE SAVE
    ============================= */

    localStorage.setItem(
      `phas_analysis_${locus.phas_id}`,
      JSON.stringify(payload)
    );

    /* =============================
       EXCEL BACKEND SAVE
    ============================= */

    try {
      await saveMetadata(payload);

      console.log(
        "Excel metadata saved"
      );
    } catch (err) {
      console.error(
        "Excel save failed",
        err
      );
    }

    /* =============================
       EXISTING CALLBACK
    ============================= */

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

          {/* NOTES + COMMENTS */}
          <div className={styles.notesSection}>

            {/* NOTES */}
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

            {/* REVIEWER COMMENTS */}
            <div className={styles.notesLabel}>
              Reviewer Comments
            </div>

            <textarea
              className={styles.notesInput}
              value={comments}
              onChange={(e) =>
                setComments(
                  e.target.value
                )
              }
              placeholder="Write reviewer comments..."
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
# Part 2 Metadata Table Viewer

```sh
Click "Metadata Table"
      ↓
Open browser table view
      ↓
See synchronized Excel contents
```
## Step 1: Edit `backend/app/routers/metadata.py`
Add:
```sh
GET /metadata
```
- metadata.py
```sh
from fastapi import APIRouter
from pydantic import BaseModel
import pandas as pd
from pathlib import Path

router = APIRouter(
    prefix="/metadata",
    tags=["metadata"]
)

EXCEL_PATH = Path("Data/phas_metadata.xlsx")


class MetadataPayload(BaseModel):
    locusId: str
    pvalue: str | float | int | None = ""

    scores: dict

    total: int

    level: str

    interpretation: str

    notes: str

    comments: str

    timestamp: str


# =========================================
# GET ALL METADATA
# =========================================

@router.get("")
async def get_metadata():

    if not EXCEL_PATH.exists():
        return {
            "records": []
        }

    df = pd.read_excel(EXCEL_PATH)

    df = df.fillna("")

    return {
        "records": df.to_dict(
            orient="records"
        )
    }


# =========================================
# SAVE METADATA
# =========================================

@router.post("/save")
async def save_metadata(
    payload: MetadataPayload
):

    if EXCEL_PATH.exists():

        df = pd.read_excel(
            EXCEL_PATH
        )

    else:

        df = pd.DataFrame(columns=[

            "PHASID",

            "Pvalue",

            "P-value(statistical support from sRNAminer)",

            "5' Distribution (Best Sample)",

            "5' Distribution (Cross-Sample)",

            "Register Bias (Best sample)",

            "Register Bias (Cross-sample)",

            "IGV (Best Sample)",

            "IGV (Cross-sample)",

            "Manual Score",

            "Label",

            "Notes",

            "Reviewer Comments",

            "Timestamp"
        ])

    row = {

        "PHASID":
            payload.locusId,

        "Pvalue":
            payload.pvalue,

        "P-value(statistical support from sRNAminer)":
            payload.scores.get(
                "pValue", 0
            ),

        "5' Distribution (Best Sample)":
            payload.scores.get(
                "distBest", 0
            ),

        "5' Distribution (Cross-Sample)":
            payload.scores.get(
                "distCross", 0
            ),

        "Register Bias (Best sample)":
            payload.scores.get(
                "regBest", 0
            ),

        "Register Bias (Cross-sample)":
            payload.scores.get(
                "regCross", 0
            ),

        "IGV (Best Sample)":
            payload.scores.get(
                "igvBest", 0
            ),

        "IGV (Cross-sample)":
            payload.scores.get(
                "igvCross", 0
            ),

        "Manual Score":
            payload.total,

        "Label":
            payload.level,

        "Notes":
            payload.notes,

        "Reviewer Comments":
            payload.comments,

        "Timestamp":
            payload.timestamp
    }

    # UPDATE EXISTING ROW
    if payload.locusId in df["PHASID"].values:

        df.loc[
            df["PHASID"] == payload.locusId,
            row.keys()
        ] = row.values()

    # APPEND NEW ROW
    else:

        df = pd.concat(
            [
                df,
                pd.DataFrame([row])
            ],
            ignore_index=True
        )

    df.to_excel(
        EXCEL_PATH,
        index=False
    )

    return {
        "status": "success"
    }
```
Purpose: return Excel contents to frontend
## Step 2: Add `MetadataTable.tsx`
Purpose: display Metadata table
```sh
mkdir MetaTable
vi MetaTable.tsx
vi MetaTable.module.css
```
- #### MetaTable.tsx
```sh
import {
  useEffect,
  useState
} from "react";

import {
  getMetadata
} from "../../services/metadataService";

import "./MetadataTable.css";

export default function MetadataTable() {

  const [records, setRecords] =
    useState<any[]>([]);

  const [loading, setLoading] =
    useState(true);

  useEffect(() => {

    getMetadata()
      .then((data) => {
        setRecords(
          data.records || []
        );
      })
      .finally(() => {
        setLoading(false);
      });

  }, []);

  if (loading) {
    return (
      <div className="metadataContainer">
        Loading metadata...
      </div>
    );
  }

  if (!records.length) {
    return (
      <div className="metadataContainer">
        No metadata found.
      </div>
    );
  }

  const columns =
    Object.keys(records[0]);

  return (
    <div className="metadataContainer">

      <h2>
        Metadata Table
      </h2>

      <div className="tableWrapper">

        <table className="metadataTable">

          <thead>
            <tr>
              {columns.map((col) => (
                <th key={col}>
                  {col}
                </th>
              ))}
            </tr>
          </thead>

          <tbody>

            {records.map(
              (row, idx) => (

              <tr key={idx}>

                {columns.map((col) => (

                  <td key={col}>
                    {
                      String(
                        row[col]
                      )
                    }
                  </td>

                ))}

              </tr>

            ))}

          </tbody>

        </table>

      </div>

    </div>
  );
}
```
- #### MetaTable.module.css
```sh
.metadataContainer {
  margin-top: 20px;
  padding: 20px;
  background: white;
  border-radius: 10px;
}

.tableWrapper {
  overflow-x: auto;
}

.metadataTable {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.metadataTable th,
.metadataTable td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.metadataTable th {
  background: #f4f4f4;
  font-weight: 600;
}

.metadataTable tr:nth-child(even) {
  background: #fafafa;
}
```
## Step 3: Edit `frontend/src/services/metadataService.ts`
Add:
```sh
getMetadata()
```
Purpose: fetch Excel rows
#### metadataService.ts
```sh
const API_BASE =
  "http://localhost:8001/api/metadata";


// ========================================
// SAVE METADATA
// ========================================

export async function saveMetadata(
  payload: any
) {

  const response = await fetch(
    `${API_BASE}/save`,
    {
      method: "POST",

      headers: {
        "Content-Type":
          "application/json",
      },

      body: JSON.stringify(payload),
    }
  );

  if (!response.ok) {
    throw new Error(
      "Failed to save metadata"
    );
  }

  return response.json();
}


// ========================================
// GET METADATA
// ========================================

export async function getMetadata() {

  const response = await fetch(
    API_BASE
  );

  if (!response.ok) {
    throw new Error(
      "Failed to fetch metadata"
    );
  }

  return response.json();
}
```
## Step 4: Edit: `frontend/src/App.tsx`
Purpose: add Metadata Table button/page/panel
#### Previous Code:
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
  // HYDRATE LOCAL STORAGE
  // =============================
  useEffect(() => {
    const loaded: Record<string, any> = {};

    Object.keys(localStorage).forEach((key) => {
      if (!key.startsWith("phas_analysis_"))
        return;

      try {
        const raw =
          localStorage.getItem(key);

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
- ## Current Code: `App.tsx`
```sh
import { useState, useEffect, useMemo } from "react";

import { Header } from "./components/Header/Header";
import { SearchBox } from "./components/SearchBox/SearchBox";
import { PHASTable } from "./components/PHASTable/PHASTable";

import PhasingPatternPanel from "./components/PhasingPatternPanel/PhasingPatternPanel";
import IGVViewer from "./components/IGVViewer/IGVViewer";

import ConfidencePanel from "./components/ConfidencePanel/ConfidencePanel";

import MetadataTable from "./components/MetadataTable/MetadataTable";

import { fetchPHASLoci } from "./services/phasService";

import type { PHASLocus } from "./types/phas";

import { getStageFromSample } from "./utils/sampleMap";

import "./App.css";

function parseRegion(region: string) {

  const [chrom, coords] =
    region.split(":");

  const [start, end] =
    coords.split("-").map(Number);

  return {
    chrom,
    start,
    end
  };
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

  const [showMetadata, setShowMetadata] =
    useState(false);

  // ========================================
  // LOAD PHAS DATA
  // ========================================

  useEffect(() => {

    setLoading(true);

    fetchPHASLoci()
      .then((data) => {

        setPhasRecords(
          data.records
        );

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
      .finally(() => {
        setLoading(false);
      });

  }, []);

  // ========================================
  // HYDRATE LOCAL STORAGE
  // ========================================

  useEffect(() => {

    const loaded:
      Record<string, any> = {};

    Object.keys(localStorage)
      .forEach((key) => {

      if (
        !key.startsWith(
          "phas_analysis_"
        )
      ) return;

      try {

        const raw =
          localStorage.getItem(
            key
          );

        if (!raw) return;

        const data =
          JSON.parse(raw);

        if (!data?.locusId)
          return;

        loaded[data.locusId] =
          data;

      } catch (e) {}

    });

    setEvaluations(loaded);

    setHydrated(true);

  }, []);

  // ========================================
  // SEARCH
  // ========================================

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

      phasRecords.filter(
        (r) =>

        r.phas_id
          .toLowerCase()
          .includes(q)

      )

    );

  }, [
    searchQuery,
    phasRecords
  ]);

  // ========================================
  // LOAD STAGES
  // ========================================

  useEffect(() => {

    if (!selectedPhas)
      return;

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

  // ========================================
  // SELECT HANDLER
  // ========================================

  const handleSelectPhasAndStage = (
    phas: PHASLocus,
    stage: string | null = null,
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

      if (match)
        derivedSample =
          match[0];

      const bestStage =
        getStageFromSample(
          phas.best_sample
        );

      if (bestStage)
        derivedStage =
          bestStage;
    }

    setSelectedSample(
      derivedSample || null
    );

    setActiveStage(
      derivedStage || "ALL"
    );

    setView("all");
  };

  // ========================================
  // ENRICHED RECORDS
  // ========================================

  const enrichedRecords =
    useMemo(() => {

    return filteredRecords.map(
      (r) => ({

      ...r,

      __evaluation:
        evaluations[
          r.phas_id
        ],

    }));

  }, [
    filteredRecords,
    evaluations
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

        <button
          style={{
            marginBottom: "15px",
            padding: "10px 15px",
            cursor: "pointer"
          }}
          onClick={() =>
            setShowMetadata(
              !showMetadata
            )
          }
        >
          {
            showMetadata
              ? "Hide Metadata Table"
              : "Show Metadata Table"
          }
        </button>

        {showMetadata && (
          <MetadataTable />
        )}

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
                locus={
                  selectedPhas
                }

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

## Error: 
- [Error] Failed to load resource: Could not connect to the server. (metadata, line 0) http://localhost:8001/api/metadata

## Solution 1: 
- In `metadataService.ts`