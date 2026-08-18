# Part 1: SETUP

## Step 1. RStudio Setup

RStudio Cloud Environment Setup
- CPUs: 8
- Memory (GB): 52
- [ ] Enable GPUs (NVIDIA Tesla T4, 1 GPU)


Install packages

```
# AnVIL packages
BiocManager::install("AnVILGCP")

# Bioconductor packages (install via BiocManager)
BiocManager::install("DESeq2")

# AI integration packages
install.packages(c("gander", "ellmer"))
```

Load Libraries

```
library(AnVILGCP)
library(tidyverse)
library(DESeq2)
library(gander)
library(ellmer)
```

Import data from google cloud

```
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/airwaycounts.csv . " )
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/sample_metadata.csv . " )
```

Load datasets for DESeq2 analysis

```
counts <- read.csv("airwaycounts.csv", row.names = 1, check.names = FALSE)
metadata <- read.csv("sample_metadata.csv", row.names = 1 )
```

## Step 2. AI Setup

> [!IMPORTANT]
> Execute following command in the Terminal (not inside R Console)

```
# Download locally and extract ollama
mkdir ollama
curl -fsSL https://github.com/ollama/ollama/releases/download/v0.24.0/ollama-linux-amd64.tar.zst | tar x --zstd -C ollama

# Launch the local ollama server
ollama/bin/ollama

# Pull the AI Model
ollama/bin/ollama pull qwen3-coder
```

## Step 3. Connect R to the AI model

> [!IMPORTANT]
> Execute following command inside R Console (not in Terminal)

```
chat <- chat_ollama(
  base_url = Sys.getenv("ollama/bin/ollama", "http://localhost:11434"),
  model = "qwen3-coder",
)
```

```
#Ask AI a question
chat$chat("Tell me one fact about bacterial genome")
```

## Step 4. Configure gander in RStudio 

Make keyboard shortcut for gander

**In RStudio: Navigate to Tools → Modify Keyboard Shortcuts… → search for “gander” → assign Shift+Cmd+g**

```
#Set gander's default chat model
options(gander.chat = chat)
```

## Step 5. Begin gandering in RStudio...

-----

# Part 2: DATA ANALYSIS USING GANDER

> [!NOTE]
> **For this part, you MUST create a new .R script that has only three commands: the two read.csv() and the object you want gander to manipulate - to keep Gander focused and avoid cluttering its context window**

## Protocol for using a gander shortcut:

> [!IMPORTANT]
> 1. HIGHLIGHT an object: 'counts' 
> 2. Evoke gander with your pre-set shortcut *Shift+Cmd+g*
> 3. ENTER PROMPT: Key information users need to know to achieve their goal.
> 4. *Wait for gander output*

# Explore counts and metadata

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Describe my counts dataset \
> PROMPT: Summarize my counts dataset

```
counts
```

See what context gander saw

```
gander_peek()
```

> [!IMPORTANT]
> HIGHLIGHT: 'metadata' \
> PROMPT: Describe my metadata file 
```
metadata
```

# Filter counts 
> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Filter genes with ≥ 1 counts in all samples, create a new counts object, and summarize

```
counts
```

:eyes: **sample gander output**

```
counts_filtered <- counts[rowSums(counts >= 1) == ncol(counts), ]
summary(counts_filtered)
```

# See what context gander 'saw'

```
gander_peek()
```

# Run DESeq2 on filtered counts 
> [!IMPORTANT]
> HIGHLIGHT: 'filtered_counts' AND 'metadata'
>     # the name of the filtered output file can vary (e.g. counts_filtered or filtered_counts or other) 
>     # highlighting both objects ensures gander views both files 
> PROMPT: Perform differential expression with filtered counts using DESeq2 and `dex` as the design condition and save the results in a new object

```
filtered counts
metadata
```

```
gander_peek()
```

:eyes: **sample gander output**

```
library(DESeq2)

# Create DESeq dataset
dds <- DESeqDataSetFromMatrix(countData = counts_filtered,
                              colData = metadata,
                              design = ~ dex)

# Run DESeq
dds <- DESeq(dds)

# Get results
results <- results(dds, contrast = c("dex", "trt", "untrt"))
```

# Examine significant genes

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on padj

:eyes: **sample gander output**

```
top_10_genes <- results[order(results$padj), ][1:10, ]
top_10_genes
```

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on fold change

:eyes: **sample gander output**

```
top_10_genes <- results[order(abs(results$log2FoldChange), decreasing = TRUE), ][1:10, ]
top_10_genes
```

# Convert Ensemble ids into gene names
> HIGHLIGHT: 'results'
> PROMPT: Convert ENSEMBL IDS to gene symbols and view first 10 gene symbols

:eyes: **sample gander output**

```
library(org.Hs.eg.db)
results_genes <- as.data.frame(results)
results_genes$external_gene_name <- mapIds(org.Hs.eg.db, keys=rownames(results_genes), column="SYMBOL", keytype="ENSEMBL", multiVals="first")
head(results_genes$external_gene_name, 10)
```

> [!CAUTION]
> May get an ERROR due to missing the required library 'org.Hs.eg.db'. 
> Can choose to manually install with `install.packages("BiocManager")`, or, use gander to troubleshoot!

Install the required package

```
BiocManager::install("DESeq2")
```

then, re-run gander-suggested code

```
library(org.Hs.eg.db)
results_genes <- as.data.frame(results)
results_genes$external_gene_name <- mapIds(org.Hs.eg.db, keys=rownames(results_genes), column="SYMBOL", keytype="ENSEMBL", multiVals="first")
head(results_genes$external_gene_name, 10)
```

# Create an MA plot

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Create an MA plot using DESeq2

:eyes: **sample gander output**

```
library(ggplot2)
plotMA(results, main="MA Plot", ylim=c(-5,5))
```

# Perform gene set enrichment analysis

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Perform gene set enrichment analysis

:eyes: **sample gander output**

```
library(clusterProfiler)
library(org.Hs.eg.db)

# Convert ENSEMBL IDs to gene symbols
gene_symbols <- mapIds(org.Hs.eg.db, 
                       keys=rownames(results), 
                       column="SYMBOL", 
                       keytype="ENSEMBL", 
                       multiVals="first")

# Filter for significant genes (padj < 0.05)
sig_genes <- names(which(results$padj < 0.05))

# Remove NA symbols
sig_genes <- sig_genes[!is.na(gene_symbols[sig_genes])]

# Perform GO enrichment analysis
go_enrichment <- enrichGO(gene = sig_genes,
                          orgDb = org.Hs.eg.db,
                          keyType = "SYMBOL",
                          ont = "BP",
                          pvalueCutoff = 0.05,
                          qvalueCutoff = 0.1)

# View results
go_enrichment
```

> [!CAUTION]
> May get an ERROR due to missing the required library 'clusterProfiler'. 
> Can choose to manually install with `BiocManager::install("clusterProfiler")`, or, use gander to troubleshoot!

Install the missing package

```
BiocManager::install("clusterProfiler")
```

re-run gander-suggested code

```
library(clusterProfiler)
library(org.Hs.eg.db)

# Convert ENSEMBL IDs to gene symbols
gene_symbols <- mapIds(org.Hs.eg.db, 
                       keys=rownames(results), 
                       column="SYMBOL", 
                       keytype="ENSEMBL", 
                       multiVals="first")

# Filter for significant genes (padj < 0.05)
sig_genes <- names(which(results$padj < 0.05))

# Remove NA symbols
sig_genes <- sig_genes[!is.na(gene_symbols[sig_genes])]

# Perform GO enrichment analysis
go_enrichment <- enrichGO(gene = sig_genes,
                          orgDb = org.Hs.eg.db,
                          keyType = "SYMBOL",
                          ont = "BP",
                          pvalueCutoff = 0.05,
                          qvalueCutoff = 0.1)

# View results
go_enrichment
```
### Currently failing at go_enrichment...need to troublshoot
