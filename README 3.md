# RNA-Seq Reference-Based Analysis — *Drosophila melanogaster*

This repo has all the results from a reference-based RNA-Seq pipeline I ran as part of a Bioinformatics course assignment. The experiment looks at what happens to gene expression in *Drosophila melanogaster* when the **pasilla** gene is knocked down via RNAi — basically, treated (knockdown) vs untreated flies, and which genes respond.

Everything was done on [Galaxy (usegalaxy.eu)](https://usegalaxy.eu) so there's no local code to run — just the outputs, plots, and the files needed to reproduce it.

---

## What's the biological question here?

The *pasilla* (*ps*) gene in fruit flies encodes an RNA-binding protein that's closely related to **NOVA1 and NOVA2** in humans — proteins involved in neuronal splicing and linked to neurological conditions. The dataset is from [Brooks et al. (2011)](https://doi.org/10.1101/gr.108662.110), who used RNAi to knock down *ps* and studied the downstream effects on gene expression. It's a well-known benchmark dataset in RNA-Seq analysis, which is also why it's great for learning the full pipeline end-to-end.

The 7 samples span both **single-end and paired-end** libraries across treated and untreated conditions — which adds a real-world complexity (you have to account for library type as a covariate in DESeq2, otherwise it messes with your results).

---

## Dataset

All data is publicly available from Zenodo: [record 6457007](https://zenodo.org/record/6457007). Download links are in `data/data_links.txt`.

| Sample | Condition | Library |
|--------|-----------|---------|
| GSM461176 | Untreated | Single-end |
| GSM461177 | Untreated | Paired-end |
| GSM461178 | Untreated | Paired-end |
| GSM461179 | Treated | Single-end |
| GSM461180 | Treated | Paired-end |
| GSM461181 | Treated | Paired-end |
| GSM461182 | Untreated | Single-end |

For the QC and mapping steps, I used subsampled FASTQ files (~5MB each) for GSM461177 and GSM461180. For DESeq2, I used pre-computed featureCounts files for all 7 samples.

Reference genome: *D. melanogaster* **dm6** (BDGP6.32), Ensembl GTF v109.

---

## Pipeline

The analysis goes through four main stages:

```
Raw FASTQs → QC & Trimming → Reference Mapping → Gene Counting → Differential Expression + GO Enrichment
```

Here's the breakdown:

### 1. Quality Control (`results/01_qc/`)

First step is always checking your data before you do anything with it. I ran **Falco** (basically a faster FastQC) on the raw reads and pulled everything together with **MultiQC** to get a clear summary across samples. Then trimmed adapters with **Cutadapt** and ran QC again on the cleaned reads to confirm things looked better.

The plots here cover the usual suspects — per-base quality scores, GC content, duplication levels, sequence counts, and the cutadapt filtering summary. Everything looked clean enough to proceed.

### 2. Read Mapping (`results/02_mapping/`)

Trimmed reads were aligned to the dm6 reference genome using **RNA-STAR**. You need a splice-aware aligner for RNA-Seq because reads can span exon-exon junctions that don't exist in the genomic sequence — a regular DNA aligner would just throw those reads out. STAR handles this well.

The output plots (via MultiQC) show mapping rates per sample. Uniquely mapped reads were consistently high, which means the data quality is solid and there aren't major contamination issues.

### 3. Read Counting (`results/03_counts/`)

Once you have aligned BAM files, **featureCounts** goes through them and counts how many reads overlap each annotated gene using the Ensembl GTF. This gives you the raw count matrix that goes into DESeq2.

The assignment plot here breaks down reads into: assigned to a gene, unassigned (ambiguous), unassigned (no overlapping feature), etc. Most reads were successfully assigned, which is what you want.

### 4. Differential Expression & GO Enrichment (`results/04_deseq2/`)

This is where the actual biology comes out. **DESeq2** takes the count matrix for all 7 samples and tests for differential expression between treated and untreated, while also modeling library type (SE vs PE) as a covariate so it doesn't confound the results.

**What the DESeq2 plots show:**

- **PCA plot** — PC1 (48% variance) separates treated from untreated cleanly. PC2 (33%) separates single-end from paired-end — exactly what you'd expect. The experimental design is working.
- **Sample-distance heatmap** — treated samples cluster together, untreated samples cluster together, replicates are similar to each other. Good reproducibility.
- **Dispersion estimates** — the shrinkage is working correctly, dispersion decreases as mean count increases (standard RNA-Seq behavior).

**GO enrichment (GOseq)** was run on the significant DE genes to find over-represented Gene Ontology terms. GOseq is important here rather than a naive Fisher test because it corrects for the fact that longer genes are more likely to be called DE just due to higher counts — a common bias in RNA-Seq. Enriched terms included:

- *Extracellular region* (CC)
- *Response to stress*, *glycogen/glucan biosynthetic process*, *septate junction assembly*, *establishment of blood-brain barrier* (BP)
- *Oxidoreductase activity*, *glutathione transferase activity* (MF)

These make biological sense — *pasilla* knockdown affects stress response and metabolic pathways, consistent with its known regulatory role.

**Heatmaps** (`heatmap_normalized.pdf` and `heatmap_zscore.pdf`) visualize the top DE genes across all samples. The normalized count heatmap shows absolute expression levels; the z-score version scales per gene so you can see the relative pattern more clearly without high-expression genes dominating the color scale.

---

## Repo Structure

```
RNA-Seq-Reference-Based-Analysis/
│
├── data/
│   └── data_links.txt           # all input file URLs (Zenodo)
│
├── results/
│   ├── 01_qc/                   # FastQC/MultiQC plots, Cutadapt summary
│   ├── 02_mapping/              # STAR alignment stats and plots
│   ├── 03_counts/               # featureCounts assignment plots
│   └── 04_deseq2/               # PCA, heatmaps, GO enrichment (PDFs)
│
├── requirements.txt             # Galaxy tool names and versions
├── .gitignore                   # excludes raw data files (FASTQs, BAMs, etc.)
└── README.md
```

Raw data files (FASTQs, BAMs, count files) are excluded from the repo via `.gitignore` since they're large — use the links in `data/data_links.txt` to get them.

---

## Tools Used

All tools run on Galaxy — no local setup needed.

| Tool | Version | Used For |
|------|---------|----------|
| Falco | 1.2.4+galaxy0 | Raw read QC |
| MultiQC | 1.27+galaxy4 | QC report aggregation |
| Cutadapt | 5.2+galaxy0 | Adapter trimming |
| RNA-STAR | 2.7.11b+galaxy0 | Splice-aware alignment |
| featureCounts | 2.1.1+galaxy0 | Gene-level read counting |
| DESeq2 | 2.11.40.8+galaxy0 | Differential expression |
| GOseq | 1.50.0+galaxy0 | GO enrichment analysis |
| heatmap2 | 3.2.0+galaxy1 | Heatmap visualization |

Full list in `requirements.txt`.

---

## How to Reproduce

1. Make a free account on [usegalaxy.eu](https://usegalaxy.eu)
2. Grab the input files from the links in `data/data_links.txt` and upload them to a new Galaxy history
3. Run the steps in order using the tool versions above:
   - Falco → MultiQC → Cutadapt → Falco again → MultiQC (QC)
   - RNA-STAR with dm6 + GTF (mapping)
   - featureCounts on the BAM outputs (counting)
   - DESeq2 on all 7 count files with treatment as the main factor and library type as covariate
   - Annotate DESeq2 output → filter significant genes → GOseq → heatmap2

---

## References

- Brooks et al. (2011). Conservation of an RNA regulatory map between *Drosophila* and mammals. *Genome Research* 21(2):193–202.
- Dobin et al. (2013). STAR: ultrafast universal RNA-seq aligner. *Bioinformatics* 29(1):15–21.
- Liao, Smyth & Shi (2014). featureCounts: an efficient general purpose program for assigning sequence reads to genomic features. *Bioinformatics* 30(7):923–930.
- Love, Huber & Anders (2014). Moderated estimation of fold change and dispersion for RNA-seq data with DESeq2. *Genome Biology* 15:550.
- Young et al. (2010). Gene ontology analysis for RNA-seq: accounting for selection bias. *Genome Biology* 11:R14.
