<p align="center">
  <img src="images/ViruBench.svg" width="200" alt="ViruBench Logo">
</p>

# Introduction
ViruBench is a scalable and reproducible Snakemake workflow designed to benchmark the performance of various long-read assemblers on viral metagenomic Nanopore data.

ViruBench employs a "semi-targeted" multi-tier assembly strategy to maximize genome recovery and compare tool performance across different levels of complexity.

# Key Features

- **Targeted Read Filtering**:
	- Pre-filters raw Nanopore reads using a custom protein database (e.g., all human viruses) to enrich for sequences of interest.
- **Multi-Tier Assembly Strategy**:
	- **Primary**: Individual de novo assemblies from multiple state-of-the-art tools.
    - **Secondary (Reassembly)**: Merging all primary contigs and re-assembling them to improve continuity.
    - **Final**: A hybrid assembly combining re-assembled contigs with unique, clustered contigs from the primary pool (via [MMseqs2](https://github.com/soedinglab/MMseqs2)).
- **Comprehensive Evaluation**: Automated benchmarking of all three tiers using:
 	- **Reference-Free:** ([CheckV](https://bitbucket.org/berkeleylab/checkv)) for completeness and quality.
    - **Reference-Based:** ([QUAST](https://github.com/ablab/quast)) for accuracy against specific viruses of interest.

This workflow is designed to help researchers evaluate which assembly tool is most appropriate for their data, especially in scenarios where reliable reference genomes are unavailable or highly diverse.

Current supported assemblers:
- [metaFlye](https://github.com/mikolmogorov/Flye)
- [Canu](https://github.com/marbl/canu)
- [Hifiasm](https://github.com/chhylp123/hifiasm)
- [Raven](https://github.com/lbcb-sci/raven)
- [PenguiN](https://github.com/soedinglab/plass)
- [Myloasm](https://github.com/bluenote-1577/myloasm)
- [MetaMDBG](https://github.com/GaetanBenoitDev/metaMDBG)
- [wtdbg2 / Redbean](https://github.com/ruanjue/wtdbg2)
- [Shasta](https://github.com/paoloshasta/shasta)
- [Miniasm](https://github.com/lh3/miniasm)

# How to run

First, clone the repo and navigate into the project directory:

```
# Clone repo
git clone https://github.com/LucvZon/ViruBench.git

cd ViruBench
```

## 1. Configure the workflow

Before starting, update config/virubench.yaml to specify your input samples, external database paths, and parameter settings.

Key sections to adjust:

- `samples`: Choose a sample name and supply a path to its corresponding fastq.gz files.
- `paths`: Absolute paths to your DIAMOND (diamond_db) database and CheckV (checkv_db) database.
- `viruses_of_interest` (optional): Fill in a species name and its corresponding FASTA reference file.
- `assemblers`: Toggle individual assemblers on (true) or off (false).
- `params`: Adjust number of threads and specific tool parameters.

**example template of config/virubench.yml**:
```
# --- Sample Configuration ---
samples:
  Sample1: "/path/to/input/reads/"
 
# --- External Databases ---
paths:
  diamond_db: "/path/to/diamond/database.dmnd"
  checkv_db: "/path/to/checkv/"
 
# --- Assembler Selection ---
assemblers:
  metaflye: true
  penguin: true
  raven: true
  canu: true
  myloasm: true
  metamdbg: true
  wtdbg2: true
  shasta: true
  miniasm: true
  hifiasm: true

# --- REASSEMBLY ---
reassembly:
  reassemble_contigs: true
 
# --- General Parameters ---
params:
  threads: 24

  # --- UNIVERSAL THRESHOLDS ---
  min_read_len: 150
  min_contig_len: 600

  # --- METAFLYE ---
  metaflye_min_overlap: 1000

  # --- PENGUIN ---
  penguin_min_contig_len: 600
  penguin_min_seq_id: 0.95
  penguin_min_overlap: 250
  penguin_min_clust_seq_id: 0.95
  penguin_min_clust_cov: 0.85 
  
  # --- CANU ---
  canu_min_read_len: 300
  canu_min_overlap: 250
  canu_min_contig_len: 600
  canu_min_cov: 5
  canu_genome_size: 200k

  # --- MYLOASM ---
  myloasm_min_seq_id: 0.95
  myloasm_min_cov: 5
  myloasm_min_overlap: 250

  # --- METAMDBG ---
  metamdbg_min_overlap: 250
  metamdbg_min_seq_id: 0.95

  # --- WTDBG2 ---
  wtdbg2_min_read_len: 300
  wtdbg2_min_contig_len: 600
  wtdbg2_min_overlap: 250
  wtdbg2_min_cov: 5
  wtdbg2_min_node_len: 512

  # --- SHASTA ---
  shasta_min_read_len: 300
  shasta_min_cov: 5

  # --- MINIASM ---
  miniasm_min_overlap: 250
  miniasm_min_cov: 5
  miniasm_min_seq_id: 0.95

  # --- RAVEN ---
  raven_min_contig_len: 1000
  raven_min_seq_id: 0.95

  # --- HIFIASM ---
  hifiasm_min_read_len: 300
  hifiasm_tip_contig_filter: 5

  # --- INSPECTOR ---
  inspector_min_depth: 30
  inspector_min_length: 1000

  # --- DIAMOND ---
  # options: fast, mid-sensitive, sensitive, more-sensitive, very-sensitive, ultra-sensitive
  diamond_sensitivity: sensitive

# --- Virus Selection (Optional) ---
viruses_of_interest:
  # Key: A species name for the virus (must match the name in the DIAMOND database)
  # Value: Path to the reference FASTA for that virus
  # Virus_name_1: "path/to/reference/genome"
  # Virus_name_2: "path/to/reference/genome"
```

## 2. Execute ViruBench

Choose **one** of the four options below depending on your environment.  
> **Note on Apptainer/Singularity bind mounts (Options 2-4)**:
> If your raw reads or external databases reside outside your `$HOME` or the current working directory, you must expose them to the container using the `--bind` flag (e.g., `--bind /data:/data`).

### Option 1: Using a local Conda environment
Best if you prefer managing dependencies locally or want to submit individual sub-jobs via the native Snakemake SLURM profile.
```bash
# Create and activate the conda environment
conda env create -f env/environment_with_versions.yml
conda activate virubench

# Run the pipeline
snakemake --snakefile workflow/snakefile.smk --cores 24
```

### Option 2: Download the container from GitHub Releases
1. Download the pre-built `virubench.sif` image from the [latest github release](https://github.com/lucvzon/virubench/releases/latest)
2. Execute the workflow:
```bash
apptainer exec \
    --bind /path/to/reads:/path/to/reads \
    --bind /path/to/databases:/path/to/databases \
    virubench.sif \
    snakemake --snakefile workflow/snakefile.smk --cores 24
```

### Option 3: Stream and run directly from GHCR
Apptainer will fetch the image directly into cache and run the pipeline in a single command.
```bash
apptainer run \
    --bind /path/to/reads:/path/to/reads \
    --bind /path/to/databases:/path/to/databases \
    oras://ghcr.io/lucvzon/virubench:1.0.0 \
    snakemake --snakefile workflow/snakefile.smk --cores 24
```

### Option 4: Pull the container from GHCR and run locally
```bash
# Pull the image once into your local directory
apptainer pull virubench.sif oras://ghcr.io/lucvzon/virubench:1.0.0

# Run the workflow
apptainer exec \
    --bind /path/to/reads:/path/to/reads \
    --bind /path/to/databases:/path/to/databases \
    virubench.sif \
    snakemake --snakefile workflow/snakefile.smk --cores 24
```

# Workflow

### 1. Read Preparation & QC
- **Input**: Raw FASTQ files for a sample.
- **Process**:
	- -> Merge all raw reads per sample.
	- -> Trim adapters and low-quality ends ([cutadapt](https://github.com/marcelm/cutadapt)).
	- -> Perform quality filtering ([fastplong](https://github.com/OpenGene/fastplong)).
- **Output**: One cleaned FASTQ file per sample.

### 2. Targeted Read Filtering
- **Input**: The cleaned FASTQ file.
- **Process**:
	- -> Classify reads against a protein database (e.g., all viral proteins) using [DIAMOND](https://github.com/bbuchfink/diamond) BLASTX.
	- -> Extract only the reads that hit the database.
- **Output**: A smaller FASTQ file containing only the reads of interest (viral "target reads").

### 3. Chimeric filtering and downsampling
- **Input**: The target reads.
- **Process**:
	- -> Reads containing inverted duplications are identified and discarded.
	- -> Resulting read data is downsampled using [BBNorm](https://bbmap.org/tools/bbnorm) to further statistically dilute out leftover chimeric reads.
- **Output**: A FASTQ file only containing downsampled, cleaned, target reads.

### 4. Denovo Assembly
- **Input**: The "target reads" FASTQ file.
- **Process**: The same set of target reads is given to each active assembler.
- **Output**: Multiple raw assembly files (one FASTA per assembler).

### 5. Contig Annotation
- **Input**: The raw assembly files from each assembler.
- **Process**:
	- -> Contigs from all assemblers are collected and annotated against the protein database (DIAMOND).
	- -> A custom script post-processes the results to assign the best taxonomic lineage to each contig.
- **Output**: An annotated table (TSV) for each assembly, linking contigs to potential viruses.

### 6. Per-Assembler Evaluation
- (These steps run independently for each assembler's output to assess its individual quality)
- **Process**:
	- -> General Stats: Calculate basic assembly metrics (N50, total length, etc.).
	- -> Read Mapping: Map all high-quality reads back to the assembly to check coverage.
	- -> Viral QC: Run CheckV to estimate viral genome completeness and check for contamination.
- Output: Individual quality reports and statistics for each assembly.

### 7. Targeted Comparative Analysis
- (This final stage brings the results together for a direct comparison on specific viruses of interest)
- **Process**:
	- -> Binning: Based on the annotations, contigs are binned into separate FASTA files by virus species.
	- -> Comparative QUAST: For each virus of interest (e.g., Monkeypox virus), the binned contigs from all assemblers are compared against the official reference genome.
	- -> Contig Mapping: The binned contigs are also mapped to the reference genome to visualize alignment.
- **Output**: A final, comparative QUAST report showing which assembler performed best for each targeted virus.

### 8. Report summary
- A Quarto book is rendered into results/report/final_summary_report/index.html.

# Outputs

- `results/1_quality_control/`: Cleaned FASTQ files.
- `results/2.1_read_classification/`: FASTQ files with target reads.
- `results/2.2_chimera_filtering/`: Target reads without chimeras.
- `results/2.3_downsampling/`: Downsampled target reads.
- `results/3_assemblies/`: Raw outputs for primary assemblers.
- `results/4_reassemblies/`: Outputs for secondary reassemblies.
- `results/6_final_assembly/`: The integrated final assembly files.
- `results/7_annotation/`: Diamond annotations for all assembled contigs.
- `results/8_stats_and_qc/`: Detailed CheckV, QUAST, and mapping logs.
- `results/report/final_summary_report/`: The interactive HTML benchmark report.

# Databases

ViruBench requires two external databases:
1. [DIAMOND](https://github.com/bbuchfink/diamond/wiki): A protein database (e.g., RVDB or NCBI Viral) formatted with diamond makedb.
2. [CheckV](https://bitbucket.org/berkeleylab/checkv): The standard CheckV database.
