# Part 1: RStudio SETUP

## Step 1. Open RStudio Environment and Configure Compute Settings

1. Click on the name of your Workspace.
2. Click on the cloud icon on the far right to view Cloud Environment options
3. In the dialogue box, click the "Settings" button under RStudio
4. You will see some default configuration options for the RStudio cloud environment, including a list of costs. Go ahead and **change RStudio Environment settings** to the following:

| Parameter | Selection |
|:-- | :-- |
| Application | RStudio |
| CPUs | 8 |
| Memory | 52 GB |
| GPU Configuration | Enable GPUs (Tobble **ON**)
| GPU Type | NVIDIA Tesla T4 |
| Number of GPUs | 1 |

5. Review the estimated hourly running cost displayed in the dialog, then scroll down and click **CREATE**

## Step 2. Launch and Open RStudio

1. It will take a few minutes for AnVIL to activate your cloud environment (request cloud instances and configure GPU drivers), so please wait for provisioning
2. When your environment is ready, its status will change to **Green (Running)**
3. To launch, click the RStudio icon, then click **Open**. RStudio will open in a new browser tab. 


# Part 2: Install packages, load libraries and get data

## Step 1. Install packages

```
# AnVIL packages
BiocManager::install("AnVILGCP")

# Bioconductor packages (install via BiocManager)
BiocManager::install("DESeq2")

# AI integration packages
install.packages(c("gander", "ellmer"))
```

## Step 2. Load Libraries

```
library(AnVILGCP)
library(tidyverse)
library(DESeq2)
library(gander)
library(ellmer)
```

## Step 3. Import data from google cloud

```
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/airwaycounts.csv . " )
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/sample_metadata.csv . " )
```

## Step 4. Load datasets for DESeq2 analysis

```
counts <- read.csv("airwaycounts.csv", row.names = 1, check.names = FALSE)
metadata <- read.csv("sample_metadata.csv", row.names = 1 )
```

# Part 3: Local AI Setup

## Step 1. Navigate to the RStudio Terminal

*Terminal tab is located next to the R Console tab on the left*

> [!IMPORTANT]
> Execute following command in the Terminal (not inside R Console)

## Step 2. Start Ollama server and download the model

```
# Create a directory and downolad the local Ollama 
mkdir ollama
curl -fsSL https://github.com/ollama/ollama/releases/download/v0.24.0/ollama-linux-amd64.tar.zst | tar x --zstd -C ollama

# Start the Ollama server
  # Runs the background service for local model hosting

ollama/bin/ollama

# Download the AI Model
ollama/bin/ollama pull qwen3-coder
```

## Step 3. Connect R to the Local AI server

*Switch from Terminal back to the R Console*

> [!IMPORTANT]
> Execute following commands inside R Console (not in Terminal)

```
# Connect R to your local Ollama instance
chat <- chat_ollama(
  base_url = Sys.getenv("ollama/bin/ollama", "http://localhost:11434"),
  model = "qwen3-coder",
)
```

```
# Test the local AI Connection - Ask AI a question!
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

# Part 3: DESeq2 GANDER

> [!NOTE]
> **A MUST for Data Analysis with Gander: Minimize Context Noise**
> 
> For this section, you MUST create a new .R script (open a **fresh `.R` script**) containing only the necessary inputs to keep Gander focused and avoid cluttering its context window**:
> 1. `read.csv()' for counts
> 2. `read.csv()`for metadata
> 3. The target R object for Gander to analyze (e.g. [imported] counts)
>
> *Keeping the script lean prevents background code clutter from interfering with Gander's responses.*

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

# Part 4: Data Analysis

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
