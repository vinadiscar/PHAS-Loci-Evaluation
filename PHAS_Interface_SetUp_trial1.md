# PHAS Loci Confidence Evaluation

Workflow:
1. Select PHAS locus
2. IGV jumps to region
3. Inspect:
read density
phasing pattern
size distribution (if separated)
4. Assign confidence that matches goal: Identify Loci with High Confidence

##### Code architecture:
React UI
↓
IGV.js viewer
↓
Genome + BAM + GFF3
↓
Manual locus navigation

This is essentially a simplified version of:
IGV
UCSC Genome Browser
But customized for PHAS evaluation.

## 1. Create Project 

```sh
ssh -l OkamuraLab 163.221.246.151 
cd "/Volumes/Install macOS Mojave/Vina/"
mkdir PHAS_Evaluation_Resource

cd PHAS_Evaluation_Resource
npm create vite@latest
# Prompted to: 
## Need to install the following packages:
## create-vite@8.3.0
## Ok to proceed? (y) y # --> click y

> npx
> create-vite

│
◇  Project name:
│  PHASER-project
│
◇  Package name:
│  phaser-project
│
◇  Select a framework:
│  React
│
◇  Select a variant:
│  JavaScript
│
◇  Use Vite 8 beta (Experimental)?:
│  No
│
◇  Install with npm and start now?
│  Yes
│
◇  Scaffolding project in /Volumes/Install macOS Mojave/Vina/PHAS_Evaluation_Resource/PHASER-project...
│
◇  Installing dependencies with npm...

added 156 packages, and audited 157 packages in 9s

33 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
│
◇  Starting dev server...

> phaser-project@0.0.0 dev
> vite


  VITE v7.3.1  ready in 464 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help

Local:   http://localhost:5173/ # It is NOT exposed to own local host (163.221.126.251) yet. --> to make this work:
1. stop the server (Ctrl+ C)
2. Restart with network exposure 
cd PHASER-project
npm run dev -- --host --port 5174 # Change 5173 to 5174

3. Quick Test
After running with --host, try opening:
http://163.221.246.151:5174/

```

## 2. Plan PHASE Loci Interface
- ### 2.1 Install React
```sh
# 1. Install React
npm install react react-dom
npm install @vitejs/plugin-react --save-dev
```

- ###### Update vite.config.js for React

```sh
vi vite.config.js
```

```sh
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
    port: 5174
  }
});
```
- ###### Save vite.config.js
- ###### Restart the dev server in the terminal:
```sh
Ctrl+ C
npm run dev -- --host --port 5174
```
- ### 2.1 Install IGV

```sh
npm install igv

# Quick check:
npm list igv
## phaser-project@0.0.0 /Volumes/Install macOS Mojave/Vina/PHAS_Evaluation_Resource/PHASER-project
### └── igv@3.7.3
```

- ### 2.2 Create Symlinks for the required Input Files
##### Files Needed:
| File             | Purpose                    | 
| ---------------- | -------------------------- |
| `tick_genome.fa` | Reference genome sequence  |
| `sample.bam`     | Aligned reads (PHAS reads) |
| `sample.bam.bai` | BAM index (must exist)     |
| `gff3 files`     | gff3 fileswith PHAS loci   |

```sh
# Create directory
cd "/Volumes/Install macOS Mojave/Vina/PHAS_Evaluation_Resource/PHASER-project/public"
mkdir data
cd data 

# Create the Synlinks

ln -s /Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam/*.bam .

ln -s /Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/bam/*.bam.bai .

ln -s /Volumes/okamura-lab-hpc/Canran_phasiRNAs/ticks/HaeL/sRNAminer/merged.PHAS22.gff3 merged.PHAS22.gff3

ln -s /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.IGV.genome.fasta GWHAMMI00000000.IGV.genome.fasta

ln -s /Volumes/okamura-lab/Vina/Genome_info/GenomeInfo_HaeL2018/genome_GWHAMMI00000000/GWHAMMI00000000.IGV.genome.fasta.fai GWHAMMI00000000.IGV.genome.fasta.fai

``` 

- ### 2.3 edit App.js to:
    - create an IGV.js container, 
    - load genome, BAM, and PHAS BED tracks.
    - make it network-accessible via http://163.221.246.151:5174/.

```sh
cd "/Volumes/Install macOS Mojave/Vina/PHAS_Evaluation_Resource/PHASER-project/src"
vi App.jsx
```
```sh
import { useEffect, useRef, useState } from "react";
import igv from "igv";

function App() {
  const igvContainer = useRef(null);
  const [browser, setBrowser] = useState(null);
  const [loci, setLoci] = useState([]);
  const [selectedLocus, setSelectedLocus] = useState("");

  // 🔹 Load IGV
  useEffect(() => {
    const options = {
      reference: {
        id: "Hlongicornis",
        name: "Haemaphysalis longicornis",
        fastaURL: "/data/GWHAMMI00000000.genome.fasta",
        indexURL: "/data/GWHAMMI00000000.genome.fasta.fai"
      },
      tracks: [
        {
          name: "Small RNA",
          type: "alignment",
          format: "bam",
          url: "/data/N002.fastq.trimmed.mc.fa.sorted.bam",
          indexURL:
            "/data/N002.fastq.trimmed.mc.fa.sorted.bam.bai",
          visibilityWindow: 10000,
          height: 300
        },
        {
          name: "PHAS22 Loci",
          type: "annotation",
          format: "gff3",
          url: "/data/merged.PHAS22.gff3",
          displayMode: "EXPANDED",
          color: "red"
        }
      ]
    };
    
    igv.createBrowser(igvContainer.current, options).then(b => {
      setBrowser(b);

      // Force first valid region
      b.search("GWHAMMI00000001:1-10000");
    });
  }, []);

  // 🔹 Parse GFF3 for loci dropdown
  useEffect(() => {
    fetch("/data/merged.PHAS22.gff3")
      .then(response => response.text())
      .then(text => {
        const lines = text.split("\n");
        const parsed = lines
          .filter(line => !line.startsWith("#"))
          .map(line => {
            const cols = line.split("\t");
            if (cols.length < 9) return null;
            return {
              chr: cols[0],
              start: cols[3],
              end: cols[4],
              id: cols[8]
            };
          })
          .filter(Boolean);

        setLoci(parsed);
      });
  }, []);

  // 🔹 Navigate to selected locus
  const goToLocus = locus => {
    if (!browser) return;
    const region = `${locus.chr}:${locus.start}-${locus.end}`;
    browser.search(region);
    setSelectedLocus(region);
  };

  return (
    <div style={{ padding: "20px", fontFamily: "Arial" }}>
      <h1>PHAS Loci Confidence Evaluation</h1>

      <div style={{ marginBottom: "20px" }}>
        <label>Select PHAS Locus: </label>
        <select
          onChange={e =>
            goToLocus(loci[e.target.value])
          }
        >
          <option>Select...</option>
          {loci.map((locus, index) => (
            <option key={index} value={index}>
              {locus.chr}:{locus.start}-{locus.end}
            </option>
          ))}
        </select>
      </div>

      <div
        ref={igvContainer}
        style={{
          height: "600px",
          border: "1px solid black",
          borderRadius: "8px"
        }}
      />

      <div style={{ marginTop: "20px" }}>
        <h3>Manual Confidence Assignment</h3>
        <p>Selected Region: {selectedLocus}</p>
        <button>High Confidence</button>
        <button>Medium</button>
        <button>Low</button>
      </div>
    </div>
  );
}

export default App;

```
## 3. Open in Brwoser to Check

```sh
# 1. Make sure Dev Server is running
cd "/Volumes/Install macOS Mojave/Vina/PHAS_Evaluation_Resource/PHASER-project"
npm run dev -- --host --port 5174

# 2. Open:

http://163.221.246.151:5174/

```

Feb 18 2026
#### Fix Errors

- ###### Issue 1: Vite cannot serve files outside project root

You are creating symlinks inside: PHASER-project/public/data
But the original files are in: /Volumes/okamura-lab-hpc/...

- ###### Fix 1: Edit vite.config.js and add:
```sh

server: {
  host: true,
  port: 5174,
  fs: {
    allow: [".."]
  }
}
```

- ###### Issue 2:  FASTA must be indexed with samtools faidx
```sh
cd /work/ma-discar/H_longicornis/GenomeInfo_HaeL2018/genome_GWHAMMI00000000
conda activate samtools_env
samtools faidx GWHAMMI00000000.genome.fasta
```


