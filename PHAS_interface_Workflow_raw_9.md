# Troubleshoot: 
> Ensure that Notes are visible accross devices and can be commented on and saved by everyone.
```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

### Modify the codes:
- App.tsx       
- ConfidencePanel.tsx
- PHASTable.tsx

### Previous Codes:
#### App.tsx
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

              {/* =========================
                  METADATA BUTTON
              ========================== */}

              <button
                style={{
                  marginTop: "15px",
                  padding: "10px 15px",
                  cursor: "pointer",
                  width: "100%"
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

              {/* =========================
                  METADATA TABLE
              ========================== */}

              {showMetadata && (
                <MetadataTable />
              )}

            </div>

          </div>

        )}

      </main>

    </div>
  );
}
```

### ConfidencePanel.tsx
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

### PHASTable.tsx

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
### metadataService.tsx
```sh
const API_BASE =
  "http://163.221.246.151:8001/api/metadata";


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

- #### metadata.py
/backend/app/routers/metadata.py
```sh
from fastapi import APIRouter
from pydantic import BaseModel
import pandas as pd
from pathlib import Path

router = APIRouter(
    prefix="/metadata",
    tags=["metadata"]
)

EXCEL_PATH = Path("backend/Data/Phas_manualAnnotation_template_ver2.xlsx")


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

    df = pd.read_excel(
        EXCEL_PATH,
        header=1
    )

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

## Issue:
- Notes and comments in the main table interface are not visible across devices

## Solution 1:
- remove `header=1` in `backend/app/routers/metadata.py` file then check:
```sh
http://163.221.246.151:8001/api/metadata
```

- Result still says: `{"records":[]}`
(which means the issues were not solved and the issue is not the GET function. )

## Solution 2:
- Since the error is not the the GET function, verify whether the backend is reading the same Excel file that the save function is writing to.

----


# Soultion 3

- Trace the original previous files before the new Metadatable was added to the interface. Purpose: to restart clean in modufying the codes. Having unfinished metadatable cpmplicate solving the problem. 

##### Original Files (Previous Codes)- `PHAS_interface_Workflow_raw_8.md (or ...raw_7.md):

- ConfidencePanel.tsx
- backend/app/main.py
- App.tsx

### Original Files (Previous Codes)
(before the new Metadatable was added to the interface)
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
- ### /backend/main.py
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
- ### App.tsx
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

- ### backend/app/config.py
```sh 
"""PHASER Configuration"""

from pathlib import Path

# =========================================================
# BASE DIRECTORY
# =========================================================
BASE_DIR = Path("/Volumes/Install macOS Mojave/Vina/PHASER")

# =========================================================
#  DATA
# =========================================================
DATA_DIR = BASE_DIR / "data"

PHAS_DATA_FILE = DATA_DIR / "phas_loci.tsv"

# REQUIRED
LIBRARY_INFO_FILE = DATA_DIR / "library_info.tsv"

# =========================================================
#  TRACK HUB
# =========================================================
TRACKHUB_BASE = Path(
    "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks"
)

# =========================================================
#  GENOME
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
#  ANNOTATION
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
#  sRNA BIGWIG DIRECTORY
# =========================================================
BW_DIR = TRACKHUB_BASE / "bw"

# =========================================================
#  mRNA BIGWIG DIRECTORY
# =========================================================
MRNA_BW_DIR = (
    TRACKHUB_BASE
    / "mRNA_bw"
    / "bw"
)

# =========================================================
# API
# =========================================================
API_PREFIX = "/api"
```

- ### backend/app/routers/phas.py

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


#  GET ALL
@router.get("", response_model=PHASListResponse)
async def get_all_phas():
    """Get all PHAS loci"""
    records = phas_service.get_all()
    return PHASListResponse(total=len(records), records=records)


#  SEARCH
@router.get("/search", response_model=PHASSearchResponse)
async def search_phas(q: str = Query(..., min_length=1)):
    """Search PHAS loci by ID or annotation"""
    records = phas_service.search(q)
    return PHASSearchResponse(query=q, total=len(records), records=records)


#  GET BY ID
@router.get("/{phas_id}", response_model=PHASRecord)
async def get_phas_by_id(phas_id: str):
    """Get PHAS locus by ID"""
    record = phas_service.get_by_id(phas_id)
    if not record:
        raise HTTPException(status_code=404, detail=f"PHAS {phas_id} not found")
    return record


#  DISPLAY REGION (FIXED HERE)
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
- #### PHASTable.tsx
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
# New updated Code

- ## backend/main.py
```sh
"""
PHAS Backend Application
PHAS Evaluation Resource Platform
"""

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import API_PREFIX

# Added evaluations router
from app.routers import (
    phas,
    tracks,
    files,
    evaluations
)

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
    expose_headers=[
        "Content-Range",
        "Accept-Ranges",
        "Content-Length"
    ]
)

# =========================================================
# ROUTERS
# =========================================================

app.include_router(
    phas.router,
    prefix=API_PREFIX
)

app.include_router(
    tracks.router,
    prefix=API_PREFIX
)

app.include_router(
    files.router,
    prefix=API_PREFIX
)

# NEW: Shared evaluations router
app.include_router(
    evaluations.router,
    prefix=API_PREFIX
)

# Optional BLAT
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
    return {
        "status": "healthy"
    }


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8001
    )
```
- ## backend/app/routers/evaluations.py
- new file created: `evaluations.py`

```sh
"""
Shared PHAS Evaluations Router

Stores confidence evaluations, notes, and reviewer
comments in a shared JSON file so that all users
can see the same annotations across devices.
"""

from pathlib import Path
import json

from fastapi import APIRouter

router = APIRouter(
    prefix="/evaluations",
    tags=["Evaluations"]
)

# =========================================================
# DATA FILE
# =========================================================

BASE_DIR = Path(
    "/Volumes/Install macOS Mojave/Vina/PHASER"
)

EVALUATION_FILE = (
    BASE_DIR
    / "data"
    / "evaluations.json"
)


# =========================================================
# HELPERS
# =========================================================

def load_data():
    """
    Load all saved evaluations.
    """

    if not EVALUATION_FILE.exists():
        return {}

    try:
        with open(
            EVALUATION_FILE,
            "r",
            encoding="utf-8"
        ) as f:
            return json.load(f)

    except Exception:
        return {}


def save_data(data):
    """
    Save evaluations to disk.
    """

    EVALUATION_FILE.parent.mkdir(
        parents=True,
        exist_ok=True
    )

    with open(
        EVALUATION_FILE,
        "w",
        encoding="utf-8"
    ) as f:
        json.dump(
            data,
            f,
            indent=2,
            ensure_ascii=False
        )


# =========================================================
# ROUTES
# =========================================================

@router.get("")
async def get_all_evaluations():
    """
    Return all evaluations.
    """

    return load_data()


@router.get("/{phas_id}")
async def get_evaluation(
    phas_id: str
):
    """
    Return evaluation for one PHAS locus.
    """

    data = load_data()

    return data.get(
        phas_id,
        {}
    )


@router.post("/{phas_id}")
async def save_evaluation(
    phas_id: str,
    payload: dict
):
    """
    Save or update evaluation.
    """

    data = load_data()

    data[phas_id] = payload

    save_data(data)

    return {
        "success": True,
        "phas_id": phas_id,
        "message": "Evaluation saved"
    }

```
- ### backend/app/routers/evaluations.py

```sh
"""
Shared PHAS Evaluations Router

Stores confidence evaluations, notes, and reviewer
comments in a shared JSON file so that all users
can see the same annotations across devices.
"""

from pathlib import Path
import json

from fastapi import APIRouter

router = APIRouter(
    prefix="/evaluations",
    tags=["Evaluations"]
)

# =========================================================
# DATA FILE
# =========================================================

BASE_DIR = Path(
    "/Volumes/Install macOS Mojave/Vina/PHASER"
)

EVALUATION_FILE = (
    BASE_DIR
    / "data"
    / "evaluations.json"
)


# =========================================================
# HELPERS
# =========================================================

def load_data():
    """
    Load all saved evaluations.
    """

    if not EVALUATION_FILE.exists():
        return {}

    try:
        with open(
            EVALUATION_FILE,
            "r",
            encoding="utf-8"
        ) as f:
            return json.load(f)

    except Exception:
        return {}


def save_data(data):
    """
    Save evaluations to disk.
    """

    EVALUATION_FILE.parent.mkdir(
        parents=True,
        exist_ok=True
    )

    with open(
        EVALUATION_FILE,
        "w",
        encoding="utf-8"
    ) as f:
        json.dump(
            data,
            f,
            indent=2,
            ensure_ascii=False
        )


# =========================================================
# ROUTES
# =========================================================

@router.get("")
async def get_all_evaluations():
    """
    Return all evaluations.
    """

    return load_data()


@router.get("/{phas_id}")
async def get_evaluation(
    phas_id: str
):
    """
    Return evaluation for one PHAS locus.
    """

    data = load_data()

    return data.get(
        phas_id,
        {}
    )


@router.post("/{phas_id}")
async def save_evaluation(
    phas_id: str,
    payload: dict
):
    """
    Save or update evaluation.
    """

    data = load_data()

    data[phas_id] = payload

    save_data(data)

    return {
        "success": True,
        "phas_id": phas_id,
        "message": "Evaluation saved"
    }
```
- ### PHASER/data/evaluations.json
- create new file `evaluations.json`

```sh
{}
```

- #### Restart to check. `bash start.sh`
Check http://163.221.246.151:8001/api/evaluations. If the result is `{}`, that means the route /api/evaluations is working
- #### Modify `ConfidencePanel.tsx
What to change:
- Removed localStorage
- Saves to /api/evaluations/{phas_id}
- Loads notes from the evaluation prop supplied by App.tsx
- Works with your shared evaluations.json
- Keeps all existing UI and scoring logic

- ## ConfidencePanel.tsx
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
          `/api/evaluations/${locus.phas_id}`,
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
- #### Update App.tsx
- major change is replacing the localStorage hydration section with a backend fetch from: `/api/evaluations`
## App.tsx
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
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
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
- ### Issues:
 
    1. I tried to save an analysis and it prompted me to "Failed to save analysis". 
    Reasons: the request never reaches FastAPI or the frontend is calling the wrong URL. 
    From Console:
    `Failed to load resource: the server responded with a status of 404 (Not Found) (evaluations)`
    Meaning, my frontend is trying to access `/api/evaluations` and `/api/evaluations/PHAS22-1` not on the on the Vite frontend server (port 5174), not on the FastAPI server (port 8001).
    2. My previous notes or previous saved analyses were not visible anymore (since local storage was replaced)

- ### Solution 1:
- Fix:
#### In App.tsx,
replace with:
```sh 
fetch(
  "http://163.221.246.151:8001/api/evaluations"
)
```
#### In ConfidencePanel.tsx, 
replace with:
```sh
await fetch(
  `http://163.221.246.151:8001/api/evaluations/${locus.phas_id}`,
```
### Updated codes:
# App.tsx
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
  // LOAD SHARED EVALUATIONS
  // =============================
  useEffect(() => {
    fetch("http://163.221.246.151:8001/api/evaluations")
      .then((res) => {
        if (!res.ok) {
          throw new Error(
            "Failed to load evaluations"
          );
        }

        return res.json();
      })
      .then((data) => {
        setEvaluations(data || {});
        setHydrated(true);
      })
      .catch((err) => {
        console.error(
          "Evaluation load error:",
          err
        );

        setEvaluations({});
        setHydrated(true);
      });
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
# ConfidencePanel.tsx
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
#### Problem solved. 
(Everyone can save the notes. When one refreshes the interface, it will not disappear. Notes can be visible by everyone using different devices)

## Solution 2
- Recover previous notes or previous saved analyses that get lost. 

### 1. First Check old saved analysis in the console if it still exists:
```sh
const evaluations = {};

Object.keys(localStorage).forEach((key) => {
  if (!key.startsWith("phas_analysis_")) return;

  try {
    const value = JSON.parse(localStorage.getItem(key));
    if (value?.locusId) {
      evaluations[value.locusId] = value;
    }
  } catch (e) {
    console.error("Error reading", key);
  }
});

console.log(JSON.stringify(evaluations, null, 2));
```
- ##### Result from checking
Sample Part of the content:
```sh
"PHAS22-598": {
    "locusId": "PHAS22-598",
    "scores": {
      "distBest": 0,
      "distCross": 0,
      "regBest": 0,
      "regCross": 0,
      "igvBest": 0,
      "igvCross": 0,
      "pValue": 1
    },
    "total": 1,
    "level": "Low",
    "interpretation": "Weak, inconsistent, or likely non-PHAS",
    "notes": "plus/minus reads signals in register distribution are flat; weak periodicity; weak 22nt spacing mapping pattern",
    "comments": "",
    "timestamp": "2026-05-15T10:07:13.234Z"
  },
```
### 2. Export the full JSON
#### 2a. copy the entire JSON to the clipboard (Chrome/Edge/Safari usually support copy() in DevTools).
- In the browser console, run
```sh
const evaluations = {};

Object.keys(localStorage).forEach((key) => {
  if (!key.startsWith("phas_analysis_")) return;

  try {
    const value = JSON.parse(localStorage.getItem(key));
    if (value?.locusId) {
      evaluations[value.locusId] = value;
    }
  } catch (e) {}
});

copy(JSON.stringify(evaluations, null, 2));
```

#### 2b. Save the old saved evaluation in a text editor
```sh
old_evaluations.json
``` 
### 3. Merge into backend
- Open `evaluations.json` from PHASER/data/evaluations.json, then replace its content `{}` with the contents of `old_evaluations.json`
```sh
cd PHASER/data
vi evaluations.json

# copy the content from `old_evaluations.json`
```
### 4. Restart (bash start.sh)
### 5. Verify http://163.221.246.151:8001/api/evaluations
- We should now see hundreds of entries in the table.

### Issues solved!
> Notes can now be visible and can be commented and saved by everyone

---
