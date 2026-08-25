```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/PHASER"
```

# Update Confidence Panel
07.01.2026
- Add link for a Scoring Manual 

## Copy PHASER_Manual_Scoring_Guide.pdf to PHASER public

```sh
cp /Volumes/okamura-lab/Vina/PHAS/PHASER_Manual_Scoring_Guide.pdf "/Volumes/Install macOS Mojave/Vina/PHASER/frontend/public"
```


# Update Code:

Modify `ConfidencePanel.tsx` and `Confidence.module.css` files
- Insert the "View Scoring Guide" link below the header and above the Total Score card.
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
### ConfidencePanel.module.css
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

