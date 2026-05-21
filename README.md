# Salamandra_RNAseq

**Bioinformatics pipeline for de novo assembly and differential expression across *Salamandra* populations with divergent reproductive strategies.**

This repository contains the scripts, job files, and configurations associated with the publication: *"Comparative transcriptomic analyses identify candidate genes for convergent reproductive shifts in a bimodal viviparous amphibian"* (BioProject: PRJNA1450543).

## 📌 Pipeline Overview
This pipeline is designed for high-performance computing (HPC) clusters utilizing a PBS/Torque resource manager. It takes raw RNA-Seq reads through quality control, trimming, de novo assembly, structural/functional annotation, and read quantification. 

The pipeline is divided into 6 sequential steps:
1. **Quality Control** (`FastQC`)
2. **Read Cleaning** (`Trim_Galore`)
3. **De Novo Assembly** (`Trinity`)
4. **Assembly Filtering & Reduction** (`TransPi` / Evigene)
5. **Functional Annotation** (`EnTAP`)
6. **Transcript Quantification** (`Salmon`)

---

## ⚙️ Dependencies & Software
Ensure the following modules and software environments are installed and available in your `$PATH` or loaded via your cluster's module system:
* `FastQC`
* `Trim_Galore`
* `Trinity` (with `BioPerl`)
* `TransPi` (via Nextflow / Singularity / Conda)
* `EnTAP` (v2.1.0+) — *Note: EnTAP execution in this script requires GLIBC 2.34+ (e.g., RHEL 9).*
* `Salmon` (v1.10.3+)
* `seqtk`

---

## 📂 Project Initialization & Directory Structure
The `.job` scripts rely on a **strict relative directory structure** to locate inputs and write outputs without manual intervention. Before submitting any jobs, initialize your project root directory exactly like this:

```text
Project_Root/
├── 1_RAW/         # Place all raw *_1.fastq.gz and *_2.fastq.gz files here
├── 2_CLEAN/       # Contains 2_Clean.job
├── 3_TRINITY/     # Contains 3_Trinity.job and TrinitySamples.txt
├── 4_TransPI/     # Contains 4_TransPI.job, nextflow.config
├── 5_Entap/       # Contains 5_Entap.job, entap_run.params, entap_config.ini
└── 6_Salmon/      # Contains 6_Salmon.job
```

---

## 🚀 Step-by-Step Execution Guide

### Step 1: Raw Read Quality Control (`1_FastQC.job`)
* **Location:** Run from within the `1_RAW` directory.
* **Command:** `qsub 1_FastQC.job`
* **Description:** Runs `FastQC` on all raw `.fastq.gz` reads to establish a baseline for read quality, adapter contamination, and sequence duplication. 
* **Output:** Results are saved to a new `FastQC_RAW` subdirectory. Check the HTML reports to identify if there are any problematic samples or issues.

### Step 2: Read Trimming & Cleaning (`2_Clean.job`)
* **Location:** Run from within the `2_CLEAN` directory.
* **Command:** `qsub 2_Clean.job`
* **Description:** Reads from `../1_RAW`. Uses `Trim_Galore` to automatically detect and remove sequencing adapters and low-quality bases. After trimming, it runs a second FastQC pass on the cleaned reads.
* **Output:** Generates `*_val_1.fq.gz` and `*_val_2.fq.gz` read pairs in `2_CLEAN`. It also compiles a `read_counts.tsv` summary file tracking how many reads were retained vs. discarded.

### Step 3: De Novo Assembly (`3_Trinity.job`)
* **Location:** Run from within the `3_TRINITY` directory.
* **Command:** `qsub 3_Trinity.job`
* **Requirement:** You **must** supply a tab-delimited `TrinitySamples.txt` file identifying the experimental conditions, replicates, and relative paths to the cleaned left/right reads from Step 2 (e.g., `../2_CLEAN/sample_1_val_1.fq.gz`).
* **Description:** Executes the `Trinity` assembler using a 510GB memory allocation. 
* **Output:** A raw de novo transcriptome assembly: `trinity_Salamandra.Trinity.fasta`.

### Step 4: Transcriptome Processing & Filtering (`4_TransPI.job`)
* **Location:** Run from within the `4_TransPI` directory.
* **Command:** `qsub 4_TransPI.job`
* **Description:** Utilizes the Nextflow-based TransPi pipeline. This step filters out artifacts, reduces redundancy (via CD-HIT), and isolates the highest quality contiguous sequences (the "okay" set) using Evigene.
* **Output:** Look for the final cleaned FASTA file in `results/evigene/` (e.g., `*.okay.fa`). 

### Step 5: Functional Annotation (`5_Entap.job`)
* **Location:** Run from within the `5_Entap` directory.
* **Prerequisite:** Copy the `*.okay.fa` file from Step 4 into this directory and ensure the filename matches the `input=` parameter in `entap_run.params`.
* **Command:** `qsub 5_Entap.job`
* **Description:** Runs the Eukaryotic Non-Model Transcriptome Annotation Pipeline (EnTAP). It incorporates structural annotation (TransDecoder) and similarity searches (EggNOG, DIAMOND) to assign functional descriptions.
  * *Custom Loop:* This script includes a custom recovery loop that identifies missing mitochondrial genes, translates them, and integrates them back into the main protein set before finalizing database searches.
* **Output:** Generates `AnnotatedContigs.txt`, providing a list of all transcript IDs that successfully received a functional annotation.

### Step 6: Transcript Quantification (`6_Salmon.job`)
* **Location:** Run from within the `6_Salmon` directory.
* **Command:** `qsub 6_Salmon.job`
* **Description:** This step prepares the data for Differential Expression (DE) analysis:
  1. Uses `seqtk` to subset the Evigene-cleaned transcriptome so it *only* includes contigs that passed EnTAP functional annotation.
  2. Builds a `Salmon` index for the subsetted transcriptome.
  3. Quantifies the expression of the cleaned reads (`../2_CLEAN/`) against the index.
* **Output:** Generates individual `*_qt` quantification folders for each sample, and summarizes the alignment success rate in `mapping_rates_summary.tsv`.

---

## 🛠️ Troubleshooting & Notes
* **Memory Limits:** The Trinity assembly (Step 3) requests 550GB of RAM buts its dependent on the size of your dataset.
* **Database Paths:** Ensure that the database paths in `entap_config.ini` and `nextflow.config` (for TransPi) point to the correct locations on your specific cluster before running.

---
