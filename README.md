# In-Silico MHC Profiling Pipeline (v3.0)
### Robust K-mer Genotyping for Low-Coverage Whole Genome Sequencing Data

**Date:** January 2026  
**Architecture:** Reference-Free Extraction + K-mer Containment Profiling ($k=15$)  
**Status:** Validated on WGS Cohort ($N=20$)

---

## 1. Abstract

The Major Histocompatibility Complex (MHC) is the most polymorphic region of the human genome. Standard bioinformatics pipelines often fail to accurately genotype MHC alleles from short-read Whole Genome Sequencing (WGS) data due to high sequence homology and the presence of intronic regions that disrupt alignment to cDNA references. 

This project implements a **Reference-Free, Alignment-Independent Pipeline** designed specifically for low-coverage, low-mapping-quality (MAPQ) WGS data. By utilizing coordinate-based extraction followed by **K-mer Containment Profiling**, this tool successfully recovers dominant Class I alleles (e.g., HLA-A) and stress-markers (MICA/MICB) where standard aligners (BWA/Bowtie) fail.

---

## 2. Background: The "Intron Problem"

Whole Genome Sequencing captures the entire genomic structure, including non-coding **introns** (~90% of gene length) and coding **exons**. However, standard HLA reference databases (IMGT/HLA) typically provide only the coding DNA sequence (CDS).

1.  **The Conflict:** Aligning a genomic read (containing introns) to a cDNA reference (exons only) forces standard algorithms (Needleman-Wunsch/Smith-Waterman) to apply massive gap penalties.
2.  **The Consequence:** Reads are either discarded as unmapped or assigned extremely low Mapping Quality (MAPQ) scores due to multi-mapping against paralogs.
3.  **The Solution:** This pipeline abandons global alignment in favor of **K-mer Containment**, detecting "exon signatures" within the genomic noise without requiring contiguous alignment.

---

## 3. Pipeline Architecture

The analysis workflow is divided into three distinct modules:

### Module A: Dynamic Ingestion & Forensic QC
* **Chromosome Normalization:** Automatically detects and resolves nomenclature discrepancies between reference builds (e.g., UCSC `chr6` vs. Ensembl `6`).
* **Quality Assessment:** Calculates Mean MAPQ per sample to quantify the "confidence" of raw alignments. Samples with Mean MAPQ < 10 are flagged as "High Noise."

### Module B: Reference-Free Consensus Extraction
Standard variant callers (e.g., GATK) filter out reads with low MAPQ to reduce false positives. In MHC analysis, this results in near-total data loss.
* **Mechanism:** A custom `pysam` extractor retrieves reads based solely on **genomic coordinates**, ignoring quality flags.
* **Philosophy:** Prioritize *sensitivity* at extraction; rely on downstream typing to provide *specificity*.

### Module C: Turbo K-mer Typing ($k=15$)
* **Decomposition:** Patient reads and Reference Alleles are decomposed into overlapping $k$-mers ($k=15$).
* **Scoring:** We calculate the **Percent Identity** ($I$) based on the intersection of $k$-mer sets:
    
    $$I = \frac{| K_{sample} \cap K_{ref} |}{| K_{ref} |} \times 100$$

* **Thresholding:**
    * **> 20%:** Valid Exon Match (Genomic Context).
    * **> 70%:** High-Confidence / Deep Coverage.

---

## 4. Results & Validation

This pipeline was validated on a cohort of 20 WGS samples (`MOT36424`–`MOT36428`).

### 4.1 Quality Control Landscape
* **Mapping Rate:** >99.9% (High)
* **Mean MAPQ:** < 8.0 (Critically Low)
* **Interpretation:** The aligner successfully placed reads but could not distinguish between homologous MHC genes, confirming that alignment-based typing was impossible.

### 4.2 Genotyping Performance
| Gene | Performance | Finding |
| :--- | :--- | :--- |
| **HLA-A** | **High** | Cohort is dominated by **A*02:01** (Identity ~75%). Matches global frequencies for Caucasian/Hispanic populations. |
| **MICA/MICB** | **High** | Recovered high allelic diversity (e.g., *008, *010, *102), validating pipeline sensitivity. |
| **HLA-B/C** | **Low** | Resulted in "No Call" due to insufficient read depth (<150 reads) in the input library. |

### 4.3 Key Figures
*(See `results/figures/` for full resolution plots)*

* **Figure 1 (QC):** Demonstrates the high-mapping / low-confidence paradox of MHC WGS data.
* **Figure 2 (Confidence):** Shows the bimodal distribution of K-mer scores (Noise vs. Signal).
* **Figure 3 (Diversity):** Contrasts HLA-A homogeneity with MICA/MICB heterogeneity.

---

## 5. Directory Structure

```text
.
├── bam_files/               # Input WGS alignment files (.bam)
├── results/
│   ├── figures/             # Generated plots (QC, Frequency, Distributions)
│   ├── task3_mhc_typing/    # Final Genotype CSVs
│   └── quality_summary.csv  # QC Metrics
├── gencode.v19.annotation.gtf
├── reference_genome.fa
├── main_pipeline.py         # Primary execution script
└── README.md                # Project Documentation
