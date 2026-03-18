# SX_Data_Extractor
AI-powered pipeline that reads TBP solvent extraction research papers (PDF) and automatically extracts experimental data - extraction efficiencies, TBP/acid concentrations, experimental conditions- from text, tables, and charts using Claude AI. Outputs a structured Excel spreadsheet ready for ML training.


# AI Scientific Data Extractor — TBP Solvent Extraction

> A multi-agent pipeline that reads chemistry research papers (PDFs) and automatically extracts structured experimental data into a formatted Excel spreadsheet — powered by [Claude](https://www.anthropic.com/claude) (Anthropic).

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Pipeline Architecture](#pipeline-architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Output Format](#output-format)
- [Lookup Tables](#lookup-tables)
- [Project Structure](#project-structure)
- [Limitations](#limitations)
- [License](#license)

---

## Overview

Reading scientific papers and manually transcribing experimental data into spreadsheets is one of the most time-consuming tasks in research. This project automates that process entirely for papers on **TBP (tributyl phosphate) solvent extraction** chemistry.

Given a PDF research paper, the pipeline:

1. Parses the full text, all tables, and all figures from the PDF.
2. Deploys multiple specialised AI agents (each backed by Claude) to extract data from each source type.
3. Evaluates any algebraic equations found in the paper to generate additional synthetic data points.
4. Merges everything into a single, clean dataset enriched with physical-chemistry properties (ionic radii, dielectric constants).
5. Exports a color-coded, annotated Excel file ready for further analysis.

The target data includes: **distribution ratios (D)**, **extraction efficiencies (E%)**, **partition coefficients**, and all associated experimental conditions (acid concentration, TBP concentration, O/A ratio, metal concentration, etc.).

---

## Features

- **Full PDF parsing** — text, embedded tables, and figure images are all extracted automatically.
- **Multi-agent AI extraction** — separate Claude agents handle text, tables, figures, and equations, each with a specialised prompt.
- **Vision-based chart digitization** — Claude reads axis labels and data series directly from plot images, handling logarithmic scales and multi-series figures.
- **Equation evaluation** — algebraic relationships found in the paper (e.g., `log D = a + b·log[TBP]`) are numerically evaluated to produce data points.
- **Built-in chemistry database** — ionic radii for 70+ cations and 7 anions, dielectric constants for 18 solvents, all sourced from literature.
- **Automatic D ↔ E% conversion** — if only D or E% is known, the other is derived automatically using the O/A ratio.
- **Color-coded Excel output** — rows are color-coded by data source (figure, table, equation, or text), with a built-in Legend sheet.
- **Multi-metal figure support** — when a single figure shows data for multiple metals, each series is tagged and processed independently.

---

## How It Works

The pipeline is composed of **5 specialised agents** that run sequentially:

### 1. Text Agent
Reads the full text of the paper and identifies the global experimental context:
- Metal being studied (symbol and full name)
- Oxidation state
- Counter-anion (e.g., nitrate, chloride)
- Organic diluent/solvent
- Fixed experimental conditions (acid concentration, TBP%, O/A ratio)
- Any algebraic equations relating D, E%, or partition coefficient to experimental variables
- Specific numerical values mentioned inline in the text

### 2. Table Agent
For every data table found in the PDF, sends the table content (as pipe-separated text) to Claude and asks it to extract one structured row per table row, including acid concentration, TBP concentration, D, E%, partition coefficient, and metal concentration.

### 3. Figure Agent (Vision)
For every figure image extracted from the PDF:
- First determines whether the image is a data plot or something else (schematic, photo, etc.).
- If it is a data plot, reads the x/y axis labels and units, identifies each data series, and digitizes the (x, y) point coordinates.
- Handles logarithmic axes correctly (reads the raw log-scale value and converts back).
- Supports multi-metal figures by tagging each series with the appropriate element symbol.

### 4. Equation Evaluator
Takes any equations found by the text agent and evaluates them numerically:
- Parses the equation string and maps symbolic names (TBP, acid, H⁺, pH) to a variable `x`.
- Determines a valid x-axis range (from the paper if stated, or from sensible defaults).
- Evaluates the equation at 10 evenly spaced points to produce synthetic data rows.
- Converts the result to D or E% depending on what the equation's left-hand side represents.

### 5. Assembler
Merges all rows from all agents into a unified DataFrame:
- Fills in missing values from global metadata extracted by the text agent.
- Looks up ionic radii and dielectric constants from the built-in chemistry tables.
- Converts TBP concentration between vol% and mol/L as needed.
- Cross-computes D and E% from each other when one is missing, using:

```
E (%) = 100 × D / (D + O/A)
D = (E/100) × (O/A) / (1 − E/100)
```

---

## Pipeline Architecture

```
PDF file
   │
   ├─── pdfplumber ──────► Full text
   │                       Data tables
   │
   ├─── PyMuPDF ─────────► Figure images
   │
   ▼
┌─────────────────────────────────────────────────┐
│                 Claude AI Agents                │
│                                                 │
│  Text Agent ──────────► metadata, equations,    │
│                          inline data points     │
│                                                 │
│  Table Agent ─────────► one row per table row   │
│                                                 │
│  Figure Agent (vision)► digitized chart points  │
│                                                 │
│  Equation Evaluator ──► synthetic data points   │
└─────────────────────────────────────────────────┘
   │
   ▼
Assembler
   │  ├── lookup ionic radii & dielectric constants
   │  ├── fill missing values from global metadata
   │  ├── convert TBP vol% ↔ mol/L
   │  └── cross-compute D ↔ E%
   │
   ▼
Formatted Excel file (.xlsx)
   ├── Sheet 1: Extracted Data (color-coded by source)
   └── Sheet 2: Legend
```

---

## Prerequisites

- Python **3.9+**
- An [Anthropic API key](https://console.anthropic.com/) (Claude Opus or Sonnet)
- The following Python packages:

| Package | Purpose |
|---|---|
| `anthropic` | Claude API client |
| `pdfplumber` | Text and table extraction from PDFs |
| `PyMuPDF` (`fitz`) | Figure/image extraction from PDFs |
| `Pillow` (`PIL`) | Image processing for vision calls |
| `pandas` | DataFrame assembly |
| `openpyxl` | Excel file creation and formatting |

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-username/ai-data-extractor.git
cd ai-data-extractor

# 2. Create and activate a virtual environment (recommended)
python -m venv venv
source venv/bin/activate        # Linux / macOS
venv\Scripts\activate           # Windows

# 3. Install dependencies
pip install anthropic pdfplumber PyMuPDF Pillow pandas openpyxl
```

---

## Configuration

### API Key
Create a plain-text file containing only your Anthropic API key (starting with `sk-ant-...`):

```
C:\Users\you\Documents\API key.txt
```

Then update the `API_KEY_PATH` variable in **Cell 1**:

```python
API_KEY_PATH = r"C:\Users\you\Documents\API key.txt"
```

### PDF and Output Paths
Edit these two variables in **Cell 1**:

```python
PDF_PATH    = r"C:\path\to\your\paper.pdf"
OUTPUT_PATH = r"C:\path\to\output.xlsx"   # auto-generated next to the PDF if left as default
```

### Model Selection
The default model is `claude-opus-4-6` (most capable). You can switch to `claude-sonnet-4-6` for faster and cheaper extraction at a slight cost to accuracy:

```python
MODEL = "claude-opus-4-6"    # most capable
# MODEL = "claude-sonnet-4-6"  # faster and cheaper
```

---

## Usage

Run the notebook cells **in order from Cell 1 to Cell 11**. The pipeline will:

1. Parse the PDF and display a summary (`X pages | Y tables | Z figures`).
2. Run each agent sequentially, logging progress to the console.
3. Print a row count summary:

```
Rows collected:
  Tables    : 12
  Figures   : 47
  Equations : 20
  Text      : 3
  TOTAL     : 82
```

4. Write the final Excel file and print its full path.

> **Note:** Processing time depends on the number of figures and tables. A typical 20-page paper with 8 figures takes approximately 2–5 minutes.

---

## Output Format

### Sheet 1 — Extracted Data

Each row represents one experimental data point. Columns:

| Column | Description |
|---|---|
| Source PDF | Name of the input PDF file |
| Figure / Table | Which figure, table, or equation the row came from |
| Metal (cation) | Element symbol (e.g., `Pu`, `U`, `Fe`) |
| Acid concentration (mol/L) | Aqueous phase acid concentration |
| Size cation extracted (Å) | Ionic radius from the built-in lookup table |
| Oxidation state cation | Integer oxidation state |
| Initial aqueous cation concentration (g/L) | Metal concentration in the aqueous phase **before** extraction |
| Anion | Counter-anion name (e.g., `nitrate`) |
| Anion concentration (mol/L) | Set equal to acid concentration |
| TBP concentration (%v/v) | TBP volume percent |
| TBP concentration (mol/L) | TBP molar concentration |
| Size Anion (Å) | Ionic radius of the anion |
| Dielectric constant solvent | Dielectric constant of the organic diluent |
| Solvent | Name of the organic diluent |
| O/A ratio | Organic-to-aqueous volume ratio |
| Distribution ratio D | Extracted D value (or derived from E%) |
| Extraction efficiency E (%) | Extracted E% (or derived from D) |
| Partition coefficient (raw) | Raw value if paper uses "partition coefficient" terminology |
| Notes / confidence | Series name, equation used, or other qualifiers |

### Row Color Coding

| Color | Source |
|---|---|
| 🔵 Blue | Figure (vision-digitized chart point) |
| 🟢 Green | Table (LLM-parsed table row) |
| 🟡 Amber | Equation (algebraic evaluation point) |
| ⬜ Grey | Text (value stated in running text) |

### Sheet 2 — Legend
Explains all columns, color codes, conversion formulas, and assumptions used during extraction.

---

## Lookup Tables

The following reference data is hardcoded in **Cell 2** and used to automatically enrich extracted rows.

### Cation Ionic Radii (Shannon, 1976)
Coverage: 70+ elements including all actinides, lanthanides, transition metals, and main-group elements. Each element maps to one or more `(oxidation_state, radius_Å)` pairs.

Example entries:

| Symbol | Oxidation State | Radius (Å) |
|---|---|---|
| U | +4 | 1.03 |
| U | +6 | 0.87 |
| Pu | +3 | 1.14 |
| Pu | +6 | 0.85 |
| Fe | +2 | 0.920 |
| Fe | +3 | 0.785 |

### Anion Ionic Radii

| Anion | Radius (Å) |
|---|---|
| fluoride | 1.19 |
| chloride | 1.67 |
| bromide | 1.82 |
| nitrate | 0.27 |
| sulfate | 0.43 |
| phosphate | 0.52 |
| perchlorate | 0.41 |

### Solvent Dielectric Constants

| Solvent | Dielectric Constant |
|---|---|
| kerosene | 1.80 |
| heptane | 1.92 |
| toluene | 2.38 |
| chloroform | 4.80 |
| TBP (pure) | 8.34 |
| 1-octanol | 10.30 |

---

## Project Structure

```
ai-data-extractor/
│
├── extractor.ipynb        # Main Jupyter notebook (Cells 1–11)
│
├── API key.txt            # Your Anthropic API key (DO NOT commit this)
│
├── your_paper.pdf         # Input PDF (not included)
│
└── your_paper_extracted.xlsx   # Output Excel file (auto-generated)
```

>  **Security**: Never commit your API key file to version control. Add it to your `.gitignore`:
> ```
> API key.txt
> *.txt
> ```

---
## Demonstrated example — Au/TBP/chloride system
The pipeline was validated against the following peer-reviewed paper:

Sadeghi, N., & Keshavarz Alamdari, E. (2016). A new approach for monitoring and controlling the extraction of gold by tri-butyl phosphate from chloride media. Minerals Engineering, 85, 34–37.
DOI: 10.1016/j.mineng.2015.10.004

**What the paper contains**

Metal: Au (gold), oxidation state +3, as AuCl₄⁻ tetrachloroaurate complex
Extractant: TBP in kerosene
Anion: chloride (HCl media, 0.25–10 mol/L)
Initial Au concentration: 68–500 mg/L
TBP range: 0.025–0.5 mol/L


The paper reports data across multiple figures, including:

E% vs [HCl] at varying TBP concentrations (Fig. 1a)
E% vs [HCl] at varying initial Au concentrations at TBP = 0.1 M (Fig. 1b)


**Validation: AI agent vs. manual extraction**
To quantify the pipeline's accuracy, AI-extracted E% values were compared against values digitized manually from the same figures, at matched experimental conditions, using Autoimeris.io. Two datasets were used:

TBP variation: 4 TBP concentrations (0.075, 0.1, 0.25, 0.5 mol/L), [Au]₀ = 0.5 g/L
[Au]₀ variation: 4 initial Au concentrations (0.068, 0.13, 0.30, 0.50 g/L), TBP = 0.1 M

Parity statistics (n = 10 matched points)
R² = 0.982
RMSE = 2.00 
AARD 1.79 % 
Mean bias (AI − manual) = +1.44 % (slight AI overestimation)

The AI agent achieves R² = 0.982 against manual digitization, with an average absolute deviation of 1.79% on extraction efficiency - well within the typical experimental uncertainty of solvent extraction measurements.
The small positive bias (+1.44%) suggests the agent slightly overestimates E% at lower extraction values, likely due to chart axis reading at the lower end of the scale. This is consistent with the expected limitations of vision-based digitization and reinforces the importance of human review.

---


---

## Limitations
- **Human review is mandatory**: This pipeline is an AI-assisted extraction tool, not an autonomous one. Extracted values must always be verified by a domain expert against the original paper before being used in any scientific analysis, database, or publication. AI agents can misread axis scales, confuse table columns, or misidentify experimental conditions — the output should be treated as a first draft, not a ground truth.
- **TBP extraction focused**: The prompts are specifically tuned for TBP solvent extraction papers. Applying it to other chemistry domains will require prompt adjustments.
- **Figure digitization accuracy**: Chart reading accuracy depends on image quality and plot complexity. Low-resolution or heavily overlapping plots may yield less accurate data.
- **Equation parsing**: Highly complex or non-standard equation formats may not parse correctly. The evaluator covers the most common forms (`log D = a + b·log[X]`).
- **PDF quality**: Scanned PDFs without embedded text layers will produce poor text extraction. A text-layer PDF (born-digital) is strongly recommended.
- **API costs**: Processing a long paper with many figures using `claude-opus-4-6` can consume significant API credits. Use `claude-sonnet-4-6` for cost-sensitive workflows.
- **PyMuPDF dependency**: Figure extraction requires `PyMuPDF`. If not installed, the figure agent is skipped (a warning is printed), and only text and table data are extracted.
---

## License

This project is released under the MIT License. See `LICENSE` for details.

---

*Built with [Claude](https://www.anthropic.com/claude) by Anthropic.*
