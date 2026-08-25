
###### PHAS_interface_Workflow_raw_16.md
- The previous gene annotation used was a public genome annotation, but it's not quiet good. Replace it with `HaeL2018_stringtie_merged_transcripts.gff3`. 

```sh
Source: /project/okamura-lab/Canran/Doctoral_thesis/results/HaeL2018_transcriptome_assembly/stringtie/HaeL2018_stringtie_merged_transcripts.gff3
/work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12
```

# Update gene annotation
## Prepare Genome annotation file 
- ### Convert .gtf annotation file to bed12 file
```sh
cd /work/ma-discar/UCSC_utilities
./gtfToGenePred -genePredExt /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.gtf /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.genePred
./genePredToBed /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.genePred /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12

head /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12
```
- ### Copy bed12 annotation file
```sh
cp /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12 /project/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018

cp /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/HaeL2018_stringtie_merged_transcripts.bed12 "/Volumes/Install macOS Mojave/Trackhubs/PHAS_tracks/annotation"

```

## Update the following files

- config.py
- tracks_service.py
- IGVViewer.tsx


## Modified Code:

- ### config.py
```sh
"""PHASER Configuration"""

from pathlib import Path

# =========================================================
#  BASE DIRECTORY
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
# TRACK HUB
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
TRANSCRIPT_BED12 = (
    ANNOTATION_DIR
    / "HaeL2018_stringtie_merged_transcripts.bed12"
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
# RPM BIGWIG ROOT
# =========================================================
RPM_BW_DIR = (
    TRACKHUB_BASE
    / "sRNA_bw"
    / "bigwig_rpm"
)

# =========================================================
# 5' END RPM sRNA BIGWIG DIRECTORY
# =========================================================
FIVEPRIME_BW_DIR = (
    RPM_BW_DIR
    / "5prime_rpm_sRNA_bw"
)

# =========================================================
# COVERAGE RPM sRNA BIGWIG DIRECTORY
# =========================================================
COVERAGE_BW_DIR = (
    RPM_BW_DIR
    / "coverage_rpm_sRNA_bw"
)

# =========================================================
#  API
# =========================================================
API_PREFIX = "/api"
```

- ### tracks_service.py
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
    TRANSCRIPT_BED12,
    BW_DIR,
    MRNA_BW_DIR,
    FIVEPRIME_BW_DIR,
    COVERAGE_BW_DIR
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
#  STRAIN COLORS
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
    #  LOAD LIBRARY INFO
    # =====================================================
    def _load_library_info(self) -> pd.DataFrame:

        if self._library_info is None:
            self._library_info = pd.read_csv(
                LIBRARY_INFO_FILE,
                sep="\t"
            )

        return self._library_info

    # =====================================================
    #  ALL TRACKS
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
    #  TRACK BY ID
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
    # GENOME FILES
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
    #  FIND BIGWIG FILES
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
    #  SINGLE SAMPLE IGV CONFIG
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
            #  GENOME ANNOTATION (BED12)
            # =================================================
            {
                "name": "Genome Annotation",

                "type": "annotation",

                "format": "bed",

                "url": (
                    f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
                ),


                "displayMode": "EXPANDED",

                "visibilityWindow": 500000,

                "height": 120,

                "color": "green"
            },

            # =================================================
            #  PHAS LOCI
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
            #  PLUS STRAND
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
            #  MINUS STRAND
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
    #  ALL IGV TRACKS
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
        #  GENOME ANNOTATION
        # =================================================
        tracks.append({

            "name": "Genome Annotation",

            "type": "annotation",

            "format": "bed",

            "url": (
                f"/api/files/annotation/{TRANSCRIPT_BED12.name}"
            ),


            "displayMode": "EXPANDED",

            "visibilityWindow": 500000,

            "height": 120,

            "color": "black"
        })

        # =================================================
        #  PHAS LOCI
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
        #  sRNA TRACKS
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
        #  mRNA TRACKS
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

        
        # =================================================
        #  5' END RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning 5' RPM BW directory...")

        for f in FIVEPRIME_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} 5' End ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/5prime_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_5prime",

                "color": "#8E24AA",

                "height": 50
            })

        # =================================================
        #  COVERAGE RPM sRNA TRACKS
        # =================================================
        print("[IGV ALL] Scanning coverage RPM BW directory...")

        for f in COVERAGE_BW_DIR.glob("*.bw"):

            name = f.name

            sample = name.split(".")[0]

            strand = "+" if ".plus." in name else "-"

            meta = parse_sample_metadata(sample)

            label = SAMPLE_LABELS.get(sample, sample)

            tracks.append({

                "name": f"{label} Coverage ({strand})",

                "type": "wig",

                "format": "bigWig",

                "url": (
                    f"/api/files/coverage_rpm_sRNA_bw/{name}"
                ),

                "sample": sample,

                "strain": meta["strain"],

                "stage": meta["stage"],

                "feeding": meta["feeding"],

                "strand": strand,

                "label": label,

                "rnaType": "sRNA_coverage",

                "color": "#F57C00",

                "height": 50
            })
        
        return {
            "genome": genome,
            "tracks": tracks
        }

    # =====================================================
    #  FILE RESOLUTION
    # =====================================================
    def resolve_file_path(
        self,
        relative_path: str
    ) -> Optional[Path]:

        if relative_path.startswith("5prime_rpm_sRNA_bw/"):

            resolved = (
                FIVEPRIME_BW_DIR
                / relative_path.replace(
                    "5prime_rpm_sRNA_bw/",
                    ""
                )
            )

        elif relative_path.startswith(
            "coverage_rpm_sRNA_bw/"
        ):

            resolved = (
                COVERAGE_BW_DIR
                / relative_path.replace(
                    "coverage_rpm_sRNA_bw/",
                    ""
                )
            )

        else:

            resolved = (
                TRACKHUB_BASE
                / relative_path
            )

        print(

            f"[FILE RESOLVE] "

            f"{relative_path} -> {resolved}"
        
        )


        return resolved

track_service = TrackService()
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
  //  SAMPLE → MATCHING MRNA LABELS
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
  // SYNC SAMPLE
  // =====================================================
  useEffect(() => {

    setFilters((prev) => ({
      ...prev,
      samples: [sample],
    }));

  }, [sample]);

  // =====================================================
  // LOAD IGV
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
        //  CREATE BROWSER
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
          //  ANNOTATION TRACKS
          // =============================================
          const annotationTracks =
            allTracks.filter(
              (t) =>
                t.format === "bed" ||
                t.format === "bigBed"
            );

          // =============================================
          //  SIGNAL TRACKS
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
                  // MRNA
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
          // SORT TRACKS
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
  //  LOCUS UPDATE
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

- ### tracks.py
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


#  IGV ROUTES FIRST (VERY IMPORTANT)
@router.get("/igv/all")
async def get_all_igv_tracks():
    return track_service.get_all_igv_tracks()


@router.get("/igv/{sample}")
async def get_igv_config(sample: str):
    config = track_service.get_igv_config(sample)

    if not config:
        raise HTTPException(status_code=404, detail=f"Sample {sample} not found")

    return config


#  GENERIC ROUTE LAST (ALWAYS LAST)
@router.get("/{library_id}", response_model=TrackFile)
async def get_track_by_id(library_id: str):
    track = track_service.get_track_by_id(library_id)
    if not track:
        raise HTTPException(status_code=404, detail=f"Track {library_id} not found")
    return track
```
